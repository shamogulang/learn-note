<center><h1>mysql多版本并发控制（mvcc）</h1></center>


我使用的mysql版本的8.0.19，mysql的innodb引擎是一个多版本并发控制的引擎，它会保存修改的记录的旧版本的信息，来支持事务的特征，比如回滚和一致性，这个特性只在可重复读和读已提交隔离级别起作用。

### 1、隐藏字段

innodb的引擎会在数据库表中为每一条记录添加三个隐藏的字段。（DB_TRX_ID，`DB_ROLL_PTR` ，`DB_ROW_ID` ）

1、占6个字节的字段DB_TRX_ID记录该记录操作时刻的事务ID(更新，删除或者是插入)

2、占7个字节的字段`DB_ROLL_PTR` 是回滚的指针，指向undo log。用于回滚到上一次的修改位置。

3、占6字节的字段`DB_ROW_ID` 是自增的ID，这个不是必须的，如果你指定了主键，这个字段就没有，如果没有指定，在生成聚集索引的时候，会要用到这个字段，就会自动加这个隐藏列。

同时每一条记录的都有一个标记位，用于标记这条记录是否被删除了。



创建两个表，一个是有主键的animals1，一个是没有主键的pet。

```sql
CREATE TABLE `animals1` (
  `id` bigint NOT NULL COMMENT '动物ID',
  `name` char(30) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL COMMENT '动物名称',
  `age` int NOT NULL COMMENT '年龄',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci COMMENT='动物表'

CREATE TABLE `pet` (
  `name` varchar(20) DEFAULT NULL,
  `owner` varchar(20) DEFAULT NULL,
  `species` varchar(20) DEFAULT NULL,
  `sex` char(1) DEFAULT NULL,
  `birth` date DEFAULT NULL,
  `death` date DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
```

如果要查看这个表结构的隐藏列的话，需要使用 SHOW EXTENDED COLUMNs 命令

SHOW EXTENDED COLUMNs from animals1

![showextend](D:\jeffchan\markdown\image\showextend.png)

可以发现，这个表里面增加了两个隐藏字段：DB_TRX_ID，`DB_ROLL_PTR` ，因为它自身有个

主键，所以就没有`DB_ROW_ID` 这个字段。



SHOW EXTENDED COLUMNs from pet

![showextend1](D:\jeffchan\markdown\image\showextend1.png)

可以发现，这个除了DB_TRX_ID，`DB_ROLL_PTR` 这两个隐藏字段外，因为本身没有指定主键，所以

就加了`DB_ROW_ID` 这个隐藏列。注意DB_TRX_ID这个id是全局递增的。



### 2、undo log

undo log分成两种，一种是insert的undo log，一种是update的undo log，这里的update是包含delete的。insert的undo log。insert undo log仅仅是在事务回滚的时候才用到，只要事务一提交，就会被删除。update undo log除了用于事务回滚之外还用在一致性读的场景，它仅仅在没有事务的场景下才会被删除。



#### 2.1、可重复读

在读取数据库的记录的时候，会使用read view快照来过滤需要返回的数据。

##### 2.11 read view的结构（截取了部分源码）：

```sql
	/** The read should not see any transaction with trx id >= this
	value. In other words, this is the "high water mark". */
	trx_id_t	m_low_limit_id;

	/** The read should see all trx ids which are strictly
	smaller (<) than this value.  In other words, this is the
	low water mark". */
	trx_id_t	m_up_limit_id;

	/** trx id of creating transaction, set to TRX_ID_MAX for free
	views. */
	trx_id_t	m_creator_trx_id;

	/** Set of RW transactions that was active when this snapshot
	was taken */
	ids_t		m_ids;

	/** The view does not need to see the undo logs for transactions
	whose transaction number is strictly smaller (<) than this value:
	they can be removed in purge if not needed by other views */
	trx_id_t	m_low_limit_no;

	/** AC-NL-RO transaction view that has been "closed". */
	bool		m_closed;
...
```



##### 2.12、是否允许看见记录的源码：

```java
{
		ut_ad(id > 0);
        // 如果记录的事务ID比m_up_limit_id小，或者等于m_creator_trx_id，可以看见
		if (id < m_up_limit_id || id == m_creator_trx_id) {

			return(true);
		}

		check_trx_id_sanity(id, name);
        // 如果记录的事务ID大于等于m_low_limit_id，不可以看见。
		if (id >= m_low_limit_id) {

			return(false);

		} else if (m_ids.empty()) {
            // 这个就表示获取记录的事务ID不在m_ids集合中。
			return(true);
		}

		const ids_t::value_type*	p = m_ids.data();
        // 如果记录在活跃Id m_ids集合中，那么不可见。
		return(!std::binary_search(p, p + m_ids.size(), id));
	}
```



