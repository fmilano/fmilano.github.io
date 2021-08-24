---
layout: post
title:  "Sharing a volume between a PC running Microsoft DOS and a modern Ubuntu server"
date:   2020-01-30 22:51:10 +0200
categories: networking retrocomputing
---
I usually maintain my father’s home network. It is a fairly small network with a couple of GNU/Linux based computers, a switch and a PC running DOS 6.22. This article comes from my failure and following frustration to configure a Samba server supporting SMBv1 in Ubuntu 18.04. This is the first Ubuntu version shipping with SMBv1 disabled by default. I did not want to fallback into a serial port connection to send back and forth files between the DOS PC and the Ubuntu server, I just wanted to share a volume that could be mounted and accessed as a disk drive from the DOS command line. Looking for alternatives to SMBv1 I found EtherDFS. This project allows sharing a volume using just the Ethernet protocol; this design decision greatly simplifies the DOS client program, with the downside of being confined to the LAN to which the computers are connected to.

**EtherDFS** uses a client-server model, running the client as a Terminate and Stay Resident (TSR) in DOS and the server as a daemon in Ubuntu.

## Server configuration

The ethersrv-linux was very easy to build in Ubuntu 18.04. Installing the build-essential package and following the included build instructions is enough for a successful build.

In the documentation it is recommended to create a FAT file system image and mount it somewhere in your system. This command sequence worked fine for that task:

```console
fallocate -l 1024M fat.img
mkfs.msdos fat.img
mount -o loop fat.img /mnt/fat
```

If you want to mount this image on startup you have to modify your /etc/fstab file.

The ethersrv-linux program accepts two parameters:

The name of the Ethernet interface in which it is going to listen to for requests and issue responses. The name of this interface could be fine running ifconfig.
The path of the directory you want to share as a network volume. In our case this is /mnt/fat, since that is the path in which we mounted the FAT image.
I wanted to run ethersrv-linux on startup, so I created a systemd service to manage the daemon. You can run ethersrv-linux at least once from the command line, so you can see in the output which drive letter is used to mount this network volume (by default it will be C:) and which MAC address you should use in the client application.

## Client configuration

The client side was a little more complicated to configure. ETHERDFS.EXE needs a packet driver for accessing the network interface card. In my particular case this was complicated, since the DOS PC is a Compaq Prolinea Net1/25s with an Integrated NetFlex-2 ENET network card. I have not found a packet driver for this particular card. There is a NDIS version 2 driver, that I had installed in this machine with the MS LAN Manager. All I needed then was a wrapper of a NDIS driver using the packet driver interface. At first I doubted if something like that existed, but googling and reading a bit I have found that this kind of wrappers have its own generic name: shim.

The shim I have used to convert from NDIS to packet driver is DIS_PKT. This shim is installed as a driver in CONFIG.SYS and then configured in PROTOCOL.INI:

```console
CONFIG.SYS
...
DEVICE=C:\LANMAN.DOS\DRIVERS\PROTMAN\PROTMAN.DOS /i:C:\LANMAN.DOS
DEVICE=C:\LANMAN.DOS\DRIVERS\ETHERNET\EPRO\EPRO.DOS
DEVICE=C:\LANMAN.DOS\DIS_PKT9.DOS
DEVICE=C:\LANMAN.DOS\DRIVERS\PROTOCOL\tcpip\tcpdrv.dos /i:C:\LANMAN.DOS
DEVICE=C:\LANMAN.DOS\DRIVERS\PROTOCOL\tcpip\nemm.dos
...
```
The PROTMAN.DOS, EPRO.DOS, tcpdrv.dos and nemm.dos were installed during the LAN Manager setup. I have added the DIS_PKT9.DOS.

```console
PROTOCOL.INI
...
[PKTDRV]
    DRIVERNAME = PKTDRV$
    BINDINGS = EPRO_NIF
    INTVEC = 0x60
    CHAINVEC = 0x66
    NOVELL = NO

[EPRO_NIF]
;Protocol.Ini section for the Compaq Integrated NetFlex-2 ENET
    DRIVERNAME = EPRO$
    IOADDRESS = 0x300
```

The EPRO_NIF tag was created during LAN Manager setup. I have added the PKTDRV tag, referencing the EPRO_NIF tag. I have left the default interrupt number (0x60) for the packet driver.

The last step is to actually run EtherDFS on startup.

```console
@REM ==== LANMAN 2.2a == DO NOT MODIFY BETWEEN THESE LINES == LANMAN 2.2a ====
SET PATH=C:\LANMAN.DOS\BASIC;%PATH%
C:\LANMAN.DOS\DRIVERS\PROTOCOL\tcpip\umb.com
LOAD TCPIP
SOCKETS
NET START WORKSTATION COMPAQ
@REM ==== LANMAN 2.2a == DO NOT MODIFY BETWEEN THESE LINES == LANMAN 2.2a ====
ETHERDFS 00:18:8B:DF:1E:46 C-F /Q
```

I have found that the NET START command was necessary, it activates the NDIS driver. I have added the last line, running ETHERDFS.EXE and pointing it to the MAC address provided by the server daemon.

## Conclusion

When testing this new configuration I considered measuring the file transfer speed between the two hosts. In the end I decided not to do it, mainly because this particular link is used to transfer relatively short (less than 1 MB) text files, and for this use case I have not noticed any (subjective) long latency or transfer times.

