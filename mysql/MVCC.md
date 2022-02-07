多版本并发控制：Multi-Version Concurrency Control。

- MySQL里为什么要用MVCC呢？
    - 解决读写带来的问题：读已提交和不可重复读
    - 无需加锁，提高读写效率
- 什么是读已提交？
    - 事务中只能读到其他事务已提交的更新
    - 举例：
        - user表中当前id=1的数据中name=“张三”
        - 开启事务A，将“张三”改为“李四”；
        - 事务B开启，`select name from user where id=1`
            - 若事务A还未提交，此时读到的是“张三”；
            - 若事务A已提交，此时读到的是“李四”；
                
                > 若事务A未提交时，事务B读到的就是“李四”，这种情况称为发生了脏读；也就是读未提交，Read Uncommitted。
                > 
- 什么是可重复读
    - 事务中对同一行数据的多次读取结果相同，不受其他事务的影响
    - 举例
        - user表中当前id=1的数据中name=“张三”
        - 开启事务A，将“张三”改为“李四”；
        - 事务B开启，`select name from user where id=1`
            - 若事务A还未提交，此时读到的是“张三”；
            - 若事务A已提交，此时读到的还是“张三”；
- 如何解决？
    - 使用undo log和readview
- 什么是undo log？
    - 事务有ACID四大特性，其中A为原子性，也就是事务中的操作要么全部完成，要么所有操作都不受影响。
    - 为了保证事务的原子性，MySQL中对所有操作保留了历史记录，在事务回滚时使用这种日志将之前的操作撤回到未修改前。这种日志即为`undo log`。
    - mysql的每行数据中除了用户定义的字段外，还有两个隐藏字段：`trx_id`和`roll_pointer`，`undo log`中也有这两个隐藏字段。
        - `trx_id`表示该行undo log由哪个事务产生的
        - `roll_pointer`可理解为一种指针，作用是为了将同一行数据的undo log串起来，形成undo log链。
        
        ![mvcc](https://caychan.oss-cn-beijing.aliyuncs.com/blog/2022-02-07-16-37-19_ef1f5231.png)
        
- 什么是ReadView？
    - 在RC和RR隔离级别下，为了保证数据的隔离性，也就是哪个数据在哪个事务版本中可见，因此在事务执行过程中引入了ReadView的设计。
    - ReadView中记录了几个重要的数据：
        - `m_ids`：生成ReadView时活跃的事务id列表
        - `min_trx_id`：`m_ids`里最小的事务id
        - `max_trx_id`：生成ReadView时**系统应该分配的下一个事务id**
        - `creator_trx_id`：创建改ReadView的事务id
- 如何利用ReadView解决可见性？
    - 如果被访问版本的`trx_id`=`creator_trx_id`，证明该版本的数据是由当前事务更新的，可见。
    - 如果被访问版本的`trx_id`小于`min_trx_id`，表明生成该版本的数据在生成ReadView时已提交，可见。
    - 如果被访问版本的`trx_id`大于等于`max_trx_id`，表明生成该版本的数据在生成ReadView时还未开启，不可见。
    - 如果被访问版本的`trx_id`在`min_trx_id`和`max_trx_id`之间，要看`trx_id`是否在`m_ids`中
        - 若在，表明生成ReadView时该版本的事务还是活跃的，不可见；
        - 若不在，表明生成ReadView时该版本的事务已提交，可见。
- 如何解决隔离性？
    - RC
        - 每次读数据时都生成一个ReadView
        - 这样每次都能读到已提交事务的更新
        - 举例：
            - `当前trx_id`为5
            - 第一次生成的ReadView中`m_ids`为[3,5,6]
            - 第二次生成的ReadView为[5,6,7]
            - 那么第二次读取时就能够读到事务trx_id=3的更新内容，但读不到事务trx_id=6的更新内容，从而实现读已提交。
    - RR
        - 第一次读数据时才生成一个ReadView，后面一直用该ReadView
        - 由于每次读数据使用的都是同一个ReadView，所以每次可见的版本是相同的，也就是可重复读。