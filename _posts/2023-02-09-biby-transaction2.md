# 事务到底是隔离的还是不隔离的？
## 事务启动
begin/start transaction 并不是一个事务的起点，执行后，等到该线程第一个操作InnoDB表的语句，事务才真正启动。使用 start transaction with consistent snapshot 命令，马上启动一个事务。

begin/start transaction 方式启动，一致性视图是在执行第一个快照读语句时创建的。

start transaction with consitent snapshot 方式启动，一致性视图是在执行语句时创建的。

## MySQL里面视图概念
1. view，用查询语句定义的虚拟表。语法是 create view ... ，查询方法与表一致。
2. InnoDB在实现MVCC时用到的一致性读视图，即 consistent read view，用于支持RC(Read Committed,读提交)和 RR(Repeatable Read，可重复读)隔离级别的实现。没有物理结构，作用是事务执行期间用来定义“我能看到什么数据”。

## 快照在MVCC里是怎么工作的
在可重复读隔离级别下，事务在启动的时候就“拍了个快照”。这个快照是基于整库的。
### 快照是怎么实现的
InnoDB里面每个事务有一个唯一的事务ID，叫做transaction id。它是在事务开始的时候向InnoDB的事务系统申请的，是按申请顺序严格递增。

每行数据也是有多个版本的。每次事务更新数据的时候，都会生成一个新的事务版本，并把transaction id 赋值给这个数据版本的事务ID，记为 row trx\_id。同时，旧的数据版本要保留，并且在新的数据版本中，能够有信息可以拿到它。

也就是说数据表中的一行记录，其实可能有多个版本(row),每个版本有自己的 row trx\_id。

### 可重复读的定义
一个事务启动的时候，能够看到所有已经提交的事务结果。但是，之后这个事务执行期间，其他事务的更新对它不可见。事务在启动的时候需要声明，“以我启动的时刻为准，如果一个数据版本是在我启动之前生成的，就认；如果是在启动后生成的，就不认，必须找到它的上一个版本”。如果“上一个版本”也不可见，就继续往前找。如果是事务自己更新的数据，自己也要认。

* 判断可见性

InnoDB为每个事务构造了一个数组，用来保存这个事务启动瞬间，当前正在“活跃”的所有事务ID。“活跃”指的就是，启动了但还没提交。

数组里面事务ID的最小值记为低水位，当前系统里面已经创建过的事务ID的最大值加1记为高水位。

这个视图数组和高水位，就组成了当前事务的一致性视图(read-view)。

数据版本的可见性规则，就是基于数据的 row trx\_id 和 这个一致性视图的对比结果得到的。

这个视图数组把 row trx\_id 分成了几种不同的情况。

\[已提交事务\](低水位)\[未提交事务集合\](高水位)\[未开始事务\]

这样，对于当前事务的启动瞬间来说，一个数据版本的row trx\_id，有以下几种可能

1. 如果落在绿色部分，表示这个版本是已提交的事务或者是当前事务自己生成的，这个数据是可见的；
2. 如果落在红色部分，表示这个版本是由将来启动的事务生成的，是肯定不可见的；
3. 如果落在黄色部分，那就包括两种情况
    * 如果row trx\_id在数组中，表示这个版本由还没有提交的事务生成的，不可见；
    * 如果row trx\_id不在数组中，表示这个版本是已经提交事务生成的，可见。
* eg:

启动瞬间拿到的数组是\[5679\]

\[5679\]10(高水位)

当前事务ID 11

则

<5 可见

\>=10 不可见

row trx 在数组中，正在执行(还未提交事务)不可见

row trx 不在数组中，可见，比如8，8不在数组中，表示已经执行完，可见

有了如上声明后，系统里面随后发生的更新，就跟这个事务看到的内容无关。之后的更新，生成的版本一定属于上面的2或者3(a)的情况，对于该事务，这些新的数据版本是不存在的（不可见的），所以这个事务的快照，就是“静态”的了。

综上，InnoDB利用了“所有数据都有多个版本”这个特性，实现了“秒级创建快照”的能力。

* 可见性分析总结

以上可见性分析规则可以总结为：

1. 版本未提交，不可见；
2. 版本已提交，但是是在视图创建后提交的，不可见；
3. 版本已提交，而且是在视图创建前提交的，可见。

## 更新逻辑
更新数据都是先读后写的，而这个读，只能读当前的值，称为“当前读”(current read)。

除了update语句外，select 语句如果加锁，也是当前读。

```bash
select k from t where id=1 lock in share mode;//读锁(S锁，共享锁)
select k from t where id=1 for update;//写锁(X锁，排它锁)
```
* 事务的可重复读能力是怎么实现的

可重复读的核心就是一致性读(consistent read);事务更新数据的时候，只能用当前读。如果当前的记录的行锁被其他事务占用的话，就需要进入锁等待。

* 读提交的逻辑与可重复读的逻辑类似，他们的主要区别是:
    * 在可重复读隔离级别下，只需要在事务开始的时候创建一致性视图，之后事务里的其他查询都共用这个一致性视图。
    * 在读提交隔离级别下，每一个语句执行前都会重新算出一个新的视图。

## 总结
简述事务启动起点，介绍视图，引入一致性读视图概念，其在实现MVCC时用于支持读提交跟可重复读。MVCC相当于在事务启动的时候拍了个快照，里面每个事务有唯一事务ID，每行数据有多个版本及相应的row trx\_id,通过事务启动时的活跃事务ID数组来判断事务可见性。在更新数据的时候用到当前读。

## 参考
[极客时间文章](https://time.geekbang.org/column/article/70562)