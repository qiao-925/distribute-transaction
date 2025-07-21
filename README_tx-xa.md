1# 此工程为分布式事务书籍源码

# tx-xa
对应书中第14章示例程序，使用Atomikos框架（springboot 2.2.6.RELEASE整合版）实现基于2PC的强一致性事务。

## 实现步骤

引入依赖：
```
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-jta-atomikos</artifactId>
        </dependency>
```

数据源替换为AtomikosDataSourceBean：（接入范围：需要执行分布式事务的节点）
@ConfigurationProperties(
prefix = "spring.jta.atomikos.datasource"
)
```
# application.properties
spring.jta.atomikos.datasource.primary.xa-data-source-class-name=...
spring.jta.atomikos.datasource.primary.unique-resource-name=ds1
spring.jta.atomikos.datasource.primary.xa-properties.url=...
spring.jta.atomikos.datasource.primary.xa-properties.user=...
# ... 其他连接池属性 ...
```

AtomikosDataSourceBean功能：
- XA连接池，存储的数据库连接资源同时包含**数据操作**和**事务管理**的功能
- 事务登记：通过threadLocal判断当前事务是否包含全局事务ID，若是，登记到全局事务中
    - 全局事务信息存储方式（内存threadLocal, 网络传输中，事务日志中）
- 本地事务行为拦截：通过代理模式和全局事务ID对分布式事务和本地事务**分流**

使用事务注解：
@Transactional(rollbackFor = Exception.class)

## 事务运行流程

1. 依赖与自动配置
   依赖加载：项目启动时，Maven/Gradle加载了spring-boot-starter-jta-atomikos依赖。
   自动配置触发：Spring Boot检测到这个依赖，其JtaAutoConfiguration被激活。它知道需要配置一个JTA分布式事务环境，而不是默认的本地事务环境。

2. 数据源替换与初始化：
   Spring Boot读取application.properties文件，找到所有以spring.jta.atomikos.datasource为前缀的配置。
   它为primary (ds1) 和 secondary (ds2) 分别创建并配置了两个AtomikosDataSourceBean实例。
   在初始化过程中，每个AtomikosDataSourceBean都创建了一个XA连接池。池中的每个连接资源都捆绑了一个物理数据库连接（用于数据操作）和一个XAResource对象（用于事务管理），做好了参与分布式事务的准备。

3. 事务的开启 (方法调用)
   注解触发AOP：业务代码调用了createOrder()方法。由于该方法被@Transactional注解标记，Spring的AOP代理会拦截此次调用。
   开启全局事务：AOP代理请求事务管理器开启事务。由于环境已被配置为JTA，它会调用JtaTransactionManager。该管理器随即委托底层的Atomikos事务管理器，由它创建一个全局唯一的事务ID (GTID)，并将其存入当前线程的ThreadLocal中。一个全局事务就此正式拉开序幕。

4. 执行与协同 (SQL操作)
   第一个数据库操作 (创建订单)：
   JPA/MyBatis需要一个连接来操作订单库。它向名为primary的AtomikosDataSourceBean请求连接。
   AtomikosDataSourceBean执行其核心功能——事务登记：它检查ThreadLocal，发现了活动的GTID。于是，它从XA连接池中取出一个连接，通过其XAResource将此连接登记到该GTID代表的全局事务中。
   它返回一个代理连接给JPA。JPA使用此代理连接执行INSERT语句。
   第二个数据库操作 (扣减库存)：
   流程与上一步完全相同，但这次是向名为secondary的AtomikosDataSourceBean请求连接。
   这个连接同样被登记到同一个GTID的全局事务中，并返回一个代理连接。JPA用它执行UPDATE语句。
   本地事务行为拦截：在执行过程中，如果JPA尝试在任何一个代理连接上调用commit()，这个调用会被代理对象拦截并忽略。这是为了保证事务的最终决策权完全掌握在Atomikos协调者手中。

5. 事务的终结 (提交或回滚)
   现在，createOrder()方法即将执行完毕，进入最终裁决阶段。
   场景A：方法成功执行完毕
   AOP代理通知JtaTransactionManager：“事务可以提交了。”
   Atomikos协调者启动两阶段提交 (2PC)：
   Prepare阶段：它向订单库和库存库的XAResource同时发出“准备”指令。两个数据库各自执行本地事务，锁定资源，并将“可以提交”的就绪状态写入自己的事务日志，然后向Atomikos报告“准备完毕”。
   Commit阶段：在收到所有参与者的“准备完毕”确认后，Atomikos向它们发出最终的“提交”指令。两个数据库同时正式提交事务，释放资源。数据永久生效。
   场景B：方法执行中抛出异常
   AOP代理捕获到异常。根据@Transactional(rollbackFor = Exception.class)的规则，它决定需要回滚。
   AOP代理通知JtaTransactionManager：“事务需要回滚。”
   Atomikos协调者立即向所有参与者（无论它是否已报告“准备完毕”）发出“回滚”指令。
   订单库和库存库根据各自的事务日志，撤销在此事务中所做的所有修改。系统数据恢复到createOrder()方法调用之前的状态。

## 事务上下文传播和数据源配置

### 一个疑问
单个节点是否要配置所有事务相关的数据源？

结论：只需配置当前节点操作的数据源即可，隐藏的事务上下文传播机制已经做好了分布式的协同。

### 事务上下文传播 —— 隐性的分布式协同机制

JTA规范：
javax.transaction.TransactionManager接口必须提供suspend()和resume(Transaction tx)这两个方法。
这是JTA为上下文传播提供的最核心、最直接的API支持。
suspend()：是导出的标准动作。它定义了如何安全地从当前线程获取事务身份并暂停其在本地的活动。
resume()：是导入的标准动作。它定义了如何将一个外来的事务身份赋予当前线程，使其“继承”该事务。

JTA拦截器：
在调用方，JTA拦截器在发送网络请求前，从当前线程导出事务ID，并将其注入到请求的元数据（如HTTP Header或RPC Attachment）中；在被调用方，拦截器从元数据中提取该ID，并将其恢复到当前线程，从而将跨网络的操作链接到同一个全局事务中。

Atomikos开源版不支持，商业版支持。
对于单节点内跨多数据源事务： 这种情况是Atomikos的经典应用场景，它完美支持单节点内的跨多数据源事务。这个过程不涉及任何网络间的上下文传播，自然也就不需要任何拦截器。所有的协调都在同一个JVM进程内完成。

## 事务协调管理器（恢复端口）—— 不可见的2PC实现（业务执行和事务执行的时序分离）

A B两个节点处于一个分布式事务中

A中的业务方法被调用，事务协调管理器启动一个全局事务，将本地事务注册到全局事务中。

当B收到调用时，事务上下文将全局事务ID传播到B节点，B节点使用恢复端口和TCP通信，向A发送后台注册消息（加入全局事务）。

当A节点的业务方法执行完成后（所有节点的调用已完成，但因为使用了新的数据源管理器对本地事务进行拦截，实际上并未真正的执行）

开启2PC的第一阶段：

根据全局事务的参与者列表，对本地事务和其他节点的参与者发起prepare调用（其他节点使用TCP通信发送prepare指令），
各节点数据库执行redo log,undo log以及资源锁定并通过TCP连接返回准备结果。

协调器收到所有结果后，先写入磁盘上的事务日志，然后发起commit或者rollback的本地调用和指令调用。

总结：
在分布式事务中
- 业务方法的执行更像是一种声明，并未真正执行
- 基于恢复端口的指令通信步骤才是2PC实现的实现
- 业务方法执行和事务执行的时序分离机制则是对本地事务的拦截











