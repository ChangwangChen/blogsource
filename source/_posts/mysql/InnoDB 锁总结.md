---
title: InnoDB 锁总结
date: 2020-08-20 13:50:45
categories: MySQL
tags: [MySQL]
---

本文中谈论的`锁(LOCK)`, 都是在 MySQL 事务中使用的锁.

### 锁的类型

#### 1. 行级锁

InnoDB 存储引擎实现了以下标准的行级锁:
 
 - `共享锁(S Lock)`, 允许事务读取一行数据
 - `排他锁(X Lock)`, 允许事务修改或者删除一行数据

行级锁的兼容情况如下:

| | X | S |
|:--:|:--:|:--:|
|X|不兼容|不兼容|
|S|兼容|兼容|

X 锁和任何的锁都不兼容, S 锁和 S 锁互相兼容. `X 锁和 S 锁都是行级锁, 兼容是指对同一行记录锁的兼容性情况`.

#### 2. 意向锁(表级锁)

InnoDB 存储引擎支持`多粒度(granular)` 锁定, 这种锁定允许事务在行级上面和表级上面锁同时存在. 为了支持在不同的粒度上面加锁, InnoDB 存储引擎支持额外的加锁方式: `意向锁(Intention Lock)`.

InnoDB 在实现意向锁的时候比较简练, 其意向锁即为表级锁. 设计的目的主要是为了在一个事务中揭示下一行将被请求的锁类型. 其支持 2 中意向锁:

- `意向共享锁 (IS Lock)`, 事务想要获得一张表中某几行的共享锁
- `意向排他锁 (IX Lock)`, 事务想要获得一张表中某几行的排他锁

```
注意: 意向锁不会阻塞除全表锁意外的任何请求.
```

`意向锁`与`表级读写锁`的兼容性如下所示:

|  | IS | IX | S | X |
|:-:|:-:|:-:|:-:|:-:|
| IS | 兼容 | 兼容 | 兼容 | 不兼容 |
| IX | 兼容 | 兼容 | 不兼容 | 不兼容 |
| S | 兼容 | 不兼容 | 兼容 | 不兼容 |
| X | 不兼容 | 不兼容 | 不兼容 | 不兼容 |

#### 3. 一致性非锁定读和一致性锁定读

- 一致性非锁定读

InnoDB 通过 MVCC(Multi Version Concurrency Control) 的方式来读取当前执行时间数据库中行的数据. `如果读取的行上面有 X 锁(正在执行删除或者更新), 不会等待 X 锁的释放`. InnoDB 在这种情况下会读取行的快照数据.

InnoDB 在 `读已提交(Read Committed)` 和 `可重复读(Repeatable Read)` 的事务隔离级别下, 都会使用一致性非锁定读. 然而在这两种隔离级别下快照数据的作用不一样:

1. Read Committed

    读取的永远是最新的快照数据.

2. Repeatable Read

    读取的是事务开始时, 第一次读取的快照数据.


- 一致性锁定读

某些情况下, 用户需要显示的对数据库读取操作进行加锁以保证数据逻辑的一致性, 这时候数据库会强制执行一致性锁定读.

InnoDB 对 SELECT 语句支持 2 种一致性锁定读的操作:

1. `SELECT ... FOR UPDATE`, 加 X 锁
2. `SELECT ... LOCK IN SHARE MODE`, 加 S 锁

加 S 锁可以和其他加 S 锁的事务并发, X 锁则会阻塞.

```
注意: 
1. 即使使用了一致性锁定读, 其他执行一致性非锁定读的语句, 是可以进行读取的
2. 一致性锁定读的语句, 必须在一个事务中使用, 事务提交锁也会自动释放. 
```

#### 4. 锁的算法

InnoDB 引擎中有 3 种行锁的算法, 分别是:
- Record Lock: 单个行记录上的锁
- Gap Lock: 间隙锁, 锁定一个范围, `但是不包含记录本身`
- Next-Key Lock: Gap Lock + Record Lock, 锁定一个范围, `并且包含记录本身`

Next-Key Lock 是为了解决 Phantom Problem(幻读问题), 其应用举例:
```
例: 在索引上有 10, 11, 13 和 20, 则:

 Next-Key Locking 的区间为:
(-∞, 10], (10, 11], (11, 13], (13, 20] 和 (20, +∞).

Gap Lock 的区间为:

(-∞, 10), (10, 11), (11, 13), (13, 20) 和 (20, +∞).
```


`如果此索引具有唯一性(唯一索引), Next-Key Lock 会降级为 Record Lock, 即只会锁住记录本身. 如果在主键或者唯一索引上面加了 X 锁, 也只会锁住相应的记录, 不会锁住范围.`

对于 InnoDB 引擎锁的选择, 总结如下:
1. 加锁语句使用的列没有索引, InnoDB 会使用表锁
2. 加锁语句使用到的列上的索引具有唯一性(主键或者唯一索引), Next-Key Locking 会锁住当前的记录, 降级为记录锁
3. 加锁语句使用到列上的索引不是唯一的, 那么会对加上 Next-Key Lock, 范围是 (previous-record, current record], 并且会对下一个区间加上 Gap Lock (`这个锁应该是一个 Share Lock, 因为可以读取, 但是不允许插入, 修改和删除`)

#### 5. 幻读问题的解决

InnoDB 引擎的默认事务隔离级别是: REPEATABLE READ. 在此级别下, InnoDB 使用 Next-Key Locking 机制来避免出现 Phantom Problem(幻读问题). 

注意:
- MVCC 机制解决了读取数据的问题, 在事务的过程中, 第一次读取数据的时候, InnoDB 会记录读取时间, 所有以后的读取, 都不会出现超过此时间的快照数据
- Next-Key Locking 机制, 对范围加锁的方式, 阻止了其他事务的写入, 也防止了幻读的发生
















