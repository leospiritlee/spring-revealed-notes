# 第六部分　Spring 的 Web MVC 框架

---

### 第22章 迈向 Spring MVC 的旅程

#### 一句话主旨

Web MVC 框架的出现，本质上是为了把「URL 到处理类的映射」和「请求处理流程」这两块每个应用都要重写一遍的通用逻辑，从业务代码里剥离出来，做成可复用、可配置的基础设施。

#### 从 Servlet 到 Model 2 的演进

**Servlet 独行天下。** Servlet 的第一个毛病是「神奇 Servlet（Magic Servlet）」：流程控制、视图渲染、业务逻辑、数据访问全部塞在 `doPost` 里，中间还夹着一堆 `out.println("<tr><td>...")`。作者给出的重构路径很朴素——把数据访问抽到 DAO、业务抽到 Service，Servlet 里只剩「取参数 → 调服务 → 输出 HTML」。但 `out.println` 无法根治：视图逻辑写死在 Java 里，改一个表格就得改代码重新编译；页面一膨胀，没人能「纵观全局」；而且视图本来就该由前台人员维护。

**JSP 时代。** 用模板化（templating）把视图搬到 `.jsp`，Servlet 里只剩 `request.setAttribute(...)` + `forward("view.jsp")`，这是本书反复回看的那段「理想形态」。但 JSP 最终会被编译成 Servlet，这个「能量」让它迅速越界：Scriptlet 里可以写任意 Java，包括 JDBC 连接。于是「神奇 Servlet」变成了「神奇 JSP」，连 web.xml 里的 URL 映射都省了——直接拿 JSP 当控制器用，流程控制散落在各个页面文件里，比原来更难维护。

**Model 1 与 Model 2。** Model 1 的贡献只是引入 JavaBean 封装业务逻辑，JSP 依然攥着流程控制不放。Model 2 才是真正的分水岭：Servlet 当控制器管流程、JSP 只负责渲染、JavaBean 封装业务并访问数据层。这已经是 Web MVC 的雏形。

顺带澄清一个概念差异：原初的 MVC 里，模型状态更新后是通过事件机制 **Push** 给视图的；Web 应用受 HTTP 请求/响应模型限制做不到这一点，只能改成由控制器从模型 **Pull** 出数据，再连同视图一起转发——这就是 Web MVC，也是后来所有请求驱动框架的共同形态。

**Model 2 剩下的问题，正是框架要解决的。** 架构图只说了「Servlet 是控制器」，没说是一个还是多个：

- **多个 Servlet 当控制器**：一个请求一个 Servlet，全靠 web.xml 的 url-pattern 匹配，配置随规模线性膨胀，流程分散、无法集中管理。
- **单个 Servlet 当集中控制器**：web.xml 不膨胀了，但膨胀转移到了 Servlet 类里——它得自己解析 URI 判断走向（用不上容器的映射能力），流程分支硬编码，改一条 URL 规则就要改代码重编译，而且这套逻辑到下个应用完全无法复用。

各种 Web MVC 框架的答案是一致的：引入 **Front Controller**（唯一入口 Servlet）+ **Page Controller**（次级控制器），把映射规则**外部化**到配置文件。Struts 是标准样本：`ActionServlet` 是 Front Controller，`Action` 是次级控制器，`struts-config.xml` 存映射。Spring MVC 沿用同一骨架，只是把每个关注点都拆成了独立可替换的角色。

### 第23章 Spring MVC 初体验

#### 一句话主旨

`DispatcherServlet` 干的活和当年的 Magic Servlet 没有本质区别——取请求信息、调处理类、选视图输出；区别只在于它把这三件事细化并委派给了五个各司其职的角色。

#### 一个请求的完整旅程

`DispatcherServlet`（`org.springframework.web.servlet.DispatcherServlet`）注册为 `*.do`，注定要服务一整组请求，因此**必须自己做 URL 匹配**（容器只帮它把 `*.do` 都丢过来）。而匹配策略可能千变万化——掐头去尾取路径、取某个参数值、看 cookie/session、甚至 Rails 式约定——写死在 Servlet 里就废了。于是有了第一个角色。

