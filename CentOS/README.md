
# Agilio OVS 2.6B Getting Started (CentOS 7.4)

## Clone IVG
```
git clone https://github.com/netronome-support/IVG.git
```

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

* Display status of running AOVS(SRIOV)
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

* Bind vfio-pci driver
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
* Add interfaces to bridge
```
BRIDGE=br0

# Delete all bridges
for br in $(ovs-vsctl list-br);
do
  ovs-vsctl --if-exists del-br $br
done

# Create a new bridge
ovs-vsctl add-br $BRIDGE

ovs-vsctl add-port $BRIDGE nfp_p0 -- set interface nfp_p0 ofport_request=1

# Add VF ports
ovs-vsctl add-port $BRIDGE nfp_v0.41 -- set interface nfp_v0.41 ofport_request=41
ovs-vsctl add-port $BRIDGE nfp_v0.42 -- set interface nfp_v0.42 ofport_request=42

#Add NORMAL RULE
ovs-ofctl del-flows br0
ovs-ofctl -O OpenFlow13 add-flow $BRIDGE actions=NORMAL


ovs-vsctl set Open_vSwitch . other_config:max-idle=300000
ovs-vsctl set Open_vSwitch . other_config:flow-limit=1000000
ovs-appctl upcall/set-flow-limit 1000000

ovs-vsctl show
ovs-ofctl show $BRIDGE
ovs-ofctl dump-flows $BRIDGE

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

* Enable hugepages
```
nr_hugepages=4194

cat /proc/mounts | grep hugetlbfs

umount /mnt/aovs-huge-2M

printCol 7 "Setting 2M"
grep hugetlbfs /proc/mounts | grep -q "pagesize=2M" || \
( mkdir -p /mnt/huge && mount nodev -t hugetlbfs -o rw,pagesize=2M /mnt/huge/ )

printCol 7 "Setting 1G"
grep hugetlbfs /proc/mounts | grep -q "pagesize=1G" || \
( mkdir -p /mnt/huge-1G && mount nodev -t hugetlbfs -o rw,pagesize=1G /mnt/huge-1G/ )

printCol 7 "/proc/mounts | grep hugetlbfs"
cat /proc/mounts | grep hugetlbfs

printCol 7 "libvirt folders"
mkdir -p /mnt/huge-1G/libvirt
mkdir -p /mnt/huge/libvirt

    chown qemu:qemu -R /mnt/huge-1G/libvirt || exit -1
    chown qemu:qemu -R /mnt/huge/libvirt || exit -1
    service libvirtd restart || exit -1
    echo $nr_hugepages > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
```
* Set permissions
```
sestatus
setenforce 0
sestatus | grep permissive
#make persistent
sed -E 's/(SELINUX=).*/\1disabled/g' -i /etc/sysconfig/selinux
```

* Configure AOVS
```
XVIO_CPU_COUNT=2

echo "CURRENT configuration"
cat /etc/netronome.conf

card_node=$(cat /sys/bus/pci/drivers/nfp/0*/numa_node | head -n1 | cut -d " " -f1)
nfp_cpu_list=$(lscpu -a -p | awk -F',' -v var="$card_node" '$4 == var {printf "%s%s",sep,$1; sep=" "} END{print ""}')
xvio_cpus_list=()
nfp_cpu_list=( $nfp_cpu_list )

for counter in $(seq 0 $((XVIO_CPU_COUNT-1)))
  do
        xvio_cpus_list+=( "${nfp_cpu_list[$counter+1]}" )
done

for counter in $(seq 0 $((XVIO_CPU_COUNT-1)))
  do
        nfp_cpu_list=( "${nfp_cpu_list[@]:1}" )
done

xvio_cpus_string=$(IFS=',';echo "${xvio_cpus_list[*]}";IFS=$' \t\n')

cat > /etc/netronome.conf << EOF
SDN_VIRTIORELAY_ENABLE=y
SDN_VIRTIORELAY_PARAM="--cpus=$xvio_cpus_string --enable-tso --enable-mrgbuf --vhost-username=qemu --vhost-groupname=kvm --huge-dir=/mnt/huge --ovsdb-sock=/var/run/openvswitch/db.sock"
SDN_FIREWALL=n
EOF

echo "NEW configuration"
cat /etc/netronome.conf

ovs-ctl status
ovs-ctl stop
ovs-ctl start
ovs-ctl status


