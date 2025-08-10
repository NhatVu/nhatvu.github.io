---
layout: post
title: "96. [POC] - Key-value cache using Postgres"
date: 2025-08-09 08:02:00 +0000
category: technical
---

Sometines, we can't use Redis or Memcached for caching pupose. One reason is license problem and that technology hasn't approved to be used inside your company. Instead, we need to design a solution that use existing infrastructure like PostgreSQL database.

Note: Using ChatGPT to plan the idea

## Core features

We need to design a caching solution that satisfy these constrains:

- Using Postgres
- Support TimeToLive or Expired time
- Support Least Recent Used (LRU) behaviour. We can active refresh cache that frequently used and/optional delete old cached
- Support unique refresh for key if the application running on multiple instances.
- Using background refresh job to simplify the app logic

## Design and important SQL queries

### Table

```sql
CREATE TABLE cache_entries (
    key TEXT PRIMARY KEY,
    value JSONB,
    expires_at TIMESTAMP,       -- When value is considered expired
    last_accessed_at TIMESTAMP, -- timestamp every time the key is read from cache. Support LRU
    refresh_started_at TIMESTAMP,     -- When value was last recomputed
    is_refreshing BOOLEAN,       -- Used to prevent duplicate refresh
    updated_at TIMESTAMP
);

```

### Logics

**User Request (can have multiple applicaitons) →**
- Check cache_entries → 
  - if fresh → use 
  - if expired → use (optional), refresh (optional) 
  - if missing → compute, insert to cache
  - when a key is used, always record the last_accessed_at. Using batch update, remove duplicated key to reduce database stress.

**Batch Worker (can have multiple workers) →**
- every X min →
    - find near-expired keys →
    - acquire lock (UPDATE with conditions) →
    - compute →
    - update value, release lock

**Cleanup Job →**
- delete old unused keys

### SQL queries for batch refresh key

a. Select key to refresh

```sql
SELECT key FROM cache_entries
WHERE expires_at <= now()
  AND last_accessed_at >= now() - interval '7 days' -- used in last 7 days
  AND (is_refreshing = false OR refresh_started_at < now() - interval '5 minutes') -- is not refreshing by other service or stuck at least 5 mintues (in case app refresh during refresh or network failure)
ORDER BY expires_at ASC
LIMIT 50;
```

- This ensures you don’t waste resources refreshing stale/inactive data.
- If refresh a key fail, make sure that key can be refresh again. For safety reason.

b. Acquiring refresh lock

```sql
UPDATE cache_entries
SET is_refreshing = true,
    refresh_started_at = now()
WHERE key = ? AND (is_refreshing = false OR refresh_started_at < now() - interval '5 minutes')
```

- Only one app instance should refresh the key. Use an atomic update query to acquire the lock
- The condition ensures you can reclaim the lock if it’s stuck older than 5 minutes (tunable)
- This returns 1 row affected only if the lock was acquired.

c. Release lock after refresh

```sql
UPDATE cache_entries
SET is_refreshing = false,
    refresh_started_at = NULL, -- must be null
    value = ?,                -- optional: update with new value
    expires_at = ?           -- optional: update expiration time

WHERE key = ?;
```

d. LRU-style cleanup

```sql
DELETE FROM cache_entries
WHERE last_accessed_at < now() - interval '7 days'
```

e. Update last access 
```sql
UPDATE cache_entries
SET last_accessed_at = now()
WHERE key = ?;
```

### Sample code 
https://github.com/NhatVu/proof-of-concept/tree/main/postgres-key-value-cache




