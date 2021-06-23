

### Installation and Setup
Download the iso installer from https://www.proxmox.com/en/downloads

Use the raspberry pi imager program to copy it to usb/external disk https://www.raspberrypi.org/software/

Ideally have two nics in the system and additional storage outside of the disk proxmox runs on.

On w530 choose the en1000 device as the management network. However, if the other nic has an IP you can access the proxmox console from it

Proxmox web console is located at https://<IP>:8006

```
vi /etc/apt/sources.list
# Add:
# not for production use
deb http://download.proxmox.com/debian buster pve-no-subscription
```
```
vi /etc/apt/sources.list.d/pve-enterprise.list
# Comment out:
# https://enterprise.proxmox.com/debian/pve buster pve-enterprise
```
```
apt-get update
apt dist-upgrade
apt install ifupdown2 # this allows you to make network interface changes without reboot
```

Any extra disks you add need to be clean of partitions so use fdisk /dev/<devicename> to clean any partitions. The 'Disklabel type' need to be GPT as well. The disk could be seen with no partitions and type set to DOS.

### Issues
Proxmox fails to install on Inspiron 5500 - try https://www.reddit.com/r/Proxmox/comments/iniknj/unable_to_install_proxmox_ve_62/
```
$ chmod 1777 /tmp
$ apt update
$ Xorg-configure
$ mv /xorg.conf.new /etc/X11/xorg.conf

$ vim /etc/X11/xorg.conf      # update Driver -> "fbdev"
$ startx
```
