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
public abstract class AbstractAutowireCapableBeanFactory extends AbstractBeanFactory implements AutowireCapableBeanFactory
{
    //---------------------------------------------------------------------
	//  AbstractBeanFactory 模板方法的实现
	//---------------------------------------------------------------------

	/**
	 * 核心方法：创建并填充 Bean 实例，应用后处理方法等等。
	 * @see #doCreateBean
	 */
	@Override
	protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) throws BeanCreationException {
        // ...

		try {
			// 尝试让 BeanPostProcessors 返回代理而非目标对象实例
			Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
			if (bean != null) {
				return bean;
			}
		}
		catch (Throwable ex) {
			throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
					"BeanPostProcessor before instantiation of bean failed", ex);
		}

		try {
            // 实际创建 Bean 的方法
			Object beanInstance = doCreateBean(beanName, mbdToUse, args);
			if (logger.isTraceEnabled()) {
				logger.trace("Finished creating instance of bean '" + beanName + "'");
			}
			return beanInstance;
		}
		catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
			throw ex;
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
		}
	}

	/**
	 * 实际创建指定的 Bean. 预创建的过程到这里已经进行，例如 {@code postProcessBeforeInstantiation} 回调.
	 * 区分于默认 Bean 实例化、使用工厂方法和自动装配构造函数。
	 * @param beanName bean 名称
	 * @param mbd 合并的 bean 的定义
	 * @param args 构造器或者工厂方法调用的参数
	 * @return bean 的新的实例
	 * @throws BeanCreationException 如果 bean 无法被创建
	 * @see #instantiateBean
	 * @see #instantiateUsingFactoryMethod
	 * @see #autowireConstructor
	 */
	protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {

		BeanWrapper instanceWrapper = null;
        // 如果是单例模式，直接从工厂 Bean 实例的缓存中取出
		if (mbd.isSingleton()) {
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
        // 非单例模式或者缓存中没有
        // 1. 实例化
		if (instanceWrapper == null) {
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}

		Object bean = instanceWrapper.getWrappedInstance();
	
        // ...

		Object exposedObject = bean;
		try {
            // 属性注入
			populateBean(beanName, mbd, instanceWrapper);
            // 初始化 Bean
			exposedObject = initializeBean(beanName, exposedObject, mbd);
		}
		catch (Throwable ex) {
			// ...
		}

        // ...

		return exposedObject;
	}
}

// 销毁逻辑：ConfigurableApplicationContext#close()
```

Spring 中为 Bean 的生命周期提供了很多扩展点，因此显得非常复杂，但是最核心的还是这四个阶段。下面围绕这四个核心阶段讨论扩展点。

# 1 Bean 自身的构造方法

- 实例化：构造函数
- 属性赋值：get/set 方法
- 初始化：init-method
- 销毁：destroy-method

# 2 容器级方法

容器级方法是 `BeanPostProcessor` 的一系列接口，主要是后处理方法，独立于 Bean，注册到容器中，在 Spring 创建任何 Bean 时后处理器都会发生作用。

## 2.1 实例化后置处理器

### 实例化前处理

实例化后处理器作用于实例化的前后，在上面介绍 `createBean` 源代码时提到，在真正创建之前，Spring 会先尝试取得一个代理对象，如果能获取到代理对象，则直接返回。相关的方法如下：


```java
public abstract class AbstractAutowireCapableBeanFactory extends AbstractBeanFactory implements AutowireCapableBeanFactory {

    // ...

    /**
	 * 应用实例化之前的后处理器, 解析指定bean是否有实例化前的快捷方式。
	 * @param beanName the name of the bean
	 * @param mbd the bean definition for the bean
	 * @return the shortcut-determined bean instance, or {@code null} if none
	 */
    @Nullable
    protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
        Object bean = null;
        if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
            // 确保在这个位置 Bean 已经被解析
            if (!mbd.isSynthetic() && this.hasInstantiationAwareBeanPostProcessors()) {
                Class<?> targetType = this.determineTargetType(beanName, mbd);
                if (targetType != null) {
                    // 实例化前的处理操作
                    bean = this.applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
                    // 此时获取到 bean，则直接调用初始化后操作，跳过中间的步骤
                    if (bean != null) {
                        bean = this.applyBeanPostProcessorsAfterInitialization(bean, beanName);
                    }
                }
            }

