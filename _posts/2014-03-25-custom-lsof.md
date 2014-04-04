---
layout: post
title: 自己裁剪一个lsof
description: 自己裁剪一个lsof
category: 工具
tags: [工具]
refer_author: wenli
refer_blog_addr:
refer_post_addr:
---
{% include JB/setup %}

自己裁剪一个lsof
=========================================================================================================================================================================================

最近项目中某程序需要删除一些文件，删除之前要查看文件是否被使用，以确保数据安全。

查看文件是否被占用可以使用lsof、fuser等工具。

比如查找某个文件被哪些进程打开，可以使用

	bash-3.00$ /usr/sbin/lsof ~/CppStudy/common.h

预先运行了一个程序把文件打开，其源代码如下：

	#include <cstdio>
	#include <fcntl.h>
	#include <unistd.h>
	int main()
	{
	    int fd = open("/home/wenlli/CppStudy/common.h", O_RDWR | O_CREAT | O_APPEND);
	    sleep(86400);
	    close(fd);
	    return 0;
	}

所以lsof结果如下：

	COMMAND     PID   USER FD  TYPE DEVICE SIZE     NODE NAME
	openfilet 25935 wenlli  3u  REG   0,22  309 14728964 /home/wenlli/CppStudy/common.h

使用lsof的代码实现很简单：

	bool IsFileBeingUsed(const string& file)
	{
	    char buf[512] = {0};
	    char cmd[512] = {0};
	    sprintf(cmd, "/usr/sbin/lsof %s", file.c_str());
	    FILE* stream = popen(cmd, "r");
	    fread(buf, 1, 511, stream);
	    pclose(stream);
	    string s(buf);
	    if (s.find(file) != string::npos)
	    {
	        return true;
	    }
	    return false;
	}

但是客户的机器出于信息安全的考虑，严格控制软件的安装，很可能就没有安装lsof。

于是决定查看lsof的源码，仿照源码实现某部分功能。

下载lsof-4.76的源码，主要分析src/main.c、src/dialects/linux/dproc.c、src/proc.c等文件。

从src/dialects/linux/dproc.c这样的目录结构来看，lsof支持很多种Unix平台，实际上它支持的平台包括：

	bash-3.00$ ll
	total 48
	drwxr-xr-x 3 wenlli employee 4096 Aug 31 2005 aix
	drwxr-xr-x 2 wenlli employee 4096 Aug 31 2005 bsdi
	drwxr-xr-x 2 wenlli employee 4096 Aug 31 2005 darwin
	drwxr-xr-x 2 wenlli employee 4096 Aug 31 2005 du
	drwxr-xr-x 3 wenlli employee 4096 Aug 31 2005 freebsd
	drwxr-xr-x 4 wenlli employee 4096 Mar  2 2000 hpux
	drwxr-xr-x 2 wenlli employee 4096 Aug 31 2005 linux
	drwxr-xr-x 2 wenlli employee 4096 Aug 31 2005 n+obsd
	drwxr-xr-x 2 wenlli employee 4096 Aug 31 2005 n+os
	drwxr-xr-x 3 wenlli employee 4096 Aug 31 2005 osr
	drwxr-xr-x 2 wenlli employee 4096 Aug 31 2005 sun
	drwxr-xr-x 3 wenlli employee 4096 Aug 31 2005 uw

它功能强大，还有大量的命令行选项，具体内容可以

	bash-3.00$ man lsof

由于只需要查看文件是否被占用的功能，分析源码必须略去各种复杂的条件编译，例如：

	#if defined(HASZONES) 
	    znhash_t *zp;
	#endif /* defined(HASZONES) */

这些宏可以在安装软件运行Configure的时候酌情选取，通过添加或者删除宏来开启或者关闭相应的功能。

进入main.c，跟大多数软件一样，最开始的工作都是：

​1. 解释命令行参数，放进特定的数据结构里面。

​2. 判断参数合法性。

​3. 处理参数冲突。

