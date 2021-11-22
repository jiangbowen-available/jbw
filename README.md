# jbw
### 创建数据库
create database if not exists myhive;
create database myhive2 location '/myhive2';
### 修改数据库 
alter database myhive2 set 
### 查看数据库信息
desc  database  myhive2;
desc database extended  myhive2; # 详细
### 删库跑路
drop  database  myhive2;
### 使用数据库
use myhive; -- 使用myhive数据
### 创建表
create table stu(id int,name string);
* eg1：
__create table if not exists stu2(id int ,name string) row format delimited fields terminated by '\t' stored as textfile location '/user/stu2';
row format delimited fields terminated by '\t'  指定字段分隔符，默认分隔符为 '\001'   
stored as 指定存储格式   
location 指定存储位置__
* eg2
__create table stu3 as select * from stu2;根据查询结果创建表__
* eg3
__create table stu4 like stu2;根据已经存在的表结构创建表__



### 插入元素
insert into stu values (1,"zhangsan");
insert into stu values (1,"zhangsan"),(2,"lisi")；
### 追加操作
load data local inpath '/export/servers/hivedatas/student.csv' into table student;
### 覆盖操作
load data local inpath '/export/servers/hivedatas/student.csv' overwrite into table student;
### 加载数据到指定分区
load data inpath '/hivedatas/techer.csv' into table techer partition(cur_date=20201210);
**注意**：   
*1.使用 load data local 表示从本地文件系统加载，文件会拷贝到hdfs上   
*2.使用 load data 表示从hdfs文件系统加载，文件会直接移动到hive相关目录下，注意不是拷贝过去，因为hive认为hdfs文件已经有副本了，没必要再次拷贝了   
*3.如果表是分区表，load 时不指定分区会报错   
*4.如果加载相同文件名的文件，会被自动重命名

## 分区分桶

分区就是分文件夹，分桶就是分文件
### 分区表
create table score(s_id string, s_score int) partitioned by (month string);
create table score2 (s_id string, s_score int) partitioned by (year string,month string,day string); 创建一个表带多个分区
**注意：   
hive表创建的时候可以用 location 指定一个文件或者文件夹，当指定文件夹时，hive会加载文件夹下的所有文件，当表中无分区时，这个文件夹下不能再有文件夹，否则报错   **

load data local inpath '/export/servers/hivedatas/score.csv' into table score partition (month='201806');加载数据到一个分区的表中
show partitions score;查看分区
alter table score add partition(month='201805'); 添加一个分区
alter table score add partition(month='201804') partition(month = '201803');同时添加多个分区
alter table score drop partition(month = '201806'); 删除分区

### 分桶表
set hive.enforce.bucketing=true; 开启分桶表
set mapreduce.job.reduces=3;设置reduce的个数
create table course (c_id string,c_name string) clustered by(c_id) into 3 buckets;创建桶表
insert overwrite table course select * from course_common cluster by(c_id); 通过insert  overwrite给桶表中加载数据最后指定桶字段


### 修改表名称
alter table old_table_name rename to new_table_name;
### 添加列
alter table score5 add columns (mycol string, mysco string);
### 更新列
alter table score5 change column mysco mysconew int;
### 删除表操作
drop table score5;
### 清空表
truncate table score6;
 **注意：truncate 和 drop：      
如果 hdfs 开启了回收站，drop 删除的表数据是可以从回收站恢复的，表结构恢复不了，需要自己重新创建；truncate 清空的表是不进回收站的，所以无法恢复truncate清空的表   
所以 truncate 一定慎用，一旦清空将无力回天**

### 直接向分区表中插入数据
insert into table score partition(month ='201807') values ('001','002','100');
### 通过load方式加载数据
load data local inpath '/export/servers/hivedatas/score.csv' overwrite into table score partition(month='201806');
### 通过查询方式加载数据
insert overwrite table score2 partition(month = '201806') select s_id,c_id,s_score from score1;
### 查询语句中创建表并加载数据
create table score2 as select * from score1;
### 在创建表时通过location指定加载数据的路径
create external table score6 (s_id string,c_id string,s_score int) row format delimited fields terminated by ',' location '/myscore';

