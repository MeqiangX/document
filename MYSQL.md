# MYSQL

## 事务隔离级别

1. 事务的特性：

   原子性(A)：要么成功，要么失败，执行过程不会被打断。

   一致性(C)：执行前后数据的完整性保持一致。

   隔离性(I)：一个事务的执行过程，不会被其他事务干扰。

   持久性(D)：事务结束后，数据永久更新到数据库

    ACID

2. 事务隔离级别：

   > 事务的隔离级别控制的是 不同事务之间的影响，也是我们能控制的唯一特性，客户端和服务端连接后，就会产生一个会话session，任何交互都是基于会话的，理论上来说，如果服务器一次只接收一个会话，等会话处理完提交后，才继续处理下一个，这样就保证了隔离性，但是这样服务端一次只能处理一个连接，性能可想而知，所以才有了事务的隔离级别给我们控制

   脏写：

   | sessionA                                   | sessionB                                   |
   | ------------------------------------------ | ------------------------------------------ |
   | begin                                      |                                            |
   |                                            | begin                                      |
   |                                            | Update student set name = 'B' where id = 2 |
   | update student set name = 'A' where id = 2 |                                            |
   |                                            | rollback                                   |
   | Commit                                     |                                            |

   多个会话操作更新一条数据，而各个会话之间相互可见影响，sessionB的提交回滚，把sessionA的修改也一并回滚了，那么当sessionA提交事务后，发现数据没有改变，便造成了脏写

   脏读：

   | sessionA                                                     | sessionB                                   |
   | ------------------------------------------------------------ | ------------------------------------------ |
   | begin                                                        |                                            |
   |                                                              | begin                                      |
   |                                                              | update student set name = 'B' where id = 2 |
   | select * from student where id = 2（此时读取到sessionB的更新结果，说明发生了脏读） |                                            |
   | commit                                                       |                                            |
   |                                                              | Rollback                                   |

   各个会话之间相互可见影响，sessionA会话读取到了sessionB的为提交数据，而后sessionB进行了回滚，结果发现sessionA读取到了错误的数据，read uncommit 读未提交。脏读

   不可重复读：

   | sessionA                                                     | sessionB                                   |
   | ------------------------------------------------------------ | ------------------------------------------ |
   | begin                                                        |                                            |
   |                                                              | begin                                      |
   | select * from student where id = 2（查询出来的name = 'A'）   |                                            |
   |                                                              | update student set name = 'B' where id = 2 |
   |                                                              | Commit;                                    |
   | select * from student where id = 2 （查询出来的name = 'B'，同一个事务中，两次查询出来的结果不同，被称为不可重复读） |                                            |
   | commit                                                       |                                            |

   多个会话相互影响，这里的会话是已提交过的事物才对其他会话造成影响，sessionA未提交前，sessionA查询的都是为修改的数据，提交后，sessionB查询的是修改并提交后的数据，被称为不可重复读，不会读取到其他事物未提交的数据

   幻读：

   | sessionA                                                     | sessionB                                              |
   | ------------------------------------------------------------ | ----------------------------------------------------- |
   | begin                                                        |                                                       |
   |                                                              | begin                                                 |
   | select * from student where name = 'A'（假设此时读到了一条记录） |                                                       |
   |                                                              | insert into student (name,age,grade) values('B',18,3) |
   |                                                              | commit                                                |
   | select * from student where name = 'B' （如果此时读取到了两条记录，则为幻读） |                                                       |
   | commit                                                       |                                                       |

   类似不可重复读

   > 并发执行事务时，会出现以上四种问题，按照严重情况来排序是：脏写>脏读>不可重复读>幻读
   >
   > sql标准中，定义了四种隔离级别，来解决这四个问题：
   >
   > 1、读未提交（READ UNCOMMITTED）：最低的隔离级别，会有脏读，不可重复读，幻读三个问题。
   >
   > 2、读已提交（READ COMMITTED）：sqlserver默认隔离级别，会有不可重复读，幻读两个问题。
   >
   > 3、可重复读（REPEATABLE READ）：避免脏读，不可重复读，会有幻读问题，mysql的默认隔离级别，但是在mysql中，此隔离级别解决掉了幻读的问题。
   >
   > 4、串行化（SERIALIZABLE）：解决所有问题
   >
   > 脏写的问题太严重，所以任何隔离界别下，都不会也不允许有这个问题出现

## MVCC

​		Multi-Version Concurrency Control，多版本并发控制，mysql利用mvcc判断在一个事务中，哪个数据能被读取，哪个数据不能被读取。

​		数据库的记录存储分为两部分，一部分是具体数据，一部分是版本控制的属性数据（隐藏），三个字段：

