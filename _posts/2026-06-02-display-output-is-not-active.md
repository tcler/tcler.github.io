---
layout: post
title: "display output is not active while qemu VM start"
---

## get "display output is not active" for a while
x86_64 HOST 用 virt-install(qemu-system-loongarch64) 启动不同架构的 linux iso 时发现过了 grub 后，VNC 客户端上
会出现短时间的(根据host性能或多或少) "display output is not active" 然后，OS 启动画面又出现，，然后尝试修改选项
--video=vga 后，这个“现象”就消失了。

## why ？
跟 XL 大概聊了一下，怀疑是因为指令模拟效率本身低 + virtio-gpu 模块加载导致的，但是如上所述 即使换了性能很好的 host
仍然有这个提示信息，只是时间缩短到1、2秒；  

然后尝试跟 DeepSeek 聊，给出的回答/猜测是，--video=vga 模式时，OS 不使用自带的驱动，而是使用中断调用 直接使用 BIOS 里
的 VGA 驱动，这样就不会重新初始化(虚拟)显卡，所以不会出现 VPN 中断~  

为了验证 AI 是不是胡乱猜测，又使用 --arch ppc64le 尝试启动 Alma-10 的 ppc64le qcow2 image，果然也出现了类似的情况，
然后在加上 --video=vga 选项后，"display output is not active" 提示就消失了~~ 看来 AI 说对了~

## 为什么以前没有发现？
以前都是自动化，没有在 VM 刚刚启动那个时间点去连 VPN 看；这次是在一个拆了大内存，只有 16G 内存的 x99 机器上，手工安装
loongarch64 的 iso 镜像，"display output is not active" 持续时间长 所以发现了~

---
没用的知识又增加了，，其实上学的时候 课本上提过 BIOS 里面的中断处理程序提供外设驱动，没想到 vga 模式，OS(至少linux) 是
直接复用 BIOS 你的驱动 ~~

---
按照习惯还是查看代码验证一下吧:
```
jiyin@max395:~/ws/tools/qemu-master$ grep -i "display output is not active" -r .
./ui/console.c:    static const char placeholder_msg[] = "Display output is not active.";
jiyin@max395:~/ws/tools/qemu-master$ vim ui/console.c

void qemu_console_set_surface(QemuConsole *con,
                             DisplaySurface *surface)
{
    static const char placeholder_msg[] = "Display output is not active.";
    DisplayState *s = con->ds;
    DisplaySurface *old_surface = con->surface;
    DisplaySurface *new_surface = surface;
    DisplayChangeListener *dcl;
    int width;
    int height;

    if (!surface) {
        if (old_surface) {
            width = surface_width(old_surface);
            height = surface_height(old_surface);
        } else {
            width = 640;
            height = 480;
        }

        new_surface = qemu_create_placeholder_surface(width, height, placeholder_msg);
    }

    assert(old_surface != new_surface);

    con->scanout.kind = SCANOUT_SURFACE;
    con->surface = new_surface;
    dpy_gfx_create_texture(con, new_surface);
    QLIST_FOREACH(dcl, &s->listeners, next) {
        if (con != dcl->con) {
            continue;
        }
        displaychangelistener_gfx_switch(dcl, new_surface, surface ? FALSE : TRUE);
    }
    dpy_gfx_destroy_texture(con, old_surface);
    qemu_free_displaysurface(old_surface);
}

jiyin@max395:~/ws/tools/qemu-master$ grep qemu_console_set_surface.*NULL -r .
./hw/display/vhost-user-gpu.c:            qemu_console_set_surface(con, NULL);
./hw/display/virtio-gpu-rutabaga.c:        qemu_console_set_surface(scanout->con, NULL);
./hw/display/virtio-gpu-rutabaga.c:    qemu_console_set_surface(scanout->con, NULL);
./hw/display/virtio-gpu-virgl.c:        qemu_console_set_surface(g->parent_obj.scanout[ss.scanout_id].con, NULL);
./hw/display/virtio-gpu-virgl.c:        qemu_console_set_surface(g->parent_obj.scanout[i].con, NULL);
./hw/display/virtio-gpu.c:    qemu_console_set_surface(scanout->con, NULL);
./hw/display/virtio-gpu.c:        qemu_console_set_surface(g->parent_obj.scanout[i].con, NULL);
jiyin@max395:~/ws/tools/qemu-master$ vim hw/display/virtio-gpu.c

static void virtio_gpu_reset_bh(void *opaque)
{
    VirtIOGPU *g = VIRTIO_GPU(opaque);
    VirtIOGPUClass *vgc = VIRTIO_GPU_GET_CLASS(g);
    struct virtio_gpu_simple_resource *res, *tmp;
    uint32_t resource_id;
    Error *local_err = NULL;
    int i = 0;

    QTAILQ_FOREACH_SAFE(res, &g->reslist, next, tmp) {
        resource_id = res->resource_id;
        vgc->resource_destroy(g, res, &local_err);
        if (local_err) {
            error_append_hint(&local_err, "%s: %s resource_destroy"
                              "for resource_id = %"PRIu32" failed.\n",
                              __func__, object_get_typename(OBJECT(g)),
                              resource_id);
            /* error_report_err frees the error object for us */
            error_report_err(local_err);
            local_err = NULL;
        }
    }

    for (i = 0; i < g->parent_obj.conf.max_outputs; i++) {
        qemu_console_set_surface(g->parent_obj.scanout[i].con, NULL);  //<--- 大概就是这里了
    }

    g->reset_finished = true;
    qemu_cond_signal(&g->reset_cond);
}

```
