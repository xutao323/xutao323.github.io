layout: post
title: "TLS acceleration with new CPU"
date: 2021-01-06
author: Tao Xu (xutao323)

All top cloud service providers (CSP) have announced 3rd gen Intel® Xeon® CPU (code name Ice Lake-SP) a few months ago. It came with a built-in feature [Crypto-NI](https://www.intel.com/content/dam/develop/external/us/en/documents/Nginx-HTTPs-with-Crypto-NI-Tuning-Guide-on-3rd-Generation-Intel-Xeon-Scalable-Processors.pdf) to accelerate crypto workloads with new AVX512 instructions. Let's use bcc/bpftrace tools to analyze the use case in TLS acceleration and see how it works.

## 1. Background

Crypto workload is very CPU intensive especially public-key RSA (Elliptic-Curve is better). Hardware vendors developed many solutions to offload or accelerate crypto tasks. For example:

- [QAT](https://01.org/intel-quickassist-technology) offload device as an add-in PCIe card or integrated in chipset.
- [ARMv8](https://developer.arm.com/documentation/100619/0401/Functional-description/About-the-Cryptographic-Extension) Cryptography Extensions.
- [KAE](https://support.huaweicloud.com/intl/en-us/devg-kunpengaccel/kunpengaccel_16_0002.html) CPU on-die PCIe accelerator.
- FPGA solutions for cloud-based hardware security module (HSM).

Some application may use OpenSSL speed command to run crypto test as a quick method to measure or saturate CPU. So crypto is a standard and heavy CPU workload. A typical use case is SSL/TLS handshake in HTTPS web server, the crypto overhead can be observed by bcc/bpftrace tools, we'll describe below.

## 2. CPU hardware

Ice Lake added new AVX512 instructions (plus SHA extension or SHA-NI) for [crypto acceleration](https://www.intel.com/content/www/us/en/architecture-and-technology/crypto-acceleration-in-xeon-scalable-processors-wp.html). Normally application software needs to change to use SIMD intrinsics. Or use the [multi-buffer](https://github.com/intel/ipp-crypto/blob/develop/sources/ippcp/crypto_mb/Readme.md) lib which provides batch submission of multiple requests and parallel async processing based on new instruction set. 

![image 1](/20220106/image1.png)

To best knowledge at the time of writing, next gen CPU (code name Sapphire Rapids or SPR) is planned with Crypto-NI to succeed Ice Lake, so we could expect this built-in crypto acceleration on future Xeon CPU. Furthermore, SPR would provide several types of on-die [accelerators](https://www.anandtech.com/show/16921/intel-sapphire-rapids-nextgen-xeon-scalable-gets-a-tiling-upgrade) including new QAT.

The funny thing is, I've also tried on my laptop (using Ubuntu WSL) with Tiger Lake CPU that succeeded Ice Lake/ICL in client market, and it works the same as server market Ice Lake/ICX. I guess it's because they have same microarchitecture features (VAES, GFNI, IFMA, VPCLMULQDQ) though not highlighted in product specification.

## 3. Software stacks

Ice Lake crypto acceleration is using same stack as QAT (qat_hw), and called "multi-buffer" (qat_sw) in [QAT_Engine](https://github.com/intel/QAT_Engine). And this extends the usage from dedicated hardware to general available CPU which is great. They also share same use case as already supported in OpenSSL (BoringSSL or BabaSSL), Nginx (Tengine), DPDK Cryptodev, [k8s ingress](https://kubernetes.io/blog/2019/04/24/hardware-accelerated-ssl/tls-termination-in-ingress-controllers-using-kubernetes-device-plugins-and-runtimeclass/), [Istio Envoy](https://01.org/kubernetes/solutions/QAT-envoy-solution), ZFS ([QZFS](https://www.usenix.org/system/files/atc19-hu.pdf)), CDN, etc.

BTW, similar optimization was added in Linux Kernel crypto as well. The [patch](https://patchwork.kernel.org/project/linux-crypto/list/?series=420405&state=*) is in RFC for some time, while kernel crypto algorithm is already using [AVX2](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=5b2efa2bb865eb784e06987c7ce98c3c835b495b). Per [official document](https://www.intel.com/content/www/us/en/architecture-and-technology/crypto-acceleration-in-xeon-scalable-processors-wp.html), the Crypto-NI AVX512 is in light power level on Ice Lake that improved the downclocking problem. Anyway, here it's in userland for CPU intensive crypto workload in OpenSSL and TLS handshake.

Ice Lake CPU is officially supported in CentOS/RHEL 8.2 or some [Linux](https://github.com/alibaba/cloud-kernel/releases/tag/ck-release-17) developed by CSP. Thanks to previous offload solutions, system default OpenSSL 1.1.1 already support async SSL and engines without changing OpenSSL binary. This is very helpful as changing system OpenSSL has big compatibility [issue](https://access.redhat.com/solutions/2728111).

![image 2](/20220106/image2.png)

Above shows 3 approaches for HTTPS TLS handshake which will be analyzed below. General CPU path is most common with no special hardware help, it can still benefit from more powerful CPU though. Ice Lake acceleration path is the approach described above using new CPU instructions. QAT offload path is the original TLS offload approach using dedicated hardware.

## 4. Scenario setup

Refer to the official [Crypto-NI](https://www.intel.com/content/dam/develop/external/us/en/documents/Nginx-HTTPs-with-Crypto-NI-Tuning-Guide-on-3rd-Generation-Intel-Xeon-Scalable-Processors.pdf) doc or [this blog](https://openanolis.cn/sig/crypto/doc/390714951012679780) (I posted it) in Chinese. Basically, it's setting up a HTTPS server with Nginx (or Tengine) using Ice Lake acceleration approach described above for TLS acceleration.

Below is using a bear metal instance from one CSP (VM is the same as it's just using new CPU instructions). Test TLS to localhost with no network overhead. It's also tried with two instances to simulate the client-server by using private network, result is same as localhost, only need more client machines when using more CPU for HTTPS server. Let's use localhost to simplify the scenario as a clean reference.

## 5. bcc/bpftrace analysis

### a. general CPU approach

### b. Ice Lake acceleration approach

### c. QAT offload approach

## 6. Performance results

### a. OpenSSL

### b. Nginx/Tengine

## Summary

