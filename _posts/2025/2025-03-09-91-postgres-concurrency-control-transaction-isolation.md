---
layout: post
title:  "91.  PostgreSQL Concurrency control: Transaction Isolation"
date:   2025-03-09 00:02:00 +0000
category: technical
---
Theo chuẩn của SQL, transaction isolation gồm 4 dạng: Serializable > Repeatable read > Read committed > Read uncommitted. Tuy nhiên, mỗi loại database sẽ hỗ trợ các dạng isolation khác nhau. Riêng PostgreSQL không hỗ trợ Read uncommited. 

### Transaction Isolation 

Serializable isolation là nghiêm ngặt nhất, các transaction phải thực hiện tuần tự, không được phép thực hiện đồng thời (no concurrent transaction). 

3 loại isolation còn lại được định nghĩa theo read phenomena, là kết quả của việc nhiều transaction được thực thi đồng thời. 

Các read phenomena là: 
- dirty read: trên cùng 1 data (row), transaction A có thể đọc dữ liệu được ghi bởi transaction B khi B chưa commit.  
- nonrepeatable read: trên cùng 1 data (row) transaction re-read dữ liệu nó đã đọc trước đó và phát hiện rằng, dữ liệu đã bị thay đổi bởi transaction khác.
- phantom read: với câu query trả về 1 tập dữ liệu, 1 transaction thi re-execute câu query, phát hiện rằng tập dữ liệu trả về bị thay đổi (thêm dòng, hoặc xoá dòng) bởi các transaction khác.  
- serialization anomaly: The result of successfully committing a group of transactions is inconsistent with all possible orderings of running those transactions one at a time.

![alt text](/assets/images/2025/89_transaction_isolation_level.png)

Quan trọng: sequence type (đựoc khai báo bởi serial) khi có thay đổi, sẽ ngay lập tức visible với transaction và không thể rollback, kể cả khi transaction không thành công. 

### Read Committed Isolation Level
Read Committed is the default isolation level in PostgreSQL. When a transaction uses this isolation level, a SELECT query (without a FOR UPDATE/SHARE clause) sees only data committed before the query began. 

Note that two successive SELECT commands can see different data, even though they are within a single transaction, if other transactions commit changes after the first SELECT starts and before the second SELECT starts.

UPDATE, DELETE, SELECT FOR UPDATE, and SELECT FOR SHARE commands behave the same as SELECT in terms of searching for target rows: they will only find target rows that were committed as of the command start time. However, such a target row might have **already** been updated (or deleted or locked) by another concurrent transaction by the time it is found. In this case, the would-be updater will wait for the first updating transaction to commit or roll back (if it is still in progress).
- If the first updater rolls back, then its effects are negated and the second updater can proceed with updating the originally found row.
- If the first updater commits, the second updater will ignore the row if the first updater deleted it, otherwise it will attempt to **apply its operation to the updated version of the row**. The search condition of the command (the WHERE clause) is re-evaluated to see if the updated version of the row still matches the search condition. If so, the second updater proceeds with its operation using the updated version of the row


More complex usage can produce undesirable results in Read Committed mode. For example, consider a DELETE command operating on data that is being both added and removed from its restriction criteria by another command, e.g., assume website is a two-row table with website.hits equaling 9 and 10:
```sql 
BEGIN;
UPDATE website SET hits = hits + 1;
-- run from another session:  DELETE FROM website WHERE hits = 10;
COMMIT;
```
The DELETE will have no effect even though there is a website.hits = 10 row before and after the UPDATE. This occurs because the pre-update row value 9 is skipped (Note: not the targeted row for DELETE sql), and when the UPDATE completes and DELETE obtains a lock, the new row value is no longer 10 but 11, which no longer matches the criteria.

The partial transaction isolation provided by Read Committed mode is adequate for many applications, and this mode is fast and simple to use; however, it is not sufficient for all cases. Applications that do complex queries and updates might require a more rigorously consistent view of the database than Read Committed mode provides.

### Repeatable Read Isolation Level
The Repeatable Read isolation level only sees data committed before the transaction began; it never sees either uncommitted data or changes committed by concurrent transactions during the transaction's execution. (However, each query does see the effects of previous updates executed within its own transaction, even though they are not yet committed.) 

This level is different from Read Committed in that a query in a repeatable read transaction sees a snapshot as of the start of the first non-transaction-control statement in the transaction, not as of the start of the current statement within the transaction. Thus, successive SELECT commands within a single transaction see the same data, i.e., they do not see changes made by other transactions that committed after their own transaction started.

Applications using this level must be prepared to retry transactions due to serialization failures.

```
ERROR:  could not serialize access due to concurrent update
```

A repeatable read transaction cannot modify or lock rows changed by other transactions after the repeatable read transaction began.

When an application receives this error message, it should abort the current transaction and retry the whole transaction from the beginning. The second time through, the transaction will see the previously-committed change as part of its initial view of the database, so there is no logical conflict in using the new version of the row as the starting point for the new transaction's update.


### Serializable Isolation Level 
The Serializable isolation level provides the strictest transaction isolation. This level emulates serial transaction execution for all committed transactions; as if transactions had been executed one after another, serially, rather than concurrently.

However, like the Repeatable Read level, applications using this level must be prepared to retry transactions due to serialization failures

Other parts, will come back later, as I don't understand at the moment.

### References 
1. [13.2. Transaction Isolation](https://www.postgresql.org/docs/16/transaction-iso.html)
