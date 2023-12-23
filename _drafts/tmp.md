Spring JPA schema evolution 

Postgres database, when changing schema on database, all data will be migrated to new schema with default value. 

backward: new code will work with old data (will not apply here)
forward: old code will work with new data (need to consider)
- Add filed: backward and forward compatibility 
- Change type: 
- Delete field: 
    + Delete in Entity: normal
    + Delete in Datbase: Error missing column. 