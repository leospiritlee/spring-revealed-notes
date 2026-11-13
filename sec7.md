# 第二部分　Spring 的 IoC 容器

### 第2章　IoC 的基本概念

#### 一句话主旨

IoC 就是一句话：**需要什么东西，别自己去拿，让别人送过来。**

#### 作者的类比：从"自己拿衣服"到"有人递衣服"

书里这个例子很生活化，但把概念讲透了：

> 出门之前得先穿件外套。以前，你得自己跑到衣柜前面取出衣服（依赖对象），然后自己穿上再出门。而现在，你只要跟"另一半"说一句"Honey，衣服拿来"，她就会心领神会地到衣柜那里为你取出衣服，再给你穿上。

"自己拿"是传统写法——被注入对象直接 `new` 依赖对象，两者硬绑定；"别人递"是 IoC——中间多了一个 **IoC Service Provider**（也就是容器），被注入对象只需要声明"我需要什么"，剩下的由容器负责。

**控制权从被注入对象手里，转移到了容器手里——这就是"反转"。**

#### 三种依赖注入方式

要让容器知道你需要什么，得通过某种方式"打招呼"。书里沿用 Martin Fowler 的分类，讲了三种：

**① 构造方法注入（constructor injection）**

在构造方法里声明依赖参数：

```java
public FXNewsProvider(IFXNewsListener newsListener, IFXNewsPersister newsPersister) {
    this.newsListener = newsListener;
    this.newsPersister = newsPersister;
}
```

类比：**你刚进酒吧的门，服务生已经把你爱喝的啤酒摆在桌上了**——坐下就能喝。

优点：对象构造完成即进入就绪状态，拿来就能用。
缺点：依赖一多，参数列表就长得难看；用反射构造时，同类型参数很难区分；构造方法不能继承、不能设默认值。

**② setter 方法注入（setter injection）**

给每个依赖属性配一对 getter/setter，容器调用 setter 把对象塞进去。

类比：**你先坐下，再慢慢决定喝什么**——百威也行，大雪也行，随意性更强。

优点：方法可以命名，描述性好；setter 可继承、可设默认值、IDE 支持好。
缺点：对象构造完还不能用，得等 setter 被调用之后才就绪。

**③ 接口注入（interface injection）**

被注入对象必须实现一个特定接口，接口里定义一个 `injectXxx` 方法，参数类型就是依赖对象的类型。

类比：**为了点一杯啤酒，你得戴上一顶啤酒杯形状的帽子**——能达成目的，但很别扭。

缺点：强制被注入对象实现不必要的接口，**侵入性强**。这种方式最早见于 Avalon 项目，今天基本处于"退役状态"。

#### 三种方式的取舍

| 方式 | 侵入性 | 就绪时机 | 今天的地位 |
|---|---|---|---|
| 构造器注入 | 弱 | 构造完即可用 | **推荐**（Spring 官方也推荐） |
| setter 注入 | 弱 | 构造后需再注入 | 常用（可选依赖、循环依赖场景） |
| 接口注入 | **强** | — | 已退役 |

#### IoC 的附加值：不只是"少写 new"

作者特意强调：IoC 的价值远不止"解耦"两个字。他用了 FXNews 项目的两次真实改动来说明：

**场景一：新增一个新闻源。** 客户说又签了 MarketWin24，需要接入。如果没有 IoC，`FXNewsProvider` 里写死了 `DowJonesNewsListener`，这个类就没法复用了，只能继承一份或者重写一份。用了 IoC 之后，`FXNewsProvider` **一行都不用改**，只要传入一个 `MarketWin24NewsListener` 就行：

```java
FXNewsProvider dowJones = new FXNewsProvider(new DowJonesNewsListener(), new DowJonesNewsPersister());
FXNewsProvider marketWin24 = new FXNewsProvider(new MarketWin24NewsListener(), new DowJonesNewsPersister());
```

**场景二：单元测试。** 想测"取新闻失败"的分支，直接塞一个 `MockNewsListener` 进去，让它抛异常即可。没有 IoC 的话，`FXNewsProvider` 和 `DowJonesNewsListener` 绑死，这个测试根本没法写。

**这就是 IoC 真正的价值：可重用性和可测试性，而不是"代码好看"。**

#### 本章结论

IoC 是一个**设计原则**（控制反转），依赖注入是它的**实现手段**。Spring 的容器就是那个"IoC Service Provider"。

记住一句话就够了：**以前是我去找依赖，现在是依赖来找我。**

---


---

### 第4章 Spring 的 IoC 容器之 BeanFactory

#### 一句话主旨

把"对象从哪来、依赖怎么接"这件事从业务代码里彻底抽走，交给一个只认 `BeanDefinition`（Bean 定义）的轻量级容器；再把容器从启动到销毁的每一步都摊开，让你看清它在哪些环节留了可以插手的扩展点。

#### 关键概念

