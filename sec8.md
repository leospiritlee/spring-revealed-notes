# 第三部分　Spring AOP 框架

---

### 第7章 一起来看 AOP

#### 一句话主旨

AOP 不是要取代 OOP，而是给 OOP 打补丁：它用 Aspect 把日志、安全、事务这类"横切关注点"模块化，再靠"织入"把它们缝回 OOP 的类里——Aspect 之于 AOP，就等于 Class 之于 OOP。

#### 为什么需要 AOP（作者的论证链）

作者的论证从"银弹"的破灭开始。软件开发一路从汇编到过程化编程（Procedural Programming），再到面向对象编程（OOP），每次都想找到一颗"搞定一切"的银弹。OOP/OOSD 被提出来时，那个时代的人大概都以为找到了。但作者说：即使 OOP，也不能很好地解决软件开发中的所有问题。

关键区分在**两类需求**上。作者用自己参与过的 CREDIT 项目举例——一个贷款业务管理系统，业务功能包括顾客贷款申请、顾客信息管理、贷款信息管理、贷款发放回收。这些属于"业务需求"，用 OOP 划分模块非常干净，需求与实现基本是**一对一**的，在系统里某个确定的点就能找到对应实现，开发和维护都方便。

但事情没完。为了调试、为了进生产环境后监控，每个业务对象都要加日志记录；业务方法执行前要有权限限制，所以要加安全检查。这些属于"系统需求"。需求本身很明确（加日志、加安全检查），可一旦要用面向对象的方式实现并集成进系统，就**不再是一个需求对一个实现**了：每个业务对象都要加，实现代码就遍及所有业务对象。图 7-2 里这个对应关系还只是 1:2，但随着业务对象增加，会变成 1:3、1:4……1:100 甚至更多。

于是有了那个著名的比喻：日志记录、安全检查、事务管理这些系统需求，就像**一把把刀"恶狠狠"地横切**到我们组织良好的各个业务功能模块之上——这就是**横切关注点（cross-cutting concern）**。OOP 能分解业务关注点，却无法避免系统需求的实现在系统中各处散落。

AOP（Aspect-Oriented Programming，面向方面编程）给出的答案是：引入 **Aspect** 概念，以模块化形式封装横切关注点。**Aspect 之于 AOP，就相当于 Class 之于 OOP**。AOP 只是对 OOP 的补足——把以 Class 形式模块化的业务需求，和以 Aspect 形式模块化的系统需求拼装到一起，整个系统就算完成了。

#### 7.1 的"尴尬"：积木与俄罗斯方块

作者用了两个类比说明 AOP 的处境。

**积木**：把系统看作积木，业务需求积木块和系统需求积木块**形状和材质可以相近甚至相同，但摆放的空间位置完全处于不同的维度**。Class 积木在一个空间面上摆，Aspect 积木在另一个空间面上摆，就能搭出一座立体的美丽城堡。

**俄罗斯方块**：可惜当前技术所限，我们能造出 AOP 的积木块，却**造不出属于 AOP 的独有空间**。现在用 OOP 开发就像玩俄罗斯方块——砖头形状就那么几种，剩下全靠你在游戏规定的空间维度里想办法把每一层填满，否则系统就 Game Over。最简单的方法明明是从我们所处的维度直接拿合适的砖头把遗漏的空间填上，但规则不允许。

结论一句话：**AOP 现在没有主权，所有 AOP 实现都得"寄生"在 OOP 的主权领土中**，系统维度也依然保持着 OOP 的"维度世界纪录"。

#### 关键概念

- **Joinpoint（连接点）**：将要在其之上进行织入操作的**系统执行点**。作者强调"基本上只要允许，程序执行过程中的任何时点都可以作为 Joinpoint"。常见类型有：方法调用（Method Call，调用对象上的执行点）、方法执行（Method Execution，被调用方法逻辑执行的时点，**对同一对象，方法调用先于方法执行**）、构造方法调用/执行、字段设置（Field Set，setter 或直接赋值本质相同）、字段获取（Field Get）、异常处理执行（Exception Handler Execution）、类初始化（Class Initialization，静态变量/静态块初始化）。循环开始的时点理论上也是 Joinpoint，但难捕捉，现有 AOP 产品大都不支持。

- **Pointcut（切点）**：**Joinpoint 的表述方式**。织入时要参照 Pointcut 才知道往哪些 Joinpoint 织。三种表述形式：直接指定方法名（简单但功能单一）、正则表达式（最普遍，JBoss AOP、Spring AOP、AspectWerkz 都支持）、特定 Pointcut 表述语言（最强但实现复杂，AspectJ 用这种；Spring 2.0 之后借用 AspectJ 的解释器也支持）。Pointcut 之间还能做逻辑运算——作者用"全校男生 / 全校女生 / 二年级一班男生"做了张表，`Males || Females` 就是全校学生；Spring 配置里用 `and`/`or`，AspectJ 里用 `&&`/`||`（毕竟 AspectJ 扩展自 Java）。

- **Advice（通知）**：**单一横切关注点逻辑的载体**，代表将被织入到 Joinpoint 的横切逻辑。**Aspect 比作 Class，Advice 就相当于 Class 中的 Method**。按执行时机与功能分为：Before、After Returning、After Throwing（Throws）、After (Finally)、Around、Introduction。Around 的类比很妙：像"叵"字，中间那一"口"是 Joinpoint，上下一横是要执行的逻辑——只在前前后后加逻辑而忽略原逻辑，就是居心"叵"测；正常应像个"巨"字，先走上横、再走 Joinpoint、再走下横。Introduction 在 AspectJ 叫 Inter-Type Declaration，在 JBoss AOP 叫 Mix-in，作者用"给你穿军装戴军帽，你就有了军人的特性"作比。

- **Aspect（方面）**：对横切关注点逻辑进行**模块化封装**的 AOP 概念实体，通常包含多个 Pointcut 及相关 Advice 定义。

- **织入（Weaving）/ 织入器（Weaver）**：作者引毛主席诗"一桥飞架南北，天堑变通途"——织入就是飞架 AOP 与 OOP 的那座桥，完成织入的那个"人"就是织入器。AspectJ 的织入器是 `ajc` 编译器；JBoss AOP 是自定义 ClassLoader；**Spring AOP 用一组类完成织入，`ProxyFactory` 是最通用的织入器**。形式各异，唯一相同的是职责。

- **目标对象（Target Object）**：符合 Pointcut 指定条件、将在织入过程中被织入横切逻辑的对象。

#### Java 平台上的五种 AOP 实现机制

| 机制 | 代表实现 | 优点 | 缺点 | 是否 Spring 采用 |
|---|---|---|---|---|
| 动态代理（Dynamic Proxy） | JDK 1.3+ 的 `java.lang.reflect.Proxy` + `InvocationHandler`；Nanning | 运行期按接口动态生成代理对象，横切逻辑集中在 `InvocationHandler` 里 | 只能对**实现了接口**的类代理；反射调用，性能比静态类稍逊 | **默认采用**（Spring AOP 默认机制） |
| 动态字节码增强 | ASM、**CGLIB** | 运行期生成子类，**不要求目标类实现接口** | 无法扩展 `final` 类/`final` 方法 | **采用**（目标对象无接口时用 CGLIB 兜底） |
| Java 代码生成 | 早期 EJB 容器（声明性事务、CMP 实体 Bean 只需声明接口） | 容器按部署描述符生成代码再编译，业务类无需写实现 | 古老，依赖部署工具/部署接口，编译期额外步骤 | 否（"现在已经退休了"） |
| 自定义类加载器 | JBoss AOP、AspectWerkz（已并入 AspectJ 项目） | 在 class 加载期改字节码，基本可对大部分类和实例织入，功能强大 | 某些应用服务器会控制整个类加载体系，此场景下可能出问题 | 否 |
| AOL 扩展 | **AspectJ**（`ajc`），另有 AspectC/C++、Aspect.Net、AspectL、AspectPHP | AOP 概念在语言中一一对应实体，表达力最强，强类型检查，可编译进静态类提升运行性能 | **要重新学一门语言扩展或新语言**，最难掌握 | 否（但 Spring 提供 AspectJ 集成） |

