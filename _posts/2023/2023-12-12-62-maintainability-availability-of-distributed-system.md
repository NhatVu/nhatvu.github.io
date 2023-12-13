---
layout: post
title:  "62. [System Design] 02. Maintainability và Availability của hệ thống phân tán"
date:   2023-12-12 00:02:00 +0000
category: technical
---

Bài tiếp sẽ nói về 2 yếu tố còn lại là Maintainability (khả năng bảo trì) và Availability (Tính sẵn sàng). [1][2]

## 1. Maintainability (Khả năng bảo trì)
Trong vòng đời phát triển phần mềm, thông thường, thời gian phát triên hệ thống chỉ chiếm 20% nguồn lực, 80% còn lại dùng để bảo trì và vận hành. Bảo trì ở đây có thể là sửa lỗi (fix bugs), thêm một tính năng mới (add a new feature), tương thích với platform/framework/library (ví dụ 1 lib CVE - Common Vulnerabilities and Exposures), ... Những hệ thống cũ còn được gọi là legacy system. 

Theo sách [1], để dễ dàng trong việc bảo trì và vận hành, khi thiết kế hệ thống, nên tuân thủ những nguyên tắc sau: 
- Operability (khả năng vận hành): đảm bảo operation team có thể dễ dàng vận hành hệ thống một cách trơn tru 
- Simplicity (Đơn giản): New engineer có thể dễ dàng hiểu được hệ thống. 
- Evolvability (khả năng tiến hoá): Có thể dễ dàng thay đổi/sửa feature theo những use case/requirements không đoán trước. Requirement luôn thay đổi, và hệ thống phải thích ứng tốt với việc này. Đặc tính này còn được gọi với những từ khác như extensibility, modifiability 

Chi tiết về từng nguyên tắc trên, có thể tham khảo ở chương 1 của sách [1] 


## 2. Availability (Tính sẵn sàng)
Tính sẵn sàng chỉ việc sẵn sàng xử lý request khi client yêu cầu. Nó được đo bằng tỉ lệ thời gian hệ thống hoạt động (up and running) trong một đơn vị thời gian (ngày, tháng hoặc năm)

Hight availability được đo bằng tỉ lệ phần trăm. 100% nghĩa là system không có downtime (zero downtime). Điều này rất khó có thể xảy ra. Thông thường, service high availability sẽ nằm trong khoảng 99% - 100%. Trong industry, 99.99% (or four nines) được xem như là đáng tin cậy, là tiêu chuẩn của các cloud vendor như Amazon, Google. 

Một số giá trị [5]:
- "Two nines" = 99% up = downtime 3.7 days/year
- "Three nines" = 99.99% up = downtime 8.8 hours/year
- "Four nines" = 99.99% up = downtime 53 mintues/year
- "Five nines" = 99.999% up = downtime 5.3 mintues/year 

Để giảm thời gian downtime, có thể cân nhắc loại bỏ các Single point of failure. [3]

Theo [3], một điểm cần lưu ý là cần phải tìm 1 điểm balance/trade-off giữa high availablity, lợi ích của high availablity và sự phức tạp của hệ thống. Đi từ 99% -> 99.99% thì dễ dàng hơn là từ 99.99% tới 99.9999% và xa hơn. Đến 1 điểm nào đó, hệ thống sẽ cực kỳ phức tạp. 

Câu hỏi cần đặt ra là nếu hệ thống sập trong 5 phút, 10 phút, 1 giờ, 1 ngày hoặc nhiều ngày, khách hàng sẽ phản ứng như thế nào? Lợi nhuận, danh tiếng của công ty bị ảnh hưởng như thế nào? .... Điều này hoàn toàn tuỳ thuộc vào từng business của công ty. 

Một số khái niệm [5]:
- Failure: toàn bộ system không hoạt động, không thể xử lý request.
- Fault: 1 bộ phận của hệ thống không hoạt động, vẫn có thể xử lý request. Ví dụ như node crash hoặc network fault 
- Fault tolerence (khả nặng chịu lỗi): hệ thống vẫn hoạt động dù có lỗi xảy ra 
- Single point of failure: 1 node/network bị lỗi, hệ thống sẽ chuyển sang trạng thái Failure

## 3. Reliability vs Availablity 
A high reliable system là 1 hệ thống không crash thường xuyên. 
A high availability system là 1 hệ thống có thể vận hành trong phần lớn thời gian 

Giả sử, 1 hệ thống là unreliable nếu nó crash thường xuyên, nhưng là high availability nếu nó nó khởi động/up lại nhanh. [4]


**Referenes:** 
1. [Book: Designing Data-Intensive Applications](https://www.amazon.com/Designing-Data-Intensive-Applications-Reliable-Maintainable/dp/1449373321)
2. [Các tính chất chính của một hệ thống phân tán](https://topdev.vn/blog/cac-tinh-chat-chinh-cua-mot-he-thong-phan-tan)
3. [Four nines and beyond: A guide to high availability infrastructure](https://www.atlassian.com/blog/statuspage/high-availability)
4. [Key Characteristics of Distributed Systems](https://medium.com/@sakalli.duran/key-characteristics-of-distributed-systems-0cc6e3ee08d3)
5. [Distributed Systems 2.4: Fault tolerance](https://www.youtube.com/watch?v=43TDfUNsM3E)