- **BeanFactory**：最基础的 IoC 容器接口，只定义"怎么取"，不定义"怎么登记"。方法基本是查询类：`getBean(String)`、`getBean(String, Class)`、`getBean(String, Object[])`、`containsBean`、`isSingleton`、`isPrototype`、`isTypeMatch`、`getType`、`getAliases`。作者提醒：通常只有独立应用的主入口类才会与这组 API 直接耦合。
- **BeanDefinitionRegistry（Bean 定义注册表）**：抽象"Bean 怎么登记"的逻辑。作者比喻：**BeanDefinitionRegistry 就像图书馆的书架，BeanFactory 是图书馆**——你借书还书都跟图书馆打交道，可书实际放在书架上；书架相对图书馆，就是它的 "BookDefinitionRegistry"。这个比喻很关键，它解释了为什么两个接口要分开。
- **DefaultListableBeanFactory**：最通用的实现类，同时实现 `BeanFactory` 与 `BeanDefinitionRegistry`，所以 `(BeanFactory) registry` 强转成立。作者特意加了小心提示：**单纯的 BeanDefinitionRegistry 无法强转到 BeanFactory**，别以为这招到哪都能用。
- **BeanDefinition**：每个受管对象在容器里都有一个实例与之对应，保存 class 类型、是否抽象类、构造方法参数、属性值等全部"生产图纸"信息；客户端要对象时，容器就是照着它把对象装配出来。主要实现是 `RootBeanDefinition` 与 `ChildBeanDefinition`。
- **BeanDefinitionReader**：外部配置文件的解析器，职责链是"读文件 → 解析 → 映射成 BeanDefinition → 塞进 BeanDefinitionRegistry"。`PropertiesBeanDefinitionReader` 管 properties，`XmlBeanDefinitionReader` 管 XML。作者强调分工：解析、装配都由 Reader 干，Registry 只不过负责保管。
- **XmlBeanFactory**：在 `DefaultListableBeanFactory` 上再包一层，把"读 XML + 建容器"压成一行 `new XmlBeanFactory(new ClassPathResource(...))`，是简化版入口。
- **BeanWrapper**：对象造好后不直接返回，而是用 `BeanWrapperImpl` 把它"包裹"起来，供后续 `setPropertyValue` 设置属性。它继承 `PropertyAccessor`，又直接或间接继承 `PropertyEditorRegistry` 和 `TypeConverter`——所以 `CustomEditorConfigurer` 注册的那批 `PropertyEditor`，最终是被它拿去干活的（构造 `BeanWrapperImpl` 时会复制一份进去）。
- **InstantiationStrategy（实例化策略）**：用策略模式决定"怎么 new"。`SimpleInstantiationStrategy` 只靠反射，不支持方法注入；`CglibSubclassingInstantiationStrategy` 在其上加了 CGLIB 动态字节码生成，能造子类，因而满足方法注入需求。**容器默认采用后者**。
- **BeanFactoryPostProcessor**：容器**启动阶段**的扩展点，在对象实例化之前修改 BeanDefinition 里保存的信息，相当于在启动阶段末尾加一道工序。典型实现：`PropertyPlaceholderConfigurer`、`PropertyOverrideConfigurer`、`CustomEditorConfigurer`。
- **BeanPostProcessor**：**实例化阶段**的扩展点，处理的是已经 new 出来的对象实例。两个方法 `postProcessBeforeInitialization(Object bean, String beanName)` 与 `postProcessAfterInitialization(...)` 分别卡在初始化方法前后。区分口诀：一个在启动阶段改图纸，一个在实例化阶段改成品。
- **Aware 接口**：让 Bean 反向拿到容器给它的"身份材料"。BeanFactory 侧三个：`BeanNameAware`（塞 beanName）、`BeanClassLoaderAware`（塞加载该 bean 的 ClassLoader）、`BeanFactoryAware`（塞容器自身）。ApplicationContext 侧四个：`ResourceLoaderAware`、`ApplicationEventPublisherAware`、`MessageSourceAware`、`ApplicationContextAware`，后四个是靠 `ApplicationContextAwareProcessor` 这个 BeanPostProcessor 完成的，机制与前三个不同。
- **scope（作用域）**：声明对象该待在什么场景、活多久——进入 scope 前容器生成并装配，脱离 scope 后容器通常销毁。作者比喻：把"中学教师"作为一个类定义，容器初始化后教师只能局限在中学这个场景里，中学就是他的 scope。
- **singleton**：一个容器内只存在一个共享实例，从第一次被请求而初始化起，一直活到容器退出，"几乎"与容器同寿。**与 GoF 的 Singleton 模式语义不同**：这里由容器保证"同一容器一个实例"，GoF 保证"同一 Classloader 一个实例"。比喻：每个幼儿园配一个滑梯，全园小朋友共用。
- **prototype**：每次请求都重新生成一个新实例，返回后容器不再持有引用，后续生命周期（含销毁）由请求方自己负责。比喻："排排坐，分果果"，每人发一个新苹果，吃完果核自己扔；若错设成 singleton，就成了大家轮流啃同一个苹果，最后来的人只能哭鼻子。有状态的对象（如保存单个顾客信息的对象）才该用它。
- **FactoryBean**：注意断句——主语是 Bean，定语是 Factory。它本身也是个注册到容器的 Bean，只是这个 Bean 的职责是"生产对象"。三个方法：`getObject()` 给出实例化逻辑，`getObjectType()` 返回产物类型（无法确定则返回 null），`isSingleton()` 决定产物是否以单例形式存在。作者称其为对付复杂实例化逻辑的"制式装备"；与容器名 BeanFactory 无关，千万别混。
- **lookup-method（方法注入）/ replaced-method（方法替换）**：前者让某方法每次被调用都向容器要一个新实例，容器用 CGLIB 动态生成子类覆写它；后者用 `MethodReplacer.reimplement(Object target, Method method, Object[] args)` 整个换掉方法体，等于一个简易方法拦截。作者把这节命名为"偷梁换柱之术"。