前两种即"第二代/动态 AOP"的典型手段，末一种（AOL 扩展 + 预编译）是"第一代/静态 AOP"。

#### 本章结论

AOP 是对 OOP 的补足而非替代；AOL 是"寄生语言"，Aspect 组件必须经"织入"才能进入 OOP 世界。要理解 Spring AOP，先得认识 AOP 国度的这几位公民——Joinpoint、Pointcut、Advice、Aspect、织入器、目标对象。

### 第8章 Spring AOP 概述及其实现机制

#### 一句话主旨

Spring AOP 是 Spring"质量三角"的一角，以 Java 为 AOL，**不追求大而全，用 20% 的 AOP 支持覆盖 80% 的需求**，底层靠"运行期为目标对象生成代理对象"来实现织入——先动态代理，不行再 CGLIB。

#### Spring AOP 的定位（和 AspectJ 的分工）

Spring AOP 与 IoC 容器、Spring 对其它 JavaEE 服务的集成，共同构成 Spring 框架的**"质量三角"**。

设计哲学是"简单而强大"：**不打算把全部 AOP 需求囊括在内，而是以有限的 20% 的 AOP 支持满足 80% 的 AOP 需求**。超出那 80% 的需求，就去求助于 AspectJ——Spring AOP 对 AspectJ 提供了很好的集成。因为 AOL 就是 Java，学习曲线比 AspectJ 那种语言扩展型实现平缓得多，能更快捷地融入开发过程。

与静态 AOP 的根本差别：AspectJ 用编译器把横切逻辑**编译织入**目标类；Spring AOP 在**运行期间**为目标对象生成一个代理对象，横切逻辑织进代理对象，**系统最终使用的是代理对象，而不是真正的目标对象**。

#### 两种实现机制

**代理模式（Proxy Pattern）的四角色**：`ISubject`（被访问资源的抽象接口）、`SubjectImpl`（具体实现，"你要访问某位明星，它就是那位明星；你要买房，它就是房主"）、`SubjectProxy`（持有 `ISubject` 实例的代理）、`Client`（访问者，只能通过代理访问）。代理是"被代理者的**影子**，只不过这个影子通常拥有更多功能"。

**静态代理的窘境**：为 `SubjectImpl` 写 `ServiceControlSubjectProxy`（零点到六点拒绝 `request()`）可行；但 `IRequestable`/`RequestableImpl` 也有 `request()`，横切逻辑完全一样，却**必须再写一个 `ServiceControlRequestableProxy`**。Joinpoint 相同，目标对象类型不同，就得单独实现一个代理。系统里有成百上千个符合 Pointcut 的目标对象时，就要写上千个代理类——**"这么玩会死人的！"**

**机制一：动态代理（JDK Dynamic Proxy）**

只需一个 `InvocationHandler` 实现（`RequestCtrlInvocationHandler`），再用 `Proxy.newProxyInstance(loader, new Class[]{ISubject.class}, handler)` 就能为任意接口生成代理。`InvocationHandler` 就是横切逻辑的载体，**作用跟 Advice 一样**。Spring AOP 默认采用此机制。

限制：**只能对实现了相应 Interface 的类使用**；若某类没有实现任何接口，就无法用它生成代理对象。原因在于动态代理的产物是一个"实现了指定接口"的新类（后文实测其类型为 `$Proxy0`），它与目标实现类之间没有继承关系——所以代理对象能转成接口类型，却**永远转不成目标实现类**（9.5.1 里作者专门让人试了这一手，结果是 `ClassCastException`）。

**机制二：动态字节码生成（CGLIB）**

原理是**继承扩展**：为目标对象生成子类，子类覆写父类方法把横切逻辑塞进去，让系统使用子类——`SubClass instanceof superClass == true`。手工为每个类写子类同样会"死人"，所以要靠 CGLIB 在运行期动态生成。

用法：实现 `net.sf.cglib.proxy.MethodInterceptor`（它扩展了 `Callback`），在 `intercept(Object, Method, Object[], MethodProxy)` 里写逻辑，末尾用 `proxy.invokeSuper(object, args)` 调用原方法；然后 `Enhancer enhancer = new Enhancer(); enhancer.setSuperclass(Requestable.class); enhancer.setCallback(new RequestCtrlCallback());` 再 `enhancer.create()`。产物的类名形如 `Requestable$$EnhancerByCGLIB$$...`。

限制：**无法覆写 `final` 方法**（也就无法扩展 `final` 类）。

**分工**：默认情况下，Spring AOP 发现目标对象实现了接口就用动态代理；没有实现任何接口，就退到 CGLIB。

### 第9章 Spring AOP 一世

#### 一句话主旨

Spring AOP 把每个 AOP 概念都落成了普通 Java 类：Joinpoint 只支持方法执行一级，Pointcut 拆成 `ClassFilter` + `MethodMatcher` 两个可组合的匹配器，Advice 遵循 AOP Alliance 规范并分 per-class / per-instance 两类，Advisor 充当 Aspect，织入则由 `ProxyFactory` / `ProxyFactoryBean` / AutoProxyCreator 三档工具完成。

**Joinpoint 的收缩**：Spring AOP **只支持方法级别的 Joinpoint，更确切地说只支持方法执行（Method Execution）类型**。理由三条：K.I.S.S.，20% 换 80%；字段级拦截会**破坏面向对象的封装**，而拦 setter/getter 完全可达同样目的；真超出 80% 就找 AspectJ。

#### Pointcut 家族

顶层接口 `org.springframework.aop.Pointcut` 只有两个方法：`getClassFilter()` 与 `getMethodMatcher()`，并提供 `Pointcut TRUE = TruePointcut.INSTANCE`（匹配所有对象的所有受支持 Joinpoint）。拆成两级是为了**重用不同级别的匹配定义、支持组合运算、允许子类只覆写其中一部分**。

`ClassFilter.matches(Class)` 做类型级匹配，`ClassFilter TRUE = TrueClassFilter.INSTANCE`。

`MethodMatcher` 是重头戏，三个方法：`matches(Method, Class)`、`isRuntime()`、`matches(Method, Class, Object[])`。分界线就是 `isRuntime()`：

- 返回 `false` → **StaticMethodMatcher**，不考虑方法参数，匹配结果可缓存，性能好；
- 返回 `true` → **DynamicMethodMatcher**，每次调用都要检查参数，无法缓存，效率差，**应尽量避免**。

规则细节：两参数 `matches` 返回 `false` 时，这就是最终结果，三参数的 `matches` 铁定不会执行。作者用 `login(String username, String password)` 举例：只想在登录前插计数，参数可忽略（static）；想按用户做特殊处理，参数必须参与匹配（dynamic）。

由此 Pointcut 分两类：`StaticMethodMatcherPointcut` 与 `DynamicMethodMatcherPointcut`，前者有性能优势，Spring 给的支持更多。

常见实现：

