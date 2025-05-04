---
title: "Khởi tạo EC2 và chạy web server cơ bản - Phần 2"
comments: true
date: 2025-04-29 11:33:00 +0700
categories: [Cloud Computing, Elastic Compute Cloud]
tags: [ec2, vpc, ssh]
mermaid: true
image: 
  path: /assets/images/2025-04-30-tao-ec2-instance-p2/01-lab-ec2.drawio.svg
  alt: "Mô hình triển khai EC2 cơ bản trong Public Subnet"
---

Mình sẽ thực hành bài lab trong quyển này ở trang 148 vì nó đáp ứng đầy đủ các mục tiêu khi làm quen với EC2.

<p>
    <a href="https://ptmkhanh29.github.io/tutorial-aws-labs/assets/files/AWS-Certified-Solutions-Architect-Associate-EXAM-GUIDE-SAA-C01.pdf" 
       target="_blank"
       rel="noopener noreferrer">
       AWS Certified Solutions Architect Associate EXAM-GUIDE (Exam SAA-C01).pdf
    </a>
</p>

![Image1](assets/images/2025-04-30-tao-ec2-instance-p2/description-lab.png)

**🎯 Mục tiêu bài lab**

- Hiểu được cách **tạo mới một EC2 Instance** trên AWS bằng giao diện Management Console.
- Biết cách **chọn VPC, subnet, AMI, instance type** một cách hợp lý.
- Biết cách **cấu hình firewall (Security Group)** để mở port SSH và HTTP phục vụ web server.
- Biết cách **gắn Elastic IP** để giữ IP cố định cho EC2.
- Thực hành **kết nối SSH vào EC2 instance** từ máy local.
- Cài đặt và cấu hình **Apache hoặc Nginx** làm Web Server trên Amazon Linux.
- Tạo và triển khai một **trang HTML đơn giản**, sau đó kiểm tra trên trình duyệt.

**🌐 Mối liên hệ giữa EC2, Public Subnet, VPC và Region**

Region → Chứa nhiều VPC → VPC chứa nhiều Subnet (Public/Private) → Subnet chứa EC2 Instance

<div class="mermaid">
flowchart LR
    A(fa:fa-globe Region) --> B(fa:fa-network-wired VPC)
    B --> C(fa:fa-cloud Public Subnet)
    C --> D(fa:fa-shield-alt Security Group)
    D --> E(fa:fa-server EC2 Instance)
    B --> F(fa:fa-exchange-alt Internet Gateway)
    F --> E
</div>

Sau khi tạo thành công EC2 instance, bước tiếp theo mình sẽ triển khai một Web Server để phục vụ trang web

---

## Các cách triển khai Web Server

### Sử dụng User Data 

> **User Data** trong EC2 là một đoạn script (bash script, cloud-init script, hoặc đơn giản là các command) mà bạn có thể cung cấp cho EC2 instance khi khởi tạo.  
> Khi instance được **launch** lần đầu tiên, EC2 sẽ tự động chạy đoạn User Data này **một lần duy nhất**.  
> Thông thường, User Data được dùng để:
>
> - Cài đặt phần mềm
>
> - Cấu hình hệ thống
>
> - Tải file từ internet
>
> - Khởi động dịch vụ...
{: .prompt-tip }

- Sau bước `Configure storage` là phần `Advanced Details`, bạn click vào `Advanced Details` để expand nó ra sẽ thấy **User Data**

![Image](assets/images/2025-04-30-tao-ec2-instance-p2/1.png)

- Dán đoạn script bạn muốn chạy vào đây. Ví dụ:


```bash
#!/bin/bash
sudo yum update -y
sudo yum install -y httpd
sudo systemctl start httpd
sudo systemctl enable httpd
echo "Hello from EC2" | sudo tee /var/www/html/index.html
```

> **Lưu ý:** Đừng quên bắt đầu script với `#!/bin/bash` (hoặc `#!/bin/sh`, `#!/bin/cloud-config` tùy loại).
{: .prompt-warning }

![Image](assets/images/2025-04-30-tao-ec2-instance-p2/2.png)

Sau đó nhấn nút `Launch instance` như bình thường để tạo EC2 instance.

