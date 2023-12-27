---
layout: post
title:  "66. [System Design] 06. Replication: phương pháp  Master - Slave. Phần 2"
date:   2023-12-27 00:02:00 +0000
category: technical
---

## Implementation of Replication logs 
Có nhiều cách để thực hiện replicaiton log. Ta sẽ xem xét một số cách thông dụng nhất: 
### Statement-based replication 
Trường hợp đơn giản nhất, leader/master sẽ log tất cả write request statement và gửi nó tới follower/slave. Với relational database, statement cính là INSERT, UPDATE, DELETE query.

Mặc dù nghe hợp lý nhưng cách này gặp những khó khăn sau khiến việc replication không thực hiện được
- Statement chứa nondeterministic function (function trả về kết quả khác nhau sau mỗi lần gọi) như now(), rand(), ... Do đó, giá trị trong mỗi replica sẽ khác nhau
- Statement use dụng auto-increasing column hoặc nó dựa vào 1 dữ liệu khác trong database (ví dụ UPDATE .. WHERE \<condition\>), chúng phải được thực thi đúng thứ tự. Điều này giới hạn khả năng concurently executing transaction 
- Side effects (ví dụ triggers, stored procedure, user-defined function,...) 

Những hạn chế trên đều có thể khắc phục, tuy nhiên, vì có quá nhiều edge cases, những phương pháp replication khác được ưa chuộng hơn.

### Write-ahead log (WAL) shipping
Log chỉ có 1 mode là append-log (thêm vào cuối file). Content là sequence of bytes chứa thông tin cần ghi vào database. Leader/Master sẽ gửi log này tới slave. 

Điểm bất lợi nhất của phương pháp này là log lưu ở dạng low-level data (byte, disk block, ..) và nó bị closely coupled (gắn chặt với) với storage engine. Nếu database thay đổi storage format (vd như khi nâng cấp version), thường sẽ không thể xử lý log 1 cách đúng đắn. 

Điều này quan trọng trong việc vận hành. Nếu replication protocol cho phép follower dùng phiên bản phần mềm mới hơn leader, ta có thể thực hiện upgrade với zero-downtime bằng cách upgrade follower trước và thực hiện falilover cho leader node. Nếu replication protocol không cho phép version missmatch, upgrade đòi hỏi downtime.

Hơn nữa, logical log có thể parse dễ dàng bởi các ứng dụng bên thứ 3. Có thể dùng cho data warehouse, custom index, ...

### Logical (row-based) log replication 
Logical log là 1 cách để tránh bị "closely coupled" vào storage engine, dễ dàng hơn trong việc giữ đặc tính backward compatible, giúp cho phép leader và follower chạy với nhiều phiên bản database khác nhau 

Logical log trong relational database thường sẽ gồm 1 chuỗi records, mô tả các thao tác write của database, gồm:
- Với insert row, log gồm tất cả giá trị mới của column 
- Với delete row, log gồm thông tin đủ để thực hiện delete. Nếu có primary key, thì dùng primary key. Nếu không có, thì log toàn bộ giá trị của row 
- Với update row, log chứa thông tin đủ để uniquely identify (định dạng duy nhất) row, và những giá trị mới của các column cần update. Theo tôi hiểu, nếu câu lệnh UPDATE yêu cầu update 100 rows, thì row-based log sẽ log giá trị của 100 row. Nhưng statement log sẽ chỉ log mỗi câu query mà thôi. 

## Problems with Replicaiton Lag 
Replicaiton lag: sự chậm trễ (delay) giữa data ghi trên leader và sự xuất hiện của data đó trên follower. 

Thường thì replication lag diễn ra trong nhỏ hơn 1s, hoặc 1 phút nếu có vấn đề về mạng. Tuy nhiên, khi giá trị này quá lớn, sẽ gây ra những vấn đề cho ứng dụng. Ở đây ta sẽ liệt kê 3 ví dụ cho vấn đề này, cũng như một số phương án xử lý. 

### Reading your own writes 
Rất nhiều application cho phép user submit data và view data sau khi submit. Data có thể là comment, customer detail, ... Khi new đata được submitted, nó buộc phải gửi tới leader, nhưng khi user read/view data, nó có thể đọc ở follower. Với asynchronous replication, new data chưa kịp replication tới toàn bộ follower node, nhưng user lại đọc ở node chưa được replicate --> user nghĩ rằng dữ liệu họ submit đã bị mất hoặc submit không thành công ---> user không vui 

Với trường hợp này, chúng ta cần read-after-write consistency, còn được gọi là read-your-own-writes consistency. Nó bảo đảm rằng khi reload page, user sẽ thấy những gì họ đã submit. Tuy nhiên, với những user khác, không đảm bảo sẽ đọc được data mới nhất. 

