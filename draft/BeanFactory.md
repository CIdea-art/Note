## AbstractBeanFactory

```java

/**
 * BeanPostProcessors to apply.
 * {@link org.springframework.context.support.AbstractApplicationContext#prepareBeanFactory(ConfigurableListableBeanFactory)}
 */
private final List<BeanPostProcessor> beanPostProcessors = new BeanPostProcessorCacheAwareList();

/** Cache of pre-filtered post-processors. */
@Nullable
private volatile BeanPostProcessorCache beanPostProcessorCache;


static class BeanPostProcessorCache {

    final List<InstantiationAwareBeanPostProcessor> instantiationAware = new ArrayList<>();

    final List<SmartInstantiationAwareBeanPostProcessor> smartInstantiationAware = new ArrayList<>();

    final List<DestructionAwareBeanPostProcessor> destructionAware = new ArrayList<>();

    final List<MergedBeanDefinitionPostProcessor> mergedDefinition = new ArrayList<>();
}
```