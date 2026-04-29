---
layout: post
title: "A first look at CXL 2.0 memory performance"
date: 2026-04-28
author: Tao Xu
---
Recently, I got two CXL 2.0 memory expander samples utilizing SCM media, neither DRAM nor NAND, in an E3.S form factor. CXL SCM is an interesting combination of two new hardware technologies. Below is an initial performance analysis comparing CXL SCM with system DRAM.

TL;DR
System level benchmarks show that the CXL SCM device is **3x** of system DRAM in load/access latency with **6.7GB/s** throughput. For real user workload, redis-benchmark on CXL memory shows around **half** throughput of DRAM baseline.

While SCM sits between DRAM and NAND, the CXL SCM device benefits from CXL low latency interface and is byte-addressible like system memory (64B cacheline). The low cost makes it a compelling alternative for large-scale caching. Additionally, the flexibility of runtime conversion between system memory and DAX storage is extra valuable. The focus in this blog is system memory scenario.

## 1. Platform

| HW | Config | Notes |
| :--- | :--- | :--- |
| CPU | Intel® Xeon® 6 | 288 cores |
| Memory | 12x 64GiB DDR5-4800 | 768GiB RAM |
| CXL Device | 2x CXL 2.0 (E3.S) | 400GiB capacity |
| OS | RHEL 9/10 Compatible | linux-6.6 |

## 2. BIOS & Topology
First, make sure BIOS has enabled CXL support. Refer to platform manuals, and an example configuration could be like:
1. Turn on/off Intel® Flat Memory Mode as here is on a Xeon6 Processor.
3. Turn off CXL related interleave options, let OS kernel manage CXL memory.
4. Explicitly enable CXL in PCIe lanes if needed, normally it's on by default.
5. PCIe Bifurcation, my machine is limited by the x2 backplane.
6. Disable PCIe ASPM support and CXL Header Bypass.
8. Set CXL Security Level to Fully Trusted.

After system boot, check `dmesg` for early CXL initialization logs like ACPI HMAT table info and PCIe registers. Below is an example from my setup.
```
[   19.420009] memory-tiers: the performance of DRAM node 1 mismatches that of the reference
               DRAM node 0.
[   19.420011]   performance of reference DRAM node 0 from ACPI HMAT:
[   19.420012]     read_latency: 91, write_latency: 91, read_bandwidth: 262144, write_bandwidth: 176128
[   19.420013]   performance of DRAM node 1 from ACPI HMAT:
[   19.420014]     read_latency: 1900, write_latency: 540, read_bandwidth: 73728, write_bandwidth: 67584
[   19.420016]   disable default DRAM node performance based abstract distance algorithm.
```

