---
layout: post
title: "Evaluating GPU IO"
date: 2026-10-26
author: Tao Xu
---

The industry-standard IO benchmark tool `fio` recently added a new ["GPU IO engine"](https://github.com/axboe/fio/pull/2113) for AMD ROCm hardware. Nvidia GPUDirect Storage (GDS) has been supported for a while, and emerging KV-cache workloads heavily rely on GPU-accelerated IO. Below is a brief analysis of AMD ROCm `hipFile` (including the base `POSIX` H2D/D2H memcpy without GPU acceleration) and [uGDS](https://github.com/ScaleX-IO/uGDS), a recent research project acting as a host CPU side equivalent to `SPDK`.

#### TL;DR
 - **GPU accelerated IO path**: directly transfer data between NVMe SSD and GPU VRAM via P2P-DMA, bypassing host CPU DRAM (bouncing buffer) to reduce latency by **2x** and CPU utilization by **3x**.
 - **uGDS**: A user-space poll mode library (irq capable for large IO) at CPU host side pushing GPU IO performance further without moving IO control plane to GPU side.

## 1. Platform setup
I'm using a consumer desktop with previous-generation hardware and fairly new software:

| Component | Config | Notes |
| :--- | :--- | :--- |
| CPU | Intel i9-10900X | mitigations=off iommu=pt hugepages=1024 hugepagesz=2M |
| GPU | AMD Radeon RX 7900 XTX | gfx1100, 24GB VRAM<br>amdgpu v7.1.3, ROCm v10 |
| NVMe SSD | KINGSTON SNV2S2000G | PCIe gen4 but downgraded to gen3 speed, QLC |
| OS | Ubuntu-24.04 (Linux-6.8) | CONFIG_PCI_P2PDMA=y<br>CONFIG_HSA_AMD_P2P=y<br>CONFIG_DMABUF_MOVE_NOTIFY=y |

System setup references:
 - [hipFile installation guide](https://rocm.docs.amd.com/projects/hipFile/en/latest/install/install.html)
 - [uGDS installation guide](https://github.com/ScaleX-IO/uGDS/blob/main/docs/installation.md)

**Notes**:
 1. Resizable BAR (ReBAR) or Smart Access Memory (SAM) is required by the `amdgpu kfd` driver (*v2327936*) for `hipFile`.
 2. Backup the data on the NVMe SSD under test: `hipFile` is on top of filesystem, whereas `uGDS` is based on user-space `libnvm` that will take exclusive control of raw NVMe device from the kernel's `nvme` block driver.
 3. Disable PCIe ACS/power/security features as required by P2P-DMA. Ideally, the NVMe SSD should share the same PCIe switch or root complex as the GPU.

(From the textbook, PCIe P2P requires specific topology, but according to my test as far as GPU VRAM is successfully pinned for NVMe SSD's DMA engine to bypass CPU DRAM's bouncing buffer, TLP traversing a longer path is still fine. Drivers may restrict this in future, so sticking to official guides is recommended.)

## 2. fio microbenchmark
With the new `gpuaccel` abstraction layer in `fio`, both `hipFile` and `cuFile` share same logic:
 - `psync`: standard posix IO pread/pwrite, no GPU IO is involved;
 - `posix`: the posix fallback/compact mode in gpuaccel, e.g. pread first and then H2D memcpy from CPU host DRAM to GPU device VRAM for GPU kernel (for pwrite, D2H memcpy first before CPU side IO stack sends the NVMe write);
 - `hipFile/cuFile`: the fastpath mode in gpuaccel, directly DMA bulk transfer NVMe data payload between NVMe SSD and GPU VRAM, without H2D or D2H memcpy;

### 2.1 IO pattern
IO pattern is basic read in 1MiB size to reach peak bandwidth for the QLC SSD. A simple fio job example can be found [here](https://github.com/ROCm/rocm-systems/issues/11919).

### 2.2 IO data

#### 📊 1MiB numjobs=1

| Metrics | psync | posix | hipFile |
| :--- | :---: | :---: | :---: |
| Throughput (MiB/s) | 1921 | 1367 | 2566 |
| IOPS | 1920 | 1367 | 2565 |
| Latency (us) | 517 | 727 | 386 |

#### 📊 256KiB numjobs=4

| Metrics | psync | posix | hipFile |
| :--- | :---: | :---: | :---: |
| Throughput (MiB/s) | 2596 | 2258 | 2616 |
| IOPS | 10.4k | 9031 | 10.5k |
| Latency (us) | 382 | 438 | 378 |

#### 📊 128KiB numjobs=8

| Metrics | psync | posix | hipFile |
| :--- | :---: | :---: | :---: |
| Throughput (MiB/s) | 1939 | 1570 | 2056 |
| IOPS | 15.5k | 12.6k | 16.4k |
| Latency (us) | 508 | 622 | 478 |


Observations:
 - The `posix` fallback/compact mode is 20-30% slower than baseline `psync pread` due to the additional `H2D memcpy`;
 - The `hipFile` fastpath eliminates host bouncing buffer (`H2D memcpy`), and the P2P-DMA path is also 1-25% faster;

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1000 320" width="100%" height="100%" style="font-family: system-ui, -apple-system, BlinkMacSystemFont, &#39;Segoe UI&#39;, Roboto, sans-serif;">
  <rect width="100%" height="100%" fill="#ffffff" />
  <g transform="translate(0, 0)">
    <text x="165" y="25" font-size="15" font-weight="bold" fill="#1e293b" text-anchor="middle">Throughput (MiB/s)</text>
    <rect x="25" y="38" width="80" height="60" fill="#ffffff" stroke="#cbd5e1" stroke-width="1" rx="2" />
    <text x="32" y="52" font-size="10" fill="#475569" font-weight="600">Backend</text>
    <rect x="32" y="58" width="12" height="8" fill="#3b82f6" />
    <text x="48" y="65" font-size="10" fill="#334155">psync</text>
    <rect x="32" y="70" width="12" height="8" fill="#f97316" />
    <text x="48" y="77" font-size="10" fill="#334155">posix</text>
    <rect x="32" y="82" width="12" height="8" fill="#22c55e" />
    <text x="48" y="89" font-size="10" fill="#334155">hipfile</text>
    <g stroke="#e2e8f0" stroke-width="1" stroke-dasharray="3,3">
      <line x1="45" y1="260" x2="310" y2="260" />
      <line x1="45" y1="225" x2="310" y2="225" />
      <line x1="45" y1="190" x2="310" y2="190" />
      <line x1="45" y1="155" x2="310" y2="155" />
      <line x1="45" y1="120" x2="310" y2="120" />
      <line x1="45" y1="85" x2="310" y2="85" />
      <line x1="45" y1="50" x2="310" y2="50" />
    </g>
    <g font-size="11" fill="#64748b" text-anchor="end">
      <text x="38" y="264">0</text>
      <text x="38" y="229">500</text>
      <text x="38" y="194">1000</text>
      <text x="38" y="159">1500</text>
      <text x="38" y="124">2000</text>
      <text x="38" y="89">2500</text>
      <text x="38" y="54">3000</text>
    </g>
    <line x1="45" y1="50" x2="45" y2="260" stroke="#cbd5e1" stroke-width="1" />
    <line x1="45" y1="260" x2="310" y2="260" stroke="#cbd5e1" stroke-width="1" />
    <rect x="60" y="125.53" width="22" height="134.47" fill="#3b82f6" stroke="#0f172a" stroke-width="0.6" />
    <text x="71" y="120.53" font-size="9" font-weight="bold" fill="#0f172a" text-anchor="middle">1921</text>
    <rect x="83" y="164.31" width="22" height="95.69" fill="#f97316" stroke="#0f172a" stroke-width="0.6" />
    <text x="94" y="159.31" font-size="9" font-weight="bold" fill="#0f172a" text-anchor="middle">1367</text>
    <rect x="106" y="80.38" width="22" height="179.62" fill="#22c55e" stroke="#0f172a" stroke-width="0.6" />
    <text x="117" y="75.38" font-size="9" font-weight="bold" fill="#0f172a" text-anchor="middle">2566</text>
    <rect x="145" y="78.28" width="22" height="181.72" fill="#3b82f6" stroke="#0f172a" stroke-width="0.6" />
    <text x="156" y="73.28" font-size="9" font-weight="bold" fill="#0f172a" text-anchor="middle">2596</text>
    <rect x="168" y="101.94" width="22" height="158.06" fill="#f97316" stroke="#0f172a" stroke-width="0.6" />
    <text x="179" y="96.94" font-size="9" font-weight="bold" fill="#0f172a" text-anchor="middle">2258</text>
    <rect x="191" y="76.88" width="22" height="183.12" fill="#22c55e" stroke="#0f172a" stroke-width="0.6" />
    <text x="202" y="71.88" font-size="9" font-weight="bold" fill="#0f172a" text-anchor="middle">2616</text>
    <rect x="230" y="124.27" width="22" height="135.73" fill="#3b82f6" stroke="#0f172a" stroke-width="0.6" />
    <text x="241" y="119.27" font-size="9" font-weight="bold" fill="#0f172a" text-anchor="middle">1939</text>
    <rect x="253" y="150.10" width="22" height="109.90" fill="#f97316" stroke="#0f172a" stroke-width="0.6" />
    <text x="264" y="145.10" font-size="9" font-weight="bold" fill="#0f172a" text-anchor="middle">1570</text>
    <rect x="276" y="116.08" width="22" height="143.92" fill="#22c55e" stroke="#0f172a" stroke-width="0.6" />
    <text x="287" y="111.08" font-size="9" font-weight="bold" fill="#0f172a" text-anchor="middle">2056</text>
    <g font-size="11" font-weight="bold" fill="#1e293b" text-anchor="middle">
      <text x="94" y="278">1MiB (numjobs=1)</text>
      <text x="179" y="278">256KiB (numjobs=4)</text>
      <text x="264" y="278">128KiB (numjobs=8)</text>
    </g>
  </g>
  <g transform="translate(333, 0)">
    <text x="165" y="25" font-size="15" font-weight="bold" fill="#1e293b" text-anchor="middle">IOPS</text>
    <rect x="25" y="38" width="80" height="60" fill="#ffffff" stroke="#cbd5e1" stroke-width="1" rx="2" />
    <text x="32" y="52" font-size="10" fill="#475569" font-weight="600">Backend</text>
    <rect x="32" y="58" width="12" height="8" fill="#3b82f6" />
    <text x="48" y="65" font-size="10" fill="#334155">psync</text>
    <rect x="32" y="70" width="12" height="8" fill="#f97316" />
    <text x="48" y="77" font-size="10" fill="#334155">posix</text>
    <rect x="32" y="82" width="12" height="8" fill="#22c55e" />
    <text x="48" y="89" font-size="10" fill="#334155">hipfile</text>
    <g stroke="#e2e8f0" stroke-width="1" stroke-dasharray="3,3">
      <line x1="50" y1="260" x2="310" y2="260" />
      <line x1="50" y1="233.75" x2="310" y2="233.75" />
      <line x1="50" y1="207.5" x2="310" y2="207.5" />
      <line x1="50" y1="181.25" x2="310" y2="181.25" />
      <line x1="50" y1="155" x2="310" y2="155" />
      <line x1="50" y1="128.75" x2="310" y2="128.75" />
      <line x1="50" y1="102.5" x2="310" y2="102.5" />
      <line x1="50" y1="76.25" x2="310" y2="76.25" />
      <line x1="50" y1="50" x2="310" y2="50" />
    </g>
    <g font-size="11" fill="#64748b" text-anchor="end">
      <text x="42" y="264">0</text>
      <text x="42" y="237.75">2500</text>
      <text x="42" y="211.5">5000</text>
      <text x="42" y="185.25">7500</text>
      <text x="42" y="159">10000</text>
      <text x="42" y="132.75">12500</text>
      <text x="42" y="106.5">15000</text>
      <text x="42" y="80.25">17500</text>
      <text x="42" y="54">20000</text>
    </g>
    <line x1="50" y1="50" x2="50" y2="260" stroke="#cbd5e1" stroke-width="1" />
    <line x1="50" y1="260" x2="310" y2="260" stroke="#cbd5e1" stroke-width="1" />
    <rect x="63" y="240.05" width="22" height="19.95" fill="#3b82f6" stroke="#0f172a" stroke-width="0.6" />
    <text x="74" y="235.05" font-size="9" font-weight="bold" fill="#0f172a" text-anchor="middle">1.9k</text>
    <rect x="86" y="245.30" width="22" height="14.70" fill="#f97316" stroke="#0f172a" stroke-width="0.6" />
    <text x="97" y="240.30" font-size="9" font-weight="bold" fill="#0f172a" text-anchor="middle">1.4k</text>
    <rect x="109" y="232.70" width="22" height="27.30" fill="#22c55e" stroke="#0f172a" stroke-width="0.6" />
    <text x="120" y="227.70" font-size="9" font-weight="bold" fill="#0f172a" text-anchor="middle">2.6k</text>
    <rect x="148" y="150.80" width="22" height="109.20" fill="#3b82f6" stroke="#0f172a" stroke-width="0.6" />
    <text x="159" y="145.80" font-size="9" font-weight="bold" fill="#0f172a" text-anchor="middle">10.4k</text>
    <rect x="171" y="165.50" width="22" height="94.50" fill="#f97316" stroke="#0f172a" stroke-width="0.6" />
    <text x="182" y="160.50" font-size="9" font-weight="bold" fill="#0f172a" text-anchor="middle">9.0k</text>
    <rect x="194" y="149.75" width="22" height="110.25" fill="#22c55e" stroke="#0f172a" stroke-width="0.6" />
    <text x="205" y="144.75" font-size="9" font-weight="bold" fill="#0f172a" text-anchor="middle">10.5k</text>
    <rect x="233" y="97.25" width="22" height="162.75" fill="#3b82f6" stroke="#0f172a" stroke-width="0.6" />
    <text x="244" y="92.25" font-size="9" font-weight="bold" fill="#0f172a" text-anchor="middle">15.5k</text>
    <rect x="256" y="127.70" width="22" height="132.30" fill="#f97316" stroke="#0f172a" stroke-width="0.6" />
    <text x="267" y="122.70" font-size="9" font-weight="bold" fill="#0f172a" text-anchor="middle">12.6k</text>
    <rect x="279" y="87.80" width="22" height="172.20" fill="#22c55e" stroke="#0f172a" stroke-width="0.6" />
    <text x="290" y="82.80" font-size="9" font-weight="bold" fill="#0f172a" text-anchor="middle">16.4k</text>
    <g font-size="11" font-weight="bold" fill="#1e293b" text-anchor="middle">
      <text x="97" y="278">1MiB (numjobs=1)</text>
      <text x="182" y="278">256KiB (numjobs=4)</text>
      <text x="267" y="278">128KiB (numjobs=8)</text>
    </g>
  </g>
  <g transform="translate(666, 0)">
    <text x="165" y="25" font-size="15" font-weight="bold" fill="#1e293b" text-anchor="middle">Latency (µs)</text>
    <rect x="25" y="38" width="80" height="60" fill="#ffffff" stroke="#cbd5e1" stroke-width="1" rx="2" />
    <text x="32" y="52" font-size="10" fill="#475569" font-weight="600">Backend</text>
    <rect x="32" y="58" width="12" height="8" fill="#3b82f6" />
    <text x="48" y="65" font-size="10" fill="#334155">psync</text>
    <rect x="32" y="70" width="12" height="8" fill="#f97316" />
    <text x="48" y="77" font-size="10" fill="#334155">posix</text>
    <rect x="32" y="82" width="12" height="8" fill="#22c55e" />
    <text x="48" y="89" font-size="10" fill="#334155">hipfile</text>
    <g stroke="#e2e8f0" stroke-width="1" stroke-dasharray="3,3">
      <line x1="45" y1="260" x2="310" y2="260" />
      <line x1="45" y1="233.75" x2="310" y2="233.75" />
      <line x1="45" y1="207.5" x2="310" y2="207.5" />
      <line x1="45" y1="181.25" x2="310" y2="181.25" />
      <line x1="45" y1="155" x2="310" y2="155" />
      <line x1="45" y1="128.75" x2="310" y2="128.75" />
      <line x1="45" y1="102.5" x2="310" y2="102.5" />
      <line x1="45" y1="76.25" x2="310" y2="76.25" />
      <line x1="45" y1="50" x2="310" y2="50" />
    </g>
    <g font-size="11" fill="#64748b" text-anchor="end">
      <text x="38" y="264">0</text>
      <text x="38" y="237.75">100</text>
      <text x="38" y="211.5">200</text>
      <text x="38" y="185.25">300</text>
      <text x="38" y="159">400</text>
      <text x="38" y="132.75">500</text>
      <text x="38" y="106.5">600</text>
      <text x="38" y="80.25">700</text>
      <text x="38" y="54">800</text>
    </g>
    <line x1="45" y1="50" x2="45" y2="260" stroke="#cbd5e1" stroke-width="1" />
    <line x1="45" y1="260" x2="310" y2="260" stroke="#cbd5e1" stroke-width="1" />
    <rect x="58" y="124.29" width="22" height="135.71" fill="#3b82f6" stroke="#0f172a" stroke-width="0.6" />
    <text x="69" y="119.29" font-size="9" font-weight="bold" fill="#0f172a" text-anchor="middle">517</text>
    <rect x="81" y="69.21" width="22" height="190.79" fill="#f97316" stroke="#0f172a" stroke-width="0.6" />
    <text x="92" y="64.21" font-size="9" font-weight="bold" fill="#0f172a" text-anchor="middle">727</text>
    <rect x="104" y="158.675" width="22" height="101.325" fill="#22c55e" stroke="#0f172a" stroke-width="0.6" />
    <text x="115" y="153.675" font-size="9" font-weight="bold" fill="#0f172a" text-anchor="middle">386</text>
    <rect x="143" y="159.725" width="22" height="100.275" fill="#3b82f6" stroke="#0f172a" stroke-width="0.6" />
    <text x="154" y="154.725" font-size="9" font-weight="bold" fill="#0f172a" text-anchor="middle">382</text>
    <rect x="166" y="145.025" width="22" height="114.975" fill="#f97316" stroke="#0f172a" stroke-width="0.6" />
    <text x="177" y="140.025" font-size="9" font-weight="bold" fill="#0f172a" text-anchor="middle">438</text>
    <rect x="189" y="160.775" width="22" height="99.225" fill="#22c55e" stroke="#0f172a" stroke-width="0.6" />
    <text x="200" y="155.775" font-size="9" font-weight="bold" fill="#0f172a" text-anchor="middle">378</text>
    <rect x="228" y="126.65" width="22" height="133.35" fill="#3b82f6" stroke="#0f172a" stroke-width="0.6" />
    <text x="239" y="121.65" font-size="9" font-weight="bold" fill="#0f172a" text-anchor="middle">508</text>
    <rect x="251" y="96.725" width="22" height="163.275" fill="#f97316" stroke="#0f172a" stroke-width="0.6" />
    <text x="262" y="91.725" font-size="9" font-weight="bold" fill="#0f172a" text-anchor="middle">622</text>
    <rect x="274" y="134.525" width="22" height="125.475" fill="#22c55e" stroke="#0f172a" stroke-width="0.6" />
    <text x="285" y="129.525" font-size="9" font-weight="bold" fill="#0f172a" text-anchor="middle">478</text>
    <g font-size="11" font-weight="bold" fill="#1e293b" text-anchor="middle">
      <text x="92" y="278">1MiB (numjobs=1)</text>
      <text x="177" y="278">256KiB (numjobs=4)</text>
      <text x="262" y="278">128KiB (numjobs=8)</text>
    </g>
  </g>
</svg>

### 2.3 CPU util
GPU accelerated IO also consumes 3x less CPU as expected:
 - From fio CPU stat: 36.8% -> 3% in user, 52% -> 25% in sys
 - From perf sched: 0.882 -> 0.374 CPUs utilized

### 2.3 latency breakdown
Use a simple `bpftrace` script to trace function call/return:
```
$ bpftrace ./stat.bt
Attaching 11 probes...
Tracing hipFile I/O operations... Press Ctrl-C to stop.
TIME(us)  TID    COMM    func(Return)  arg(latency)
2525415   48764  fio     __pread64     10653 bytes
2525436   48764  fio     __pread64     10653       24us
2534500   48764  fio     __pread64     236   bytes
2534505   48764  fio     __pread64     236         8us
2543135   48764  fio     __pread64     1048576 bytes
2552165   48764  fio     __pread64     1048576     9032us
2552252   48764  fio     libhipfile_memcpy 1048576 bytes
2568026   48764  fio     libhipfile_memcpy 0       15776us
2568080   48764  fio     __pread64     1048576 bytes
2568531   48764  fio     __pread64     1048576     453us
2568556   48764  fio     libhipfile_memcpy 1048576 bytes
2568820   48764  fio     libhipfile_memcpy 0       265us
2568847   48764  fio     __pread64     1048576 bytes
2569270   48764  fio     __pread64     1048576     424us
2569300   48764  fio     libhipfile_memcpy 1048576 bytes
2569515   48764  fio     libhipfile_memcpy 0       216us
2569529   48764  fio     __pread64     1048576 bytes
2569958   48764  fio     __pread64     1048576     429us
2569971   48764  fio     libhipfile_memcpy 1048576 bytes
2570239   48764  fio     libhipfile_memcpy 0       268us
2570252   48764  fio     __pread64     1048576 bytes
2570659   48764  fio     __pread64     1048576     407us
2570679   48764  fio     libhipfile_memcpy 1048576 bytes
2570908   48764  fio     libhipfile_memcpy 0       229us
^C

$ bpftrace ./stat.bt
Attaching 11 probes...
Tracing hipFile I/O operations... Press Ctrl-C to stop.
TIME(us)  TID    COMM    func(Return)  arg(latency)
4645899   48804  fio     __pread64     10653 bytes
4645919   48804  fio     __pread64     10653       23us
4654640   48804  fio     __pread64     236   bytes
4654644   48804  fio     __pread64     236         6us
4663909   48804  fio     hipFileRead   1048576 bytes
4663964   48804  fio     hipAmdFileRead 1048576 bytes
4672801   48804  fio     hipAmdFileRead(R) 1048576   8838us
4672848   48804  fio     hipFileRead   1048576     8941us
4672870   48804  fio     hipFileRead   1048576 bytes
4672878   48804  fio     hipAmdFileRead 1048576 bytes
4673268   48804  fio     hipAmdFileRead(R) 1048576   390us
4673276   48804  fio     hipFileRead   1048576     407us
4673286   48804  fio     hipFileRead   1048576 bytes
4673291   48804  fio     hipAmdFileRead 1048576 bytes
4673636   48804  fio     hipAmdFileRead(R) 1048576   345us
4673642   48804  fio     hipFileRead   1048576     357us
4673650   48804  fio     hipFileRead   1048576 bytes
4673654   48804  fio     hipAmdFileRead 1048576 bytes
4673997   48804  fio     hipAmdFileRead(R) 1048576   343us
4674004   48804  fio     hipFileRead   1048576     354us
4674011   48804  fio     hipFileRead   1048576 bytes
4674014   48804  fio     hipAmdFileRead 1048576 bytes
4674354   48804  fio     hipAmdFileRead(R) 1048576   339us
4674360   48804  fio     hipFileRead   1048576     350us
^C
```

The `hipFile` fastpath latency stablizes at around **350us** for 1MiB read. The `posix pread` slowpath latency is around **430us**, and H2D memcpy takes another **270us** to copy the read data from CPU host DRAM (bouncing buffer) to GPU device memory. Altogether, `hipFile` is **2x** faster while consume **3x** less CPU compute.

```
==================================================================================
1. POSIX Path (High CPU Overhead | ~89.6% CPU | BW: 1351 MiB/s)
   [ CPU Cost: usr=36.8%, sys=52.8% (ctx=21.2k) ]
==================================================================================

fio (App)                Host DRAM               NVMe SSD               GPU VRAM
    |                        |                       |                     |
    |--- 1. pread64() ------------------------------>|                     |
    |                        |<-2. NVMe read (430us)-|                     |
    |                        |                       |                     |
    |--- 3. H2D memcpy --------------------------------------------------->|
    |                        |-- 4. Copy DRAM to VRAM (270us) ------------>|
    |                        |                       |                     |

================================================================================
2. hipFile Path (Low CPU Overhead | ~27.3% CPU | BW: 2290 MiB/s)
   [ CPU Cost: usr=2.8%, sys=24.6% (ctx=10.4k) ]
================================================================================

fio (App)                Host DRAM               NVMe SSD               GPU VRAM
    |                        |                       |                     |
    |--- 1. hipFileRead() -------------------------->|                     |
    |                        |                       |-2. P2P-DMA (350us)->|
    |                        |                       |                     |
```

## 3. uGDS
There're attempts to push control plane further to GPU (XIO), so GPU would take all control (or a dedicated DPU, just not CPU, check this recent [SDC session](https://www.snia.org/sniadeveloper/session/19686)). Before moving there, and potentially re-creating IO stack for GPU, the CPU host side can leverage user-space poll mode driver like SPDK to bypass OS kernel layers. `uGDS` is a research project just for this in GPU accelerated IO path.

I'm fortunate to collaborate with ScaleX-IO researchers when the research paper is published. And I view this as the new performant baseline before GPU take over both control and data planes. And below is a brief comparison.

Note: uGDS is user-space block engine, like a disk drive that GPU can manage directly. While `hipFile or cuFile` is with filesystem, so not a strict apple-to-apple comparison. However, emerging KV cache workload fits more without filesystem as it's utilizing memory cache not file.

## 3.1 sync benchmark
`fio` doesn't have uGDS. Below is an example for a simple 1MiB sync read comparison:
```
IO_DIR=/mnt/nvme3/test_fio GPU_DEV_IDS=0 /opt/rocm/bin/ais-stats ./fio ./examples/hipfile.fio

sudo ./build/bench_ugds_hip -f /dev/ugds_drv0 -l 4G -s 1M -d 0 -t 1 -i 1 -m read
```

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 900 320" width="100%" height="100%" style="font-family: system-ui, -apple-system, BlinkMacSystemFont, &#39;Segoe UI&#39;, Roboto, sans-serif;">
  <defs>
    <pattern id="stripe-pattern" width="10" height="10" patternTransform="rotate(45 0 0)" patternUnits="userSpaceOnUse">
      <line x1="0" y1="0" x2="0" y2="10" stroke="#1d4ed8" stroke-width="3" opacity="0.4" />
      <line x1="0" y1="0" x2="0" y2="10" stroke="#93c5fd" stroke-width="7" opacity="0.6" />
    </pattern>
  </defs>
  <rect width="100%" height="100%" fill="#ffffff" />
  <g transform="translate(0, 0)">
    <text x="150" y="25" font-size="15" font-weight="bold" fill="#1e293b" text-anchor="middle">Throughput (GiB/s)</text>
    <rect x="25" y="38" width="80" height="60" fill="#ffffff" stroke="#cbd5e1" stroke-width="1" rx="2" />
    <text x="32" y="52" font-size="10" fill="#475569" font-weight="600">Backend</text>
    <rect x="32" y="58" width="12" height="8" fill="#1d4ed8" />
    <text x="48" y="65" font-size="10" fill="#334155">posix</text>
    <rect x="32" y="70" width="12" height="8" fill="#ea580c" />
    <text x="48" y="77" font-size="10" fill="#334155">hipfile</text>
    <rect x="32" y="82" width="12" height="8" fill="#16a34a" />
    <text x="48" y="89" font-size="10" fill="#334155">uGDS</text>
    <g stroke="#e2e8f0" stroke-width="1" stroke-dasharray="3,3">
      <line x1="40" y1="260" x2="280" y2="260" />
      <line x1="40" y1="230" x2="280" y2="230" />
      <line x1="40" y1="200" x2="280" y2="200" />
      <line x1="40" y1="170" x2="280" y2="170" />
      <line x1="40" y1="140" x2="280" y2="140" />
      <line x1="40" y1="110" x2="280" y2="110" />
      <line x1="40" y1="80" x2="280" y2="80" />
      <line x1="40" y1="50" x2="280" y2="50" />
    </g>
    <g font-size="11" fill="#64748b" text-anchor="end">
      <text x="32" y="264">0.0</text>
      <text x="32" y="234">0.5</text>
      <text x="32" y="204">1.0</text>
      <text x="32" y="174">1.5</text>
      <text x="32" y="144">2.0</text>
      <text x="32" y="114">2.5</text>
      <text x="32" y="84">3.0</text>
      <text x="32" y="54">3.5</text>
    </g>
    <line x1="40" y1="50" x2="40" y2="260" stroke="#cbd5e1" stroke-width="1" />
    <line x1="40" y1="260" x2="280" y2="260" stroke="#cbd5e1" stroke-width="1" />
    <rect x="65" y="182" width="45" height="78" fill="#1d4ed8" stroke="#0f172a" stroke-width="0.8" />
    <text x="87.5" y="176" font-size="12" font-weight="bold" fill="#0f172a" text-anchor="middle">1.3</text>
    <rect x="137.5" y="104" width="45" height="156" fill="#ea580c" stroke="#0f172a" stroke-width="0.8" />
    <text x="160" y="98" font-size="12" font-weight="bold" fill="#0f172a" text-anchor="middle">2.6</text>
    <rect x="210" y="92" width="45" height="168" fill="#16a34a" stroke="#0f172a" stroke-width="0.8" />
    <text x="232.5" y="86" font-size="12" font-weight="bold" fill="#0f172a" text-anchor="middle">2.8</text>
    <g font-size="12" font-weight="bold" fill="#1e293b" text-anchor="middle">
      <text x="87.5" y="278">posix</text>
      <text x="160" y="278">hipfile</text>
      <text x="232.5" y="278">uGDS</text>
    </g>
  </g>
  <g transform="translate(300, 0)">
    <text x="150" y="25" font-size="15" font-weight="bold" fill="#1e293b" text-anchor="middle">Latency Breakdown (µs)</text>
    <rect x="135" y="38" width="140" height="45" fill="#ffffff" stroke="#cbd5e1" stroke-width="1" rx="2" />
    <text x="205" y="50" font-size="10" fill="#475569" font-weight="600" text-anchor="middle">Components</text>
    <rect x="142" y="56" width="12" height="8" fill="#1d4ed8" />
    <text x="158" y="63" font-size="9" fill="#334155">1MiB sync read</text>
    <rect x="142" y="68" width="12" height="8" fill="url(#stripe-pattern)" stroke="#1d4ed8" stroke-width="0.5" />
    <text x="158" y="75" font-size="9" fill="#334155">H2D memcpy (breakdown)</text>
    <g stroke="#e2e8f0" stroke-width="1" stroke-dasharray="3,3">
      <line x1="40" y1="260" x2="280" y2="260" />
      <line x1="40" y1="233.75" x2="280" y2="233.75" />
      <line x1="40" y1="207.5" x2="280" y2="207.5" />
      <line x1="40" y1="181.25" x2="280" y2="181.25" />
      <line x1="40" y1="155" x2="280" y2="155" />
      <line x1="40" y1="128.75" x2="280" y2="128.75" />
      <line x1="40" y1="102.5" x2="280" y2="102.5" />
      <line x1="40" y1="76.25" x2="280" y2="76.25" />
      <line x1="40" y1="50" x2="280" y2="50" />
    </g>
    <g font-size="11" fill="#64748b" text-anchor="end">
      <text x="32" y="264">0</text>
      <text x="32" y="237.75">100</text>
      <text x="32" y="211.5">200</text>
      <text x="32" y="185.25">300</text>
      <text x="32" y="159">400</text>
      <text x="32" y="132.75">500</text>
      <text x="32" y="106.5">600</text>
      <text x="32" y="80.25">700</text>
      <text x="32" y="54">800</text>
    </g>
    <line x1="40" y1="50" x2="40" y2="260" stroke="#cbd5e1" stroke-width="1" />
    <line x1="40" y1="260" x2="280" y2="260" stroke="#cbd5e1" stroke-width="1" />
    <rect x="65" y="147.125" width="45" height="112.875" fill="#1d4ed8" stroke="#0f172a" stroke-width="0.8" />
    <text x="87.5" y="200" font-size="11" font-weight="bold" fill="#ffffff" text-anchor="middle">430</text>
    <rect x="65" y="76.25" width="45" height="70.875" fill="url(#stripe-pattern)" stroke="#0f172a" stroke-width="0.8" />
    <text x="87.5" y="118" font-size="11" font-weight="bold" fill="#0f172a" text-anchor="middle">270</text>
    <text x="87.5" y="70" font-size="12" font-weight="bold" fill="#0f172a" text-anchor="middle">700</text>
    <rect x="137.5" y="162.875" width="45" height="97.125" fill="#ea580c" stroke="#0f172a" stroke-width="0.8" />
    <text x="160" y="156.875" font-size="12" font-weight="bold" fill="#0f172a" text-anchor="middle">370</text>
    <rect x="210" y="170.75" width="45" height="89.25" fill="#16a34a" stroke="#0f172a" stroke-width="0.8" />
    <text x="232.5" y="164.75" font-size="12" font-weight="bold" fill="#0f172a" text-anchor="middle">340</text>
    <g font-size="12" font-weight="bold" fill="#1e293b" text-anchor="middle">
      <text x="87.5" y="278">posix</text>
      <text x="160" y="278">hipfile</text>
      <text x="232.5" y="278">uGDS</text>
    </g>
  </g>
  <g transform="translate(600, 0)">
    <text x="150" y="25" font-size="15" font-weight="bold" fill="#1e293b" text-anchor="middle">CPU Utilization (%)</text>
    <rect x="25" y="38" width="80" height="60" fill="#ffffff" stroke="#cbd5e1" stroke-width="1" rx="2" />
    <text x="32" y="52" font-size="10" fill="#475569" font-weight="600">Backend</text>
    <rect x="32" y="58" width="12" height="8" fill="#1d4ed8" />
    <text x="48" y="65" font-size="10" fill="#334155">posix</text>
    <rect x="32" y="70" width="12" height="8" fill="#ea580c" />
    <text x="48" y="77" font-size="10" fill="#334155">hipfile</text>
    <rect x="32" y="82" width="12" height="8" fill="#16a34a" />
    <text x="48" y="89" font-size="10" fill="#334155">uGDS</text>
    <g stroke="#e2e8f0" stroke-width="1" stroke-dasharray="3,3">
      <line x1="40" y1="260" x2="280" y2="260" />
      <line x1="40" y1="225" x2="280" y2="225" />
      <line x1="40" y1="190" x2="280" y2="190" />
      <line x1="40" y1="155" x2="280" y2="155" />
      <line x1="40" y1="120" x2="280" y2="120" />
      <line x1="40" y1="85" x2="280" y2="85" />
      <line x1="40" y1="50" x2="280" y2="50" />
    </g>
    <g font-size="11" fill="#64748b" text-anchor="end">
      <text x="32" y="264">0</text>
      <text x="32" y="229">20</text>
      <text x="32" y="194">40</text>
      <text x="32" y="159">60</text>
      <text x="32" y="124">80</text>
      <text x="32" y="89">100</text>
      <text x="32" y="54">120</text>
    </g>
    <line x1="40" y1="50" x2="40" y2="260" stroke="#cbd5e1" stroke-width="1" />
    <line x1="40" y1="260" x2="280" y2="260" stroke="#cbd5e1" stroke-width="1" />
    <rect x="65" y="137.5" width="45" height="122.5" fill="#1d4ed8" stroke="#0f172a" stroke-width="0.8" />
    <text x="87.5" y="131.5" font-size="12" font-weight="bold" fill="#0f172a" text-anchor="middle">70%</text>
    <rect x="137.5" y="190" width="45" height="70" fill="#ea580c" stroke="#0f172a" stroke-width="0.8" />
    <text x="160" y="184" font-size="12" font-weight="bold" fill="#0f172a" text-anchor="middle">40%</text>
    <rect x="210" y="92" width="45" height="168" fill="#16a34a" stroke="#0f172a" stroke-width="0.8" />
    <text x="232.5" y="86" font-size="12" font-weight="bold" fill="#0f172a" text-anchor="middle">96%</text>
    <g font-size="12" font-weight="bold" fill="#1e293b" text-anchor="middle">
      <text x="87.5" y="278">posix</text>
      <text x="160" y="278">hipfile</text>
      <text x="232.5" y="278">uGDS</text>
    </g>
  </g>
</svg>

uGDS is indeed faster. The QLC SSD is the bottleneck here and uGDS is able to push to the max bandwidth from `hipFile`'s 2.6 GiB/s to 2.8GiB/s for GPU accelerated IO, and modern PCIe gen5 or gen6 SSD would benefit more. It consumes one CPU core with 96% usr% as expected and a nature from PMD, compared with ~40% from `hipFile`.

uGDS supports batch and async while `fio`'s new gpuaccel doesn't, let's skip here.

## 3.2 driver limiations
While analyzing uGDS, I noticed a few driver limitations. This research project still works well on most consumer and enterprise hardware though.

a. One limitation is 4GiB max VRAM ([code](https://github.com/ScaleX-IO/uGDS/blob/05bada9174eed5055cf886fac385ff191cf86f6a/drv/pci.c#L604)), and larger buf registration will hit OOM. Maybe this is like similar ReBAR check in GPU driver, and 4GiB buffer is sufficient for storage. One potential optimization is to re-use the buffer in uGDS for its sync engine.

b. Given above GPU memory limitation, I tried multi-threading with 4GiB pinned GPU buffer each and hit another OOM at 50% of GPU total VRAM. This turns out to be a bug in `amdkfd` driver introduced just last month ([commit](https://github.com/torvalds/linux/commit/fa7c99f04f6dd299388e9282812b14e95558ac8e)). AMD engineers caught the bug 2 weeks after and proposed the [fix](https://lists.freedesktop.org/archives/amd-gfx/2026-September/153665.html), such a rapid opensource development cycle!

c. NVMe SSD also has limitations. I'm using consumer hardware and it's working, but with limitations as shown in [uGDS #58](https://github.com/ScaleX-IO/uGDS/issues/58):

```
cap.MQES is 256
SQ size is 64B, at 256 queue depth SQ size is 16KiB, fits in 2MiB hugepage
CQ size is 16B, at 256 queue depth CQ size is 4KiB, a normal 4KiB page
```

In general, use enterprise hardware if possible and be careful with consumer alternatives. Cloud instance is something inbetween depend on the instance's capability.

## 4. what's next
My previous blog posts explored CPU and CXL memory expansion. Putting it all togeter, it's clear that data movement is changing rapidly across memory tiers in DRAM, CXL memory, and GPU storage. Since LMCache has already integrated GDS and uGDS, CXL support is partially done, let's see how they work together in real KV-cache workloads.

----
