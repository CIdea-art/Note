本篇目的在于理清流程，细节方面建议参照源码。

# 一、AOP创建过程，切面代理的创建类AnnotationAwareAspectJAutoProxyCreator

**`AnnotationAwareAspectJAutoProxyCreator`**用于创建bean的代理类。



## AOP代理织入点

首先介绍ProxyCreator中的几个属性。

- advisedBeans

```java
/**
 * advisedBeans是一个bean的代理状态表。key不存在时表示未执行，判断是否需要代理。key存在且有值时，true：正在创建代理，false：不需要代理（系统类、已完成等）
 */
private final Map<Object, Boolean> advisedBeans = new ConcurrentHashMap<>(256);
```
### 常规代理

常规代理又分为是否允许循环依赖的两种情况。

在Spring常规对象创建过程中，采用的是一个链式过程，在bean初始化依赖时，会触发被引用对象的创建过程然后引用。若两个对象互相依赖，则会进入无限创建的死循环。

因此定义了一个K-V结构（early**命名），在对象实例化时就将其加入映射表，初始化依赖时尝试从该表中取出依赖的对象，否则创建依赖对象。

`#earlyProxyReferences`就是代理过程中同样作用的一个属性，防止代理过程中因bean与代理示例不是同一个实例引起的问题，提前实例化代理类并缓存进表中。其中无值则表示未执行过代理，有值则相反。

```java
/**
 * 代理创建过程中bean的引用Map，与BeanFactory中的early类似，value存储的是#doCreateBean()时创建的原生bean，代表bean已触发过代理创建过程
 */
private final Map<Object, Object> earlyProxyReferences = new ConcurrentHashMap<>(16);
```

#### 允许循环依赖

由于代理与原生bean不是同一个引用，因此创建bean时调用`#getEarlyBeanReference()`提前触发代理过程返回代理对象，将bean加入`#earlyProxyReferences`代表已触发过代理创建过程，并调用`#addSingletonFactory()`方法将返回的代理对象会加到beanMap中。

```java
protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
      throws BeanCreationException {
	......
    
    // Eagerly cache singletons to be able to resolve circular references
    // even when triggered by lifecycle interfaces like BeanFactoryAware.
    // 解决单例模式的循环依赖
    boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
                                          isSingletonCurrentlyInCreation(beanName));
    if (earlySingletonExposure) {
        if (logger.isTraceEnabled()) {
            logger.trace("Eagerly caching bean '" + beanName +
                         "' to allow for resolving potential circular references");
        }
        // 提前将 bean 实例（无论是否被代理替换）加入到 beanMap 中
        // 防止循环依赖，防止代理对象之间依赖引用异常（引用指向未被代理的原生对象）
        addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
    }
    ......
    // Initialize the bean instance.
    Object exposedObject = bean;
    ......
    
    // 初始化bean完成后，下文续
}

protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
    Object exposedObject = bean;
    if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
        for (SmartInstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().smartInstantiationAware) {
            exposedObject = bp.getEarlyBeanReference(exposedObject, beanName);
        }
    }
    return exposedObject;
}
```

`AbstractAutoProxyCreator`实现`#getEarlyBeanReference()`方法，提前调用`#wrapIfNecessary`尝试创建代理并记录到`#earlyProxyReferences`表中。

```java
// AbstractAutoProxyCreator.class
@Override
public Object getEarlyBeanReference(Object bean, String beanName) {
    Object cacheKey = getCacheKey(bean.getClass(), beanName);
    this.earlyProxyReferences.put(cacheKey, bean);
    // 下文说明#wrapIfNecessary()方法
    return wrapIfNecessary(bean, beanName, cacheKey);
}
```

bean初始化完成后，若有提前缓存的对象，则代替返回。


```java
// 接上文doCreateBean()方法初始化bean后
if (earlySingletonExposure) {
	// 获取 earlySingletonReference
	Object earlySingletonReference = getSingleton(beanName, false);
	if (earlySingletonReference != null) {
		if (exposedObject == bean) {
            // 如果 exposedObject 在初始化方法initializeBean中没有被改变
            // 则表示其已经在getEarlyBeanReference()中提前代理了对象，跳过了代理过程，beanMap中缓存的代理对象代替原始bean返回
			exposedObject = earlySingletonReference;
		}
        ......
}
......
```

#### 不允许循环依赖

是`BeanPostProcessor`、`InstantiationAwareBeanPostProcessor`的实现类，AOP代理主要借助这两个接口在Spring IOC中的作用。注册方式根据情况区分，第三节说明。

> 接口实现在父类**`AbstractAutoProxyCreator`**中。
>
> - `InstantiationAwareBeanPostProcessor`，bean实例化前后的处理器
>   - `#postProcessBeforeInstantiation()`，实例化的前置处理。默认返回null，即不处理。重写返回自定义创建的bean时，会中断Spring bean的创建过程并代替将要创建的bean。
>   - `#postProcessAfterInstantiation()`，实例化后置处理，默认true，表示执行成功，失败则中断处理链路。一般在此处进行Spring的属性填充，如`@Autowrite`等。
> - `BeanPostProcessor`，bean初始化前后的处理器
>   - `#postProcessBeforeInitialization()`，bean的初始化前置处理，默认返回bean，不处理。通常用于标记各类接口，填充属性，诸如`Aware`的各类子接口。后文举例说明。
>   - `#postProcessAfterInitialization()`，bean的初始化后置处理，默认返回bean，不处理。通常用于一层层的代理包装原bean。
>   - 返回null则中断处理链路，继续创建bean的后续流程。
>
> 在默认`DefaultListableBeanFactory`的执行顺序为：
>
> 1. 实例化前置，`#postProcessBeforeInstantiation()`
> 2. 初始化前置，`#postProcessBeforeInitialization()`
> 3. 初始化后置，`#postProcessAfterInitialization()`
> 4. 实例化后置，`#postProcessBeforeInstantiation()`
>
> 这里仅简要介绍接口，方便后文解释。具体流程、示例参照另一篇《Spring IOC》

使用`#postProcessAfterInitialization()`实现，在bean初始化后完成。

```java
public abstract class AbstractAutoProxyCreator extends ProxyProcessorSupport
		implements SmartInstantiationAwareBeanPostProcessor, BeanFactoryAware {

    @Override
    public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
       if (bean != null) {
          // 获取当前bean的key：如果beanName不为空，则以beanName为key，如果为FactoryBean类型，
          // 前面还会添加&符号，如果beanName为空，则以当前bean对应的class为key
          Object cacheKey = getCacheKey(bean.getClass(), beanName);
          // 判断当前bean是否正在被代理或代理完成，否则进行封装
          if (this.earlyProxyReferences.remove(cacheKey) != bean) {
             // 对当前bean进行封装，一般的spring代理都在此进行
             return wrapIfNecessary(bean, beanName, cacheKey);
          }
       }
       return bean;
    }
}
```
大时代
```java 
	protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
		// 判断当前 bean 是否在 TargetSource 缓存中存在，如果存在，则直接返回当前 bean。
		// 这里进行如此判断的原因是在上文中，我们讲解了如何通过自己声明的 TargetSource 进行目标 bean 的封装，
		// 在封装之后其实就已经对封装之后的 bean 进行了代理，并且添加到了 targetSourcedBeans 缓存中。因而这里判断得到
		// 当前缓存中已经存在当前 bean，则说明该 bean 已经被代理过，这样就可以直接返回当前 bean。
		if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
			return bean;
		}
		// 这里advisedBeans缓存了不需要代理的bean，如果缓存中存在，则可以直接返回
		if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
			return bean;
		}
		// 这里isInfrastructureClass()用于判断当前bean是否为Spring系统自带的bean，自带的bean是不用进行代理的；
		// shouldSkip()则用于判断当前bean是否应该被略过
		if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
			this.advisedBeans.put(cacheKey, Boolean.FALSE);
			return bean;
		}

		// Create proxy if we have advice.
		// 获取当前bean的Advices和Advisors，Advice表示需要织入的切面逻辑（@Before、@After和@Around等），而Advisor则表示将切面逻辑进行封装之后的织入者。
		Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
		if (specificInterceptors != DO_NOT_PROXY) {
			// 对当前bean的代理状态进行缓存
			this.advisedBeans.put(cacheKey, Boolean.TRUE);
			Object proxy = createProxy(
					bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
			// 缓存生成的代理bean的类型，并且返回生成的代理bean
			this.proxyTypes.put(cacheKey, proxy.getClass());
			return proxy;
		}

		this.advisedBeans.put(cacheKey, Boolean.FALSE);
		return bean;
	}
    ...
}
```



