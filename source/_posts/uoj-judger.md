---
title: A GO reimplement of UOJ judger
date: 2019-04-13 20:51:44
tags:
---

Reimplement of UOJ run program in GO: [go-judger](https://github.com/criyle/go-judger). Start after I found libseccomp that uses seccomp filter introduced in linux 3.8 (2013). Since I have participated that project ([uoj](https://github.com/vfleaking/uoj)) only a little, I decided to try to do some contributions.

## Original implements

The original run program restricted resources (CPU, memory, output) and file access by `ptrace`. Including following steps:

Setup up step after fork in child:

1. Set resource limits by `setrlimit`
2. Set environment variables
3. Set input / output files
4. `execv`

Tracing after fork in parent:

Setup `ptrace` options when trapped at `execv`

1. `wait4` at syscall entrance
2. Check resource usage, wait status, signals and syscall black list to determine terminate or soft ban
3. `ptrace syscall` enter syscall
4. `wait4` at syscall exits
5. Set syscall return value
6. `ptrace syscall` exit syscall

In this senario, the traced process required to stop for each syscall and for both entrance and exiting. For harmless syscalls (e.g. `brk`, `read`), this introduces some resource overhead.

## Reimplements

For the newly implemented `seccomp` BPF filter provided by `libseccomp`, this kind of syscall will handled by the kernel to avoid too much context switch. Also, for a single traced syscall, `seccomp` will only be triggled once.

Thus, the new implement becomes.

Setup step after fork in child:

1. Set resource limits by `setrlimit`
2. Set input / output files
3. Load `seccomp` filter
4. Stop itself by `SIGSTOP`
5. `execve` with environment variables

Tracing after fork in parent:

Setup `ptrace` options when trapped by `SIGSTOP`

1. `wait4` at seccomp event
2. Check resource usage, wait status, signals, and call syscall event handles. Handle determins whether to terminates or soft ban
3. `ptrace continue` enter syscall

Notice that `SIGSTOP` before `execve` is required since if `execve` is traced but the `ptrace` option have not set up yet, `ENOSYS` will returned to `execve`. Safe syscalls was allowed by the filter so there is no `ptrace` event triggered by safe syscalls.

Also, by setting syscall number to `-1` and return value to the register, the soft ban mechanisn becomes much efficient.

With all that implemented, `process_vm_readv` is used to speed up copy syscall argument instead of `ptrace peekdata`.

## Conclution

In conclution, by restrict CPU time, memory, output, syscalls and file access, run program is able to block potential attacks.

Since GO language does not provides official implements for `fork` for runtime duplication issue, it took some time to figure out the usage of raw syscall interface. Because after fork in child, I cannot call any go function, I did buffed the seccomp filter to allow load it after fork. Also, `process_vm_readv` is not provided so I wrote a wrapper for it.
