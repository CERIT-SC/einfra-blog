---
date: "2025-03-06T13:14:44Z"
title: "Protein Structure Prediction: A Web Application for Researchers"
thumbnail: "/img/foldify/thumbnail.png"
description: "Web application that simplifies protein structure prediction with AlphaFold tools."
tags: ["Romana Ďuráčiová", "CERIT-SC", "AlphaFold3", "Protein Prediction", "Kubernetes", "Bioinformatics"]
colormode: true
---

# Introduction

Protein structure prediction is a cornerstone of bioinformatics and molecular biology, with applications in drug discovery, disease research, and synthetic biology. However, traditional computational methods can be complex and resource-intensive, making them inaccessible to many researchers.

To address these challenges, we have developed a web application that simplifies process of protein structure prediction. Our platform integrates models like AlphaFold3, AlphaFold2, and optimised ColabFold, OmegaFold and ESMFold.

# Why Protein Structure Prediction Matters

Proteins are the building blocks of every organism, and their functions are determined by their 3D structures. Understanding protein structures helps in:

-   Drug discovery: Identifying potential drug targets and designing new medication.
-   Disease research: Understanding how mutations affect protein function.

Traditional experimental methods, such as X-ray crystallography and cryo-electron microscopy, are time-consuming and costly. Computational tools bridge this gap, offering faster and more accessible predictions.

# Overview of the web application

Foldify, our web application, streamlines the process of protein structure prediction, making it faster and more user-friendly. It also provides real-time feedback to guide users in case of incorrect input. Key features of the application include:

-   **Multiple Prediction Models in One Place:** Support for AlphaFold3, AlphaFold2, ColabFold, OmegaFold and ESMFold.
-   **User-Friendly Interface:** Simple job submission with minimal required input data with error feedback and hints.
-   **Built-In 3D Visualization:** Explore protein structures directly in your browser using the Mol\* Viewer..
-   **Real-Time Monitoring:** Track prediction progress and receive e-mail notification upon job completion.

![Figure1](/img/foldify/dashboard-foldify.png)

# How It Works

Foldify breaks down the prediction process into simple, intuitive steps:

1. **Select a Prediction Model:** Choose from AlphaFold3, AlphaFold2, ColabFold, OmegaFold or ESMFold prediction tools.
2. **Insert Input Data:** Provide an amino acid sequence. _For AlphaFold3, you can also upload a JSON configuration file._
3. **Configure Parameters:** Customize settings based on your preferences.
4. **Submit the Job:** The computaion request is processed in a Kubernetes cluster for execution.
5. **Monitor Progress:** rack the status of your job in real-time on the Dashboard.
6. **Download or Visualize Results:** Once the job is complete, download the results or explore the 3D structure and computation parameters directly in the web interface.

![Figure2](/img/foldify/result-foldify.png)

# Conclusion

Foldify makes protein structure prediction accessible, efficient, and scalable. Whether you're a seasoned researcher or a student delving into structural biology, our platform equips you with the tools you need to succeed.

We invite you to experience protein structure prediction with [Foldify](https://foldify.cloud.e-infra.cz/). Your feedback is invaluable as we continue to improve and expand our platform.

[[Access the Web Application]](https://foldify.cloud.e-infra.cz/) | [[Documentation]](https://docs-ng.cerit.io/en/docs/web-apps/foldify)
