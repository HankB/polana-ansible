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

## Phase 1 - provision SSD

***NOTE: Be absolutely certain that the correct SSD device is provided***

### provision-Debian.yml

```text
 ansible-playbook provision-Debian.yml -b -K --extra-vars "ssd_dev=/dev/sdb \
    os_image=/home/hbarta/Downloads/Pi/Debian/20230425_raspi_4_bullseye.img.xz \
    new_host_name=somehostname"
```
