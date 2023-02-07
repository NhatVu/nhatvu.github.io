---
layout: post
title:  "29. Timezone cho distributed web service "
date:   2023-02-07 00:01:00 +0000
category: technical
---
## 1. Vấn đề
Ngày tháng khi insert vào database thường có dạng như sau: 

> 2021-03-25 10:00:00

Sẽ là đơn giản nếu local and server machine có chung timezone. Hoặc với những quốc gia có timezine cố định như Việt Nam (chỉ có 1 timezone, không có day light saving) thì vấn đề này ít xảy ra. 

Tuy nhiên, với sự phát triển của cloud, một hệ thống server có thể deploy ở nhiều vùng khác nhau (Anh, Mĩ, Ấn, ...). Hoặc server và client được deploy ở nhiều vùng khác nhau. Với dữ liệu format như trên, ta không thể suy luận được, vì không có timezone.

## 2. Một số lưu ý 
### 2.1 Không lưu datetime dưới dạng varchar/text
Không lưu ngày tháng dưới dạng text vì nó sẽ mất đi những hỗ trợ của RDMS. Quan trọng nhất là ko có sự hỗ trợ cho trường hợp day light saving, thứ rất khó để xử lý bằng tay. Thêm nữa, với dạng text, phải chọn format phù hợp để sorting operation đúng đắn. Do đó, luốn lưu ngày tháng dưới dạng datetime hoặc timestamp.    

### 2.2 Luôn lưu UTC timezone trên server, dùng server time. 
Với UTC format, dễ dàng chuyển đổi qua lại giữa các timezone khác nhau. Hơn nữa, ta cũng không cần phải lưu thông tin timezone trong database. 

Khi cần hiển thị thời gian tại cliet, chuyển đổi UTC thành user's timezone 
Tóm lại, với server, lưu bằng UTC. Với client, chuyển đổi UTC bằng user's timezone 

### 2.3 Không xử lý timezone một cách thủ công khi dùng framework 
>
#for plain hibernate \
hibernate.jdbc.time_zone=UTC
>
#for Spring boot jpa \
spring.jpa.properties.hibernate.jdbc.time_zone=UTC

### 2.4 Tránh sử dụng legacy package java.util. Dùng java.time 
Không dử dụng các class cũ, vì rất phức tạp. Từ java 8 trở đi, sử dụng package java.time. 

Với timezone, tránh sử dụng offset, mà hãy dùng ZoneId. Vì offset của 1 zoneId có thể thay đổi tùy theo điều chỉnh của chính phủ. Ví dụ, hiện tại ZoneID của Hà Nội là +7. Nhưng sau này, chính phủ có thể chỉnh thành +7.5. Tuy nhiên, cũng phải kiểm tra xem JDBC driver hỗ trợ OffsetDateTime hay ZoneDateTime.  



References: 
1. [Guide to time zone handling for REST APIs in Java](https://www.linkedin.com/pulse/guide-time-zone-handling-rest-apis-java-anushka-darshana/)
2. [How to Handle Database Timezones](https://www.databasestar.com/database-timezones/)
3. [An introduction to java.time](https://yawk.at/java.time/)