## export导出与import 导入
create table techer2 like techer; --依据已有表结构创建表
### 内部表
export table techer to '/export/techer';
import table techer2 from '/export/techer';
### insert导出
insert overwrite local directory '/export/servers/exporthive' select * from score;将查询的结果导出到本地
insert overwrite local directory '/export/servers/exporthive' row format delimited fields terminated by '\t' collection items terminated by '#' select * from student;将查询的结果格式化导出到本地
insert overwrite directory '/export/servers/exporthive' row format delimited fields terminated by '\t' collection items terminated by '#' select * from score;将查询的结果导出到HDFS上(没有local)
### Hadoop命令导出到本地
dfs -get /export/servers/exporthive/000000_0 /export/servers/exporthive/local.txt;
### hive shell 命令导出
基本语法：
hive -f/-e 执行语句或者脚本 > file
hive -e "select * from myhive.score;" > /export/servers/exporthive/score.txt
hive -f export.sh > /export/servers/exporthive/score.txt
### export导出到HDFS上
export table score to '/export/exporthive/score';

## hive的DQL查询语法
### 单表查询
**SELECT [ALL | DISTINCT] select_expr, select_expr, ...
FROM table_reference
[WHERE where_condition]
[GROUP BY col_list [HAVING condition]]
[CLUSTER BY col_list
| [DISTRIBUTE BY col_list] [SORT BY| ORDER BY col_list]
]
[LIMIT number]**

select * from score where s_score < 60;
select s_id ,avg(s_score) from score group by s_id;
select s_id ,avg(s_score) avgscore from score group by s_id having avgscore > 85;
**如果使用 group by 分组，则 select 后面只能写分组的字段或者聚合函数   
where和having区别：   
1 having是在 group by 分完组之后再对数据进行筛选，所以having 要筛选的字段只能是分组字段或者聚合函数   
2 where 是从数据表中的字段直接进行的筛选的，所以不能跟在gruop by后面，也不能使用聚合函数**

### join
select * from techer t [inner] join course c on t.t_id = c.t_id; inner 可省略 只有进行连接的两个表中都存在与连接条件相匹配的数据才会被保留下来
select * from techer t left join course c on t.t_id = c.t_id; -- outer可省略
select * from techer t right join course c on t.t_id = c.t_id;
SELECT * FROM techer t FULL JOIN course c ON t.t_id = c.t_id ;FULL OUTER JOIN 满外(全外)连接: 将会返回所有表中符合条件的所有记录。如果任一表的指定字段没有符合条件的值的话，那么就使用NULL值替代。
**注：
**1. hive2版本已经支持不等值连接，就是 join on条件后面可以使用大于小于符号了;并且也支持 join on 条件后跟or (早前版本 on 后只支持 = 和 and，不支持 \> \< 和 or)     
2.如hive执行引擎使用MapReduce，一个join就会启动一个job，一条sql语句中如有多个join，则会启动多个job
注意：表之间用逗号(,)连接和 inner join 是一样的   
select * from table_a,table_b where table_a.id=table_b.id;      
它们的执行效率没有区别，只是书写方式不同，用逗号是sql 89标准，join 是sql 92标准。用逗号连接后面过滤条件用 where ，用 join 连接后面过滤条件是 on。
**

### order by 排序
ASC（ascend）: 升序（默认） DESC（descend）: 降序
SELECT * FROM student s LEFT JOIN score sco ON s.s_id = sco.s_id ORDER BY sco.s_score DESC;
**注意：order by 是全局排序，所以最后只有一个reduce，也就是在一个节点执行，如果数据量太大，就会耗费较长时间

### sort by 局部排序
每个MapReduce内部进行排序，对全局结果集来说不是排序
set mapreduce.job.reduces=3;设置reduce个数
set mapreduce.job.reduces;查看设置reduce个数
select * from score sort by s_score;查询成绩按照成绩降序排列
insert overwrite local directory '/export/servers/hivedatas/sort' select * from score sort by s_score;将查询结果导入到文件中（按照成绩降序排列）

