---
title: Spring源码学习（第二阶段）- IoC容器核心实现
description: 深入理解BeanFactory体系结构、ApplicationContext实现、Bean生命周期管理、循环依赖解决方案和配置解析机制
date: 2025-09-19
lastModified: 2025-09-19
category: Tech
isPinned: false
isRecommended: false
---

# Spring源码学习（第二阶段）- IoC容器核心实现

## 🎯 学习目标

- 深入理解BeanFactory体系结构
- 掌握ApplicationContext的实现原理
- 理解Bean的完整生命周期
- 掌握Spring如何解决循环依赖
- 理解配置解析的底层机制

## 🏭 1. BeanFactory体系结构

### 1.1 BeanFactory继承体系

```
🏭 BeanFactory继承体系 - "工厂管理层级"

BeanFactory (顶级接口)
├── "基础工厂管理员" - 只能按名字找产品
│   └── getBean(String name)
│
├── ListableBeanFactory
│   └── "仓库管理员" - 能列出所有产品清单
│       ├── getBeanNamesForType(Class type)
│       └── getBeansOfType(Class type)
│
├── HierarchicalBeanFactory
│   └── "分级管理员" - 知道上级工厂是谁
│       └── getParentBeanFactory()
│
├── ConfigurableBeanFactory
│   └── "配置管理员" - 能调整工厂设置
│       ├── addBeanPostProcessor()
│       └── registerScope()
│
└── DefaultListableBeanFactory
    └── "全能工厂长" - 集所有能力于一身
        └── 实现了所有接口的功能
```

### 1.2 设计模式分析：接口隔离原则 + 组合模式

```java
// 接口隔离原则：不同角色只看到需要的接口
public class OrderService {
    // 只需要基本的获取Bean功能
    private BeanFactory beanFactory;

    public void processOrder() {
        PaymentService payment = beanFactory.getBean(PaymentService.class);
    }
}

public class BeanScanner {
    // 需要列举所有Bean的功能
    private ListableBeanFactory listableBeanFactory;

    public void scanAllBeans() {
        String[] beanNames = listableBeanFactory.getBeanDefinitionNames();
    }
}
```

### 1.3 DefaultListableBeanFactory核心数据结构

```java
public class DefaultListableBeanFactory extends AbstractAutowireCapableBeanFactory
        implements ConfigurableListableBeanFactory, BeanDefinitionRegistry, Serializable {

    // 🗂️ "产品目录" - 存储所有Bean的定义信息
    private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>(256);

    // 📋 "产品名单" - 按注册顺序存储Bean名称
    private volatile List<String> beanDefinitionNames = new ArrayList<>(256);

    // 🏭 "成品仓库" - 存储已创建的单例Bean
    private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

    // 🔧 "生产线" - Bean后置处理器列表
    private final List<BeanPostProcessor> beanPostProcessors = new ArrayList<>();
}
```

**生活化比喻**：
- **beanDefinitionMap**: 就像工厂的"产品设计图纸库"，每个图纸说明如何制造产品
- **beanDefinitionNames**: 就像"生产计划表"，按顺序列出要生产的产品
- **singletonObjects**: 就像"成品仓库"，存放已经生产好的产品
- **beanPostProcessors**: 就像"质检流水线"，每个产品都要经过这些检查

## 🏢 2. ApplicationContext实现

### 2.1 ApplicationContext vs BeanFactory

**生活化比喻**：
- **BeanFactory**: 像一个小作坊，只管生产产品
- **ApplicationContext**: 像一个现代化企业，不仅生产产品，还有人事部、财务部、市场部等

### 2.2 ClassPathXmlApplicationContext启动流程深度解析

#### 2.2.1 模板方法模式的经典应用