m_low_limit_id： 高水位线，取值未下一个待分配的事务ID，当事务ID大于等于这个值的时候，表示再创建该read view时，对应的事务ID还有被分配，所以当前read view对于该记录不可见。

m_up_limit_id： 低水位线，当事务ID小于这个值的时候，表示在创建该read view前，记录的事务已经提交，该记录可见，取值为还没提交的事务ID中最小的一个。

m_creator_trx_id： 当前session的对应的事务ID，如果当前记录的事务id和这个一样，可以看见。

m_ids： 当read view快照生成的那一刻，活跃的事务ID，就是还没有提交的事务ID

m_low_limit_no: 当事务小于这个值的时候，这些undo logs可以通过purge操作移除。

m_closed： 当前read view是否关闭了。



##### 2.23、实际例子

###### 2.231、开一个session A

```sql
begin;

insert into animals1(id, name, age) values(1,"jeffchan",26);

// 当前系统分配的事务id
SELECT TRX_ID FROM INFORMATION_SCHEMA.INNODB_TRX  WHERE TRX_MYSQL_THREAD_ID = CONNECTION_ID();
```



![tx0](D:\jeffchan\markdown\image\tx0.png)

上述中 可以查看到事务的id=19979



###### 2.232、再开一个session B

```sql
begin;

insert into animals1(id,name, age) values(2,"jeffchan",26);

SELECT TRX_ID FROM INFORMATION_SCHEMA.INNODB_TRX  WHERE TRX_MYSQL_THREAD_ID = CONNECTION_ID();
```

此时查看到对应的事务ID为：id=19979

![tx1](D:\jeffchan\markdown\image\tx1.png)

此时对于session A如果执行查询语句：select  * from animals1 、

![tx2](D:\jeffchan\markdown\image\tx2.png)

同样的session B也只能查看到自己的那一条id=2的记录。

这里互相能看见自己插入的记录，是因为上述的源码中：id == m_creator_trx_id条件，起作用了。

现在将上述的两个session 都执行commit。那么两个终端查看的记录都一样了。



在session A中开启事务，然后修改id=1的记录的name为jeffchan1

![tx4](D:\jeffchan\markdown\image\tx4.png)

在session B中开启事务，然后修改id=2的记录的name为jeffchan2

![tx5](D:\jeffchan\markdown\image\tx5.png)



此时的undo log链为：

| id   | name      | age  | DB_TRX_ID | `DB_ROLL_PTR` | (不存在的列，表示地址) |
| ---- | --------- | ---- | --------- | ------------- | ---------------------- |
| 1    | jeffchan1 | 26   | 19988     | 2             | 1                      |
| 1    | jeffchan  | 26   | 19979     | null          | 2                      |



| id   | name      | age  | DB_TRX_ID | `DB_ROLL_PTR` | (不存在的列，表示地址) |
| ---- | --------- | ---- | --------- | ------------- | ---------------------- |
| 2    | jeffchan2 | 26   | 19994     | 4             | 3                      |
| 2    | jeffchan  | 26   | 19980     | null          | 4                      |



1、此时对于session A 来说：如果它执行select操作，会生成一个read view（我这里是假设就只有我手动的这些数据库操作）：

m_low_limit_id=19995

m_up_limit_id=19988

m_creator_trx_id=19988

m_ids: [19988, 19994]

然后开始从上到下匹配： 对于id=1的记录，发现：DB_TRX_ID（19988）= m_creator_trx_id，那么返回第一条，对于id=2的记录：DB_TRX_ID （19994）在m_ids中，所以返回不可见。然后再对比第二条记录：

DB_TRX_ID(19980) <  m_up_limit_id(19994)，返回可见，所以返回结果如下：

![tx7](D:\jeffchan\markdown\image\tx7.png)





2、此时对于session B 来说：如果它执行select操作，会生成一个read view（我这里是假设就只有我手动的这些数据库操作）：

m_low_limit_id=19995

m_up_limit_id=19988

m_creator_trx_id=19994

m_ids: [19988, 19994]

然后开始从上到下匹配： 对于id=1的记录，发现：DB_TRX_ID（19988）在m_ids中，所以不可见，然后继续往下查找：DB_TRX_ID （19979） < m_up_limit_id (19988)，可见，直接返回。

对于id=2的记录：DB_TRX_ID （19994）= m_creator_trx_id（19994），直接返回。

![tx8](D:\jeffchan\markdown\image\tx8.png)



3、开一个session C: 如果它执行select操作，会生成一个read view（我这里是假设就只有我手动的这些数据库操作）：

m_low_limit_id=19995

m_up_limit_id=19988

m_creator_trx_id （这个值肯定不是19994或者19988）

m_ids: [19988, 19994]



然后开始从上到下匹配： 对于id=1的记录，发现：DB_TRX_ID（19988）在m_ids中，所以不可见，然后继续往下查找：DB_TRX_ID （19979） < m_up_limit_id (19988)，可见，直接返回。

