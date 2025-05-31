---
title: "Tạo S3 Bucket & Gán Bucket Policy với AWS CLI"
date: 2025-05-17
categories: [Storage, Simple Storage Service]
tags: [aws cli, s3, bucket policy]
comments: true
---

Tiếp nối 2 bài lab tạo S3 Bucket và cấp quyền quản lý dùng ACL với Bucket Policy bằng AWS Console. Ở bài viết này mình sẽ dùng AWS CLI để tối ưu hóa công việc.

## 🎯 Mục tiêu

Hướng dẫn cách **tạo S3 bucket** và **gán bucket policy public read** bằng **AWS CLI**, sử dụng **Access Key & Secret Key** từ máy cá nhân.

Bên cạnh đó, bài lab giúp **hiểu rõ cơ chế Block Public Access của S3**, vì đây là thứ sẽ "Deny" các policy public nếu không cấu hình đúng.

---

## 🛠️ Các bước thực hiện

### 1. Cài đặt AWS CLI

>AWS CLI (Command Line Interface) là công cụ giúp mình làm việc với AWS bằng dòng lệnh, thay vì phải click trên giao diện AWS Console. 
> - AWS CLI version 1 (v1.x): cơ bản, đầy đủ các lệnh phổ biến, nhưng không hỗ trợ nhiều tính năng mới, giao diện cũ, nhiều hạn chế
> - AWS CLI version 2 (v2.x): hỗ trợ đầy đủ dịch vụ AWS mới nhất. Một số tính năng chỉ có trên version2 và không có trên version1
{: .prompt-info }

**Cài đặt AWS CLI trên Windows (dùng PowerShell)**

Nó sẽ tự động tải và run file .msi của AWS CLI ngay khi tải xong

```bash
msiexec.exe /i https://awscli.amazonaws.com/AWSCLIV2.msi
```

Click Next → Next → Next → Install → Finish

Kiểm tra version sau khi tải xong

```bash
aws --version
# Nếu trả về giống như vầy là đã tải và install aws cli thành công
# aws-cli/2.x.x Python/3.x.x Windows/x86_64
```

**Cài đặt AWS CLI trên Linux (Ubuntu)**

```bash
cd ~
mkdir -p ./aws
cd ./aws
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```
![Image](/assets/images/2025-05-17-tao-s3-bucket-va-gang-bucket-policies-bang-aws-cli/1.png)

Cài đặt thành công

![Image](/assets/images/2025-05-17-tao-s3-bucket-va-gang-bucket-policies-bang-aws-cli/2.png)

Kiểm tra version

![Image](/assets/images/2025-05-17-tao-s3-bucket-va-gang-bucket-policies-bang-aws-cli/3.png)

### 2. Lấy Access Key & Secret Key

Truy cập service IAM, sau đó chọn `Manage access keys`

![Image](/assets/images/2025-05-17-tao-s3-bucket-va-gang-bucket-policies-bang-aws-cli/4.png)

Tiến hành tạo Access Key mới: ở mục `Access keys` nhấn vào `Create access key`

![Image](/assets/images/2025-05-17-tao-s3-bucket-va-gang-bucket-policies-bang-aws-cli/5.png)

Do mình sử dụng `root user` để tạo Access key nên sẽ bị AWS cảnh báo như ảnh bên dưới. AWS đề nghị sử dụng IAM user để thay tác với Access key, tuy nhiên đây chỉ là bài lab demo nhỏ, không phải môi trường Production nên mình có thể tạm thời dùng root user để thực hành. 

> Trong bài lab này mình dùng root access key cho nhanh gọn khi học. Nhưng bạn cần nhớ rõ:
> 
> - Chỉ dùng root user khi thật sự cần thiết (thay đổi billing, MFA, account settings…)
> 
> - Khi làm việc thật tế, hãy tạo IAM user/role đúng chuẩn.
>
> - Luôn bật MFA và quản lý access key cẩn thận.
{: .prompt-danger }

