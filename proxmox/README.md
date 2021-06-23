

### Installation and Setup
Download the iso installer from https://www.proxmox.com/en/downloads

Use the raspberry pi imager program to copy it to usb/external disk https://www.raspberrypi.org/software/

Ideally have two nics in the system and addtional storage outside of the disk proxmox runs on

On w530 choose the en1000 device as the management network. However, if the other nic has an IP you can access the proxmox console from it

Proxmox web console is located at https://<IP>:8006


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
