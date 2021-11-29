---
title: "Convert VM from OVA to QCOW2 and run on QEMU/KVM"
date: 2021-11-29 12:00 -0700
categories: [Virtualization]
tags: [vmware,qemu,kvm]
excerpt: "Sometimes you don't have the environment to run a virtual appliance the way it is meant to be. Then you need to get creative..."
classes: wide
header:
  image: /assets/images/powershell_header-1280x200.png
  teaser: /assets/images/powershell-250.png
  og_image: /assets/images/powershell-250.png
---

Sometimes you don't have the environment to run a virtual appliance the way it is meant to be. Then you need to get creative. Here I am showing how I took a virtual appliance, that was delivered in VMWare's `OVA` format, and converted it to OpenStack's `QCOW2` format so it would run on QEMU and KVM.

## Situation

I was handed a virtual appliance that had to be run in a customer's OpenStack environment. For licensing reasons there was no way to run the appliance on ESXi in this particular case. Unfortunately this is exactly what the appliance was built for: VMWare's ESXi resp. vSphere, and there was no alternative build process set up yet. This might change in the future, but for now I had to find a way to make this work.

So my plan was to run a virtual machine with CentOS 7 in vSphere (something baremetal would have worked as well, but using our existing infrastructure was more convenient). Then this would run QEMU with KVM and operate the appliance from a converted virtual hard disk file using `libvirtd`.

>This post refers to the steps to make that conversion working on CentOS 7. If you run a different OS you need to adjust the commands accordingly.
{: .notice--warning }

## Preparations

Some things have to be prepared first to make all of that work. The easiest way for me to do this was to set up a system which later runs all the commands and the virtual appliance too. However, there is no need to do all of that on the same system, just go with a setup that fits your circumstances best.

### Install tools

The necessary tools to convert the virtual hard disk need to be installed. In CentOS 7 that happens by installing QEMU and its surrounding packages.

```bash
install -y qemu qemu-common qemu-img qemu-kvm-common qemu-system-x86 qemu-user bridge-utils
```

### Set up environment

In RedHat/CentOS the default location to store virtual disks, images etc. is `/var/lib/libvirt/images`. So let's set a variable and make sure that directory does exist:

```bash
export VMDIR=/var/lib/libvirt/images
mkdir -p $VMDIR
```

Then let's give the new appliance VM a short distinct name - e.g. orienting on its purpose and version number - and create its own directory.

```bash
export VMNAME=edge-151
mkdir -p $VMDIR/$VMNAME
```

Also we need an environmental variable set for the QEMU virtual filesystem handling:

```bash
export LIBGUESTFS_BACKEND=direct
```

## Conversion

To get a vSphere appliance image working on QEMU we first need to convert the encapsuled disk image into a supported format, here `QCOW2`.

### Extract the `OVA` file