Vậy làm sao thể implement read-after-write consistency với mô hình leader-follower replication? Có một số kỹ thuật như sau:
- Nếu dữ liệu mà user có thể thay đổi, đọc ở leader. Những dữ liệu khác, đọc ở follower. Điều này yêu cầu phải có cách để biết dữ liệu nào thay đổi, dữ liệu nào không. Ví dụ với user profile, luôn luôn đọc leader nếu user A request user profile của user A. Nếu user A request user profile của những user khác, đọc follower. Sẽ cần biết chính xác phần data nào đọc ở leader, phần data nào đọc ở follower. Theo tôi cách này khá phức tạp. 
- Nếu hầu hết data đều có thể edit được thì phương pháp trên không hiệu quả, bởi mọi request sẽ read ở leader. Trong trường hợp này, những tiêu chuẩn khác được sử dụng để kiểm tra xem nên đọc ở đâu. Ví dụ, bạn có thể theo dõi last time update, đọc ở leader nếu vừa được update trong 1 phút. Cần chú ý tới trường hợp user sử dụng nhiều devices. Nếu user nhập data trong 1 device và xem ở nhiều device khác, user cũng nên thấy được thông tin mới nhất. 
- Ngoài ra còn có cách liên quan tới logical time, tôi vẫn chưa hiểu.

### Monotoic Reads
Monotoic read là 1 dạng consistency, không nghiêm ngặt bằng strong consistency nhưng nghiêm ngặt hơn eventual consistency (eventual consistency < monotoic read < storng consistency). Nó đảm nếu 1 user gọi nhiều read operation, sẽ không thấy giá trị cũ sau khi đã thấy được giá trị mới. 

Ở đây, sách dùng thuật ngữ "time doesn't go backward". Nếu vẽ trục thời gian sẽ dễ hình dung hơn. Giả dụ giá trị cũ ở time t1, giá trị mới ở time t2, và t1 < t2. Thì sau khi thấy giá trị ở t2 một lần, ta không thể thấy giá trị ở t1 nữa. Monotoic read sẽ đảm bảo điều này. 

Một cách để đạt được điều này là đảm bảo rằng 1 user luôn đọc cùng 1 replica. Có thể chọn replica bằng cách hash userId. Nếu replica failed, tự động reroute tới replica khác. 

Nếu một vài userId là hot key, được truy suất nhiều và lại nằm trên cùng 1 replica, sẽ gây quá tải cho server. Nên giải quyết ntn? Cần tìm hiểu sâu hơn về Monotoic read này.

### Consistent Prefix Reads
Consistent prefix reads đảm bảo rằng nếu 1 chuỗi write request xảy ra theo 1 trình tự cho trước, bất kỳ ai đọc những những thông tin này cũng sẽ thấy chúng xuất hiện theo đúng trình tự. 

Ví dụ: \
request 1. User A: write A. Ví dụ longer lag, maybe 10s \
request 2. User B: write B. Write ngay lập tức.

Giả sử có 1 bên thứ 3 sẽ nhận thông báo nếu write thành công. Vì replicaiton lag, bên thứ 3 sẽ nhận message từ request 2 trước, và sau đó là request 1. 

Đây là vấn đề thường xuyên xuất hiện trong sharded database, sẽ nói chi tiêt hơn ở chapter 6. 

## Solution for Replicaiton lag 
Khi làm việc với hệ thống có đặc tính eventually consistent system, cần phải nghĩ cẩn thận vệ cách hành xử của ứng dụng nếu replication lag tăng tới vài phút hoặc vài giờ. Nếu câu trả lời là "no problem", ta không cần quan tâm vấn đề này nữa. Nhưng nếu câu trả lời là gây ra trải nghiệm người dùng tệ, ta cần phải design 1 system với stronger consistency, ví dụ như read-after-write. Đặc biệt, giả dụ rằng replication là sync, trong khi thực tế là async, chính là nguyên nhân gây nên nhiều lỗi 

Như đã thảo luận ở trên, ta có thể cung cấp gỉai pháp đảm bảo stronger consistency, nhưng lại thực hiện ở application level hơn là ở database level - giả dụ như read leader thay vì follow trong vài trường hợp. Điều này thì phức tạp và có thể dễ dẫn đến lỗi sai.


**Remain question**
1. Với row-based replication, xử lý concurrent write request như thế nào? Tìm hiểu sâu hơn với Postgres 
2. Đọc sâu hơn về read-after-write consistency 
3. Đọc sâu hơn về monotoic read.

**Referenes:** 
1. [Book: Designing Data-Intensive Applications](https://www.amazon.com/Designing-Data-Intensive-Applications-Reliable-Maintainable/dp/1449373321)