```java
// AbstractApplicationContext.refresh() - 模板方法
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        // 🎯 模板方法模式：定义算法骨架，子类实现具体步骤

        // 第1步：准备刷新上下文环境
        prepareRefresh();

        // 第2步：初始化BeanFactory，并进行XML文件读取
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // 第3步：对BeanFactory进行各种功能填充
        prepareBeanFactory(beanFactory);

        try {
            // 第4步：子类覆盖方法做额外处理 (钩子方法)
            postProcessBeanFactory(beanFactory);

            // 第5步：激活各种BeanFactory处理器
            invokeBeanFactoryPostProcessors(beanFactory);

            // 第6步：注册拦截Bean创建的Bean处理器
            registerBeanPostProcessors(beanFactory);

            // 第7步：为上下文初始化Message源，国际化处理
            initMessageSource();

            // 第8步：初始化应用消息广播器
            initApplicationEventMulticaster();

            // 第9步：留给子类来初始化其它的Bean (钩子方法)
            onRefresh();

            // 第10步：在所有注册的bean中查找Listener bean，注册到消息广播器中
            registerListeners();

            // 第11步：初始化剩下的单实例（非惰性的）
            finishBeanFactoryInitialization(beanFactory);

            // 第12步：完成刷新过程，通知生命周期处理器
            finishRefresh();
        } catch (BeansException ex) {
            destroyBeans();
            cancelRefresh(ex);
            throw ex;
        }
    }
}
```

#### 2.2.2 关键步骤详解

##### 步骤2：obtainFreshBeanFactory() - "建造工厂"

```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
    // 🏗️ 刷新BeanFactory
    refreshBeanFactory();

    // 🏭 获取BeanFactory
    ConfigurableListableBeanFactory beanFactory = getBeanFactory();
    return beanFactory;
}

// AbstractRefreshableApplicationContext.refreshBeanFactory()
protected final void refreshBeanFactory() throws BeansException {
    // 如果已经有工厂了，先销毁
    if (hasBeanFactory()) {
        destroyBeans();
        closeBeanFactory();
    }

    // 🎯 创建新的DefaultListableBeanFactory
    DefaultListableBeanFactory beanFactory = createBeanFactory();
    beanFactory.setSerializationId(getId());

    // 🔧 定制BeanFactory
    customizeBeanFactory(beanFactory);

    // 📖 加载Bean定义
    loadBeanDefinitions(beanFactory);

    synchronized (this.beanFactoryMonitor) {
        this.beanFactory = beanFactory;
    }
}
```

**生活化比喻**：这就像建造一个新工厂的过程
1. 先拆除旧工厂（如果有的话）
2. 建造新的厂房（DefaultListableBeanFactory）
3. 安装生产设备（customizeBeanFactory）
4. 导入生产图纸（loadBeanDefinitions）

##### 步骤11：finishBeanFactoryInitialization() - "批量生产"

```java
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
    // 🔧 设置类型转换服务
    if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
            beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
        beanFactory.setConversionService(
                beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
    }

    // 🏭 实例化所有剩余的（非lazy-init）单例Bean
    beanFactory.preInstantiateSingletons();
}
```

## 🔄 3. Bean生命周期管理

### 3.1 Bean的完整生命周期

```
🔄 Bean生命周期管理 - "从图纸到成品"

1. 📋 BeanDefinition注册    - "登记生产图纸"
2. 🏗️ 实例化 (Instantiation) - "制造毛坯"
3. 🔧 属性注入 (Population)   - "安装零部件"
4. 🎯 Aware接口回调         - "连接外部系统"
5. 🔍 BeanPostProcessor前置处理 - "质检1"
6. ⚡ InitializingBean回调   - "系统启动"
7. 🎛️ init-method执行       - "自定义初始化"
8. 🔍 BeanPostProcessor后置处理 - "质检2"
9. 🚀 Bean就绪可用          - "产品出厂"
10. 💥 销毁回调             - "产品回收"
```

### 3.2 设计模式分析：观察者模式 + 策略模式

```java
// Bean生命周期中的观察者模式
public class BeanLifecycleDemo {

    // 1. 实例化阶段 - 策略模式选择实例化方式
    public Object instantiate() {
        // 策略1：构造器实例化
        // 策略2：工厂方法实例化
        // 策略3：FactoryBean实例化
    }

    // 2. 属性注入阶段 - 观察者模式通知处理器
    public void populateBean() {
        // 通知所有InstantiationAwareBeanPostProcessor
        for (BeanPostProcessor processor : getBeanPostProcessors()) {
            if (processor instanceof InstantiationAwareBeanPostProcessor) {
                processor.postProcessPropertyValues(...);
            }
        }
    }

    // 3. 初始化阶段 - 观察者模式
    public void initializeBean() {
        // 前置处理 - 通知所有观察者
        applyBeanPostProcessorsBeforeInitialization();

        // 执行初始化
        invokeInitMethods();

        // 后置处理 - 通知所有观察者
        applyBeanPostProcessorsAfterInitialization();
    }
}
```

