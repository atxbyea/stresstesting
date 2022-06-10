#Installing host

##Debian\Ubuntu based install of tftpd \ apache2 \ dhcp-server 

apt install isc-dhcp-server tftpd-hpa syslinux-efi syslinux-common apache2 sshpass debootstrap

##configuring the IP address of the pxe adapter

`nano /etc/netplan/01-netcfg.yaml`

```yaml
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
    yourIF:
      dhcp4: false
      addresses: [10.1.0.1/20]
```

`netplan apply`


##create folders

```bash
mkdir /tftpboot
mkdir /tftpboot/image
mkdir /tftpboot/ubuntu
mkdir /tftpboot/runconfirm
mkdir /tftpboot/image/casper
mkdir /tftpboot/pxelinux.cfg

cd /usr/lib/syslinux/modules/efi64
cp ldlinux.e64 /tftpboot
cp {libutil.c32,menu.c32} /tftpboot


cd /tmp
wget https://mirrors.edge.kernel.org/pub/linux/utils/boot/syslinux/Testing/6.04/syslinux-6.04-pre1.tar.gz
tar -zxvf syslinux-6.04-pre1.tar.gz
cp /tmp/syslinux-6.04-pre1/efi64/efi/syslinux.efi /tftpboot
```


##configure tftpd
`nano /etc/default/tftpd-hpa`

```
TFTP_DIRECTORY="/tftpboot"
TFTP_ADDRESS="10.1.0.1:69"
```

`systemctl restart tftpd-hpa`

##configure dhcp server

`nano /etc/dhcp/dhcpd.conf`

```
subnet 10.1.0.0 netmask 255.255.240.0 {
        range 10.1.1.1 10.1.15.200;
        option routers 10.1.0.1;
        default-lease-time 3600;
        max-lease-time 86400;
        next-server 10.1.0.1;
        option bootfile-name "syslinux.efi";
}
authoritative;
```

`nano /etc/default/isc-dhcp-server`

`INTERFACESv4="yourIF"`

`systemctl restart isc-dhcp-server`

##copy out netboot kernel\initrd
```
cd /tmp
wget http://releases.ubuntu.com/focal/ubuntu-20.04.4-live-server-amd64.iso
mount ubuntu-20.04.4-live-server-amd64.iso /mnt
cp /mnt/casper{vmlinuz,initrd} /tftpboot/images/casper
```
##configure pxe boot menu

`nano /tftpboot/pxelinux.cfg/default`

```
UI menu.c32
PROMPT 1
TIMEOUT 1
LABEL Stresslinux Net Boot
         MENU LABEL EirikZ Stress Your DC
         KERNEL http://10.1.0.1/image/casper/vmlinuz
         APPEND root=/dev/nfs initrd=http://10.1.0.1/image/casper/initrd nfsroot=10.1.0.1:/tftpboot/ubuntu ip=dhcp ro boot=nfs nfsboot=nfs   
         TEXT HELP
                 It's getting hot in here
         ENDTEXT
```

##configure apache2

`nano /etc/apache2/sites-enabled/000-default.conf`

`DocumentRoot /tftpboot`

`nano /etc/apache2/apache2.conf`

```
<Directory /tftpboot>
	Options Indexes FollowSymLinks
	AllowOverride None
	Require all granted
</Directory>
```

`apache2ctl restart`


##configure NFS exports

`nano /etc/exports`

```
/home           10.1.0.0/20(rw,no_root_squash,async,insecure)
/opt            10.1.0.0/20(rw,no_root_squash,async,insecure)
/tftpboot/ubuntu 10.1.0.0/20(rw,no_root_squash,async,insecure)
/tftpboot/runconfirm 10.1.0.0/20(rw,no_root_squash,async,insecure)
```

Reload NFS exports

`exportfs -rv`



##creating the nfs diskless root

`debootstrap --arch=amd64 --variant=minbase focal /tftpboot/ubuntu http://archive.ubuntu.com/ubuntu`

Then enter the chroot

