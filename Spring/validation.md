# ConstraintValidator

`ConstraintValidator`接口提供校验方法`#isValid()`。

`#isVaild()`方法传入要校验的值和检验其上下文，校验器实现该接口的校验过程。

类头部定义了两个泛型，用来匹配检验目标。`A`匹配注解；`T`匹配校验类型。

```java
public interface ConstraintValidator<A extends Annotation, T> {

   default void initialize(A constraintAnnotation) {
   }

   boolean isValid(T value, ConstraintValidatorContext context);
}
```
例子`AssertTrueValidator`

```java
public class AssertTrueValidator implements ConstraintValidator<AssertTrue, Boolean> {

   @Override
   public boolean isValid(Boolean bool, ConstraintValidatorContext constraintValidatorContext) {
      //null values are valid
      return bool == null || bool;
   }
}
```



# BeanMetaData

- `constraintDescriptor`描述器，constraintValidatorClasses

# BeanMetaDataManager

创建BeanMetaData对象

```java
public <T> BeanMetaData<T> getBeanMetaData(Class<T> beanClass) {
   	return getOrCreateBeanMetaData( beanClass, false );
}

private <T> BeanMetaData<T> getOrCreateBeanMetaData(Class<T> beanClass, boolean allowUnconstrainedTypeSingleton) {
Contracts.assertNotNull( beanClass, MESSAGES.beanTypeCannotBeNull() );

    BeanMetaData<T> beanMetaData = (BeanMetaData<T>) beanMetaDataCache.get( beanClass );

    // create a new BeanMetaData in case none is cached
    if ( beanMetaData == null ) {
    	beanMetaData = createBeanMetaData( beanClass );
        if ( !beanMetaData.hasConstraints() && allowUnconstrainedTypeSingleton ) {
            beanMetaData = (BeanMetaData<T>) UnconstrainedEntityMetaDataSingleton.getSingleton();
        }

        final BeanMetaData<T> cachedBeanMetaData = (BeanMetaData<T>) beanMetaDataCache.putIfAbsent(
                beanClass,
                beanMetaData
                );
        if ( cachedBeanMetaData != null ) {
            beanMetaData = cachedBeanMetaData;
        }
    }

    if ( beanMetaData instanceof UnconstrainedEntityMetaDataSingleton && !allowUnconstrainedTypeSingleton ) {
        beanMetaData = createBeanMetaData( beanClass );
        beanMetaDataCache.put(
                beanClass,
                beanMetaData
                );
    }

    return beanMetaData;
}
```

# MethodValidationInterceptor

