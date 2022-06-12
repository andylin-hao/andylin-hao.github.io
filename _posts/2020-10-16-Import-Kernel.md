---
title: Importing Linux Kernel Code to VS Code and Enjoying Code Navigation
date: 2020-10-16
permalink: /posts/import-kernel
tags: 
    - Kernel 
    - Code
---

In this post I'll take you to import kernel code to VS Code for a nice code viewing and developing experience. You can say goodbye to CLion which eats your memory without mercy. This tutorial will be based on the Android-x86 source tree which contains Linux kernel almost identical to the upstream kernel.

* The first step is installing VS Code of course. I'll simply skip this step for any experienced developers.
* Now search and install the C/C++ Extension in VS Code's extension market.
* Build the kernel before you do anything else.
* Use a magical script named ``gen_compile_commands.py`` in the ``kernel/script`` file. If it does not exist, you can find it in the upstream Linux kernel repository (https://github.com/torvalds/linux/blob/master/scripts/gen_compile_commands.py).
* Run the command by ``python gen_compile_commands.py -d ${path-to-kernel-build-directory}``. In the case of Android-x86, the corresponding kernel build path is ``out/target/product/{x86_64}/obj/kernel`` ({} denotes that this name can vary depending on your chosen build target).
* After that you can find you ``compile_commands.json`` in the kernel build path.
* Now go back to VS Code and enter ``Ctrl+Shift+P`` to initiate command execution.
* Enter ``>C/C++: Edit Configurations (UI)`` (you should get prompted even if you have not finished the command).
* Now open ``Advanced Settings`` and find the ``Compile commands`` setting.
* Enter the path to your ``compile_commands.json`` and you are good to go! Enjoy  your VS Code journey.