### 自定义对象代理

通过设置`#customTargetSourceCreators`属性自定义代理目标，匹配时提前创建代理，中断后续流程

- targetSourcedBeans

```java
/**
 * targetSourcedBeans是一个用来存储代理aop代理beanName的集合，用ConcurrentHashMap实现的Set
 */
private final Set<String> targetSourcedBeans = Collections.newSetFromMap(new ConcurrentHashMap<>(16));
```

```java
public abstract class AbstractAutoProxyCreator extends ProxyProcessorSupport
		implements SmartInstantiationAwareBeanPostProcessor, BeanFactoryAware {

    @Override
    public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) {
		// #getCacheKey()生成cacheKey
        Object cacheKey = getCacheKey(beanClass, beanName);
        if (!StringUtils.hasLength(beanName) || !this.targetSourcedBeans.contains(beanName)) {
            // TargetSource 缓存中不包含当前 bean
            if (this.advisedBeans.containsKey(cacheKey)) {
                // advisedBeans key包含cacheKey，代表当前 bean 已被处理，不处理
                return null;
            }
            if (isInfrastructureClass(beanClass) || shouldSkip(beanClass, beanName)) {
                // 当前 bean 切面逻辑不会包含的 bean, Advice/Pointcut/Advisor/AopInfrastructureBean
                // 或是系统 bean
                // 不需要代理
                // 将当前 bean 的处理状态缓存到 advisedBeans 中，false代表不需要代理
                this.advisedBeans.put(cacheKey, Boolean.FALSE);
                return null;
            }
        }

        // Create proxy here if we have a custom TargetSource.
        // Suppresses unnecessary default instantiation of the target bean:
        // The TargetSource will handle target instances in a custom fashion.
        // 获取封装当前 bean 的自定义 TargetSource 对象，从 TargetSource 中获取当前 bean 对象，并且判断是否需要将切面逻辑应用在当前 bean 上。
        // 如果不存在，则直接退出当前方法
        TargetSource targetSource = getCustomTargetSource(beanClass, beanName);
        if (targetSource != null) {
            // 在TargetSource中没有进行缓存，并且应该被切面逻辑环绕，但是目前还未生成代理对象的bean
            if (StringUtils.hasLength(beanName)) {
                this.targetSourcedBeans.add(beanName);
            }
            // 获取能够应用当前bean的切面逻辑
            Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(beanClass, beanName, targetSource);
            // 根据切面逻辑为当前 bean 生成代理对象
            Object proxy = createProxy(beanClass, beanName, specificInterceptors, targetSource);
            // 对生成的代理对象进行缓存
            this.proxyTypes.put(cacheKey, proxy.getClass());
            // 直接返回生成的代理对象，从而使后续 bean 的创建工作短路
            return proxy;
        }

        return null;
    }
    
    /**
	 * xml bean装配
	 * @see org.springframework.aop.framework.autoproxy.target.LazyInitTargetSourceCreator
	 */
    @Nullable
    private TargetSourceCreator[] customTargetSourceCreators;

    public void setCustomTargetSourceCreators(TargetSourceCreator... targetSourceCreators) {
        this.customTargetSourceCreators = targetSourceCreators;
    }
    ...

    @Nullable
    protected TargetSource getCustomTargetSource(Class<?> beanClass, String beanName) {
        // We can't create fancy target sources for directly registered singletons.
        if (this.customTargetSourceCreators != null &&
            this.beanFactory != null && this.beanFactory.containsBean(beanName)) {
            for (TargetSourceCreator tsc : this.customTargetSourceCreators) {
                TargetSource ts = tsc.getTargetSource(beanClass, beanName);
                if (ts != null) {
                    // Found a matching TargetSource.
                    if (logger.isTraceEnabled()) {
                        logger.trace("TargetSourceCreator [" + tsc +
                                     "] found custom TargetSource for bean with name '" + beanName + "'");
                    }
                    return ts;
                }
            }
        }

        // No custom TargetSource found.
        return null;
    }
}
```


## 创建代理

### 获取可用的拦截器getAdvicesAndAdvisorsForBean(...)

该方法主要调用findEligibleAdvisors()。

findEligibleAdvisors()中进行了三个过程。

<1.1.1>获取所有 Advisor 的实现类。

<1.1.2>过滤出能够应用到当前 bean 的Advisor。

<1.1.3>排序。

```java
// AbstractAdvisorAutoProxyCreator.class，AnnotationAwareAspectJAutoProxyCreator的上级类

@Override
@Nullable
protected Object[] getAdvicesAndAdvisorsForBean(
    Class<?> beanClass, String beanName, @Nullable TargetSource targetSource) {

    // 为目标 bean 查找可用的 Advisor
    List<Advisor> advisors = findEligibleAdvisors(beanClass, beanName);
    if (advisors.isEmpty()) {
        return DO_NOT_PROXY;
    }
    return advisors.toArray();
}

protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {
    // <1.1.1>从 beanFactory 中获取所有 Advisor，具体实现后续再详解，暂时略过
    List<Advisor> candidateAdvisors = findCandidateAdvisors();
    // <1.1.2>对获取到的所有 Advisor 进行判断，看其切面定义是否可以应用到当前 bean，从而得到最终需要应用的 Advisor
    List<Advisor> eligibleAdvisors = findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
    // 用于对目标 Advisor 进行扩展
    /**
	 * @see ExposeInvocationInterceptor.ADVISOR
	 */
    extendAdvisors(eligibleAdvisors);
    if (!eligibleAdvisors.isEmpty()) {
        // <1.1.3>对需要代理的Advisor按照一定的规则进行排序
        eligibleAdvisors = sortAdvisors(eligibleAdvisors);
    }
    return eligibleAdvisors;
}

protected List<Advisor> findAdvisorsThatCanApply(
    List<Advisor> candidateAdvisors, Class<?> beanClass, String beanName) {

    ProxyCreationContext.setCurrentProxiedBeanName(beanName);
    try {
        return AopUtils.findAdvisorsThatCanApply(candidateAdvisors, beanClass);
    }
    finally {
        ProxyCreationContext.setCurrentProxiedBeanName(null);
    }
}
```


#### <1.1.1>findCandidateAdvisors

