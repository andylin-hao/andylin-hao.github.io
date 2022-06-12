---
title: Linux Graphics Stack
date: 2021-04-25 17:00:55
permalink: /posts/linux-graphics-stack
tags: 
    - Linux 
    - Graphics
---

Due to heavy historical burden, the Linux graphics stack is extremely complex with fragmented and intricate software components. As a result, it's quite exhausting to fully grasp its core idea and principles. This has posed significant challenges to my recent work on graphics virtualization of Android. Fortunately, after days of struggling with its sources and prior technical posts, I may have obtained some levels of understandings regarding the framework of the Linux graphics stack. SO, before I completely lose track of the story and to benefit other unfortunate comrades, I decide to document everything here in this post. 

Hope you'll find it useful.

# Overview
Before we get started, I'll just throw an architectural graph at you for three purposes:

1) for you to have some basic ideas of the graphics stack's various components to be discussed in this post, 
2) for you to reference each component's position in the whole stack when reading the parts to follow, and
3) for you to understand the chaos and challenges ahead.

Here it comes:

<img src="{{site.url}}/images/posts/graphics stack.svg">

Note that this graph only demonstrates a common architecture of today's Linux graphics stack built upon the Direct Rendering Infrastructure (DRI).
As other architectures are now rarely used, they are out of this post's scope.

Now you are familiarized with the architecture, I'll next introduce the components within and their rationale behind by walking you through the history and showing you the reasons they came to be in the first place.

# Indirect Rendering

Nothing is complex in the beginning. The same goes for the Linux graphics. In the early days, only one software in the system is responsible for graphics rendering and communicating with the GPU --- X server. Applications use ``libX11`` to tell the X server how to render graphics. After the X server receives hardware-independent rendering commands (defined in the X11 protocol, thus known as Device-Independent X, or ``DIX``) from applications, it then translates these commands into hardware languages that the GPU can understand (e.g., register writes and memory allocations). This translation software module inside the X server is commonly referred to as Device-Dependent X driver, or ``DDX`` driver. Here “driver” is a graphic argot that differs from what you have probably learned in OS design --- the ``DDX`` driver actually operates in the user space rather than in the kernel space as most OS drivers do. Intuitively, to accommodate heterogeneous GPUs, many ``DDX`` drivers should be developed accordingly.

At that time, only 2D rendering is necessary for computer graphics. Therefore, the initial ``libX11`` only includes 2D-related operations. However, with the advent of 3D graphics, the above-described architecture soon fails to satisfy people's needs. To perform 3D rendering on heterogeneous hardware,
OpenGL is developed as the new GPU-agnostic interfaces for 3D operations. Specifically, OpenGL is only a series of function specifications that dictate only “what they do” but do not restrict “how to do it”. Thus, the actual OpenGL implementation is totally delegated to the system, which normally can be found in the ``libGL.so`` your 3D applications link to.

To fully exploit the hardware capability and avoid complete refactoring of the whole stack, the initial implementation of OpenGL is built upon the 2D X server architecture. The technique that makes this possible is what we call the GL eXtenstion (``GLX``) of the X11 protocol. ``GLX`` is a wapper that wraps OpenGL function calls in X11. The X server then similarly receives X11 commands and translates them into the hardware language as before. Since this process involves wrapping into X11 and intervention of the X server rather than directly asks GPU to perform 3D operations, it is thus later known as indirect rendering (the left path of our architecture graph).

Unfortunately, soon this indirect approach became insufficient for emerging 3D-intensive applications (e.g., 3D games) as it takes unnecessary detour to reach hardware. To address this, direct rendering is proposed.

# Direct Rendering
