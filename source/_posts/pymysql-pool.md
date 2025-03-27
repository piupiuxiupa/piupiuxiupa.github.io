---
title: pymysql与连接池
date: 2024-12-09 20:11:33
tags: python
categories: Posts
---

# pymysql 与连接池

## pymysql

```python
import pymysql

# 1. 连接到 MySQL 数据库
connection = pymysql.connect(
    host='localhost',         # 数据库地址
    user='root',              # 数据库用户名
    password='your_password', # 数据库密码
    database='test_db',       # 数据库名称（需事先创建好）
    charset='utf8mb4',        # 设置字符集
    cursorclass=pymysql.cursors.DictCursor  # 返回结果为字典格式
)

try:
    # 2. 创建游标对象
    with connection.cursor() as cursor:
        # 创建表
        create_table_query = """
        CREATE TABLE IF NOT EXISTS users (
            id INT AUTO_INCREMENT PRIMARY KEY,
            name VARCHAR(100) NOT NULL,
            age INT NOT NULL,
            email VARCHAR(100)
        );
        """
        cursor.execute(create_table_query)

        # 插入数据
        insert_query = "INSERT INTO users (name, age, email) VALUES (%s, %s, %s)"
        cursor.execute(insert_query, ("Alice", 25, "alice@example.com"))
        cursor.execute(insert_query, ("Bob", 30, "bob@example.com"))

        # 提交事务
        connection.commit()

        # 查询数据
        select_query = "SELECT * FROM users"
        cursor.execute(select_query)
        results = cursor.fetchall()
        print("查询结果:")
        for row in results:
            print(row)

        # 更新数据
        update_query = "UPDATE users SET age = %s WHERE name = %s"
        cursor.execute(update_query, (28, "Alice"))
        connection.commit()

        # 删除数据
        delete_query = "DELETE FROM users WHERE name = %s"
        cursor.execute(delete_query, ("Bob",))
        connection.commit()

finally:
    # 3. 关闭数据库连接
    connection.close()
```

## 连接池

在使用 pymysql 连接 MySQL 时，可以通过引入 连接池 来提高数据库操作效率，避免频繁建立和关闭连接带来的开销。

1. 安装依赖

    为了使用连接池，可以通过 DBUtils 库的 PooledDB 模块实现连接池管理。

    ```bash
    pip install DBUtils
    ```

2. 使用 DBUtils.PooledDB 实现连接池

    ```python
    from DBUtils.PooledDB import PooledDB
    import pymysql

    # 创建连接池
    pool = PooledDB(
        creator=pymysql,          # 使用 pymysql 作为连接器
        maxconnections=5,         # 最大连接数
        mincached=2,              # 初始化时创建的连接数
        maxcached=3,              # 连接池中最多可缓存的连接数
        blocking=True,            # 达到最大连接数后是否阻塞，True 为阻塞
        ping=0,                   # 检测连接是否可用，0 表示从不检测
        host="localhost",         # MySQL 主机地址
        user="root",              # 用户名
        password="your_password", # 密码
        database="test_db",       # 数据库名
        charset="utf8mb4"         # 字符集
    )

    # 从连接池获取连接
    def get_connection():
        return pool.connection()

    # 使用连接池连接数据库
    def execute_query(query, params=None):
        conn = get_connection()   # 从连接池获取连接
        try:
            with conn.cursor() as cursor:
                cursor.execute(query, params)
                if query.strip().upper().startswith("SELECT"):
                    result = cursor.fetchall()  # 查询语句，返回结果
                    return result
                else:
                    conn.commit()  # 非查询语句，提交事务
        except Exception as e:
            print("Error:", e)
            conn.rollback()  # 回滚事务
        finally:
            conn.close()  # 将连接返回到连接池

    # 示例：插入数据
    execute_query(
        "INSERT INTO users (name, age, email) VALUES (%s, %s, %s)",
        ("Alice", 25, "alice@example.com")
    )

    # 示例：查询数据
    results = execute_query("SELECT * FROM users")
    print(results)
    ```

3. 参数说明

    连接池参数
   - creator：指定使用的数据库连接模块，这里为 pymysql。
   - maxconnections：连接池允许的最大连接数，0 表示不限制连接数。
   - mincached：初始化时创建的连接数。
   - maxcached：池中允许的最大空闲连接数。
   - blocking：如果连接数已达到最大值，是否阻塞等待可用连接。True 表示阻塞。
   - ping：控制是否在获取连接时检查连接的可用性
       - 0：从不检查；
       - 1：执行 ping 检测；
       - 2：检查 socket；
       - 4：检查断开情况；
       - 可以组合值如 3 表示 ping + socket 检查

    MySQL 连接参数
    - host：MySQL 数据库地址（例如 localhost 或 IP 地址）。
    - user：MySQL 用户名。
    - password：MySQL 密码。
    - database：指定连接的数据库名。
    - charset：设置字符集，推荐使用 utf8mb4。
