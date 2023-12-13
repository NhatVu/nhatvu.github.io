---
layout: post
title:  "63. [System Design] 03. Data Model: Relational Model versus Document Model"
date:   2023-12-12 00:02:00 +0000
category: technical
---

Data model là 1 đặc tả về cách mà database tổ chức dữ liệu. Bài này sẽ tập chung nói về 2 kiểu data model: Relational Model (mô hình dữ liệu quan hệ) và Document model. Theo tôi hiểu, Relational model thường gắn liền với SQL. Còn NoSQL bao gồm nhiều kiểu khác nhau như key-value, document, graph, time series, column-oriented, ... 

## 1. SQL: Relational Model là gì? 
Relational model sẽ tổ chức dữ liệu thành từng bảng (Table in SQL), mỗi bảng gồm nhiều dòng không có thứ tự. Một dòng tương tới với 1 sample/record data. Mỗi dòng sẽ nhiều fields và số fields tương ứng với số cột trong bảng. Thích hợp cho những dữ liệu có cấu trúc (structured data). 

Relational model có lịch sử 1960-1970 và vẫn còn được dùng rộng rãi hiện nay. Chúng thường được dùng cho các transaction processing (banking, sales, airline reservation, stock,...) và batch process (payroll, invoices, reporting) [1]. Theo tôi, những ứng dụng nào cần tính chất ACID (Atomicity, Consistency, Isolation, Durability ) thì sử dụng SQL là hợp lý.

Một số SQL server phổ biến, có thể kể đến như Oracle, MySQL, Postgres, MS SQL server.

## 2. Những hạn chế của Relational Model? 
Tuy nhiên, Relational Model cũng đối mặt với nhiều giới hạn khi lượng người dùng internet tăng lên. Ngoài vấn đề về phí bản quyền, việc fixed data schema, SQL còn đối diện với những khó khăn quan trọng sau: [1][2]
### Agility and Programmability [1][2]: 
Lập trình hướng đối tượng (OOP) rất phố biến hiện nay, tuy nhiên, nếu data lưu dưới dạng relational table, cần thiết phải có 1 layer giữa application code và database để mapping dữ liệu. Tình trạng này gọi là Object-Relational mapping hay impedance mismatch. 

Object-Relational mapping (ORM) framework như Hibernate có thể gỉam đi lượng boilerplate code (note: 1 dạng template code, dùng để map từ row sang object) cần phải viết, nhưg chúng không để nào xoá nhoà đi sự khác biệt về bản chất giưa Object và Relation

### Flexibility (Tính linh hoạt) [2]
Theo lý thuyết, relational model cực kỳ linh hoạt và có thể mô hình bất kỳ dạng dữ liệu nào. Tuy nhiên, trong thực thế, vì quá quen thuộc với relational model, việc thiết kế sẽ tập trung vào dữ liệu dạng bảng, thay vì tập trung vào data storage schema (mô hình lưu trữ dữ liệu) thoả mãn yêu cầu nghiệp vụ (business requirements). Và không phải tất cả thông tin có thể dễ dàng chuyển đổi thành dạng dữ liệu dạng bảng 

Hơn nữa, việc chuẩn hoá dữ liệu (data normalization [3]) sẽ tạo ra nhiều narrow tables (???). Hệ quả là hầu hết queries đều yêu cầu lệnh join giữa các bảng. Điều này thường chậm và sử dụng nhiều tài nguyên. 

Trong sách [1] có ví dụ về Linkedin Profile. Ngoài những thông tin duy nhất cho 1 user như user_id, name, những thông tin khác có thể ở dạng số nhiều như education, contact info, job positions (làm nhiều việc 1 lúc). Để có thể lấy đầy đủ các thông tin này, ta cần phải join tất cả các bảng liên quan như userProfile, education, contact, jobPositions, ... 

