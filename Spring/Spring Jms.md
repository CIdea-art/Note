Spring通过加载spring.factories文件中的配置进行配置类的自动加载。

通过spring-boot-autoconfigure包下的spring.factories文件中的配置自动导入配置类。

## ConnectionFactory

spring.factories

```xml-dtd
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
...略过
org.springframework.boot.autoconfigure.jms.activemq.ActiveMQAutoConfiguration,\
org.springframework.boot.autoconfigure.jms.artemis.ArtemisAutoConfiguration,\
...略过
```

可见Spring Jms仅实现了两种消息机制。@ConditionalOnClass注解会根据项目中是否存在指定包的类决定是否生效。

以ActiveMQAutoConfiguration为例。

```java
@Configuration
@AutoConfigureBefore(JmsAutoConfiguration.class)
@AutoConfigureAfter({ JndiConnectionFactoryAutoConfiguration.class })
// 两个 Condition 使配置仅在 Spring 缺失 ConnectionFactory.class 实例且存在 activeMQ 包时生效
@ConditionalOnClass({ ConnectionFactory.class, ActiveMQConnectionFactory.class })
@ConditionalOnMissingBean(ConnectionFactory.class)
// 启用相关配置
@EnableConfigurationProperties({ ActiveMQProperties.class, JmsProperties.class })
@Import({ ActiveMQXAConnectionFactoryConfiguration.class,
      ActiveMQConnectionFactoryConfiguration.class })
public class ActiveMQAutoConfiguration {

}
```

重点查看ActiveMQConnectionFactoryConfiguration配置，该类对ConnectionFactory进行了初始化。

ActiveMQConnectionFactoryConfiguration内部配置类SimpleConnectionFactoryConfiguration。

@ConditionalOnProperty#matchIfMissing属性为true时保证在配置缺失时默认生效。

```java
@Configuration
@ConditionalOnClass(CachingConnectionFactory.class)
@ConditionalOnProperty(prefix = "spring.activemq.pool", name = "enabled", havingValue = "false", matchIfMissing = true)
static class SimpleConnectionFactoryConfiguration {

    // <1>
    private final JmsProperties jmsProperties;
    
    // <2>
    private final ActiveMQProperties properties;

    // <3>
    private final List<ActiveMQConnectionFactoryCustomizer> connectionFactoryCustomizers;

    SimpleConnectionFactoryConfiguration(JmsProperties jmsProperties,
        ActiveMQProperties properties,
        ObjectProvider<ActiveMQConnectionFactoryCustomizer> connectionFactoryCustomizers) {
    	this.jmsProperties = jmsProperties;
      	this.properties = properties;
      	this.connectionFactoryCustomizers = connectionFactoryCustomizers
        	.orderedStream().collect(Collectors.toList());
   	}
	......
}
```

<1>jmsProperties为Jms的相关配置。todo

<2>properties为mq的连接配置。

<3>todo



内部配置方法。同样的逻辑，默认使用缓存，创建CachingConnectionFactory。CachingConnectionFactory类封装了ConnextionFactory对象，用于缓存代理，具体实现todo。createConnectionFactory()方法只是创建初始化并返回了一个ActiveMQConnectionFactory对象。因此，最终的实现都是ActiveMQConnectionFactory。

```java
@Bean
@ConditionalOnProperty(prefix = "spring.jms.cache", name = "enabled", havingValue = "true", matchIfMissing = true)
public CachingConnectionFactory cachingJmsConnectionFactory() {
    JmsProperties.Cache cacheProperties = this.jmsProperties.getCache();
    CachingConnectionFactory connectionFactory = new CachingConnectionFactory(
        createConnectionFactory());
    connectionFactory.setCacheConsumers(cacheProperties.isConsumers());
    connectionFactory.setCacheProducers(cacheProperties.isProducers());
    connectionFactory.setSessionCacheSize(cacheProperties.getSessionCacheSize());
    return connectionFactory;
}

@Bean
@ConditionalOnProperty(prefix = "spring.jms.cache", name = "enabled", havingValue = "false")
public ActiveMQConnectionFactory jmsConnectionFactory() {
    return createConnectionFactory();
}

private ActiveMQConnectionFactory createConnectionFactory() {
    return new ActiveMQConnectionFactoryFactory(this.properties,
                                                this.connectionFactoryCustomizers)
        .createConnectionFactory(ActiveMQConnectionFactory.class);
}
```

## JmsTemplate

spring.factories

```xml-dtd
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
...略过
org.springframework.boot.autoconfigure.jms.JmsAutoConfiguration,\
...略过
```





```java
@Configuration
@ConditionalOnClass({ Message.class, JmsTemplate.class })
@ConditionalOnBean(ConnectionFactory.class)
@EnableConfigurationProperties(JmsProperties.class)
@Import(JmsAnnotationDrivenConfiguration.class)
public class JmsAutoConfiguration {

   @Configuration
   protected static class JmsTemplateConfiguration {

      private final JmsProperties properties;

      private final ObjectProvider<DestinationResolver> destinationResolver;

      private final ObjectProvider<MessageConverter> messageConverter;

      public JmsTemplateConfiguration(JmsProperties properties,
            ObjectProvider<DestinationResolver> destinationResolver,
            ObjectProvider<MessageConverter> messageConverter) {
         this.properties = properties;
         this.destinationResolver = destinationResolver;
         this.messageConverter = messageConverter;
      }

       /**
        * 创建、初始化JmsTemplate并注入Spring
        */
      @Bean
      @ConditionalOnMissingBean
      @ConditionalOnSingleCandidate(ConnectionFactory.class)
      public JmsTemplate jmsTemplate(ConnectionFactory connectionFactory) {
         PropertyMapper map = PropertyMapper.get();
         JmsTemplate template = new JmsTemplate(connectionFactory);
         template.setPubSubDomain(this.properties.isPubSubDomain());
         map.from(this.destinationResolver::getIfUnique).whenNonNull()
               .to(template::setDestinationResolver);
         map.from(this.messageConverter::getIfUnique).whenNonNull()
               .to(template::setMessageConverter);
         mapTemplateProperties(this.properties.getTemplate(), template);
         return template;
      }

      private void mapTemplateProperties(Template properties, JmsTemplate template) {
         PropertyMapper map = PropertyMapper.get();
         map.from(properties::getDefaultDestination).whenNonNull()
               .to(template::setDefaultDestinationName);
         map.from(properties::getDeliveryDelay).whenNonNull().as(Duration::toMillis)
               .to(template::setDeliveryDelay);
         map.from(properties::determineQosEnabled).to(template::setExplicitQosEnabled);
         map.from(properties::getDeliveryMode).whenNonNull().as(DeliveryMode::getValue)
               .to(template::setDeliveryMode);
         map.from(properties::getPriority).whenNonNull().to(template::setPriority);
         map.from(properties::getTimeToLive).whenNonNull().as(Duration::toMillis)
               .to(template::setTimeToLive);
         map.from(properties::getReceiveTimeout).whenNonNull().as(Duration::toMillis)
               .to(template::setReceiveTimeout);
      }

   }
	......
}
```

