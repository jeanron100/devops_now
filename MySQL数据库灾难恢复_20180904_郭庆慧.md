#MySQL数据库灾难恢复
@(MySQL)[灾难恢复]

作者：郭庆慧

**灾难恢复**是意外情况造成数据丢失时，通过极端手段进行恢复的一种方式。
**灾难恢复**说明备份已经无效，或者没有备份，所以一定要做好备份，不要用到灾难恢复。


[toc]

**本文模拟的灾难环境**
- 1、实例无法正常启动，数据文件都还存在。
- 2、表delete部分数据或者被truncate
- 3、表被drop或者database 被drop
**相关工具介绍**
- 1、MySQL数据库实例。
- 2、percona-data-recovery-tool-for-innodb
  **[源码下载地址]**
  https://launchpad.net/percona-data-recovery-tool-for-innodb/trunk/release-0.5/+download/percona-data-recovery-tool-for-innodb-0.5.tar.gz
**[相关文档]**
https://github.com/percona/innodb-data-recovery-tool-docs
- 3 [undrop-for-innodb](https://github.com/twindb/undrop-for-innodb)
**开源地址**：https://github.com/twindb/undrop-for-innodb
##1实例无法正常启动
*环境*：MySQL 5.7.21
模拟故障：实例无法启动，数据文件还在。
恢复前的表结构和数据
```sql

mysql> show create table t1 \G;
*************************** 1. row ***************************
       Table: t1
Create Table: CREATE TABLE `t1` (
  `id` int(11) NOT NULL,
  `name` varchar(20) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1
1 row in set (0.00 sec)

ERROR: 
No query specified

mysql> select * from t1;
+----+------+
| id | name |
+----+------+
|  1 | a    |
|  4 | c    |
|  7 | b    |
| 11 | e    |
| 20 | d    |
| 30 | b    |
+----+------+
6 rows in set (0.00 sec)
```

###1.1**约束条件**
开启innodb_file_per_table=on。
> 该参数表示表有独立的表空间，5.6.6默认开启
###1.2恢复*.frm文件
- 1、新的实例
- 2、创建新的数据库
```sql
mysql> create database db1;
mysql> use db1;
```
- 3、创建同名表
```sql
mysql> create table t1(col1 int);
```
- 4、复制*.frm 覆盖当前实例的。t1.frm文件
- 5、添加参数innodb_force_recovery=6到my.cnf文件
> innodb_force_recovery默认值是0，6代表实例启动不校验idb文件

- 6、重启实例
- 7、从错误日志找出表的字段个数
InnoDB: Table db1/t1 contains 1 user defined columns in InnoDB, **but 2 columns** in MySQL
###1.3恢复表结构
重复1.2的操作
**注意**：需要注释掉#innodb_force_recovery=6
**第二次建表语句**
create table t1(col1 int,col2 int);
**获得建表语句**
```sql
Create Table: CREATE TABLE `t1` (
  `id` int(11) NOT NULL,
  `name` varchar(20) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1
1 row in set (0.00 sec)
```
###1.4创建新表
- 1、注释参数)remove innodb_force_recovery=6 
- 2、重启实例
- 3、删除旧表
- 4、创建新表指定row_format=compact
```sql
CREATE TABLE `t1` (
  `id` int(11) NOT NULL,
  `name` varchar(20) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB  row_format=compact

```
###1.5恢复*.ibd文件
- 1、alter table t1 discard tablespace;
- 2、覆盖idb文件，重启实例
- 3、alter table t1 import tablespace;
#2、表truancte或者delete
>该情况下mysql被删除的数据页会很快被覆盖，所以先要把表的数据文件拷贝出来，保护现场，此种情况无法保证完全恢复，如果删除的数据被覆盖，只能恢复部分数据。
##2.1安装恢复工具
```bash

tar -zxf percona-data-recovery-tool-for-innodb-0.5.tar.gz 
cd percona-data-recovery-tool-for-innodb-0.5/mysql-source/
./configure 
cd ..
make
```

##2.2恢复数据

```bash

# cd /data/
# mkdir -p /data/mysql/recover
# cd /data/mysql/recover
# cd db3/
# cp t.* /data/mysql/recover

cd /usr/local/
cd percona-data-recovery-tool/

# ./page_parser -5 -f /data/db3/t.ibd
# cd pages-1517613235/

# ./create_defs.pl --user root --password mysql --db db3 --table t1 > include/table_defs.h
# cat include/table_defs.h

[root@node1 percona-data-recovery-tool]# make

[root@node1 percona-data-recovery-tool]# make

# ./constraints_parser -5 -f pages-1517613235/FIL_PAGE_INDEX/0-42/ > /data/mysql/recover/t1.sql

# cat /data/mysql/recover/t.sql | wc -l
# head -10 /data/mysql/recover/t1.sql
```



