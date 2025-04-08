---
date: '2025-04-08T11:00:00Z'
title: 'High Available Ceph S3'
thumbnail: '/img/ceph/ceph.png'
description: "High Availability and scalability of Ceph S3 with ExaBGP, HAProxy, and multiple RGW instances"
tags: ["Michal Strnad", "Jan Horníček", "CESNET", "CEPH", "S3", "HAProxy"]
colormode: true
draft: false
---

## High Availability and scalability of Ceph S3 with ExaBGP, HAProxy, and multiple RGW instances

**Building your own S3-compatible object storage with Ceph offers
great flexibility, but it also requires careful planning to ensure high
availability, scalability, and reliability. In this article, we'll take
a closer look at a practical architecture that uses ExaBGP, HAProxy, and
multiple RGW instances on each server to create a setup that can handle
growing demands while maintaining performance and uptime.**

### What is Ceph?

**Ceph** is an open-source, software-defined storage platform
designed for **scalability**, **resilience**, and
**flexibility**. Unlike traditional storage solutions such as SANs,
tape libraries, or monolithic disk arrays, Ceph doesn't rely on
expensive specialized hardware. Instead, it distributes data across
**many commodity servers**, each with its own set of **standard
hard drives or SSDs**.

This approach provides not only cost efficiency (it depends on the
chosen architecture and hardware), but also **true horizontal
scalability** --- you can grow capacity and performance simply by
adding more servers and disks. Ceph automatically handles replication,
failure recovery, and data distribution, making it well-suited for
cloud-native workloads, backup storage, and internal object storage
services.

### What is the S3 Protocol?

**S3** is a RESTful object storage protocol originally developed by
Amazon Web Services, now widely adopted as the industry standard for
cloud storage APIs. It allows clients to store and retrieve files
(objects) in buckets (top-level "directories") via simple HTTP-based
operations such as PUT, GET, DELETE, and LIST.

Thanks to its popularity and ecosystem support, many tools and
applications integrate with S3 out of the box. Ceph includes a component
called **Rados Gateway (RGW)**, which provides a fully
**S3-compatible API** on top of Ceph's distributed backend. This
makes it possible to create public or private, scalable S3 storage
clouds that working seamlessly with existing tools like the AWS CLI,
rclone, or backup software --- while retaining full control over your
infrastructure.

Now that we\'ve covered the basics of Ceph and its role as an
S3-compatible object storage system, let\'s dive into the main content
of this article.

### Architecture Overview

This architecture combines DNS round-robin, HAProxy, and ExaBGP to
provide a scalable, highly available, and resilient S3-compatible object
storage solution using Ceph. DNS round-robin distributes traffic across
multiple servers, while HAProxy manages local load balancing and
failover for RGW instances. ExaBGP enhances the setup by dynamically
managing routing based on server health, ensuring seamless traffic
distribution and failover without requiring DNS changes. This setup
provides efficient load distribution, fault tolerance, and scalability
with minimal downtime.

### DNS Round-Robin for load distribution

In order to access the S3 service the DNS resolution has to be done
first. In our case, the domain s3.example.cz is configured with multiple
A records, each pointing to the public IP address of a different server
handling the S3 protocol:

- s3.example.cz. IN A 192.0.2.101

- s3.example.cz. IN A 192.0.2.102

- s3.example.cz. IN A 192.0.2.103

When performing DNS resolution the clients obtain one of these IP
addresses (depending on resolver policy), which results in basic load
distribution across all available servers. In DNS round-robin, the DNS
resolver returns different IP addresses for each request, helping to
spread the traffic evenly across multiple servers.

While simple and effective for initial distribution, DNS round-robin
alone does not guarantee consistent failover or service awareness.
That's why it's only the first layer in a broader strategy focused on
high availability and SPOF elimination --- complemented by BGP-based
routing and local load balancing (we\'ll talk about that in a moment).

### Multiple RGWs Behind HAProxy

A single RGW (Rados Gateway) process has practical limitations in terms
of how many concurrent requests it can handle. Under high load, it may
become a bottleneck or even crash if overwhelmed by requests. To address
this, we run **multiple independent RGW instances** on each physical
node, each listening on its own port.

The HAProxy placed in front of these instances plays multiple key roles:

1.  **SSL termination** -- all incoming HTTPS traffic is securely
    terminated at HAProxy
2.  **Local load balancing** -- requests are distributed across all RGW
    processes on the same server, maximizing parallelism and resilience
3.  **Connection shaping & routing** -- based on the client IP address,
    User-Agent, or specific paths, HAProxy can direct traffic to
    dedicated RGW instances (e.g., reserved for specific tenants or
    services)
4.  **Traffic control** -- rate limiting or request shaping can be
    applied to prevent any single client or workload from overwhelming
    the system

This model allows each server to scale **horizontally** on its own: by
adding more RGW processes, it can serve more simultaneous connections,
improve throughput, and absorb traffic spikes more gracefully. If one
RGW crashes or becomes unhealthy, HAProxy can stop routing traffic to it
--- ensuring uninterrupted service.

Critically, this also enables **multi-tenancy** and advanced routing
rules, making it possible to dedicate certain backends to specific
applications or business units without physically isolating them.

### Traffic routing with ExaBGP

To ensure high availability and granular control over traffic
distribution, each physical server in the cluster runs an instance of
**ExaBGP**, a programmable BGP daemon. These ExaBGP daemons
establish **BGP sessions** with the nearest upstream routers or L3
switches. This enables servers to **advertise or withdraw** IP
addresses in real-time, based on their own health state and internal
logic.

Configuration of every server contains **all the public IP addresses**
assigned to the S3 domain (e.g. those listed in DNS A records for
*s3.example.cz*). However, to avoid duplicate traffic and imbalance, a
**weighted preference model** is applied across the cluster:

-   Server A advertises IP1 with the **highest preference**, and other
    IPs with progressively lower weights
-   Server B advertises IP2 with the highest weight, and lower
    preferences for others
-   Server C does the same for IP3, and so on\...

This setup ensures that incoming traffic is **evenly distributed**
across all nodes, while maintaining the ability of any node to serve any
IP address if needed. If a server fails or enters maintenance mode,
ExaBGP automatically withdraws its BGP announcements, and the routers
reroute the traffic to other healthy nodes --- preserving service
availability without changing DNS or client-side configuration.

If we put together all the described components, we can construct the
following diagram, which shows the entire path from the client to data
access in Ceph.

{{< image src="img/ceph/ceph_s3.png" mode="false" class="rounded-3 mt-3 w-64" >}}

### Summary

This architecture delivers high availability, resilience, and
performance without relying heavily on complex L2 failover or
centralized appliances. DNS round-robin distributes traffic across
multiple nodes, while ExaBGP ensures that only healthy nodes respond to
client requests. With HAProxy in place, traffic is in controlled manner
evenly distributed across multiple RGW instances, enabling horizontal
scaling and ensuring uninterrupted service in case of failures. This
solution combines DNS, HAProxy, and ExaBGP to provide a robust
multi-layered high-availability strategy allowing the system to scale
efficiently while maintaining reliability and minimizing service
disruptions.
