---
title: PostgreSQL-本地事务与全局事务
date: 2018-09-24 15:06:54
subtitle: postgres_transaction
tags:
- postgresql
---

## 简介

数据库事物的控制，不支持跨节点。因为当涉及跨conn的事务时，存在可见性的问题（conn间的操作，在未提交前互相不可见）。

全局事务（conn不变）:
先precommit (存入系统表，返回对应主键id， 其中使用gtid作为前缀，gtid为业务拼接的全局事务id) ，全部ok后，查系统表(gtid作为前缀)将符合前缀的全部commit，遇到失败就rollback。存在部分commit、rollback失败时，其余的仍要尽力完成。

本地事务:
直接commit，失败就回滚。

## 功能调测

```
2018/09/10
跨conn:
场景一
A_Conn起事务， inset， precommit。
B_Conn起事务， insert...卡住，直到Acommit后报错。
场景二：
A_Conn起事务， inset;
B_Conn起事务， insert...卡住...
结论:
事务启动后， insert时已经把关键key锁住了， precommit只是把具体的数据落地(存入数据库，事务状态完全保存在磁盘上)，只是此时仍数据"隐藏状态"，其它conn不可见。
所以有种说法， precommit后，即使没有commit也不影响数据commit的成功。
```

## 参考资料

http://www.jasongj.com/big_data/two_phase_commit/