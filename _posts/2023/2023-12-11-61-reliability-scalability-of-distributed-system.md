---
layout: post
title:  "61. [System Design] 01. Reliability and scalability của hệ thống phân tán"
date:   2023-12-21 00:01:00 +0000
category: technical
---

Bài này sẽ bắt đầu chuỗi bài về System Design (thiết kế hệ thống), đặc biệt chú trọng vào Distributed system (hệ thống phân tán) cho những ứng dụng có lượng tải lớn. 

Khung sườn cho series này sẽ dựa vào cấu trúc của cuốn **Designing Data-Intensive Applications: The Big Ideas Behind Reliable, Scalable, and Maintainable Systems** của tác giả **Martin Kleppmann**. Ngoài ra cũng sẽ tìm kiếm thêm những nguồn tham khảo khác. 

Chuỗi bài viết này nhằm tạo động lực giúp tôi tìm hiểu sâu sắc và bài bản hơn về hệ thống phân tán cho ứng dụng lớn. Với kiến thức hạn hẹp của mình, chắc hẳn sẽ có những lầm lẫn, thiếu sót, rất mong nhận được sựu đóng góp của các bạn nếu có tình cờ đọc được. Mọi đóng góp xin để tạo Pull Request hoặc Issue tại địa chỉ: [https://github.com/NhatVu/nhatvu.github.io](https://github.com/NhatVu/nhatvu.github.io). Xin cám ơn mọi người. 

Bài viết đầu tiên sẽ tập trung nói về những yếu tố căn bản của một hệ thống phân tán, gồm Reliability (độ tin cậy) và scalability (khả năng mở rộng). Bài tiếp sẽ nói về 2 yếu tố còn lại là Availability (Tính sẵn sàng) và maintainability (khả năng bảo trì) [1][2]

## 1. Reliability (Độ tin cậy)
Reliability đảm bảo hệ thống vẫn hoạt động bình thường và chính xác ngay cả khi có lỗi (fault) xảy ra. Lỗi ở đây có thể đến từ phần cứng, phần mềm hoặc con người. Khi nhắc tới hệ thống, ta nói đến nhiều server cùng kết hợp lại để hình thành 1 hệ thống phân tán. Tức là mặc dù có lỗi, nhưng business logic vẩn đảm bảo đúng.

Hệ thống có thể hoạt động bình thường và chính xác khi có lỗi xảy ra, còn được gọi là fault-tolerance system. Lưu ý, ở đây hệ thống chỉ chịu được một số lỗi nhất định, chứ nó không bao quát toàn bộ các loại lỗi.

Với một số công ty, để kiểm tra khả năng chịu lỗi của hệ thống, thường sẽ có những bài test định kỳ, khi họ kill 1 hoặc nhiều process/applicaiton mà không báo trước. Nhũng lỗi quan trọng nhất là những lỗi không được handle. Với bài test này, mức độ tự tin về khả năng chịu lỗi của hệ thống sẽ được tăng lên. Lỗi xảy đến lúc ta không ngờ, và không có chuẩn bị nhất.

Lỗi luôn luôn có thể xảy ra và xuất phát từ:
- Hardware (phần cứng): hard disk đến tuổi đời, ... 
- Software: lỗi hệ điều hành, lỗi framework, library, network chậm dẫn tới việc gọi external service bị timeout, hay lỗi cascading failures (khi một phần nhỏ bị lỗi/chậm, dẫn đến những service gọi nó bị lỗi/chậm, dẫn đến những service ở tầng cao hơn bị lỗi/chậm, dẫn đến sập toàn bộ hệ thống)
- Con người: nhập sai input, thực hiện sai quy trình, ... 

## 2. Scalability (Khả năng mở rộng)
Scalability chỉ khả năng mở rộng của hệ thống nhằm đáp ứng với lượng tải (load) tăng. 

Tuỳ thuộc vào từng hệ thống, lượng tải (load) có thể định nghĩa bằng request per second đối với web application, tỉ lệ read:write đối với database, hit rate với cache, ... 

Khi nói tới tải, cũng cần nói đến normal rate và peak rate. Peaking rate xảy ra khi có 1 biến cố/sự kiện nào đó khiến lượng tải tăng đột biến. Đó có thể là tin nóng, idol live stream, các này lễ tết, ... 

Một số khái niệm: 
- Response time: (thời gian phản hồi): là tổng thời gian khi client gửi request và nhận được response. Response time = network transmit time + queueing delays + service processing time. Thường dùng để mô tả performance of application 
- Latency (độ trễ/sự chờ): thời gian request chờ để được xử lý. Latency = network transmit time + queueing delays. Thông thường dùng để mô tả sự tắc nghẽn mạng (network cogestion) hoặc khoảng cách vật lý xa giữa client và server [3] 
- Throughput (thông lượng): với networking, thông lượng chỉ lượng dữ liệu thực sự đi qua mạng trong một đơn vị thời gian. Với ứng dụng web, có thể xem thông lượng là lượng request thực mà ứng dụng có thể xử lý trong một đơn vị thời gian
- Băng thông (băng thông): với networking, băng thông là lượng dữ liệu tối đa có thể đi qua mạng trong 1 đơn vị thời gian. Với ứng dụng web, có thể xem bandwidth là lượng request tối đa mà ứng dụng có thể xử lý trong một đơn vị thời gian. 

Đế thích ứng với lượng tải tăng, thường có 2 cách scale phổ biến: vertical scaling và horizontal scaling.

- Vertical scaling (scale the chiều dọc): Thêm CPU, RAM, ổ cứng, làm cho máy mạnh hơn để có thể đáp ứng nhiều request hơn. Tuy nhiên, luôn có giới hạn có RAM, CPU và ổ cứng 
- Horizontal scaling (scale theo chiều ngang): phân phối/chia đều lượng tải tới nhiều máy nhỏ khác nhau. 

Tuỳ vào từng trường hợp mà việc vertical scaling hoặc horizontal scaling sẽ được chọn. Không có 1 giải pháp nào phù hợp cho tất cả các trường hợp. Điều này phải thật sự chú ý vì thời gian viết bài này, thời của microservice, horizontal scaling gần như trở thành giải pháp mặc định. 

Lấy một ví dụ trong sách [1], hệ thống được thiết kế để xử lý 100K request/s, mỗi request 1kB sẽ rất khác với hệ thống xử lý 3 request/phút, mỗi request 2GB, mặc dù cả 2 hệ thống đều có chung giá trị throughput. 

Một ví dụ khác là việc Amazon Prime đã chuyển từ kiến trúc microservice sang monolothic cho ứng dụng xử lý video của họ, và kết quả là giảm tới 90% chi phí [4]. Vấn đề chủ yếu ở đây chính là việc truyền tải dữ liệu trung gian (intermediate data) trong quá trình xử lý data giữa các microservice tốn nhiều băng thông và thời gian. Thay vào đó, xử lý trên video trên cùng 1 máy sẽ không tốn chi phí này.


**Referene:** 
1. [Book: Designing Data-Intensive Applications](https://www.amazon.com/Designing-Data-Intensive-Applications-Reliable-Maintainable/dp/1449373321)
2. [Các tính chất chính của một hệ thống phân tán](https://topdev.vn/blog/cac-tinh-chat-chinh-cua-mot-he-thong-phan-tan)
3. [What is the difference between latency and response time?](https://stackoverflow.com/questions/58082389/what-is-the-difference-between-latency-and-response-time)
4. [Scaling up the Prime Video audio/video monitoring service and reducing costs by 90%](https://www.primevideotech.com/video-streaming/scaling-up-the-prime-video-audio-video-monitoring-service-and-reducing-costs-by-90)