On `linux-6.6`, CXL memory expanders appear as distinct NUMA nodes without local CPU. 
```
# numactl -H
available: 3 nodes (0-2)
node 0 cpus: 0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31 32 33 34 35 36 37 38 39 40 41 42 43 44 45 46 47 48 49 50 51 52 53 54 55 56 57 58 59 60 61 62 63 64 65 66 67 68 69 70 71 72 73 74 75 76 77 78 79 80 81 82 83 84 85 86 87 88 89 90 91 92 93 94 95 96 97 98 99 100 101 102 103 104 105 106 107 108 109 110 111 112 113 114 115 116 117 118 119 120 121 122 123 124 125 126 127 128 129 130 131 132 133 134 135 136 137 138 139 140 141 142 143 144 145 146 147 148 149 150 151 152 153 154 155 156 157 158 159 160 161 162 163 164 165 166 167 168 169 170 171 172 173 174 175 176 177 178 179 180 181 182 183 184 185 186 187 188 189 190 191 192 193 194 195 196 197 198 199 200 201 202 203 204 205 206 207 208 209 210 211 212 213 214 215 216 217 218 219 220 221 222 223 224 225 226 227 228 229 230 231 232 233 234 235 236 237 238 239 240 241 242 243 244 245 246 247 248 249 250 251 252 253 254 255 256 257 258 259 260 261 262 263 264 265 266 267 268 269 270 271 272 273 274 275 276 277 278 279 280 281 282 283 284 285 286 287
node 0 size: 770884 MB
node 0 free: 767538 MB
node 1 cpus:
node 1 size: 201599 MB
node 1 free: 201413 MB
node 2 cpus:
node 2 size: 201572 MB
node 2 free: 201383 MB
node distances:
node   0   1   2 
  0:  10  14  14 
  1:  14  10  16 
  2:  14  16  10
```
## 3. PCIe Bandwidth
While the CXL devices support PCIe Gen5 x8, the current E3.S backplane in my machine is limited to x2 lanes. 
```
[   17.028167] pci 0000:35:00.0: 63.014 Gb/s available PCIe bandwidth, limited by 32.0 GT/s PCIe x2 link at 0000:34:02.0 (capable of 252.056 Gb/s with 32.0 GT/s PCIe x8 link)
[   17.060150] pci 0000:36:00.0: 63.014 Gb/s available PCIe bandwidth, limited by 32.0 GT/s PCIe x2 link at 0000:34:06.0 (capable of 252.056 Gb/s with 32.0 GT/s PCIe x8 link)
```
This bottleneck means we are testing the SCM media latency rather than the CXL throughput. Production CXL devices will likely feature 10x the capacity of these early 200GiB samples, meaning the actual bandwidth-per-GiB ratio should remain quite similar. All things considered, this is still a fair real user scenario to start benchmarking before I get a x8 backplane.
## 4. CXL device status
Check CXL device status with `ndctl` and `cxl-cli` tools via linux in-tree drivers.

`cxl-cli`

`cxl-cli` is the main tool for managing CXL device. Because Linux CXL drivers change rapidly and carry some complexity from legacy Optane support, it wasn't plug-and-play. After digging through driver logs, tweaking BIOS settings, and even submitting a bug fix ([#298](https://github.com/pmem/ndctl/issues/298)), I can see two memory regions for the two CXL devices:
```
# cxl version
78
# cxl list -Ru
[
  {
    "region":"region0",
    "resource":"0xc000000000",
    "size":"200.00 GiB (214.75 GB)",
    "type":"pmem",
    "interleave_ways":1,
    "interleave_granularity":256,
    "decode_state":"commit"
  },
  {
    "region":"region1",
    "resource":"0x28cc0000000",
    "size":"200.00 GiB (214.75 GB)",
    "type":"pmem",
    "interleave_ways":1,
    "interleave_granularity":256,
    "decode_state":"commit"
  }
]
```
`ndctl`

Similarly, `ndctl` is able to see the two regions passed up from CXL modules.
```
# ndctl version
78
# ndctl list -Ru
[
  {
    "dev":"region1",
    "size":"200.00 GiB (214.75 GB)",
    "align":"16.00 MiB (16.78 MB)",
    "available_size":"200.00 GiB (214.75 GB)",
    "max_available_extent":"200.00 GiB (214.75 GB)",
    "type":"pmem",
    "iset_id":"0x757364a175736407",
    "persistence_domain":"memory_controller"
  },
  {
    "dev":"region0",
    "size":"200.00 GiB (214.75 GB)",
    "align":"16.00 MiB (16.78 MB)",
    "available_size":"200.00 GiB (214.75 GB)",
    "max_available_extent":"200.00 GiB (214.75 GB)",
    "type":"pmem",
    "iset_id":"0x7573649775736402",
    "persistence_domain":"memory_controller"
  }
]
```
`daxctl`

While these CXL SCM devices are capable of operating as CXL PMem DAX device (`fsdax` or `devdax`), I skipped DAX here as initial focus is on the CXL system memory tier.
## 5. Benchmarks
Below are some standard benchmarking analysis.
### a. Single thread load latency - lmbench

