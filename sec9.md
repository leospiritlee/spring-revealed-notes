# 第四部分　使用 Spring 访问数据

---

### 第13章 统一的数据访问异常层次体系

#### 一句话主旨

DAO 模式本想把"数据存在哪、怎么取"屏蔽在接口之后，结果 checked 的 `SQLException` 反过来把接口定义拖成了空中楼阁；Spring 用一套 unchecked 的 `DataAccessException` 体系加异常转译，把这个洞补上。

#### 为什么 JDBC 的异常处理是场灾难

作者的推演是从一个干净的 DAO 例子开始的：`ICustomerDao` 只声明 `findCustomerByPK(String)` 和 `updateCustomerStatus(Customer)`，`CustomerService` 只依赖接口，实现类从 `JdbcCustomerDao` 换成 `LdapCustomerDao` 时客户端一行不用改。这是"梦想"。

一旦把真实的 JDBC 代码填进 `JdbcCustomerDao`，问题立刻出现——`getConnection()`、`executeQuery()` 都会抛 `SQLException`，而它是 checked exception，编译器逼你处理：

- **自己吞掉？** 那客户端根本不知道数据访问出了问题。
- **往上抛？** 方法签名就得写成 `throws SQLException`，接口也跟着被污染。`ICustomerDao` 对客户端本该是通用的，现在却因为"底层用了 JDBC"而要求客户端 `catch` 一个 JDBC 特有的异常，这与 DAO 模式的初衷正好相反。
- **再来一个实现就更糟。** 加入 `LdapCustomerDao` 要抛 `NamingException`，接口只好改成 `throws SQLException, NamingException`。作者的原话是：数据访问接口的定义"现在变成了空中楼阁，我们无法最终定义并确定这个接口"。

第二层灾难在 `SQLException` 自身的设计上。JDBC 规范没有把具体异常情况子类化，而是用 `ErrorCode` 区分，可 `ErrorCode` 的规范又留给了各个数据库厂商：

- MySQL 里 `ErrorCode = 1` 表示连不上数据库，`1062` 表示主键冲突；
- Oracle 里 `17002`、`17447` 属于获取资源失败，`1`、`1400`、`1722`、`2291` 属于一致性冲突。

于是客户端拿到异常后，必须先判断"当前连的是什么库"，再 `getErrorCode()`，再对着厂商的错误码表比对，才知道到底出了什么事。作者感叹：如果当初 JDBC 规范明确规定"`ErrorCode=1` 代表数据库连接不上，`ErrorCode=2` 代表表不存在……"，今天处理 `SQLException` 也不会这么痛苦。（书里也补了一句：后来的 JDBC 规范确实吸收了 Spring 这部分经验。）

#### Spring 的解法

分两步走，一步比一步狠。

**第一步：包成 unchecked。** 既然在 DAO 内部处理不掉、直接抛又污染接口，就用 `RuntimeException` 包装后再抛（书里代码清单 13-5 就是 `catch (SQLException e) { throw new RuntimeException(e); }`），接口签名恢复干净。理由是：数据访问失败对客户端而言是系统的 **Fault**，客户端对 Fault 最有效的处理方式就是"不处理"（需要时再捕获重试），所以 unchecked 才合适。

**第二步：分类转译（Exception Translation）。** 只包一层 `RuntimeException` 还不够——错误信息的解析仍然要靠客户端按厂商分支。正确做法是把"解析"收进 DAO 或工具类，并按语义给每类错误分配一个异常类型：

- 连不上数据库、连不上 LDAP → `ResourceFailureException`
- 主键冲突、其他资源冲突 → `DataIntegrityViolationException`

这就是 Spring 现成的轮子：`org.springframework.dao.DataAccessException` 为统领（unchecked），下面按职能分家：