```
mount --bind /dev /tftpboot/ubuntu/dev
mount --bind /run /tftpboot/ubuntu/run
chroot /tftpboot/ubuntu
mount none -t proc /proc
mount none -t sysfs /sys
mount none -t devpts /dev/pts
export HOME=/root
export LC_ALL=C
```

Populate the sources
```
cat <<EOF > /etc/apt/sources.list
deb http://us.archive.ubuntu.com/ubuntu/ focal main restricted universe multiverse 
deb-src http://us.archive.ubuntu.com/ubuntu/ focal main restricted universe multiverse
deb http://us.archive.ubuntu.com/ubuntu/ focal-security main restricted universe multiverse 
deb-src http://us.archive.ubuntu.com/ubuntu/ focal-security main restricted universe multiverse
deb http://us.archive.ubuntu.com/ubuntu/ focal-updates main restricted universe multiverse 
deb-src http://us.archive.ubuntu.com/ubuntu/ focal-updates main restricted universe multiverse    
EOF
```

Install software and do some local changes

```
apt-get update
apt install -y libterm-readline-gnu-perl systemd-sysv 
dbus-uuidgen > /etc/machine-id
ln -fs /etc/machine-id /var/lib/dbus/machine-id
dpkg-divert --local --rename --add /sbin/initctl
ln -s /bin/true /sbin/initctl
apt install -y stress cron dmidecode ipmitool nfs-common openssh-server ubuntu-standard casper lupin-casper discover os-prober network-manager resolvconf net-tools locales grub-common grub-pc grub-pc-bin grub2-common vim nano less curl apt-transport-https
apt install -y --no-install-recommends linux-generic
dpkg-reconfigure locales
select whatever you want
dpkg-reconfigure resolvconf
```

Populate networkmanager

```
cat <<EOF > /etc/NetworkManager/NetworkManager.conf
[main]
rc-manager=resolvconf
plugins=ifupdown,keyfile
dns=dnsmasq
[ifupdown]
managed=false
EOF
```

Edit crontab to do 
1. Start stress testing on boot
2. Turn on UID of chassis (should work for Dell and HPE)
3. Append system serial number to a NFS share so you know what servers have run

```
crontab -e

@reboot /usr/bin/stress -c 8 -i 8 -m 8
@reboot /usr/bin/ipmitool chassis identify force
@reboot /usr/bin/dmidecode -s system-serial-number >> /mnt/runconfirm/servers.txt
```

`mkdir /mnt/runconfirm`

Edit fstab to accomodate NFS boot

```
proc            /proc           proc    defaults        0       0
/dev/nfs        /               nfs     defaults        1       1
none            /tmp            tmpfs   defaults        0       0
none            /var/tmp        tmpfs   defaults        0       0
10.1.0.1:/home /home          nfs     defaults        0       0
10.1.0.1:/opt /opt            nfs     defaults        0       0
10.1.0.1:/tftpboot/runconfirm	/mnt/runconfirm	nfs	defaults	0	0
```


Reconfigure and enable ssh

```
dpkg-reconfigure network-manager

chmod a+r /boot/vmlinuz*
systemctl enable ssh
```

Edit kernel posix

`nano /etc/kernel/postinst.d/chmod-vmlinuz`
```
#!/bin/sh -e

chmod 0644 /boot/vmlinuz-*
```
`chmod a+x /etc/kernel/postinst.d/chmod-vmlinuz`

Edit initramfs.conf to load netboot modules

`nano /etc/initramfs-tools/initramfs.conf`
`MODULES=netboot`

Recreate initramfs

`update-initramfs -u`

Change root password

`passwd`

Cleanup and leaving chroot

```
truncate -s 0 /etc/machine-id
rm /sbin/initctl
dpkg-divert --rename --remove /sbin/initctl

apt-get clean
rm -rf /tmp/* ~/.bash_history
umount /proc
umount /sys
umount /dev/pts
export HISTSIZE=0
exit

sudo umount /tftpboot/ubuntu/dev
sudo umount /tftpboot/ubuntu/run
```



