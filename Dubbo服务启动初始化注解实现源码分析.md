> 白菜Java自习室 涵盖核心知识

> Dubbo的版本2.7.x，SpringBoot整合

# 概要

Dubbo注解配置是基于**Spring注解配置**的，巧妙地利用Spring进行配置简化。建议同学们优先去了解Spring注解配置后阅读。

**@EnableDubbo @EnableDubboConfig @DubboComponentScan** 这三个注解位于dubbo-config模块的dubbo-config-spring项目下的org.apache.dubbo.config.spring.context.annotation包中，是主要的初始化配置注解。

**@DubboService @DubboReference**
这二个注解位于dubbo-common模块的dubbo-common项目下的org.apache.dubbo.config.annotation包中，基于@DubboComponentScan扫描实现了初始化。

# @EnableDubbo

@EnableDubbo注解并未提供多余的功能仅是将@DubboComponentScan和@EnableDubboConfig组合起来方便用户使用，简单的看下代码。

```
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
@EnableDubboConfig
@DubboComponentScan
public @interface EnableDubbo {

    @AliasFor(annotation = DubboComponentScan.class, attribute = "basePackages")
    String[] scanBasePackages() default {};

    @AliasFor(annotation = DubboComponentScan.class, attribute = "basePackageClasses")
    Class<?>[] scanBasePackageClasses() default {};

    @AliasFor(annotation = EnableDubboConfig.class, attribute = "multiple")
    boolean multipleConfig() default true;

}
```

# @EnableDubboConfig

@EnableDubboConfig注解的目的就是帮我们简化这些(ApplicationConfig、ModuleConfig、RegistryConfig、ProtocolConfig、MonitorConfig、ProviderConfig、ConsumerConfig等)配置，只要将相应的配置项通过.properties文件交由@PropertySource注解由Spring加载到Environment中，Dubbo即可通过Environment中相应的属性与ApplicationConfig对象同名的属性进行绑定，而这个过程都是自动的。本身代码中作者给出了注释，问题变得显而易见。

```
/**
 * As  a convenient and multiple {@link EnableDubboConfigBinding}
 * in default behavior , is equal to single bean bindings with below convention prefixes of properties:
 * <ul>
 * <li>{@link ApplicationConfig} binding to property : "dubbo.application"</li>
 * <li>{@link ModuleConfig} binding to property :  "dubbo.module"</li>
 * <li>{@link RegistryConfig} binding to property :  "dubbo.registry"</li>
 * <li>{@link ProtocolConfig} binding to property :  "dubbo.protocol"</li>
 * <li>{@link MonitorConfig} binding to property :  "dubbo.monitor"</li>
 * <li>{@link ProviderConfig} binding to property :  "dubbo.provider"</li>
 * <li>{@link ConsumerConfig} binding to property :  "dubbo.consumer"</li>
 * </ul>
 * <p>
 * In contrast, on multiple bean bindings that requires to set {@link #multiple()} to be <code>true</code> :
 * <ul>
 * <li>{@link ApplicationConfig} binding to property : "dubbo.applications"</li>
 * <li>{@link ModuleConfig} binding to property :  "dubbo.modules"</li>
 * <li>{@link RegistryConfig} binding to property :  "dubbo.registries"</li>
 * <li>{@link ProtocolConfig} binding to property :  "dubbo.protocols"</li>
 * <li>{@link MonitorConfig} binding to property :  "dubbo.monitors"</li>
 * <li>{@link ProviderConfig} binding to property :  "dubbo.providers"</li>
 * <li>{@link ConsumerConfig} binding to property :  "dubbo.consumers"</li>
 * </ul>
 *
 * @see EnableDubboConfigBinding
 * @see DubboConfigConfiguration
 * @see DubboConfigConfigurationRegistrar
 * @since 2.5.8
 */
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
@Import(DubboConfigConfigurationRegistrar.class)
public @interface EnableDubboConfig {

    boolean multiple() default true;

}
```

## DubboConfigConfigurationRegistrar

