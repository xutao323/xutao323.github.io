---
layout: post
title: "CXL memory performance (part 2)"
date: 2026-05-19
author: Tao Xu
---

Following up on the previous post, ["A first look at CXL 2.0 memory performance"](https://xutao323.github.io/2026/04/28/a-first-look-at-cxl-perf.html), below is an additional analysis after upgrading the PCIe backplane from x2 to x4 lanes and applying Sub-NUMA Clustering (SNC).

#### TL;DR
 - **PCIe bandwidth**: upgrading from x2 to x4 lanes showed little performance gain, confirming that interface bandwidth is not a bottleneck here.
 - **NUMA per Die**: enabling NUMA per Die (Intel SNC or AMD NPS) significantly reduced DRAM access latency by splitting the processor socket into 2 local compute Dies.

*Note: both outcomes are in line with the expectations from the analysis in previous post.*

## 1. More lanes
The PCIe backplane was upgraded from x2 to x4 lanes for the E3.S slots. Both CXL memory expanders are now running at PCIe Gen5 x4 bandwidth, doubling of the previous configuration:
```
[   17.175302] pci 0000:35:00.0: 126.028 Gb/s available PCIe bandwidth, limited by 32.0 GT/s PCIe x4 link at 0000:34:02.0 (capable of 252.056 Gb/s with 32.0 GT/s PCIe x8 link)
[   17.207322] pci 0000:37:00.0: 126.028 Gb/s available PCIe bandwidth, limited by 32.0 GT/s PCIe x4 link at 0000:34:06.0 (capable of 252.056 Gb/s with 32.0 GT/s PCIe x8 link)
```

The ACPI HMAT table remains unchanged, as it's static BIOS info.
```
[   19.547157] memory-tiers: the performance of DRAM node 2 mismatches that of the reference
               DRAM node 0.
[   19.547158]   performance of reference DRAM node 0 from ACPI HMAT:
[   19.547159]     read_latency: 102, write_latency: 102, read_bandwidth: 262144, write_bandwidth: 176128
[   19.547161]   performance of DRAM node 2 from ACPI HMAT:
[   19.547162]     read_latency: 1900, write_latency: 540, read_bandwidth: 73728, write_bandwidth: 67584
[   19.547163]   disable default DRAM node performance based abstract distance algorithm.

```

## 2. Memory topology
Previously, system DRAM is in default UMA topology. I have since configured NUMA per Die to enable local memory access on the CPU's compute Die (by Intel SNC or AMD NPS setting).

Hierarchy from `lscpu`:
```
L1d cache:                               9 MiB (288 instances)
L1i cache:                               18 MiB (288 instances)
L2 cache:                                288 MiB (72 instances)
L3 cache:                                216 MiB (1 instance)
NUMA node(s):                            4
NUMA node0 CPU(s):                       0-143
NUMA node1 CPU(s):                       144-287
NUMA node2 CPU(s):                       
NUMA node3 CPU(s):                       
```

This configuration updates the topology into four nodes, where Nodes 0 and 1 paritition L3 cache and DRAM symmetrically, and Nodes 2 and 3 represent the two CPU-less CXL memory expanders.

Resulting memory topology:
```
# numactl -H
available: 4 nodes (0-3)
node 0 cpus: 0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31 32 33 34 35 36 37 38 39 40 41 42 43 44 45 46 47 48 49 50 51 52 53 54 55 56 57 58 59 60 61 62 63 64 65 66 67 68 69 70 71 72 73 74 75 76 77 78 79 80 81 82 83 84 85 86 87 88 89 90 91 92 93 94 95 96 97 98 99 100 101 102 103 104 105 106 107 108 109 110 111 112 113 114 115 116 117 118 119 120 121 122 123 124 125 126 127 128 129 130 131 132 133 134 135 136 137 138 139 140 141 142 143
node 0 size: 383849 MB
node 0 free: 374279 MB
node 1 cpus: 144 145 146 147 148 149 150 151 152 153 154 155 156 157 158 159 160 161 162 163 164 165 166 167 168 169 170 171 172 173 174 175 176 177 178 179 180 181 182 183 184 185 186 187 188 189 190 191 192 193 194 195 196 197 198 199 200 201 202 203 204 205 206 207 208 209 210 211 212 213 214 215 216 217 218 219 220 221 222 223 224 225 226 227 228 229 230 231 232 233 234 235 236 237 238 239 240 241 242 243 244 245 246 247 248 249 250 251 252 253 254 255 256 257 258 259 260 261 262 263 264 265 266 267 268 269 270 271 272 273 274 275 276 277 278 279 280 281 282 283 284 285 286 287
node 1 size: 387035 MB
node 1 free: 385483 MB
node 2 cpus:
node 2 size: 201599 MB
node 2 free: 201465 MB
node 3 cpus:
node 3 size: 201572 MB
node 3 free: 201426 MB
node distances:
node   0   1   2   3 
  0:  10  12  14  14 
  1:  12  10  14  14 
  2:  14  14  10  16 
  3:  14  14  16  10
```

## 3. Latency impact
### From `lmbench`:
SNC splits L3 cache and DRAM across local and remote compute dies, successfully reducing cycles and latency compared to the previous UMA configuration:
 - Local L3: latency dropped from 23ns to 17ns with prefetch, non-prefetch from 73ns to 60ns.
 - Local DRAM: latency from 67ns to 59ns with prefetch, non-prefetch from 211ns to 178ns.

The x4 backplane upgrade helped a bit, reducing access latency from 685ns to 663ns. This is expected because the CXL memory expanders I use are utilizing SCM medium which is the bottleneck (behind CXL device controller) instead of interface bandwidth.

| Hierarchy | Size | Local<br>prefetch | Remote<br>prefetch | CXL<br>prefetch | Local<br>no PF | Remote<br>no PF | CXL<br>no PF |
| :--- | :--- | :---: | :---: | :---: | :---: | :---: | :---: |
| L1d Cache | 32KiB | <-- | 1.12ns | --> | <-- | 1.12ns | --> |
| L2 Cache | 4MiB | <-- | 7.42ns | --> | <-- | 10.75ns | --> |
| L3 Cache (SNC split) | 216MiB | 17.3ns | 28ns | 24ns | 60ns | 90ns | 73ns |
| Memory | 10GiB | 58.9ns | 75ns | 301ns | 178ns | 227ns | 663ns |

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
  <rect x="100" y="75" width="175" height="155" fill="white" stroke="#cbd5e1" rx="4" opacity="0.95" />
  <g font-size="12" fill="#1e293b">
    <line x1="115" y1="93" x2="145" y2="93" stroke="#2563eb" stroke-width="3" />
    <text x="155" y="97">node0 (local)</text>    
    <line x1="115" y1="114" x2="145" y2="114" stroke="#0891b2" stroke-width="3" />
    <text x="155" y="118">node0 noPF</text>
    <line x1="115" y1="135" x2="145" y2="135" stroke="#16a34a" stroke-width="3" />
    <text x="155" y="139">node1 (remote)</text>
    <line x1="115" y1="156" x2="145" y2="156" stroke="#8fce00" stroke-width="3" />
    <text x="155" y="160">node1 noPF</text>
    <line x1="115" y1="177" x2="145" y2="177" stroke="#ea580c" stroke-width="3" />
    <text x="155" y="181">node2 (CXL)</text>
    <line x1="115" y1="198" x2="145" y2="198" stroke="#dc2626" stroke-width="3" />
    <text x="155" y="202">node2 noPF</text>
  </g>
  <g transform="translate(420, 440) scale(3, -0.5714)">
    <polyline fill="none" stroke="#2563eb" stroke-width="2.5" stroke-linejoin="round" vector-effect="non-scaling-stroke" 
      points="-110,1.112 -100,1.112 -90,1.112 -84.1,1.112 -80,1.113 -74.1,1.112 -70,1.114 -64.1,1.113 -60,1.114 -54.1,1.137 -50,1.313 -44.1,7.403 -40,6.710 -34.1,6.725 -30,6.804 -24.1,7.315 -20,7.422 -14.1,7.423 -10,7.423 -4.1,7.417 0,7.418 5.8,7.416 10,7.415 15.8,7.542 20,11.375 25.8,16.335 30,17.150 35.8,17.278 40,17.381 45.8,17.339 50,17.277 55.8,17.406 60,18.012 65.8,23.539 70,31.038 75.8,43.703 80,50.110 85.8,54.220 90,57.410 95.8,58.491 100,58.637 105.8,58.912 110,58.934 115.8,58.885 120,58.703 125.8,58.242 130,58.848" />
    <polyline fill="none" stroke="#0891b2" stroke-width="2.5" stroke-linejoin="round" vector-effect="non-scaling-stroke" 
      points="-110,1.112 -100,1.112 -90,1.112 -84.1,1.112 -80,1.112 -74.1,1.112 -70,1.112 -64.1,1.112 -60,1.112 -54.1,1.113 -50,1.114 -44.1,7.414 -40,7.414 -34.1,7.415 -30,7.415 -24.1,7.416 -20,10.752 -14.1,10.752 -10,10.752 -4.1,10.753 0,10.753 5.8,10.752 10,10.753 15.8,12.668 20,20.706 25.8,35.894 30,41.583 35.8,49.814 40,54.814 45.8,58.159 50,58.710 55.8,60.547 60,62.331 65.8,75.241 70,93.030 75.8,119.090 80,134.958 85.8,156.079 90,166.702 95.8,174.070 100,175.844 105.8,177.042 110,177.678 115.8,179.407 120,178.467 125.8,178.676 130,178.726" />
    <polyline fill="none" stroke="#16a34a" stroke-width="2.5" stroke-linejoin="round" vector-effect="non-scaling-stroke" 
      points="-110,1.112 -100,1.112 -90,1.112 -84.1,1.112 -80,1.114 -74.1,1.112 -70,1.115 -64.1,1.114 -60,1.114 -54.1,1.167 -50,1.253 -44.1,7.404 -40,6.700 -34.1,6.727 -30,6.736 -24.1,7.361 -20,7.451 -14.1,7.423 -10,7.423 -4.1,7.426 0,7.418 5.8,7.416 10,7.416 15.8,7.439 20,14.124 25.8,25.462 30,27.412 35.8,27.822 40,28.023 45.8,28.003 50,27.822 55.8,27.908 60,28.970 65.8,34.768 70,42.495 75.8,54.514 80,61.248 85.8,66.383 90,69.048 95.8,71.408 100,72.864 105.8,74.065 110,74.441 115.8,74.862 120,75.196 125.8,75.082 130,75.412" />
    <polyline fill="none" stroke="#8fce00" stroke-width="2.5" stroke-linejoin="round" vector-effect="non-scaling-stroke" 
      points="-110,1.112 -100,1.112 -90,1.112 -84.1,1.112 -80,1.112 -74.1,1.112 -70,1.112 -64.1,1.112 -60,1.112 -54.1,1.113 -50,1.114 -44.1,7.414 -40,7.414 -34.1,7.415 -30,7.415 -24.1,7.416 -20,10.752 -14.1,10.756 -10,10.757 -4.1,10.752 0,10.752 5.8,10.753 10,10.755 15.8,13.650 20,25.599 25.8,58.512 30,69.407 35.8,77.024 40,81.526 45.8,85.493 50,86.211 55.8,88.313 60,90.820 65.8,103.187 70,124.689 75.8,149.541 80,167.897 85.8,198.095 90,213.505 95.8,222.370 100,224.504 105.8,225.906 110,226.430 115.8,226.801 120,226.875 125.8,227.028 130,226.656" />
    <polyline fill="none" stroke="#ea580c" stroke-width="2.5" stroke-linejoin="round" vector-effect="non-scaling-stroke" 
      points="-110,1.112 -100,1.112 -90,1.112 -84.1,1.112 -80,1.113 -74.1,1.112 -70,1.115 -64.1,1.114 -60,1.114 -54.1,1.139 -50,1.316 -44.1,7.402 -40,6.710 -34.1,6.722 -30,6.807 -24.1,7.315 -20,7.421 -14.1,7.423 -10,7.423 -4.1,7.416 0,7.418 5.8,7.416 10,7.416 15.8,7.416 20,7.832 25.8,22.461 30,22.997 35.8,23.332 40,23.223 45.8,23.401 50,23.594 55.8,24.121 60,23.952 65.8,24.894 70,26.967 75.8,53.099 80,89.971 85.8,155.480 90,194.027 95.8,238.838 100,257.448 105.8,275.189 110,282.603 115.8,291.034 120,294.114 125.8,297.833 130,301.203" />
    <polyline fill="none" stroke="#dc2626" stroke-width="2.5" stroke-linejoin="round" vector-effect="non-scaling-stroke" 
      points="-110,1.112 -100,1.112 -90,1.112 -84.1,1.112 -80,1.112 -74.1,1.112 -70,1.112 -64.1,1.112 -60,1.112 -54.1,1.113 -50,1.114 -44.1,7.414 -40,7.414 -34.1,7.415 -30,7.415 -24.1,7.416 -20,10.752 -14.1,10.752 -10,10.752 -4.1,10.752 0,10.752 5.8,10.752 10,10.911 15.8,10.754 20,12.053 25.8,51.525 30,56.142 35.8,63.669 40,68.953 45.8,72.656 50,72.782 55.8,73.044 60,74.619 65.8,75.575 70,95.428 75.8,161.671 80,240.693 85.8,398.795 90,518.741 95.8,602.421 100,626.398 105.8,646.025 110,651.280 115.8,657.801 120,661.545 125.8,662.534 130,662.895" />
  </g>
</svg>

### From `mlc`:
Comparing with data from last time:
 - UMA vs SNC: the 143.5ns interleaved latency splits into local 128.7ns and remote 157.5ns.
 - x2 vs x4 CXL lanes: more bandwidth helped a bit from 621ns to 601ns.

| idle latency | numa0/local | numa1/remote | numa2/CXL | numa3/CXL |
| :--- | :--- | :--- | :--- | :--- |
| default | 128.7ns | 157.5ns | 601.4ns | 600.8ns |
| -e (PF) | 9.9ns | 11.1ns | 37ns | 36.8ns |
| -l128 (128B) | 140.6ns | 166.2ns | 602.1ns | 601.4ns |
| -r -X | 144.4ns | 171.2ns | 605.4ns | 603.5ns |

SNC partition significantly optimized core-to-core (c2c) cache latency from 109ns (previous UMA data) to 80ns:

| c2c latency | Local Socket L2->L2 HIT | HITM |
| :--- | :--- | :--- |
| default | 79.5ns | 79.7ns |
| -e (PF) | 6.5ns | 6.6ns |
| -l128 (128B) | 78.9ns | 79ns |
| -r -X | 80.9ns | 81ns |

## 4. Bandwidth impact
Comparing `mlc` results against previous post:
 - UMA vs SNC: the 400GB/s interleaved bandwidth splits into 204GB/s local and 158GB/s remote per die bandwidth.
 - x2 vs x4 CXL lanes: doubling interface bandwidth didn't help much, just from 6.5GB/s to 6.8GB/s.

| Max BW (base 10) | numa0/local | numa1/remote | numa2/CXL | numa3/CXL |
| :--- | :--- | :--- | :--- | :--- |
| default | 204GB/s | 158GB/s | 6.4GB/s | 6.4GB/s |
| -e (PF) | 204GB/s | 158GB/s | 6.4GB/s | 6.8GB/s |
| -l128 (128B) | 204GB/s | 132GB/s | 5.5GB/s | 5.5GB/s |
| -r -X | 204GB/s | 158GB/s | 6.4GB/s | 6.4GB/s |

## 5. Re-test redis
Observations:
 - **SNC**: partitioning L3 and DRAM results in remote redis performance roughly 10% lower than local.
 - **CXL**: Real-world application redis throughput remains around 50% of local DRAM performance across almost all test cases. One exception is the `SADD` operation, which scales to 80% of DRAM due to being strictly CPU/compute-bound.
 - **Scaling**: moving from x2 to x4 PCIe backplane delivered only a 1–5% benefit, in line with what we've seen previously from synthetic micro-benchmarks (lmbench and mlc).

<svg xmlns="http://www.w3.org/2000/svg" width="80%" height="80%" style="font-family:system-ui,-apple-system,sans-serif" viewBox="0 0 900 520">
  <rect width="100%" height="100%" fill="#f8fafc"/>
  <text x="90" y="30" fill="#1e293b" font-size="20" font-weight="bold">redis-benchmark: DRAM vs CXL Throughput (SNC split)</text>
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
  <rect width="120" height="110" x="100" y="75" fill="#fff" stroke="#cbd5e1" opacity=".9" rx="4"/>
  <g fill="#1e293b" font-size="12">
    <path stroke="#1f5a7a" stroke-width="12" d="M115 95h25"/>
    <text x="150" y="100">SNC0</text>
    <path stroke="#0ea5e9" stroke-width="12" d="M115 120h25"/>
    <text x="150" y="125">SNC1</text>
    <path stroke="#ea580c" stroke-width="12" d="M115 145h25"/>
    <text x="150" y="150">CXL1</text>
    <path stroke="#facc15" stroke-width="12" d="M115 170h25"/>
    <text x="150" y="175">CXL2</text>
  </g>
  <path fill="#1f5a7a" d="M146 310.3h22V440h-22z"/>
  <path fill="#0ea5e9" d="M168 321.4h22V440h-22z"/>
  <path fill="#ea580c" d="M190 376.9h22V440h-22z"/>
  <path fill="#facc15" d="M212 377.4h22V440h-22z"/>
  <path fill="#1f5a7a" d="M286 232.9h22V440h-22z"/>
  <path fill="#0ea5e9" d="M308 262.0h22V440h-22z"/>
  <path fill="#ea580c" d="M330 358.6h22V440h-22z"/>
  <path fill="#facc15" d="M352 358.6h22V440h-22z"/>
  <path fill="#1f5a7a" d="M426 168.9h22V440h-22z"/>
  <path fill="#0ea5e9" d="M448 202.9h22V440h-22z"/>
  <path fill="#ea580c" d="M470 314.9h22V440h-22z"/>
  <path fill="#facc15" d="M492 314.9h22V440h-22z"/>
  <path fill="#1f5a7a" d="M566 114.6h22V440h-22z"/>
  <path fill="#0ea5e9" d="M588 147.7h22V440h-22z"/>
  <path fill="#ea580c" d="M610 181.4h22V440h-22z"/>
  <path fill="#facc15" d="M632 187.1h22V440h-22z"/>
  <path fill="#1f5a7a" d="M706 310.6h22V440h-22z"/>
  <path fill="#0ea5e9" d="M728 322.6h22V440h-22z"/>
  <path fill="#ea580c" d="M750 375.1h22V440h-22z"/>
  <path fill="#facc15" d="M772 374.0h22V440h-22z"/>
  <g fill="#1e293b" font-size="11" font-weight="500" text-anchor="middle">
    <text x="157" y="304.3">454</text>
    <text x="179" y="315.4">415</text>
    <text x="201" y="370.9">221</text>
    <text x="223" y="371.4">219</text>
    <text x="297" y="226.9">725</text>
    <text x="319" y="256.0">623</text>
    <text x="341" y="352.6">285</text>
    <text x="363" y="352.6">285</text>
    <text x="437" y="162.9">949</text>
    <text x="459" y="196.9">830</text>
    <text x="481" y="308.9">438</text>
    <text x="503" y="308.9">438</text>
    <text x="577" y="108.6">1139</text>
    <text x="599" y="141.7">1023</text>
    <text x="621" y="175.4">905</text>
    <text x="643" y="181.1">885</text>
    <text x="717" y="304.6">453</text>
    <text x="739" y="316.6">411</text>
    <text x="761" y="369.1">227</text>
    <text x="783" y="368.0">231</text>
  </g>
</svg>

## 6. Future work
With the above preliminary analysis of the CXL 2.0 SCM device, more work can be done in the following areas:
 - **Storage DAX**: These CXL SCM devices are capable of operating as a CXL memory device, a CXL PMem DAX device, or a combination of both. This flexibility is a unique feature that maximizes system resource utilization across both memory and storage. For example, recent NVIDIA GTC introduced CMX™ (Context Memory Storage) as a "G3.5" tier sitting between G3 local SSDs and G4 shared object/file storage. Similarly, CXL can be viewed as a "G2.5" layer in this architecture.
 - **CXL 3.0 PMU**: Although we have captured the end-to-end latency for this CXL 2.0 device, a dedicated PMU is unfortunately only defined starting with the CXL 3.0 specification. However, we can still obtain a latency breakdown and distribution by leveraging the PCIe PMUs in the processor's I/O die.
 - **More workloads**: Evaluate performance across emerging memory-demanding workloads, such as MLPerf Storage v2.0 checkpointing and KV cache management, etc.

----
