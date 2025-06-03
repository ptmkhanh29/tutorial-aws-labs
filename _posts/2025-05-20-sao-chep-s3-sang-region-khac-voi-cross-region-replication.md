---
title: "Hướng dẫn sao chép S3 sang Region khác với Cross-Region Replication"
date: 2025-06-03
tags: [AWS, S3, CRR, Versioning, Backup, Dự phòng]
categories: [aws-labs]
comments: true
---

## 🎯 Mục tiêu

Thiết lập **S3 Cross-Region Replication (CRR)** để tự động sao chép dữ liệu từ một bucket tại Singapore (ap-southeast-1) sang một bucket khác tại US East (us-east-1), **sử dụng AWS Console**.

---

## 🧰 Yêu cầu

- Tài khoản AWS với quyền quản trị S3 và IAM 

- Hai region AWS khác nhau: ví dụ `ap-southeast-1` và `us-east-1`

> Phần cấu hình Cross-Region Replication (CRR) có thể chia thành **2 trường hợp chính**:  
> 1. Sao chép dữ liệu giữa 2 bucket thuộc cùng một tài khoản AWS
> 2. Sao chép dữ liệu giữa 2 tài khoản AWS khác nhau (cross-account replication)
> 
> Do mình chỉ có 1 tài khoản AWS thôi nên mình chỉ thực hành với trường hợp 1: **Sao chép dữ liệu giữa 2 bucket thuộc cùng một tài khoản AWS**
{: .prompt-danger }

## 🛠️ Các bước cấu hình

### 1. Tạo 2 bucket ở 2 region khác nhau

Vào AWS Console > truy cập S3 > chọn **Create bucket**

![Image](assets/images/2025-05-20-sao-chep-s3-sang-region-khac-voi-cross-region-replication/1.png)

>Bạn lưu ý nhớ chọn lại đúng region trước khi tạo, ở góc phải trên cùng, kế bên tên username
{: .prompt-danger }

#### 1.1 Tạo bucket nguồn (source bucket)

**General configuration**

- AWS Region: `Asia Pacific (Singapore) ap-southeast-1`
- Bucket type: General Purpose
- Bucket name: `lab-07-s3-src-crr`

![Image](assets/images/2025-05-20-sao-chep-s3-sang-region-khac-voi-cross-region-replication/2.png)

**Object Ownership**
- Chọn `ACLs disabled (recommended)` nếu cả hai bucket cùng thuộc một tài khoản AWS.  
- Chọn `ACLs enabled` nếu thực hiện sao chép giữa hai tài khoản khác nhau. ❗ Vì nếu không bật ACL, object sau khi replicate sẽ vẫn thuộc account nguồn, account đích không thể thao tác (xóa/sửa).
- 👉 Trong khuôn khổ bài viết mính sẽ chọn `ACLs disabled (recommended)`

**Block Public Access settings for this bucket**
- Giữ nguyên mặc định (✔ BẬT toàn bộ) nếu bạn chỉ dùng CRR để sao lưu hoặc đồng bộ dữ liệu giữa các bucket.
- Chỉ tắt nếu bạn cần **public file (ví dụ: website tĩnh, hoặc xem file để test public file)**, và đã cấu hình kỹ Bucket Policy hoặc IAM.
- 👉 Trong khuôn khổ bài viết mình sẽ **TẮT** `Block all public access`

![Image](assets/images/2025-05-20-sao-chep-s3-sang-region-khac-voi-cross-region-replication/3.png)

**Bucket Versioning:**
- ⚠️ **Bắt buộc bật `Enable`** để CRR hoạt động.

**Default encryption:**
- Chọn: `Server-side encryption with Amazon S3 managed keys (SSE-S3)`
- 👉 Đây là lựa chọn bảo mật mặc định, không cần cấu hình thêm, tương thích tốt với CRR.

![Image](assets/images/2025-05-20-sao-chep-s3-sang-region-khac-voi-cross-region-replication/4.png)

#### 1.2 Tạo Bucket đích (destination bucket)
- Bucket name: `lab-07-s3-dest-crr`
- Cấu hình gần giống bucket nguồn: bật Versioning, ACLs disabled, có thể chọn cùng loại encryption (SSE-S3).
- Bắt buộc chọn Region **KHÁC** với bucket nguồn để kích hoạt CRR, ở đây mình chọn Region là `us-east-1`
- Có thể bật/tắt Block Public Access tùy nhu cầu kiểm tra truy cập file replicate.

**Kết quả**

Hai bucket ở 2 region khác nhau đã được mình tạo xong

![Image](assets/images/2025-05-20-sao-chep-s3-sang-region-khac-voi-cross-region-replication/5.png)

