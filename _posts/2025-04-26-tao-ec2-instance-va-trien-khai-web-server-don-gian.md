---
title: "Khởi tạo EC2 và chạy web server cơ bản - Phần 1"
comments: true
date: 2025-04-26 11:33:00 +0700
categories: [Cloud Computing, Elastic Compute Cloud]
tags: [ec2, vpc]
mermaid: true
image: 
  path: /assets/images/2025-04-30-tao-ec2-instance/01-lab-ec2.drawio.svg
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

![Image1](assets/images/2025-04-30-tao-ec2-instance/description-lab.png)

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

-----------------

## Yêu cầu

Trước khi bắt đầu, bạn cần:

- Một tài khoản AWS (mình dùng free tier).
- Đăng nhập AWS Management Console.
- Chọn khu vực (region) phù hợp. Mình chọn khu vực Singapore thì sẽ là `ap-southeast-1`
- SSH client (trên Linux/macOS dùng Terminal, trên Windows có thể dùng Git Bash hoặc PuTTY, Mobaxterm).
- Hiểu cơ bản về Virtual Private Cloud (VPC), Subnet (public và private), và Security Groups.

-----------------

## Các bước thực hiện

Để tạo một EC2 instance cơ bản trên AWS, bạn cần thực hiện các bước sau:

- Lựa chọn Amazon Machine Image (AMI) phù hợp (Linux/Windows)
 
- Chọn Instance Type (t2.micro cho free tier)
 
- Chọn VPC và Subnet. Kích hoạt Public IP nếu cần

- Cấu hình storage

- Thêm tags (optional)

- Cấu hình Security Group: thiết lập firewall, mở port SSH (22), HTTP (80), HTTPS (443)

- Khởi chạy instance: tạo hoặc chọn EC2 `Key Pair` để kết nối, sau đó xác nhận và launch instance


### 1. Truy cập dịch vụ EC2

Truy cập AWS Management Console. Đăng nhập bằng tài khoản AWS của bạn

Trong AWS Console, tìm kiếm "EC2" hoặc chọn **Services** > **Compute** > **EC2**

![Image1](assets/images/2025-04-30-tao-ec2-instance/3.png)

### 2. Chọn Region phù hợp

Mình ở Việt Nam có thể chọn region sau:

`Asia Pacific (Singapore) ap-southeast-1` → Ping thấp nhất (~30-50ms)

![Image1](assets/images/2025-04-30-tao-ec2-instance/2.png)

### 3. Khởi tạo EC2 instance

Trong trang EC2 Dashboard, chọn **Launch Instance**

![Image1](assets/images/2025-04-30-tao-ec2-instance/4.png)

Nhập tên instance ở phần Name and tags (VD: `01-lab-ec2`)

![Image1](assets/images/2025-04-30-tao-ec2-instance/5.png)

### 4. Chọn Amazon Machine Image (AMI)

Chọn AMI phù hợp (thường dùng):

  - **Amazon Linux 2023** (miễn phí)
  
  - **Ubuntu Server 22.04 LTS** (phổ biến cho dev)
  
  - **Windows Server** (có phí)

Mình chọn Amazon Linux 2023 version free tier. Nếu bạn nào muốn dùng Ubuntu cũng được nhưng nếu sử dụng free tier thì nhớ lưu ý nó cần phải hiện tại Free tier bên cạnh nhé

![Image1](assets/images/2025-04-30-tao-ec2-instance/6.png)

### 5. Chọn instance type

Mình dùng `t2.micro` là loại general purpose instance nằm trong Free Tier của AWS.

>`t2.micro` cung cấp cấu hình tối thiểu đủ dùng cho bài lab: 1 vCPU và 1 GiB RAM.
{: .prompt-tip }

![Image1](assets/images/2025-04-30-tao-ec2-instance/8.png)

Nếu bạn muốn dùng instance type khác có thể tham khảo bảng sau

| Instance Type Family     | Instance Types                  |
|--------------------------|----------------------------------|
| General Purpose          | T3, T2, M5, M4                   |
| Compute Optimized        | C5, C4                           |
| Memory Optimized         | X1e, X1, R5, R4, z1d             |
| Accelerated Computing    | P3, P2, G3, F1                   |
| Storage Optimized        | H1, I3, D2                       |

