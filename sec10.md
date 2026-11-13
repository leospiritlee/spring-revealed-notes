# 第五部分　事务管理

---

### 第17章 有关事务的楔子

#### 一句话主旨

事务是"以可控方式对数据资源进行访问的一组操作"，它存在的唯一目的是让系统状态在并发与故障之下仍然"正确"；ACID 是四条限定，隔离级别是隔离性上唯一可调的旋钮。

#### 事务的四个特性（ACID）各自防的是什么

- **原子性（Atomicity）**：防"做了一半"。事务内的操作是不可分割的整体，要么全部提交成功，要么其中一个失败就全部失败。作者借《士兵突击》的"钢七连"打比方——口号得从"不抛弃、不放弃"改成"要么不抛弃，要么就全部放弃"。
- **一致性（Consistency）**：防"业务规则被破坏"。事务执行前后，数据资源都必须处于满足一致性检查的状态。证券账户的例子：银行 8 万 + 证券 2 万 = 10 万，从银行转 5 万到证券之后，必须是 3 万 + 7 万 = 10 万；总和变了，就是不一致。
- **隔离性（Isolation）**：防"并发事务互相踩"。它规定多个事务同时访问同一数据资源时相互影响的程度，具体影响程度由隔离级别（Isolation Level）决定——这是 ACID 里唯一"可调"的一维。
- **持久性（Durability）**：防"提交后丢失"。事务一旦成功提交，变更即被记载且不可逆转，即使发生系统灾难也能找回；通常靠冗余存储、多数据网络备份等方式保证。

#### 并发问题与隔离级别

| 并发问题 | 含义 | READ_UNCOMMITTED | READ_COMMITTED | REPEATABLE_READ | SERIALIZABLE |
| --- | --- | --- | --- | --- | --- |
| 脏读 Dirty Read | 读到别的事务尚未提交的更新；对方一回滚，你读到的就是脏数据 | 无法避免 | 可避免 | 可避免 | 可避免 |
| 不可重复读 Non-Repeatable Read | 同一事务内两次读同一笔数据，结果不同（别人 update 并提交） | 无法避免 | 无法避免 | 可避免 | 可避免 |
| 幻读 Phantom Read | 同一查询在一个事务内多次执行，结果集条数不同（别人 insert/delete 并提交），针对多笔记录 | 无法避免 | 无法避免 | 无法避免 | 可避免 |
| 第一类丢失更新 | 两个事务更新同一行，先提交者的更新被后者的回滚"抹掉" | 无法避免 | 可避免 | 可避免 | 可避免 |
| 第二类丢失更新 | 两个事务都读到同一旧值，各自算完写回，后写者覆盖先写者，先写者的更新凭空消失 | 无法避免 | 无法避免 | 无法避免 | 可避免 |

前三种是书中正文逐条展开的；后两种书中未展开，按标准定义补入（第一类靠"写锁持有到事务结束"即可防，第二类必须串行化或显式加锁/版本号）。

四条隔离级别从弱到强：Read Uncommitted 以低隔离度换高性能，脏读、不可重复读、幻读一个都挡不住；Read Committed 是大部分数据库的默认级别，只挡住脏读；Repeatable Read 保证事务内对同一笔数据的读取结果一致，挡脏读和不可重复读，挡不住幻读；Serializable 最严格，所有事务依次顺序执行，什么问题都能避免，但并发度下降、吞吐上不去，所以极少使用，实践中多用较低隔离级别加并发锁。

两个方向性结论：隔离级别与系统并发性成反比、与数据一致性成正比；调参时要平衡，但"保证数据一致性的考虑，应该优于对系统性能的考虑"。另外，数据库各有默认级别（多数是 Read Committed，Hsqldb 用 Read Uncommitted，Oracle 只支持 Read Committed 和 Serializable）；指定了数据库不支持的级别，会被默认级别悄悄替换——EJB、Spring、JDBC 都只是"提要求"，最终是否照办由底层数据资源决定。

#### 本章结论

一个典型事务场景有四个参与者：Resource Manager（RM，存并管理数据资源状态，如数据库、JMS 消息服务器）、Transaction Processing Monitor（TPM/TP Monitor，协调多 RM 的分布式事务，通常由应用服务器担当）、Transaction Manager（TM，TP Monitor 的核心模块，负责多 RM 协调并提供事务界定与事务上下文传播）、Application（事务边界的触发点）。

