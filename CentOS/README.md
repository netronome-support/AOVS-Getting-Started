
# Agilio OVS 2.6B Getting Started (CentOS 7.4)

## Install Patched Kernel
* Install patched kernel packages supplied on [Netronome Support Site](https://support.netronome.com)
```
rpm -i kernel*.rpm
```

* Verify patch
```
CMD:
setpci -d 19ee:4000 0xFFC.L

Expected output:
ffffffff
```

## Prerequisites

    yum -y install epel-release
    yum -y install make autoconf automake libtool gcc gcc-c++ libpcap-devel \
    readline-devel jansson-devel libevent libevent-devel libtool openssl-devel \
    bison flex gawk hwloc gettext texinfo rpm-build \
    redhat-rpm-config graphviz python-devel python python-devel tcl-devel \
    tk-devel texinfo dkms zip unzip pkgconfig wget patch minicom libusb \
    libusb-devel psmisc libnl3-devel libftdi pciutils \
    zeromq3 zeromq3-devel protobuf-c-compiler protobuf-compiler protobuf-python \
    protobuf-c-devel python-six numactl-libs python-ethtool kvm qemu-kvm \
    python-virtinst libvirt libvirt-python virt-manager libguestfs-tools \
    cloud-utils virt-install lvm2 wget git net-tools libguestfs-tools

## Install AOVS

* Install packages
```
    rpm -ivh *.rpm
```
>**NOTE:** Flash firmware if required
```
/opt/netronome/bin/nfp-update-flash.sh
```
* Reboot system
```
reboot
```

## Check status

* The following output indicates that AOVS **hasn't been launched** yet: 
```
CMD:
ovs-ctl status

OUTPUT:
SDN Health Report:
------------------
Userspace Process: ovsdb-server              ... [FAIL]
Userspace Process: ovs-vswitchd              ... [FAIL]
Userspace Process: virtiorelayd              ... [SKIPPED]
Kernel Module: nfp                           ... [PASS]
Kernel Module: nfp_cmsg                      ... [PASS]
Kernel Module: nfp_fallback                  ... [PASS]
Kernel Module: nfp_offloads                  ... [FAIL]
Kernel Module: nfp_conntrack                 ... [SKIPPED]
Kernel Module: openvswitch                   ... [FAIL]
NFP: Firmware Loaded                         ... [PASS]
NFP: Flow Processing Cores Responsive        ... [PASS]
NFP: Control Message Channel Responsive      ... [PASS]
NFP: Fallback Traffic Divert Disabled        ... [WARN]
NFP: Ingress NBI Backed Up                   ... [PASS]
NFP: Card Detected                           ... [PASS]
NFP: ECC Errors                              ... [PASS]
NFP: Parity Errors                           ... [PASS]
NFP: Configurator Version                    ... [PASS]
NFP: PCIe Gen3x8                             ... [PASS]

Overall System State     [FAIL]
```

* Start AOVS
```
CMD:
ovs-ctl start

OUTPUT:
 * Starting Agilio OvS 2.6.B
/etc/openvswitch/conf.db does not exist ... (warning).
Creating empty database /etc/openvswitch/conf.db           [  OK  ]
Starting ovsdb-server                                      [  OK  ]
system ID not configured, please use --system-id ... failed!
Configuring Open vSwitch system IDs                        [  OK  ]
Starting ovs-vswitchd                                      [  OK  ]
Enabling remote OVSDB managers                             [  OK  ]
 * Agilio OvS 2.6.B started successfully
 * Running firmware image /lib/firmware/nfp_firmware2.6.B_4001.nffw

```

* Display status of running AOVS
```
CMD:
ovs-ctl status

OUTPUT:
SDN Health Report:
------------------
Userspace Process: ovsdb-server              ... [PASS]
Userspace Process: ovs-vswitchd              ... [PASS]
Userspace Process: virtiorelayd              ... [SKIPPED]
Kernel Module: nfp                           ... [PASS]
Kernel Module: nfp_cmsg                      ... [PASS]
Kernel Module: nfp_fallback                  ... [PASS]
Kernel Module: nfp_offloads                  ... [PASS]
Kernel Module: nfp_conntrack                 ... [SKIPPED]
Kernel Module: openvswitch                   ... [PASS]
NFP: Firmware Loaded                         ... [PASS]
NFP: Flow Processing Cores Responsive        ... [PASS]
NFP: Control Message Channel Responsive      ... [PASS]
NFP: Fallback Traffic Divert Disabled        ... [PASS]
NFP: Ingress NBI Backed Up                   ... [PASS]
NFP: Card Detected                           ... [PASS]
NFP: ECC Errors                              ... [PASS]
NFP: Parity Errors                           ... [PASS]
NFP: Configurator Version                    ... [PASS]
NFP: PCIe Gen3x8                             ... [PASS]

Overall System State     [PASS]

ovsdb-server is running with pid 8971
ovs-vswitchd is running with pid 10156

```

## Host driver(nfp)
* Bind driver and configure interface
```
#Locate the dpdk-devbind.py script
updatedb
DPDK_DEVBIND=$(find /opt/netronome/ -iname dpdk-devbind.py)

#Grab the PCI address of VF nfp_v0.1
PCIA="$(ethtool -i nfp_v0.1 | grep bus | cut -d ' ' -f 5)"

  #Bind the VF to nfp_netvf
  echo $DPDK_DEVBIND --bind nfp $PCIA
  $DPDK_DEVBIND --bind nfp $PCIA

  echo $DPDK_DEVBIND --status
  $DPDK_DEVBIND --status | grep $PCIA

#Get netdev name
ETH=$($DPDK_DEVBIND --status | grep $PCIA | cut -d ' ' -f 4 | cut -d '=' -f 2)

#Assign IP to netdev and up the interface
ip a add 10.0.0.1/24 dev $ETH
ip link set dev $ETH up
```

* Add interfaces to bridge
```
BRIDGE=br0

# Delete all existing bridges
for br in $(ovs-vsctl list-br);
do
  ovs-vsctl --if-exists del-br $br
done

# Create a new bridge
ovs-vsctl add-br $BRIDGE

# Add physical ports
ovs-vsctl add-port $BRIDGE nfp_p0 -- set interface nfp_p0 ofport_request=1

# Add VF ports
ovs-vsctl add-port $BRIDGE nfp_v0.1 -- set interface nfp_v0.1 ofport_request=1

ovs-vsctl set Open_vSwitch . other_config:max-idle=300000
ovs-vsctl set Open_vSwitch . other_config:flow-limit=1000000
ovs-appctl upcall/set-flow-limit 1000000

ovs-vsctl show
ovs-ofctl show $BRIDGE
ovs-ofctl dump-flows $BRIDGE
```

## Create VM

* Use [these](https://github.com/netronome-support/IVG/tree/master/aovs_2.6B/vm_creator/ubuntu) scripts to create a VM.

   1) Clone/Download the entire directory

   2) Run _**x_create_backing_image.sh**_ to create backing image(Do this only **once**)

   3) Run _**y_create_vm_from_backing.sh**_ to create VMs