1. **DispatcherServlet（Front Controller）**：接住请求，开始编排。
2. **HandlerMapping（协调人）**：`getHandler(request)`，负责回答「这个 URL 该由谁处理」。它返回的其实是一个 `HandlerExecutionChain`（内含 Handler + 一组 HandlerInterceptor），第23章为了降低理解成本先简化为「返回一个 Controller」。**返回：Handler。**
3. **HandlerAdapter（适配器，第25章才正式登场）**：Handler 可能是任意类型，DispatcherServlet 不能 `if-else` 枚举，需要适配器用统一方式调用它。**返回：ModelAndView。**
4. **Controller（Page Controller）**：`handleRequest(request, response)`，真正干活——取参数、调 Service、组装结果。**返回：ModelAndView。**
5. **ModelAndView**：一个纯数据对象，装两样东西——视图信息（逻辑视图名，或直接的 View 实例）+ 模型数据（`ModelMap`）。它的价值是切断「请求处理」与「视图渲染」的编译期耦合。
6. **ViewResolver（视图定位器）**：`resolveViewName(viewName, locale)`，把逻辑视图名翻译成具体的 View 实例。**返回：View。**
7. **View**：`render(model, request, response)`，把模板和模型数据合并，通过 `HttpServletResponse` 的 `getWriter()`（文本，如 JSP/Velocity）或 `getOutputStream()`（二进制，如 PDF/Excel）输出。
8. **响应**回到客户端，`DispatcherServlet` 的处理流程结束。

为什么要 View 这层抽象？因为可用视图技术远不止 JSP，Velocity/FreeMarker 根本不依赖 `request` 传数据，也不走 `RequestDispatcher`。但只要把「取模型 → 取模板 → 用对应技术 API 合并 → 通过 response 输出」这四步封装进 View 实现，DispatcherServlet 就只需要面对 `View` 接口，无需关心具体技术。

**物理结构**（这部分是实践章节里最值得记住的）：一个 Spring MVC 应用仍是标准 Servlet 应用，只是多两层容器。

- `web.xml` 里 `ContextLoaderListener` 加载 **ROOT WebApplicationContext**，默认读 `/WEB-INF/applicationContext.xml`，放 DataSource、DAO、Service 等中间层对象。可用 `contextConfigLocation` 参数改成多文件（逗号/空格分隔，支持 ANT 路径）。
- `DispatcherServlet` 加载 `<servlet-name>-servlet.xml`（如 `controller-servlet.xml`），构建**子容器**，放 HandlerMapping、Controller、ViewResolver 等 Web 层组件。子容器以 ROOT 为父容器，所以 Web 层可以直接注入 Service。
- 子容器同样可用 `contextConfigLocation` 初始化参数拆分多文件。

两个默认值要记住：没有配置任何 HandlerMapping 时默认用 `BeanNameUrlHandlerMapping`；没有配置任何 ViewResolver 时默认用 `InternalResourceViewResolver`。

### 第24章 主要角色

#### HandlerMapping

接口只有一个方法：

```java
HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception;
```

职责边界很清晰：**只负责「找」，不负责「调」**。名字叫 HandlerMapping 而不是 ControllerMapping，是因为 Spring MVC 里的次级控制器统称 Handler，不限于 `Controller`（还可以是 Struts 的 Action、带注解的 POJO）。

常见实现：

| 实现类 | 映射依据 |
|---|---|
| `BeanNameUrlHandlerMapping` | bean 名称 == 请求路径，如 `<bean name="/ttmRateList.do">` |
| `SimpleUrlHandlerMapping` | `mappings` 属性显式配置，支持 `/**/*list.do`、`/user*.do` 等 ANT 模式 |
| `ControllerClassNameHandlerMapping` | 类名约定（第27章 CoC） |
| `DefaultAnnotationHandlerMapping` | 类/方法上的 `@RequestMapping`（第26章） |

`BeanNameUrlHandlerMapping` 只是「传话人」，强制视图里的链接路径与 Handler 的 beanName 一致，表现层和 Web 层分属不同人时很别扭；`SimpleUrlHandlerMapping` 让两边各自自由，由它居中「统筹」，还能用通配符把一组请求映射给同一个 Handler。

**多个 HandlerMapping 时的查找顺序**：全部实现 `Ordered` 接口，按 `order` 升序询问；`order` 未指定默认 `Integer.MAX_VALUE`（最低优先级）。**第一个返回可用 Handler 的就胜出，后面的不再询问**——所以顺序即优先级，需要兜底的那个放最后。

#### Controller 家族

接口同样只有一个方法：`ModelAndView handleRequest(request, response)`。但直接实现接口意味着请求参数抽取、编码、国际化、Session 管理全要自己来，于是有了继承体系，分两派。

**自由挥洒派**（熟悉 Servlet API 就很亲切，但细节要自己操心）：

