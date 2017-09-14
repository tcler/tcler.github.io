---
layout: post
title: "systemtap 修改系统调用参数"
---

折腾了半天，终于搞明白怎么用 systemtap 修改函数出参了，总结一下学到的东西:
1. 依赖: yum install yum-utils dnf-utils && debuginfo-install kernel
2. 如果 probe 是 function() , 可以直接用 $varname 获取参数或当前上下文中的变量值
3. ~~如果 probe 是 function().return , 没有办法获取参数和变量的值, 只能直接修改 $return~~
4. ~~ 但是可以利用全局变量 在 probe function() 上下文中将 想要修改的参数地址保存, 然后在 function().return 中修改 ~~
5. 更正: 经过验证上面 3 的说法是错的, 发现其实是 return probe 里写法不同,需要写成 @entry($var) @entry($$vars->$)
``` 
probe syscall.statfs.return {
        if (kernel_string(@entry($pathname)) == @1) {
                printf("end: $$vars$ = %s\n", @entry($$vars->$))
                @cast(@entry($buf), "struct statfs")->f_blocks = $2;
                @cast(@entry($buf), "struct statfs")->f_bavail = 0;
                @cast(@entry($buf), "struct statfs")->f_bfree = 0;
        }
}
```
6. tip: 字符串参数无法直接打印, 需要 kernel_string()/user_string() 转换
7. tip: 类型无法知道, 可以用 @cast(var, "type") 做转换
8. tip: @cast(var, "char *") 不可用, 还是用 kernel_string()/user_string() 吧
9. tip: $$parms$ 参数列表, $$vars$ 变量列表(包括参数和其他变量)
10. 命令行参数：数值型 $1 $2 $3 ..., 字符串型 @1 @2 @3 ...;  $  @ 只表示类型
11. -G NAME=VALUE 命令行定义全局变量
12. -c 直接调用测试用的命令
13. 疑问: 笔误写成 printf("%s\n", path)  竟然没有报错，还打印出了 pathname 的字符串值(外面多了"") ???

下面是最终折腾出来的例子(修改特定 mountpoint statfs() 的返回值,看df输出值边界)
```
#!/usr/bin/stap -vg

# How to get parameters of function/syscall
# (1) $varname or @var("varname") or @var("varname@src/file.c")
# (2) https://sourceware.org/systemtap/langref/Probe_points.html #DWARF-less probing
#   '''
#   In the absence of debugging information, ...
#
#     asmlinkage ssize_t sys_read(unsigned int fd, char __user *buf, size_t count);
#
#   You can obtain the values of fd, buf, and count, respectively, as uint_arg(1),
#   pointer_arg(2), and ulong_arg(3).
#   In this case, your probe code must first call asmlinkage(), because on some
#   architectures the asmlinkage attribute affects how the function’s arguments are passed.
#   '''

# because can not printf pointer_arg() as string
# `-> I've solved this problem by use kernel_string()
function printstr (indent, name, var) %{
        STAP_PRINTF("%sEmbedded C STAP_PRINTF: %s = %s\n", STAP_ARG_indent, STAP_ARG_name, STAP_ARG_var);
%}

#global testpath = "/mnt/image"
global testpath = @1
global intestpath = ""

global statfs_path
global statfs_buf

probe syscall.statfs
{
        printf("\n")
        printf("statfs begin: $$vars$(%s)\n", $$vars$)
        printf("statfs begin: $$parms$(%s)\n", $$parms$)
        statfs_path = pointer_arg(1)
        path_str = user_string(statfs_path);
        if (path_str == testpath) {
                intestpath = testpath

                printf("Start ----------------\n")
                statfs_buf = @var("buf");

                # @var("varname") vs pointer_arg(position)
                printf("  [debug]@var(\"pathname\")=%d pointer_arg(1)=%d\n", @var("pathname"), pointer_arg(1))

                # output pathname as string
                printstr("  [debug]", "$pathname", $pathname)
                printf("  [debug]systemtap printf: kernel_string($pathname) = %s\n", kernel_string($pathname))
        }
}
probe syscall.statfs.return
{
        if (intestpath == testpath) {
                printf("  return: Modify statfs(%s) at return point\n", kernel_string(statfs_path));
                #@cast(statfs_buf, "struct statfs")->f_blocks = 9223372036854775807; #2^63-1
                #@cast(statfs_buf, "struct statfs")->f_blocks = 18446744073709551615; #2^64-1
                @cast(statfs_buf, "struct statfs")->f_blocks = $2;
                @cast(statfs_buf, "struct statfs")->f_bavail = 0;
                @cast(statfs_buf, "struct statfs")->f_bfree = 0;
                intestpath = "";
                printf("END\n")
        }
}
```
```
[yjh@ws nfs]$ LANG=C sudo stap -g ./statfs.stp /mnt/image 18446744073709551615  -c 'df /mnt/image'
Filesystem                1K-blocks  Used Available Use% Mounted on
10.66.12.250:/nfs_nospace         -     -         0    - /mnt/image