            mbd.beforeInstantiationResolved = bean != null;
        }

        // 到这里，要么 bean 是空，要么已经初始化并且走了后置操作
        return bean;
    }

   /**
	 * 应用 InstantiationAwareBeanPostProcessors 到指定的 bean definition
	 * (by class and name), 调用对应的 {@code postProcessBeforeInstantiation} 方法.
     * 任何返回的对象都将被用作 bean，而不是实际实例化目标 bean。
     * 来自后处理器的{@code null}返回值将导致目标bean被实例化。
	 * @param beanClass the class of the bean to be instantiated
	 * @param beanName the name of the bean
	 * @return the bean object to use instead of a default instance of the target bean, or {@code null}
	 * @see InstantiationAwareBeanPostProcessor#postProcessBeforeInstantiation
	 */
	@Nullable
	protected Object applyBeanPostProcessorsBeforeInstantiation(Class<?> beanClass, String beanName) {
		for (InstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().instantiationAware) {
            // 有一个实例化 Aware Bean 后置处理器处理结果非空，直接返回
			Object result = bp.postProcessBeforeInstantiation(beanClass, beanName);
			if (result != null) {
				return result;
			}
		}
		return null;
	}
}
```

可以看到，这里是用迭代器取出 `BeanPostProcessorCache` 的实例化 Aware 的后处理器，这部分的初始化逻辑为：

```java
public abstract class AbstractBeanFactory extends FactoryBeanRegistrySupport implements ConfigurableBeanFactory {
    /**
	 * Return the internal cache of pre-filtered post-processors,
	 * freshly (re-)building it if necessary.
	 * @since 5.3
	 */
	BeanPostProcessorCache getBeanPostProcessorCache() {
		synchronized (this.beanPostProcessors) {
			BeanPostProcessorCache bppCache = this.beanPostProcessorCache;
			if (bppCache == null) {
				bppCache = new BeanPostProcessorCache();
				for (BeanPostProcessor bpp : this.beanPostProcessors) {
					if (bpp instanceof InstantiationAwareBeanPostProcessor instantiationAwareBpp) {
						bppCache.instantiationAware.add(instantiationAwareBpp);
						if (bpp instanceof SmartInstantiationAwareBeanPostProcessor smartInstantiationAwareBpp) {
							bppCache.smartInstantiationAware.add(smartInstantiationAwareBpp);
						}
					}
					if (bpp instanceof DestructionAwareBeanPostProcessor destructionAwareBpp) {
						bppCache.destructionAware.add(destructionAwareBpp);
					}
					if (bpp instanceof MergedBeanDefinitionPostProcessor mergedBeanDefBpp) {
						bppCache.mergedDefinition.add(mergedBeanDefBpp);
					}
				}
				this.beanPostProcessorCache = bppCache;
			}
			return bppCache;
		}
	}
}
```

这里互斥访问 `BeanPostProcessorCacheAwareList` 本质上是一个 `CopyOnWriteArrayList`。
因此这里在实例化前的操作是将所有上下文中的 `InstantiationAwareBeanPostProcessor` 全部过一遍，一旦有一个后置处理器返回了非空，就直接返回这个代理对象，不需要再走后续初始化前的流程。

通常的博客只会解释到这一层面，至于这些接口有哪些具体的实现可以帮助更清楚的理解整个流程的则比较少，这里针对 Spring 框架和比较知名的第三方框架中对一些源代码部分的具体实现进行补充。

### AOP

首先，就是 Spring 框架中最重要的 AOP。当一个 Bean 需要被增强（事务管理、安全检查）时，Spring 会在实例化之前创建代理对象，确保后续对该 Bean 的调用都经过代理对象。

具体到 AOP 部分的代码，`AbstractAutoProxyCreator` 会在 `postProcessBeforeInstantiation` 方法中，根据一些条件判断是否要为当前正在处理的 Bean 创建代理对象。如果需要创建代理，它会返回一个代理对象，这样后续的 Bean 创建流程将直接使用这个代理对象而不是原始的 Bean 对象。

```java
@SuppressWarnings("serial")
public abstract class AbstractAutoProxyCreator extends ProxyProcessorSupport
		implements SmartInstantiationAwareBeanPostProcessor, BeanFactoryAware {
    
    @Override
	public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) {
		Object cacheKey = getCacheKey(beanClass, beanName);

		if (!StringUtils.hasLength(beanName) || !this.targetSourcedBeans.contains(beanName)) {
			if (this.advisedBeans.containsKey(cacheKey)) {
				return null;
			}
			if (isInfrastructureClass(beanClass) || shouldSkip(beanClass, beanName)) {
				this.advisedBeans.put(cacheKey, Boolean.FALSE);
				return null;
			}
		}

		// 如果有自定义的 @Target，在这里创建代理对象
		TargetSource targetSource = getCustomTargetSource(beanClass, beanName);
		if (targetSource != null) {
			if (StringUtils.hasLength(beanName)) {
				this.targetSourcedBeans.add(beanName);
			}
			Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(beanClass, beanName, targetSource);
            // 创建代理对象
			Object proxy = createProxy(beanClass, beanName, specificInterceptors, targetSource);
			this.proxyTypes.put(cacheKey, proxy.getClass());
			return proxy;
		}

		return null;
	}
}
```

因此，实例化前的后置操作可以用一个代理对象来代替实际生成的 Bean 对象。

### 实例化后处理

在实例化完成后，也可以通过后置处理器进行处理，Spring 中相关代码如下：

```java
public abstract class AbstractAutowireCapableBeanFactory extends AbstractBeanFactory implements AutowireCapableBeanFactory {
	/**
	 * 对 Bean 进行属性赋值
	 * @param beanName the name of the bean
	 * @param mbd the bean definition for the bean
	 * @param bw the BeanWrapper with bean instance
	 */
	protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
		if (bw == null) {
			if (mbd.hasPropertyValues()) {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
			}
			else {
				// 对于 null 实例跳过属性注入阶段
				return;
			}
		}

