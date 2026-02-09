---
title: SQL Notes
date: 2025-11-06 20:41:49
tags: database
categories: Notes
excerpt: SQL 常用语句
---

## SQL Notes

### 查询

```sql
SELECT * FROM table_name;

-- 分页查询
SELECT * FROM table_name
LIMIT num OFFSET offset;

SELECT *
FROM (
    SELECT *,
           ROW_NUMBER() OVER (ORDER BY id) AS rn
    FROM your_table
) t
WHERE rn BETWEEN 11 AND 20;

-- 分组查询
SELECT column1, column2, ...
FROM table_name
GROUP BY column1, column2, ...;

-- 聚合查询
SELECT column1, aggregate_function(column2)
FROM table_name
GROUP BY column1;

-- 排序查询
SELECT * FROM table_name
ORDER BY column1, column2, ... ASC|DESC;

-- 联表查询
SELECT * FROM table1
JOIN table2 ON table1.id = table2.id;

-- 查询时间区间
SELECT * FROM table_name
WHERE date_column BETWEEN 'start_date' AND 'end_date';
-- 也可以用大于小于号查询
SELECT * FROM table_name
WHERE date_column > 'start_date' AND date_column < 'end_date';

-- 查询几天前的数据
SELECT * FROM table_name
WHERE date_column >= DATE_SUB(CURDATE(), INTERVAL 7 DAY);
-- 也可用减号查询
SELECT * FROM table_name
WHERE date_column >= CURDATE() - INTERVAL 7 DAY;

-- 去重查询
SELECT DISTINCT column1, column2, ...
FROM table_name;
-- distinct只能显示指定列或者所有列进行去重
-- 可以用row_number()函数针对部分列进行去重后，再筛选出需要的列的数据
SELECT column1, column2, ...
FROM (
    SELECT column1, column2, ...,
           ROW_NUMBER() OVER (PARTITION BY column1, column2, ... ORDER BY column1, column2, ...) AS row_num
    FROM table_name
) AS subquery
WHERE row_num = 1;

-- 分组查询后筛选
SELECT column1, aggregate_function(column2)
FROM table_name
GROUP BY column1
HAVING aggregate_function(column2) > value;

-- 复杂嵌套查询
SELECT t.login_id, t.url_path, t.module_name, a.display_name
FROM 
(
	SELECT DISTINCT login_id , url_path ,
		case
			when url_path like "/searchv1/%" THEN CONCAT(SUBSTRING_INDEX(module_name, '-', 1), '(旧)')
			when url_path like "/searchv2/%" THEN SUBSTRING_INDEX(module_name, '-', 1)
			else module_name
		end as module_name
	FROM test.module_info
	WHERE 
        event_time >= "2025-10-23 00:00:00"  AND 
		login_id IS NOT NULL
		AND login_id NOT IN (
			SELECT loginName FROM `test`.`user_info`
			WHERE state = 1 AND `deptCode` = '11111' AND loginName IS NOT NULL
		)
	ORDER BY login_id
) AS t
LEFT JOIN 
(
	SELECT `test`.`user_info`.`loginName` AS `login_name`,
	   CONCAT(`test`.`user_info`.`realName`, '(', `test`.`user_info`.`deptName`, ')') AS `display_name`
	FROM
	   `test`.`user_info`
	WHERE
	   `test`.`user_info`.`state` = 1
) AS a
ON t.login_id = a.login_name
WHERE a.display_name IS NOT NULL
ORDER BY t.login_id
```

> 在查询时建议明确指定需要查询的列，避免查询所有列（`SELECT *`），因为这会增加查询的负担

### 表操作

表设计细节：

- 表示状态用TINYINT节约空间
- 表示时间用DATETIME类型
- doris数据库表用主键表类型时，STRING属性列不能作为键
- 冗余字段记录更新时间、创建时间、操作来源，比如：`UPDATE_TIME`、`CREATE_TIME`、`OPERATOR`

```sql
-- mysql
CREATE TABLE table_name (
    column1 datatype,
    column2 datatype,
    -- 插入数据时触发
    create_time datetime default current_timestamp on insert    current_timestamp,
    -- 更新数据时触发
    update_time datetime default current_timestamp on update current_timestamp,
    ...
);
```

Doris 的建表语句中可以指定建表属性，包括：