```java
// AbstractAdvisorAutoProxyCreator.class
protected List<Advisor> findCandidateAdvisors() {
    Assert.state(this.advisorRetrievalHelper != null, "No BeanFactoryAdvisorRetrievalHelper available");
    return this.advisorRetrievalHelper.findAdvisorBeans();
}

// BeanFactoryAdvisorRetrievalHelper.class
public List<Advisor> findAdvisorBeans() {
   // Determine list of advisor bean names, if not cached already.
   String[] advisorNames = this.cachedAdvisorBeanNames;
   if (advisorNames == null) {
      // 缓存为空
      // Do not initialize FactoryBeans here: We need to leave all regular beans
      // uninitialized to let the auto-proxy creator apply to them!
      // 获取当前BeanFactory中所有实现了Advisor接口的bean的名称
      advisorNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
            this.beanFactory, Advisor.class, true, false);
      this.cachedAdvisorBeanNames = advisorNames;
   }
   if (advisorNames.length == 0) {
      return new ArrayList<>();
   }

   // 对获取到的实现Advisor接口的bean的名称进行遍历
   List<Advisor> advisors = new ArrayList<>();
   for (String name : advisorNames) {
      // isEligibleBean()是提供的一个hook方法，用于子类对Advisor进行过滤，这里默认返回值都是true
      if (isEligibleBean(name)) {
         if (this.beanFactory.isCurrentlyInCreation(name)) {
            // 如果当前 bean 还在创建过程中，则略过
            // [2]其创建完成之后会为其判断是否需要织入切面逻辑
            if (logger.isTraceEnabled()) {
               logger.trace("Skipping currently created advisor '" + name + "'");
            }
            continue;
         }
         try {
            advisors.add(this.beanFactory.getBean(name, Advisor.class));
         }
         catch (BeanCreationException ex) {
            Throwable rootCause = ex.getMostSpecificCause();
            if (rootCause instanceof BeanCurrentlyInCreationException) {
               BeanCreationException bce = (BeanCreationException) rootCause;
               String bceBeanName = bce.getBeanName();
               if (bceBeanName != null && this.beanFactory.isCurrentlyInCreation(bceBeanName)) {
                  // 如果其它线程正在处理当前 bean 的创建
                  if (logger.isTraceEnabled()) {
                     logger.trace("Skipping advisor '" + name +
                           "' with dependency on currently created bean: " + ex.getMessage());
                  }
                  // Ignore: indicates a reference back to the bean we're trying to advise.
                  // We want to find advisors other than the currently created bean itself.
                  continue;
               }
            }
            throw ex;
         }
      }
   }
   return advisors;
}
```

[^2]: 正在创建中的advisor会被跳过，创建完成后如何再次织入？

#### <1.1.2>findAdvisorsThatCanApply()

```java
// AopUtils.class
public static List<Advisor> findAdvisorsThatCanApply(List<Advisor> candidateAdvisors, Class<?> clazz) {
	if (candidateAdvisors.isEmpty()) {
		return candidateAdvisors;
	}
	List<Advisor> eligibleAdvisors = new ArrayList<>();
	for (Advisor candidate : candidateAdvisors) {
		// 判断是否为IntroductionAdvisor，并且判断是否可以应用到当前类上，根据 IntroductionAdvisor#getClassFilter()#matches(Class) 进行匹配
		if (candidate instanceof IntroductionAdvisor && canApply(candidate, clazz)) {
			eligibleAdvisors.add(candidate);
		}
	}
	// 根据上一步是否添加过 IntroductionAdvisor 类型
	// 通过canApply()方法判断当前 advisor 是否可以应用到当前 bean
	boolean hasIntroductions = !eligibleAdvisors.isEmpty();
	for (Advisor candidate : candidateAdvisors) {
		if (candidate instanceof IntroductionAdvisor) {
			// already processed
			continue;
		}
		if (canApply(candidate, clazz, hasIntroductions)) {
			eligibleAdvisors.add(candidate);
		}
	}
	return eligibleAdvisors;
}

public static boolean canApply(Advisor advisor, Class<?> targetClass) {
    return canApply(advisor, targetClass, false);
}

public static boolean canApply(Advisor advisor, Class<?> targetClass, boolean hasIntroductions) {
    if (advisor instanceof IntroductionAdvisor) {
		// 当前 advisor 为 IntroductionAdvisor，按照IntroductionAdvisor的方式进行过滤
        return ((IntroductionAdvisor) advisor).getClassFilter().matches(targetClass);
    }
    else if (advisor instanceof PointcutAdvisor) {
        PointcutAdvisor pca = (PointcutAdvisor) advisor;
        return canApply(pca.getPointcut(), targetClass, hasIntroductions);
    }
    else {
        // It doesn't have a pointcut so we assume it applies.
        return true;
    }
}

public static boolean canApply(Pointcut pc, Class<?> targetClass, boolean hasIntroductions) {
    Assert.notNull(pc, "Pointcut must not be null");
    // 1、过滤类
    // 获取当前 advisor 的 CalssFilter，并且调用其 matches() 方法判断当前切点表达式是否与目标 bean 匹配，
    // 这里ClassFilter指代的切点表达式主要是当前切面类上使用的 @Aspect 注解中所指代的切点表达式
    if (!pc.getClassFilter().matches(targetClass)) {
        return false;
    }

    // 2、过滤方法a
    // 判断如果当前Advisor所指代的方法的切点表达式如果是对任意方法都放行，则直接返回
    MethodMatcher methodMatcher = pc.getMethodMatcher();
    if (methodMatcher == MethodMatcher.TRUE) {
        // No need to iterate the methods if we're matching any method anyway...
        return true;
    }

    // 2、过滤方法b
    // 这里将 MethodMatcher 强转为 IntroductionAwareMethodMatcher 类型的原因在于，
    // 如果目标类不包含 Introduction 类型的 Advisor，那么使用
    // IntroductionAwareMethodMatcher.matches()方法进行匹配判断时可以提升匹配的效率，
    // 其会判断目标 bean 中没有使用 Introduction 织入新的方法，则可以使用该方法进行静态匹配，从而提升效率
    // 因为 Introduction 类型的 Advisor 可以往目标类中织入新的方法，新的方法也可能是被AOP环绕的方法
    IntroductionAwareMethodMatcher introductionAwareMethodMatcher = null;
    if (methodMatcher instanceof IntroductionAwareMethodMatcher) {
        introductionAwareMethodMatcher = (IntroductionAwareMethodMatcher) methodMatcher;
    }

    // 获取目标类的所有接口
    Set<Class<?>> classes = new LinkedHashSet<>();
    if (!Proxy.isProxyClass(targetClass)) {
        classes.add(ClassUtils.getUserClass(targetClass));
    }
    classes.addAll(ClassUtils.getAllInterfacesForClassAsSet(targetClass));

    for (Class<?> clazz : classes) {
        // 获取目标接口的所有方法
        Method[] methods = ReflectionUtils.getAllDeclaredMethods(clazz);
        for (Method method : methods) {
            // 如果当前 MethodMatcher 也是 IntroductionAwareMethodMatcher 类型，则使用该类型
            // 的方法进行匹配，从而达到提升效率的目的；否则使用MethodMatcher.matches()方法进行匹配
            if (introductionAwareMethodMatcher != null ?
                introductionAwareMethodMatcher.matches(method, targetClass, hasIntroductions) :
                methodMatcher.matches(method, targetClass)) {
                return true;
            }
        }
    }

    return false;
}
```

### createProxy(...)

```java
// AbstractAutoProxyCreator.class
protected Object createProxy(Class<?> beanClass, @Nullable String beanName,
      @Nullable Object[] specificInterceptors, TargetSource targetSource) {

   // 如果当前beanFactory实现了ConfigurableListableBeanFactory接口
   // 则将需要被代理的对象暴露出来？？？？意义不明
   if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
      AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
   }

   // 创建代理工厂
   ProxyFactory proxyFactory = new ProxyFactory();
   // 复制 proxyTargetClass，exposeProxy 等属性
   proxyFactory.copyFrom(this);

   // proxyTargetClass默认为false，使用的是Jdk代理来织入切面逻辑。
   if (!proxyFactory.isProxyTargetClass()) {
			// 如果当前设置了使用Jdk代理目标类，判断目标类是否设置了preserveTargetClass属性
      if (shouldProxyTargetClass(beanClass, beanName)) {
         // 设置了 preserveTargetClass 属性为 true，强制使用Cglib代理目标类
         proxyFactory.setProxyTargetClass(true);
      }
      else {
         // 判断目标类是否实现了相关代理接口(isConfigurationCallbackInterface()、isInternalLanguageInterface())，是则使用cglib代理
         evaluateProxyInterfaces(beanClass, proxyFactory);
      }
   }

   // 将需要织入的切面逻辑都转换为Advisor对象，specificInterceptors + interceptorNames 对应的bean
   Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
   proxyFactory.addAdvisors(advisors);
   proxyFactory.setTargetSource(targetSource);
   // 提供的hook钩子方法，供子类实现以实现对代理工厂的定制
   customizeProxyFactory(proxyFactory);

   proxyFactory.setFrozen(this.freezeProxy);
   // 当前判断逻辑默认返回false，子类可进行重写
   // 对于AnnotationAwareAspectJAutoProxyCreator，其重写了该方法返回true
   // 因为其已经对获取到的Advisor进行了过滤，后面不需要在对目标类进行重新匹配了
   if (advisorsPreFiltered()) {
      proxyFactory.setPreFiltered(true);
   }

   return proxyFactory.getProxy(getProxyClassLoader());
}
```