### Performance and Scalability [2]
Khi lượng tải tăng, SQL server thường scale up bằng việc tăng thêm resources như RAM, CPU, disk. Tuy nhiên, tới 1 điểm nào đó, việc scale up không còn khả thi nữa, buộc ta phải scale out (tức horizontal scale) bằng việc chia data (partitioning data) thành nhiều database nhỏ hơn và mỗi database này được chạy ở 1 máy khác nhau. Data có thể partition theo row hoặc column. Khi có yêu cầu query dữ liệu, query sẽ được route tới partition thích hợp. Cách làm này được gọi là **sharding**. Một số nhà phát triên SQL server cung cấp giải pháp sharding nhưng kèm với licensing costs. Ta cũng có thể thực hiện manual sharding. 

Tuy nhiên, việc sharding này ảnh hưởng nghiêm trọng tới query performance. Trường hợp cần join data, và data nằm trên các parition khác nhau hoặc máy khác nhau, điều này có nghĩa cần thực hiện di chuyển dữ liệu qua mạng để có thể thực hiện phép join. Hoặc 1 transaction cần update dữ liệu nằm ở nhiều parition khác nhau, điều này ảnh hưởng tới response time của ứng dụng và throughput mạng.

Vì đảm bảo tính ACID, SQL server có những cơ chế phức tạp. Và nó có thể ảnh hưởng tới performance.

## 3. Sự ra đời của NoSQL
Khi việc scale SQL server không đáp ứng kịp với lượng load tăng, các tổ chức lớn quyết định tạo ra giải pháp cho riêng họ. Và từ đó, NoSQL ra đời. NoSQL (viết tắt của Not only SQL) chỉ tập các database không tuân theo chuẩn SQL. NoSQL bao gồm nhiều kiểu khác nhau như key-value, document, graph, time series, column-oriented, ...

Theo [1], một số động lực chính thúc đẩy sự ra đời của NoSQL gồm:
- Việc dễ dàng scale và có khả năng scale mạnh hơn SQL 
- Khả năng xử lý lượng dữ liệu lớn và lượng ghi lớn 
- Open-source, để giảm chi phí licensing 
- Những operation đặc biệt mà không được hỗ trợ bởi SQL, cho những use case đặc biệt
- Dynamic data model 

### 4. Document model 
Document là tập hợp của key và value. Mỗi value có thể là primative type hoặc là list hoặc là child document. Một document sẽ bao gồm toàn bộ data liên quan tới entity, không cần phải join như Relational model. 

Trở lại ví dụ về LinkedIn profile, ta có thể thiết kế 1 userProfile JSON, trong đó bao gồm list education object, list position object, ... Như vậy, khi cần lấy thông tin profile, chỉ cần 1 câu query, không cần phải join dữ liệu. Đặc tính này gọi là *data locality*, tức là những thông tin liên quan đề nằm ở 1 chỗ, và 1 query là đủ [1]

Document model hỗ trợ tốt cho one-to-many relationship. Ví dụ 1 user có nhiều job, nhiều bằng cấp, nhiều địa chỉ. 

### 5. Relational vs Document model 
Theo [1], những lập luận chính thiên hướng về Document model là vì schema flexibility, performance tốt hơn vì data locality, và với một số ứng dụng, data structure "gần gũi" với business requirements hơn.

Tuy nhiên, với document model, ta không thể truy suất trực tiếp với nested document, mà phải đi theo từng cấp. Với những document mà nested level không cao, điều này hoàn toàn ổn.

Document model có thể/hoặc không support join operation. Tuy nhiên điều này không là vấn đề lớn. 

Theo tôi, vì 1 document chứa tất thông tin cần thiết, không cần thực hiện join, nên nếu parition dữ liệu thành nhiều phần, khi thực hiện query, sẽ không gặp phải vấn đề di chuyển dữ liệu như bên SQL. 


**Referenes:** 
1. [Book: Designing Data-Intensive Applications](https://www.amazon.com/Designing-Data-Intensive-Applications-Reliable-Maintainable/dp/1449373321)
2. [Chap 1: Data Storage for Modern High-Performance Business Applications](https://learn.microsoft.com/en-us/previous-versions/msp-n-p/dn313285(v=pandp.10))
3. [Description of the database normalization basics](https://learn.microsoft.com/en-us/office/troubleshoot/access/database-normalization-description)
