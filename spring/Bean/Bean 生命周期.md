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

		// 给定任何 InstantiationAwareBeanPostProcessors 在属性赋值前修改 bean 状态的机会. 
        // 例如，可以用于进行字段注入
		if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
			for (InstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().instantiationAware) {
				if (!bp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
					return;
				}
			}
		}

		PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);

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
		if (hasInstantiationAwareBeanPostProcessors()) {
			if (pvs == null) {
				pvs = mbd.getPropertyValues();
			}
			for (InstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().instantiationAware) {
				PropertyValues pvsToUse = bp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
				if (pvsToUse == null) {
					return;
				}
				pvs = pvsToUse;
			}
		}

		boolean needsDepCheck = (mbd.getDependencyCheck() != AbstractBeanDefinition.DEPENDENCY_CHECK_NONE);
		if (needsDepCheck) {
			PropertyDescriptor[] filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
			checkDependencies(beanName, mbd, filteredPds, pvs);
		}

		if (pvs != null) {
			applyPropertyValues(beanName, mbd, bw, pvs);
		}
	}
}
```