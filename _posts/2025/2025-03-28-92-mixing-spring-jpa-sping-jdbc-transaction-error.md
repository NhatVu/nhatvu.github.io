---
layout: post
title:  "92. Mixing Spring JPA and Spring JDBC error: Row was updated or deleted by another transaction (or unsaved-value mapping was incorrect)"
date:   2025-03-28 00:02:00 +0000
category: technical
---

Gần đây, tôi gặp 1 trường hợp lỗi rất thú vị khi 1 project sử dụng 2 công nghệ khác nhau cho persistent layer là Spring JPA và Spring JDBC. 

Về tổng quan, Spring JDBC sẽ handle ở low level, sử dụng trực tiếp JDBC driver nhưng sẽ được viết lại các method để tránh boilerplate code. Spring JPA là 1 đặc tả ORM của Spring, và mặc định sử dụng Hibernate ORM. Với Spring JPA, Hibernate sẽ quyết định khi nào câu lệnh được gửi tới database. Do đó, khi gọi JPA.save(), ta sẽ không biết khi nào câu lệnh SQL insert/update sẽ được gửi tới database. 

## Tình huống
Tình huống tôi gặp, khái quát như sau. Giả sử có 1 danh sách list<Student>, gồm student A, B, và C. Và các học sinh này đã có tồn tại trong database 
```java 
jpa.saveAll(list);
jdbc.delete(học sinh C)
```
Sau khi kết thúc hàm, tới lúc commit transaction, chương trình báo lỗi: <span style="background:yellow">*Row was updated or deleted by another transaction (or unsaved-value mapping was incorrect)* </span>

Tôi sẽ liệt kê các giả định đã đặt ra và cách kiểm chứng chúng. 

## Giả định 1: Spring JPA và Spring JDBC không cùng share chung 1 transaction

Giả định này dựa trên error message: "Row was updated or deleted by another transaction". 

Tuy nhiên, sau khi checking DataSourceConfig, và nội bộ Spring JDBC code, có thể kết luận rằng, Spring JPA và Spring JDBC trong trường hợp tôi gặp phải, cùng sử dụng 1 active connection và cùng 1 transaction (xem thêm hàm `org.springframework.jdbc.datasource.DataSourceUtils.getConnection()`)

Để kiểm tra lại, có thể kiểm tra bảng `SELECT * FROM pg_stat_activity` của postgres, có thể thấy, khi thực hiện câu lệnh jpa và jdbc, chúng đều sử dụng chung 1 connection. 

### Giả định 2: Mixing between JPA and JDBC.

Sau khi hỏi Copilot, nghiêng về giả định việc sử dụng chung giữa JPA và JDBC. Để chứng thực, tôi chỉnh application.properties by, cho in các câu lệnh SQL mà Hibernate sẽ thực thi. 

Sau khi kiểm tra log, phát hiện thứ tự thực hiện câu lệnh là: 
```sql 
select * from ....  -- from JPA 
delete A -- from JDBC 
Update A -- from JPA 
```

Từ thứ tự thực hiện này, có thể nghiêng về lỗi : *unsaved-value mapping was incorrect*

Để khắc việc việc này, ta có thể thêm hàm jpa.flush(). Gặp hàm này, hibernate sẽ ngay lập tức gửi câu SQL tới database, như thế, ta có thể đảm bảo câu lệnh sql được thực hiện theo mong muốn của ta. Khi đó, thứ tự sql sẽ là:
```java
jpa.saveAll(list)
jpa.flush()
jdbc.delete(A)
```

```sql 
select * from ....  -- from JPA 
Update A -- from JPA 
delete A -- from JDBC 
```

Với việc thêm jpa.flush(), đã fix được lỗi trên.

### Kết luận 
Ta không nên kết hợp giữa 2 công nghệ khác nhau khi chúng cùng 1 mục đích, vì có thể tạo nên những lỗi "kỳ lạ". 

Mặc đù ORM rất tiện cho nhiều trường hợp nhưng mình không phải là fan của ORM framework =)))) 