| 异常 | 职责 |
| --- | --- |
| `CleanupFailureDataAccessException` | 操作成功、清理资源失败（如关连接时又抛 `SQLException`） |
| `DataAccessResourceFailureException` | 拿不到数据资源，典型是数据库服务器挂了；JDBC 子类为 `CannotGetJdbcConnectionException` |
| `DataSourceLookupFailureException` | 从 JNDI 等处查找 `DataSource` 失败 |
| `ConcurrencyFailureException` | 并发失败，细分 `OptimisticLockingFailureException`、`PessimisticLockingFailureException`（再分 `CannotAcquireLockException`、`DeadlockLoserDataAccessException`） |
| `InvalidDataAccessApiUsageException` | API 用错了，如拿 `getForObject()` 去跑返回多行的 SQL |
| `InvalidDataAccessResourceUsageException` | 访问资源的方式错了，如 SQL 语法错；JDBC 子类 `BadSqlGrammarException`，Hibernate 子类 `HibernateQueryException` |
| `DataRetrievalFailureException` | 预期存在的数据取不到 |
| `PermissionDeniedDataAccessException` | 无权限 |
| `DataIntegrityViolationException` | 违反一致性约束，如主键重复 |
| `UncategorizedDataAccessException` | 兜底，`abstract`，供各技术扩展 |

作者特别提示 `DataIntegrityViolationException` 是少数值得主动捕获的：比如"插入撞主键就改成更新"，`catch (DataIntegrityViolationException e)` 后降级即可，而不像其他 `DataAccessException` 那样放任不管。

把 `SQLException` 翻译成这套体系的接口是 `org.springframework.jdbc.support.SQLExceptionTranslator`，方法签名 `DataAccessException translate(String task, String sql, SQLException ex)`——注意它连"当时在干什么任务、跑的是哪条 SQL"都一起收进来，因为报错信息里需要这些上下文。

#### 本章结论

异常层次体系是整个 Spring 数据访问层的"提纲挈领"之物。一套语义完整的 unchecked 体系定义好之后，不管底层换成 JDBC、Hibernate 还是 iBATIS，只要它把自己的异常转译进来，客户端的异常处理逻辑就"岿然不动"。

### 第14章 JDBC API 的最佳实践

#### 一句话主旨

把 JDBC 那套雷打不动的六步流程用模板方法收进一个类，把变化的部分（SQL、参数、结果映射）交给 Callback，DAO 从此只写自己真正关心的那几行。

#### JdbcTemplate 解决的三件事

作者说 `JdbcTemplate` 主要关注两件事（封装 JDBC 代码、统一异常转译），但把它从"能跑的雏形"变成"能用的 Helper"还靠第三件：

1. **资源获取与释放。** 原来每个 DAO 都要写 `con = ds.getConnection()` … `finally { con.close(); }`，还要记得 `stmt.close()`、`rs.close()`。团队里人员水平不齐，作者列了几种常见"惊喜"：`Statement` 用完不关指望 `Connection` 关时顺带关掉（并非所有驱动都这么干）；创建了多个 `ResultSet`/`Statement` 只清了最外层；最要命的是 `rs2` 遍历时手一抖写成了 `rs1`。`JdbcTemplate` 把 `DataSourceUtils.getConnection()` / `releaseConnection()` 收进模板方法，泄漏问题从"靠人守规矩"变成"结构上不可能"。
2. **异常转换。** 模板方法内 `catch (SQLException)` 后交给 `SQLExceptionTranslator`，客户端只会见到 `DataAccessException` 的子类。
3. **把"变化的部分"抽成 Callback。** 只用模板方法的话，`JdbcTemplate` 得顶着 `abstract` 帽子，每次数据访问都要写个子类——"这也实在太不地道了"。引入 `StatementCallback` 之后，它变成 `class` 而非 `abstract class`，可以独立使用。作者对二者关系的定义很准：**Callback 与模板类之间是"服务与被服务"的关系**——模板类想请 Callback 做事，就要提供资源（通过 `doInStatement` 把 `Statement` 交出去）；Callback 用完不管收尾，模板类负责关掉。因为 Java 没有 Ruby 那样的 Block，只好用匿名内部类代替。

#### 核心 API

