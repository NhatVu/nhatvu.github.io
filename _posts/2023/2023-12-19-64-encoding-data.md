---
layout: post
title:  "64. [System Design] 04. Encoding data"
date:   2023-12-19 00:02:00 +0000
category: technical
---

Trong quá trình phát triển phần mềm, đôi khi schema của dữ liệu sẽ bị thay đổi theo thời gian. Các thao tác như thêm field, xoá field, thay đổi kiểu dữ liệu thường sẽ diễn ra và chúng đòi hỏi hệ thống phải thích ứng với những thay đổi này. 

**Backward compatibility** (tương thích ngược) nghĩa là code mới (newer code) có thể đọc dữ liệu được tạo bởi code cũ (old code). **Forward compatibility** (tương thích tiến tới) nghĩa là code cũ có thể đọc được đata được tạo bởi code mới [1] 

Backward compatibility không khó để đạt được, vì developer sẽ biết schema của code cũ. Tuy nhiên, với forward compatibility, code cũ phải đảm bảo ignore những field được thêm vào từ code mới, cũng như thích ứng với những field bị xoá, thay đổi field type. Ta sẽ nói chi tiết hơn ở phần sau 

Với application/program, data thường được biểu diễn bởi ít nhất 2 dạng sau: 
- Khi nằm trên memory, data sẽ ở dạng object, list, array, ... Những dạng cấu trúc dữ liệu giúp tối ưu việc truy suất và sử dụng bởi CPU (thông thường thông qua pointer)
- Khi data cần truyền đi giữa các service trên network hoặc lưu trữ lại vào ổ cứng, dữ liệu cần được đưa về dạng về dạng self-contained sequence of byte, không cần dùng pointer. 

Chúng ta cần phải chuyển đổi qua lại giữa 2 cách biểu diễn này. Việc chuyển từ in-memory representation sang binary sequence được gọi là encoding (hoặc serialization hoặc marshalling). Ngược lại, chuyển từ binary sequence sang in-memory representation được gọi là decoding (hoặc arsing, deserialization, unmarshalling) [1] 

Có nhiều phương pháp cho việc encoding/decoding. Chúng ta sẽ tìm hiểu những dạng phổ biến nhất. 

### 1. Language-Specific Formats 
Hầu hết các ngôn ngữ lập trình đều hỗ trợ việc encode in-memory object thành byte sequence. Ví với Java là java.io.Serializable, Python là pickle, ... Ngoài ra còn có 3rd thư viện như Kryo cho Java. Dùng các thư viện này thì thuận tiện, và cần ít công sức trong việc decode. Tuy nhiên, có những hạn chế như: 
- Bị "buộc chặt" với ngôn ngữ lập trình cụ thể. Ví dụ nếu encode bằng Java thì phải decode bằng Java. Tuy nhiên, một phần của hệ thống có thể dùng một ngôn ngữ lập trình khác. Ví dụ backend dùng Java, front end dùng Javascript
- Gây ra vấn đề về bảo mật. 
- Hiệu suất thấp, dùng nhiều CPU, đặc biệt là với Java 

### 2. JSON, XML, and Binary Variants
JSON, XML và các dạng encode binary không phụ thuộc vào ngôn ngữ lập trình và được sử dụng rộng rãi trong việc truyền thông tin giữa các hệ thống. XML thường bị phê bình vì phức tạp và dư thừa 

XML, JSON, CSV:
- Human readable 
- Ambiguity (sự mơ hồ/nhập nhằng) về kiểu dữ liệu số. XML và CSV không thể phân biệt kiểu String và Number. JSON có thể phân biệt giữa String và number, tuy nhiên, không thể phân biệt được integer, long hay float. Một kinh nghiệm xương máu là đối với JSON, nếu là kiểu Long thì nên dùng String, vì bộ parse số của Javascript sẽ bị lỗi nếu số Long lớn, dẫn đến Server trả về đúng, nhưng Javasript trên browser parse bị sai 
- XML và JSON hỗ trợ ký tự Unicode, nhưng không hỗ trợ kiểu binary string (ví dụ muốn lưu ảnh, SSH key, ...). Để vượt qua hạn chế này, có thể encode binary string dưới dạng Base64. 

