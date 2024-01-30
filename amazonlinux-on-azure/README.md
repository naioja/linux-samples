# Running Amazon Linux on Azure

Install pre-requisite package:

```bash
apt update
apt install -y unzip guestfs-tools qemu-utils
```

Download the image hyperv image and checkshum file on a linux box:

```bash
wget https://cdn.amazonlinux.com/os-images/2.0.20240109.0/hyperv/amzn2-hyperv-2.0.20240109.0-x86_64.xfs.gpt.vhdx.zip
wget https://cdn.amazonlinux.com/os-images/2.0.20240109.0/hyperv/SHA256SUMS
```

Check that the zip archive sha256sum matches the one in the file:

```bash
sha256sum --check SHA256SUMS 
```

and the output shoud be name of the zip file followed by OK: `amzn2-hyperv-2.0.20240109.0-x86_64.xfs.gpt.vhdx.zip: OK`

Unzip the file and convert it to a `raw` linux image format for further changes:

```bash
unzip amzn2-hyperv-2.0.20240109.0-x86_64.xfs.gpt.vhdx.zip
```

```bash
Archive:  amzn2-hyperv-2.0.20240109.0-x86_64.xfs.gpt.vhdx.zip
inflating: amzn2-hyperv-2.0.20240109.0-x86_64.xfs.gpt.vhdx  
```

```bash
VHDX_IMG='amzn2-hyperv-2.0.20240109.0-x86_64.xfs.gpt.vhdx'
RAW_IMG='amzn2-hyperv-2.0.20240109.0-x86_64.xfs.gpt.raw'

qemu-img convert -f vhdx -O raw $VHDX_IMG $RAW_IMG
```

Using `losetup` with elevated priviledges mount the raw image to a filesystem directory.
Alternatively guestfish and guestmount can be used to perform the same spteps below.

```bash
sudo losetup -f -P amzn2-hyperv-2.0.20240109.0-x86_64.xfs.gpt.raw
```

check the current

```bash
losetup -l
NAME       SIZELIMIT OFFSET AUTOCLEAR RO BACK-FILE                                                                 DIO LOG-SEC
/dev/loop6         0      0         0  0 /home/azureuser/amznlinux/amzn2-hyperv-2.0.20240109.0-x86_64.xfs.gpt.raw    0     512
```

figuring out which partition of `/dev/loop6` to mount can be done by simply by running :

```bash
sudo fdisk -l amzn2-hyperv-2.0.20240109.0-x86_64.xfs.gpt.raw 

Disk amzn2-hyperv-2.0.20240109.0-x86_64.xfs.gpt.raw: 25 GiB, 26843545600 bytes, 52428800 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 01229C8D-28CC-4981-8D30-DFF94A0941EE

Device                                            Start      End  Sectors Size Type
amzn2-hyperv-2.0.20240109.0-x86_64.xfs.gpt.raw1    4096 52428766 52424671  25G Linux filesystem
amzn2-hyperv-2.0.20240109.0-x86_64.xfs.gpt.raw128  2048     4095     2048   1M BIOS boot

Partition table entries are not in disk order.
```

mount the first partition of `/dev/loop6p1` to a filesystem directory

```bash
sudo mount /dev/loop6p1 /srv
```

Checking that amazonlinux filesystem is now mounted :

```bash
cat /srv/etc/os-release 

NAME="Amazon Linux"
VERSION="2"
ID="amzn"
ID_LIKE="centos rhel fedora"
VERSION_ID="2"
PRETTY_NAME="Amazon Linux 2"
ANSI_COLOR="0;33"
CPE_NAME="cpe:2.3:o:amazon:amazon_linux:2"
HOME_URL="https://amazonlinux.com/"
SUPPORT_END="2025-06-30"
```

Following the [Azure documentation](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/create-upload-generic) regarding generic linux distros :

