---
date: '2025-10-08T12:26:32Z'
title: 'TESP-API: Enabling Standardized Task Execution in e-Infrastructur'
thumbnail: '/img/tesp-api/pulsar.png'
author: "Boris Jurič"
description: "Enabling Standardized Task Execution in e-Infrastructure"
tags: ["Boris Jurič", "CERIT-SC", "CESNET", "TESP-API", "Galaxy"]
colormode: true
draft: false
---

# CESNET introduces TESP-API — a lightweight microservice implementing the GA4GH Task Execution Service (TES) standard, bridging scientific platforms with distributed compute resources


The ability to submit and manage computing tasks across different environments is essential for modern e-infrastructure. The **GA4GH Task Execution Service (TES)** defines a standard interface for doing exactly this, ensuring that diverse workflow systems can interoperate with shared computational backends.

To bring this standard into real use, **CESNET** has developed **TESP-API** — a standalone microservice that translates TES requests into executable tasks for **Pulsar**, a distributed job execution system widely used in the Galaxy Project ecosystem.  
The source code is available at [github.com/CESNET/tesp-api](https://github.com/CESNET/tesp-api)



{{< image src="/img/tesp-api/tesp_diagram.png" class="rounded" wrapper="text-center w-40" >}}


## How TESP Works

TESP-API implements the GA4GH TES specification and acts as a bridge between TES clients and computational backends.  
When a task is submitted, TESP generates the corresponding **Pulsar execution commands**, manages job staging, and delegates file transfers directly to the compute node.  
This **direct storage–worker–storage** approach avoids unnecessary data hops, improving performance and reliability.  
TESP currently supports **HTTP, FTP, and S3** transfer protocols.

## Deployment and Use

TESP can be deployed via Docker Compose in two configurations:
- **Self-contained mode** – runs both TESP and a Pulsar instance, suitable for testing or local development.  
- **Standalone mode** – connects to an existing Pulsar service, ideal for integration into production systems.

At **usegalaxy.cz**, the national Galaxy instance operated by **CESNET**, TESP-API is already used for selected workloads under production-like conditions.  
This demonstrates its ability to integrate distributed data storage, computing nodes, and user-facing platforms in a unified, standards-based workflow.

## Outlook

TESP-API shows how open standards such as GA4GH TES can be turned into **practical, deployable services** within national and international research e-infrastructures.  
By connecting high-level scientific environments like Galaxy to underlying compute and storage resources, TESP helps move toward **fully interoperable, production-ready distributed computing**.

---

**TESP-API** is an evolving component of CESNET’s infrastructure toolkit — bridging research workflows, compute resources, and data services through open, standardized interfaces.



