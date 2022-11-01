---
title: Building A Sideload Shell Program in Android
date: 2022-10-26
permalink: /posts/android-shell
tags: 
    - Android
    - Java
    - C/C++
    - Shell
---

A normal Android app almost always comes with graphics interfaces (except for services) and limited privileges.
In this post I'll show how to write a program (in Java or C/C++) that runs in headless mode and possesses system-level privileges,
which can be used to perform various tasks in practice such as system service monitoring, debugging and performance profiling.

Such a program can be side-loaded into the system at run time (which requires the shell privilege of `adb` but does not require root), 
and executed as a daemon process in the background, 
thus providing immense flexibility if you wish to access certain hidden system service interfaces.
For example, Android Studio uses a similar shell program to perform its debugging and CPU/memory monitoring in practice.

## Building a Java Shell

### Writing Makefile
A normal Java shell program is easy to build in Android.
All you need to do is write a simple `Main.java` file, and provide a `main` function there.
Everything is exactly the same as you would do in building a Java program in PC/server platforms.
To make the Java program, you'll need an `Android.mk` (or `Android.bp` for higher Android versions), which should look like this:
```make
# For Android.bp you can always use the androidmk tool provided by AOSP to convert the Android.mk file into bp file
LOCAL_PATH:= $(call my-dir)

include $(CLEAR_VARS)
LOCAL_SRC_FILES := $(call all-subdir-java-files)
LOCAL_MODULE := Mainlib
LOCAL_MODULE_STEM := Main
include $(BUILD_JAVA_LIBRARY)
```

### Creating Android Context

In many cases, your Java program may need to interface with system services.
To do this, your program should be more than a traditional Java program but possess an Android context.
However, normally such a context is only accessible to an Android app.
What you need to do is thus disguising your program as an app process (without loosing higher privileges).

To this end, you should first prepare a main looper upon start and acquire the system context.
Note that you should also ensure that the main thread is still running when you use this context to acquire system services.
The code should look like this.
```java
public static void main(String[] args) {
    // Prepare the main looper
    Looper.prepareMainLooper();
    Context systemContext = ActivityThread.systemMain().getSystemContext();
    PackageManager packageManager = systemContext.getPackageManager();

    System.out.println("Hello World");

    // This ensures that the main thread keeps running
    // You can also execute your logic in a separate thread 
    // and wait for the thread to finish here, 
    // so that you don't need to call the loop() func
    Looper.loop();
}

```

### Building
To build the Java program, you'll need the compile utilities in Android source.
Thus, it's necessary that you build the program within AOSP's code base.
To this end, you'll need first to download and build AOSP's source code (see the instructions [here](https://source.android.com/setup/build/downloading)).
And put your Java source and `Android.mk` in the framework directory.
The recommended directory is in `frameworks/base/cmds` where several built-in shell programs reside.
Make your own sub-directory there and put everything in it.
And then you can build the program by `mmm frameworks/base/cmds/XXX`, where `XXX` is your sub-directory for holding your source and makefile.

### Executing the Java Program

To run the Java program you just build,
you should write a shell program like this.

```sh
base=/data/local/tmp

mkdir /data/local/tmp/dalvik-cache
trap "" HUP
# com.example.hello should be replaced with your actual Java class
ANDROID_DATA=$base app_process -Djava.class.path=$base/classes.dex $base com.example.hello "$@"
```
Now push the shell script to your phone's `/data/local/tmp` path.
Then push the `classes.dex` in your Android build output path (`out/target/common/obj/JAVA_LIBRARIES/Mainlib_intermediates`) to your phone's `/data/local/tmp` path as well.
Next you should enter `adb shell` and `chmod +x` to grant your shell script the execution right.
Finally you can execute the shell script to run the Java program.
If you wish to execute your program as a daemon process in the background,
you can use the following command to run the shell script.
```sh
setsid XXX.sh >/dev/null 2>&1 < /dev/null
```

## Building a Native Shell

For a native shell, the steps should remain largely unchanged.
One difference is the makefile, which should look like this now.
```make
LOCAL_PATH:= $(call my-dir)
include $(CLEAR_VARS)

LOCAL_SRC_FILES:= \
	main.cpp

LOCAL_SHARED_LIBRARIES := \
	libbase \
	libutils \
	liblog \
    libcutils \
	libbinder

LOCAL_MODULE:= Main

LOCAL_MULTILIB := both
LOCAL_MODULE_STEM_32 := $(LOCAL_MODULE)32
LOCAL_MODULE_STEM_64 := $(LOCAL_MODULE)64

include $(BUILD_EXECUTABLE)
```

This makefile should allow building both 32-bit and 64-bit shell programs simultaneously.
You can find the built binary (named `Main32` and `Main64`) in `out/target/product/generic_x86_64/system/bin` .
Note that depending on your AOSP lunch target, `generic_x86_64` could be other paths.

This binary program does not require an additional shell script to execute.
You can directly push the binary to `/data/local/tmp` and execute it there.
To daemonize your program, you can simply call the `daemon()` function of Linux in your source.