### 2. Tạo Replication Rule cho bucket nguồn

> Thiết lập Cross-Region Replication (CRR) luôn được thực hiện trên bucket nguồn (source bucket)
{: .prompt-info }

Vào bucket **nguồn** `lab-07-s3-src-crr` > tab **Management**. Nhấn **[Create replication rule]** để tiến hành cấu hình Rule

![Image](assets/images/2025-05-20-sao-chep-s3-sang-region-khac-voi-cross-region-replication/6.png)

**Replication rule configuration**
- Replication rule name: `replicate-to-us-east`
- Status: Enabled

![Image](assets/images/2025-05-20-sao-chep-s3-sang-region-khac-voi-cross-region-replication/7.png)

**Source bucket**
- Choose a rule scope: Apply to all objects in the bucket

![Image](assets/images/2025-05-20-sao-chep-s3-sang-region-khac-voi-cross-region-replication/8.png)


**Destination**
- Destination: Chọn bucket đích của bạn, mình chọn `lab-07-s3-dest-crr`
- Destination Region: sẽ tự động hiển thị (`us-east-1`)

![Image](assets/images/2025-05-20-sao-chep-s3-sang-region-khac-voi-cross-region-replication/9.png)

**IAM Role**
- Chọn `Create new role`: AWS sẽ tự động tạo role và gán đủ quyền

![Image](assets/images/2025-05-20-sao-chep-s3-sang-region-khac-voi-cross-region-replication/10.png)

**Encryption và Destination storage class**: mặc định

**Additional replication options**
- Replication Time Control (RTC): Đảm bảo 99.99% object mới được replicate trong vòng 15 phút, có SLA và tính phí.
- Replication metrics: Theo dõi số lượng, dung lượng object đang chờ replicate và lỗi qua CloudWatch (có tính phí CloudWatch).
- Delete marker replication: Cho phép replicate các delete marker được tạo khi xóa object (trừ delete marker từ lifecycle rule).
- Replica modification sync: Cho phép đồng bộ các thay đổi metadata từ bucket đích ngược về bucket nguồn.

Do khuôn khổ bài viết và mình cũng chỉ muốn dùng Free Tier nên không bật option nào cả.

![Image](assets/images/2025-05-20-sao-chep-s3-sang-region-khac-voi-cross-region-replication/11.png)

Sau đó nhấn `Save` để lưu lại cấu hình. 

Sau khi tạo replication rule, AWS sẽ hiện một popup hỏi bạn có muốn replicate các object **đã tồn tại trước đó** không.

- `No, do not replicate existing objects.` (**Khuyến nghị**): Chỉ replicate object mới từ thời điểm này trở đi. Không phát sinh thêm chi phí.

- `Yes, replicate existing objects.`: Dùng AWS Batch Operations để replicate toàn bộ object cũ. **Có thể phát sinh phí.**

👉 Với khuôn khổ bài viết và mình chỉ sử dụng Free Tier, mình sẽ chọn **“No”**, sau đó nhấn Submit.

![Image](assets/images/2025-05-20-sao-chep-s3-sang-region-khac-voi-cross-region-replication/12.png)

Replication Rule cho bucket nguồn đã được cấu hình xong.

![Image](assets/images/2025-05-20-sao-chep-s3-sang-region-khac-voi-cross-region-replication/13.png)

### 3. Kiểm tra hoạt động của CRR

Quay lại bucket **nguồn** `lab-07-s3-src-crr` > Vào tab **Objects** > **Upload** một file bất kỳ

Mình upload một file ảnh vào bucket nguồn để test thử.

![Image](assets/images/2025-05-20-sao-chep-s3-sang-region-khac-voi-cross-region-replication/14.png)

Chờ khoảng 1–2 phút, vào bucket **đích** > tab **Objects** > Kiểm tra xem file đã được replicate sang chưa. File ảnh mà mình upload ở bucket nguồn đã xuất hiện của bucket đích tức là mình đã replicate thành công.

![Image](assets/images/2025-05-20-sao-chep-s3-sang-region-khac-voi-cross-region-replication/15.png)

## ✅ Kết luận

Qua bài lab này, chúng ta đã:

- Tạo 2 bucket ở 2 region khác nhau
- Bật versioning để kích hoạt khả năng replication
- Tạo CRR rule để tự động sao chép object từ region Singapore sang US
- Kiểm tra kết quả thành công qua giao diện Console

🔐 Đây là nền tảng quan trọng trong các hệ thống **backup phân vùng địa lý** và **Disaster Recovery (DR)** trong môi trường cloud AWS.

>**Lưu ý:** Mọi người nên xóa tài nguyên sau khi đã làm xong lab tránh bị tốn tiền oan nhé
{: .prompt-warning }