`JdbcTemplate` 实现 `JdbcOperations`，父类 `JdbcAccessor` 提供两个关键属性：`dataSource` 和 `exceptionTranslator`。

模板方法按 Callback 暴露的 API 自由度分四组，每组有一个核心实现，其他重载最终都调到它：

- **面向 Connection**：`ConnectionCallback`，直接把 `java.sql.Connection` 交出去。自由度过大，除集成遗留系统外应避免。
- **面向 Statement**：核心是 `execute(StatementCallback)`，公开 `java.sql.Statement`，处理静态 SQL。
- **面向 PreparedStatement**：`PreparedStatementCreator` 拿 `Connection` 造 PS，再交给 `PreparedStatementCallback`；另有更轻的 `PreparedStatementSetter` 只管设占位符参数（顺带免于 SQL 注入）。
- **面向 CallableStatement**：`CallableStatementCreator` + `CallableStatementCallback`，用于存储过程。

常用的便利方法：

- 查询单值：`queryForInt`、`queryForLong`、`queryForObject(String, Class)`、`queryForMap`。若 SQL 返回多行却用了这组，会抛 `IncorrectResultSizeDataAccessException`。
- 查询多行：`queryForList(String[, Object[] args])`，返回 `List`，每个元素是一个以列名为键的 `Map`。
- 自定义结果：`query(...)` 配三种 Callback —— `ResultSetExtractor`（`extractData(ResultSet)`，自己遍历、自己组装，返回 `Object`）、`RowMapper`（`mapRow(ResultSet, int rowNum)`，只管一行，`query` 直接返回 `List`）、`RowCallbackHandler`（`processRow(ResultSet)`，只管一行，`query` 返回 `void`）。
- 更新：`update(String)`、`update(String, Object[])`、`update(String, Object[], int[])`，另有 `update(String, PreparedStatementSetter)` 和 `update(PreparedStatementCreator)`。
- 无返回值操作：`execute(String)`，适合 `create table`、`drop table`。
- 批量：`batchUpdate(String[])` 与 `batchUpdate(String, BatchPreparedStatementSetter)`（需实现 `getBatchSize()` 和 `setValues(ps, i)`）。

`RowMapper` 与 `RowCallbackHandler` 之所以能"只管一行"，是因为内部有 `RowMapperResultSetExtractor` 和 `RowCallbackHandlerResultSetExtractor` 替它们遍历——"归根结底内部工作是由 `ResultSetExtractor` 做"，前两者只是简化用法。

#### 代码要点

用 `RowMapper` 查一组顾客，这是日常最常用的形态：

```java
List customerList = jdbcTemplate.query("select * from customer", new RowMapper() {
    public Object mapRow(ResultSet rs, int rowNumber) throws SQLException {
        Customer customer = new Customer();
        customer.setFirstName(rs.getString(1));
        customer.setLastName(rs.getString(2));
        return customer;
    }
});
```

带参数的更新，`JdbcTemplate` 内部替你构造 `PreparedStatement`：

```java
int affectedRows = jdbcTemplate.update(
    "update customer set customerName=?, age=? where customerId=?",
    new Object[] { "Daniel", new Integer(36), new Integer(101) });
```

批量插入，两个方法各司其职：

```java
jdbcTemplate.batchUpdate("insert into customer value(?, ?)", new BatchPreparedStatementSetter() {
    public int getBatchSize() { return customers.size(); }
    public void setValues(PreparedStatement ps, int i) throws SQLException {
        Customer c = (Customer) customers.get(i);
        ps.setString(1, c.getFirstName());
        ps.setString(2, c.getLastName());
    }
});
```

存储过程则可以直接用 `call` 模板方法，用 `SqlParameter` / `SqlOutParameter` / `SqlInOutParameter` 声明参数表，返回 `Map`：

```java
List<SqlParameter> parameters = new ArrayList<SqlParameter>();
parameters.add(new SqlParameter(Types.VARCHAR));
parameters.add(new SqlOutParameter("SQL", Types.VARCHAR));
parameters.add(new SqlInOutParameter("COUNT", Types.INTEGER));
Map result = jdbcTemplate.call(csc, parameters);
```