| Hierarchy | Size | Sequential (Prefetch) | Random (No PF) |
| :--- | :--- | :--- | :--- |
| L1d Cache | 32KiB | 1.12ns | 1.12ns |
| L2 Cache | 4MiB | 7.41ns | 10.75ns |
| L3 Cache (LLC) | 216MiB | 23ns | 73ns |
| Local node (DRAM) | 768GiB | 67ns | 211ns |
| Remote node (CXL) | 200GiB | 296ns | 685ns |

As expected, access latency from the CXL SCM device is around `685ns`, 3x of local DRAM's `211ns`. Although the backplane is limited at PCIe gen5 x2 bandwidth, the main latency shall come from the SCM medium. And it's the end-to-end round-trip latency from CPU to CXL device medium, as seen by system software.

Since the system has one processor, the two CXL devices are symmetric and give same results. BIOS may have settings to interleave the two CXL devices into one NUMA node, but I haven't tried as it's just a first look.

Raw latency data plot:

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 900 520" width="80%" height="80%" style="font-family: system-ui, -apple-system, sans-serif;">
  <rect width="100%" height="100%" fill="#f8fafc" />
  <text x="90" y="30" font-size="20" font-weight="bold" fill="#1e293b">NUMA Memory Latency (lat_mem_rd)</text>
  <text x="90" y="50" font-size="14" fill="#64748b">Latency across nodes (Log X, Linear Y)</text>
  <g stroke="#cbd5e1" stroke-dasharray="4" stroke-width="1">
    <line x1="90" y1="440" x2="810" y2="440" />
    <line x1="90" y1="382.86" x2="810" y2="382.86" />
    <line x1="90" y1="325.72" x2="810" y2="325.72" />
    <line x1="90" y1="268.58" x2="810" y2="268.58" />
    <line x1="90" y1="211.44" x2="810" y2="211.44" />
    <line x1="90" y1="154.30" x2="810" y2="154.30" />
    <line x1="90" y1="97.16" x2="810" y2="97.16" />
    <line x1="90" y1="40.02" x2="810" y2="40.02" />
  </g>
  <g font-size="12" fill="#475569" text-anchor="end">
    <text x="80" y="444">0 ns</text>
    <text x="80" y="386.86">100 ns</text>
    <text x="80" y="329.72">200 ns</text>
    <text x="80" y="272.58">300 ns</text>
    <text x="80" y="215.44">400 ns</text>
    <text x="80" y="158.30">500 ns</text>
    <text x="80" y="101.16">600 ns</text>
    <text x="80" y="44.02">700 ns</text>
  </g>
  <g stroke="#e2e8f0" stroke-width="1">
    <line x1="120" y1="40" x2="120" y2="440" />
    <line x1="270" y1="40" x2="270" y2="440" />
    <line x1="420" y1="40" x2="420" y2="440" />
    <line x1="570" y1="40" x2="570" y2="440" />
    <line x1="720" y1="40" x2="720" y2="440" />
    <line x1="810" y1="40" x2="810" y2="440" />
  </g>
  <g stroke="#94a3b8" stroke-dasharray="5,5" stroke-width="1.5">
    <line x1="270" y1="72" x2="270" y2="440" />
    <line x1="480" y1="72" x2="480" y2="440" />
    <line x1="652.7" y1="72" x2="652.7" y2="440" />
  </g>
  <g font-size="11" font-weight="bold" fill="#475569" text-anchor="middle">
    <text x="270" y="64">L1d (32 KiB)</text>
    <text x="480" y="64">L2 (4 MiB)</text>
    <text x="652.7" y="64">L3 (216 MiB)</text>
  </g>
  <g font-size="12" fill="#475569" text-anchor="middle">
    <text x="120" y="460">1 KiB</text>
    <text x="270" y="460">32 KiB</text>
    <text x="420" y="460">1 MiB</text>
    <text x="570" y="460">32 MiB</text>
    <text x="720" y="460">1 GiB</text>
    <text x="810" y="460">8 GiB</text>
  </g>
  <polyline points="90,40 90,440 810,440" fill="none" stroke="#94a3b8" stroke-width="2" />
  <rect x="100" y="75" width="160" height="110" fill="white" stroke="#cbd5e1" rx="4" opacity="0.95" />
  <g font-size="12" fill="#1e293b">
    <line x1="115" y1="93" x2="145" y2="93" stroke="#2563eb" stroke-width="3" />
    <text x="155" y="97">node0 (DRAM)</text>
    <line x1="115" y1="118" x2="145" y2="118" stroke="#0891b2" stroke-width="3" />
    <text x="155" y="122">node0 noPF</text>
    <line x1="115" y1="143" x2="145" y2="143" stroke="#ea580c" stroke-width="3" />
    <text x="155" y="147">node1 (CXL)</text>
    <line x1="115" y1="168" x2="145" y2="168" stroke="#dc2626" stroke-width="3" />
    <text x="155" y="172">node1 noPF</text>
  </g>
  <g transform="translate(420, 440) scale(3, -0.5714)">
    <polyline fill="none" stroke="#2563eb" stroke-width="2.5" stroke-linejoin="round" vector-effect="non-scaling-stroke" points=" -110,1.112 -100,1.112 -90,1.112 -84.1,1.112 -80,1.114 -74.1,1.112 -70,1.116 -64.1,1.112 -60,1.115 -54.1,1.113 -50,1.114 -44.1,7.414 -40,6.826 -34.1,6.815 -30,6.799 -24.1,7.361 -20,7.404 -14.1,7.411 -10,7.415 -4.1,7.410 0,7.414 5.8,7.413 10,7.414 15.8,9.025 20,12.949 25.8,21.017 30,22.465 35.8,22.714 40,22.740 45.8,22.778 50,22.703 55.8,22.723 60,22.731 65.8,22.956 70,23.850 75.8,28.402 80,36.592 85.8,48.350 90,55.058 95.8,60.210 100,62.278 105.8,65.738 110,62.059 115.8,66.431 120,66.592 125.8,65.797 130,67.023 "/>
    <polyline fill="none" stroke="#0891b2" stroke-width="2.5" stroke-linejoin="round" vector-effect="non-scaling-stroke" points=" -110,1.112 -100,1.112 -90,1.112 -84.1,1.112 -80,1.112 -74.1,1.112 -70,1.112 -64.1,1.112 -60,1.112 -54.1,1.113 -50,1.114 -44.1,7.414 -40,7.414 -34.1,7.414 -30,7.415 -24.1,7.416 -20,10.750 -14.1,10.751 -10,10.751 -4.1,10.751 0,10.751 5.8,10.751 10,10.751 15.8,13.117 20,21.547 25.8,47.786 30,56.343 35.8,63.214 40,67.811 45.8,72.028 50,71.873 55.8,72.081 60,72.796 65.8,80.568 70,89.754 75.8,106.348 80,125.665 85.8,159.251 90,176.107 95.8,190.920 100,196.993 105.8,206.383 110,208.130 115.8,209.988 120,210.664 125.8,211.166 130,210.842 "/>
    <polyline fill="none" stroke="#ea580c" stroke-width="2.5" stroke-linejoin="round" vector-effect="non-scaling-stroke" points=" -110,1.112 -100,1.112 -90,1.112 -84.1,1.112 -80,1.112 -74.1,1.112 -70,1.115 -64.1,1.112 -60,1.116 -54.1,1.113 -50,1.114 -44.1,7.414 -40,6.829 -34.1,6.816 -30,6.796 -24.1,7.363 -20,7.402 -14.1,7.412 -10,7.414 -4.1,7.410 0,7.413 5.8,7.414 10,7.413 15.8,7.413 20,7.582 25.8,21.927 30,22.619 35.8,22.684 40,22.850 45.8,22.782 50,23.139 55.8,23.845 60,24.030 65.8,22.797 70,23.813 75.8,36.766 80,90.340 85.8,159.757 90,196.773 95.8,243.371 100,256.874 105.8,274.119 110,283.302 115.8,290.441 120,291.990 125.8,296.157 130,296.583 "/>
    <polyline fill="none" stroke="#dc2626" stroke-width="2.5" stroke-linejoin="round" vector-effect="non-scaling-stroke" points=" -110,1.112 -100,1.112 -90,1.112 -84.1,1.112 -80,1.112 -74.1,1.112 -70,1.112 -64.1,1.112 -60,1.112 -54.1,1.113 -50,1.114 -44.1,7.414 -40,7.414 -34.1,7.414 -30,7.414 -24.1,7.415 -20,10.751 -14.1,10.751 -10,10.750 -4.1,10.751 0,10.751 5.8,10.751 10,10.751 15.8,10.753 20,18.485 25.8,49.319 30,56.568 35.8,64.011 40,69.444 45.8,72.658 50,72.684 55.8,72.892 60,74.358 65.8,76.214 70,76.667 75.8,104.162 80,217.978 85.8,399.945 90,555.883 95.8,616.236 100,648.328 105.8,665.037 110,673.620 115.8,678.318 120,683.847 125.8,683.425 130,685.463 "/>
  </g>