按参与 RM 的多寡，事务分两类：多个 RM 参与的叫全局事务/分布式事务（Distributed Transaction），需 TP Monitor 用两阶段提交协调；只有一个 RM 参与的叫局部事务（Local Transaction），直接使用 RM 内置的事务支持即可，既降低复杂度又避免协调开销。**判据是"当前事务涉及几个 RM"，不是"系统里有几个 RM"**——即使系统挂了十台数据库，只要这个事务只更新一个库，它就是局部事务。

### 第18章 群雄逐鹿下的 Java 事务管理

#### 一句话主旨

Java 平台的事务 API"高中低档一应俱全"，但局部事务被绑死在具体数据访问 API 上、分布式事务只有 JTA 一条路、声明式事务又必须买 EJB 容器——能力够了，抽象没有。

#### 局部事务 vs 分布式事务

JDBC 的局部事务骨架：

```java
Connection connection = dataSource.getConnection();
connection.setAutoCommit(false);
try {
    // 数据访问
    connection.commit();
} catch (SQLException e) {
    connection.rollback();
} finally {
    connection.close();
}
```

同样的形状换个壳就出现在别处：Hibernate 用 `Session.beginTransaction()` 拿到 `Transaction`，再 `commit()` / `rollback()`；JMS 用 `con.createSession(true, Session.AUTO_ACKNOWLEDGE)` 创建事务型 Session，再 `session.commit()` / `rollback()`；JCA 则通过 `javax.resource.spi.LocalTransaction` 暴露局部事务控制，但只能与 Java EE 应用服务器集成，应用也就被绑到了那台服务器上。

共同点很致命：**事务控制语句长在数据访问 API 上**，用哪门技术就得学哪套 API，事务代码与数据访问代码必然纠缠。

分布式事务这边，标准答案是 JTA。编程式用法是 JNDI 查找 `javax.transaction.UserTransaction`，然后 `ut.begin()` / `ut.commit()` / `ut.rollback()`；在 EJB 里（BMT，Bean Managed Transaction）可以直接 `ctx.getUserTransaction()`。代价很直观：一个"开始—结束"的简单界定，牵出 `NamingException`、`NotSupportedException`、`SystemException`、`SecurityException`、`IllegalStateException`、`RollbackException`、`HeuristicMixedException`、`HeuristicRollbackException` 七八个 checked exception。

两阶段提交（Two-Phase Commit）的代价：多 RM 场景需要 TP Monitor 协调，TM 用 2PC 保证整个事务的 ACID。作者用西式婚礼打比方——牧师是 TM，先分别问两位新人（两个 RM）"I do?"，双方都应答才宣布结为夫妻（提交成功），任一方有疑议则婚礼作废、双方回滚到单身状态。代价是额外的协调开销与复杂度。TP Monitor 实现时通常会先数一数参与事务的 RM 数目，只有一个就绕开 2PC 做优化；但即便如此，也不如直接用 RM 内置的局部事务来得简单高效。

JCA 在分布式场景里只负责"把 EIS（Enterprise Information System）接进来"，真正控制跨系统事务还是靠 JTA——**JTA 是 Java 平台上管理分布式事务的唯一标准接口**。JTA 本身只是接口规范，实现留给厂商：应用服务器自带，或独立产品如 JOTM、Atomikos、JBoss Transactions。声明式用法（CMT，Container Managed Transaction）则是在部署描述符里写 `<transaction-type>Container</transaction-type>` 与 `<trans-attribute>Required</trans-attribute>`，应用代码里一行事务代码都没有。

#### 为什么 Spring 要自己造一层抽象

作者的批评集中在四条：

1. **局部事务绑定到具体数据访问方式**。JDBC 靠 `java.sql.Connection`，Hibernate 靠 `org.hibernate.Session` 和 `Transaction`。结果是事务管理代码与数据访问代码、业务逻辑代码相互混杂，可重用性下降，事务代码在 DAO 层和 Service 层到处散落。"各种数据访问方式只提供了简单的事务 API，但没有更高层次的抽象来帮助我们隔离事务与数据访问两个方面的过紧耦合。"
2. **事务的异常处理混乱**。事务处理中的异常本质上都不可恢复，理应抛 unchecked exception 并有统一父类；现实是没有统一的事务异常体系，而且大量 API 抛的还是 checked exception，逼着客户端 catch——`UserTransaction` 就是反面教材。
3. **事务处理 API 的多样性**。开发人员对事务的需求其实只有"界定边界"一件事，却要在各种数据访问 API 与 JTA 之间徘徊；选择多，反而让优势变成繁杂。
4. **CMT 声明式事务的局限**。CMT 确实优雅，业务对象里干干净净，但它强制依赖 EJB 容器：要么花钱买应用服务器授权，要么为一个声明式事务引入本不需要的应用服务器。

