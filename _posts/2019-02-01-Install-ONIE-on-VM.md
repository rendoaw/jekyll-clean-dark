---
layout: post
comments: true
title: "Install ONIE on VM"
categories: blog
descriptions: Install ONIE on VM
tags: 
  - onie
  - whitebox
date: 2019-02-01T10:39:55-04:01
---


# Install ONIE on VM

Basically, it is the same way as installing ONIE on real hardware. The only difference is, we need to use the KVM version of ONIE recovery ISO image.

Follow this guide for creating ONIE recovery iso for KVM image: https://github.com/opencomputeproject/onie/blob/master/machine/kvm_x86_64/INSTALL
Note: the same ISO will work for VMWare VM as well.

The rest of this post will simply provide some screenshot of ONIE installation process.

### Step 1: create VM

Few thing important:
* make sure the disk controller is SATA. For some reason, if i use IDE or virtio, ONIE installation is stuck.
* Put the ONIE iso file as cdrom drive
* make sure the VM can boot from HDD (primary and CDROM (secondary)
* sample XML file for libvirt
* make sure you have virtual serial console. You can use virsh console <vmname> or like me, i prefer to setup telnet port for VM serial console


```
<domain type='kvm'>
  <name>onie-master-test</name>
  <memory unit='KiB'>4096000</memory>
  <currentMemory unit='KiB'>4096000</currentMemory>
  <vcpu placement='static'>2</vcpu>
  <sysinfo type='smbios'>
    <system>
      <entry name='family'>Virtual Machine</entry>
    </system>
  </sysinfo>
  <os>
    <type arch='x86_64' machine='pc-i440fx-xenial'>hvm</type>
    <boot dev='hd'/>
    <boot dev='cdrom'/>
    <smbios mode='sysinfo'/>
  </os>
  <features>
    <acpi/>
    <apic/>
  </features>
  <cpu mode='host-model'>
    <model fallback='allow'/>
  </cpu>
  <clock offset='utc'>
    <timer name='pit' tickpolicy='delay'/>
    <timer name='rtc' tickpolicy='catchup'/>
    <timer name='hpet' present='no'/>
  </clock>
  <on_poweroff>destroy</on_poweroff>
  <on_reboot>restart</on_reboot>
  <on_crash>destroy</on_crash>
  <devices>
    <emulator>/usr/bin/qemu-system-x86_64</emulator>
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2' cache='directsync'/>
      <source file='/data/kvm/run/onie-master-test.qcow2'/>
      <target dev='sda' bus='sata'/>
      <address type='drive' controller='0' bus='0' target='0' unit='0'/>
    </disk>
    <disk type='file' device='cdrom'>
      <driver name='qemu' type='raw'/>
      <source file='/data/kvm/onie-recovery-x86_64-kvm_x86_64-r0.iso'/>
      <target dev='hdc' bus='ide'/>
      <readonly/>
      <address type='drive' controller='0' bus='1' target='0' unit='0'/>
    </disk>
    <controller type='usb' index='0'>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x01' function='0x2'/>
    </controller>
    <controller type='pci' index='0' model='pci-root'/>
    <controller type='ide' index='0'>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x01' function='0x1'/>
    </controller>
    <controller type='sata' index='0'>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x09' function='0x0'/>
    </controller>
    <interface type='bridge'>
      <source bridge='br1'/>
      <model type='e1000'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x04' function='0x0'/>
    </interface>
    <serial type='tcp'>
      <source mode='bind' host='0.0.0.0' service='40031'/>
      <protocol type='telnet'/>
      <target port='0'/>
    </serial>
    <console type='tcp'>
      <source mode='bind' host='0.0.0.0' service='40031'/>
      <protocol type='telnet'/>
      <target type='serial' port='0'/>
    </console>
    <input type='tablet' bus='usb'/>
    <input type='mouse' bus='ps2'/>
    <input type='keyboard' bus='ps2'/>
    <graphics type='vnc' port='-1' autoport='yes' listen='0.0.0.0'>
      <listen type='address' address='0.0.0.0'/>
    </graphics>
    <video>
      <model type='cirrus' vram='16384' heads='1'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x02' function='0x0'/>
    </video>
    <memballoon model='virtio'>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
    </memballoon>
  </devices>
</domain>

```
  
  
### Launch the VM

If everything right, you will see the GRUB menu below. Choose "Embed ONIE" to install ONIE on the hard drive

```
                             GNU GRUB  version 2.02

 +----------------------------------------------------------------------------+
 | ONIE: Rescue                                                               |
 |*ONIE: Embed ONIE                                                           |
 |                                                                            |
 |                                                                            |
 |                                                                            |
 |                                                                            |
 |                                                                            |
 |                                                                            |
 |                                                                            |
 |                                                                            |
 |                                                                            |
 |                                                                            |
 +----------------------------------------------------------------------------+

      Use the ^ and v keys to select which entry is highlighted.
      Press enter to boot the selected OS, `e' to edit the commands
      before booting or `c' for a command-line.


ONIE: Embedding ONIE ...
Platform  : x86_64-kvm_x86_64-r0
Version   : master-201901271435
Build Date: 2019-01-27T14:35-05:00
Info: Mounting kernel filesystems... done.
Info: BIOS mode: legacy
Running demonstration platform init pre_arch routines...
Running demonstration platform init post_arch routines...
network_driver: Running demonstration pre_init routines...
network_driver: Running ASIC/SDK init routines...
network_driver: Running demonstration post_init routines...
Info: Using eth0 MAC address: 52:54:00:2f:ca:56
Info: eth0:  Checking link... up.
Info: Trying DHCPv4 on interface: eth0
ONIE: Using DHCPv4 addr: eth0: 192.168.1.161 / 255.255.255.0
Starting: klogd... done.
Starting: dropbear ssh daemon... done.
Starting: telnetd... done.
discover: ONIE embed mode detected.  Running updater.
Starting: discover... done.

Please press Enter to activate this console. Info: eth0:  Checking link... up.
Info: Trying DHCPv4 on interface: eth0
ONIE: Using DHCPv4 addr: eth0: 192.168.1.161 / 255.255.255.0
ONIE: Starting ONIE Service Discovery
Info: Found static url: file:///lib/onie/onie-updater



ONIE: Executing installer: file:///lib/onie/onie-updater
Verifying image checksum ... OK.
Preparing image archive ... OK.
Success: test_install_sharing mechanism
ONIE: Version       : master-201901271435
ONIE: Architecture  : x86_64
ONIE: Machine       : kvm_x86_64
ONIE: Machine Rev   : 0
ONIE: Config Version: 1
ONIE: Build Date    : 2019-01-27T14:35-05:00
Installing ONIE on: /dev/sda
Pre installation hook
Post installation hook
ERROR: Problems accessing sys_eeprom
Notice:  Invalid TLV header found.  Using default contents.
Notice:  Invalid TLV checksum found.  Using default contents.
ERROR: open(/dev/vda): No such file or directory
ONIE: ERROR: Firmware update URL: file:///lib/onie/onie-updater
ONIE: ERROR: Firmware update version: master-201901271435
Failure: Unable to install image: file:///lib/onie/onie-updater
Info: Attempting http://192.168.1.20/onie-updater-x86_64-kvm_x86_64-r0 ...
Info: Attempting http://192.168.1.20/onie-updater-x86_64-kvm_x86_64-r0.bin ...
Info: Attempting http://192.168.1.20/onie-updater-x86_64-kvm_x86_64 ...
Info: Attempting http://192.168.1.20/onie-updater-x86_64-kvm_x86_64.bin ...
Info: Attempting http://192.168.1.20/onie-updater-kvm_x86_64 ...
Info: Attempting http://192.168.1.20/onie-updater-kvm_x86_64.bin ...
Info: Attempting http://192.168.1.20/onie-updater-x86_64-qemu ...
Info: Attempting http://192.168.1.20/onie-updater-x86_64-qemu.bin ...
Info: Attempting http://192.168.1.20/onie-updater-x86_64 ...
Info: Attempting http://192.168.1.20/onie-updater-x86_64.bin ...
Info: Attempting http://192.168.1.20/onie-updater ...
Info: Attempting http://192.168.1.20/onie-updater.bin ...
Info: Attempting http://192.168.1.1/onie-updater-x86_64-kvm_x86_64-r0 ...
...
```

* You can ignore the onie-updater at the end. Basically it is ONIE try to update itself.
  
  
  
### Reboot the VM and install the OS

In the example below, I am installing Openswitch image via ONIE. This step is the same as installing OS on ONIE on real hardware. 

```
                            GNU GRUB  version 2.02

 +----------------------------------------------------------------------------+
 |*ONIE: Install OS                                                           |
 | ONIE: Rescue                                                               |
 | ONIE: Uninstall OS                                                         |
 | ONIE: Update ONIE                                                          |
 | ONIE: Embed ONIE                                                           |
 |                                                                            |
 |                                                                            |
 |                                                                            |
 |                                                                            |
 |                                                                            |
 |                                                                            |
 |                                                                            |
 +----------------------------------------------------------------------------+

      Use the ^ and v keys to select which entry is highlighted.
      Press enter to boot the selected OS, `e' to edit the commands
      before booting or `c' for a command-line.


ONIE: OS Install Mode ...
Platform  : x86_64-kvm_x86_64-r0
Version   : master-201901271435
Build Date: 2019-01-27T14:35-05:00
Info: Mounting kernel filesystems... done.
Info: Mounting ONIE-BOOT on /mnt/onie-boot ...
Info: BIOS mode: legacy
Running demonstration platform init pre_arch routines...
Running demonstration platform init post_arch routines...
Info: Making NOS install boot mode persistent.
Installing for i386-pc platform.
Installation finished. No error reported.
network_driver: Running demonstration pre_init routines...
network_driver: Running ASIC/SDK init routines...
network_driver: Running demonstration post_init routines...
Info: Using eth0 MAC address: 52:54:00:2f:ca:56
Info: eth0:  Checking link... up.
Info: Trying DHCPv4 on interface: eth0
ONIE: Using DHCPv4 addr: eth0: 192.168.1.161 / 255.255.255.0
Starting: klogd... done.
Starting: dropbear ssh daemon... done.
Starting: telnetd... done.
discover: installer mode detected.  Running installer.
Starting: discover... done.

Please press Enter to activate this console. Info: eth0:  Checking link... up.
Info: Trying DHCPv4 on interface: eth0
ONIE: Using DHCPv4 addr: eth0: 192.168.1.161 / 255.255.255.0
ONIE: Starting ONIE Service Discovery

To check the install status inspect /var/log/onie.log.
Try this:  tail -f /var/log/onie.log

NOTICE: ONIE started in NOS install mode.  Install mode persists
NOTICE: until a NOS installer runs successfully.

** Installer Mode Enabled **
ONIE:/ # onie-stop
discover: installer mode detected.
Stopping: discover... done.
ONIE:/ # install_url http://192.168.1.245:8000/images/PKGS_OPX-3.1.0-installer-x
86_64.bin
```



That's it. 



