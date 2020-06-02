# 锁

## 读锁与写锁

- 读锁：共享锁、Shared Locks、S锁。

- 写锁：排他锁、Exclusive Locks、X锁。

  |      | X锁  | S 锁   |
  | :--- | ---- | ------ |
  | X锁  | 冲突 | 冲突   |
  | S 锁 | 冲突 | 不冲突 |



### 读操作

对于普通 SELECT 语句，InnoDB 不会加任何锁

select ... lock in Shara mode

将查找到的数据加上一个S锁，允许其他事务继续获取这些记录的S锁，不能获取这些记录的X锁（会阻塞）

select ... for update

将查找到的数据加上一个X锁，不允许其他事务获取这些记录的S锁和X锁。

###  写操作

- DELETE：删除一条数据时，先对记录加X锁，再执行删除操作。
- INSERT：插入一条记录时，会先加***\*隐式锁\****来保护这条新插入的记录在本事务提交前不被别的事务访问到。
- UPDATE
  1. 如果被更新的列，修改前后没有导致存储空间变化，那么会先给记录加X锁，再直接对记录进行修改。
  2. 如果被更新的列，修改前后导致存储空间发生了变化，那么会先给记录加X锁，然后将记录删掉，再Insert一条新记录。

隐式锁：一个事务插入一条记录后，还未提交，这条记录会保存本次事务id，而其他事务如果想来读取这个记录会发现事务id不对应，所以相当于在插入一条记录时，隐式的给这条记录加了一把隐式锁。

##  行锁与表锁

### 行锁

- LOCK_REC_NOT_GAP： 单 个 行 记 录 上 的 锁 。
- LOCK_GAP：间隙锁，锁定一个范围，但不包括记录本身。GAP锁的目的，是为了防止同一事务的 两 次 当 前 读 ， 出 现 幻 读 的 情 况 。
- LOCK_ORDINARY：锁定一个范围，并且锁定记录本身。对于行的查询，都是采用该方法，主要目的是解决幻读的问题。

间隙锁（LOCK_GAP  GAP锁）

READ COMMITTED级别下

**查询使用的主键**

```sql
mysql> begin;
Query OK, 0 rows affected (0.00 sec)
mysql> select * from t1 where a = 1 for update;

+---+------+------+------+------+
| a | b    | c    | d    | e    |
+---+------+------+------+------+
| 1 | 1    | 1    | 1    | 1    |
+---+------+------+------+------+
1  row in set (0.00 sec)
```

```sql
mysql> select * from t1 where a = 1 for update; -- 会阻塞
mysql> select * from t1 where a = 2 for update;	-- 不会阻塞
```

总结：查询使用的是主键时，只需要在主键值对应的那一个条数据加锁即可。

**查询使用的唯一索引**

```sql
mysql> begin;
Query OK, 0 rows affected (0.00 sec)
mysql> select * from t1 where b = 1 for update;

+---+------+------+------+------+
| a | b    | c    | d    | e    |
+---+------+------+------+------+
| 1 | 1    | 1    | 1    | 1    |
+---+------+------+------+------+
1  row in set (0.00 sec)
```

```sql
mysql> select * from t1 where b = 1 for update; -- 会阻塞
mysql> select * from t1 where b = 2 for update;	-- 不会阻塞
```

总结：查询使用的是唯一索引时，只需要对查询值所对应的唯一索引记录项和对应的聚集索引上的项加锁即可。

**查询使用的普通索引**

```sql
mysql> begin;
Query OK, 0 rows affected (0.00 sec)
mysql> select * from t1 where e = '6' for update;

+---+------+------+------+------+
| a | b    | c    | d    | e    |
+---+------+------+------+------+
| 6 | 6    | 1    | 4    | 6    |
| 12| 12   | 1    | 1    | 6    |
+---+------+------+------+------+
2  rows in set (0.00 sec)
```

```sql
mysql> select * from t1 where a = 6 for update; -- 会阻塞
mysql> select * from t1 where a = 12 for update;	-- 会阻塞

mysql> select * from t1 where a = 1 for update;	-- 不会阻塞
mysql> select * from t1 where a = 2 for update;	-- 不会阻塞

mysql> insert t1(b,c,d,e) values(20,1,1,'51');	-- 不会阻塞
mysql> insert t1(b,c,d,e) values(21,1,1,'61');  -- 不会阻塞
```

总结：查询使用的是普通索引时，会对满足条件的索引记录都加上锁，同时对这些索引记录对应的聚集索引上的项也加锁。

**查询使用没有用到索引**

```sql
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from t1 where c = '1' for update;
+---+------+------+------+------+
| a | b    | c    | d    | e    |
+---+------+------+------+------+
| 1 | 6    | 1    | 4    | 6    |
| 12| 12   | 1    | 1    | 6    |
+---+------+------+------+------+
2  rows in set (0.00 sec)
```

```sql
mysql> select * from t1 where a = 1 for update; -- 会阻塞
mysql> select * from t1 where a = 3 for update; -- 不会阻塞
```

总结：查询的时候没有走索引，也只会对满足条件的记录加锁。

**REPEATABLE READ级别下**

查询使用主键

和RC隔离级别一样。

查询使用唯一索引

和RC隔离级别一样。

查询使用的普通索引

