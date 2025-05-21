---
title: "Triển khai Static Website với AWS S3, CloudFront, Route 53, AWS Certificate Manager và Redirect - Phần 1"
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

## 🎯 Mục tiêu

Sau khi hoàn thành bài lab, bạn sẽ triển khai được một static website với các tính năng như sau:

- Truy cập website qua domain thật, ví dụ `https://khanhphan.blog` với SSL bảo mật.

- Giúp website tải nhanh ở mọi nơi thông qua CloudFront CDN.

- Route 53 phân giải domain chính xác về CloudFront.

- CloudFront Function tự động redirect từ `www.domain` về `domain` cho lỗi 301 Redirect.

- S3 bucket được bảo vệ hoàn toàn, chỉ cho phép CloudFront truy cập qua OAC.

---

## 🧩 Kiến trúc tổng quan
Sơ đồ mình vẽ dưới đây giúp minh họa các service mình sẽ sử dụng trong quá trình triển khai static website với domain thật `khanhphan.blog` và sử dụng giao thức HTTPS:

![Image](assets/images/2025-05-18-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p1/description-lab.svg)


| Service                           | Vai trò                                                               |
| --------------------------------- | --------------------------------------------------------------------- |
| **Amazon S3**                     | Lưu trữ nội dung website tĩnh (HTML, CSS, JS, ảnh...)                 |
| **CloudFront**                    | CDN phân phối nội dung nhanh, bật HTTPS, cache nội dung               |
| **Origin Access Control (OAC)**   | Giúp CloudFront truy cập S3 mà không cần public bucket                |
| **AWS Certificate Manager (ACM)** | Cung cấp chứng chỉ SSL miễn phí cho domain                            |
| **Amazon Route 53**               | Quản lý DNS, trỏ domain `khanhphan.blog` về CloudFront                |
| **CloudFront Function**           | Xử lý redirect từ `www.khanhphan.blog` về `khanhphan.blog` (HTTP 301) |

---

Trong bài lab này mình sẽ chia làm các phần nhỏ hơn:

1. Tạo và xác thực chứng chỉ SSL với AWS Certificate Manager

2. Cấu hình CloudFront phân phối website và tích hợp SSL

Trong loạt bài này mình sẽ sử dụng cả AWS web console, AWS CLI và Terraform. Thật ra mình chỉ tính làm CLI thôi nhưng nghĩ lại cần thực hành trước với giao diện để hiểu rõ hơn cấu hình thì mới apply tốt CLI được, bài có thể sẽ hơi dài, mong mọi người thông cảm.

## 🧱 Yêu cầu

