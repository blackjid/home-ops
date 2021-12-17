sudo mkfs.ext4 /dev/sda

DISK_UUID=$(sudo blkid -s UUID -o value /dev/sda)

sudo mkdir /mnt/longhorn

sudo mount -t ext4 /dev/sda /mnt/longhorn

echo UUID=`sudo blkid -s UUID -o value /dev/sda` /mnt/longhorn ext4 defaults 0 2 | sudo tee -a /etc/fstab


for i in $(seq 1 7); do
  sudo mkdir -p /mnt/${DISK_UUID}/vol${i} /mnt/local-disks/${DISK_UUID}_vol${i};
  sudo mount --bind /mnt/${DISK_UUID}/vol${i} /mnt/local-disks/${DISK_UUID}_vol${i};
done

for i in $(seq 1 7); do
  echo /mnt/${DISK_UUID}/vol${i} /mnt/local-disks/${DISK_UUID}_vol${i} none bind 0 0 | sudo tee -a /etc/fstab
done