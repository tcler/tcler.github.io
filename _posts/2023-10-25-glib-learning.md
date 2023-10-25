---
layout: post
title: "glib learning (glib 入坑)"
---

## 简单记录一下 glib 入坑过程
最近参与 Restraint 项目改进，发现这个项目主要是基于 glib 开发的；之前也了解到 QEMU 项目中也使用了 glib。
但是仍然觉得，如果标准 C 能实现的功能 我干嘛非要用 glib 的封装呢，于是在一些简单功能中坚持使用 C 标准库函数，
标准 C 不支持的功能再使用 glib；然后奇怪的事情发生了：本地编译没问题，但是提交自动构建后，竟然失败，竟然提示
说 strdup() strtok_r() 不是标准 C 函数，原来他们只是 glibc 的扩展，[C23](https://en.wikipedia.org/wiki/C23_(C_standard_revision)) 版本上才支持(而目前C23还没发表)；

于是就改用 glib 的版本，用了几天发现确实真香，除了字符串处理函数，还有很多其他更高层级功能的封装，决定正式入坑 ; )  

[glib doc](https://docs.gtk.org/glib/)  

GLib is a general-purpose, portable utility library, which provides many useful data types, macros, type conversions, string utilities, file utilities, a mainloop abstraction, and so on.

---
## code example
刚开始学，只用了几个 string 处理函数，和一个文件判断函数，在这里简单记录一下:  
```
]$ cat ~/glib-example.c 
#include <stdio.h>
#include <glib.h>

//compile command:
//gcc `pkg-config --cflags --libs glib-2.0` -Wall me.c

static void _parse_fetch_options(gchar *fetch_opts)
{
    char *opts, *str, *token, *saveptr;
    if (fetch_opts == NULL)
        return;

    opts = (char *)g_strdup(fetch_opts);
    printf("optsptr = %p\n", opts);
    for (str = opts; ; str = NULL) {
        token = strtok_r(str, " ", &saveptr);
        if (token == NULL)
		break;
        if (0 == strncmp("retry=", token, 6)) {
            printf("-- retry=%d\n", atoi(&token[6]));
        } else if (0 == strncmp("timeo=", token, 6)) {
            printf("-- timeo=%d\n", atoi(&token[6]));
        } else if (0 == strncmp("abort_recipe_when_fetch_fail", token, 5)) {
            printf("-- abort str = %s\n", token);
        } else if (0 == strncmp("keepchanges", token, 11)) {
            printf("-- keepchanges str = %s\n", token);
        }
    }
    printf("optsptr = %p  after strtok_r\n", opts);
    g_free(opts);
}

static void parse_fetch_options(gchar *fetch_opts)
{
    gint i, j;
    gchar **opts, **optsj, *token, *tokenj;

    if (fetch_opts == NULL)
        return;

    opts = g_strsplit(fetch_opts, " ", 0);
    for (i = 0; token = opts[i], token != NULL; i++) {
        optsj = g_strsplit(token, ",", 0);
        for (j = 0; tokenj = optsj[j], tokenj != NULL; j++) {
            if (0 == strncmp("retry=", tokenj, 6)) {
                printf("- retry=%d\n", atoi(&tokenj[6]));
            } else if (0 == strncmp("timeo=", tokenj, 6)) {
                printf("- timeo=%d\n", atoi(&tokenj[6]));
            } else if (0 == strncmp("abort_recipe_when_fetch_fail", tokenj, 5)) {
                printf("- abort str = %s\n", tokenj);
            } else if (0 == strncmp("keepchanges", tokenj, 11)) {
                printf("- keepchanges str = %s\n", tokenj);
            } else {
                printf("- unexpected str = %s\n", tokenj);
            }
        }
        g_strfreev(optsj);
    }
    g_strfreev(opts);
}

int main(void) {
	gchar *str = "abc,def,xyz,hahaha";
	gchar *opts = "  abort_xx,timeo=88,retry=77  keepchanges=yes abc,def,xyz,hahaha";
	gchar **arr;
	gint i;

	printf("str: %s\n", str);
	arr = g_strsplit(str, ",", 0);
	for (i = 0; arr[i] != NULL; i++)
		printf("- arr[%d] = %s\n", i, arr[i]);
	printf("\n");

	parse_fetch_options(NULL);
	printf("opts: %s\n", opts);
	parse_fetch_options(opts);
	_parse_fetch_options(opts);
	printf("\n");

	gchar *basestr = "a b c d --fetch-opts=keepchanges,retry=6";
	gchar *substr = "fetch-opts=";
	printf("basestr: %s, substr: %s\n", basestr, substr);
	gchar *task_fetch_opts_str = g_strrstr(basestr, substr);
	printf("- strstr = %s\n", task_fetch_opts_str);
}
```

文件测试函数用法见: [g_file_test example](https://github.com/tcler/restraint/blob/master/src/task.c#L116)  
