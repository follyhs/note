# Mysql事物笔记

## 事务

 场景：小明向小强转账10元

- 原子性(Atomicity)

  转账操作是一个不可分割的操作，要么转失败，要么转成功，不能存在中间的状态，也就是转了一半的这种情况。我们把这种要么全做，要么全不做的规则称之为原子性。

- 隔离性(Isolation)

  另外一个场景：

  1. 小明向小强转账10元

  2. 小明向小红转账10元

  隔离性表示上面两个操作是不能相互影响的

- 一致性(Consistency)

  对于上面的转账场景，一致性表示每一次转账完成后，都需要保证整个系统的余额等于所有账户的收入减去所有账户的支出。

  如果不遵循原子性，也就是如果小明向小强转账10元，但是只转了一半，小明账户少了10元，小强账户并没有增加，所以没有满足一致性了。

  同样，如果不满足隔离性，也有可能导致破坏一致性。

  所以说，数据库某些操作的原子性和隔离性都是保证一致性的一种手段，在操作执行完成后保证符合所有既定的约束则是一种结果。

  实际上我们也可以对表建立约束来保证一致性。

- 持久性(Durability)

  对于转账的交易记录，需要永久保存。

### 事务的概念

我们把需要保证原子性、隔离性、一致性和持久性的一个或多个数据库操作称之为一个事务。

### 事务的使用

BEGIN语句代表开启一个事务，后边的单词WORK可有可无。开启事务后，就可以继续写若干条语句，这些语句都属于刚刚开启的这个事务。

#### BEGIN[WOEK];

```sql
mysql> BEGIN;
Query OK, 0 rows affected (0.00 sec)
mysql> sql...
```

#### START TRANSACTION;

START TRANSACTION语句和BEGIN语句有着相同的功效，都标志着开启一个事务，比如这样：

```sql
mysql> START TRANSACTION;
Query OK, 0 rows affected (0.00 sec)
mysql> sql...
```

#### 事务的提交

```sql
mysql> BEGIN;
Query OK, 0 rows affected (0.00 sec)
mysql> UPDATE account SET balance = balance - 10 WHERE id = 1; Query OK, 1 row affected (0.02 sec)
Rows matched: 1	Changed: 1	Warnings: 0
mysql> UPDATE account SET balance = balance + 10 WHERE id = 2; Query OK, 1 row affected (0.00 sec)
Rows matched: 1	Changed: 1	Warnings: 0
mysql> COMMIT;
Query OK, 0 rows affected (0.00 sec)
```

#### 手动停止事务

```sql
mysql> BEGIN;
Query OK, 0 rows affected (0.00 sec)
mysql> UPDATE account SET balance = balance - 10 WHERE id = 1; Query OK, 1 row affected (0.00 sec)
Rows matched: 1	Changed: 1	Warnings: 0
mysql> UPDATE account SET balance = balance + 1 WHERE id = 2; Query OK, 1 row affected (0.00 sec)
Rows matched: 1	Changed: 1	Warnings: 0
mysql> ROLLBACK;
Query OK, 0 rows affected (0.00 sec)
```

这里需要强调一下，ROLLBACK语句是我们程序员手动的去回滚事务时才去使用的，如果事务在执行过程中遇到了某些错误而无法继续执行的话，事务自身会自动的回滚。

#### 自动提交事务

```sql
mysql> SHOW VARIABLES LIKE 'autocommit';
```

默认情况下，如果我们不显式的使用START TRANSACTION或者BEGIN语句开启一个事务，那么每一条语句都算是一个独立的事务，这种特性称之为事务的自动提交

如果我们想关闭这种自动提交的功能，可以使用下边两种方法之一:

- 显式的的使用START TRANSACTION或者BEGIN语句开启一个事务。这样在本次事务提交或者回滚前会暂时关闭掉自动提交的功能。
- 把系统变量autocommit的值设置为OFF，就像这样：SET auto commit = OFF; 这样的话，我们写入的多条语句就算是属于同一个事务了，直到我们显式的写出COMMIT语句来把这个事务提交掉，或者显式的写出ROLLBACK语句来把这个事务回滚掉。