```sql
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from t1 where e = 6 for update;
+---+------+------+------+------+
| a | b    | c    | d    | e    |
+---+------+------+------+------+
| 6 | 6    | 1    | 4    | 6    |
| 12| 12   | 1    | 1    | 6    |
+---+------+------+------+------+
2  rows in set (0.00 sec)
```



```sql
mysql> select * from t1 where a = 6 for update; -- 会阻塞
mysql> select * from t1 where a = 12 for update; -- 会阻塞

mysql> select * from t1 where a = 1 for update; -- 不会阻塞
mysql> select * from t1 where a = 2 for update; -- 不会阻塞

mysql> insert t1(b,c,d,e) values(20,1,1,'51');	-- 会阻塞
mysql> insert t1(b,c,d,e) values(21,1,1,'61');  -- 会阻塞
```

总结：REPEATABLE READ级别可以解决幻读，解决的方式就是加了GAP锁。

**查询使用没有用到索引**

```sql
mysql> select * from t1 where e = 6 for update;
+---+------+------+------+------+
| a | b    | c    | d    | e    |
+---+------+------+------+------+
| 1 | 1    | 1    | 1    | 1    |
| 2 |  2   | 1    | 2    | 2    |
| 4 | 3    | 1    | 1    | 4    |
| 6 | 6    | 1    | 4    | 6    |
| 8 | 8    | 1    | 8    | 8    |
| 10| 10   | 1    | 2    | 10   |
| 12| 12   | 1    | 1    | 6    |
+---+------+------+------+------+
7  rows in set (0.00 sec)
```

```sql
mysql> select * from t1 where a = 1 for update; -- 会阻塞
mysql> select * from t1 where a = 2 for update; -- 会阻塞

mysql> select * from t1 where a = 3 for update; -- 不会阻塞
mysql> select * from t1 where a = 7 for update; -- 会阻塞(在READ COMMITTED级别中不会阻塞，跟解决幻读有关系)
```

总结：查询的时候没有走索引，会对表中所有的记录以及间隙加锁。

## 表锁

表级别的S锁 X锁

在对某个表执行SELECT、INSERT、DELETE、UPDATE语句时，InnoDB存储引擎是不会为这个表添加表级别的

S锁或者X锁的。

在对某个表执行ALTER TABLE、DROP TABLE这些DDL语句时，其他事务对这个表执行SELECT、INSERT、DELETE、UPDATE的语句会发生阻塞，或者，某个事务对某个表执行SELECT、INSERT、DELETE、UPDATE语    句时，其他事务对这个表执行DDL语句也会发生阻塞。这个过程是通过使用的元数据锁（英文名：Metadata Locks，简称MDL）来实现的，并不是使用的表级别的S锁和X锁。

- LOCK TABLES t1 READ：对表t1加表级别的S锁。
- LOCK TABLES t1 WRITE：对表t1加表级别的S锁。

尽量不用这两种方式去加锁，因为InnoDB的优点就是行锁，所以尽量使用行锁，性能更高。

## IS锁  IX锁

- IS锁：意向共享锁、Intention Shared Lock。当事务准备在某条记录上加S锁时，需要先在表级别加一个IS锁。

- IX锁，意向排他锁、Intention Exclusive Lock。当事务准备在某条记录上加X锁时，需要先在表级别加一个IX锁。

IS、IX锁是表级锁，它们的提出仅仅为了在之后加表级别的S锁和X锁时可以快速判断表中的记录是否被上锁，以避免用遍历的方式来查看表中有没有上锁的记录。

AUTO-INC 锁

- 在执行插入语句时就在表级别加一个AUTO-INC锁，然后为每条待插入记录的AUTO_INCREMENT 修饰的列分配递增的值，在该语句执行结束后，再把AUTO-INC锁释放掉。这样一个事务在持有AUTO-INC锁的过程中，其他事务的插入语句都要被阻塞，可以保证一个语句中分配的递增值是连续的。
- 采用一个轻量级的锁，在为插入语句生成AUTO_INCREMENT修饰的列的值时获取一下这个轻量级锁，然后生成本次插入语句需要用到的AUTO_INCREMENT列的值之后，就把该轻量级锁释放掉， 并不需要等到整个插入语句执行完才释放锁。

系统变量 innodb_autoinc_lock_mode:

- innodb_autoinc_lock_mode值为0：采用AUTO-INC锁。
- innodb_autoinc_lock_mode值为2：采用轻量级锁。
- 当innodb_autoinc_lock_mode值为1：当插入记录数不确定是采用AUTO-INC锁，当插入记录数确定时采用轻量级锁。

## 悲观锁

悲观锁用的就是数据库的行锁，认为数据库会发生并发冲突，直接上来就把数据锁住，其他事务不能修改，直至提交了当前事务。

## 乐观锁

乐观锁其实是一种思想，认为不会锁定的情况下去更新数据，如果发现不对劲，才不更新(回滚)。在数据库中往往添加一个version字段来实现。

## 死锁

。。。

## 避免死锁

- 以固定的顺序访问表和行
- 大事务拆小，大事务更容易产生死锁
- 在同一个事务中，尽可能做到一次锁定所需要的所有资源，减少死锁概率
- 降低隔离级别（下下签）
- 为表添加合理的索引