# polana-ansible

Playbooks to provision and configure HASS on polana (Pi 3B/4B or similar). The initial plan assumes an SSD will be used for storage (vs. SD card) and that the host will boot from the SSD.

Tasks fall into 2 general categories. First loading the OS to the target SSD on another host and then configuring the system as needed once it is up and running. More specifically

Phase 1

1. `blkdiscard` SSD.
1. Copy OS to the SSD.
1. Expand the root partition and create another partition (to use for ZFS.)
1. hostname?
1. Create user?
1. Create ZFS pool on the target SSD and create filesystems for various usage.

```text
rpool/home/hbarta               579G   276G     81.7G  /home/hbarta
rpool/var                      4.15G   276G       96K  /var
rpool/var/lib                  1.23M   276G       96K  /var/lib
rpool/var/lib/docker            592K   276G      512K  /var/lib/docker
rpool/var/cache                3.76G   276G     3.70G  /var/cache
rpool/var/log                   403M   276G      400M  /var/log
```

Phase 2

1. Install ZFS and copy/mount directories.
1. Install packages from repos.
1. Set hostname (if not previously done.)
1. Add user (if not previously done.)
1. Install Docker.
1. Install Mosquitto, MariaDB and HASS Docker images. and backup data from other host. (And create any useful ZFS filesystems as needed.)
1. Install Checkmk agent.


## Status

* `Read device information` results in `"ssd_info.stdout_lines": "VARIABLE IS NOT DEFINED!"`
* `Create partition for ZFS ` results in

```text
fatal: [localhost]: FAILED! => {"changed": false, "err": "Error: You requested a partition from 0.00kiB to 117220824kiB (sectors 0..234441647).\nThe closest location we can manage is 0.50kiB to 4096kiB (sectors 1..8191).\n", "msg": "Error while running parted script: /usr/sbin/parted -s -m -a optimal /dev/sdb -- unit KiB mkpart primary 0% 100%", "out": "", "rc": 1}
```

## Phase 1 - provision SSD

***NOTE: Be absolutely certain that the correct SSD device is provided***

### provision-Debian.yml

```text
ansible-playbook provision-Debian.yml -b -K --extra-vars "ssd_dev=/dev/sdc \
    os_image=/home/hbarta/Downloads/Pi/Debian/20230425_raspi_4_bullseye.img.xz \
    new_host_name=polana2 poolname=polana_tank"
```

## Errata

The starting point for the third partition is hard coded and depends on the size to which the 2nd partition is resized. If the size of the 2nd partitiopn is changed, the starting point for the third partition can be determined by e.g.

```text
hbarta@olive:~/Programming/polana-ansible$ echo "unit s print" |sudo parted  /dev/sdc
GNU Parted 3.5
Using /dev/sdc
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) unit s print                                                     
Model: ATA SATA SSD (scsi)
Disk /dev/sdc: 234441648s
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags: 

Number  Start     End        Size       Type     File system  Flags
 1      8192s     1048575s   1040384s   primary  fat16        lba
 2      1048576s  62914559s  61865984s  primary  ext4

(parted)                                                                  
hbarta@olive:~/Programming/polana-ansible$
```

Since the 2nd partition ends at `62914559s` the start of the third partition is `62914559s`.
