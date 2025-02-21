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
   - Commodity network connection is between **40** and **100 Mbps**.

### VNC’s Additional Drawback  

If a VNC session is attached to an existing X11 session, efficiency drops significantly. A more optimized approach involves starting a dedicated VNC server on the remote machine to ensure efficient data transport.  

---

With these limitations in mind, we explored more advanced open-source alternatives that enable high-performance remote streaming on NVIDIA hardware. Stay tuned for more details! 🚀

## WebRTC Enters the Stage

WebRTC (Web Real-Time Communication) is an open-source protocol designed for low-latency, real-time communication over the web. It enables **peer-to-peer (P2P) connections** for audio, video, and data transfer without requiring additional plugins.

For remote desktops and game streaming, WebRTC offers significant advantages over traditional protocols like VNC or X11 forwarding. The only requirement for the client (user) is a web browser, which simplifies accessibility but also introduces some limitations, as discussed later.

### Why Use WebRTC for Remote Desktops and Game Streaming?

1. **Ultra-Low Latency**
   - WebRTC is optimized for real-time communication, achieving latencies as low as **10-50ms**, which is crucial for gaming and interactive desktop environments.

2. **Efficient Video Encoding with NVENC**
   - WebRTC supports **hardware-accelerated encoding (H.264, VP8, VP9, AV1)**, reducing CPU load and improving frame rates.
   - When combined with **NVIDIA NVENC**, it enables high-quality, low-latency streaming at **1080p 60 FPS or 4K 60 FPS**.

3. **Adaptive Bitrate Streaming (ABR)**
   - WebRTC dynamically adjusts video quality based on available network bandwidth.
   - This prevents lag and stuttering, ensuring a smooth experience even on unstable networks.

4. **Reliable Connectivity in NAT-Restricted Environments**
   - Most home networks use **Network Address Translation (NAT)**, which can create connection barriers.
   - WebRTC leverages **STUN/TURN servers** to establish reliable connections even in restrictive network conditions.

5. **Built-in Input and Audio Support**
   - WebRTC supports **bidirectional input streaming**, including mouse, keyboard, and game controllers.
   - Audio is transmitted using the **Opus codec**, ensuring high-quality sound with minimal bandwidth usage.

6. **Secure and Encrypted**
   - WebRTC enforces **DTLS and SRTP encryption** by default, making it more secure than traditional remote desktop protocols, some of which may transmit data unencrypted.

WebRTC’s real-time performance, adaptive streaming, and security make it a powerful solution for remote desktops and game streaming. However, as we’ll explore later, relying solely on a web browser does come with some trade-offs.


## See the Result

{{< youtube id="UNiEZSw_3KE" autoplay=false autotitle=false title="Witcher" >}}