- row_id：非必需，如果表里有主键或unique键，则不会添加

- Transaction_id事务id：必需，表示这行数据是由哪个事务id创建的

- Roll_pointer回退指针：必需，指向这行数据的上一个版本

  | --     | --             |              |      |
  | ------ | -------------- | ------------ | ---- |
  | Row_id | Transaction_id | Roll_pointer | Data |

  事务id-trx_id，当我们开启一个事务时，并不会马上获得事务id，在执行select语句时，是没有事务id的（事务id为0），只有当执行insert/update/delete语句时，才能获得事务id。

  和MVCC紧密相关的是transaction_id和roll_pointer两个字段

  <img src='/Users/xiaoqiang/Pictures/mvcc.jpg'></img>

  ​	如上图，第一次插入为trx_id=9的那行数据，可以看到它的trx_id为空，事务已经提交完了，其余两条是后面其他的会话提交的修改事务，可以看出roll_pointer指向上一个事务提交的旧版本，这个结构类似于单向链表，这就符合“多版本并发控制”里“多版本”的概念，而兵法控制是如何体现的呢，这里引出了一个新概念：ReadView

  ​	对于READ UNCOMMITTED，可以读取到其他事务还没提交的数据，所以直接把这行数据的最新版本读出来就可以了，对于SERIALIZABLE，用行锁的方式来访问记录。

  ​	那么就只剩下READ COMMITTED和REPEATABLE READ了，这两个事务隔离级别都要保证读到的数据是其他事务已经提交的，那么问题就是-“我到底可以读取到这个数据的哪个版本”，为了解决这个问题，ReadView就出现了，包含下面四个内容：

  ​	1、m_ids：表示在生成ReadView时，系统中活跃的事务id集合。

  ​	2、min_trx_id：表示在生成ReadView时，系统中活跃的最小事务id，也是m_ids中的最小值。m_ids是活跃事务，也就是未提交事务列表的最小值。

  ​	3、max_trx_id：表示在生成ReadView时，系统应该分配给下一个事务的id。

  ​	4、creator_trx_id：表示生成该ReadView的事务id。

  > 如果被访问的版本的trx_id和ReadView中的creator_trx_id相同，证明生成当前版本就是由你造成的（可能是m_ids未提交列表中的某一个，对自己是可见的），所以该版本是可以读出来的。
  >
  > ​	如果被访问版本的trx_id小于ReadView的min_trx_id的值，表示当前版本的事务已经提交了，所以也是可以读出来的。因为mix_trx_id表示的是下一个将被提交的事务id，所以它的上一个就是已经提交的事务。
  >
  > ​	如果被访问的版本的trx_id在min_trx_id和max_trx_id之间，那需要判断trx_id是否在m_ids中，如果在活跃事务列表中，那是该版本的事务就是未提交的，所以该版本数据不可读取；如果该版本的trx_id不再m_ids中，表示事务已经被提交，不在未提交列表中，即改版本是可以被读取的。
  >
  > ​	如果被访问的版本的trx_id大于或者等于max_trx_id，那么表示生成该版本的事务在当前事务生成ReadView之后才开启，即也是不可见的。

  图示：

  <img src='/Users/xiaoqiang/Pictures/readview.png'></img>

  ​	以READ COMMITTED和REPEATABLE READ为例：

  ​	1、READ COMMITTED——每次读取数据都会创建ReadView

  ​		假设现在系统只有一个活跃的事务T，事务id是100，事务中修改了数据，但是还没提交，形成的版本链如下图：

  <img src='/Users/xiaoqiang/Pictures/readviewdemo.png'></img>

  ​	

  ​	现在事务A启动，执行select语句，此时创建一个ReadView，m_ids=[100]，min_trx_id = 100，max_trx_id = 101，creator_trx_id = 0；

  ​	事务A执行查询语句：1、取最新的数据版本，name=梦境地底王，trx_id=100，trx_id在m_ids里面，说明当前事务是未提交事务（活跃事务），即这个版本是由还没提交的事务创建的，所以这个版本不可见。2、顺着roll_pointer找上个版本，trx_id=99，不在m_ids里面，trx_id < min_trx_id，所以当前版本的事务是已提交的，所以该版本可见，故事务A执行后，读取到的是trx_id=99 name=地底王 这条记录。

  2、REPEATABLE READ——首次读取事务会创建ReadView

  还是上面那个例子，REPAETABLE READ第一次读取创建的ReadView是和READ COMMITTED是一样的，读取的也是trx_id=99这条，但是当事务A再次执行这条查询语句时（第一次查询未提交），不会再次生成ReadView，用的还是上次的ReadView，所以判断流程还是和上面一样的，读取到的数据还是trx_id=99这条

## 索引