### 6. Tạo key pair

> Key pair trong EC2 là cặp khóa mã hóa dùng để xác thực khi bạn SSH vào EC2 instance. Gồm hai phần:
>
> + Private key (.pem hoặc .ppk): bạn lưu lại trên máy cá nhân.
>
> + Public key: AWS tự động gán vào EC2 instance khi bạn tạo.
{: .prompt-info }
Bạn có thể tạo key pair trước khi vào tạo Instance, hoặc tạo trực tiếp trong quá trình tạo Instance thông qua nút `Create new key pair`

Chọn "Create new key pair"

![Image1](assets/images/2025-04-30-tao-ec2-instance/9.png)

+ Đặt tên (VD: `01-lab-ec2-keypair`)

+ Key pair type: chọn RSA (.pem) 

+ Private key file format: chọn .pem

Sau đó nhấn `Create key pair`

**Lưu ý**: Download key pair về máy và lưu giữ cẩn thận

![Image1](assets/images/2025-04-30-tao-ec2-instance/10.png)

Kết quả 

![Image1](assets/images/2025-04-30-tao-ec2-instance/11.png)

### 7. Cấu hình Network Settings

🌐 Cấu hình Network Settings gồm những gì?

| Mục cấu hình              | Mô tả                                                                  |
|---------------------------|--------------------------------------------------------------------------------|
| **Network**                  | Chọn Virtual Private Cloud (mạng ảo) để instance nằm trong đó.                  |
| **Subnet**               | Chọn subnet cụ thể trong VPC (thường là Public Subnet nếu cần gán IP Public). |
| **Auto-assign Public IP**| Bật nếu muốn truy cập instance từ Internet qua SSH.                            |
| **Firewall (Security Group)** | Chọn hoặc tạo Security Group để mở cổng (VD: SSH, HTTP).                  |


> Trong bài lab này chúng ta sẽ sử dụng VPC, Subnet và Security Group `default` mà AWS đã tạo sẵn cho mỗi AWS Region.
{: .prompt-warning }

![Image1](assets/images/2025-04-30-tao-ec2-instance/12.png)

Nhấn nút `Edit` để tiến hành cấu hình

**- VPC - required:** Chọn default VPC (nếu bạn đã có VPC riêng có thể chọn VPC của mình).

**- Subnet:** Chọn subnet có Availability Zone tùy thích (VD: ap-southeast-1a)

**- Auto-assign Public IP:** Enable để EC2 được cấp IP Public

> **Lưu ý về VPC và Subnet default**:
>
> - Mỗi `Region` đều có một `Default VPC`, và mỗi `Availability Zone (AZ)` trong region đó sẽ có `Subnet default` 
>
> - Các `Subnet default` này đều đã được cấu hình như **Public subnet**: có route `0.0.0.0/0` ra **Internet Gateway (IGW)**, cho phép EC2 instance bên trong giao tiếp với internet.
>
> - Khi bạn tạo EC2 instance trong default subnet, chỉ cần bật **Auto-assign Public IP** là instance có thể SSH từ ngoài vào hoặc truy cập Internet mà **không cần cấu hình IGW hoặc route table thủ công**
{: .prompt-tip }

![Image1](assets/images/2025-04-30-tao-ec2-instance/13.png)

**- Security Group**:

> Security Group là tường lửa ảo (virtual firewall) dùng để kiểm soát lưu lượng vào và ra cho EC2 instance.
>
> - Mỗi **VPC** sẽ có một **Security Group default**, và **Security Group này áp dụng được cho tất cả các subnet** nằm trong cùng VPC đó – dù là subnet mặc định hay subnet do bạn tự tạo sau này.
>
> - Khi bạn tạo EC2 instance trong default subnet mà không chọn Security Group nào, AWS sẽ tự động gán **default Security Group của VPC đó** cho instance.
>
> - Bạn có thể tạo Security Group riêng cho từng instance nếu muốn (vào **EC2 Dashboard > Security Groups > Create Security Group**), nhưng trong bài lab này mình sẽ dùng **Security Group default**.
{: .prompt-info }


![Image1](assets/images/2025-04-30-tao-ec2-instance/14.png)

