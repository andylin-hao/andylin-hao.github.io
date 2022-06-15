---
title: Include Intel Houdini in Android-x86
date: 2022-02-13
permalink: /posts/houdini-x86
tags: 
    - Android-x86 
    - Intel Houdini
---

In this post I'll document the necessary steps for incorporating Intel Houdini in Android-x86 9. Note that this does not work on Android 10+.

Prior methods of `enable_nativebridge` no longer works on Android 8+ of Android-x86 due to missing libraries for downloads. Fortunately, [BlissOS](https://github.com/BlissRoms-x86/android_vendor_google_chromeos-x86) has provided an alternative solution by extracting the Houdini libraries from ChromeOS. Here's how it's done.

1) Create a `vendor/google` folder at the root directory if necessary, and run `git clone https://github.com/BlissRoms-x86/android_vendor_google_chromeos-x86 vendor/google/chromeos-x86`

2) Add the following to `device/generic/common/BoardConfig.mk`.

  ```makefile
  BOARD_SEPOLICY_DIRS += vendor/google/chromeos-x86/sepolicy
  -include vendor/google/chromeos-x86/board/native_bridge_arm_on_x86.mk
  ```

3) Remove the `hal_drm_widevine.te` file at `device/generic/common/sepolicy/nonplat`.

4) Add the following to `device/generic/common/device.mk`.

  ```makefile
  $(call inherit-product-if-exists, vendor/google/chromeos-x86/target/houdini.mk)
  $(call inherit-product-if-exists, vendor/google/chromeos-x86/target/native_bridge_arm_on_x86.mk)
  $(call inherit-product-if-exists, vendor/google/chromeos-x86/target/widevine.mk)
  ```

5) Modify `device/generic/common/nativebridge/nativebridge.mk` to the following.

  ```makefile
  #
  # Copyright (C) 2015 The Android-x86 Open Source Project
  #
  # Licensed under the GNU General Public License Version 2 or later.
  # You may not use this file except in compliance with the License.
  # You may obtain a copy of the License at
  #
  #      http://www.gnu.org/licenses/gpl.html
  #
  
  # Enable native bridge
  WITH_NATIVE_BRIDGE := true
  
  # Native Bridge ABI List
  NATIVE_BRIDGE_ABI_LIST_32_BIT := armeabi-v7a armeabi
  NATIVE_BRIDGE_ABI_LIST_64_BIT := arm64-v8a
  NATIVE_BRIDGE_ABI_LIST := x86_64 x86 arm64-v8a
  
  LOCAL_SRC_FILES := bin/enable_nativebridge
  
  PRODUCT_COPY_FILES := $(foreach f,$(LOCAL_SRC_FILES),$(LOCAL_PATH)/$(f):system/$(f)) \
      $(LOCAL_PATH)/OEMBlackList:$(TARGET_COPY_OUT_VENDOR)/etc/misc/.OEMBlackList \
      $(LOCAL_PATH)/OEMWhiteList:$(TARGET_COPY_OUT_VENDOR)/etc/misc/.OEMWhiteList \
      $(LOCAL_PATH)/ThirdPartySO:$(TARGET_COPY_OUT_VENDOR)/etc/misc/.ThirdPartySO \
  
  PRODUCT_PROPERTY_OVERRIDES := \
      ro.dalvik.vm.isa.arm=x86 \
      ro.enable.native.bridge.exec=1 \
  
  # ifeq ($(TARGET_SUPPORTS_64_BIT_APPS),true)
  PRODUCT_PROPERTY_OVERRIDES += \
      ro.dalvik.vm.isa.arm64=x86_64 \
      ro.enable.native.bridge.exec64=1
  # endif
  
  PRODUCT_PROPERTY_OVERRIDES := ro.dalvik.vm.native.bridge=libhoudini.so
  
  # ifneq ($(HOUDINI_PREINSTALL),intel)
  # PRODUCT_DEFAULT_PROPERTY_OVERRIDES := ro.dalvik.vm.native.bridge=libnb.so
  
  # PRODUCT_PACKAGES := libnb
  # endif
  
  $(call inherit-product-if-exists,vendor/intel/houdini/houdini.mk)
  ```