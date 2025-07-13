## 定义具有扩展能力的接口

1. 定义业务接口类。定义接口实现类，配制`@SofaService`注解向其它模块暴露服务。其它模块使用`@SofaReference`注解引用暴露的bean。
2. 实现`void registerExtension(Extension) throws EXception`方法。在该方法中对扩展进行注册。
3. 模块spring配置文件定义bean。

<!--按设计来说这种方法应该有一个接口来定义行为，然而继承并实现Extensiable#registerExtension(Extension)方法反而导致sofa-modules加载失败？？？-->

```java
package com.alipay.sofa.service.api.component;

public interface Extensible {
    /**
     * 注册扩展能力
     */
    void registerExtension(Extension var1) throws Exception;

    void unregisterExtension(Extension var1) throws Exception;
}
```

`Extension`为对扩展能力的定义对象。

```java
package com.alipay.sofa.service.api.component;

public interface Extension {
    ComponentName getComponentName();

    ComponentName getTargetComponentName();

    String getExtensionPoint();

    Element getElement();

    Object[] getContributions();

    ClassLoader getAppClassLoader();

    String getId();
}
```

## 定义并暴露扩展点

以注解方式为说明。

定义的扩展点能够被其它模块进行自定义的覆盖。

1. `@XObject`注解定义要扩展的点。`value`属性为扩展点的名称。

   ```java
   package com.alipay.sofa.common.xmap.annotation
   
   /**
    * @author <a href="mailto:bs@nuxeo.com">Bogdan Stefanescu</a>
    * @since 2.6.0
    */
   @Target(ElementType.TYPE)
   @Retention(RetentionPolicy.RUNTIME)
   public @interface XObject {
   
       /**
        * An xpath expression specifying the XML node to bind to.
        *
        * @return the node xpath
        */
       String value() default "";
   
       /**
        *
        * @return
        */
       String[] order() default {};
   
   }
   ```

2. `@XNode`、`@XNodeSpring`等注解定义扩展的内容。`value`为扩展点内容的名称，`type`类型。

   ```java
   package com.alipay.sofa.common.xmap.spring;
   
   /**
    * @author xi.hux@alipay.com
    * @since 2.6.0
    */
   @XMemberAnnotation(XMemberAnnotation.NODE_SPRING)
   @Target({ ElementType.FIELD, ElementType.METHOD })
   @Retention(RetentionPolicy.RUNTIME)
   public @interface XNodeSpring {
   
       /**
        * Get the node xpath
        *
        * @return the node xpath
        */
       String value();
   
       /**
        * Whether need to trim or not
        *
        * @return trim or not
        */
       boolean trim() default true;
   
       /**
        * Get the type of items
        *
        * @return the type of items
        */
       Class type() default Object.class;
   
   }
   ```

3. 在模块spring配置文件中定义扩展接口的扩展点，即关联到上一步定义接口。

   ```xml
   <sofa:extension-point name="datasourcePoint" ref="datasourceExtension">
       <sofa:object class="com.glmapper.bridge.datasource.DatasourceExtensionDescriptor"/>
   </sofa:extension-point>
   ```

## 扩展

在配置文件中定义扩展实现和要扩展的点。

```xml
<!--扩展的实现-->
<bean id="oracleDatasourceBean" class="com.glmapper.bridge.custom.OracleDatasourceBean"/>
<!--实现关联扩展点-->
<sofa:extension bean="datasourceExtension" point="datasourcePoint">
    <sofa:content>
        <datasourcePoint>
            <!--@XNode注解，基本类型-->
            <!--@XNdoeSpring注解，对应spring IOC容器中的实例名，在此例中为上方定义的扩展实现的id-->
            <value>oracleDatasourceBean</value>
        </datasourcePoint>
    </sofa:content>
</sofa:extension>
```

注意，若配置文件不在sofa-modules模块中，则可能需要手动配置扫描配置文件。