---
layout: post
title: "Trying out continuous profiling"
date: 2023-12-20
author: Tao Xu
---

Recently I tried out continuous profiling tools ([Inspector-Gadget](https://www.inspektor-gadget.io/) and [Pyroscope](https://pyroscope.io/)). My observations are summarised bellow.

Inspector-Gadget is a `eBPF`-based plugin tool for developers familiar with `bcc` or `bpftrace`. And Pyroscope is a continuous profiling platform for application users. I have both of them installed and background services running in the VM minikube cluster.

## 1.Setup
1. Machine: 4 core 8 GiB VM, Apple M2 Pro arm64 CPU, native virtualization
2. OS: [Fedora 38](https://mac.getutm.app/gallery/fedora-38) (also tried CentOS-Stream 9, Ubuntu 22), Linux 6.5.10 (also tried linux [V1 patch](https://lore.kernel.org/linux-fsdevel/20220614180949.102914-1-bfoster@redhat.com/)), 4 KiB page size (also tried 16 KiB)
3. Go: go version go1.20.10 linux/arm64
4. Install `minikube`: https://minikube.sigs.k8s.io/docs/start/
5. Enable `Inspektor Gadget`: https://minikube.sigs.k8s.io/docs/handbook/addons/inspektor-gadget/
6. Install `Pyroscope`: https://grafana.com/docs/pyroscope/latest/get-started/

Note: pyroscope supports deployment by single bianry, docker and kubernetes. I tried docker and kubernetes. [AWS blog](https://aws.amazon.com/cn/blogs/china/continuous-profiling-of-eks-container-services-to-diagnose-application-performance-with-pyroscope/) (in Chinese) is a useful document for quick installation.

## 2.Obervation
After enabled the eBPF tracing agent, I left the cluster running idle for a few days. Host system utilization is low. Pyroscope eBPF agent keeps generating stack traces for all the target pods and writes to the local storage directory `/data`. It consumes less than 100MiB storage for 5 days depending on the number of target pods (I have 13 pods) and sampling frequency (default 97 Hz).

Then I found `os.ReadDir("/proc")` sometimes dominate the CPU time while CPU util% is low like <=5%. Here is the [flamegraph](https://flamegraph.com/share/d7c8341a-6d5e-11ee-b135-0a855ce7b751). <img src="/images/20231011_pyroscope.png" width=800>

## 3.Verification
`os.ReadDir()` is a wrapper of `getdents64()` like `glibc`, confirm it by tracing (2nd column is latency):
```
# perf trace -e getdents64 -p `pidof gadgettracermanager` sleep 3
     0.000 ( 0.335 ms): gadgettracerma/8544 getdents64(fd: 28, dirent: 0x40031a4000, count: 8192) = 8184
     0.397 ( 0.093 ms): gadgettracerma/8544 getdents64(fd: 28, dirent: 0x40031a4000, count: 8192) = 688
     0.665 ( 0.187 ms): gadgettracerma/8544 getdents64(fd: 28, dirent: 0x40031a4000, count: 8192) = 0
  1000.545 ( 1.035 ms): gadgettracerma/8544 getdents64(fd: 28, dirent: 0x4000af2000, count: 8192) = 8184
  1001.785 ( 0.258 ms): gadgettracerma/8544 getdents64(fd: 28, dirent: 0x4000af2000, count: 8192) = 688
  1002.070 ( 0.058 ms): gadgettracerma/8544 getdents64(fd: 28, dirent: 0x4000af2000, count: 8192) = 0
  2000.946 ( 1.208 ms): gadgettracerma/8544 getdents64(fd: 28, dirent: 0x40031a4000, count: 8192) = 8184
  2002.478 ( 0.199 ms): gadgettracerma/8544 getdents64(fd: 28, dirent: 0x40031a4000, count: 8192) = 688
  2002.820 ( 0.087 ms): gadgettracerma/8544 getdents64(fd: 28, dirent: 0x40031a4000, count: 8192) = 0
```

The monitoring daemon calls `os.ReadDir("/proc")` every 1s to get all process IDs, and latency spikes (_1ms to 2ms_) can be observed. The dirent buffer size is **8KiB**, one `os.ReadDir()` results in **3** `getdents64()` to get all the PIDs (roughly 300).

Trace `ps` to see how `glibc` behaves:
```
# perf trace -e getdents64 ps -eo pid > /dev/null
     0.000 ( 0.206 ms): ps/74672 getdents64(fd: 5, dirent: 0xaaaad37a9080, count: 32768) = 9552
     3.328 ( 0.023 ms): ps/74672 getdents64(fd: 5, dirent: 0xaaaad37a9080, count: 32768) = 0
```

`glibc` uses **32KiB** dirent buffer (from [commit](https://sourceware.org/git/?p=glibc.git;a=commit;h=4b962c9e859de23b461d61f860dbd3f21311e83a) and [mail](https://inbox.sourceware.org/libc-alpha/87lfml9gl7.fsf@mid.deneb.enyo.de/)), so result in **2** `getdents64()`, less syscalls. Latency is lower probably because of directory cache (dcache) hit warmed up by the monitoring daemon every 1s, see the reproduce program traces below.

## 4.Reproduce
This `os.ReadDir()` hotspot can be reproduced without the monitoring system:
```
# cat readdir.go
package main

import (
    "fmt"
    "os"
    "strconv"
    "time"
)

func repeat_routine(dir string, loop int) {
    var output []int
    for i := 0; i < loop; i++ {
        dirEntries, err := os.ReadDir(dir)
        check(err)
        output = nil
        for _, entry := range dirEntries {
            pid, err := strconv.Atoi(entry.Name())
            if err != nil {
                // entry is not a process directory. Ignore.
                continue
            }
            output = append(output, pid)
        }
    }
    fmt.Println(output)
}

func check(e error) {
    if e != nil {
        panic(e)
    }
}

func main() {
    if len(os.Args) < 4 {
        fmt.Println("Need input: dir loop runtime");
        os.Exit(-1)
    }

    dir := os.Args[1]
    count, err := strconv.Atoi(os.Args[2])
    check(err)
    runtime, err := strconv.Atoi(os.Args[3])
    check(err)

    for i := 0; i < runtime; i++ {
        repeat_routine(dir, count)
        time.Sleep(time.Second)
    }
}
```

Run above reproduce program and monitoring daemons at the same time:
```
# perf trace -e getdents64 ./readdir_go /proc 1 3 > /dev/null
     0.000 ( 0.272 ms): readdir_go/65676 getdents64(fd: 3, dirent: 0x40000b2000, count: 8192) = 8184
     0.314 ( 0.054 ms): readdir_go/65676 getdents64(fd: 3, dirent: 0x40000b2000, count: 8192) = 688
     0.384 ( 0.016 ms): readdir_go/65676 getdents64(fd: 3, dirent: 0x40000b2000, count: 8192) = 0
  1000.748 ( 0.368 ms): readdir_go/65676 getdents64(fd: 3, dirent: 0x4000108000, count: 8192) = 8184
  1001.295 ( 0.233 ms): readdir_go/65676 getdents64(fd: 3, dirent: 0x4000108000, count: 8192) = 688
  1001.591 ( 0.044 ms): readdir_go/65676 getdents64(fd: 3, dirent: 0x4000108000, count: 8192) = 0
  2003.251 ( 0.329 ms): readdir_go/65676 getdents64(fd: 3, dirent: 0x40000b2000, count: 8192) = 8184
  2003.802 ( 0.117 ms): readdir_go/65676 getdents64(fd: 3, dirent: 0x40000b2000, count: 8192) = 688
  2004.061 ( 0.070 ms): readdir_go/65676 getdents64(fd: 3, dirent: 0x40000b2000, count: 8192) = 0
```

Stop the monitoring daemons and run the program alone (8KiB buffer is sufficient when less threads):
```
# perf trace -e getdents64 ./readdir_go /proc 1 3 > /dev/null
     0.000 ( 0.271 ms): readdir_go/66797 getdents64(fd: 3, dirent: 0x4000120000, count: 8192) = 7920
     0.319 ( 0.018 ms): readdir_go/66797 getdents64(fd: 3, dirent: 0x4000120000, count: 8192) = 0
  1005.351 ( 1.412 ms): readdir_go/66797 getdents64(fd: 3, dirent: 0x40000a4000, count: 8192) = 7920
  1007.017 ( 0.200 ms): readdir_go/66797 getdents64(fd: 3, dirent: 0x40000a4000, count: 8192) = 0
  2012.598 ( 1.014 ms): readdir_go/66797 getdents64(fd: 3, dirent: 0x4000120000, count: 8192) = 7920
  2013.985 ( 0.126 ms): readdir_go/66797 getdents64(fd: 3, dirent: 0x4000120000, count: 8192) = 0
```

## 5.Bottom-up analysis
### a.System level
Some search results ([linux patch](https://lore.kernel.org/linux-fsdevel/20220614180949.102914-1-bfoster@redhat.com/) and [usenix slides](https://www.usenix.org/system/files/srecon22emea_slides_liku.pdf)) show Linux `procfs` has scalability issue in `proc_pid_readdir()` when many threads. Here it's a 4 core 8 GiB arm64 VM with 300 threads, and system is idle without any workloads yet. I did some verifications:
1. Guest VM is running 4KiB page size kernel while host processor's native page size is 16KiB. Memory traversal may be less efficient in this manner. I tried 16KiB page guest kernel, but not helpful.
2. I tried VMs in the cloud with same 4 core 8 GiB configuration and same software, `os.ReadDir("/proc")`'s max latency is lower than desktop VM: AMD Epyc 7T83 VM is < **0.28ms**, ARM Neoverse N2 VM is < **0.36ms**.
3. I applied the Linux kernel patch V1 in [3], it's helpful to eliminate radix-tree hotspot, but remaining part `__d_lookup()` then dominates in `proc_pid_readdir()`. Here is the [flamegraph](https://flamegraph.com/share/6d993cc0-926c-11ee-b135-0a855ce7b751). <img src="/images/20231201_pyroscope.png" width=800>
4. This linux [commit](https://github.com/torvalds/linux/commit/3ba4bceef23206349d4130ddf140819b365de7c8) mentioned `proc_pid_readdir()` can take more than 50ms and added `cond_resched()` in it.

So `os.ReadDir("/proc")` can be slow in Linux systems, sometimes because of the processor, sometimes system loads.

### b.Standard library
I found this `go` [issue](https://github.com/golang/go/issues/24015) increased the `blockSize` to 8KiB to fix missing CIFS file bug. It's somewhat similar to the comment in `glibc` [2], so increase to the same 32KiB lower limit as `glibc` may be good for both compatibility and performance described above.

Checked `solaris/illumos libc` as a reference, filesystem independent dirent buffer lower limit is [8KiB](https://github.com/illumos/illumos-gate/blob/6a72ec1a152c1e6da18624f19eacab252bb91662/usr/src/head/dirent.h#L47), and upper limit is [64KiB](https://github.com/illumos/illumos-gate/blob/6a72ec1a152c1e6da18624f19eacab252bb91662/usr/src/uts/common/sys/dirent.h#L94).

### c.Application level
Monitoring software checks PIDs often, some may change to use `sysfs` instead of `procfs`, like the `cgroup.procs` file for container process where linux sub-system stores the PIDs in one memory place without frequent memory traversal in `/proc`.

## 5.Suggestion
I opend a `go` [proposal](https://github.com/golang/go/issues/64597) to increase the dirent buffer `blockSize` from **8KiB to 32KiB** in `src/os/dir_unix.go` for `os.ReadDir()` to align with `glibc` (check [commit](https://sourceware.org/git/?p=glibc.git;a=commit;h=4b962c9e859de23b461d61f860dbd3f21311e83a) and [mail](https://inbox.sourceware.org/libc-alpha/87lfml9gl7.fsf@mid.deneb.enyo.de/)).
```
diff --git a/src/os/dir_unix.go b/src/os/dir_unix.go
index 266a78acaf..6e8c821c21 100644
--- a/src/os/dir_unix.go
+++ b/src/os/dir_unix.go
@@ -23,7 +23,7 @@ type dirInfo struct {
 
 const (
        // More than 5760 to work around https://golang.org/issue/24015.
-       blockSize = 8192
+       blockSize = 32768
 )
```