| Pointcut | 匹配什么 |
|---|---|
| `NameMatchMethodPointcut` | 按**方法名**匹配，`setMappedName` / `setMappedNames`，支持 `*` 通配（`"match*"`、`"*matches"`、`"mat*es"`）。**无法匹配重载方法**，因为它只看名字、不看参数 |
| `JdkRegexpMethodPointcut` | 基于 JDK 1.4+ 标准正则（`java.util.regex`），继承自 `AbstractRegexpMethodPointcut`，用 `pattern` / `patterns` 属性 |
| `Perl5RegexpMethodPointcut` | 同上，但用 Jakarta ORO 的 Perl5 风格正则，需把 ORO jar 放进 classpath |
| `AnnotationMatchingPointcut` | JDK 5+，按注解匹配。三个构造：只给类级注解 → 该类所有方法全匹配；只给方法级 → 所有标了该注解的方法都匹配（**不分哪个类**）；两者都给 → 必须同时满足，包围圈最小。静态工厂 `forClassAnnotation` / `forMethodAnnotation` |
| `ComposablePointcut` | Pointcut 的**逻辑运算**：`union`（并）、`intersection`（交），还能与 `ClassFilter` / `MethodMatcher` 直接组合。只做 Pointcut 间运算可用工具类 `Pointcuts.union/intersection` |
| `ControlFlowPointcut` | 最特殊的：匹配**程序的调用流程**，而非单一 Joinpoint 特征。`new ControlFlowPointcut(TargetCaller.class)` 表示只有 `TargetCaller` 里调用的 `method1` 才被拦截，别处调用不拦；可再传第二个参数指定方法名 `"callMethod"`。运行期要查调用栈、每次调用都查，**性能差** |

**扩展 Pointcut**：`StaticMethodMatcherPointcut` 已把 `ClassFilter` 默认设为 `TRUE`、`isRuntime()` 固定返回 `false`、三参数 `matches` 抛 `UnsupportedOperationException`，**只需实现两参数 `matches`**（例：`QueryMethodPointcut` 匹配 dao 包下 `get` 开头的方法）。`DynamicMethodMatcherPointcut` 则 `getClassFilter()` 返回 `TRUE`、`isRuntime()` 恒 `true`、两参数 `matches` 默认返回 `true`，**只需实现三参数 `matches`**（例：`PKeySpecificQueryMethodPointcut` 额外比对 `args[0]` 是否 `"12345"`）。作者提醒：动手前先查现成的。

**IoC 容器中的 Pointcut**：它们都是普通 Java 对象，可以 `<bean id="nameMatchPointcut" class="...NameMatchMethodPointcut">` 注册、可注入依赖，完全合情合理。

#### Advice 家族

Spring AOP 加入了 **AOP Alliance**（作者注：更确切地说是 Rod Johnson 本人），Advice 实现全部遵循其接口：`org.aopalliance.aop.Advice` 是根，`BeforeAdvice`、`AfterAdvice`、`ThrowsAdvice` 都是**标志接口**（无方法），环绕走 `org.aopalliance.intercept.MethodInterceptor`。

**per-class 类型**：实例可在目标类的所有实例间共享，只做方法拦截，**不为目标对象保存状态、不添加新特性**。除 Introduction 外全部属于此类。

- **Before Advice** → 实现 `org.springframework.aop.MethodBeforeAdvice`：`before(Method method, Object[] args, Object target)`。执行完流程从 Joinpoint 继续，通常不打断，但可抛异常中断。`BeforeAdvice` 空接口是为将来属性级拦截留扩展余地。示例 `ResourceSetupBeforeAdvice`（路径不存在就 `FileUtils.forceMkdir`）。
- **Throws Advice** → 实现 `org.springframework.aop.ThrowsAdvice`（对应 After Throwing）。接口无方法，靠**命名与签名约定**：`void afterThrowing([Method, args, target], ThrowableSubclass)`，方括号里三个参数可省；同一类里可写多个 `afterThrowing` 重载，框架用反射挑选。示例 `ExceptionBarrierThrowsAdvice` 捕获 `RuntimeException` 后发邮件通知运维，属 **Fault Barrier** 模式。
- **After Returning Advice** → 实现 `org.springframework.aop.AfterReturningAdvice`：`afterReturning(Object returnValue, Method, Object[] args, Object target)`。能读到返回值、方法、参数、目标对象，但**不能修改返回值**（与通常 AfterReturning 的特性有出入）；只在正常返回时执行，所以**不适合做资源清理**。示例 `TaskExecutionAfterReturningAdvice` 往状态表插批处理运行记录。
- **Around Advice** → Spring **没有自己的接口**，直接用 `org.aopalliance.intercept.MethodInterceptor`：`Object invoke(MethodInvocation invocation)`。`invocation.proceed()` 决定调用链是否继续，**不调用就"短路"**，同 Joinpoint 上后续拦截器与方法逻辑都不会执行。示例 `PerformanceMethodInterceptor`（Stopwatch 计时）、`DiscountMethodInterceptor`（促销期把返回值乘 80/100——这正是 AfterReturningAdvice 做不到的）。
- **注意**：Spring AOP **没有提供 After (Finally) Advice**，资源清理类逻辑得靠 Around 承载。

**per-instance 类型**：不在实例间共享，**为每个目标实例保存各自的状态**。作者类比：打卡机是 per-class——全公司共用一个；每人桌上那台电脑是 per-instance——里面存着各自的资料。**Spring 中 Introduction 是唯一的 per-instance 型 Advice**。

Introduction 的机制：`IntroductionInterceptor extends MethodInterceptor, DynamicIntroductionAdvice`；`DynamicIntroductionAdvice.implementsInterface(Class intf)` 用来界定"我能为哪些接口提供拦截"。两条实现分支：**动态分支**（`DynamicIntroductionAdvice`，运行期才判定目标接口）与**静态可配置分支**（`IntroductionInfo.getInterfaces()`，自身就带目标接口信息，织入时不必再指定）。新增接口上的方法调用通常**不必调 `proceed()`**，因为当前位置已是"航程"的终点。

两个现成实现，也是本章最容易混的点：

- `DelegatingIntroductionInterceptor`：自己不实现新逻辑，**委派**给一个 delegate 实例。但它是"**伪军**"——只持有同一个 delegate 供目标类所有实例共享，"它往哪里去放对应各个目标对象实例的状态啊？"，**根本没兑现 per-instance 的承诺**。
- `DelegatePerTargetObjectIntroductionInterceptor`：内部持有**目标对象 → Introduction 实现实例**的映射，以目标对象实例为键取"自己家锅里的饭"，取不到就新建并登记。构造时传实现类与接口类型：`new DelegatePerTargetObjectIntroductionInterceptor(DelegateImpl.class, IDelegate.class)`。

作者还吐槽这两个类的可扩展性：实例变量没公开，该 `protected` 的方法写成了 `private`，`DelegatePerTargetObjectIntroductionInterceptor` 干脆找不到有用的扩展点。性能上，Spring 用动态代理织入 Introduction，比 AspectJ 静态织入逊色不少，必要时可考虑 AspectJ。

#### Aspect = Pointcut + Advice

**`Advisor` 就是 Spring 里的 Aspect**。但它是"特殊的 Aspect"：正常 Aspect 可以有多个 Pointcut 和多个 Advice，**Advisor 通常只持有一个 Pointcut 和一个 Advice**。

两支：`PointcutAdvisor`（`getPointcut()` + 继承来的 `getAdvice()`）与 `IntroductionAdvisor`。

`PointcutAdvisor` 家族：

- `DefaultPointcutAdvisor`——"大弟子"，**最通用**。除 **Introduction 型 Advice 之外**，任何 Pointcut、任何 Advice 都能用；可构造时传 `(pointcut, advice)`、`(advice)`，也可 `setPointcut` / `setAdvice`。
- `NameMatchMethodPointcutAdvisor`——细化版，Pointcut 类型**被限定为 `NameMatchMethodPointcut` 且外部不可改**，`setMappedName` / `setMappedNames` 实际是在操作内部那个实例；Advice 除 Introduction 外任意。
- `RegexpMethodPointcutAdvisor`——内部持一个 `AbstractRegexpMethodPointcut`，**默认用 `JdkRegexpMethodPointcut`**，要 Perl5 就 `setPerl5(true)`。
- `DefaultBeanFactoryPointcutAdvisor`——绑死 BeanFactory，用 `adviceBeanName` 按 bean 名关联 Advice，**只有 Pointcut 匹配成功才去实例化 Advice**，降低容器启动初期 Advisor 与 Advice 的耦合。

