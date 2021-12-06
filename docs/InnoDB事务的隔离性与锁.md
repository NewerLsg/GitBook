# InnoDB事务的隔离性与锁

## 隔离性的定义

​		MySQL可同时支持多个客户端会话，由于会话共享数据库数据，因此在执行事务时可能出现竞态。隔离性是描述数据库在竞态下需要满足的约束。 

## 事务隔离级别

InnoDB中的最小任务单元被称为事务。根据不同的约束强度，事务的隔离性对应的分为4个级别。按约束强度由低到高排序，分别是：

- 未提交读(READ UNCOMMITTED) : A事务可以读取B事务未提交的修改；
- 提交读(READ COMMITTED) : A事务可以读取B事务已提交的修改；
- 可重复读(REPEATABLE READ) : A事务可以读取B事务已提交的修改，且每次读取的结果均相同；
- 串行化(SERIALIZABLE) : 事务顺序的执行，从根源上消除竟态的产生；

不难得知：随着事务隔离级别提高，数据一致性逐渐提升，但并发能力会降低。因此InnoDB默认的隔离级别是REPEATABLE READ，而非SERIALIZABLE。

## InnoDB中的锁

​		为实现事务间的隔离，InnoDB使用多种锁策略，可粗略分为悲观锁和乐观锁。

### 乐观锁 

​		实现方式为MVCC(MultiVersion Currency Control)。可以简述为事务通过读取undo log，获取数据的历史版本，从而达到无需加锁即可读数据。	

### 悲观锁   

​		根据兼容性可分为共享锁(shared-lock, S) 和排它锁(exclusive-lock, X)、意向锁共享锁(intent shared lock, IS)、意向排它锁(intent-exclusive-lock, IX)，其兼容性可见下表：

|        | S    | X    | IS   | IX   |
| ------ | ---- | ---- | ---- | ---- |
| **S**  | 兼容 | 冲突 | 兼容 | 冲突 |
| **X**  | 冲突 | 冲突 | 冲突 | 冲突 |
| **IS** | 兼容 | 冲突 | 兼容 | 兼容 |
| **IX** | 冲突 | 冲突 | 兼容 | 兼容 |

说明 : 

1. 兼容是指锁间不互斥，已存在的锁不会阻塞后续加锁；冲突是指锁间互斥，存在的锁将阻塞后续加锁；
2. 意向锁一般指表锁，获取**行锁**时会自动对表加意向锁(也即两把锁，一把行锁，一把表锁)。当需要获取**表锁**时，可通过意向锁判断是否冲突，避免要逐行判断是否有行锁存在。

### 不同功能的锁

​		按锁的功能或使用场景分类，InnoDB中的锁可描述为以下不同的锁类型：

* Record Locks(记录锁)，施加于聚簇索引(clustered index)记录。

> ​		记录锁被视为行锁。若表未显示定义主键&非空唯一键，InnoDB会创建隐藏列来作为聚簇索引，利用该聚簇索引锁住记录。即便通过非索引 or 次级索引作为查询条件，最终锁也是施加于聚簇索引记录上面的。这一点可以通过 row format 中的行锁标识印证。

* Gap Locks(间隙锁)，施加于记录之间，第一行之前或者最后一行之后。简言之：锁住无记录的开区间。

> ​		间隙锁可防止幻行(phantom row)，阻止其他事务向该区间内插入数据。不同事务的Gap Locks是可以兼容(无论是共享还是排它)，因为它们的目的是一样的。间隙锁并非在所有事务级别可用，在Read Committed级别及以下时，针对于普通的搜索和扫描不会施加间隙锁，仅在外键约束(foreign-key constraint checking)和重复键校验(duplicate-key checking)时使用。

* Next-Key Locks，施加于记录和该记录之前间隙，也就是说Next-Key Locks是Record Locks + Gap Locks的组合。

> ​		Nex-Key锁是一个组合的概念，在实际应用时，针对于最后间隙，是通过最后一条记录值 + 上确界伪记录(supremum  pseudo - record)来实现的。例如：
>
> ​		假设表中包含以下值的记录5、10、15，那么对应的间隙锁的范围可为以下：
>
> ​		（negative infinity, 5]，（5, 10]，（10，15]，(15,  positive infinity)

* Insert-Intent Locks(插入意向锁)，一种执行insert操作前施加的共享gap锁，有别于前面的所述的一般意向锁，非表级。

> ​	    插入意向锁作为意向锁是不需要显示加锁的，InnoDB会在插入操作前，自动对该行记录 + 行前gap施加插入意向锁。
>
> ​		因为是共享锁，因此对于相同gap内的插入(自动获取的gap是相同的)，但是插入记录的位置是不一样的，因此是不会冲突的。例如：
>
> ​	 			A 事务:  insert into test(id) values(2) ;    B事务 :  insert into test(id) values(3);  
>
> ​		B不会阻塞。但是以下就会冲突：
>
> ​				A事务： begin transaction;   select * from test where id > 2 for update;
>
> ​	 	 	  B事务：insert into test (id) values(3); 
>
> ​		B会阻塞。阻塞的原因在于：B在插入记录前会取获取插入意向锁，但是由于A事务已经施加了Next-Key锁，导致B无法获取该共享锁而阻塞。

