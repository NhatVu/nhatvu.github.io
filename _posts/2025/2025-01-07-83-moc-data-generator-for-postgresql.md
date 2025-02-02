---
layout: post
title:  "83.  Mock data generator for PostgreSQL"
date:   2025-01-07 00:02:00 +0000
category: technical
---
- [1. Tạo schema](#1-tạo-schema)
- [2. Tạo lorem text](#2-tạo-lorem-text)
- [3. Tạo hàm cho mỗi bảng](#3-tạo-hàm-cho-mỗi-bảng)
- [4. Gọi hàm tạo test data](#4-gọi-hàm-tạo-test-data)
  - [4.1. Thảo luận](#41-thảo-luận)
- [5. Tablesample](#5-tablesample)
- [6. References](#6-references)

Bài viết này dựa trên ý tưởng của bài "Creating PostgreSQL Test Data with SQL, PL/pgSQL, and Python" [1]. Tuy nhiên, tôi cụ thể hoá từng bước và thêm một số câu hỏi cần thảo luận về việc tạo mock data cho PostgreSQL 

Trong quá trình phát triển phần mềm, việc tạo mock data giúp việc test phần mềm dễ hơn. Khi gặp sự cố, có thể tạo các dữ liệu tương tự ở PROD và test ở DEV hoặc STAGE. Ta cũng có thể dùng mock data để test performance của ứng dụng, giả sử lượng data do ứng dụng sinh ra trong 5-10 năm tới.

Trên thị trường, có tool hỗ trợ việc này, cả trả phí lẫn miễn phí. Với các tool miễn phí, thường chỉ giới hạn 1 lượng mock data nhất định, khoảng nhỏ hơn 500 rows. Điều này không đủ để thực hiện load test. 

Tôi cũng từng viết các ứng dụng tạo mock data bằng Java. Tuy nhiên, việc này tốt rất nhiều thời gian và đôi lúc khá khó khăn. Bài viết ở [1] gợi ý tôi 1 hướng mới trong việc tạo mock khi sử dụng chính các tool cung cấp bởi database. Với stored procedured, hàm  generate_series và hàm random, ta có thể tạo ra nhiều dạng dữ liệu khác nhau. 

Bài này tôi sẽ tập trung theo hướng sử dụng PL/pgSQL hơn là SQL. 


### 1. Tạo schema 

```sql

CREATE SCHEMA music;

SET search_path = music, $user, public;

/*
 * table schema creation 
 */

CREATE TABLE artists (
    artist_id serial PRIMARY KEY,
    name TEXT NOT NULL
);

CREATE TABLE albums (
    album_id serial PRIMARY KEY,
    artist_id INTEGER REFERENCES artists(artist_id) ON DELETE CASCADE,
    title TEXT NOT NULL,
    released DATE NOT NULL
);

CREATE TABLE genres (
    genre_id serial PRIMARY KEY,
    name TEXT NOT NULL
);

CREATE TABLE album_genres (
    album_id INTEGER REFERENCES albums(album_id) ON DELETE CASCADE,
    genre_id INTEGER REFERENCES genres(genre_id) ON DELETE CASCADE,
    PRIMARY KEY (album_id, genre_id)
);
```

### 2. Tạo lorem text 
Đối với các field dạng text như tên, địa chỉ, title, ... random text rất khó nhìn và trông không thực. Ta sẽ tạo 1 bảng chứa các word trong tiếng anh, sau đó, tạo 1 hàm để tạo ngẫu nhiên 1 câu chứa n word. Như vậy, sẽ dễ nhìn hơn rất nhiều 

**Tạo bảng words và thử chạy random từng word** 

```sql
-- Trong hướng dẫn tạo TEMPORARY table, nhưng tôi tạo luôn permanent table, vì lười phải tạo lại mỗi khi kết thúc session  
CREATE TABLE words (word TEXT);

-- when using docker OR remote server, the file need to be in docker or that remote server. 
-- So, user can use import function of dbeaver to import data with local file
-- Can download top 100k english word at: https://gist.githubusercontent.com/h3xx/1976236/raw/bbabb412261386673eff521dddbe1dc815373b1d/wiki-100k.txt
COPY words (word) FROM '/usr/share/dict/words' WHERE word NOT LIKE '%''%';

select count(*) from words ;

-- Randomly order the table and pick the first result
SELECT * FROM words ORDER BY random() LIMIT 1;
```

**Tạo hàm tạo câu với n word ngẫu nhiên**

```sql
/*
 * Create a function to generate random sentence in plpsql style 
 */
CREATE OR REPLACE FUNCTION generate_random_title(num_words int default 1) 
RETURNS text AS $$
DECLARE
  words text;
BEGIN
  SELECT initcap(array_to_string(array(
    SELECT * FROM words ORDER BY random() LIMIT num_words
  ), ' ')) INTO words;
  RETURN words;
END;
$$ LANGUAGE plpgsql;

-- test function
SELECT generate_random_title(ceil(random() * 4 + _g * 0)::int)
FROM generate_series(1, 4) AS _g;
```

### 3. Tạo hàm cho mỗi bảng 
Trước tiên, xoá data trên toàn bộ bảng theo thứ tự

```sql 
TRUNCATE albums, artists, genres, album_genres;
```

Hàm tạo mock data cho từng bảng 
```sql 
--- create genres 
CREATE OR REPLACE FUNCTION generate_genres(genre_options text[] default array['Hip Hop', 'Jazz', 'Rock', 'Electronic']) 
RETURNS void AS $$
BEGIN
  -- Convert each array option into a row and insert them into genres table.
  INSERT INTO genres (name) SELECT unnest(genre_options);
END;
$$ LANGUAGE plpgsql;

--------------

-- create artists
CREATE OR REPLACE FUNCTION generate_artists(size int default 10) 
RETURNS void AS $$
DECLARE
  artist_name text;
BEGIN
  FOR i IN 1..size LOOP
    SELECT generate_random_title(ceil(random() * 2)::int) INTO artist_name;
    -- About 50% of the time, add 'DJ ' to the front of the artist's name.
    IF random() > 0.5 THEN
      artist_name = 'DJ ' || artist_name;
    END IF;
    INSERT INTO artists (name)
    SELECT artist_name;
  END LOOP;
END;
$$ LANGUAGE plpgsql;

---------- generate album
CREATE OR REPLACE FUNCTION generate_albums(size int default 10) 
RETURNS void AS $$
DECLARE
  artist_name text;
BEGIN
  FOR i IN 1..size LOOP
    INSERT INTO albums (artist_id, title, released)
    VALUES (
      (SELECT artist_id FROM artists ORDER BY random() LIMIT 1),
      (SELECT generate_random_title(ceil(random() * 2)::int)),
      (now() - '5 years'::interval * random())::date
    );
  END LOOP;
END;
$$ LANGUAGE plpgsql;

--- generate album_genres 
CREATE OR REPLACE FUNCTION generate_albums_genres(size int default 10) 
RETURNS void AS $$
DECLARE
  dj_album RECORD;
BEGIN
  FOR i in 1..size LOOP
    INSERT INTO album_genres (album_id, genre_id)
    VALUES (
      (SELECT album_id FROM albums ORDER BY random() LIMIT 1),
      (SELECT genre_id FROM genres ORDER BY random() LIMIT 1)
    )
    -- If we insert a row that already exists, do nothing (don't raise an error)
    ON CONFLICT DO NOTHING;
  END LOOP;
 
  -- Ensure all albums by a 'DJ' artist belong to the Electronic genre.
  FOR dj_album IN
    SELECT albums.* FROM albums
    INNER JOIN artists USING (artist_id)
    WHERE artists.name LIKE 'DJ %'
  LOOP
    RAISE NOTICE 'Ensuring DJ album % belongs to Electronic genre!', quote_literal(dj_album.title);
    INSERT INTO album_genres (album_id, genre_id)
    SELECT dj_album.album_id, (SELECT genre_id FROM genres WHERE name = 'Electronic')
    -- If we insert a row that already exists, do nothing (don't raise an error)
    ON CONFLICT DO NOTHING;
  END LOOP;
END;
$$ LANGUAGE plpgsql;
```

### 4. Gọi hàm tạo test data 

```sql
-- create genres 
select * from generate_genres();

select * from genres ;

-- create 10k artists cost 1m2s
select * from generate_artists(10000)

select count(*) from artists ;

-- create 50k alumns cost 5m51s
select generate_albums(50000);

select count(*) from albums ;

-- create 10k albums_genres cost 27s
select generate_albums_genres(10000);

select count(*) from album_genres ;
```

#### 4.1. Thảo luận 
- Việc lấy ngẫu nhiên 1 dòng trong bảng chưa tối ưu, hiện tại ta thực hiện `SELECT album_id FROM albums ORDER BY random() LIMIT 1`. Việc này cần sort toàn bộ bảng theo `random()`. Nếu bảng lớn, việc này tốn nhiều thời gian và memory vì `ORDER BY` cần lấy tất cả rows, sau đó mới thực hiện sort. Để hạn chế việc này, ta có thể dùng câu lệnh thay thế `SELECT artist_id FROM artists where artist_id > floor(random() * 10000)  limit 1`. Việc thực hiện riêng rẻ thì câu lệnh 2 nhanh hơn cầu lệnh gốc, nhưng khi áp dụng vào việc generate mock data, tôi vẫn chưa thấy sự khác biệt. Một nguyên nhân có thể nghĩ đến là dữ liệu hiện tại vẫn đang nhỏ. Nếu có từ 5m - 10m dòng, có lẽ sẽ thấy sự khác biệt ?

### 5. Tablesample 
Để khắc phục hạn chế của `ORDER BY random() limit 1`, Postgres từ phiên bản 9.5 cung cấp `TABLESPACE`, cho phép trả về random sample [2]. Khi sử dụng explain, ta thấy `TABLESPACE` sử dụng `SAMPLESCAN` method, hoàn toàn các so với việc SeqScan hay IndexScan.

Hiện tại, Postgres cung cấp 2 phương pháp sample chính: SYSTEM và BERNOULLI
- SYSTEM: sử dụng random IO, nhanh hơn BERNOULLI. Thực hiện sample ở block/page level. Nó đọc và trả về page ngẫu nhiên từ disk. Nên đôi khi, giữa 2 lần chạy, kết quả trả về giống nhau. Để hạn chế tình trạng này, ta có thể thêm dòng `ORDER BY random()` 
- BERNOULLI: sử dụng sequentail IO. Thực hiện ở tuple level, cho kết quả sample distribution tốt hơn SYSTEM. Tuy nhiên, với dữ liệu lớn, sẽ chậm hơn SYSTEM 

Ví dụ:

```sql
SELECT album_id FROM albums tablesample system(1) ORDER BY random() 

--- equivalent query: không đảm bảo số row giống nhau, xấp xỉ mà thôi 
SELECT album_id FROM albums  ORDER BY random() < 0.01
```

SYSTEM và BERNOULLI đều yêu cầu percentage parameter, % return rows. Tuy nhiên, cần lưu ý, câu query trên có thể trả về null. Nếu percentage nhỏ, tỉ lệ trả về null lớn hơn. 

Một lưu ý quan trọng là `WHERE` clause sẽ được áp dụng sau khi thực hiện sampling. Để hiểu rõ hơn, có thể dùng lệnh explain để xem. Để khắc phục việc này, ta có thể tạo ra 1 temp table, và thực hiện sample trên temp table đó. Tuy nhiên, kỹ thuật này tôi vẫn chưa nghiên cứu kỹ về hệ quả của temp table. 

Viết lại hàm `generate_albums_genres` có sử dụng sample method 

```sql
CREATE OR REPLACE FUNCTION generate_albums_genres_using_tablesample(size int default 10) 
RETURNS void AS $$
DECLARE
  dj_album RECORD;
  sample_album_id int;
BEGIN
  FOR i in 1..size loop
	SELECT album_id 
	into sample_album_id
	FROM albums tablesample system(1) ORDER BY random() LIMIT 1;

	if sample_album_id is null then 
		continue; -- skip when sample id is null
	end if;

    INSERT INTO album_genres (album_id, genre_id)
    VALUES (
      sample_album_id,
      (SELECT genre_id FROM genres ORDER BY random() LIMIT 1)
    )
    -- If we insert a row that already exists, do nothing (don't raise an error)
    ON CONFLICT DO NOTHING;
  END LOOP;
 
  -- Ensure all albums by a 'DJ' artist belong to the Electronic genre.
  FOR dj_album IN
    SELECT albums.* FROM albums
    INNER JOIN artists USING (artist_id)
    WHERE artists.name LIKE 'DJ %'
  LOOP
    RAISE NOTICE 'Ensuring DJ album % belongs to Electronic genre!', quote_literal(dj_album.title);
    INSERT INTO album_genres (album_id, genre_id)
    SELECT dj_album.album_id, (SELECT genre_id FROM genres WHERE name = 'Electronic')
    -- If we insert a row that already exists, do nothing (don't raise an error)
    ON CONFLICT DO NOTHING;
  END LOOP;
END;
$$ LANGUAGE plpgsql;
```

Đoạn code để đo tốc độ query 
```sql 
DO $$
DECLARE
    start_time TIMESTAMP;
    end_time TIMESTAMP;
    elapsed INTERVAL;
BEGIN
    -- Capture start time
    start_time := clock_timestamp();

    -- Your SQL/PLpgSQL logic here 
   -- 10k: 1535 ms
   -- 10k: 1500 ms
   -- 50k: 5065
   -- PERFORM generate_albums_genres_using_tablesample(10000);
   
   -- 10k: 59138 ms
   -- 10k: 59546 ms
   	PERFORM generate_albums_genres(10000);

    -- Capture end time
    end_time := clock_timestamp();

    -- Calculate elapsed time
    elapsed := end_time - start_time;

    -- Display the result
    RAISE NOTICE 'Execution Time : % ms', EXTRACT(MILLISECONDS FROM elapsed);
END $$;
```

So sánh khi generate 10k records

| No | generate_albums_genres | generate_albums_genres_using_tablesample |
|----|------------------------|------------------------------------------|
| 1  | 59138 ms               | 1535 ms                                  |
| 2  | 59546 ms               | 1500 ms                                  |


### 6. References 
1. [Creating PostgreSQL Test Data with SQL, PL/pgSQL, and Python](https://www.tangramvision.com/blog/creating-postgresql-test-data-with-sql-pl-pgsql-and-python)
2. [Tablesample In PostgreSQL 9.5](https://www.enterprisedb.com/blog/tablesample-postgresql-95)
