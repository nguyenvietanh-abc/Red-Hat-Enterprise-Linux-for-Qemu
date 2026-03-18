# Red-Hat-Enterprise-Linux-for-Qemu-with_Oracle_VirtualBox_Manager  
Kiểm tra môi trường cho linux - ubuntu và RHEL
1/ Kiểm tra system (Mục đích đáp ứng yêu cầu bộ nhớ, cpu, và hostnamectl(không cần thiết))

$: cat /etc/redhat-release 
root@rhel-yocto:~# cat /etc/redhat-release 
Red Hat Enterprise Linux release 10.1 (Coughlan)

$: free  -h
root@rhel-yocto:~# free -h
               total        used        free      shared  buff/cache   available
Mem:           7.5Gi       639Mi       6.6Gi       9.7Mi       548Mi       6.9Gi
Swap:          7.9Gi          0B       7.9Gi
root@rhel-yocto:~# 

df -h && hostnamectl
lsblk -f

2/ Thực hiện mở rộng dung lượng ổ cứng ảo (resize more VLI )
on host:
$: VBoxManager modifymedium disk "/path/to/my.vdi" --resize 150000
Example: VBoxManage modifymedium disk "/home/anhviet/VirtualBox VMs/Windows10/Windows10.vdi" --resize 150000 (Tôi sử dụng hard disk cho Máy ảo win trước đây :))
-> reboot
3/ Expand LVM on RHEL
    lsblk -f
    sudo fdisk /dev/sda
    (n → tạo extended/logical partitions mới, t → đổi type 8e cho LVM, w để lưu)
    sudo partprobe
    sudo pvcreate /dev/sda7
    sudo vgextend rhel /dev/sda7
    sudo lvextend -l +100%FREE /dev/mapper/rhel-root
    sudo xfs_growfs /
    df -h /
    free -h
    vgs
    lvs

4. Đặt hostname 
    sudo hostnamectl set-hostname rhel-yocto
    sudo reboot

5. Tạo user (yocto-dev) 
    sudo useradd -m -s /bin/bash yocto-dev
    sudo passwd yocto-dev
    sudo usermod -aG wheel yocto-dev
    su - yocto-dev

6. Set up tools basic
    sudo dnf install -y vim wget net-tools nfs-utils firewalld openssh-server
    htop không có trong repo mặc định (tự build :))

7. Caasau hình SSH
    sudo systemctl enable --now sshd
    sudo firewall-cmd --permanent --add-service=ssh
    sudo firewall-cmd --reload
    ip addr show | grep "inet " | grep -v 127

8. Cấu hình NFS server 
    sudo mkdir -p /nfs/yocto-rootfs
    sudo chown -R yocto-dev:yocto-dev /nfs
    sudo chmod 777 /nfs/yocto-rootfs
    echo "/nfs/yocto-rootfs *(rw,sync,no_root_squash,no_subtree_check)" | sudo tee -a /etc/exports
    sudo exportfs -ra
    sudo systemctl enable --now nfs-server
    sudo firewall-cmd --permanent --add-service=nfs --add-service=mountd --add-service=rpc-bind
    sudo firewall-cmd --reload
    sudo exportfs -v
    sudo ss -tuln | grep -E '2049|111'

9. Test NFS từ host tới VM
    sudo apt update && sudo apt install -y nfs-common
    sudo mkdir -p /mnt/rhel-nfs
    sudo mount -t nfs 192.168.1.4:/nfs/yocto-rootfs /mnt/rhel-nfs
    touch /mnt/rhel-nfs/test-from-ubuntu.txt
    ls /mnt/rhel-nfs
    sudo umount /mnt/rhel-nfs

10. Kết quả như được thể hiện từ hình ảnh: 
![lvs](images/rootfsLVS.png)