#### 隐式提交

当我们使用START TRANSACTION或者BEGIN语句开启了一个事务，或者把系统变量autocommit的值设置为OFF时，事务就不会进行自动提交，但是如果我们输入了某些语句之后就会悄悄的提交掉，就像我们输入了COMMIT语句了一样，这种因为某些特殊的语句而导致事务提交的情况称为隐式提交，这些会导致事务隐式提交的语句包括：

- 定义或修改数据库对象的数据定义语言（Data definition language，缩写为：DDL）。所谓的数据库对象，指的就是数据库、表、视图、存储过程等等这些东西。当我们使用CREATE、ALTER、DROP等语句去修改这些所谓的数据库对象时，就会隐式的提交前边语句所属于的事务。

- 隐式使用或修改mysql数据库中的表：当我们使用ALTER USER、CREATE USER、DROP USER、GRANT、RENAME USER、SET PASSWORD等语句时也会隐式的提交前边语句所属于的事务。

- 事务控制或关于锁定的语句：当我们在一个事务还没提交或者回滚时就又使用START TRANSACTION或者BEGIN语句开启了另一个事务时，会隐式的提交上一个事务。或者当前的autocommit系统变量的值为OFF，我们手动把它调为ON时，也会隐式的提交前边语句所属的事务。或者使用LOCK TABLES、UNLOCK TABLES等关于锁定的语句也会隐式的提交前边语句所属的事务。

- 加载数据的语句：比如我们使用LOAD DATA语句来批量往数据库中导入数据时，也会隐式的提交前边语句所属的事务。

- 其它的一些语句：使用ANALYZE TABLE、CACHE INDEX、CHECK TABLE、FLUSH、 LOAD INDEX INTO CACHE、OPTIMIZE TABLE、REPAIR TABLE、RESET等语句也会隐式的提交前边语句所属的事务。

  

### 隔离性

```sql
--  修改隔离级别
mysql> set session transaction isolation level read uncommitted;
-- 查看隔离级别
mysql> select @@tx_isolation;
```

### 未提交读（READ UNCOMMITTED）

一个事务可以读到其他事务还没有提交的数据，会出现脏读。

一个事务读到了另一个未提交事务修改过的数据，这就是脏读

### 已提交读（READ COMMITED）

一个事务只能读到另一个已经提交的事务修改过的数据，并且其他事务每对该数据进行一次修改并提交后，该事务都能查询得到最新值，会出现不可重复读、幻读。

如果一个事务先根据某些条件查询出一些记录，之后另一个事务又向表中插入了符合这些条件的记录，原先的事务再次按照该条件查询时，能把另一个事务插入的记录也读出来，这就是幻读。

### 可重复读（REPEATABLE READ）

一个事务第一次读过某条记录后，即使其他事务修改了该记录的值并且提交，该事务之后再读该条记录时，读到的仍是第一次读到的值，而不是每次都读到不同的数据，这就是可重复读，这种隔离级别解决了不可重复，但是还是会出现幻读。

### 串行化（SERIALIZABLE）

以上3种隔离级别都允许对同一条记录同时进行读-读、读-写、写-读的并发操作，如果我们不允许读-写、写-读 的并发操作，可以使用SERIALIZABLE隔离级别，这种隔离基金因为对同一条记录的操作都是串行的，所以不会出现脏读、幻读等现象。

## 总结

- READ UNCOMMITTED隔离级别下，可能发生**脏读** 、**不可重复读**和**幻读**问题。

- READ COMMITTED隔离级别下，可能发生**不可重复读**和**幻读**问题，但是不会发生**脏读**问题。

- REPEATABLE READ隔离级别下，可能发生**幻读**问题，不会发生**脏读**和**不可重复读**的问题

- SERIALIZABLE隔离级别下，各种问题都不可以发生。

- 注意：这四种隔离级别是SQL的标准定义，不同的数据库会有不同的实现，特别需要注意的是

  **MySQL 在REPEATABLE READ 隔离级别下，是可以禁止幻读问题的发生的**