Click vào `I understand creating a root access key is not a best practice, but I still want to create one`, sau đó click `Create access key`.

![Image](/assets/images/2025-05-17-tao-s3-bucket-va-gang-bucket-policies-bang-aws-cli/6.png)

Access key được tạo thành công, bạn download file .csv về để lưu trữ `Access key ID` và `Secret access key`, là 2 thứ cần thiết để config AWS CLI trên máy tính cá nhân.

![Image](/assets/images/2025-05-17-tao-s3-bucket-va-gang-bucket-policies-bang-aws-cli/7.png)

### 3. Cấu hình AWS CLI

Sử dụng lệnh sau để thiết lập cấu hình, nó sẽ yêu cầu bạn nhập các thông tin quan trọng

```bash
aws configure
```
Nhập thông tin 

```bash
AWS Access Key ID [None]: <Nhập Access Key ID vào đây>
AWS Secret Access Key [None]: <Nhập  Secret Access Key vào đây>
Default region name [None]: ap-southeast-1
Default output format [None]: json
```

![Image](/assets/images/2025-05-17-tao-s3-bucket-va-gang-bucket-policies-bang-aws-cli/9.png)

Bước config trên là cấu hình cho profile mặc định `[default]`. Nó sẽ lưu thông tin của bạn tại 2 file `~/.aws/credentials` và `~/.aws/config`. Nếu sau này bạn muốn tạo nhiều profile khác nhau (cho dev, prod,...), bạn sẽ cần thêm nhiều block [profile-name] vào đây.

File `~/.aws/credentials` lưu Access Key & Secret Key

![Image](/assets/images/2025-05-17-tao-s3-bucket-va-gang-bucket-policies-bang-aws-cli/10.png)

File `~/.aws/config` lưu region, output format, profile name

![Image](/assets/images/2025-05-17-tao-s3-bucket-va-gang-bucket-policies-bang-aws-cli/11.png)

Do mình sửa dụng profile `default` nên khi dùng AWS CLI mình sẽ không thêm flag `--profile <tên-profile>` phía sau

```bash
# Dùng profile default
aws s3 ls
# Dùng profile dev
aws s3 ls --profile dev
```

### 4. Tạo S3 Bucket và tắt Block Public Access

#### 4.1 Tạo S3 Bucket

Lệnh `create-bucket` chỉ để tạo "vỏ" bucket với tên và region. Các tính năng khác như: Versioning, Encryption, Public Access Block, Logging, Lifecycle Rules sẽ bổ sung sau bằng các lệnh update tương ứng.

```bash
aws s3api create-bucket \
  --bucket lab-04-s3-ptmkhanh29 \
  --region ap-southeast-1 \
  --create-bucket-configuration LocationConstraint=ap-southeast-1
```

📌 Giải thích:

+ `--bucket`: tên bucket (phải unique toàn cầu)

+ `--region`: vùng muốn tạo bucket (ví dụ: `ap-southeast-1`)

+ `--create-bucket-configuration`: bắt buộc khi không dùng region `us-east-1`

Kết quả khi tạo thành công

![Image](/assets/images/2025-05-17-tao-s3-bucket-va-gang-bucket-policies-bang-aws-cli/12.png)

Bạn cũng có thể truy cập S3 trên web để kiểm tra lại

![Image](/assets/images/2025-05-17-tao-s3-bucket-va-gang-bucket-policies-bang-aws-cli/13.png)

#### 4.2 Tắt Block Public Access cho bucket

> Cần tắt BlockPublicPolicy để apply Bucket policy
{: .prompt-danger }

Bạn có thể tham khảo document này để hiểu thêm về các option của `put-public-access-block` <a href="https://docs.aws.amazon.com/cli/latest/reference/s3api/put-public-access-block.html" target="_blank" rel="noopener noreferrer">Document</a> và lí do mình gắng True, False cho nó.

```bash
aws s3api put-public-access-block \
  --bucket lab-04-s3-ptmkhanh29 \
  --public-access-block-configuration "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=false,RestrictPublicBuckets=false"
```

