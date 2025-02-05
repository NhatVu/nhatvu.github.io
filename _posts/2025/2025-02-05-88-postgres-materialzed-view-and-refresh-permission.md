---
layout: post
title:  "88.  PostgreSQL: Materialized View and Refresh Permission"
date:   2025-02-05 00:02:00 +0000
category: technical
---

- [1. Materialized View (MatView) là gì?](#1-materialized-view-matview-là-gì)
- [2. Refresh MatView?](#2-refresh-matview)
- [3. Vấn đề permission khi refresh](#3-vấn-đề-permission-khi-refresh)
- [4. References](#4-references)


### 1. Materialized View (MatView) là gì? 

MatView là 1 bảng ảo được tạo bởi câu SELECT query, và lưu trữ lại kết quả vào hard disk. Khác với View, View không lưu trữ kết quả ở đĩa cứng.

Vì kết quả đã được tính toán sẵn, nên khi có câu lệnh query vào MatView, nó sẽ không tính toán lại. Điều này giúp câu lệnh nhanh hơn. 

Tuy nhiên, nó gặp phải 3 khó khăn chính [1]
- Làm hệ thống trở nên phức tạp hơn, gia tăng độ phức tạp khi maintenance.
- Dữ liệu trên view không up-to-date. Với Postgres, cần phải refresh định kì và thủ công. 
- Postgres chỉ hỗ trợ Full-refresh, tính toán lại toàn bộ view nên quá trình này tiêu tốn nhiều tài nguyên.

### 2. Refresh MatView?

Giả sử ta có view name: mymatview. Để refresh MatView, ta có thể dùng câu lệnh: 

```sql
REFRESH MATERIALIZED VIEW mymatview;
```

Tuy nhiên, câu lệnh trên sẽ block mymatview, các concurrent user khác sẽ không thể truy cập khi view đang refresh. Điều này là không thể chấp nhận nếu matview phục vụ real-time user 

Để khắc phục điều này, ta có thể thêm keyword CONCURRENTLY
```sql
REFRESH MATERIALIZED VIEW CONCURRENTLY mymatview;
```

Thêm CONCURRENTLY sẽ khiến việc refresh view chậm hơn. Tuy nhiên, các concurrent user khác có thể sử dụng view khi view đang refresh. Muốn sử dụng CONCURRENTLY, buộc phải tạo 1 unique index cho view. Nếu không, sẽ báo lỗi. [4] 


### 3. Vấn đề permission khi refresh 
Một lưu ý rất quan trọng, theo [4], "To execute this command you must be the owner of the materialized view". Tức là, để chạy lệnh refresh, user thực thi câu lệnh phải là owner của MatView. 

Tuy nhiên, sẽ có trường hợp ta quản lý schema, table và view bằng Flyway. Ta dùng flyway_user để tạo table, matview. Sau khi tạo xong, sẽ lock flyway_user lại. Lúc này, owner của matview là flyway_user nhưng nó đã bị lock. Làm sao để ta thực hiện scheduler refresh? 

Giả sử user ta dùng cho webApp là my_user. Nếu sử dụng my_user để refresh, sẽ báo lỗi `ERROR:  must be owner of materialized view test_mv`

[2] cho ta 1 lựa chọn hợp lý. Ta sử dụng role membership để xử lý chuyện này. 

- Bước 1: tạo 1 role có tên `owner_mat_view`. Cần đảm bảo role mới tạo có quyền đọc tất cả các bảng trong câu lệnh tạo view. 
- Bước 2: Đổi owner của view về role owner_mat_view. `ALTER MATERIALIZED VIEW test_mv OWNER TO owner_mat_view;`
- Bước 3: Gán quyền của role owner_mat_view cho user my_user `GRANT owner_mat_view TO my_user;` Lúc này, my_user sẽ kế thừa toàn bộ quyền của owner_mat_view.
- Bước 4: Đổi role sang my_view. `SET ROLE my_user;`
- Bước 5: Thực hiện refresh với user my_user. `REFRESH MATERIALIZED VIEW test_mv;` ==> thành công 



### 4. References 
1. [What is a Materialized View?](https://aws.amazon.com/what-is/materialized-view/)
2. [Postgres Permissions and Materialized Views](https://blog.rustprooflabs.com/2021/07/postgres-permission-mat-view)
3. [39.3. Materialized Views](https://www.postgresql.org/docs/current/rules-materializedviews.html)
4. [REFRESH MATERIALIZED VIEW](https://www.postgresql.org/docs/16/sql-refreshmaterializedview.html)

