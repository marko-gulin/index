[Home](../../README.md) > KB-0001

# Network-Attached Storage with Raspberry Pi 5

This article provides a step-by-step guide on how to build and configure a **Network-Attached Storage** (NAS) using a Raspberry Pi 5. This setup offers a cost-effective and customizable solution for managing and accessing your data.

## Hardware

Basic configuration (`302.26 €`):
- Raspberry Pi 5 8GB Quad-Core [(Amazon)](https://www.amazon.fr/-/en/dp/B0CK2FCG1K)
- Raspberry Pi 5 USB-C 27W Power Supply [(Amazon)](https://www.amazon.fr/-/en/dp/B0CM46P7MC)
- SanDisk 128GB Extreme Pro microSDXC card [(Amazon)](https://www.amazon.fr/-/en/dp/B09X7DNF6G)
- Geekworm X1004 Dual M.2 NVMe SSD Shield [(Amazon)](https://www.amazon.fr/-/en/dp/B0D22JPQRB)
- Geekworm P579-H500 Enclosure with Active Cooler [(Amazon)](https://www.amazon.fr/-/en/dp/B0CT4P893Q)
- Eaton Ellipse ECO 650 Off-Line UPS EL650USBFR [(Amazon)](https://www.amazon.fr/dp/B0052QV9MK)

Main storage (`276.98 €`):
- 2x TeamGroup MP44L 2TB SLC Cache NVMe 1.4 PCIe Gen 4x4 M.2 2280 SSD [(Amazon)](https://www.amazon.fr/-/en/dp/B0B9Y48V73)

Backup storage (`129.50 €`):
- Western Digital 5TB External HDD USB3.0 [(Amazon)](https://www.amazon.fr/-/en/dp/B07X41PWTY)

:moneybag: The total cost estimation (`708.74 €`) is for March 11, 2025.

## Install Raspberry Pi OS Lite

Download and install the latest **Raspberry Pi Imager**:

> https://www.raspberrypi.com/software/

Insert the microSD card into your computer. If your computer does not have a microSD card slot, use a USB adapter:
- SanDisk MobileMate UHS-I microSD Reader/Writer USB3.0 [(Amazon)](https://www.amazon.fr/-/en/dp/B07G5JV2B5)

Open the **Raspberry Pi Imager** and set the following:
- Raspberry Pi Device: `Raspberry Pi 5`
- Operating System: `Raspberry Pi OS Lite (64-bit)`, found under the *Raspberry Pi OS (other)* group
- Storage: `<select the SD card>`

![Raspberry Pi Imager](img/imager.png)<br>
**Figure 1.** Raspberry Pi Imager

After clicking *NEXT*, the application will ask a question:

> Would you like to apply OS customisation settings?

Click *EDIT SETTINGS* and configure the settings:
- Set hostname: `raspnas.local` (feel free to set another hostname)
- Set the username and password
- Uncheck *Configure wireless LAN* (OpenMediaVault disables WiFi by default)
- Under *SERVICES*, check *Enable SSH* and select *Use password authentication*

![Raspberry Pi OS Customisation](img/customisation.png)<br>
**Figure 2.** Raspberry Pi OS Customisation

Click *SAVE* followed by *YES*. Writing and verifying the image to the SD card takes about 1 minute.

Connect the Raspberry Pi to the power supply and Ethernet.

## Connect via SSH

Open the terminal and connect to the Raspberry Pi via SSH:

```
ssh home@raspnas.local
home@raspnas.local's password: *****
```

where `home` and `*****` are the username and password, and `raspnas` is the hostname (see Fig. 2). When you are connecting to the Raspberry Pi via SSH for the first time, the console will ask you:

```
The authenticity of host 'raspnas.local (192.168.X.Y)' can't be established.
ED25519 key fingerprint is SHA256:....
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? 
```

Type `yes` and press *ENTER*. Once connected, first make sure your system is up to date:

```bash
sudo apt update && sudo apt upgrade -y
```

### Check SSD disks

List information about storage devices (block devices) to make sure NVMe SSD disks are properly connected:

```bash
lsblk
```

Example output:

```bash
mmcblk0     179:0    0 119.1G  0 disk
├─mmcblk0p1 179:1    0   512M  0 part /boot/firmware
└─mmcblk0p2 179:2    0 118.6G  0 part /
nvme0n1     259:0    0   1.9T  0 disk
nvme1n1     259:1    0   1.9T  0 disk
```

:warning: It is reported by multiple users that PCIe flex cables supplied with the X1004 shield are not of the best quality. If you do not see the SSD disks, first try replacing the PCIe flex cable.

The file system should be removed from the SSDs if you plan on setting them up in RAID (recommended). Check the existing file system:

```bash
lsblk -f
```

Example output:

```bash
NAME        FSTYPE            FSVER LABEL     UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
mmcblk0
├─mmcblk0p1 vfat              FAT32 bootfs    4EF5-6F55                             455.3M    11% /boot/firmware
└─mmcblk0p2 ext4              1.0   rootfs    ce208fd3-38a8-424a-87a2-cd44114eb820  108.3G     2% /
nvme0n1     linux_raid_member 1.2   raspnas:0 ac072fda-32c6-f1bd-e0cd-0b1994f0a769
nvme1n1     linux_raid_member 1.2   raspnas:0 ac072fda-32c6-f1bd-e0cd-0b1994f0a769
```

Here it is shown that `nvme0n1` and `nvme1n1` disks already have the `linux_raid_member` file system, and that they are not mounted (see the *MOUNTPOINTS* column). If they are mounted, it is recommended to unmount the disks before deleting the file system:

```bash
sudo umount /dev/nvme0n1
sudo umount /dev/nvme1n1
```

Once unmounted, remove the file system:

```bash
sudo wipefs --all /dev/nvme0n1
sudo wipefs --all /dev/nvme1n1
```

Example output:

```bash
/dev/nvme0n1: 4 bytes were erased at offset 0x00001000 (linux_raid_member): fc 4e 2b a9
/dev/nvme1n1: 4 bytes were erased at offset 0x00001000 (linux_raid_member): fc 4e 2b a9
```

This removes all file system signatures without affecting the partition table.

### Install OpenMediaVault

OpenMediaVault (OMV) is a free and open-source NAS solution based on Debian Linux. It provides a web-based interface for managing storage, making it easy to set up a home or small office NAS without deep Linux knowledge. Install OpenMediaVault (OMV) as follows:

```bash
wget -O - https://github.com/OpenMediaVault-Plugin-Developers/installScript/raw/master/install | sudo bash
```

The OMV installation process takes about 5 minutes, after which it automatically reboots the Raspberry Pi. The console will output:

```bash
Connection to raspnas.local closed by remote host.
```

:warning: The OMV disables WiFi connection by default. It can be re-enabled once installation completes, but OMV works much better with an Ethernet connection.

Connect again via SSH and install the OMV-Extras as follows:

```bash
wget -O - https://github.com/OpenMediaVault-Plugin-Developers/packages/raw/master/install | sudo bash
```

## Configure OpenMediaVault

### Login Credentials

Open a web browser and go to `raspnas.local`. The default login credentials are:

```
User name: admin
Password: openmediavault
```

It is highly recommended to change the default OMV login credentials for security reasons. The default credentials for OMV are well-known and can be easily exploited if someone gains unauthorized access to your network. Click on the user icon in the top right corner and select *Change Password*.

### Dashboard

Navigate to the *Dashboard* where there will be information about the dashboard not being configured. Configure the dashboard. For example, select CPU Utilization, Disk Temperatures, Load Average, Memory, Services, System Information, System Time, and Uptime.

### Setup RAID1 (Mirror)

To configure SSD disks in RAID, navigate to `System > Plugins` and install the *openmediavault Linux MD (Multiple Device) plugin*.

Navigate to `Storage > Multiple Device` and click on *Create*:
- Level: `Mirror` (equivalent to RAID1)
- Devices: `<select the two SSD disks>`

Note that SSD disks must be free of any file system on them; otherwise, they will not be shown under the *Devices*. Apply pending configuration changes. For a 2TB SSD disk, it takes about 160 minutes to finish.

Navigate to `Storage > File Systems` and click on *Mount an existing file system*:
- File system: `/dev/md0`
- Usage Warning Threshold: `85%` (default setting)

Click *Save* followed by applying pending configuration changes.

### Setup Shared Folder and Samba

Navigate to `Storage > Shared Folders` and click *Create*:
- Name: `Cloud` (or any other shared folder name)
- File System: `/dev/md0`
- Relative Path: `<leave default>`, in my case `Cloud/`
- Permissions: `Administrator: read/write, Users: read/write, Others: no access`

Click *Save* followed by applying pending configuration changes.

Samba is an open-source software that allows file and printer sharing between Linux/Unix systems and Windows, macOS, and other devices using the SMB/CIFS (Server Message Block/Common Internet File System) protocol. It enables seamless access to shared folders across different operating systems. Navigate to `Services > SMB/CIFS > Settings`, select *Enabled* and make sure that under *Minimum protocol version* is `SMB2` or higher. Click *Save* followed by applying pending configuration changes.

Navigate to `Services > SMB/CIFS > Shares` and click *Create*:
- Shared folder: `Cloud`
- Public: `No`
- Select *Browseable*, *Inherit permissions*, *Enable recycle bin*

Click *Save* followed by applying pending configuration changes.

Navigate to `Users > Users`, select user `home` and click *Edit*. Set the password and click *Save* followed by applying pending configuration changes.

### Connect to Shared Folder from a Windows Machine

Open Windows Explorer and navigate to `\\raspnas.local\Cloud`. Enter your credentials and click on *Remember my credentials*. Right-click on the *Cloud* directory and select *Map network drive...*. This will add the shared folder as a network drive.

Sometimes the Windows OS complains about not being able to access the remote directory. To resolve this issue, open the **Registry Editor** and navigate to:

```
Computer\HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\LanmanWorkstation\Parameters
```

Set the `AllowInsecureGuestAuth` value to `1`. If the parameter does not exist, create it as a *DWORD (32-Bit) Value* as explained in:

> https://forums.raspberrypi.com/viewtopic.php?t=200482

## Backup to External HDD

The **3-2-1 rule of data storage** is a best practice for data backup and disaster recovery: keep at least 3 copies of your data, store the copies on at least 2 different storage types, and keep 1 copy offsite. It ensures your data is safe and recoverable in case of failure, corruption, or cyberattacks.

Connect your external HDD to the Raspberry Pi and make sure it does not have a file system via `lsblk -f`. In the OMV web interface, navigate to `Storage > File Systems`, click *Create*, and select the *EXT4* file system. The file system creation takes about 5 minutes for a 5TB disk.

RSync (Remote Sync) is a powerful and efficient file-copying tool commonly used for synchronizing files and directories between different locations. It is widely used for backups, mirroring, and incremental file transfers due to its ability to only copy differences between source and destination, saving time and bandwidth. The OMV web interface provides RSync under *Services*, but unfortunately, it is only possible to synchronize a selected directory, not the entire drive.

Create a script on the Raspberry Pi which mounts the external HDD and synchronizes the data:

```bash backup_raid.sh
#!/bin/bash

DEV="/dev/sda1"
HDD="/srv/dev-disk-by-uuid-HDD"
RAID="/srv/dev-disk-by-uuid-RAID"

sudo mount -n $DEV $HDD
sudo rsync -avxHAX --delete $RAID $HDD

if [[ "$1" == "-u" ]]; then
    sudo umount $HDD
fi
```

Make sure to specify the proper external HDD device name in the `DEV` variable, and mount points in the `HDD` and `RAID` variables. Make the script executable and run it as follows:

```bash
chmod +x backup_raid.sh
./backup_raid.sh -u
```

The script takes an optional `-u` argument which unmounts the external HDD after the synchronization is complete. You can verify that the memory used on the RAID and the external HDD is the same via the `df -h` command.

:warning: Note that the Windows OS does not have native support for the EXT4 file system. Check the following link for some recommendations:

> https://superuser.com/questions/37512/how-to-read-ext4-partitions-on-windows

## Backup the SD card

There is a realistic chance that the SD card on the Raspberry Pi will fail, especially in the event of an uncontrolled power cut. The data on the SSD disks remains intact, but the main problem is that everything needs to be reconfigured. In order to prevent that, it is recommended to do a backup of the SD card to the RAID.

Create a script to backup the SD card:

```bash backup_sd.sh
#!/bin/bash

DEV="/dev/mmcblk0"
RAID="/srv/dev-disk-by-uuid-RAID"

DIR="Cloud/RPi_Backups"
FILE="rpi_$(date +%F).img.gz"

mkdir -p $RAID/$DIR
sudo dd if=$DEV bs=4M status=progress | gzip > $RAID/$DIR/$FILE

echo "SD card backup completed: $DIR/$FILE"
```

Make sure to specify the proper SD card device name in the `DEV` variable, and the RAID mount point in the `RAID` variable. Make the script executable and run it as follows:

```bash
chmod +x backup_sd.sh
./backup_sd.sh
```

It takes about 40 minutes to complete a backup of a 128GB SD card.

To restore the SD card, create a script:

```bash flash_sd.sh
#!/bin/bash

RAID="/srv/dev-disk-by-uuid-RAID"

DIR="Cloud/RPi_Backups"
FILE=$(ls -t $RAID/$DIR | head -n 1)

gunzip -c $RAID/$DIR/$FILE | sudo dd of=$1 bs=4M status=progress

echo "SD card flash completed: $DIR/$FILE -> $1"
```

Make the script executable and run it as follows:

```bash
chmod +x flash_sd.sh
./flash_sd.sh /dev/sda1
```

Replace `/dev/sda1` with the appropriate device name for the SD card. Make sure that the backup SD card size is equal to or larger than the original.

## Uninterruptible power supply

It is highly recommended to use an Uninterruptible Power Supply (UPS) with a Raspberry Pi 5 NAS, especially for avoiding file system corruption and protecting RAID1 integrity. The Raspberry Pi 5 board is powered via a USB-C Type port which supports the Power Delivery (PD) negotiation, allowing the board to request higher currents from a compatible power adapter. The board requires 5 V, 3 A (15 W) at minimum, which works for most setups with light peripherals. However, when using NVMe SSD disks, PCIe adapter, and/or high-power USB devices, it is recommended to use a 5 V, 5 A (25 W) power supply. It is important to use only RPi-compatible USB-C chargers, since some USB-C chargers default to higher voltages (e.g., 9 V or 12 V) if they do not detect a proper PD handshake, which can damage the Raspbery Pi board as it works at a fixed 5 V input.

There are a dozen of off-the-shelf UPS solutions for Raspberry Pi devices, but not that many that satisfy the following requirements:
- Rated at 5 V, 5 A (25 W) and compatible with the Raspberry Pi PD negotation.
- Based on either lead-acid or LiFePO4 batteries, as they are much less prone to thermal runaway than Li-Ion batteries.

Geekworm has several UPS shields for the Raspberry Pi 5, but they are all based on Li-Ion batteries. There are a few solutions that are based on LiFePO4 batteries (see [LiFePO₄weredPi+™](httpslifepo4wered.comlifepo4wered-pi+.html)), but they are usually not mechanically compatible with the Geekworm X1004 shield and enclosure. In addition to this, most general-purpose UPS solutions typically used with routers are rated at 5 V, 3 A (15 W) which is not sufficient for this application.

The UPS specified in the **Hardware** section is an industrial-grade 400 W UPS with 4x AC outputs, surge protection, and a USB interface for diagnostics. It is bulky and probably overkill for this application, especillay since the UPS is needed only for a safe shutdown in the case of an unexpected power cut.