- `AbstractController`：整个体系的起源，用模板方法模式处理三个通用关注点——支持的请求方法（GET/POST）、页面缓存设置、执行流程在 Session 上的同步。子类只实现 `handleRequestInternal(request, response)`。适合简单请求。
- `MultiActionController`：一个类管一组逻辑相近的请求（典型是同一实体的 CRUD），不必一个请求一个类。处理方法签名 `methodName(request, response[, session | command])`，返回值四种语义：`ModelAndView` 正常返回；`Map` 只有模型数据，逻辑视图名交给 `RequestToViewNameTranslator` 按默认规则推；`void` 表示方法自己完成了渲染输出；`String` 表示只有逻辑视图名。哪个请求进哪个方法，由 `MethodNameResolver` 决定——默认 `InternalPathMethodNameResolver`（取 URL 最后一个 `/` 之后、去掉扩展名，可加 prefix/suffix）、`PropertiesMethodNameResolver`（显式配映射，支持 ANT）、`ParameterMethodNameResolver`（按 `action=xxx` 参数值，或按 `methodParamNames` 检测一组参数名）。

**规范操作派**（把通用逻辑规范化，代价是要遵守「制度」）：

- `BaseCommandController`：提供两样能力——请求参数自动绑定到 command 对象、统一的 Validator 验证。command 类似 Struts 的 FormBean，但不强制类型，最好是 JavaBean。绑定靠 `ServletRequestDataBinder`（内部是 `BeanWrapperImpl` + PropertyEditor 做类型转换），参数名对应属性名，嵌套集合用 `phoneNumbers[0].number`、`map['key'].number` 这类表达式（注意中间对象不能为 null，否则 NPE）。验证靠 `Validator.supports()` 界定范围、`validate(target, errors)` 干活，错误分 Global Error（`errors.reject`）与 Field Error（`errors.rejectValue`）；嵌套对象验证前后要 `pushNestedPath` / `popNestedPath`。
- `AbstractFormController` → `SimpleFormController`：面向单表单。核心是 `isFormSubmission(request)`（默认判断是否 POST）把流程切成两段。**显示阶段**：`formBackingObject()` 造 command（更新场景要覆写它，把库里数据加载进来）→ `initBinder()` 定制 DataBinder → 若 `bindOnNewForm=true` 则先绑定 → `referenceData()` 提供下拉列表之类的辅助数据（与 command 里存的「选中值」是两回事）→ 返回以 `formView` 为视图名的 ModelAndView。**提交阶段**：取/重建 command → 绑定（扩展点 `onBind`）→ 调 Validator（扩展点 `onBindAndValidate`）→ 有错误就回表单页 → 若 `isFormChangeRequest` 为真（只是页面变动不是最终提交）也回表单页 → 否则 `doSubmitAction(command)` 或覆写 `onSubmit` → 转向 `successView`。
- `AbstractWizardFormController`：多页向导。所有页面共用一个 command，靠特殊参数驱动：`_target0/_target1...` 指定跳到第几页（同时把本页数据绑到 command）、`_finish` 结束并调 `processFinish()`、`_cancel` 取消并调 `processCancel()`。必须实现的只有 `processFinish`。

其他：`AbstractCommandController`（加强版 AbstractController，绑定验证后只写 `handle(...)`）、`ParameterizableViewController`（无逻辑，直接返回指定视图，把静态页面纳入统一体系）、`UrlFilenameViewController`（一组静态页面，如帮助文档）、`ServletForwardingController` / `ServletWrappingController`（把已有 Servlet 纳入 MVC）。

> ⚠️ 已过时：以上 `AbstractController`、`MultiActionController`、`SimpleFormController`、`AbstractWizardFormController`、`AbstractCommandController` 全部是 XML 配置时代的产物。Spring 3.0 起被标为 `@Deprecated`，Spring 4 已彻底移除，取而代之的是 `@Controller` + `@RequestMapping`（第26章）。今天读它们的主要价值是理解「一个请求处理流程里究竟有哪些可复用的关注点」——数据绑定、验证、表单两段式流程、向导分页，这些关注点在注解版里换了形式但一个都没少。

#### ModelAndView

一个数据对象，两部分：视图信息 + 模型数据。

- 视图信息可以是**逻辑视图名**（`new ModelAndView("ttmRateList")`）或**View 实例**。推荐前者：交给 ViewResolver 解析才能保留「换视图技术只改配置」的灵活性。若直接给 View 实例，DispatcherServlet 跳过 ViewResolver 直接 `render`。
- 模型数据以 `ModelMap` 保存，通过 `addObject(key, value)` 添加，key 要与视图模板里的标识符对应。JSP/JSTL 视图把模型数据塞进 `request` 属性；Velocity 视图则复制进自己的 `Context`。
- `handleRequest` 返回 `null` 是合法的，表示 Controller 自己完成了渲染。

