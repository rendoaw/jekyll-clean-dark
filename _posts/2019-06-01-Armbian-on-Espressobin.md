---
layout: post
comments: true
title: "Armbian on Espressobin v5"
categories: blog
descriptions: My personal attempt of installing armbian on Espressobin v5
tags: 
  - espressobin
  - armbian
  - u-boot
date: 2019-06-01T10:39:55-04:01
---



I have been using espressobin v5 with archlinux for a bit more than a year. I opted for archlinux because the instruction was much clearer than ubuntu, and based on some forum, archlinux is more stable. 
I use the espressobin as my home gateway with the following functionalities
NAT gateway
OpenVPN server
Netflow reporter
in theory it support ipt_netflow module, but it was unstable. I was using pmacct instead.
SNMP daemon for BW monitoring
Docker for running pi-hole

Everything was fine, but i still want to try to get ubuntu or at least debian based linux. I know I can't expect this to be smooth process, so in addition to get a new micro sd card, i also bought a mikrotik RB750GR3 as my temporary home gateway until i am satisfied with my new espressobin setup. 


As my first attempt, i am following [http://wiki.espressobin.net/tiki-index.php?page=Boot+from+removable+storage+-+Ubuntu](http://wiki.espressobin.net/tiki-index.php?page=Boot+from+removable+storage+-+Ubuntu). The instruction is complete and i can get the ubuntu 16.04 installed and running in my espressobin. As mentioned in the instruction, ip routing is not enabled by default, which is fine, i guess. But, for unknown reason, i could not even connect it to the network. I tried to put static IP or DHCP on any interface (wan, lan, eth0, ..), then after few minutes, the system crashed. 
Since i don't have any idea how to fix it, i started to check armbian distribution from [https://www.armbian.com/espressobin/](https://www.armbian.com/espressobin/)

Armbian instruction looks scary at the beginning since it involves updating the u-boot. I decided to try, and here are the steps.


### Find out the exact model

U-Boot image are very hardware specific. The correct image name will have the following naming format: 

```
flash-image-(ddr3|ddr4)-MEM-RAM_CHIPS-CPU_DDR.bin
```

From the existing u-boot output via serial console below, i know that my board has 1GB RAM, CPU clock 1000MHz DDR clock 800MHz.
Based on other source, i also know that espressobin v5 is using DDR3. 


```
Model: Marvell Armada 3720 Community Board ESPRESSOBin
       CPU     1000 [MHz]
       L2      800 [MHz]
       TClock  200 [MHz]
       DDR     800 [MHz]
DRAM:  1 GiB
```

Now, the only missing part is whether my board has 1 or 2 RAM chips. Unfortunately, v5 picture from [http://wiki.espressobin.net/tiki-index.php?page=Ports+and+Interfaces#ESPRESSObin_v5_and_older_revisions](http://wiki.espressobin.net/tiki-index.php?page=Ports+and+Interfaces#ESPRESSObin_v5_and_older_revisions) does not mention which one is the RAM chip clearly, unlike the v7 picture. 
By comparing v5 and v7, and looking at my board, i am quite sure i have 1 RAM Chip, so in this case i need to download this file to USB drive [https://dl.armbian.com/espressobin/u-boot/flash-image-ddr3-1g-1cs-1000_800.bin](https://dl.armbian.com/espressobin/u-boot/flash-image-ddr3-1g-1cs-1000_800.bin)


### Update the u-boot

Connect the USB drive to USB3 port. Make sure the USB drive is formatted as FAT. Then running the following command 


```
bubt flash-image-ddr3-1g-1cs-1000_800.bin spi usb
```


### Reboot 

* Unplug USB
* power-cycle the board 



### Check the new u-boot version

From the following output, now i have 2018 version of U-boot

```
TIM-1.0
WTMI-devel-18.12.1-e6bb176
WTMI: system early-init
SVC REV: 3, CPU VDD voltage: 1.143V
NOTICE:  Booting Trusted Firmware
NOTICE:  BL1: v1.5(release):1f8ca7e (Marvell-devel-18.12.2)
NOTICE:  BL1: Built : 16:22:05, May 21 2019
NOTICE:  BL1: Booting BL2
NOTICE:  BL2: v1.5(release):1f8ca7e (Marvell-devel-18.12.2)
NOTICE:  BL2: Built : 16:22:06, May 21 2019
NOTICE:  BL1: Booting BL31
NOTICE:  BL31: v1.5(release):1f8ca7e (Marvell-devel-18.12.2)
NOTICE:  BL31: Built : 16:2

U-Boot 2018.03-devel-18.12.3-gc9aa92c-armbian (Feb 20 2019 - 09:45:04 +0100)

Model: Marvell Armada 3720 Community Board ESPRESSOBin
       CPU     1000 [MHz]
       L2      800 [MHz]
       TClock  200 [MHz]
       DDR     800 [MHz]
DRAM:  1 GiB
Comphy chip #0:
Comphy-0: USB3          5 Gbps
Comphy-1: PEX0          2.5 Gbps
Comphy-2: SATA0         6 Gbps
SATA link 0 timeout.
AHCI 0001.0300 32 slots 1 ports 6 Gbps 0x1 impl SATA mode
flags: ncq led only pmp fbss pio slum part sxs
PCIE-0: Link down
MMC:   sdhci@d0000: 0, sdhci@d8000: 1
Loading Environment from SPI Flash... SF: Detected w25q32dw with page size 256 Bytes, erase size 4 KiB, total 4 MiB
OK
Model: Marvell Armada 3720 Community Board ESPRESSOBin
Net:   eth0: neta@30000 [PRIME]
Hit any key to stop autoboot:  0

Marvell>>
```


### Flash the Armbian image to micro sd card

I choose Armbian Stretch image from [https://dl.armbian.com/espressobin/Debian_stretch_next.7z](https://dl.armbian.com/espressobin/Debian_stretch_next.7z) and then use balenaEtcher to flash it. 


### Boot the armbian

[https://www.armbian.com/espressobin/](https://www.armbian.com/espressobin/) has instruction on how to setup the boot environment, but unfortunately it did not work for me. 
After several trial and error, looking at few examples from the armbian/espressobin forum, i managed to get the proper environment


```
Marvell>> env default -a
## Resetting to default environment
Marvell>> setenv fdt_addr 0x6000000
Marvell>> setenv kernel_addr 0x7000000
Marvell>> setenv loadaddr 0x8000000
Marvell>> setenv initrd_size 0x2000000
Marvell>> setenv initrd_addr 0x1100000
Marvell>> setenv scriptaddr 0x6d00000
Marvell>> setenv initrd_image uInitrd
Marvell>> setenv bs 'mmc0'
Marvell>>
Marvell>>
Marvell>> setenv boot_prefixes '/ /boot/'
Marvell>> setenv image_name boot/Image
Marvell>>
Marvell>>
Marvell>> setenv fdt_name boot/dtb/marvell/armada-3720-espressobin.dtb
Marvell>>
Marvell>> setenv bootcmd 'mmc dev 0; ext4load mmc 0:1 $kernel_addr $image_name;ext4load mmc 0:1 $fdt_addr $fdt_name;setenv bootargs $console root=/dev/mmcblk1p1 rw rootwait net.ifnames=0 biosdevname=0; booti $kernel_addr - $fdt_addr'
Marvell>> saveenv
Saving Environment to SPI Flash... SF: Detected w25q32dw with page size 256 Bytes, erase size 4 KiB, total 4 MiB
Erasing SPI flash...Writing to SPI flash...done
OK

Marvell>> run bootcmd
switch to partitions #0, OK
mmc0 is current device
16421376 bytes read in 709 ms (22.1 MiB/s)
8942 bytes read in 18 ms (484.4 KiB/s)
## Flattened Device Tree blob at 06000000
   Booting using the fdt blob at 0x6000000
   Using Device Tree in place at 0000000006000000, end 00000000060052ed

Starting kernel ...

[    0.000000] Booting Linux on physical CPU 0x0000000000 [0x410fd034]
[    0.000000] Linux version 4.19.20-mvebu64 (root@armbian.com) (gcc version 7.2.1 20171011 (Linaro GCC 7.2-2017.11)) #5.75 SMP PREEMPT Fri Feb 8 09:54:18 CET 2019
[    0.000000] Machine model: Globalscale Marvell ESPRESSOBin Board
[    0.000000] earlycon: ar3700_uart0 at MMIO 0x00000000d0012000 (options '')
[    0.000000] bootconsole [ar3700_uart0] enabled
[    0.000000] efi: Getting EFI parameters from FDT:
[    0.000000] efi: UEFI not found.
[    0.000000] cma: Reserved 16 MiB at 0x000000003f000000
[    0.000000] NUMA: No NUMA configuration found
[    0.000000] NUMA: Faking a node at [mem 0x0000000000000000-0x000000003fffffff]
[    0.000000] NUMA: NODE_DATA [mem 0x3efea240-0x3efeb9ff]



[  OK  ] Started Authorization Manager.
[  OK  ] Started Getty on tty1.
[  OK  ] Started Serial Getty on ttyMV0.
[  OK  ] Reached target Login Prompts.
[  OK  ] Started LSB: Start NTP daemon.
[  OK  ] Reached target Multi-User System.
[  OK  ] Reached target Graphical Interface.
         Starting Update UTMP about System Runlevel Changes...
[  OK  ] Started Update UTMP about System Runlevel Changes.

Debian GNU/Linux 9 espressobin ttyMV0

espressobin login: root
Password:
You are required to change your password immediately (root enforced)
Changing password for root.
(current) UNIX password:


espressobin login: root
Password:
You are required to change your password immediately (root enforced)
Changing password for root.
(current) UNIX password:
Enter new UNIX password:
Retype new UNIX password:
 _____                                   _     _
| ____|___ _ __  _ __ ___  ___ ___  ___ | |__ (_)_ __
|  _| / __| '_ \| '__/ _ \/ __/ __|/ _ \| '_ \| | '_ \
| |___\__ \ |_) | | |  __/\__ \__ \ (_) | |_) | | | | |
|_____|___/ .__/|_|  \___||___/___/\___/|_.__/|_|_| |_|
          |_|

Welcome to ARMBIAN 5.75 stable Debian GNU/Linux 9 (stretch) 4.19.20-mvebu64
System load:   0.05 0.14 0.08   Up time:       5 min
Memory usage:  7 % of 990MB     IP:            192.168.1.104
Usage of /:    2% of 58G

[ General system configuration (beta): armbian-config ]

New to Armbian? Check the documentation first: https://docs.armbian.com


Thank you for choosing Armbian! Support: www.armbian.com

Creating a new user account. Press <Ctrl-C> to abort
```


### Result so far

* During the first boot, armbian will do the resizing to expand its filesystem and make it the same as the maximum sd card capacity. 
* During the first login, it will ask you to change root password and setup a non-root user. 


### TO DO

* It seems that ipt_netflow is not enabled in the kernel. I can use pmaacct again, but maybe i'll try to recompile the kernel to see what happen.
* So far, it has been running for more than a day. No major issue, except, there was a single occurance where the OS is completely hang. Serial console was not responding either. I can't reproduce it yet.


So, let see how it goes, and if everything seems ok and stable, i may put it as my main home gateway again.




