---
title: Oracle Notes
date: 2024-11-20 16:33:11
categories: Notes
excerpt: Oracle数据库的一些简单笔记
---

# oracle

## 登录oracle
```bash
su - oracle
sqlplus / as sysdba

sqlplus username/password@ip/instance_name
```

## 修改oracle密码策略
```sql
# 查看所有的PDB
show pdbs;
# 切换到指定PDB
alter session set contaniner=PDB_NAME;
# 再次查看pdb，可以看到已经切换到指定PDB

# 修改密码策略
alter profile default limit FAILED_LOGIN_ATTEMPTS UNLIMITED;
alter profile default limit PASSWORD_LIFE_TIME UNLIMITED;
alter profile default limit PASSWORD_LOCK_TIME UNLIMITED;
alter profile default limit PASSWORD_REUSE_TIME UNLIMITED;
alter profile default limit PASSWORD_REUSE_MAX UNLIMITED;
alter profile default limit PASSWORD_VERIFY_FUNCTION NULL;
alter profile default limit PASSWORD_GRACE_TIME UNLIMITED;
```
