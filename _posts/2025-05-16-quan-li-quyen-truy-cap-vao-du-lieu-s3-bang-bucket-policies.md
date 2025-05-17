---
title: Quản lý quyền truy cập vào dữ liệu S3 bằng Bucket Policy
date: 2025-05-16
categories: [Storage, Simple Storage Service]
tags: [s3, bucket policy]
mermaid: true
---

## 🎯 Mục tiêu
Hướng dẫn cách sử dụng **Bucket Policy** để cấp quyền public file trên S3, thay vì dùng ACL, giúp quản lý tập trung và bảo mật hơn.

-------------------

## 📚 Tài nguyên hiện có

- Bucket: **lab03-s3-ptmkhanh29**, mình đã tạo bucket này ở bài lab trước đó <a href="https://ptmkhanh29.github.io/tutorial-aws-labs/posts/tao-s3-bucket-va-upload-file" target="_blank" rel="noopener noreferrer">Tạo S3 Bucket và Public File sử dụng Access Control List</a>

- Lúc tạo Bucket mình cũng đã `enable ACL`, nhưng giờ mình muốn quản lý quyền truy cập đúng chuẩn với **Bucket Policy**

> **Bucket Policy vẫn dùng song song với ACL.** Nhưng nếu Bucket Policy là Deny, thì ACL có Allow cũng vô hiệu
> 
> - **Bucket Policy** = Quản lý tập trung từ trên xuống.
>
> - **ACL** = Quản lý chi tiết từng file nhỏ.
>
> - **Best Practice:** luôn ưu tiên Bucket Policy để kiểm soát quyền truy cập S3.
{: .prompt-warning }

## 🛠️ Các bước thực hiện

### 1. Tạo folder mới

Vào S3 truy cập Bucket (của mình là `lab03-s3-ptmkhanh29`), nhấn `Create Folder`

![Image1](assets/images/2025-05-16-quan-li-quyen-truy-cap-bang-bucket-policies/1.png)

Cấu hình cho Folder như sau

- Folder: tạo folder name là `public-files`.

- Server-side encryption: chọn `Don't specify an encryption key`

![Image1](assets/images/2025-05-16-quan-li-quyen-truy-cap-bang-bucket-policies/2.png)

Sau đó click Create folder. Một folder đã được tạo thành công

![Image1](assets/images/2025-05-16-quan-li-quyen-truy-cap-bang-bucket-policies/3.png)

> Về `folder` trong S3 thực chất là gì?
> 
> S3 không có folder thật như filesystem. Đây chỉ là cách mà AWS UI giả lập để User dễ nhìn khi cấu hình Bucket.
> Folder ở đây thực tế là prefix của object key.
> 
> **Ví dụ:** bạn upload file hello.txt vào folder public-files, thì key thật của object đó là: `public-files/hello.txt`
{: .prompt-tip }

### 2. Upload file demo

Mình tiến hành upload một file demo `aws-s3.md` vào folder `public-files` để test thử.

Click vào `public-files`, click `Upload`, click `Add files`, sau đó click Upload lần nữa để upload file vào thư mục `public-files`

![Image1](assets/images/2025-05-16-quan-li-quyen-truy-cap-bang-bucket-policies/4.png)

File đã được upload thành công

![Image1](assets/images/2025-05-16-quan-li-quyen-truy-cap-bang-bucket-policies/5.png)

Vào `Object overview` của file tiến hành test thử. Do chưa cấp bất kì Bucket Policy get object hay Make public gì nên chắc chắn sẽ không truy cập được file.

![Image1](assets/images/2025-05-16-quan-li-quyen-truy-cap-bang-bucket-policies/10.png)

File không truy cập được

![Image1](assets/images/2025-05-16-quan-li-quyen-truy-cap-bang-bucket-policies/11.png)

### 3. Tạo Bucket Policy

Click vào tab `Permissions`

![Image1](assets/images/2025-05-16-quan-li-quyen-truy-cap-bang-bucket-policies/6.png)

Kéo xuống phần `Bucket Policies`

![Image1](assets/images/2025-05-16-quan-li-quyen-truy-cap-bang-bucket-policies/7.png)