`IntroductionAdvisor` 分支"纯粹就是为 Introduction 而生的"：**只能做类级别拦截，只能用 Introduction 型 Advice**，不能像 PointcutAdvisor 那样配任意 Pointcut。唯一默认实现是 `DefaultIntroductionAdvisor`。Introduction 的 Advice 与 Advisor **不能和其他 Advice/Advisor 混用**。

**Ordered**：多个 Advisor 的 Pointcut 命中同一 Joinpoint 时，会在此处串成一条拦截链。顺序号决定优先级，**顺序号越小优先级越高，越靠外层**；建议从 0 或 1 起（小于 0 原则上留给框架）；不显式指定就按**声明顺序**，先声明的号最小、优先级最大。作者讲了同事大鹏的真实故障：`PermissionAuthAdvisor` 抛出的异常 `ExceptionBarrierAdvisor` 捕获不到——因为声明顺序反了，`ExceptionBarrier` 落到了内层，调用流程"已经经过"它才出错。彻底解法不是调配置顺序，而是显式给每个 Advisor 设 `order`（各 Advisor 实现类已实现 `org.springframework.core.Ordered`，配置里写 `<property name="order" value="0"/>` 即可）。

#### 织入的三种方式

Spring AOP 的织入器是 `org.springframework.aop.framework.ProxyFactory`（最基本的一个）。

**一、ProxyFactory 编码式**

```java
ProxyFactory weaver = new ProxyFactory(yourTargetObject);
weaver.addAdvisor(advisor);
Object proxyObject = weaver.getProxy();
```

只需两样"原材料"：目标对象、Aspect（即 Advisor）。也可以 `addAdvice(...)` 直接给 Advice，此时 ProxyFactory 内部会替你造 Advisor：非 Introduction 型 → Pointcut 用 `Pointcut.TRUE`（这些"没穿衣服"的 Advice 会作用到所有可识别 Joinpoint）；`IntroductionInfo` 子类 → 造 `DefaultIntroductionAdvisor`；`DynamicIntroductionAdvice` 子类 → 抛 `AopConfigException`（拿不到目标接口信息）。

- **基于接口**：`setInterfaces(new Class[]{ITask.class})`；或干脆不设，只要目标类至少实现一个接口，且没把 `optimize` / `proxyTargetClass` 设为 `true`，就走 JDK 代理。
- **基于类**：满足三者之一即走 CGLIB——目标类没实现任何接口（不论 `proxyTargetClass` 为何）、`proxyTargetClass=true`、`optimize=true`。
- **Introduction 织入**：新接口**必须**通过 `setInterfaces` 显式指定，且只需指定接口不需 Pointcut；原目标对象走接口代理还是类代理可自由选择。

**看清 ProxyFactory 的本质**（作者说这是"画匠"到"画师"的一步）：

- `AopProxy`（`getProxy()` / `getProxy(ClassLoader)`）抽象了两种代理机制，实现有 `JdkDynamicAopProxy`（**同时实现 `InvocationHandler`**）与 `Cglib2AopProxy`。
- `AopProxyFactory.createAopProxy(AdvisedSupport)` 用抽象工厂封装选择逻辑，唯一实现 `DefaultAopProxyFactory`，伪代码就一句：`if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config))` 造 Cglib2AopProxy，否则造 JdkDynamicAopProxy。
- `AdvisedSupport` 是"生成代理对象所需信息的载体"，一半来自 `ProxyConfig`（5 个 boolean：`proxyTargetClass`、`optimize`、`opaque`（能否转成 `Advised`）、`exposeProxy`（把代理绑到 ThreadLocal，可用 `AopContext.currentProxy()` 取）、`frozen`（配置冻结，优化性能）），一半来自 `Advised`（目标类、Advice、Advisor，可增删查询）。
- `ProxyCreatorSupport` 继承 `AdvisedSupport`，内部持一个 `AopProxyFactory`（默认 `DefaultAopProxyFactory`），提供 `createAopProxy()`。`ProxyFactory` 就是它的子类——**集 AopProxy 与 AdvisedSupport 于一身**，前者负责"取得代理对象"，后者负责"设置信息"。它的兄弟还有 `ProxyFactoryBean`、`AspectJProxyFactory`。

**二、ProxyFactoryBean 声明式**

断词是 **Proxy + FactoryBean**，不是 ProxyFactory + Bean——它本质是"用来生产 Proxy 的 FactoryBean"。容器里别的对象依赖它时，拿到的是 `getObject()` 返回的**代理对象**，这就是它能在容器里游刃有余的原因。`getObject()` 内部：`initializeAdvisorChain()` → 单例走 `getSingletonInstance()` 缓存，非单例走 `newPrototypeInstance()`。

独有属性：`proxyInterfaces`（Collection，等价于 `interfaces`）、`interceptorNames`（一次列出多个 Advice/拦截器/Advisor，替代一个个 `addAdvisor`；**特例**：没设 target 时可在最后一个元素放目标对象的 bean 名；元素名后加 `*` 可搜出所有匹配的 **global advisor**）、`singleton`（默认 true；需要**有状态代理**时——比如用 Introduction——才设 false）、`autodetectInterfaces`（默认 true，自动探测目标类实现的接口）。`proxyTargetClass=true` 可强制类代理，但客户端此后必须转成 `MockTask` 而非 `ITask`。

**三、自动代理**

一个个配 `ProxyFactoryBean`"估计得累得吐血"。自动代理基于 IoC 的 `BeanPostProcessor`，更准确说是 `InstantiationAwareBeanPostProcessor`——容器一旦发现有这种后处理器，就**直接用它构造的实例返回，走"短路"，不进正常实例化流程**，于是返回的已是代理对象。

- `BeanNameAutoProxyCreator`——"**半自动步枪**"：用 `beanNames` 指定要代理的 bean 名（支持 `target*`、`*service` 通配，`String[]` 还能用逗号分隔），`interceptorNames` 指定拦截器。可注册多个实例来细化范围。
- `DefaultAdvisorAutoProxyCreator`——"**全自动步枪**"：注册一行 bean 定义即可，它自动搜寻容器内所有 `Advisor`，按各自拦截信息为符合条件的目标对象生成代理。**只对 Advisor 有效**（只有 Advisor 才同时具备 Pointcut 与 Advice）。范围大，容易误伤，需细化 Advisor 定义或改用 BeanNameAutoProxyCreator。
- 扩展：继承 `AbstractAutoProxyCreator`（如 `BeanNameAutoProxyCreator`）或 `AbstractAdvisorAutoProxyCreator`（如 `DefaultAdvisorAutoProxyCreator`），只写规则匹配逻辑。Spring 2.0 后还有 `AspectJAwareAdvisorAutoProxyCreator` 及其子类 `AnnotationAwareAspectJAutoProxyCreator`。
- 便利点：所有 AutoProxyCreator 的父类都继承 `ProxyConfig`，**一处设 `proxyTargetClass` 就全局生效**，不必逐个 ProxyFactoryBean 配。

#### TargetSource 是什么，为什么要它

`TargetSource` 是**目标对象的"壳"或者说容器**，插足于调用链与实际目标对象之间。调用经历层层拦截到达链尾时，Spring AOP 不直接调目标对象，而是先向 `TargetSource.getTarget()` 要一个具体目标对象再调。**每次方法调用都会触发 `getTarget()`**——这就是它能做文章的地方：你可以借此控制每次调用落到哪个实例上（对象池、按规则轮换、动态替换）。

