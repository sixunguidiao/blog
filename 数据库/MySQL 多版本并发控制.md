## MySQL 多版本并发控制 (MVCC)

多版本并发控制（Multi-Version Concurrency Control, MVCC）是 MySQL 的 InnoDB 存储引擎实现隔离级别的一种具体方式，用于实现提交读和可重复读这两种隔离级别。而未提交读隔离级别总是读取最新的数据行，无需使用 MVCC。可串行化隔离级别需要对所有读取的行都加锁，单纯使用 MVCC 无法实现。

MVCC最大的优势：读不加锁，读写不冲突。在读多写少的应用中，读写不冲突是非常重要的，极大的增加了系统的并发性能

## 版本号

- 系统版本号：是一个递增的数字，每开始一个新的事务，系统版本号就会自动递增。
- 事务版本号：事务开始时的系统版本号。

## 隐藏的列

MVCC 在每行记录后面都保存着两个隐藏的列，用来存储两个版本号：

- 创建版本号：指示创建一个数据行的快照时的系统版本号；
- 删除版本号：如果该快照的删除版本号大于当前事务版本号表示该快照有效，否则表示该快照已经被删除了。

## Undo 日志 (Undo log)

Undo log用于数据的撤回操作，它记录了修改的反向操作，比如，插入对应删除，修改对应修改为原来的数据，通过Undo log可以实现事务回滚，并且可以根据 Undo log 回溯到某个特定的版本的数据，实现MVCC。

MVCC 使用到的快照存储在 Undo 日志中，该日志通过回滚指针把一个数据行（Record）的所有快照连接起来。