由此提出四个改进方向，也就是 Spring 事务框架的设计目标：抽象局部事务以隔离耦合、把 checked 异常转译为 unchecked 并统一异常体系、统一事务界定的编程模型、让普通 POJO 也能享受声明式事务。

至于"大多数应用根本不需要分布式事务"，伏笔在第 17 章就埋好了：判据是"当前事务涉及几个 RM"。绝大多数业务方法只动一个数据源，它就是局部事务。为一个单库事务引入 TP Monitor 和 2PC，换来的只有复杂度和性能负担——这也是 Spring 默认推荐 `DataSourceTransactionManager` 的原因。

#### 本章结论

Java 平台不缺事务能力，缺的是统一抽象。四个问题对应四个设计目标，Spring 的事务抽象框架正是冲着它们来的。

### 第19章 Spring 事务王国的架构

#### 一句话主旨

Spring 把"事务界定"收敛成三个接口：`PlatformTransactionManager` 管边界、`TransactionDefinition` 管属性、`TransactionStatus` 管运行期状态；具体实现按 Strategy 插拔，公共流程由 `AbstractPlatformTransactionManager` 用模板方法固化。

#### 三个核心接口

**TransactionDefinition**：定义"这个事务长什么样"，四个属性——隔离级别（Isolation）、传播行为（Propagation Behavior）、超时时间（Timeout）、是否只读（ReadOnly）。隔离级别有五个常量：`ISOLATION_DEFAULT`、`ISOLATION_READ_UNCOMMITTED`、`ISOLATION_READ_COMMITTED`、`ISOLATION_REPEATABLE_READ`、`ISOLATION_SERIALIZABLE`；超时用 `TIMEOUT_DEFAULT`（值 -1，表示采用事务系统默认超时）。

默认实现 `DefaultTransactionDefinition` 给出全套默认值：`PROPAGATION_REQUIRED` + `ISOLATION_DEFAULT` + `TIMEOUT_DEFAULT` + `readOnly=false`。声明式事务用的 `TransactionAttribute` 继承它，只多加一个方法 `boolean rollbackOn(Throwable ex)`，回答"抛什么异常该回滚"；`DefaultTransactionAttribute` 默认对 unchecked exception 回滚，`RuleBasedTransactionAttribute` 支持一串 `RollbackRuleAttribute` / `NoRollbackRuleAttribute` 规则做匹配。

**TransactionStatus**：描述"事务跑到哪一步了"。能做三件事——查询事务状态、用 `setRollbackOnly()` 把当前事务标记为只能回滚、借助继承来的 `SavepointManager` 在当前事务里创建内部嵌套事务（`createSavepoint()` / `rollbackToSavepoint()` / `releaseSavepoint()`）。继承层次很浅：`AbstractTransactionStatus` 之下是 `DefaultTransactionStatus`（框架内部主力）和 `SimpleTransactionStatus`（目前基本只用于测试）。

**PlatformTransactionManager**：只做三件事。

```java
public interface PlatformTransactionManager {
    TransactionStatus getTransaction(TransactionDefinition definition) throws TransactionException;
    void commit(TransactionStatus status) throws TransactionException;
    void rollback(TransactionStatus status) throws TransactionException;
}
```

它是策略顶层接口，具体实施交给实现类。局部事务方向：`DataSourceTransactionManager`（JDBC/iBATIS）、`HibernateTransactionManager`、`JdoTransactionManager`、`JpaTransactionManager`、`TopLinkTransactionManager`、`JmsTransactionManager`、`CciLocalTransactionManager`；全局事务方向：`JtaTransactionManager`。数据访问技术与事务管理器一一对应；若同时用 Hibernate 和 JDBC/iBATIS，只要 SessionFactory 与 JDBC 引用同一个 DataSource，用 `HibernateTransactionManager` 就能统一管。