		// ...

		// 给定任何 InstantiationAwareBeanPostProcessors 在属性赋值前修改 bean 状态的机会
        // 例如，可以用于进行字段注入
		if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
			for (InstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().instantiationAware) {
                // 实例化后后置处理器调用
				if (!bp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
					return;
				}
			}
		}

		// 省略部分代码将在下文进行讲解
	}
}
```

当需要根据特定的条件进行属性注入，或者根据特定条件决定是否为某个 Bean 创建 AOP 代理时，可以通过返回 `false` 阻止属性注入。
结合后续代码，实例化后的后置方法可以结合特定条件直接跳过属性注入和属性依赖注入，同时跳过 `postProcessProperties`，但是不会影响初始化和 `BeanPostProcessor` 的执行。

## 2.2 属性后处理器 和 属性注入/填充

```java
public abstract class AbstractAutowireCapableBeanFactory extends AbstractBeanFactory implements AutowireCapableBeanFactory {
    protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {

        // 实例化后后置处理器检查是否需要进行属性注入和属性填充
        // 到这边则需要进行属性填充
        PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);

        // 确定自动装配模式
		int resolvedAutowireMode = mbd.getResolvedAutowireMode();
		if (resolvedAutowireMode == AUTOWIRE_BY_NAME || resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
			MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
			// 按名称注入
			if (resolvedAutowireMode == AUTOWIRE_BY_NAME) {
				autowireByName(beanName, mbd, bw, newPvs);
			}
			// 按类型注入
			if (resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
				autowireByType(beanName, mbd, bw, newPvs);
			}
			pvs = newPvs;
		}
        // 存在实例化感知的 Bean 后置处理器
		if (hasInstantiationAwareBeanPostProcessors()) {
			if (pvs == null) {
				pvs = mbd.getPropertyValues();
			}
            // 逐个调用，进行修改或增强
			for (InstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().instantiationAware) {
                // 【属性后处理器】
				PropertyValues pvsToUse = bp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
                // 返回 null 表示不再进行后面的属性注入，直接返回
				if (pvsToUse == null) {
					return;
				}
				pvs = pvsToUse;
			}
		}

		boolean needsDepCheck = (mbd.getDependencyCheck() != AbstractBeanDefinition.DEPENDENCY_CHECK_NONE);
		if (needsDepCheck) {
            // 过滤出需要进行依赖检查的属性描述符
			PropertyDescriptor[] filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
            // 依赖检查，依赖的 Bean 在容器中是否存在
			checkDependencies(beanName, mbd, filteredPds, pvs);
		}

        // 属性注入
		if (pvs != null) {
			applyPropertyValues(beanName, mbd, bw, pvs);
		}
    }
}
```

在属性注入之前，存在【属性后置处理器】的扩展点

### 
在属性注入之前，通过 `postProcessProperties` 可以在属性注入 Bean 之前进行校验和转换，确保数据的合法性和正确性。

例如，在一个用户注册的场景中，对用户输入的用户名、密码等属性进行校验，确保用户名符合特定的格式要求，密码满足一定的强度要求。

```java
 public class DataValidationBeanPostProcessor implements InstantiationAwareBeanPostProcessor {
    @Override
    public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) throws BeansException {
        if (bean instanceof UserRegistrationBean) {
            UserRegistrationBean userBean = (UserRegistrationBean) bean;
            // 校验用户名
            if (!isValidUsername(userBean.getUsername())) {
                throw new IllegalArgumentException("Invalid username");
            }
            // 校验密码强度
            if (!isStrongPassword(userBean.getPassword())) {
                throw new IllegalArgumentException("Password is not strong enough");
            }
        }
        return pvs;
    }
 }
