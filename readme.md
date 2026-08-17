quick evaluation
```bash
lsblk
sudo parted /dev/sda print free
sudo pvs /dev/sda3
sudo vgs rootvg
sudo lvs /dev/mapper/rootvg-opt
df -h /opt
```

CREATE LOLGICAL VOLUME AND MOUNT FILESYSTEM
```bash
################################################
# CREATE LOLGICAL VOLUME AND MOUNT FILESYSTEM
################################################

# find and empty disk
lsblk

# to lvm entire disk
pvcreate /dev/sdb
pvs # proof
pvdisplay /dev/sdb # proof

# create volume group (for this case we will only have /dev/sdb device in this volume group)
vgcreate vg00 /dev/sdb
vgs # proof
vgdisplay vg00 # proof

# create logical volume using remaining space in group
lvcreate -n new_lv -l 100%FREE vg00
lvs # proof
lvdisplay # locate path, in this case it will be /dev/vg00/new_lv

# create filesystem
mkfs.xfs /dev/vg00/new_lv # use path from previous command
# mkfs -t ext4 /dev/vg00/new_lv # for ext4

# mount filesystem
mkdir -p /newpartition
mount /dev/vg00/new_lv /newpartition/

# proof
df -hT
lsblk
```

EXTEND LOLGICAL VOLUME WITH NEWLY ADDED DISK
```bash
################################################
# EXTEND LOLGICAL VOLUME WITH NEWLY ADDED DISK
################################################

lsblk
pvcreate /dev/sdC
vgs
vgextend vg00 /dev/sdc
lvs
lvextend -l +100%FREE /dev/vg00/new_lv
xfs_growfs /dev/vg00/new_lv
df -hT
```

REMOVE FILESYSTEM AND LOGICAL VOLUME
```bash
################################################
# REMOVE FILESYSTEM AND LOGICAL VOLUME
################################################

# wipe signature and metadata/magic strings from the logical device
wipefs -af /dev/vg00/new_lv # note this is a logical device

# unmount filesystem
umount /newpartition

# Disable LVM
lvchange -an /dev/vg00/new_lv

# remove lv
lvremove /dev/vg00/new_lv
lvs # proof no longer exists

# disable vg
vgchange -an vg00

# remove vg
vgremove vg00
vgs # proof no longer exists

# remove PV
pvremove /dev/sdb

# proof
lsblk

# wipe signature and metadata/magic strings from the physical device
wipefs -af /dev/sdb
```

To extend a logical volume partition and filesystem after a virtual disk has been expanded 
```
################################################
# 1. FIX GPT BACKUP HEADER
################################################
# READ BEFORE: Check for the GPT warning and locate the Free Space
sudo parted /dev/sda print free

# ACTION: Safely move the GPT backup header to the end of the 256GB boundary
sudo sgdisk -e /dev/sda

# PROOF AFTER: Verify the warning is gone and the 157GB Free Space is at the bottom
sudo parted /dev/sda print free

################################################
# 2. GROW THE PHYSICAL PARTITION
################################################
# READ BEFORE: Check the current size of partition 3 (should be ~108G)
lsblk /dev/sda

# ACTION: Expand partition 3 to consume the unallocated raw space
sudo growpart /dev/sda 3

# PROOF AFTER: Verify partition 3 has grown to fill the disk (should be ~254G)
lsblk /dev/sda

################################################
# 3. RESIZE THE PHYSICAL VOLUME (LVM)
################################################
# READ BEFORE: Check current physical free space on sda3 (should be <14G)
sudo pvs /dev/sda3

# ACTION: Tell LVM the underlying partition (/dev/sda3) has grown
sudo pvresize /dev/sda3

# PROOF AFTER: Verify the new space is registered in LVM (PFree should be ~171G)
sudo pvs /dev/sda3

################################################
# 4. EXTEND LOGICAL VOLUME & FILESYSTEM
################################################
# READ BEFORE: Check current size of /opt at the LVM and OS levels (should be 10G)
sudo lvs /dev/mapper/rootvg-opt
df -h /opt

# ACTION: Expand /opt to exactly 128GB (-r handles the filesystem expansion automatically)
sudo lvextend -r -L 128G /dev/mapper/rootvg-opt

# PROOF AFTER: Confirm /opt is exactly 128GB and check remaining free space in rootvg
sudo lvs /dev/mapper/rootvg-opt
df -h /opt
sudo vgs rootvg
```