- Đã mua domain trước đó (domain của mình là `khanhphan.blog`).
- Đã upload static website lên bucket S3 có tên trùng với tên domain. Mọi người có thể tham khảo bài viết trước nếu chưa upload static website [Triển khai Static Website lên S3 Bucket](https://ptmkhanh29.github.io/tutorial-aws-labs/posts/trien-khai-static-website-tren-s3-bucket/)
- Cách làm: hoặc Web console, hoặc AWS CLI hoặc Terraform

---

## 🛠️ Các bước thực hiện

### 1. Chuyển Nameserver từ Domain provider sang Route 53

>Để xác minh quyền sở hữu domain và cấp chứng chỉ SSL với AWS Certificate Manager (ACM), bạn cần có quyền quản lý hệ thống DNS của domain. Điều này yêu cầu trỏ domain đang quản lý ở Domain provider - nơi bạn mua domain về Route 53 của AWS – nơi bạn sẽ tạo bản ghi CNAME xác minh ACM.
>
>Nếu không trỏ nameserver về Route 53, chúng ta sẽ không thể tạo bản ghi DNS cần thiết, và quá trình cấp SSL sẽ thất bại.
{: .prompt-info }

#### 1.1 Tạo Hosted Zone cho domain ở Route 53

**Sử dụng AWS Console**

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

**Sử dụng AWS CLI**

Tương tự, mình sử dụng AWS CLI như sau để tạo Hosted Zone cho domain của mình. Bạn có thể tham khảo docs này để hiểu hơn về các option của [CLI](https://docs.aws.amazon.com/cli/latest/reference/route53/create-hosted-zone.html) mà mình đã sử dụng cho command bên dưới

```bash
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

**Sử dụng AWS Console**

Truy cập AWS Console > Certificate Manager. Ở cột Sidebar bên trái click vào `List certificates`. Click `Request`

![Image](assets/images/2025-05-18-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p1/19.png)

- Certificate type: `Request a public certificate` → Là SSL công khai, chọn tùy chọn này để dùng cho CloudFront/S3. Sau đó click `Next` để tiếp tục

![Image](assets/images/2025-05-18-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p1/20.png)

- Domain names: Nhập tên miền, của mình là `khanhphan.blog`

- Validation method: DNS validation

- Key Algorithm: RSA 2048

![Image](assets/images/2025-05-18-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p1/21.png)

Sau đó click `Request`. Sau đó bạn kéo xuống section `Domains` để lấy `CNAME name` và `CNAME value`

![Image](assets/images/2025-05-18-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p1/22.png)

**Sử dụng AWS CLI**

```bash
aws acm request-certificate \
  --domain-name khanhphan.blog \
  --validation-method DNS \
  --region us-east-1
```

📌 Lưu lại `CertificateArn` để sử dụng cho bước 2

```yaml
{
  "CertificateArn": "arn:aws:acm:us-east-1:898508216915:certificate/4c6e2285-4fc2-4c8f-b455-468100e48aff"
}
```

![Image](assets/images/2025-05-18-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p1/1.png)

Sau khi đó có `CertificateArn` chúng ta tiến hành lấy bản ghi DNS

```bash
#! <CertificateArn> điền CertificateArn đã lưu ở bước 1 vào đây
aws acm describe-certificate \
  --certificate-arn <CertificateArn> \
  --region us-east-1
```

Trong kết quả trả về, sẽ có block ResourceRecord chứ thông tin bản ghi CNAME như sau:

```json
"ResourceRecord": {
  "Name": "<_abcde>.khanhphan.blog.",
  "Type": "CNAME",
  "Value": "_xyz123.acm-validations.aws."
}
```

![Image](assets/images/2025-05-18-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p1/2.png)

Đây là bản ghi CNAME của mình

```json
"ResourceRecord": {
  "Name": "_ebbc3b0b8cad57d6bf8e99514f0c29be.khanhphan.blog.",
  "Type": "CNAME",
  "Value": "_9630ae438864949041ef512326f8945d.xlfgrmvvlj.acm-validations.aws."
}
```

Chúng ta sẽ cần lưu lại 3 giá trị của Name, Type, Value lại để tạo bản ghi CNAME tương ứng trong Route 53 để xác minh quyền sở hữu domain này ở bước sau.

Trước khi vào bước xác thực (validate chứng chỉ SSL) bạn có thể vào `AWS Certificate Manager` để check thử status của chứng chỉ SSL như sau.

Bạn có thể thấy status của nó đang là `Pending validation`, tức là dù trước đó bạn đã yêu cầu ACM tạo cho bạn một SSL, tuy nhiên SSL này vẫn chưa được xác thực. OK, bây giờ mình sẽ sang bước 3 để làm nốt phần xác thực SSL.

![Image](assets/images/2025-05-18-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p1/11.png)

#### 2.3. Tạo bản ghi CNAME trong Route 53 để xác thực chứng chỉ SSL

> Đây là bước cực kỳ quan trọng để xác thực yêu cầu chứng chỉ SSL đã gửi đến ACM ở bước trước đó.
> 
> Việc tạo bản ghi CNAME giúp chứng minh rằng bạn thật sự sở hữu domain, từ đó ACM mới **THỰC SỰ** cấp chứng chỉ SSL và cho phép CloudFront bật HTTPS.
{: .prompt-danger }

**Sử dụng AWS Console**

Vào AWS Console > Route 53 > Chọn Hosted Zone đã tạo ở bước 1 > Click `Create Record`

![Image](assets/images/2025-05-18-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p1/23.png)

Tiến hành điền thông tin bản ghi:

+ Record name: Mình dán CNAME name của mình vào, ví dụ `_ebbc3b0b8cad57d6bf8e99514f0c29be.khanhphan.blog.`

+ Record type: Chọn CNAME

+ Value: Dán dòng _yyyyy.acm-validations.aws.

+ TTL (Time to live): Có thể để mặc định (300 giây)

Nhấn `Create records`

Một bản ghi CNAME đã được tạo thành công

![Image](assets/images/2025-05-18-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p1/24.png)

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

Vậy là đã xong phần tạo và xác thực chứng chỉ SSL. Sang bài tiếp theo mình sẽ thực hành ...