内部机制值得记一笔：`AbstractPlatformTransactionManager.getTransaction()` 先 `doGetTransaction()` 判断有无当前事务；有则交给 `handleExistingTransaction()` 按传播行为分支（NEVER 抛异常、NOT_SUPPORTED 挂起后返回空事务、REQUIRES_NEW 挂起再 `doBegin`、NESTED 建 Savepoint、其余复用当前事务）；无则 MANDATORY 抛异常、REQUIRED/REQUIRES_NEW/NESTED 走 `doBegin`。`commit()` 提交前先检查 rollbackOnly，被标记就转成回滚。子类只需实现 `doGetTransaction`、`isExistingTransaction`、`doBegin`、`doSuspend`、`doResume`、`doCommit`、`doRollback`、`doSetRollbackOnly`、`doCleanupAfterCompletion` 这些钩子。

#### 七种传播行为逐个解释

| 传播行为 | 当前有事务 | 当前无事务 |
| --- | --- | --- |
| `PROPAGATION_REQUIRED` | 加入当前事务 | 新建一个事务（默认值，绝大多数场景用它） |
| `PROPAGATION_SUPPORTS` | 加入当前事务 | 以无事务方式直接执行（适合查询） |
| `PROPAGATION_MANDATORY` | 加入当前事务 | 抛异常（强制调用方已开事务） |
| `PROPAGATION_REQUIRES_NEW` | 挂起外层，另起一个全新事务 | 新建一个事务 |
| `PROPAGATION_NOT_SUPPORTED` | 挂起当前事务，以无事务方式执行 | 直接执行 |
| `PROPAGATION_NEVER` | 抛异常（严禁在事务中运行） | 以无事务方式执行 |
| `PROPAGATION_NESTED` | 在当前事务内开一个嵌套子事务（靠 Savepoint） | 等同 REQUIRED，新建事务 |

两个高频混淆点：

**REQUIRES_NEW 与 NESTED**。REQUIRES_NEW 新建的事务与外层"同一档次"，地位相同，外层被挂起（suspend），二者状态各自独立、互不干扰；NESTED 建的是"寄生于"外层的子事务（sub-transaction），地位小一号，外层仍处于 active，二者共享事务状态。所以内层 NESTED 回滚可以只回到 Savepoint、外层继续走；外层一回滚，子事务也一起没了。典型用法：大事务拆成若干小事务，`A.service()` 里 `try { B.service(); } catch (Exception e) { C.service(); }`，B、C 都用 NESTED，主键冲突就换条路走。NESTED 需要实现类支持——只有 `DataSourceTransactionManager` 配合 JDBC 3.0 驱动（还要数据库和驱动配合）才支持，JTA 规范并不要求。

**SUPPORTS 与 NOT_SUPPORTED**。`A.service()` 先 update 再调 `B.service()`：B 若是 SUPPORTS，能读到 A 尚未提交的最新值（同一事务）；若是 NOT_SUPPORTED，就读不到（A 的事务还没提交，除非隔离级别是 Read Uncommitted）。

#### 本章结论

三接口 + 策略 + 模板方法。`AbstractPlatformTransactionManager` 把"判断有无事务、按传播行为分支、挂起恢复、提交前查 readOnly、回滚后清理、触发 Synchronization 回调"全部固化在 final 的 `getTransaction`/`commit`/`rollback` 里，子类只填与具体资源相关的钩子。切换事务策略通常只是换一个实现类或改一行配置：单库用 `DataSourceTransactionManager`，日后要跨库就换 `JtaTransactionManager`，应用代码不受冲击。

### 第20章 使用 Spring 进行事务管理

#### 一句话主旨

编程式事务把边界写进代码，声明式事务把边界写进元数据、由 AOP 织入；日常应选后者，而它的全部魔法不过是"一个拦截器 + 一份方法到事务属性的映射"。

#### 编程式事务

裸用 `PlatformTransactionManager` 的骨架：

```java
DefaultTransactionDefinition definition = new DefaultTransactionDefinition();
definition.setTimeout(20);
TransactionStatus txStatus = transactionManager.getTransaction(definition);
try {
    // 业务逻辑
} catch (ApplicationException e) {
    transactionManager.rollback(txStatus);
    throw e;
} catch (RuntimeException e) {
    transactionManager.rollback(txStatus);
    throw e;
} catch (Error e) {
    transactionManager.rollback(txStatus);
    throw e;
}
transactionManager.commit(txStatus);
```