![img](https://cyc2018.github.io/CS-Notes/pics/e41405a8-7c05-4f70-8092-e961e28d3112.jpg)

## 实现过程

以下实现过程针对可重复读隔离级别。

当开始新一个事务时，该事务的版本号肯定会大于当前所有数据行快照的创建版本号，理解这一点很关键。

### 1. SELECT

多个事务必须读取到同一个数据行的快照，并且这个快照是距离现在最近的一个有效快照。但是也有例外，如果有一个事务正在修改该数据行，那么它可以读取事务本身所做的修改，而不用和其它事务的读取结果一致。

把没有对一个数据行做修改的事务称为 T，T 所要读取的数据行快照的创建版本号必须小于 T 的版本号，因为如果大于或者等于 T 的版本号，那么表示该数据行快照是其它事务的最新修改，因此不能去读取它。除此之外，T 所要读取的数据行快照的删除版本号必须大于 T 的版本号，因为如果小于等于 T 的版本号，那么表示该数据行快照是已经被删除的，不应该去读取它。

### 2. INSERT

将当前系统版本号作为数据行快照的创建版本号。

### 3. DELETE

将当前系统版本号作为数据行快照的删除版本号。

### 4. UPDATE

将当前系统版本号作为**更新前的数据行快照**的删除版本号，并将当前系统版本号作为**更新后的数据行快照**的创建版本号。可以理解为先执行 DELETE 后执行 INSERT。

## 举例说明

```sql
create table mvcctest( 
id int primary key auto_increment, 
name varchar(20));
```

**transaction 1:**

```sql
start transaction;
insert into mvcctest values(NULL,'mi');
insert into mvcctest values(NULL,'kong');
commit;
```

假设系统初始事务ID为1；

| ID   | NAME | 创建时间 | 过期时间  |
| ---- | ---- | -------- | --------- |
| 1    | mi   | 1        | undefined |
| 2    | kong | 1        | undefined |

**transaction 2:**

```sql
start transaction;
select * from mvcctest;  //(1)
select * from mvcctest;  //(2)
commit
```

### SELECT

假设当执行事务2的过程中，准备执行语句(2)时，开始执行事务3：

**transaction 3:**

```SQL
start transaction;
insert into mvcctest values(NULL,'qu');
commit;
```

| ID   | NAME | 创建时间 | 过期时间  |
| ---- | ---- | -------- | --------- |
| 1    | mi   | 1        | undefined |
| 2    | kong | 1        | undefined |
| 3    | qu   | 3        | undefined |

事务3执行完毕，开始执行事务2的语句2，由于事务2只能查询创建时间小于等于2的，所以事务3新增的记录在事务2中是查不出来的，这就通过乐观锁的方式避免了幻读的产生 (原博客是这么说的，但是这里“避免了幻读的产生”实际上是不对的，**MVCC不能解决幻读**，除非本事务当中只有快照读，但是这种实际生产中没有什么意义。参考 [MVCC 能解决幻读吗？](https://www.jianshu.com/p/cef49aeff36b) )。

### UPDATE

假设当执行事务2的过程中，准备执行语句(2)时，开始执行事务4：

**transaction session 4:**

```sql
start transaction;
update mvcctest set name = 'fan' where id = 2;
commit;
```

InnoDB执行UPDATE，实际上是新插入了一行记录，并保存其创建时间为当前事务的ID，同时保存当前事务ID到要UPDATE的行的删除时间。

| ID   | NAME | 创建时间 | 过期时间  |
| ---- | ---- | -------- | --------- |
| 1    | mi   | 1        | undefined |
| 2    | kong | 1        | 4         |
| 2    | fan  | 4        | undefined |

事务4执行完毕，开始执行事务2 语句2，由于事务2只能查询创建时间小于等于2的，所以事务4修改的记录在事务2中是查不出来的，这样就保证了事务在两次读取时读取到的数据的状态是一致的。

### DELETE

假设当执行事务2的过程中，准备执行语句(2)时，开始执行事务5：

**transaction session 5:**

```sql
start transaction;
delete from mvcctest where id = 2;
commit;
```

| ID   | NAME | 创建时间 | 过期时间  |
| ---- | ---- | -------- | --------- |
| 1    | mi   | 1        | undefined |
| 2    | kong | 1        | 5         |

事务5执行完毕，开始执行事务2 语句2，由于事务2只能查询创建时间小于等于2、并且过期时间大于等于2，所以id=2的记录在事务2 语句2中，也是可以查出来的,这样就保证了事务在两次读取时读取到的数据的状态是一致的。

## Next-Key Locks

Next-Key Locks 是 MySQL 的 InnoDB 存储引擎的一种锁实现。

**MVCC 不能解决幻读的问题，Next-Key Locks 就是为了解决这个问题而存在的。在可重复读（REPEATABLE READ）隔离级别下，使用 MVCC + Next-Key Locks 可以解决幻读问题。**

### Record Locks

**锁定一个记录上的索引，而不是记录本身。**

如果表没有设置索引，InnoDB 会自动在主键上创建隐藏的聚簇索引，因此 Record Locks 依然可以使用。

### Gap Locks

锁定索引之间的间隙，但是不包含索引本身。例如当一个事务执行以下语句，其它事务就不能在 t.c 中插入 15。

```sql
SELECT c FROM t WHERE c BETWEEN 10 and 20 FOR UPDATE;
```

### Next-Key Locks

它是 Record Locks 和 Gap Locks 的结合，不仅锁定一个记录上的索引，也锁定索引之间的间隙。例如一个索引包含以下值：10, 11, 13, and 20，那么就需要锁定以下区间：

```sql
(negative infinity, 10]
(10, 11]
(11, 13]
(13, 20]
(20, positive infinity)
```

不过，**当查询的索引含有唯一属性时，InnoDB 会对 Next-Key Lock 进行优化，将其降级为 Record Lock，即仅锁住索引本身，而不是范围。**

## 参考资料

[多版本并发控制](https://cyc2018.github.io/CS-Notes/#/notes/%E6%95%B0%E6%8D%AE%E5%BA%93%E7%B3%BB%E7%BB%9F%E5%8E%9F%E7%90%86)

[MYSQL MVCC实现原理](https://www.jianshu.com/p/f692d4f8a53e)

[MVCC 能解决幻读吗？](https://www.jianshu.com/p/cef49aeff36b)

