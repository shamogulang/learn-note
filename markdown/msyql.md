# <center>mysql</center>

## 1、使用教程

假设你已经安装和启动了mysql服务器（我这里是windows版本的）了，那么可以使用下面的指令连接服务器：

```mysql
mysql -u root -p 然后回车：输入密码即可登录，下面是登录后的登录信息。

C:\Users\caraliu>mysql -u root -p
Enter password: ******
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 13
Server version: 8.0.19 MySQL Community Server - GPL

Copyright (c) 2000, 2020, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
```



如果你写了很多指令后，想要清除历史记录，可以输入指令： **system cls;**   注意后面的分号是要的。



退出可以exit，quit指令



服务器没有启动会报异常：

```mysql
ERROR 2003 (HY000): Can't connect to MySQL server on 'localhost' (10061)
```



密码错误会报异常：

```mysql
ERROR 1045 (28000): Access denied for user 'root'@'localhost' (using password: YES)
```



查看当前登录的用户：

```mysql
select user();  

返回结果：
+----------------+
| user()         |
+----------------+
| root@localhost |
+----------------+
1 row in set (0.00 sec)

```



查看版本和日期：

```mysql
select version(),current_date;

返回结果：
+-----------+--------------+
| version() | current_date |
+-----------+--------------+
| 8.0.19    | 2020-05-30   |
+-----------+--------------+
1 row in set (0.07 sec)
```



如果你敲了一半的指令不想要了，你可以在语句的最后加上 \c ,那么就可以回到重新输入语句的地方,比如下面的语句，后面加了 \c 回车后就又回到  mysql>  

```mysql
mysql> select * from user\c
mysql>
```



查看所有的数据库：

```mysql
mysql> show databases;

结果如下：
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
6 rows in set (0.06 sec)
```



创建数据库；

```msyql
create database caraliu;
```



操作对应的数据库(这个指令可以不要分号)：

```mysql
use caraliu
```



查看该库下的表格：(没创建表格，所以没有)

```mysql
mysql> show tables;
Empty set (0.00 sec)
```



创建表格：

```mysql
mysql>  CREATE TABLE pet (name VARCHAR(20), owner VARCHAR(20),
    ->        species VARCHAR(20), sex CHAR(1), birth DATE, death DATE);
Query OK, 0 rows affected (0.35 sec)


# 然后创建查看：
mysql> show tables;
+-------------------+
| Tables_in_caraliu |
+-------------------+
| pet               |
+-------------------+
1 row in set (0.00 sec)
```



查看表格的结构：

```mysql
# 第一种方式
mysql> desc pet;
+---------+-------------+------+-----+---------+-------+
| Field   | Type        | Null | Key | Default | Extra |
+---------+-------------+------+-----+---------+-------+
| name    | varchar(20) | YES  |     | NULL    |       |
| owner   | varchar(20) | YES  |     | NULL    |       |
| species | varchar(20) | YES  |     | NULL    |       |
| sex     | char(1)     | YES  |     | NULL    |       |
| birth   | date        | YES  |     | NULL    |       |
| death   | date        | YES  |     | NULL    |       |
+---------+-------------+------+-----+---------+-------+
6 rows in set (0.04 sec)

#第二种方式
mysql> describe pet;
+---------+-------------+------+-----+---------+-------+
| Field   | Type        | Null | Key | Default | Extra |
+---------+-------------+------+-----+---------+-------+
| name    | varchar(20) | YES  |     | NULL    |       |
| owner   | varchar(20) | YES  |     | NULL    |       |
| species | varchar(20) | YES  |     | NULL    |       |
| sex     | char(1)     | YES  |     | NULL    |       |
| birth   | date        | YES  |     | NULL    |       |
| death   | date        | YES  |     | NULL    |       |
+---------+-------------+------+-----+---------+-------+
6 rows in set (0.00 sec)
```



插入一行记录：

```mysql
INSERT INTO pet VALUES ('Puffball','Diane','hamster','f','1999-03-30',NULL);
```



查看记录：

```mysql
SELECT 需要的字段,*表示全部
   FROM 哪一张表
        WHERE 满足的条件;
 
# 查看全部：
mysql> select * from pet;
+----------+-------+---------+------+------------+-------+
| name     | owner | species | sex  | birth      | death |
+----------+-------+---------+------+------------+-------+
| Puffball | Diane | hamster | f    | 1999-03-30 | NULL  |
+----------+-------+---------+------+------------+-------+
1 row in set (0.00 sec)

# 查看特定的列：

mysql> select name, owner from pet;
+----------+-------+
| name     | owner |
+----------+-------+
| Puffball | Diane |
+----------+-------+
1 row in set (0.00 sec)

# 取别名，可以加 as 关键词，也可以直接一个空格
mysql> select name as petName, owner  petOwner from pet;
+----------+----------+
| petName  | petOwner |
+----------+----------+
| Puffball | Diane    |
+----------+----------+
1 row in set (0.00 sec)

# 查看特定名称的记录：

mysql> select name as petName, owner  petOwner from pet where pet.name='Puffball';
+----------+----------+
| petName  | petOwner |
+----------+----------+
| Puffball | Diane    |
+----------+----------+
1 row in set (0.00 sec)

# 多重条件，and 与 or 或，建议多重条件的时候，用（）来区分优先级，看起来比较清晰点
mysql> select name as petName, owner  petOwner from pet where pet.name='Puffball' and pet.owner='diane';
+----------+----------+
| petName  | petOwner |
+----------+----------+
| Puffball | Diane    |
+----------+----------+
1 row in set (0.00 sec)
```