​4. 特定的初始化工作

	/*
	 * Create option mask.
	 */ 
	    (void) snpf(options, sizeof(options),
	        "?a%sbc\:D:d:%sf:F:g:hi:%slL:%sMnNo:Op:Pr:%ssS:tT:u:UvVwx:%s%s", ...);
	
	/*
	 * Loop through options.
	 */ 
	    while ((c = GetOpt(argc, argv, options, &rv)) \!= EOF) {
	        ...
	    }
	
	/*
	 * Process the file arguments.
	 */
	    if (GOx1 < argc) {
	        if (ck_file_arg(GOx1, argc, argv, Ffilesys, 0, (struct stat *)NULL))
	            usage(1, 0, 0);
	    }
	
	/*
	 * Do dialect-specific initialization.
	 */
	    initialize();

紧接着进入具体工作的代码：

	do {
	    gather_proc_info();
	    if (Nlproc > 1) {
	        ...
	        for (i = 0; i < Nlproc; i++) {
	            slp[i] = &Lproc[i];
	        }
	        (void) qsort((QSORT_P *)slp, (size_t)Nlproc, (size_t)sizeof(struct lproc *), comppid);
	    }
	    if ((n = Nlproc)) {
	        for (lf = Lf, print_init(); PrPass < 2; PrPass++) {
	            for (i = n = 0; i < Nlproc; i++) {
	                Lp = (Nlproc > 1) ? slp[i] : &Lproc[i];
	                if (Lp->pss) {
	                    if (print_proc()) 
	                        n++;
	                }
	                if (RptTm && PrPass)(
	                    void) free_lproc(Lp);
	            }
	        }
	        Lf = lf;
	    }
	    if (RptTm)
	    {
	        ...
	        (void) sleep(RptTm);
	        ...
	    }
	} while (RptTm);

RptTm是用户设定的定时时间，单位是秒，如果用户

	bash-3.00$ /usr/sbin/lsof -r 10 ~/CppStudy/common.h

表示lsof每10秒显示一次结果。

gather\_proc\_info()用于获取系统中各个进程的信息，所有数据放在Lproc指向的数据结构里面。

运行

	for (i = 0; i < Nlproc; i++) {
	    slp[i] = &Lproc[i];
	}

之后，通过slp也能索引到数据。

qsort函数用于对数据按照pid进行排序。

如果我们单独运行

	bash-3.00$ /usr/sbin/lsof

init进程（pid为1）的数据将最先显示。

print\_proc()用于打印进程的信息。

所以为了实现项目的需求，核心代码就是gather\_proc\_info()和print\_proc()。

	void gather_proc_info()
	{
	    ...
	    ps = opendir(PROCFS);
	    while ((dp = readdir(ps))) {
	        for (f = n = pid = 0, cp = dp->d_name; *cp; cp++) {
	            if (!isdigit((unsigned char)*cp)) {
	                f = 1;
	                break;
	            }
	            pid = pid * 10 + (*cp - '0');
	            n++;
	        }
	        if (f) {
	            continue;
	        }
	        ...
	    }
	    ...
	}

从isdigit((unsigned
char)\*cp)可以看出，程序需要的是/proc/下面以数字命名的目录，而这些目录里面都是相应进程的所有资料。比如：

	bash-3.00$ ps aux | grep openfiletest
	wenlli    9597  0.0  0.0  6992  672 pts/66   S    10:04   0:00 ./openfiletest
	wenlli   15164  0.0  0.0 51064  632 pts/8    S+   12:04   0:00 grep openfiletest
	bash-3.00$ ll /proc/9597
	total 0
	dr-xr-xr-x  2 wenlli employee 0 Mar 14 12:03 attr
	-r--------  1 wenlli employee 0 Mar 14 12:03 auxv
	-r--r--r--  1 wenlli employee 0 Mar 14 12:03 cmdline
	lrwxrwxrwx  1 wenlli employee 0 Mar 14 12:03 cwd -> /home/wenlli/CppStudy
	-r--------  1 wenlli employee 0 Mar 14 12:03 environ
	lrwxrwxrwx  1 wenlli employee 0 Mar 14 12:03 exe -> /home/wenlli/CppStudy/openfiletest
	dr-x------  2 wenlli employee 0 Mar 14 12:03 fd
	-rw-r--r--  1 wenlli employee 0 Mar 14 12:03 loginuid
	-r--------  1 wenlli employee 0 Mar 14 12:03 maps
	-rw-------  1 wenlli employee 0 Mar 14 12:03 mem
	-r--r--r--  1 wenlli employee 0 Mar 14 12:03 mounts
	lrwxrwxrwx  1 wenlli employee 0 Mar 14 12:03 root -> /
	-r--r--r--  1 wenlli employee 0 Mar 14 12:03 stat
	-r--r--r--  1 wenlli employee 0 Mar 14 12:03 statm
	-r--r--r--  1 wenlli employee 0 Mar 14 12:03 status
	dr-xr-xr-x  3 wenlli employee 0 Mar 14 12:03 task
	-r--r--r--  1 wenlli employee 0 Mar 14 12:03 wchan