平时用 `setTarget()` / `setTargetName()` 设的目标对象，框架内部也会用 `SingletonTargetSource` 包一层，所以调用链终点是统一处理的。

现成实现（均在 `org.springframework.aop.target`）：

- `SingletonTargetSource`——持一个实例，永远返回它，**用得最多而我们往往不知道**（`setTarget` 的默认封装）。
- `PrototypeTargetSource`——每次调用返回**新实例**；要求目标 bean 的 scope 为 `prototype`，且必须用 `targetBeanName` 指定名字而非 `ref` 引用。
- `HotSwappableTargetSource`——作者最觉得有用。`swap(Object newTarget)` 在**运行期**换掉实现并返回旧对象，对调用者透明。作者 2005 年用它解决双数据源热备切换：`ThrowsAdvice` 捕获数据库异常后调 `swap` 换数据源。
- `CommonsPoolTargetSource`——借 Jakarta Commons Pool 做对象池；不够用可扩 `AbstractPoolingTargetSource`。
- `ThreadLocalTargetSource`——对 JDK `ThreadLocal` 的简单封装，每个线程一个目标实例；scope 必须 `prototype`，否则"异常伺候"。

接口只有四个方法：`getTargetClass()`、`isStatic()`（是否总返回同一实例，`SingletonTargetSource` 返回 true）、`getTarget()`（核心）、`releaseTarget(Object)`（`isStatic()` 为 false 时在调用结束后回调，多数时候可空实现）。作者示范了 `AlternativeTargetSource`：内部计数器，奇偶交替返回两个 `ITask`，实测输出 `Task2/Task1/Task2/Task1/Task2`。

#### 作者强调的坑

1. **注入目标对象而不是代理对象**——"初学者容易犯的错误"。把没织入任何横切逻辑的 `task` 注入依赖方，一定不会产生任何拦截效果。好实践：把目标对象声明为**内部 bean**，从根上杜绝误引用。
2. **代理对象不能强转成目标实现类**。`(MockTask) weaver.getProxy()` 会抛 `ClassCastException`，因为类型是 `$Proxy0`；只有走基于类的代理（CGLIB 生成 `MockTask` 的子类）时，才转得成目标具体类。
3. **Advisor 顺序**：不显式指定 `order` 就按声明顺序，Throws Advice 可能因此捕获不到别的 Advice 抛出的异常（大鹏的故障）。
4. **`DelegatingIntroductionInterceptor` 是"伪军"**：共享同一个 delegate，多实例状态会串。要真 per-instance 就换 `DelegatePerTargetObjectIntroductionInterceptor`，或把相关 bean 全部设成 `prototype`。
5. **正则 Pointcut 必须匹配整个方法签名**：`.*dosth.*` 能命中，`dosth.*` 命中不了。
6. **`ControlFlowPointcut` 与 `DynamicMethodMatcher` 性能差**（前者每次查调用栈，后者每次查参数且无法缓存），非必要不用。
7. **Introduction 的 Advice/Advisor 不能与其他类型混用**。
8. **`ThreadLocalTargetSource` / `PrototypeTargetSource` 依赖 `prototype` scope**。
9. **`DefaultAdvisorAutoProxyCreator` 打击面太大**，容易把横切逻辑织到不该织的对象上。
10. **写自定义 Pointcut 前先翻一遍现成的**——"务必先查看一下是否已经有可用的 Pointcut 实现"。

#### 本章结论

第一代 Spring AOP 确立的概念与原理，是整个框架的地基：后续版本无论怎么加特性，都秉承这些理念。理解它们，既能更好地使用 Spring AOP，也是扩展它的前提。


---

### 第10章 Spring AOP 二世

#### 一句话主旨

Spring 2.0 给 Spring AOP 换了张 AspectJ 的"皮"：用 POJO 加注解（或 Schema XML）声明 Aspect、Advice、Pointcut，但底层织入仍是 1.x 的代理体系，AspectJ 只贡献了 Pointcut 表达式的解析与匹配——"还是中国人，只不过多学了门外语"。

#### @AspectJ 形式的写法

一个最小可用、可直接跑的完整例子（目标类 + 切面 + 织入）：

```java
// 1. 目标类：普通 POJO，不需要实现任何接口
public class Foo {
    public void method1() { System.out.println("method1 execution."); }
    public void method2() { System.out.println("method2 execution."); }
}

// 2. 切面：普通 POJO + @Aspect
@Aspect
public class PerformanceTraceAspect {
    private final Log logger = LogFactory.getLog(PerformanceTraceAspect.class);

    // 3. 切入点：@Pointcut 标注在一个 void 方法上，方法名即 Pointcut Signature
    @Pointcut("execution(public void *.method1()) || execution(public void *.method2())")
    public void pointcutName() {}

    // 4. 通知：@Around 引用上面的 Pointcut Signature
    @Around("pointcutName()")
    public Object performanceTrace(ProceedingJoinPoint joinPoint) throws Throwable {
        StopWatch watch = new StopWatch();
        try {
            watch.start();
            return joinPoint.proceed();          // 放行到下一个拦截器 / 目标方法
        } finally {
            watch.stop();
            if (logger.isInfoEnabled()) {
                logger.info("Pr in method[" + joinPoint.getSignature().getName()
                        + "]>>>>>" + watch.toString());
            }
        }
    }
}
```

织入有两条路，二选一：

```java
// 方式 A：编程式织入（一般只用于测试）
AspectJProxyFactory weaver = new AspectJProxyFactory();
weaver.setProxyTargetClass(true);
weaver.setTarget(new Foo());
weaver.addAspect(PerformanceTraceAspect.class);
Foo proxy = (Foo) weaver.getProxy();
proxy.method1();
```

```xml
<!-- 方式 B：容器自动代理（推荐），一行搞定 -->
<aop:aspectj-autoproxy proxy-target-class="true"/>
<bean id="target" class="...Foo"/>
<bean id="performanceAspect" class="...PerformanceTraceAspect"/>
```

`<aop:aspectj-autoproxy>` 背后注册的就是 `org.springframework.aop.aspectj.annotation.AnnotationAwareAspectJAutoProxyCreator`（父类 `AspectJAwareAdvisorAutoProxyCreator`，祖父 `AbstractAdvisorAutoProxyCreator`）。也可以直接把它当 bean 声明，效果等价，只是啰嗦。使用 @AspectJ 必须把 `aspectjweaver.jar` 和 `aspectjrt.jar` 放进 Classpath——解析表达式要借 AspectJ 的类库。

#### Pointcut 表达式语法速查

`@Pointcut` 是方法级注解，必须挂在一个方法上，这个方法就是 Pointcut Signature。规则只有一条硬约束：**返回类型必须是 void**。修饰符语义同 Java：`public` 的 Signature 可被其他 Aspect 引用（写成 `包名.类名.方法名()`），`private` 只能本 Aspect 内用。表达式之间可用 `&&`、`||`、`!` 组合。

Spring AOP 只支持**方法执行**这一种 Joinpoint，所以只认下面这几个标志符：

| 标志符 | 匹配什么 | 例子 |
|---|---|---|
| `execution` | 按方法签名匹配方法执行，唯一能限定返回值/方法名/参数类型的 | `execution(public void cn.spring21.service.UserService.save(..))` |
| `within` | 指定类型（类或包）内声明的**所有**方法，静态匹配 | `within(cn.spring21.service.*)` |
| `this` | **代理对象**（proxy）的类型 | `this(cn.spring21.service.UserService)` |
| `target` | **目标对象**（target）的类型 | `target(cn.spring21.service.UserServiceImpl)` |
| `args` | 运行时**实参**的类型与个数，动态匹配 | `args(java.lang.String, ..)` |
| `@within` | 类上标注了指定注解 → 该类所有方法，静态匹配 | `@within(com.foo.Loggable)` |
| `@target` | 目标对象类上有指定注解，运行时动态匹配 | `@target(com.foo.Loggable)` |
| `@args` | 实参的**类型**上标注了指定注解 | `@args(com.foo.Sensitive)` |
| `@annotation` | **方法**上标注了指定注解 | `@annotation(org.springframework.transaction.annotation.Transactional)` |

