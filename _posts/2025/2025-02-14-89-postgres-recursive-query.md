---
layout: post
title:  "89.  PostgreSQL: Recursive query"
date:   2025-02-14 00:02:00 +0000
category: technical
---
- [1. Cú pháp recursive query](#1-cú-pháp-recursive-query)
- [2. Ví dụ](#2-ví-dụ)
- [3. References](#3-references)


PostgresSQL cung cấp cú pháp `WITH RECURSIVE` cho phép thực hiện đệ quy trong câu lệnh SQL. Điều này cực kỳ hữu ích khi thao tác với hierarchy data structure như tree, graph. 

### 1. Cú pháp recursive query 
Theo [1], cú pháp của câu lệnh được định nghĩa như sau 
```sql
WITH RECURSIVE cte_name (column1, column2, ...)
AS(
    -- anchor member
    SELECT select_list FROM table1 WHERE condition

    UNION [ALL]

    -- recursive term
    SELECT select_list FROM cte_name WHERE recursive_condition
)
SELECT * FROM cte_name;
```

Based on [1], In this syntax:

- cte_name: Specify the name of the CTE. You can reference this CTE name in the subsequent parts of the query.
- column1, column2, … Specify the columns selected in both the anchor and recursive members. These columns define the CTE’s structure.
- Anchor member: Responsible for forming the base result set of the CTE structure.
- Recursive member: Refer to the CTE name itself. It combines with the anchor member using the UNION or UNION ALL operator.
- recursive_condition: Is a condition used in the recursive member that determines how the recursion stops.
  
PostgreSQL executes a recursive CTE in the following sequence:

- First, execute the anchor member to create the base result set (R0).
- Second, execute the recursive member with Ri as an input to return the result set Ri+1 as the output.
- Third, repeat step 2 until an empty set is returned. (termination check)
- Finally, return the final result set that is a UNION or UNION ALL of the result sets R0, R1, … Rn.

Tuy nhiên, lưu ý về cycle loop và depth level traversal khi chạy câu query. 

### 2. Ví dụ 
Dựa vào ví dụ ở [2], ta sẽ viết câu SQL ngăn chặn cycle loop, cũng như thông tin về depth_level 

```sql 
CREATE TABLE "graph" ("id" integer, "parent_id" integer);

INSERT INTO "graph" VALUES
(1, null), -- root element
(2, 1),
(3, 1),
(4, 3), 
(5, 4), 
(3, 5); -- endless loop
```

Mô hình graph 
```
1 --+--> 2
    |
    +--> 3 --> 4 --> 5 --+
         ^               |
         |               |
         +---------------+
```

Câu query, có chỉnh sửa 
```sql 
WITH RECURSIVE cte(id, parent_id, depth, path) AS (
  SELECT id,
         parent_id,
         0,
         array_remove(ARRAY[parent_id, id], null) as "path"
  FROM graph
  WHERE parent_id = 1

  UNION ALL

  SELECT graph.id,
         graph.parent_id,
         cte.depth + 1,
         cte.path || graph.id
  FROM graph
    INNER JOIN cte
    ON graph.parent_id = cte.id
    AND graph.id <> all(path)
    AND cte.depth < 8 -- default limit depth level
)
SELECT *
FROM cte
```

Note:
- Anything compare with null will return null. Because our parent_id can be null as first, to let algorithm run correctly, we need to remove the null value at anchor phase by using `array_remove` function 


Sample output when searching with parent is = 1

```
id|parent_id|depth|path     |
--+---------+-----+---------+
 2|        1|    0|{1,2}    |
 3|        1|    0|{1,3}    |
 4|        3|    1|{1,3,4}  |
 5|        4|    2|{1,3,4,5}|
```

### 3. References 
1. [PostgreSQL Recursive Query](https://neon.tech/postgresql/postgresql-tutorial/postgresql-recursive-query)
2. [Prevent infinite loop in recursive query in Postgresql](https://stackoverflow.com/questions/51025607/prevent-infinite-loop-in-recursive-query-in-postgresql)

