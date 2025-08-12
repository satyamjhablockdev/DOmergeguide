# DigitalOcean volume merge guide ⚡
#### ~Satyam Jha
You can  create five **100 GB** DigitalOcean block volumes and **combine them into one big storage pool** so they act as a single filesystem.

#### We will mount **all five DO block volumes as one big filesystem** using LVM. Below is a safe, copy-paste guide. It will **erase** those new volumes, which is fine since they’re unused. Still DYOR ⚠ 
---

# Goal

To Create one 500-GB filesystem at `/mnt/data` from:

`
volume-sfo2-01, volume-sfo2-02, volume-sfo2-03, volume-sfo2-04, volume-sfo2-05
`

![Volumes](Screenshot%202025-08-12%20223959.png)



(You can later move big folders to `/mnt/data` or even migrate `/` to it.)

---

# Steps (DigitalOcean Ubuntu)
<H3>Note: First You need to create 5x100 GB manually format & mount Volumes:</H3>

![Volumesguide](Screenshot%202025-08-12%20225648.png)
### 0) Prep

```bash
sudo apt-get update
sudo apt-get install -y lvm2 parted gdisk
```

### 1) Identify the five attached volumes

DigitalOcean exposes stable “by-id” names. Check them:

```bash
ls -l /dev/disk/by-id | grep DO_Volume
```

You should see entries like:

`
/dev/disk/by-id/scsi-0DO_Volume_volume-sfo2-01 -> ../../vdb
...
`

### 2) Put them in a list (edit if your names differ)

```bash
VOLS=(
/dev/disk/by-id/scsi-0DO_Volume_volume-sfo2-01
/dev/disk/by-id/scsi-0DO_Volume_volume-sfo2-02
/dev/disk/by-id/scsi-0DO_Volume_volume-sfo2-03
/dev/disk/by-id/scsi-0DO_Volume_volume-sfo2-04
/dev/disk/by-id/scsi-0DO_Volume_volume-sfo2-05
)

```

### 3) Partition each disk for LVM (DESTROYS data on them)

```bash
for VOL in "${VOLS[@]}"; do
  DEV=$(readlink -f "$VOL")           # resolve to /dev/vdb, /dev/vdc, ...
  sudo wipefs -a "$DEV"
  sudo sgdisk --zap-all "$DEV"
  sudo parted -s "$DEV" mklabel gpt
  sudo parted -s "$DEV" mkpart primary 1MiB 100%
  sudo parted -s "$DEV" set 1 lvm on
done
sudo partprobe
```

### 4) Create LVM PVs, VG, and one big LV

```bash
# Build a list of the new partitions (…1 on each device)
PVS=()
for VOL in "${VOLS[@]}"; do
  DEV=$(readlink -f "$VOL")
  PVS+=("${DEV}1")
done

# Make PVs and the VG
sudo pvcreate "${PVS[@]}"
sudo vgcreate bigpool "${PVS[@]}"

# Make one big logical volume
sudo lvcreate -l 100%FREE -n data bigpool
```

### 5) Create a filesystem and mount it

```bash
sudo mkfs.ext4 /dev/bigpool/data       # (use mkfs.xfs if you prefer XFS)
sudo mkdir -p /mnt/data
sudo mount /dev/bigpool/data /mnt/data
```

### 6) Make it persistent (fstab)

```bash
UUID=$(blkid -s UUID -o value /dev/bigpool/data)
echo "UUID=$UUID  /mnt/data  ext4  defaults,nofail,discard  0  0" | sudo tee -a /etc/fstab

# (Optional) enable weekly TRIM, good for SSD-backed volumes
sudo systemctl enable --now fstrim.timer
```

### 7) Verify

```bash
lsblk
pvs; vgs; lvs
df -h /mnt/data
```
![Description of image](Screenshot%202025-08-12%20224222.png)

You should see `/mnt/data` ≈ **500 GB**.

---

## What next?

* Start **using `/mnt/data`** for heavy stuff:

  * Docker: move `/var/lib/docker`
  * Databases: `/var/lib/postgresql`, `/var/lib/mysql`
  * App data/logs: `/srv`, `/var/www`, `/var/log`
    Use rsync + symlink or bind mounts. I can give exact commands for whichever you use.
  * Star 🌟 the repo last and IMP step 😅
