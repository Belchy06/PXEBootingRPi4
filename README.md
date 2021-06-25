# PXEBootingRPi4
## Last tested: 25/06/2021
This guide is based off of: https://www.reddit.com/r/raspberry_pi/comments/l7bzq8/guide_pxe_booting_to_a_raspberry_pi_4/ written by u/Offbeatalchemy, updated with some extra information obtained from the troubles I had following the guide.

In this guide, our server will double as both the DHCP and our TFTP server.  
This guide also uses Raspbian Lite images to boot the client pi. There's no reason any other ARM based Linux distro won't work, I just haven't personally tried them.

# Assumptions
This guide works for networks where only the server and pi are on the network. For larger networks, you may experience interference from other routers with DHCP enabled. Disable the DHCP feature of these routers and you should be good to go! 


# Server prep
This guide uses a Ubuntu 20.04.2LTS virtual machine running on VirtualBox as the server, although you can use the host just fine.
A virtual disk size of 25GB proves sufficient, I tested 20GB and that was not enough space. Also, ensure that you setup the VM's network adapter to be the `bridged adapter` and choose the name of the adapter you're using in the next dropdown.
Using a virtual machine ensures a fresh install. If you run into software conflicts, you're on your own.

First, install the needed programs:
```
sudo apt update
sudo apt install nfs-kernel-server kpartx unzip -y
```
Second, we'll ceate a few directories for hosting the boot files. These files need to be created from the very root of the files system. This can be achieved by running `cd ../..`. Double check you are in the correct directory by running `ls`, you should see around 19 results with folders such as `bin`, `dev`, `lib`, etc... 
```
sudo mkdir /srv/tftpboot
sudo mkdir /srv/nfs
```

Thirdly, install `dnsmasq`. This package will listen for PXE requests and handle TFTP.
```
sudo apt update
sudo apt install dnsmasq -y
```
It is normal to see an error after installing dnsmasq. Ubuntu 18.04+ comes with `systemd-resolve` which binds to port 53 and therefore conflicts with the `dnsmasq` port. Run the following commands to disable the `systemd-resolve` service:
```
sudo systemctl disable systemd-resolved
sudo systemctl stop systemd-resolved
```
also, remove the symlinked `resolv.conf` file:
```
ls -lh /etc/resolv.conf
sudo rm /etc/resolv.conf
```
then, create a new `resolv.conf` file:
```
echo nameserver 8.8.8.8 | sudo tee /etc/resolv.conf
```
and finally restart the `dnsmasq` service:
```
sudo systemctl restart dnsmasq
```

Next, we'll need to edit the `dnsmasq.conf` file. Start by finding your VM's external IP address with `hostname -I`. In this case my server IP is `10.115.11.182`. Any time you see this value make sure to subsitute it with your IP.
Notice how the first three octets of the `dhcp-range` match that of the server IP:
```
cat > /etc/dnsmasq.conf << EOF
dhcp-range=10.115.11.0,10.115.11.254,12h
log-dhcp
enable-tftp
tftp-root=/srv/tftpboot
pxe-service=0,"Raspberry Pi Boot"
dhcp-boot=pxelinux.0,10.115.11.182
EOF
```
and then restart `dnsmasq`:
```
sudo systemctl restart dnsmasq
```

# Client prep - readying the client pi for network boot
Now, we need to get the client pi ready to pxe boot by enabling network booting fallback. But first, we need to make sure the firmware is up to date:
```sudo apt update && sudo apt upgrade -y```

Then open raspi-config

```sudo raspi-config```

Navigate through to and enable
```
Advanced Options > Boot Order > Network Boot
```
Before you reboot the client pi, you'll need the serial number for this pi. You'll need this for later, note that the serial should be 8 characters
```
cat /proc/cpuinfo | grep Serial | awk -F ': ' '{print $2}' | tail -c 9
```

You can now reboot the pi without an SD card and will bring you to a boot screen that will loop while looking from a PXE server to boot from
```
sudo reboot
```

# Setting up the image
Back on the Ubuntu VM now. These next few steps are much easier if you're root:
```
sudo -s
```
Make a temporary area for the files
```
mkdir /tmp/pxestuff
cd /tmp/pxestuff
```
Download and unzip the latest raspbian lite image
```
wget -O raspbian_lite_latest.zip https://downloads.raspberrypi.org/raspbian_lite_latest
unzip raspbian_lite_latest.zip
```

Mount the partitions from the image
```
kpartx -a -v *.img
mkdir {bootmnt,rootmnt}
```
The output from this will look something like:
```
add map loop5p1 (253:0): 0 523288 linear 7:5 8192
add map loop5p2 (253:1): 0 3080192 linear 7:5 532480
```
Take note of the value after `map`, you'll need this in the next step. Then:
```
mount /dev/mapper/loop5p1 bootmnt/
mount /dev/mapper/loop5p2 rootmnt/
```
Use the serial number we noted from the client pi along with an arbitrary name for the pi and the IP of Ubuntu VM, initialize some vars for use in the following steps:
```
PI_SERIAL=12345678

KICKSTART_IP=10.115.11.182

PI_NAME=NameOfPi
```

Copy the image unique to this pi and updates the bootfiles and firmware
```
mkdir -p /srv/nfs/${PI_NAME}
mkdir -p /srv/tftpboot/${PI_SERIAL}
cp -a rootmnt/* /srv/nfs/${PI_NAME}
cp -a bootmnt/* /srv/nfs/${PI_NAME}/boot/
rm /srv/nfs/${PI_NAME}/boot/start4.elf
rm /srv/nfs/${PI_NAME}/boot/fixup4.dat
wget https://github.com/Hexxeh/rpi-firmware/raw/stable/start4.elf -P /srv/nfs/${PI_NAME}/boot/
wget https://github.com/Hexxeh/rpi-firmware/raw/stable/fixup4.dat -P /srv/nfs/${PI_NAME}/boot/
```
Update the mounts on the server so the pi can grab the files
```
echo "/srv/nfs/${PI_NAME}/boot /srv/tftpboot/${PI_SERIAL} none defaults,bind 0 0" >> /etc/fstab
echo "/srv/nfs/${PI_NAME} *(rw,sync,no_subtree_check,no_root_squash)" >> /etc/exports
mount /srv/tftpboot/${PI_SERIAL}/
touch /srv/nfs/${PI_NAME}/boot/ssh
sed -i /UUID/d /srv/nfs/${PI_NAME}/etc/fstab
echo "console=serial0,115200 console=tty root=/dev/nfs nfsroot=${KICKSTART_IP}:/srv/nfs/${PI_NAME},vers=3 rw ip=dhcp rootwait elevator=deadline" > /srv/nfs/${PI_NAME}/boot/cmdline.txt
```

Clean up
```
systemctl restart rpcbind
systemctl restart nfs-server
umount bootmnt/
umount rootmnt/
rm /tmp/pxestuff -r
```

# Test the image
On the server, run:
```
journalctl -xefu dnsmasq
```
to view the output of dnsmasq, plug in your pi and it should boot into raspbian lite.

It may restart more than once, especially between updates, that's normal.
