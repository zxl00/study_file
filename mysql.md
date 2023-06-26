## mysql主从概述

![](https://gitee.com/root_007/md_file_image/raw/master/202302071241320.png)

> mysql主从复制过程：
>
> 1. 从库生成2个线程，一个I/O线程，一个SQL线程
> 2. I/O线程去请求主库的binlog，并将得到的binlog日志写到relay log文件中
> 3. 主库会生成一个log dump线程，用来给从库I/O线程传binlog
> 4. SQL线程会读取relay log文件中的日志，并解析成具体的操作，来实现主从操作一致，最终数据一致

## mysql主从复制方式

​		binlog记录格式

- 基于行

  简介

  > 这种方式是将实际的数据记录到二进制文件中，简单说，就是更改的数据记录进去，例如：一个更新语句更新了3个表，就会记录每个表的更新

  优缺点

  > 优点：不会出现某些特定情况下的存储过程、函数、trigger的调用或者出发无法被正确复制的问题
  >
  > 缺点：会产生大量的日志，尤其是修改table的时候会让日志暴增，同时增加binlog同步时间，也不能通过解析binlog获取执行过的sql，只能看到发生的data变更

- 基于语句

  简介

  > 这种方式是举例那些可以造成数据更改的语句，例如：一个更新语句更新了3个表，就会记录这个更改3个表的语句

  优缺点

  > 优点：减少了binlog日志量，节约I/O,提高性能
  >
  > 缺点：某些情况下，会导致主从节点数据不一致，例如sleep()、now()

- 混合模式

  简介

  > 基于行和基于语句的混合，对于一般的复制使用基于语句的模式保存到binlog中，对于基于语句模式无法复制的操作则使用基于行的模式来保存，mysql会根据执行的sql语句选择日志保存方式

## mysql查询流程

![image-20230203162653302](https://gitee.com/root_007/md_file_image/raw/master/202302071242241.png)

> 这里只是简单的一个select流程，其中很多细节没有体现，执行器前的预处理、优化器等
>
> 预处理器：判断表和字段是否存在
>
> 优化器：确定执行计划
>
> 执行器：负责具体的执行

## mysql更新流程

![image-20230206095852095](https://gitee.com/root_007/md_file_image/raw/master/202302071242409.png)

> undo log：
>
> ​	属于逻辑日志，记录一个变化过程，例如执行一个delete，undolog会记录一个insert，执行一个update，undolog会记录一个相反的update
>
> ​	用于撤销回退的日志，当我们更新数据时，还没有到提交的步骤，这时mysql崩溃了，就可以利用undolog来进行回滚
>
> ​	实现事务回滚，保障事务的原子性，事务在处理过程中，如果出现了错误，或者用户执行rollback语句，mysql可以利用undolog中的历史数据将数据恢复到事务之前的状态
>
> ​	实现MVCC(多版本控制)关键因素之一，mvcc是通过readview+undolog实现的，undolog为每条记录保存多份历史数据，mysql在执行seletc语句时，会根据事务的readview里的信息，顺着undolog的版本链找到满足其可见性的记录
>
> redo log：
>
> ​	redolog是物理日志，记录了某个数据页做了什么修改，对于xxx表空间中国的yyy数据页zzz偏移量的地方做了什么aaa更新，每当执行一个事务就会产生这样的一条物理日志
>
> ​	在事务提交时，只需要将redolog持久化到磁盘即可，可以不需要将buffer里的脏页数据(加锁页)持久化到磁盘
>
> ​	redolog是循环写的方式，相当于一个环形，innodb用write pos表示redolog当前记录写到的位置，用checkpoint表示当前要擦除的位置
>
> ![image-20230206101443692](https://gitee.com/root_007/md_file_image/raw/master/202302071242447.png)

## mysql主从同步常见问题

- 主从复制延迟

  > 主库写binlog不及时
  >
  > dump线程压力过大
  >
  > IO线程阻塞
  >
  > 表缺乏主键或唯一索引
  >
  > 大事务
  >
  > 主库DML请求频繁
  >
  > 主库对大表执行DDL语句
  >
  > 从库压力过大