#### ViewResolver

```java
View resolveViewName(String viewName, Locale locale) throws Exception;
```

带 `Locale` 参数是为了让同一个逻辑视图名能按语言返回不同 View（国际化视图）。绝大多数实现继承 `AbstractCachingViewResolver`，默认缓存 View 实例（生产环境合理，开发时可 `setCache(false)` 关掉）。

**面向单一视图类型**（都继承 `UrlBasedViewResolver`，不需要配映射，一比一对应一种 View）：`InternalResourceViewResolver`（JSP，也是默认兜底）、`FreeMarkerViewResolver`、`VelocityViewResolver`、`JasperReportsViewResolver`、`XsltViewResolver`。它们的核心配置就是 **prefix + suffix**：逻辑视图名为 `ttmRateList`、prefix 为 `/WEB-INF/jsp/`、suffix 为 `.jsp`，则实际模板是 `/WEB-INF/jsp/ttmRateList.jsp`。作用是把「逻辑名」与「物理路径」解耦——Controller 只认逻辑名，模板放哪儿、用什么后缀由配置决定。

**面向多视图类型**（需要显式配置「逻辑名 → View 定义」的映射，可混合多种视图技术）：`ResourceBundleViewResolver`（properties 文件，`views.properties`，是唯一支持视图国际化的）、`XmlViewResolver`（`/WEB-INF/views.xml`，Spring XML 格式，不支持国际化）、`BeanNameViewResolver`（View 直接注册成 bean，适合原型或小应用）。

**多个 ViewResolver 时的查找顺序**：同样按 `Ordered` 排序，逐个询问，**第一个返回非 null 的就停**。这里有个必须记住的坑：`InternalResourceViewResolver` 这类 `UrlBasedViewResolver` 子类即使找不到模板也**不会返回 null**，所以绝不能给它高优先级，否则后面的 Resolver 永远轮不到。正确做法是把 `ResourceBundleViewResolver` / `XmlViewResolver` 这种「找不到就返回 null」的放前面，`InternalResourceViewResolver` 放最低优先级当兜底。

### 第25章 更多家族成员

#### MultipartResolver（文件上传）

位置在 HandlerMapping **之前**。`application/x-www-form-urlencoded` 支撑不了文件上传，RFC 1867 加了 `multipart/form-data`；解析这堆字节流的活交给 Commons FileUpload 或 COS 之类的库，Spring 只做一层策略抽象：

```java
boolean isMultipart(HttpServletRequest request);
MultipartHttpServletRequest resolveMultipart(HttpServletRequest request);
```

流程：DispatcherServlet 按 **bean 名称 `multipartResolver`** 找实现 → `isMultipart` 判断 → 是则 `resolveMultipart` 返回 `MultipartHttpServletRequest`，**用装饰器模式把原 request「偷梁换柱」**，后续所有环节拿到的都是它（所以能 `getFile("fileToUpload")` 拿到 `MultipartFile`）；不是则原样返回。处理结束后 DispatcherServlet 保证调用 `cleanupMultipart` 释放临时资源。实现类：`CommonsMultipartResolver`、`CosMultipartResolver`。因为 request 被换了，`<input type="file">` 在 Controller 里和普通文本域一样好处理，甚至可以直接声明成 command 对象的 `MultipartFile` 属性交给 `SimpleFormController` 绑定。

#### HandlerAdapter（为什么需要它——Handler 可以是任意类型）

第24章说 HandlerMapping「返回一个 Controller」是简化说法。真实情况：`HandlerExecutionChain` 里装的是 `Object` 类型的 Handler，Spring MVC 中**任何能处理 Web 请求的对象都叫 Handler**，Controller 只是其中一种。于是 DispatcherServlet 面临一个它无法回答的问题：这个 Object 该调哪个方法？

硬编码 `if-else` 枚举所有 Handler 类型毫无扩展性，所以把「调用」这件事外包：

```java
boolean supports(Object handler);
ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler);
long getLastModified(HttpServletRequest request, Object handler);
```

DispatcherServlet 拿到 Handler 后遍历所有 HandlerAdapter，找 `supports` 为 true 的那个，调 `handle`，拿回统一的 ModelAndView。`getLastModified` 只是给 HTTP `Last-Modified` 头提供值，不支持就返回 -1。

关键结论：**要支持一种新 Handler 类型，只需要新增一个 HandlerAdapter，DispatcherServlet 一行都不用改**。这也是「Handler 自身可以没有任何限制」（甚至是个裸 POJO）的真正原因——限制被 HandlerMapping（负责找得到）和 HandlerAdapter（负责调得动）分担了。

