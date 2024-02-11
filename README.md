## Setup Raid 10 on EC2 with ansible.
##### This playbook performs the following tasks:

* Install mdadm: Installs the RAID management tool.
* Zero superblocks: Clears existing RAID configurations on the disks.
* Create RAID 10 array: Combines the disks into a RAID 10 array.
* Create filesystem: Formats the RAID array with the ext4 filesystem.
* Mount RAID array: Mounts the RAID array to /mnt/raid10.
* Save RAID configuration: Ensures the RAID setup persists across reboots.
* Verify RAID configuration.
* Check all RAID arrays.

1. Login to EC2 and list available disk.
```bash
[root@ip-172-31-25-139 ~]# fdisk -l
Disk /dev/xvda: 40 GiB, 42949672960 bytes, 83886080 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: AF668AF2-2DD9-47EF-9CD5-26F76EB55FE5

Device       Start      End  Sectors Size Type
/dev/xvda1   24576 83886046 83861471  40G Linux filesystem
/dev/xvda127 22528    24575     2048   1M BIOS boot
/dev/xvda128  2048    22527    20480  10M EFI System

Partition table entries are not in disk order.


Disk /dev/xvdd: 40 GiB, 42949672960 bytes, 83886080 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/xvde: 40 GiB, 42949672960 bytes, 83886080 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/xvdb: 40 GiB, 42949672960 bytes, 83886080 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/xvdc: 40 GiB, 42949672960 bytes, 83886080 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/xvdf: 40 GiB, 42949672960 bytes, 83886080 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```
2. Run playbook.

```bash
ansible-playbook setup_raid10_local.yml
```

3. setup_raid10_local.yml

```bash
---
- name: Configure RAID 10 Locally
  hosts: localhost
  become: true
  gather_facts: no
  tasks:
    - name: Install mdadm
      package:
        name: mdadm
        state: present

    - name: Zero superblocks on all drives
      command: mdadm --zero-superblock --force /dev/xvd{{ item }}
      loop: ['b', 'c', 'd', 'f']

    - name: Create RAID 10 array
      command: >
        mdadm --create --verbose /dev/md0 --level=10 --raid-devices=4
        /dev/xvdb /dev/xvdc /dev/xvdd /dev/xvdf

    - name: Create filesystem on RAID array
      filesystem:
        fstype: ext4
        dev: /dev/md0

    - name: Ensure the RAID array is mounted
      mount:
        path: /mnt/raid10
        src: "/dev/md0"
        fstype: "ext4"
        state: mounted

    - name: Save RAID configuration
      command: bash -c "mdadm --detail --scan >> /etc/mdadm.conf"

    # Task to check all RAID arrays
    - name: Verify RAID 10 setup
      command: mdadm --detail /dev/md0
      register: raid_detail

    - name: Print RAID detail
      debug:
        msg: "{{ raid_detail.stdout_lines }}"

    - name: Check All RAID Arrays
      command: cat /proc/mdstat
      register: raid_status

    - name: Print RAID Arrays Status
      debug:
        msg: "{{ raid_status.stdout }}"
``` 

4. Results.

![Verify RAID configuration](images/ArrayStatus.png)

![Check all RAID arrays.](images/VerifyRaid.png)