> Trong lab này yêu cầu từ máy host cá nhân SSH đến EC2 instance nên Security Group cần mở port 22 để allow SSH. Phần này sau khi tạo Instance xong chúng ta sẽ quay lại để chỉnh sửa sau.
{: .prompt-warning }

### 8. Cấu hình Storage

Khi tạo một EC2 instance, bạn cần gắn kèm ổ lưu trữ cho EC2 instance của mình. AWS gọi các ổ đĩa này là **EC2 Storage Volumes**.

**📦 EC2 Storage Volumes là gì?**

- Là nơi lưu trữ dữ liệu cho instance EC2: hệ điều hành, ứng dụng, file, cơ sở dữ liệu,...
- Có thể gắn nhiều volume vào một instance nếu cần.
- Khi tạo mới instance, AWS sẽ yêu cầu bạn cấu hình ít nhất một volume.

AWS cung cấp hai loại storage chính cho EC2:

| Loại Storage             | Mô tả |
|---------------------------|-------|
| **EBS (Elastic Block Store)** | Ổ đĩa mạng, có thể tháo gỡ hoặc gắn vào instance khác, **giữ dữ liệu** sau khi instance bị terminate. |
| **Instance Store**        | Ổ đĩa vật lý gắn trực tiếp vào phần cứng server, **cực nhanh** nhưng **mất dữ liệu** khi instance bị terminate hoặc stop. |

> **Lưu ý**: Các instance nhỏ như `t2.micro` **chỉ hỗ trợ EBS**, không có Instance Store.
> 
> AWS mặc định gắn 1 ổ **EBS** vào instance:
>
> - **Volume Type**: `gp3` (SSD chuẩn phổ thông, hiệu suất cao, giá rẻ)
>
> - **Size**: 8 GiB (có thể chỉnh tùy ý, tối đa 30GB)
{: .prompt-tip }

> Free Tier chỉ miễn phí 30GB tổng EBS storage (gp2/gp3).
>
> Nếu tạo nhiều volume, tổng vượt 30GB sẽ bị tính phí.
>
> Để tránh bị tính phí, hãy xóa volume sau khi terminate instance (nếu không cần backup)
{: .prompt-warning }

![Image1](assets/images/2025-04-30-tao-ec2-instance/15.png)

### 9. Review & Launch EC2 Instance

Trước khi launch, kiểm tra các thiết lập quan trọng

![Image1](assets/images/2025-04-30-tao-ec2-instance/16.png)

Click `Launch Instance`. Chờ khoảng 2-3 phút để nó khởi tạo

EC2 instance đã được tạo thành công.

![Image1](assets/images/2025-04-30-tao-ec2-instance/17.png)

### 10. Sửa Security Group để SSH đến EC2

Chọn instance vừa tạo. Click vào tab `Security` sau đó click vào Security Group của EC2 để tiến hành edit rule

![Image1](assets/images/2025-04-30-tao-ec2-instance/18.png)

Tại mục `Inbound rules`, chọn `Edit inbound rules`

![Image1](assets/images/2025-04-30-tao-ec2-instance/19.png)

Cấu hình rule SSH:

```yaml
Type: SSH
Protocol: TCP
Port range: 22
Source: [Chọn một]
 - My IP (AWS tự điền IP của bạn)
 or 
 - Custom: 0.0.0.0/0 (cho phép mọi IP - chỉ dùng test)
```

Sau đó nhấn vào nút `Save rules`

![Image1](assets/images/2025-04-30-tao-ec2-instance/20.png)

Kết quả:

![Image1](assets/images/2025-04-30-tao-ec2-instance/21.png)

-----------------
## Gán Elastic IP cho EC2 Instance

> Khi bạn khởi động lại EC2 instance, Publish IP mặc định có thể thay đổi.  
> Để giữ một địa chỉ IP cố định, mình sẽ cấp và gán một **Elastic IP (EIP)** – đây là Publish IP tĩnh do AWS cung cấp miễn phí (nếu đang gán cho một EC2 đang chạy).
{: .prompt-warning }

> Elastic IP sẽ không bị thay đổi khi bạn stop/start EC2, giúp giữ ổn định việc truy cập.
{: .prompt-tip }