```

再者，可以根据运行时的条件动态地为 Bean 的属性赋值。例如，从配置文件、数据库或其他外部资源中读取属性值，并将其注入到 Bean 中。

```java
public class DynamicPropertyBeanPostProcessor implements InstantiationAwareBeanPostProcessor {
    @Override
    public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) throws BeansException {
        if (bean instanceof ConfigurationBean) {
            ConfigurationBean configBean = (ConfigurationBean) bean;
            // 从配置文件中读取属性值
            String propertyValue = readPropertyFromConfigFile();
            pvs.addPropertyValue("dynamicProperty", propertyValue);
        }
        return pvs;
    }
}
```

如果需要阻止属性注入，则将返回值设为 `null` 即可，在迭代过程中检测到则直接从方法返回，从而阻止属性注入。

## 2.3 BeanPostProcessor

实例化后置方法是一种特殊的 Bean 后置方法。更加广义的 `BeanPostProcessor` 在实例化前后的扩展点，即最开头提到的方法 `initializeBean`。该方法的详细源代码如下：

```java
public abstract class AbstractAutowireCapableBeanFactory extends AbstractBeanFactory
		implements AutowireCapableBeanFactory {
    
    // ...

    /**
	 * 初始化给定的 Bean 实例, 使用工厂回调和初始化方法以及 Bean 后置处理器。
	 * <p>对于传统定义的 Bean，在 {@link #createBean} 方法中调用 {@link #initializeBean}.
     * 用于处理已经存在的 Bean 实例。
	 * @param beanName 工厂中的 bean 名称（用于debug）
	 * @param bean 需要实例化的新的 Bean
	 * @param mbd 给定已经存在的 Bean 实例的定义
	 * @return 初始化后的 Bean 实例（包装器）
	 * @see BeanNameAware
	 * @see BeanClassLoaderAware
	 * @see BeanFactoryAware
	 * @see #applyBeanPostProcessorsBeforeInitialization
	 * @see #invokeInitMethods
	 * @see #applyBeanPostProcessorsAfterInitialization
	 */
	@SuppressWarnings("deprecation")
	protected Object initializeBean(String beanName, Object bean, @Nullable RootBeanDefinition mbd) {
		invokeAwareMethods(beanName, bean);

		Object wrappedBean = bean;
		if (mbd == null || !mbd.isSynthetic()) {
            // 【实例化前的后置处理器方法】
			wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
		}

		try {
            // 【调用实例化方法】
			invokeInitMethods(beanName, wrappedBean, mbd);
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					(mbd != null ? mbd.getResourceDescription() : null), beanName, ex.getMessage(), ex);
		}
		if (mbd == null || !mbd.isSynthetic()) {
            // 【实例化后的后置处理器方法】
			wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
		}

		return wrappedBean;
	}
}
```

可以看到，初始化 Bean 实际上封装了调用初始化方法和 `BeanPostProcessor` 在调用初始化方法前后的后置处理方法，这也是初始化位置的扩展点。

### 初始化前后的后置处理作用

先看源代码：

```java
public abstract class AbstractAutowireCapableBeanFactory extends AbstractBeanFactory
		implements AutowireCapableBeanFactory {
    
    // ...
    
    // 初始化前的处理方法
    @Deprecated(since = "6.1")
	@Override
	public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
			throws BeansException {

		Object result = existingBean;
		for (BeanPostProcessor processor : getBeanPostProcessors()) {
			Object current = processor.postProcessBeforeInitialization(result, beanName);
			if (current == null) {
                // 返回 null 则停止后续操作，中止初始化
				return result;
			}
            // 否则更新实例化后的 bean
			result = current;
		}
		return result;
	}

    // 和上述方法类似
	@Deprecated(since = "6.1")
	@Override
	public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
			throws BeansException {

		Object result = existingBean;
		for (BeanPostProcessor processor : getBeanPostProcessors()) {
			Object current = processor.postProcessAfterInitialization(result, beanName);
			if (current == null) {
				return result;
			}
			result = current;
		}
		return result;
	}
}
```

到这里，先不介绍初始化前后的后处理方法的作用，而是回过头来思考一个问题：

> 为什么 Spring 需要将 Bean 生命周期细分为三个阶段？实例化、属性赋值和初始化有什么区别？

实际上这也是在理解每个扩展点应用之后更形而上的思考。从 Bean 自身的状态来看：

- **实例化：在内存中为 Bean 实例分配空间。** Spring 会通过构造器或者工厂方法创建实例，但是此时的 Bean 是无状态的，相当于从无到有生产了一批零件，但是还没有组装成机器。
- **属性赋值：根据配置信息将依赖项和属性注入到 Bean 中。** Spring 容器会根据配置信息进行依赖注入。此过程结束后，Bean 才拥有正确的状态，可以执行其预定的功能，相当于为机器安装电池、涂润滑油等，让机器能够按照期望的方式方式运转。
- **初始化：执行设置或者启动过程。** 相当于机器在运行前开启通信模块等，和系统其他模块验证身份等。

通过细化各个过程，可以实现 AOP 等更加灵活的行为，从而体现出 Spring IoC 的强大功能。

因此，初始化前后的后置操作可以用于：

- 添加额外的配置，例如为服务类添加系统环境变量、外部配置文件等；
- 数据校验，例如检查数据访问对象 DAO 的连接参数是否正确，或者进行格式转换
- 添加缓存或性能优化
- 日志记录和监控

示例代码如下：

```java
// 数据访问连接示例
import org.springframework.beans.BeansException;
import org.springframework.beans.factory.config.BeanPostProcessor;