> Vì instance EC2 đã được khởi tạo trước, chúng ta không thể sử dụng User Data để tự động cài đặt và cấu hình web server. 
>
> User Data chỉ được áp dụng khi instance khởi động lần đầu. 
>
> →  Do đó, mình sẽ SSH vào máy và thực hiện các lệnh cần thiết trực tiếp, giúp dễ dàng kiểm soát và giải thích các bước cài đặt blog một cách chi tiết hơn khi demo.
{: .prompt-warning }
---

### SSH vào Instance và tự cài đặt Web Server

#### 1. Cập nhật hệ thống

``` sh
# Amazon Linux
sudo dnf update -y
```

![Image](assets/images/2025-04-30-tao-ec2-instance-p2/11.png)

#### 2. Cài đặt Web Server

- Nếu dùng **Apache**:

``` sh
sudo dnf install httpd -y
```

- Nếu dùng **Nginx**:

``` sh
sudo dnf install nginx -y
```

![Image](assets/images/2025-04-30-tao-ec2-instance-p2/12.png)

#### 3. Khởi động và kích hoạt dịch vụ
- Nếu dùng **Apache**:

``` sh
# Amazon Linux
sudo systemctl start httpd
sudo systemctl enable httpd
```

![Image](assets/images/2025-04-30-tao-ec2-instance-p2/17.png)

Sử dụng lệnh `sudo systemctl status httpd` để kiểm tra trang trạng thái service Apache. Nếu bạn thấy `active (running)` như ảnh nghĩa là service đã được khởi động thành công.

![Image](assets/images/2025-04-30-tao-ec2-instance-p2/18.png)

- Nếu dùng **Nginx**:

```sh 
sudo systemctl start nginx
sudo systemctl enable nginx
```

Sử dụng lệnh `sudo systemctl status nginx` để kiểm tra trang trạng thái service nginx. Nếu bạn thấy `active (running)` như ảnh nghĩa là service đã được khởi động thành công.

![Image](assets/images/2025-04-30-tao-ec2-instance-p2/13.png)

#### 4. Tạo file HTML demo

Sau khi Web Server hoạt động, bạn cần tạo một file `.html` đơn giản để kiểm tra hoạt động.

> Khi truy cập `http://<EC2 Public IP>`, Web Server sẽ tìm file `index.html` trong thư mục mặc định (Document Root):  
> - **Apache:** `/var/www/html/index.html`  
> - **Nginx:** `/usr/share/nginx/html/index.html`
{: .prompt-tip }

- Đối với Apache:

Mở file bằng `sudo nano /var/www/html/index.html`

Sau đó xóa nội dung cũ và dán nội dung sau vào file và lưu lại:

```html
<!DOCTYPE html>
<html>
  <head>
    <title>Apache Demo</title>
  </head>
  <body>
    <h1>Welcome to my Apache Web Server on EC2!</h1>
  </body>
</html>
```

- Đối với Nginx:

> Khi cài Nginx, file `/usr/share/nginx/html/index.html` sẽ chứa trang mặc định "Welcome to nginx!".  
> Bạn có thể sửa hoặc thay thế nội dung trong file này để tạo trang web của riêng mình.
{: .prompt-tip }

![Mặc định của Nginx](assets/images/2025-04-30-tao-ec2-instance-p2/14.png)
_Trang mặc định của Nginx_

Mở file `sudo nano /usr/share/nginx/html/index.html`

Xóa nội dung cũ, dán nội dung html mới vào và lưu lại:

```html
<!DOCTYPE html>
<html>
  <head>
    <title>Nginx Demo</title>
  </head>
  <body>
    <h1>Welcome to my Nginx Web Server on EC2!</h1>
  </body>
</html>
```

![Image](assets/images/2025-04-30-tao-ec2-instance-p2/15.png)

#### 5. Kiểm tra kết quả

Truy cập địa chỉ:

```plaintext
http://<Public-IP>
```

Nếu hiện trang "Welcome" như đã tạo, thì bạn đã triển khai thành công Web Server trên EC2!

- Đối với Apache:

![Image](assets/images/2025-04-30-tao-ec2-instance-p2/19.png)

- Đối với Nginx:

![Image](assets/images/2025-04-30-tao-ec2-instance-p2/16.png)