好处是完全可控；坏处是太底层——光异常处理就够忙活，每个方法都抄一遍，重复代码惊人。

`TransactionTemplate` 用模板方法 + Callback 把这段固化：

```java
TransactionTemplate txTemplate = new TransactionTemplate(transactionManager);
txTemplate.execute(new TransactionCallbackWithoutResult() {
    protected void doInTransactionWithoutResult(TransactionStatus txStatus) {
        // 事务操作
    }
});
```

两个 Callback：`TransactionCallback`（有返回值）与 `TransactionCallbackWithoutResult`（无返回值）。模板捕获 Callback 里抛出的 unchecked exception 并回滚，然后原样上抛，于是你只需处理业务异常。想让事务回滚有两条路：抛 unchecked exception（checked 得先在内部转译），或 `txStatus.setRollbackOnly()`。后者有个"一箭双雕"用法——既回滚又不让异常往上冒；作者特别提醒此时一定要记日志，否则会出现"该删的数据还在、日志里啥也没有"的排查噩梦。

用 `TransactionStatus` 做 Savepoint 嵌套事务（转账时主账户失败则转备用账户）：

```java
Object savepoint = txStatus.createSavepoint();
try {
    deposit("MAIN_ACCOUNT_ID", transferAmount);
} catch (DepositException ex) {
    txStatus.rollbackToSavepoint(savepoint);
    deposit("SECONDARY_ACCOUNT_ID", transferAmount);
} finally {
    txStatus.releaseSavepoint(savepoint);
}
```

这里若改用 REQUIRES_NEW 开新事务，取款与存款就分属两个事务，直接违反 ACID。同样只有 JDBC 3.0 驱动下的 `DataSourceTransactionManager` 可用。

#### 声明式事务

XML 路线有四代写法，能力一样、配置越来越薄：`ProxyFactoryBean` + `TransactionInterceptor`（最原始，也最能看清本质）→ `TransactionProxyFactoryBean`（一站式，把拦截器收进自己肚子里）→ `BeanNameAutoProxyCreator`（自动代理，新增业务对象只需把 beanName 塞进 `beanNames`）→ Spring 2.x 的 `<tx:advice>` + `<aop:config>`。

```xml
<tx:advice id="txAdvice" transaction-manager="transactionManager">
    <tx:attributes>
        <tx:method name="getQuote*" propagation="SUPPORTS" read-only="true" timeout="20"/>
        <tx:method name="saveQuote"/>
        <tx:method name="updateQuote"/>
        <tx:method name="deleteQuote"/>
    </tx:attributes>
</tx:advice>

<aop:config>
    <aop:pointcut id="txServices"
        expression="execution(* cn.spring21.unveilspring.IQuoteService.*(..))"/>
    <aop:advisor pointcut-ref="txServices" advice-ref="txAdvice"/>
</aop:config>
```

`<tx:method/>` 只有 `name` 必填（可用 `*` 通配），其余不写就取 `DefaultTransactionDefinition` 的默认值（propagation 默认 REQUIRED、isolation 默认 DEFAULT、timeout 默认 -1、read-only 默认 false）。

注解路线：`@Transactional`（`@Target({METHOD, TYPE})`、`@Retention(RUNTIME)`、`@Inherited`），属性与 `<tx:method/>` 几乎一一对应——`propagation` / `isolation` / `timeout` / `readOnly` / `rollbackFor` / `rollbackForClassName` / `noRollbackFor` / `noRollbackForClassName`。开启只需一行：

```xml
<tx:annotation-driven transaction-manager="transactionManager"/>
```

类上的标注会被方法上的标注覆盖，公共属性提到类级别能少写很多。作者提示：**标在实现类或实现类方法上，别标接口**——用 CGLIB 生成代理时读不到接口上的注解。

XML 里 String 形式的事务属性有固定规则（由 `TransactionAttributeEditor` 解析成 `RuleBasedTransactionAttribute`）：

```
PROPAGATION_NAME,[ISOLATION_NAME],[readOnly],[timeout_N],[+Exception1],[-Exception2]
```

