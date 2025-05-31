---
title: "Triển khai Static Website với AWS S3, CloudFront, Route 53, SSL và Redirect - Phần 2"
date: 2025-05-26
categories: [Storage, Simple Storage Service]
tags: [s3, cloudfront, https, ssl, acm, route53, cloudfront function, cli, terraform]
toc: true
comments: true
mermaid: true
image:
  path: /assets/images/2025-05-26-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p2/description-lab.svg
  alt: "Kiến trúc phân phối website qua CloudFront và HTTPS"
---

Ở phần trước, chúng ta đã hoàn tất việc tạo chứng chỉ SSL với ACM và xác minh quyền sở hữu domain qua Route 53.
Trong phần này, mình sẽ hướng dẫn bạn cách tạo một CloudFront Distribution, kết nối với S3 để phân phối nội dung tĩnh, tích hợp chứng chỉ SSL, đảm bảo website có thể truy cập nhanh, an toàn qua domain thật như `https://khanhphan.blog`.

## 🎯 Mục tiêu

- Tạo CloudFront Distribution để phân phối nội dung từ S3 website

- Gắn chứng chỉ SSL đã xác minh từ trước [Triển khai Static Website với AWS S3, CloudFront, Route 53, SSL và Redirect - Phần 1](https://ptmkhanh29.github.io/tutorial-aws-labs/posts/trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p1/)

- Cấu hình bảo mật cho S3 bằng **Origin Access Control (OAC)**

- Trỏ domain về CloudFront qua Route 53

- Giải thích chi tiết từng cấu hình để bạn hiểu rõ cách CloudFront hoạt động

## 🛠️ Các bước thực hiện

### 1. Tạo CloudFront Distribution

**Sử dụng AWS Console**

Vào AWS Console, truy cập `CloudFront`. Chọn **Create a CloudFront distribution**

![Image](assets/images/2025-05-26-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p2/1.png)

---

#### Cấu hình Origin

**1. Origin domain**

Với static website dùng S3, bạn nên chọn bucket kiểu REST endpoint, ví dụ mình chọn bucket của mình:
`khanhphan.blog.s3.amazonaws.com`

**KHÔNG** chọn endpoint dạng `.s3-website-`... vì nó không hỗ trợ HTTPS

![Image](assets/images/2025-05-26-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p2/2.png)

**2. Origin access**

Chọn **"Origin access control settings (recommended)"**, mục đích chỉ cho phép CloudFront đọc dữ liệu của S3 thông qua OAC

>Bạn có thể hiểu đơn giản như sau: 
>- Nếu không dùng OAC, thì bạn sẽ phải mở bucket S3 để public access, tức là ai biết link file cũng có thể tải về → ⚠️ Nguy hiểm
>- Nếu dùng OAC, thì:
>    - Bucket của bạn private hoàn toàn
>    - Chỉ CloudFront mới có quyền đọc dữ liệu từ S3 (qua chữ ký bảo mật SigV4)
>    - Bạn vẫn phân phối nội dung toàn cầu mà không lo có ai đó truy cập bucket của mình
{: .prompt-tip }

**3. Origin access control**

Nếu bạn chưa có OAC nào có thể click **Create new OAC** để tạo một OAC mới:
  - **Name**: Đặt tên cho OAC, ví dụ mình đặt `oac-lab-07`
  - **Signing behavior**: Chọn `Sign requests (recommended)`
  - Ở đây, mình bỏ qua tùy chọn `Do not override authorization header` (vì mình dùng cho static website, không cần xử lý auth đặc biệt)

Click `Create`

![Image](assets/images/2025-05-26-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p2/3.png){: width="972" height="589" }

⚠️ Sau khi chọn OAC, chúng ta sẽ thấy cảnh báo “You must update the S3 bucket policy”. Bạn **có thể cập nhật policy sau khi tạo CloudFront xong**, AWS sẽ cung cấp đoạn JSON sẵn cho bạn dán vào bucket.

![Image](assets/images/2025-05-26-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p2/4.png)

---

#### Cấu hình Default cache behavior (hành vi phân phối)

- **Compress objects automatically**: Chọn `Yes` để tự động nén file tĩnh (HTML, CSS, JS...) trước khi gửi đến trình duyệt, tăng tốc độ tải trang.

- **Viewer protocol policy**: `Redirect HTTP to HTTPS`  
  Giúp người dùng luôn truy cập HTTPS, tăng bảo mật

- **Allowed HTTP methods**: `GET, HEAD` (Static web chỉ cần 2 method này)

- **Restrict viewer access**: chọn `No`

![Image](assets/images/2025-05-26-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p2/5.png)

- **Cache key and origin requests**: chọn `Cache policy and origin request policy (recommended)`, trong phần Cache policy chọn `CachingOptimized`

![Image](assets/images/2025-05-26-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p2/6.png)

#### Cấu hình Function associations

- **Function associations** là nơi cho phép bạn gắn thêm các hàm xử lý (như CloudFront Function) để can thiệp vào request hoặc response – ví dụ như redirect, thêm header, kiểm tra điều kiện...

- Tuy nhiên, vì mình chưa tạo sẵn CloudFront Function ở bước này nên cứ để mặc định “No association”.
Mình sẽ quay lại cấu hình sau khi tạo function để redirect từ `www.domain` về `domain` trong phần sau của bài.

![Image](assets/images/2025-05-26-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p2/7.png)

#### Cấu hình Web Application Firewall (WAF)

- Chọn `Do not enable security protections`

![Image](assets/images/2025-05-26-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p2/8.png)

#### Cấu hình Setting

- **Price class:** Chọn region mà CloudFront sẽ phân phối nội dung tới. Chọn `Use all edge locations (best performance)`

- **Alternate domain name (CNAME):** dùng để gán domain thật. Mình thêm cả domain `www.khanhphan.blog` để sau đó có thể redirect từ `www.domain` về `domain`

![Image](assets/images/2025-05-26-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p2/9.png)

- **Custom SSL certificate**: gán chứng chỉ SSL mà chúng ta đã làm ở phần 1

![Image](assets/images/2025-05-26-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p2/10.png)

- **Default root object:** gõ file truy cập gốc (/) vào đây, bài trước mình làm là `index.html`

![Image](assets/images/2025-05-26-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p2/11.png)

Bấm **Create Distribution**. Chờ vài phút để CloudFront khởi tạo

![Image](assets/images/2025-05-26-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p2/12.png)

📌 **Ghi lại Distribution ID** để dùng cho các bước sau sau (invalid cache, gắn vào S3 policy...)

![Image](assets/images/2025-05-26-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p2/13.png)

ID ở đây là phần đuôi của ARN

```yaml
arn:aws:cloudfront::898508216915:distribution/EZ5U1BANEK2AO
                                              ↑ Đây chính là ID
```
Ta được Distribution ID `EZ5U1BANEK2AO`

---

### 2. Update Bucket Policy cho S3 (dùng với OAC)

Sau khi tạo CloudFront Distribution với Origin Access Control (OAC), bạn cần update Bucket policy của S3 để cấp quyền cho CloudFront có thể đọc dữ liệu từ S3 bucket của bạn.

Nếu không làm bước này, người truy cập website sẽ gặp lỗi 403 Access Denied.

Sau khi tạo xong Distribution, CloudFront sẽ cung cấp sẵn cho bạn template của policy, bạn chỉ việc copy nó và update vào S3 của mình thôi.

Truy cập vào CloudFront Distribution vừa tạo, chọn tab `Origin`

![Image](assets/images/2025-05-26-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p2/14.png)

Click vào Origin name, click `Edit`

![Image](assets/images/2025-05-26-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p2/15.png)

Click nút `Copy Policy` để copy policy mà CloudFront đề xuất, sau đó click `Go to S3 bucket permissions`

![Image](assets/images/2025-05-26-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p2/16.png)

AWS sẽ đẩy bạn qua S3 Bucket của mình, bạn click tab `Permission` để tiến hành update lại Bucket Policy

![Image](assets/images/2025-05-26-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p2/17.png)

Kéo xuống tới Section `Bucket policy`, click Edit để dán policy trước đó vào

![Image](assets/images/2025-05-26-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p2/18.png)

`"Version": "2008-10-17"` trong policy hơi cũ, nên dùng bản mới "2012-10-17", chỗ này giúp mình sữa lại "2012-10-17" 

Click `Save changes` để lưu lại policy mới

![Image](assets/images/2025-05-26-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p2/20.png)

Nếu S3 bucket của bạn trước đó không bật `Block all public access` thì bạn cần bật lại nó. Bởi vì nếu bạn đã cấu hình CloudFront dùng Origin Access Control (OAC) để truy xuất nội dung từ S3, thì chỉ CloudFront mới cần quyền đọc từ S3. Nếu không bật tùy chọn này, người ngoài vẫn có thể truy cập đường dẫn trực tiếp đến file trong bucket nếu họ biết URL.

Trong Section `Block public access (bucket settings)`, click Edit

![Image](assets/images/2025-05-26-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p2/21.png)

Click `Block all public access`, sau đó click `Save changes`

![Image](assets/images/2025-05-26-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p2/22.png)

Sau đó, mình lấy `Distribution domain name` để truy cập vào static web của mình [https://d1ghtn35286knf.cloudfront.net](https://d1ghtn35286knf.cloudfront.net)

![Image](assets/images/2025-05-26-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p2/23.png)

### 3. Trỏ domain về CloudFront với Route 53

Sau khi tạo CloudFront Distribution để phân phối nội dung website tĩnh từ S3, việc tiếp theo là trỏ tên miền cá nhân như (`khanhphan.blog` và cả `www.khanhphan.blog` nếu muốn) về CloudFront. Đây là bước quan trọng để người dùng truy cập website của bạn bằng domain thật sự, thay vì URL mặc định như `https://d1ghtn35286knf.cloudfront.net`.

>**Vì sao cần tạo bản ghi A (Alias)?**
>
>Trong hệ thống DNS, bản ghi A thường trỏ đến địa chỉ IP. Tuy nhiên, với các dịch vụ AWS như CloudFront, ta không biết IP cụ thể vì nó là hệ thống phân phối toàn cầu. Do đó AWS cung cấp **bản ghi A kiểu Alias** – cho phép bạn:
>
>- Trỏ domain gốc (`khanhphan.blog`) **về CloudFront**
>- Trỏ subdomain (`www.khanhphan.blog`) **về CloudFront**
>- Sử dụng HTTPS với SSL (đã gắn SSL certificate từ ACM)
{: .prompt-tip }

Vào Route 53 > Hosted Zones > Chọn **Hosted Zone** domain của bạn (ví dụ của mình `khanhphan.blog`) > Bấm `Create record`

**📌 Tạo bản ghi cho domain gốc `khanhphan.blog`**

- **Record name**: *(để trống – áp dụng cho root domain)*
- **Record type**: A – IPv4 address
- **Alias**: Enable
- **Alias target**: chọn **CloudFront distribution của bạn** (ví dụ: `d1ghtn35286knf.cloudfront.net`)
- **Routing policy**: Simple
- **Evaluate target health**: No

Bấm **Create record**

![Image](assets/images/2025-05-26-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p2/24.png)

**📌 Tạo bản ghi cho subdomain `www.khanhphan.blog`**

Lặp lại bước trên, chỉ khác:

- **Record name**: `www`
- Các phần khác giữ nguyên như bản ghi root.

![Image](assets/images/2025-05-26-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p2/25.png)

Khi hoàn tất, bạn sẽ có 2 bản ghi A kiểu Alias:

| Name                  | Type | Alias Target                         |
|-----------------------|------|--------------------------------------|
| khanhphan.blog        | A    | d1ghtn35286knf.cloudfront.net        |
| www.khanhphan.blog    | A    | d1ghtn35286knf.cloudfront.net        |

Bạn có thể kiểm tra bằng cách dùng lệnh:

```bash
dig khanhphan.blog +short
dig www.khanhphan.blog +short
```

Nếu bạn thấy nó trả về địa chỉ IP của CloudFront tức là đã trỏ về thành công. Sau đó bạn tiến hành truy cập domain của mình trên trình duyệt, ví dụ của mình `https://khanhphan.blog`, mình đã truy cập được static website của mình rồi nè. Ok giờ mình chỉ còn phần redirect nữa thôi.

![Image](assets/images/2025-05-26-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p2/26.png)

### 4. Redirect từ www → non-www với CloudFront Function

Sau khi đã trỏ cả `www.khanhphan.blog` và `khanhphan.blog` về CloudFront, có một vấn đề hay gặp: người dùng truy cập vào `www.khanhphan.blog` nhưng nội dung chính lại nằm ở `khanhphan.blog`.

Do đó, chúng ta cần có một feature đó là tự động **redirect từ www → non-www** (301 redirect). Ở tính năng này chúng ta sẽ sử dụng CloudFront Function, vậy CloudFront Function là gì?

>CloudFront Function là một hàm nhỏ chạy ở **Edge Location**, xử lý các request trước khi đến origin. Nó cực kỳ nhẹ và nhanh, rất phù hợp để thực hiện redirect như:
>- www → non-www
>- HTTP → HTTPS (nếu chưa làm ở behavior)
>- thêm hoặc xoá trailing slash
{: .prompt-tip }

#### 4.1 Tạo CloudFront Function

Vào AWS Console > CloudFront > Functions > Chọn **Create function**

| Trường | Giá trị |
|--------|--------|
| Name | `redirect-www-to-root` |
| Comment | `Redirect www to non-www` |
| Runtime | `cloudfront-js-2.0` |

Bấm **Create function**

![Image](assets/images/2025-05-26-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p2/27.png)

Dán đoạn code dưới đây vào phần editor

```javascript
function handler(event) {
  var request = event.request;
  var host = request.headers.host.value;

  if (host.startsWith("www.")) {
    var newHost = host.replace(/^www\./, "");
    var redirectUrl = `https://${newHost}${request.uri}`;

    return {
      statusCode: 301,
      statusDescription: "Moved Permanently",
      headers: {
        location: { value: redirectUrl }
      }
    };
  }

  return request;
}
```

Bấm `Save changes` để lưu code lại. Sau đó bạn bấm vào tab `Publish`, click `Publish Function`

![Image](assets/images/2025-05-26-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p2/28.png)

Để kiểm tra xem function có được publish thành công hay không thì ở trong section Function code, bạn click tab Live sẽ thấy function đã được thêm vào như ảnh

![Image](assets/images/2025-05-26-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p2/29.png)

#### 4.2 Gắn function vào Behavior của CloudFront

Vào CloudFront > Chọn Distribution của bạn > Tab `Behaviors`, chọn behavior mặc định Default (*) > Click `Edit`

![Image](assets/images/2025-05-26-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p2/30.png)

Tìm section `Function associations` bên dưới, ở `Viewer request`, bạn chọn như sau:

- Function type: CloudFront Functions
- Function ARN / Name: redirect-www-to-root

Sau đó click `Save changes` để lưu lại.

![Image](assets/images/2025-05-26-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p2/31.png)

Sau đó, bạn truy cập `https://www.khanhphan.blog`, nó sẽ tự động redirect về `https://khanhphan.blog`