Binary Encode: 
- Binary encode sẽ tiết kiệm dung lượng hơn là JSON hoặc XML. Tuy nhiên, ta không thể đọc trực tiếp file này, mà cần phải dùng những công cụ chuyên biệt. Có nhiều cách để thực hiện binary encode như thrift, gRPC, ...

### 3. Thrift and Protocol Buffers 
Thrift và Protocal Buffers (sau này còn có gRPC) là thư viện hỗ trợ binary encoding, và cùng dựa trên những nguyên lý chung. Thirft và gRPC sẽ định nghĩa đối tượng cần truyền tải, các hàm hỗ trợ, ... (Xem kỹ hơn trong sách và tutorial). 
Ví dụ: 
```thrift 
struct Person {
1: required string userName,
2: optional i64 favoriteNumber, 
3: optional list<string> interests
}
```
Khi dùng Thrift hay gRPC, chúng hỗ trợ các schema và nhiều kiểu dữ liệu đa dạng hơn. Sẽ tránh bị nhập nhằng khi encode/decode. 

Những con số 1, 2 bên cạnh userName, favoriteNumber được gọi là *field tags*. Field tag này phải được giữ nguyên và không được thay đổi khi sử dụng, vì thư viện sẽ dựa vào số này để encode và decode giá trị. Nếu số 1 đang là userName, ta sửa nó thành số 2, thì khi decoding, nó sẽ lấy giá trị số 2 ở data cũ (chính là favoriteNumber) dùng cho số 1 (chính là userName) ==> gây lỗi. Tuy nhiên ta có thể tự do đổi tên, vì thư viện sử dụng field tag 

Thirft và gRPC hỗ trợ tốt với việc backward và forward compatibility. 
- Backward compatibility (code mới đọc data cũ): phải đảm bảo rằng những field mới thêm, không được sử dụng required, vì data cũ không có những field này. Nếu dùng required, sẽ gây lỗi. Không được xoá những field có sử dụng required.
- Forward compatibility (code cũ đọc data mới): Data mới sẽ có thể nhiều hoặc ít field hơn code cũ, những field nào không nhận ra sẽ tự động bị ignore. Nhất thiết vẫn phải đảm bảo có đủ những field được đánh dấu là required. 
- Về việc thay đổi databtype, có thể xảy ra nhưng phải cẩn thận. Ví dụ field từ 32bit lên 64bit. Sẽ có khả năng bị sai số khi code cũ (32 bit) đọc dữ liệu được tạo bởi code mới (64 bit). 

### 4. Avro 
Avro là 1 dạng binary encoding khác với Thrift và gRPC, được Hadoop sử dụng. Avro không sử dụng field tag như Thrift hoặc gRPC. Avro hỗ trợ dynamic generated schema. Cần phải nói thêm, Thrift không thích hợp trong trường hợp của Hadoop. 

Một số context sử dụng Avro:
- File lớn với rất nhiều record. Trong trường hợp của Hadoop, 1 file có thể gồm hàng triệu dòng với cùng Schema. 
- Gửi nhiều records bằng network connection: Vì Avro hỗ trợ việc nén file, nên dung lượng gửi sẽ giảm đi nhiều 

Ngoài ra, còn có kiểu dữ liệu né theo cột như parquet. Mình cũng chưa hiểu rõ lắm về 2 dạng này.

### 5. Kết luận
Mỗi một kiểu encoding/decoding sẽ được sử dụng tuỳ theo ngữ cảnh. Bài viết tổng hợp những kiểu thông dụng, cho những ngữ cảnh thông dụng. 
- Nếu muốn lưu 1 Java Object (list, map, custom object,...) thì sử dụng buil-in Java.io.Serializable hoặc Kryo 
- Nếu giao tiếp giữa backend và frontend, thì có thể sử dụng JSON 
- Nếu giao tiếp giữa các internal service (còn được gọi là middleware) thì có thể dùng Thrift hoặc gRPC 
- Nếu muốn lưu file, có thể sử dụng JSON, CSV nếu không yêu cầu khắt khe về kiểu dữ liệu, human-readable, và không lớn. Với những file lớn, có hàng triệu record, khắt khe về schema, kiểu dữ liệu và dung lượng, có thể cần nhắc lưu dưới dạng Avro hoặc Parquet 


**Referenes:** 
1. [Book: Designing Data-Intensive Applications](https://www.amazon.com/Designing-Data-Intensive-Applications-Reliable-Maintainable/dp/1449373321)