#### 核心流程：Bean 的一生

作者先把容器劈成两个阶段：**容器启动阶段**（收集图纸）与 **Bean 实例化阶段**（按图纸生产）。原话是：第一阶段只是"根据图纸装配生产线"，第二阶段才是"用装配好的生产线生产具体产品"。两者各自留了扩展点，可以按场景插自定义逻辑。

**阶段一：加载 BeanDefinition（启动阶段）**

`BeanDefinitionReader.loadBeanDefinitions(...)` 读入 Configuration Metadata → 解析 → 编组成 `BeanDefinition` → `BeanDefinitionRegistry.registerBeanDefinition(...)` 登记。此时容器手里只有图纸，一个对象都没造出来。这个阶段偏准备性，重点是对象管理信息的收集，验证性、辅助性的工作也可以在这时做。

收尾工序是 `BeanFactoryPostProcessor.postProcessBeanFactory(beanFactory)`：BeanFactory 要自己手动调；ApplicationContext 会自动识别配置文件里的实现类并应用。

**阶段二：实例化（由 `getBean()` 触发）**

入口是 `AbstractBeanFactory.getBean()`：先查缓存，命中直接返回（prototype 除外）；未命中才转 `AbstractAutowireCapableBeanFactory.createBean()`。要注意 `getBean()` 既可能被客户端显式调用，也可能被容器隐式调用，隐式有两种情形：一是 BeanFactory 下对象 A 被首次请求时，它依赖的 B 还没实例化，容器内部就先替它 `getBean` 一次 B；二是 ApplicationContext 在启动阶段收尾后，紧接着对所有 bean 定义循环调用 `getBean()`。这也是"BeanFactory 惰性、ApplicationContext 饿汉"的根源。之后按图 4-10 走：

1. **实例化对象**：`createBean` 依 BeanDefinition 提供的信息，选 `InstantiationStrategy`（默认 `CglibSubclassingInstantiationStrategy`）反射或 CGLIB 造实例。
2. **包装**：造完不直接返回，包成 `BeanWrapperImpl`，并把 `CustomEditorConfigurer` 注册的 `PropertyEditor` 复制进去——这一步就是为了下一步设属性。
3. **属性填充**：容器调 `BeanWrapper.setPropertyValue(...)` 注入依赖，顺带靠 `TypeConverter` 完成 String 到目标类型的转换。
4. **Aware 回调**：检测 `BeanNameAware` 调 `setBeanName`；检测 `BeanClassLoaderAware` 调 `setBeanClassLoader`；检测 `BeanFactoryAware` 调 `setBeanFactory`，把容器自身塞进去。
5. **BeanPostProcessor 前置**：遍历所有 BPP，调 `postProcessBeforeInitialization(bean, beanName)`。ApplicationContext 那四个 Aware 就在这一步由 `ApplicationContextAwareProcessor` 依次 `instanceof` 判断后注入。
6. **初始化方法**：若实现 `InitializingBean`，调 `afterPropertiesSet()`；再调 `<bean init-method="...">` 指定的方法（或用 `<beans default-init-method>` 统一指定）。两者都有时按此先后顺序都执行。
7. **BeanPostProcessor 后置**：调 `postProcessAfterInitialization(bean, beanName)`，AOP 代理多半在此织入，如 `BeanNameAutoProxyCreator`。
8. **注册销毁回调**：仅对 singleton，若实现 `DisposableBean` 或有 `destroy-method`，注册一个销毁 Callback，然后对象进入"在用"状态。
9. **销毁**：容器关闭时执行回调。BeanFactory 要手动 `((ConfigurableListableBeanFactory) container).destroySingletons()`；ApplicationContext 用 `((AbstractApplicationContext) container).registerShutdownHook()`，底层是 `Runtime.addShutdownHook()`。

特例：`InstantiationAwareBeanPostProcessor` 会在第 1 步**之前**介入，直接造出对象并返回，造成流程"短路"，不再走后续常规步骤；Spring 内部用它做动态代理之类的事。

#### 代码/配置要点

