---
title: Building Pixel 7 AOSP and Android Kernel
date: 2023-04-25
categories: [android]
tags: [setup]
---

## Overview

Purpose of this blog is to build Pixel 7 AOSP and Kernel

This blog will build `userdebug` build with `hwasan` (hardware address sanitizer). You can check different kind of build variant [here](https://source.android.com/docs/setup/create/new-device#build-variants).
It should also work with Pixel 6. Just grab the correct version of AOSP and Kernel.

## Hardware

* Pixel (Blog will be using `Pixel 7` codename `Panther`) but should work with different version of pixel with minor tweak

## Software

* AOSP (Android + Precompiled Kernel)
* [Vendor Drivers](https://developers.google.com/android/drivers) (Usually close-source binary such as Modem)
* Android Kernel (Kernel. `Optional` only if you want custom `.config`)

### Finding AOSP, Vendor Driver and Android Kernel Version

You can match BUILD ID with AOSP version


## Downloading Repos

### Downloading Android TQ2A.230305.008.A1
```
mkdir aosp
cd aosp
repo init -u https://android.googlesource.com/platform/manifest -b android-13.0.0_r33
repo sync -c -j`nproc`
```

#### Downloading Proprietary Driver for AOSP

Find the version from [here](https://developers.google.com/android/drivers)

Once aosp is all pulled down grab the driver and extract it.

```
# Inside aosp dir
# cd aosp
wget https://dl.google.com/dl/android/aosp/google_devices-panther-tq2a.230305.008.a1-1b085d99.tgz
tar xzvf google_devices-panther-tq2a.230305.008.a1-1b085d99.tgz

./extract-google_devices-panther.sh
```

### Downloading Kernel
```
mkdir android-kernel
cd android-kernel
repo init -u https://android.googlesource.com/kernel/manifest -b android-gs-pantah-5.10-android13-qpr2
repo sync -c -j`nproc`
```

## High Level Overview of Building and Flashing custom AOSP and Kernel

1. Build and Flash AOSP with verity and verification disabled. If you don't do this, you won't be able to flash custom kernel.
	2. (Optional) Check if verity and verification is off using `avbctl` command.
2. Build Kernel and Verify Kernel Config
3. Flash Kernel using Manual Method

## Building AOSP and Flashing


### Build AOSP

[Android 7.0 and higher includes support for building the entire Android platform with ASan at once. (If you're building a release higher than Android 9, HWASan is a better choice.)](https://source.android.com/docs/security/test/asan)

`export SANITIZE_TARGET=<option>` are not really necessary since we are going to flash custom kernel anyways but it is good to know.

#### Building with HWASAN
```
source build/envsetup.sh
lunch aosp_panther-userdebug
export SANITIZE_TARGET=hwaddress
m -j
```

#### Building with KASAN

```
source build/envsetup.sh
lunch aosp_panther-userdebug
export SANITIZE_TARGET=address
m -j
```

### Flashing

Get to fast boot by  `adb reboot bootloader` or holding down `power` + `vol down` button for 5 seconds.

### (Optional) Unlocking Flash

If you havn't done this before, unlock the flash
```
fastboot flashing unlock
```

### Flash everything

Make sure you append `--disable-verity --disable-verification` or else you are not disabling verity and you will get stuck in bootloop.

```
fastboot flashall -w --disable-verity --disable-verification 
```

If you get an error try below

```
ANDROID_PRODUCT_OUT=`pwd` fastboot flashall -w --disable-verity --disable-verification 
```

### Checking if verity and verification is disabled

Once Android is booted type this in terminal with ADB cable connected

```
# adb to run as root shell
adb root

adb shell

# Check if verity and verification is disabled
# It should say it is disabled

avbctl get-verity 
avbctl get-verification
```

![](/assets/img/2023-05-02-23-27-41.png)

## Building Kernel

### Update Kernel Config via menuconfig

There are two ways to do this. I recommend using the first method

#### Using Fragment and Menuconfig

```
BUILD_CONFIG=private/gs-google/build.config.cloudripper FRAGMENT_CONFIG=./private/gs-google/arch/arm64/configs/cloudripper_gki.fragment ./build/config.sh menuconfig
```


#### Updating gki_defconfig (Not Recommended Way)

Update `android-kernel/private/gs-google/arch/arm64/configs/gki_defconfig` with config from below (either `KASAN` or `HWASAN`) with `CONFIG_KCOV`



#### Config for Fuzzing with KASAN

Note: Seems like for Pixel 7 HWASAN was already configured as default. I still had to enable `CONFIG_KCOV`. Check the `.config` after doing `menuconfig`

To choose between `HWASAN` or `KASAN` it depends on info below:
[Android 7.0 and higher includes support for building the entire Android platform with ASan at once. (If you're building a release higher than Android 9, HWASan is a better choice.)](https://source.android.com/docs/security/test/asan)

##### Kernel with HWASAN (Hardware Address Sanitizer)

```
# Memory bug detector
CONFIG_KASAN=y
CONFIG_KASAN_HW_TAGS=y
CONFIG_SLUB=y
CONFIG_SLUB_DEBUG=y

# Coverage collection
CONFIG_KCOV=y
```

##### Kernel with KASAN (Software Based Address Sanitizer)

```
# Memory bug detector
CONFIG_KASAN=y
CONFIG_KASAN_INLINE=y
CONFIG_SLUB=y
CONFIG_SLUB_DEBUG=y

# Coverage collection
CONFIG_KCOV=y
```

### Checking the Config

You can check if you have valid config by running menuconfig or looking at `.config`

`.config` path is  `out/android13-gs-pixel-5.10/private/gs-google/.config`

and 

menuconfig commmand is below

```
BUILD_CONFIG=private/gs-google/build.config.cloudripper FRAGMENT_CONFIG=./private/gs-google/arch/arm64/configs/cloudripper_gki.fragment ./build/config.sh menuconfig
```

### Build Kernel

```bash
# cloudripper is another codename for pixel 7.
# Change it to correct one for your pixel

BUILD_CONFIG=private/gs-google/build.config.cloudripper build/build.sh
```


#### (OPTIONAL to Speedup Compile Time) Building without LTO 

You can disable LTO by setting `FAST_BUILD=1` like below

```
FAST_BUILD=1 BUILD_CONFIG=private/gs-google/build.config.cloudripper build/build.sh
```

#### (OPTIONAL to Speedup Compile Time) Building without doing mrproper

You can disable `mrproper` by setting `SKIP_MRPROPER=1` like below

```
SKIP_MRPROPER=1 BUILD_CONFIG=private/gs-google/build.config.cloudripper build/build.sh
```

### Flashing Kernel (Manual Method)
  
vendor_dlkm.img needs to be flashed in fastbootd, while the other images needs to be flashed via fastboot/bootloader.  
  
#### Get to fastbootd
```
# From running phone:  
adb reboot fastboot  
  
# From fastboot/bootloader:  
fastboot reboot fastboot  
```

#### Flash vendor_dlkm.img  
Once in fastbootd:  

```
fastboot flash vendor_dlkm vendor_dlkm.img  
```

then reboot to bootloader

```
fastboot reboot bootloader
```
  
  
#### Flash boot.img, dtbo.img and vendor_kernel_boot.img

Now in fastboot (bootloader) flash boot.img, dtbo.img and vendor_kernel_boot.img  
  
```
fastboot flash boot boot.img  
fastboot flash dtbo dtbo.img  
fastboot flash vendor_kernel_boot vendor_kernel_boot.img
```

#### (Optional Read) Why Manual Method?

Instead of copying the `*.ko` and `Image.lz4` like most of blog tells you to do, you can just flash all kernel and kernel modules using `fastboot` command.
Also I could never get it to work like that. It seems like others are having the same issue as well from this [fourm](https://forum.xda-developers.com/t/aosp-custom-kernel-development-for-pixel-6a-bluejay-for-android-12l-version.4564549/#post-88440335)
Besides, this is way faster anyways if you are doing security research. I think there is two reasons why it didn't work.

1. It was using `vendor_dlkm.img` from vendor driver extract instead of using our compiled one. `vendor_dlkm.img` holds all of vendor driver such as `Mali` and `Exynos` drivers. This will cause linux to not load because of kernel `uname -r` not matching. More on [here](https://source.android.com/docs/core/architecture/kernel/loadable-kernel-modules)
2. I don't think my image had verity and verification disabled. Alot of tutorial such as Magisk and custom rom will tell you to do `fastboot flash vbmeta --disable-verity --disable-verification vbmeta.img` to disable it. But I think this only works with precompiled AOSP + Kernel. Besides doing this when you are doing flashall is way easier and faster.




## Recovery

Sometimes, phone might get into weird state, such as infinite bootloop or getting stuck on google. Shown on this [fourm](https://forum.xda-developers.com/t/aosp-custom-kernel-development-for-pixel-6a-bluejay-for-android-12l-version.4564549/#post-88440335)

If this happens you can get to bootloader via `adb reboot bootloader` or holding down `power` + `vol down` button for 5 seconds.

Then get the [factory images](https://developers.google.com/android/images) and `flash-all.sh`


## Extra Knowledge

## Resources

### Pixel 3 Build

* https://michael137.github.io/2020-04-16-android-kernel/
	* https://groups.google.com/g/android-building/c/ou630PviyDc

### Pixel 4 Build

* https://junsun.net/wordpress/2021/02/build-flash-and-un-flash-aosp-image-on-pixel-phones/

### Pixel 6 Build

* https://zhyfeng.github.io/posts/blog-post-2/

### Syzkaller Build

https://source.android.com/docs/core/tests/debug/kasan-kcov?hl=zh-cn
