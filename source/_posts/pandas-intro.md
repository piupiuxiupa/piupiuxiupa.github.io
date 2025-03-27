---
title: Pandas Intro
date: 2025-03-27 20:37:30
tags: python
categories: Posts
---

# Pandas 简介

## 安装

```bash
pip install pandas
```

## Pandas 基础操作

1. 字典生成表格数据

    ```python
    dict_sample={
        "A": [11,22,33,44]
        "B": [22,33,44,55]
        "C": ['a','b','c','d']
    }
    df = pd.DataFrame(dict_sample)
    display(df)
    ```

2. 行索引和列索引

    ```python
    # 行索引
    df.index
    # 输出结果：
    # RangeIndex(start=0, stop=4, step=1)
    # 列索引
    df.columns
    # 输出结果：
    # Index(['A', 'B', 'C'], dtype='object')
    # 转化为列表
    df.columns.tolist()
    # 输出结果：
    # ['A', 'B', 'C']
    df.index.tolist()   
    # 输出结果：
    # [0, 1, 2, 3]
    ```

3. 指定行索引列索引

    ```python
    # 重新赋值行索引
    df.index = ['a','b','c','d']
    # 重新赋值列索引
    df.columns = ['A','B','C']
    # 单独改变某一列列名
    df = df.rename(columns={'A':'a'})
    # 直接修改原数据，不返回新对象
    df.rename(columns={'A':'a'},inplace=True)
    # 转换回字典
    df_dict = df.to_dict()

    ## 单独指定行索引
    df = pd.DataFrame(dict_sample,index=['a','b','c','d'])
    ```

4. 四种选取数据的方法

    ```python
    # 选取某一列，且返回为Series类型
    df['A']
    # 选取某一列，且返回为DataFrame类型
    df[['A']]
    # loc 根据索引选取数据
    df.loc['a'，:]
    # 选取某几行， 且返回为DataFrame类型
    df.loc[['a','b']，:]
    # iloc 根据位置选取数据，从0开始编号
    df.iloc[0,：]
    # 选取某几行某几列
    df.iloc[[0,1], :]
    df.iloc[[0,1],[0,1]]
    df.iloc[0:2,0:2]
    # 选取某行某列的一个元素
    df.at[0,'A']
    ```

## 常用操作

1. 外部数据读取

    ```python
    # 读取csv文件
    # 常用参数：index_col 索引行； sep 分隔符；
    df = pd.read_csv('data.csv')  
    ```

2. 打印预览

    ```python
    # 打印前几行数据
    df.head()
    # 打印后几行数据
    df.tail()
    # 打印随机几行数据
    df.sample()
    ```

3. 基本信息预览

    ```python
    # 基本信息预览
    df.info()   
    df.describe()
    df.dtypes
    df.shape
    ```

4. 统计性信息

    ```python
    # 统计性信息
    df.count()
    df.mean()
    df['A'].unique()
    # 根据列的值分类，对每个类型进行计数
    df[['A','B']].groupby(['A']).count()
    ```

5. 索引设置

    ```python
    # 索引设置
    # 将行索引变为列
    df.reset_index()   
    # set_index 与 reset_index 相反
    df.set_index(['A','B'])
    ```

6. 排序

    ```python
    # 排序
    # 根据某一列排序
    df.sort_values(by='A',ascending=True)      
    # 根据多列排序
    df.sort_values(by=['A','B'],ascending=[True,False])
    ```

7. 拼接

    ```python
    # 拼接
    # 横向拼接
    df1 = pd.DataFrame({'A': [1, 2], 'B': [3, 4]})
    df2 = pd.DataFrame({'A': [5, 6], 'B': [7, 8]})  
    pd.concat([df1, df2], axis=1)   
    # 输出结果：
    #    A  B  A  B
    # 0  1  3  5  7
    # 1  2  4  6  8 
    # 纵向拼接
    pd.concat([df1, df2], axis=0)
    # 输出结果：
    #    A  B
    # 0  1  3
    # 1  2  4
    # 0  5  7
    # 1  6  8

    # merge 合并
    # 左连接
    df1 = pd.DataFrame({'key': ['A', 'B', 'C', 'D'], 'value': [1, 2, 3, 4]})
    df2 = pd.DataFrame({'key': ['B', 'D', 'E', 'F'], 'value': [5, 6, 7, 8]})    
    pd.merge(df1, df2, on='key', how='left')    
    # 右连接
    pd.merge(df1, df2, on='key', how='right')
    # 内连接
    pd.merge(df1, df2, on='key', how='inner')   
    # 外连接
    pd.merge(df1, df2, on='key', how='outer')
    ```
