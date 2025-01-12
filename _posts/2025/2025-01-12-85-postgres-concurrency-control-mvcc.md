---
layout: post
title:  "85.  PostgreSQL Concurrency control: MVCC"
date:   2025-01-12 00:02:00 +0000
category: technical
---
- [1. Giới thiệu: MVCC](#1-giới-thiệu-mvcc)
  - [1.1. Các nguyên tắc chính:](#11-các-nguyên-tắc-chính)
  - [1.2. Ưu Điểm Của MVCC](#12-ưu-điểm-của-mvcc)
  - [1.3. Hạn Chế Của MVCC](#13-hạn-chế-của-mvcc)
  - [1.4. Demo](#14-demo)
- [2. References](#2-references)

Series bài này tổng hợp các kiến thức liên quan tới concurrency control cho Postgres. Mục đích để hiểu rõ hơn cách thức PostgreSQL quản lý 2 hoặc nhiều session truy cập 1 dữ liệu tại cùng 1 thời điểm. 

Bài viết có sử dụng ChatGPT để tổng hợp và tạo ví dụ 

### 1. Giới thiệu: MVCC 
PostgreSQL đảm bảo data consistency bằng multiversion model (Multiversion Concurrency Control, viết tắt là MVCC). Lưu ý, mỗi database sẽ có cơ chế quản lý concurrency khác nhau. 

MVCC dựa trên ý tưởng lưu trữ nhiều phiên bản (data version hoặc data snapshot) của dữ liệu trong cơ sở dữ liệu. Khi một transaction thực hiện thao tác, như Insert, Update hoặc Delete dữ liệu, **một phiên bản mới của dữ liệu sẽ được tạo ra thay vì ghi đè lên dữ liệu cũ ngay lập tức**. 

Điều này có nghĩa, mỗi SQL statement sẽ thấy một snapshot of data dựa vào thời gian thực thi câu lệnh, không quan tâm tới trạng thái hiện tại của data. Cơ chế này giúp ngăn cản việc sử dụng inconsistent data đang được thay đổi bởi các transaction khác. Sẽ nói rõ hơn ở mục Transaction Isolation

PostgreSQL cung cấp table-level and row-level locking cho những ứng dụng không cần phải lock toàn bộ bảng. 

#### 1.1. Các nguyên tắc chính:
a. Không ghi đè ngay lập tức: Khi một giao dịch sửa đổi dữ liệu, phiên bản gốc vẫn được giữ nguyên cho các giao dịch khác đang hoạt động.
Một bản sao mới của dữ liệu được tạo cho các thay đổi của giao dịch hiện tại.

b. Tách biệt giữa các giao dịch: Giao dịch chỉ "thấy" phiên bản dữ liệu phù hợp với thời điểm nó bắt đầu, tránh ảnh hưởng bởi các giao dịch khác.
  
c. Xóa phiên bản cũ: Các phiên bản cũ của dữ liệu sẽ được giữ lại cho đến khi không còn giao dịch nào cần chúng, sau đó PostgreSQL dọn dẹp bằng VACUUM. Việc thực thi VACUUM sẽ không làm giảm size của table.

#### 1.2. Ưu Điểm Của MVCC
a. Tránh khóa cứng (Hard Lock): MVCC hạn chế việc sử dụng khóa bằng cách cho phép các giao dịch đọc dữ liệu cũ mà không chờ giao dịch khác hoàn thành.
  
b. Cải thiện hiệu năng: Các giao dịch đọc không bị chặn bởi các giao dịch ghi, và các giao dịch ghi sẽ không bị chặn với các giao dịch đọc. Điều này giúp tăng khả năng xử lý đồng thời.

c. Hỗ trợ nhiều mức độ cô lập (Isolation Levels): MVCC dễ dàng hỗ trợ các mức cô lập như Read Committed, Repeatable Read, và Serializable.

#### 1.3. Hạn Chế Của MVCC
a. Tốn không gian lưu trữ: Các phiên bản cũ của dữ liệu sẽ chiếm thêm không gian cho đến khi được dọn dẹp bởi VACUUM. Điều này có thể PostgreSQL không tối ưu cho những ứng dụng cần ghi nhiều, update liên tục. (Ví dụ như Notion) Đây cũng là một trong những lý do khiến Uber chuyển từ Postgres qua Mysql [2]. 

b. Quản lý phức tạp: Việc dọn dẹp phiên bản cũ đòi hỏi thêm tài nguyên và có thể làm tăng thời gian thực hiện.

#### 1.4. Demo 

Tạo table 
```sql
set search_path = mastering_postgres ;

CREATE TABLE test_table (
    id SERIAL PRIMARY KEY,
    value TEXT
);

-- Insert some initial data
INSERT INTO test_table (value) VALUES ('Initial Value');

-- Select ctid của bảng hiện tại
SELECT ctid, xmin, xmax, * from test_table; 

/*
 ctid  |  xmin   | xmax | id |     value
-------+---------+------+----+---------------
 (0,1) | 2357896 |    0 |  1 | Initial Value
*/
```


| **Line Number** | **Transaction 1**                       | **Transaction 2**   |
|--------|---------------------------------------|-------------|
| 1      | BEGIN;            | BEGIN;         |
| 2      | `UPDATE test_table SET value = 'Updated Value' WHERE id = 1;` <br><br>-- At this point, the update is not yet committed. <br> -- Other transactions won't see this change. |    |
| 3      | `SELECT ctid, * FROM test_table;` <br><br> Note that the ctid change to (0, 2) even the transaction has not been commit. | `SELECT ctid, * FROM test_table;` <br><br> Value of (id, value) will be (1, Initial Value) <br> -- The "Updated Value" from Transaction 1 is NOT visible because it's not committed.   <br> (ctid) = (0, 1), still refer to old row      |
| 4      | `COMMIT;` <br> -- The change is now committed and visible to other transactions.           |    |
| 5      |         | `SELECT citd, * FROM test_table;` <br> -- Read again, and see value 'Updated Value'         |
| 6      |         | `COMMIT;`       |


### 2. References 
1. [How does MCVV work?](https://vladmihalcea.com/how-does-mvcc-multi-version-concurrency-control-work/)
2. [Why Uber Engineering Switched from Postgres to MySQL](https://www.uber.com/en-IE/blog/postgres-to-mysql-migration/)