对于id=2的记录：DB_TRX_ID （19994）在m_ids中，不可见，继续往下查找：发现DB_TRX_ID（19980）< m_up_limit_id=19988，可见，直接返回。

![tx6](D:\jeffchan\markdown\image\tx6.png)



4、此时提交session A的代码。但是因为这里的隔离是可重复读，所以read view 还是原来的那个read view，所以看到的记录是一样的，并不会改变。



5、此时的session C的记录会变，因为是开启一个新的事务，那么此时的read view为：

m_low_limit_id=19995

m_up_limit_id=19988

m_creator_trx_id （这个值肯定不是19994或者19988）

m_ids: [19994]

然后开始从上到下匹配： 对于id=1的记录，发现：DB_TRX_ID（19988）不在m_ids中，所以可见返回；

对于id=2的记录：DB_TRX_ID （19994）>= m_low_limit_id=19994，不可见，继续往下查找：发现DB_TRX_ID（19980）< m_up_limit_id=19988，可见，直接返回。

![tx9](D:\jeffchan\markdown\image\tx9.png)



##### 2.24、未解决的问题

mvcc在可重复读中，还是不可以解决幻读的问题，就是更新操作时候，会更新的其他事务已经提交的记录，不管其他事务是否先于当前事务还是晚于当前事务。查看的记录时，是否查看得到是根据版本号和对应的快照read view来决定的，但是事务里的更新不一样，它更新的是实际的记录，所以，比如你在一个事务里执行全局的update操作，此时你查看可能只有三条记录，但是你修改时，可能其他事务已经提交了一条记录，此时你修改的话，会修改到那条提交的记录，此时你的undo log中，会多出一条别的事务提交的记录，同时对应的事务ID是你当前的事务ID，此时你在当前事务再去select的话，你发现多一条记录。

同样的将隔离级别改成可重复读。

![tx10](D:\jeffchan\markdown\image\tx10.png)

可以发现只有三条记录，此时开启一个新的session，插入一条记录，然后提交。

insert into animals1(id, name , age) values (4, 'jeffchan4', 26)

此时在一开始的session中，全局修改记录，将年龄修改成25岁。

此时发现有提示有4条记录被修改：

![tx11](D:\jeffchan\markdown\image\tx11.png)

你此时一查，发现奇怪的多了一条记录（注意哦，我此时还没有提交事务哦，所以同一个事务中，还是出现了多次读不一致的情况）。

![tx13](D:\jeffchan\markdown\image\tx13.png)



#### 2.2、读已提交

对于这个隔离基本，不同的是，每次select的时候，都会生成一个新的read view的，所以在同一个事务中，会出现同一条记录，查询多次，结果会变化的结果。

首先要修改mysql的隔离级别：设置全局的事务隔离级别。

SET global TRANSACTION ISOLATION LEVEL Read committed;

（这个只对新启的session才起作用。参数可以为：Read uncommitted，Read committed，Repeatable，Serializable）



如果不想启用新的session,  也可以通过修改每个session的事务隔离级别。

SET SESSION TRANSACTION ISOLATION LEVEL Read COMMITTED



设置完后，需要查看下是否设置成功：select @@transaction_isolation

这个级别的记录查看是一样的，只不过不同的是，在读已提交级别，每次查询的时候，都会生成一个read view，所以对于已经提交的记录，在同一个事务中可以被查到。

还是一样，session A修改id = 1的记录，不提交先，然后session B修改id=2的记录，然后查看记录发现：可以看到自身修改的记录。但是session A的修改，还是看不到，此时需要将session A提交，然后在没有提交事务的session B中执行:  SELECT * from animals1 ,发现可以看到session A中的修改。



#### 2.3、读未提交和串行化

##### 2.3.1、读未提交

对于读未提交，不需要用到mvcc，因为它每次都是读取最新的一条记录，并不会去关心版本号。具体的实验，也是一样的，修改事务的隔离级别，然后开启不同的session,在事务里看能否看到其他session还没有提交的修改。

SET SESSION TRANSACTION ISOLATION LEVEL Read UNCOMMITTED

select @@transaction_isolation  要确保隔离级别设置成功。

##### 2.31、串行化

对于串行化，不需要用到mvcc，一般不会使用，因为太消耗性能，注意这里的维度也是针对行级的维度。

SET SESSION TRANSACTION ISOLATION LEVEL SERIALIZABLE

同样需要查看是否设置隔离级别成功与否才进行实验。select @@transaction_isolation  

这个隔离级别的记录，如果各种事务都是查询，那么可以查看，不会阻塞。

但是只要有一个事务里，先执行了更新操作，在没有提交之前，其他事务连查看对应更新的记录都会阻塞。

如果一个事务先执行了查询操作且事务没有提交的话，其他事务不能执行更新被查看了的行记录。