**execution 的完整格式**（只有返回类型、方法名、参数三部分必填，其余可省）：

```
execution(modifiers-pattern? ret-type-pattern declaring-type-pattern?
          name-pattern(param-pattern) throws-pattern?)
```

两个通配符：`*` 匹配任意一段（一个 word），`..` 匹配任意多层包或任意多个参数。

```java
execution(public void Foo.doSomething(String))   // 最精确
execution(void doSomething(String))              // 省掉修饰符和类型
execution(* *(String))                           // 任意返回、任意方法、一个 String 参数
execution(* *(*))                                // 任意方法、恰好一个任意类型参数
execution(void cn.spring21.*.doSomething(*))     // 只到 cn.spring21 这一层
execution(void cn.spring21..*.doSomething(*))    // cn.spring21 及其所有子包
execution(void *.doSomething(..))                // 0 到多个参数，类型不限
execution(void doSomething(String, *))           // 两个参数，第二个任意
execution(void doSomething(.., String))          // 最后一个参数必须是 String
execution(void doSomething(*, String...))        // 第一个任意，其后是 String 及更多
```

**this 与 target 要特别小心**：AspectJ 原义里 `this` 指调用方（caller）、`target` 指被调方（callee）；Spring AOP 里 `this` 指代理对象、`target` 指目标对象。因为代理对象与目标对象通常实现同一接口（CGLIB 下代理是目标的子类），`this(ProxyInterface)` 和 `target(ProxyInterface)` 效果差不多；但写成具体实现类 `this(TargetFoo)` / `target(TargetFoo)` 时，基于接口的代理下只有 `target` 能匹配。两者也常组合使用来卡死规则：

```java
execution(void cn.spring21.*.doSomething(*)) && this(TargetFoo)
this(ProxyInterface) && target(ProxyInterface2)          // 代理只暴露接口 1，目标实现了接口 1+2
this(IntroductionInterface) && target(TargetObjectType)  // Introduction 给代理加了接口
```

**args 是动态的**，这是它和 execution 的关键差别：

```java
args(cn.spring21.unveilspring.domain.User)
// 能匹配 public boolean login(User user)
// 也能匹配 public boolean login(Object user)，只要运行时传的是 User 实例
// 但 execution(* *(User)) 匹配不到 login(Object)，因为它是静态匹配
```

理解这张表的另一条主线是**静态匹配 vs 动态匹配**：`execution`、`within`、`@within`、`@annotation` 只看方法签名和类/方法上的注解，在代理创建时就能算完；`args`、`@args`、`this`、`target`、`@target` 要看运行时传进来的实参或实际对象，只能在调用发生的那一刻判定。这也解释了为什么 `args` 能匹配 `login(Object)` 而 `execution(* *(User))` 不能。

日常最常用的几组，可以直接抄：

```java
// 某个包下所有 service 的公有方法
execution(public * cn.spring21.service..*.*(..))

// 所有 getter（按命名约定）
execution(* cn.spring21..*.get*(..))

// 只拦带 @Transactional 的方法
@annotation(org.springframework.transaction.annotation.Transactional)

// 只拦实现了某接口的 bean 的方法
execution(* *(..)) && this(cn.spring21.service.BizService)
```

`@within` 只接受注解类型，且对被该注解标注的**类**生效；`@target` 语义相近但运行时判定。`@args` 只接受注解类型，且**一个 @args 里只能写一个注解**（写两个 Spring AOP 不理睬，用 `&&` 拼）。`@annotation` 只认方法级注解（`@Target(ElementType.METHOD)`），典型用法就是 `@annotation(org.springframework.transaction.annotation.Transactional)`。

AspectJ 里的 `initialization`、`get`、`set`、`handler` 在 Spring AOP 中**不支持**，写了会抛 `IllegalArgumentException`。2.5 起额外支持 AspectJ 本身没有的 `bean()`。

内部实现：所有 @AspectJ 表达式最终都被包装成 `org.springframework.aop.aspectj.AspectJExpressionPointcut`——它照样实现 Spring 的 `Pointcut`（`ClassFilter` + `MethodMatcher`），只是把匹配工作委托给 AspectJ 的 `PointcutParser` 解析出的 `PointcutExpression`。所以匹配入口没变，变的只是"谁来算"。也可以直接 `new AspectJExpressionPointcut()` 后 `setExpression("execution(* someMethodName(..))")`，在 1.x 风格里照样能用。

#### 五种 Advice 注解

| 注解 | 对应 1.x 概念 | 方法能拿到什么 |
|---|---|---|
| `@Before` | MethodBeforeAdvice | 可选首参 `org.aspectj.lang.JoinPoint`；`args()`/`@annotation()` 绑定的业务参数 |
| `@AfterReturning` | AfterReturningAdvice | `JoinPoint`；`returning="retValue"` 绑定返回值 |
| `@AfterThrowing` | ThrowsAdvice | `JoinPoint`；`throwing="e"` 绑定异常实例 |
| `@After` | 1.x 没有对应接口 | `JoinPoint`；方法参数绑定（finally 语义） |
| `@Around` | MethodInterceptor | **必须**首参 `ProceedingJoinPoint`，`proceed()` / `proceed(Object[])` |

`JoinPoint` 提供 `getArgs()`、`getThis()`（代理对象）、`getTarget()`（目标对象）、`getSignature()`（方法签名）。除 `@Around` 与 Introduction 外，其余 Advice 都可以声明它，且可省略。

参数绑定靠"名字对上"：

```java
@Before(value = "execution(boolean *.execute(String, ..)) && args(taskName)")
public void setupResourcesBefore(String taskName) throws Throwable { ... }
// args 里的名字必须与方法参数名一致

@Pointcut("execution(boolean *.execute(String, ..)) && args(taskName)")
private void resourceSetupJoinpoints(String taskName) {}

@Before(value = "resourceSetupJoinpoints(taskName)")
public void setupResourcesBefore(String taskName) throws Throwable { ... }

// 一次绑多个：JoinPoint 永远排第一
@Before(value = "execution(boolean *.execute(String, ..)) && args(taskName) && @annotation(txInfo)")
public void setupResourcesBefore(JoinPoint jp, String taskName, Transactional txInfo) { ... }
```

```java
@AfterThrowing(pointcut = "execution(boolean *.execute(String, ..))", throwing = "e")
public void afterThrowing(JoinPoint jp, RuntimeException e) { ... }

@AfterReturning(pointcut = "execution(boolean *.execute(String, ..))", returning = "retValue")
public void taskExecutionCompleted(JoinPoint jp, boolean retValue) { ... }
```

`@Around` 最灵活：可以不调 `proceed()`（短路）、可以调多次（重试）、可以换参数再放行——`proceed(new Object[]{ modifiedTaskName })`。多参数时按方法定义顺序给，才能同时兼容 Spring AOP 与 AspectJ。

Introduction 用 `@DeclareParents`，注意它标的是**字段**不是方法：

```java
@Aspect
public class IntroductionAspect {
    @DeclareParents(value = "cn.spring21.unveilspring.service.*",
                    defaultImpl = CounterImpl.class)
    public ICounter counter;
}
```

Introduction 是 per-instance 的，目标对象 scope 通常应设 `prototype`。

