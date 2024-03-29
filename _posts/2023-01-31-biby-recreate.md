# 为什么表数据删掉一半，表文件大小不变?
## 问题
数据表删了一半的数据，表文件大小没变

## 数据表的空间回收
InnoDB表包含两部分：表结构定义和数据

在MySQL8.0以前，表结构存在以.frm为后缀的文件里

MySQL8.0版本允许把表结构定义放在系统数据表中

### innodb\_file\_per\_table
表数据既可以存在共享表空间里，也可以是单独文件。这个行为由参数 innodb\_file\_per\_table 控制。

* OFF，表数据放在系统共享表空间，也就是跟数据字典放在一起
* ON，每个InnoDB表数据存储在一个以 .ibd 为后缀的文件中

从MySQL5.6.6开始，它的默认值就是ON

建议将该值设置为ON。一个表单独存储为一个文件更容易管理。

![image](images/xCNcQJdneYVCKuU-sVvzuF3HgVWi4YGqIZA74S35IwE.png)

* drop table

在不需要这个表的时候，通过 drop table 命令，系统会直接删除该文件。

如果放在共享表空间，即使表删掉了，空间也是不会回收的。

在删除整个表的时候，使用 drop table 命令回收表空间。

但是我们遇到的更多删除数据的场景是删除某些行，这时候就会遇到问题：表中的数据被删除了，但是表空间却没有被回收。

### 数据删除流程
InnoDB使用B+树结构组织数据。删除数据时，会把记录标记为删除，在后续插入数据时，如果插入数据在该记录区间，可能会复用这个位置。磁盘文件不会缩小。

InnoDB的数据是按页存储的，如果我们删掉了一个数据页上的所有记录，则整个数据页可以被复用。

* 数据页复用跟记录复用的区别

记录的复用，只限于符合条件范围的数据。

数据页可以复用到任何位置。

如果相邻的两个数据页利用率都很小，系统就会把这两个页上的数据合到其中一个页上，另一个数据页被标记为可复用。

* delete

如果我们用delete命令把整个表的数据删除，所有的数据页都会被标记为可复用。但是磁盘上，文件不会表小。

delete命令只是把记录的位置，或者数据页标记为“可复用”，磁盘文件的大小不会变。即delete命令不能回收表空间。

* 空洞

可以复用，而没有被使用的空间。

* 删除数据会造成空洞，插入数据也会

插入数据的时候数据页已满，申请新数据页保存数据。页分裂完成后，即可能留下空洞。

更新索引上的值。也是会造成空洞。

经过大量增删改的表，都可能存在空洞。如果能够把这些空洞去掉，就能达到收缩表空间的目的。

重建表，可以达到这样的目的。

### 重建表
假设表A需要做空间收缩

可以新建一个与表A结构相同的表B，按照主键ID递增的顺序，把数据一行一行从表A读出来再插入到表B中。

如果把表B作为临时表，数据从表A导入表B的操作完成后，用表B替换A，从效果上看，就起到了收缩表A空间的作用。

可以使用命令 alter table A engine=InnoDB 来重建表

在MySQL5.5版本之前，这个命令的执行流程跟上述差不多，区别只是临时表B不需要自己创建，MySQL会自动完成转存数据、交换表名、删除旧表的操作。

花时间最多的步骤是往临时表插入数据的过程，在这个过程中，有新的数据要写入到表A的话，就会造成数据丢失。因此，在整个DDL过程中，表A中不能有更新。也就是说，这个DDL不是Online的。

MySQL5.6版本引入的Online DDL，对这个操作流程做了优化。

* 优化后流程

1. 建立一个临时文件，扫描表A主键的所有数据页；
2. 用数据页中表A的记录生成B+树，存储到临时文件中；
3. 生成临时文件的过程中，将所有对A的操作记录在一个日志文件（row log）中；
4. 临时文件生成后，将日志文件中的操作应用到临时文件，得到一个逻辑数据上与表A相同的数据文件；
5. 用临时文件替换表A的数据文件；

由于日志文件记录和重放操作功能的存在，优化后的流程允许在重建表的过程中，对表A做增删改操作。(这也就是Online DDL名字的来源)。

上述的重建方法都会扫描原表数据和构建临时文件。对于很大的表来说，这个操作是很消耗CPU和IO资源的。(线上业务要小心的控制操作时间或者使用gh-ost项目)。

### Online 和 inplace
优化后，重建表过程中，重建出来的数据放在“tmp\_file”中，这个临时文件是InnoDB在内部创建出来的。整个DDL过程都在InnoDB内部完成。对于Server层来说，没有把数据挪动到临时表，是一个“原地”操作，这就是“inplace”名称的来源。

我们重建表的语句 alter table t engine=InnoDB,其隐含的意思是

```sql
alter table t engine=innodb,ALGORITHM=inplace;
```
跟inplace对应的就是拷贝表的方式，用法是

```sql
alter table t engint=innodb,ALGORITHM=copy;
```
ALGOITHM=copy，表示强制拷贝表。

* inplace 和 Online 两个逻辑之间的关系

1.DDL过程如果是Online的，就一定是inplace的；

2.反过来未必，inplace的DDL，可能不是Online的。截止到MySQL8.0，添加全文索引（FULLTEXT index）和空间索引（SPATIAL index）就属于这种情况。

* optimize table、analyze table、alter table的区别

alter table t (recreate)就是上述优化后的流程（MySQL5.6开始）

analyze table t 不是重建表，只是对表的索引信息做重新统计，没有修改数据，这个过程中添加了MDL读锁。

optimize table t 等于 recreate + analyze。

## 总结
InnoDB使用B+树结构组织数据。在执行delete操作时，只会将位置或者数据页标记为可复用，磁盘文件不会缩小。在该数据组织方式下，数据的增删改会造成数据空洞（可以复用，但没复用的位置）。重建表操作能够把这些空洞去掉，达到收缩空间的目的。重建表在5.6版本以前不知道Online，5.6版本优化为Online DDL。

* 如何收缩数据表空间

1. delete操作，不可以收缩表空间
2. drop table，可以收缩表空间
3. truncate = drop + create，可以收缩表空间
4. alter table，（重建表）可以做空间收缩