### 3.3 核心方法：doCreateBean()

```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args) {

    // 🏗️ 第1步：实例化Bean - "制造毛坯"
    BeanWrapper instanceWrapper = null;

    // 🔍 检查是否是单例且已经在缓存中
    if (mbd.isSingleton()) {
        // factoryBeanInstanceCache: 工厂Bean实例缓存
        instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
    }

    if (instanceWrapper == null) {
        // 🎯 核心：创建Bean实例（调用构造器或工厂方法）
        instanceWrapper = createBeanInstance(beanName, mbd, args);
    }

    // 🎁 获取实际的Bean对象（BeanWrapper是包装器）
    final Object bean = instanceWrapper.getWrappedInstance();
    Class<?> beanType = instanceWrapper.getWrappedClass();

    // 🔧 第2步：MergedBeanDefinitionPostProcessor处理 - "登记产品信息"
    synchronized (mbd.postProcessingLock) {
        if (!mbd.postProcessed) {
            // 🎯 让后置处理器处理合并后的Bean定义
            // 例如：@Autowired、@Value等注解的预处理
            applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
            mbd.postProcessed = true;
        }
    }

    // 🔄 第3步：早期单例暴露 - "解决循环依赖"
    boolean earlySingletonExposure = (
        mbd.isSingleton() &&                    // 是单例
        this.allowCircularReferences &&        // 允许循环引用
        isSingletonCurrentlyInCreation(beanName) // 当前正在创建中
    );

    if (earlySingletonExposure) {
        // 🏭 将Bean工厂放入三级缓存
        addSingletonFactory(beanName, new ObjectFactory<Object>() {
            @Override
            public Object getObject() throws BeansException {
                // 🎯 获取早期Bean引用（可能是代理对象）
                return getEarlyBeanReference(beanName, mbd, bean);
            }
        });
    }

    // 🎯 第4步：属性注入 - "安装零部件"
    Object exposedObject = bean;
    populateBean(beanName, mbd, instanceWrapper);

    // ⚡ 第5步：初始化Bean - "系统启动"
    exposedObject = initializeBean(beanName, exposedObject, mbd);

    // 🔍 第6步：循环依赖检查
    if (earlySingletonExposure) {
        Object earlySingletonReference = getSingleton(beanName, false);
        if (earlySingletonReference != null) {
            if (exposedObject == bean) {
                exposedObject = earlySingletonReference;
            }
            // ... 循环依赖冲突检查
        }
    }

    return exposedObject;
}
```

### 3.4 关键对象详解

#### BeanWrapper - "产品包装器"

```java
// BeanWrapper就像产品的包装盒
public interface BeanWrapper extends ConfigurablePropertyAccessor {
    // 获取被包装的实际对象
    Object getWrappedInstance();

    // 获取被包装对象的类型
    Class<?> getWrappedClass();

    // 设置属性值（支持嵌套属性）
    void setPropertyValue(String propertyName, Object value);
}

// 生活化比喻
BeanWrapper wrapper = new BeanWrapperImpl(new Student());
wrapper.setPropertyValue("name", "张三");        // 直接属性
wrapper.setPropertyValue("address.city", "北京"); // 嵌套属性
```

#### RootBeanDefinition - "产品设计图"

```java
// RootBeanDefinition包含了创建Bean所需的所有信息
public class RootBeanDefinition extends AbstractBeanDefinition {
    private Class<?> beanClass;           // Bean的类型
    private String factoryMethodName;     // 工厂方法名
    private ConstructorArgumentValues constructorArgumentValues; // 构造参数
    private MutablePropertyValues propertyValues; // 属性值
    private String initMethodName;        // 初始化方法
    private String destroyMethodName;     // 销毁方法
    // ... 更多配置信息
}
```

