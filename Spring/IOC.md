# 前言

文中spring boot版本为**2.5.0**，对应spring版本**5.3.7**。视spring版本代码可能会有差异

## 实例化

## 初始化

AbstractAutowireCapableBeanFactory

责任链模式

```java
@Nullable
protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
    Object bean = null;
    if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
        // Make sure bean class is actually resolved at this point.
        if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
            Class<?> targetType = determineTargetType(beanName, mbd);
            if (targetType != null) {
                // 实例化前置处理，若在此处返回bean，则会导致bean的创建短路。即用返回的bean代替bean的创建
                bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
                if (bean != null) {
                    // 若返回了自定义的实例化bean，进行bean初始化的后置处理，此处并不是后置处理的唯一调用
                    bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
                }
            }
        }
        mbd.beforeInstantiationResolved = (bean != null);
    }
    return bean;
}
```





```java
@Nullable
protected Object applyBeanPostProcessorsBeforeInstantiation(Class<?> beanClass, String beanName) {
    for (InstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().instantiationAware) {
        //InstantiationAwareBeanPostProcessor是BeanPostProcessor的子接口
        // 用于使spring创建bean的过程短路，使用自定义创建的bean代替spring bean
        Object result = bp.postProcessBeforeInstantiation(beanClass, beanName);
        if (result != null) {
            return result;
        }
    }
    return null;
}
```





```java
@Override
public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
      throws BeansException {

    Object result = existingBean;
    for (BeanPostProcessor processor : getBeanPostProcessors()) {
        // 初始化后置处理
        Object current = processor.postProcessAfterInitialization(result, beanName);
        if (current == null) {
            // 返回null则代表结束，返回上一次的result
            return result;
        }
        // 修改 result
        result = current;
    }
    return result;
}
```

### BeanPostProcessor

`BeanPostProcessor`接口，为Factory提供了对bean初始化前后进行处理的hook。调用时bean已经实例化。

- `#postProcessBeforeInitialization()`，bean的初始化前置处理，用于标记各类接口，填充属性，诸如`Aware`的各类子接口。后文举例说明。

- `#postProcessAfterInitialization()`，bean的初始化后置处理，通常用于一层层的代理包装原bean。

  默认实现均为返回原bean，不对bean进行处理，不打断流程，若返回null则中断处理链路，继续创建bean的后续流程。

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
阅读`BeanPostProcessor`的几个实现类方便理解。

#### ApplicationListenerDetector

`ApplicationListenerDetector`是`ApplicationListener`监听器的探测器。

- `#postProcessAfterInitialization()`，当bean实现了`ApplicationListener`接口且是单例时即视为监听器，将bean加入到Spring上下文对象ApplicationContext的监听器集合中。

```java
class ApplicationListenerDetector implements DestructionAwareBeanPostProcessor, MergedBeanDefinitionPostProcessor {

    private final transient Map<String, Boolean> singletonNames = new ConcurrentHashMap<>(256);

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        if (bean instanceof ApplicationListener) {
            // potentially not detected as a listener by getBeanNamesForType retrieval
            Boolean flag = this.singletonNames.get(beanName);
            if (Boolean.TRUE.equals(flag)) {
                // singleton bean (top-level or inner): register on the fly
                this.applicationContext.addApplicationListener((ApplicationListener<?>) bean);
            }
            else if (Boolean.FALSE.equals(flag)) {
                if (logger.isWarnEnabled() && !this.applicationContext.containsBean(beanName)) {
                    // inner bean with other scope - can't reliably process events
                    logger.warn("Inner bean '" + beanName + "' implements ApplicationListener interface " +
                                "but is not reachable for event multicasting by its containing ApplicationContext " +
                                "because it does not have singleton scope. Only top-level listener beans are allowed " +
                                "to be of non-singleton scope.");
                }
                this.singletonNames.remove(beanName);
            }
        }
        return bean;
    }
	...
}
```
#### ConfigurationClassPostProcessor$ImportAwareBeanPostProcessor

实现了`InstantiationAwareBeanPostProcessor`的两个方法。

- `#postProcessBeforeInitialization()`，若bean实现了`ImportAware`接口，则调用bean的`ImportAware#setImportMetadata()`实现方法。
- `#postProcessProperties()`，若bean实现了，`BeanFactoryAware`的子类，`EnhancedConfiguration`接口，则调用bean的`EnhancedConfiguration#setBeanFactory()`实现方法。

```java
private static class ImportAwareBeanPostProcessor extends InstantiationAwareBeanPostProcessorAdapter {
    
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        if (bean instanceof ImportAware) {
            ImportRegistry ir = this.beanFactory.getBean(IMPORT_REGISTRY_BEAN_NAME, ImportRegistry.class);
            AnnotationMetadata importingClass = ir.getImportingClassFor(ClassUtils.getUserClass(bean).getName());
            if (importingClass != null) {
                ((ImportAware) bean).setImportMetadata(importingClass);
            }
        }
        return bean;
    }

    @Override
    public PropertyValues postProcessProperties(@Nullable PropertyValues pvs, Object bean, String beanName) {
        // Inject the BeanFactory before AutowiredAnnotationBeanPostProcessor's
        // postProcessProperties method attempts to autowire other configuration beans.
        if (bean instanceof EnhancedConfiguration) {
            ((EnhancedConfiguration) bean).setBeanFactory(this.beanFactory);
        }
        return pvs;
    }
    ...
}
```