#### JdbcDaoSupport 与 DataSource 的装配

`JdbcTemplate` 的兄弟有两个：`NamedParameterJdbcTemplate`（用 `:filename` 代替 `?`，内部持有一个 `JdbcOperations`，先由 `NamedParameterUtils` 解析命名参数、替换回 `?`，再委托给内部 `JdbcTemplate`——作者说它"就是在 `JdbcTemplate` 基础上添加一层解析网"），以及 `SimpleJdbcTemplate`（集前两者功能于一身，加 varargs、autoboxing、泛型，作者说它是"又在 `NamedParameterJdbcTemplate` 的基础上套了另一层网"）。命名参数由 `SqlParameterSource` 提供，实现有 `MapSqlParameterSource` 和 `BeanPropertySqlParameterSource`。

`DataSource` 是 Spring 数据访问层唯一的资源入口，分三类：简单实现（`DriverManagerDataSource` 每次给新连接、`SingleConnectionDataSource` 只维护一个 `Connection`，可 `setSuppressClose(true)`）、带连接池的实现（DBCP、C3PO，生产环境用这一类）、支持分布式事务的 `XADataSource`。取用方式分本地注入和 JNDI 远程查找（`JndiObjectFactoryBean` 或 `<jndi:lookup>`）。要自定义，可继承 `AbstractDataSource`，其中 `AbstractRoutingDataSource` 专门用于多数据源路由；要给现有 `DataSource` 加行为，则继承 `DelegatingDataSource`（`UserCredentialsDataSourceAdapter`、`TransactionAwareDataSourceProxy`、`LazyConnectionDataSourceProxy` 都是它的现成子类）。

DAO 侧的统一收口是 `org.springframework.jdbc.core.support.JdbcDaoSupport`：子类直接 `extends JdbcDaoSupport implements IDaoInterface`，用 `getJdbcTemplate()` 干活，`DataSource` 和 `JdbcTemplate` 都不必自己声明。

#### 本章结论

Spring 给了两套 JDBC 实践：基于 `JdbcTemplate` 的和基于操作对象的（`RdbmsOperation` 体系下的 `MappingSqlQuery`、`SqlUpdate`、`BatchSqlUpdate`、`StoredProcedure` 等，全部最终仍由 `JdbcTemplate` 执行）。后者在实践中并不如预想中受欢迎，除 `StoredProcedure` 确实简化了存储过程调用，其余大多可被 `JdbcTemplate` 替代——这个判断在最后一章得到了作者的明确回应。

### 第15章 Spring 对各种 ORM 的集成

#### 一句话主旨

ORM 面对的是和 JDBC 一模一样的三件事——资源、异常、事务，所以 Spring 用完全一样的套路（FactoryBean 管连接工厂、Template 管资源、Callback 管业务、DaoSupport 管继承）把它们逐一收编，差异只在类名。

#### 集成 Hibernate 的两种方式

先看"旧日的冬眠时光"。Hibernate 参考文档给的 `HibernateUtil` 用 `ThreadLocal` 管 `Session`，本身没什么可指摘；但当团队按功能模块（而非按分层）分工，每个开发人员都要同时负责自己模块的 DAO 和 Service 时，就会写出这样的 `Service`：

```java
getDao().beginTransaction();
getDao().saveOperation(mockPO);
getDao().commit();
// catch HibernateException / Exception → rollback()
// finally → getDao().closeSession();
```

事务、`Session` 资源、异常处理三件事纠缠在一处，人人都精明也可能有疏漏，于是"时不时发生资源泄漏之类的事故"。作者说他去年还在项目里见到因此导致的数据库泄漏。