#### buildAdvisors()

确定bean的拦截器，包含入参提供的特有拦截器和内部属性定义的公共拦截器。

```java
// AbstractAutoProxyCreator.class
protected Advisor[] buildAdvisors(@Nullable String beanName, @Nullable Object[] specificInterceptors) {
   // Handle prototypes correctly...
   // 根据 interceptorNames 属性解析通用的拦截器，默认是空
   Advisor[] commonInterceptors = resolveInterceptorNames();

   // 将特有拦截器和公共拦截器放入同一个集合
   List<Object> allInterceptors = new ArrayList<>();
   if (specificInterceptors != null) {
      allInterceptors.addAll(Arrays.asList(specificInterceptors));
      if (commonInterceptors.length > 0) {
         if (this.applyCommonInterceptorsFirst) {
            allInterceptors.addAll(0, Arrays.asList(commonInterceptors));
         }
         else {
            allInterceptors.addAll(Arrays.asList(commonInterceptors));
         }
      }
   }
   if (logger.isTraceEnabled()) {
      int nrOfCommonInterceptors = commonInterceptors.length;
      int nrOfSpecificInterceptors = (specificInterceptors != null ? specificInterceptors.length : 0);
      logger.trace("Creating implicit proxy for bean '" + beanName + "' with " + nrOfCommonInterceptors +
            " common interceptors and " + nrOfSpecificInterceptors + " specific interceptors");
   }

   Advisor[] advisors = new Advisor[allInterceptors.size()];
   for (int i = 0; i < allInterceptors.size(); i++) {
      // 适配成 Advisor 对象
      advisors[i] = this.advisorAdapterRegistry.wrap(allInterceptors.get(i));
   }
   return advisors;
}
```

#### proxyFactory.getProxy()

创建代理对象。此处根据判断返回的是Jdk代理还是Cglib代理。

```java
// ProxyFactory.class
public Object getProxy(@Nullable ClassLoader classLoader) {
   // 首先获取AopProxy对象
   // 其主要有两个实现：JdkDynamicAopProxy 和 ObjenesisCglibAopProxy，ObjenesisCglibAopProxy是CglibAopProxy的子类
   // 分别用于Jdk和Cglib代理类的生成，其getProxy()方法则用于获取具体的代理对象
   return createAopProxy().getProxy(classLoader);
}

// ProxyCreatorSupport.class，ProxyFactory上级类，这里调用继承的方法
/**
 * aop代理类工厂
 * 默认在无参构造中实例化默认的实现类DefaultAopProxyFactory
 * @see #ProxyCreatorSupport() 
 */
private AopProxyFactory aopProxyFactory;

protected final synchronized AopProxy createAopProxy() {
    if (!this.active) {
        activate();
    }
    // 调用AopProxyFactory#createAopProxy()
    return getAopProxyFactory().createAopProxy(this);
}
public AopProxyFactory getAopProxyFactory() {
    return this.aopProxyFactory;
}
```

DefaultAopProxyFactory

```java
// DefaultAopProxyFactory.class，AopProxyFactory接口的默认实现类，是上面中aopProxyFactory属性的默认实例对象
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
    // 判断当前类是否需要进行运行时优化，或者是指定了使用Cglib代理的方式，再或者是目标类没有用户提供的相关接口，则使用Cglib代理实现代理逻辑的织入
    if (!IN_NATIVE_IMAGE &&
        (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config))) {
        Class<?> targetClass = config.getTargetClass();
        if (targetClass == null) {
            throw new AopConfigException("TargetSource cannot determine target class: " +
                                         "Either an interface or a target is required for proxy creation.");
        }
        // 如果被代理的类是一个接口，或者被代理的类是使用Jdk代理生成的类，此时还是使用Jdk代理
        if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
            return new JdkDynamicAopProxy(config);
        }
        return new ObjenesisCglibAopProxy(config);
    }
    else {
        return new JdkDynamicAopProxy(config);
    }
}
```



# 二、代理类解析

## Cglib

### getProxy()

```java
// CglibAopProxy.class，ObjenesisCglibAopProxy的父类
protected final AdvisedSupport advised;

public Object getProxy(@Nullable ClassLoader classLoader) {
   if (logger.isTraceEnabled()) {
      logger.trace("Creating CGLIB proxy: " + this.advised.getTargetSource());
   }

   try {
      Class<?> rootClass = this.advised.getTargetClass();
      Assert.state(rootClass != null, "Target class must be available for creating a CGLIB proxy");

      // 判断当前类是否是已经通过Cglib代理生成的类，如果是的，则获取其原始父类，
      // 并将其接口设置到需要代理的接口中
      Class<?> proxySuperClass = rootClass;
      if (rootClass.getName().contains(ClassUtils.CGLIB_CLASS_SEPARATOR)) {
         proxySuperClass = rootClass.getSuperclass();
         Class<?>[] additionalInterfaces = rootClass.getInterfaces();
         for (Class<?> additionalInterface : additionalInterfaces) {
            this.advised.addInterface(additionalInterface);
         }
      }

      // 对目标类的方法进行检查，并递归检查父类直到Object.class
      // 该方法仅仅是检查并打印出不能代理的方法，不会影响代理的创建
      // 1. 目标方法不能使用final修饰；
      // 2. 目标方法必须是public或protect修饰的；
      // 满足全部条件则当前方法才能代理，不能代理的方法会被略过
      // Validate the class, writing log messages as necessary.
      validateClassIfNecessary(proxySuperClass, classLoader);

      // Configure CGLIB Enhancer...
      Enhancer enhancer = createEnhancer();
      if (classLoader != null) {
         enhancer.setClassLoader(classLoader);
         if (classLoader instanceof SmartClassLoader &&
               ((SmartClassLoader) classLoader).isClassReloadable(proxySuperClass)) {
            enhancer.setUseCache(false);
         }
      }
      enhancer.setSuperclass(proxySuperClass);
      // 这里AopProxyUtils.completeProxiedInterfaces()方法的主要目的是为要生成的代理类
      // 增加SpringProxy，Advised，DecoratingProxy三个需要实现的接口。这里三个接口的作用如下：
      // 1. SpringProxy：是一个空接口，用于标记当前生成的代理类是Spring生成的代理类；
      // 2. Advised：Spring生成代理类所使用的属性都保存在该接口中，
      //    包括Advisor，Advice和其他相关属性；
      // 3. DecoratingProxy：该接口用于获取当前代理对象所代理的目标对象的Class类型。
      enhancer.setInterfaces(AopProxyUtils.completeProxiedInterfaces(this.advised));
      enhancer.setNamingPolicy(SpringNamingPolicy.INSTANCE);
      enhancer.setStrategy(new ClassLoaderAwareGeneratorStrategy(classLoader));

      // 获取当前需要织入到代理类中的逻辑
      Callback[] callbacks = getCallbacks(rootClass);
      Class<?>[] types = new Class<?>[callbacks.length];
      for (int x = 0; x < types.length; x++) {
         types[x] = callbacks[x].getClass();
      }
      // 设置代理类中各个方法将要使用的切面逻辑，这里ProxyCallbackFilter.accept()方法返回
      // 的整型值正好一一对应上面Callback数组中各个切面逻辑的下标，也就是说这里的CallbackFilter
      // 的作用正好指定了代理类中各个方法将要使用Callback数组中的哪个或哪几个切面逻辑
      // fixedInterceptorMap only populated at this point, after getCallbacks call above
      enhancer.setCallbackFilter(new ProxyCallbackFilter(
            this.advised.getConfigurationOnlyCopy(), this.fixedInterceptorMap, this.fixedInterceptorOffset));
      enhancer.setCallbackTypes(types);

      // Generate the proxy class and create a proxy instance.
      return createProxyClassAndInstance(enhancer, callbacks);
   }
   catch (CodeGenerationException | IllegalArgumentException ex) {
      throw new AopConfigException("Could not generate CGLIB subclass of " + this.advised.getTargetClass() +
            ": Common causes of this problem include using a final class or a non-visible class",
            ex);
   }
   catch (Throwable ex) {
      // TargetSource.getTarget() failed
      throw new AopConfigException("Unexpected AOP exception", ex);
   }
}

protected Enhancer createEnhancer() {
    return new Enhancer();
}

protected Object createProxyClassAndInstance(Enhancer enhancer, Callback[] callbacks) {
    enhancer.setInterceptDuringConstruction(false);
    enhancer.setCallbacks(callbacks);
    return (this.constructorArgs != null && this.constructorArgTypes != null ?
            enhancer.create(this.constructorArgTypes, this.constructorArgs) :
            enhancer.create());
}
```