我们可以看到openfiletest进程的资料。

接着对于每个/proc/\$pid/，程序处理如下：

	void gather_proc_info()
	{
	    ...
	    // Process /proc/$pid/stat
	    ...
	    // Process /proc/$pid/cwd
	    ...
	    // Process /proc/$pid/root
	    ...
	    // Process /proc/$pid/exe
	    ...
	    // Process /proc/$pid/maps
	    ...
	    // Process /proc/$pid/fd/
	    ...
	}

在“Process
/proc/\$pid/fd/”的过程中，我们获得了这个进程所打开的文件的信息。

	bash-3.00$ ll /proc/9597/fd
	total 4
	lrwx------  1 wenlli employee 64 Mar 14 15:01 0 -> /dev/pts/66
	lrwx------  1 wenlli employee 64 Mar 14 15:01 1 -> /dev/pts/66
	lrwx------  1 wenlli employee 64 Mar 14 15:01 2 -> /dev/pts/66
	lrwx------  1 wenlli employee 64 Mar 14 15:01 3 -> /home/wenlli/CppStudy/common.h

从这里我们可以很清晰看到，openfiletest打开了/home/wenlli/Cppstudy/common.h。

gather\_proc\_info()把所有进程的信息都记录在struct lproc
\*Lproc指向的结构体的数组中。每次调用proc.c文件中的alloc\_proc函数，每一个进程的信息都写进其中一个结构体里面

	        static int sz = 0;
	
	    if (!Lproc) {
	        if (!(Lproc = (struct lproc *)malloc(
	              (MALLOC_S)(LPROCINCR * sizeof(struct lproc)))))
	        {
	        (void) fprintf(stderr,
	            "%s: no malloc space for %d local proc structures\n",
	            Pn, LPROCINCR);
	        Exit(1);
	        }
	        sz = LPROCINCR;
	    } else if ((Nlproc + 1) > sz) {
	        sz += LPROCINCR;
	        if (!(Lproc = (struct lproc *)realloc((MALLOC_P *)Lproc,
	              (MALLOC_S)(sz * sizeof(struct lproc)))))
	        {
	        (void) fprintf(stderr,
	            "%s: no realloc space for %d local proc structures\n",
	            Pn, sz);
	        Exit(1);
	        }
	    }
	    Lp = &Lproc[Nlproc++];
	    Lp->pid = pid;
	    Lp->pgid = pgid;
	    Lp->ppid = ppid;
	    Lp->file = (struct lfile *)NULL;
	    Lp->sf = (short)sf;
	    Lp->pss = (short)pss;
	    Lp->uid = (uid_t)uid;
	/*
	 * Allocate space for the full command name and copy it there.
	 */
	    if (!(Lp->cmd = mkstrcpy(cmd, (MALLOC_S *)NULL))) {
	        (void) fprintf(stderr, "%s: PID %d, no space for command name: ",
	        Pn, pid);
	        safestrprt(cmd, stderr, 1);
	        Exit(1);
	    }

为了减少内存分配，这里先预先分配LPROCINCR（LPROCINCR==128）个结构体的大小，如果空间不够，再重分配内存，每次递增LPROCINCR个结构体的大小。

lproc结构体的定义如下：

	struct lproc {
	    char *cmd;         /* command name */
	    short sf;          /* select flags -- SEL* symbols */
	    short pss;         /* state: 0 = not selected
	                     *    1 = wholly selected
	                     *    2 = partially selected */
	    int pid;           /* process ID */
	    int pgid;          /* process group ID */
	    int ppid;          /* parent process ID */
	    uid_t uid;          /* user ID */
	
	# if  defined(HASZONES)
	    char *zn;          /* zone name */
	# endif /* defined(HASZONES) */
	
	    struct lfile *file;     /* open files of process */
	};

