---
title: "Tạo EBS Instance và gáng nó cho EC2 Instance"
comments: true
date: 2025-05-03 20:33:00 +0700
categories: [Storage, Elastic Block Store]
tags: [ec2, ebs, ssh]
image: 
  path: /assets/images/2025-04-30-tao-ec2-instance/01-lab-ec2.drawio.svg
  alt: "Mô hình triển khai EC2 cơ bản trong Public Subnet"
---

**Yêu cầu bài lab:** Mình trích bài Lab 4.2 trang 154

<p>
    <a href="https://ptmkhanh29.github.io/tutorial-aws-labs/assets/files/AWS-Certified-Solutions-Architect-Associate-EXAM-GUIDE-SAA-C01.pdf" 
       target="_blank"
       rel="noopener noreferrer">
       AWS Certified Solutions Architect Associate EXAM-GUIDE (Exam SAA-C01).pdf
    </a>
</p>

![Image1](assets/images/2025-05-03-tao-va-gang-ebs-cho-ec2-instance/description-lab.png)

-------------

Bạn có thể tạo thêm **EBS volume** và **gắn nó vào EC2 instance** hiện tại của mình để mở rộng dung lượng lưu trữ. Dưới đây là các bước chi tiết để thực hiện điều này.

## Kiểm tra tên device của EBS hiện tại (nếu có)

Trước khi tạo thêm EBS, bạn cần kiểm tra tên của các device EBS đã được gắn vào EC2 để đảm bảo gắn EBS mới đúng tên. Bạn có thể dùng 1 trong 2 cách sau

### Kiểm tra trong **AWS Management Console**

1. **Đăng nhập vào AWS Management Console** và truy cập **EC2 Dashboard**.
2. Trong phần **Instances**, chọn **EC2 instance** mà bạn muốn kiểm tra.
3. Cuộn xuống phần **Block Devices** trong tab **Storage** của instance.
4. Bạn sẽ thấy danh sách các **EBS volumes** đã attach vào EC2. Tên thiết bị (device name) sẽ hiển thị như `/dev/sda1`, `/dev/xvda`, v.v.

Trong EC2 Instance của mình thì device name là `/dev/xvda` với Volume size là 8GB.

![Image1](assets/images/2025-05-03-tao-va-gang-ebs-cho-ec2-instance/1.png)

### Kiểm tra trực tiếp trên **EC2 Instance**

1. SSH vào EC2 instance của bạn.
2. Sử dụng lệnh `lsblk` để liệt kê tất cả các ổ đĩa và phân vùng. Bạn sẽ thấy tên thiết bị như sau:
   
``` sh
xvda      202:0    0   8G  0 disk
├─xvda1   202:1    0   8G  0 part /
├─xvda127 259:0    0   1M  0 part
└─xvda128 259:1    0  10M  0 part /boot/efi
```
![Image1](assets/images/2025-05-03-tao-va-gang-ebs-cho-ec2-instance/2.png)

Trong đó:

- `xvda` là device name chính của volume EBS.

- `xvda1` là phân vùng được mount vào thư mục root /.

Bây giờ bạn đã biết tên device của EBS cũ và có thể gắn EBS mới vào tên khác.

## Tạo EBS volume mới

Đăng nhập vào AWS Console. Truy cập dịch vụ EC2, chọn Region đúng với Region của EC2 mà bạn cần thêm EBS Volume

Trong menu bên trái, trong section `Elastic Block Store`, chọn `Volumes`

![Image1](assets/images/2025-05-03-tao-va-gang-ebs-cho-ec2-instance/3.png)

Click "Create volume"

![Image1](assets/images/2025-05-03-tao-va-gang-ebs-cho-ec2-instance/4.png)

Cấu hình Volume:

|--------------------|--------------------------------|----------------|
| **Volume type**    | `gp3`                         | - gp3 rẻ hơn gp2 ~20% khi vượt Free Tier<br>- Hiệu năng ổn định hơn gp2 |
| **Size**           | **Tối đa 22GB**               | - Free Tier cho phép tổng 30GB EBS<br>- Đã dùng 8GB → Còn lại 22GB miễn phí |
| **Availability Zone** | Cùng AZ với EC2 cần attach           | - VD: `us-east-1a` (phải khớp chính xác) |
| **Tags**           | `Key: Name`, `Value: Data-Volume` | - Giúp quản lý cost tracking sau này |

![Image1](assets/images/2025-05-03-tao-va-gang-ebs-cho-ec2-instance/5.png)

Click `Create volume`

## Attach volume vào EC2 instance

Trong danh sách Volumes, chọn volume vừa tạo. 2. Click `Actions`, chọn `Attach volume`

