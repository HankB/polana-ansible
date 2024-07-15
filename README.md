# polana-ansible

Playbooks to provision and configure a Raspberry Pi (Pi 3B/4B or similar) with the intent to migrate an existing install to the newly conmfigured host. (The actual migration is beyond the scope of this project at present.) The initial plan assumes an SSD will be used for storage (vs. SD card) and that the host will boot from the SSD. Tested Debian images can be found at <https://raspi.debian.net/tested-images/>.

This is planned to be a "once and done" process so polish is eschewed in favor of expedience. Many of the tasks are likely to be otherwise useful so they can be polished then as appropriate. In particular, things such as host name, pool name, user name and probably other things are hard coded rather than being parameterized.

Tasks fall into 2 general categories. First loading the OS to the target SSD on another host and then configuring the system as needed once it is up and running. More specifically

Phase 1 - SSD in another host

1. `blkdiscard` SSD.
1. Copy OS to the SSD.
1. Expand the root partition and create another partition (to use for ZFS.)
1. hostname
1. Create ZFS pool on the target SSD. (filesystems creation is done later in the target environment so the ZFS cache is correct. I suppose this requires ZFS in the 'other' host.)

```text
rpool/home/hbarta               579G   276G     81.7G  /home/hbarta
rpool/var                      4.15G   276G       96K  /var
rpool/var/lib                  1.23M   276G       96K  /var/lib
rpool/var/lib/docker            592K   276G      512K  /var/lib/docker
rpool/var/cache                3.76G   276G     3.70G  /var/cache
rpool/var/log                   403M   276G      400M  /var/log
```

Phase 2 - first part

The tested Debian inage does not include Python which is normally used by Ansible so the first stage in the target environment simply installs that.

1. Update and upgrade packages.
1. Install Python.

Phase 2 - second part

This does the heavy lifting to prepare the environment including

1. Install some useful packages from repos.
1. Set locale and timezone
1. Install ZFS and create desired filesystems (for user and Docker)
1. Install Docker.
1. Add user.
1. Install Mosquitto, MariaDB and HASS Docker images. and backup data from other host. (And create any useful ZFS filesystems as needed.) (TODO?)
1. Install Checkmk agent. (TODO)

## Status

This set of playbooks has evolved into a more general set of playbooks, mostly to configure Debian hosts on Raspbnerry Pis.

I need to review the issues. At present I am usinig this as a starting point for my Debian based Pi 3/4 hosts.

**Problems!** See issue #1. I've been all over the map with this and while trying to sort the issue with disappearing containers, have encountered issues with the OS seeming to lose connection with the SSD. This is not ready for use.

* `Read device information` results in `"ssd_info.stdout_lines": "VARIABLE IS NOT DEFINED!"` (At present not needed)
* Playbooks as presently coded are working for a Pi 4B/8GB. There is room for improvement. (Have been tested with Pi 3B/3B+ and work.)
* `second-boot-Debian.yml` supports Bullseye and is no longer supported and has known bugs. If you really need to use it, some fixes will be needed. Feel free to file an issue and submit a PR.

## TODO

* Decide if a playbook or script is more suitable for migrating Docker containers from `polana` to `polana2`. This involves running commands on both systems and automation is set aside for now.
* Produce a playbook for typical user settings.
* Disable root login? Need to be sure `sudo` is working.
* Maybe I should version and release?
* Configure WiFi.
* Test with Bookworm.- done Bullseye no longer suypported.
* Test on Pi 3B/3B+ - done and working.

## Phase 1 - provision SSD

***NOTE: Be absolutely certain that the correct SSD device is provided. (Yeah, I did.)***

### provision-Debian.yml

```text
ansible-playbook provision-Debian.yml -b -K --extra-vars "ssd_dev=/dev/sdc \
    os_image=/home/hbarta/Downloads/Pi/Debian/20230425_raspi_4_bullseye.img.xz \
    new_host_name=polana2 poolname=polana_tank part_prefix=p\
    eth_mac=dc:a6:32:bf:65:b7 wifi_mac=dc:a6:32:bf:65:b8"
```

Command w/out reloading the image, mostly for testing

```text
ansible-playbook provision-Debian.yml -b -K --extra-vars "ssd_dev=/dev/sdc \
    new_host_name=polana2 poolname=tank part_prefix=p\
    eth_mac=dc:a6:32:bf:65:b7 wifi_mac=dc:a6:32:bf:65:b8"
```

(Please note difference for spoofing between `provision-Debian.yml` and `provision-Debian-lite.yml` until the latter is modified to match on `Driver`)

### first-boot-Debian.yml

The Debian install does not include Python so this playbook installs it so subsequent playbooks can use regular playbook tasks.

*Note: SSH to `root@new_host_name` from command line first or this playbook will fail.*

```text
ansible-playbook first-boot-Debian.yml -i inventory -u root
```

### second-boot-Debian.yml, second-boot-bookworm-Debian.yml

This performs the bulk of the setup and configuration.

```text
ansible-playbook second-boot-Debian.yml -i inventory -u root \
     --extra-vars "poolname=tank"
ansible-playbook second-boot-bookworm-Debian.yml -i inventory -u root \
     --extra-vars "poolname=tank"
ansible-playbook second-boot-bookworm-lite-Debian.yml \
    -i inventory -l trixi -u root
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

Since the 2nd partition ends at `62914559s` the start of the third partition is `629145660s`.

## Contributing

I would be delighted if someone found this interesting and useful enough to tweak it and contribute their changes back. This includes anything from fixing typos to refactoring the playbooks and parameterizing the (too many) hard coded things. Of course I reserve the right to reject any PRs that compromise my ability to use this, but I doubt that is going to happen.
