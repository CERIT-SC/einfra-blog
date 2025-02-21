---
date: '2025-02-20T18:26:32Z'
title: 'Witcher in a Browser'
thumbnail: '/img/witcher/screen1.png'
author: "Lukáš Hejtmánek"
description: "How we run remote desktops"
tags: ["WebRTC", "Remote Desktop", "CERIT-SC"]
colormode: true
---

# NVIDIA GeForce NOW On-Premise?<br/>Yes, It's Possible!  

We successfully implemented an open-source alternative to NVIDIA's closed GeForce NOW system on a Linux platform using NVIDIA hardware.  

## Remote Desktops  

Remote desktops have been around for a long time. The need to connect from a local computer to a remote machine with a graphical interface is well-established. Various solutions exist for Linux-based operating systems, including:  

- **X11 Forwarding & Tunneling** – Simple but inefficient, often slow.  
- **VNC (TigerVNC, RealVNC, etc.)** – Ideal for basic desktop tasks (terminals, office applications) due to high image quality (no compression artifacts) and low latency (smooth mouse and keyboard response).  

However, even VNC has its limitations:  

1. **Inefficiency for Dynamic Content** – VNC struggles when large portions of the screen frequently change.  
2. **Lack of 3D Acceleration Support** – Rendering 3D-accelerated content remotely requires transferring frames at high rates (e.g., 60 FPS for a smooth experience).  
3. **Bandwidth Constraints** – To match HDMI-level quality, we need:  
   - **1080p (1920×1080) at 60 FPS** → up to **3 Gbps**  
   - **4K (3840×2160) at 60 FPS** → up to **12 Gbps**  

### VNC’s Additional Drawback  

If a VNC session is attached to an existing X11 session, efficiency drops significantly. A more optimized approach involves starting a dedicated VNC server on the remote machine to ensure efficient data transport.  

---

With these limitations in mind, we explored more advanced open-source alternatives that enable high-performance remote streaming on NVIDIA hardware. Stay tuned for more details! 🚀


## See the Result

{{< youtube id="UNiEZSw_3KE" autoplay=false autotitle=false title="Witcher" >}}