对应关系一目了然：`Controller` → `SimpleControllerHandlerAdapter`；`ThrowawayController` → `ThrowawayControllerHandlerAdapter`；注解 Handler → `AnnotationMethodHandlerAdapter`。默认后备还有 `HttpRequestHandlerAdapter`。注册方式是把 Adapter 放进 DispatcherServlet 的 WebApplicationContext，按类型自动检测；一旦自定义了 Adapter，就要把默认那几个也一起显式声明，否则原有 Handler 类型会失效。

> ⚠️ 已过时：`ThrowawayController`（无状态依赖、`execute()` 无参、靠 prototype scope 每次新建）在 Spring 2.5 已 deprecated，Spring 3.0 移除。它的设计意图——Handler 不依赖 Servlet API、可测试性好——后来由注解 Controller 更好地实现了。

#### HandlerInterceptor（三个回调时机）

`HandlerExecutionChain` 的另一半就是这个拦截器数组，它解释了 HandlerMapping 为什么返回 Chain 而不是裸 Handler。

| 方法 | 时机 | 能力 |
|---|---|---|
| `boolean preHandle(req, res, handler)` | HandlerAdapter 调用 Handler **之前** | 返回 false 可**阻断**后续（包括后续拦截器和 Handler）；这是唯一能阻断的时机，适合做权限、状态检查 |
| `void postHandle(req, res, handler, mav)` | Handler 执行完、**视图渲染之前** | 能拿到 ModelAndView 做后处理（补公共模型数据、改视图名）；void，不能阻断 |
| `void afterCompletion(req, res, handler, ex)` | 整个流程结束、**视图渲染完之后** | 无论是否异常都会执行；可拿 `ex` 做统一记录，可做资源清理；void，此时已无需阻断 |

注册入口在 HandlerMapping 上（`AbstractHandlerMapping.setInterceptors(...)`），拦截器随 Handler 一起被打包进 Chain 返回。**因为可以给不同 HandlerMapping 配不同拦截器，而 HandlerMapping 又可以按 order 分管不同请求，所以拦截范围可以做到很细**。

现成实现：`UserRoleAuthorizationInterceptor`（`isUserInRole` 校验，失败默认返回 403，可覆写 `handleNotAuthorized`）、`WebContentInterceptor`（校验请求方法、`requiresSession`、用 `cacheSeconds` 设置缓存相关 HTTP 头）；另有 `LocaleChangeInterceptor`、`ThemeChangeInterceptor`（见下）。自定义时通常继承 `HandlerInterceptorAdapter` 而非直接实现接口，避免实现全部三个方法（与 AWT 的 `MouseAdapter` 同一套路）。**HandlerInterceptor 之外的选择**是 Servlet 规范的 `Filter`——毕竟 Spring MVC 仍构建在 Servlet API 之上。

#### HandlerExceptionResolver（异常怎么变成错误页）

Handler 接口签名是 `throws Exception`，看似违反「精确声明受检异常」的最佳实践，但框架无法预知各应用 Handler 会抛什么业务异常，只能放开。兜底的就是这个角色：

```java
ModelAndView resolveException(HttpServletRequest request, HttpServletResponse response,
                              Object handler, Exception ex);
```

作用范围**仅限 Handler 查找与执行期间**。它和 Handler 是「双子座」：Handler 正常返回 ModelAndView，异常则由它返回一个「错误页逻辑视图名 + 异常信息」的 ModelAndView，**后续流程与正常路径完全合流**——照样交给 ViewResolver 和 View 渲染。框架只提供了一个实现 `SimpleMappingExceptionResolver`，但基本够用：

- `exceptionMappings`（Properties）：异常类名 → 错误页逻辑视图名
- `defaultErrorView`：匹配不上时的兜底视图
- `defaultStatusCode`：默认 HTTP 状态码（如 500 / 404）
- `exceptionAttribute`：暴露给错误页的异常对象的 key，默认 `exception`，设为 null 则不暴露
- `mappedHandlers` / `mappedHandlerClasses`：限定只处理哪些 Handler 的异常；不设则处理全部
- `order`：可配多个 Resolver，按优先级询问

> ⚠️ 陷阱：`SimpleMappingExceptionResolver` 匹配异常类型时用的是**类名字符串的 `indexOf` 局部匹配**，不是类型匹配。若同时配了 `Exception` 和 `TradeRateExpireException`，实际抛 `a.b.c.exceptions.TradeRateExpireException` 时可能先命中 `Exception` 对应的页面。规避办法：映射键一律写全限定类名。

