# 第七部分　Spring 对 J2EE 服务的集成

---

# 《Spring揭秘》第七部分（第28–33章）精华笔记

> 覆盖 `s_ch28.txt`：第27章小结残页起，至第33章末。原书 2009 年，Spring 2.5 时代。

#### 涉及哪些 J2EE 服务（先厘清实际章节）

本部分共 6 章，覆盖面比"J2EE 全家桶"窄得多，实际清单如下：

| 章 | 主题 | 关键类 |
|---|---|---|
| 28 | JNDI | JndiTemplate、JndiObjectFactoryBean |
| 29 | JMS | JmsTemplate、MessageListenerContainer |
| 30 | E-mail（JavaMail） | MailSender、JavaMailSender、MimeMessageHelper |
| 31 | 任务调度与线程池 | Quartz/Timer 的 FactoryBean、TaskExecutor |
| 32 | 扩展篇 | 用 MailMonitor 重构串起 28–31 |
| 33 | 远程方案 Remoting | Service Exporter / Service Accessor |

**没有的内容**：没有 EJB 集成章，也没有 JMX 章——作者只在 33.5.1 把 JMX 当作"下一段旅程"留给读者；JTA 仅在 JMS 事务里以 JtaTransactionManager 一笔带过；JCA 未涉及。EJB 全程只作反面参照（MDB、Entity Bean、"Without EJB"）。

这六章其实只讲了一个方法论：**凡是"取资源、开连接、干活、清资源、翻译异常"这五步重复出现的 API，就抽一个 Template 加一个 Callback**。JndiTemplate、JmsTemplate、JavaMailSenderImpl、TimerFactoryBean 内部全是同一副骨架，所以读的时候不必逐章死记类名，抓住"模板 + 回调 + FactoryBean 包装 + 异常转译"这四件套即可。

#### JNDI 支持

JNDI（Java Naming and Directory Interface）分 API（给应用）与 SPI（给厂商）两层，价值在于给资源访问加一道"隔离层"：资源部署在哪、绑什么名字，应用不必知道。JMS 取 ConnectionFactory、EJB 取 Home 接口都靠它，所以放在本部分开头当铺垫。

原生写法是：构造 Hashtable 填 `INITIAL_CONTEXT_FACTORY` 与 `PROVIDER_URL`、`new InitialContext(env)`、`lookup("jdbc/xxx")`、finally 里 close 并吞掉 NamingException。真正有用的只有 `lookup` 那一行，其余全是可复用的噪音——这正是模板方法出场的信号。

- **JndiTemplate**（`org.springframework.jndi`）：又一次模板方法模式。核心是 `execute(JndiCallback)`——模板负责创建与关闭 Context，回调只写 `doInContext(ctx)`；常规操作直接用便捷方法 `lookup`/`bind`/`unbind`/`rebind`。environment 可在 XML 中配置 `java.naming.factory.initial`、`java.naming.provider.url`，再把 JndiTemplate 注入给使用者，避免把环境参数硬编码。
- **JndiObjectFactoryBean**：FactoryBean，按 `jndiName` 查找，并把**查到的资源本身**（而不是 FactoryBean）注入，使业务对象既不耦合 JNDI API 也不耦合 Spring API。2.0 之后可简写为 `<jee:jndi-lookup>`。三个开关：`lookupOnStartup`（默认 true）、`cache`（默认 true）、`defaultObject`（查找失败兜底）；前两者任一设为 false 时必须给出 `proxyInterfaces`，因为内部改用 JndiObjectTargetSource 生成接口代理、按需查找。

一句话概括设计意图：**JndiTemplate 解决"写起来烦"，JndiObjectFactoryBean 解决"耦合太紧"**——后者才是依赖注入场景下的正解。

#### 其他服务各自的集成方式

**JMS（第29章）**：原生 JMS 要七步——JNDI 取受管理对象、建 Connection、建 Session、建 MessageProducer、建 Message、send、逐层 close；接收侧只是把 Producer 换成 Consumer 并调 `Connection.start()`。更要命的是 JMSException 体系是 checked 的，网络断了应用其实无能为力，却被逼着层层 catch。JmsTemplate 正是针对这两点：封住七步套路，只依赖一个 ConnectionFactory；另立一套 unchecked 异常。发送三档：`execute(ProducerCallback)`、`send(MessageCreator)`、`convertAndSend(MessageConverter + MessagePostProcessor)`；接收侧有 `receive` / `receiveSelected` / `receiveAndConvert` / `receiveSelectedAndConvert`，另有 `browse` 只读查看队列；`defaultDestination`、`receiveTimeout`、DestinationResolver（Dynamic / Jndi / BeanFactory 三种实现）负责细节。

