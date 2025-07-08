---
date: '2025-07-08T08:00:00Z'
title: 'BeeGFS in MetaCentrum'
thumbnail: '/img/beegfs/beegfs.png'
description: "High-Speed Filesystem for Demanding Computations"
tags: ["Jak Krčmář", "Ivana Křenková", "CESNET", "MetaCentrum", "BeeGFS"]
colormode: true
draft: true
---



## What is BeeGFS?

**BeeGFS (Beyond Extensible Enterprise File System)** is a **parallel distributed filesystem** designed specifically for the needs of high-performance computing (HPC). It is used in computing clusters, scientific simulations, machine learning, genomics, and everywhere large datasets and fast parallel access are essential.
Developed with a strong focus on performance and designed for ease of use, simple installation, and management, BeeGFS is one of the leading parallel file systems that continues to grow and gain significant popularity in the community. BeeGFS has evolved into a world-wide valued filesystem offering maximum performance, scalability, high flexibility, and robustness.

BeeGFS combines multiple storage servers to provide a highly scalable shared network file system with striped file contents. This way, it allows users to overcome the tight performance limitations of single servers, single network interconnects, a limited number of hard drives, etc. In such a system, high throughput demands of large numbers of clients can easily be satisfied, but even a single client can benefit from the aggregated performance of all the storage servers in the system.

This is made possible by a separation of metadata and file contents. While storage servers are responsible for storing stripes of the actual contents of user files, metadata servers do the coordination of file placement and striping among the storage servers and inform the clients about certain file details when necessary. When accessing file contents, BeeGFS clients directly contact the storage servers to perform file I/O and communicate with multiple servers simultaneously, giving your applications truly parallel access to the file data. To keep the metadata access latency (e.g., directory lookups) at a minimum, BeeGFS also allows you to distribute the metadata across multiple servers so that each of the metadata servers stores a part of the global file system namespace.

On the server side, BeeGFS runs as normal user-space daemons without any special requirements on the operating system. The BeeGFS client is implemented as a Linux kernel module which provides a normal mount-point so that your applications can directly access the BeeGFS storage system and do not need to be modified to take advantage of BeeGFS. The module can be installed on all supported Linux kernels without the need for any patches.

{{< image src="img/beegfs/beegfs_architecture.png" class="rounded" wrapper="text-center w-40" >}}


### Key Advantages

- **HPC Technologies** – BeeGFS is built on highly efficient and scalable multithreaded core components with native RDMA support. File system nodes can serve RDMA (InfiniBand, Omni-Path, RoCE) and TCP/IP network connections at the same time and automatically switch to a redundant connection path in case any of them fails.
- **Distributed architecture** – One of the most fundamental concepts of BeeGFS is the strict avoidance of architectural bottlenecks. Striping file contents across multiple storage servers is only one part of this concept. Another important aspect is the distribution of file system metadata (e.g., directory information) across multiple metadata servers. Large systems and metadata intensive applications, in general, can greatly profit from the latter feature.
- **High Throughput** – BeeGFS is designed to deliver high bandwidth and low latency, making it suitable for data-intensive HPC workloads. Simple remote file systems like NFS do not only have serious performance problems in case of highly concurrent access, they can even corrupt data when multiple clients write to the same shared file, which is a typical use-case for cluster applications. BeeGFS was specifically designed with such use-cases in mind to deliver optimal robustness and performance in situations of high I/O load.
- **Scalable Metadata and Storage** – Metadata and storage servers can be scaled independently, allowing for flexible and efficient scaling as workloads grow.

### Limitations

- No native full POSIX distributed locking – some applications expecting strict file locking may behave incorrectly.


## When to Use BeeGFS in MetaCentrum

At MetaCenter, we’ve adopted BeeGFS (the BeeGFS General File System) to meet the increasing challenges of data-intensive research across various scientific disciplines. 
BeeGFS is available as a **temporary working directory** via the `scratch.shared` resource on cluster `bee.cerit-sc.cz`.

Here's when BeeGFS is the right choice for your jobs:

- **High-Performance Computing (HPC)** – Scientific computing often involves working with massive datasets and parallel workloads. BeeGFS is designed to efficiently handle large files and parallel input/output operations, which are critical for performance in HPC environments.

- **Machine Learning and AI** – Training machine learning models requires fast, concurrent access to large volumes of data. BeeGFS provides the high-throughput and low-latency access needed to accelerate the training process, helping researchers iterate and innovate faster.

- **Simulations, Rendering, Genomics, and Big Data Research** – BeeGFS excels in scenarios where datasets range from tens of terabytes to petabytes—such as 3D rendering, complex simulations, genomic sequencing, or any domain where data scale and speed matter. Its ability to distribute data across multiple storage servers ensures both scalability and performance.

- **University Compute Clusters** – Managing storage across academic clusters can be complex. BeeGFS simplifies administration with its modular design, while providing high availability and fault tolerance—key requirements in multi-user, shared environments.

- **Beyond NFS and SMB** – Traditional shared file systems like NFS or SMB often struggle with I/O bottlenecks, especially under heavy load. BeeGFS eliminates these limitations by distributing both metadata and data across multiple servers, enabling balanced and high-speed access across the entire cluster.


## Performance Results: BeeGFS vs. Local Scratch

We compared performance of BeeGFS (`scratch_beegfs`) and node-local scratch (`scratch_local`) using synthetic benchmarks with parallel I/O.

### 📊 Performance Comparison: BeeGFS vs. Local Scratch

| Test                 | BeeGFS (128 threads)   | scratch\_local (8 threads) | Comment                                       |
| -------------------- | ---------------------- | -------------------------- | --------------------------------------------- |
| **Sequential Write** | 22.8 GiB/s (24.5 GB/s) | 7946 MiB/s (8.3 GB/s)      | BeeGFS \~3× faster due to massive parallelism |
| **Sequential Read**  | 23.7 GiB/s (25.4 GB/s) | **43.8 GiB/s (47.0 GB/s)** | Local disk is faster in read – low overhead   |
| **Random Write**     | 486k IOPS, 1899 MiB/s  | 168k IOPS, 656 MiB/s       | BeeGFS delivers \~3× more IOPS                |
| **Random Read**      | 598k IOPS, 2335 MiB/s  | 238k IOPS, 930 MiB/s       | BeeGFS leads again in random access           |

### 📝 Interpretation

- **BeeGFS** BeeGFS excels at random I/O and sequential write performance
- **Local scratch** Local scratch excels in sequential read performance and total read volume on single node.
- **Use BeeGFS** for workloads with high random I/O and mixed-read/write patterns.
- **Use local scratch** for read-heavy sequential workloads (e.g., data analysis).


## Overview: Scratch Storage Types in MetaCentrum

| Scratch Type     | Speed                        | Persistence                     | Best For                                   |
| ---------------- | ---------------------------- | ------------------------------- | ------------------------------------------ |
| `scratch_shm`    | 🟢 Fastest (RAM)             | Lost after job ends             | Ultra-fast, short-lived temp data          |
| `scratch_local`  | 🟡 Very fast (local SSD/HDD) | Temp data survives node failure | Single-node jobs with high throughput      |
| `scratch_beegfs` | 🔵 Scalable parallel I/O     | Shared, cleaned after \~14 days | Multi-node, parallel jobs needing fast I/O |


Here's a bar chart comparing BeeGFS and Local Scratch across key performance metrics. As shown:

- BeeGFS dominates in random I/O (both read and write).
- Local Scratch excels in sequential read bandwidth.

{{< image src="img/beegfs/beegfs_chart.png" class="rounded" wrapper="text-center w-40" >}}

## Summary

If your computational work involves:

- Processing large datasets in parallel,
- High-throughput or high-IOPS requirements,
- Multi-node jobs in an HPC environment,

then `BeeGFS` is an excellent choice. With BeeGFS, you can achieve **multi-GB/s throughput and hundreds of thousands of IOPS**, helping your applications run faster and more efficiently.

More details: [MetaCentrum Scratch Documentation](https://docs.metacentrum.cz/en/docs/computing/resources/resources#scratch-directory)