插入多条记录：

```mysql
INSERT INTO pet VALUES 
('Puffball0','Diane','hamster','f','1996-03-30',NULL),('Puffball1','Caraliu','hamster','m','1995-03-30',NULL),('Puffball2','Diane','hamster','f','1989-05-30',NULL),('Puffball3','Jeffchan','hamster','f','1990-03-30',NULL);
```



按照时间排序：desc 降序， asc 升序

```mysql
mysql> select * from pet order by birth desc;

# 返回结果
+-----------+----------+---------+------+------------+-------+
| name      | owner    | species | sex  | birth      | death |
+-----------+----------+---------+------+------------+-------+
| Puffball  | Diane    | hamster | f    | 1999-03-30 | NULL  |
| Puffball0 | Diane    | hamster | f    | 1996-03-30 | NULL  |
| Puffball1 | Caraliu  | hamster | m    | 1995-03-30 | NULL  |
| Puffball3 | Jeffchan | hamster | f    | 1990-03-30 | NULL  |
| Puffball2 | Diane    | hamster | f    | 1989-05-30 | NULL  |
+-----------+----------+---------+------+------------+-------+
5 rows in set (0.05 sec)
```



查看多少年龄：

```mysql
这个 current_date 和 curdate() 方法一样都能返回当前日期

mysql> select name,birth, TimeStampDiff(year,birth,current_date) as age from pet;

# 返回结果
+-----------+------------+------+
| name      | birth      | age  |
+-----------+------------+------+
| Puffball  | 1999-03-30 |   21 |
| Puffball0 | 1996-03-30 |   24 |
| Puffball1 | 1995-03-30 |   25 |
| Puffball2 | 1989-05-30 |   31 |
| Puffball3 | 1990-03-30 |   30 |
+-----------+------------+------+
5 rows in set (0.00 sec)
```



查看已经还活着的宠物：

```mysql
mysql> select * from  pet where death is null;

# 返回结果
+-----------+----------+---------+------+------------+-------+
| name      | owner    | species | sex  | birth      | death |
+-----------+----------+---------+------+------------+-------+
| Puffball0 | Diane    | hamster | f    | 1996-03-30 | NULL  |
| Puffball1 | Caraliu  | hamster | m    | 1995-03-30 | NULL  |
| Puffball2 | Diane    | hamster | f    | 1989-05-30 | NULL  |
| Puffball3 | Jeffchan | hamster | f    | 1990-03-30 | NULL  |
+-----------+----------+---------+------+------------+-------+
4 rows in set (0.00 sec)

# is null 为空，is not null 不为空 注意这个不能使用=或者<>，因为null不是特定的值
```



查看下个月生日的宠物：

```mysql
mysql> select * from pet where month(birth)=month(date_add(curdate(),interval 1 day));

#返回结果
+-----------+----------+---------+------+------------+-------+
| name      | owner    | species | sex  | birth      | death |
+-----------+----------+---------+------+------------+-------+
| Puffball3 | Jeffchan | hamster | f    | 1994-06-06 | NULL  |
+-----------+----------+---------+------+------------+-------+
1 row in set (0.00 sec)

# 这里用到了几个函数： month(日期)，表示返回日期的月份；date_add(日期，interval 1 day）在日期的基础上，加一天的时间。

还可以通过下面的方式：

mysql> select * from pet where month(birth)=mod(month(current_date),12)+1;
+-----------+----------+---------+------+------------+-------+
| name      | owner    | species | sex  | birth      | death |
+-----------+----------+---------+------+------------+-------+
| Puffball3 | Jeffchan | hamster | f    | 1994-06-06 | NULL  |
+-----------+----------+---------+------+------------+-------+
1 row in set (0.00 sec)

mod(month(current_date),12)表示对当前月份在12内取模。其中也就是会返回0 - 11 范围的数字，因为是算下一个月，所以要加上1 
```



模式匹配：

```mysql
# 使用%来匹配任意数量的字符  cara%以cara开头的字符，%cara以cara结尾的字符，%cara%任意包含cara的字符
# _匹配任意一个字符
mysql> select * from pet where name like '%1';

# 返回结果
+-----------+---------+---------+------+------------+-------+
| name      | owner   | species | sex  | birth      | death |
+-----------+---------+---------+------+------------+-------+
| Puffball1 | Caraliu | hamster | m    | 1995-03-30 | NULL  |
+-----------+---------+---------+------+------------+-------+
1 row in set (0.00 sec)


mysql> select * from pet where name like '________';

# 返回结果，获取有八个字符的记录
+----------+-------+---------+------+------------+------------+
| name     | owner | species | sex  | birth      | death      |
+----------+-------+---------+------+------------+------------+
| Puffball | Diane | hamster | f    | 1999-03-30 | 2008-07-09 |
+----------+-------+---------+------+------------+------------+
1 row in set (0.00 sec)

# 注意这里都是用like关键字
```



