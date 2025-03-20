---
date: '2025-03-20T14:00:00Z'
title: 'JupyterHub on Demand'
thumbnail: '/img/jupyterhub/thumbnail.png'
description: "Running customized instances of JupyterHub"
tags: ["Lukáš Hejtmánek", "CERIT-SC", "JupyterHub", "Kubernetes", "Customization"]
colormode: true
draft: true
---

# Here Comes the Hotstepper 

Jupyter Notebook is a widely recognized and well-established graphical interface primarily used for programming in Python, although its original intent was to support a combination of Julia, Python, and R (hence the name *Jupyter*). Its popularity stems largely from its accessibility—users can easily run it via a web browser, which appears to be the only prerequisite beyond basic infrastructure. For multi-user environments, **JupyterHub** comes into play as a customizable web interface that manages the underlying infrastructure required to run Jupyter Notebooks. JupyterHub provides essential features such as user authentication, resource management, and oversight of notebook lifecycles, making it an ideal solution for collaborative or shared computing environments.

The example below demonstrates **Jupyter Notebook** running in split-screen mode. On the left side, the basic launcher interface is visible, while the right side displays a notebook containing simple cells with Python code and its corresponding output.

![jupyter-notebook](/img/jupyterhub/notebook1.png)

## JupyterHub in Infrastructure – Or Why Should I Care?

{{< table wrap=true >}}
| I shoud if                                                      | I should not if                                                  |
|-----------------------------------------------------------------|------------------------------------------------------------------|
| * I need hardware I don’t currently have, e.g., NVIDIA H100 GPU | * I already have a powerful home supercomputer                   |
| * I need access to shared storages                              | * I keep everything on a USB flash drive                         |
| * I want to run my code in a managed environment                | * I prefer managing everything myself, like a multitasking ninja |
| * I want to share and collaborate on my code and data           | * I always work alone and don’t need collaboration               |
{{< /table >}}
