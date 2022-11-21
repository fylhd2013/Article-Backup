[toc]

# 测试过程中遇到的高频查询问题：
1. 表是增量表还是全量表
2. 主键问题
3. 两个字段的关系，是一对多，还是一对一等等

验证方式：
## 如何判断一张表是增量表还是全量表
1. 看每天的数据量变化，如果每天都差不多，是全量，如果变化很大，是增量。
验证语句：`select dt,count(1) from 库名.表名 group by dt order by dt desc;`
2. 单独看一个客户号，如果不是每个dt里都有他，那这张表肯定不是全量，因为全量不会漏客户的。
验证语句：`select dt from 库名.表名 where cust_id = '' group by dt order by dt desc;`


## 检查表的主键是否正常
 两种方式二选一即可
1. 方式1：
```sql
SELECT CUST_ID,DT,COUNT(*)  FROM 库名.表名 WHERE DT = '20221020' GROUP BY 1,2 HAVING COUNT(*) >1;
```

2. 方式2：
```sql
SELECT
'表名' AS NAME
, SUM(CASE WHEN COALESCE(CUST_ID, '')='' THEN 1 ELSE 0 END) AS NULL_COUNT --主键为空校验
,COUNT(DISTINCT CUST_ID) KEY --重复性校验
,COUNT(1) AS ROW_NUM --总行数校验
FROM 库名.表名
WHERE DT = '20221020'
;
```


## 判断两个字段间一对多或多对一或一对一的关系
两个字段间的关系
**测试id与name的一对多关系**
1. 方式1：
以下SQL会报错，报错原因 GROUP BY
```sql
SELECT ID,`NAME`,COUNT(*)
FROM 库名.表名
GROUP BY ID
HAVING COUNT(`NAME`)>1;
```
修改后：
```sql
SELECT ID, COUNT(DISTINCT `NAME`)
FROM 库名.表名
GROUP BY ID
HAVING COUNT(DISTINCT `NAME`)>1;
```

2. 方式2：
row_number()

```sql
SELECT T.ID,T.NAME FROM (
SELECT ID
,NAME
,ROW_NUMBER() OVER (PARTITION BY ID ORDER BY NAME DESC ) AS RN 
 FROM 库名.表名
 WHERE  DT = '20221020'
)T
WHERE T.RN >2
;
```



这样就说明id与name是一对多的关系

- 类似地可以验证，同一客户号有是否有多条借据编号，多条贷款编号等等。
- **扩展：多对多关系，上述SQL中id与name位置互换后，查询有值，就说明两者是多对多关系**

