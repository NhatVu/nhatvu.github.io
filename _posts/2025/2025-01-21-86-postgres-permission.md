---
layout: post
title:  "86.  PostgreSQL: Permission overview"
date:   2025-01-19 00:02:00 +0000
category: technical
---
- [1. Database roles](#1-database-roles)
  - [1.1. Role attribute](#11-role-attribute)
  - [1.2. Role membership](#12-role-membership)
  - [1.3. Drop role](#13-drop-role)
  - [1.4. Change role inside session](#14-change-role-inside-session)
- [2. Privileges](#2-privileges)
- [3. Use case](#3-use-case)
- [4. References](#4-references)

Tuần rồi tôi gặp 1 số vấn đề liên quan đến Postgres permission. Tuy đã giải quyết xong nhưng thực sự, tôi vẫn chưa hiểu rõ lắm. Bài này viết ra với mục đích tổng hợp kiến thức liên quan tới Postgres permission ở dạng cơ bản, với hy vọng, trong quá trình đọc thêm tài liệu để viết bài, tôi có thể lý giải rõ hơn khó khăn đã gặp. 

### 1. Database roles 
Thông thường, khi tiếp cận với database, ta thường không quá mức để ý đến vấn đề phân quyền (permission). Khi develop ứng dụng, ta thường sử dụng quyền cao nhất để thực hiện tác thao tác xoá, sửa cho thuận tiện. Tuy nhiên, khi deploy lên Production, ta cần phân quyền để bảo mật và giới hạn rủi ro. 

PostgreSQL quản lý quyền bằng khái niệm *role*.[3]. Một role có thể là database user, group of database user, role, ... tuỳ vào cách mà ta thiết lập. Điều khác biệt duy nhất giữa role và user là role không có attribute LOGIN.

Role có thể own database object 
(ví dụ table, function) và cấp đặc quyền 
(privileges) trên những object này cho các role khác. 

Permission trong Postgres chia làm nhiều tầng khác nhau: database instance, schema, table, function,... Đôi khi, có quyền ở table, nhưng thiếu quyền ở schema có thể phát sinh lỗi khi thực hiện câu lệnh.

PostgreSQL role tồn tại ở instance level. Tức là tất cả database đều thấy được role đó. [4] 

```sql
-- Tạo role 
-- name = tên role cần tạo
CREATE ROLE <name>;

-- Xoá role 
DROP ROLE <name>;

-- Để xác định những role đang tồn tại
-- có thể bỏ pg_catalog schema. 
-- Câu sql này tương đương với psql: \du 
SELECT rolname FROM pg_catalog.pg_roles;

```

Thông thường, khi khởi tạo postgres, ta sẽ có role postgres với quyền superuser. Đây là quyền cao nhất trong Postgres, tương đương với root trong Linux. 

#### 1.1. Role attribute
Khi tạo role, ta có thể kèm theo các attribute với các ý nghĩa khác nhau. Ví dụ
- LOGIN: cho phép role này login hay không
- SUPPERUSER: role này có phải là super user hay không. Không nên lạm dụng super user role 
- CREATEDB: role này có được phép tạo database hay không 
- CREATEROLE: role này có được phép tạo role mới hay không 
- PASSWORD: set password cho role

Dựa vào attribute LOGIN, ta có thể linh hoạt trong việc tạm thời deactive 1 user bất kỳ. Giả sử, ta có user flyway_db_user, chuyên dùng để tạo bảng, thay đổi bảng thông qua việc sử dụng Flyway. User này chỉ nên được mở, cho phép LOGIN khi có yêu cầu. Việc này giúp quản lý các schema, table trong database. Tránh việc tạo quá nhiều bảng, bảng không đồng nhất giữa các môi trường DEV, STAGE, PROD. 

#### 1.2. Role membership 
Để thuận lợi cho việc quản lý, role thường được gom nhóm thành 1 group. Các privilege sẽ được grant, revoke dựa theo group permission. Trong PostgreSQL, điều này thực hiện bằng cách tạo 1 role đại điện cho group, sau đó grant membership của group đó cho individual user 

```sql
-- syntax 
GRANT group_role TO role1, ... ;
REVOKE group_role FROM role1, ... ;
```

Ta sẽ lấy ví dụ trong PostgreSQL document, để minh hoạ về roles, và inherit 

```sql
-- role joe được xem như user, và có thể login  
CREATE ROLE joe LOGIN;

-- tạo role 
CREATE ROLE admin;
CREATE ROLE wheel;
CREATE ROLE island;

-- grant members
GRANT admin TO joe WITH INHERIT TRUE;
GRANT wheel TO admin WITH INHERIT FALSE;
GRANT island TO joe WITH INHERIT TRUE, SET FALSE;

/*
Relationship 
wheel -> admin -> joe 
island -> joe 
*/
```

Để kiểm tra xem user có thuộc vào 1 group nào không, ta có thể dùng DBEaver để kiểm tra, đây cũng là cách dễ nhất nếu ta không thường xuyên tiếp xúc với command line. Tuy nhiên, PostgreSQL có hỗ trợ hàm để kiểm tra membership. Điểm lợi của hàm này là ta có thể kiểm tra direct và indirect membership 
```sql 
-- pg_has_role ( [ user name or oid, ] role text or oid, privilege text ) → boolean
-- Does user have privilege for role? 

-- nhắc lại, relationship là wheel -> admin -> joe 
select pg_has_role('joe', 'admin', 'MEMBER'); -- true, direct relationship 
select pg_has_role('joe', 'wheel', 'MEMBER'); -- true, indirect relationship 
select pg_has_role('admin', 'wheel', 'MEMBER'); -- true 


select pg_has_role('wheel', 'admin', 'MEMBER'); -- false 
```

#### 1.3. Drop role 
Mỗi role có thể own nhiều database object hoặc được grant 1 số privilege. Trước khi sử dụng lệnh `DROP ROLE`, ta cần chuyển owner của database object tới cho user khác. 

Có thể sử dụng lệnh `ALTER` để chuyển đổi owner 
```sql 
ALTER TABLE bobs_table OWNER TO alice;
```

Tuy nhiên, trong thực tế, việc này rất khó. Tôi vẫn chưa tìm ra cách nào để xác định các database object mà 1 role/user sở hữu. Một cách khác là sử dụng tổ hợp lệnh sau 

```sql
REASSIGN OWNED BY doomed_role TO successor_role;
DROP OWNED BY doomed_role;
-- repeat the above commands in each database of the cluster
DROP ROLE doomed_role;
```

#### 1.4. Change role inside session 
Ta có thể chuyển đổi qua lại giữa các role trong session đang làm việc mà không cần phải login bằng 1 user khác. Việc này khá thuận tiện. Việc chuyển đổi này có được thực hiện hay không tuỳ vào thuộc vào config khi tạo role hoặc khi GRANT.

```sql
-- xem hiện tại đang ở role này 
SELECT current_role;

-- thay đổi role 
SET ROLE db_rw_user;

-- trả về role cũ 
RESET ROLE;
```

### 2. Privileges
Khi 1 database object được khởi tạo, nó sẽ được gán 1 owner. Thông thường, owner sẽ là role chạy lệnh tạo. Owner hoặc superuser có thể thực hiện tất cả operation với database object. Để user/role khác thực hiện các operation trên db object, cần phải cấp privileges. 

Có nhiều privileges khác nhau: SELECT, INSERT, UPDATE, DELETE, TRUNCATE, REFERENCES, TRIGGER, CREATE, CONNECT, TEMPORARY, EXECUTE, USAGE, SET and ALTER SYSTEM. 

Quyền thay đổi (vd như thêm cột) hay xoá 1 object thuộc về owner hoặc superuser. Tuy nhiên, ta có thể assign owner cho 1 role khác bằng câu lệnh `ALTER`

```sql
-- chỉ owner hoặc superuser được phép thực hiện câu lệnh này.

ALTER TABLE table_name OWNER TO new_owner;
```

Một số privileges cần chú ý:
- SELECT, INSERT, UPDATE, DELETE, TRUNCATE, REFERENCES, TRIGGER: những privilege này khá dễ hiểu 
- CREATE: 
  - Với database, cho phép tạo mới schema, cho phép cài đặt extension 
  - Với schema, co phép tạo mới table 
  - Không rõ về khái niệm tablespaces 
- CONNECT: Cho phép kết nối với database, quyền này được kiểm tra lúc login. Do đó, nếu muốn lock user nào đó, chỉ cần REVOKE quyền này là đủ. 
- EXECUTE: cho phép gọi function hoặc procedure. Quyền này chỉ áp dụng cho function và procedure
- USAGE
  - Không rõ về procedural language 
  - Với schema, cho phép truy câp các object trong schema đó, giả sử rằng điều kiện về owner đã thoả mãn. Nó cho phép grantee "tìm kiếm" object trong schema. Thiếu quyền này, không thể insert bảng có foreign key.
  - Với sequence, cho phép sử dụng currval và nextval function 

### 3. Use case
Đây là 1 use case tôi gặp, khi dùng postgres user khởi tạo 2 bảng, có foreign key reference. Sau đó, chuyển owner của 2 bảng này sang 1 user khác. Từ đây, ta không thể insert vào bảng có foreign key, dù sử dụng superuser. Đây chính là nguyên nhân khiến tôi tìm hiểu sâu hơn về Postgres permission 

```sql

-- create user db_rw_user 
create user db_rw_user password 'db_rw_user';

-- create database and grant connect permission to db_rw_user
create database test; 
grant connect on database test to db_rw_user;

-- create schema using postgres 
create schema test_permission;

-- set search_path
set search_path=test_permission;

-- create table 
create table student(id int, name text, primary key(id));

create table course(studentId int, course text);

ALTER TABLE course
      ADD CONSTRAINT fk_course FOREIGN KEY (studentId) 
          REFERENCES student (id);
         
-- change owner of table to db_rw user
alter table student owner to db_rw_user;
alter table course owner to db_rw_user;

-- student is a standalone table, don't have nay foreign key reference, so we can insert it 
insert into student (id, name) values (1, 'name 1');      
insert into student (id, name) values (2, 'name 2');   

-- fail to insert course table, because we have foreign key reference 
--  ERROR: permission denied for schema test_permission
insert into course (studentid, course) values(1, 'course 1');


-- first, we need to check the privilege of table and user db_rw_user 
-- user db_rw_user has all permission on table 
SELECT 
     tablename
     ,usename
     ,HAS_TABLE_PRIVILEGE(users.usename, tablename, 'select') AS sel
     ,HAS_TABLE_PRIVILEGE(users.usename, tablename, 'insert') AS ins
     ,HAS_TABLE_PRIVILEGE(users.usename, tablename, 'update') AS upd
     ,HAS_TABLE_PRIVILEGE(users.usename, tablename, 'delete') AS del
     ,HAS_TABLE_PRIVILEGE(users.usename, tablename, 'references') AS refs
FROM
(SELECT * from pg_tables
WHERE schemaname = 'test_permission' and tablename in ('student','course')) as tables
,(SELECT * FROM pg_user where usename = 'db_rw_user') AS users;


-- next, checking schema permission.
-- user db_rw_user don't have USAGE permission on test_permission schema ==> prevent "look up" for foreign key 
SELECT has_schema_privilege('db_rw_user', 'test_permission', 'USAGE') AS has_usage_permission;


-- Try to grant USAGE privilege 
GRANT USAGE ON SCHEMA test_permission TO db_rw_user;

SELECT has_schema_privilege('db_rw_user', 'test_permission', 'USAGE') AS has_usage_permission;

-- Then, try to insert student record again 
-- it works
insert into course (studentid, course) values(1, 'course 1'); 
```

### 4. References 
1. [PostgreSQL documentation - 5.7. Privileges](https://www.postgresql.org/docs/16/ddl-priv.html)
2. [PostgreSQL documentation - Part VI. Reference - GRANT command](https://www.postgresql.org/docs/current/sql-grant.html)
3. [PostgreSQL documentation - Chapter 22. Database Roles](https://www.postgresql.org/docs/16/user-manag.html)
4. [PostgreSQL documentation - Part VI. Reference - SET ROLE command](https://www.postgresql.org/docs/current/sql-set-role.html)
5. Mastering Postgres 13