同步"请求—应答"场景书中给了三套方案，取舍很有参考价值：为每个客户端配一对队列（性能好但队列管理爆炸）；单队列加 Message Selector 过滤（省队列但有过滤开销）；用 Temporary Queue 作 `JMSReplyTo`（既不建大量队列也不用 selector，但必须让收发处于同一 Connection，所以要用 `execute(SessionCallback)`）。异步侧用 MessageListenerContainer 取代 MDB：Simple（标准 setMessageListener，受 Provider 线程约束）、Default（自建线程循环调同步 receive，最常用）、ServerSession（需 Provider 支持 ServerSessionPool）。POJO 只实现 MessageListener 即可，也可用 SessionAwareMessageListener 拿到 Session，或用 MessageListenerAdapter + delegate + defaultListenerMethod 完全免接口。异常转译由 `JmsUtils.convertJmsAccessException` 完成，用模板或容器时已自动完成。事务三档：`sessionTransacted` 走本地，JmsTransactionManager 管消息独立事务，只有必须与数据库同事务时才上 JtaTransactionManager。

**E-mail（第30章）**：两层抽象——MailMessage（SimpleMailMessage 纯文本 / MimeMailMessage 富文本）与 MailSender（只能发 SimpleMailMessage），再由 JavaMailSender 补上 MIME 能力，实现类 JavaMailSenderImpl。好处是准备邮件内容时用语义明确的 setter，不必记 `RecipientType.CC` 这类 JavaMail 特有枚举。实战写法是 JavaMailSender + MimeMessagePreparator 回调 + MimeMessageHelper（`setText(html, true)`、`addInline`、`addAttachment`），不必自己拼 Multipart/BodyPart。COS 支持在 2.5 已移除；模板邮件则再抽象一层 `ITemplateMailAgent`，用 `VelocityEngineUtils.mergeTemplate` 合并模板，模板可放 classpath 或数据库（DataSourceResourceLoader 只认 JNDI DataSource，需扩展 VelocityEngineFactoryBean 覆写 `postProcessVelocityEngine`）。

**任务调度与线程池（第31章）**：先补 Quartz 的四个角色——Job（被调度的任务）、JobDetail（执行上下文，二者通过 JobDataMap 通信）、Trigger（触发规则，SimpleTrigger 或 cron 表达式驱动的 CronTrigger）、Scheduler（总管）；一个 Trigger 只对一个 Job，但多个 Trigger 可以叠加到同一 Job，规则取并集，若要"大范围覆盖再挖掉几段"则配合 Calendar 排除。Spring 侧用 FactoryBean 全套接管——JobDetailBean、MethodInvokingJobDetailFactoryBean（直接指定 targetObject/targetMethod，免写 Job，代价是产出的 JobDetail 不可序列化）、SimpleTriggerBean/CronTriggerBean、SchedulerFactoryBean（随容器启停并注册 triggers）。QuartzJobBean 侵入性太强，Quartz 1.5+ 之后由 SpringBeanJobFactory 取代。JDK Timer 侧同理：MethodInvokingTimerTaskFactoryBean 免去继承 TimerTask，ScheduledTimerTask 把任务与 Trigger 信息打包，TimerFactoryBean 统一调度。TaskExecutor 是为兼容 Java 1.4 而生的 Executor 孪生接口，实现有 Sync、SimpleAsync、ThreadPoolTaskExecutor（首选）、ConcurrentTaskExecutor（桥接任意 Executor）以及 Timer/SimpleThreadPool/WorkManager 等。这一章真正的结论是：**调度方式与业务逻辑无关，改的只是配置**，所以业务对象要独立存在，别为了能被调度就往里塞 Job/TimerTask。

**扩展篇（第32章）**：拿作者 2004 年的 MailMonitor 批处理开刀——它继承了 QuartzJobBean、把 JavaMail 取信代码与业务处理搅在一起、收件后逐封循环处理。重构三步：脱离 QuartzJobBean 改用 SpringBeanJobFactory；把 JavaMail 收信的资源管理抽成可复用的 Helper（Spring 只给了发信设施，没给收信设施）；把逐封处理拆成 Runnable 交给 TaskExecutor。这一章是全部分的复盘练习题，价值在"如何识别侵入式代码"，而不在任何新 API。

