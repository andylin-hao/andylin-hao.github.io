---
title: Step Debugging Android Native Framework in VS Code
date: 2022-10-26
permalink: /posts/debug-android
tags: 
    - Android
    - Debugging
    - C/C++
    - VS Code
---

Tracing the execution flow of Android Framework and debugging your own system components are significantly more convenient with the step debugging capability.
It saves the trouble of frequently instrumenting the Android sources.
This post will show how to enable this capability using LLDB and VS Code.

## Basic Environment

I test the debugging method in a Linux host environment running an Android emulator (both [QEMU-KVM with Android-x86](https://github.com/TrinityEmulator/TrinityEmulator/wiki/Setup-of-Other-Emulators#qemu-kvm) and Google Android Emulator are fine).

For the host environment, you need to install both LLDB (`sudo apt install -y lldb`) and Android SDK (installed via Android Studio and the SDK commandline tool).

For the guest (i.e., emulators or real devices running your own Android OS) environment,
    it's better if you have root access.
No root access is also fine but you cannot debug via apps that are not debuggable.
Of course, the guest OS should be compiled by you whose symbol files are thus available to you in the output path of the source base.

## Guest Setup

1) Push the `lldb-server` executable in your Android SDK to the (emulated/real) device you wish to debug. The executable can be found in `Sdk/ndk/${NDK_VERSION}/toolchains/llvm/prebuilt/linux-x86_64/lib64/clang/${clang VERSION}/lib/linux/x86_64` as long as you have installed NDK in your SDK path (select NDK in your SDK management page in Android Studio). To push the executable, use the following command:
```bash
adb push lldb-server /data/local/tmp
```

2) Type the following commands to start the server and listens to a unix socket for connections:
```bash
adb root # if your device has root access
adb shell "chmod 777 /data/local/tmp/lldb-server"
adb shell "/data/local/tmp/lldb-server p --server --listen unix-abstract:///data/local/tmp/debug.sock"
```

3) For native components, we are either dealing with a native executable, e.g., a native system service, or a native library.
For the former, you can directly obtain its process ID (PID) via `top` or `ps`. 
For the latter, you need to run an app that uses the library and acquire the app's PID.
This post focuses on the latter case.

4) Attaching the debugger to an already running app will be detailed later. Here we'll first look at *how to pause an app process' startup before the LLDB debugger is attached to it*.

This is useful when the component you wish to debug is still quite buggy and can easily crash the app before the debugger can even attach.
The Android developer blog gives a detailed introduction of how to realize this via the jdb Java debugger. You can find it [here](https://source.android.com/docs/core/tests/debug/gdb#app-startup).

An important note is that the `jdb attach` command will not work if your Android Studio is also running.
Because Android Studio continuously monitors and preempts any available jdb connections,
    rendering your connection attempts useless.

Also, naturally, to realize pausing before LLDB attaching, you should always attach jdb (and let the app continues in the Java layer) after LLDB is attached (to be detailed later).

## Host Setup
At the host side, you should have an Android/AOSP code base which matches the Android OS run in your guest machine.

1) Open the Android code base using VS Code.

2) Install the [CodeLLDB](https://marketplace.visualstudio.com/items?itemName=vadimcn.vscode-lldb) extension in your VS Code.

3) Click the `Run` tab of VS Code, select `Add configuration`, and select LLDB. This will create a `launch.json` file in your `.vscode` folder. Replace its content as follows
```json
{
"configurations": [
{
    "initCommands": [
        "settings append target.exec-search-paths ${ANDROID_SRC}/out/target/product/x86_64/symbols/system/lib64/ ${ANDROID_SRC}/out/target/product/x86_64/symbols/system/lib64/hw ${ANDROID_SRC}/out/target/product/x86_64/symbols/system/lib64/egl ${ANDROID_SRC}/out/target/product/x86_64/symbols/system/lib64/extractors ${ANDROID_SRC}/out/target/product/x86_64/symbols/system/vendor/lib64/ ${ANDROID_SRC}/out/target/product/x86_64/symbols/system/vendor/lib64/hw ${ANDROID_SRC}/out/target/product/x86_64/symbols/system/vendor/lib64/egl ${ANDROID_SRC}/out/target/product/x86_64/symbols/system/vendor/lib64/dri ${ANDROID_SRC}/out/target/product/x86_64/symbols/system/vendor/lib64/extractors ${ANDROID_SRC}/out/target/product/x86_64/symbols/system/vendor/lib64/mediacas ${ANDROID_SRC}/out/target/product/x86_64/symbols/system/vendor/lib64/mediadrm ${ANDROID_SRC}/out/target/product/x86_64/symbols/system/vendor/lib64/soundfx",
        "platform select remote-android", 
        "platform connect unix-abstract-connect:///data/local/tmp/debug.sock",
    ], 
    // Map the current path to the Android code base
    "sourceMap": {
        "": "${ANDROID_SRC}", 
        "/b/f/w": "${ANDROID_SRC}", 
        ".": "${ANDROID_SRC}"
    }, 
    "pid": ${PID}, 
    "name": "Remote Android Debug", 
    "relativePathBase": "${ANDROID_SRC}", 
    "request": "attach", 
    "type": "lldb"
}
]
}
```
where `${ANDROID_SRC}` is the path to your Android code base and `${PID}` is the process ID of the process you wish to debug.
Also, note that the `target.exec-search-paths` appended in the `initCommands` are not always the same for different Android versions. You need to check the paths and see if they actually exist, and whether there're missing folders that are not included.

4) Click the debug button of VS Code to begin debugging. Remember to change the `${PID}` for different debuggees.

5) Once the LLDB is attached, your can attach jdb to let the app continue execution. Now you are put breakpoints anywhere in VS Code to realize step debugging.
