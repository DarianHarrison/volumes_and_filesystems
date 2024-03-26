```sh
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