只有传播行为必填，逗号分隔；`+` 前缀表示抛该异常也照样提交，`-` 前缀表示抛该异常要回滚。因为 unchecked 默认就回滚，自定义规则主要用于 checked 类型的应用异常。

#### 声明式事务的本质

剥掉所有配置糖，只剩三样东西：

1. **一个 `MethodInterceptor`**——`TransactionInterceptor`。它在 `invoke()` 里 `getTransaction` → `invocation.proceed()` → 抛异常则 `rollback`、正常则 `commit`。书里给的 `PrototypeTransactionInterceptor` 就是它的原形。
2. **一份"方法 → 事务属性"的映射**——`TransactionAttributeSource`，实现有 `NameMatchTransactionAttributeSource`（按方法名）、`MethodMapTransactionAttributeSource`（按 `Method` 实例）、`AnnotationTransactionAttributeSource`（读注解）。
3. **一个 `PlatformTransactionManager`**。

剩下的都是 AOP 的活：生成代理、匹配切点、织入。所以"事务不生效"几乎总能归到两类原因——**调用没经过代理**，或者**属性没匹配上方法**。

#### 作者强调的坑

- **默认只对 RuntimeException 回滚**。`DefaultTransactionAttribute.rollbackOn()` 只认 unchecked；抛 checked exception 时事务照样提交。要改就配 `rollback-for` / `rollbackFor`，或 XML 里写 `-QuoteException`。
- **别把 target 注入给使用方**。原话是"一定不要把 `quoteServiceTarget` 当作 `quoteService` 注入"，否则拿到的是裸对象，拦截器根本没机会上场。同一机制的另一面是自调用：类内部 `this.method()` 不经过代理，事务属性形同虚设。
- **方法得是 public**。Spring AOP 基于代理，非 public 方法拦不住。
- **只读事务只是提示**。`readOnly` 是给 ResourceManager 的优化提示（hint），是否真的优化由 RM 决定，不是强制只读。
- **数据访问技术与事务管理器必须配对**。JDBC 配 `DataSourceTransactionManager`、Hibernate 配 `HibernateTransactionManager`，否则就是作者说的"让 A 公司老板管 B 公司的事，B 公司员工当然不理他"。
- **`TransactionTemplate` 的 Callback 不能声明 checked exception**。需要向上抛出原始 checked 异常时，它不如裸用 `PlatformTransactionManager`。
- **`setRollbackOnly()` 之后必须记日志**，否则事务静默回滚，排查成本极高。

#### 本章结论

日常用声明式（注解优先，配置量最小）；需要 Savepoint、特殊回滚判断这类精细控制时退回编程式，且优先 `TransactionTemplate` 而非裸用 `PlatformTransactionManager`。两者不是竞争关系，声明式的底层正是编程式 API。

### 第21章 Spring 事务管理之扩展篇

#### 一句话主旨

Spring 事务能"无侵入"地把同一个连接串起整条调用链，靠的是 ThreadLocal；能"一个接口换遍所有数据访问技术"，靠的是 Strategy——这两个模式不是装饰，而是事务框架的结构本身。

#### ThreadLocal 为什么是事务管理的基石

问题回到第 19 章：Service 层开事务，DAO 层用连接，怎么保证两个 DAO 用的是同一个 `java.sql.Connection`？最直接的办法是 connection-passing——把 Connection 当参数一路传下去。这条路走不通：DAO 的方法签名立刻被绑死在 JDBC 上，换成 Hibernate 就得改成传 Session。

Spring 的解法是把连接放到"一个统一的地方"：事务开始时取得连接并绑定到当前线程，DAO 需要时从当前线程取，事务结束时提交/回滚并解绑。这个"统一的地方"就是 `org.springframework.transaction.support.TransactionSynchronizationManager`。DAO 层通过 `DataSourceUtils.getConnection(dataSource)` 拿连接——当前线程绑了就用绑定的那个，没绑才从 DataSource 取新的。这就是"用 Spring 事务时必须走 `DataSourceUtils`"的原因（`JdbcTemplate` 内部已经这么做了）。对应的还有 `SessionFactoryUtils`（Hibernate）、`PersistenceManagerFactoryUtils`（JDO）。