Copy your original OVA file into your virtual machine image library (I'm using `$VMDIR` for this). The create a temporary directory - or if it exists already, make sure it is cleaned up - and unpack the OVA's contents:

```bash
mkdir -p $VMDIR/tmp
rm -rf $VMDIR/tmp/*
tar xf $VMDIR/$VMNAME-vsphere.ova -C $VMDIR/tmp
```

### Convert the virtual disk

QEMU comes with the `qemu-img` command that is not only used to create new images, but also for converting between different formats. Use it to make the virtual disk conversion, adjust the pattern to match the actual name of the `VMDK` file.

If there are more than one disk images than repeat the step for each of them and remember to attach them all to the new VM later.

```bash
qemu-img convert -f vmdk $VMDIR/tmp/*-disk1.vmdk $VMDIR/$VMNAME.qcow2
```

This step will leave the file at its *intended* size: i.e. if the original image of the virtual appliance was configured with a size of 100GB, this new `QCOW2` image will be as big, too. So there is another step to reduce that size to what the file actually needs on the file system:

```bash
qemu-img convert -O qcow2 $VMDIR/$VMNAME.qcow2 $VMDIR/$VMNAME-shrunk.qcow2
```

The bigger file can be removed, and so can the other files that came with the `OVA`:

>Remember to get other information from the original `VMX` configuration file first, i.e. specific `MAC addresses` that you may need to configure in the VM to satisfy an existing DHCP config or so.
{: .notice--warning }

```bash
rm -f $VMDIR/$VMNAME.qcow2
rm -rf $VMDIR/tmp/*
```

The last step here is to copy/move the new disk image to your virtual machine's directory:

```bash
cp $VMDIR/$VMNAME-shrunk.qcow2 $VMDIR/$VMNAME/$VMNAME.qcow2
```

Or - as I like to do - create a snapshot from the image which makes it faster and more efficient to start over if needed. This also allows for running multiple virtual machines from the same image.

```bash
cd $VMDIR/$VMNAME
qemu-img create -b ../$VMNAME-shrunk.qcow2 -f qcow2 $VMNAME.qcow2 100G
```

### Prepare a config ISO image

To use cloud images you may need a config ISO image that the system would mount as CD-ROM during a cloud-init run. If that image doesn't exist yet you can use the following examples and modify them to fit your needs.
Both files are written in the YAML format, a simple to understand descriptive language that you may already know from other DevOps tools.

#### Create the cloud-init config files

The first file provides meta information about the system, like internal and external hostnames:

```bash
cat <<EOF> $VMDIR/$VMNAME/meta-data
instance-id: $VMNAME
local-hostname: $VMNAME
EOF
```

The second file contains instructions for cloud-init to execute during the initial setup. In this basic example the first user is created and set up with an SSH key and sudo permission, so you can log in right after the system came up and start working on it.

```bash
cat <<EOF> $VMDIR/$VMNAME/user-data
#cloud-config 
preserve_hostname: False
hostname: $VMNAME
fqdn: $VMNAME.yourdomain.local
users:
    - default
    - name: mcfisch
      groups: ['wheel']
      shell: /bin/bash
      sudo: ALL=(ALL) NOPASSWD:ALL
      ssh-authorized-keys:
        - ssh-ed25519 AAAACDFMSNLKGDJSRETWOEIIIDFGJDT+SORk4v+dsERFGvsdfgseT35Fv35Vssdoegff KVM Lab SSH Login mcfisch
output:
  all: ">> /var/log/cloud-init.log"
ssh_genkeytypes: ['ed25519', 'rsa']
ssh_authorized_keys:
  - ssh-ed25519 AAAACDFMSNLKGDJSRETWOEIIIDFGJDT+SORk4v+dsERFGvsdfgseT35Fv35Vssdoegff KVM Lab SSH Login key mcfisch
timezone: America/Los_Angeles
runcmd:
  - systemctl stop network && systemctl start network
  - yum -y remove cloud-init
EOF
```

Since this file allows for a lot more than this - including arbitrary file creation - you can define an entire system that does it supposed work without you ever logging in personally. This is one way to keep cloud images simple, clean and easily maintainable by separating them from the customization, which then can easily be managed in a code repository.

#### Create the ISO file

Now these files need to go into an ISO image, and all that's needed for this is the `genisoimage` command that can be installed from the systems package repo:

```bash
yum install genisoimage
```

Then run the following command to built an ISO file with the 2 files created above:

```bash
genisoimage -output $VMDIR/$VMNAME/$VMNAME-config.iso -volid config -joliet -r meta-data user-data
```

#### Modify an existing ISO image

In case you already have an image that just needs to get adjusted, you can do that, too. Just mount the image, copy its contents somewhere else, then modify them and rebuild the image:

```bash
# mount and copy the existing files to a local directory
MNTDIR=/mnt
TMPDIR=/tmp
mount $VMDIR/$VMNAME-config.iso $MNTDIR
mkdir -p $TMPDIR
cp $MNTDIR/* $TMPDIR/
umount $MNTDIR
# make your changes in $TMPDIR, then repack it
genisoimage -output $VMDIR/$VMNAME-config-new.iso -volid config -joliet -r $TMPDIR/*
rm -rf $TMPDIR/*
```

## Configure a bridged virtual network

If your VM needs a regular IP address within your network over which it can be reached, serve incoming requests etc. then you will need a bridged network the VM can use. For that simply create an additional KVM network and a local bridge it can use.
The culprit here is that, depending on your particular setup, in the moment you reconfigure your network interfaces you may lose access to the system. So make sure you have direct access to its console before following these steps.

### Add virtual bridge to physical interface

This example is for an additional interface, that is attached to a virtual bridge, which then is assigned a fixed IP, so the host system has two network interfaces connected.

```bash
brctl addbr br0
ip link set dev br0 up
ip addr add 192.168.1.10/24 dev br0
```

The same affect can be achieved by directly adding/editing the configuration files:

```text
cat <<EOF >/etc/sysconfig/network-scripts/ifcfg-br0
STP=yes
TYPE=Bridge
BOOTPROTO=static
NAME=br0
DEVICE=br0
ONBOOT=yes
IPADDR=192.168.1.10
PREFIX=24
GATEWAY=192.168.1.1
DNS1=192.168.1.1
PROXY_METHOD=none
BROWSER_ONLY=no
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=no
AUTOCONNECT_SLAVES=yes
DOMAIN=yourdomain.local
BRIDGING_OPTS=priority=32768
EOF

cat <<EOF > /etc/sysconfig/network-scripts/ifcfg-ens224 
TYPE=Ethernet
NAME=ens224
DEVICE=ens224
ONBOOT=yes
BRIDGE=br0
BOOTPROTO=none
DEFROUTE=yes
EOF

systemctl restart network
```

### Create a virtual network

To be able to use this new bridge we also need a virtual network defined that QEMU can attack virtual machines to.
Specify the respective configuration in an `XML` file, then import that into the QEMU setup.

```bash
cat <<EOF > host-bridge.xml
<network>
  <name>host-bridge</name>
  <forward mode='bridge'/>
  <bridge name='br0'/>
</network>
EOF
virsh net-define host-bridge.xml
virsh net-start host-bridge
virsh net-autostart host-bridge
```

### Enable packet forwarding

The following part my differ from what your system needs, it totally depends on how the host system is configured. In my case, where I was provided a ready-to-use VM in vSphere by another team, I had to follow these steps to make sure virtual machines nested in QEMU/KVM inside a vSphere machine can actually talk to the local network.

```bash
cat <<EOF > /etc/udev/rules.d/99-bridge.rules
ACTION=="add", SUBSYSTEM=="module", KERNEL=="br_netfilter", RUN+="/sbin/sysctl -p /etc/sysctl.d/bridge.conf"
EOF
cat <<EOF > /etc/sysctl.d/bridge.conf
net.bridge.bridge-nf-call-ip6tables=0
net.bridge.bridge-nf-call-iptables=0
net.bridge.bridge-nf-call-arptables=0
EOF
echo "NM_BOND_BRIDGE_VLAN_ENABLED=yes" >> /etc/sysconfig/network
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.d/99-sysctl.conf
sysctl -p /etc/sysctl.d/99-sysctl.conf
service NetworkManager restart
```

### Enable promiscuous mode on vSphere vSwitch port

If you're attempting this setup as I did in vSphere, you (or the respective vSphere admin) probably need to set up a port or port group that is configured to operate in `promiscuous mode` and allows for `address spoofing`. The reason for this potentially dangerous setup is, that the nested virtual network internally uses a routed connection between the VM's and the bridge interface. So this physical interface needs to listen to all packets of the incoming traffic, not just those addressed to its own MAC address (`promiscuous mode`), and it needs to send packets in the name of the VM's interfaces and their MAC addresses (`address spoofing`).

To keep the security impact of these settings as low as possible I'd recommend using a dedicated port group for this VM in vSphere that has the following settings enabled:

- `Promiscuous mode`
- `MAC address changes`
- `Forged transmits`

However, I'd recommend turning these on one at a time and testing after each if your connections start working.

## Setup the VM

Once the disk and CD-ROM images are in place it is time to configure the actual VM. The process for an official cloud-image of CentOS 7 for instance is relatively straightforward:

```bash
virt-install --import --name $VMNAME --virt-type=kvm --memory 2048 --vcpus 2 --cpu host --disk $VMDIR/$VMNAME/$VMNAME.qcow2,format=qcow2,device=disk --hvm --disk $VMDIR/$VMNAME/$VMNAME-seed.iso,device=cdrom --network network=host-bridge --os-type=linux --os-variant=centos7.0 --graphics vnc --noautoconsole
```

However, for my specific case I needed a specific setup that required a couple things:

- 3 NICs
- static IP addresses that will be utilized by a package installed by the config ISO
- fixed MAC addresses to use the company's DHCP that I preconfigured for these

To satisfy these I used the following command:

```bash
virt-install --import --name $VMNAME --virt-type=kvm --memory 8192 --vcpus 8 --cpu host --disk $VMDIR/$VMNAME/$VMNAME.qcow2,format=qcow2,device=disk,bus=ide --hvm --disk $VMDIR/$VMNAME/$VMNAME-config.iso,device=cdrom --network=network=host-bridge,model=virtio,mac=52:54:00:a7:85:d8 --network=network=host-bridge,model=virtio,mac=52:54:00:7b:b3:9f --network=network=host-bridge,model=virtio,mac=52:54:00:79:09:c5 --os-type=linux --os-variant=centos7.0 --graphics vnc --noautoconsole
```

Not only does that set the basic VM parameters as well as the hard-coded MAC addresses, it also specifies a key setting for the successful conversion of (at least my) vSphere virtual appliance to QEMU/KVM:

>`--disk $VMDIR/$VMNAME/$VMNAME.qcow2,format=qcow2,device=disk,bus=ide`

The `bus=ide` at the end of the disk definition was crucial as the VMs wouldn't boot with any other configuration. This may vary for your appliance, so if an appliance won't boot check if setting another controller type makes it work. The most commonly used type for instance is `virtio`, but depending on your version of KVM it might be something like `scsi0.0`, `ahci.0` or `sata` instead, usually accompanied by a device id. Check the documentation for your particular QEMU/KVM installation.

## Operation

With this all set the new VM should be ready to be used.

### Start/stop the VM

The last thing to do is to start the VM up:

```bash
virsh start $VMNAME
```

If the VM needs to be shutdown this can be done so:

```bash
virsh shutdown $VMNAME
```

To pull the VM's virtual power plug run this:

```bash
virsh destroy $VMNAME
```

This would be also helpful when you're going to replace the VM's disk but want to keep the actual VM configuration.

Now if that's not the case and you need to completely start over - or simply clean up - the following command will remove the VM from inventory:

```bash
virsh undefine $VMNAME
```

### View/edit the VM configuration

However, if you ever need to access or change the VM's configuration, it is possible to just output it to the command line:

```bash
virsh dumpxml $VMNAME
```

Or load it into your systems's default editor to make permanent changes to the VM setup:

```bash
virsh edit $VMNAME
```

### Connect to VM console

Sometimes there is no way to access the VM remotely - maybe the network interfaces are getting moved into Docker containers, maybe it has no route to your other networks, no SSH server is installed, or the appliance just doesn't allow for any incoming connections. Either wey, there are other ways to access a VM's console through the host system. What worked best for me was VNC. The VMs in such setup get VNC display IDs attached, starting with `0`, and incrementing with each VM that get's started.

To access any of them in a restricted network I use the port forwarding/tunnelling functionality that SSH provides. Starting at `5900` the last digit of the port number represents the VNC display ID: `5900` connects to VM `0`, `5901` connects to VM `1`, etc.

To access any of these VMs I add the parameters `-L <VNC port>:127.0.0.1:<local port>` to my SSH command:

I then point a VNC viewer to `localhost:<VNC display ID>` (not the full port) and are connected to the respective VM's actual console.

```bash
ssh -L 5900:127.0.0.1:5900 root@192.168.1.10
```

## Conclusion

At this point I had a working procedure to manually convert and run a VMware appliance on QEMU. This is still lacking the implementation in OpenStack, but at least it is a workaround to get going. OpenStack nodes usually have the needed services and drivers already installed, so for the moment this setup will do it. Finding a way to convert the CD-ROM image into an OpenStack config drive is the one task on my ToDo list, building a Jenkins job for this procedure is another.

*Happy virtualizing!*