@EnableDubboConfig背后的注册处理就是通过这个(@Import(DubboConfigConfigurationRegistrar.class)完成的，DubboConfigConfigurationRegistrar继承实现于Spring提供的一个接口ImportBeanDefinitionRegistrar，这个类会在Spring中注册两个对象DubboConfigConfiguration.Single和DubboConfigConfiguration.Multiple。

```
package org.springframework.context.annotation;

public interface ImportBeanDefinitionRegistrar {
    void registerBeanDefinitions(AnnotationMetadata var1, BeanDefinitionRegistry var2);
}
```

```
public class DubboConfigConfigurationRegistrar implements ImportBeanDefinitionRegistrar {

    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {

        AnnotationAttributes attributes = AnnotationAttributes.fromMap(
                importingClassMetadata.getAnnotationAttributes(EnableDubboConfig.class.getName()));

        boolean multiple = attributes.getBoolean("multiple");

        registerBeans(registry, DubboConfigConfiguration.Single.class);

        if (multiple) {
            registerBeans(registry, DubboConfigConfiguration.Multiple.class);
        }

        registerCommonBeans(registry);
    }
}
```

## DubboConfigConfiguration

查看DubboConfigConfiguration可知，Single和Multiple本身并没有任何作用，只是它头上的@EnableConfigurationBeanBinding的注解背后的解析器ConfigurationBeanBindingRegistrar起了作用。

```
public class DubboConfigConfiguration {

    @EnableConfigurationBeanBindings({
            @EnableConfigurationBeanBinding(prefix = "dubbo.application", type = ApplicationConfig.class),
            @EnableConfigurationBeanBinding(prefix = "dubbo.module", type = ModuleConfig.class),
            @EnableConfigurationBeanBinding(prefix = "dubbo.registry", type = RegistryConfig.class),
            @EnableConfigurationBeanBinding(prefix = "dubbo.protocol", type = ProtocolConfig.class),
            @EnableConfigurationBeanBinding(prefix = "dubbo.monitor", type = MonitorConfig.class),
            @EnableConfigurationBeanBinding(prefix = "dubbo.provider", type = ProviderConfig.class),
            @EnableConfigurationBeanBinding(prefix = "dubbo.consumer", type = ConsumerConfig.class),
            @EnableConfigurationBeanBinding(prefix = "dubbo.config-center", type = ConfigCenterBean.class),
            @EnableConfigurationBeanBinding(prefix = "dubbo.metadata-report", type = MetadataReportConfig.class),
            @EnableConfigurationBeanBinding(prefix = "dubbo.metrics", type = MetricsConfig.class),
            @EnableConfigurationBeanBinding(prefix = "dubbo.ssl", type = SslConfig.class)
    })
    public static class Single {

    }

    @EnableConfigurationBeanBindings({
            @EnableConfigurationBeanBinding(prefix = "dubbo.applications", type = ApplicationConfig.class, multiple = true),
            @EnableConfigurationBeanBinding(prefix = "dubbo.modules", type = ModuleConfig.class, multiple = true),
            @EnableConfigurationBeanBinding(prefix = "dubbo.registries", type = RegistryConfig.class, multiple = true),
            @EnableConfigurationBeanBinding(prefix = "dubbo.protocols", type = ProtocolConfig.class, multiple = true),
            @EnableConfigurationBeanBinding(prefix = "dubbo.monitors", type = MonitorConfig.class, multiple = true),
            @EnableConfigurationBeanBinding(prefix = "dubbo.providers", type = ProviderConfig.class, multiple = true),
            @EnableConfigurationBeanBinding(prefix = "dubbo.consumers", type = ConsumerConfig.class, multiple = true),
            @EnableConfigurationBeanBinding(prefix = "dubbo.config-centers", type = ConfigCenterBean.class, multiple = true),
            @EnableConfigurationBeanBinding(prefix = "dubbo.metadata-reports", type = MetadataReportConfig.class, multiple = true),
            @EnableConfigurationBeanBinding(prefix = "dubbo.metricses", type = MetricsConfig.class, multiple = true)
    })
    public static class Multiple {

    }
}
```

### @EnableConfigurationBeanBinding

@EnableConfigurationBeanBinding 注解prefix的值代表.properties文件配置项key的前缀，完整的key就会映射到type代表类的属性。

```
@Target({ElementType.TYPE, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import({ConfigurationBeanBindingRegistrar.class})
public @interface EnableConfigurationBeanBinding {
    boolean DEFAULT_MULTIPLE = false;
    boolean DEFAULT_IGNORE_UNKNOWN_FIELDS = true;
    boolean DEFAULT_IGNORE_INVALID_FIELDS = true;

    String prefix();

    Class<?> type();

    boolean multiple() default false;

    boolean ignoreUnknownFields() default true;

    boolean ignoreInvalidFields() default true;
}
```

### ConfigurationBeanBindingRegistrar

ConfigurationBeanBindingRegistrar读取@EnableConfigurationBeanBinding的type属性来注册bean。

```
public class ConfigurationBeanBindingRegistrar implements ImportBeanDefinitionRegistrar, EnvironmentAware {

    // ......
    
    public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
        Map<String, Object> attributes = metadata.getAnnotationAttributes(ENABLE_CONFIGURATION_BINDING_CLASS_NAME);
        this.registerConfigurationBeanDefinitions(attributes, registry);
    }

    public void registerConfigurationBeanDefinitions(Map<String, Object> attributes, BeanDefinitionRegistry registry) {
        String prefix = (String)AnnotationUtils.getRequiredAttribute(attributes, "prefix");
        prefix = this.environment.resolvePlaceholders(prefix);
        Class<?> configClass = (Class)AnnotationUtils.getRequiredAttribute(attributes, "type");
        boolean multiple = (Boolean)AnnotationUtils.getAttribute(attributes, "multiple", false);
        boolean ignoreUnknownFields = (Boolean)AnnotationUtils.getAttribute(attributes, "ignoreUnknownFields", true);
        boolean ignoreInvalidFields = (Boolean)AnnotationUtils.getAttribute(attributes, "ignoreInvalidFields", true);
        this.registerConfigurationBeans(prefix, configClass, multiple, ignoreUnknownFields, ignoreInvalidFields, registry);
    }

    private void registerConfigurationBeans(String prefix, Class<?> configClass, boolean multiple, boolean ignoreUnknownFields, boolean ignoreInvalidFields, BeanDefinitionRegistry registry) {
        Map<String, Object> configurationProperties = PropertySourcesUtils.getSubProperties(this.environment.getPropertySources(), this.environment, prefix);
        if (CollectionUtils.isEmpty(configurationProperties)) {
            if (this.log.isDebugEnabled()) {
                this.log.debug("There is no property for binding to configuration class [" + configClass.getName() + "] within prefix [" + prefix + "]");
            }

        } else {
            Set<String> beanNames = multiple ? this.resolveMultipleBeanNames(configurationProperties) : Collections.singleton(this.resolveSingleBeanName(configurationProperties, configClass, registry));
            Iterator var9 = beanNames.iterator();

            while(var9.hasNext()) {
                String beanName = (String)var9.next();
                this.registerConfigurationBean(beanName, configClass, multiple, ignoreUnknownFields, ignoreInvalidFields, configurationProperties, registry);
            }

            this.registerConfigurationBindingBeanPostProcessor(registry);
        }
    }
    
    private void registerConfigurationBean(String beanName, Class<?> configClass, boolean multiple, boolean ignoreUnknownFields, boolean ignoreInvalidFields, Map<String, Object> configurationProperties, BeanDefinitionRegistry registry) {
        BeanDefinitionBuilder builder = BeanDefinitionBuilder.rootBeanDefinition(configClass);
        AbstractBeanDefinition beanDefinition = builder.getBeanDefinition();
        this.setSource(beanDefinition);
        Map<String, Object> subProperties = this.resolveSubProperties(multiple, beanName, configurationProperties);
        ConfigurationBeanBindingPostProcessor.initBeanMetadataAttributes(beanDefinition, subProperties, ignoreUnknownFields, ignoreInvalidFields);
        registry.registerBeanDefinition(beanName, beanDefinition);
        if (this.log.isInfoEnabled()) {
            this.log.info("The configuration bean definition [name : " + beanName + ", content : " + beanDefinition + "] has been registered.");
        }

    }
    
    private void registerConfigurationBindingBeanPostProcessor(BeanDefinitionRegistry registry) {
        BeanRegistrar.registerInfrastructureBean(registry, "configurationBeanBindingPostProcessor", ConfigurationBeanBindingPostProcessor.class);
    }
    
    // ......
    
}
```

### ConfigurationBeanBindingPostProcessor

ConfigurationBeanBindingPostProcessor 继承实现于Spring提供的一个接口BeanPostProcessor，BeanPostProcessor是Spring IOC容器给我们提供的一个扩展接口，在Spring容器中完成bean实例化、配置以及其他初始化方法前后添加一些自己处理逻辑，调用postProcessBeforeInitialization()方法来进行数据绑定。

```
package org.springframework.beans.factory.config;

public interface BeanPostProcessor {
    Object postProcessBeforeInitialization(Object var1, String var2) throws BeansException;

    Object postProcessAfterInitialization(Object var1, String var2) throws BeansException;
}
```

```
public class ConfigurationBeanBindingPostProcessor implements BeanPostProcessor, BeanFactoryAware {

    // ......
    
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        BeanDefinition beanDefinition = this.getNullableBeanDefinition(beanName);
        if (this.isConfigurationBean(bean, beanDefinition)) {
            this.bindConfigurationBean(bean, beanDefinition);
            this.customize(beanName, bean);
        }

        return bean;
    }

    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }

    // ......
    
    private void bindConfigurationBean(Object configurationBean, BeanDefinition beanDefinition) {
        Map<String, Object> configurationProperties = getConfigurationProperties(beanDefinition);
        boolean ignoreUnknownFields = getIgnoreUnknownFields(beanDefinition);
        boolean ignoreInvalidFields = getIgnoreInvalidFields(beanDefinition);
        this.getConfigurationBeanBinder().bind(configurationProperties, ignoreUnknownFields, ignoreInvalidFields, configurationBean);
        if (this.log.isInfoEnabled()) {
            this.log.info("The configuration bean [" + configurationBean + "] have been binding by the configuration properties [" + configurationProperties + "]");
        }

    }

    private void customize(String beanName, Object configurationBean) {
        Iterator var3 = this.getConfigurationBeanCustomizers().iterator();

        while(var3.hasNext()) {
            ConfigurationBeanCustomizer customizer = (ConfigurationBeanCustomizer)var3.next();
            customizer.customize(beanName, configurationBean);
        }

    }

    // ......
    
    public ConfigurationBeanBinder getConfigurationBeanBinder() {
        if (this.configurationBeanBinder == null) {
            this.initConfigurationBeanBinder();
        }

        return this.configurationBeanBinder;
    }
    
    private void initConfigurationBeanBinder() {
        if (this.configurationBeanBinder == null) {
            try {
                this.configurationBeanBinder = (ConfigurationBeanBinder)this.beanFactory.getBean(ConfigurationBeanBinder.class);
            } catch (BeansException var2) {
                if (this.log.isInfoEnabled()) {
                    this.log.info("configurationBeanBinder Bean can't be found in ApplicationContext.");
                }

                this.configurationBeanBinder = this.defaultConfigurationBeanBinder();
            }
        }
    }
    
    // ......

}
```

# @DubboComponentScan

@DubboComponentScan是Dubbo组件扫描注释，扫描类路径以获取将自动注册为Spring bean的注释组件，背后的处理器是DubboComponentScanRegistrar。

```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(DubboComponentScanRegistrar.class)
public @interface DubboComponentScan {

    String[] value() default {};

    String[] basePackages() default {};

    Class<?>[] basePackageClasses() default {};

}
```

## DubboComponentScanRegistrar

DubboComponentScanRegistrar继承实现于Spring提供的一个接口ImportBeanDefinitionRegistrar。registerServiceAnnotationBeanPostProcessor()方法注册一个ServiceAnnotationBeanPostProcessor，用来扫描@Service，registerCommonBeans()方法用来为使用@Reference等的Bean注册。

```
package org.springframework.context.annotation;

public interface ImportBeanDefinitionRegistrar {
    void registerBeanDefinitions(AnnotationMetadata var1, BeanDefinitionRegistry var2);
}

```

```
public class DubboComponentScanRegistrar implements ImportBeanDefinitionRegistrar {

    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {

        Set<String> packagesToScan = getPackagesToScan(importingClassMetadata);

        registerServiceAnnotationBeanPostProcessor(packagesToScan, registry);
        
        registerCommonBeans(registry);
    }

    private void registerServiceAnnotationBeanPostProcessor(Set<String> packagesToScan, BeanDefinitionRegistry registry) {

        BeanDefinitionBuilder builder = rootBeanDefinition(ServiceAnnotationBeanPostProcessor.class);
        builder.addConstructorArgValue(packagesToScan);
        builder.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
        AbstractBeanDefinition beanDefinition = builder.getBeanDefinition();
        BeanDefinitionReaderUtils.registerWithGeneratedName(beanDefinition, registry);

    }

    private Set<String> getPackagesToScan(AnnotationMetadata metadata) {
        AnnotationAttributes attributes = AnnotationAttributes.fromMap(
                metadata.getAnnotationAttributes(DubboComponentScan.class.getName()));
        String[] basePackages = attributes.getStringArray("basePackages");
        Class<?>[] basePackageClasses = attributes.getClassArray("basePackageClasses");
        String[] value = attributes.getStringArray("value");
        // Appends value array attributes
        Set<String> packagesToScan = new LinkedHashSet<String>(Arrays.asList(value));
        packagesToScan.addAll(Arrays.asList(basePackages));
        for (Class<?> basePackageClass : basePackageClasses) {
            packagesToScan.add(ClassUtils.getPackageName(basePackageClass));
        }
        if (packagesToScan.isEmpty()) {
            return Collections.singleton(ClassUtils.getPackageName(metadata.getClassName()));
        }
        return packagesToScan;
    }

}
```

## @DubboService

### ServiceAnnotationBeanPostProcessor

ServiceAnnotationBeanPostProcessor类已被弃用，其新的实现查看ServiceClassPostProcessor。

```
@Deprecated
public class ServiceAnnotationBeanPostProcessor extends ServiceClassPostProcessor {
    // ......
}
```

ServiceClassPostProcessor继承实现于Spring提供的一个接口BeanDefinitionRegistryPostProcessor。当ServiceClassPostProcessor的postProcessBeanDefinitionRegistry()方法调用时，会扫描Spring的@Service注解、Dubbo的@Service注解和@DubboService注解的类并注册为Spring Bean。

```
package org.springframework.beans.factory.support;

public interface BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor {
    void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry var1) throws BeansException;
}
```

```
public class ServiceClassPostProcessor implements BeanDefinitionRegistryPostProcessor, EnvironmentAware,
        ResourceLoaderAware, BeanClassLoaderAware {

    private final static List<Class<? extends Annotation>> serviceAnnotationTypes = asList(
            DubboService.class,
            Service.class,
            com.alibaba.dubbo.config.annotation.Service.class
    );

    // ......
    
    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {

        registerBeans(registry, DubboBootstrapApplicationListener.class);

        Set<String> resolvedPackagesToScan = resolvePackagesToScan(packagesToScan);

        if (!CollectionUtils.isEmpty(resolvedPackagesToScan)) {
            registerServiceBeans(resolvedPackagesToScan, registry);
        } else {
            if (logger.isWarnEnabled()) {
                logger.warn("packagesToScan is empty , ServiceBean registry will be ignored!");
            }
        }

    }
    
    // ......
}
```

registerServiceBeans()方法中除了将@Service注解的类注册为Spring Bean，内部还调用registerServiceBean()方法注册一个ServiceBean，这个对象内部有个ref属性指向@Service类bean。

```
    /**
     * Registers {@link ServiceBean} from new annotated {@link Service} {@link BeanDefinition}
     *
     * @param beanDefinitionHolder
     * @param registry
     * @param scanner
     * @see ServiceBean
     * @see BeanDefinition
     */
    private void registerServiceBean(BeanDefinitionHolder beanDefinitionHolder, BeanDefinitionRegistry registry,
                                     DubboClassPathBeanDefinitionScanner scanner) {

        Class<?> beanClass = resolveClass(beanDefinitionHolder);

        Annotation service = findServiceAnnotation(beanClass);

        AnnotationAttributes serviceAnnotationAttributes = getAnnotationAttributes(service, false, false);

        Class<?> interfaceClass = resolveServiceInterfaceClass(serviceAnnotationAttributes, beanClass);

        String annotatedServiceBeanName = beanDefinitionHolder.getBeanName();

        AbstractBeanDefinition serviceBeanDefinition =
                buildServiceBeanDefinition(service, serviceAnnotationAttributes, interfaceClass, annotatedServiceBeanName);

        // ServiceBean Bean name
        String beanName = generateServiceBeanName(serviceAnnotationAttributes, interfaceClass);

        if (scanner.checkCandidate(beanName, serviceBeanDefinition)) { // check duplicated candidate bean
            registry.registerBeanDefinition(beanName, serviceBeanDefinition);
            if (logger.isInfoEnabled()) {
                logger.info("The BeanDefinition[" + serviceBeanDefinition +
                        "] of ServiceBean has been registered with name : " + beanName);
            }
        } else {
            if (logger.isWarnEnabled()) {
                logger.warn("The Duplicated BeanDefinition[" + serviceBeanDefinition +
                        "] of ServiceBean[ bean name : " + beanName +
                        "] was be found , Did @DubboComponentScan scan to same package in many times?");
            }
        }

    }
```

### ServiceBean

ServiceBean继承于ServiceConfig，并且实现了InitializingBean接口，在afterPropertiesSet方法被调用的时候会通过applicationContext查找Dubbo Provider需要的配置，而这些配置早已通过@EnableDubboConfig注册到了Spring。

```
public class ServiceBean<T> extends ServiceConfig<T> implements InitializingBean, DisposableBean,
        ApplicationContextAware, BeanNameAware, ApplicationEventPublisherAware {
        
    // ......
    
    @Override
    public void afterPropertiesSet() throws Exception {
        if (StringUtils.isEmpty(getPath())) {
            if (StringUtils.isNotEmpty(beanName)
                    && StringUtils.isNotEmpty(getInterface())
                    && beanName.startsWith(getInterface())) {
                setPath(beanName);
            }
        }
    }
    
}
```

## @DubboReference

### ReferenceAnnotationBeanPostProcessor

ReferenceAnnotationBeanPostProcessor继承实现于Spring提供的一个抽象类AbstractAnnotationBeanPostProcessor。当ReferenceAnnotationBeanPostProcessor的doGetInjectedBean()方法调用时，会扫描Spring的@Reference注解、Dubbo的@Reference注解和@DubboReference注解并通过JDK动态代理创建一个代理对象，这个代理对象的目标对象是ReferenceBean，这个ReferenceBean继承于在API配置中提到的ReferenceConfig。

```
package com.alibaba.spring.beans.factory.annotation;

public abstract class AbstractAnnotationBeanPostProcessor extends InstantiationAwareBeanPostProcessorAdapter implements MergedBeanDefinitionPostProcessor, PriorityOrdered, BeanFactoryAware, BeanClassLoaderAware, EnvironmentAware, DisposableBean {
    // ......
}
```

```
public class ReferenceAnnotationBeanPostProcessor extends AbstractAnnotationBeanPostProcessor implements
        ApplicationContextAware, ApplicationListener<ServiceBeanExportedEvent> {

    // ......
    
    public ReferenceAnnotationBeanPostProcessor() {
        super(
        DubboReference.class, 
        Reference.class, 
        com.alibaba.dubbo.config.annotation.Reference.class);
    }

    // ......
    
    @Override
    protected Object doGetInjectedBean(AnnotationAttributes attributes, Object bean, String beanName, Class<?> injectedType,
                                       InjectionMetadata.InjectedElement injectedElement) throws Exception {

        String referencedBeanName = buildReferencedBeanName(attributes, injectedType);

        String referenceBeanName = getReferenceBeanName(attributes, injectedType);

        ReferenceBean referenceBean = buildReferenceBeanIfAbsent(referenceBeanName, attributes, injectedType);

        boolean localServiceBean = isLocalServiceBean(referencedBeanName, referenceBean, attributes);

        registerReferenceBean(referencedBeanName, referenceBean, attributes, localServiceBean, injectedType);

        cacheInjectedReferenceBean(referenceBean, injectedElement);

        return getOrCreateProxy(referencedBeanName, referenceBean, localServiceBean, injectedType);
    }
    
    // ......
    
    private ReferenceBean buildReferenceBeanIfAbsent(String referenceBeanName, AnnotationAttributes attributes, Class<?> referencedType) throws Exception {
        ReferenceBean<?> referenceBean = referenceBeanCache.get(referenceBeanName);
        if (referenceBean == null) {
            ReferenceBeanBuilder beanBuilder = ReferenceBeanBuilder
                    .create(attributes, applicationContext)
                    .interfaceClass(referencedType);
            referenceBean = beanBuilder.build();
            referenceBeanCache.put(referenceBeanName, referenceBean);
        } else if (!referencedType.isAssignableFrom(referenceBean.getInterfaceClass())) {
            throw new IllegalArgumentException("reference bean name " + referenceBeanName + " has been duplicated, but interfaceClass " +
                    referenceBean.getInterfaceClass().getName() + " cannot be assigned to " + referencedType.getName());
        }
        return referenceBean;
    }
    
    private void registerReferenceBean(String referencedBeanName, ReferenceBean referenceBean, AnnotationAttributes attributes, boolean localServiceBean, Class<?> interfaceClass) {

        ConfigurableListableBeanFactory beanFactory = getBeanFactory();

        String beanName = getReferenceBeanName(attributes, interfaceClass);

        if (localServiceBean) {
            AbstractBeanDefinition beanDefinition = (AbstractBeanDefinition) beanFactory.getBeanDefinition(referencedBeanName);
            RuntimeBeanReference runtimeBeanReference = (RuntimeBeanReference) beanDefinition.getPropertyValues().get("ref");
            String serviceBeanName = runtimeBeanReference.getBeanName();
            beanFactory.registerAlias(serviceBeanName, beanName);
        } else {
            if (!beanFactory.containsBean(beanName)) {
                beanFactory.registerSingleton(beanName, referenceBean);
            }
        }
    }
    
    private void cacheInjectedReferenceBean(ReferenceBean referenceBean, InjectionMetadata.InjectedElement injectedElement) {
        if (injectedElement.getMember() instanceof Field) {
            injectedFieldReferenceBeanCache.put(injectedElement, referenceBean);
        } else if (injectedElement.getMember() instanceof Method) {
            injectedMethodReferenceBeanCache.put(injectedElement, referenceBean);
        }
    }

    private Object getOrCreateProxy(String referencedBeanName, ReferenceBean referenceBean, boolean localServiceBean, Class<?> serviceInterfaceType) {
        if (localServiceBean) {
            return newProxyInstance(getClassLoader(), new Class[]{serviceInterfaceType},
                    newReferencedBeanInvocationHandler(referencedBeanName));
        } else {
            exportServiceBeanIfNecessary(referencedBeanName); 
            return referenceBean.get();
        }
    }
    
    // ......
    
}
```
### ReferenceBean

ReferenceBean是通过ReferenceBeanBuilder的build方法，首先通过doBuild方法new一个空ReferenceBean对象，然后调用configureBean方法完成这个空对象的配置。

```
public class ReferenceBean<T> extends ReferenceConfig<T> implements FactoryBean,
        ApplicationContextAware, InitializingBean, DisposableBean {
        
    // ......
    
    @Override
    public void afterPropertiesSet() throws Exception {
    
        prepareDubboConfigBeans();

        if (init == null) {
            init = false;
        
        if (shouldInit()) {
            getObject();
        }
    }
    
    private void prepareDubboConfigBeans() {
        beansOfTypeIncludingAncestors(applicationContext, ApplicationConfig.class);
        beansOfTypeIncludingAncestors(applicationContext, ModuleConfig.class);
        beansOfTypeIncludingAncestors(applicationContext, RegistryConfig.class);
        beansOfTypeIncludingAncestors(applicationContext, ProtocolConfig.class);
        beansOfTypeIncludingAncestors(applicationContext, MonitorConfig.class);
        beansOfTypeIncludingAncestors(applicationContext, ProviderConfig.class);
        beansOfTypeIncludingAncestors(applicationContext, ConsumerConfig.class);
        beansOfTypeIncludingAncestors(applicationContext, ConfigCenterBean.class);
        beansOfTypeIncludingAncestors(applicationContext, MetadataReportConfig.class);
        beansOfTypeIncludingAncestors(applicationContext, MetricsConfig.class);
        beansOfTypeIncludingAncestors(applicationContext, SslConfig.class);
    }
}
```

## DubboBeanUtils

DubboBeanUtils在DubboConfigConfigurationRegistrar，DubboComponentScanRegistrar中都有被调用到。DubboBeanUtils使用的registerInfrastructureBean()方法其实就是调用BeanRegistrar.registerInfrastructureBean()方法来进行数据绑定。

```
public interface DubboBeanUtils {

    /**
     * Register the common beans
     *
     * @param registry {@link BeanDefinitionRegistry}
     * @see ReferenceAnnotationBeanPostProcessor
     * @see DubboConfigDefaultPropertyValueBeanPostProcessor
     * @see DubboConfigAliasPostProcessor
     * @see DubboLifecycleComponentApplicationListener
     * @see DubboBootstrapApplicationListener
     */
    static void registerCommonBeans(BeanDefinitionRegistry registry) {
    
        registerInfrastructureBean(registry, ReferenceAnnotationBeanPostProcessor.BEAN_NAME,
                ReferenceAnnotationBeanPostProcessor.class);

        registerInfrastructureBean(registry, DubboConfigAliasPostProcessor.BEAN_NAME,
                DubboConfigAliasPostProcessor.class);

        registerInfrastructureBean(registry, DubboLifecycleComponentApplicationListener.BEAN_NAME,
                DubboLifecycleComponentApplicationListener.class);
                
        registerInfrastructureBean(registry, DubboBootstrapApplicationListener.BEAN_NAME,
                DubboBootstrapApplicationListener.class);

        registerInfrastructureBean(registry, DubboConfigDefaultPropertyValueBeanPostProcessor.BEAN_NAME,
                DubboConfigDefaultPropertyValueBeanPostProcessor.class);
    }
}
```

### BeanRegistrar

BeanRegistrar.registerInfrastructureBean()方法是抽象出一个通过bean定义、bean处理器名称和bean处理器类型三个参数为注册bean数据绑定的一个公共方法。提取了如ReferenceAnnotationBeanPostProcessor，DubboConfigAliasPostProcessor等多种处理器的数据绑定公共部分逻辑。

```
public abstract class BeanRegistrar {

    // ......
    
    public static boolean registerInfrastructureBean(BeanDefinitionRegistry beanDefinitionRegistry, String beanName, Class<?> beanType) {
        boolean registered = false;
        if (!beanDefinitionRegistry.containsBeanDefinition(beanName)) {
            RootBeanDefinition beanDefinition = new RootBeanDefinition(beanType);
            beanDefinition.setRole(2);
            beanDefinitionRegistry.registerBeanDefinition(beanName, beanDefinition);
            registered = true;
            if (log.isInfoEnabled()) {
                log.info("The Infrastructure bean definition [" + beanDefinition + "with name [" + beanName + "] has been registered.");
            }
        }

        return registered;
    }
    
    // ......
    
}
```