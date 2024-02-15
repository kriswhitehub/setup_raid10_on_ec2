## Configuring Raid 10 on EC2 EBS with Ansible.
### Raids

1. RAID0 = Striping
2. RAID1 = Mirroring
3. RAID5 = Single Disk Distributed Parity
4. RAID6 = Double Disk Distributed Parity
5. RAID10 = Combine of Mirror & Stripe. (Nested RAID)

![AmazonEC2](images/amzonec2.png)        ![Ansible](images/ansiblelogo.png)             ![EBS](images/ebslogo.jpg)

RAID 10, also known as RAID 1+0, is a hybrid RAID configuration that combines the features of RAID 1 (mirroring) and RAID 0 (striping). It provides a balance of redundancy, performance, and storage efficiency. Here's a breakdown of its key components and benefits:

Components of RAID 10
Mirroring (RAID 1): In mirroring, data is duplicated across two or more disks. This means every piece of data is written identically to two separate drives, ensuring that if one drive fails, a complete and immediate copy is available on the other, providing redundancy and improving data integrity.

Striping (RAID 0): In striping, data is split into blocks and spread across two or more disks without parity (data redundancy). This improves performance because multiple disks can read and write data simultaneously, but it does not provide data protection on its own.

How RAID 10 Works
RAID 10 creates a striped set from a series of mirrored drives. In a four-disk RAID 10 setup, for example, two disks would form one mirrored pair where data is duplicated, and another two disks would form a second mirrored pair. The data is then striped across these pairs.
This setup requires at least four disks to implement: two for the first mirrored pair and two for the second.
Benefits of RAID 10
High Performance: The striping aspect allows for faster data reads and writes, which is beneficial for applications requiring high disk performance.

Data Redundancy: The mirroring component provides excellent data protection. If a disk in one of the mirrored pairs fails, the system can continue to operate using the other mirrored disk without any data loss.

Improved Fault Tolerance: RAID 10 can sustain multiple simultaneous disk failures as long as no two failed disks are part of the same mirrored pair.

Good for Databases: The combination of performance and redundancy makes RAID 10 an excellent choice for database servers and applications that require both speed and data integrity.

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
