---
date: '2025-03-20T14:00:00Z'
title: 'JupyterHub on Demand'
thumbnail: '/img/jupyterhub/thumbnail.png'
description: "Running customized instances of JupyterHub"
tags: ["Lukáš Hejtmánek", "CERIT-SC", "JupyterHub", "Kubernetes", "Customization"]
colormode: true
draft: true
---

# Jupyter

Jupyter Notebook is a widely recognized and well-established graphical interface primarily used for programming in Python, although its original intent was to support a combination of Julia, Python, and R (hence the name *Jupyter*). Its popularity stems largely from its accessibility—users can easily run it via a web browser, which appears to be the only prerequisite beyond basic infrastructure. For multi-user environments, **JupyterHub** comes into play as a customizable web interface that manages the underlying infrastructure required to run Jupyter Notebooks. JupyterHub provides essential features such as user authentication, resource management, and oversight of notebook lifecycles, making it an ideal solution for collaborative or shared computing environments.

The example below demonstrates **Jupyter Notebook** running in split-screen mode. On the left side, the basic launcher interface is visible, while the right side displays a notebook containing simple cells with Python code and its corresponding output.

![jupyter-notebook](/img/jupyterhub/notebook1.png)

---

## JupyterHub in Infrastructure – Or Why Should I Care?

{{< table wrap=true >}}
| I shoud if:                                                                     | I should not if:                                                                 |
|---------------------------------------------------------------------------------|----------------------------------------------------------------------------------|
| <ul><li>I prefer a modern UI for computing.</li><li>I require hardware I don’t currently own, such as an NVIDIA H100 GPU.</li><li>I need access to shared storages.</li><li>I want to run my code in a managed environment.</li><li>I want to share and collaborate on my code and data.</li></ul> | <ul><li>I enjoy working with Bash scripting.</li><li>I already have a powerful home supercomputer.</li><li>I keep everything on a USB flash drive.</li><li>I prefer managing everything myself, like a multitasking ninja</li><li>I always work alone and don’t need collaboration</li></ul>                   |
{{< /table >}}

### Courses

Have you ever encountered a course—such as one for RStudio—that starts with a long list of pre-flight checks?

- Install R and RStudio on your computer, often involving multiple steps.
  - You may need to resolve hardware architecture issues (e.g., x86_64, ARM64, or Apple Silicon).
  - Check if your computer (laptop) has enough disk space and memory to run everything smoothly.
- Download a script and open it in RStudio.
- Install the required packages.
  - Handle dependency issues manually, since R does not automatically install necessary system and development libraries.
  - Some R packages take a long time to install.
- Remember where everything is installed, so you can run RStudio when the course begins.

By packaging such a course into a **JupyterHub instance**, students only need a functional web browser. Everything else—software, dependencies, and environment setup—can be preconfigured by the course maintainer. This setup is also scalable, ensuring a smooth experience regardless of the number of participants.

# Here Comes the Hotstepper

We provide **managed JupyterHub instances** running in **Kubernetes** on **CERIT-SC**. Our primary instance is available at [https://hub.cloud.e-infra.cz](https://hub.cloud.e-infra.cz), alongside several customized instances accessible via custom URLs.  

Each instance features **user access control** via **groups and SSO**. Depending on the instance, various configuration options are available. However, not all options are selectable in every instance—some use predefined defaults.  

## Features and Options  

- **Notebook Instance Limits**  
  - Some instances allow only **one Jupyter Notebook per user**.  
  - Others permit **multiple concurrent Notebooks per user**.  

- **Available Docker Images**  
  - Some instances offer a **fixed** Docker image for Jupyter Notebooks.  
  - Others provide a **set of images** users can choose from.  
  - In some cases, users can specify a **custom Docker image** for their Notebook.  

- **Storage Options**  
  - Persistent home directory for the Notebook.  
  - Option to delete the existing home directory when starting a new session.  
  - Ability to attach external storage, such as **MetaCentrum storage** and **Project directories**.  

- **Resource Allocation**  
  - Selectable **CPU count**.  
  - Configurable **memory allocation**.  
  - Choice of **GPU type** (if available).  

- **Usage Statistics**  
  - Number of **available GPUs**, as they are a scarce resource.  
  - **Resource consumption metrics** in **CZK** over the past **90 days**.  
