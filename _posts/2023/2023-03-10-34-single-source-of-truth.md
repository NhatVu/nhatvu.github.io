---
layout: post
title:  "34. Single source of truth principle"
date:   2023-03-10 00:01:00 +0000
category: technical
---

Khi phát triển phần mềm, sẽ đến giai đoạn cần phải đọc dữ liệu từ nhiều nguồn khác nhau (REST API, database, cache) hoặc gặp tình huống phải ghi dữ liệu đồng thời vào 2 database khác nhau. Nếu thiết kế API không cẩn thận, sẽ dẫn đến trường hợp ghi vào database này, nhưng lại không ghi vào database còn lại. Từ đó, gây ra những lỗi rất khó để phát hiện.

Đặt tình huống, giả sử table A chứa dữ liệu dùng cho REST API và table B được dùng cho Framework. Theo logic, 1 field được cập nhật trong table A, phải được cập nhật vào framework table và ngược lại. Giả sử class A chứa hàm cần thiết cho table A, class B chứa hàm cần thiết cho table B. Ta có thể cập nhật dữ liệu như sau

```
// entity là dữ liệu cần cập nhật
A.save(entity)
B.save(entity)
```

Mỗi khi cần cập nhật dữ liệu, đều phải gọi riêng rẽ 2 hàm như trên. Điều này dễ dẫn đến lỗi sai nếu mộ người mới vào team, không hiểu logic này, và gọi thiếu hàm. Không chỉ người mới, người cũ cũng có thể quên mất logic này.

Để tránh tình trạng trên, ta có thể tạo 1 class C, chứa tất cả logic liên quan đến class A và B. Client chỉ làm việc với C, mà không làm việc với A và B. class C cũng có hàm save(), và trong này gọi A.save() và B.save()

```java
public class C {
	public void save(Object entity){
		A.save(entity);
		B.save(entity);
	}
}
```

Bằng cách làm việc thông qua class C, ta có thể tránh tình trạng bất đồng bộ dữ liệu giữa A và B.

Single source of truth tức là việc ghi dữ liệu chỉ xảy ra tại 1 nơi (primary location, tức là C), sau đó dữ liệu này được lan truyền ra các node trong hệ thống (Ở đây là A và B). 

Việc inconsistent dữ liệu gây ra những lỗi rất kỳ quặc, phi logic nếu chỉ nhìn vào code và cực kỳ khó phát hiện. Single source of truth là 1 giải pháp cho việc này.


