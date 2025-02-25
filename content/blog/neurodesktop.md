---
date: '2025-02-25T13:54:44Z'
title: 'Neurodesktop for Kubernetes'
thumbnail: '/img/neurodesktop/thumbnail.png'
description: "Running Neurodesktop applications as native pods in Kubernetes"
tags: ["Kristián Kováč", "CERIT-SC", "Neurodesktop", "Singluarity", "Kubernetes", "Jupyterhub"]
colormode: true
---

# Improving Neurodesktop for Kubernetes: Running applications as native pods

[Neurodesktop](https://www.neurodesk.org/) was originally designed to run each application using Singularity containers, ensuring isolated execution and simplified application distribution. However, this approach presents challenges in our Kubernetes deployment, where running Singularity containers would require pods to be launched under the root user. This setup would not be ideal due to security concerns and the additional complexity of managing containers inside other containers. 

To address these issues, we modified Neurodesktop’s architecture to run applications as native Kubernetes pods.

## What is Neurodesktop?

Neurodesktop is a containerized computational environment designed to provide researchers with an easy-to-use and fully reproducible platform for neuroimaging and data analysis. It offers a pre-configured, Linux-based desktop environment, allowing users to work with various [neuroimaging applications](https://www.neurodesk.org/docs/overview/applications/) without dealing with complex installations or system dependencies. 

Users can access Neurodesktop through their web browser, eliminating the need for local installation and enabling remote accessibility from anywhere. It seamlessly integrates with JupyterHub, allowing researchers to transition between Jupyter notebooks and a full graphical interface. By leveraging containerization, Neurodesktop provides a portable, efficient, and scalable solution that allows users to focus on their scientific work without worrying about software configuration and compatibility issues.

## The Challenge: Running applications in Kubernetes

While Singularity is effective in standalone environments, its integration into Kubernetes presented significant challenges. The primary issue is the requirement for privilege escalation, making it incompatible with the security model of Kubernetes, which does not allow pods to run with elevated privileges in a secure and manageable way. 

Additionally, Singularity's approach of nesting containers within the other (main) container does not align well with the design of Kubernetes. 

Given these constraints, a more Kubernetes-native solution was necessary to enhance Neurodesktop’s security and scalability in a Kubernetes environment.

## The Solution: Running applications as Native Pods

To resolve these challenges, we re-engineered Neurodesktop’s execution model, transitioning from a Singularity-based architecture to one where each application runs as its own Kubernetes pod. Each user accesses Neurodesktop through a primary JupyterHub-managed pod, which serves as their main working environment. When an application is launched, a dedicated Kubernetes pod is dynamically created for that application, ensuring an isolated and efficient runtime. 

Luckily, all of the applications available in Neurodesktop are already containerized using Docker. By leveraging these [existing Docker containers](https://hub.docker.com/u/vnmd), we can run them directly in Kubernetes without the need for Singularity. This approach simplifies the deployment process and ensures that each application runs natively within the Kubernetes cluster.

Additionally, each application pod starts an extra container running [Xpra](https://github.com/Xpra-org/xpra), which streams GUI applications back to the user’s main pod. This approach allows users to interact with the application seamlessly through their web browser, maintaining the user-friendly experience of Neurodesktop.

The image below demonstrates three applications running in the Neurodesktop environment, each mapped to a different Kubernetes pod. At the top, a terminal window displays the output of the `kubectl get pods` command. Each of the application windows on the desktop corresponds to a distinct pod, showing how Neurodesktop dynamically launches and manages containerized applications. In this particular example, one window is a terminal session running Julia, another window displays a FreeSurfer interface, and the third one runs GIMP. This setup illustrates the orchestration of multiple tools, all isolated in their own pods and seamlessly streamed to the user’s main session.

![Figure1](/img/neurodesktop/preview.png)

## Conclusion

By transitioning to a Kubernetes-native architecture, we have improved the security, scalability, and usability of Neurodesktop in a Kubernetes deployment. This new approach ensures that each application runs as a native Kubernetes pod, eliminating the need for privilege escalation and simplifying the management of applications within the cluster. Users can now enjoy a seamless experience while working with their favorite neuroimaging applications in Neurodesktop. This enhancement represents a significant step forward in providing researchers with a powerful and user-friendly platform for neuroimaging and data analysis in Kubernetes environments.