### 3.5 三种初始化方式及执行顺序

#### 3.5.1 三种初始化方式

1. **@PostConstruct注解** - JSR-250标准
2. **InitializingBean接口** - Spring接口
3. **init-method配置** - XML配置或@Bean注解

#### 3.5.2 执行顺序（重要！）

```
🔄 Bean初始化执行顺序 - "三道启动程序"

1️⃣ @PostConstruct        - "系统自检"
2️⃣ InitializingBean      - "Spring启动"
3️⃣ init-method          - "用户自定义启动"
```

#### 3.5.3 实际代码示例

```java
@Component
public class LifecycleBean implements InitializingBean, DisposableBean, BeanNameAware, BeanFactoryAware {

    // 构造器
    public LifecycleBean() {
        System.out.println("1. 构造器执行 - LifecycleBean()");
    }

    // Setter方法 - 属性注入时调用
    public void setName(String name) {
        this.name = name;
        System.out.println("2. 属性注入 - setName(): " + name);
    }

    // BeanNameAware接口 - Aware接口回调
    @Override
    public void setBeanName(String beanName) {
        this.beanName = beanName;
        System.out.println("3. BeanNameAware - setBeanName(): " + beanName);
    }

    // BeanFactoryAware接口 - Aware接口回调
    @Override
    public void setBeanFactory(BeanFactory beanFactory) {
        this.beanFactory = beanFactory;
        System.out.println("4. BeanFactoryAware - setBeanFactory()");
    }

    // @PostConstruct注解 - 第一种初始化方式
    @PostConstruct
    public void postConstruct() {
        System.out.println("5. @PostConstruct - postConstruct()");
    }

    // InitializingBean接口 - 第二种初始化方式
    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("6. InitializingBean - afterPropertiesSet()");
    }

    // init-method - 第三种初始化方式
    public void customInit() {
        System.out.println("7. init-method - customInit()");
    }

    // 业务方法
    public void doSomething() {
        System.out.println("8. Bean就绪 - doSomething(): " + name);
    }

    // @PreDestroy注解 - 销毁前回调
    @PreDestroy
    public void preDestroy() {
        System.out.println("9. @PreDestroy - preDestroy()");
    }

    // DisposableBean接口 - 销毁回调
    @Override
    public void destroy() throws Exception {
        System.out.println("10. DisposableBean - destroy()");
    }

    // destroy-method - 自定义销毁方法
    public void customDestroy() {
        System.out.println("11. destroy-method - customDestroy()");
    }
}
```

## 🥚 4. 循环依赖解决方案

### 4.1 什么是循环依赖？

**生活化比喻**：就像两个人互相等对方先说话
- A对象需要B对象
- B对象需要A对象
- 如果不特殊处理，就会无限等待

```java
// 循环依赖示例
@Component
public class ServiceA {
    @Autowired
    private ServiceB serviceB;  // A需要B
}

@Component
public class ServiceB {
    @Autowired
    private ServiceA serviceA;  // B需要A
}
```

### 4.2 Spring的"三级缓存"解决方案

#### 4.2.1 设计模式分析：缓存模式 + 工厂模式

```java
public class DefaultSingletonBeanRegistry {

    // 🏆 一级缓存：成品仓库 - 存放完全初始化好的Bean
    private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

    // 🔧 二级缓存：半成品仓库 - 存放实例化但未初始化的Bean
    private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);

    // 🏭 三级缓存：工厂仓库 - 存放Bean工厂
    private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
}
```

**生活化比喻**：
- **一级缓存**: 成品仓库，存放完全做好的产品
- **二级缓存**: 半成品仓库，存放做了一半的产品
- **三级缓存**: 工厂车间，存放生产产品的工厂

#### 4.2.2 循环依赖解决流程