`<beans>` 是顶层"统帅"，它用五个 default-* 属性统一给下属 `<bean>` 设默认值，避免逐个重复：`default-lazy-init`（true/false）、`default-autowire`（no/byName/byType/constructor/autodetect）、`default-dependency-check`（none/simple/object/all）、`default-init-method`、`default-destroy-method`。

直接编码注册（最能看清容器底层在干什么）：

```java
AbstractBeanDefinition newsProvider = new RootBeanDefinition(FXNewsProvider.class, true);
registry.registerBeanDefinition("djNewsProvider", newsProvider);

ConstructorArgumentValues argValues = new ConstructorArgumentValues();
argValues.addIndexedArgumentValue(0, newsListener);
argValues.addIndexedArgumentValue(1, newsPersister);
newsProvider.setConstructorArgumentValues(argValues);

MutablePropertyValues pvs = new MutablePropertyValues();
pvs.addPropertyValue(new PropertyValue("newsListener", newsListener));
newsProvider.setPropertyValues(pvs);
```

properties 格式的独有语法（`.(class)` 声明类型，`$0(ref)` 按构造参数下标注入，`(ref)` 不可省，否则会被当成 String）：

```properties
djNewsProvider.(class)=..FXNewsProvider
djNewsProvider.$0(ref)=djListener
djNewsProvider.$1(ref)=djPersister
djNewsProvider.newsListener(ref)=djListener
```

XML 的两种注入 + 内嵌集合元素：

```xml
<bean id="djNewsProvider" class="..FXNewsProvider">
  <constructor-arg index="0"><ref bean="djNewsListener"/></constructor-arg>
  <constructor-arg index="1" ref="djNewsPersister"/>
  <property name="newsListener" ref="djNewsListener"/>
  <property name="params">
    <list><value>something</value><ref bean="someBeanName"/></list>
  </property>
</bean>
```

继承与模板（`abstract="true"` 的定义不对应任何对象、不被实例化，也是少数可以省略 class 的场景）：

```xml
<bean id="newsProviderTemplate" abstract="true">
  <property name="newPersistener"><ref bean="djNewsPersister"/></property>
</bean>
<bean id="subNewsProvider" parent="newsProviderTemplate" class="..SpecificFXNewsProvider">
  <property name="newsListener"><ref bean="specificNewsListener"/></property>
</bean>
```

prototype 陷阱的解法——方法注入与方法替换：

```xml
<bean id="newsBean" class="..domain.FXNewsBean" singleton="false"/>
<bean id="mockPersister" class="..impl.MockNewsPersister">
  <lookup-method name="getNewsBean" bean="newsBean"/>
</bean>

<bean id="djNewsProvider" class="..FXNewsProvider">
  <replaced-method name="getAndPersistNews" replacer="providerReplacer"/>
</bean>
```

工厂方法与 FactoryBean（静态工厂用 `factory-method`，实例工厂用 `factory-bean` + `factory-method`；`<constructor-arg>` 传的是**工厂方法**的参数，不是工厂类的构造参数）：

```xml
<bean id="bar" class="..StaticBarInterfaceFactory" factory-method="getInstance"/>
<bean id="barFactory" class="..NonStaticBarInterfaceFactory"/>
<bean id="bar2" factory-bean="barFactory" factory-method="getInstance"/>
```

#### 作者反复强调的坑

1. **BeanFactory 默认延迟初始化，ApplicationContext 不是。** 前者要等 `getBean()` 第一次被调（显式或隐式）才实例化；后者启动完就把所有 singleton 定义实例化完，所以拿到 ApplicationContext 引用时对象已经齐了——证据在 `AbstractApplicationContext.refresh()`。默认行为通常是对的，问题能在第一时间暴露，代价是启动变慢。
2. **lazy-init 会被依赖方拖累而失效。** 若某个非延迟的 bean 通过 `<ref>` 依赖了 `lazy-init="true"` 的 bean，容器按依赖顺序还是得先实例化它，"延迟初始化的良好打算泡汤"。要真延迟，得让整条依赖链都延迟——用 `<beans default-lazy-init="true">` 一声令下最省事。
3. **prototype 注入到单例里，只注入一次。** 单例持有的是容器第一次给的引用，之后不再向容器申请，所以每次调 `getNewsBean()` 拿到的都是同一个对象（输出两次相同的 `@1662dc8`）。解法三选一：`<lookup-method>`、实现 `BeanFactoryAware` 自己 `getBean("newsBean")`、注入 `ObjectFactoryCreatingFactoryBean` 产出的 `ObjectFactory`（后者隔离了对 BeanFactory 的直接引用，还可用 `ServiceLocatorFactoryBean` 换成自定义工厂接口）。
4. **BeanPostProcessor 与 BeanFactoryPostProcessor 别搞混，注册方式也不同。** BeanFactory 要手写 `addBeanPostProcessor(...)` 与 `postProcessBeanFactory(...)`；ApplicationContext 只要在 XML 里配个 `<bean>` 就自动识别加载。用 BeanFactory 时忘了手动调，扩展逻辑等于白写。
5. **销毁方法不会自动执行。** `DisposableBean` / `destroy-method` 只是注册回调；不调 `destroySingletons()` 或 `registerShutdownHook()`，自定义销毁逻辑"形同虚设，根本不会被执行"。而且这套规则**完全不覆盖 prototype**——容器发完就不管了。
6. **autowire 与 dependency-check 都是双刃剑。** byType 下多出一个同类型 bean 系统就崩，byName 下把同名 bean 换了类型就出问题；自动绑定对原生类型、String、Class 及其数组**无效**；手工明确指定的绑定总会覆盖自动绑定。作者态度很直白：有好的 XML 编辑器，不介意多敲几个字符，起码对系统行为有完全把握。
7. **注解方式不是"万能替代品"。** 作者说 BeanFactory"几乎"支持所有方式，原因有二：Spring 2.5 之前框架并未正式支持基于注解的依赖注入；即便到 2.5，若不用 classpath-scanning，注解方式仍部分依赖 XML。用注解时 `@Autowired` 只负责"声明要注入什么"，`@Component` 还得靠 `<context:component-scan base-package="..."/>` 这个"触发器"去扫描注册，两者缺一不可。
8. **几个容易踩的小语法。** `<value></value>` 得到的是空串 `""` 而不是 null，要 null 必须用 `<null/>`；`index` 从 0 开始；内部 `<bean>` 的 id 不是必需的；`<ref>` 的 local 只能指同文件、parent 只能指父容器、bean 基本通吃，所以平时直接用 bean；想提前校验 beanName 拼写用 `<idref>` 而不是 `<value>`。

