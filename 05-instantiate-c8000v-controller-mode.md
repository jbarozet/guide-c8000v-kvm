# Create and Boot C8000v (Controller Mode)

## 1. Bootstrap File

### Overview

To instantiate a Virtual Router in controller mode and make sure it can register to the SD-WAN controllers (vBond, vManage, vSmart), you need to create a so called day0 configuration (bootstrap configuration) and pass that day0 configuration upon bootup. This file is MIME encoded and has to be copied to the flash. 

Note: 
- Virtual router (C8000v): file name must be exactly: `ciscosdwan_cloud_init.cfg`
- Hardware router: file name must be exactly: `ciscosdwan.cfg`

Upon bootup, CSR1000v/Catalyst8000v SD-WAN XE router will search bootflash: or usbflash: for filename `ciscosdwan_cloud_init.cfg`. 

In a KVM (or OpenStack) context, CSR1000v/C8000v only supports file-based **config-drive** configuration, CSR1000v/C8000v does not support **meta-data** or **user-data** as Day0 delivery mechanisms. You can create an ISO image with that file that attaches to the instance when it boots. The instance can mount this **cdrom** and read `ciscosdwan_cloud_init.cfg` from it to get information.

This day0/bootstrap file is a mime-encoded file that contains 2 sections:
- text/cloud-config
- text/cloud-boothook

### Section 1 - text/cloud-config

Contains the global parameters like uuid, token, org-name, vbond, root-ca cert  viptela properties encoded in the part in YAML format and others:
- vinitparam:
  - uuid
  - vbond
  - otp
  - org
  - rcc
- ca-certs
- format-partition (vManage only)

The following example illustrates a cloud-config section:

```
#cloud-config
vinitparam:
 - otp : 5073e722b501d9fe964386d473f035ec
 - vbond : vbond1.cisco.com
 - uuid : CSR-00C7BCB9-53EC-0BFC-FBF5-B0A347D92CAB
 - org : jmb-mamatus-sdwan
```

From version 17.2, “rcc” became just a binary switch. When rcc has any values, cloud-init finds ca-certs. “rcc: 1” is enough, however by giving cert itself to rcc, it can work on previous versions as well. ca-certs can be configured in the general way of cloud-init.

