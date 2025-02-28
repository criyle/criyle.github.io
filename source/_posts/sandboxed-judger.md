---
title: Running a judger inside a container sandbox
date: 2020-01-19 19:19:08
tags:
---

It have been a long time after last post and the sandbox technology have been improved a lot. By combining unix socket and container pooling from [vijos/jd4](https://github.com/vijos/jd4) and cgroup checking and unshare container from [syzoj/judge-v3](https://github.com/syzoj/judge-v3), the judge is able to safely run arbitrary code in isolated environment.

<!--more-->

## Design of judge-v3 and jd4

In the design of `judge-v3`, The daemon receives task from website and then pass into runner to execute. The runner would start the running task through simple-sandbox. The simple-sandbox run a program requires creating / destroying containers repeatedly. The container is created through a dedicated readonly rootfs with bind mounting output directories. The container is created through unshare and chroot and privilege is dropped by changing process user.

In the design of `jd4`, containers are created in advance through fork and unshare. File system is shared through read-only or read-write output bind mounts to the host file system. Child process inside the container is connected through a unix socket and controlled through a RPC interface. New process pid is passed by creating new unix socket as file inside the bind mount and the process pass credential through oob data. The lifetime of the new process is managed through the parent process.

## Design of pre-forked GO-sandbox

By taking the design of both implementation, the `go-sandbox` chooses dumb, pre-forked and isolated runner. The file system of the sandbox is created through read-only bind mounting from host file system with 2 small tmpfs. Command is passed through a unix socket pair together with files and child pid. To ensure security, the privilege is dropped through changing process user or set capabilities.

The RPC interface between host and container daemon are:

+ `ping`: alive check
+ `conf`: set the running configuration (user / group)
+ `open`: open / create multiple files inside the container
+ `delete`: unlink / rmdir file inside the container
+ `reset`: remove all files under /w and /tmp
+ `execve`: execute and wait single process inside the container

## Potential attacks

+ forever sleep: pooling through CPU usage and assumes at least 40% utilization
+ creates arbitrary files: readonly & tmpfs limited at 8MB
+ large executable: tmpfs limited at 8MB
+ c / c++ includes `/etc/passwd`: not bind mounted
+ network download: unshared net
+ overwrite `/proc/1/exe`: unprivileged
+ open `/proc/1/fds/3`: unprivileged
+ fork bomb: cgroup pid max
+ large memory: cgroup memory

## Design of GO-judge

By taking the design from `judge-v3`, the `go-judge` have 2 layers. The client interface connect to the website to receive the tasks and test data. The first layer parses the task and data from the client interface and pass into the message queue interface for the runner. The runner receives run tasks from the queue and run through the `go-sandbox` and pass back the run results.

## Conclusion

Long time not writing documents... I am too lazy...