# Bean 生命周期

Bean 生命周期时 Spring 最核心的 IoC、AOP 之外最重要的概念。Bean 是 Spring IoC 容器实例化、组装和管理的对象。

Spring 中 Bean 的生命周期主要针对的是 singleton 。普通的 Java 对象的生命周期只有**实例化**和**对象回收**。

由于 Spring 接管了 Bean 的生命周期，通过反射创建对象，因此具有更复杂的生命周期：

- 实例化
- 属性赋值
- 初始化
- 销毁

其中核心逻辑的源代码如下：
```java
// AbstractAutowireCapableBeanFactory
protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) throws BeanCreationException {
    BeanWrapper instanceWrapper = null;
    // 如果是单例模式，则从缓存 factoryBeanInstanceCache 获取，并将缓存清除
    if (mbd.isSingleton()) {
        instanceWrapper = (BeanWrapper)this.factoryBeanInstanceCache.remove(beanName);
    }

    if (instanceWrapper == null) {
    	// 实例化
        instanceWrapper = this.createBeanInstance(beanName, mbd, args);
    }

    // ...

    Object exposedObject = bean;

    try {
    	// 属性赋值
        this.populateBean(beanName, mbd, instanceWrapper);
        // 初始化
        exposedObject = this.initializeBean(beanName, exposedObject, mbd);
    } catch (Throwable var18) {
        // ...
    }

    // ...
}

// 销毁逻辑：ConfigurableApplicationContext#close()
```

Spring 中为 Bean 的生命周期提供了很多扩展点，因此显得非常复杂，但是最核心的还是这四个阶段。下面围绕这四个核心阶段讨论扩展点。

# 一、Bean 自身的构造方法

- 实例化：构造函数
- 属性赋值：get/set 方法
- 初始化：init-method
- 销毁：destroy-method

# 二、容器级方法

容器级方法是 `BeanPostProcessor` 的一系列接口，主要是后处理方法，独立于 Bean，注册到容器中，在 Spring 创建任何 Bean 时后处理器都会发生作用。这一部分的底层逻辑实现为：

```java
public abstract class AbstractAutowireCapableBeanFactory extends AbstractBeanFactory implements AutowireCapableBeanFactory {

    // ...

    @Nullable
    protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
        Object bean = null;
        if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
            if (!mbd.isSynthetic() && this.hasInstantiationAwareBeanPostProcessors()) {
                Class<?> targetType = this.determineTargetType(beanName, mbd);
                if (targetType != null) {
                    bean = this.applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
                    if (bean != null) {
                        bean = this.applyBeanPostProcessorsAfterInitialization(bean, beanName);
                    }
                }
            }

            mbd.beforeInstantiationResolved = bean != null;
        }

        return bean;
    }

    @Nullable
    protected Object applyBeanPostProcessorsBeforeInstantiation(Class<?> beanClass, String beanName) {
        Iterator var3 = this.getBeanPostProcessorCache().instantiationAware.iterator();

        Object result;
        do {
            if (!var3.hasNext()) {
                return null;
            }

            InstantiationAwareBeanPostProcessor bp = (InstantiationAwareBeanPostProcessor)var3.next();
            result = bp.postProcessBeforeInstantiation(beanClass, beanName);
        } while(result == null);

        return result;
    }

    @Deprecated(
        since = "6.1"
    )
    public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName) throws BeansException {
        Object result = existingBean;

        Object current;
        for(Iterator var4 = this.getBeanPostProcessors().iterator(); var4.hasNext(); result = current) {
            BeanPostProcessor processor = (BeanPostProcessor)var4.next();
            current = processor.postProcessAfterInitialization(result, beanName);
            if (current == null) {
                return result;
            }
        }

        return result;
    }
}
```

可以看到，这里是用迭代器取出 `BeanPostProcessorCacheAwareList` 的迭代器，本质上是一个 `CopyOnWriteArrayList`。