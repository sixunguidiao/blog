# MySQL 分区

## 什么是数据库分区

以mysql为例。mysql数据库中的数据是以文件的形势存在磁盘上的，默认放在/mysql/data下面（可以通过my.cnf中的datadir来查看），一张表主要对应着三个文件，一个是frm存放表结构的，一个是myd存放表数据的，一个是myi存表索引的（这里特指 MyISAM 引擎，因为 InnoDB 引擎数据和所有都是在一个文件中的）。如果一张表的数据量太大的话，那么myd,myi就会变的很大，查找数据就会变的很慢，这个时候我们可以利用mysql的分区功能，在物理上将这一张表对应的三个文件，分割成许多个小块，这样呢，我们查找一条数据时，就不用全部查找了，只要知道这条数据在哪一块，然后在那一块找就行了。如果表的数据太大，可能一个磁盘放不下，这个时候，我们可以把数据分配到不同的磁盘里面去。

分区的两种方式：

1. 横向分区

什么是横向分区呢？就是横着来分区了，举例来说明一下，假如有100W条数据，分成十份，前10W条数据放到第一个分区，第二个10W条数据放到第二个分区，依此类推。也就是把表分成了十分，根用merge来分表，有点像哦。取出一条数据的时候，这条数据包含了表结构中的所有字段，也就是说横向分区，并没有改变表的结构。

2. 纵向分区

什么是纵向分区呢？就是竖来分区了，举例来说明，在设计用户表的时候，开始的时候没有考虑好，而把个人的所有信息都放到了一张表里面去，这样这个表里面就会有比较大的字段，如个人简介，而这些简介呢，也许不会有好多人去看，所以等到有人要看的时候，在去查找，分表的时候，可以把这样的大字段，分开来。

## 分区的优点

1. 使用分区可以将数据分在多个磁盘上，分区数据可以被分步到不同的物理位置， 可以做分布式有效利用多个磁盘；
2. 根据查找条件，也就是where后面的条件，查找只查找相应的分区不用全部查找了；
3. 进行大数据搜索时可以进行并行处理；
4. 跨多个磁盘来分散数据查询，来获得更大的查询吞吐量；

## 分区表的限制

1.  一个表最多只能有1024个分区。

2. 在MySQL 5.1中，分区表达式必须是整数，或者是返回整数的表达式。在MySQL 5.5中，某些场景可以直接使用列进行分区。

3. 如果分区字段中含有主键或者唯一索引的列，那么所有主键列和唯一索引列都必须包含。

4. 分区表无法使用外键约束。

## MySQL 的分区

MySQL 的分区主要支持以下几种方式：

### Range

按照RANGE分区的表是通过如下一种方式进行分区的，每个分区包含那些分区表达式的值位于一个给定的连续区间内的行：

```mysql
//创建range分区表  
mysql> CREATE TABLE IF NOT EXISTS `user` (  
 ->   `id` int(11) NOT NULL AUTO_INCREMENT COMMENT '用户ID',  
 ->   `name` varchar(50) NOT NULL DEFAULT '' COMMENT '名称',  
 ->   `sex` int(1) NOT NULL DEFAULT '0' COMMENT '0为男，1为女',  
 ->   PRIMARY KEY (`id`)  
 -> ) ENGINE=MyISAM  DEFAULT CHARSET=utf8 AUTO_INCREMENT=1  
 -> PARTITION BY RANGE (id) (  
 ->     PARTITION p0 VALUES LESS THAN (3),  
 ->     PARTITION p1 VALUES LESS THAN (6),  
 ->     PARTITION p2 VALUES LESS THAN (9),  
 ->     PARTITION p3 VALUES LESS THAN (12),  
 ->     PARTITION p4 VALUES LESS THAN MAXVALUE  
 -> );  
```

也可以对现有的表进行分区：

```mysql
mysql> alter table aa partition by RANGE(id)  
 -> (PARTITION p1 VALUES less than (1),  
 -> PARTITION p2 VALUES less than (5),  
 -> PARTITION p3 VALUES less than MAXVALUE);
```

### List 分区

LIST分区中每个分区的定义和选择是基于某列的值从属于一个值列表集中的一个值，而RANGE分区是从属于一个连续区间值的集合（也就是说 Range 是基于一个连续的区间，List 是基于一个离散的区间）。

