---
layout: post
title: 【Mysql】分布式锁的实现
---







> 分布式知识是考验一个程序员知识面广度和深度很好的度量标准，而分布式锁又是其中非常重要的一个知识点。可以说面试只要谈到分布式，没有不问分布式锁的。



## 分布式锁的5大特性

![image-20230310131145147](../assets/Ml_ALS.assets/image-20230310131145147.png)

![image-20230310131206980](../assets/Ml_ALS.assets/image-20230310131206980.png)

![image-20230310131253643](../assets/Ml_ALS.assets/image-20230310131253643.png)

![image-20230310131303095](../assets/Ml_ALS.assets/image-20230310131303095.png)

![image-20230310131307510](../assets/Ml_ALS.assets/image-20230310131307510.png)



## 分布式锁场景

关于分布式锁使用的场景大体分成两类：逐一执行、唯一执行；

### 逐一执行

逐一执行顾名思义，就是多台服务器同一时间只有一台服务器执行，其他服务器等待，当执行服务器执行完成后其他服务器会继续抢占执行。

例如：grus中的worker消费redis队列就是逐一执行；

### 唯一执行

唯一执行类似于leader的概念，多台服务器同一时间只有一台执行，其他服务器放弃执行

例如：grus中master定时查看Task列表，找出长时间不更新的作业；




## 基于记录

**适用于唯一执行场景**

要实现分布式锁，最简单的方式可能就是直接创建一张锁表，然后通过操作该表中的数据来实现了。当我们想要获得锁的时候，就可以在该表中增加一条记录，想要释放锁的时候就删除这条记录。

为了更好的演示，我们先创建一张数据库表，参考如下：

```sql
CREATE TABLE `database_lock` (
	`id` BIGINT NOT NULL AUTO_INCREMENT,
	`resource` int NOT NULL COMMENT '锁定的资源',
	`description` varchar(1024) NOT NULL DEFAULT "" COMMENT '描述',
  `updatetime` TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
	PRIMARY KEY (`id`),
	UNIQUE KEY `uiq_idx_resource` (`resource`) 
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='数据库分布式锁表';

```

**其中update_time是用来判断加锁时间，用于后续定时任务解决使用！**

当我们想要获得锁时，可以插入一条数据：

```sql
INSERT INTO database_lock(resource, description) VALUES (1, 'lock');
```

注意：在表database_lock中，resource字段做了**唯一性约束**，这样如果有多个请求同时提交到数据库的话，数据库可以保证只有一个操作可以成功（其它的会报错：ERROR 1062 (23000): Duplicate entry ‘1’ for key ‘uiq_idx_resource’），那么我们就可以认为操作成功的那个请求获得了锁。

当需要释放锁的时，可以删除这条数据：

```sql
DELETE FROM database_lock WHERE resource=1;
```

这种实现方式非常的简单，但是需要注意以下几点：

1. 这种锁没有失效时间，一旦释放锁的操作失败就会导致锁记录一直在数据库中，其它线程无法获得锁。这个缺陷也很好解决，比如可以做一个定时任务去定时清理。
2. 这种锁的可靠性依赖于数据库。建议设置备库，避免单点，进一步提高可靠性。
3. 这种锁是非阻塞的，因为插入数据失败之后会直接报错，想要获得锁就需要再次操作。如果需要阻塞式的，可以弄个for循环、while循环之类的，直至INSERT成功再返回。
4. 这种锁也是非可重入的，因为同一个线程在没有释放锁之前无法再次获得锁，因为数据库中已经存在同一份记录了。想要实现可重入锁，可以在数据库中添加一些字段，比如获得锁的主机信息、线程信息等，那么在再次获得锁的时候可以先查询数据，如果当前的主机信息和线程信息等能被查到的话，可以直接把锁分配给它。

**注意：假设应用抢占锁后宕机，此时无法释放锁，我们在所有服务上都做一个定时任务去查询database_lock表，根据当前时间减去上锁时更新的updatetime事件，如果相差大于我们规定的过期时间则将此条数据删除即可！**

## 乐观锁

**适用于唯一执行场景**

顾名思义，系统认为数据的更新在大多数情况下是不会产生冲突的，只在数据库更新操作提交的时候才对数据作冲突检测。如果检测的结果出现了与预期数据不一致的情况，则返回失败信息。

乐观锁大多数是基于数据版本(version)的记录机制实现的。何谓数据版本号？即为数据增加一个版本标识，在基于数据库表的版本解决方案中，一般是通过为数据库表添加一个 “version”字段来实现读取出数据时，将此版本号一同读出，之后更新时，对此版本号加1。在更新过程中，会对版本号进行比较，如果是一致的，没有发生改变，则会成功执行本次操作；如果版本号不一致，则会更新失败。