1. we need to make sure all kernel drivers are loaded 

    Source:

    [Prepare a Red Hat-based virtual machine for Azure](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/redhat-create-upload-vhd)
    [Prepare a CentOS-based virtual machine for Azure](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/create-upload-centos)
    [How Accelerated Networking works in Linux and FreeBSD VMs](https://learn.microsoft.com/en-us/azure/virtual-network/accelerated-networking-how-it-works)

```bash
sudo cat << EOF > /srv/etc/dracut.conf.d/azure.conf
add_drivers+=" hv_vmbus hv_netvsc hv_storvsc "
add_drivers+=" nvme pci-hyperv "
add_drivers+=" mlx5_core mlx4_core "
add_drivers+=" udf vfat "
EOF
```

2. blacklist kernel modules

```bash
cat << EOF > /srv/etc/modprobe.d/blacklist.conf
blacklist nouveau
options nouveau modeset=0
blacklist lbm-nouveau
blacklist floppy
EOF
```

3. create udev rules for SRIOV 

    Source:

    [Prepare a Red Hat-based virtual machine for Azure](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/redhat-create-upload-vhd)
    [Prepare a CentOS-based virtual machine for Azure](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/create-upload-centos)

```bash
cat << EOF > /srv/etc/udev/rules.d/99-azure-sriov.rules
SUBSYSTEM=="net", DRIVERS=="hv_pci", ACTION=="add", ENV{NM_UNMANAGED}="1"
EOF
```

4. create udev rules for PTP clock device

    Source:

    [Time sync for Linux VMs in Azure](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/time-sync)

```bash
cat << EOF > /srv/etc/udev/rules.d/99-azure-ptp.rules
ACTION!="add", GOTO="ptp_hyperv"
SUBSYSTEM=="ptp", ATTR{clock_name}=="hyperv", SYMLINK += "ptp_hyperv"
LABEL="ptp_hyperv"
EOF
```

5. configure chrony to use `/dev/ptp0`

    Source:

    [Time sync for Linux VMs in Azure](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/time-sync)

```bash
cat << EOF > /srv/etc/chrony.conf
driftfile /var/lib/chrony/chrony.drift
logdir /var/log/chrony
maxupdateskew 100.0
refclock PHC /dev/ptp_hyperv poll 3 dpoll -2
makestep 1.0 -1
EOF
```

6. configure the network interface

    Source:

    [Prepare a Red Hat-based virtual machine for Azure](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/redhat-create-upload-vhd)
    [Prepare a CentOS-based virtual machine for Azure](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/create-upload-centos)

```bash
echo 'IPV4_DHCP_TIMEOUT=300' >> /srv/etc/sysconfig/network-scripts/ifcfg-eth0
echo 'IPV6INIT=no' >> /srv/etc/sysconfig/network-scripts/ifcfg-eth0
echo 'NOZEROCONF=yes' >> /srv/etc/sysconfig/network-scripts/ifcfg-eth0
```

7. enable the Azure datasource

    Source:

    [Prepare a Red Hat-based virtual machine for Azure](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/redhat-create-upload-vhd)
    [Prepare a CentOS-based virtual machine for Azure](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/create-upload-centos)

```bash
cat << EOF > /srv/etc/cloud/cloud.cfg.d/91-azure-datasource.cfg
datasource_list: [ Azure ]
datasource:
    Azure:
        apply_network_config: False
EOF
```

8. enable KVP for reporting provisioning telemetry

    Source:

    [cloud-init support for virtual machines in Azure](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/using-cloud-init#telemetry)

```bash
cat << EOF > /srv/etc/cloud/cloud.cfg.d/10-azure-kvp.cfg
# This configuration file enables provisioning telemetry reporting
reporting:
    logging:
        type: log
    telemetry:
        type: hyperv
EOF
```

9. chroot to the linux distro and install the following required packages :

    Source:

    [Prepare a Red Hat-based virtual machine for Azure](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/redhat-create-upload-vhd)
    [Prepare a CentOS-based virtual machine for Azure](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/create-upload-centos)

```bash
sudo chroot /srv yum install -y hyperv-daemons
```

10. regenerate all initrams

    Source:

    [Prepare a Red Hat-based virtual machine for Azure](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/redhat-create-upload-vhd)
    [Prepare a CentOS-based virtual machine for Azure](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/create-upload-centos)

```bash
sudo chroot /srv dracut -f --regenerate-all
```

## Now prepare the image to be imported in Azure

Unmount the image

```bash
sudo umount /srv
sudo losetup -d /dev/loop6
```

Resize and convert to vhd the image

Source: [Prepare Linux for imaging in Azure](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/create-upload-generic#resize-vhds)

```bash
rawdisk="amzn2-hyperv-2.0.20240109.0-x86_64.xfs.gpt.raw"
vhddisk="amzn2-hyperv-2-0-20240109-0-x86_64-xfs-gpt.vhd"

MB=$((1024*1024))
size=$(qemu-img info -f raw --output json "$rawdisk" | gawk 'match($0, /"virtual-size": ([0-9]+),/, val) {print val[1]}')
rounded_size=$(((($size+$MB-1)/$MB)*$MB))
echo "Rounded Size = $rounded_size"

qemu-img resize $rawdisk $rounded_size

qemu-img convert -f raw -o subformat=fixed,force_size -O vpc $rawdisk $vhddisk
```

Upload the image to Azure blob storage

Source: [Upload a VHD or VHDX](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/disks-upload-vhd-to-managed-disk-cli#upload-a-vhd-or-vhdx)

```bash
sas_URI=''
azcopy copy $vhddisk "$sas_URI" --blob-type PageBlob
```
