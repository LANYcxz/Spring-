---
title: Spring源码学习（第一阶段）- 基础准备与核心概念理解
description: Spring整体架构认知、核心概念深度理解、设计模式应用分析
date: 2025-09-12
lastModified: 2025-09-12
category: Tech
isPinned: false
isRecommended: false
---

# Spring源码学习（第一阶段）- 基础准备与核心概念理解

## 🎯 学习目标

- 建立Spring整体认知，理解核心概念
- 掌握Spring中的关键设计模式应用
- 理解Spring容器的启动流程

## 🏗️ Spring整体架构

### Spring架构的"四大金刚"

```
📦 Spring Framework
├── 🏭 Core Container (核心容器) - "工厂管理系统"
│   ├── spring-core      - 基础工具类
│   ├── spring-beans     - Bean定义和管理
│   ├── spring-context   - 应用上下文
│   └── spring-expression - 表达式语言
├── 🎯 AOP (面向切面编程) - "质量检测系统"
│   └── spring-aop       - 切面编程支持
├── 💾 Data Access (数据访问) - "仓储系统"
│   ├── spring-jdbc      - JDBC支持
│   └── spring-tx        - 事务管理
└── 🌐 Web (Web层) - "销售展示系统"
    ├── spring-web       - Web基础
    └── spring-webmvc    - MVC框架
```

### 生活化比喻

**Spring就像一个现代化的汽车工厂**：
- **Core Container**: 工厂的管理系统，负责零部件的生产和装配
- **AOP**: 质量检测系统，在关键节点自动进行检测
- **Data Access**: 仓储系统，管理原材料的存取
- **Web**: 销售展示系统，对外提供服务

## 🏭 核心概念深度理解

### 1. IoC容器 - "智能工厂管理系统"

#### 核心原理
- **控制反转**: 对象创建的控制权从程序员转移给了Spring容器
- **依赖注入**: 容器自动将依赖的对象注入到目标对象中

#### 代码对比
```java
// 传统方式 - 你需要自己管理对象
public class TraditionalWay {
    public void doSomething() {
        // 😰 需要手动创建和管理依赖
        Student student = new Student();
        SimpleBean bean = new SimpleBean(student);
        bean.send();
    }
}

// Spring方式 - IoC容器帮你管理
public class SpringWay {
    public void doSomething() {
        // 😊 容器自动管理，你只需要"要"
        ApplicationContext context = new ClassPathXmlApplicationContext("config.xml");
        SimpleBean bean = context.getBean(SimpleBean.class);
        bean.send(); // student已经自动注入了！
    }
}
```

### 2. AOP - "质量检测系统"

#### 核心原理
- **代理模式**: Spring在运行时为目标对象创建代理对象
- **方法拦截**: 代理对象拦截方法调用，在调用前后执行额外逻辑
- **透明性**: 客户端代码无感知，依然调用原始接口

#### 工作流程
```
客户端调用 → 代理对象 → 拦截器链 → 目标对象
           ↑         ↓
         AOP增强   原始方法
```

#### 配置示例
```xml
<aop:config expose-proxy="true">
    <aop:advisor advice-ref="simpleMethodInterceptor"
                 pointcut="execution(* aop.SimpleAopBean.*(..))" />
</aop:config>
```

## 🎨 设计模式应用分析

### 1. 工厂模式 + 单例模式 (IoC容器)

**应用场景**: Bean的创建和管理
- **工厂模式**: BeanFactory负责创建各种类型的Bean
- **单例模式**: 默认情况下Bean都是单例的

### 2. 代理模式 + 装饰器模式 + 责任链模式 (AOP)

**应用场景**: 切面编程实现
- **代理模式**: 为目标对象创建代理
- **装饰器模式**: 在不修改原对象的基础上增加功能
- **责任链模式**: 多个拦截器形成调用链

### 3. 模板方法模式 (容器启动流程)

**应用场景**: ApplicationContext的refresh()方法

