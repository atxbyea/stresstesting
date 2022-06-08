#Installing host

##Debian\Ubuntu based install of tftpd \ apache2 \ dhcp-server 

apt install isc-dhcp-server tftpd-hpa syslinux-efi syslinux-common apache2 sshpass

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
cd /usr/lib/syslinux/modules/efi64
cp ldlinux.e64 /tftpboot
cp {libutil.c32,menu.c32} /tftpboot
mkdir /tftpboot/pxelinux.cfg

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
        range 10.1.0.100 10.1.15.200;
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

##copy stresslinux into the tftpboot store

```bash
mkdir /tftpboot/image
cd /tftpboot/image
wget http://download.obs.j0ke.net/stresslinux:/42.3/images/stresslinux.x86_64-1.0.3-Build20.62.tgz
tar -zxvf stresslinux.x86_64-1.0.3-Build20.62.tgz
```

##configure pxe boot menu

`nano /tftpboot/pxelinux.cfg/default`

```
UI menu.c32
PROMPT 1
TIMEOUT 1
LABEL Stresslinux Net Boot
         MENU LABEL Stresslinux
         KERNEL http://10.1.0.1/image/initrd-netboot-suse-leap-42.1.x86_64-2.42.1.kernel.4.4.180-102-default
         APPEND initrd=http://10.1.0.1/image/initrd-netboot-suse-leap-42.1.x86_64-2.42.1.gz ramdisk_size=1959936
         TEXT HELP
                 Stresslinux PXE boot
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

##fetch the kiwi pxe files

```bash
cd /tmp
mkdir stresslinuxpxe
cd stresslinuxpxe
wget http://www.stresslinux.org/testing/stresslinux-pxe-sample-config.tar.bz2
tar xjf stresslinux-pxe-sample-config.tar.bz2
mv KIWI /tftpboot
```

`nano /tftpboot/KIWI/config.default`
`IMAGE=/dev/ram1;stresslinux.x86_64;1.0.3;10.1.0.1;4096;compressed`


#create local ssh keys and scripts to copy keys to servers
`ssh-keygen`
`nano ~/.ssh/config`
`StrictHostKeyChecking no`




`nano /tftpboot/copykeys`

```
for server in `cat server.txt`;  
do  
    sshpass -p "stress" ssh-copy-id -i ~/.ssh/id_rsa.pub stress@$server  
done
```

`nano /tftpboot/createlist`

```
seq 0 15 | while read INNER; do
  seq 0 255 | while read OUTER; do
    echo "10.1.${INNER}.${OUTER}" >> /tftpboot/servers.txt
  done
done
```

`nano /tftpboot/executestress`

```
for server in `cat server.txt`;
do
ssh stress@$server "stress -c 32 -i 8 -m 16"
done
```


##create list, copy keys, execute stress
`cd /tftpboot`
`chmod +x {copykeys,createlist,executestress}`
`./createlist`
`./copykeys`