```
🔄 循环依赖解决过程 - "A和B互相依赖"

1. 创建A对象
   ├── 实例化A (调用构造器)
   ├── 将A的工厂放入三级缓存
   └── 开始属性注入，发现需要B

2. 创建B对象
   ├── 实例化B (调用构造器)
   ├── 将B的工厂放入三级缓存
   └── 开始属性注入，发现需要A

3. 获取A对象
   ├── 一级缓存没有A
   ├── 二级缓存没有A
   ├── 三级缓存有A的工厂
   ├── 调用工厂创建A的早期引用
   ├── 将A放入二级缓存
   └── 返回A给B

4. B完成初始化
   ├── B获得了A的引用
   ├── B完成属性注入和初始化
   └── B放入一级缓存

5. A完成初始化
   ├── A获得了B的引用
   ├── A完成属性注入和初始化
   └── A放入一级缓存
```

#### 4.2.3 关键代码分析

```java
// DefaultSingletonBeanRegistry.getSingleton()
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    // 🏆 先从一级缓存获取
    Object singletonObject = this.singletonObjects.get(beanName);

    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        synchronized (this.singletonObjects) {
            // 🔧 再从二级缓存获取
            singletonObject = this.earlySingletonObjects.get(beanName);

            if (singletonObject == null && allowEarlyReference) {
                // 🏭 最后从三级缓存获取工厂
                ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                if (singletonFactory != null) {
                    // 调用工厂创建对象
                    singletonObject = singletonFactory.getObject();
                    // 放入二级缓存
                    this.earlySingletonObjects.put(beanName, singletonObject);
                    // 从三级缓存移除
                    this.singletonFactories.remove(beanName);
                }
            }
        }
    }
    return singletonObject;
}
```

## 📋 5. 配置解析机制

### 5.1 Spring支持的三种配置方式

```
📋 Spring配置方式演进史

1️⃣ XML配置 (Spring 1.x)     - "纸质图纸"
2️⃣ 注解配置 (Spring 2.x)    - "电子标签"
3️⃣ Java配置 (Spring 3.x)   - "程序化图纸"
```

### 5.2 XML配置解析

#### 5.2.1 XML解析流程详解

```
🔄 XML配置解析流程 - "从图纸到指令"

1️⃣ 资源加载 (Resource Loading)
   ├── ClassPathResource("config.xml")
   └── 转换为InputStream

2️⃣ DOM解析 (Document Parsing)
   ├── DocumentBuilderFactory创建解析器
   ├── 解析XML为Document对象
   └── 验证XML格式（XSD/DTD）

3️⃣ 元素遍历 (Element Traversal)
   ├── 获取根元素<beans>
   ├── 遍历子元素
   └── 区分默认/自定义命名空间

4️⃣ BeanDefinition创建
   ├── 解析<bean>元素属性
   ├── 创建GenericBeanDefinition
   └── 设置Bean的各种属性

5️⃣ 注册到容器
   ├── 存入beanDefinitionMap
   └── 添加到beanDefinitionNames
```

#### 5.2.2 设计模式分析：策略模式 + 模板方法模式

```java
// XML解析中的策略模式
public class XmlParsingDemo {

    // 策略模式：不同的BeanDefinitionReader处理不同格式
    public void loadBeanDefinitions() {
        BeanDefinitionReader reader;

        if (isXmlConfig()) {
            reader = new XmlBeanDefinitionReader(beanFactory);     // XML策略
        } else if (isAnnotationConfig()) {
            reader = new AnnotatedBeanDefinitionReader(beanFactory); // 注解策略
        } else {
            reader = new GroovyBeanDefinitionReader(beanFactory);   // Groovy策略
        }

        reader.loadBeanDefinitions(resource);
    }

    // 模板方法模式：解析流程固定，具体解析策略可变
    public void parseElement(Element element) {
        if (isDefaultNamespace(element)) {
            // 默认命名空间解析策略
            parseDefaultElement(element);
        } else {
            // 自定义命名空间解析策略
            parseCustomElement(element);
        }
    }
}
```

#### 5.2.3 核心类分析

##### XmlBeanDefinitionReader - "XML图纸解读器"

