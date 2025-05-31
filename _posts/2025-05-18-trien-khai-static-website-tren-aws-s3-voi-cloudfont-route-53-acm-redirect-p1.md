---
title: "Triển khai Static Website với AWS S3, CloudFront, Route 53, SSL và Redirect - Phần 1"
date: 2025-05-18
categories: [Storage, Simple Storage Service]
tags: [s3, cloudfront, https, ssl, acm, route53, cloudfront function, cli, "terraform"]
toc: true
comments: true
mermaid: true
image: 
  path: /assets/images/2025-05-18-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p1/description-lab.svg
  alt: "Mô hình triển khai EC2 cơ bản trong Public Subnet"
---

## 💼 Case Study 

**Đề bài:** Đưa website cá nhân lên internet với domain riêng, HTTPS và tốc độ toàn cầu

Bạn vừa hoàn thành một website portfolio cá nhân gồm các file tĩnh (HTML, CSS, JS, ảnh dự án). Bây giờ, bạn muốn triển khai website ra internet thật sự với các yêu cầu rõ ràng như sau:

**Yêu cầu của bài toán**

- Website cần truy cập nhanh toàn cầu, kể cả từ châu Á, châu Âu hay Mỹ

- Phải có domain thật, ví dụ: https://khanhphan.blog

- Tự động redirect từ www.khanhphan.blog về domain chính

- Website được bảo vệ bằng SSL hợp lệ, không có cảnh báo bảo mật trình duyệt

- Nội dung phải được cache lại ở các edge location, tránh gọi lại về server gốc S3

- Tuyệt đối không để S3 bucket public, chỉ CloudFront được phép truy cập

- Giải pháp cần ổn định, bảo mật, chi phí thấp và đúng chuẩn production

## 🧪 Kiến trúc đề xuất

> Đây là kiến trúc được sử dụng rộng rãi để deploy blog cá nhân, landing page, trang giới thiệu, sản phẩm cá nhân... theo tiêu chuẩn cloud-native

![Image](assets/images/2025-05-18-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p1/description-lab.svg)


| Service                           | Vai trò                                                               |
| --------------------------------- | --------------------------------------------------------------------- |
| **Amazon S3**                     | Lưu trữ nội dung website tĩnh (HTML, CSS, JS, ảnh...)                 |
| **CloudFront**                    | CDN phân phối nội dung nhanh, bật HTTPS, cache nội dung               |
| **Origin Access Control (OAC)**   | Giúp CloudFront truy cập S3 mà không cần public bucket                |
| **AWS Certificate Manager (ACM)** | Cung cấp chứng chỉ SSL miễn phí cho domain                            |
| **Amazon Route 53**               | Quản lý DNS, trỏ domain `khanhphan.blog` về CloudFront                |
| **CloudFront Function**           | Xử lý redirect từ `www.khanhphan.blog` về `khanhphan.blog` (HTTP 301) |

Trong bài lab này mình sẽ chia làm các phần nhỏ hơn:

1. Tạo và xác thực chứng chỉ SSL với AWS Certificate Manager cho domain

2. Cấu hình CloudFront để phân phối website, tích hợp SSL và kết nối với S3

Trong loạt bài này mình sẽ sử dụng cả AWS web console, AWS CLI và Terraform. Thật ra mình chỉ tính làm CLI thôi nhưng nghĩ lại cần thực hành trước với giao diện để hiểu rõ hơn cấu hình thì mới apply tốt CLI được, bài có thể sẽ hơi dài, mong mọi người thông cảm.

## 🧱 Yêu cầu