为了更好的理解数据库乐观锁在实际项目中的使用，这里就列举一个典型的电商库存的例子。一个电商平台都会存在商品的库存，当用户进行购买的时候就会对库存进行操作（库存减1代表已经卖出了一件）。我们将这个库存模型用下面的一张表optimistic_lock来表述，参考如下：

```sql
CREATE TABLE `optimistic_lock` (
	`id` BIGINT NOT NULL AUTO_INCREMENT,
	`resource` int NOT NULL COMMENT '锁定的资源',
	`version` int NOT NULL COMMENT '版本信息',
	`created_at` datetime COMMENT '创建时间',
	`updated_at` datetime COMMENT '更新时间',
	`deleted_at` datetime COMMENT '删除时间', 
	PRIMARY KEY (`id`),
	UNIQUE KEY `uiq_idx_resource` (`resource`) 
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='数据库分布式锁表';
```

其中：id表示主键；resource表示具体操作的资源，在这里也就是特指库存；version表示版本号。

在使用乐观锁之前要确保表中有相应的数据，比如：

```sql
INSERT INTO optimistic_lock(resource, version, created_at, updated_at) VALUES(20, 1, CURTIME(), CURTIME());
```

如果只是一个线程进行操作，数据库本身就能保证操作的正确性。主要步骤如下：

1. STEP1 - 获取资源：SELECT resource FROM optimistic_lock WHERE id = 1

2. STEP2 - 执行业务逻辑
3. STEP3 - 更新资源：UPDATE optimistic_lock SET resource = resource -1 WHERE id = 1

然而在并发的情况下就会产生一些意想不到的问题：比如两个线程同时购买一件商品，在数据库层面实际操作应该是库存（resource）减2，但是由于是高并发的情况，第一个线程执行之后（执行了STEP1、STEP2但是还没有完成STEP3），第二个线程在购买相同的商品（执行STEP1），此时查询出的库存并没有完成减1的动作，那么最终会导致2个线程购买的商品却出现库存只减1的情况。在引入了version字段之后，那么具体的操作就会演变成下面的内容：

1. STEP1 - 获取资源： SELECT resource, version FROM optimistic_lock WHERE id = 1
2. STEP2 - 执行业务逻辑
3. STEP3 - 更新资源：UPDATE optimistic_lock SET resource = resource -1, version = version + 1 WHERE id = 1 AND version = oldVersion

其实，借助更新时间戳（updated_at）也可以实现乐观锁，和采用version字段的方式相似：更新操作执行前线获取记录当前的更新时间，在提交更新时，检测当前更新时间是否与更新开始时获取的更新时间戳相等。

注意：乐观锁的优点比较明显，由于在检测数据冲突时并不依赖数据库本身的锁机制，不会影响请求的性能，当产生并发且并发量较小的时候只有少部分请求会失败。缺点是需要对表的设计增加额外的字段，增加了数据库的冗余，另外，当应用并发量高的时候，version值在频繁变化，则会导致大量请求失败，影响系统的可用性。我们通过上述sql语句还可以看到，数据库锁都是作用于同一行数据记录上，这就导致一个明显的缺点，在一些特殊场景，如大促、秒杀等活动开展的时候，大量的请求同时请求同一条记录的行锁，会对数据库产生很大的写压力。所以综合数据库乐观锁的优缺点，乐观锁比较适合并发量不高，并且写操作不频繁的场景。

**缺点：不适合大量请求，会涉及大量事务回滚从而导致性能瓶颈**





## 悲观锁

**适用于逐一执行场景**

除了可以通过增删操作数据库表中的记录以外，我们还可以借助数据库中自带的锁来实现分布式锁。在查询语句后面增加FOR UPDATE，数据库会在查询过程中给数据库表增加悲观锁，也称排他锁。当某条记录被加上悲观锁之后，其它线程也就无法再改行上增加悲观锁。

悲观锁，与乐观锁相反，总是假设最坏的情况，它认为数据的更新在大多数情况下是会产生冲突的。

在使用悲观锁的同时，我们需要注意一下锁的级别。MySQL InnoDB引起在加锁的时候，只有明确地指定主键(或索引)的才会执行行锁 (只锁住被选取的数据)，否则MySQL 将会执行表锁(将整个数据表单给锁住)。

在使用悲观锁时，我们必须关闭MySQL数据库的自动提交属性（参考下面的示例），因为MySQL默认使用autocommit模式，也就是说，当你执行一个更新操作后，MySQL会立刻将结果进行提交。