```java
public class XmlBeanDefinitionReader extends AbstractBeanDefinitionReader {

    // 🔧 核心方法：加载Bean定义
    public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
        // 1. 资源编码处理
        return loadBeanDefinitions(new EncodedResource(resource));
    }

    public int loadBeanDefinitions(EncodedResource encodedResource) {
        // 2. 获取输入流
        InputStream inputStream = encodedResource.getResource().getInputStream();
        InputSource inputSource = new InputSource(inputStream);

        // 3. 核心解析逻辑
        return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
    }

    protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource) {
        // 4. 解析XML为Document
        Document doc = doLoadDocument(inputSource, resource);

        // 5. 注册BeanDefinition
        return registerBeanDefinitions(doc, resource);
    }
}
```

##### BeanDefinitionParserDelegate - "解析委托者"

```java
public class BeanDefinitionParserDelegate {

    // 🎯 解析<bean>元素的核心方法
    public BeanDefinitionHolder parseBeanDefinitionElement(Element ele, BeanDefinition containingBean) {

        // 1. 解析id和name属性
        String id = ele.getAttribute(ID_ATTRIBUTE);
        String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);

        // 2. 处理别名
        List<String> aliases = new ArrayList<>();
        if (StringUtils.hasLength(nameAttr)) {
            String[] nameArr = StringUtils.tokenizeToStringArray(nameAttr, MULTI_VALUE_ATTRIBUTE_DELIMITERS);
            aliases.addAll(Arrays.asList(nameArr));
        }

        // 3. 确定beanName
        String beanName = id;
        if (!StringUtils.hasText(beanName) && !aliases.isEmpty()) {
            beanName = aliases.remove(0);
        }

        // 4. 解析Bean定义
        AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName, containingBean);

        // 5. 创建BeanDefinitionHolder
        return new BeanDefinitionHolder(beanDefinition, beanName, StringUtils.toStringArray(aliases));
    }
}
```

### 5.3 注解配置解析

#### 5.3.1 注解配置的优势

**生活化比喻**：
- **XML配置**: 像纸质说明书，需要单独维护
- **注解配置**: 像商品上的条形码，信息就在商品本身上

#### 5.3.2 设计模式分析：访问者模式 + 反射模式

```java
// 注解解析中的访问者模式
public class AnnotationParsingDemo {

    // 访问者模式：不同的BeanPostProcessor处理不同的注解
    public void processAnnotations(Object bean) {

        // AutowiredAnnotationBeanPostProcessor - 处理@Autowired
        if (hasAutowiredAnnotation(bean)) {
            autowiredProcessor.postProcessPropertyValues(bean);
        }

        // CommonAnnotationBeanPostProcessor - 处理@PostConstruct/@PreDestroy
        if (hasLifecycleAnnotation(bean)) {
            commonProcessor.postProcessBeforeInitialization(bean);
        }

        // ConfigurationClassPostProcessor - 处理@Configuration
        if (hasConfigurationAnnotation(bean)) {
            configurationProcessor.postProcessBeanDefinitionRegistry(bean);
        }
    }
}
```

#### 5.3.3 关键注解处理器

##### AutowiredAnnotationBeanPostProcessor - "@Autowired处理器"

```java
public class AutowiredAnnotationBeanPostProcessor implements BeanPostProcessor {

    // 🔍 扫描@Autowired注解
    public PropertyValues postProcessPropertyValues(PropertyValues pvs, PropertyDescriptor[] pds,
                                                   Object bean, String beanName) {

        // 1. 查找需要注入的字段和方法
        InjectionMetadata metadata = findAutowiringMetadata(beanName, bean.getClass(), pvs);

        // 2. 执行注入
        metadata.inject(bean, beanName, pvs);

        return pvs;
    }

    private InjectionMetadata findAutowiringMetadata(String beanName, Class<?> clazz, PropertyValues pvs) {

        // 🎯 反射扫描类的字段和方法
        Class<?> targetClass = clazz;
        List<InjectionMetadata.InjectedElement> elements = new ArrayList<>();

        while (targetClass != null && targetClass != Object.class) {
            List<InjectionMetadata.InjectedElement> currElements = new ArrayList<>();

            // 扫描字段上的@Autowired注解
            ReflectionUtils.doWithLocalFields(targetClass, field -> {
                AnnotationAttributes ann = findAutowiredAnnotation(field);
                if (ann != null) {
                    currElements.add(new AutowiredFieldElement(field, ann.getBoolean("required")));
                }
            });

            // 扫描方法上的@Autowired注解
            ReflectionUtils.doWithLocalMethods(targetClass, method -> {
                AnnotationAttributes ann = findAutowiredAnnotation(method);
                if (ann != null) {
                    currElements.add(new AutowiredMethodElement(method, ann.getBoolean("required")));
                }
            });

            elements.addAll(0, currElements);
            targetClass = targetClass.getSuperclass();
        }

        return new InjectionMetadata(clazz, elements);
    }
}
```

