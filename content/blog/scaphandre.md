---
date: '2025-07-10T09:40:00Z'
title: 'Watts Up in your Cluster?'
thumbnail: '/img/scaphandre/scaphandre.png'
description: "Measuring Power Consumption per Process by Scaphandre"
tags: ["Josef Handl", "CERIT-SC", "Scaphandre", "RAPL", "Power Measurement"]
colormode: true
draft: true
---

In recent years, environmental concerns—particularly those related to energy consumption—have become a major focus across most sectors. This includes the field of IT infrastructure, where growing awareness of the carbon footprint associated with computation is prompting a shift in how infrastructure usage is accounted. More organizations are now starting to include environmental impact as a metric alongside performance and cost when evaluating their systems.

As part of this shift, there is a need for more fine-grained monitoring of energy consumption. Traditional approaches—measuring power usage at the level of Power Distribution Units (PDUs) or individual power supplies—are no longer sufficient. To support more accurate and responsible reporting, power consumption must increasingly be measured at the level of individual processes running on each node.

## Scaphandre and Technical Background
[Scaphandre](https://github.com/hubblo-org/scaphandre) is an open-source tool designed to measure power consumption at the process level and export power metrics in several ways. The tool is designed for Linux and Windows systems and is distributed as a binary, or packaged as docker image. For Kubernetes users helm chart is available for an easy deployment.

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

These energy counters report consumption in microjoules per CPU socket. Although they provide estimates rather than exact measurements, they are generally considered reliable enough for comparative analysis and usage-based reporting.

RAPL is essential, but on modern systems with multitasking with hundreds tasks, and even multiple CPU sockets, correctly assigning energy consumption to specific processes can be challenging.


### Jiffy
A *jiffy* in Linux represents the kernel's basic unit of time, defined as the interval between two timer interrupts. Its duration depends on the system configuration, typically 1ms (for 1000 Hz), 4ms (for 250 Hz), or 10ms (for 100 Hz), as set by the `CONFIG_HZ` in the kernel parameter. The system uses jiffies to track uptime, CPU utilization, process runtime, and for various scheduling tasks. The kernel maintains several jiffy counters to track the number of ticks since system boot, or tracking time consumed by individual processes. These values are found in `/proc/stat` and `/proc/[pid]/stat`, respectively.

![total-time-share](/img/scaphandre/total-time-share.png)  
*Simple visualization of the time intervals (jiffies) that system counts for individual processes and the whole machine. Taken from the [Scaphandre documentation](https://hubblo-org.github.io/scaphandre-documentation/explanations/how-scaph-computes-per-process-power-consumption.html)*


### From Jiffies to Joules

By periodically sampling the elapsed time (measured in *jiffies*) and energy usage data from the `powercap` subsystem, it is possible to estimate the energy consumption of individual processes. The total energy consumed by a socket can be distributed among the processes running on it based on their relative CPU time (*jiffies*) during the sampling interval.

![power-and-share-of-usage](/img/scaphandre/power-and-share-of-usage.png)  
*The figure shows interleaved jiffies of individual processes that participates on the total power consumption. Taken from the [Scaphandre documentation](https://hubblo-org.github.io/scaphandre-documentation/explanations/how-scaph-computes-per-process-power-consumption.html)*

This is the whole technical magic behind the Scapahandre.

## Using the Scaphandre

### Processes in containers
In the container era, it would be beneficial to map the energy consumption of processes to their respective containers and also Kubernetes pods. Scaphandre can do that.

In containerized environments, processes run within *cgroups*, which isolate and manage resource usage. Scaphandre can connect to the Docker socket or Kubernetes API and map the individual processes to their respective containers, pods, namespaces and more.

### Exporters
Scaphandre provides several *exporters* -- methods to provide the final metrics. It could be simple `stdout` but also `json`, `prometheus`, `qemu` and more. We utilized `prometheus` exporter.

## Visualization
To visualize the metrics, we use Grafana and PromQL to query the Prometheus.

Since we provide large Kubernetes clusters and their resources to the academic community, directly visualizing individual processes or pods offers limited insight. This is partly because pods belonging to the same computational workflow may be distributed across different nodes, and often behave more like short-lived *tasks* than persistent *services*. In this context, the most meaningful level of aggregation is the Kubernetes *Namespace*, which groups related workloads under a common organizational unit.

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
*Final dashboard with two charts -- first showing the power consumption per namespace for the whole Kubernetes cluster in watts, and the second showing pie chart with the amount of energy in watthours consumed by the individual namespaces within selected time range.*

The query of the first chart is essentially the similar like in the previous example. The query of the pie chart consist of computing an average over the selected time range multiplied by the time range:
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

The RAPL sensors are limited only to CPUs and memory, not for the rest of components. If the cluster is equipped with a lot of drives or even GPUs, power consumption of the CPU may easily become insignificant.
The support for GPU measurement in Scaphandre seems to be [planned](https://github.com/hubblo-org/scaphandre/issues/24). For now, it is needed to arrange the GPU measurement differently. For NVIDIA GPUs, simple tools like `nvidia-smi` or more advanced like [`NVIDIA DCGM`](https://github.com/NVIDIA/DCGM) can be used.

## Additional Notes
In addition to basic process measurement and container mapping, Scaphandre also supports analyzing processes running inside KVM/QEMU-based virtual machines. Because RAPL sensors are not directly accessible from within virtual machines, it is necessary to deploy a Scaphandre agent on the hypervisor in such setups. When doing so, `qemu` processes are exported by Scaphandre to monitor VM resource usage.

During the writing of this article (July 2025), Scaphandre has several imperfections such as missing support for *cgroup v2* and insufficient handling of the overflowing energy counters. Therefore, we made a [fork](https://github.com/CERIT-SC/scaphandre) of the original repo that tries to improve this shortages.