📌 Giải thích:

- **BlockPublicAcls=true**  
  + Chặn việc cấp quyền public cho object thông qua ACL (Access Control List).  
  + Mục đích: tránh ai đó public nhầm từng file bằng ACL.

- **IgnorePublicAcls=true**  
  + Bỏ qua toàn bộ ACL public đã gán trước đó.  
  + Dù file có ACL public, vẫn bị coi như không có tác dụng.

- **BlockPublicPolicy=false**  
  + Cho phép bucket nhận **Bucket Policy dạng public** (Principal="*").  
  + Vì mình chủ động muốn public folder `public-files/` qua Bucket Policy.

- **RestrictPublicBuckets=false**  
  + Cho phép public dữ liệu ra ngoài Internet, không giới hạn trong trusted account.

### 5. Gán Bucket Policy bằng AWS CLI

```bash
# Tạo thư mục resources/04-lab-s3 trong ~/aws
mkdir -p ~/aws/resources/04-lab-s3

# Tạo file bucket-policy.json với nội dung policy
cat > ~/aws/resources/04-lab-s3/bucket-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadAccessForPublicFiles",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::lab-04-s3-ptmkhanh29/public-files/*"
    }
  ]
}
EOF
```

Kiểm tra lại nội dung file

![Image](/assets/images/2025-05-17-tao-s3-bucket-va-gang-bucket-policies-bang-aws-cli/14.png)

Sau đó, apply bucket policy từ file vừa tạo

```bash
aws s3api put-bucket-policy \
  --bucket lab-04-s3-ptmkhanh29 \
  --policy file://~/aws/resources/04-lab-s3/bucket-policy.json
```

Kiểm tra lại policy đã gáng

```bash
aws s3api get-bucket-policy \
  --bucket lab-04-s3-ptmkhanh29
```

![Image](/assets/images/2025-05-17-tao-s3-bucket-va-gang-bucket-policies-bang-aws-cli/15.png)

Bạn cũng có thể truy cập UI web để kiểm tra lại một lần nữa

![Image](/assets/images/2025-05-17-tao-s3-bucket-va-gang-bucket-policies-bang-aws-cli/16.png)

## 📤 Upload file demo để kiểm tra Bucket Policy

Giờ mình sẽ upload thử 1 file lên folder `public-files/` để kiểm tra quyền truy cập công khai.

Ví dụ, mình có file ảnh `image_test.png` nằm trong thư mục `~/aws/resources/04-lab-s3`

```bash
aws s3 cp ~/aws/resources/04-lab-s3/image_test.png s3://lab-04-s3-ptmkhanh29/public-files/
```

> Cú pháp `s3://<bucket-name>/<prefix>/` giúp bạn upload file vào đúng folder (prefix) public-files/
{: .prompt-tip }

![Image](/assets/images/2025-05-17-tao-s3-bucket-va-gang-bucket-policies-bang-aws-cli/17.png)

Kiểm tra truy cập file qua url trả về

`https://lab-04-s3-ptmkhanh29.s3.ap-southeast-1.amazonaws.com/public-files/image_test.png`

Bạn sẽ thấy file hiện ra nghĩa là Bucket Policy đang thực hiện đúng yêu cầu public toàn bộ file có prefix `public-files` ra internet

![Image](/assets/images/2025-05-17-tao-s3-bucket-va-gang-bucket-policies-bang-aws-cli/18.png)

## ✅ Kết luận

Vậy là chúng ta đã hoàn thành bài lab quản lý quyền truy cập tài nguyên S3 thông qua AWS CLI, cũng như hiểu rõ hơn về **Bucket Policy** và **Block Public Access**.

👉 Bài lab tiếp theo trong series S3 mình sẽ cố gắng phát triển các phần nâng cao hơn như:

- Static Website Hosting trên S3

- Kết hợp CloudFront + SSL

- Lifecycle Rules để tiết kiệm chi phí