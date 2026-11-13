# 二、全书最值得记住的 12 个洞察

这一节是我从全书里抽出来的"承重墙"。如果只记十二句话，记这些。

**1. IoC 的本质是"控制反转"，不是"依赖注入"。**
依赖注入（DI）只是实现 IoC 的三种手段之一（构造器注入、setter 注入、接口注入）。书里特意不从"解耦"角度讲，而从"对象绑定方式"角度讲——因为这样你能直接看到：老写法是 `new DowJonesNewsListener()` 写死在代码里，新写法是运行期由容器塞进来。**控制权转移了，这才叫反转。**

**2. BeanFactory 是微内核，ApplicationContext 是套壳。**
ApplicationContext 没有另起一套容器内核，它继承 BeanFactory，再用"实现接口 + 把活委派出去"的方式挂上四项能力：资源加载、国际化、事件发布、AOP 集成。**理解了这层继承关系，就不会把两者当成两个不同的容器。**

**3. Bean 的一生是一条固定流水线，所有扩展点都挂在上面。**
加载 BeanDefinition → 实例化 → 属性填充 → Aware 回调 → `BeanPostProcessor` 前置 → 初始化方法 → `BeanPostProcessor` 后置 → 使用 → 销毁。
你今天写的每一个"Spring 扩展"（自定义注解、拦截器、代理），本质上都是在这条线上找个钩子挂上去。

**4. AOP 不是取代 OOP，是给 OOP 打补丁。**
Aspect 之于 AOP，等于 Class 之于 OOP。日志、安全、事务这些"横切关注点"，用 OOP 写会散落在所有业务类里；用 Aspect 写就能收拢成一个模块。**但 AOP 至今没有自己的"主权空间"，只能寄生在 OOP 的类结构里——这就是它一切限制的根源。**

**5. Spring AOP 是"代理"，不是"改字节码"。**
它靠运行期生成代理对象来织入，所以只能拦**方法调用**：拦不住字段访问、拦不住构造器、更拦不住"同一个类里 `this.xxx()` 的自调用"。**所有"切面没生效"的怪事，根子都在这一句。**

**6. 声明式事务 = AOP + TransactionInterceptor。**
`@Transactional` 没有任何魔法，它只是给方法贴了个标记，然后由拦截器在方法前后开启/提交/回滚事务。看穿这层，那些经典困惑（为什么自调用失效？为什么 private 方法无效？为什么默认只回滚 RuntimeException？）**全都能自己推出来，不用背。**

**7. 事务的"无侵入"靠 ThreadLocal。**
DAO 层拿连接不需要传参数，是因为 `TransactionSynchronizationManager` 把连接绑在了当前线程上。这也是为什么事务**跨线程会失效**——新线程的 ThreadLocal 是空的。

**8. "模板方法 + Callback"是 Spring 的通用设计语言。**
JdbcTemplate、HibernateTemplate、TransactionTemplate、RestTemplate……全是同一个套路：**不变的流程写在父类，变化的逻辑交给回调**。看懂一个，剩下七个不用看。

**9. 统一异常体系比统一 API 更重要。**
JDBC 的 `SQLException` 是 checked 的，逼着每个 DAO 方法都 `throws`，接口就被具体技术污染了。Spring 用一套 unchecked 的 `DataAccessException` 体系 + 异常转译，把这个洞补上。**这是"面向接口编程"能不能真正落地的前提。**

**10. Spring MVC 的一切设计都是为了"可替换"。**
HandlerMapping、HandlerAdapter、ViewResolver 全是策略接口。所以它才能同时支持 XML 配置、注解 Controller、REST——**不是它预见了未来，而是它把每个可能变化的点都做成了插槽。**

**11. 容器事件是观察者模式的教科书实现。**
`ApplicationEvent` + `ApplicationListener`，加上 `ApplicationEventMulticaster` 做分发。理解了它，你就理解了 Spring 内部大量"解耦"的手法。

**12. 这本书真正值钱的不是知识点，是它的论证方式。**
每个概念都先问"没有它会怎样"，再用一个具体项目（FXNews 新闻系统、CREDIT 贷款系统）演示，最后才给 API。**学会这个顺序，你学任何框架都会快一倍。**

---