### 1. Mở trang Elastic IPs
Truy cập AWS Console → Dịch vụ **EC2**. Trong thanh sidebar, chọn **Elastic IPs** (dưới phần Network & Security)

![Image1](assets/images/2025-04-30-tao-ec2-instance/26.png)

### 2. Allocate Elastic IP
> Luôn đảm bảo bạn đang ở đúng Region trước khi cấp Elastic IP, để có thể gán được IP này cho EC2 mong muốn. 
{: .prompt-tip }

Click nút **Allocate Elastic IP address**

![Image1](assets/images/2025-04-30-tao-ec2-instance/27.png)

Ở bước cấu hình, để mặc định vùng (Region) và lựa chọn nguồn là **Amazon's pool of IPv4 addresses**. Click **Allocate**

![Image1](assets/images/2025-04-30-tao-ec2-instance/28.png)

Elastic IP được cấp phát thành công

![Image1](assets/images/2025-04-30-tao-ec2-instance/29.png)

### 3. Gán Elastic IP cho EC2
Sau khi được cấp IP, click chọn địa chỉ IP vừa tạo. Chọn **Actions → Associate Elastic IP address**

![Image1](assets/images/2025-04-30-tao-ec2-instance/30.png)

Trong cửa sổ Associate Elastic IP address, cấu hình như sau:

- Ở mục **Resource type**, chọn **Instance**
- Chọn đúng EC2 instance bạn muốn gán EIP
- Device name (nếu hiện): để mặc định

Click **Associate**

![Image1](assets/images/2025-04-30-tao-ec2-instance/31.png)

Thành công, Elastic IP đã được gáng vào EC2 intansce

![Image1](assets/images/2025-04-30-tao-ec2-instance/32.png)

### 4. Kiểm tra

Quay lại danh sách EC2 Instances → Instance của bạn. Cột **Public IPv4 address** giờ sẽ hiển thị Elastic IP. Dùng địa chỉ IP này để SSH hoặc truy cập website

![Image1](assets/images/2025-04-30-tao-ec2-instance/33.png)


> Elastic IP chỉ **miễn phí khi đang gán cho EC2 đang chạy**.  
> Nếu bạn **ngắt kết nối** hoặc EC2 **bị stop**, IP vẫn còn đó nhưng sẽ **bị tính phí nhỏ mỗi giờ** (~0.005 USD/h).  
> → Nếu không dùng nữa, nên **Release Elastic IP** để tránh bị tính phí ngầm.
{: .prompt-danger }

-----------------

## SSH vào EC2 Instance

### 1. SSH sử dụng Mobaxterm

Mở MobaXterm → Chọn Session → SSH

```yaml
Remote host:
  - Nhập Public IPv4 của EC2 (tìm trong EC2 Console)
Specify username:
  - Amazon Linux: ec2-user
  - Ubuntu: ubuntu
  - CentOS: centos
Advanced SSH settings:
  - Tick "Use private key"
  - Browse chọn file .pem đã tải về
```

![Image1](assets/images/2025-04-30-tao-ec2-instance/34.png)

Click **OK** → MobaXterm sẽ tự động thiết lập kết nối. Lần đầu sẽ hỏi xác nhận fingerprint, gõ Yes

SSH đến EC2 instance thành công

![Image1](assets/images/2025-04-30-tao-ec2-instance/35.png)

### 2. SSH sử dụng Git Bash

Mở Git Bash → cd đến vị trí thư mục chứa file .pem → dùng lệnh chmod để cấp quyền thực thi file .pem

```bash
chmod +x 01-lab-ec2-keypair.pem
```

Sử dụng lệnh sau để connect SSH

```bash
ssh -i <your-key-pair.pem> ec2-user@<your-instance-public-dns>
```

Bạn có thể lấy `<your-instance-public-dns>` trong phần Summary của EC2 Instance vừa tạo

![Image1](assets/images/2025-04-30-tao-ec2-instance/36.png)

Kết nối thành công

![Image1](assets/images/2025-04-30-tao-ec2-instance/37.png)

Sau khi kết nối SSH đến EC2 thành công, mình sẽ tiến hành triển khai web server trong bài viết sau.