statfs begin: $$vars$(pathname=140726166415397 buf=140726166408800 ret=137)
statfs begin: $$parms$(pathname=140726166415397 buf=140726166408800)
Start ----------------
  [debug]@var("pathname")=140726166415397 pointer_arg(1)=140726166415397
  [debug]Embedded C STAP_PRINTF: $pathname = /mnt/image
  [debug]systemtap printf: kernel_string($pathname) = /mnt/image
  return: Modify statfs(/mnt/image) at return point
END
```
```
[yjh@ws nfs]$ LANG=C sudo ./statfs.stp /mnt/image 18446744073709551615  -c 'df /mnt/image'
Pass 1: parsed user script and 473 library scripts using 138512virt/47712res/7012shr/41020data kb, in 90usr/10sys/99real ms.
Pass 2: analyzed script: 5 probes, 15 functions, 6 embeds, 6 globals using 325588virt/236280res/8388shr/228096data kb, in 1150usr/140sys/1326real ms.
Pass 3: using cached /root/.systemtap/cache/82/stap_82b979c3e159cdbe9e88995301d158c6_22624.c
Pass 4: using cached /root/.systemtap/cache/82/stap_82b979c3e159cdbe9e88995301d158c6_22624.ko
Pass 5: starting run.
Filesystem                1K-blocks  Used Available Use% Mounted on
10.66.12.250:/nfs_nospace         -     -         0    - /mnt/image

statfs begin: $$vars$(pathname=140730627438659 buf=140730627434544 ret=137)
statfs begin: $$parms$(pathname=140730627438659 buf=140730627434544)
Start ----------------
  [debug]@var("pathname")=140730627438659 pointer_arg(1)=140730627438659
  [debug]Embedded C STAP_PRINTF: $pathname = /mnt/image
  [debug]systemtap printf: kernel_string($pathname) = /mnt/image
  return: Modify statfs(/mnt/image) at return point
END
Pass 5: run completed in 10usr/30sys/390real ms.
```

上面是学习调试过程产生的充满注释、验证代码、debug代码的例子，简化后的代码:

修改出参 struct statfs \*buf
```
[yjh@ws nfs]$ cat statfs.stp
#!/usr/bin/stap -vg

probe syscall.statfs.return {
        if (kernel_string(@entry($pathname)) == @1) {
                #printf("end: $$vars$ = %s\n", @entry($$vars->$))
                @cast(@entry($buf), "struct statfs")->f_blocks = $2;
                @cast(@entry($buf), "struct statfs")->f_bavail = 0;
                @cast(@entry($buf), "struct statfs")->f_bfree = 0;
        }
}
[yjh@ws nfs]$ groovy -e 'println new BigInteger(2).pow(64)'
18446744073709551616
[yjh@ws nfs]$ LANG=C sudo stap -g ./statfs.stp /mnt/image 18446744073709551615  -c 'df /mnt/image'
Filesystem                1K-blocks  Used Available Use% Mounted on
10.66.12.250:/nfs_nospace         -     -         0    - /mnt/image
[yjh@ws nfs]$ LANG=C sudo stap -g ./statfs.stp /mnt/image 18446744073709551614  -c 'df /mnt/image'
Filesystem                1K-blocks  Used Available Use% Mounted on
10.66.12.250:/nfs_nospace         -     -         0    - /mnt/image
[yjh@ws nfs]$ LANG=C sudo stap -g ./statfs.stp /mnt/image 18446744073709551613  -c 'df /mnt/image'
Filesystem                              1K-blocks                    Used Available Use% Mounted on
10.66.12.250:/nfs_nospace 18889465931478580851712 18889465931478580851712         0 100% /mnt/image
```

修改入参 char \*pathname
```
[yjh@ws nfs]$ cat statfs2.stp
#!/usr/bin/stap -vg

