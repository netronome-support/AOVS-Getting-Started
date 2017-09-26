# Agilio OVS 2.6B Getting Started (CentOS)

## Install Patched Kernel
* Install kernel packages supplied on [Netronome Support Site](https://support.netronome.com)
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