**🧪 Kiểm tra**

Mình sẽ kiểm tra bằng command sau

```bash
curl -I https://www.khanhphan.blog
```

Nếu kết quả trả về có nội dung như sau thì bạn đã redirect thành công

```yaml
HTTP/2 301
location: https://khanhphan.blog/
```

![Image](assets/images/2025-05-26-trien-khai-static-website-tren-aws-s3-voi-cloudfont-route-53-acm-redirect-p2/32.png)

## ✅ Kết luận

Sau hành trình từng bước cấu hình, kiểm tra, và triển khai — cuối cùng chúng ta cũng đã hoàn tất toàn bộ hạ tầng website cá nhân chuẩn production trên AWS.

Mình tin rằng sau khi hoàn tất bài lab này, bạn không chỉ biết cách làm static website với S3, mà còn thực sự hiểu:
- Làm sao để một domain truy cập được trên khắp thế giới
- Làm sao để giữ website bảo mật mà vẫn truy cập nhanh
- Làm sao để giải quyết từng lỗi nhỏ, từng bước cấu hình quan trọng mà AWS không thể hiện rõ.

Nếu bạn thấy hữu ích, hãy cho mình xin một Stars cho repo Git của bài viết này nha. Xin cảm ơn mọi người.

[![Repo](https://img.shields.io/badge/GitHub-ptmkhanh29%2Faws--s3--static--website--using--cloudfront--route53-blue?logo=github)](https://github.com/ptmkhanh29/aws-s3-static-website-using-cloudfront-route53) [![Stars](https://img.shields.io/github/stars/ptmkhanh29/aws-s3-static-website-using-cloudfront-route53?style=social)](https://github.com/ptmkhanh29/aws-s3-static-website-using-cloudfront-route53/stargazers) [![Forks](https://img.shields.io/github/forks/ptmkhanh29/aws-s3-static-website-using-cloudfront-route53?style=social)](https://github.com/ptmkhanh29/aws-s3-static-website-using-cloudfront-route53/network/members)