**方式一：`HibernateTemplate`。** 核心模板方法是 `execute(HibernateCallback action, boolean exposeNativeSession)`，剥掉事务相关代码后极其干净：`getSession()` → `action.doInHibernate(session)` → `catch (HibernateException) convertHibernateAccessException` → `finally closeSession()`。异常转译由 `SessionFactoryUtils.convertHibernateAccessException(HibernateException)` 完成（`SQLException` 仍交给 `SQLExceptionTranslator`）。查询方法分四组：`get`（查不到返回 `null`）、`load`（查不到抛异常）、`find`/`findByNamedParam`/`findByNamedQuery`/`findByNamedQueryAndNamedParam`、`iterate`（返回 `Iterator`，大数据量时用）；更新侧是 `save`/`saveOrUpdate`/`update`/`delete`。

`SessionFactory` 由 `LocalSessionFactoryBean`（或注解版 `AnnotationSessionFactoryBean`，多出 `annotatedClasses`、`annotatedPackages`）以 FactoryBean 形式提供；Spring 刻意使用容器里的 `DataSource` 而非 Hibernate 自己的 `ConnectionProvider`，就是为了把事务纳入统一抽象。作者顺带吐槽了把 `SessionFactory` 绑到 JNDI 的做法：它持有大量应用独有的信息（比如 Domain Object），"就好像你自己家的电视机必须放到一个地方去保管，当你要看的时候，还需要到那个地方去申请领回一样"。

**方式二：`HibernateDaoSupport`。** 和 `JdbcDaoSupport` 使命相同：子类 `extends HibernateDaoSupport`，用 `getHibernateTemplate()` 即可；它还公开 `convertHibernateAccessException`，即使 DAO 里直接用原生 Hibernate API 也能享受转译。

#### 集成 iBATIS

作者对三者的比喻很传神：**Hibernate 是自动步枪，JDBC 是手动步枪，iBATIS 是半自动步枪。** iBATIS 没有对象查询语言、没有透明持久化，但学习曲线极低，"只要精通 SQL 就基本没有太多问题"。

旧用法是用 `SqlMapClient`，三种调用方式里只有自动提交那种清爽；一旦加上 `startTransaction()`/`commitTransaction()`/`endTransaction()` 和异常处理，代码就"不像开始那么清爽了"，若人人如此复制，系统稳定性堪忧。

Spring 选择基于 `SqlMapSession` 的方式集成（更灵活，可把数据源和事务管理器改为外部注入），落点是 `org.springframework.orm.ibatis.SqlMapClientTemplate`。核心是 `execute(SqlMapClientCallback)`：先 `sqlMapClient.openSession()`，再按当前事务设置决定 `Connection` 从哪来（若底层是 `TransactionAwareDataSourceProxy` 就直接 `getConnection()`，否则 `DataSourceUtils.doGetConnection()`），然后 `action.doInSqlMapClient(session)`，`catch (SQLException)` 交给 `SQLExceptionTranslator`，`finally` 里关连接、关 `SqlMapSession`。`SqlMapClientCallback` 只暴露一个 `SqlMapExecutor`，拿到它之后"在 iBATIS 的世界里面想干什么就干什么"，包括 `startBatch()` 批量。

`SqlMapClient` 由 `SqlMapClientFactoryBean` 提供，DAO 基类是 `SqlMapClientDaoSupport`，用 `getSqlMapClientTemplate()` 干活。

#### 对 JPA / JDO 的集成

一模一样的四件套，只换名字：

| | 连接工厂 | FactoryBean | Template / Callback | DAO 基类 | 异常转译工具 |
| --- | --- | --- | --- | --- | --- |
| JDBC | `DataSource` | — | `JdbcTemplate` / `StatementCallback` 等 | `JdbcDaoSupport` | `SQLExceptionTranslator` |
| Hibernate | `SessionFactory` | `LocalSessionFactoryBean` | `HibernateTemplate` / `HibernateCallback` | `HibernateDaoSupport` | `SessionFactoryUtils` |
| iBATIS | `SqlMapClient` | `SqlMapClientFactoryBean` | `SqlMapClientTemplate` / `SqlMapClientCallback` | `SqlMapClientDaoSupport` | `SQLExceptionTranslator` |
| JDO | `PersistenceManagerFactory` | `LocalPersistenceManagerFactoryBean` | `JdoTemplate` / `JdoCallback`（公开 `PersistenceManager`） | `JdoDaoSupport` | `PersistenceManagerFactoryUtils.convertJdoAccessException` |
| TopLink | `SessionFactory`（Spring 自己抽象 `ServerSession`） | `LocalSessionFactoryBean` | `TopLinkTemplate` / `TopLinkCallback` | `TopLinkDaoSupport` | `convertTopLinkAccessException` |
| JPA | `EntityManagerFactory` | `LocalEntityManagerFactoryBean` / `LocalContainerEntityManagerFactoryBean` | `JpaTemplate` / `JpaCallback`（公开 `EntityManager`） | `JpaDaoSupport` | `EntityManagerFactoryUtils.convertJpaAccessExceptionIfPossible` |

