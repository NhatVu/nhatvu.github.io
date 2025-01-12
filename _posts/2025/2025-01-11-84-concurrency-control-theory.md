---
layout: post
title:  "84.  Concurrency control theory"
date:   2025-01-11 00:02:00 +0000
category: technical
---
- [1. Transaction](#1-transaction)
- [2. Concurrency Control](#2-concurrency-control)
  - [2.1. Unrepeatable Reads](#21-unrepeatable-reads)
  - [2.2. Dirty Reads](#22-dirty-reads)
  - [2.3. Lost Updates](#23-lost-updates)
  - [2.4. Phantom Read](#24-phantom-read)
- [3. Exclusive lock và Shared lock](#3-exclusive-lock-và-shared-lock)
  - [3.1. Exclusive lock (Khoá độc quyền)](#31-exclusive-lock-khoá-độc-quyền)
  - [3.2. Shared lock](#32-shared-lock)
- [4. References](#4-references)



Khi sử dụng database, câu hỏi đặt ra nếu nhiều user read/write cùng 1 tài nguyên tại cùng 1 thời điểm, database sẽ xử lý như thế nào? 

Để hiểu rõ hơn cách database xử lý, trước hết ta cần hiểu tổng quan các khái niệm về Concurrency control theroy. Bài này tổng hợp từ 1 bài giảng của trường CMU - Introduction to database [1] 

Bài viết có sử dụng ChatGPT.

### 1. Transaction 
Transaction (giao dịch) là một tập hợp một hoặc nhiều các thao tác (operations: Read or Write) được thực thi như một đơn vị công việc duy nhất. Các thao tác trong một transaction phải tuân thủ nguyên tắc ACID 

- A - Atomicity (Tính nguyên tử):
  
  - Toàn bộ các thao tác trong một transaction phải được thực hiện toàn bộ hoặc không thực hiện gì cả. 
  - Nếu có lỗi xảy ra giữa chừng, toàn bộ các thay đổi sẽ bị hủy (rollback).

- C - Consistency (Tính nhất quán):
  - Transaction phải đảm bảo dữ liệu luôn ở trạng thái hợp lệ, không vi phạm các ràng buộc (Integrity constrains như foreign key, unique, ...) hoặc quy tắc kinh doanh.

- I - Isolation (Tính độc lập):
  - Các transaction đang chạy đồng thời không được gây ảnh hưởng lẫn nhau. 

- D - Durability (Tính bền vững):
  - Sau khi một transaction hoàn tất (commit), các thay đổi phải được lưu vĩnh viễn, ngay cả khi hệ thống gặp sự cố. Database có thể sử dụng WAL để đảm bảo durability

![alt text](/assets/images/2025/84_aicd_overview.png)

### 2. Concurrency Control 
Concurrency control protocol là cách mà database quyết định interleaving (xen kẽ) operations từ nhiều transaction tại runtime.

![alt text](/assets/images/2025/84_serial_execution.png)

Trong ví dụ trên, cả 2 kết quả đều hợp lệ, dù giá trị A và B là khác nhau khi chạy theo thứ tự T1 -> T2 và T2 -> T1. Thực tế là ta không thể control thứ tự chạy của T1 và T2.

Có một số dạng conflict phổ biến
#### 2.1. Unrepeatable Reads
![alt text](/assets/images/2025/84_unrepeatable_read.png)

Như định nghĩa, unrepeatable read xảy ra khi trong cùng 1 transaction, giá trị của 1 object trả về khác nhau sau mỗi lần gọi. Trong ví dụ trên, trong T1, giá trị của A là lần gọi đầu tiên là 10, ở lần gọi 2 là 19. 

#### 2.2. Dirty Reads
![alt text](/assets/images/2025/84_dirty_read.png)

Transaction read dữ liệu được ghi bởi transaction khác, nhưng transaction đó chưa commit. 

Ví dụ Transaction 1 cập nhật giá trị của A lên 12, chưa commit, và sau đó, transaction 2 đọc giá trị A là 12. 

#### 2.3. Lost Updates 
![alt text](/assets/images/2025/84_lost_update.png)

Khi hai transaction cùng đọc và ghi vào cùng một dữ liệu, kết quả ghi của transaction này có thể ghi đè kết quả của transaction kia.

Trong ví dụ trên, kết quả sau cùng sẽ là A = 19, B = Alice ==> không hợp lệ 

Giá trị hợp lệ sẽ là, hoặc (A=10, B=Alice) hoặc (A=19, B=Bob)

#### 2.4. Phantom Read 
Một transaction thực hiện cùng một truy vấn hai lần và thấy số lượng kết quả khác nhau vì một transaction khác đã thêm hoặc xóa dữ liệu.

Ví dụ:

- Transaction 1 chạy truy vấn: SELECT * FROM accounts WHERE balance > 100.
- Transaction 2 thêm một tài khoản mới có balance = 200 và commit.
- Transaction 1 chạy lại truy vấn và thấy kết quả thay đổi.


### 3. Exclusive lock và Shared lock
Exclusive lock và Shared lock là 2 khái niệm nên biết trong Concurrency Control theory 

#### 3.1. Exclusive lock (Khoá độc quyền)
Loại khóa này đảm bảo rằng chỉ một giao dịch duy nhất có thể truy cập vào tài nguyên (như một hàng, một bảng) tại cùng một thời điểm.

Các giao dịch khác bị chặn cho đến khi giao dịch giữ khóa độc quyền kết thúc (COMMIT hoặc ROLLBACK).

Khoá Exclusive lock được sử dụng cho tác tác vụ ghi như INSERT, UPDATE, DELETE 

```sql

-- Giao dịch 1: Cập nhật dữ liệu và giữ Exclusive Lock
BEGIN;

UPDATE test_table SET value = '150' WHERE id = 1;

-- Sau đó, trong transaction 1, row có id = 1, value = 150. Để transction tiếp tục chạy 

-- Giao dịch 2: Cố gắng đọc hoặc ghi dữ liệu cùng dòng sẽ bị chặn.
BEGIN;

SELECT * from test_table where id = 1;
-- câu lệnh trả về kết quả id = 1, value = 'Updated value'. Giá trị cũ.
```

#### 3.2. Shared lock 

Loại khóa này cho phép nhiều transaction đọc dữ liệu cùng lúc, nhưng không cho phép transaction khác ghi dữ liệu lên tài nguyên đang bị khóa chia sẻ.

Tài nguyên bị khóa chia sẻ vẫn có thể bị khóa thêm bởi các giao dịch khác, miễn là chúng không yêu cầu Exclusive Lock.

Khi một giao dịch thực hiện các thao tác đọc có khóa (LOCK FOR SHARE), như: SELECT ... FOR SHARE

Dùng trong các trường hợp cần đảm bảo dữ liệu không bị thay đổi trong khi đang đọc.

```sql
-- Giao dịch 1: Đọc dữ liệu với Shared Lock
BEGIN;

SELECT * FROM test_table WHERE id = 1 FOR SHARE;

-- Giao dịch 2: Cũng có thể đọc dữ liệu với Shared Lock.
BEGIN;

SELECT * FROM test_table WHERE id = 1 FOR SHARE;

-- Giao dịch 3: Cố gắng cập nhật hoặc xóa dữ liệu
-- Default behaviour: câu lệnh dưới sẽ rơi vài trạng thái WAITING. Chờ cho transaction 1 và 2 commit hoặc rollback, sau đó sẽ thực hiện. Tuy nhiên, hết wait_timeout, sẽ báo lỗi. 
BEGIN;

UPDATE test_table SET value = '200' WHERE id = 1; 

```



### 4. References 
1. [#16 - Concurrency Control Theory ✸ Firebolt Database Talk (CMU Intro to Database Systems)](https://www.youtube.com/watch?v=4y656idl8ZU&list=PLSE8ODhjZXjYDBpQnSymaectKjxCy6BYq&index=17)
