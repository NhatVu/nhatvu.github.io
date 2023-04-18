---
layout: post
title:  "39. JDBC connection leak detection trong Spring Boot"
date:   2023-04-18 00:01:00 +0000
category: technical
---
Mặc định, Spring Boot phiên bản gần đây sử dụng Hikari pool. Vậy nên, bài này sẽ nói chủ yếu về các config cho Hikari 

Thông thường, chi phí tạo connection tới database lớn, nên kỹ thuật pooling giúp tối ưu hóa việc này bằng cách tạo trước 1 lượng connection. Sau đó đặt trong pool. Khi nào sử dụng thì lấy ra, sử dụng xong thì trả lại vào pool. Nếu không thể lấy ra thêm connection (sau khi đã kiểm tra các config khác), exception Unable to get jdbc ... sẽ được tạo ra. 

Các lỗi thế này thông thường xuất hiện khi lượng tải cao, thế nên trong quá trình dev, ít xuất hiện. Và vì vậy, rất khó để phát hiện lỗi này. Đôi lúc, ta chỉ thấy request xử lý hơi lâu 1 tí, mà không biết, hệ thống đã chạm ngưỡng về lượng tải. 

Để phát hiện việc này, Hikari cung cấp config **spring.datasource.hikari.leak-detection-threshold**. Dựa vào log, ta sẽ phát hiện được ngưỡng tải hiện tại của hệ thống. 

Ngoài lượng tải lớn, việc không còn connection trong pool có thể do việc ứng dụng giữ db connection lâu. Nó có thể do câu query chưa tối ưu nên chạy chậm, mở connection mà quên đóng hoặc mở transaction nhưng không commit transaction. 

Ta có thể kiểm tra db, xem có bao nhiêu connection đang được sử dụng, được sử dụng bởi ai, trạng thái của mỗi connection là gì, đã tồn tại được trong bao lâu, câu query cuối cùng là gì, ... Việc này sẽ giúp ta thu nhỏ mục tiêu điều tra.

Việc phát hiện và xử lý connection leak là khó. Nếu dùng Spring JPA thì tự thân framework đã hỗ trợ rất nhiều về việc tránh connection leak. Thế nên, một khi xảy ra, việc phát hiện và xử lý lại khó gấp bội.
