# LVM and Filesystems - Homework 03

## Assignment
- Create separate volume for /home
- Create mirrored volume for /var
- Configure /etc/fstab for auto-mounting
- Work with snapshots: create files, snapshot, delete files, restore

---

## Environment

| Component | Details |
|-----------|---------|
| Host OS | macOS (Apple Silicon M3) |
| Virtualization | UTM with Apple Virtualization |
| Guest OS | Ubuntu 22.04.5 Server ARM64 |

---

## Part 1: Create Mirrored Volume for /var

### Create Physical Volumes
```bash
pvcreate /dev/vdb /dev/vdc
```

### Create Volume Group
```bash
vgcreate vg_var /dev/vdb /dev/vdc
```

### Create Mirrored Logical Volume
```bash
lvcreate -L 950M -m1 -n lv_var vg_var
```

### Create Filesystem and Move /var
```bash
mkfs.ext4 /dev/vg_var/lv_var
mount /dev/vg_var/lv_var /mnt
cp -aR /var/* /mnt/
mkdir /tmp/oldvar && mv /var/* /tmp/oldvar
umount /mnt
mount /dev/vg_var/lv_var /var
```

### Add to fstab
```bash
echo "$(blkid | grep lv_var | awk '{print $2}') /var ext4 defaults 0 0" >> /etc/fstab
```

---

## Part 2: Create Volume for /home

### Create Logical Volume
```bash
lvcreate -n lv_home -L 2G /dev/ubuntu-vg
```

### Create Filesystem and Move /home
```bash
mkfs.ext4 /dev/ubuntu-vg/lv_home
mount /dev/ubuntu-vg/lv_home /mnt
cp -aR /home/* /mnt/
rm -rf /home/*
umount /mnt
mount /dev/ubuntu-vg/lv_home /home
```

### Add to fstab
```bash
echo "$(blkid | grep lv_home | awk '{print $2}') /home ext4 defaults 0 0" >> /etc/fstab
```

---

## Part 3: Working with Snapshots

### Generate Test Files
```bash
touch /home/file{1..20}
ls /home
```

**Output:**
```text
file1 file2 file3 ... file20
```

### Create Snapshot
```bash
lvcreate -L 100M -s -n home_snap /dev/ubuntu-vg/lv_home
```

### Delete Some Files
```bash
rm -f /home/file{11..20}
ls /home
```

**Output:**
```text
file1 file2 file3 ... file10
```

### Restore from Snapshot
```bash
umount /home
lvconvert --merge /dev/ubuntu-vg/home_snap
mount /dev/ubuntu-vg/lv_home /home
ls /home
```

**Output:**
```text
file1 file2 file3 ... file20
```

Files successfully restored!

---

## Final Configuration

### lsblk Output
```text
vda                       252:0    0   25G  0 disk
├─vda1                    252:1    0    1G  0 part /boot/efi
├─vda2                    252:2    0    2G  0 part /boot
└─vda3                    252:3    0 21.9G  0 part
  ├─ubuntu--vg-ubuntu--lv 253:0    0   11G  0 lvm  /
  └─ubuntu--vg-lv_home    253:6    0    2G  0 lvm  /home
vdb                       252:16   0    1G  0 disk
├─vg_var-lv_var_rmeta_0   253:1    0    4M  0 lvm
│ └─vg_var-lv_var         253:5    0  952M  0 lvm  /var
└─vg_var-lv_var_rimage_0  253:2    0  952M  0 lvm
  └─vg_var-lv_var         253:5    0  952M  0 lvm  /var
vdc                       252:32   0    1G  0 disk
├─vg_var-lv_var_rmeta_1   253:3    0    4M  0 lvm
│ └─vg_var-lv_var         253:5    0  952M  0 lvm  /var
└─vg_var-lv_var_rimage_1  253:4    0  952M  0 lvm
  └─vg_var-lv_var         253:5    0  952M  0 lvm  /var
```

### lvs Output
```text
LV        VG        Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
lv_home   ubuntu-vg -wi-ao----   2.00g
ubuntu-lv ubuntu-vg -wi-ao----  10.97g
lv_var    vg_var    rwi-aor--- 952.00m                                    100.00
```

---

## Summary

| Mount Point | Volume | Type | Size |
|-------------|--------|------|------|
| / | ubuntu-vg/ubuntu-lv | ext4 | 11G |
| /home | ubuntu-vg/lv_home | ext4 | 2G |
| /var | vg_var/lv_var | ext4 mirror | 952M |

---

## References

- https://wiki.ubuntu.com/Lvm
- https://linux.die.net/man/8/lvcreate
