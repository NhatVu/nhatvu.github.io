---
layout: post
title:  "71. [DevOps] Kiểm tra code coverage, code quality với Jacoco plugin và SonarQube"
date:   2024-05-03 00:02:00 +0000
category: technical
---

Testing là một khâu quan trọng trong quá trình phát triển phần mềm. Việc viết unit test thường được xem như là chuẩn tối thiểu nhằm đảm bảo các module hoạt động đúng đắn. Tuy nhiên, ta cũng cần có công cụ để kiểm tra xem bộ test đã viết đã cover/chưa cover những trường hợp nào. Ngoài ra, ta cũng cần có công cụ nhằm kiểm tra xem chất lượng code có đạt một chuẩn đề sẵn hay không. Code hạn chế mắc phải những lỗi thường gặp như NullPointerException, không close FileInputStream, if/else quá nhiều, không dùng constant cho String lặp lại nhiều lần, ... Thêm vào đó, ta cũng nên có tool để kiểm tra những thư viện nào đang sử dụng gặp phải lỗi về bảo mật. Tuy nhiên, theo tôi biết thì SonarQube không hỗ trợ tốt trong vấn đề security scanning.

Bài này giới thiệu 2 công cụ mã nguồn mở thường được sử dụng. Mỗi loại có ưu, nhược điểm riêng.

### Code coverage 
Trước hết, code coverage là một khái niệm trong testing, chỉ độ phủ unit test trên những đoạn code đã dược viết. 

Code coverage có thể xét trên nhiều khía cạnh như method coverage, line coverage, đánh đó hơn là branch coverage (đảm bảo phải cover toàn bộ các condition trong lệnh if/else). 

Theo Atlassian và Google, code coverage đạt trên 80% là đủ tốt. Không nhất thiết phải đạt đến mức 95%-99% vì tốt nhiều thời gian nhưng không mang lại quá nhiều lợi ích. [1][2] 

### Jacoco plugin 
Jacoco là 1 plugin miễn phí, được tích hợp với Maven, với chức năng chính là phân tích code coverage cho project. Jacoco hỗ trợ cả line coverage và method coverage. Điểm mạnh của Jacoco là nó có thể chạy độc lập (standalone mode), không cần tích hợp hoặc gọi external service.

Hình dưới là minh hoạ Jacoco report

![Jacoco plugin](/assets/images/2024/71-jacoco.png)

Khi click chi tiết vào từng java file, nó cũng sẽ chỉ chi tiết phần nào chưa được cover hoặc bị sót if condition. 

Tuy nhiên, điểm yếu lớn nhất là nó không thể kiểm tra được code quality. 

Theo tôi thì Jacoco là 1 plugin tốt để tích hợp

### SonarQube
SonarQube là một open-source application cho phép ta kiểm tra code coverage và code quality. SonarQube tồn tại dưới dạng server, không tồn tại dưới dạng standalone service như Jacoco. Vậy nên quá trình tích hợp sẽ phức tạp hơn. 

Tuy nhiên, SonarQube cũng cung cấp 1 plugin tên SonarLint trên Intellij, cho phép thực hiện việc kiểm tra code quality mà không cần kết nối tới SonarQube server. Điểm hạn chế là chức năng của nó không được đầy đủ và toàn diện như khi dùng với server. Điểm lợi là dùng được ở local, đẩy nhanh quá trình sửa lỗi.

![alt text](/assets/images/2024/71-sonar.png)

### Tích hợp Unit test, Sonar Analysis vào Jenkin pipeline 
Để mọi thứ được diễn ra 1 cách tự động, theo tôi, thuận tiện nhất là nên tích hợp Unit test và sonar analysis trực tiếp vào Jenkin pipeline. Nếu một trong 2 bọn chúng không thành công, thì Jenkin pipeline fail và không được phép deploy/merged code. 

Nếu có thể, thì thiết lập cơ chế, sao cho mỗi commit lên git, đều trigger Jekin pipeline nhằm chạy unit test và sonar. Đây là ý tưởng ship left trong testing, test và fix lỗi sớm nhất khi có thể. Điều này sẽ làm quá trình merged code vào develop dễ dàng và tự tin hơn. Chúng ra không nên đợi tới khi code chuẩn bị được merged vào develop, lúc đó mới đi sửa lỗi thì sẽ rất áp lực. Nhất là với áp lực thời gian, ta thường exclude vài file source code nhằm vượt qua được sonar threshold. Đây không phải là 1 good practice. 

Thêm vào đó, với mỗi phase như PRs review, deployment, create artifact, đều cần đẩm bảo unit test và sonar được chạy.

Với Jacoco và SonarQube, những công cụ open-source, sẽ giúp nâng tầm source code của chúng ta lên một tầng cao mới.

**Referenes:** 
1. [What is code coverage?](https://www.atlassian.com/continuous-delivery/software-testing/code-coverage)
2. [Code Coverage Best Practices](https://testing.googleblog.com/2020/08/code-coverage-best-practices.html)