对于openfiletest，取值举例如下：

	(gdb) p *Lp
	$59 = {cmd = 0x5741e0 "openfiletest", sf = 0, pss = 2, pid = 4190, pgid = 4190, ppid = 3954, uid = 1680, file = 0x574240}

lproc的数据成员file是lfile类型，记录了进程详细的打开文件的信息，取值举例如下：

	(gdb) p \*Lf
	$55 = {access = 117 'u', lock = 32 ' ', dev_def = 1 '\001', inp_ty = 1 '\001', is_com = 0 '\0', is_nfs = 0 '\0',
	is_stream = 0 '\0', lmi_srch = 0 '\0', nlink_def = 0 '\0', off_def = 0 '\0', rdev_def = 0 '\0', sz_def = 1 '\001',
	fd = "   3\000\000\000", iproto = "\000\000\000\000\000\000\000", type = "REG\000\000\000\000", sf = 64, ch = \-1, ntype = 32,
	off = 0, sz = 309, dev = 22, rdev = 34841, inode = 14728964, nlink = 0, dev_ch = 0x0, fsdir = 0x0, fsdev = 0x0, li = \{\{af = 0,
	p = 0, ia = {a4 = \{s_addr = 0\}, a6 = {in6_u = {u6_addr8 = "\000\000\000\000\000\000\000\000 CW\000\000\000\000",
	u6_addr16 = \{0, 0, 0, 0, 17184, 87, 0, 0\}, u6_addr32 = \{0, 0, 5718816, 0\}}}}}, {af = 0, p = \-1, ia = {a4 = \{
	s_addr = 3474118656\}, a6 = {in6_u = {u6_addr8 = "\000Ø\022Ï4\000\000\0000CW\000\000\000\000", u6_addr16 = \{55296,
	53010, 52, 0, 17200, 87, 0, 0\}, u6_addr32 = {3474118656, 52, 5718832, 0}}}}}}, lts = \{type = \-1, state = \{i = 0,
	ui = 0\}, rq = 0, sq = 0, rqs = 0 '\0', sqs = 0 '\0'\}, nm = 0x573de0 "/home/wenlli/CppStudy/common.h", nma = 0x0,
	next = 0x0}

结合结构体的数据和print\_proc、print\_file函数代码，我们可以找到它们的对应关系：

	COMMAND     PID   USER FD  TYPE DEVICE SIZE     NODE NAME
	openfilet 25935 wenlli  3u  REG   0,22  309 14728964 /home/wenlli/CppStudy/common.h

<table>
<thead>
<tr>
<th>列数据</th> 
<th>结构体数据</th>
</tr>
</thead>
 <tbody>
<tr>
<td>COMMAND</td> <td>Lp-\>cmd</td>
</tr>
<tr>
<td>  PID </td>   <td>   Lp-\>pid</td>
</tr>
<tr>
 <td> USER </td>  <td>   Lp-\>uid</td>
</tr>
<tr>
<td>  FD </td>   <td>Lf-\>fd<br> Lf-\>access</td>
				 
</tr>  
<tr>            
 <td> TYPE</td>  <td>    Lf-\>type</td>
</tr>
<tr>
 <td> DEVICE</td> <td>   Lf-\>dev</td>
</tr>
<tr>
 <td> SIZE </td>  <td>   Lf-\>sz</td>
</tr>
<tr>
 <td> NODE </td> <td>    Lf-\>inode</td>
</tr>
<tr>
<td>  NAME</td>  <td>    Lf-\>nm</td>
</tr>
</tbody>
</table>

到这里为止，我们已经找出lsof如何实现项目里面需要的功能，我们可以改写

	bool IsFileBeingUsed(const string& file)
	{
	    ...
	}

其实这里还有一种偷懒的方法，通过读取

	bash-3.00$ ls -l /proc/*/fd 2>/dev/null | grep /home/wenlli/CppStudy/common.h
	lrwx------  1 wenlli employee 64 Mar 17 15:00 3 -> /home/wenlli/CppStudy/common.h

的输出就能简单实现功能。

如果需要实现lsof的其他功能，可以继续参考源码的实现。

更多procfs的信息请参考 [http://en.wikipedia.org/wiki/Procfs](http://en.wikipedia.org/wiki/Procfs)。