Ở đây mình có một Statement Template json Bucket Policies khá dễ hiểu để mọi người hình dung

```json
  {
    "Sid": "TênDễHiểu",
    "Effect": "Allow hoặc Deny",
    "Principal": "Ai được phép? '*' là mọi người",
    "Action": "Hành động gì? VD: s3:GetObject",
    "Resource": "Cái gì? VD: arn:aws:s3:::tên-bucket/*",
    "Condition": { "Điều kiện gì? (nếu có)" }
  }
```

| Field      | Ý nghĩa                           | Cách dùng đơn giản |
|------------|------------------------------------|-------------------|
| **Sid**    | Tên định danh        | Đặt gì cũng được, nhưng nên rõ nghĩa.<br>VD: `PublicReadForImages` để sau này dễ hiểu rule đó dùng làm gì. |
| **Effect** | Allow hoặc Deny        | Chỉ có 2 giá trị: <br>`"Allow"` (cho phép) <br>`"Deny"` (từ chối). |
| **Principal** | Ai được phép?       | `"*"` là tất cả mọi người (public).<br>Nếu muốn chỉ định cụ thể thì dùng ARN của user hoặc role. |
| **Action**  | Hành động gì? | Ví dụ: <br>`s3:GetObject` (tải file) <br>`s3:PutObject` (upload file).<br>Có thể là 1 hoặc nhiều action (list). |
| **Resource** | Cái gì bị áp dụng <br>(bucket/folder/file) | Đây là **đối tượng bị tác động** (bucket, folder, file). Dạng chuẩn là ARN.<br>Ví dụ: <br>`arn:aws:s3:::my-bucket` (cả bucket) <br>`arn:aws:s3:::my-bucket/*` (tất cả file trong bucket) <br>`arn:aws:s3:::my-bucket/folder/*` (chỉ folder đó) <br>`arn:aws:s3:::my-bucket/folder/file.txt` (chỉ file cụ thể). |
| **Condition** | Điều kiện gì? (nếu có)     | Dùng để giới hạn rule thêm 1 lớp, ví dụ chỉ cho phép IP nào đó,<br>chỉ khi dùng MFA, v.v. Có thể bỏ trống nếu không cần. |


Dựa trên template trên, mình đã tạo đoạn json Document cho Bucket Policies cho phép tất cả mọi người truy cập vào tất cả file trong thư mục `public-files` của bucket `lab03-s3-ptmkhanh29`

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadAccessForPublicFiles",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::lab03-s3-ptmkhanh29/public-files/*"
    }
  ]
}
```
Kéo xuống `Bucket policy`, click `Edit` để sửa lại Policy

![Image1](assets/images/2025-05-16-quan-li-quyen-truy-cap-bang-bucket-policies/8.png)

Sau đó click `Save change` để lưu lại

![Image1](assets/images/2025-05-16-quan-li-quyen-truy-cap-bang-bucket-policies/9.png)

Bucket Policies được update thành công

![Image1](assets/images/2025-05-16-quan-li-quyen-truy-cap-bang-bucket-policies/12.png)

Quay lại phần `Object overview` của file tiến hành kiểm tra

![Image1](assets/images/2025-05-16-quan-li-quyen-truy-cap-bang-bucket-policies/10.png)

Mình click vào `Object URL` thì có thể download được file `aws-s3.md` về máy

![Image1](assets/images/2025-05-16-quan-li-quyen-truy-cap-bang-bucket-policies/13.png)

Vậy là chúng ta đã thành công cấp quyền truy cập cho Bucket thông qua Bucket Policies

## ✅ Kết luận

Vậy là mình đã hoàn thành xong bài lab để hiểu cách cấu hình quản lý quyền truy cập cho dữ liệu của S3 bằng Bucket Policy. Thay vì phải cấu hình tay cho từng file với ACL như ở bài lab trước, giờ chúng ta chỉ cần 1 đoạn policy đã cấp được quyền cho cả thư mục.

👉 Bài lab tiếp theo trong series S3 mình sẽ cố gắng phát triển các phần nâng cao hơn như:

- Static Website Hosting trên S3

- Kết hợp CloudFront + SSL

- Lifecycle Rules để tiết kiệm chi phí