public class ValidationBeanPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        if (bean instanceof DataAccessObject) {
            DataAccessObject dao = (DataAccessObject) bean;
            if (!isValidConnection(dao.getConnectionParameters())) {
                throw new IllegalArgumentException("Invalid connection parameters");
            }
            return dao;
        }
        return bean;
    }

    private boolean isValidConnection(String connectionParameters) {
        // 检查连接参数是否有效
        return true;
    }
}


// 日志监控示例
import org.springframework.beans.BeansException;
import org.springframework.beans.factory.config.BeanPostProcessor;

public class LoggingBeanPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        if (bean instanceof MonitoredService) {
            MonitoredService service = (MonitoredService) bean;
            return new MonitoredServiceWrapper(service);
        }
        return bean;
    }

    class MonitoredServiceWrapper implements MonitoredService {
        private MonitoredService delegate;

        public MonitoredServiceWrapper(MonitoredService delegate) {
            this.delegate = delegate;
        }

        @Override
        public void executeMethod() {
            long startTime = System.currentTimeMillis();
            try {
                delegate.executeMethod();
            } finally {
                long endTime = System.currentTimeMillis();
                long executionTime = endTime - startTime;
                // 记录执行时间和方法调用信息
                logExecutionTime(executionTime);
            }
        }

        private void logExecutionTime(long executionTime) {
            // 实现日志记录逻辑
        }
    }
}
```

因此：
- 如果我们需要在实际的项目中自定义一个在初始化前后的 `BeanPostProcessor`，那此时类的属性一定已经注入或者赋值了，可以在接口中针对属性进行操作。
- 如果我们需要自定义属性赋值操作，则需要在属性赋值/依赖注入之前发生。
- 如果需要在真正创建 Bean 为其分配内存空间之前进行操作，例如创建代理进行 AOP，或者在实例化之后立刻操作，则需要在实例化前后进行操作。

这就是所谓的 timing。

### 初始化前处理

由于 `BeanPostProcessor` 是一个接口，默认的初始化前置处理是直接返回 Bean：

```java
/**
 * 工厂钩子允许对新的 bean 实例的自定义修改；例如，检查是否实现了特定的标记接口或者用代理封装 bean。
 * <p>典型的例子：
 * - 后处理器实现 {@link #postProcessBeforeInitialization} 来通过标记接口填充 bean
 * - 实现 {@link #postProcessAfterInitialization} 来封装为代理
 *
 * <h3>注册</h3>
 * <p>在 {@code ApplicationContext} 中，可以自动检测在其 Bean 定义中的
 * {@code BeanPostProcessor}。一旦检测到，这些后处理器将应用于随后创建的任何 Bean。
 * 在普通的 {@code BeanFactory} 中，可以通过编程方式注册后处理器。
 * 注册后，这些后处理器将应用于通过该 BeanFactory 创建的所有 Bean。
 *
 * <h3>排序</h3>
 * <p>{@code ApplicationContext} 会根据 {@link org.springframework.core.PriorityOrdered} 
 * 和 {@link org.springframework.core.Ordered} 接口的语义对自动检测到的
 * {@code BeanPostProcessor} 进行排序。
 * 但是，对于编程式注册的 {@code BeanPostProcessor}，其实现的
 * {@link org.springframework.core.PriorityOrdered} 和
 * {@link org.springframework.core.Ordered} 接口的排序语义将被忽略。
 *
 * @author Juergen Hoeller
 * @author Sam Brannen
 * @since 10.10.2003
 * @see InstantiationAwareBeanPostProcessor
 * @see DestructionAwareBeanPostProcessor
 * @see ConfigurableBeanFactory#addBeanPostProcessor
 * @see BeanFactoryPostProcessor
 */
public interface BeanPostProcessor {

	/**
	 * 在任何初始化回调 (例如 InitializingBean's {@code afterPropertiesSet}
	 * 或者自定义的 init-method）之前调用. 此时 bean 已经被属性值填充
	 * 返回的 bean 可能是原来实例的封装。
	 * @see org.springframework.beans.factory.InitializingBean#afterPropertiesSet
	 */
	@Nullable
	default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}
}
```

因此，我们看一下 Spring 内置的具体实现类的调用 `ApplicationContextAwareProcessor#`