查看各种性别的宠物的数量：

```mysql
mysql> select sex,count(*) from pet group by sex;

# 返回结果
+------+----------+
| sex  | count(*) |
+------+----------+
| f    |        4 |
| m    |        1 |
+------+----------+
2 rows in set (0.00 sec)

# 以group by 进行分组，然后算出每一组中的记录的数量
```



## 2、查询语法

基本环境：

```mysql
# 创建数据库
CREATE TABLE shop (    article INT UNSIGNED  DEFAULT '0000' NOT NULL,    dealer  CHAR(20)      DEFAULT ''     NOT NULL,    price   DECIMAL(16,2) DEFAULT '0.00' NOT NULL,    PRIMARY KEY(article, dealer));
# 插入表记录
INSERT INTO shop VALUES    (1,'A',3.45),(1,'B',3.99),(2,'A',10.99),(3,'B',1.45),    (3,'C',1.69),(3,'D',1.25),(4,'D',19.95);

mysql> select * from shop;

+---------+--------+-------+
| article | dealer | price |
+---------+--------+-------+
|       1 | A      |  3.45 |
|       1 | B      |  3.99 |
|       2 | A      | 10.99 |
|       3 | B      |  1.45 |
|       3 | C      |  1.69 |
|       3 | D      |  1.25 |
|       4 | D      | 19.95 |
+---------+--------+-------+
7 rows in set (0.00 sec)

```



查看article的最大值

```mysql
mysql> select max(article) from shop;

+--------------+
| max(article) |
+--------------+
|            4 |
+--------------+
1 row in set (0.02 sec)

# 查出article的最大值整个记录

# 第一种方式
mysql> select * from shop where article = (select max(article) from shop);
+---------+--------+-------+
| article | dealer | price |
+---------+--------+-------+
|       4 | D      | 19.95 |
+---------+--------+-------+
1 row in set (0.04 sec)

# 第二种方式：注意这种方式，只能返回一条记录，如果有多条的话，不建议使用
mysql> select * from shop order by article desc limit 1;
+---------+--------+-------+
| article | dealer | price |
+---------+--------+-------+
|       4 | D      | 19.95 |
+---------+--------+-------+
1 row in set (0.00 sec)

# 第三种方式，通过left join获取记录
mysql> select s1.* from shop s1 left join shop s2 on s1.article < s2.article where s2.article is null;
+---------+--------+-------+
| article | dealer | price |
+---------+--------+-------+
|       4 | D      | 19.95 |
+---------+--------+-------+
1 row in set (0.03 sec)
```



找出某一个组中的最大值：

```mysql
mysql> select article , max(price) from shop group by article;
+---------+------------+
| article | max(price) |
+---------+------------+
|       1 |       3.99 |
|       2 |      10.99 |
|       3 |       1.69 |
|       4 |      19.95 |
+---------+------------+
4 rows in set (0.00 sec)

# 获取其中的记录，第一种方式
mysql> select * from shop s1 join (select article,max(price) as price from shop group by article) as s2 on s1.article=s2.article and s1.price = s2.price;
+---------+--------+-------+---------+-------+
| article | dealer | price | article | price |
+---------+--------+-------+---------+-------+
|       1 | B      |  3.99 |       1 |  3.99 |
|       2 | A      | 10.99 |       2 | 10.99 |
|       3 | C      |  1.69 |       3 |  1.69 |
|       4 | D      | 19.95 |       4 | 19.95 |
+---------+--------+-------+---------+-------+
4 rows in set (0.00 sec)

# 第二种方式
mysql> select  s1.* from shop s1 left join shop s2 on s1.article = s2.article and s1.price < s2.price where s2.article is null ;
+---------+--------+-------+
| article | dealer | price |
+---------+--------+-------+
|       1 | B      |  3.99 |
|       2 | A      | 10.99 |
|       3 | C      |  1.69 |
|       4 | D      | 19.95 |
+---------+--------+-------+
4 rows in set (0.00 sec)
```



使用auto_increment,自增，当插入记录的时候，会主动生产一个自增的值：

```mysql
# 创建表
mysql> CREATE TABLE animals (     id MEDIUMINT NOT NULL AUTO_INCREMENT,     name CHAR(30) NOT NULL,     PRIMARY KEY (id) );
Query OK, 0 rows affected (1.40 sec)

# 插入记录，会主动生成id
mysql> INSERT INTO animals (name) VALUES    ('dog'),('cat'),('penguin'),    ('lax'),('whale'),('ostrich');
Query OK, 6 rows affected (0.04 sec)
Records: 6  Duplicates: 0  Warnings: 0

mysql> select * from animals;

# 主动生成id
+----+---------+
| id | name    |
+----+---------+
|  1 | dog     |
|  2 | cat     |
|  3 | penguin |
|  4 | lax     |
|  5 | whale   |
|  6 | ostrich |
+----+---------+
6 rows in set (0.00 sec)
```