```mysql
//这种方式失败  
mysql> CREATE TABLE IF NOT EXISTS `list_part` (  
 ->   `id` int(11) NOT NULL AUTO_INCREMENT COMMENT '用户ID',  
 ->   `province_id` int(2) NOT NULL DEFAULT 0 COMMENT '省',  
 ->   `name` varchar(50) NOT NULL DEFAULT '' COMMENT '名称',  
 ->   `sex` int(1) NOT NULL DEFAULT '0' COMMENT '0为男，1为女',  
 ->   PRIMARY KEY (`id`)  
 -> ) ENGINE=INNODB  DEFAULT CHARSET=utf8 AUTO_INCREMENT=1  
 -> PARTITION BY LIST (province_id) (  
 ->     PARTITION p0 VALUES IN (1,2,3,4,5,6,7,8),  
 ->     PARTITION p1 VALUES IN (9,10,11,12,16,21),  
 ->     PARTITION p2 VALUES IN (13,14,15,19),  
 ->     PARTITION p3 VALUES IN (17,18,20,22,23,24)  
 -> );  
ERROR 1503 (HY000): A PRIMARY KEY must include all columns in the table's partitioning function 
 
//这种方式成功 
mysql> CREATE TABLE IF NOT EXISTS `list_part` ( 
 ->   `id` int(11) NOT NULL  COMMENT '用户ID', 
 ->   `province_id` int(2) NOT NULL DEFAULT 0 COMMENT '省', 
 ->   `name` varchar(50) NOT NULL DEFAULT '' COMMENT '名称', 
 ->   `sex` int(1) NOT NULL DEFAULT '0' COMMENT '0为男，1为女'  
 -> ) ENGINE=INNODB  DEFAULT CHARSET=utf8  
 -> PARTITION BY LIST (province_id) (  
 ->     PARTITION p0 VALUES IN (1,2,3,4,5,6,7,8),  
 ->     PARTITION p1 VALUES IN (9,10,11,12,16,21),  
 ->     PARTITION p2 VALUES IN (13,14,15,19),  
 ->     PARTITION p3 VALUES IN (17,18,20,22,23,24)  
 -> );
```

**注意，上面这个创建list分区时，如果有主键的话，分区时主键必须在其中，不然就会报错。如果我不用主键，分区就创建成功了，一般情况下，一个张表肯定会有一个主键，这算是一个分区的局限性吧。**

### Hash 分区

HASH分区主要用来确保数据在预先确定数目的分区中平均分布，你所要做的只是基于将要被哈希的数据指定一个列值或表达式，以 及指定被分区的表将要被分割成的分区数量。

```mysql
mysql> CREATE TABLE IF NOT EXISTS `hash_part` (  
 ->   `id` int(11) NOT NULL AUTO_INCREMENT COMMENT '评论ID',  
 ->   `comment` varchar(1000) NOT NULL DEFAULT '' COMMENT '评论',  
 ->   `ip` varchar(25) NOT NULL DEFAULT '' COMMENT '来源IP',  
 ->   PRIMARY KEY (`id`)  
 -> ) ENGINE=INNODB  DEFAULT CHARSET=utf8 AUTO_INCREMENT=1  
 -> PARTITION BY HASH(id)  
 -> PARTITIONS 3;
```

### key 分区

按照KEY进行分区类似于按照HASH分区，除了HASH分区使用的用户定义的表达式，而KEY分区的哈希函数是由MySQL 服务器提供。

```mysql
mysql> CREATE TABLE IF NOT EXISTS `key_part` (  
 ->   `news_id` int(11) NOT NULL  COMMENT '新闻ID',  
 ->   `content` varchar(1000) NOT NULL DEFAULT '' COMMENT '新闻内容',  
 ->   `u_id` varchar(25) NOT NULL DEFAULT '' COMMENT '来源IP',  
 ->   `create_time` DATE NOT NULL DEFAULT '0000-00-00 00:00:00' COMMENT '时间'  
 -> ) ENGINE=INNODB  DEFAULT CHARSET=utf8  
 -> PARTITION BY LINEAR HASH(YEAR(create_time))  
 -> PARTITIONS 3;
```

### 子分区

子分区是分区表中每个分区的再次分割，子分区既可以使用HASH希分区，也可以使用KEY分区。这也被称为复合分区（composite partitioning）。

1. 如果一个分区中创建了子分区，其他分区也要有子分区；
2. 如果创建子分区，每个分区中的子分区数必须相同；
3. 同一分区内的子分区的名字不能相同，不同分区内的子分区的名字可以相同（5.1.50不适用）；

## 分区表中比较坑的地方

1. 分区表表达式的值为NULL时，如上例中的year（order_date）为空的时候，该条记录会被存放到第一个分区（特殊分区）；
2. 如果定义的索引列和分区列不匹配，会导致无法做分区过滤（分区过滤的意思是根据分区列找到对应的分区）。即字段a为索引列，b为分区列。因为每个分区都有独立的索引，所以扫描列b上的索引会导致全部分区的扫描；
3. 分区中的Range分区类型，在数据量比较大的时候，定位到具体的分区上，性能不如其他的分区类型：Key分区和Hash分区。这两种分区方式找到确定分区都是线性时间；
4. 打开及锁表的成本很高：当查询访问分区表的时候，MySQL需要打开并锁住所有的底层表（一个表经过分区之后对应多个底层表，一个分区对应一个底层表）。这个操作发生在分区过滤之前，无法通过分区过滤降低次开销，与分区类型也无关（由于锁表发生在分区过滤之前，因此相对于没做分区时只需要锁住一个表而言，这里需要锁住多个表，开销变大）。因此，类似根据主键查找单行的操作也会带来额外的开销。通常需要批量操作和减少分区数量降低该项性能消耗。

5. 部分维护分区的成本开销比较高。如重组分区或者类似alter语句的操作，这些操作需要复制数据。与alter table类似，需要先创建临时分区，复制之后删除原分区；

[mysql分区功能详细介绍，以及实例](<http://blog.51yip.com/mysql/1013.html>)

[一步一步学习MySQL分区表](<http://km.oa.com/group/15624/articles/show/206490?kmref=search&from_page=1&no=1>)

