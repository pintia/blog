---
title: glibc 更新导致的断错误排查
date: 2018-06-27 09:37:17
tags:
---

## 0x00 起因
姥姥在检查 PAT 练习题的过程当中，发现一道题曾经能过的标准程序现在在两个测试点上会产生段错误。题目链接：[英文版](https://pintia.cn/problem-sets/994805148990160896/problems/994805150751768576)

## 0x01 排查

拿到姥姥的标准程序，是个 C 的程序，在网站上看了看同学们的提交，几乎全是 C++，并且也没有遇到同样的问题，所以总体上讲这个情况影响可能不是特别大。

观察出现问题的测试点，发现是两个数据量最大的 case。那看样子这个问题和 case 大小相关。

尝试手跑命令重现，发现一个奇怪的现象，如果不在沙盒里运行，程序毫无问题，运行结果也没有错。

{% asset_img no_sandbox.png no_sandbox %}

但是放进沙盒运行就直接抛段错误。

{% asset_img sandbox.png sandbox %}

而后我们尝试在沙盒当中获得 segmentation fault 可能产生的 core dump 无果。再尝试在沙盒当中运行 gdb 进行调试，但由于 gdb 本身会使用被沙盒禁止的系统调用，所以不能进行断点调试之类的，所以这次索性将所有系统调用开放，允许程序使用。这时候神奇的事请发生了，原本段错误的程序竟然能够正常运行了。

去除系统调用限制

{% asset_img no_restrict_syscall.png no_restrict_syscall %}

那问题就简单了，经过几次二分查找之后，我们确定了造成段错误的系统调用限制 `sysinfo`

## 0x02 验证

我们可以验证一下不同数据集，同样的程序调用系统调用有什么区别

小数据集

{% asset_img small_case.png small_case %}

大数据集

{% asset_img big_case.png big_case %}

没错，在大数据集情况下，这个程序会额外调用一个 sysinfo 系统调用，而它是被沙盒禁用的，所以产生了段错误。

## 0x03 继续排查

经过几次“print 大法好”调试之后，我们发现 sysinfo 是在 `qsort(E, M, sizeof(struct edge), ComparW);` 这行代码当中发生的。那 qsort 为什么需要调用 sysinfo 呢？通过搜索我们找到了一个 patch [getsysstats: use sysinfo() instead of parsing /proc/meminfo](https://patchwork.ozlabs.org/patch/507018/)

我们节选一些：

> Profiling git's test suite, Linus noted [1] that a disproportionatelylarge amount of time was spent reading /proc/meminfo. This is done bythe glibc functions get_phys_pages and get_avphys_pages, but they onlyneed the MemTotal and MemFree fields, respectively. That sameinformation can be obtained with a single syscall, sysinfo, instead ofsix: open, fstat, mmap, read, close, munmap. While sysinfo alsoprovides more than necessary, it does a lot less work than what thekernel needs to do to provide the entire /proc/meminfo. Both strace -Tand in-app microbenchmarks shows that the sysinfo() approach isroughly an order of magnitude faster.

意思是说 Linus 发现，glibc 在获取内存相关信息的时候，是直接文件读 `/proc/meminfo`的，而这些信息在 `sysinfo` 这个系统调用当中就有，而且性能要好得多，因为文件操作确实很慢。

> So it seems that any application that uses qsort on a moderately sizedarray will incur this cost (once), which is obviously proportionatelymore expensive for lots of short-lived processes (such as the git testsuite).

他也提到了，当 qsort 执行在一个较大规模的数组上时，这个情况就会发生一次，这对于 git 测试集来说是非常昂贵的性能开销。

我们可以尝试抓一下这个 sysinfo 在哪里，通过 gdb `catch syscall` 这个方便的功能，我们抓到了调用栈

{% asset_img gdb.png gdb %}

我们可以看一下源码，[msort.c#164](https://sourceware.org/git/?p=glibc.git;a=blob;f=stdlib/msort.c;h=266c2538c07e86d058359d47388fe21cbfdb525a;hb=HEAD#l164)

所以当 `size < 1024`时，qsort 直接使用 stack。不然就会获取系统内存信息，做一些内存空间的判断，然后 `malloc`

## 0x04 后记

引入这个问题，是因为大概在今年 4 月份，我们将 judger 的运行基础镜像从 ubuntu 14.04 升级到了 ubuntu 16.04。而新的 judger 代码已经上线，所以现在的 judger 已经不做 sysinfo 这个系统调用的过滤了。
