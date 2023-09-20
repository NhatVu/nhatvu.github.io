---
layout: post
title:  "58. Mã hoá đồng bộ và bất đồng bộ"
date:   2023-09-20 00:01:00 +0000
category: technical
---
Outline 
1. Encode và encryption 
2. Mã hoá đồng bộ (symmetric) và bất đồng bộ (asymmetric)
3. Token và microservice

## 1. Encode và encryption 
### 1.1 Encode 
Encode là quá trình chuyển đổi data sang 1 định dạng mới, dựa theo những mô hình công khai sẵn có (publicly available) và có thể dễ dàng đảo ngược. Qúa trình đảo ngược gọi là decode. 

Encode chỉ chuyển dữ liệu sang 1 định dạng khác để có thể sử dụng trên 1 hệ thống khác. Ví dụ: chuyển từ text sang byte 

Encode không hề bảo mật.

Thuật toán hay sử dụng: ASCII, Unicode, URL Encoding, Base64

### 1.2 Encryption 
Encryption là quá trình chuyển đổi data sang 1 định dạng mới, và chỉ những đối tượng cụ thể mới có thể đảo ngược. Quá trình đảo ngược gọi là decryption

Mục đính của encryption là giữ bí mật cho dữ liệu, chỉ một vài người có key mới biết được nội dung. Đảm bảo chỉ những người đủ điều kiện mới có thể truy suất dữ liệu 

Thuật toán hay sử dụng: AES, RSA 

Thuật ngữ

Plain text --> encryption --> Cipher text 


## 2. Mã hoá đồng bộ (symmetric) và bất đồng bộ (asymmetric)
### 2.1 Symmetric encryption 
Symmetric encryption dùng 1 secret key cho cả encrypt và decrypt. Key này được shared chung

Điểm mạnh là mã hoá đồng bộ nhanh, ít tốt CPU và Cipher text không phình ra so với dữ liệu gốc.

Điểm yếu là tính bảo mật thấp, do phải shared secret key. 

Thuật toán: AES (128, 192 hoặc 256 bit. Phổ biến), ChaCha20. Không dùng DES, RC4

### 2.2 Asymmetric encryption
Asymmetric encryption, còn được gọi là public-key encryption, sử dụng 1 cặp public key - private key: dữ liệu được encrypt bởi private key thì được decrypt bởi public key. Nếu encrypt bởi public key thì được decrypt bởi private key. 2 key này có quan hệ toán học với nhau 

Điểm mạnh là bảo mật hơn symmetric encryption, do không shared private key 

Tuy nhiên, điểm yếu là quá trình encrypt diễn ra chậm, tốn nhiều CPU và cipher text thường phình to hơn dữ liệu gốc.

Thuật toán phổ biến: RSA (phổ biến), Diffie-Hellman, ECDSA, ECDH. Không dùng DSA 

## 3. Token và microservice
Giả sử, ta có 1 kiến trúc microserivce và sử dụng symmetric encryption như sau 
![Symmetric encryption](/assets/images/2023/2023_09_20_symmetric_encryption.png "Symmetric encryption")
- Auth service: có nhiệm vụ login, cấp access token và verify access token 
- Image service, resource service: request tới các service này sẽ kèm theo access token. Nếu access token hợp lệ, được phép sử dụng service 

Thông thường, Auth serivce sẽ làm nhiệm vụ cấp và verify token. Khi Image serivce cần verify token, nó sẽ gọi api của Auth service. Tuy nhiên, việc này cũng tạo ra bottleneck tại Auth serivce tại một vài thời điểm, nhất là khi số lượng service tăng lên. Để tránh điều này, ta có thể verify token ngay tại Image serivce, ko cần gọi Auth service nữa. 

Nếu ta sử dụng symmetric encryption, việc leak secret key là có thể xảy ra. Lúc này, hacker có thể tự tạo 1 valid access token để truy cập vào Image service. Hơn nữa, không thể verify tại client-side, vì không bao giờ được phép để secret key phía client 

Một giải pháp là sử dụng asymmetric encryption. Sử dụng private key để sign và public key để verify. Private Key chỉ tồn tại tại Auth service --> không ai có thể tạo token ngoài Auth service. Public key sẽ được phân phối tới các service thông qua 1 url chứa toàn bộ available public key (URl gọi là json web keys set). Ta có thể cache lại public key này trong 1 khoảng thời gian. Consumer service như Image serivce, sẽ dùng public key để verify token.

![Asymmetric encryption](/assets/images/2023/2023_09_20_asymmetric_encryption.png "Asymmetric encryption")

Lưu ý, với cách làm trên, sau khi verify token tại Image service, ta trực tiếp sử dụng thông tin trong payload của token, mà không kiểm tra lại xem token này có được lưu trong database hay không. Để làm điều này, cần đảm bảo payload trong token không bị thay đổi bởi bên thứ 3. Signature trong Json Web Token cho phép ta thực hiện việc này. 

Tuy nhiên, vì không lưu nội dung token trong database, không thể thực hiện tính năng revoke token tức thời được. (Dù cho có set expried time thấp, nhưng vẫn không tức thời). Do số lượng revoke token không nhiều, ta có thể lưu token bị revoke vào database và kiểm tra mỗi khi verify token. Lúc này, có thể sẽ bị bottleneck ở khúc kiểm tra revoke token. Căn cứ vào lượng request mà chọn phương án phù hợp.

**Referene:** 
1. [How to use JWT with RSA key-pair in micro-services](https://www.youtube.com/watch?v=KQPuPbaf7vk)
2. [Using JWT in a Microservice Architecture](https://dzone.com/articles/using-jwt-in-a-microservice-architecture)
3. [Encryption - Symmetric Encryption vs Asymmetric Encryption - Cryptography - Practical TLS](https://www.youtube.com/watch?v=o_g-M7UBqI8)