### distribute by  分区排序
distribute by：类似MR中partition，进行分区，结合sort by使用
set mapreduce.job.reduces=7;设置reduce的个数，将我们对应的s_id划分到对应的reduce当中去
select * from score distribute by s_id sort by s_score;通过distribute by 进行数据的分区
**注意：Hive要求 distribute by 语句要写在 sort by 语句之前

### cluster by
当distribute by和sort by字段相同时，可以使用cluster by方式.
cluster by除了具有distribute by的功能外还兼具sort by的功能。但是排序只能是正序排序，不能指定排序规则为ASC或者DESC。
select * from score cluster by s_id;
等价于
select * from score distribute by s_id sort by s_id;

### 聚合函数
聚合操作时要注意null值   
* count(*) 包含null值，统计所有行数   
* count(id) 不包含null值   
* min 求最小值是不包含null，除非所有值都是null   
* avg 求平均值也是不包含null

* 非空集合总体变量函数: var_pop
语法: var_pop(col)
返回值: double
说明: 统计结果集中col非空集合的总体变量（忽略null）

* 非空集合样本变量函数: var_samp
语法: var_samp (col)
返回值: double
说明: 统计结果集中col非空集合的样本变量（忽略null）

* 总体标准偏离函数: stddev_pop
语法: stddev_pop(col)
返回值: double
说明: 该函数计算总体标准偏离，并返回总体变量的平方根，其返回值与VAR_POP函数的平方根相同

* 中位数函数: percentile
语法: percentile(BIGINT col, p)
返回值: double
说明: 求准确的第pth个百分位数，p必须介于0和1之间，但是col字段目前只支持整数，不支持浮点数类型

### 关系运算
*支持：等值(=)、不等值(!= 或 <>)、小于(<)、小于等于(<=)、大于(>)、大于等于(>=)、空值判断(is null)、非空判断(is not null)

* LIKE比较: LIKE
语法: A LIKE B
操作类型: strings
描述: 如果字符串A或者字符串B为NULL，则返回NULL；如果字符串A符合表达式B 的正则语法，则为TRUE；否则为FALSE。B中字符”_”表示任意单个字符，而字符”%”表示任意数量的字符
* JAVA的LIKE操作: RLIKE
语法: A RLIKE B
操作类型: strings
描述: 如果字符串A或者字符串B为NULL，则返回NULL；如果字符串A符合JAVA正则表达式B的正则语法，则为TRUE；否则为FALSE。
* REGEXP操作: REGEXP
语法: A REGEXP B
操作类型: strings
描述: 功能与RLIKE相同
示例：select 1 from tableName where 'footbar' REGEXP '^f.*r$';

### 数学运算
支持所有数值类型：加(+)、减(-)、乘(*)、除(/)、取余(%)、位与(&)、位或(|)、位异或(^)、位取反(~)

### 逻辑运算
支持：逻辑与(and)、逻辑或(or)、逻辑非(not)

### 数值运算

* 取整函数: round 
语法: round(double a)
返回值: BIGINT
说明: 返回double类型的整数值部分 （遵循四舍五入）
示例：select round(3.1415926) from tableName;
结果：3
* 指定精度取整函数: round
语法: round(double a, int d)
返回值: DOUBLE
说明: 返回指定精度d的double类型
hive> select round(3.1415926,4) from tableName;
* 向下取整函数: floor
语法: floor(double a)
返回值: BIGINT
说明: 返回等于或者小于该double变量的最大的整数
hive> select floor(3.641) from tableName;
* 向上取整函数: ceil
语法: ceil(double a)
返回值: BIGINT
说明: 返回等于或者大于该double变量的最小的整数
hive> select ceil(3.1415926) from tableName;
* 取随机数函数: rand
语法: rand(),rand(int seed)
返回值: double
说明: 返回一个0到1范围内的随机数。如果指定种子seed，则会等到一个稳定的随机数序列
hive> select rand() from tableName; -- 每次执行此语句得到的结果都不同
hive> select rand(100) ; -- 只要指定种子，每次执行此语句得到的结果一样的
* 自然指数函数: exp
语法: exp(double a)
返回值: double
说明: 返回自然对数e的a次方
hive> select exp(2) ;
* 以10为底对数函数: log10
语法: log10(double a)
返回值: double
说明: 返回以10为底的a的对数
hive> select log10(100) ;
