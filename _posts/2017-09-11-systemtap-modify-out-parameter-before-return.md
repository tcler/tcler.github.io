---
layout: post
title: "systemtap 修改系统调用输出参数"
---

折腾了半天，终于搞明白怎么用 systemtap 修改函数出参了，总结一下学到的东西:
1. probe 是 function() , 可以直接用 $varname 获取参数或当前上下文中的变量值
2. probe 是 function().return , 没有办法获取参数和变量的值, 只能直接修改 $return
3. 但是可以利用全局变量 在 probe function() 上下文中将 想要修改的参数地址保存, 然后在 function().return 中修改
4. tip: 字符串参数无法直接打印, 需要 kernel_string()/user_string() 转换
5. tip: 类型无法知道, 可以用 @cast(var, "type") 做转换
6. tip: @cast(var, "char *") 不可用, 还是用 kernel_string()/user_string() 吧

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
function printstr (str) %{
        STAP_PRINTF("print string: %s\n", STAP_ARG_str);
%}

global testpath = "/mnt/image"
global intestpath = ""

global statfs_path
global statfs_buf

probe syscall.statfs
{
        printf("statfs begin: $$parms$(%s)\n", $$parms$)
        statfs_path = pointer_arg(1)
        path_str = user_string(statfs_path);
        if (path_str == testpath) {
                intestpath = testpath

                printf("-------- Start --------\n")
                printf("pathname: @var(\"pathname\")=%d pointer_arg(1)=%d\n", @var("pathname"), pointer_arg(1))
                statfs_buf = @var("buf");

                # output pathname as string
                printstr(statfs_path)
                printf("pathname === %s\n", kernel_string(statfs_path))
        }
}
probe syscall.statfs.return
{
        if (intestpath == testpath) {
                printf("Modify statfs(%s) at return point\n", kernel_string(statfs_path));
                #@cast(statfs_buf, "struct statfs")->f_blocks = 9223372036854775807; #2^63-1
                @cast(statfs_buf, "struct statfs")->f_blocks = 18446744073709551615; #2^64-1
                @cast(statfs_buf, "struct statfs")->f_blocks = 18446744073709551613; #2^64-1-2
                @cast(statfs_buf, "struct statfs")->f_bavail = 0;
                @cast(statfs_buf, "struct statfs")->f_bfree = 0;
                intestpath = "";
                printf("END\n")
        }
}
```

Tips: Groovy 进制转换, BigInteger() 处理超过 2^64 的大数
```
    groovy -e 'println new BigInteger("18889465931478580851712").toString(2)' #?
    groovy -e 'println new BigInteger(2).pow(74).toString()'
    groovy -e 'println new BigInteger(2).pow(74).subtract(new BigInteger("18889465931478580851712"))' #3072
    groovy -e 'println Long.toString(184467440737095516, 16)' #different
```

Tips: Google bug: (18889465931478580854784 - 18889465931478580851712) 结果是 0