```java
/**
 * 这个实现类 {@link BeanPostProcessor} 为 Bean 提供了：
 * {@link org.springframework.context.ApplicationContext ApplicationContext}：应用上下文，
 * {@link org.springframework.core.env.Environment Environment}：环境变量，
 * {@link StringValueResolver}：字符串解析器，
 * {@link org.springframework.core.metrics.ApplicationStartup ApplicationStartup}：应用启动信息，
 * {@code ApplicationContext} 会为实现了：
 * {@link EnvironmentAware},
 * {@link EmbeddedValueResolverAware}, 
 * {@link ResourceLoaderAware},
 * {@link ApplicationEventPublisherAware}, 
 * {@link MessageSourceAware},
 * {@link ApplicationStartupAware},
 * {@link ApplicationContextAware} 
 * 接口的 Bean 提供对应的对象。
 *
 * <p>应用上下文（ApplicationContext）会自动将这个BeanPostProcessor注册到其底层的 Bean 工厂中。
 * 开发人员在应用中通常不需要直接使用这个BeanPostProcessor，它是由 Spring 框架在内部自动处理的。
 *
 * @author Juergen Hoeller
 * @author Costin Leau
 * @author Chris Beams
 * @author Sam Brannen
 * @since 10.10.2003
 * @see org.springframework.context.EnvironmentAware
 * @see org.springframework.context.EmbeddedValueResolverAware
 * @see org.springframework.context.ResourceLoaderAware
 * @see org.springframework.context.ApplicationEventPublisherAware
 * @see org.springframework.context.MessageSourceAware
 * @see org.springframework.context.ApplicationStartupAware
 * @see org.springframework.context.ApplicationContextAware
 * @see org.springframework.context.support.AbstractApplicationContext#refresh()
 */
class ApplicationContextAwareProcessor implements BeanPostProcessor {

    private final ConfigurableApplicationContext applicationContext;

	private final StringValueResolver embeddedValueResolver;


	/**
	 * 为给定上下文创建新的 ApplicationContextAwareProcessor。
	 */
	public ApplicationContextAwareProcessor(ConfigurableApplicationContext applicationContext) {
		this.applicationContext = applicationContext;
		this.embeddedValueResolver = new EmbeddedValueResolver(applicationContext.getBeanFactory());
	}


	@Override
	@Nullable
	public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        // 实现了感知接口，就调用对应的方法
		if (bean instanceof Aware) {
			invokeAwareInterfaces(bean);
		}
		return bean;
	}

	private void invokeAwareInterfaces(Object bean) {
		if (bean instanceof EnvironmentAware environmentAware) {
			environmentAware.setEnvironment(this.applicationContext.getEnvironment());
		}
		if (bean instanceof EmbeddedValueResolverAware embeddedValueResolverAware) {
			embeddedValueResolverAware.setEmbeddedValueResolver(this.embeddedValueResolver);
		}
		if (bean instanceof ResourceLoaderAware resourceLoaderAware) {
			resourceLoaderAware.setResourceLoader(this.applicationContext);
		}
		if (bean instanceof ApplicationEventPublisherAware applicationEventPublisherAware) {
            applicationEventPublisherAware.setApplicationEventPublisher(this.applicationContext);
		}
		if (bean instanceof MessageSourceAware messageSourceAware) {
			messageSourceAware.setMessageSource(this.applicationContext);
		}
		if (bean instanceof ApplicationStartupAware applicationStartupAware) {
			applicationStartupAware.setApplicationStartup(this.applicationContext.getApplicationStartup());
		}
		if (bean instanceof ApplicationContextAware applicationContextAware) {
			applicationContextAware.setApplicationContext(this.applicationContext);
		}
	}
}
```

在初始化之前，如果 Bean 实现了这些感知器，就可以把上下文对应的对象提供给 bean。参数都是 `ApplicationContextAware`，因为它本身也实现了这些接口。

至于这些接口为什么都已感知（Aware）作为后缀，我推测是因为 Bean 实现这些接口后就可以“感知”到上下文中存储的对应变量。

这些感知接口具体的功能如下：

- `Environment`：应用程序的运行环境信息。可以根据不同环境加载不同配置文件，从而修改 Bean 的行为。
- `EmbeddedValueResolver`：解析嵌入在字符串中的占位符，配置文件动态替换字符串占位符，例如 `${property.key}`。
- `ResourceLoader`：用以加载各种资源，例如属性文件、XML 文件。
- `ApplicationEventPublisher`：发布自定义的应用程序事件，实现事件驱动的架构。
- `MessageSource`：访问消息源，获取国际化的消息。
- `ApplicationStartup`：获取应用程序的启动信息，在应用启动时记录启动时间、监控启动过程中的性能指标等。
- `ApplicationContext`：使 Bean 能够获取应用程序上下文，从而可以访问 Spring 容器中的其他 Bean、获取配置信息等。用于进行依赖注入等。

