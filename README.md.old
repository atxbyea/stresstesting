# stresstesting
POC for stresstesting large customer installs before handover

- PXE boot server running on ubuntu
- TFTP server running on the same server


POC was done with 2x KVM VMs running in a private bridge, but should be easily reproducable in real hardware

Booting stresslinux
copying ssh keys to all
initiating the stress script for all


Install tftp server and dhcp server

configure the ip of the dhcp interface 

nano /etc/netplan/01-netcfg.yaml


network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: true
      dhcp6: false
      optional: true
      nameservers:
        addresses: [4.2.2.1, 4.2.2.2, 208.67.220.220]

    eth2:
      dhcp4: no
      addresses: [192.168.10.123/24]




apt install -y isc-dhcp-server tftpd-hpa

edit /etc/default/isc-dhcp-server
tell it which interface to present dhcp addresses to
INTERFACESv4="eth2"

edit /etc/dhcp/dhcpd.conf

default-lease-time 600;
max-lease-time 7200;
subnet 192.168.10.0 netmask 255.255.255.0 {
    range 192.168.10.50 192.168.10.100;
    option subnet-mask 255.255.255.0;
    option routers 192.168.10.123;
    option broadcast-address 192.168.10.255;
    filename "pxelinux.0";
    next-server 192.168.10.123;

restart dhcp server

cd /var/lib/tftpboot
wget http://www.stresslinux.org/testing/stresslinux-pxe-sample-config.tar.bz2 
tar xfj stresslinux-pxe-sample-config.tar.bz2

then edit KIWI/config.default to

IMAGE=/dev/ram1;stresslinux.x86_64;1.0.3;192.168.10.123;4096;compressed

then edit pxelinux.cfg/default to


LABEL stresslinux
        MENU LABEL ^Stresslinux x86-64 0.7.177
        KERNEL image/initrd-netboot-suse-leap-42.1.x86_64-2.42.1.kernel.4.4.180-102-default
        APPEND initrd=image/initrd-netboot-suse-leap-42.1.x86_64-2.42.1.gz ramdisk_size=1959936


Then download these images

 wget http://download.obs.j0ke.net/stresslinux:/42.3/images/stresslinux.x86_64-1.0.3-Build20.62.tgz -O /var/lib/tftpboot/image/stresslinux.x86_64-1.0.3-Build20.62.tgz
 cd /var/lib/tftpboot/image
 tar xfz stresslinux.x86_64-1.0.3-Build20.62.tgz
 
 
 
