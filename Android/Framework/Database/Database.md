# DB

## 范式

* 原子性：

  字段不可再拆分。

  例子：省、市、街道

* 唯一性

  表中的一行是唯一的，加上id。

* 避免冗余性

  要求表中不能有其他表中存在的、存储相同信息的字段。需要使用外键。