> 疑问：为什么两次Aware的填充不在一个方法中？

#### ApplicationContextAwareProcessor

经典的各类`*Aware`接口实现方案。

`ApplicationContextAwareProcessor`和上一小节的`ImportAwareBeanPostProcessor`类似，实现了`BeanPostProcessor#postProcessBeforeInitialization()`接口。

- `#postProcessBeforeInitialization()`，若bean至少实现了`EnvironmentAware`、`EmbeddedValueResolverAware`、`ResourceLoaderAware`、`ApplicationEventPublisherAware`、`MessageSourceAware`、`ApplicationContextAware`、`ApplicationStartupAware`接口，分别按次序进行对应的`#set***()`处理。从处理的Aware的类名和本类名可以看出，都是些上下文相关的对象。

常用的实现`ApplicationContextAware`接口从而获取spring上下文对象applicationContext就是在该类set的。

```java
class ApplicationContextAwareProcessor implements BeanPostProcessor {

   private final ConfigurableApplicationContext applicationContext;

   private final StringValueResolver embeddedValueResolver;

   /**
    * Create a new ApplicationContextAwareProcessor for the given context.
    */
   public ApplicationContextAwareProcessor(ConfigurableApplicationContext applicationContext) {
      this.applicationContext = applicationContext;
      this.embeddedValueResolver = new EmbeddedValueResolver(applicationContext.getBeanFactory());
   }

   @Override
   @Nullable
   public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
      if (!(bean instanceof EnvironmentAware || bean instanceof EmbeddedValueResolverAware ||
            bean instanceof ResourceLoaderAware || bean instanceof ApplicationEventPublisherAware ||
            bean instanceof MessageSourceAware || bean instanceof ApplicationContextAware ||
            bean instanceof ApplicationStartupAware)) {
         return bean;
      }

      AccessControlContext acc = null;

      if (System.getSecurityManager() != null) {
         acc = this.applicationContext.getBeanFactory().getAccessControlContext();
      }

      if (acc != null) {
         AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
            invokeAwareInterfaces(bean);
            return null;
         }, acc);
      }
      else {
         invokeAwareInterfaces(bean);
      }

      return bean;
   }

   private void invokeAwareInterfaces(Object bean) {
      if (bean instanceof EnvironmentAware) {
         ((EnvironmentAware) bean).setEnvironment(this.applicationContext.getEnvironment());
      }
      if (bean instanceof EmbeddedValueResolverAware) {
         ((EmbeddedValueResolverAware) bean).setEmbeddedValueResolver(this.embeddedValueResolver);
      }
      if (bean instanceof ResourceLoaderAware) {
         ((ResourceLoaderAware) bean).setResourceLoader(this.applicationContext);
      }
      if (bean instanceof ApplicationEventPublisherAware) {
         ((ApplicationEventPublisherAware) bean).setApplicationEventPublisher(this.applicationContext);
      }
      if (bean instanceof MessageSourceAware) {
         ((MessageSourceAware) bean).setMessageSource(this.applicationContext);
      }
      if (bean instanceof ApplicationStartupAware) {
         ((ApplicationStartupAware) bean).setApplicationStartup(this.applicationContext.getApplicationStartup());
      }
      if (bean instanceof ApplicationContextAware) {
         ((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
      }
   }

}
```



#### WebApplicationContextServletContextAwareProcessor

`WebApplicationContextServletContextAwareProcessor`实现`BeanPostProcessor`的方法都在父类ServletContextAwareProcessor`中。

与大多数例子一样，只是执行对应的`@set***()`方法，在该类中为`ServletContext`和`ServletConfig`。

```java
public class ServletContextAwareProcessor implements BeanPostProcessor {

       @Override
       public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
          if (getServletContext() != null && bean instanceof ServletContextAware) {
             ((ServletContextAware) bean).setServletContext(getServletContext());
          }
          if (getServletConfig() != null && bean instanceof ServletConfigAware) {
             ((ServletConfigAware) bean).setServletConfig(getServletConfig());
          }
          return bean;
       }

       @Override
       public Object postProcessAfterInitialization(Object bean, String beanName) {
          return bean;
       }
	...
}
```

### InstantiationAwareBeanPostProcessor

bean实例化前后置处理器。

`InstantiationAwareBeanPostProcessor`接口，继承并扩展了`BeanPostProcessor`接口。为Factory提供了对bean进行自定义修改的hook。

- `#postProcessBeforeInstantiation()`，实例化的前置处理。默认返回null，即不处理。重写返回自定义创建的bean时，会中断Spring bean的创建过程并代替将要创建的bean。参照上文中的`AbstractAutowireCapableBeanFactory#resolveBeforeInstantiation()`工厂类方法。AOP的一部分逻辑在此处。
- `#postProcessAfterInstantiation()`，实例化的后置处理，表示执行成功，失败则中断处理链路。一般在此处进行Spring的属性填充，如`@Autowrite`等。填充过程在`AbstractAutowireCapableBeanFactory#populateBean()`方法。
- `#postProcessProperties()`，自定义填充属性。

```java
public interface InstantiationAwareBeanPostProcessor extends BeanPostProcessor {

    @Nullable
    default Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
        return null;
    }

    default boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException {
        return true;
    }
    
	@Nullable
	default PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName)
			throws BeansException {

		return null;
	}
    ...
}
```

> 具体应用事例参照另一篇《AOP》

#### AutowiredAnnotationBeanPostProcessor



#### CommonAnnotationBeanPostProcessor