</svg>

Refer to the cache hierarchy from `lscpu`:
```
Caches (sum of all):         
  L1d:                       9 MiB (288 instances)
  L1i:                       18 MiB (288 instances)
  L2:                        288 MiB (72 instances)
  L3:                        216 MiB (1 instance)
NUMA:                        
  NUMA node(s):              3
  NUMA node0 CPU(s):         0-287
  NUMA node1 CPU(s):         
  NUMA node2 CPU(s):
```
### b. Multi-threaded throughput - MLC
Run Intel MLC for more system level bandwidth and latency benchmarks:
 - Default run: disabled prefetch, `200MB` buffer size, etc.
 - With prefetch: `mlc -e`
 - With 128B cacheline: `mlc -l128`
 - With random buffer (no prefetch) and no SMT: `mlc -r -X`

Latency numbers are similar to previous results from `lmbench's lat_mem_rd`:

| idle latency | numa 0 (DRAM) | numa 1 (CXL) | numa 2 (CXL) |
| :--- | :--- | :--- | :--- |
| default | 143.5ns | 621ns | 620.7ns |
| -e (PF) | 12.2ns | 38.7ns | 38.7ns |
| -l128 (128B) | 153.4ns | 622.7ns | 622.6ns |
| -r -X | 156.5ns | 624.6ns | 624.5ns |

