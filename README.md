# Google Fiber 2Gbps Technicolor Gateway bypass instructions:

## Overview

The general gist of it is this: You need something that can negotiate GPON SFP at 2.5Gbps. Unfortunately, the list of devices (that I know of) that can do something like this are few and far between. Most SFP+ supported devices will only negotiate fiber at either 1Gbps or 10Gbps, let alone getting it support GPON SFP in general. Mikrotik's, as far as I know, don't support GPON SFP at all. 

You basically have 2 choices:

1) You can use a supported switch, like the EdgeSwitch ES-XG-16 as the EdgeSwitches support running SFP+ at 2.5Gbps and 5Gbps negotiation speed. Unifi devices, as far as I know, do not. The UDM Pro is included in that "do not" category. Ubiquiti may enable/add support eventually, but as of 02/25/2021, it does not work. You can use it, but it will default to 1Gbps negotiation.

2) This is the route I went, and what I'll be describing here. You can use a supported NIC that allows for negotiation of GPON SFP modules specifically at 2.5Gbps. This will require some firmware modification and patched drivers, and, from what I can tell, is supported via pfsense or opnsense for now. This seems to be working on pfsense 2.4.x/2.5, as well as opnsense 20.7 and 21.1.2 (I tested on 21.1.2)

## Background

A general DSLReports thread (https://www.dslreports.com/forum/r32230041-Internet-Bypassing-the-HH3K-up-to-2-5Gbps-using-a-BCM57810S-NIC) goes over in much greater detail how this process works, but I'll do a simplified version based on number 2:

## The Process

The path I chose required I purchase a specific NIC, a Broadcom BCM57810S-based NIC. This comes in many flavors, the one I purchased was new, from Amazon for about $125 (https://smile.amazon.com/gp/product/B06XHGFD69). It comes with a high and low-profile adapter so it can fit in any case. You have other options such as a Dell Y40PH as well as a Nokia G-010S-A (highly not recommended due to some link issues). I chose the 10GTek from Amazon mainly because it doesn't have a fan and required no weird soldering or anything to get it working. YMMV with this depending on the card chosen.

### Required Files

Before you start, you'll need a few utilities:

1) Patched if_bxe.ko drivers (unsure if necessary past FreeBSD 12-based pf firewalls)
2) DOS-based diagnostic tool for the BCM57810S-based NIC

These files can be downloaded via this repo, anything else necessary will be updated and uploaded accordingly. For now, this is all I needed to get this working.

### Modifying the Card

1) Create a DOS bootable USB drive with a tool like Rufus and copy the entire NX2_ev.zip contents to the base of the USB.
2) Plug the card into the system (I recommend unplugging all other PCI devices while you're making these changes) and boot to the USB (non-UEFI).
3) When you get to the command prompt, run the following command: `ediag.exe -b10eng`
4) That should open up the diagnostic tool in engineering mode, and it should show your two ports, take note of the fact both of them, somewhere in the output note "10G" for 10 Gigabit
5) It doesn't seem like you'll be able to run any commands here, depending on your text buffer, but type the following commands in and run them, hitting enter after each one:

**NOTE**: I recommend only modifying *one* port on the card, leaving the other one as a potential LAN output at full 10Gbps. Of course if you'd like to modify both, you can.

**NOTE**: For my 10GTek, device "1" was the port closest to the motherboard while plugged in, device "2" would be the one further away. This may vary depending on the card you're using. If you're only going to modify one and don't really care which port you modify, just use device 1 and just trial and error which one works when you're done.

```
device 1
nvm cfg
6
35=70
36=70
56=6
59=6
save
exit
```

6) After this, you can go back into the tool with `ediag.exe -b10eng` and one of the ports should show "2.5G" instead of "10G". If so, you've successfully modified the card.

At this point, hardware modifications should be done, so now we'll move onto pf/opnsense.

### Modifying pfsense/opnsense driver loading

This may not be necessary for pfsense if it autoloads the if_bxe.ko driver (broadcom), but even if it does, something like this necessarily hurt. As far as I know, this is required for OPNsense.

1) Download the patched if_bxe.ko from here (if you don't trust it, you can compile it yourself with patches detailed in the thread posted above.
2) Unzip and send the modified .ko driver file to your pf/opnsense instance.
3) Back up the current driver in `/boot/kernel/` (easiest way is to run `mv /boot/kernel/if_bxe.ko /boot/kernel/if_bxe.bak` and replace it with the one you downloaded here.
4) If on pfsense, either echo in or manually modify a file, `/boot/loader.conf.local` with the following line:

```
if_bxe_load="YES"
```

5) If on opnsense, do the same as 4, but with the following lines:

```
hw.bxe.interrupt_mode="1"
net.inet.tcp.tso="0"
if_bxe_load="YES"
```

6) Disable Hardware CRC (hardware checksum offload), Hardware TSO (hardware tcp segmentation offload), Hardware LSO (hardware large receive offload) and VLAN Hardware Filtering (pfsense and opnsense)
7) Reboot your box, and redo your interfaces, choosing the bxe# that your 2.5Gbps port was set to for WAN, and the other for LAN (or whatever else you want to use). At this point, you should see the link come "UP" and indicate 2500 speed vs 10000.
8) When the interface and any VLANs associated with it initialize, you will see a bunch of messages about HWFILTER and others "not supported." This is normal and due to the fact those options for hardware offload were disabled, the card is trying to initialize those functions on its own. You can ignore those and be on your way. **You may need to reboot one more time if the interfaces/routes don't necessarily take immediately.**

That's it! It should be set up, no funky VLAN or CoS tagging required, should be just plug and play and you can safely store your Google-provided gateway for it's not needed anymore.

See reddit thread [here](https://old.reddit.com/r/googlefiber/comments/lscvj5/2gbps_gateway_bypass_confirmed_full_speed_working/) for some discussion and other questions.

# Notes

Updates may (and most likely will) replace your if_bxe.ko driver back to what comes with the original install. Be prepared to need to copy it again after you update pf/opnsense. Luckily, the driver still works but will not negotiate 2.5Gbps correctly, so you'll be stuck at 1Gbps until you re-copy the file over and reboot.
