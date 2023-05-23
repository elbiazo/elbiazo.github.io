---
title: srsRan 4G Setup
date: 2023-05-22
categories: [baseband]
tags: [setup]
---
# Hardware

* BladeRfMicro A4
* [sysmocom sim](https://shop.sysmocom.de/sysmoISIM-SJA2-SIM-USIM-ISIM-Card-10-pack-with-ADM-keys/sysmoISIM-SJA2-10p-adm)
* Card reader [Omnikey CardMan](https://shop.sysmocom.de/Omnikey-CardMan-6121-USB-CCID-interface-2FF-sized/cm6121) or  [MCR3512](https://www.aliexpress.us/item/2251832818004690.html?gatewayAdapt=glo2usa4itemAdapt&_randl_shipto=US)

# Software

* Ubuntu 22.04
* VMware Workstation

# Install Ubuntu in VMware

change usb setting `USB Compatibility` to `USB 3.1`

![](/assets/img/2023-04-25-20-45-36.png)

# Update/Match IMSI with MCC/MNC

pushing mcc mnc and changing the imsi to match
the first five digit of sim card need to match mcc mnc . for us its 001 01 (test network)

# Example Config

```
./pySim-prog.py -p 0 -t sysmoISIM-SJA2 -a 89907613 -x 001 -y 01 -i 001010000052250 -s 8988211000002592504 -o F6CA46797341FC2B2FCA22CF0C165D4B -k 9BA13241D5D4A8B2466BA4EF19A1B971 
```

# Install BladeRF

## Dependencies

Programs will silently fail if you don't have these dependencies

```bash
# deps for common
sudo apt install libusb-1.0.0-dev libusb-1.0.0 build-essential cmake

```

https://github.com/Nuand/bladeRF/tree/master/host

## Testing if bladeRf Works

In terminal, type

```bash
bladeRf-cli -p
```

Should look similar to below

![](/assets/img/2023-04-25-20-46-04.png)


## Fetch the lastest FPGA image
get the fpga [here](https://www.nuand.com/fpga_images/)

I have the BladeRF A4 so I will install it using A4 but replace command below with your own FPGA

```bash
wget https://www.nuand.com/fpga/hostedxA4-latest.rbf # make sure you use your fpga version

bladeRF-cli -L ./hostedxA4-latest.rbf
```

## Fetch the lastest firmware image

get the firmware [here](https://www.nuand.com/fx3_images/)

```bash
wget https://www.nuand.com/fx3/bladeRF_fw_latest.img
bladeRF-cli  -f ./bladeRF_fw_latest.img
```

Then power cycle the SDR to load the firmware

## Check if everything is loaded
```bash
bladeRF-cli -e info -e version
```

![](/assets/img/2023-04-25-20-46-33.png)

# Installing srsRan

We need to compile srsRan from source. Follow this documentation


[https://docs.srsran.com/projects/4g/en/latest/general/source/1_installation.html](https://docs.srsran.com/projects/4g/en/latest/general/source/1_installation.html)


# Updating Config

## Update user_db.csv

Update user_db.csv with your sim card imsi and ki like example `https://github.com/srsran/srsRAN_4G/blob/master/srsepc/user_db.csv.example`


# Running 

Folow this. TLDR you have to [https://docs.srsran.com/projects/4g/en/latest/usermanuals/source/1_setup.html](https://docs.srsran.com/projects/4g/en/latest/usermanuals/source/1_setup.html)

# Diagnostic

To check your mobile card setting on andriod you can dial
`*#*#4636#*#*`

# Can't find Mobile Network?

On your enb.conf for me it was `/root/.config/srsran/enb.conf` look for `dl_earfcn` under `[rf]`. Change values to `800, 1800, 2600` One of those and see if apn will show up. Samsung Basebands are usually very annoying when it comes to these.

For Pixel 7 it was `2600`