The theoretical bandwidth from a CXL-2.0 PCIe-gen5 32GT/s x2 is around 7.4GB/s, and here we get around 6.5GB/s. Max bandwidth may be limited by the x2 backplane, but the SCM medium is still the major constraint, not the interface.

| Max BW (base 10) | numa 0 (DRAM) | numa 1 (CXL) | numa 2 (CXL) |
| :--- | :--- | :--- | :--- |
| default | 400.8GB/s | 6.5GB/s | 6.1GB/s |
| -e (PF) | 400.5GB/s | 6.4GB/s | 6.1GB/s |
| -l128 (128B) | 404.4GB/s | 5.5GB/s | 5.6GB/s |
| -r -X | 400.6GB/s | 6.1GB/s | 6.1GB/s |

The CXL's NUMA nodes are CPU-less, so don't have c2c latency data as there's no PE.

| c2c latency | Local Socket L2->L2 HIT | HITM |
| :--- | :--- | :--- |
| default | 108.8ns | 109ns |
| -e (PF) | 8.6ns | 8.8ns |
| -l128 (128B) | 107.9ns | 108.1ns |
| -r -X | 109.9ns | 110.1ns |

Notably, MLC crashed when I tried using 1GB hugepages via libhugetlbfs. It's not surprising given the fast pace of CXL driver development, and we'll probably see better hugepage support in CXL memory soon.

### c. Application impact - Redis

To evaluate real-world impact, benchmark in-memory database `Redis`.

> Redis server v=7.2.7 sha=00000000:0 malloc=jemalloc-5.3.0 bits=64 build=5e1520dc02ce3c24

Bind `redis-server` to CXL node via `numactl --membind`, and run built-in `redis-benchmark` locally from node 0.

