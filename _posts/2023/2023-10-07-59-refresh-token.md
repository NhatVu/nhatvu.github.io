---
layout: post
title:  "59. Access token và Refresh token"
date:   2023-10-07 00:01:00 +0000
category: technical
---
Outline 
1. Sơ lược về Access token
2. Tại sao cần có Refresh token?
3. Refresh Token rotation
4. Refresh Token Automatic Reuse Detection

Tôi vẫn luôn tự hỏi nên xử lý Refresh Token thế nào nếu lỡ nó bị leak, dù đã sử dụng cơ chế one-time token. Đến nay, mới thấy thiếu sót, ngoài cơ chế one-time token, nên tiến thêm 1 bước xa hơn nữa là invalid toàn bộ refresh token đã cấp. Bởi theo lý thuyết, 1 refresh token chỉ được sử dụng 1 lần. Nếu nó được sử dụng lần 2, rất có khả năng nó đã bị leak ra ngoài bởi hacker.

Bài này sẽ tổng hợp những kiến thức gom nhặt được về access token và refresh token.

## 1. Sơ lược về Access token 
Thông thường, khi đăng nhập hoặc dùng OAuth, phía backend, cụ thể là Auth service, sẽ trả về cho ta 1 cặp access token và refresh token. 

Access token này được dùng để truy cập vào các protected service thay mặt cho user. Lúc này, ta có thể lấy được thông tin cần thiết. Nếu truyền sai access token, protected service sẽ reject request này. 

Theo lý thuyết, access token có thể bị đánh cắp bởi hacker. Lúc này, khi có access token hợp lệ, hacker có thể truy cập được protected service như là người dùng thực sự. Hệ thống không có cách nào để phân biệt đâu là valid user, đâu là hacker. Để giảm thiểu thiệt hại bởi việc bị leak token, access token thường có life span ngắn, thường là theo giờ hoặc theo ngày. 

Một số hệ thống thiết kế theo dạng white list, tức là sẽ lưu loại toàn bộ token đã được tạo. [1, 2] Mỗi lần verify token, Auth service sẽ kiểm tra database xem token này có hợp lệ hay không. Điều này sẽ gây nghẽn cổ chai tại Auth service khi hệ thống lớn tới 1 mức nhất định. Tuy nhiên, với cách làm white list này, ta có thể thực hiện chức năng thu hồi token. Nhưng nếu life span của token thấp, tôi nghĩ có thể chịu rủi ro mà bỏ qua phần check token trong database này.

Với việc sử dụng thuật toán bất dồng bộ để sign và verify token, ta có thể verify ngay tại protected service mà không cần gọi tới Auth service nữa.[5]


## 2. Tại sao cần có Refresh token?
Tuy nhiên, nếu life span của access token ngắn, sẽ dẫn tới việc user bị log out và đăng nhập lại liên tục. Với các OAuth flow, có thể bypass việc này bằng implicit flow nhưng vẫn tốn 2 request: 1 để tạo Oauth code, 2 để tạo access token. Điều này dẫn tới việc generate 1 lượng lớn oauth code không cần thiết. Để tránh làm ảnh hương tới trải nghiệm người dùng, khi đăng nhập thành công, hệ thống còn trả thêm 1 refresh token và access token sẽ được tạo lại từ refresh token này.

Khác với access token, refresh token có life span dài hơn, nhưng khuyến nghị không quá 1 năm. Hơn nữa, refresh token chỉ tương tác duy nhất với Auth service, không tương tác với protected service. 

## 3. Refresh Token rotation
Cơ chế này đảm bảo rằng, mỗi lần tạo mới access token bằng refresh token, thì 1 cặp access token - refresh token mới được tạo ra. Và refresh token vùa dùng để tạo bị vô hiệu hoá (invalied)

Ví dụ: Ta có sẵn cặp AT 1 - RT 1 \\
Dùng RT 1 ->> tạo AT 2 - RT 2 \\
Lúc này, RT 1 bị vô hiệu hoá. 

## 4. Refresh Token Automatic Reuse Detection
Giả sử refresh token bị leak, hacker dùng refresh token để tạo ra 1 cặp token mới. Hệ thống nên ứng xử như thế nào? Làm sao để phát hiện?

Theo khuyến nghị, hệ thống nên invalid toàn bộ refresh token đã cấp. Lúc này, khi access token hết hạn, user buộc phải đăng nhập lại. Với các này, ta buộc phải lưu danh sách những token đã cấp

Để phát hiện refresh token bị leak, ta sẽ có 1 danh sách hoặc 1 attribute đánh dấu những refresh token đã được sử dụng. Nếu nó được sử dụng lần thứ 2, có khả năng đã bị bên thứ 3 đánh cắp, cho nên toàn bộ refresh token cần phải bị invalid. Rất có khả năng đây ra thao tác sai từ chính user, tuy nhiên, Auth service không phân biệt được đâu là request từ user, đâu là request từ hacker, vậy nên sẽ thực hiện chiến thuật "thà giết nhầm còn hơn bỏ xót".  

Việc invalid toàn bộ refresh token là có lợi cho hệ thống. Tuy nhiên, nếu code phía client hoặc 1 service nào đó không sửa lỗi security, thì vòng lặp user đăng nhập -> cấp token -> invalid token sẽ diễn ra liên tục, gây khó chịu cho user. Có thể detect vụ này bằng 1 thông báo ở api tạo token bằng refresh token nếu nó bị gọi liên tục bởi 1 user hoặc app, có khả năng đã xảy ra vấn đề. Đây là cơ chế Rate Limiter.

So với access token, việc sử dụng refresh token diễn ra không thường xuyên, chỉ khi access token hết hạn. Bởi vậy, lượng tải cũng nhỏ hơn nhiều so với access token.  Do đó, có thể tự tin dùng white list method để lưu lại toàn bộ refresh token (có thể trong 3 hoặc 6 tháng) 

**Referene:** 
1. [Token (JWT) Triển khai hệ thống tự động phát hiện Token đã được sử dụng bởi Hacker và cách xử lý!](https://www.youtube.com/watch?v=1HHvCfAu008)
2. [Token (JWT) Làm sao thu hồi một token bị HACK và một vài câu hỏi về mức độ an toàn khi sử dụng token](https://www.youtube.com/watch?v=93fTk16-st0)
3. [Refresh Token Rotation](https://auth0.com/docs/secure/tokens/refresh-tokens/refresh-token-rotation)
4. [What Are Refresh Tokens and How to Use Them Securely](https://auth0.com/blog/refresh-tokens-what-are-they-and-when-to-use-them/)
5. [58. Mã hoá đồng bộ và bất đồng bộ]({% post_url 2023/2023-09-20-58-ma-hoa-dong-bo-va-bat-dong-bo %})