- Đã mua domain trước đó (domain của mình là `khanhphan.blog`).
- Đã upload static website lên bucket S3 có tên trùng với tên domain. Mọi người có thể tham khảo bài viết trước nếu chưa upload static website [Triển khai Static Website lên S3 Bucket](https://ptmkhanh29.github.io/tutorial-aws-labs/posts/trien-khai-static-website-tren-s3-bucket/)
- Cách làm: hoặc Web console, hoặc AWS CLI hoặc Terraform. Vì điều kiện bài viết nên mình sẽ hướng dẫn bằng Web console trước để mọi người hiểu về config, sau đó là AWS CLI. Cuối bài sẽ có script Terraform để apply các step trên. Mọi người lưu ý chỉ làm 1 trong 3 cách thôi nhé.

---

## 🛠️ Các bước thực hiện

### 1. Chuyển Nameserver từ Domain provider sang Route 53

>Để xác minh quyền sở hữu domain và cấp chứng chỉ SSL với AWS Certificate Manager (ACM), bạn cần có quyền quản lý hệ thống DNS của domain. Điều này yêu cầu trỏ domain đang quản lý ở Domain provider - nơi bạn mua domain về Route 53 của AWS – nơi bạn sẽ tạo bản ghi CNAME xác minh ACM.
>
>Nếu không trỏ nameserver về Route 53, chúng ta sẽ không thể tạo bản ghi DNS cần thiết, và quá trình cấp SSL sẽ thất bại.
{: .prompt-info }

#### 1.1 Tạo Hosted Zone cho domain ở Route 53

Truy cập AWS Console > Route 53. Ở thanh sidbar bên trái chọn `Hosted Zones`

![Image](assets/images/2025-05-18-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p1/14.png)

Click `Create Hosted Zone` để tạo Hosted Zone cho domain

![Image](assets/images/2025-05-18-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p1/15.png)

Bạn cấu hình cho `Hosted Zone` của mình như sau:

+ Domain name: là tên domain của bạn, ví dụ ở đây là `khanhphan.blog`

+ Type: Public hosted zone

Sau đó click `Create Hosted Zone`

![Image](assets/images/2025-05-18-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p1/16.png)

Tới đây `Hosted Zone` của mình đã được tạo thành công. Bạn expand section `Hosted zone details` ra để lưu lại 2 phần quan trọng

- `Id`: ID của Hosted Zone (mình sẽ dùng để tạo bản ghi DNS), nó có dạng `/hostedzone/<Hosted zone ID>`

`/hostedzone/Z08218493EOPVIZVEXDYE`

- `NameServers`: 4 dòng NameServers cần dùng để trỏ về Hostinger

```
ns-183.awsdns-22.com
ns-1816.awsdns-35.co.uk
ns-1515.awsdns-61.org
ns-671.awsdns-19.net
```

>**Lưu ý:** do mình demo tạo với 2 cách là Web Console và CLI nên 2 giá trị của `Hosted Zone ID` và `Nameservers` của 2 cách tạo khác nhau. Bạn dùng 1 trong 2 cách là được.
{: .prompt-danger }

#### 1.2 Chỉnh sửa Nameservers trong Domain Provider

Domain Provider của mình là Hostinger. Mình đăng nhập rồi truy cập mục `Domains`, chọn domain name `khanhphan.blog`

![Image](assets/images/2025-05-18-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p1/8.png)

Chọn mục DNS / Nameservers từ sidebar bên trái

![Image](assets/images/2025-05-18-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p1/9.png)

Trong Nameservers hiện tại của mình có 2 Nameserver mặc định là 

```
ns1.dns-parking.com
ns2.dns-parking.com
```

Nhấn `Change Nameservers`, mình sửa 2 NS mặc định này thành 4 NS mà Route 53 đã cung cấp ban nãy (mình lấy trong cách làm với CLI), sau đó nhấn Save để lưu lại

![Image](assets/images/2025-05-18-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p1/10.png)

Sau khi apply thay đổi, chúng ta sẽ cần chờ 5–60 phút để DNS cập nhật toàn cầu. Sau đó test lại như sau

```bash
nslookup -type=ns <domain_name>
#Ex: nslookup -type=ns khanhphan.blog
```
![Image](assets/images/2025-05-18-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p1/18.png)


Vậy là mình đã hoàn tất việc thay đổi Nameserver thì NS mặc định của Domain Provider sang NS của Route 53. Ở phần tiếp theo chúng ta sẽ tiến hành xác minh domain qua DNS để cấp chứng chỉ SSL thông qua ACM để AWS xác thực domain này là của mình và cho phép domain sử dụng các service.

### 2. Xác minh domain qua DNS để cấp chứng chỉ SSL với ACM

#### 2.1. Gửi yêu cầu tạo chứng chỉ SSL bằng ACM và lấy bản ghi DNS

> **Mục đích:** Để bật HTTPS cho website khi truy cập qua CloudFront.
> 
> Chứng chỉ SSL giúp trình duyệt hiển thị biểu tượng 🔒 bảo mật, mã hóa kết nối giữa người dùng và website (CloudFront), tăng độ tin cậy và bảo vệ dữ liệu khỏi bị nghe lén.
{: .prompt-info }

> **Lưu ý:** phải tạo ở region `us-east-1` bởi vì CloudFront chỉ hỗ trợ sử dụng chứng chỉ SSL từ ACM ở region này.
{: .prompt-danger }

Truy cập AWS Console > Certificate Manager. Ở cột Sidebar bên trái click vào `List certificates`. Click `Request`

![Image](assets/images/2025-05-18-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p1/19.png)

- Certificate type: `Request a public certificate` → Là SSL công khai, chọn tùy chọn này để dùng cho CloudFront/S3. Sau đó click `Next` để tiếp tục

![Image](assets/images/2025-05-18-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p1/20.png)

- Domain names: Nhập tên miền chính mà bạn sở hữu, của mình là `khanhphan.blog`. Do mình muốn redirect từ `www.khanhphan.blog` về `khanhphan.blog`, nên mình sẽ thêm cả `www.khanhphan.blog` vào đây luôn

- Validation method: DNS validation

- Key Algorithm: RSA 2048

![Image](assets/images/2025-05-18-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p1/21.png)

Sau đó click `Request`.

Khi vừa tạo xong chứng chỉ SSL, bạn sẽ thấy trạng thái của 2 domain là `Pending validation`, tức là dù SSL đã được tạo nhưng nó vẫn chưa được Validate, vì ACM đang chờ bạn xác minh quyền sở hữu domain bằng cách tạo bản ghi CNAME trong DNS (Route 53), lúc này trạng thái sẽ tự động chuyển sang `Success` và sau đó là `Issued`.

Chúng ta sẽ cần lưu lại 3 giá trị của Name, Type, Value của 2 domain để tạo bản ghi CNAME tương ứng trong Route 53.

![Image](assets/images/2025-05-18-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p1/25.png)

#### 2.2. Tạo bản ghi CNAME trong Route 53 để xác thực chứng chỉ SSL

> Đây là bước cực kỳ quan trọng để xác thực yêu cầu chứng chỉ SSL đã gửi đến ACM ở bước trước đó.
> 
> Việc tạo bản ghi CNAME giúp chứng minh rằng bạn thật sự sở hữu domain, từ đó ACM mới **THỰC SỰ** cấp chứng chỉ SSL và cho phép CloudFront bật HTTPS.
{: .prompt-danger }

Vào AWS Console > Route 53 > Chọn Hosted Zone đã tạo ở bước 1 > Click `Create Record`

![Image](assets/images/2025-05-18-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p1/23.png)

Vì mình đang tạo chứng chỉ SSL cho 2 domain nên sẽ cần tạo 2 bản ghi CNAME trong Route 53 để xác minh mỗi domain một bản ghi.

✅ Cách điền Record

📌 Đối với domain `khanhphan.blog`

+ CNAME name (từ ACM): `_ebbc3b0b8cad57d6bf8e99514f0c29be.khanhphan.blog.`

+ CNAME value (từ ACM): `_9630ae438864949041ef512326f8945d.xlfgrmvvlj.acm-validations.aws.`

>Khi tạo record trong Route 53:
>+ Record name: _ebbc3b0b8cad57d6bf8e99514f0c29be (bỏ đuôi `.khanhphan.blog.` vì Route 53 sẽ tự thêm vào)
>+ Value: _9630ae438864949041ef512326f8945d.xlfgrmvvlj.acm-validations.aws.
>+ Record type: CNAME
{: .prompt-danger }

Nhấn `Create records`

![Image](assets/images/2025-05-18-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p1/27.png)

📌 Đối với domain `www.khanhphan.blog`

>Khi tạo record trong Route 53:
>+ Record name: _aed77fe6dc7721423af629eeec710728.www (bỏ đuôi `.khanhphan.blog.` vì Route 53 tự thêm domain gốc)
>+ Value: _49100fd55647c5add4af92bffc26daf5.xlfgrmvvlj.acm-validations.aws.
>+ Record type: CNAME
{: .prompt-danger }

Ok, hai bản ghi CNAME cho 2 domain tương ứng đã được tạo thành công

![Image](assets/images/2025-05-18-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p1/28.png)

Bạn chờ vài phút rồi load lại chứng chỉ SSL sẽ thấy Status của 2 domain chuyển từ `Pending validation` sang `Success` nghĩa là đã xác minh chứng chỉ SSL thành công.

Sau khi bạn đã tạo đúng các bản ghi CNAME trong Route 53, Status của chứng chỉ SSL ở ACM có thể vẫn hiển thị là `Pending validation`. Đừng lo, hệ thống cần một khoảng thời gian để xác minh DNS – thường từ vài phút đến khoảng 30 phút. Bạn chỉ cần chờ một lúc rồi tải lại trang. Khi trạng thái chuyển sang `Success` như ảnh, tức là chứng chỉ đã được xác minh thành công và sẵn sàng để sử dụng.

![Image](assets/images/2025-05-18-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p1/29.png)

**Sử dụng AWS CLI**

```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id <your-hosted-zone-id> \
  --change-batch '{
    "Comment": "ACM domain validation for SSL",
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "<thay_bang_name_trong_ResourceRecord>",
        "Type": "CNAME",
        "TTL": 300,
        "ResourceRecords": [{
          "Value": "<thay_bang_value_trong_ResourceRecord>"
        }]
      }
    }]
  }'
```

Kết quả trả về

![Image](assets/images/2025-05-18-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p1/12.png)

**⏳ Chờ xác minh chứng chỉ**

**Sử dụng AWS Console**

**Sử dụng AWS CLI**

`"Status": "PENDING"` nghĩa là Route 53 chưa hoàn tất việc áp dụng bản ghi DNS, không phải lỗi. Chúng ta sẽ đợi vài phút rồi chạy lệnh `describe-certificate` để theo dõi tiến trình xác minh SSL

Kiểm tra trạng thái của chứng chỉ SSL

```bash
aws acm describe-certificate \
  --certificate-arn <CertificateArn> \
  --region us-east-1 \
  --query "Certificate.Status"
```

Hoặc nếu bạn dùng AWS Console bạn quay lại trang chứng chỉ trong ACM, status của SSL sẽ chuyển từ `Pending validation` sang `Issued`, tức là đã xác minh thành công, chứng chỉ SSL đã được cấp.

![Image](assets/images/2025-05-18-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p1/13.png)

## ⚙️ Thực hành với AWS CLI

### 1. Tạo Hosted Zone cho domain ở Route 53:

> Tài liệu tham khảo: [AWS CLI create Hosted Zone](https://docs.aws.amazon.com/cli/latest/reference/route53/create-hosted-zone.html) 

```bash
# Tạo hosted zone với tên trùng với tên domain của bạn
aws route53 create-hosted-zone \
  --name khanhphan.blog \
  --caller-reference "$(date +%s)" \
  --hosted-zone-config Comment="Hosted zone for khanhphan.blog",PrivateZone=false
```

Kết quả mình nhận được 1 block json chứa 2 thông tin quan trọng mà mình cần sử dụng

- `Id`: ID của Hosted Zone (mình sẽ dùng để tạo bản ghi DNS)

`/hostedzone/Z01355962M1J001DOC1Q0`

- `NameServers`: 4 dòng NameServers cần dùng để trỏ về Hostinger

```json
"NameServers": [
    "ns-826.awsdns-39.net",
    "ns-1114.awsdns-11.org",
    "ns-261.awsdns-32.com",
    "ns-1646.awsdns-13.co.uk"
]
```

![Image](assets/images/2025-05-18-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p1/7.png)

### 2. Chỉnh sửa Nameservers trong Domain Provider

Tham khảo bước trước [Chỉnh sửa Nameservers trong Domain Provider](#12-chỉnh-sửa-nameservers-trong-domain-provider)

### 3. Gửi yêu cầu tạo chứng chỉ SSL bằng ACM và lấy bản ghi DNS

```bash
#! Lưu ý: option --subject-alternative-names là thêm tên miền phụ
aws acm request-certificate \
  --domain-name khanhphan.blog \
  --subject-alternative-names www.khanhphan.blog \
  --validation-method DNS \
  --region us-east-1
```

📌 Lưu lại `CertificateArn` để sử dụng cho bước 2

```yaml
{
  "CertificateArn": "arn:aws:acm:us-east-1:898508216915:certificate/69b4313f-df53-4a77-a2dc-5ce8a6c3718c"
}
```

Sau khi đó có `CertificateArn` chúng ta tiến hành lấy bản ghi DNS

```bash
#! <CertificateArn> điền CertificateArn đã lưu ở bước 1 vào đây
aws acm describe-certificate \
  --certificate-arn "arn:aws:acm:us-east-1:898508216915:certificate/69b4313f-df53-4a77-a2dc-5ce8a6c3718c" \
  --region us-east-1
```

Sau khi chạy lệnh `aws acm describe-certificate`, bạn sẽ thấy trong phần `DomainValidationOptions` xuất hiện thông tin của từng bản ghi CNAME mà AWS yêu cầu bạn tạo để xác minh quyền sở hữu domain. Mỗi domain hoặc subdomain (ví dụ: `khanhphan.blog` và `www.khanhphan.blog`) sẽ có một cặp Name và Value dùng để tạo bản ghi CNAME trong Route 53. Bạn cần sao chép chính xác các giá trị này (gồm cả dấu chấm ở cuối) để tạo bản ghi CNAME trong Route 53.

![Image](assets/images/2025-05-18-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p1/31.png)

**❓ Status `PENDING_VALIDATION` là sao?**

>Trạng thái "PENDING_VALIDATION" có nghĩa là AWS ACM vẫn đang chờ xác minh rằng bạn đã thực sự tạo bản ghi CNAME tương ứng trong DNS. Đây là bước kiểm tra xem bạn có thật sự sở hữu domain này hay hơn. Ok sang bước tiếp theo chúng ta sẽ tìm cách để làm sao cho AWS hiểu domain này thật sự là của chúng ta.

### 4. Tạo bản ghi CNAME trong Route 53 để xác thực chứng chỉ SSL

- Lấy `HostedZoneId` của domain

```bash
aws route53 list-hosted-zones-by-name \
  --dns-name khanhphan.blog \
  --query "HostedZones[0].Id" \
  --output text
```

Kết quả: 

```yaml
/hostedzone/Z01355962M1J001DOC1Q0
```

- Tạo file `create-cname-lab-06.json` để batch tạo 2 bản ghi DNS, bạn cần thay giá trị cho `Name` và `Value` từ output đã chạy của lệnh `describe-certificate` trước đó), ví dụ của mình:

```json
{
  "Comment": "Add CNAME records for SSL validation",
  "Changes": [
    {
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "_ebbc3b0b8cad57d6bf8e99514f0c29be.khanhphan.blog.",
        "Type": "CNAME",
        "TTL": 300,
        "ResourceRecords": [
          {
            "Value": "_9630ae438864949041ef512326f8945d.xlfgrmvvlj.acm-validations.aws."
          }
        ]
      }
    },
    {
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "_aed77fe6dc7721423af629eeec710728.www.khanhphan.blog.",
        "Type": "CNAME",
        "TTL": 300,
        "ResourceRecords": [
          {
            "Value": "_49100fd55647c5add4af92bffc26daf5.xlfgrmvvlj.acm-validations.aws."
          }
        ]
      }
    }
  ]
}
```

- Gửi lệnh tạo bản ghi:

```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id Z01355962M1J001DOC1Q0 \
  --change-batch file://create-cname-lab-06.json
```

![Image](assets/images/2025-05-18-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p1/33.png)

Khi bạn đã tạo đúng bản ghi CNAME và DNS được đồng bộ, trạng thái `PENDING VALIDATION` sẽ tự động chuyển thành `SUCCESS` – tức là chứng chỉ SSL đã được xác minh thành công và sẵn sàng sử dụng.

Bạn cần chờ một chút, sau đó chạy lại lệnh kiểm tra để xác nhận:

```bash
aws acm describe-certificate
  --certificate-arn arn:aws:acm:us-east-1:898508216915:certificate/69b4313f-df53-4a77-a2dc-5ce8a6c3718c \
  --region us-east-1 \
  --query "Certificate.DomainValidationOptions[*].{Domain:DomainName, Status:ValidationStatus}" \
  --output table
```

![Image](assets/images/2025-05-18-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p1/34.png)

## ✅ Kết luận

Vậy là trong phần 1 này, mình đã hoàn tất việc tạo chứng chỉ SSL và xác minh domain qua Route 53 – là bước nền quan trọng để triển khai website với HTTPS.

Ở phần tiếp theo, mình sẽ cấu hình CloudFront để phân phối nội dung (CloudFront Distribution), bật SSL và tối ưu hiệu suất bằng CDN.

Cảm ơn mọi người đã theo dõi, hẹn gặp lại mọi người ở phần 2 !!!