```

* Launch AOVS
```
ovs-ctl start

#display status
CMD:
ovs-ctl status

OUTPUT:
SDN Health Report:
------------------
Userspace Process: ovsdb-server              ... [PASS]
Userspace Process: ovs-vswitchd              ... [PASS]
Userspace Process: virtiorelayd              ... [PASS]
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
```

* Bind igb driver
```
PCIA="$(ethtool -i nfp_v0.39 | grep bus | cut -d ' ' -f 5)"
PCIB="$(ethtool -i nfp_v0.40 | grep bus | cut -d ' ' -f 5)"

interface_list=($PCIA $PCIB)
driver=igb_uio
mp=uio
# updatedb
DPDK_DEVBIND=$(find /opt/ -iname dpdk-devbind.py | head -1)
DRKO=$(find /opt/ -iname 'igb_uio.ko' | head -1 )
echo "loading driver"
modprobe $mp
insmod $DRKO
echo "DPDK_DEVBIND: $DPDK_DEVBIND"
for interface in ${interface_list[@]};
do
  echo $DPDK_DEVBIND --bind $driver $interface
  $DPDK_DEVBIND --bind $driver $interface
done
echo $DPDK_DEVBIND --status
$DPDK_DEVBIND --status | grep "$PCIA\|$PCIB"
```

* Add interfaces to bridge
```
BRIDGE=br0

# Delete all bridges
for br in $(ovs-vsctl list-br);
do
  ovs-vsctl --if-exists del-br $br
done

# Create a new bridge
ovs-vsctl add-br $BRIDGE

ovs-vsctl add-port $BRIDGE nfp_p0 -- set interface nfp_p0 ofport_request=1

#Add VF's
ovs-vsctl add-port $BRIDGE nfp_v0.39 -- set interface nfp_v0.39 ofport_request=39 external_ids:virtio_relay=39
ovs-vsctl add-port $BRIDGE nfp_v0.40 -- set interface nfp_v0.40 ofport_request=40 external_ids:virtio_relay=40

ovs-ofctl del-flows $BRIDGE

ovs-ofctl -O OpenFlow13 add-flow $BRIDGE actions=NORMAL

ovs-vsctl set Open_vSwitch . other_config:max-idle=300000
ovs-vsctl set Open_vSwitch . other_config:flow-limit=1000000
ovs-appctl upcall/set-flow-limit 1000000

ovs-vsctl show
ovs-ofctl show $BRIDGE
ovs-ofctl dump-flows $BRIDGE

```


* Add interface to Guest XML
```
VM_NAME=vm1
VM_CPU=4
max_memory=$(virsh dominfo $VM_NAME | grep 'Max memory:' | awk '{print $3}')

# Remove vhostuser interface
EDITOR='sed -i "/<interface type=.vhostuser.>/,/<\/interface>/d"' virsh edit $VM_NAME
EDITOR='sed -i "/<hostdev mode=.subsystem. type=.pci./,/<\/hostdev>/d"' virsh edit $VM_NAME

# Add vhostuser interfaces
# nfp_v0.40 --> 0000:81:0d.0
# nfp_v0.41 --> 0000:81:0d.1

bus=$(ethtool -i nfp_v0.42 | grep bus-info | awk '{print $5}' | awk -F ':' '{print $2}')
# Add vhostuser interfaces
EDITOR='sed -i "/<devices/a \<interface type=\"vhostuser\">  <source type=\"unix\" path=\"/tmp/virtiorelay39.sock\" mode=\"client\"\/>  <model type=\"virtio\"/>  <driver name=\"vhost\" queues=\"1\"\/>  <address type=\"pci\" domain=\"0x0000\" bus=\"0x01\" slot=\"0x0a\" function=\"0x0\"\/><\/interface>"' virsh edit $VM_NAME
EDITOR='sed -i "/<devices/a \<interface type=\"vhostuser\">  <source type=\"unix\" path=\"/tmp/virtiorelay40.sock\" mode=\"client\"\/>  <model type=\"virtio\"/>  <driver name=\"vhost\" queues=\"1\"\/>  <address type=\"pci\" domain=\"0x0000\" bus=\"0x01\" slot=\"0x0b\" function=\"0x0\"\/><\/interface>"' virsh edit $VM_NAME

