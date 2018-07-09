---
layout: post
title:  "What happens when I mount procfs from host to a Docker container, and why?"
date:   2018-07-09 13:34:11 +0800
categories: jekyll update
---

Docker has been a major component of my daily work since several years ago. It
brings advantages on isolation, replication and scalability to our services.
In some places, people are also using Docker to create reproducible &
redistributable software development experiences, which makes developers refrain
from managing dependencies and tests runtime environment. In my case,
I also use Docker for system monitoring. Here's where the story begins.

Monitoring the performance and stats of a running Linux, like CPU/Memory usage,
network traffic stats, running processes and disk I/O information, is a
historical topic. And there has been many tools created for this kind of tasks,
like (net/io/vm)stat utilities, and modern solutions like
[prometheus](https://prometheus.io/) and [netdata](https://github.com/firehol/netdata).
But no matter what the choice is, all these tools relies on one thing provided
by Linux kernel, [procfs](http://man7.org/linux/man-pages/man5/proc.5.html)

I'm not going to talk too much about procfs, here's a quote from man7.org
``` text
       The proc filesystem is a pseudo-filesystem which provides an
       interface to kernel data structures.  It is commonly mounted at
       /proc.
```
So basically, to combine the advantages of Docker with these system monitoring
tools, we can simply start a container with argument: `-v /proc:/host/proc` and
make the tool fetch system metrics from `/host/proc` inside the container.
Sounds great right? A redistributable system monitoring solution with security
protections provided by docker. Unfortunately, the answer is no.

Suppose I'm running a program inside a container

``` shell
docker run -ti --rm --name program alpine tail -f /etc/passwd
```
and a similar program directly in the host

``` shell
tail -f /etc/passwd
```

and I'm running a monitoring tool too.

``` shell
docker run -ti -v /proc:/host/proc --rm --name monitor alpine sh
```

Here's what I can get from `docker ps` and `docker top` and `ps`:

``` shell
[root@nyangawa-dev ~]# docker ps
CONTAINER ID        IMAGE               COMMAND                 CREATED             STATUS              PORTS               NAMES
f8aa1237bfca        alpine              "tail -f /etc/passwd"   52 seconds ago      Up 51 seconds                           program
e430d048c06a        alpine              "sh"                    4 minutes ago       Up 3 minutes                            monitor

[root@nyangawa-dev ~]# docker top f8a
UID                 PID                 PPID                C                   STIME               TTY                 TIME                CMD
root                28596               28578               0                   13:38               pts/0               00:00:00            tail -f /etc/passwd

[root@nyangawa-dev ~]# ps -ef | grep passwd | grep -v grep
root     28562 27432  0 13:38 pts/0    00:00:00 docker run -ti --rm --name program alpine tail -f /etc/passwd
root     28596 28578  0 13:38 pts/0    00:00:00 tail -f /etc/passwd
root     28703 28700  0 13:41 pts/3    00:00:00 tail -f /etc/passwd
```
Here, program A is running inside a container and opening/reading a sensitive file
while program B is not in a container and reading a sensitive file too. Now guess,
what happens when I execute the following commands in `monitor` container?
``` shell
# fd 3 is the first fd opened by the process which is the 'sensitive' file
# while 0,1,2 are reserved by stdio
ls -alh /host/proc/28596/fd/3
ls -alh /host/proc/28703/fd/3
```

The answer is quite straightforward if the reader is experienced in Linux, but
for those who does not have much experience, things are surprisingly different.

``` shell
/ # ls -alh /host/proc/28596/fd/3
lr-x------    1 root     root          64 Jul  9 05:43 /host/proc/28596/fd/3 -> /etc/passwd

/ # ls -alh /host/proc/28703/fd/3
ls: /host/proc/28703/fd/3: cannot read link: Permission denied
lr-x------    1 root     root          64 Jul  9 05:49 /host/proc/28703/fd/3
```

Monitor fails to stat the file descriptors of a process from the host, but for
another process running inside a container, there are no such limits. And here
comes another question: is `/host/proc/28596/fd/3` writable to monitor?

The answer is true:

``` shell
/ # echo 'nyangawa:x:1026:1026::/tmp:/bin/bash' >> /host/proc/28596/fd/3
/ # tail /host/proc/28596/fd/3
ntp:x:123:123:NTP:/var/empty:/sbin/nologin
smmsp:x:209:209:smmsp:/var/spool/mqueue:/sbin/nologin
guest:x:405:100:guest:/dev/null:/sbin/nologin
nobody:x:65534:65534:nobody:/:/sbin/nologin
nyangawa:x:1026:1026::/tmp:/bin/bash
```

Why? What's the reason of the difference here? How can I find it?
My first try was to use `strace` by running `strace ls -alh /host/proc/28703/fd/3`
in a container started with `docker run -v /proc:/host/proc -ti --cap-add SYS_PTRACE --rm alpine sh`.
Surprisingly (or not), the fd is readable now. The only difference here is that I
added `--cap-add SYS_PTRACE` to the new container. Although I didn't make it to
reproduce the problem, I can conclude that it might be a PTRACE related issue.

So it comes to the kernel now. I am pretty sure that it was because the `stat`
syscall returned `EPERM` to prevent me from following the symlink. I wrote a
simple program to prove this.

``` c++
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>
int main(){
struct stat sb1, sb2;
    int ret1 = stat("/host/proc/28596/fd/3", &sb1);
    int ret2 = stat("/host/proc/28703/fd/3", &sb2);
    printf("stat /host/proc/28596/fd/3 ret: %d\n", ret1);
    printf("stat /host/proc/28703/fd/3 ret: %d\n", ret2);
    return 0;
}
```
Output:

``` shell
[root@nyangawa-dev workplace]# gcc test.c && docker cp a.out monitor:/ && docker exec -ti monitor /a.out
stat /host/proc/28596/fd/3 ret: 0
stat /host/proc/28703/fd/3 ret: -1
```

Something in kernel refused my monitor process to access that file descriptor in procfs.
There are several ways to debug kernel, a traditional way is to start a linux in
a VM and attach a debugger to it. That's a little bit heavy for my question here.
So I decided to use a simpler way to trace the syscall which is [ftrace](https://www.kernel.org/doc/Documentation/trace/ftrace.txt)
and a very convenient wrapper called [trace-cmd](https://linux.die.net/man/1/trace-cmd)

Here is the results:
``` text
           a.out-22437 [003] 67534.327141: funcgraph_entry:        0.223 us   |                  security_inode_follow_link();
           a.out-22437 [003] 67534.327143: funcgraph_entry:                   |                  proc_pid_get_link() {
           a.out-22437 [003] 67534.327144: funcgraph_entry:                   |                    proc_pid_get_link.part.0() {
           a.out-22437 [003] 67534.327145: funcgraph_entry:                   |                      proc_fd_access_allowed() {
           a.out-22437 [003] 67534.327145: funcgraph_entry:                   |                        get_pid_task() {
           a.out-22437 [003] 67534.327146: funcgraph_entry:        0.170 us   |                          __rcu_read_lock();
           a.out-22437 [003] 67534.327148: funcgraph_entry:        0.173 us   |                          __rcu_read_unlock();
           a.out-22437 [003] 67534.327149: funcgraph_exit:         3.156 us   |                        }
           a.out-22437 [003] 67534.327150: funcgraph_entry:                   |                        ptrace_may_access() {
           a.out-22437 [003] 67534.327151: funcgraph_entry:                   |                          _raw_spin_lock() {
           a.out-22437 [003] 67534.327151: funcgraph_entry:        0.176 us   |                            preempt_count_add();
           a.out-22437 [003] 67534.327153: funcgraph_exit:         1.586 us   |                          }
           a.out-22437 [003] 67534.327154: funcgraph_entry:                   |                          __ptrace_may_access() {
           a.out-22437 [003] 67534.327154: funcgraph_entry:        0.163 us   |                            __rcu_read_lock();
           a.out-22437 [003] 67534.327156: funcgraph_entry:        0.176 us   |                            __rcu_read_unlock();
           a.out-22437 [003] 67534.327157: funcgraph_entry:                   |                            security_ptrace_access_check() {
           a.out-22437 [003] 67534.327159: funcgraph_entry:                   |                              cap_ptrace_access_check() {
           a.out-22437 [003] 67534.327159: funcgraph_entry:        0.171 us   |                                __rcu_read_lock();
           a.out-22437 [003] 67534.327161: funcgraph_entry:                   |                                ns_capable() {
           a.out-22437 [003] 67534.327162: funcgraph_entry:                   |                                  ns_capable_common() {
           a.out-22437 [003] 67534.327163: funcgraph_entry:                   |                                    security_capable() {
           a.out-22437 [003] 67534.327164: funcgraph_entry:        0.217 us   |                                      cap_capable();
           a.out-22437 [003] 67534.327166: funcgraph_exit:         2.339 us   |                                    }
           a.out-22437 [003] 67534.327167: funcgraph_exit:         3.860 us   |                                  }
           a.out-22437 [003] 67534.327167: funcgraph_exit:         5.437 us   |                                }
           a.out-22437 [003] 67534.327168: funcgraph_entry:        0.170 us   |                                __rcu_read_unlock();
           a.out-22437 [003] 67534.327169: funcgraph_exit:       + 10.145 us  |                              }
           a.out-22437 [003] 67534.327170: funcgraph_exit:       + 12.140 us  |                            }
           a.out-22437 [003] 67534.327171: funcgraph_exit:       + 16.779 us  |                          }
           a.out-22437 [003] 67534.327172: funcgraph_entry:                   |                          _raw_spin_unlock() {
           a.out-22437 [003] 67534.327172: funcgraph_entry:        0.163 us   |                            preempt_count_sub();
           a.out-22437 [003] 67534.327174: funcgraph_exit:         1.560 us   |                          }
           a.out-22437 [003] 67534.327174: funcgraph_exit:       + 24.007 us  |                        }
           a.out-22437 [003] 67534.327175: funcgraph_exit:       + 29.886 us  |                      }
           a.out-22437 [003] 67534.327176: funcgraph_exit:       + 31.548 us  |                    }
```

``` text
           a.out-22505 [002] 67607.750818: funcgraph_entry:        0.232 us   |                  security_inode_follow_link();
           a.out-22505 [002] 67607.750821: funcgraph_entry:                   |                  proc_pid_get_link() {
           a.out-22505 [002] 67607.750821: funcgraph_entry:                   |                    proc_pid_get_link.part.0() {
           a.out-22505 [002] 67607.750822: funcgraph_entry:                   |                      proc_fd_access_allowed() {
           a.out-22505 [002] 67607.750823: funcgraph_entry:                   |                        get_pid_task() {
           a.out-22505 [002] 67607.750824: funcgraph_entry:        0.170 us   |                          __rcu_read_lock();
           a.out-22505 [002] 67607.750825: funcgraph_entry:        0.160 us   |                          __rcu_read_unlock();
           a.out-22505 [002] 67607.750827: funcgraph_exit:         3.007 us   |                        }
           a.out-22505 [002] 67607.750827: funcgraph_entry:                   |                        ptrace_may_access() {
           a.out-22505 [002] 67607.750828: funcgraph_entry:                   |                          _raw_spin_lock() {
           a.out-22505 [002] 67607.750829: funcgraph_entry:        0.170 us   |                            preempt_count_add();
           a.out-22505 [002] 67607.750832: funcgraph_exit:         2.607 us   |                          }
           a.out-22505 [002] 67607.750833: funcgraph_entry:                   |                          __ptrace_may_access() {
           a.out-22505 [002] 67607.750835: funcgraph_entry:        0.159 us   |                            __rcu_read_lock();
           a.out-22505 [002] 67607.750836: funcgraph_entry:        0.163 us   |                            __rcu_read_unlock();
           a.out-22505 [002] 67607.750838: funcgraph_entry:                   |                            security_ptrace_access_check() {
           a.out-22505 [002] 67607.750839: funcgraph_entry:                   |                              cap_ptrace_access_check() {
           a.out-22505 [002] 67607.750840: funcgraph_entry:        0.163 us   |                                __rcu_read_lock();
           a.out-22505 [002] 67607.750842: funcgraph_entry:        0.170 us   |                                __rcu_read_unlock();
           a.out-22505 [002] 67607.750843: funcgraph_exit:         3.243 us   |                              }
           a.out-22505 [002] 67607.750844: funcgraph_entry:        0.188 us   |                              yama_ptrace_access_check();
           a.out-22505 [002] 67607.750846: funcgraph_exit:         7.191 us   |                            }
           a.out-22505 [002] 67607.750846: funcgraph_exit:       + 11.820 us  |                          }
           a.out-22505 [002] 67607.750847: funcgraph_entry:                   |                          _raw_spin_unlock() {
           a.out-22505 [002] 67607.750848: funcgraph_entry:        0.163 us   |                            preempt_count_sub();
           a.out-22505 [002] 67607.750849: funcgraph_exit:         1.597 us   |                          }
           a.out-22505 [002] 67607.750850: funcgraph_exit:       + 21.771 us  |                        }
           a.out-22505 [002] 67607.750851: funcgraph_exit:       + 27.566 us  |                      }
           a.out-22505 [002] 67607.750851: funcgraph_entry:                   |                      proc_fd_link() {
           a.out-22505 [002] 67607.750852: funcgraph_entry:                   |                        get_pid_task() {
           a.out-22505 [002] 67607.750853: funcgraph_entry:        0.166 us   |                          __rcu_read_lock();
           a.out-22505 [002] 67607.750854: funcgraph_entry:        0.160 us   |                          __rcu_read_unlock();
           a.out-22505 [002] 67607.750856: funcgraph_exit:         2.986 us   |                        }
           a.out-22505 [002] 67607.750856: funcgraph_entry:                   |                        get_files_struct() {

```

It looks like the different part is around `cap_ptrace_access_check`.
Here we can browse the source code [https://elixir.bootlin.com/linux/latest/source/security/commoncap.c#L139]

``` c++
/**
 * cap_ptrace_access_check - Determine whether the current process may access
 *			   another
 * @child: The process to be accessed
 * @mode: The mode of attachment.
 *
 * If we are in the same or an ancestor user_ns and have all the target
 * task's capabilities, then ptrace access is allowed.
 * If we have the ptrace capability to the target user_ns, then ptrace
 * access is allowed.
 * Else denied.
 *
 * Determine whether a process may access another, returning 0 if permission
 * granted, -ve if denied.
 */
int cap_ptrace_access_check(struct task_struct *child, unsigned int mode)
{
	int ret = 0;
	const struct cred *cred, *child_cred;
	const kernel_cap_t *caller_caps;

	rcu_read_lock();
	cred = current_cred();
	child_cred = __task_cred(child);
	if (mode & PTRACE_MODE_FSCREDS)
		caller_caps = &cred->cap_effective;
	else
		caller_caps = &cred->cap_permitted;
	if (cred->user_ns == child_cred->user_ns &&
	    cap_issubset(child_cred->cap_permitted, *caller_caps))
		goto out;
	if (ns_capable(child_cred->user_ns, CAP_SYS_PTRACE))
		goto out;
	ret = -EPERM;
out:
	rcu_read_unlock();
	return ret;
}
```

Finally, things are quite clear here. Docker, by default, does not isolate user
namespaces. So the monitor, program a and program b are running in the same
user namespace. Then the check comes to `cap_issubset(child_cred->cap_permitted, *caller_caps)`
, for containers, if not explicitly set, they have same capabilities so it checks
therefore fds are accessible. But for processes in host, `By default Docker
drops all capabilities except those needed, a whitelist instead of a blacklist
approach. You can see a full list of available capabilities in Linux manpages.`
according to [Docker security guide](https://docs.docker.com/engine/security/security/)

After all, everything is clear. And I found it afterwards in [ptrace man page](http://man7.org/linux/man-pages/man2/ptrace.2.html) :)

``` text
          b) Deny access if neither of the following is true:

             · The caller and the target process are in the same user
               namespace, and the caller's capabilities are a proper
               superset of the target process's permitted capabilities.

             · The caller has the CAP_SYS_PTRACE capability in the target
               process's user namespace.

             Note that the commoncap LSM does not distinguish between
             PTRACE_MODE_READ and PTRACE_MODE_ATTACH.
```