`ObjenesisCglibAopProxy`主要做了代理的创建工作，包含cglib代理类的核心参数callbacks和callbackFilter。

#### getCallbacks()

核心的织入逻辑在getCallbacks()中初始化的Callback对象数组。该部分内容仅说明获取callbacks的过程和callbacks中包含的Callback，具体Callback的作用和实现参照下文ProxyCallbackFilter的路由逻辑。

```java
private Callback[] getCallbacks(Class<?> rootClass) throws Exception {
   // Parameters used for optimization choices...
   boolean exposeProxy = this.advised.isExposeProxy();
   boolean isFrozen = this.advised.isFrozen();
   boolean isStatic = this.advised.getTargetSource().isStatic();

   // 用户自定义的代理逻辑的主要织入类
   // Choose an "aop" interceptor (used for AOP calls).
   Callback aopInterceptor = new DynamicAdvisedInterceptor(this.advised);

   // Choose a "straight to target" interceptor. (used for calls that are
   // unadvised but can return this). May be required to expose the proxy.
   // 判断如果要暴露代理对象，如果是，则使用AopContext设置将代理对象设置到ThreadLocal中
   // 用户则可以通过AopContext获取目标对象
   Callback targetInterceptor;
   if (exposeProxy) {
      // 判断被代理的对象是否是静态的，如果是静态的，则将目标对象缓存起来，每次都使用该对象即可，
      // 如果目标对象是动态的，则在DynamicUnadvisedExposedInterceptor中每次都生成一个新的
      // 目标对象，以织入后面的代理逻辑
      targetInterceptor = (isStatic ?
            new StaticUnadvisedExposedInterceptor(this.advised.getTargetSource().getTarget()) :
            new DynamicUnadvisedExposedInterceptor(this.advised.getTargetSource()));
   }
   else {
      // 下面两个类与上面两个的唯一区别就在于是否使用AopContext暴露生成的代理对象
      targetInterceptor = (isStatic ?
            new StaticUnadvisedInterceptor(this.advised.getTargetSource().getTarget()) :
            new DynamicUnadvisedInterceptor(this.advised.getTargetSource()));
   }

   // Choose a "direct to target" dispatcher (used for
   // unadvised calls to static targets that cannot return this).
   Callback targetDispatcher = (isStatic ?
         new StaticDispatcher(this.advised.getTargetSource().getTarget()) : new SerializableNoOp());

   // 当前Callback用于一般的不用被代理的方法，这些方法将获取到的callback组装为一个数组
   Callback[] mainCallbacks = new Callback[] {
         aopInterceptor,  // for normal advice
         targetInterceptor,  // invoke target without considering advice, if optimized
         new SerializableNoOp(),  // no override for methods mapped to this
         targetDispatcher, this.advisedDispatcher,
         new EqualsInterceptor(this.advised),
         new HashCodeInterceptor(this.advised)
   };

   Callback[] callbacks;

   // If the target is a static one and the advice chain is frozen,
   // then we can make some optimizations by sending the AOP calls
   // direct to the target using the fixed chain for that method.
   // 如果目标对象是静态的，也即可以缓存的，并且切面逻辑的调用链是固定的，
   // 则对目标对象和整个调用链进行缓存
   if (isStatic && isFrozen) {
      Method[] methods = rootClass.getMethods();
      Callback[] fixedCallbacks = new Callback[methods.length];
      this.fixedInterceptorMap = new HashMap<>(methods.length);

      // TODO: small memory optimization here (can skip creation for methods with no advice)
      for (int x = 0; x < methods.length; x++) {
         // 获取目标对象的切面逻辑
         Method method = methods[x];
         List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, rootClass);
         fixedCallbacks[x] = new FixedChainStaticTargetInterceptor(
               chain, this.advised.getTargetSource().getTarget(), this.advised.getTargetClass());
         // 对调用链进行缓存
         this.fixedInterceptorMap.put(method, x);
      }

      // Now copy both the callbacks from mainCallbacks
      // and fixedCallbacks into the callbacks array.
      // 将生成的静态调用链存入Callback数组中
      callbacks = new Callback[mainCallbacks.length + fixedCallbacks.length];
      System.arraycopy(mainCallbacks, 0, callbacks, 0, mainCallbacks.length);
      System.arraycopy(fixedCallbacks, 0, callbacks, mainCallbacks.length, fixedCallbacks.length);
      // 这里fixedInterceptorOffset记录了当前静态的调用链的切面逻辑的起始位置，
      // 这里记录的用处在于后面使用CallbackFilter的时候，如果发现是静态的调用链，
      // 则直接通过该参数获取相应的调用链，而直接略过了前面的动态调用链
      this.fixedInterceptorOffset = mainCallbacks.length;
   }
   else {
      callbacks = mainCallbacks;
   }
   return callbacks;
}
```



#### ProxyCallbackFilter

先来看cglib代理中CallbackFilter接口的实现ProxyCallbackFilter，CglibAopProxy的内部类。CallbackFilter接口accpet()的返回值决定了代理类实际执行Callback。

