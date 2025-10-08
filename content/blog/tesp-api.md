---
date: '2025-10-08T12:26:32Z'
title: 'Galaxy TESP-API'
thumbnail: '/img/tesp-api/pulsar.png'
author: "Boris Jurič"
description: "Bridging Galaxy, Pulsar, and the GA4GH TES Standard"
tags: ["Boris Jurič", "CERIT-SC", "TESP-API", "Galaxy"]
colormode: true
draft: true
---

# TESP-API: Bridging Galaxy, Pulsar, and the GA4GH TES Standard

*Introducing TESP — a microservice connecting Galaxy and Pulsar through the GA4GH Task Execution Service (TES) interface, enabling standardized and efficient job execution across distributed environments.*

The Galaxy Project thrives on interoperability — connecting tools, data, and compute resources seamlessly. To extend this philosophy to broader research infrastructures, we’ve been developing TESP-API, a service that implements the GA4GH Task Execution Service (TES) standard and acts as a bridge between Galaxy, Pulsar, and the wider TES ecosystem.

My name is Boris Jurič, and I’ve been working on TESP-API at CESNET as part of our ongoing effort to enhance Galaxy’s integration with national and international compute infrastructures. The project is available at [Galaxy TESP-API repository https://github.com/CESNET/tesp-api] (https://github.com/CESNET/tesp-api).

{{< image src="/img/tesp-api/tesp_diagram.png" class="rounded" wrapper="text-center w-40" >}}

## What is TESP-API?

The TESP API is a RESTful interface specification that enables Galaxy to delegate tool execution to external services while maintaining consistent job management capabilities. This new API builds upon lessons learned from previous integration approaches and provides a more robust framework for connecting Galaxy with specialized computational resources.

TESP is a lightweight microservice that translates TES-compliant job submissions into Pulsar tasks. It exposes a REST API following the GA4GH TES specification, allowing Galaxy and other TES clients to submit jobs to remote compute resources in a standardized way.

A key feature of TESP is direct data staging — input and output files move end-to-end between storage and compute nodes, bypassing intermediaries like Galaxy, TESP, or Pulsar itself. This design minimizes data transfers, improves performance, and aligns with Galaxy’s distributed compute model. TESP currently supports HTTP, FTP, and S3 data transfers.



## Deployment and Integration

TESP can be deployed via Docker Compose in two modes:

 - Embedded mode – runs both TESP and a Pulsar instance, suitable for local testing and development.
 - Standalone mode – connects to an existing Pulsar service, ideal for production or hybrid setups.

At usegalaxy.cz, operated by CESNET, TESP is already running in proof-of-concept conditions. Galaxy submits TES jobs to TESP, which manages their execution through Pulsar. Although it currently handles selected workloads, TESP is being actively optimized toward full production integration.


## Benefits for the Community

The TESP API enables:
- Easier integration of specialized computational resources
- Better resource utilization across distributed infrastructures
- Simplified development of tool execution backends
- Improved scalability for large-scale analyses



## Outlook

TESP-API is a practical bridge that enables another set of real scientific workloads on Galaxy–Pulsar infrastructures. It helps Galaxy to be a more connected and interoperable platform — one step closer to a unified compute ecosystem across research infrastructures. You can follow the project or contribute at [Galaxy TESP-API repository] (https://github.com/CESNET/tesp-api).


