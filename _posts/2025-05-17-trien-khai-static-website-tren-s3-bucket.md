---
title: "Triển khai Static Website lên S3 Bucket"
date: 2025-05-17
categories: [Storage, Simple Storage Service]
tags: [s3, static website, hosting, cli]
toc: true
comments: true
mermaid: true
---

## 🎯 Mục tiêu

Hướng dẫn triển khai một **website tĩnh (HTML, CSS)** đơn giản lên Amazon S3, sử dụng tính năng **Static Website Hosting**.

Sau bài lab này bạn sẽ:
- Tạo được bucket cho web.
- Cấu hình S3 làm web host.
- Upload file web lên S3.
- Cấp quyền public để truy cập website qua HTTP.

> Lưu ý: Website này **chỉ có HTTP, không có HTTPS**. Bài viết tiếp theo mình sẽ xử lý HTTPS bằng CloudFront.
{: .prompt-tip }

---

## 📁 Chuẩn bị nội dung website

Mình sẽ clone repo static web có sẵn này về máy 
<a href="https://github.com/codewithsadee/gamics" 
    target="_blank"
    rel="noopener noreferrer">
    https://github.com/codewithsadee/gamics
</a>

```bash
mkdir -p ~/aws/resources/05-lab-s3
cd ~/aws/resources/05-lab-s3
git clone https://github.com/codewithsadee/gamics
cd gamics
```

![Image](assets/images/2025-05-17-trien-khai-static-website-tren-s3-bucket/1.png)


Bởi vì repo này chưa xử lí lỗi 404 nên mình sẽ tạo 1 file `error.html` trong thư mục `gamics` với nội dung như sau

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>404 - Page Not Found</title>
    <style>
        body {
        font-family: sans-serif;
        text-align: center;
        margin-top: 50px;
        }
    </style>
</head>
<body>
    <h1>404 - Page Not Found</h1>
    <p>Trang bạn tìm không tồn tại. Hãy quay lại trang chủ.</p>
    <a href="/">← Về trang chủ</a>
</body>
</html>
```

## 🛠️ Các bước triển khai

> Khi bạn dùng S3 để làm Static Website Hosting, AWS sẽ tạo cho bạn 1 website endpoint dạng
>   `http://<bucket-name>.s3-website-<region>.amazonaws.com`
> 
> Bởi vì ở bài lab tiếp theo chúng ta sẽ tiến hành gắn domain thật cho nên cần đặt tên bucket giống với tên của domain.
> Mình đã có một domain là `khanhphan.blog`, do đó mình sẽ tạo bucket có tên là `khanhphan.blog`
{: .prompt-tip }

### 1. Tạo bucket giống với domain name

```bash
aws s3api create-bucket \
  --bucket khanhphan.blog \
  --region ap-southeast-1 \
  --create-bucket-configuration LocationConstraint=ap-southeast-1
```
![Image](assets/images/2025-05-17-trien-khai-static-website-tren-s3-bucket/2.png)

### 2. Bật chế độ Static Website Hosting

```bash
aws s3 website s3://khanhphan.blog/ \
  --index-document index.html \
  --error-document error.html
```

### 3. Tắt Block Public Access để cho phép truy cập từ internet

```bash
aws s3api put-public-access-block \
  --bucket khanhphan.blog \
  --public-access-block-configuration \
  "BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false"
```

### 4. Gán Bucket Policy cho phép public-read toàn bộ file

Tạo file `bucket-policy.json` với nội dung như sau

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadForWebsite",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::khanhphan.blog/*"
    }
  ]
}
```
Apply bucket policy

```bash
aws s3api put-bucket-policy \
  --bucket khanhphan.blog \
  --policy file://bucket-policy.json
```

### 5. Upload file website

```bash
aws s3 cp ~/aws/resources/05-lab-s3/gamics s3://khanhphan.blog/ --recursive
```

Trong đó `~/aws/resources/05-lab-s3/gamics` là thư mục chứa static website gồm index.html, error.html,...

Truy cập bucket trên web UI để kiểm tra lại

![Image](assets/images/2025-05-17-trien-khai-static-website-tren-s3-bucket/3.png)

### 6. Truy cập kết quả

Mình truy cập được static website qua endpoint:

`http://khanhphan.blog.s3-website-ap-southeast-1.amazonaws.com`

Nếu đúng web sẽ hiển thị phần nội dung của file `index.html` như ảnh

![Image](assets/images/2025-05-17-trien-khai-static-website-tren-s3-bucket/4.png)

Mình sẽ thử truy cập vào 1 file có tên bất kì để test hoạt động của 404 page, AWS S3 sẽ tự động trả về nội dung của `error.html`

![Image](assets/images/2025-05-17-trien-khai-static-website-tren-s3-bucket/5.png)

## ✅ Kết luận

Vậy là chúng ta đã hoàn thành bài lab hosting static website bằng cách sử dụng S3 Bucket.

👉 Bài lab tiếp theo trong series S3 mình sẽ cố gắng phát triển phần nâng cao hơn nối tiếp bài lab này bằng cách:

- Kết hợp Route 53 + CloudFront + SSL

- Lifecycle Rules để tiết kiệm chi phí