##### CommonAnnotationBeanPostProcessor - "JSR-250注解处理器"

```java
public class CommonAnnotationBeanPostProcessor implements BeanPostProcessor {

    // 🎯 处理@PostConstruct注解
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        LifecycleMetadata metadata = findLifecycleMetadata(bean.getClass());
        metadata.invokeInitMethods(bean, beanName);
        return bean;
    }

    // 🎯 处理@PreDestroy注解
    public void postProcessBeforeDestruction(Object bean, String beanName) {
        LifecycleMetadata metadata = findLifecycleMetadata(bean.getClass());
        metadata.invokeDestroyMethods(bean, beanName);
    }

    private LifecycleMetadata findLifecycleMetadata(Class<?> clazz) {

        // 扫描@PostConstruct和@PreDestroy注解
        List<LifecycleElement> initMethods = new ArrayList<>();
        List<LifecycleElement> destroyMethods = new ArrayList<>();

        Class<?> targetClass = clazz;
        while (targetClass != null && targetClass != Object.class) {

            ReflectionUtils.doWithLocalMethods(targetClass, method -> {
                if (method.isAnnotationPresent(PostConstruct.class)) {
                    initMethods.add(new LifecycleElement(method));
                }
                if (method.isAnnotationPresent(PreDestroy.class)) {
                    destroyMethods.add(new LifecycleElement(method));
                }
            });

            targetClass = targetClass.getSuperclass();
        }

        return new LifecycleMetadata(clazz, initMethods, destroyMethods);
    }
}
```

### 5.4 Java配置解析

#### 5.4.1 Java配置的特点

**生活化比喻**：
- **XML配置**: 像填表格，格式固定
- **注解配置**: 像贴标签，简单直接
- **Java配置**: 像写程序，灵活强大

#### 5.4.2 Java配置解析机制

##### ConfigurationClassPostProcessor - "Java配置处理器"

```java
public class ConfigurationClassPostProcessor implements BeanDefinitionRegistryPostProcessor {

    // 🎯 核心方法：处理@Configuration类
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {

        // 1. 查找所有@Configuration类
        Set<BeanDefinitionHolder> configCandidates = new LinkedHashSet<>();
        String[] candidateNames = registry.getBeanDefinitionNames();

        for (String beanName : candidateNames) {
            BeanDefinition beanDef = registry.getBeanDefinition(beanName);
            if (ConfigurationClassUtils.isConfigurationCandidate(beanDef)) {
                configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
            }
        }

        // 2. 解析@Configuration类
        ConfigurationClassParser parser = new ConfigurationClassParser(...);
        parser.parse(configCandidates);

        // 3. 注册@Bean方法产生的BeanDefinition
        this.reader.loadBeanDefinitions(parser.getConfigurationClasses());
    }
}
```

#### 5.4.3 设计模式分析：建造者模式 + 代理模式

```java
// Java配置中的建造者模式
public class JavaConfigDemo {

    @Configuration
    public class DatabaseConfig {

        // 建造者模式：逐步构建复杂对象
        @Bean
        public DataSource dataSource() {
            return DataSourceBuilder.create()
                    .driverClassName("com.mysql.jdbc.Driver")
                    .url("jdbc:mysql://localhost:3306/test")
                    .username("root")
                    .password("password")
                    .build();
        }

        @Bean
        public JdbcTemplate jdbcTemplate(DataSource dataSource) {
            return new JdbcTemplate(dataSource);
        }
    }

    // 代理模式：@Configuration类会被CGLIB增强
    // 确保@Bean方法的单例语义
    @Configuration
    public class EnhancedConfig {

        @Bean
        public ServiceA serviceA() {
            return new ServiceA(serviceB()); // 多次调用返回同一个实例
        }

        @Bean
        public ServiceB serviceB() {
            return new ServiceB();
        }
    }
}
```