#### 本章结论

Spring 的 IoC 容器主要有两种：BeanFactory 和 ApplicationContext，本章聚焦基础容器 BeanFactory。作者先对比"用 BeanFactory 前后"的差别——变的只是依赖解决方式，从自己 Pull 变成让容器 Push，系统该怎么设计还怎么设计；再说明 BeanFactory 作为 IoC Service Provider，支持编码、properties、XML、注解多种 Configuration Metadata 组织方式，其中 **XML 是支持最完整、功能最强大的表达方式**，所以用整整一节把 `<beans>` 与 `<bean>` 的潜力挖了一遍。最后掀开黑匣子，讲了启动阶段（BeanDefinition 收集 + BeanFactoryPostProcessor 干预）与实例化阶段（Bean 的一生）两条主线。落点是一句预告：BeanFactory 是基础容器但并非唯一容器，ApplicationContext 构建于其上并提供了许多额外特性，下一章见。


---

### 第5章 Spring IoC 容器 ApplicationContext

#### 一句话主旨

ApplicationContext 没有另起一套容器内核，它站在 BeanFactory 之上，用「实现接口 + 把活委派出去」的方式挂上了四项能力：统一资源加载（ResourceLoader / ResourcePatternResolver）、国际化信息（MessageSource）、容器内事件发布（ApplicationEventPublisher）、以及简化的多配置文件加载。

#### 关键概念

- **InputStreamSource / Resource**：`org.springframework.core.io.InputStreamSource` 只定义 `getInputStream()`；`Resource` 继承它，共定义 7 个方法，用于查询资源状态、访问内容、按相对路径派生新资源（`createRelative(String relativePath)`、`getFilename()`、`getDescription()` 等）。文中出现的实现有 `ClassPathResource`、`FileSystemResource`、`UrlResource`。想自定义资源，作者建议继承 `org.springframework.core.io.AbstractResource` 抽象类而非硬实现接口——「什么？让我给个实现的例子？算了吧，目前我还没碰到这样的需求。」
- **ResourceLoader**：`Resource getResource(String location)` + `ClassLoader getClassLoader()`，另有常量 `CLASSPATH_URL_PREFIX`。作者认为叫「统一资源定位器」或许更贴切。
- **ResourcePatternResolver**：继承 `ResourceLoader`，新增 `Resource[] getResources(String locationPattern)` 与新前缀 `classpath*:`，一次返回多个资源。
- **MessageSource**：三个 `getMessage(...)` 重载，把「先按 Locale 取 ResourceBundle、再按键查条目」两步并成一步。
- **ApplicationEvent / ApplicationListener**：前者继承 `java.util.EventObject`，是抽象类；后者继承 `java.util.EventListener`，只有 `onApplicationEvent(ApplicationEvent)` 一个方法。
- **ApplicationContext 比 BeanFactory 多了什么**：`ResourcePatternResolver`（因而间接是 `ResourceLoader`）、`MessageSource`、`ApplicationEventPublisher`（`void publishEvent(ApplicationEvent event)`），外加多配置文件批量加载。

#### 三个能力逐个拆

##### 统一资源加载

查找逻辑分三层：

