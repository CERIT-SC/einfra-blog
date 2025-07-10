---
date: '2025-07-10T09:40:00Z'
title: 'Watts Up in your Cluster?'
thumbnail: '/img/scaphandre/scaphandre.png'
description: "Measuring Power Consumption per Process by Scaphandre"
tags: ["Josef Handl", "CERIT-SC", "Scaphandre", "RAPL", "Power Measurement"]
colormode: true
draft: true
---


# TODO

When operating computational clusters or batch systems for the academic community, one of the most important aspects is monitoring resource usage by users. Classic metrics such as CPU, memory and disk usage can be extended with energy consumption - not only per node using the Power Supplies or the Power Distribution Units equipped with a power measurement, but even per process running on individual nodes. 


## Scaphandre and Technical Background
[Scaphandre](https://github.com/hubblo-org/scaphandre) is an open-source tool designed to measure power consumption at the process level and export metric in several ways. The tool is designed for Linux and Windows systems and is distributed as a binary, or packaged as docker image. For Kubernetes users helm chart is available for an easy deployment.

First, let's understand the magic behind Scaphandre.

### RAPL Sensors
To measure energy consumption per process, hardware support is essential. Modern x86 CPUs from both Intel and AMD include integrated *RAPL* (*Running Average Power Limit*) sensors. RAPL provides estimated energy consumption data for several domains, such as the entire CPU package, block with cores, iGPU, DRAM etc. Availability of the particular RAPL domains can vary, depending on the CPU generation and vendor.

![rapl](/img/scaphandre/rapl.png)  
*RAPL domains. Taken from the [Scaphandre documentation](https://hubblo-org.github.io/scaphandre-documentation/explanations/rapl-domains.html)*

On Linux, the access to these sensors provides `powercap` subsystem (`/sys/class/powercap/`).
An example of the consumed energy by the first CPU socket:
```bash
> sudo cat /sys/class/powercap/intel-rapl:0/energy_uj
9758110032
```

These energy counters are computing in microjoules. While they offer only estimates rather than exact measurements, they are considered to be reliable enough for comparative analysis and usage-based reporting.

RAPL is essential, but on modern systems with multitasking with hundreds tasks, and even multiple CPU sockets, correctly assigning energy consumption to specific processes can be challenging.


### Jiffy
A *jiffy* in Linux represents the kernel's basic unit of time, defined as the interval between two timer interrupts. Its duration depends on the system configuration, typically 1ms (for 1000 Hz), 4ms (for 250 Hz), or 10ms (for 100 Hz), as set by the `CONFIG_HZ` in the kernel parameter. The system uses jiffies to track uptime, CPU utilization, process runtime, and for various scheduling tasks. The kernel maintains several jiffy counters to track the number of ticks since system boot, or tracking time consumed by individual processes. These values are found in `/proc/stat` and `/proc/[pid]/stat`, respectively.

![total-time-share](/img/scaphandre/total-time-share.png)  
*Simple visualization of the time intervals (jiffies) that system counts for individual processes and the whole machine. Taken from the [Scaphandre documentation](https://hubblo-org.github.io/scaphandre-documentation/explanations/how-scaph-computes-per-process-power-consumption.html)*


### From Jiffies to Joules

By periodic reading of the elapsed time counted in *jiffies* and the consumed energy from the `powercap` subsystem, it is possible to calculate the energy consumed by individual processes.


![power-and-share-of-usage](/img/scaphandre/power-and-share-of-usage.png)  
*The figure shows interleaved jiffies of individual processes that participates on the total power consumption. Taken from the [Scaphandre documentation](https://hubblo-org.github.io/scaphandre-documentation/explanations/how-scaph-computes-per-process-power-consumption.html)*

This is the whole technical magic behind the Scapahandre.

## Using the Scaphandre


### Processes in containers
In the container era, it would be beneficial to map the energy consumption of processes to their respective containers and also Kubernetes pods. Scaphandre can do that.

In containerized environments, processes run within *cgroups*, which isolate and manage resource usage. Scaphandre can connect to the Docker socket or Kubernetes API and map the individual processes to their respective containers, pods, namespaces and more.

### Exporters
Scaphandre provides several *exporters* - methods to provide the final metrics. It could be simple `stdout` but also `json`, `prometheus`, `qemu` and more. We prefer `prometheus` exporter.


## Visualization
To visualize the metrics, we use Grafana and PromQL to query the Prometheus.

We provide large Kubernetes clusters and its resources for the academic community. To directly visualize the processes and pods is not very helpful. Except for the fact that each pod within the same computational workflow can be spawned on different node, pods may act more like short lifetime *tasks* than persistent *services*. In this case, the useful aggregating entity is Kubernates *Namespace*.

![grafana-power-specific-namespace](/img/scaphandre/grafana-power-specific-namespace.png)  
*This chart shows 2 queries together to visualize the difference between power consumption of the individual Kubernetes Pods within specific Namespace and the total power consumption of the whole Namespace.*

In our case, the total consumption is the useful information. Used queries are:

A simple query to show all pods in specific namespace (metrics are in microwatts):
```
scaph_process_power_consumption_microwatts{kubernetes_pod_namespace="loslab-comp"} / 1e6
```
A query aggregating the pods consumption for the whole namespace (metrics are in microwatts):
```
quantile_over_time(0.9, 
    (sum by(kubernetes_pod_namespace) (
        scaph_process_power_consumption_microwatts{kubernetes_pod_namespace="loslab-comp"})
    )[5m:1m]
) / 1e6
```

![grafana-power-dashboard](/img/scaphandre/grafana-power-dashboard.png)  
*Final dashboard with two charts - first showing the power consumption per namespace for the whole Kubernetes cluster in watts, and the second showing pie chart with the amount of energy in watthours consumed by the individual namespaces within selected time range.*

The query of the first chart is essentaily the simillar like in the previous example. The query of the pie chart consist of computing an average over the selected time range multiplied by the time range:
```
(
    avg_over_time(
        (sum by (kubernetes_pod_namespace) (
            scaph_process_power_consumption_microwatts{kubernetes_pod_namespace!=""})
        )[$__range:1m]
    ) / 1e6
) * ($__range_s / 3600)
```


## Limitations

The RAPL sensors are limited only to CPUs and memory, not for the rest of components. If your cluster is equipped with a lot of drives or even GPUs, power consumption of the CPU may easily become insignificant.
The support for GPU measurement in Scaphandre seems to be [planned](https://github.com/hubblo-org/scaphandre/issues/24). For now you need to arrange the GPU measurement differently. For Nvidia GPUs, you can use simple tools like `nvidia-smi` or more advanced like [`NVIDIA DCGM`](https://github.com/NVIDIA/DCGM).

## Additional Notes
In addition to the basic process measument and container mapping, Scaphandre also provides a way to analyze processes within KVM/QEMU-based virtual machines. Thanks to the fact that the RAPL sensors are not easily accessible in virtual machines, you will also need to deploy a Scaphandre agent on the hypervisor for this kind of setup. In this case, you are looking for `qemu` exported in the Scaphandre.

During the writing of this article (July 2025), Scaphandre has several imperfections such as missing support for *cgroup v2* and insufficient handling of the overflowing energy counters. Therefore, we made a [fork](https://github.com/CERIT-SC/scaphandre) of the original repo that tries to improve this shortages.