#### LocaleResolver（国际化）

ViewResolver 的 `resolveViewName(viewName, locale)` 里的 `locale` 从哪来？由 LocaleResolver 提供：

```java
Locale resolveLocale(HttpServletRequest request);
void setLocale(HttpServletRequest request, HttpServletResponse response, Locale locale);
```

实现（都在 `org.springframework.web.servlet.i18n`）：`FixedLocaleResolver`（固定值，也是**默认**）、`AcceptHeaderLocaleResolver`（读 `Accept-Language` 头）、`SessionLocaleResolver`（存 Session，可改）、`CookieLocaleResolver`（存 Cookie，可改）。前两个无法变更 Locale，所以要支持「点链接切语言」只能用后两个。**bean 名称必须是 `localeResolver`**。

它的「足迹」串起整条国际化链路：请求进入前，DispatcherServlet 把 Locale 以 `LocaleContext` 形式绑定到当前线程 → ViewResolver 用它做视图查找 → 视图内部通过 `RequestContext`（JSP 下即 `<spring:message>` 标签）取消息 → 最后清理线程绑定。切换语言靠 `LocaleChangeInterceptor`：读某个请求参数（默认 `locale`）的值，调 `LocaleResolver.setLocale` 改掉，配置在 HandlerMapping 的 interceptors 里。

#### ThemeResolver（主题）

主题就是换一套静态资源（背景图、CSS），让页面整体风格变化。三个角色：

- **ThemeSource**：`Theme getTheme(String themeName)`。`WebApplicationContext` 自身实现了该接口但不干活，一律委派给名为 **`themeSource`** 的 bean。常用 `ResourceBundleThemeSource`，用 properties 定义每个主题的资源（`theme.background.image=...`），因为基于 ResourceBundle，所以**同一主题还能按 Locale 提供不同资源**（`blue.properties` / `blue_zh_CN.properties`）。
- **ThemeResolver**：`resolveThemeName(request)` / `setThemeName(request, response, themeName)`。实现与 LocaleResolver 三缺一：`FixedThemeResolver`（默认）、`SessionThemeResolver`、`CookieThemeResolver`——没有 Accept-Header 版，因为 HTTP 头里没有「主题」这种东西。**bean 名称必须是 `themeResolver`**。
- **ThemeChangeInterceptor**：读参数（默认 `theme`）切换主题。

视图里用 `<spring:theme code="theme.background.image"/>` 取资源；非 JSP 视图则通过公开的 `RequestContext`（`rc.getThemeMessage(...)`）访问。

### 第26章 基于注解的 Controller

#### 一句话主旨

注解 Controller 没有任何魔法——它依然是一个 Handler，只是把「识别 Handler」的契约从 **Java 接口**换成了 **注解 + 反射**，因此必须配一套新的 HandlerMapping 和 HandlerAdapter。

#### 注解 Controller 的本质

传统 Handler 靠类型做标志：`instanceof Controller` 就知道该调 `handleRequest`，`instanceof ThrowawayController` 就知道该调 `execute`。注解 Controller 是个纯 POJO，没有接口、没有父类、不依赖 Servlet API，那框架怎么知道它是 Handler、又该调哪个方法？

作者用「回到 Spring 2.5 之前，自己实现一个」的方式回答，把问题拆成两问：**谁来找**（HandlerMapping）、**谁来调**（HandlerAdapter）。这正好复用了第25章「自定义 Handler 需要左右臂」的结论。

**自定义 HandlerMapping**：遍历所有候选对象，用反射读类上的 `@RequestMapping`，与请求路径比对，命中就包成 `HandlerExecutionChain` 返回。官方版是 `DefaultAnnotationHandlerMapping`，它先扫描 classpath 拿到所有标了 `@Controller` 的对象——这一步正是靠 `<context:component-scan/>` 完成的，也是 `@Controller` 成为必需注解的原因（component-scan 内置支持它，换成别的注解就得自己写扫描逻辑）。

**自定义 HandlerAdapter**：`supports` 判断类上有 `@Controller`/`@RequestMapping`；`handle` 用反射找到标了 `@RequestMapping` 的方法，把请求参数绑定到方法参数，`method.invoke`，再把返回值转成 ModelAndView。官方版是 `AnnotationMethodHandlerAdapter`。

真正的工作量在「绑定」和「返回值归一化」上，原型实现绕过了这些坑，而生产级实现必须处理：

