# Agilio OVS Getting Started (CentOS)

##Install Patched Kernel


##Prerequisites

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
    cloud-utils virt-install lvm2 wget git net-tools
    # guestfish
    yum -y install libguestfs-tools

##Install AOVS

    rpm -ivh *.rpm

>**NOTE:** Flash firmware if required
```
/opt/netronome/bin/nfp-update-flash.sh
```