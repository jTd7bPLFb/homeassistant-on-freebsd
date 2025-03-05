# HomeAssistant on FreeBSD using bhyve

Run HomeAssistant on FreeBSD using bhyve

**Basic Packages needed**

```
pkg update
pkg install vm-bhyve
pkg install bhyve-firmware
pkg install qemu-tools
```

**Setup and vm-bhyve and firmware for uefi**

```
zfs create -o mountpoint=/vm zroot/vm

pkg install vm-bhyve
sysrc vm_enable="YES"
sysrc vm_dir="zfs:zroot/vm"
vm init
cp /usr/local/share/examples/vm-bhyve/* /vm/.templates/

pkg install bhyve-firmware
pkg install qemu-tools
```

**Create switch if we don't have a bridge already**

*if you have a bridge already, create a manual switch and add it to the bridge*

I am using wlan0 (wifi network) for the switch.
The idea is to passtrhough USB hub with NIC to the HomeAssistant VM and use that exlusively for HomeAssistant
```
vm switch create public
vm switch add public wlan0
```
**Download latest hassos VM image**

bhyve/vm-bhyve supports raw or qcow2 images, so we can use the VM images provided by HomeAssistant team directly
```
vm img https://github.com/home-assistant/operating-system/releases/download/14.2/haos_ova-14.2.qcow2.xz
```

**Create and start the VM**

```
vm create -t debian -c 2 -s 64G -m 2GB -i haos_ova-14.2.qcow2 hass
```

Before starting, edit the /vm/hass/hass.conf file and replace loader="grub" with loader="uefi"

Afterwards, start the vm and confirm it is up and running.\
You can connect to the console of the vm now

```
vm start hass
vm list
vm console hass
```

**Passtrhu**

Passthru on FreeBSD is possible only on controller level. So, if you bave just one USB hub in your PC (laptop, like I did), you need to passthru whole controller.\
Maybe this will change in the future, but for now, this is the only way to do it.

First, find the device you want to passthru.

```
vm passthru
```

In my case, it was\
usb0       0/21/0       Yes          Celeron/Pentium Silver Processor USB 3.0 xHCI Controller

Now, you need to detach the device from the host.\
This is done by putting the appropriate line in /boot/loader.conf file.

This frees the 0/21/0 pci device to from host, and prepares it to be passed thru to the VM.

```
# VM
vmm_load="YES"
pptdevs="0/21/0"
```

Now, we are almost there.\
Edit the /vm/hass/hass.conf file once again, and add
```
passthru0="0/21/0"
``` 
to it.

Reboot the machine to activate the changes.

After machine is back up, make sure the device is managed by ppt
```
vm passthru
```
ppt0       0/21/0       Yes          Celeron/Pentium Silver Processor USB 3.0 xHCI Controller

and now, after you start the VM, you should see the usb contoller (with NIC and USB ports to attach your z-wave/zigbee usb sticks to).

**HASSOS console**

You can access the HASSOS console using 
```
vm console hass
```
and configure basic networking from there.\
Afterwards, you should be able to access the web gui of HomeAssistant and continue the installation as needed from there.



## References
[https://github.com/churchers/vm-bhyve](https://github.com/churchers/vm-bhyve)

[https://blog.david-reid.com/home-assistant-on-freebsd/](https://blog.david-reid.com/home-assistant-on-freebsd/)

[https://github.com/home-assistant/](https://github.com/home-assistant/)