![Image1](assets/images/2025-05-03-tao-va-gang-ebs-cho-ec2-instance/6.png)

Cấu hình:
   - Instance: Chọn EC2 instance muốn gắn
   - Device name: Nhập tên thiết bị (vd: /dev/sdf)

> Theo khuyến nghị chính thức từ AWS, bạn có thể chọn bất kỳ tên device nào trong khoảng `/dev/sdf` đến `/dev/sdp` (hoặc `/dev/xvdf` đến `/dev/xvdp` tùy kernel) cho data volumes
{: .prompt-tip }

Click `Attach volume`

![Image1](assets/images/2025-05-03-tao-va-gang-ebs-cho-ec2-instance/7.png)

## Kết nối với EC2 và định dạng volume

### SSH vào EC2 instance

``` sh
ssh -i "your-key.pem" ec2-user@your-instance-public-dns
```

### Kiểm tra volume mới

``` bash
lsblk
```

![Image1](assets/images/2025-05-03-tao-va-gang-ebs-cho-ec2-instance/8.png)

### Format EBS volume mới

>Volume mới như tờ giấy trắng, khi bạn tạo EBS volume từ AWS Console/API, nó chỉ là "khối lưu trữ thô" chưa có hệ thống tệp (filesystem). Giống như mua ổ cứng mới, bạn phải format (NTFS/FAT cho Windows, XFS/EXT4 cho Linux) trước khi sử dụng.
{: .prompt-info }

Bạn có thể dùng lệnh sau để kiểm tra volume trước khi format

``` sh
sudo file -s /dev/xvdf
# Output: "/dev/xvdf: data"  → Chưa có filesystem  → Cần format
```

![Image1](assets/images/2025-05-03-tao-va-gang-ebs-cho-ec2-instance/9.png)

Tiến hành format volume

``` sh
# Thay xvdf bằng device name thực tế của bạn
sudo mkfs -t xfs /dev/xvdf
sudo file -s /dev/xvdf
# Output: "/dev/xvdf: SGI XFS filesystem"  → Đã sẵn sàng dùng
```

![Image1](assets/images/2025-05-03-tao-va-gang-ebs-cho-ec2-instance/10.png)

### Tạo mount point và mount volume:

> Linux không tự động mount EBS volumes (khác với Windows). Do đó, bạn phải chỉ định mount point 
> 
> Ví dụ: `/data`, `/mnt/volume1`,..
{: .prompt-info }

``` sh
# Tạo mount point
sudo mkdir /data
```
![Image1](assets/images/2025-05-03-tao-va-gang-ebs-cho-ec2-instance/11.png)

``` sh
# Mount volume
sudo mount /dev/xvdf /data
```

Kiểm tra lại bằng lệnh

``` sh
df -h /data
```
Output hiển thị dung lượng volume đã mount

![Image1](assets/images/2025-05-03-tao-va-gang-ebs-cho-ec2-instance/12.png)

### Mount tự động khi reboot

> - Bởi vì, volume sẽ tự reject khỏi hệ thống sau khi reboot. Do đó, cần cấu hình giúp mount tự động khi khởi động.
> 
> - Chúng ta sẽ tiến hành config tệp `/etc/fstab` - là một tệp cấu hình quan trọng trên Linux, có nhiệm vụ khai báo các thiết bị lưu trữ (disk, partition) sẽ được mount tự động khi hệ thống khởi động.
{: .prompt-warning }

Lấy UUID của volume

``` sh
sudo blkid /dev/xvdf
# /dev/xvdf: UUID="b1110f5f-b6af-4479-b9f7-127adab9dc85" BLOCK_SIZE="512" TYPE="xfs"
# Ghi lại UUID: b1110f5f-b6af-4479-b9f7-127adab9dc85
```

Thêm vào `/etc/fstab` bằng UUID

``` sh
echo 'UUID=b1110f5f-b6af-4479-b9f7-127adab9dc85 /data xfs defaults,nofail 0 0' | sudo tee -a /etc/fstab
# Kiểm tra
cat /etc/fstab
```
![Image1](assets/images/2025-05-03-tao-va-gang-ebs-cho-ec2-instance/13.png)

### Kiểm tra lại

Để kiểm tra mount tự động, thử khởi động lại EC2 và kiểm tra xem Volume có được mount lại không:

``` sh
sudo reboot
```

Sau khi EC2 khởi động lại, kiểm tra:

``` sh
df -h
```

![Image1](assets/images/2025-05-03-tao-va-gang-ebs-cho-ec2-instance/14.png)

Vậy là đã hoàn thành việc tạo và gắn EBS mới vào EC2 instance!