## 2.4 初始化

```java
public abstract class AbstractAutowireCapableBeanFactory extends AbstractBeanFactory
		implements AutowireCapableBeanFactory {
    
    // ...

    /**
     * 在 Bean 的所有属性都被设置后初始化自身，并知悉自己的 Bean 工厂。
     * 这就意味着检查这个 bean 是否实现了 {@link InitializingBean} 或者定义了自定义方法，
     * 并调用对应的回调函数。
	 * @param beanName 工厂中的 Bean 名称 (用于 debug)
	 * @param bean 需要初始化的新的 Bean 实例
	 * @param mbd 如果给定 Bean 实例，则是创建 Bean 的归并的 Bean 定义，也可以是 {@code null}
	 * @throws Throwable 调用阶段由初始化方法抛出
	 * @see #invokeCustomInitMethod
	 */
	protected void invokeInitMethods(String beanName, Object bean, @Nullable RootBeanDefinition mbd) throws Throwable {
        
        // 是否实现了初始化接口
		boolean isInitializingBean = (bean instanceof InitializingBean);
		if (isInitializingBean && (mbd == null || !mbd.hasAnyExternallyManagedInitMethod("afterPropertiesSet"))) {
			if (logger.isTraceEnabled()) {
				logger.trace("Invoking afterPropertiesSet() on bean with name '" + beanName + "'");
			}
            // 执行【afterPropertySet】
			((InitializingBean) bean).afterPropertiesSet();
        }

		if (mbd != null && bean.getClass() != NullBean.class) {
			String[] initMethodNames = mbd.getInitMethodNames();
			if (initMethodNames != null) {
                // 依次调用自定义的初始化方法
				for (String initMethodName : initMethodNames) {
					if (StringUtils.hasLength(initMethodName) &&
							!(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
							!mbd.hasAnyExternallyManagedInitMethod(initMethodName)) {
                        // 自定义【init-method】
						invokeCustomInitMethod(beanName, bean, mbd, initMethodName);
					}
				}
			}
		}
	}
}
```

## 3 工厂级后处理方法（BeanFactoryProcessor）

Spring IoC 容器初始化的关键环节就在 `org.springframework.context.support.AbstractApplicationContext#refresh` 方法，其中在最开始部分就调用

```java
public abstract class AbstractApplicationContext extends DefaultResourceLoader
		implements ConfigurableApplicationContext {
    @Override
	public void refresh() throws BeansException, IllegalStateException {
		this.startupShutdownLock.lock();
		try {
            // ...

			// 告知子类刷新内部的 bean 工厂
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// 准备 Bean 工厂
			prepareBeanFactory(beanFactory);

			try {
				// 在上下文子类中允许 bean 工厂的处理器
				postProcessBeanFactory(beanFactory);

				StartupStep beanPostProcess = this.applicationStartup.start("spring.context.beans.post-process");
				// 调用【工厂的处理方法】，作为 beans 注册到容器上下文.
				invokeBeanFactoryPostProcessors(beanFactory);
				// 注册 bean 处理器拦截 bean 的创建
				registerBeanPostProcessors(beanFactory);
				beanPostProcess.end();

				// ...
			}
            // ...
        }
	}

    // ...

    /**
	 * 实例化并调用所有的已经注册的 BeanFactoryPostProcessor bean,
     * 如果显式指定顺序则按照顺序。
	 * <p>必须在单例实例化前调用.
	 */
	protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
        // 实际调用的接口
		PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());

		// 检测 LoadTimeWeaver 并准备编织（如果在此期间找到）
		// (e.g.例如通过 ConfigurationClassPostProcessor 的 @Bean 方法注册)
		if (!NativeDetector.inNativeImage() && beanFactory.getTempClassLoader() == null &&
				beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
			beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
			beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
		}
	}
}
```

其中调用工厂处理方法，实际执行的类是 `PostProcessorRegistrationDelegate`，这是一个 `final` 类，强制使用组合调用

