---
layout: post
title:  "[转] 使用 kprobe / SystemTap 观察内核函数"
tags:   [kprobe,SystemTap]
author: github.com/ceyes
---

这两天测的一个kernel的bug总是无法重现，怀疑是方法不对，于是想到根据 Call Trace 和 patch，找到触发bug的那个函数，先确认自己的测试步骤是否有走到那个函数。顺便学习了kprobe 和 systemtap。

接下来以`ip_rcv`为实例，用两种方法实现“如果调用了这个函数，则打印'Hit it'”  
(arch=x86_64)

<!-- more -->

Kprobe
------

ref:

 1. <http://www.ibm.com/developerworks/cn/linux/l-cn-systemtap1/index.html>
 2. <http://lxr.free-electrons.com/source/samples/kprobes/>

kprobe是一个动态地收集调试和性能信息的工具，用户用它几乎可以跟踪任何函数或被执行的指令以及一些异步事件（如timer）。它的基本工作机制是：用户指定一个探测点，并把一个用户定义的处理函数关联到该探测点，当内核执行到该探测点时，相应的关联函数被执行，然后继续执行正常的代码路径。

kprobe实现了三种类型的探测点: kprobes, jprobes和kretprobes (也叫返回探测点)。 kprobes是可以被插入到内核的任何指令位置的探测点，jprobes则只能被插入到一个内核函数的入口，而kretprobes则是在指定的内核函数返回时才被执行。

一般使用kprobe的程序实现作一个内核模块，模块的初始化函数来负责安装探测点，退出函数卸载那些被安装的探测点。kprobe提供了接口函数来安装或卸载探测点。


kprobe.c:

```
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/kprobes.h>
#include <linux/kallsyms.h>

struct kprobe kp;

static int hander_pre(struct kprobe *p, struct pt_regs *regs)
{
	printk("Hit it\n");
	return 0;
}

static void hander_post(struct kprobe *p, struct pt_regs *regs,
			unsigned long flags)
{
}

static int __init kprobe_init(void)
{
	int ret;
	kp.pre_handler = hander_pre;
	kp.post_handler = hander_post;

	kp.addr = (kprobe_opcode_t *) (kprobe_opcode_t *) kallsyms_lookup_name("ip_rcv");
	/*
	 * old kernel not have kallsyms_lookup_name(), then we have to manually get the addr
	 * by `cat /proc/kallsyms | awk '/ip_rcv$/ {print $1}'`, finally don't forget add "0x"
	 * eg : kp.addr = 0xffffffff81493ce0;
	 */
	if (kp.addr == NULL) {
		return 1;
	}

	ret = register_kprobe(&kp);
	if (ret < 0) {
		printk(KERN_INFO "register_kprobe failed, returned %d\n", ret);
		return ret;
	}
	printk(KERN_INFO "Planted kprobe at %p\n", kp.addr);
	return 0;
}

static void __exit kprobe_exit(void)
{
	unregister_kprobe(&kp);
	printk(KERN_INFO "kprobe at %p unregistered\n", kp.addr);
}

module_init(kprobe_init);
module_exit(kprobe_exit);
MODULE_LICENSE("GPL");
```

Makefile:

```
obj-m := kprobe.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules
clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean


```


接下来要做的就是编译成模块、加载、卸载……，注意编译模块需要`yum install -y kernel-devel`


SystemTap
---------

ref:

 1. <http://www.ibm.com/developerworks/cn/linux/l-cn-systemtap3/index.html>
 2. <http://www.ibm.com/developerworks/cn/linux/l-systemtap/>
 3. <http://sourceware.org/systemtap/documentation.html>


### 安装

SystemTap 需要`kernel-debuginfo`，所以安装即:

	yum install -y systemtap kernel-debuginfo

可以用简单测试一下systemtap 是否安装成功，by

	stap -ve 'probe begin { log("hello world") exit() }'

### 简介

SystemTap的基本工作原理就是：event/handler，运行systemtap脚本产生的加载模块时刻监控事件的发生，一旦发生，内核就调用相关的handler处理。

SystemTap脚本以.stp为扩展名，其基本格式如下所示：

	probe event {statements}

允许一个探针内多个event，以,隔开，任一个event发生时，都会执行statements，各个语句之间不需要特殊的结束符号标记。而且可以在一个statements block中包含其他的statements block。
也可以使用函数：

	function function_name(arguments) {statements}
	probe event {function_name(arguments)}

Event: 探针，有许多预定义模式，分为同步和异步的:

* asynchronous
    <pre>
    begin                                      # 在脚本开始时触发
    end                                        # 在脚本结束时触发
    timer.jiffies(1000)                        # 每隔 1000 个内核 jiffy 触发一次
    timer.ms(200)                              # 每隔 200 毫秒触发一次
    timer.ms(200).randomize(50)                # 每隔 200 毫秒触发一次，带有线性分布的随机附加时间（-50 到 +50）
    </pre>

* synchronous
    <pre>
    kernel.function("sys_sync")                # 调用 sys_sync 时触发
    kernel.function("sys_sync").call           # 同上
    kernel.function("sys_sync").return         # 返回 sys_sync 时触发
    kernel.syscall.*                           # 进行任何系统调用时触发
    kernel.function("*@kernel/fork.c: 934")    # 到达 fork.c 的第 934 行时触发
    module("ext3").function("ext3_file_write") # 调用 ext3 write 函数时触发
    </pre>

Handler: 触发探针时需要执行的代码, 提供了许多有用的函数

* 
    <pre>
    log()
    printf()
    print_backtrace()
    println()
    tid()                                       # 当前线程ID
    uid()                                       # 当前用户ID
    cpu()                                       # 当前CPU号
    pid()                                       # 当前进程ID
    </pre>

### 使用

SystemTap的使用像写脚本一样很简单，有点像awk的感觉。

解决开始提出的问题：

```
 # cat << EOF > hit-ip_rcv.stp
probe kernel.function("ip_rcv")
{
	log("hit it")
}

EOF

 # stap  hit-ip_rcv.stp
 ...

```

也可以写成一行命令:

 * `stap -e 'probe kernel.function("ip_rcv") { log("hit it")}'`
 * `stap -e 'probe kernel.function("ip_rcv") { print_backtrace(); exit()}'`

So easy ~


END
