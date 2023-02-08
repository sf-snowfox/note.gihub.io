# count(\*)这么慢，我该怎么办？
## count(\*)的实现方式
不同的MySQL引擎中，count(\*)有不同的实现方式。

MyISAM引擎把一个表的总行数存在磁盘上，执行count(\*)会直接返回这个数，效率很高。

InnoDB执行count(\*)的时候，需要把数据一行一行从引擎里面读出来，然后累积计数。

以上是没有过滤条件的count(\*)，如果加了where条件，MyISAM表也是不能返回得这么快的。

* InnoBD为什么不能把count(\*)数字存起来

即使在同一时刻的多个查询，由于多版本并发控制(MVCC)的原因，InnoDB表“应该返回多少行”也是不确定的。

* 计算count(\*)的例子

假设表t中现有10000条记录，我们设计三个用户并行的会话

1. 会话A先启动事务并查询一次表的总行数；
2. 会话B启动事务，插入一行记录后，查询表的总行数；
3. 会话C先启动一个单独的语句，插入一行记录后，查询表的总行数；

假设从上到下是按照时间顺序执行的，同一行语句是在同一时刻执行的

|会话A|会话B|会话C|
| ----- | ----- | ----- |
|begin;| | |
|select count(\*) from t;| | |
| | |insert int t (插入一行);|
| |begin;| |
| |insert int t (插入一行);| |
|select count(\*) from t;<br>(返回10000)|select count(\*) from t;<br>(返回10002)|select count(\*) from t;<br>(返回10001)|

在最后一个时刻，三个会话A、B、C同时查询表t的总行数，拿到的结果不同。

这和InnoDB的事务设计有关，可重复读是它默认的隔离级别。在代码实现上就是通过多版本并发控制，也就是MVCC来实现的。每一行记录都要判断自己是否对这个会话可见。因此对于count(\*)请求来说，InnoDB只好把数据一行一行的读出依次判断，可见的行才能够用于计算“基于这个查询”的表的总行数。

* count(\*)操作的优化

InnoDB是索引组织表，主键索引树的叶子节点是数据，而普通索引树的叶子节点是主键值。因此，普通索引树比主键索引树小很多。对于count(\*)的操作，遍历哪个索引树得到的结果逻辑上都是一样的。因此，MySQL优化器会找到最小的那棵树来遍历。

在保证逻辑正确的前提下，尽量减少扫描的数据量，是数据库系统设计的通用法则之一。

* show table status

show table status 命令输出结果也有一个table\_rows 用于显示这个表当前有多少行。这个值是从索引采样估算得来的，也很不准。（官方文档说误差有40%-50%）。

因此,show table status命令显示的行数不能直接使用。

## 自己计数
如果有一个页面经常要显示交易系统的操作记录总数，我们只能自己计数。

* 用缓存系统保存计数

存在问题

丢失数据

计数不精确

* 在数据库保存计数

把这个计数放到数据库里单独一张计数表C

1. 解决奔溃丢失问题，InnoDB支持奔溃恢复不丢失数据
2. 使用InnoDB事务特性，计数精确。

## 不同count用法
* count()语义

count()是一个聚合函数，对于返回的结果集，一行行的判断，如果count()函数的参数不是NULL，累计值就加1，否则不加，最后返回累计值。

所以，count(\*)、count(主键ID)、count(1)都表示返回满足条件的结果集的总行数；而count(字段)，则表示返回满足条件的数据行里面，参数“字段”不为NULL的总个数。

* 分析性能差别原则

1. Server层要什么就给什么;
2. InnoDB只给必要的值;
3. 现在的优化器只优化了count(\*)的语义为“取行数”,其他“显而易见”的优化并没有做；
* count(主键ID)

InnoDB引擎遍历整张表，把每一行的id值都取出来，返回给Server层。server层拿到id后，判断不可能为空的，就按行累加。

* count(1)

InnoDB引擎遍历整张表，但不取值。server层对于返回的每一行，放一个数字“1”进去，判断不可能会是为空的，按行累加。
g
count(1)比count(主键ID)执行得快。因为从引擎返回id会涉及到解析数据行，以及拷贝字段值的操作。

* count(字段)

1. 如果“字段”定义为not null，一行行地从记录里面读出这个字段，判断不能为null，按行累加；
2. 如果“字段”允许为null，执行的时候，判断到有可能是null，要把值取出来再判断一下，不是null才累加。
* count(\*)

不会把字段全部取出来，专门做了优化，不取值。count(\*)肯定不是null，按行累加。

按效率排序：count(字段)<count(主键ID)<count(1)≈count(\*)。

因此，尽量使用count(\*)。

## 总结
1. MyISAM表虽然count(\*)很快,但是不支持事务；
2. show table status 命令虽然返回很快，但是不准确；
3. InnoDB表直接count(\*)会遍历全表，虽然结果准确，但会导致性能问题；

在数据量比较大，频繁需要count展示条数的场景，需要自己存储计数值。方案是用InnoDB表存储并使用事务来保证数值准确。

* cout性能对比

count(字段)<count(主键ID)<count(1)≈count(\*)。