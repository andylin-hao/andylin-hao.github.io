---
title: Enable VoLTE/ViLTE on Pixel in Unlisted Regions
date: 2022-10-25
permalink: /posts/volte-pixel
tags: 
    - Android
    - VoLTE
    - ViLTE
    - Cellular
    - Video Call
    - Pixel
---

In this post I'll document the steps to enable VoLTE/ViLTE (or their counterparts in 5G such as VoNR) on Google Pixel devices in unlisted regions, e.g., China.
For Chinese operators (e.g., China Unicom),
Pixel devices are already shipped with the corresponding Qualcomm modem mbn files,
which are placed under `/system/vendor/rfs/msm/mpss/readonly/vendor/mbn/mcfg_sw/generic/China/`.
However, such files are not enabled by default, which renders VoLTE/ViLTE unusable in China. 

The solution I use is based on Magisk, which enables patching the readonly system partition and modifies system properties to turn on VoLTE/ViLTE.
This method does not require flashing a new Android image, therefore is ready-to-use for stock Pixel devices.
However, if you're interested in building an Android image from source with VoLTE/ViLTE enabled by default, the required steps will also be discussed in the post.

I'll use a Pixel 3 XL running Android 11 as an example throughout the post. 
Though the method should be generalizable to other Pixel models and Android versions.

## Enable VoLTE/ViLTE with Magisk

Magisk enables systemless root and thus provide a means to modifying the readonly system partition and many system properties.
Follow the steps below to install Magisk on your Pixel device and make your own patch to turn on VoLTE/ViLTE.

### Magisk Installation
To install Magisk, first make sure you can get your hands on your Pixel's `boot.img` image.
To this end, I recommend that you download and flash a factory image of your Pixel following the instructions [here](https://developers.google.com/android/images).
After downloading the factory image, 
you should extract the `boot.img` file from the `image-XXX.zip` archive.
Once you finish flashing the factory image,
install the Magisk app provided [here](https://github.com/topjohnwu/Magisk/releases) on your Pixel.
Then use `adb push` to send the `boot.img` file extracted above to your phone's `/sdcard` directory (or any writable directory as you see fit).
Next open the Magisk app, click `install`, select `Select and Patch a File`, and then select the `boot.img` file sent to the phone to begin installation.
Now reboot your device and you're all set.

### Making Magisk Patch

With the above steps, you should be able to see that Magisk is installed and enabled on the device.
The next step would be to make a Magisk patch that can be installed to the system through Magisk.
I provide an example for Pixel 3 XL in this GitHub [repo] (https://github.com/andylin-hao/Pixel-VoLTE-Patch).
Generally, the patch involve modifying system properties and Qualcomm's `mbn` component.
For system properties, the corresponding modifications are shown in the `common/system.prop` file of the GitHub repo, which are (inspired by voltenabler)
```
# Debug Options
persist.dbg.ims_volte_enable=1
persist.dbg.volte_avail_ovr=1
persist.dbg.vt_avail_ovr=1
persist.dbg.wfc_avail_ovr=1

# Data Options
persist.data.iwlan.enable=true
persist.data.iwlan=1
persist.data.iwlan.ipsec.ap=1

# Radio Options
persist.radio.volte.dan_support=true
persist.radio.rat_on=combine
persist.radio.data_ltd_sys_ind=1
persist.radio.data_con_rprt=1
persist.radio.calls.on.ims=1
persist.radio.VT_ENABLE=1

# Other Options
persist.sys.cust.lte_config=true
persist.rcs.supported=1
```

Apart from system properties, `/system/vendor/rfs/msm/mpss/readonly/vendor/mbn/mcfg_sw/mbn_sw.txt` should also be modified to include Chinese operators.
For example, if you are using a China Unicom SIM card, just add the following to the `mbn_sw.txt` file.
```
mcfg_sw/generic/China/CU/Commercial/OpenMkt/mcfg_sw.mbn
mcfg_sw/generic/China/CU/Commercial/VoLTE/mcfg_sw.mbn
mcfg_sw/generic/China/CU/Lab/Test/mcfg_sw.mbn
```
However, for different devices, the `mcfg_sw.mbn` files may not lie in the same paths.
You should extract the whole `mcfg_sw` folder from your own phone (which can be found in the same path, i.e., `/system/vendor/rfs/msm/mpss/readonly/vendor/mbn/mcfg_sw/` in the phone) and modify the `mbn_sw.txt` file in the folder according to the `mcfg_sw.mbn` files included in your `generic` folder.
Also, the `mcfg_sw.mbn` files provided in my repo are extracted from Pixel 3 XL and not tested on other devices.
You should always extract the `mcfg_sw.mbn` files from your own device, and replace the corresponding `mcfg_sw.mbn` files in my repo with your own `mcfg_sw.mbn` files.

### Patch Installation

Compress the repo into a zip archive and push it to your device.
Open Magisk, click `Modules`, and install the zip archive you just push.
Now reboot, you should see that VoLTE/ViLTE are enabled in your data connection settings.
You can also check this by typing `*#*#4636#*#*` in your Dialer app.
However, although everything seems to be on, you may still fail to connect when initializing a video call.
This is because you still do not have an `ims` APN.
You should manually add an `ims` APN in your data connection settings.
To do this, fill the `Name`, `APN`, `APN Type` fields with `ims`, `APN protocol` filed with `IPV4/IPV6`, `APN roaming protocol` field with `IPV4`, and the other fields as default.
Turn on/off airplane mode, and you should be able to make a video call now.

## Enable VoLTE/ViLTE when Building an Android Image

When building Android from source, you should be able to do similar modifications to your built images.
I've successfully done this with LineageOS, but am still working to port to AOSP.
For LineageOS, the steps are quite simple.
First, modify the `device.mk` file of your Pixel model and add the above system properties.
Next, modify the `mbn_sw.txt` in the `vendor` folder (LineageOS will extract this file from your device as well before building).