> numactl -C 144-175 -m 0 redis-benchmark -c 100 -d 1024 -t set,get,incr,hset,sadd --threads 8 -n 10000000 -P 100 -r 10000000

Edit `redis.config`, below is the config I use:
> tcp-backlog 4096
> save ""
> maxmemory 100gb
> maxmemory-policy noeviction
> io-threads 8
> io-threads-do-reads no
> shutdown-on-sigint nosave
> shutdown-on-sigterm nosave

Cleanup for each run, and check memory usage by `redis-cli`:
> redis-cli CONFIG SET save ""
> redis-cli FLUSHALL
> redis-cli -i 1 --stat

The SET/GET benchmarks below illustrate RPS throughput (Y-axis) relative to the active memory usage reported by `redis-cli --stat` (X-axis).
1. **Small datasets (20MB - 200MB)**: Surprisingly, bind to CXL memory shows better result. I guess it's due to the `jemalloc` caching allocator, combined with the fact that file maps are still utilizing DRAM (as verified via `/proc/PID/smaps`).
2. **Large datasets (2GB - 10GB)**: as memory usage increases, the heap workload became the dominant factor, and we see real impact from CXL memory. CXL throughput dropped to **half** of DRAM. This aligns with above findings of the **3x** raw CXL access latency compared to DRAM. Note this simulates the worse case where Redis is restricted solely to the CXL memory tier.

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 900 550" width="80%" height="80%" style="font-family: system-ui, -apple-system, sans-serif;">
  <rect height="100%" fill="#fff"/>
  <text x="390" y="25" fill="#1f2937" font-size="20" font-weight="600" text-anchor="middle">redis-benchmark: DRAM vs. CXL SCM Throughput</text>
  <text x="390" y="40" fill="#6b7280" font-size="14" font-weight="400" text-anchor="middle">(redis-benchmark -c 100 -d 1024 --threads 8 -n 10000000 -P 100)</text>
  <text x="390" y="535" fill="#4b5563" font-size="12" font-weight="500" text-anchor="middle">Dataset Size</text>
  <text x="-240" y="20" fill="#4b5563" font-size="12" font-weight="500" text-anchor="middle" transform="rotate(-90)">Throughput (RPS in K)</text>
  <g stroke="#e5e7eb">
    <path stroke="#9ca3af" d="M70 460h650"/>
    <text x="60" y="464" fill="#6b7280" stroke="none" font-size="11" text-anchor="end">0</text>
    <path d="M70 402.9h650"/>
    <text x="60" y="406.9" fill="#6b7280" stroke="none" font-size="11" text-anchor="end">200</text>
    <path d="M70 345.7h650"/>
    <text x="60" y="349.7" fill="#6b7280" stroke="none" font-size="11" text-anchor="end">400</text>
    <path d="M70 288.6h650"/>
    <text x="60" y="292.6" fill="#6b7280" stroke="none" font-size="11" text-anchor="end">600</text>
    <path d="M70 231.4h650"/>
    <text x="60" y="235.4" fill="#6b7280" stroke="none" font-size="11" text-anchor="end">800</text>
    <path d="M70 174.3h650"/>
    <text x="60" y="178.3" fill="#6b7280" stroke="none" font-size="11" text-anchor="end">1000</text>
    <path d="M70 117.1h650"/>
    <text x="60" y="121.1" fill="#6b7280" stroke="none" font-size="11" text-anchor="end">1200</text>
    <path d="M70 60h650"/>
    <text x="60" y="64" fill="#6b7280" stroke="none" font-size="11" text-anchor="end">1400</text>
  </g>
  <rect width="33.6" height="237.4" x="78.1" y="222.6" fill="#2563eb" rx="1"/>
  <rect width="33.6" height="247.7" x="114.7" y="212.3" fill="#dc2626" rx="1"/>
  <rect width="33.6" height="242.3" x="151.3" y="217.7" fill="#60a5fa" rx="1"/>
  <rect width="33.6" height="334.9" x="187.9" y="125.1" fill="#f87171" rx="1"/>
  <rect width="33.6" height="207.4" x="240.6" y="252.6" fill="#2563eb" rx="1"/>
  <rect width="33.6" height="211.1" x="277.2" y="248.9" fill="#dc2626" rx="1"/>
  <rect width="33.6" height="210.9" x="313.8" y="249.1" fill="#60a5fa" rx="1"/>
  <rect width="33.6" height="264.9" x="350.4" y="195.1" fill="#f87171" rx="1"/>
  <rect width="33.6" height="156" x="403.1" y="304" fill="#2563eb" rx="1"/>
  <rect width="33.6" height="84.9" x="439.7" y="375.1" fill="#dc2626" rx="1"/>
  <rect width="33.6" height="154" x="476.3" y="306" fill="#60a5fa" rx="1"/>
  <rect width="33.6" height="86.3" x="512.9" y="373.7" fill="#f87171" rx="1"/>
  <rect width="33.6" height="126.6" x="565.6" y="333.4" fill="#2563eb" rx="1"/>
  <rect width="33.6" height="60" x="602.2" y="400" fill="#dc2626" rx="1"/>
  <rect width="33.6" height="193.1" x="638.8" y="266.9" fill="#60a5fa" rx="1"/>
  <rect width="33.6" height="78" x="675.4" y="382" fill="#f87171" rx="1"/>
  <text x="94.9" y="216.6" fill="#333" font-size="11" text-anchor="middle">831</text>
  <text x="131.5" y="206.3" fill="#333" font-size="11" text-anchor="middle">867</text>
  <text x="168.1" y="211.7" fill="#333" font-size="11" text-anchor="middle">848</text>
  <text x="204.7" y="119.1" fill="#333" font-size="11" text-anchor="middle">1172</text>
  <text x="257.4" y="246.6" fill="#333" font-size="11" text-anchor="middle">726</text>
  <text x="294" y="242.9" fill="#333" font-size="11" text-anchor="middle">739</text>
  <text x="330.6" y="243.1" fill="#333" font-size="11" text-anchor="middle">738</text>
  <text x="367.2" y="189.1" fill="#333" font-size="11" text-anchor="middle">927</text>
  <text x="419.9" y="298" fill="#333" font-size="11" text-anchor="middle">546</text>
  <text x="456.5" y="369.1" fill="#333" font-size="11" text-anchor="middle">297</text>
  <text x="493.1" y="300" fill="#333" font-size="11" text-anchor="middle">539</text>
  <text x="529.7" y="367.7" fill="#333" font-size="11" text-anchor="middle">302</text>
  <text x="582.4" y="327.4" fill="#333" font-size="11" text-anchor="middle">443</text>
  <text x="619" y="394" fill="#333" font-size="11" text-anchor="middle">210</text>
  <text x="655.6" y="260.9" fill="#333" font-size="11" text-anchor="middle">676</text>
  <text x="692.2" y="376" fill="#333" font-size="11" text-anchor="middle">273</text>
  <g fill="#4b5563" font-size="12" font-weight="500" text-anchor="middle">
    <text x="133.1" y="480">20MB</text>
    <text x="133.1" y="495" fill="#6b7280" font-weight="400">(10K Keys)</text>
    <text x="295.6" y="480">200MB</text>
    <text x="295.6" y="495" fill="#6b7280" font-weight="400">(100K Keys)</text>
    <text x="458.1" y="480">2GB</text>
    <text x="458.1" y="495" fill="#6b7280" font-weight="400">(1M Keys)</text>
    <text x="620.6" y="480">10GB</text>
    <text x="620.6" y="495" fill="#6b7280" font-weight="400">(10M Keys)</text>
  </g>
  <g fill="#374151" font-size="12" transform="translate(740 60)">
    <rect width="16" height="4" fill="#2563eb" rx="2"/>
    <text x="24" y="6">SET_DRAM</text>
    <rect width="16" height="4" y="24" fill="#dc2626" rx="2"/>
    <text x="24" y="30">SET_CXL</text>
    <rect width="16" height="4" y="48" fill="#60a5fa" rx="2"/>
    <text x="24" y="54">GET_DRAM</text>
    <rect width="16" height="4" y="72" fill="#f87171" rx="2"/>
    <text x="24" y="78">GET_CXL</text>
  </g>