```java
// CglibAopProxy.class
/**
 * {@link DynamicAdvisedInterceptor}
 */
private static final int AOP_PROXY = 0;
/**
 * exposeProxy、isStatic
 * {@link StaticUnadvisedExposedInterceptor}/{@link DynamicUnadvisedExposedInterceptor}{@link StaticUnadvisedInterceptor}/{@link DynamicUnadvisedInterceptor}
 */
private static final int INVOKE_TARGET = 1;
/**
 * {@link SerializableNoOp}
 */
private static final int NO_OVERRIDE = 2;
/**
 * Callback targetDispatcher = (isStatic ? new StaticDispatcher(this.advised.getTargetSource().getTarget()) : new SerializableNoOp());
 * {@link StaticDispatcher}/{@link SerializableNoOp}
 */
private static final int DISPATCH_TARGET = 3;
/**
 * {@link AdvisedDispatcher}
 */
private static final int DISPATCH_ADVISED = 4;
/**
 * {@link EqualsInterceptor}
 */
private static final int INVOKE_EQUALS = 5;
/**
 * {@link HashCodeInterceptor}
 */
private static final int INVOKE_HASHCODE = 6;

// ProxyCallbackFilter.class
public int accept(Method method) {
    if (AopUtils.isFinalizeMethod(method)) {
        logger.trace("Found finalize() method - using NO_OVERRIDE");
        // 2: Object#finalize()回收方法不执行aop
        return NO_OVERRIDE;
    }
    if (!this.advised.isOpaque() && method.getDeclaringClass().isInterface() &&
        method.getDeclaringClass().isAssignableFrom(Advised.class)) {
        if (logger.isTraceEnabled()) {
            logger.trace("Method is declared on Advised interface: " + method);
        }
        // 4: Advised接口的方法，防止Advised对象被Advised切面环绕
        return DISPATCH_ADVISED;
    }
    // We must always proxy equals, to direct calls to this.
    if (AopUtils.isEqualsMethod(method)) {
        if (logger.isTraceEnabled()) {
            logger.trace("Found 'equals' method: " + method);
        }
        // 5: 代理equals()
        return INVOKE_EQUALS;
    }
    // We must always calculate hashCode based on the proxy.
    if (AopUtils.isHashCodeMethod(method)) {
        if (logger.isTraceEnabled()) {
            logger.trace("Found 'hashCode' method: " + method);
        }
        // 6: 代理hashCode()
        return INVOKE_HASHCODE;
    }
    Class<?> targetClass = this.advised.getTargetClass();
    // Proxy is not yet available, but that shouldn't matter.
    List<?> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
    boolean haveAdvice = !chain.isEmpty();
    // TODO Charlotte: 2020/12/23 exposeProxy属性什么意思？什么作用
    boolean exposeProxy = this.advised.isExposeProxy();
    // TODO Charlotte: 2020/12/23 isStatic属性什么意思
    boolean isStatic = this.advised.getTargetSource().isStatic();
    // TODO Charlotte: 2020/12/23 isFrozen属性什么意思
    boolean isFrozen = this.advised.isFrozen();
    if (haveAdvice || !isFrozen) {
        // If exposing the proxy, then AOP_PROXY must be used.
        // 存在切面或者非冻结配置
        if (exposeProxy) {
            if (logger.isTraceEnabled()) {
                logger.trace("Must expose proxy on advised method: " + method);
            }
            // 0: 暴露，设置切面
            return AOP_PROXY;
        }
        // Check to see if we have fixed interceptor to serve this method.
        // Else use the AOP_PROXY.
        if (isStatic && isFrozen && this.fixedInterceptorMap.containsKey(method)) {
            // TODO Charlotte: 2020/12/23 isStatic && isFrozen的意义
            // 固定的Callback映射集合fixedInterceptorMap，在getCallbacks()方法中缓存过
            if (logger.isTraceEnabled()) {
                logger.trace("Method has advice and optimizations are enabled: " + method);
            }
            // We know that we are optimizing so we can use the FixedStaticChainInterceptors.
            int index = this.fixedInterceptorMap.get(method);
            return (index + this.fixedInterceptorOffset);
        }
        else {
            if (logger.isTraceEnabled()) {
                logger.trace("Unable to apply any optimizations to advised method: " + method);
            }
            // 0: 默认切面
            return AOP_PROXY;
        }
    }
    else {
        // 无advice且冻结，不调用AOP(0)
        // See if the return type of the method is outside the class hierarchy of the target type.
        // If so we know it never needs to have return type massage and can use a dispatcher.
        // If the proxy is being exposed, then must use the interceptor the correct one is already
        // configured. If the target is not static, then we cannot use a dispatcher because the
        // target needs to be explicitly released after the invocation.
        if (exposeProxy || !isStatic) {
            // 1: 暴露且为静态类，
            return INVOKE_TARGET;
        }
        Class<?> returnType = method.getReturnType();
        if (targetClass != null && returnType.isAssignableFrom(targetClass)) {
            if (logger.isTraceEnabled()) {
                logger.trace("Method return type is assignable from target type and " +
                             "may therefore return 'this' - using INVOKE_TARGET: " + method);
            }
            // 1: 自旋
            return INVOKE_TARGET;
        }
        else {
            if (logger.isTraceEnabled()) {
                logger.trace("Method return type ensures 'this' cannot be returned - " +
                             "using DISPATCH_TARGET: " + method);
            }
            // 3: 
            return DISPATCH_TARGET;
        }
    }
}
```

从分支大致能够看出来，haveAdvice即存在切面时，大多数条件下返回的为AOP_PROXY，即0，回顾上文getCallbacks()的返回值，对应为`DynamicAdvisedInterceptor`。

#### DynamicAdvisedInterceptor

CglibAopProxy的内部类，实现了cglib的MethodInterceptor接口，AOP代理的织入代理逻辑的核心Callback。

