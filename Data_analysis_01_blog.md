# 数据分析由浅入深实战

[toc]

# 基础信息（数据层面）

面对一张数据表，我们需要知道的信息。

1. **表结构**
2. **更新策略（增量或全量）**
3. **主键**
4. **字段枚举值，空值率**
5. **字段极值**

## 查询技巧

1. 表结构

   `describe 库名.表名;`

2. 更新策略（增量或全量）

   - 看每天的数据量变化，如果每天都差不多，是全量，如果变化很大，是增量。
   - 单独看一个客户号，如果不是每个dt里都有他，那这张表肯定不是全量，因为全量不会漏客户。

   多张表pin

   ```sql
   select '1' as nm,dt,count(*) from 库名.表名 group by dt order by dt desc limit 30 union all
   select '2' as nm,dt,count(*) from 库名.表名 group by dt order by dt desc limit 30 ;
   ```

   ```sql
   select dt from 库名.表名 where cust_id = '' group by dt  order by dt desc;
   ```

   

3. 主键

   两种方式二选一即可

   ```sql
   -- 方式1：
   select 主键字段,dt,count(*)  from 库名.表名 where dt = '20221010' group by 1,2 having count(*) >1;
   ```

   ```sql
   -- 方式2：
   select
   '贷款催收' as name
   , sum(case when coalesce(主键字段, '')='' then 1 else 0 end) as null_count --主键为空校验
   ,count(distinct 主键字段) key --重复性校验
   ,count(1) as row_num --总行数校验
   from 库名.表名
   where dt = '20221010';
   ```

   

4. 字段枚举值，空值率

   - 枚举值 - 分布情况

   ```sql
   select '字段1' as name, count(distinct 字段1) as js from  库名.表名 WHERE DT = '20221010' union all
   select '字段2' as name, count(distinct 字段2) as js from  库名.表名 WHERE DT = '20221010' union all
   select '字段3' as name, count(distinct 字段3) as js from  库名.表名 WHERE DT = '20221010';
   ```

   - 枚举值 - 单个字段这样看：

   ```sql
   select 字段1, count(*) from 表名.库名 where dt='20221010' group by 字段1;
   ```

   这样可以看到这个字段有哪些取值，以及每个取值的数据量是多少

   - 枚举值 - 多个字段这样看：(类似于，检查脏数据的办法，按长度降序排序检查)

   ```sql
   select  '字段1'as name, cast(字段1 as string) as value from 库名.表名 where dt = '20221010'  group by 字段1 order by length(字段1) desc limit 50 union all
   select  '字段2'as name, cast(字段2 as string) as value from 库名.表名 where dt = '20221010'  group by 字段2 order by length(字段2) desc limit 50;
   ```

   - 空值 - 多个字段这样看：

   ```sql
   -- 方式1：
   select '字段1'as nm, count(*) as js from 库名.表名 where dt = '20221010' and nvl(字段1,'')='' union all
   select '字段2'as nm, count(*) as js from 库名.表名 where dt = '20221010' and nvl(字段2,'')='' ;
   ```

   ```sql
   -- 方式2：
   select 
    sum(case when nvl(字段1,'')='' then 1 else 0 end ) / count(*) as 字段1
   ,sum(case when nvl(字段2,'')='' then 1 else 0 end ) / count(*) as 字段2
   ,sum(case when nvl(字段3,'')='' then 1 else 0 end ) / count(*) as 字段3
   ,sum(case when nvl(字段4,'')='' then 1 else 0 end ) / count(*) as 字段4
   from 库名.表名 where dt = '20221010' ;
   ```

   

5. 字段极值

   - 检查每个字段值的最大长度值，是否超过该字段规定的长度值，可以将多个字段union all，同时查看每个字段值的长度

   ```sql
   select '字段1'as name, 字段1 as value ,ln from (select 字段1,length(字段1) as ln, row_number() over (partition by 字段1 order by length(字段1) desc ) as rn from 库名.表名 where dt = '20221010' ) t where t.rn = 1 limit 10 union all
   select '字段2'as name, 字段2 as value ,ln from (select 字段2,length(字段2) as ln, row_number() over (partition by 字段2 order by length(字段2) desc ) as rn from 库名.表名 where dt = '20221010' ) t where t.rn = 1 limit 10 ;
   ```



# 进阶查询（业务层面）

面对差异数据时，需一步步排查原因，定位问题，下面列举几个排查角度供参考：

1. 分析**关联字段或过滤字段**的限制，对查询结果的数据量是否有影响时，可进行多次比较，将每次条件的变化写成一个子查询，查看每段子查询的数据量。
2. 检查过滤条件：
   - **删除标识字段**
   - **dt限制**（增量用dt<='20221010'，全量用dt='20221010'）
   - **各种时间，各种状态限制**（审批状态是否是通过，贷款余额是否大于0，核销状态是否是未核销）
3. 检查关联条件：
   - 用证件号关联，用流水号关联，用合同号关联
   - **inner join**少数时，看两张表里的关联字段的枚举值是否一样多，是否有空值，是否存在一张表的字段值较少，导致有部分值关联不上
4. 用四五个反例找共性，找到后需验证剩余部分数据，是否也具有该共性。
5. **row_number()随机取值**
6. 字段与字段间的关系判断
    - 一对一
    - 一对多