EDITOR='sed -i "/<numa>/,/<\/numa>/d"' virsh edit $VM_NAME
EDITOR='sed -i "/vcpu/d"' virsh edit $VM_NAME
EDITOR='sed -i "/<cpu/,/<\/cpu>/d"' virsh edit $VM_NAME
EDITOR='sed -i "/<memoryBacking>/,/<\/memoryBacking>/d"' virsh edit $VM_NAME

virsh setvcpus $VM_NAME $VM_CPU --config --maximum
virsh setvcpus $VM_NAME $VM_CPU --config
# MemoryBacking
EDITOR='sed -i "/<domain/a \<memoryBacking><hugepages><page size=\"2048\" unit=\"KiB\" nodeset=\"0\"\/><\/hugepages><\/memoryBacking>"' virsh edit $VM_NAME
#EDITOR='sed -i "/<domain/a \<memoryBacking><hugepages><page size=\"2048\" unit=\"KiB\"/hugepages><\/memoryBacking>"' virsh edit $VM_NAME
# CPU
echo max_memory: $max_memory
echo VM_CPU: $VM_CPU
EDITOR='sed -i "/<domain/a \<cpu mode=\"host-model\"><model fallback=\"allow\"\/><numa><cell id=\"0\" cpus=\"0-'$((VM_CPU-1))'\" memory=\"'${max_memory}'\" unit=\"KiB\" memAccess=\"shared\"\/><\/numa><\/cpu>"' virsh edit $VM_NAME

```

>XML interface example:
```
<interface type='vhostuser'>
  <mac address='ba:5c:ba:2a:5d:2e'/>
  <source type='unix' path='/tmp/virtiorelay0.sock' mode='client'/>
  <model type='virtio'/>
  <address type='pci' domain='0x0000' bus='0x00' slot='0x10' function='0x0'/>
</interface>
```

## Access VM

* Boot VM and SSH in
```
VM_NAME=vm1
virsh start $VM_NAME
ssh root@$(virsh net-dhcp-leases default | awk '/'"$VM_NAME"'/ {print $5}' | cut -d"/" -f1)
```

>**NOTE:** "error: unsupported configuration: huge pages per NUMA node are not supported with this QEMU" indicates you need to upgrade QEMU:
```
virsh version
Compiled against library: libvirt 3.2.0
Using library: libvirt 3.2.0
Using API: QEMU 3.2.0
Running hypervisor: QEMU 2.0.0

```
yum install centos-release-qemu-ev.noarch
yum install qemu-kvm-ev libvirt libvirt-python libguestfs-tools virt-install

service libvirtd restart

# Kernel - OVS(Intel)

* Install Intel driver
```
rm -rf /usr/local/src/i40e
wget https://github.com/netronome-support/IVG/raw/master/aovs_2.6B/test_case_11_kovs_vxlan_uni_intel/i40e-2.1.26.tar.gz -P /usr/local/src/i40e
cd /usr/local/src/i40e

tar zxf i40e*.tar.gz
cd i40e*/src
make install
rmmod i40e
modprobe i40e

```
* Configure Media mode

Download and extract [QCU configuration utility](https://github.com/netronome-support/AOVS-Getting-Started/raw/master/CentOS/Intel/QCU.zip)
```
QCU:
#list Intel devices
./qcu64e /DEVICES
./qcu64e /NIC=1 /INFO
#configure port(s)
./qcu64e /NIC=1 /SET 4x10

Reboot

``` 

* Create Intel VFs
```
echo 4 > /sys/bus/pci/devices/0000\:08\:00.0/sriov_numvfs

```

* Add interfaces to bridge
```
BRIDGE=br0
PORT=ens3f0

# Delete all bridges
for br in $(ovs-vsctl list-br);
do
  ovs-vsctl --if-exists del-br $br
done

# Create a new bridge
ovs-vsctl add-br $BRIDGE
ovs-vsctl add-port $BRIDGE $PORT -- set interface $PORT ofport_request=1

ovs-vsctl show

```


* Add bridge to Guest XML
```
VM_NAME=vm1
BRIDGE=br0

cat > /tmp/interface << EOL
<interface type='bridge'>
    <source bridge='$BRIDGE'/><virtualport type='openvswitch'/>
    <model type='virtio'/>
    <address type='pci' domain='0x0000' bus='0x01' slot='0xa' function='0x1'/>
</interface>
EOL

virsh attach-device $VM_NAME /tmp/interface --config


```












---
