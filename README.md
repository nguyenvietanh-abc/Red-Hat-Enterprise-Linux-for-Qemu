# RHEL on QEMU with Oracle VirtualBox Manager

Hướng dẫn thiết lập môi trường **Red Hat Enterprise Linux (RHEL)** chạy trên VirtualBox, phục vụ mục đích phát triển Yocto / embedded Linux.

---

## Mục lục

1. [Kiểm tra hệ thống](#1-kiểm-tra-hệ-thống)
2. [Mở rộng dung lượng ổ cứng ảo](#2-mở-rộng-dung-lượng-ổ-cứng-ảo)
3. [Expand LVM trên RHEL](#3-expand-lvm-trên-rhel)
4. [Đặt hostname](#4-đặt-hostname)
5. [Tạo user phát triển](#5-tạo-user-phát-triển)
6. [Cài đặt công cụ cơ bản](#6-cài-đặt-công-cụ-cơ-bản)
7. [Cấu hình SSH](#7-cấu-hình-ssh)
8. [Cấu hình NFS Server](#8-cấu-hình-nfs-server)
9. [Test NFS từ Host tới VM](#9-test-nfs-từ-host-tới-vm)
10. [Kết quả](#10-kết-quả)

---

## 1. Kiểm tra hệ thống

Xác nhận phiên bản OS, RAM, và dung lượng đĩa trước khi bắt đầu.

```bash
cat /etc/redhat-release
```
```
Red Hat Enterprise Linux release 10.1 (Coughlan)
```

```bash
free -h
```
```
               total        used        free      shared  buff/cache   available
Mem:           7.5Gi       639Mi       6.6Gi       9.7Mi       548Mi       6.9Gi
Swap:          7.9Gi          0B       7.9Gi
```

```bash
df -h && hostnamectl
lsblk -f
```

---

## 2. Mở rộng dung lượng ổ cứng ảo

Thực hiện **trên máy host** trước khi khởi động VM.

```bash
VBoxManage modifymedium disk "/path/to/your.vdi" --resize 150000
```

> **Ví dụ:**
> ```bash
> VBoxManage modifymedium disk "/home/anhviet/VirtualBox VMs/RHEL/rhel.vdi" --resize 150000
> ```
> *(150000 MB ≈ 150 GB)*

Sau đó **reboot** VM.

---

## 3. Expand LVM trên RHEL

Sau khi VM khởi động lại, mở rộng LVM để sử dụng không gian vừa cấp phát.

```bash
# Kiểm tra phân vùng hiện tại
lsblk -f

# Tạo phân vùng mới với fdisk
sudo fdisk /dev/sda
# Trong fdisk:
#   n  → tạo phân vùng mới
#   t  → đổi type sang 8e (Linux LVM)
#   w  → lưu và thoát

# Cập nhật bảng phân vùng
sudo partprobe

# Thêm physical volume mới vào LVM
sudo pvcreate /dev/sda7
sudo vgextend rhel /dev/sda7

# Mở rộng logical volume và filesystem
sudo lvextend -l +100%FREE /dev/mapper/rhel-root
sudo xfs_growfs /

# Kiểm tra kết quả
df -h /
free -h
vgs
lvs
```

---

## 4. Đặt hostname

```bash
sudo hostnamectl set-hostname rhel-yocto
sudo reboot
```

---

## 5. Tạo user phát triển

Tạo user `yocto-dev` với quyền sudo.

```bash
sudo useradd -m -s /bin/bash yocto-dev
sudo passwd yocto-dev
sudo usermod -aG wheel yocto-dev

# Chuyển sang user mới
su - yocto-dev
```

---

## 6. Cài đặt công cụ cơ bản

```bash
sudo dnf install -y vim wget net-tools nfs-utils firewalld openssh-server
```

> **Lưu ý:** `htop` không có trong repo mặc định của RHEL. Cần tự build từ source hoặc thêm EPEL repo.

---

## 7. Cấu hình SSH

```bash
# Bật và kích hoạt SSH daemon
sudo systemctl enable --now sshd

# Mở firewall cho SSH
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --reload

# Kiểm tra IP của VM
ip addr show | grep "inet " | grep -v 127
```

---

## 8. Cấu hình NFS Server

Chia sẻ thư mục rootfs qua NFS để các thiết bị embedded có thể mount qua mạng.

```bash
# Tạo thư mục export
sudo mkdir -p /nfs/yocto-rootfs
sudo chown -R yocto-dev:yocto-dev /nfs
sudo chmod 777 /nfs/yocto-rootfs

# Thêm cấu hình export
echo "/nfs/yocto-rootfs *(rw,sync,no_root_squash,no_subtree_check)" | sudo tee -a /etc/exports

# Áp dụng cấu hình và khởi động NFS
sudo exportfs -ra
sudo systemctl enable --now nfs-server

# Mở firewall cho NFS
sudo firewall-cmd --permanent --add-service=nfs --add-service=mountd --add-service=rpc-bind
sudo firewall-cmd --reload

# Xác nhận
sudo exportfs -v
sudo ss -tuln | grep -E '2049|111'
```

---

## 9. Test NFS từ Host tới VM

Thực hiện **trên máy host (Ubuntu)**.

```bash
# Cài NFS client
sudo apt update && sudo apt install -y nfs-common

# Mount thử nghiệm
sudo mkdir -p /mnt/rhel-nfs
sudo mount -t nfs 192.168.1.4:/nfs/yocto-rootfs /mnt/rhel-nfs

# Tạo file test
touch /mnt/rhel-nfs/test-from-ubuntu.txt
ls /mnt/rhel-nfs

# Unmount sau khi test
sudo umount /mnt/rhel-nfs
```

> Thay `192.168.1.4` bằng IP thực của VM RHEL (lấy từ bước 7).

---

## 10. Kết quả

Sau khi hoàn tất, kết quả LVM và filesystem trông như sau:

![LVM Result](images/rootfsLVS.png)

---

## Môi trường tham khảo

| Thành phần | Chi tiết |
|---|---|
| Guest OS | RHEL 10.1 (Coughlan) |
| Hypervisor | Oracle VirtualBox |
| RAM cấp phát | 7.5 GiB |
| Disk sau resize | ~150 GB |
| NFS Export | `/nfs/yocto-rootfs` |
| Host OS (test) | Ubuntu |