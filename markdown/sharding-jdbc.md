<center><h1>sharding-jdbc</h1></center>

sharding-jdbc的执行原理，当接受到一条SQL的时候，会执行：

SQL解析 -> SQL路由 ->  SQL改写



SQL解析：

词法解析

语法解析



SQL路由

多片路由

单片路由

全库表路由



SQL改写



SQL执行

内存限制模式:   不限制连接数

连接限制模式： 严格控制连接数，OLTP场景



结果归并



内存归并

流水归并

装饰者归并



公共表： 会冗余一份数据

