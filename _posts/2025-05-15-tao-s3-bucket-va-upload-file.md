---
title: Tạo S3 Bucket và Public File sử dụng Access Control List
date: 2025-05-15
categories: [Storage, Simple Storage Service]
tags: [S3, ACL]
---

## Mục tiêu
- Tạo một S3 Bucket.
- Upload file lên bucket.
- Thiết lập quyền public cho file thông qua Object **ACL (Access Control List)**, để file có thể truy cập công khai qua internet

## Các bước thực hiện

### Bước 1: Truy cập AWS S3 Console

- Vào trang quản lý S3: [https://s3.console.aws.amazon.com/s3](https://s3.console.aws.amazon.com/s3).

- Hoặc, trong AWS Console, tìm kiếm “S3”

![Image](assets/images/2025-05-15-tao-s3-bucket-va-upload-file/1.png)

### Bước 2: Tạo mới một Bucket

>**Lưu ý**: Khi tạo S3 bucket, nên chọn Region gần với EC2 hoặc user truy cập nhất để tối ưu tốc độ và chi phí. S3 là global service nên các resource ở Region khác vẫn có thể truy cập được bình thường, chỉ khác biệt về độ trễ.
{: .prompt-warning }

Chọn Region: mình chọn region Singapore

![Image](assets/images/2025-05-15-tao-s3-bucket-va-upload-file/2.png)

Click **Create bucket**

![Image](assets/images/2025-05-15-tao-s3-bucket-va-upload-file/3.png)

Điền thông tin cho Bucket:

**1. General configuration**

   - **Region**: chọn khu vực gần bạn nhất (ví dụ: `Asia Pacific (Singapore)`). Do mình đã chọn Region trước đó rồi nên phần này sẽ mặc định là `Asia Pacific (Singapore) ap-southeast-1`

   - **Bucket name**: đặt tên duy nhất trên toàn cầu (ví dụ: `my-first-s3-lab-yourusername`).

   - **Bucket type:** chọn General purpose. Đây là loại bucket tiêu chuẩn, phù hợp với tất cả các trường hợp sử dụng thông thường và được áp dụng Free Tier

![Image](assets/images/2025-05-15-tao-s3-bucket-va-upload-file/4.png)

**2. Object Ownership**

> Mặc định sẽ là ACLs disabled (recommended) → Nghĩa là quyền truy cập sẽ được quản lý bằng Policies (tốt hơn về security).
>
> Tuy nhiên, nếu bạn muốn dùng **Make public** cho từng file bằng ACL, thì cần chọn: `ACLs enabled` → Cho phép sử dụng ACL để cấp quyền public cho object.
> Ở phạm vi bài lab này mình sẽ chọn `ACLs enabled`
{: .prompt-warning }

Sau đó chọn `Bucket owner preferred (recommended)` để đảm bảo mọi object upload lên bucket đều thuộc sở hữu của bạn, kể cả khi upload từ user hoặc account khác.

![Image](assets/images/2025-05-15-tao-s3-bucket-va-upload-file/5.png)

**3. Block Public Access settings for this bucket**

   - **Bỏ chọn** `Block all public access` (cho phép public ra ngoài internet).
   - Tick vào ô xác nhận `I acknowledge that the current settings might result in this bucket and the objects within becoming public`.

![Image](assets/images/2025-05-15-tao-s3-bucket-va-upload-file/6.png)

**4. Bucket Versioning**

Không bật trong bài lab này vì không cần quản lý phiên bản file. Chỉ bật khi cần khôi phục object cũ hoặc giữ lịch sử file → Chọn `Disable`

**5. Tags:** Có thể để mặc định

**6. Default encryption**

- Encryption type: chọn SSE-S3 (Mã hóa bằng Amazon S3 managed keys (AWS tự quản lý khoá))

- Bucket Key: Enable (mặc định, không ảnh hưởng tới SSE-S3)

![Image](assets/images/2025-05-15-tao-s3-bucket-va-upload-file/7.png)

Click **Create bucket**

![Image](assets/images/2025-05-15-tao-s3-bucket-va-upload-file/8.png)

Vậy là một Bucket đã được tạo thành công.

### Bước 3: Upload File vào Bucket
Vào bucket vừa tạo. Click **Upload**

![Image](assets/images/2025-05-15-tao-s3-bucket-va-upload-file/9.png)

Chọn file từ máy tính (ảnh, file text, v.v.).

![Image](assets/images/2025-05-15-tao-s3-bucket-va-upload-file/10.png)

Click **Upload**.

![Image](assets/images/2025-05-15-tao-s3-bucket-va-upload-file/11.png)

Mình đã upload 2 file thành công

![Image](assets/images/2025-05-15-tao-s3-bucket-va-upload-file/12.png)

### Bước 4: Cấp quyền Public cho file

Sau khi upload thành công → Chọn file vừa upload. Click **Object actions** → **Make public using ACL**.

![Image](assets/images/2025-05-15-tao-s3-bucket-va-upload-file/13.png)

Xác nhận cấp quyền public bằng cách click **Make public**

![Image](assets/images/2025-05-15-tao-s3-bucket-va-upload-file/14.png)

### Bước 5: Lấy URL file để truy cập

Bạn truy cập vào phần Object overview của file. Click vào file → Kéo xuống dưới phần **Object URL**

![Image](assets/images/2025-05-15-tao-s3-bucket-va-upload-file/15.png)

Copy URL và mở trên trình duyệt. Nếu bạn thấy file hiện lên tức là đã thành công upload file và thiết lập quyền public để truy cập từ internet cho nó.

---

## Kết luận

Dùng ACL để public file rất nhanh và dễ thao tác — chỉ vài cú click là xong. Nhưng khi số lượng file nhiều, việc phải chỉnh từng file thủ công sẽ rất mất thời gian. Ngoài ra, ACL không phải cách quản lý permission tốt nhất — ACL đơn giản nhưng kém hiệu quả khi muốn kiểm soát bảo mật cho cả bucket hay nhiều người dùng khác nhau upload vào bucket đó.

### Tóm lại:
- ACL chỉ hợp khi bạn cần demo để biết cách tạo và sử dụng S3 Bucket.

- Với dự án thật → nên dùng Bucket Policies hoặc IAM Policies để quản lý quyền tốt hơn.