1. `DefaultResourceLoader.getResource()`：先看路径是否 `classpath:` 打头，是则造 `ClassPathResource`；否则试 URL，只要不抛 `MalformedURLException` 就造 `UrlResource`；仍不行则委派 `getResourceByPath(String)`，其默认实现还是造 `ClassPathResource`。
2. `FileSystemResourceLoader` 继承 `DefaultResourceLoader`，只覆写 `getResourceByPath(String)`，改从文件系统加载并返回 `FileSystemResource`。
3. `PathMatchingResourcePatternResolver` 支持 Ant 风格 `**/*.suffix` 与 `classpath*:`；构造时可传入一个 `ResourceLoader` 替换内部默认的 `DefaultResourceLoader`，不传则「在加载资源的行为上会与 DefaultResourceLoader 基本相同，只存在返回的 Resource 数量上的差异」。

ApplicationContext 怎么「白捡」这些能力：`ApplicationContext` 继承 `ResourcePatternResolver`；所有实现类直接或间接继承 `AbstractApplicationContext`，后者继承 `DefaultResourceLoader`（`getResource` 直接复用），内部又声明一个 `resourcePatternResolver`（实例为 `PathMatchingResourcePatternResolver`），构造时把**自身**这个 `DefaultResourceLoader` 传进去。作者原话：「说白了，ApplicationContext 的实现类在作为 ResourceLoader 或者 ResourcePatternResolver 时候的行为，完全就是委派给了 PathMatchingResourcePatternResolver 和 DefaultResourceLoader 来做。」

注入侧两条路：依赖 `ResourceLoader` 时，可 `implements ResourceLoaderAware`（`setResourceLoader`）或 `ApplicationContextAware`（`setApplicationContext`），容器启动时把自身塞进去，配置里只剩一个 `<bean id="fooBar" class="...FooBar"/>`，作者感叹「哇，简洁多了不是嘛？」；但若只需要 ResourceLoader，用 `ResourceLoaderAware` 更克制，`ApplicationContextAware`「相对来说过于宽泛了些」。依赖 `Resource` 时，`ApplicationContext` 启动伊始通过 `ResourceEditorRegistrar` 注册 `ResourceEditor`，于是 `<property name="template" value="..resources.default_template.vm"/>` 里的字符串能自动转成 `Resource`；要注入一组 Resource，则用 `ResourceArrayPropertyEditor`，经 `CustomEditorConfigurer` 告知容器。

##### 国际化

Java SE 底座是 `Locale` + `ResourceBundle`：`messages.properties` / `messages_zh_CN.properties`，`messages` 是 basename，后面追加语言或地区代码；`ResourceBundle.getBundle(baseName, locale)` 取包。Spring 的 `MessageSource` 把这层包起来。

`ApplicationContext` 默认委派给容器中**名为 `messageSource`** 的 `MessageSource` 实现；找不到就默认实例化一个不含任何内容的 `StaticMessageSource`。三种实现：`StaticMessageSource`（编程式 `addMessage`，多用于测试，「不应该用于正式的生产环境」）、`ResourceBundleMessageSource`（缓存 ResourceBundle 与 MessageFormat，生产首选）、`ReloadableResourceBundleMessageSource`（用 `cacheSeconds` 定期检查 properties 是否变更，可经 ResourceLoader 加载，如 `file:conf/messages`）。三者都能脱离容器以 Standalone 形式独立运行。

业务对象取用的两条路：实现 `MessageSourceAware`（作者提醒这使「该业务对象对 ApplicationContext 容器的依赖性就太强了，显得容器具有较强的侵入性」），或老实 setter 注入那个 `messageSource` bean（`<property name="messageSource" ref="messageSource"/>`）。Web 应用里把 ApplicationContext 公开给视图层、让标签直接取国际化信息，才是 ApplicationContext 实现该接口的额外理由。

##### 容器内事件发布

Java SE 标准三件套：自定义事件（继承 `EventObject`）、自定义监听器（继承 `EventListener`）、事件发布者（内部持一个 `List`）。作者用 `MethodExecutionEvent`（带 `methodName`）+ `MethodExecutionEventListener`（`onMethodBegin` / `onMethodEnd`）+ `MethodExeuctionEventPublisher`（`methodToMonitor()` 在方法首尾各发一次事件）走了一遍完整流程。

作者指出这个自造发布者只操心两件事：一是「具体时点上自定义事件的发布」（`methodToMonitor()` 是事件源头），二是「自定义事件监听器的管理」（`addMethodExecutionEventListener`、`removeListener`、`removeAllListeners`）。

Spring 版把三者换成 `ApplicationEvent` / `ApplicationListener` / `ApplicationEventPublisher`，但发布职责再往下转包给 `ApplicationEventMulticaster`：接口 → `AbstractApplicationEventMulticaster`（管监听器注册）→ `SimpleApplicationEventMulticaster`（管发布，默认用 `SyncTaskExecutor`，同步顺序）。容器启动时先找名为 `applicationEventMulticaster` 的 bean，没有才默认建一个 `SimpleApplicationEventMulticaster`。内置事件三种：`ContextClosedEvent`、`ContextRefreshedEvent`、`RequestHandledEvent`（子类 `ServletRequestHandledEvent` 提供 Servlet 相关事件）。

