---
layout: post
title:  "81.  Book: SQL Performance Explained"
date:   2024-12-14 00:02:00 +0000
category: technical
---
Mặc dù đã có lịch sử lâu đời nhưng hiện nay, SQL vẫn được sử dụng rộng rãi. Tôi tin ít nhất những người học lập trình đều có một vài môn học liên quan đến SQL. SQL vẫn có chỗ đứng nhất định giữa sự nổi lên của nhiều công nghệ NoSQL như key-value, document based, graph, time-series, ... 

SQL cho phép ta tách biệt giữa what và how. Câu lệnh SQL dễ dàng cho ta biết cần phải thực hiện những gì (what) và ẩn đi việc thực hiện nó như thế nào (how). Thông thường, ta chỉ được dạy về "what" và "how" để database tự vận hành 

Tuy nhiên, khi gặp về vấn đề performance, cần tối ưu hoá câu lệnh SQL, ta cần phải biết một số những kiến thức nhất định về việc database xử lý câu lệnh như thế nào. "Đánh index giúp tăng tốc câu lệnh SQL" là 1 giải pháp thường được nghe. Đúng nhưng chưa đủ. Và trong một số trường hợp, đánh index không có tác dụng. 

Để hiểu rõ hơn về database index và internal implemenation of database, cuốn sách **"SQL Performance Explained"** của Markus Winand [1] là 1 nguồn tham khảo rất có giá trị. Với việc trình bày chi tiết cấu tạo database index, cách database tổ chức lưu trữ dữ liệu dưới dạng block trên disk, các chiến thuật quét dữ liệu khác nhau (ex: full scan, index scan, index only scan, bitmap scan), ... ta có thể hiểu rõ hơn nên index như thế nào, index những cột nào để tối ưu hoá câu SQL. 

Với bản web edition, tác giả cung cấp script để tạo database và dữ liệu test. Do đó, ta có thể vừa đọc, vừa thực hành những kiến thức vừa học. 

Tôi tin rằng, cuốn sách sẽ mang lại giá trị nhất định cho những bạn muốn tìm hiểu về tối ưu hoá SQL. 

Ngoài ra, cuốn **"PostgreSQL Query Optimization"** cũng là 1 nguồn tham khảo tốt. Tuy nhiên, cuốn này không có bản web edition.


### References 
1. [Book: SQL Performance Explained by Markus Winand](https://use-the-index-luke.com/)
2. **PostgreSQL Query Optimization** by Henrietta Dombrovskaya, Boris Novikov, Anna Bailliekova (Apress, 2021).

