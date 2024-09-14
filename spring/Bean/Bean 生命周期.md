# Bean 生命周期

Bean 生命周期时 Spring 最核心的 IoC、AOP 之外最重要的概念。Bean 是 Spring IoC 容器实例化、组装和管理的对象。

Spring 中 Bean 的生命周期主要针对的是 singleton 。普通的 Java 对象的生命周期只有**实例化**和**对象回收**。

由于 Spring 接管了 Bean 的生命周期，通过反射创建对象，因此具有更复杂的生命周期：

- 实例化
- 属性赋值
- 初始化
- 销毁