```java
// 后处理器注册委托
final class PostProcessorRegistrationDelegate {

	private PostProcessorRegistrationDelegate() {
	}


	public static void invokeBeanFactoryPostProcessors(
			ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {

		Set<String> processedBeans = new HashSet<>();

        // 如果实现了 `BeanDefinitionRegistry`，这是一个 Bean 定义的注册器
		if (beanFactory instanceof BeanDefinitionRegistry registry) {
            // 处理 BeanFacotry 的父接口，作用于整个 BeanFactory
            // 可以修改 Bean 的属性、替换定义，但是不能添加新的定义
            List<BeanFactoryPostProcessor> regularPostProcessors = new ArrayList<>();
			// 实现了 BeanFactoryPostProcessor 接口的子接口，作用于 BeanDefinitionRegistry
            // 除了父接口功能，可以添加新的 Bean 的定义
            List<BeanDefinitionRegistryPostProcessor> registryProcessors = new ArrayList<>();

			for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
				// 如果是注册器的处理器
                if (postProcessor instanceof BeanDefinitionRegistryPostProcessor registryProcessor) {
                    // 就处理一下注册器
					registryProcessor.postProcessBeanDefinitionRegistry(registry);
					// 再把注册器处理器添加到列表
                    registryProcessors.add(registryProcessor);
				}
				else {
                    // 只能添加，只是普通的 Bean 工厂处理器
					regularPostProcessors.add(postProcessor);
				}
			}

			// 先往下看，下文会说明这个列表存储的是：
            // 实现了 PriorityOrdered/Order 接口的 BeanDefinitionRegistryPostProcessors
			List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<>();

            // 首先，调用实现 PriorityOrdered 接口的 BeanDefinitionRegistryPostProcessors
            // 获取所有 Bean Definition 注册器处理器类的 Bean 名称
			String[] postProcessorNames =
					beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
			for (String ppName : postProcessorNames) {
                // 如果 Bean 实现了 PriorityOrdered 接口
				if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
					currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
					processedBeans.add(ppName);
				}
			}

			sortPostProcessors(currentRegistryProcessors, beanFactory);
			registryProcessors.addAll(currentRegistryProcessors);
			// 调用所有这些处理器，把注册器（工厂）、启动器也扔进去
            invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry, beanFactory.getApplicationStartup());
			currentRegistryProcessors.clear();

            // 接下来，调用实现 Ordered 接口的 BeanDefinitionRegistryPostProcessors
			postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
			for (String ppName : postProcessorNames) {
                // 没有实现 PriorityOrdered 接口但是实现了 Ordered 接口的 BeanDefinitionRegistryPostProcessor
				if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
					currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
					processedBeans.add(ppName);
				}
			}

            // 同样的逻辑
			sortPostProcessors(currentRegistryProcessors, beanFactory);
			registryProcessors.addAll(currentRegistryProcessors);
			invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry, beanFactory.getApplicationStartup());
			currentRegistryProcessors.clear();

			// 最后，调用其他的处理器的执行方法
			boolean reiterate = true;
			while (reiterate) {
				reiterate = false;
				postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
				for (String ppName : postProcessorNames) {
                    // 找出还没有执行的处理器
					if (!processedBeans.contains(ppName)) {
						currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
						processedBeans.add(ppName);
						reiterate = true;
					}
				}
				sortPostProcessors(currentRegistryProcessors, beanFactory);
				registryProcessors.addAll(currentRegistryProcessors);
				invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry, beanFactory.getApplicationStartup());
				currentRegistryProcessors.clear();
			}

			// 调用目前已经执行过的所有的 postProcessBeanFactory 的回调.
			invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);
			invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
		}

		else {
            // 只是普通的工厂，调用用上下文实例注册的工厂处理器
			invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
		}

		// Do not initialize FactoryBeans here: We need to leave all regular beans
		// uninitialized to let the bean factory post-processors apply to them!
		String[] postProcessorNames =
				beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);

		// 将 BeanFactoryPostProcessors 按照是否实现 PriorityOrdered, Ordered 接口分为三类.
		List<BeanFactoryPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
		List<String> orderedPostProcessorNames = new ArrayList<>();
		List<String> nonOrderedPostProcessorNames = new ArrayList<>();
		for (String ppName : postProcessorNames) {
			if (processedBeans.contains(ppName)) {
				// 跳过，在第一阶段已经处理过了
			}
			else if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
				priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
			}
			else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
				orderedPostProcessorNames.add(ppName);
			}
			else {
				nonOrderedPostProcessorNames.add(ppName);
			}
		}

		// 首先，调用实现了 PriorityOrdered 接口的 BeanFactoryPostProcessors.
		sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
		invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);

		// 接着，调用实现了 Ordered 接口的 BeanFactoryPostProcessors.
		List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<>(orderedPostProcessorNames.size());
		for (String postProcessorName : orderedPostProcessorNames) {
			orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
		}
		sortPostProcessors(orderedPostProcessors, beanFactory);
		invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);

		// 最后，调用其他所有的 BeanFactoryPostProcessors.
		List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<>(nonOrderedPostProcessorNames.size());
		for (String postProcessorName : nonOrderedPostProcessorNames) {
			nonOrderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
		}
		invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);

		// 清除缓存的归并 Bean 定义，由于后置处理器可能修改了最初的元数据
        // 例如，替换占位符的值...
		beanFactory.clearMetadataCache();
	}
}
```