- 分桶数 (buckets)：决定数据在表中的分布；
- 存储介质 (storage_medium)：控制数据的存储方式，如使用 HDD、SSD 或远程共享存储；
- 副本数 (replication_num)：控制数据副本的数量，以保证数据的冗余和可靠性；
- 冷热分离存储策略 (storage_policy) ：控制数据的冷热分离存储的迁移策略；

```sql
-- doris建表
-- 与mysql不同，doris需要根据业务场景选择不同的表类型
-- 明细表（Duplicate Key Table）：允许指定的 Key 列重复，Doirs 存储层保留所有写入的数据，适用于必须保留所有原始数据记录的情况；

-- 主键表（Unique Key Table）：每一行的 Key 值唯一，可确保给定的 Key 列不会存在重复行，Doris 存储层对每个 key 只保留最新写入的数据，适用于数据更新的情况；

-- 聚合表（Aggregate Key Table）：可根据 Key 列聚合数据，Doris 存储层保留聚合后的数据，从而可以减少存储空间和提升查询性能；通常用于需要汇总或聚合信息（如总数或平均值）的情况。

CREATE TABLE IF NOT EXISTS example_tbl_unique
(
    user_id LARGEINT NOT NULL,
    user_name VARCHAR(50) NOT NULL,
    city VARCHAR(20),
    age SMALLINT,
    sex TINYINT,
    -- doris不支持 ON UPDATE CURRENT_TIMESTAMP，需要手动更新
    UPDATE_TIME DATETIME DEFAULT CURRENT_TIMESTAMP,
    CREATE_TIME DATETIME DEFAULT CURRENT_TIMESTAMP,
    OPERATOR VARCHAR(50)
)
UNIQUE KEY(user_id, user_name)
DISTRIBUTED BY HASH(user_id) BUCKETS 10
PROPERTIES (
    "enable_unique_key_merge_on_write" = "true"
);

-- 修改表结构
ALTER TABLE table_name
MODIFY column_name datatype;

ALTER TABLE user_info ADD COLUMN status_tiny TINYINT DEFAULT 0 COMMENT '状态';

-- 在某列之后添加列
-- doris的TINYINT类型指定默认值的时候需要加单引号，不能直接写数字
ALTER TABLE user_info ADD COLUMN status_tiny TINYINT DEFAULT '0' COMMENT '状态' AFTER user_name;


UPDATE user_info SET status_tiny = CAST(status AS TINYINT);

-- 删除列
ALTER TABLE user_info DROP COLUMN status;

-- 重命名列
ALTER TABLE user_info RENAME COLUMN status_tiny status; 

-- 删除旧表，重命名新表为旧表名
DROP TABLE example;
ALTER TABLE example_new RENAME example;
```

### 数据操作

doris的insert会触发列默认值，update不会触发列默认值，需要手动指定值

```sql
-- 导入旧数据
INSERT INTO example_new SELECT * FROM example;

-- 更新数据
UPDATE example_new SET status = '1' WHERE user_id = 123;
```

### 权限修改

```sql
-- doris授权需要单独指定
CREATE USER 'username'@'%' IDENTIFIED BY 'password'; 
GRANT CREATE_PRIV ON test.* TO 'username'@'%';
GRANT DROP_PRIV ON test.* TO 'username'@'%';
GRANT ALTER_PRIV ON test.* TO 'username'@'%';
GRANT SELECT_PRIV ON test.* TO 'username'@'%';
GRANT LOAD_PRIV ON test.* TO 'username'@'%';

-- 回收权限
REVOKE CREATE_PRIV ON test.* FROM 'username'@'%';
REVOKE DROP_PRIV ON test.* FROM 'username'@'%';
REVOKE ALTER_PRIV ON test.* FROM 'username'@'%';
REVOKE SELECT_PRIV ON test.* FROM 'username'@'%';
REVOKE LOAD_PRIV ON test.* FROM 'username'@'%';


-- mysql授权所有权限
GRANT ALL PRIVILEGES ON test.* TO 'username'@'%';
FLUSH PRIVILEGES;
```

```sql
-- oracle 权限修改
-- with grant option 可以将该权限转授给其他pattern
GRANT SELECT ON 'pattern'.'table' TO 'user' WITH GRANT OPTION;
-- 回收权限
REVOKE SELECT ON 'pattern'.'table' FROM 'user';
```
