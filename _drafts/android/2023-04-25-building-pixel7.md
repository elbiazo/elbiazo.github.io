---
title: Building Pixel 7 AOSP and Android Kernel
date: 2023-04-25
categories: [android]
tags: [setup]
---

# Overview

Purpose of this blog is to build Pixel 7 AOSP and Kernel

This blog will build `userdebug` build with `hwasan` (hardware address sanitizer). You can check different kind of build variant [here](https://source.android.com/docs/setup/create/new-device#build-variants).

# Hardware

* Pixel (Blog will be using `Pixel 7` codename `Panther`) but should work with different version of pixel with minor tweak

# Software

* AOSP (Android + Precompiled Kernel)
* [Vendor Drivers](https://developers.google.com/android/drivers) (Usually close-source binary such as Modem)
* Android Kernel (Kernel. `Optional` only if you want custom `.config`)

## Finding AOSP, Vendor Driver and Android Kernel Version

You can match BUILD ID with AOSP version
https://www.google.com/search?q=android+code+names&oq=&aqs=edge.0.69i59j0i512l5j0i10i512j69i60l2.1915j0j1&sourceid=chrome&ie=UTF-8

# Before Starting
****
You can skip to `Building AOSP and Flashing` step if you are just looking to enable Address Sanitizer. If you want to enable any other kernel config such as `CONFIG_KCOV`, start from Building Kernel

# Downloading Repos

## Downloading Android TD1A.220804.031
```
mkdir aosp
cd aosp
repo init -u https://android.googlesource.com/platform/manifest -b android-13.0.0_r11
repo sync -c -j`nproc`
```

### Downloading Proprietary Driver for AOSP

Find the version from [here](https://developers.google.com/android/drivers)

Once aosp is all pulled down grab the driver and extract it.

```
# Inside aosp dir
# cd aosp
wget https://dl.google.com/dl/android/aosp/google_devices-panther-td1a.220804.031-0041fc65.tgz
tar xzvf google_devices-panther-td1a.220804.031-0041fc65.tgz
./extract-google_devices-panther.sh
```

## Downloading Kernel
```
mkdir android-kernel
cd android-kernel
repo init -u https://android.googlesource.com/kernel/manifest -b android-gs-pantah-5.10-android13-d1 
repo sync -c -j`nproc`
build/build.sh
```

# Building Kernel

## Update Kernel Config via menuconfig

There are two ways to do this. I recommend using the first method

### Using Fragment and Menuconfig

```
BUILD_CONFIG=private/gs-google/build.config.cloudripper FRAGMENT_CONFIG=./private/gs-google/arch/arm64/configs/cloudripper_gki.fragment ./build/config.sh menuconfig
```


### Updating gki_defconfig (Not Recommended Way)

Update `android-kernel/private/gs-google/arch/arm64/configs/gki_defconfig` with config from below (either `KASAN` or `HWASAN`) with `CONFIG_KCOV`



### Config for Fuzzing with KASAN

Note: Seems like for Pixel 7 HWASAN was already configured as default. I still had to enable `CONFIG_KCOV`. Check the `.config` after doing `menuconfig`

To choose between `HWASAN` or `KASAN` it depends on info below:
[Android 7.0 and higher includes support for building the entire Android platform with ASan at once. (If you're building a release higher than Android 9, HWASan is a better choice.)](https://source.android.com/docs/security/test/asan)

#### Kernel with HWASAN (Hardware Address Sanitizer)

```
# Memory bug detector
CONFIG_KASAN=y
CONFIG_KASAN_HW_TAGS=y
CONFIG_SLUB=y
CONFIG_SLUB_DEBUG=y

CONFIG_KCOV=y
```

#### Kernel with KASAN (Software Based Address Sanitizer)

```
# Memory bug detector
CONFIG_KASAN=y
CONFIG_KASAN_INLINE=y
CONFIG_SLUB=y
CONFIG_SLUB_DEBUG=y

# Coverage collection
CONFIG_KCOV=y
```

## Checking the Config

You can check if you have valid config by running menuconfig or looking at `.config`

`.config` path is  `out/android13-gs-pixel-5.10/private/gs-google/.config`

and 

menuconfig commmand is below

```
BUILD_CONFIG=private/gs-google/build.config.cloudripper FRAGMENT_CONFIG=./private/gs-google/arch/arm64/configs/cloudripper_gki.fragment ./build/config.sh menuconfig
```

## Build Kernel

```bash
# cloudripper is another codename for pixel 7.
# Change it to correct one for your pixel

BUILD_CONFIG=private/gs-google/build.config.cloudripper build/build.sh
```


### (OPTIONAL) Building without LTO

You can disable LTO by setting `FAST_BUILD=1` like below

```
FAST_BUILD=1 BUILD_CONFIG=private/gs-google/build.config.cloudripper build/build.sh
```

## Copy Kernel to AOSP

```
cp $KERNEL/out/android13-gs-pixel-5.10/dist/* $AOSP/device/google/pantah-kernel/
```

# Building AOSP and Flashing

## Build AOSP

[Android 7.0 and higher includes support for building the entire Android platform with ASan at once. (If you're building a release higher than Android 9, HWASan is a better choice.)](https://source.android.com/docs/security/test/asan)

### Building with HWASAN
```
source build/envsetup.sh
lunch aosp_panther-userdebug
export SANITIZE_TARGET=hwaddress
m -j
```

### Building with KASAN

```
source build/envsetup.sh
lunch aosp_panther-userdebug
export SANITIZE_TARGET=address
m -j
```

## Flashing

Get to fast boot by  `adb reboot bootloader` or holding down `power` + `vol down` button for 5 seconds.

## (Optional) Unlocking Flash

If you havn't done this before, unlock the flash
```
fastboot flashing unlock
```

## Flash everything

```
fastboot flashall -w
```

If you get an error try below

```
ANDROID_PRODUCT_OUT=`pwd` fastboot flashall -w
```


# Recovery

Sometimes, phone might get into weird state, such as infinite bootloop or getting stuck on google. Shown on this [fourm](https://forum.xda-developers.com/t/aosp-custom-kernel-development-for-pixel-6a-bluejay-for-android-12l-version.4564549/#post-88440335)

If this happens you can get to bootloader via `adb reboot bootloader` or holding down `power` + `vol down` button for 5 seconds.

Then get the [factory images](https://developers.google.com/android/images) and `flash-all.sh`


# Extra Knowledge

## Updating Kernel Config via Default Config (Not Recommended)

Really this is not recommended because you are changing default `.config`. It will always use this config and it might be just wrong. It is just here as reminder.


# Resources

## Pixel 3 Build

* https://michael137.github.io/2020-04-16-android-kernel/
	* https://groups.google.com/g/android-building/c/ou630PviyDc

## Pixel 4 Build

* https://junsun.net/wordpress/2021/02/build-flash-and-un-flash-aosp-image-on-pixel-phones/

## Pixel 6 Build

* https://zhyfeng.github.io/posts/blog-post-2/

## Syzkaller Build

https://source.android.com/docs/core/tests/debug/kasan-kcov?hl=zh-cn