7. 表与表的关系判断
   - 数据量是否一致
   - 是否是包含关系
   - 是否是交集关系，有共有的数据，也有各自独有的数据
   - 换表时，说明为什么用A表不用B表



## 查询技巧

1. 查看每段子查询的数据量

   - 纵向展示：

   ```sql
   select '1' as nm, count(*) from ( 子查询1 ) t1   union all
   select '2' as nm, count(*) from ( 子查询2 ) t1   ;
   ```

   - 横向展示：

   ```sql
   select
    t1.*
   ,t2.*
   ,t3.*
   ,t4.*
   from      (select '1' as nm, count(*) as cnt from ( 子查询1 ) t ) t1
   left join (select '2' as nm, count(*) as cnt from ( 子查询2 ) t ) t2 on 1=1
   left join (select '3' as nm, count(*) as cnt from ( 子查询3 ) t ) t3 on 1=1
   left join (select '4' as nm, count(*) as cnt from ( 子查询4 ) t ) t4 on 1=1;
   ```

2. row_number()随机取值

   ```sql
   select 分组字段,排序字段,count(1) from 库名.表名 where DT = '20221010'  group by 分组字段,排序字段 order by count(1) desc limit 5;
   ```

   count(1)>1意味着该表存在分组字段,排序字段相同的情况下，其余字段有不同值，如果按排序字段取最新一条时，依然会随机取其中一条。

   

3. 两字段比较，字段与字段间的关系判断

   ```sql
   -- 方式1：
   select 字段1, count(distinct 字段2)
   from 库名.表名
   group by 字段1
   having count(distinct 字段2)>1;
   ```

   ```sql
   -- 方式2：
   select t.字段1,t.字段2 from (
   select 字段1,字段2,row_number() over (partition by 字段1 order by 字段2 desc ) as rn
    from 库名.表名
   )t
   where t.rn >1;
   ```

   这样就说明字段1与字段2是一对多的关系
   扩展：多对多关系，上述SQL中字段1与字段2位置互换后，查询有值，就说明两者是多对多关系

4. 两表比较，表与表的关系判断

   - 两表比较，找出表1中独有的cust_id

   ```sql
   -- 方式1：
   select t1.cust_id,* from 表1 t1 where t1.dt = '20221010'
   and t1.cust_id is not null
   and
   not exists (select 1 from 表2 t2 where t2.dt = '20221010'
   and t1.cust_id = t2.cust_id
   );
   ```

   ```sql
   -- 方式2：
   select t1.cust_id  from 表1 t1 where t1.dt = '20221010'
   left join 表2 t2
   on t1.cust_id = t2.cust_id
   where t2.cust_id is null;
   ```

   ```sql
   -- 方式3:
   select t1.cust_id  from 表1 t1 where t1.dt = '20221010'
   and t1.cust_id not in (select distinct t2.cust_id  from 表2 t2 );
   ```

   - 如果表1，表2 一模一样，那下面ABC查出来的数是一样的

   反过来，如果表1，表2所有字段union后，ABC查出来的数一样，表明表1，表2是一样的表

   ```sql
   select count(*) from (select distinct cust_id,其余字段 from 表1 t1 where t1.dt = '20221010') A;
   select count(*) from (select distinct cust_id,其余字段 from 表2 t1 where t1.dt = '20221010') B;
   select count(*) from (
   select distinct cust_id,其余字段 from 表1 t1 where t1.dt = '20221010'
   union
   select distinct cust_id,其余字段 from 表2 t1 where t1.dt = '20221010'
   ) C;
   ```

   找出两表差异数据量，各种独有的数据量以及公共部分的数据量等等

   ```sql
   select '表1独有的数据量（主键）' as nm,count(1) as js from (
   select distinct 主键 from 表1 where 主键 not in (select distinct 主键 from 表2 )
   )t1
   union all
   select '表2独有的数据量（主键）' as nm,count(1) as js from (
   select distinct 主键 from 表2 where 主键 not in (select distinct 主键 from 表1 )
   )t1
   union all
   select '表1表2交集数据量' as nm,count(1) as js from (
   select distinct t1.主键 from 表1 t1
   inner join 表2 t2
   on t1.主键=t2.主键
   )t1;
   ```



# 结果汇报（业务层面）

业务老师验收时，准备物料不仅限于白皮书，需求文档，码值表，手动维护的产品表等，类似于表层级关系图，也可以留个备份。

1. 白皮书
2. 需求文档
3. 码值表
4. 产品表
5. 表层级关系图



## 汇报技巧

1. 记住高频使用的几张表（产品表，各个表的主表等），做到看到源表可以回忆起源表存放了哪些产品的客户数据。

2. 给业务或开发老师展示查询结果时，开发老师关注问题定位到哪个脚本，最好哪一段逻辑有问题，业务老师关注哪一个时间段的哪些产品的客户有问题。

> 在定位问题时，尽可能找出**时间，系统分区，产品号**。也就是找出哪张表，哪一天dt，哪一个产品，哪一个分区。列举反例时，拿到最细粒度的编号（客户级就是客户号，明细级一般是借据编号、合同编号等），可以先取消dt限制，查看该客户的其他时间字段的区间范围（最早时间，最近时间），其他时间字段举例：生成时间，创建时间，更新时间，删除时间等等。