* auto-inc	 Locks(自增锁)，带有AUTO_INCREMENT列的表在插入时会先获取自增锁，该锁属于表级锁。

> ​		自增锁是InnoDB表中包含自增列时所需要的锁，目的是为了保证自增列的单向增长(向一个方向增加，不能回退)。很显然，自增锁特别容易成为插入时的"热点"，更加需要注意在基于SBR方式复制下其行为影响是不一样的。因此在Innodb中，自增锁有其独特的工作模式。
>
> ​		在阐述自增锁的工作模式前，先介绍MySQL中对插入操作(insert - like)的分类：
>
> 1. simple insert Operation
>
>     插入数量可以在事前预估的插入语句。包括单行/多行插入或替换（不带子查询的）。
>
> 2. bulk insert Operation
>
>     插入数量无法在事前预估的插入语句。包括：insert ... select 、replace ... select、load data 。
>
> 3. mixed-mode insert Operation
>
>     插入数量有一个上限值。包括：insert ... on duplicate key update，还有形如insert into test(id, val) values (1, 'v1'), (null,'v2'), (null, 'v3')，指定了主键值，但不是所有的行都指定了。 
>
> ​		自增锁有三种工作模式，通过配置innodb_autoinc_lock_mode进行设置:
>
> 1. traditional mode， innodb_autoinc_lock_mode = 0
>
>     在传统模式下，所有insert-like都需要先获取特殊的一个表级Auto - Inc锁，保持到语句执行完成(而不是事务执行完成)。因此保证inc值的分配对于语句影响的记录是连续的。那么在基于SBR方式的复制下，语句重播对应的结果也是一样的。但是因为容易成为并发的热点，因此该方式并非数据库的默认配置。
>
> 2. consecutive mode，innodb_autoinc_lock_mode = 1，8.0之前默认的模式。
>
>      在连续模式下，不同的插入类型有不同的处理。但可以保证同一个语句获取的值时连续的。
>
>     ​		simple类型Insert Operation再需要获取表级锁，因为插入行数是确定的，所以通过一个mutex变量来获取所需要增长的量级，分配后即可释放，不需要等待语句执行。尽管不需要获取表级锁，但是如果当前auto-inc lock被其他事务获取，同样需要等待。
>
>     ​		bulk 类型Insert Operation获取表级锁，行为与在traditional mode下是一样的。
>
>     ​		mixed类型Insert Operation行为和simple类型一样，不过它获取的值有可能是比其真实插入的数据量要多，所以会导致出现"空洞"。不过在事务回滚的情况下也会出现"空洞"，所以并不需要特别的机制去消除这个现象。
>
> 3. interleaved mode，innodb_autoinc_lock_mode = 2，8.0版本默认的模式。
>
>     交叉模式完全舍弃了表级锁的方式，所有的语句并发的执行，只能保证自增列是唯一且单向增长的。由于多个语句交叉执行，并不保证单个语句获取的值是连续的。**在交叉模式下，基于SBR方式的复制是不安全的**。**随着将交叉模式设置为默认的工作模式，8.0也将基于RBR方式的复制设置为默认方式，而非SBR.**

## 锁在事务中的应用

​	开启一个事务不会去申请锁，只有执行具体的DML语句才会申请锁。申请锁的类型与DML的类型和事务隔离级别是相关的。

### SELECT操作使用的锁

读操作可以分为一致性无锁读(Consistent Nonlocking Reads)和锁读(Locking Reads)

### 一致性无锁读

​	即前面提到的，利用MVCC读取数据的历史版本，而非读取DB的数据快照。其读取的历史版本包含在读取点之前已提交事务的集合。

​	在READ COMMITTED 和 REPEATABLE READ 会默认的读取MVCC中的数据版本。也有不同点：READ COMMITTED 一直会读取最新的快照，而REPEATABLE READ只会读取第一次读的快照，保证多次读到的结果是一致的。

​	在SERIALIZABLE级别下的一致性无锁读，除使用唯一键&唯一查询条件之外的读操作，均会自动加Shared Next-Key Locks。

​	有一个异常点：因为可以读取到自身事务中变更的数据，会产生以下情况：

​	假如表定义&包含数据如下：

```
mysql> desc t_test;
+-------+--------------+------+-----+---------+-------+
| Field | Type         | Null | Key | Default | Extra |
+-------+--------------+------+-----+---------+-------+
| id    | bigint(10)   | NO   | PRI | 0       |       |
| val   | varchar(128) | YES  |     | NULL    |       |
+-------+--------------+------+-----+---------+-------+
mysql> select * from t_test where id > 1 and id < 5;
+----+-------+
| id | val   |
+----+-------+
|  2 | test2 |
|  3 | test2 |
|  4 | test4 |
+----+-------+
```