function newpath(opath, npath) %{
        char *p = (char *)STAP_ARG_opath;
        strcpy(p, STAP_ARG_npath);
%}
probe syscall.statfs {
        if (kernel_string($pathname) == @1) {
                newpath($pathname, @2);
        }
}
[yjh@ws nfs]$ LANG=C sudo stap -g ./statfs2.stp /mnt/image  /boot  -c 'df -Th /mnt/image'
Filesystem                Type  Size  Used Avail Use% Mounted on
10.66.12.250:/nfs_nospace nfs4   23G   11G   11G  51% /mnt/image
                          #^^^ 奇怪,这里fstype没有受影响,但是
[yjh@ws nfs]$ LANG=C df -Th /boot
Filesystem              Type  Size  Used Avail Use% Mounted on
/dev/mapper/fedora-root ext4   23G   11G   11G  51% /
```

这次的例子只尝试了 kernel.function() 和 kernel.function().return; 只尝试了修改参数;

遗留问题: (等试过后再更新)
1. 如何修改 errno 值
2. 如何读取/修改 局部/栈 变量
3. .call .inline .label return.maxactive 等 probe 都是什么含义??
4. etc.

Tips: 所有探测点类型
```
kernel.function(PATTERN)
kernel.function(PATTERN).call
kernel.function(PATTERN).return
kernel.function(PATTERN).return.maxactive(VALUE)
kernel.function(PATTERN).inline
kernel.function(PATTERN).label(LPATTERN)
module(MPATTERN).function(PATTERN)
module(MPATTERN).function(PATTERN).call
module(MPATTERN).function(PATTERN).return.maxactive(VALUE)
module(MPATTERN).function(PATTERN).inline
kernel.statement(PATTERN)
kernel.statement(ADDRESS).absolute
module(MPATTERN).statement(PATTERN)
process(PROCESSPATH).function(PATTERN)
process(PROCESSPATH).function(PATTERN).call
process(PROCESSPATH).function(PATTERN).return
process(PROCESSPATH).function(PATTERN).inline
process(PROCESSPATH).statement(PATTERN)
```

Tips: 查看所有的函数探测点
```
stap -l 'kernel.function("*")'
stap -l 'module("nfs").function("*")'
stap -l 'module("nfs").function("*"), module("nfsd").function("*")'
stap -l 'module("nfs*").function("*")
stap -l 'syscall.*'
sudo debuginfo-install cifs-utils nfs-utils
stap -l 'process("/usr/sbin/mount.cifs").function("*")'
stap -l 'process("/usr/sbin/rpc.nfsd").function("*")'
```

Tips: stap -l 'syscall.\*' 只能看到系统调用名, 怎么看对应的函数定义呢?
```
[yjh@ws nfs]$ stap -l 'kernel.function("*")' | egrep -i '"(compat_)?sys_' | grep statfs
kernel.function("SyS_fstatfs64@fs/statfs.c:203")
kernel.function("SyS_fstatfs@fs/statfs.c:194")
kernel.function("SyS_statfs64@fs/statfs.c:182")
kernel.function("SyS_statfs@fs/statfs.c:173")
kernel.function("SyS_ustat@fs/statfs.c:229")
kernel.function("compat_SyS_fstatfs64@fs/statfs.c:347")
kernel.function("compat_SyS_fstatfs@fs/statfs.c:291")
kernel.function("compat_SyS_statfs64@fs/statfs.c:333")
kernel.function("compat_SyS_statfs@fs/statfs.c:282")
kernel.function("compat_SyS_ustat@fs/statfs.c:366")
[yjh@ws nfs]$ find /usr/src -name statfs.c
/usr/src/debug/kernel-4.12.fc26/linux-4.12.9-300.fc26.x86_64/fs/statfs.c
[yjh@ws nfs]$ rpm -qf /usr/src/debug/kernel-4.12.fc26/linux-4.12.9-300.fc26.x86_64/fs/statfs.c
kernel-debuginfo-common-x86_64-4.12.9-300.fc26.x86_64
```

Tips: Groovy 进制转换, BigInteger() 处理超过 2^64 的大数
```
    groovy -e 'println new BigInteger("18889465931478580851712").toString(2)' #?
    groovy -e 'println new BigInteger(2).pow(74).toString()'
    groovy -e 'println new BigInteger(2).pow(74).subtract(new BigInteger("18889465931478580851712"))' #3072
    groovy -e 'println Long.toString(184467440737095516, 16)' #different
```

Tips: Google bug: (18889465931478580854784 - 18889465931478580851712) 结果是 0

Ref: 这个总结的比较全面
```
SystemTap使用技巧
https://segmentfault.com/a/1190000010774974
```