- `@RequestMapping(method=POST)` 的语义校验（GET 请求不能进这个方法）
- **方法参数名的获取**——反射 API 拿不到参数名，请求参数就无从对应；Spring 2.5 借助 ObjectWeb ASM 读字节码解决
- 方法参数上的注解（`@RequestParam` 等）带来的分支逻辑
- 无 Servlet API 依赖时如何提供 Session 支持
- 返回值类型的枚举与 ModelAndView 转换

两个官方实现都在 DispatcherServlet 初始化时**默认注册**，多数情况下无需显式声明。

> ⚠️ 已过时（名称层面）：Spring 3.1 起，这两个类被 `RequestMappingHandlerMapping` / `RequestMappingHandlerAdapter` 取代（后者基于 `@RequestMapping` 元数据统一处理，并支持 `@PathVariable`、内容协商等）。`DefaultAnnotationHandlerMapping` / `AnnotationMethodHandlerAdapter` 在 3.1 后 deprecated，4.x 移除。

#### 常用注解

- **`@Controller`**：身份标志。两个作用——被 `<context:component-scan/>` 纳入 IoC 容器（从而获得 DI、AOP 等「名门」待遇）；被 `DefaultAnnotationHandlerMapping` 列入候选。
- **`@RequestMapping`**：双重身份。标在**类**上是映射前缀（`"/reportSetting.anno"`），标在**方法**上是「这是请求处理方法」+ 可选限定条件。方法级可用 `method`（GET/POST/...）和 `params`（`"locale=en"` 按值区分、`"delete"` 按参数是否存在区分）细化分流——效果等价于 `MultiActionController` + `ParameterMethodNameResolver`。两种典型组合：类级给前缀 + 方法级按 method 区分（等价 `SimpleFormController` 的显示/提交两段）；或全部标在方法级、每个方法自带完整路径（等价 `MultiActionController`）。
- **`@RequestParam`**：显式指定「请求参数名 → 方法参数」的对应关系，用于摆脱命名必须一致的约束。`required` 默认 `true`，参数缺失会抛异常；设 `false` 则给该类型的默认值。
- **`@PathVariable`**：绑定 URI Template 中占位符的值，如 `@RequestMapping("/hotels/{hotel}/bookings/{booking}")` 配 `@PathVariable("hotel") long hotelId`。**注意：本书成书于 Spring 2.5，此注解尚不存在，作者是在第27章作为 Spring 3.0 展望介绍的**，Spring 3.0 正式提供。
- **`@ModelAttribute`**：标在**方法**上 = 只「存」——该方法返回值自动进模型，等价 `SimpleFormController.referenceData()`；标在**参数**上 = 「取」——从模型里取出对象注入方法参数，同时该参数还兼作 command 对象参与数据绑定。两者合用就构成「先准备、再绑定、再处理」的完整闭环。
- **`@SessionAttributes`**：只能标在**类**上，按属性名（`@SessionAttributes("command")`）或类型（`types=ReportSettings.class`）把模型数据提升到 Session。它解决的是一个很实际的问题：GET 请求放进模型的 command 对象，与 POST 请求从模型里取出的 command 对象**是两个不同请求里的两个对象**，要跨请求复用同一个，只能靠 Session。配合 `SessionStatus.setComplete()` 在用完后清理。
- 配套：**`@InitBinder`**（定制 DataBinder，如注册自定义 PropertyEditor；无返回值、至少要有一个 `WebDataBinder` 参数）；`Errors` / `BindingResult`（拿绑定与验证结果，**声明必须紧跟在 command 参数之后**，其他参数无顺序要求）；`SessionStatus`（管理 Session 状态）。

#### 请求参数怎么绑定到方法参数

分两条路：

**一、特殊类型由框架注入**（不参与「参数名匹配」，框架直接给现成对象）：`HttpServletRequest/Response`、`HttpSession`、`WebRequest`、`Locale`（来自 LocaleResolver）、`InputStream/Reader`、`OutputStream/Writer`、`Map`/`ModelMap`（直接操作模型数据）、`Errors`/`BindingResult`、`SessionStatus`。原理很朴素——框架在调用方法前把准备好的对象塞进参数数组，调用后再把 `ModelMap` 里的内容并进 ModelAndView。

**二、普通类型按名称匹配绑定**：请求参数名 == 方法参数名，值自动做类型转换后注入（`?age=30&author=Darren` → `processMethod(int age, String author)`）。也可以声明 JavaBean 参数，按「参数名 == Bean 属性名」绑定（`bookName` → `book.bookName`）——这就相当于声明了一个 command 对象，`SimpleFormController` 的能力用一行方法签名拿到了。