JPA 是 Sun 在 Java EE 5 之后提出的 ORM 统一标准，"就好像当年的 JDBC 标准一样，只不过 JPA 是面向 ORM 的统一"。Spring 2.0 之后提供支持，`LocalEntityManagerFactoryBean` 面向 Java SE 独立应用和测试（读 `META-INF/persistence.xml`），`LocalContainerEntityManagerFactoryBean` 面向需要定制的场景（可指定 `dataSource`、`persistenceXmlLocation`、`LoadTimeWeaver`）。

#### 本章结论

集成方式"几乎可以用千篇一律来形容"：模板方法类统一建模资源管理，回调接口承载具体访问逻辑，FactoryBean 把连接工厂接进 IoC 容器，异常统一转译进 `DataAccessException`，事务统一收进事务抽象层。Spring 2.5 之后核心包只保留 Hibernate 3、iBATIS 2、JDO、JPA 和 TopLink 的支持。作者强调：集成只是"让我们在使用 ORM 的过程中活得更加舒服一些"，选哪个 ORM 取决于场景，与集成无关；要发挥某个技术的最大功效，仍然得精通那个技术本身。

### 第16章 Spring 数据访问之扩展篇

#### 一句话主旨

把"模板方法 + Callback"从数据访问的具体技术里抽出来，当成一种通用的设计语言（FTP、HTTP 都能套），再顺手把多数据源这个绕不开的工程问题拆成两种拓扑。

#### 模板方法 + Callback 这个套路的本质

作者先讲动机，引的是 *Bitter Java* 的理念：**资源的获取与释放必须尽可能放在一起**。JDBC、Hibernate、iBATIS 在资源管理上的共性就是"用完必须安全释放"，而一旦把这件事散落到代码各处，就必然出泄漏。所以 Spring 用模板方法把"获取—使用—释放"这条不变流程写死在一处，把变化的部分（每次到底做什么）通过 Callback 交出去。

然后他用 FTP 做了个漂亮的对照实验。直接照 `FTPClient` 的 Javadoc 示例写，就是"连接、检查 reply code、登录、传文件、logout、disconnect"外加 `IOException` 处理——"与直接使用 JDBC API 一样让人尴尬"。第一版封装 `Phase1FtpUtility` 提供 `login()` / `doTransfer()` / `logout()`，只是少写几行，资源管理仍然留给每一处调用方，而且 `doTransfer()` 逻辑写死了，别人要用别的传输方式就得子类化。第二版 `FTPClientTemplate` 才是答案：

```java
public void execute(FTPClientCallback callback) throws IOException {
    FTPClient ftp = new FTPClient();
    ftp.connect(server, getPort());
    // 检查连接、登录、校验
    callback.processFTPRequest(ftp);   // 变化的部分
    ftp.logout();
    // 断开
}
```

`FTPClientCallback` 只有一个 `processFTPRequest(FTPClient ftp)`。这和 `JdbcTemplate.execute(StatementCallback)` 是同一个骨架：**模板方法负责不变的流程和资源收尾，Callback 拿到"句柄"去做变化的事，且不必关心收尾**。常用的具体操作（`listFileNames` 等）再作为便利方法加到模板类里，内部构造匿名 Callback 委托给 `execute`。`HttpClientTemplate` 同理——作者只起了个头，把剩下的留给读者。