```sql
mysql> SET AUTOCOMMIT = 0;
Query OK, 0 rows affected (0.00 sec)
```


这样在使用FOR UPDATE获得锁之后可以执行相应的业务逻辑，执行完之后再使用COMMIT来释放锁。

我们不妨沿用前面的database_lock表来具体表述一下用法。假设有一线程A需要获得锁并执行相应的操作，那么它的具体步骤如下：

1. STEP1 - 获取锁：SELECT * FROM database_lock WHERE id = 1 FOR UPDATE;。

2. STEP2 - 执行业务逻辑。
3. STEP3 - 释放锁：COMMIT。

如果另一个线程B在线程A释放锁之前执行STEP1，那么它会被阻塞，直至线程A释放锁之后才能继续。注意，如果线程A长时间未释放锁，那么线程B会报错，参考如下（lock wait time可以通过innodb_lock_wait_timeout来进行配置）：

```sql
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
```


上面的示例中演示了指定主键并且能查询到数据的过程（触发行锁），如果查不到数据那么也就无从“锁”起了。

如果未指定主键（或者索引）且能查询到数据，那么就会触发表锁，比如STEP1改为执行（这里的version只是当做一个普通的字段来使用，与上面的乐观锁无关）：

```sql
SELECT * FROM database_lock WHERE description='lock' FOR UPDATE;
```


或者主键不明确也会触发表锁，又比如STEP1改为执行：

```sql
SELECT * FROM database_lock WHERE id>0 FOR UPDATE;
```


注意，虽然我们可以显示使用行级锁（指定可查询的主键或索引），但是MySQL会对查询进行优化，即便在条件中使用了索引字段，但是否真的使用索引来检索数据是由MySQL通过判断不同执行计划的代价来决定的，如果MySQL认为全表扫描效率更高，比如对一些很小的表，它有可能不会使用索引，在这种情况下InnoDB将使用表锁，而不是行锁。

在悲观锁中，每一次行数据的访问都是独占的，只有当正在访问该行数据的请求事务提交以后，其他请求才能依次访问该数据，否则将阻塞等待锁的获取。悲观锁可以严格保证数据访问的安全。但是缺点也明显，即每次请求都会额外产生加锁的开销且未获取到锁的请求将会阻塞等待锁的获取，在高并发环境下，容易造成大量请求阻塞，影响系统可用性。另外，悲观锁使用不当还可能产生死锁的情况。


**缺点也很明显，要考虑上锁超时所导致的报错问题。**

代码实现：

```java
package com.haizhi.grus.worker;

import lombok.extern.slf4j.Slf4j;
import org.junit.Test;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;

public class LockExample {

    @Test
    public void test() throws InterruptedException {
        new Thread(this::run, "conn1").start();
        new Thread(this::run, "conn2").start();
        new Thread(this::run, "conn3").start();
        Thread.sleep(100000);
    }

    public void run() {
        Connection conn = null;
        try {
            // 建立数据库连接
            String DB_URL = "jdbc:mysql://localhost:3306/test";
            String USER = "root";
            String PASS = "root";
            conn = DriverManager.getConnection(DB_URL, USER, PASS);

            // 开始事务
            conn.setAutoCommit(false);

            // 尝试获取锁
            PreparedStatement stmt = conn.prepareStatement("SELECT * FROM lock_table FOR UPDATE");
            ResultSet rs = stmt.executeQuery();

            if (rs.next()) {
                // 获取锁成功，执行相应的操作
                System.out.println(Thread.currentThread().getName() + ", 获取锁成功，执行相应的操作");
                if ("conn1".equals(Thread.currentThread().getName())) {
                    throw new InterruptedException(" 故意异常 ");
                } else {
                    Thread.sleep(20000);
                }
                // 提交事务，释放锁
                conn.commit();
                System.out.println(Thread.currentThread().getName() + ", 提交事务，释放锁");
            } else {
                // 获取锁失败，回滚事务
                conn.rollback();
                System.out.println(Thread.currentThread().getName() + ", 获取锁失败，回滚事务");
            }
        } catch (Throwable e) {
            e.printStackTrace();
            // 处理异常
            // ...
        } finally {
            // 关闭数据库连接
            if (conn != null) {
                try {
                    conn.close();
                } catch (SQLException throwables) {
                    throwables.printStackTrace();
                }
            }
        }
    }
}
```



## 总结

使用Mysql实现分布式锁有三种方式：唯一索引（insert、delete）、乐观锁（版本号）、悲观锁（select for update）。

**优点：**

- 简单，使用方便，不需要引入Redis、Zookeeper等中间件

**缺点：**

- 不适合高并发的场景
- db操作性能较差