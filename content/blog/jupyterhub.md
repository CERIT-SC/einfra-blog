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

Jupyter Notebook is a widely recognized and well-established graphical interface primarily used for programming in Python, although its original intent was to support a combination of Julia, Python, and R (hence the name *Jupyter*). Its popularity stems largely from its accessibility—users can easily run it via a web browser, which appears to be the only prerequisite beyond basic infrastructure. For multi-user environments, **JupyterHub** comes into play as a customizable web interface, providing access to the underlying infrastructure needed to run Jupyter Notebooks for multiple users. It offers essential features such as user authentication, resource management, and oversight of notebook lifecycles, making it an ideal solution for collaborative or shared computing environments.

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

## Special Features  

The following special features are available in some or all of the JupyterHub instances we manage.  

- **Persistent Images**
  - We **do not remove old Docker images** and retain previous versions, ensuring users have a **reproducible environment**.  

- **Non-Jupyter Notebooks**
  JupyterHub allows integration with additional software tools, enabling users to work with:  

  - **RStudio**  
    - We offer multiple **RStudio** images within JupyterHub, allowing users to launch RStudio instead of a Jupyter Notebook.  

  - **MATLAB**  
    - MathWorks provides a proxy that integrates **MATLAB** into JupyterHub, enabling users to run a full graphical MATLAB session in a web browser.  

  - **Classic Linux Desktop**  
    - Using [noVNC](https://novnc.com/), it is possible to run an entire **remote Linux desktop** within JupyterHub.  

  - **TensorBoard**  
    - For **TensorFlow**, users can utilize **TensorBoard** within a Jupyter Notebook, accessing it similarly to a standard Notebook.  

  - **Google Colab Environment**  
    - We provide a **Google Colab-compatible** Docker image, allowing users to run the same environment as **Google Colab**—without its limitations—on our **on-premise infrastructure**.  

- **Special Storage Options**
  - Some instances have **pre-configured storage access**, where different user groups have distinct permissions:  
    - A **restricted** group may access only their *home* directory and a *shared* (optionally read-only) storage.  
    - Another group may have **full access** to all *home* directories and shared storage with **read-write** permissions.  

- **Running Additional Jobs**
  - Some instances allow access to the **Kubernetes API**, enabling users to spawn additional background jobs **without requiring extra credentials**.  
  - **Resource quotas** can be enforced on both **Notebooks** and **jobs** to manage resource usage.  

- **AI Integration**
  - Our main JupyterHub instance integrates **[Jupyter-AI](https://jupyter-ai.readthedocs.io/en/latest/)**, providing access to **GPT-4o-Mini** and **GPT-4o** language models to assist with coding and problem-solving.

  ![ai-chat](/img/jupyterhub/aichat.png) 

- **VS Code Integration**
  - Some Docker images come with **[code-server](https://coder.com/)** pre-installed, allowing users to work in a **VS Code-like environment** instead of the standard JupyterLab interface.  

- **SSH Access**
  - If **external access** to a running Notebook instance is required, users can launch an instance with an appropriate Docker image and **connect via SSH**.  
  - Due to **IPv4 address exhaustion**, this feature is currently **limited to IPv6 access only**.  
  - SSH access can be useful for connecting a **local instance of VS Code** to a running Notebook instance.  

- **NBGrader**
  - [NBGrader](https://nbgrader.readthedocs.io/en/stable/) is a tool designed to enhance course support, allowing instructors to create notebook-based assignments with coding exercises and written responses.  
  - We have extended and adapted this tool for seamless integration with our Notebook instances.  

- **Installing Packages**  
  - Most images come with [Conda](https://anaconda.org/anaconda/conda) pre-installed, allowing users to install additional packages easily.  
  - Our images are configured to support the `sudo` command, enabling users to install system packages within a running Notebook. However, these installations are **temporary** and will be lost when the Notebook is restarted.

- **Automatic Cleanup of Unused Notebook Instances**
  - Notebook instances do not have a **fixed running time limit**, allowing users to work without interruption unless resources are idle for extended periods.  
  - We have developed a custom controller that **monitors resource usage** and **notifies users** about idle Notebooks with low activity.  
  - If no action is taken, the Notebook is **automatically terminated** after a few days, freeing up resources.

- **Exposing Web Servers from Notebooks**  
  - We enable the exposure of web (HTTP) servers running inside a Notebook to a public URL, similar to Cloudflare.  
  - The exposed server will be assigned the domain `<name>.flare.cloud.e-infra.cz` and will automatically receive an SSL certificate.  

# In Preparation  

- **Enhanced AI Integration**  
  - We are developing a **Welcome overlay** that will summarize a user's recent activities when reopening an old Notebook.  
  - We are working on an **automatic Docker image builder** that generates custom images based on user requests.  
    - Example request: *I need a Notebook image based on Ubuntu 24.04 with Python 3.12 and CUDA 12.8.*  
    - The AI will generate a `Dockerfile`, build the Docker image, and push it to our Docker registry, making it available in JupyterHub.

- **Even Better UI Design**  
  - We are currently **revamping the UI** to enhance usability and improve the overall experience.  
    - The **image selector** will be **searchable** and more intuitive.  
    - **Improved personal usage statistics** will be available directly within JupyterHub pages.  

# Instances on Demand

Users can deploy **simple JupyterHub instances** on their own within our Kubernetes infrastructure. However, this basic setup comes with some limitations, such as **restricted storage options** and **no user isolation** when accessing the Kubernetes API.  

For a **fully featured setup** with advanced capabilities, users can request a custom deployment by contacting us at [k8s@cerit-sc.cz](mailto:k8s@cerit-sc.cz).  