</svg>

Extend to compare `SET, GET, INCR, SADD, HSET` operations, using same key range of `-r 10M` and resulting Redis memory usage is now `30GiB`.
1. **SET/GET/INCR/HSET**: clearly `memory-bound` workload, binding Redis to CXL memory tier results in **less than half** of the throughput seen from DRAM.
2. **SADD**: more `compute-bound` and thus less impacted, CXL memory throughput is **74%** of DRAM baseline.

<svg xmlns="http://www.w3.org/2000/svg" width="80%" height="80%" style="font-family:system-ui,-apple-system,sans-serif" viewBox="0 0 900 520">
  <rect width="100%" height="100%" fill="#f8fafc"/>
  <text x="90" y="30" fill="#1e293b" font-size="20" font-weight="bold"> redis-benchmark: DRAM vs CXL SCM throughput (30 GiB memory)</text>
  <text x="90" y="50" fill="#64748b" font-size="14">redis-benchmark -c 100 -d 1024 --threads 8 -n 10000000 -P 100</text>
  <g stroke="#cbd5e1" stroke-dasharray="4">
    <path d="M90 440h720M90 382.9h720M90 325.7h720M90 268.6h720M90 211.4h720M90 154.3h720M90 97.2h720M90 40h720"/>
  </g>
  <g fill="#475569" font-size="12" text-anchor="end">
    <text x="80" y="444">0</text>
    <text x="80" y="386.9">200K</text>
    <text x="80" y="329.7">400K</text>
    <text x="80" y="272.6">600K</text>
    <text x="80" y="215.4">800K</text>
    <text x="80" y="158.3">1000K</text>
    <text x="80" y="101.2">1200K</text>
  </g>
  <g fill="#475569" font-size="14" font-weight="500" text-anchor="middle">
    <text x="190" y="465">SET</text>
    <text x="330" y="465">GET</text>
    <text x="470" y="465">INCR</text>
    <text x="610" y="465">SADD</text>
    <text x="750" y="465">HSET</text>
  </g>
  <rect width="120" height="60" x="100" y="75" fill="#fff" stroke="#cbd5e1" opacity=".9" rx="4"/>
  <g fill="#1e293b" font-size="12">
    <path stroke="#2563eb" stroke-width="12" d="M115 95h25"/>
    <text x="150" y="100">DRAM</text>
    <path stroke="#dc2626" stroke-width="12" d="M115 120h25"/>
    <text x="150" y="125">CXL</text>
  </g>
  <path fill="#2563eb" d="M160 307.4h30V440h-30z"/>
  <path fill="#dc2626" d="M190 377.7h30V440h-30z"/>
  <path fill="#2563eb" d="M300 236.3h30V440h-30z"/>
  <path fill="#dc2626" d="M330 360.8h30V440h-30z"/>
  <path fill="#2563eb" d="M440 168.8h30V440h-30z"/>
  <path fill="#dc2626" d="M470 317.7h30V440h-30z"/>
  <path fill="#2563eb" d="M580 104.8h30V440h-30z"/>
  <path fill="#dc2626" d="M610 192.6h30V440h-30z"/>
  <path fill="#2563eb" d="M720 310.6h30V440h-30z"/>
  <path fill="#dc2626" d="M750 375.1h30V440h-30z"/>
  <g fill="#1e293b" font-size="16" font-weight="500" text-anchor="middle">
    <text x="175" y="300">464</text>
    <text x="205" y="370">218</text>
    <text x="315" y="228">713</text>
    <text x="345" y="352">277</text>
    <text x="455" y="160">949</text>
    <text x="485" y="310">428</text>
    <text x="595" y="96">1173</text>
    <text x="625" y="184">866</text>
    <text x="735" y="302">453</text>
    <text x="765" y="367">227</text>
  </g>
</svg>

---