```java
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        // 模板方法模式：定义了固定的算法骨架
        prepareRefresh();                    // 步骤1：准备
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory(); // 步骤2：获取工厂
        prepareBeanFactory(beanFactory);     // 步骤3：配置工厂

        try {
            postProcessBeanFactory(beanFactory);     // 🎯 钩子方法：子类扩展点
            invokeBeanFactoryPostProcessors(beanFactory);
            registerBeanPostProcessors(beanFactory);
            initMessageSource();
            initApplicationEventMulticaster();
            onRefresh();                     // 🎯 钩子方法：子类扩展点
            registerListeners();
            finishBeanFactoryInitialization(beanFactory);
            finishRefresh();
        } catch (BeansException ex) {
            destroyBeans();
            cancelRefresh(ex);
            throw ex;
        }
    }
}
```

**设计精髓**：
- **模板方法模式**: refresh()定义了容器启动的标准流程
- **钩子方法**: postProcessBeanFactory()、onRefresh()等让子类可以定制行为
- **策略模式**: 不同的ApplicationContext实现可以有不同的策略

## 🚀 Spring启动流程

### Spring启动的"十二道工序"

```
🏭 Spring容器启动流程 (refresh方法的12个步骤)
1. prepareRefresh()          - 准备工作，设置启动时间
2. obtainFreshBeanFactory()  - 创建BeanFactory，解析配置文件
3. prepareBeanFactory()      - 配置BeanFactory的基本属性
4. postProcessBeanFactory()  - 子类扩展点
5. invokeBeanFactoryPostProcessors() - 执行BeanFactory后置处理器
6. registerBeanPostProcessors()      - 注册Bean后置处理器
7. initMessageSource()       - 国际化支持
8. initApplicationEventMulticaster() - 事件广播器
9. onRefresh()              - 子类扩展点
10. registerListeners()     - 注册监听器
11. finishBeanFactoryInitialization() - 实例化所有单例Bean
12. finishRefresh()         - 完成刷新，发布事件
```

### 启动流程的设计原因

- **职责分离**: 每个步骤负责特定的初始化工作，便于维护和扩展
- **依赖顺序**: 某些组件依赖其他组件，必须按顺序初始化
- **扩展点**: 提供多个扩展点让开发者可以在特定阶段介入
- **异常处理**: 分步骤便于定位问题和异常恢复
- **性能优化**: 可以在不同阶段进行不同的优化策略

## 🤔 思考题与答案

### 1. 为什么Spring要使用IoC容器而不是直接new对象？

**答案**：
- **解耦合**: 直接new对象会造成强耦合，修改依赖时需要修改所有相关代码
- **统一管理**: IoC容器可以统一管理对象的生命周期、作用域、初始化等
- **配置灵活**: 可以通过配置文件或注解灵活地改变对象的创建方式
- **AOP支持**: 只有容器管理的对象才能享受AOP等高级特性
- **单例保证**: 容器可以确保单例对象在整个应用中只有一个实例

### 2. AOP是如何在不修改原始代码的情况下增加功能的？

**答案**：
- **代理模式**: Spring在运行时为目标对象创建代理对象
- **方法拦截**: 代理对象拦截方法调用，在调用前后执行额外逻辑
- **字节码增强**: 使用JDK动态代理或CGLIB生成增强的子类
- **透明性**: 客户端代码无感知，依然调用原始接口

### 3. Spring的启动流程为什么要分这么多步骤？

**答案**：
- **职责分离**: 每个步骤负责特定的初始化工作，便于维护和扩展
- **依赖顺序**: 某些组件依赖其他组件，必须按顺序初始化
- **扩展点**: 提供多个扩展点让开发者可以在特定阶段介入
- **异常处理**: 分步骤便于定位问题和异常恢复
- **性能优化**: 可以在不同阶段进行不同的优化策略

## 📚 关键知识点总结

1. **Spring整体架构**: 四大模块的职责分工
2. **核心概念**: IoC容器、AOP、Bean的基本概念
3. **设计模式应用**:
   - 工厂模式 + 单例模式 (IoC容器)
   - 代理模式 + 装饰器模式 (AOP)
   - 模板方法模式 (容器启动流程)
4. **启动流程**: Spring容器的12个启动步骤

## 🎯 下一阶段预告

接下来我们将深入**第二阶段：IoC容器核心实现**，详细分析：
- BeanFactory的工作原理
- Bean的生命周期管理
- Spring如何解决循环依赖
- 配置解析机制的实现

---

*学习建议：理论与实践结合，每学习一个知识点都要动手调试源码验证*