**Remoting（第33章）**：先立态度——分布式能带来负载均衡、容灾、统一部署，但代价是网络延迟、并发与故障处理，所以书中引 Martin Fowler 的话："不要分布你的对象"；日常八成应用不需要分布式，Spring 提倡的是先写好本地业务对象，需要时再加远程外壳。技术上 Spring 走 RPC 路线（客户端阻塞等结果），核心是"Service Exporter + Service Accessor"组合——Exporter 收请求、解组、反射调用本地对象、再把结果编组回传；Accessor 是 ProxyFactoryBean 形态的客户端代理，让调用方以为在本地调用。异常统一为 unchecked 的 RemoteAccessException 家族（Lookup / Connect / Proxy / Invocation 四种 Failure），把各家远程机制的专有异常转译掉。具体方案：RMI（RmiServiceExporter / RmiProxyFactoryBean；接口不合 RMI 规范时自动走 RMI Invoker，把 Remote 与 RemoteException 转嫁给 RmiInvocationHandler，业务接口保持干净）；HTTP Invoker（Exporter 是 HttpRequestHandler，挂在 DispatcherServlet 上，走 Java 序列化，对防火墙友好）；Hessian（二进制）/ Burlap（XML）；Web 服务（Spring 只给 ServletEndpointSupport，公开端靠 CXF 的 `<jaxws:endpoint>`，访问端可用 JaxWsProxyFactoryBean 或 Spring 的 JaxWs/JaxRpcPortProxyFactoryBean）；JMS Remoting（JmsInvokerServiceExporter 本身是 SessionAwareMessageListener，配 MessageListenerContainer；客户端 JmsInvokerProxyFactoryBean，消息体是装 RemoteInvocation 的 ObjectMessage，返回值走临时队列；功能不足时可换 Codehaus 的 Lingo）。扩展点是"一个中心、两个基本点"：RemoteInvocation 携带 methodName / parameterTypes / arguments / attributes，Factory 负责造它、Executor 负责执行它，且只对 RMI、HTTP Invoker、JMS 三种方案有效。注意 Spring Remoting 提供的都是**无状态**远程服务，这是它能保持轻量的前提。

#### ⚠️ 过时提示

**可以直接跳过**：JMS 1.02 相关（JmsTemplate102 等）；COS 邮件（2.5 已删）；QuartzJobBean；JDK Timer 全套（调度漂移、单个任务异常会拖垮后续调度，Java 5 起就该用 ScheduledExecutorService）；RMI / Hessian / Burlap / HTTP Invoker / JMS Invoker 的具体配置（Spring 6.0 已移除 Hessian、Burlap、HTTP Invoker 与 JMS Invoker，RMI 亦废弃）；Velocity 邮件模板（Spring 4.3 废弃、5.0 移除）。

**仍有价值**：JndiTemplate 与 JndiObjectFactoryBean 的思路（把查找这件脏活外包给 FactoryBean）在配置外部化、多环境部署中依然成立；JmsTemplate 与 DefaultMessageListenerContainer 至今是 Spring JMS 的核心，只是改用 Boot 自动配置加 `@JmsListener`；MimeMessageHelper 仍是发 HTML、带附件邮件的标准写法；TaskExecutor / ThreadPoolTaskExecutor 是今天 `@Async` 的底座；Exporter/Accessor 与 RemoteInvocation 的抽象，是理解 Feign、gRPC、HTTP 客户端代理的思想原型。33.5.2 预告的 RestTemplate 确实来了，但它今天自己也进入维护模式，被 WebClient / RestClient 取代——这段"展望"本身就是很好的技术史样本。

**只读一小时的取舍**：先看第29章的 JmsTemplate 与 DefaultMessageListenerContainer（今天仍在用），再看第30章的 MimeMessageHelper，最后读第33章的"Exporter/Accessor + 异常转译 + 无状态"三段架构说明；JNDI、Timer、RMI/Hessian 那些具体配置可以整段跳过。

### 附录

**本切片没有附录。** 扫描件从第27章小结残页（p0591）开始，到 p0673（书页 664）第33章 33.5.2 结束，全文以"拥抱变化，拥抱新的Spring吧"收尾；`ocr.txt` 全文也检不到"附录"字样，因此不存在可整理的附录条目。书末真正承担收尾作用的是两节：

- **33.5.1 拉开 JMX 演出的序幕**：以"Rollover 必须 6:10 准点关市，而容器启动太慢"为引子，提出让程序常驻、用远程管理替代反复启动，把 JMX（JSR 160）留给读者自学。要点：精简 bean 定义、lazy-init 只是权宜，真正的解法是改变部署形态。
- **33.5.2 Spring 3.0 展望**：REST 支持与 RestTemplate、更轻量的 Web 服务、OXM（Object/XML Mapping）抽象层。

书中散落的延伸阅读，整理成表：

| 主题 | 书中给出的去处 |
|---|---|
| JNDI | Sun《The JNDI Tutorial》 |
| JMS | JMS Specification（约 140 页，建议通读） |
| Quartz | OpenSymphony 官网；《Quartz Job Scheduling Framework》 |
| JMS Remoting 增强 | Codehaus Lingo |
| Web 服务 | Spring Web Services、Apache CXF、xFire、Axis2 |
| 批处理 | Spring Batch |
| RestTemplate | Spring 团队博客 2009-03-27 一文 |

---
