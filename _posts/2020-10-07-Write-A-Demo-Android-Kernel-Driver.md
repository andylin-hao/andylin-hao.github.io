---
title: Write A Demo Android Kernel Driver
date: 2020-10-07 13:57:45
permalink: /posts/demo-driver
tags: 
    - Android 
    - Kernel 
    - Driver
---

This post will talk about how to write a small "hello-world" driver in Android kernel.

## Environment

* Android version: Android-x86 (q-x86, x86_64_userdebug)
* QEMU: 5.1
* Host: Ubuntu 20.04

## A Demo Driver

* Create a new directory in ${android-source-root}/kernel/drivers named ``hello``

* ``cd ${android-source-root}/kernel/drivers/hello``

* Create a new C file named ``hello.c``, with the following content:

  ```c
  #include <linux/module.h>  
  #include <linux/kernel.h>  
  #include <linux/fs.h>  
  #include <linux/miscdevice.h>  
  
  MODULE_LICENSE("GPL");  
  MODULE_AUTHOR("AndyLin");  
  MODULE_DESCRIPTION("Hello Kernel Device");  
  MODULE_VERSION("1.0");  
  
  #define CMD_COMMAND 0x1336  
  
  long hello_ioctl(struct file *filp,
      unsigned int cmd,  
      unsigned long arg){  
      switch(cmd){  
      case CMD_COMMAND:  
          printk("Hello Module hello_ioctl() exced");  
          break;  
      default:  
          printk("Hello Module unknown ioctl cmd");  
      }  
      return 0;  
  }  
  
  struct file_operations hello_fops = {
      unlocked_ioctl: hello_ioctl  
  };  
  
  static struct miscdevice hello_device = {
      minor: MISC_DYNAMIC_MINOR,  
      name: "hello",  
      fops: &hello_fops,  
      mode: 777
  };  
  
  static int hello_begin(void){  
      int ret;  
      ret = misc_register(&hello_device);
      if(ret)  
          printk("Failed to register misc device");  
      else  
          printk("Hello Module successfully loaded");  
  
      return ret;  
  }  
  
  static void hello_exit(void){  
      misc_deregister(&hello_device);
      printk("Hello module exit"); 
      return; 
  }  
  
  module_init(hello_begin);
  module_exit(hello_exit); 
  ```

* In the ``${android-source-root}``, run the following commands to build the kernel first:

  ```shell
  source build/envsetup.sh
  lunch android_x86_64-userdebug
  make kernel -j${thread_num}
  ```

* Create a ``MakeFile`` in the ``hello`` directory:

  ```c
  obj-m := hello.o  
  
  // This is why we build kernel first -- in order to generate the kernel directory
  KERNELDIR := ${android-source-root}/out/target/product/x86_64/obj/kernel/
  PWD :=$(shell pwd)  
  ARCH=x86_64
  CROSS_COMPILE=${android-source-root}/prebuilts/gcc/linux-x86/x86/x86_64-linux-android-4.9/bin/x86_64-linux-android-  
  CC=$(CROSS_COMPILE)gcc  
  LD=$(CROSS_COMPILE)ld  
  CFLAGS_MODULE=-fno-pic  
  
  all:  
  	make -C $(KERNELDIR) ARCH=$(ARCH) CROSS_COMPILE=$(CROSS_COMPILE) M=$(PWD) modules
  clean:    
  	rm *.o *.ko *.mod.c *.order *.symvers
  ```

* Now we can finally ``make``

* After ``make``, you can see a ``hello.ko`` file in the directory

* Now you can build your Android-x86 and run it atop QEMU (you might run into compilation errors due to dirty ``kernel`` directory caused by ``make kernel``; if that happens, you can first run ``make mrproper`` in the ``kernel`` directory before building the entire system)

* Connect to the system via ``adb`` (note that you should first turn on your WiFi in Android, and you can connect to it by running ``adb connect localhost:4444``, if your QEMU booting command contains the following: ``-net nic -net user,hostfwd=tcp::4444-:555``

* Use ``adb`` to push the ``hello.ko`` to Android's any directory as you please

* In ``adb shell``, run ``insmod ${path-to-the-hello.ko-file}``

* Now you are all done! You can use ``dmesg | grep Hello`` to check the output in the driver's code