顺带一个观察：Spring 自己的三个 JDBC 模板就是这个套路的层层叠加——`NamedParameterJdbcTemplate` 在 `JdbcTemplate` 上套一层解析网，`SimpleJdbcTemplate` 又在 `NamedParameterJdbcTemplate` 上套一层 Java 5 语法糖的网。

#### 多数据源怎么处理

生产环境很少只有一个库（容灾、负载均衡都会带来多台），作者从两个角度拆：

**一、"主权独立"。** 每个 `DataSource` 都对外独立承担公开数据库资源的职能，客户端明确知道该用哪个。典型场景是数据性质不同——FX 项目里 `mainDataSource` 存顾客与 FX Broker 的交易信息，`infoDataSource` 存汇率和系统履历，两者在逻辑上天然可分。配置上就是两个 `DataSource`、两个 `JdbcTemplate`，各自注入各自的 DAO（也可以用 `parent="dataAccessResourceSupport"` 让一个 DAO 同时持有两个）。作者的评价：这种管理"最简单，也是比较容易管理的"，在考虑更动态的方案之前，先评估现有方式是否已经够用。

**二、"合纵连横"。** 这就是社区常说的"多数据源动态切换"。此时各个数据源对数据访问类而言已经丧失"独立自主"地位，所有交互都要经过一个"盟主"——它本质也是一个 `DataSource`，但职责是协调联盟内的多个 `DataSource`，最终由它决定这次访问用哪一个。适用场景有三：多机热备（HA，一台挂了透明切换）、地位相当的多台同时 Active 做负载均衡、以及启动时无法确定、必须运行期按条件判定用哪个库。

实现方式是自定义 `DataSource`。Spring 2.0.1 之后有了 `AbstractRoutingDataSource`，继承它只需实现一个方法：

```java
protected abstract Object determineCurrentLookupKey();
```

`getConnection()` 时，它按这个 key 从内部那组 `DataSource` 里挑一个再委托。书里的原型 `PrototypeLoadBalanceDataSource` 用 `ReentrantLock` 加计数器轮询三个库；配置上用 `<util:map>` 定义 key→`DataSource` 的映射，再配 `MapDataSourceLookup` 替换掉默认的（按 key 走 JNDI 的）查找行为。作者提示：真实场景里 key 通常绑到当前线程，`determineCurrentLookupKey()` 直接从线程取，这样切换时机更灵活（可以在 DAO 里绑，也可以在拦截器里按事件绑）。

作者还提醒两个坑：其一，两种方式不是互斥的，可以组合——把"合纵连横"的整体塞进"主权独立"的场景，或反过来以 `SessionFactory` 一级替代 `DataSource` 一级；但"复杂度的过多引入有些时候并非必要"，提倡 K.I.S.S.。其二，把"合纵连横"类推到 Hibernate 时要小心：把多个数据源注入 `SessionFactory` 再动态切换，若开了二级缓存（与 `SessionFactory` 挂钩），缓存内容可能与当前数据源不匹配，切换时要么清空要么同步，否则并发下难免出问题；不能接受就退回到以 `SessionFactory` 为单位做路由（自己实现一个 `AbstractRoutingSessionFactory`）。

书末的 Spring 3.0 展望给了三个判断：全面升级到 Java 5 后，泛型化的 `JdbcTemplate` 仍是主力，`SimpleJdbcTemplate` 可能只在特定场景保留；基于操作对象的方式除 `StoredProcedure` 外不再推荐；ORM 集成将更多向 JPA 倾斜，并预计放弃对 TopLink 的支持。

#### 本章结论

这一章的价值不在 API，而在把前三章的共性显式化了：**模板方法 + Callback 是 Spring 数据访问层统一的"设计语言"**，凡是"取资源—用资源—还资源"的场景都可以照此办理。多数据源问题则被还原成一句工程判断——先看能不能"主权独立"，不能再用 `AbstractRoutingDataSource` 造一个"盟主"。


---
