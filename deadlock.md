# 导表失败（耗时8分钟）

问题语句:insert into ...  on duplicate key update 

原日志： Error updating database. Cause: com.mysql.cj.jdbc.exceptions.MySQLTransactionRollbackException: Lock wait timeout exceeded; try restarting transaction

上次原因：连续多次点击导表，上一次事务1并未结束，下一次事务2获取锁时间过长而失败。

事务1Batchinsert获取当前行X锁,事务2需要检测唯一索引(uni)是否冲突获取S锁，阻塞

但是可能诱发的问题：如果多人重复导表（或者一个人多次点击导表或着一人导表一人更新活动）可能造成数据库死锁。

| 时间线 | 事务1                           | 事务2                                           | 事务3                                 |
| ------ | ------------------------------- | ----------------------------------------------- | ------------------------------------- |
| 1      | insert into t_at_active_product | insert into t_at_active_product                 | insert into t_at_active_product       |
| 2      | 获取当前行X锁                   |                                                 |                                       |
| 3      |                                 | 需要检测唯一索引是否冲突获取S锁，阻塞           | 需要检测唯一索引是否冲突获取S锁，阻塞 |
| 4      | commit;                         | 获取到到S锁                                     | 获取到S锁                             |
| 5      |                                 | 发现唯一索引冲突，执行Update语句(此时S锁未释放) | 发现唯一索引冲突，执行Update语句      |
| 6      |                                 | 获取该行的X锁，被事务3的S锁阻塞                 | 获取该行的X锁，被事务2的S锁阻塞       |
| 7      |                                 | 发现死锁，回滚该事务                            | update成功,commit;                    |

解决方法：插入的时候使用select * for update,加X锁，从而不会加S锁。

# 活动库存更新失败

活动库存没更新是因为商品没有传SKUCODE过来导致ERP无法更新库存

* and __tag__:__path__: "/home/fingo/fingo-littlec/logs/marketing/info.log" and aec4cb94-d41c-486b-b45b-2f9b335dcc3d and "c.f.l.m.m.s.SpuCountryInfoService" and 获取spu信息*
* 2020-06-05 16:30~2020-06-05 16:51



# APP商品

## 添加活动方式

- 导表
- 手动

## 活动商品位置

- 限时抢购
- 日常活动（banner 页面搭建）
- 新品上架 大马现货（页面搭建）
- 爆款好物
- 瀑布流（首页设置）

