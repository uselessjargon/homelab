
## Learning objectives
- initial installation and setup
- running lxc containers
- ha setup with live partition migration
- ceph setup
- rocking homelab server(s) 


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

### LXC Containers
LXC containers require templates to run vs images like regular VMs. You can click on your local storage in proxmox and download templates like ubuntu, centos, alpine etc.   

My first LXC container was based on the centos8 template that I downloaded in the proxmox console. I had a few issues initially. First. I had the container set to use dhcp for the interface but after start it was not showing up on my dhcp server. Eventually, I turned off the proxmox firewall on the interface and the container got an IP address after a restart. Another issue I had was I unable to ssh into the container and kept getting connection refused message. This turned out to be the centos8 image didn't have an ssh server installed. I had to had to attach to the container by running `lxc-attach 101` on the pve node, then install and start the ssh server. After that I was able to login to the container via ssh using the ssh key I uploaded during creation. One other thing of note is you need to put a password in for the container when you create it but that password didn't work for logging into the node as root. It's possible I set it wrong, like had caplocks on, since I was able to set the root password once in the container using the ssh key and then login as root in the pve console for the container.

### Issues
Proxmox fails to install on Inspiron 5500 - run below steps from https://www.reddit.com/r/Proxmox/comments/iniknj/unable_to_install_proxmox_ve_62/
```
$ chmod 1777 /tmp
$ apt update
$ Xorg -configure
$ mv /xorg.conf.new /etc/X11/xorg.conf

$ vim /etc/X11/xorg.conf      # update Driver under device section to "fbdev"
$ startx
```
