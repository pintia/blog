---
title: judger-performance-tuning
date: 2018-11-15 18:21:01
tags: 技术
---

## 0x00 起因

首先，这是一篇迟到了非常久的 blog，这个性能优化早在今年 6 月份就已经上线，这篇文章将细致的梳理一下这次调优的过程。

调优的对象就是 PTA 使用的评测机，该代码已经开源，原本是 quark 学长维护的版本，而现在在我们团队的 git 仓库当中，分别是 [ljudge](https://github.com/pintia/ljudge) 和 [lrun](https://github.com/pintia/lrun) 。

首先简单讲一下评测机的部署方案：

* 每个评测机处于一个 docker 容器当中
* 每台云主机可以 host 多个 docker 容器
* 每个评测机都是单线程工作的，占用一个核心
* 每台云主机根据自身核心数来决定最大承担 judger 容器的个数

发现 judger 的性能问题，是在一次承担考试的，我因为观察到评测队列堆积，所以进行了一次扩容，将云主机上的容器个数调整到了最大值。但不幸的是，从监控来看，评测队列的消化速度几乎没有增加。之后也遇到过几次同样的情况，所以最终决定进行一次性能问题的排查。

## 0x01 排查

准备一些脚本：

entrypoint.sh

```bash
#!/bin/bash

cd /app/ljudge/examples/a-plus-b
for ((i=1;i<=100;i++));
do
	ljudge -u a.c -i 1.in -o 1.out -i 2.in -o 2.out 2>/dev/null >/dev/null
done
```

在容器当中 judge 一个最简单的 a+b 的程序，循环 100 次，放进容器当中备用

```bash
#!/bin/bash

CMD='docker run --rm --privileged --entrypoint=/app/ljudge/examples/a-plus-b/entrypoint.sh --cpus=1 pintia/ljudge-docker:benchmark'

echo '1 container'
time bash -c "${CMD}"

echo '2 container'
time bash -c "${CMD} & ${CMD}"

echo '3 container'
time bash -c "${CMD} & ${CMD} & ${CMD}"

echo '4 container'
time bash -c "${CMD} & ${CMD} & ${CMD} & ${CMD}"

echo '5 container'
time bash -c "${CMD} & ${CMD} & ${CMD} & ${CMD} & ${CMD}"

echo '6 container'
time bash -c "${CMD} & ${CMD} & ${CMD} & ${CMD} & ${CMD} & ${CMD}"
```

分别启动 1 ~ 6 个容器，这里的 `--entrypoint` 就是指定刚才放进容器当中的脚本，测试所需要的时间（脚本写的有点 low 见谅）

在一台 8 核心的机器上，得到了测试结果

```bash

1 container
real	0m39.932s
user	0m0.020s
sys	0m0.012s

2 container
real	0m56.090s
user	0m0.032s
sys	0m0.008s

3 container
real	1m11.888s
user	0m0.016s
sys	0m0.004s

4 container
real	1m30.698s
user	0m0.060s
sys	0m0.020s

5 container
real	1m47.885s
user	0m0.028s
sys	0m0.012s

6 container
real	2m2.968s
user	0m0.048s
sys	0m0.024s
```

{% asset_img time_by_container_count.png time_by_container_count %}

很明显可以看出，在一台核心数足够的机器上启用多个 judger，消耗的时间竟然是线性增加的，也就是说，在单机上通过增加 judger 实例数进行的扩容效果几乎没有，多个 judger 之间势必有资源发生了抢占。

## 0x02 继续排查

那确认了这种部署模式的问题之后，接下去就需要找是什么东西在多个 judger 之间发生了竞争。排查持续了一天，期间使用了各种工具比如 perf，strace 等等，但都没有确定问题所在。但一个偶然的尝试发现了一些问题。

{% asset_img copy_net_ns_in_top.png copy_net_ns_in_top %}

通过观察，可以发现 lrun 进程频繁的 sleep 在 copy_net_ns 这个地方。确认了一下代码，每一次评测都会创建一个新的 network namespace。所以根据表现，大胆猜测 network namespace creation 应该是一个比较耗时而且无法并发的调用。但 judger 确实需要进行网络隔离，所以在这里我尝试绕过这个行为。

因为 judger 同一个时间只会评测一个程序，也就是说同一个时间至多只会存在一个沙盒，所以我可以事先创建好一个已经被隔离开的 network namespace，而让每次评测是将这个 network namespace 设置到当前沙盒上，且不需要考虑多个沙盒的情况。

带着这个思路简单增加了一些代码：

```c++
    // switch to existed netns for better performance if needed
    if ((clone_flags & CLONE_NEWNET) == CLONE_NEWNET && arg.reuse_netns) {
        INFO("try to reuse network namespace");
        int netns_fd = open(netns::LRUN_NETNS_PATH, O_RDONLY);
        if (netns_fd < 0) {
            // create reusable net ns
            INFO("create reusable network namespace")
            int result = system(netns::LRUN_NETNS_CMD);
            if (result != 0) {
                ERROR("can not create network namespace");
                return -3;
            } else {
                netns_fd = open(netns::LRUN_NETNS_PATH, O_RDONLY);
                if (netns_fd < 0) {
                    ERROR("can not open reusable network namespace");
                    return -3;
                }
            }
        }

        INFO("set network ns to %s", netns::LRUN_NETNS_PATH);
        // older glibc does not have setns
        if (syscall(SYS_setns, netns_fd, CLONE_NEWNET)) {
            ERROR("can not set network namespace");
            return -3;
        };
        close(netns_fd);

        clone_flags ^= CLONE_NEWNET;
    }
```

如果没有 reusable network namespace，则创建一个新的，如果有并且可用，就将当前沙盒的 network namespace 设置成它。

## 0x03 验证

准备脚本：

entrypoint-netns.sh

```bash
#!/bin/bash

cd /app/ljudge/examples/a-plus-b
for ((i=1;i<=100;i++));
do
	ljudge --reuse-netns -u a.c -i 1.in -o 1.out -i 2.in -o 2.out 2>/dev/null >/dev/null
done
```

在之前的测试脚本当中加入测试优化过的 judger 的代码

```bash
#!/bin/bash

CMD='docker run --rm --privileged --entrypoint=/app/ljudge/examples/a-plus-b/entrypoint.sh --cpus=1 pintia/ljudge-docker:benchmark'
echo 'No reuse netns, 1 container'
time bash -c "${CMD}"

echo 'No reuse netns, 2 container'
time bash -c "${CMD} & ${CMD}"

echo 'No reuse netns, 3 container'
time bash -c "${CMD} & ${CMD} & ${CMD}"

echo 'No reuse netns, 4 container'
time bash -c "${CMD} & ${CMD} & ${CMD} & ${CMD}"

echo 'No reuse netns, 5 container'
time bash -c "${CMD} & ${CMD} & ${CMD} & ${CMD} & ${CMD}"

echo 'No reuse netns, 6 container'
time bash -c "${CMD} & ${CMD} & ${CMD} & ${CMD} & ${CMD} & ${CMD}"

CMD_NETNS='docker run --rm --privileged --entrypoint=/app/ljudge/examples/a-plus-b/entrypoint-netns.sh --cpus=1 pintia/ljudge-docker:benchmark'

echo 'Reuse netns, 1 container'
time bash -c "${CMD_NETNS}"

echo 'Reuse netns, 2 container'
time bash -c "${CMD_NETNS} & ${CMD_NETNS}"

echo 'Reuse netns, 3 container'
time bash -c "${CMD_NETNS} & ${CMD_NETNS} & ${CMD_NETNS}"

echo 'Reuse netns, 4 container'
time bash -c "${CMD_NETNS} & ${CMD_NETNS} & ${CMD_NETNS} & ${CMD_NETNS}"

echo 'Reuse netns, 5 container'
time bash -c "${CMD_NETNS} & ${CMD_NETNS} & ${CMD_NETNS} & ${CMD_NETNS} & ${CMD_NETNS}"

echo 'Reuse netns, 6 container'
time bash -c "${CMD_NETNS} & ${CMD_NETNS} & ${CMD_NETNS} & ${CMD_NETNS} & ${CMD_NETNS} & ${CMD_NETNS}"
```

对照测试！得到结果

```
root@judger-test:~# bash benchmark.sh 
No reuse netns, 1 container
real	0m40.134s
user	0m0.044s
sys	0m0.020s

No reuse netns, 2 container
real	0m56.876s
user	0m0.068s
sys	0m0.044s

No reuse netns, 3 container
real	1m13.356s
user	0m0.032s
sys	0m0.020s

No reuse netns, 4 container
real	1m28.670s
user	0m0.072s
sys	0m0.040s

No reuse netns, 5 container
real	1m44.280s
user	0m0.104s
sys	0m0.068s

No reuse netns, 6 container
real	2m2.403s
user	0m0.104s
sys	0m0.072s

Reuse netns, 1 container
real	0m20.290s
user	0m0.040s
sys	0m0.016s

Reuse netns, 2 container
real	0m20.003s
user	0m0.076s
sys	0m0.036s

Reuse netns, 3 container
real	0m20.299s
user	0m0.100s
sys	0m0.060s

Reuse netns, 4 container
real	0m20.947s
user	0m0.088s
sys	0m0.068s

Reuse netns, 5 container
real	0m21.493s
user	0m0.032s
sys	0m0.020s

Reuse netns, 6 container
real	0m23.164s
user	0m0.108s
sys	0m0.056s
```

{% asset_img compare.png compare %}

可以看到优化之后，在同一台机器上开核心数以内的 judger 实例，总时长并不会有明显增加，也就是说扩容的目标达到了。

至此这次 judger 的性能优化就算是完成了~