改造示例的关键：因为 `ApplicationListener` 只有 `onApplicationEvent` 一个入口，原先带两个方法的监听器接口失去意义，于是把 `MethodExecutionStatus` 塞进事件对象，监听器里 `if(evt instanceof MethodExecutionEvent)` 再分支；发布者改为 `implements ApplicationEventPublisherAware`，调注入的 `eventPublisher.publishEvent(...)`。作者提示：若每次发布把当前系统时间一并带过去，「你完全可以通过这样的方式来监控系统性能了！要知道，完全可以在这个的基础上往简单的 AOP 迈进哦！」

#### 作者强调的坑

1. **`DefaultResourceLoader` 会「造」一个不存在的资源**：`getResource("D:/spring21site/README")` 返回的是 `ClassPathResource` 而非预期的 `FileSystemResource`，`exists()` 为 false，且不抛异常——「这是由 DefaultResourceLoader 的资源查找逻辑所决定的」。要预期类型就换 `FileSystemResourceLoader`。
2. **无前缀路径的默认落点因容器而异**：`ClassPathXmlApplicationContext("conf/appContext.xml")` 等价于加了 `classpath:`；`FileSystemXmlApplicationContext` 同参数则走文件系统。
3. **`classpath:` 只在实例化阶段「强制」一次**（书中的「小心」）：即便用 `classpath:` 让 `FileSystemXmlApplicationContext` 从 classpath 读 bean 定义，容器启动后它作为 ResourceLoader 加载别的资源时，**不是每处都写 `classpath:` 就仍会回落到文件系统**。
4. **BeanFactory 不认 Resource**：默认不提供 Resource 的 PropertyEditor，必须自己注册；ApplicationContext 才自动注册 `ResourceEditor`。
5. **properties 编码**：规范要求 ISO-8859-1，中文得先用 `native2ascii` 之类工具转码。
6. **`messageSource` 这个名字是硬约定**：名字不对就静默退化成空的 `StaticMessageSource`，调用不报错但取不到内容。
7. **`ReloadableResourceBundleMessageSource` 别把资源放 classpath**，否则定期刷新失去意义。
8. **监听器要能注销**：不提供 remove 方法，注册过的实例会被发布者一直引用，即使早已废弃，造成「隐性的内存泄漏」。发布时还要对监听器列表做 safe-copy，避免处理期间增删影响遍历；且发布是同步顺序执行，「事件监听器的处理逻辑应该尽量简短」。
9. **事件机制别越界**：只适合单一容器内的简单消息通知，不适合分布式、多进程、多容器；想靠 Remoting「曲折一点」实现复杂需求，「难免弊大于利，失大于得」。

#### 本章结论

四项新特性没有一项是重写内核，全是「实现某接口 + 把活委派给专门组件」：资源靠 `PathMatchingResourcePatternResolver` 与 `DefaultResourceLoader`，国际化靠容器里那个叫 `messageSource` 的 bean，事件靠 `ApplicationEventMulticaster`。多配置加载更简单，构造器直接收 `String[]`，甚至用通配符 `conf/**/*.springxml`；作者自嘲拿 `XmlBeanFactory` 层层嵌套父子容器的对比是「故意给出了一个比较损的对比」，`<import>` 的真正问题在于「需要时刻关注主配置文件与其他配置文件的一致性」。另外 `ClassPathXmlApplicationContext` 还能以某个类（如 `MessengerService.class`）在 classpath 中的位置为基准定位同目录的 `services.xml`、`daos.xml`，不必写全路径。

### 第6章 Spring IoC 容器之扩展篇

#### 一句话主旨

Spring 2.5 的注解式依赖注入只换了依赖关系的「表达方式」，底层机制没变——依旧是构造 `BeanDefinition` 注册进容器、再由 `BeanPostProcessor` 完成注入；classpath-scanning 则进一步把那些「光秃秃的 bean 定义」也从 XML 里清出去。

#### 关键概念

