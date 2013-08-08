---
layout: post
title: Redis源码阅读之设置进程的Title
---


Redis服务器在启动时会修改在ps命令中显示的cmd的显示内容(显示如下图)。

![image](/images/redis-setproctitle-res.png)

Redis在main函数中调用redisSetProcTitle(argv[0])方法，代码如下:
{%highlight c%}
void redisSetProcTitle(char *title) {
#ifdef USE_SETPROCTITLE
    setproctitle("%s %s:%d",
        title,
        server.bindaddr ? server.bindaddr : "*",
        server.port);
#else
    REDIS_NOTUSED(title);
#endif
}
{%endhighlight%}
setproctitle方法在setproctitle.c文件中，当os是NetBSD，FreeBSD，OpenBSD时直接调用系统的setproctitle。
当是Linux或者Mac Os时通过修改main的argv[0]的内容来改变显示的进程名称。在修改argv[0]的内容前必修先把argv和environ的内容copy到新的位置。

##深度探讨之Linux如何显示进程的名字
在Linux下可以通过cat /proc/pid/cmdline 获取进程的cmdline。
查找Linux代码发现是通过读取程序的main函数的argv[0]来实现的，也就是上面的setproctitle的实现的原理。
代码见Linux源码目录的fs/proc/base.c文件，cmdline的 实现代码如下：
{%highlight c%}
static int proc_pid_cmdline(struct task_struct *task, char * buffer)
{
	int res = 0;
	unsigned int len;
	struct mm_struct *mm = get_task_mm(task);
	if (!mm)
		goto out;
	if (!mm->arg_end)
		goto out_mm;	/* Shh! No looking before we're done */

 	len = mm->arg_end - mm->arg_start;
 
	if (len > PAGE_SIZE)
		len = PAGE_SIZE;
 
	res = access_process_vm(task, mm->arg_start, buffer, len, 0);

	// If the nul at the end of args has been overwritten, then
	// assume application is using setproctitle(3).
	if (res > 0 && buffer[res-1] != '\0' && len < PAGE_SIZE) {
		len = strnlen(buffer, res);
		if (len < res) {
		    res = len;
		} else {
			len = mm->env_end - mm->env_start;
			if (len > PAGE_SIZE - res)
				len = PAGE_SIZE - res;
			res += access_process_vm(task, mm->env_start, buffer+res, len, 0);
			res = strnlen(buffer, res);
		}
	}
out_mm:
	mmput(mm);
out:
	return res;
}
{%endhighlight%}




