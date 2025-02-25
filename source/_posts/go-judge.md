---
title: 基于容器化技术的一个沙箱
date: 2020-04-29 17:39:11
tags:
---

## 起源

本项目原本的目的是用 GO 重构之前参与过的一个 OJ 的评测系统，目前完成了沙箱的部分分享一下。

## 需求

评测系统通常需要对提交的代码进行编译和运行。通常运行的算法代码并不需要特殊的权限和系统访问。沙箱需要限制住恶意代码对于评测系统运行的可能的破坏行为。

一个沙箱的实现包含了：

+ 安全： 沙箱内的程序不允许进行超出计算需求的系统访问。包括网络访问，未授权的文件系统访问。
+ 限制： 沙箱内的程序仅能使用限定的 CPU 时间和 内存
+ 快速： 运行时的额外开销小

## 实现选择

### 基于 `seccomp` `ptrace` + `setrlimit` 的沙箱

利用 linux 提供的 BPF seccomp，允许安全的 `syscall` （如 `write` 运行）。对于文件系统访问的 `syscall` (如 `open`)，利用 linux 提供的 `ptrace` 和 `process_vm_readv` 读取沙箱中程序系统调用参数的值获取文件访问的路径。得到路径之后用白名单的形式判断是否为恶意访问。CPU 时间和内存的限制由 `setrlimit` 设置。

优点：实现简单，不需要权限

缺点：对于每一个文件访问系统调用都需要上下文切换，大概 `20%` 额外开销

### 基于 `unshare` `clone` 和 `cgroup` 的沙箱

得益于容器技术的进步，linux 的 `clone` 和 `unshare` 系统调用可以在与宿主机隔离的环境中运行程序的能力。利用 `clone` 系统调用创建新的 `mount`, `IPC`, `net`, `pid`, `user`, `uts` `namespace`。在运行程序之前通过 `bind mount` 和 `pivot_root` 来隔离运行环境的文件系统。用 `cgroup` 的 `cpuacct.usage` 轮询 和 `memory.limit_in_bytes` 实现CPU 时间和内存的限制。

优点：没有上下文切换的开销。`cgroup` 统计数据更准确

缺点：`cgroup` 需要 `root` 权限。`unshare clone` 有 `20 ms` 的额外开销

### 改进后的容器池

为了减少创建容器的开销，用类似 `linux daemon` 的思想创建在容器中运行的 “客户端” 由在宿主机上运行的 “控制端” 控制。这样由 “客户端” 运行的程序就不需要重新创建容器的文件系统。跨进程通信使用了 `socketpair` 创建的 `unix socket` 并由 `gob` 编码。同时为了减少文件由 `socket` 内容传递所产生的多次复制， 利用 `unix socket oob` 可以传送文件描述符的特性直接传送文件 `fd`。

Exec Command

![UML](/images/sandbox.png)

这样用类似 RPC 的方式实现了程序生命周期的控制。同时提供了 `open`, `delete`, `reset` command 进行文件操作。

优点：减少了创建容器的额外开销

缺点：需要 `root` 权限或者 `privilleged docker`

[Sandbox的实现](https://github.com/criyle/go-sandbox)

## Rest API 接口

基于容器化技术的沙箱需要用 `root` 权限或者 `privilleged docker` 来运行，但是评测系统的逻辑并不需要 `root` 权限。基于权限最小化原则，把沙箱单独拆分出来以 REST API 的形式提供服务。

Web 框架使用了 [GIN](https://github.com/gin-gonic/gin)。提供了文件的 CRUD `/file` 和运行单个或者多个程序（用管道链接标准输入输出）的 `/run`。

[ExecutorServer的实现](https://github.com/criyle/go-judge)

`go get github.com/criyle/go-judge/cmd/executorserver && sudo ~/go/bin/executorserver`

### 跨平台

设计好 REST API 接口后发现似乎并没有太大的平台相关性。借助于 duck interface 的特性，在 windows 上使用 `low mandatory level token` + `JobObject` 简单的实现了一个沙箱作为跨平台的一个验证。

### 跑分

用 `go test -bench` 测试大概有 `+1ms (2.06ms - 0.99ms)` 额外开销。用 postman 测试 REST API 大概 `+5ms` 额外延迟。

## demo

用 Vue.js 和 WebSocket 简单糊了个小测试站。前端放在 heroku 上，后端部署在性能很菜的树莓派上。[goj.ac](https://goj.ac)。

## 最后

原本的目的是为了写点代码来学习 GO 语言。然后在学习过程中重构了几次沙箱后学到了不少工程方面的设计知识，写一个库和用一个库的心态也有很大区别。

沙箱的部分实现也可以拿出来单独使用。

+ [forkExec](https://github.com/criyle/go-sandbox/tree/master/pkg/forkexec) linux fork exec 核心库，用来创建容器和加载 Seccomp BPF
+ [unixSocket](https://github.com/criyle/go-sandbox/tree/master/pkg/unixsocket) linux unix socket 传递接收文件描述符
+ [memFd](https://github.com/criyle/go-sandbox/tree/master/pkg/memfd) linux memfd 创建内存文件
+ [container](https://github.com/criyle/go-sandbox/tree/master/container) linux 容器池的实现
+ [envExec](https://github.com/criyle/go-judge/tree/master/pkg/envexec) 在环境中运行单个或多个程序的定义和实现
