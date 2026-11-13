# 结语：这本书在今天该怎么用

《Spring 揭秘》写于 2009 年，但它回答的问题至今没变：**一个框架凭什么让成千上万个对象自动装配起来，又凭什么能在不改业务代码的前提下横切进去加逻辑。**

这两个问题的答案，就是 Spring 的全部：**IoC 容器 + AOP**。剩下的数据访问、事务、MVC，都是这两样东西的应用。

所以读这本书，别把它当"Spring 2.5 使用手册"——那部分已经死了。把它当**"Spring 为什么长这样"的解释书**，那部分永远活着。

十七年里，Spring 的 API 换了一茬又一茬：XML 没了、`SimpleFormController` 没了、`HibernateTemplate` 不推荐了。但 `BeanDefinition` 还在，`BeanPostProcessor` 还在，`ProxyFactory` 还在，`PlatformTransactionManager` 还在。

**变的都是皮，没变的是骨。这本书讲的是骨。**