- **@Autowired**：byType 语义，但比 `autowire="byType"` 灵活，可标注三处——字段（`private`/`protected`/`public` 都行）、构造方法（抢了 `autowire="constructor"` 的活，按构造参数类型定依赖）、任意方法（不限 setter，只要方法有需要注入的参数，如 `setUp(...)`）。
- **@Qualifier**：byName 的注解版。「既然 IoC 容器无法自己从多个同一类型的实例中选取我们真正想要的那个，那么我们不妨就使用 @Qualifier 直接点名要哪个好了。」可标字段，也可标方法/构造方法参数；还能标注注解类型以做自定义。
- **@Resource（JSR250）**：byName 语义，`@Resource(name="djNewsListener")`，按名字找同 beanName 的实例；可标字段或方法，位置与 @Autowired 大致相同。
- **@PostConstruct / @PreDestroy**：不服务于依赖注入，管生命周期，对应 `InitializingBean`/`DisposableBean` 与 `init-method`/`destroy-method`，用哪套「可以根据个人的喜好自己决定」。
- **为什么注解需要 BPP**：书中先用一段原型代码点破——遍历容器里所有 bean，用反射看字段/构造方法/方法上有没有 @Autowired，有就 `setAccessiableIfNecessary(f)` 后 `f.set(getBeanByTypeFromContainer())`。看到这段「笨办法」，自然就想到把它做成一个 `BeanPostProcessor`，而 Spring 正好提供了现成的实现，所以「很幸运，我们不用自己去实现它了」。
- **两个牵线人**：`AutowiredAnnotationBeanPostProcessor`（Spring 派）、`CommonAnnotationBeanPostProcessor`（JSR250 派）。注解本身不会自己生效——「天上永远不会掉馅饼」。
- **classpath-scanning**：从 base package 起扫描，扫到带注解的类就提取信息构建 `BeanDefinition` 注册进容器；默认认 `@Component` 及其细化版 `@Repository`、`@Service`、`@Controller`。

#### 代码/配置要点

1. 只用 @Autowired 不够，必须把 BPP 交给容器：
```xml
<bean class="org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor"/>
```
2. JSR250 派：
```xml
<bean class="org.springframework.context.annotation.CommonAnnotationBeanPostProcessor"/>
```
3. 一次配齐（须用 XSD 命名空间，`spring-context-2.5.xsd`）：
```xml
<context:annotation-config/>
```
它注册四个：`AutowiredAnnotationBeanPostProcessor`、`CommonAnnotationBeanPostProcessor`、`PersistenceAnnotationBeanPostProcessor`、`RequiredAnnotationBeanPostProcessor`——「可谓一举四得」。
4. 开启扫描：
```xml
<context:component-scan base-package="org.spring21"/>
```
`base-package` 可用逗号分隔多个包；类上写 `@Component`，要改名写 `@Component("djNewsListener")`。
5. 定制扫描范围：
```xml
<context:component-scan base-package="org.spring21">
  <context:include-filter type="annotation" expression="cn.spring21.annotation.FXService"/>
  <context:exclude-filter type="aspectj" expression=".."/>
</context:component-scan>
```
`type` 四选一：`annotation`、`assignable`、`regex`、`aspectj`；命名规则还可用 `name-generator` 换掉默认的 `BeanNameGenerator`。
6. 6.2 展望（M2 已见雏形）：`FactoryBean<T>`、`ApplicationListener<E extends ApplicationEvent>` 已泛型化；`@FactoryMethod` 的猜想实现有两条路——扫描期顺手检查该类并写入 `BeanDefinition`，或新增 `BeanFactoryPostProcessor` 统一处理（前提是 `BeanFactoryPostProcessor` 接口开放更多接口）。

#### 作者强调的坑

1. **「两片国土」**：依赖信息一半在 Java 源码（注解）、一半在 XML，作者直言「你心里一定觉得很不爽。实际上，我也是，这不是折腾人吗？」——这正是 6.1.3 的动机。
2. **容器类型限制**：这套玩法「需要我们使用 ApplicationContext 类型的容器，否则还得做点儿多余的准备工作」（BeanFactory 得手工注册 BPP）。
3. **同类型多实例**：@Autowired 撞上 `DowJonesNewsListener` 与 `ReutersNewsListener` 两个实现就懵，必须 @Qualifier 点名。
4. **两派别混着用**：`@Autowired+@Qualifier` 与 `@Resource` 是两个派系，「使用一个派别的注解就可以了」；`<context:annotation-config>` 虽两边都支持，混用「只要别造成使用上的混乱就行」。
5. **component-scan 会「多管闲事」**：它默认 `annotation-config="true"`，连带把两个 BPP 注册了，所以没写 `<context:annotation-config>` 也能注入；不想要就改成 `false`，但「我想没有太好的理由非要这么做吧？」
6. **默认命名规则的边角**：`DowJonesNewsPersister` → `dowJonesNewsPersister`（首字母小写），但对开头字母都大写的类名（`FXNewsProvider`），beanName 实际就等于类名，得 `getBean("FXNewsProvider")` 才取得到——「这与默认的命名生成规则有些小小的差异」。
7. **完全注解化走不通**：第三方类库没法加 `@Component`，「如果非要使用完全基于注解的依赖注入的话，或许会遇到一些过不去的坎儿」，该退回 XML 就退回。

#### 本章结论

Spring 最初并不支持注解注入，2.5 引入时必须在「维护整个框架设计与实现的一致性」和「新方式对整个框架的冲击」之间权衡；结果是从整体上保持了一致——注解只是 `BeanDefinition` 与 `BeanPostProcessor` 之上的另一层表达，没有第二套内核。作者的判断很务实：仍会部分依赖 XML，但「通过 20% 的工作却可以带来 80% 的效果，这本身已经是最好的结果了」；而基于 XML 的依赖注入依旧是「Spring 提供的最基本、也最为强大的表达方式」。


---