## Configure GRUB

* Add kernel parameters

   vi /etc/default/grub
```
GRUB_CMDLINE_LINUX_DEFAULT="intel_iommu=on iommu=pt intremap=on default_hugepagesz=2M hugepagesz=2M hugepages=2048 isolcpus=2-27,30-55 intel_idle.max_cstate=0 processor.max_cstate=0 idle=mwait intel_pstate=disable"
```

* Apply changes
```
grub2-mkconfig -o /boot/grub2/grub.cfg
reboot
```

* Confirm applied changes
```
cat /proc/cmdline
```

## SRIOV(vfio-pci)

* Bind vfio-pci to interface(s)
```
PCIA="$(ethtool -i nfp_v0.41 | grep bus | cut -d ' ' -f 5)"
PCIB="$(ethtool -i nfp_v0.42 | grep bus | cut -d ' ' -f 5)"

# Bind VF's using vfio-pci driver
driver=vfio-pci

DPDK_DEVBIND=$(find /opt/netronome -iname dpdk-devbind.py | head -1)
if [ "$DPDK_DEVBIND" == "" ]; then
  echo "ERROR: could not find dpdk-devbind.py tool"
  exit -1
fi

echo "loading driver"
modprobe $driver
echo "DPDK_DEVBIND: $DPDK_DEVBIND"
echo $DPDK_DEVBIND --bind $driver $PCIA
$DPDK_DEVBIND --bind $driver $PCIA

echo $DPDK_DEVBIND --bind $driver $PCIB
$DPDK_DEVBIND --bind $driver $PCIB

echo $DPDK_DEVBIND --status
$DPDK_DEVBIND --status | grep "$PCIA\|$PCIB"

```
* Add interface to Guest XML
```
VM_NAME=vm1
bus=$(ethtool -i nfp_v0.42 | grep bus-info | awk '{print $5}' | awk -F ':' '{print $2}')
EDITOR='sed -i "/<devices/a \<hostdev mode=\"subsystem\" type=\"pci\" managed=\"yes\">  <source> <address domain=\"0x0000\" bus=\"0x'${bus}'\" slot=\"0x0d\" function=\"0x1\"\/> <\/source>  <address type=\"pci\" domain=\"0x0000\" bus=\"0x00\" slot=\"0x0a\" function=\"0x0\"\/> <\/hostdev>"' virsh edit $VM_NAME
```
>XML interface example:
```
<hostdev mode='subsystem' type='pci' managed='yes'>
  <source>
    <address domain='0x0000' bus='0x05' slot='0x08' function='0x0'/>
  </source>
  <address type='pci' domain='0x0000' bus='0x00' slot='0x06' function='0x0'/>
</hostdev>

```
* Boot VM and SSH in
```
VM_NAME=vm1
virsh start $VM_NAME
ssh root@$(virsh net-dhcp-leases default | awk '/'"$VM_NAME"'/ {print $5}' | cut -d"/" -f1)
```
* Upgrade VM kernel
```
wget http://kernel.ubuntu.com/~kernel-ppa/mainline/v4.13/linux-headers-4.13.0-041300_4.13.0-041300.201709031731_all.deb
wget http://kernel.ubuntu.com/~kernel-ppa/mainline/v4.13/linux-image-4.13.0-041300-generic_4.13.0-041300.201709031731_amd64.deb
wget http://kernel.ubuntu.com/~kernel-ppa/mainline/v4.13/linux-headers-4.13.0-041300-generic_4.13.0-041300.201709031731_amd64.deb
```

git clone https://github.com/Netronome/nfp-drv-kmods
cd nfp-drv-kmods
make install





## XVIO AKA Virtio-relay(igb_uio)





---