```java
private static class DynamicAdvisedInterceptor implements MethodInterceptor, Serializable {

   private final AdvisedSupport advised;

   public DynamicAdvisedInterceptor(AdvisedSupport advised) {
      this.advised = advised;
   }

   @Override
   @Nullable
   public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
      Object oldProxy = null;
      boolean setProxyContext = false;
      Object target = null;
      // 通过TargetSource获取目标对象
      TargetSource targetSource = this.advised.getTargetSource();
      try {
         // 判断如果需要暴露代理对象，则将当前代理对象设置到ThreadLocal中
         if (this.advised.exposeProxy) {
            // Make invocation available if necessary.
            oldProxy = AopContext.setCurrentProxy(proxy);
            setProxyContext = true;
         }
         // Get as late as possible to minimize the time we "own" the target, in case it comes from a pool...
         target = targetSource.getTarget();
         Class<?> targetClass = (target != null ? target.getClass() : null);
         // 获取目标对象切面逻辑的环绕链
         List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
         Object retVal;
         // Check whether we only have one InvokerInterceptor: that is,
         // no real advice, but just reflective invocation of the target.
         if (chain.isEmpty() && Modifier.isPublic(method.getModifiers())) {
            // <1>chain.isEmpty()，即AOP_PROXY对应的Callback不存在切面，即CallbackFilter调用中 AdvisedSupport#isFrozen()为真
            // We can skip creating a MethodInvocation: just invoke the target directly.
            // Note that the final invoker must be an InvokerInterceptor, so we know
            // it does nothing but a reflective operation on the target, and no hot
            // swapping or fancy proxying.
            // 对参数进行处理，以使其与目标方法的参数类型一致，尤其对于数组类型，
            // 会单独处理其数据类型与实际类型一致
            Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
            retVal = methodProxy.invoke(target, argsToUse);
         }
         else {
            // <2>AOP代理的实际执行类，通过生成的调用链，对目标方法进行环绕调用
            // We need to create a method invocation...
            retVal = new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed();
         }
         // 对返回值进行处理，如果返回值就是当前目标对象，那么将代理生成的代理对象返回；
         // 如果返回值为空，并且返回值类型是非void的基本数据类型，则抛出异常；
         // 如果上述两个条件都不符合，则直接将生成的返回值返回
         retVal = processReturnType(proxy, target, method, retVal);
         return retVal;
      }
      finally {
         // 如果目标对象不是静态的，则调用TargetSource.releaseTarget()方法释放目标对象
         if (target != null && !targetSource.isStatic()) {
            targetSource.releaseTarget(target);
         }
         // 将代理对象设置为前面（外层逻辑）调用设置的对象，以防止暴露出来的代理对象不一致
         if (setProxyContext) {
            // Restore old proxy.
            AopContext.setCurrentProxy(oldProxy);
         }
      }
   }

   @Override
   public boolean equals(@Nullable Object other) {
      return (this == other ||
            (other instanceof DynamicAdvisedInterceptor &&
                  this.advised.equals(((DynamicAdvisedInterceptor) other).advised)));
   }

   /**
    * CGLIB uses this to drive proxy creation.
    */
   @Override
   public int hashCode() {
      return this.advised.hashCode();
   }
}

// AdvisedSupport.class，实现Advised接口
private transient Map<MethodCacheKey, List<Object>> methodCache;

public List<Object> getInterceptorsAndDynamicInterceptionAdvice(Method method, @Nullable Class<?> targetClass) {
    MethodCacheKey cacheKey = new MethodCacheKey(method);
    List<Object> cached = this.methodCache.get(cacheKey);
    if (cached == null) {
        // 获取advice调用链
        cached = this.advisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice(
            this, method, targetClass);
        this.methodCache.put(cacheKey, cached);
    }
    return cached;
}

// DefaultAdvisorChainFactory.class，接口AdvisorChainFactory的默认实现类
@Override
public List<Object> getInterceptorsAndDynamicInterceptionAdvice(
    Advised config, Method method, @Nullable Class<?> targetClass) {

    // This is somewhat tricky... We have to process introductions first,
    // but we need to preserve order in the ultimate list.
    AdvisorAdapterRegistry registry = GlobalAdvisorAdapterRegistry.getInstance();
    Advisor[] advisors = config.getAdvisors();
    List<Object> interceptorList = new ArrayList<>(advisors.length);
    Class<?> actualClass = (targetClass != null ? targetClass : method.getDeclaringClass());
    Boolean hasIntroductions = null;

    for (Advisor advisor : advisors) {
        if (advisor instanceof PointcutAdvisor) {
            // Add it conditionally.
            PointcutAdvisor pointcutAdvisor = (PointcutAdvisor) advisor;
            // 这里判断切面逻辑的调用链是否提前进行过过滤，如果进行过，则不再进行目标方法的匹配，
            // 如果没有，则再进行一次匹配。
            if (config.isPreFiltered() || pointcutAdvisor.getPointcut().getClassFilter().matches(actualClass)) {
                // 未过滤过且满足pointcut匹配条件的
                // 这里我们使用的AnnotationAwareAspectJAutoProxyCreator在生成切面逻辑的时候就已经进行了过滤，因而这里返回的是true，本文最开始也对这里进行了讲解
                MethodMatcher mm = pointcutAdvisor.getPointcut().getMethodMatcher();
                boolean match;
                // 这里进行匹配的时候，首先会检查是否为IntroductionAwareMethodMatcher类型的
                // Matcher，如果是，则调用其定义的matches()方法进行匹配，如果不是，则直接调用
                // 当前切面的matches()方法进行匹配。这里由于前面进行匹配时可能存在部分在静态匹配时
                // 无法确认的方法匹配结果，因而这里调用是必要的，而对于能够确认的匹配逻辑，这里调用
                // 也是非常迅速的，因为前面已经对匹配结果进行了缓存
                if (mm instanceof IntroductionAwareMethodMatcher) {
                    if (hasIntroductions == null) {
                        // 判断切面逻辑中是否有IntroductionAdvisor类型的Advisor
                        hasIntroductions = hasMatchingIntroductions(advisors, actualClass);
                    }
                    match = ((IntroductionAwareMethodMatcher) mm).matches(method, actualClass, hasIntroductions);
                }
                else {
                    match = mm.matches(method, actualClass);
                }
                if (match) {
                    // 将Advisor对象转换为MethodInterceptor数组
                    MethodInterceptor[] interceptors = registry.getInterceptors(advisor);
                    if (mm.isRuntime()) {
                        // Creating a new object instance in the getInterceptors() method
                        // isn't a problem as we normally cache created chains.
                        // 判断如果是动态匹配，则使用InterceptorAndDynamicMethodMatcher对其进行封装
                        for (MethodInterceptor interceptor : interceptors) {
                            interceptorList.add(new InterceptorAndDynamicMethodMatcher(interceptor, mm));
                        }
                    }
                    else {
                        // 如果是静态匹配，则直接将调用链返回
                        interceptorList.addAll(Arrays.asList(interceptors));
                    }
                }
            }
        }
        else if (advisor instanceof IntroductionAdvisor) {
            IntroductionAdvisor ia = (IntroductionAdvisor) advisor;
            // 判断如果为IntroductionAdvisor类型的Advisor，则将调用链封装为Interceptor数组
            if (config.isPreFiltered() || ia.getClassFilter().matches(actualClass)) {
                Interceptor[] interceptors = registry.getInterceptors(advisor);
                interceptorList.addAll(Arrays.asList(interceptors));
            }
        }
        else {
            // 这里是提供的使用自定义的转换器对Advisor进行转换的逻辑，因为getInterceptors()方法中
            // 会使用相应的Adapter对目标Advisor进行匹配，如果能匹配上，通过其getInterceptor()方法
            // 将自定义的Advice转换为MethodInterceptor对象
            Interceptor[] interceptors = registry.getInterceptors(advisor);
            interceptorList.addAll(Arrays.asList(interceptors));
        }
    }

    return interceptorList;
}

/**
	 * Determine whether the Advisors contain matching introductions.
	 */
private static boolean hasMatchingIntroductions(Advisor[] advisors, Class<?> actualClass) {
    for (Advisor advisor : advisors) {
        if (advisor instanceof IntroductionAdvisor) {
            IntroductionAdvisor ia = (IntroductionAdvisor) advisor;
            if (ia.getClassFilter().matches(actualClass)) {
                return true;
            }
        }
    }
    return false;
}

// DefaultAdvisorAdapterRegistry.class
public MethodInterceptor[] getInterceptors(Advisor advisor) throws UnknownAdviceTypeException {
    List<MethodInterceptor> interceptors = new ArrayList<>(3);
    Advice advice = advisor.getAdvice();
    if (advice instanceof MethodInterceptor) {
        interceptors.add((MethodInterceptor) advice);
    }
    for (AdvisorAdapter adapter : this.adapters) {
        if (adapter.supportsAdvice(advice)) {
            interceptors.add(adapter.getInterceptor(advisor));
        }
    }
    if (interceptors.isEmpty()) {
        throw new UnknownAdviceTypeException(advisor.getAdvice());
    }
    return interceptors.toArray(new MethodInterceptor[0]);
}
```

#### CglibMethodInvocation

AOP切面最终的执行类。Joinpoint接口表示运行时的连接点（AOP术语），**封装了SpringAop中切面行为**如`getTarget()`、`getArgs()`等接口。一般用于切面注解`@Aspect`等标注的切面方法入参。

![image-20201229104602177](C:\Users\Charlotte\AppData\Roaming\Typora\typora-user-images\image-20201229104602177.png)![image-20201229112922733](C:\Users\Charlotte\AppData\Roaming\Typora\typora-user-images\image-20201229112922733.png)

```java
private static class CglibMethodInvocation extends ReflectiveMethodInvocation {

   @Nullable
   private final MethodProxy methodProxy;

   public CglibMethodInvocation(Object proxy, @Nullable Object target, Method method,
         Object[] arguments, @Nullable Class<?> targetClass,
         List<Object> interceptorsAndDynamicMethodMatchers, MethodProxy methodProxy) {

      super(proxy, target, method, arguments, targetClass, interceptorsAndDynamicMethodMatchers);

      // Only use method proxy for public methods not derived from java.lang.Object
      this.methodProxy = (Modifier.isPublic(method.getModifiers()) &&
            method.getDeclaringClass() != Object.class && !AopUtils.isEqualsMethod(method) &&
            !AopUtils.isHashCodeMethod(method) && !AopUtils.isToStringMethod(method) ?
            methodProxy : null);
   }

   @Override
   @Nullable
   public Object proceed() throws Throwable {
      try {
         return super.proceed();
      }
      catch (RuntimeException ex) {
         throw ex;
      }
      catch (Exception ex) {
         if (ReflectionUtils.declaresException(getMethod(), ex.getClass())) {
            throw ex;
         }
         else {
            throw new UndeclaredThrowableException(ex);
         }
      }
   }

   /**
    * Gives a marginal performance improvement versus using reflection to
    * invoke the target when invoking public methods.
    */
   @Override
   protected Object invokeJoinpoint() throws Throwable {
      if (this.methodProxy != null) {
         return this.methodProxy.invoke(this.target, this.arguments);
      }
      else {
         return super.invokeJoinpoint();
      }
   }
}
```





## Jdk

```java
// JdkDynamicAopProxy.class
public Object getProxy(@Nullable ClassLoader classLoader) {
   if (logger.isTraceEnabled()) {
      logger.trace("Creating JDK dynamic proxy: " + this.advised.getTargetSource());
   }
   Class<?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised, true);
   findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
   return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
}
```







# 三、注册AnnotationAwareAspectJAutoProxyCreator

