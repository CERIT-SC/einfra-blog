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

### Key Advantages

- **High performance** – scalable I/O throughput and IOPS.
- **Distributed architecture** – no bottlenecks like in traditional NFS setups.
- **Flexible deployment** – runs on standard hardware, supports RDMA, Infiniband, and Ethernet.
- **Scalability** – performance increases with added nodes.

### Limitations

- **No native full POSIX distributed locking** – some applications expecting strict file locking may behave incorrectly.

---

## When to Use BeeGFS in MetaCentrum

BeeGFS is ideal for demanding jobs that:

- Work with large files or huge numbers of small files.
- Use many threads/processes reading or writing in parallel.
- Run across multiple compute nodes.

---

## Using BeeGFS in MetaCentrum

BeeGFS is available in MetaCentrum as a **temporary working directory** via the `scratch_beegfs` resource.

In a batch job, you request it like this:

```bash
#PBS -l scratch_beegfs=100g
```

During job execution, the `$SCRATCHDIR` environment variable will point to your BeeGFS working directory:

```bash
cp input.dat $SCRATCHDIR/
cd $SCRATCHDIR
./run_simulation
cp output.dat $HOME/
```

> ⚠️ **Important:** Always clean up your scratch directory after the job ends:
>
> ```bash
> rm -rf $SCRATCHDIR/*
> ```
>
> If not cleared manually, the directory is deleted automatically after \~14 days (or earlier if disk space runs low).

---

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

- **BeeGFS** scales much better in **parallel I/O scenarios**, especially with many threads or MPI processes across nodes.
- **Local scratch** shines in **sequential read workloads** on a single node.
- Choose based on your job’s I/O pattern – for high parallelism, **BeeGFS is a clear win**.

---

## Overview: Scratch Storage Types in MetaCentrum

| Scratch Type     | Speed                        | Persistence                     | Best For                                   |
| ---------------- | ---------------------------- | ------------------------------- | ------------------------------------------ |
| `scratch_shm`    | 🟢 Fastest (RAM)             | Lost after job ends             | Ultra-fast, short-lived temp data          |
| `scratch_local`  | 🟡 Very fast (local SSD/HDD) | Temp data survives node failure | Single-node jobs with high throughput      |
| `scratch_beegfs` | 🔵 Scalable parallel I/O     | Shared, cleaned after \~14 days | Multi-node, parallel jobs needing fast I/O |

More details:\
👉 [MetaCentrum Scratch Documentation](https://docs.metacentrum.cz/en/docs/computing/resources/resources#scratch-directory)

---

## Summary

If your computational work involves:

- Processing large datasets in parallel,
- High-throughput or high-IOPS requirements,
- Multi-node jobs in an HPC environment,

then `scratch_beegfs` is an excellent choice. With BeeGFS, you can achieve **multi-GB/s throughput and hundreds of thousands of IOPS**, helping your applications run faster and more efficiently.

