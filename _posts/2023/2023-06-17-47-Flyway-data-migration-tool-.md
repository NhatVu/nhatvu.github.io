---
layout: post
title:  "47. Flyway - Data migration tool"
date:   2023-06-17 00:01:00 +0000
category: technical
---

### Tại sao nên sử dụng Flyway
Trong quá trình phát triển phần mềm, tới 1 giai đoạn nào đó, ta cần phải thay đổi data schema như thêm field, thêm bảng, xóa cột, .... Mặc dù việc này diễn ra không thường xuyên nhưng nếu có mismatch giữa database schema và code, application có thể bị sai hoặc crash. Thêm nữa, với việc phát triển nhiều feature song song với nhau, dev, test, live trên nhiều môi trường, ta cần biết chính xác mỗi môi trường đang sử dụng database schema nào. Từ đó, database migration tool ngày càng trở nên thông dụng hơn. Và Flyway là 1 trong những tool được sử dụng rộng rãi nhất. 

Nếu ta thay đổi database schema thủ công, rất khó hoặc không thể trả lời các câu hỏi sau [1]: 
- What state is the database in on this machine?
- Has this script already been applied or not?
- Has the quick fix in production been applied in test afterwards?
- How do you set up a new database instance?

Với việc dùng Flyway, ta có thể kiểm soát version của database schema. Ta có thể [1]: 
- Recreate a database from scratch
- Make it clear at all times what state a database is in
- Migrate in a deterministic way from your current version of the database to a newer one

Tóm lại, Flyway được xem như là single source of truth of schema.

### Flyway hoạt động như thế nào? 
Với Flyway, mọi hành động làm thay đổi db schema được gọi là migration. Migration có thể là versioned (theo phiên bản) hoặc repeatable. Ở bài này, tôi chỉ nói về versioned mà thôi. 

Flyway sẽ dùng 1 bảng tên **flyway_schema_history** để theo dõi việc migration. Với Versioned migration, ta sẽ có 3 thông tin là version, description và checksum. Mỗi version là duy nhất, description mô tả file script làm gì và checksum dùng để detect accident change cho file script. 

Với Versioned, mỗi file script chỉ được apply 1 lần. Thông tin này sẽ được lưu vào bảng flyway_schema_history. Lần tiếp theo chạy migration, flyway sẽ kiểm tra bảng này, và chạy những script có version lớn hơn version lớn nhất trong bảng. 

Với default config, Flyway sẽ đọc file từ folder **classpath:db/migration**. Versioned migration sẽ có file format là V<VERSION>__<NAME>.sql với VERSION là underscore-seperated version (vd 1 hoặc 2_1 hoặc 2_1_5). Lưu ý, cách giữa VERSION và NAME là double-underscore. 

### Config Flyway trong Spring Boot 
Tham khảo code tại: https://github.com/NhatVu/SpringBootDemo/commit/ca8ff5493795c48fca4d61b4184dde4075aed200

Các bước thông thường là thêm flyway-core vào pom.xml file. Tiếp theo kiểm tra config trong file application.properties, với prefix spring.flyway 

Chi tiết hơn, xem tại mục 18.9.5. Use a Higher-level Database Migration Tool, link: https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#howto.data-initialization.migration-tool.flyway

### Một số lỗi khi dùng Flyway 
#### 1. Checksum mismatch 
```
FlywayException: Validate failed. Migration Checksum mismatch for migration 2
-> Applied to database : 1499248173
-> Resolved locally    : -1729781252
```

Lỗi này do việc thay đổi file sql, dẫn đến checksum thay đổi. Cách đơn giản nhất là cập nhật lại checksum mới có record có file bị thay đổi. 

#### 2. File với version mới không migration 
Việc này thường xảy ra khi merged code từ nhiều feature branch. Flyway chỉ kiểm tra version number, không check toàn bộ file. Phải đảm bảo version ta cần chạy chưa được thực thi bao giờ. 


**Referene:** 
1. [Why database migrations](https://documentation.red-gate.com/fd/why-database-migrations-184127574.html)
2. [18.9.5. Use a Higher-level Database Migration Tool](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#howto.data-initialization.migration-tool.flyway)