[Example](https://cloudinit.readthedocs.io/en/latest/topics/examples.html)

**otp** is the one time password for c8000v. Giving the chassis number as uuid and serial number as otp, C8000v boots up with those information already configured.

**vbond** is it’s vbond address, and org is Organization Name. When these are in cloud-config, C8000v is initialized with those information.

**format-partition**: 1 is specifically for vManage. With it, 2nd partition of HDD will be formatted automatically. Otherwise, VM will keep wait for user to confirm to format it and initialization stops.


### Section 2 - text/cloud-boothook

Each node has confd (Tail-F confd) inside, and it is the configuration database. Entered configuration here is imported into confd CDB when it is initialized.

Contains the configuration of the device. The following example illustrates a cloud-boothook section:

```
#cloud-boothook
  system
   chassis-number        C8K-D9900516-21E0-9F83-1A8A-E53E5115EA4D 
   system-ip             10.0.0.109
   site-id               109
   port-offset           0
   sp-organization-name  example 
   organization-name     example 
   console-baud-rate     9600
   vbond vbond1.cisco.com port 12346
  !
  sdwan
   interface GigabitEthernet1
    tunnel-interface
     encapsulation ipsec
     color biz-internet
     no allow-service bgp
     allow-service dhcp
     allow-service dns
     allow-service icmp
     allow-service sshd
     no allow-service netconf
     no allow-service ntp
     no allow-service ospf
     no allow-service stun
     no allow-service https
     no allow-service snmp
     no allow-service bfd
    exit
   exit
   !
  !
  !
  hostname C8000v-jmb
  !
  username admin privilege 15 secret 9 $9$3/IE1lEJ2/EH4.$1e7Vht.ttRRV3yZsESiDnSXqnTUV3hhLWEA/9Z4VSmc
  username jmb privilege 15 secret 9 $9$1kwK3kwD1lAM2E$KwOkebfHTiu8naZLCtM5A1cVgBfw4R4Vhsp6nlXhQAU
  !
  !
  vrf definition 10
   rd 1:10
   address-family ipv4
    route-target export 1:10
    route-target import 1:10
    exit-address-family
   !
   address-family ipv6
    exit-address-family
   !
  !
  vrf definition Mgmt-intf
   description MANAGEMENT
   rd          1:512
   address-family ipv4
    route-target export 1:512
    route-target import 1:512
    exit-address-family
   !
   address-family ipv6
    exit-address-family
   !
  !
  ip host vbond1.cisco.com 10.10.10.10 
  !
  interface GigabitEthernet1
   description   TRANSPORT - INET
   no shutdown
   ip address dhcp
  exit
  !
  interface Loopback10
   no shutdown
   arp timeout 1200
   vrf forwarding 10
   ip address 10.0.0.109 255.255.255.255
   ip mtu 1500
  exit
  !
  interface Tunnel1
   no shutdown
   ip unnumbered GigabitEthernet1
   tunnel source GigabitEthernet1
   tunnel mode sdwan
  exit
  !
  clock timezone UTC 0 0
  !
  aaa authentication login default local
  aaa authorization exec default local
  !
  line con 0
   speed    9600
   stopbits 1
  !
  line vty 0 4
   transport input ssh
  !
  line vty 5 80
   transport input ssh
  !
 !
!
```



## 2. Creating Bootstrap file from vManage

Create a Device Template and attach that template to your device. Enter all parameters and deploy.

Then go to Configuration > Device > WAN Edge List

Click on the 3-dots on the right of a device and pick "Generate bootstrap configuration"

![virt-manager](img/csr1000v-kvm-09.png)

For KVM and Openstack select Cloud-Init. (VMWare uses Encoded String). Then click OK.

You can either download the file and SCP it across to the host server, or copy and paste the contents via a terminal to the server. Use the method you are most comfortable with. That gives you the bootstrap config that you can apply when you instantiate the VM:

![virt-manager](img/csr1000v-kvm-10.png)

vManage will generate a cloud-init file that contains the cloud-config and cloud-boothook parts. This file is MIME encoded and can be used to load the VM (ciscosdwan_cloud_init.cfg file) or for hardware device (ciscosdwan.cfg). This file has to be copied to the flash. 

Once downloaded, you can then tune this file according to your needs.

> Note: 
> - The cloud-init config file will need to be named ciscosdwan_cloud_init.cfg for the C8000v to be able to load it.


## 3. Creating Bootstrap file using Linux tools

You can utilize `write-mime-multipart` application. It’s included in the `cloud-utils` package, so installation can be done in this way:
- Ubuntu or Debian: `apt-get install cloud-utils`
- Redhat or CentOS: `yum install cloud-utils`

You have to build 2 files:
+ cloud-config => cloud-config.txt
+ cloud-boothook => cloud-boothook.txt

The best option is to take a bootstrap file generated by vManage as a reference and change as required.

Example of cloud-config.txt:

```
#cloud-config
vinitparam:
 - uuid : CSR-E5E74EA5-B976-6584-CDFC-3D521AC73D24
 - vbond : 10.60.23.134
 - otp : 8c15259f59e7b815c8f272dbe58d3630
 - org : ADT Labs Paris
 - rcc : true
ca-certs:
  remove-defaults: false
  trusted:
  - |
   -----BEGIN CERTIFICATE-----
            [SNIP]
   -----END CERTIFICATE----------END CERTIFICATE-----
```

Example of cloud-boothook:

```
#cloud-boothook
  system
   chassis-number        C8K-D9900516-21E0-9F83-1A8A-E53E5115EA4D 
   system-ip             10.0.0.109
   site-id               109
   port-offset           0
   sp-organization-name  example 
   organization-name     example 
   console-baud-rate     9600
   vbond vbond1.cisco.com port 12346
  !

[Cut for brevity]

```

When you have each data in text files, you can combine them and construct a multipart text:
- `write-mime-multipart --output=ciscosdwan_cloud_init.cfg cloud-config.txt:text/cloud-config cloud-boothook.txt:text/cloud-boothook`


## 4. To pass Bootstrap File 

The CSR1000v/Catalyst8000v SD-WAN XE router will search for the filename `ciscosdwan_cloud_init.cfg`, which is the boostrap file generated by vManage (or by linux tools). We need to create an iso image that contains this file. This will be mounted as a cdrom and used by cloud-init to bootstrap de VM.

Create the iso image to pass the ciscosdwan_cloud_init.cfg bootstrap file to C8000v:
- `mkisofs -l -o config.iso ciscosdwan_cloud_init.cfg` 

This file config.iso will be used to create the C8000v with virt-install. The Bootstrap Configuration will be passed to the router at bootup.

> **Note**: Do not use the following command: `cloud-localds config.iso <configuration_filename>`
> 
> That creates a cdrom image that contains 2 files:
> - user-data
> - metadata
>
> This is typically used by VMs supporting cloud-init and not supported by CSR1000v/C8000v.



## 5. Create and boot the C8000v SD-WAN using virt-install (CLI)

Download image from CCO: https://software.cisco.com/download/home

Go to:
+ Downloads Home 
+ > Routers 
+ > Cloud Edge
+ > Catalyst 8000V Edge Software
+ > IOS XE Software

Then copy this file to KVM disk folder - this will be the image disk used by the CSR.

Example: `cp c8000v-universalk9_16G_serial.17.07.01a.qcow2 /home/jmb/kvm/disks/c8000v.qcow2`

Boot up the cEdge VM with Bootstrap configuration file as iso file:

```
# virt-install \
    --name=cedge \
    --os-type=linux \
    --os-variant=rhel4.0 \
    --arch=x86_64 \
    --cpu host \
    --vcpus=2 \
    --hvm \
    --ram=4096 \
    --disk path=c8000v.qcow2,size=16,device=disk,bus=ide,format=qcow2 \
    --disk path=config.iso,device=cdrom \
    --network=network:default,model=virtio \
    --network=network:service-net,model=virtio \
    --graphics none \
    --import
```

Notes:

+ `--import `: skip the OS installation process, and build a guest around an existing disk image. The device used for booting is the first device specified via "--disk" or "--file".
+ `--graphics none` : disable graphics support and use text based console
+ `--vnc` : Setup a virtual console in the guest and export it as a VNC server in the host. Unless the "--vncport" parameter is also provided, the VNC server will run on the first free port number at 5900 or above. The actual VNC display allocated can be obtained using the "vncdisplay" command to "virsh" (or virt-viewer can be used which handles this detail for the use).
+ `--vncport=VNCPORT` : Request a permanent, statically assigned port number for the guest VNC console. Use of this option is discouraged as other guests may automatically choose to run on this port causing a clash.

Once you see the VM is booted up login with the username/password admin.


## 6. Create and boot the Cisco C8000v SD-WAN using virt-manager (Graphical Interface)

virt-manager, also known as Virtual Machine Manager, is a graphical tool for creating and managing guest virtual machines.

Step 1 - Launch the virt-manager GUI. Click Create a new virtual machine.

![virt-manager](img/csr1000v-kvm-11.png)

Step 2 - Do one of the following: 

+ For .qcow2: Select Import existing disk image.
+ For .iso: Select Local install media (ISO image or CDROM).
+ ![virt-manager](img/csr1000v-kvm-12.png)

In this example, we want to import the qcow2 image.

Step 3 – Click on Browse to select the CSR qcow2 or iso file location.

![virt-manager](img/csr1000v-kvm-13.png)

Step4 - Click on Browse Local

![virt-manager](img/csr1000v-kvm-14.png)

Step5 - Select the disk image you want:

![virt-manager](img/csr1000v-kvm-15.png)

Step6 - Select the Operating System:

![virt-manager](img/csr1000v-kvm-16.png)

Step 7 - Configure the memory and CPU parameters.

![virt-manager](img/csr1000v-kvm-17.png)

Step 8 - Click “Customize configuration before install” and click finish.

![virt-manager](img/csr1000v-kvm-18.png)

Step 9 – By default there is one network interface mapped to network “default”. Click add Hardware to add a new network interface

![virt-manager](img/csr1000v-kvm-19.png)

Step 10 – Select the service-net network. The first default interface is mapped to “default”. Click finish.

![virt-manager](img/csr1000v-kvm-20.png)

Step 11 – Click Begin Installation on the top left corner of the window.

![virt-manager](img/csr1000v-kvm-21.png)

Once you see the VM is booted up login with the username/password admin.

<br>

## 7. Connect to console

List VMs running

```bash
# virsh list
 Id   Name                 State
------------------------------------
 2    cedge                running


#
```

List all VMs

```bash
# virsh list --all
 Id   Name                    State
----------------------------------------
 1    cedge                   running
 -    CSR-classic-16.11.01a   shut off
 -    CSR-classic-16.12.1a    shut off
 -    CSR-sdwan-16.12.1e      shut off

#
```

The default behaviour is to launch a VNC client to display the graphical console, or to run the "virsh" "console" command to display the text console.

The ```virsh console domain [--devname devicename] [--force] [--safe]``` command connects the virtual serial console for the guest virtual machine. This is very useful for example for guests that do not provide VNC or SPICE protocols (and thus does not offer video display for GUI tools) and that do not have network connection (and thus cannot be interacted with using SSH).

The optional ```--devname``` parameter refers to the device alias of an alternate console, serial, or parallel device configured for the guest virtual machine. If this parameter is omitted, the primary console will be opened. If the ```--safe``` option is specified, the connection is only attempted if the driver supports safe console handling. This option specifies that the server has to ensure exclusive access to console devices. Optionally, the force option may be specified, which requests to disconnect any existing sessions, such as in the case of a broken connection.

Connect to the console of the CSR1000v SD-WAN using the following command:

```bash
# virsh console cedge
Connected to domain CSR-sdwan-16.12.1e
Escape character is ^]
%IOSXEBOOT-4-BOOT_SRC: (rp/0): Checking grub versions 2.0 vs 2.0
%IOSXEBOOT-4-BOOT_SRC: (rp/0): Bootloader upgrade not necessary.


              Restricted Rights Legend


Use, duplication, or disclosure by the Government is
subject to restrictions as set forth in subparagraph
(c) of the Commercial Computer Software - Restricted
Rights clause at FAR sec. 52.227-19 and subparagraph
(c) (1) (ii) of the Rights in Technical Data and Computer
Software clause at DFARS sec. 252.227-7013.


           Cisco Systems, Inc.
           170 West Tasman Drive
           San Jose, California 95134-1706

```

<br>

## 8. VM Configuration File

Your VM config file which is in XML format. The config file is located at /etc/libvirt/qemu directory.

```bash
# ls -l /etc/libvirt/qemu
total 24
-rw------- 1 root root 3801 nov.  26 17:10 CSR-classic-16.11.01a.xml
-rw------- 1 root root 3758 nov.  26 17:20 CSR-classic-16.12.1a.xml
-rw------- 1 root root 3533 nov.  27 08:32 cedge.xml
-rw------- 1 root root 3533 nov.  27 08:32 cedge2.xml
drwxr-xr-x 3 root root 4096 sept. 20 13:19 networks/
#
```

This is an auto-generated file.

Changes to this xml configuration should be made using: `virsh edit cedge`

The output is an xml representation of the virtual machine properties, or, using virsh terminology, a domain. If you want to change, for example, the number of vcpus, you just have to find the relevant tag and change the value.

Then reboot the virtual machine for the settings to be applied: `virsh reboot cedge`

<br>

## 9. Shutdown the VM

To shut down the VM: `virsh shutdown cedge`

Brute force shutdown: `virsh destroy my_csr_vm`


<br>

## 10. Cloning a guest

Another utility, virt-clone can be used to create a new virtual machine by cloning an existing one. To proceed, we must first ensure that the guest to be cloned is down, than we run:

```bash
# virt-clone \
—original=my_csr_vm \
—name=my_csr_vm_clone \
--file=/home/jmb/kvm/disks/CSR-classic.qcow2
```

<br>

## 11. Adding Bootstrap File to the flash (Option)

If you did not provide the Bootstrap Config as iso file (mounted as CDROM) during the VM creation, you can still copy the bootstrap file into the file and reset the router to start from scratch. If there is no config in NVRAM, then the SD-WAN router will search bootflash: or usbflash: for filename ciscosdwan_cloud_init.cfg

Copy the bootstrap file generated by vManage or created using Linux tools into the router bootflash – Make sure you use the exact name: ciscosdwan_cloud_init.cfg

```
Router#copy scp: bootflash:
Address or name of remote host []? 192.168.1.107
Source username [admin]? jmb
Source filename []? CSR-cloudinit-KVM.txt
Destination filename [CSR-cloudinit-KVM.txt]? ciscosdwan_cloud_init.cfg
Password:
 Sending file modes: C0644 7569 CSR-cloudinit-KVM.txt
!
7569 bytes copied in 9.024 secs (839 bytes/sec)
Router#
```

Reset the SDWAN software in order for the chassis serial number from the ciscosdwan_cloud_init.cfg file to be applied to the device.
```request platform software sdwan software reset```

Once the router is back online, login and confirm that the chassis number is the same as the one from the Bootstrap File.
Check handling of ciscosdwan_cloud_init.cfg file

```
Router#show log process plogd internal | i sdwan|Fail
```

<br>