#### 5.4.4 @Configuration类的增强机制

```java
// CGLIB增强后的@Configuration类
public class ConfigClass$$EnhancedBySpringCGLIB extends ConfigClass {

    @Override
    public ServiceB serviceB() {
        // 🎯 增强逻辑：检查容器中是否已存在
        String beanName = "serviceB";
        if (this.beanFactory.containsSingleton(beanName)) {
            return (ServiceB) this.beanFactory.getBean(beanName);
        }

        // 如果不存在，调用原始方法创建
        ServiceB bean = super.serviceB();

        // 注册到容器
        this.beanFactory.registerSingleton(beanName, bean);
        return bean;
    }
}
```

## 📊 5.5 三种配置方式对比

| 配置方式 | 优点 | 缺点 | 适用场景 |
|---------|------|------|----------|
| **XML配置** | 集中管理、解耦 | 冗长、类型不安全 | 大型项目、第三方库 |
| **注解配置** | 简洁、类型安全 | 分散、编译时绑定 | 业务代码、快速开发 |
| **Java配置** | 灵活、可编程 | 学习成本高 | 复杂配置、条件装配 |

## 🎯 6. 配置解析机制总结

### 6.1 设计模式应用总结

1. **策略模式** - 不同的BeanDefinitionReader处理不同配置格式
2. **模板方法模式** - 配置解析的固定流程
3. **访问者模式** - BeanPostProcessor处理不同注解
4. **建造者模式** - 复杂Bean的构建过程
5. **代理模式** - @Configuration类的CGLIB增强
6. **工厂模式** - BeanDefinition的创建
7. **缓存模式** - 三级缓存解决循环依赖

### 6.2 配置解析的核心流程

```
🔄 配置解析统一流程

1️⃣ 配置发现 (Configuration Discovery)
   ├── XML文件扫描
   ├── 注解类扫描
   └── Java配置类扫描

2️⃣ 元数据提取 (Metadata Extraction)
   ├── 解析配置信息
   ├── 提取Bean定义
   └── 处理依赖关系

3️⃣ BeanDefinition创建
   ├── 创建GenericBeanDefinition
   ├── 设置Bean属性
   └── 处理作用域和生命周期

4️⃣ 注册到容器
   ├── 存入beanDefinitionMap
   ├── 添加到beanDefinitionNames
   └── 处理别名和依赖
```

## 📚 7. 核心知识点回顾

### 7.1 BeanFactory体系结构
- 接口隔离原则的完美体现
- DefaultListableBeanFactory的核心数据结构
- 不同接口的职责分工

### 7.2 ApplicationContext实现
- 模板方法模式的经典应用
- 12个启动步骤的协调配合
- 企业级功能的无缝集成

### 7.3 Bean生命周期管理
- 10个生命周期阶段的精确控制
- 观察者模式的广泛应用
- 三种初始化方式的执行顺序

### 7.4 循环依赖解决方案
- 三级缓存的巧妙设计
- 缓存模式和工厂模式的结合
- 早期引用暴露的时机控制

### 7.5 配置解析机制
- 三种配置方式的统一处理
- 策略模式的灵活应用
- BeanDefinition的创建和注册

## 🎨 8. 设计模式应用汇总

| 设计模式 | 应用场景 | 核心价值 |
|---------|----------|----------|
| **工厂模式** | BeanFactory体系 | 统一对象创建 |
| **单例模式** | Bean作用域管理 | 确保唯一实例 |
| **模板方法模式** | ApplicationContext.refresh() | 固定算法骨架 |
| **策略模式** | 配置解析、实例化策略 | 算法可替换 |
| **观察者模式** | Bean生命周期回调 | 事件通知机制 |
| **代理模式** | AOP、@Configuration增强 | 功能增强 |
| **缓存模式** | 三级缓存 | 性能优化 |
| **访问者模式** | 注解处理 | 操作与结构分离 |



