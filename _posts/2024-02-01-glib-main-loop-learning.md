---
layout: post
title: "glib main loop learning"
---

今天继续学习 glib: main-loop, event-loop

---
## talk is cheap ...
```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <glib.h>

/*
 * compile:
     gcc `pkg-config --cflags --libs glib-2.0` -Wall glib-main-loop-learn.c
 */

/* -------------------------------------------------------------------------- */
/* example 1: timer loop */
GMainLoop *Loop;
gint Counter = 128;

gboolean callback_timeo(gpointer data);

/*
 * 功能: 超时回调函数，调用计数超过 Counter 后，注销定时器
 */
gboolean callback_timeo(gpointer data)
{
	g_print(".");
	if ((--Counter%32) == 0)
		g_print("\n");

	if (Counter == 0) {
		//退出循环
		g_main_loop_quit(Loop);

		//注销定时器
		return FALSE;
	}
	//定时器继续运行
	return TRUE;
}

/*
 * 功能: 开启主循环，设置定时任务，在回调 Counter 次后 退出主循环
 * 解析: 主线程跑起来后 (g_main_loop_run) 程序会一直阻塞等待定时器触发回调，
 *       直到显式调用 g_main_loop_quit. 主循环退出后，整个程序就结束了
 */
int main01()
{
	//- 新建一个 glib 主循环
	Loop = g_main_loop_new(NULL, FALSE);

	//- 设置定时器, 每 0.1 秒调用 callback_timeo
	g_timeout_add(100, callback_timeo, NULL);

	//- 让循环体跑起来
	g_main_loop_run(Loop);
	g_main_loop_unref(Loop);

	return 0;
}

/* -------------------------------------------------------------------------- */
/* example 2: event loop */
GMainContext *Context;

gboolean callback_stdin(GIOChannel *channel);
void add_stdin_source(GMainContext *context);
gboolean callback_timeo2(gpointer data);
void add_timeo_source(GMainContext *context);

/*
 * 功能: stdin 有数据可读时的回调函数: 读一行字符串，然后反转输出
 */
gboolean callback_stdin(GIOChannel *channel)
{
	gchar* str;
	gsize len;

	//从 channel(stdin) 读取一行字符串
	g_io_channel_read_line(channel, &str, &len, NULL, NULL);

	//去掉回车键
	while (len > 0 && (str[len-1] == '\r' || str[len-1] == '\n'))
		str[--len]='\0';

	//反转字符串
	for (; len > 0; len--)
		g_print("%c", str[len-1]);
	g_print("\n");

	//判断结束符
	if (strcasecmp(str, "q") == 0) {
		g_main_loop_quit(Loop);
	}
	g_free(str);

	return G_SOURCE_CONTINUE;  //or G_SOURCE_REMOVE if want only trigger once
}

/* event loop
 * - 所有需要异步操作的地方都可以用event loop
 * - event loop 的三个基本结构：GMainLoop, GMainContext 和 GSource
 * 关系：GMainLoop -> GMainContext -> {GSource1, GSource2, GSource3......}
 * GSource 是具体的各种Event处理逻辑。可以把 GMainContext 理解为 GSource 的容器
 * 函数 g_source_attach: 把 GSource 加到 GMainContext
 */

/* ref/unref
 * - 具有引用计数功能的对象一般都会提供两个函数:
 *   `-> ref 用于增加引用计数，unref 用于减少引用计数，计数为 0 时销毁对象
 * - 引用计数是追踪对象生命周期最常用的方法，一方面保证对象在有人使用时不会被销毁
 *   `-> 另外一方面又保证不会因为忘记销毁对象而造成内存泄漏
 */
void add_stdin_source(GMainContext *context)
{
	GIOChannel* channel;
	GSource* source;

	//create a GIOChannel -> stdin
	channel = g_io_channel_unix_new(fileno(stdin));

	//G_IO_IN 表示监视 channel 的可读取状态
	source = g_io_create_watch(channel, G_IO_IN);
	g_io_channel_unref(channel);

	//为 GSource 添加回调参数，以及回调函数的参数
	g_source_set_callback(source, (GSourceFunc)callback_stdin, channel, NULL);

	//把 GSource 附加到 GMainContext
	g_source_attach(source, context);
	g_source_unref(source);
}

gboolean callback_timeo2(gpointer data)
{
	g_print(".");
	if ((--Counter % 32) == 0)
		g_print("\n");

	if (Counter == 0) {
		return G_SOURCE_REMOVE;   //注销定时器
	}
	return G_SOURCE_CONTINUE;   //定时器继续运行
}

void add_timeo_source(GMainContext *context)
{
	GSource* source;

	source = g_timeout_source_new(100);
	g_source_set_callback(source, (GSourceFunc)callback_timeo2, NULL, NULL);

	g_source_attach(source, context);
	g_source_unref(source);
}

/* main */
int main(int argc, char **argv)
{
	if (argc == 1 || strcasecmp(argv[1], "-h") == 0) {
		g_print("Usage: %s <1|2|3>  #1. timeout, 2. event loop, 3. both\n", argv[0]);
		return 0;
	}

	if (strcasecmp(argv[1], "1") == 0)
		return main01();

	//新建一个 GMainContext
	Context = g_main_context_new();

	//attatch a stdin read GSource to Context
	add_stdin_source(Context);

	//attatch a time-out GSource to Context
	if (argc > 2 || strcasecmp(argv[1], "3") == 0) {
		add_timeo_source(Context);
	}

	//create GMainLoop with context above
	Loop = g_main_loop_new(Context, FALSE);

	g_print("input string('q' to quit):\n");

	g_main_loop_run(Loop);

	g_main_loop_unref(Loop);
	g_main_context_unref(Context);

	return 0;
}
```
