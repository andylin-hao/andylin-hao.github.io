---
title: Incorporate OpenGApps into Android-x86
date: 2021-11-15
permalink: /posts/incorporate-opengapps
tags: 
    - Android-x86 
    - OpenGApps
---

In this post I'll show the configurations required to build Android-x86 source with OpenGApps,
    which provides Google Play and GMS so that your customized Android-x86 can enjoy apps from Google Play.

1) Add the following contents to the end of `.repo/manifests/defaults.xml`.

  ```xml
  <remote name="opengapps" fetch="https://github.com/opengapps/"  />
  <remote name="opengapps-gitlab" fetch="https://gitlab.opengapps.org/opengapps/"  />
  <project path="vendor/opengapps/build" name="aosp_build" revision="master" remote="opengapps" />
  <project path="vendor/opengapps/sources/all" name="all" clone-depth="1" revision="master" remote="opengapps-gitlab" />
  <project path="vendor/opengapps/sources/x86" name="x86" clone-depth="1" revision="master" remote="opengapps-gitlab" />
  <project path="vendor/opengapps/sources/x86_64" name="x86_64" clone-depth="1" revision="master" remote="opengapps-gitlab" />
  ```

Note that the above contents may change as OpenGApps upgrades. Check the latest [here](https://github.com/opengapps/aosp_build#getting-started).

2) Locate the following content at `device/generic/common/device.mk`.

  ```makefile
  # Get GMS
  GAPPS_VARIANT ?= pico
  $(call inherit-product-if-exists,$(if $(wildcard vendor/google/products/gms.mk),vendor/google/products/gms.mk,vendor/opengapps/build/opengapps-packages.mk))
  ```

  3) Modify it to the following.

  ```makefile
  # Get GMS
  GAPPS_VARIANT := pico
  $(call inherit-product-if-exists,$(if $(wildcard vendor/google/products/gms.mk),vendor/google/products/gms.mk,vendor/opengapps/build/opengapps-packages.mk))
  ```

4) Run `repo sync`, `git lfs install`, and `repo forall -c git lfs pull` at the root directory of your Android-x86 source.

5) Add a `CleanSpec.mk` at `vendor/opengapps/build` with the following contents.

  ```makefile
  $(call add-clean-step, rm -rf $(PRODUCT_OUT)/system/priv-app/*)
  $(call add-clean-step, rm -rf $(PRODUCT_OUT)/system/app/*)
  ```

6) Add a `Android.mk` at `vendor/opengapps/build/modules` with the following contents so as to avoid building `ActionServices` which only provides ARM source.

  ```makefile
  include $(call all-named-subdir-makefiles,$(GAPPS_PRODUCT_PACKAGES))
  ```

7) Remove the following from `vendor/opengapps/build/opengapps-packages.mk`.

  ```makefile
  MarkupGoogle \
  GoogleCamera \
  SetupWizard \ # This is the boot-time setup wizard. You can keep it in Android 9, but do remove it in Android 10 as it incurs issues at boot time.
  ```

8) `make` your Android-x86 image.