于是整条调用链上没有任何一个方法多出 Connection 参数，事务边界却把跨越多个 DAO 的访问圈成了同一个事务。作者的比喻是：Connection 像大河上的一条船，从启航（事务开始）到航程结束（事务完成），沿岸都能与它打交道；发木船还是发铁轮看情况——JDBC 发 Connection，Hibernate 发 Session。

第二个理由同样重要：Connection 是有状态、非线程安全的对象，一个 Connection 对应一个事务；若多线程共享同一个 Connection，事务管理就"失控"了。ThreadLocal 天然给每个线程一份副本。

实现上要澄清一个常见误解：**数据并不存在 ThreadLocal 里**。每个 Thread 都有一个 `ThreadLocal.ThreadLocalMap` 类型的实例变量 `threadLocals`，`ThreadLocal.set()` 只是"以自己为 key，把值写进当前线程的 threadLocals"。ThreadLocal 更像一个窗口或车站，真正的"车厢"是线程自己的 threadLocals：线程是线路，threadLocals 是线上的车，ThreadLocal 是车站。

定位也要说清：ThreadLocal 与 synchronized 横向上没关系，纵向上服务于同一目的——线程安全。synchronized 是"一个人同时对付三个歹徒，靠辗转腾挪、左躲右闪"；ThreadLocal 是"拔一把猴毛变出三个分身，一人对付一个"。所以"ThreadLocal 不是用来解决共享问题的"这句话并不错——它恰恰是靠"不共享"来解决问题的；真要共享，它就不管了。

应用场景四类：管理线程安全（给每个线程一份非线程安全对象的副本）、在当前执行流程内传递数据（connection-passing 的替代）、性能优化（以空间换时间，避免排队）、per-thread singleton（初始化代价大的资源首次创建后绑到线程上反复使用）。作者用 ThreadLocal 改造了第 16 章的 `AbstractRoutingDataSource`：定义 `DataSources` 枚举（MAIN/INFO/DBLINK），由 `DataSourceTypeManager` 持有 `ThreadLocal<DataSources>` 并提供静态 `get/set/reset`，`determineCurrentLookupKey()` 直接 `return DataSourceTypeManager.get()`；业务侧一句 `DataSourceTypeManager.set(DataSources.INFO)` 就完成数据源切换，设置点可以放在线程启动处，也可以放在 AOP 拦截器里按条件判断。

#### 策略模式在事务里的应用

Strategy 模式的本意是封装一系列可以互相替换的算法逻辑，使算法的演化独立于使用它们的客户端代码。作者的反面教材是信贷系统按还款方式（等额本金、等额本息、一次还本付息）用 if-else 堆计算逻辑，并点出三个后果：客户端代码与算法逻辑混杂，复杂难维护；耦合太强，改算法或加算法都要动客户端，两者无法独立演化；同一逻辑单元里塞多种算法，只能靠多重条件语句区分。

Spring 的用法：`PlatformTransactionManager` 就是策略接口，`getTransaction`/`commit`/`rollback` 是策略的"算法签名"，`DataSourceTransactionManager`、`HibernateTransactionManager`、`JtaTransactionManager` 等是可互换的具体策略。Service 只依赖接口，所以"从单库切到分布式，只是把 `DataSourceTransactionManager` 换成 `JtaTransactionManager`"，应用代码不用改。`JtaTransactionManager` 内部又把各 JTA 厂商 `TransactionManager` 位置与能力的差异再包一层，最终委派给具体 JTA 实现——策略套策略。

再往下一层，实现类的继承体系用的是模板方法模式：`AbstractPlatformTransactionManager` 把固定流程写死在 `final` 的 `getTransaction`/`commit`/`rollback` 里，只把 `doGetTransaction`、`doBegin`、`doCommit`、`doRollback` 等钩子留给子类。**策略负责"选谁"，模板方法负责"怎么跑"**，两者合起来才是 Spring 事务框架的完整骨架。

#### 本章结论

ThreadLocal 解决"资源怎么在调用链上传递而不污染方法签名"，Strategy 解决"实现怎么替换而不影响调用方"，模板方法解决"公共流程怎么复用"。看懂这三个模式，Spring 事务框架对你就没有黑盒了；反过来，任何一个"事务不生效"的疑难，都能顺着这三条线索定位——资源有没有绑到当前线程、调用的到底是不是代理、传播行为在有无事务两种情形下分别做了什么。


---