启用AOP时会向容器中注册`AnnotationAwareAspectJAutoProxyCreator`。

> todo
>
> AnnotationAwareAspectJAutoProxyCreator继承了AbstractAdvisorAutoProxyCreator

通过工具类`AopConfigUtils#registerAspectJAnnotationAutoProxyCreatorIfNecessary(...)`进行注册，调用节点后文说明。

```java
	@Nullable
	public static BeanDefinition registerAspectJAnnotationAutoProxyCreatorIfNecessary(BeanDefinitionRegistry registry) {
		return registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry, null);
	}

	@Nullable
	public static BeanDefinition registerAspectJAnnotationAutoProxyCreatorIfNecessary(
			BeanDefinitionRegistry registry, @Nullable Object source) {

		return registerOrEscalateApcAsRequired(AnnotationAwareAspectJAutoProxyCreator.class, registry, source);
	}
```

`#registerAspectJAnnotationAutoProxyCreatorIfNecessary(...)`方法通过重载一律调用了`#registerOrEscalateApcAsRequired(...)`方法注册class。

## 注册或升级AspectJAwareAdvisorAutoProxyCreator

方法`#registerOrEscalateApcAsRequired(...)`

注册的beanName统一为`#AUTO_PROXY_CREATOR_BEAN_NAME`常量`org.springframework.aop.config.internalAutoProxyCreator`

<1>当beanName存在时，若不为同一个class则根据内部配置的class优先级进行覆盖。AnnotationAwareAspectJAutoProxyCreator在此流程中为最高优先级。优先级判断方法下文接<1.1>。

<2>当beanName在registry中不存在时，配置并注册。

```java
@Nullable
private static BeanDefinition registerOrEscalateApcAsRequired(
      Class<?> cls, BeanDefinitionRegistry registry, @Nullable Object source) {

   Assert.notNull(registry, "BeanDefinitionRegistry must not be null");

   if (registry.containsBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME)) {
      // <1>如果已存在 beanName = #AUTO_PROXY_CREATOR_BEAN_NAME
      BeanDefinition apcDefinition = registry.getBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME);
      if (!cls.getName().equals(apcDefinition.getBeanClassName())) {
         // class 不同，判断优先级 priority
         int currentPriority = findPriorityForClass(apcDefinition.getBeanClassName());
         int requiredPriority = findPriorityForClass(cls);
         if (currentPriority < requiredPriority) {
            // 优先级更大，覆盖 BeanDefinition#beanClassName
            apcDefinition.setBeanClassName(cls.getName());
         }
      }
      return null;
   }

   // <2>不存在，注册
   // 设置其执行优先级为最高优先级，并且标识该 bean 为 Spring 的系统 Bean ，设置完之后则对该 bean 进行注册
   RootBeanDefinition beanDefinition = new RootBeanDefinition(cls);
   beanDefinition.setSource(source);
   beanDefinition.getPropertyValues().add("order", Ordered.HIGHEST_PRECEDENCE);
   beanDefinition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
   registry.registerBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME, beanDefinition);
   return beanDefinition;
}
```

<1.1>优先级判断方式。

```java
private static final List<Class<?>> APC_PRIORITY_LIST = new ArrayList<>(3);

static {
    // Set up the escalation list...
    // 后入为重
    APC_PRIORITY_LIST.add(InfrastructureAdvisorAutoProxyCreator.class);
    APC_PRIORITY_LIST.add(AspectJAwareAdvisorAutoProxyCreator.class);
    APC_PRIORITY_LIST.add(AnnotationAwareAspectJAutoProxyCreator.class);
}

private static int findPriorityForClass(@Nullable String className) {
   for (int i = 0; i < APC_PRIORITY_LIST.size(); i++) {
      Class<?> clazz = APC_PRIORITY_LIST.get(i);
      if (clazz.getName().equals(className)) {
         return i;
      }
   }
   throw new IllegalArgumentException(
         "Class name [" + className + "] is not a known auto-proxy creator class");
}
```

## 注册方式

1. xml配置

   `AopNamespaceHandler#inint(...)`初始化方法中向namespace中注册。

   ```java
   public class AopNamespaceHandler extends NamespaceHandlerSupport {
      @Override
      public void init() {
         ...
         /**
          * http\://www.springframework.org/schema/aop=org.springframework.aop.config.AopNamespaceHandler
          * <aop:aspectj-autoproxy/>
          */
         registerBeanDefinitionParser("aspectj-autoproxy", new AspectJAutoProxyBeanDefinitionParser());
         ...
      }
   
   }
   ```

   `AspectJAutoProxyBeanDefinitionParser`类实现了`BeanDefinitionParser`接口的`#parse(...)`方法，在`#parse(...)`方法中调用`AopConfigUtils`注册`AnnotationAwareAspectJAutoProxyCreator`，并配置相关属性。

2. 注解启用

   在`spring bean`上加入注解即启用配置。重点在于注解`@Import`导入了`AspectJAutoProxyRegistrar`

   ```java
   @Target(ElementType.TYPE)
   @Retention(RetentionPolicy.RUNTIME)
   @Documented
   @Import(AspectJAutoProxyRegistrar.class)
   public @interface EnableAspectJAutoProxy {
       ...
   }
   ```

   `AspectJAutoProxyRegistrar`实现了`ImportBeanDefinitionRegistrar`接口的`#registerBeanDefinitions(...)`方法，在方法中做了与`1`类似的操作。

3. autoConfiguration

   在`spring boot`中的`aop`自动配置。

   `@ConditionalOnProperty#matchIfMissing`属性为当缺少配置时是否加载，默认`false`。

   在代码中可以看到，该类仅仅是根据配置项自动使用了上一条注册方式的注解`@EnableAspectJAutoProxy`。

   根据<1><2>，spring boot无配置时默认是启用`aop`并由`cglib`代理的。

   ```java
   @Configuration(proxyBeanMethods = false)
   // <1>无spring.aop.auto配置或为true时默认生效
   @ConditionalOnProperty(prefix = "spring.aop", name = "auto", havingValue = "true", matchIfMissing = true)
   public class AopAutoConfiguration {
   
      @Configuration(proxyBeanMethods = false)
      @ConditionalOnClass(Advice.class)
      static class AspectJAutoProxyingConfiguration {
   
         @Configuration(proxyBeanMethods = false)
         @EnableAspectJAutoProxy(proxyTargetClass = false)
         @ConditionalOnProperty(prefix = "spring.aop", name = "proxy-target-class", havingValue = "false",
               matchIfMissing = false)
         static class JdkDynamicAutoProxyConfiguration {
   
         }
   
         @Configuration(proxyBeanMethods = false)
         // <2>默认 cglib 代理 aop
         @EnableAspectJAutoProxy(proxyTargetClass = true)
         @ConditionalOnProperty(prefix = "spring.aop", name = "proxy-target-class", havingValue = "true",
               matchIfMissing = true)
         static class CglibAutoProxyConfiguration {
   
         }
   
      }
      ...
   }
   ```

# 草稿

## BeanPostProcessor

Spring bean加载中的前置后置处理

```java
public interface BeanPostProcessor {

	@Nullable
	default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}

	@Nullable
	default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}

}
```

## BeanDefinitionRegistyPostProcessor

```java
public interface BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor {

   void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException;

}
```



# TODO

## IntroductionAdvisor、PointcutAdvisor
## MethodMatcher

## AdvisedSupport

## MethodInterceptor与Callback的关系

## AOP调用链中的类InterceptorAndDynamicMethodMatcher

# 提问

- 为什么`@Sync`、`@Transactional`等注解加在私有方法时无法生效

> 因为这些基于AOP的注解都是通过Spring代理实现的，而无论是Cglib还是jdk均无法代理私有方法

- 为什么自身调用`@Sync`、`@Transactional`等注解的public方法时无法生效

> 与AOP代理同理，只会对从外部调用生效，而内部调用是`this`，`this.method()`直接指向了对应方法的内存地址，不经过代理