**执行顺序**：同一个 Aspect 内，按声明顺序定优先级，先声明者优先级高——对 Before 是"先跑"，对 AfterReturning 是"后跑"（所以输出是 beforeOne、beforeTwo、…、afterReturningTwo、afterReturningOne）。不同 Aspect 之间，让 Aspect 实现 `org.springframework.core.Ordered`，`getOrder()` 值小者优先；不实现则顺序不确定。

**实例化模式**：默认 singleton；Spring 2.0 起只支持 singleton、`perthis`、`pertarget`（`@Aspect("perthis(execution(...))")`），且 bean 定义不能再写 `scope="singleton"`，否则报错。

#### 基于 Schema 的 AOP

先引入命名空间，再写 `<aop:config>`：

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
           http://www.springframework.org/schema/beans/spring-beans-2.0.xsd
           http://www.springframework.org/schema/aop
           http://www.springframework.org/schema/aop/spring-aop-2.0.xsd">

  <aop:config proxy-target-class="true">
    <aop:pointcut id="bizPointcut"
                  expression="execution(public void cn.spring21.service.*.*(..))"/>

    <aop:aspect id="myAspect" ref="schemaBasedAspect" order="2">
      <aop:pointcut id="privatePointcut" expression="execution(public void *.dosth())"/>
      <aop:before          pointcut-ref="privatePointcut" method="doBefore"/>
      <aop:after-returning pointcut-ref="privatePointcut" method="doAfterReturning"
                           returning="retValue"/>
      <aop:after-throwing  pointcut-ref="privatePointcut" method="doAfterThrowing"
                           throwing="e"/>
      <aop:after           pointcut-ref="privatePointcut" method="doAfter"/>
      <aop:around          pointcut-ref="privatePointcut" method="doProfile"/>
      <aop:declare-parents types-matching="cn.spring21.unveilspring.target.*"
                           implement-interface="cn.spring21.unveilspring.introduction.ICounter"
                           default-impl="cn.spring21.unveilspring.introduction.CounterImpl"/>
    </aop:aspect>
  </aop:config>

  <bean id="schemaBasedAspect" class="...SchemaBasedAspect"/>
</beans>
```

几个硬性细节：`<aop:config>` 只有 `proxy-target-class` 一个属性；子元素必须按 `<aop:pointcut>` → `<aop:advisor>` → `<aop:aspect>` 的顺序出现，同类可并列多个；一个文件里可以有多个 `<aop:config>`。`<aop:pointcut>` 放在 `<aop:config>` 下是全局共享，放在 `<aop:aspect>` 内相当于 private。XML 里 `&&` 是非法字符，逻辑运算符改用 `and` / `or` / `not`，框架解析时会替换回来。Advice 的参数名要么与方法参数同名，要么用 `arg-names` 显式指定。

`<aop:advisor>` 是 1.x Advisor 的等价物：`id`、`pointcut-ref`、`advice-ref`、`order`，另有一个只能写 AspectJ 表达式的 `pointcut` 属性。若走 `<aop:pointcut type="regex">` 配正则，注意这个 `type` 在当时的实现里被 `AopNamespaceHandler` 忽略，实际仍按 AspectJ 规则解析（JIRA SPR-4180），应避免使用。

**为什么说"两者本质是同一套实现"**：`<aop:config>` 底层仍交给 AutoProxyCreator 干活，只是由命名空间处理器把 XML 翻译成 Pointcut、Advisor、Aspect 这些 Spring 原生概念；`<aop:aspect>` 里的每个 `<aop:before>`/`<aop:around>` 最终注册的都是一条 Spring 的 Advisor（Advice 仍是 MethodInterceptor 那一套）。@AspectJ 那条路也一样——`AnnotationAwareAspectJAutoProxyCreator` 本身就是 `AbstractAdvisorAutoProxyCreator` 的子类，它反射读出注解后构造的同样是 `AspectJExpressionPointcut` + Spring Advisor。两条路在"生成 Advisor 列表 → 自动代理 → 拦截器链"这一段完全汇合，差别只在信息载体是注解还是 XML。所以 Schema AOP 不需要 Java 5，也不需要 `@Aspect` 注解，是给 Java 1.4 项目的折中方案；反过来它只支持 singleton 实例化模式，比 @AspectJ 少一点能力。

#### 作者强调的坑

- **编程式织入与容器织入行为不一致**。`AspectJProxyFactory.addAspect()` 的顺序决定 Advice 顺序，`Ordered` 接口被无视；只有走容器自动代理时 `Ordered` 才生效。所以除了测试，一律用容器自动代理。
- **`<aop:config>` 不要和显式声明的 AutoProxyCreator 混用**，否则可能出现某些 Advice 根本没织进去（官方文档和论坛都提过）。
- **Around 的首参必须是 `ProceedingJoinPoint`**，不能省、不能换成 `JoinPoint`；其余四种可以省。
- **`args`/`@annotation` 绑定的参数名必须与方法参数名逐字相同**，改名即失效（编译不报错，运行时拿不到值）。
- **`@args` 一次只能写一个注解类型**，多个请用 `&&`。
- **`@within` 静态、`@target` 动态**；`@annotation` 只认方法级注解，标在类上不生效。
- **`perthis`/`pertarget` 的 Aspect 不能配 singleton scope**，目标对象一般也要 prototype。
- **@AspectJ 需要 Java 5+ 与 aspectjweaver/aspectjrt**；Schema 方式不需要 Java 5，但只有 singleton。
- 所有 `@` 开头的标志符都只接受注解类型，注解本身要求 `RetentionPolicy.RUNTIME`。

#### 本章结论

至此 Spring AOP 有三种用法：第一代基于接口的 Advice（1.x）、@AspectJ（2.0）、基于 Schema（2.0）。功能上三者等效，因为后两代的底层仍是第一代的概念与实体。作者的选择是 @AspectJ，理由五条：纯 POJO 声明、注解作载体便于 IDE 重构、AOP 信息集中一处、强类型参数免去 `Object[]` 与强转、可用 Aspect 模块化管理公用 Pointcut。若受限于 Java 1.4，则退而用 Schema；若团队一直用第一代，也可以只把配置迁到 Schema——"新瓶装旧酒"。

### 第11章 AOP 应用案例

#### 一句话主旨

AOP 的最佳实践都长一个样：找出散落各处的、与业务无关的**横切关注点**（cross-cutting concern），收拢成一个 Aspect——异常处理的 Fault Barrier、安全检查、缓存，莫不如此。

#### 案例一：异常处理（Fault Barrier 模式）

先分清两类异常。`java.lang.Error` 与 `java.lang.RuntimeException` 及其子类合称 **unchecked exception**（编译器不检查）；`java.lang.Exception` 去掉 RuntimeException 分支是 **checked exception**（编译期强制处理）。

关键差别不在语法而在受众：checked exception 表示**可恢复、意料之中**的非正常状态，信息面向应用程序，调用方按类型分别处理（相当于升级版的错误码）；unchecked exception 表示**不可恢复**的严重故障——数据库挂了、网线断了、服务器崩了，信息是给人看的，让运维据此判断哪里出了问题、要不要人工介入。

那篇 "Effective Java Exception" 把前者叫 **Contingency**，后者叫 **Fault**。Fault 的处理逻辑高度同质：记日志、发邮件/短信通知责任人，几乎无差别对待。既然如此，它就不该在每个调用点各写一遍——这就是横切关注点。专门兜住所有 Fault 的 Aspect 就叫 **Fault Barrier**。

```java
@Aspect
public class ExceptionBarrierAspect {
    private JavaMailSender mailSender;
    private String[] receiptions;