A、B事务按以下顺序执行，隔离级别为Repeatable Read

A事务：																

```sql
mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from t_test where id > 1 and id < 5; # 第一次查询
+----+-------+
| id | val   |
+----+-------+
|  2 | test2 |
|  3 | test2 |
|  4 | test4 |
+----+-------+
3 rows in set (0.00 sec)

mysql> update t_test set val = 'test44' where id = 4;
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql> select * from t_test where id > 1 and id < 5; # 第二次查询前事务B提交
+----+--------+
| id | val    |
+----+--------+
|  2 | test2  |
|  3 | test2  |         # 事务B已经修改了这条数据，但此处读取的是历史版本
|  4 | test44 |			# 事务A内修改的数据，对本事务可见。 但是两个组合起来却是数据库中从未存在过的状态。
+----+--------+
3 rows in set (0.00 sec)
```

B事务

```sql
mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> update t_test set val='test3' where id = '3';
Query OK, 1 row affected (0.02 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql> commit; # 在事务A第二次查询前提交
Query OK, 0 rows affected (0.01 sec)
```

结果：

​	在A第二次读取时，会读到一个数据库从未存在过的状态。因此在涉及到修改数据的事务中，需要显示的使用锁读。

### 锁读

​	锁读，顾名思义就是在获取锁的前提下，进行记录读取。InnoDB支持两种类型的锁读语句：

1. select ... LOCK IN SHARE MODE ，获取共享锁
2. select ... for update，获取排它锁

​	但是在不同的情况下，获取的锁类型也是不一样的

- 查询使用唯一键且使用唯一查询条件且记录存在 ： Record - Locks.
- 其他情况视事务的隔离级别，如果是REPEATABLE READ或SERIALIZABLE，则会加Gap-Locks 或 Next-Key Locks，否则仅仅只对扫描的记录加Record-Locks。

### UPDATE & DELETE 操作使用的锁

​	和锁读类似。但均是排它锁，且锁类型与隔离级别无关，仅和检索类型有关。

- 查询使用唯一键且使用唯一查询条件且记录存在：Exclusvice Record Locks.

- 其他情况则加Exclusvice  Gap-Locks 或Exclusvice  Next-Key Locks

    额外的，当UPDATE操作修改聚簇索引时，会隐式的在次级索引上加锁。


### INSERT操作使用的锁

​    INSERT操作会获取被插入行的排它锁，且如前面讲到，在插入前会还获取共享意向锁。

​	需要注意的是，如果INSERT操作在插入记录时发生 duplicate-key 错误，会转而获取改行共享锁。例如：

```sql
表定义：
mysql> desc t_test;
+-------+--------------+------+-----+---------+-------+
| Field | Type         | Null | Key | Default | Extra |
+-------+--------------+------+-----+---------+-------+
| id    | bigint(10)   | NO   | PRI | 0       |       |
| val   | varchar(128) | YES  |     | NULL    |       |
+-------+--------------+------+-----+---------+-------+
2 rows in set (0.00 sec)

```

​	事务A

```sql
mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> insert into t_test values(9, 'test9');
Query OK, 1 row affected (0.00 sec)

mysql> rollback;
Query OK, 0 rows affected (0.01 sec)

mysql>
```

​	事务B

```sql
mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> insert into t_test values(9,'test99');
Query OK, 1 row affected (6.82 sec)

mysql>
```

   事务C

```sql
mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> insert into t_test values(9, 'test999');
ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction
mysql>
```

​	结果：

​		A、B和C事务均插入记录，A先于B、C操作，但未提交。B、C两个因为发生duplicate - key 错误，转而去获取该行共享锁，因为A事务已经获取该行的排它锁，所以B/C均阻塞在INSERT语句。当A事务回滚后，释放了行上的排它锁。而B、C均因为对方已在行上获取了共享锁，而无法获取排它锁进入死锁状态。

​	针对于INSERT ... ON DUPLICATE KEY UPDATE，与上面不同的是，如果发生DUPLICATE KEY错误，它将获取排它锁，而非共享锁。如果是主键冲突，只需要获取Exclusvice Record Locks，如果是普通唯一键冲突，则获取Exclusvice  Next-Key Locks。

### REPLACE操作使用的锁

​	在唯一键冲突情况下，加Exclusvice Next-Key Locks，非冲突的情况下和Insert一样加Exclusvice Record - Locks.

### LOCK TABLE

​	lock table是mysql服务层的表级锁，非InnoDB层面的锁。

## 死锁处理

​	通过innodb_deadlock_detect来设置InnoDB是否主动进行死锁检测。检测到死锁后，执行的策略是通过将其中一个事务进行回滚，取决于哪个事务影响行数更少。

​	如果innodb_deadlock_detect配置被关闭，那么InnoDB将通过innodb_lock_wait_timeout来进行判断，如果锁等待超时，则会回滚事务。