几个必须知道的边界：

- 名字对不上时，引用类型绑成 `null`，**原始类型（`int` 等）无法接受 null，会在转换时抛 `TypeMismatchException`**。
- 想改命名规则用 `@RequestParam`；想改类型转换规则用 `@InitBinder`（单个 Controller 内）或给 `AnnotationMethodHandlerAdapter` 配 `WebBindingInitializer`（多个 Controller 共享，避免重复写 `@InitBinder`）。
- 返回值只允许四种：`ModelAndView`（视图 + 模型）、`String`（逻辑视图名）、`ModelMap`（只有模型，视图名按请求路径推）、`void`（都从请求路径推）。限制返回值类型是为了让 Adapter 的实现可控——否则就得无限枚举。

### 第27章 扩展篇

#### Convention Over Configuration 在 Spring MVC 里体现为什么

CoC（惯例优于配置）的主张是：能用约定推出的映射关系就不要写配置。它与外部配置并不冲突，只是约定优先、配置兜底。Spring 2.0 起在 Spring MVC 里落地为三个约定：

**1. 请求 → Handler 的约定：`ControllerClassNameHandlerMapping`。** 去掉 `Controller` 后缀的类名小写即路径：`SampleController` → `/sample*`；对 `MultiActionController` 则用前缀映射 `/sample/*`（配合 `InternalPathMethodNameResolver` 就得到 Rails 式的 `/sample/list.do` → `list()`）。可定制：`pathPrefix`（加路径前缀）、`basePackage`（按包名形成 `group/sample.do` 这样的路径，适合按模块分包）、`excludedClasses` / `excludedPackages`。它随 DispatcherServlet 默认启用（与 `BeanNameUrlHandlerMapping` 一起），所以只要按约定命名并注册 bean，映射配置全省了。

**2. 模型数据的约定：`ModelAndView.addObject(obj)` 免 key。** 单个对象取类名首字母小写（`Customer` → `customer`）；集合或数组取元素类型小写加 `List`（`Customer[]` → `customerList`）。

**3. 请求 → 视图的约定：`RequestToViewNameTranslator`。** 当 ModelAndView 里没有逻辑视图名时，ViewResolver 无能为力——它接口里根本没有 `HttpServletRequest`，无法从 URL 猜视图名。所以引入这个角色：`getViewName(request)`，默认实现 `DefaultRequestToViewNameTranslator`，规则与 `ControllerClassNameHandlerMapping` 对称（`/sample.do` → `sample`，`/module/sample.do` → `module/sample`），可用 prefix/suffix 修饰。三者串起来，一个 `SampleController` + 一个 `sample.jsp` 就能跑通，全程零映射配置。

#### 作者对 Spring 3.0 的展望

作者认为 Spring 3.0 对 Spring MVC 的改动幅度会是全书最大的一块，并给出四条判断：

1. **`@MVC` 成为主流**。2.5 之后注解 Controller 已是最推荐的实践，3.0 会进一步强化，「注解驱动的 Spring MVC 开发将红遍整个天」。
2. **REST 支持进入核心**。这是重头戏，会大幅降低用 Spring MVC 写 REST 应用的门槛，配套装备包括：`ContentNegotiatingViewResolver`（按请求的内容类型协商返回哪种视图）、**URI Template + `@PathVariable`**（`/hotels/{hotel}/bookings/{booking}` 直接把路径片段绑到方法参数）、新的 View 实现（`AbstractAtomFeedView`、`AbstractRssFeedView`、`MarshallingView`、`JacksonJsonView`）、`HiddenHttpMethodFilter`（HTML 表单只能发 GET/POST，用它把 `_method` 参数翻译成 PUT/DELETE，对后端透明）。
3. **可能正式支持 JSF 2.0**。尚未定局，取决于是否会对现有架构造成大改动。
4. **Spring Portal MVC 升级**支持 Portlet 2.0，同样走向注解化。

回看这四条，第 1、2 条基本全部兑现（`@MVC` 与 REST 支持确实是 Spring 3.0 的主线），第 3 条最终没有成为 Spring MVC 的核心方向，第 4 条则随着 Portlet 技术本身的式微而淡出——这恰好也说明了本书的定位：它记录的是一个框架在「XML 配置时代」向「注解时代」转折的那个瞬间，而书中那些被拆得极细的角色（HandlerMapping / HandlerAdapter / ViewResolver / HandlerInterceptor / HandlerExceptionResolver / LocaleResolver / ThemeResolver），至今仍是理解 Spring MVC 内部结构最有效的坐标系。


---