    @AfterThrowing(pointcut = "execution(boolean *.execute(String, ..))", throwing = "e")
    public void afterThrowing(RuntimeException e) {
        final String exceptionMessage = ExceptionUtils.getFullStackTrace(e);
        getMailSender().send(new MimeMessagePreparator() {
            public void prepare(MimeMessage message) throws Exception {
                MimeMessageHelper helper = new MimeMessageHelper(message);
                helper.setSubject("...");
                helper.setTo(getReceiptions());
                helper.setText(exceptionMessage);
            }
        });
    }
    // getter/setter 略
}
```

要点：`throwing="e"` 把异常绑到方法参数；可按异常类型声明多个 `@AfterThrowing` 方法分别处理（对应 1.x ThrowsAdvice 的多方法重载）；不需要异常对象时就不声明参数。这个 Aspect 就是第 9 章 `ExceptionBarrierThrowsAdvice` 的注解版——同一个模式，两代写法。

边界也要说清楚：Fault Barrier 只兜 unchecked exception，别把 checked exception 也一股脑丢进去。后者承载的是业务语义（余额不足、库存不够），需要调用方按类型分支处理，一旦被统一吞掉再统一记日志，契约就废了。另外，unchecked exception 也可以被方法显式声明、被调用方 catch，这是特例；Barrier 的价值在于给"没人管的那部分"兜底，而不是取代就地处理。

#### 案例二：安全检查

`javax.servlet.Filter` 是 Servlet 规范给的"半个 AOP"，能给 Web 应用做资源访问控制，但只限 Web 领域。AOP 则不限领域：任何"某个点被访问前必须先验身份/权限"的地方都适用。

```java
@Aspect
public class SecurityAspect {
    @Around("execution(* cn.spring21.service.*.*(..))")
    public Object doCheck(ProceedingJoinPoint pjp) throws Throwable {
        if (isIllegalRequest(pjp)) {
            throw new SecurityCheckingException("necessary information");
        }
        return pjp.proceed();   // 校验通过才放行
    }
}
```

用 `@Around` 而不是 `@Before`，是因为"不满足条件就不放行"需要在拦截器里直接短路——不调 `proceed()` 即可，异常直接抛给调用方。落地前先别造轮子：Acegi（后更名并并入 Spring Portfolio，即 **Spring Security**）已提供完整的认证、授权、访问控制。

#### 案例三：缓存

缓存提升性能，但它是**系统需求**而非业务需求；缓存点散布各处，写进业务代码就会污染业务逻辑。用 Aspect 把它剥出来：

```java
@Aspect
public class CachingAspect {
    private static Map cache = new LRUMap(5);

    @Around("execution(* cn.spring21.service.*.find*(..)) && args(key)")
    public Object doCache(ProceedingJoinPoint pjp, Object key) throws Throwable {
        if (cache.containsKey(key)) {
            return cache.get(key);          // 命中，直接返回，不调 proceed
        } else {
            Object retValue = pjp.proceed(); // 未命中，执行原逻辑
            cache.put(key, retValue);
            return retValue;
        }
    }
}
```

这是原型代码，**没有同步**，生产环境必须考虑并发与失效策略。另外，`key` 怎么选决定了命中率与正确性：通常用"方法签名 + 实参"组合，只读、幂等、结果与调用次数无关的方法才适合缓存。现实做法是接现成缓存产品（EhCache、JBossCache），再用 Spring Modules 的声明式缓存把织入交给配置。

#### 本章结论

三个案例的共同骨架是：识别横切关注点 → 用 Aspect 收拢 → 用 Pointcut 决定织入点 → 用 Around/AfterThrowing 决定拦截行为。AOP 的适用面远不止这三类，更多场景要在实际开发中自己挖掘。

### 第12章 Spring AOP 之扩展篇

#### 一句话主旨

Spring AOP 用代理实现，所以"目标对象内部 `this` 调用另一个方法"不经过代理，切面必然失效；解法是把代理对象暴露给目标对象自己（`AopContext.currentProxy()` + `exposeProxy=true`），代价是与 Spring API 耦合。

#### 自调用导致切面失效的问题

**现象。** 目标类内部有嵌套调用：

```java
public class NestableInvocationBO {
    public void method1() {
        method2();                                  // 同一个对象内部调用
        System.out.println("method1 executed!");
    }
    public void method2() {
        System.out.println("method2 executed!");
    }
}
```

切面同时拦截 method1 和 method2（`@Pointcut("method1() || method2()")` + `@Around`）。从外部调 `proxy.method2()`，日志有 `Pr in method[method2]`，拦截成功；从外部调 `proxy.method1()`，日志里**只有** `Pr in method[method1]`，method1 内部那次 method2 的调用没有被拦到。

**原因。** 横切逻辑只织进了 `ProxyObject` 的方法里，`TargetObject` 本身是干净的。调用时序是：

```
proxy.method1
  → 拦截器链（记录开始时间 …）
  → target.method1          ← 调用转到目标对象，此后全程在 TargetObject 上
      → this.method2        ← 即 target.method2，不是 proxy.method2
  → 拦截器链（记录结束时间）
```

也就是说，进入代理的只有"第一次"调用；一旦 `proceed()` 把控制权交给目标对象，后续所有调用都发生在目标对象内部，`this` 指向的是 `TargetObject` 而非 `ProxyObject`，而 method2 的拦截器只挂在 `ProxyObject.method2` 上，自然无从触发。换一种说法：拦截器链是"包"在最外层调用外面的，`proceed()` 是链的出口而不是入口，出了这个出口就不再回来——链只对穿过代理边界的那一次调用负责。对照一下：如果目标对象依赖的是**别的**对象，只要注入的是那个对象的代理，拦截就正常——问题只出在"依赖自身"。

**解决方案。** 思路一致：依赖别人时注入别人的代理，那依赖自己时就注入自己的代理。Spring 提供 `AopContext.currentProxy()` 取当前目标对象对应的代理：

```java
public class NestableInvocationBO {
    public void method1() {
        ((NestableInvocationBO) AopContext.currentProxy()).method2();  // 走代理
        System.out.println("method1 executed!");
    }
    public void method2() { System.out.println("method2 executed!"); }
}
```

前提是把 `ProxyConfig`（或其子类）的 `exposeProxy` 打开，否则 `currentProxy()` 抛异常：

```java
AspectJProxyFactory weaver = new AspectJProxyFactory(new NestableInvocationBO());
weaver.setProxyTargetClass(true);
weaver.setExposeProxy(true);           // 关键
weaver.addAspect(PerformanceTraceAspect.class);
```

XML 方式对应 `<aop:config expose-proxy="true">`。这样两次 method2 都会被拦截。

但目标对象直接调用 Spring API 太难看，作者给了几种解耦思路：

1. 目标对象声明一个"自身代理"的实例变量，由构造器或 setter 注入 `AopContext.currentProxy()` 的结果；
2. 声明 `getThis()` 之类的方法，用 IoC 的**方法注入**或**方法替换**把它的实现换成 `return AopContext.currentProxy()`，调用处写 `getThis().method2()`；
3. 引入一个 wrapper 类持有 `getProxy()`，目标对象只依赖 wrapper，把 Spring API 隔在外面；
4. 定义统一接口，用 `BeanPostProcessor` 批量把实现类里"取当前代理"的方法逻辑替换掉——原理同方法替换，只是借容器批量处理。

最后一句提醒很实在：这个坑是**代理机制**固有的。AspectJ 直接把横切逻辑织进目标对象，代理与目标合为一体，压根不存在自调用失效；如果接受不了这些绕法，换 AspectJ 也是一种选择。

#### 本章结论

自调用失效不是 bug 而是代理模型的必然推论：拦截只发生在"调用穿过代理边界"的那一刻，`this` 调用不穿边界。记住"代理之外才拦截"，排查这类问题时先问一句：这次调用是从代理进来的，还是从对象内部发出的？Spring AOP 在简单与强大之间取得了不错的平衡，用对了很省事，用错了要先理解它的边界在哪。


---
