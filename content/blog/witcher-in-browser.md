---
date: '2025-02-20T18:26:32Z'
title: 'Witcher in a Browser'
thumbnail: '/img/witcher/screen1.png'
description: "How we run remote desktops"
tags: ["Lukáš Hejtmánek", "CERIT-SC", "WebRTC", "Remote Desktop"]
colormode: true
---

# NVIDIA GeForce NOW On-Premise?<br/>Yes, It's Possible!  

We successfully deployed an open-source alternative to NVIDIA's closed GeForce NOW system on a Linux platform using NVIDIA hardware.  

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

## Selkies Project

The [Selkies](https://github.com/selkies-project) project provides a full-stack remote desktop solution based on the WebRTC protocol. It leverages the [GStreamer](https://gstreamer.freedesktop.org/) multimedia framework on the remote side to capture screen and audio, encode them into an audio/video stream (preferably H.264 + Opus), and transmit them to the client.  

Selkies also includes a sample web interface that connects a user’s web browser to the remote desktop, enabling keyboard and mouse event forwarding from the client to the remote system. Additionally, a reference STUN/TURN server implementation is provided, which must be deployed on a public address to facilitate communication when both the remote desktop and the user are behind firewalls or private networks.  

For more details, refer to the [quick start guide](https://selkies-project.github.io/selkies-gstreamer/start/).  

### Image Encoding

Capturing and encoding images is a resource-intensive process. To ensure a smooth user experience, hardware acceleration is essential, as CPU-based H.264 encoders are not fast enough for real-time 1080p video. Since CPU-based encoding cannot fully leverage parallel processing, we use NVIDIA GPUs to accelerate both image capture and encoding.  

However, data center compute GPUs such as the A100 and H100 lack video encoding capabilities, making them unsuitable for this purpose. Instead, we use NVIDIA A40, A10, or L4 GPUs for acceleration.  

### Image Capture

We have improved the original image capture and encoding pipeline, which previously relied on GStreamer's `ximagesrc` plugin. This plugin uses the `XGetShmImage` call to the X11 server, which is a synchronous operation. Although this function takes only about 4ms per 1080p frame, at 60 FPS, it blocks the X11 server for approximately 25% of the available frame time. This results in a 25% performance degradation and noticeable lag in the remote X11 session.  

Another drawback of this approach is that uncompressed frame data is transferred from GPU to CPU memory—a slow operation—only to be immediately transferred back to GPU memory for encoding. This places additional strain on the system bus and further degrades performance.  

#### NVIDIA Capture SDK

To overcome these issues, we replaced `ximagesrc` with the [NVIDIA Capture SDK](https://developer.nvidia.com/capture-sdk). This SDK can capture frames directly from the GPU framebuffer and, when used with [NVENC](https://developer.nvidia.com/video-codec-sdk), returns an encoded video stream that is significantly smaller than the raw uncompressed frame.  

We implemented this solution as a GStreamer plugin, which is available on [GitHub](https://github.com/CERIT-SC/gstreamer-nvimagesrc).  

## Deployment  

Our entire solution is deployed on our Kubernetes platform as a Rancher application. For more details, see our [documentation](https://docs.cerit.io/en/docs/rancher-apps/desktop).  

Using this approach, we provide scientists with easy access to desktop applications such as [Blender](https://www.blender.org/), [MATLAB](https://www.mathworks.com/products/matlab.html), and [ANSYS](https://www.ansys.com/)—allowing them to leverage the computing power of the e-INFRA CZ data center directly from a web browser.  

For deployment manifests, visit our [GitHub Rancher repository](https://github.com/CERIT-SC/rancher-apps).  

To showcase the capabilities of our project, we also tested running a computer game, [The Witcher 2](https://www.thewitcher.com/en/witcher2). Although an older title, it has a native Linux version and demands significant computing resources. Whether surprising or not, the game runs seamlessly in a web browser, offering an experience comparable to NVIDIA GeForce NOW 🚀.

## See the Result  

{{< youtube id="UNiEZSw_3KE" autoplay=false autotitle=false title="Witcher" >}}

## Final Thoughts  

- As mentioned above, using a web browser has some drawbacks. The most significant issue is that the largest latency comes from the browser buffers (around 100ms), and there is very little that can be done to mitigate this. Moreover, there are basically no alternative WebRTC implementations available, such as standalone applications. Even if such an option exists, it still relies on a web browser core. Additionally, users often open multiple tabs, and different tabs may further increase latency and introduce lags in the displayed frame. This happens because browsers do not seem to prioritize the tab running the WebRTC stream.  

- Encoding bitrate has a direct impact on video and image quality. A reasonable bitrate of around **10 Mbps for 1080p** is more than acceptable for video games or remote desktop video content. However, text, menus, or terminal applications may still exhibit visible compression artifacts. The most notable quality issue arises from differences between the remote and local screen resolutions. For example, if the remote resolution is **1080p** but the local display is **4K**, the image may appear slightly blurry. In such cases, **VNC provides far better image quality**. Additionally, H.264 compression struggles with solid-colored areas, creating "snow-like" artifacts.  

- **4K frames at 60 FPS** are particularly challenging for both the remote and client sides. On the remote side, encoding a **4K frame takes about 11ms**, and at **60 FPS**, the system has only **16ms** in total to process a single frame—leaving just **4–5ms** for transmission to the client. On the client side, if the web browser cannot utilize hardware acceleration, a CPU-based decoder will likely be too slow to handle a **4K 60 FPS** video stream.  

- **Does it matter whether we use 30 FPS or 60 FPS?** Yes, it does. The remote side is always at least **one frame behind** because the frame must be drawn before it can be captured. Similarly, on the local side, the frame must be drawn and flipped before appearing on the screen. At **30 FPS**, a single frame takes **33ms (1/30 sec)**, resulting in a total latency of at least **66ms**. At **60 FPS**, each frame takes **16ms (1/60 sec)**, reducing total latency to **32ms**.  

- **Can we use compression other than H.264?** It depends. **VP8 and VP9** are generally supported but often provide **worse subjective quality** than H.264. **H.265 (HEVC)** should offer better quality, but it must be **explicitly supported by the browser**—WebRTC support for H.265 is independent of standard video playback capabilities. An even better option would be **AV1 compression**, but beyond browser support, **the encoding hardware must also support it**. **NVIDIA** has offered AV1 encoding since the **Ada** generation, which includes **L4, L40, and L40s** datacenter hardware—but not the **A40**.  
