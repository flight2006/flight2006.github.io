---
layout: post
category: spring
title: spring学习笔记之IoC原理
tagline: by flight
tags: spring IoC
---
Spring另外一个核心工具便是IoC容器，本篇主要理清IoC源码的骨架，分析其内部工作原理，使在阅读Spring这部分源码时能大致清楚某部分代码的作用。

<!--more-->

#IoC容器
最主要完成了对象的创建和依赖的管理注入等。

##控制反转（Inversion of Control ,IoC)
所谓控制反转就是应用本身不负责依赖对象的创建及维护，依赖对象的创建及维护由外部容器来负责。这样控制权就由应用转移到了外部容器，控制权的转移就是所谓反转。

##依赖注入(Dependency Injection)
所谓依赖注入就是指:在运行期间，由外部容器动态地将依赖对象注入到组件中（构造方法和set方法)

#主要接口
    Resource
    BeanDefinition
    BeanDefinitionReader
    BeanFactory
    ApplicationContext

以上五个都是接口，都有各式各样的实现，正是这 5 个接口定义了 spring ioc 容器的基本代码组件结构。而其组件各种实现的组合关系组成了一个运行时的具体容器。
#各组件详解
##1.resource
是对资源的抽象，每一个接口实现类都代表了一种资源类型，如 ClasspathResource 、 URLResource ， FileSystemResource 等。每一个资源类型都封装了对某一种特定资源的访问策略。它是 spring 资源访问策略的一个基础实现，应用在很多场景。
![](http://dl.iteye.com/upload/attachment/536177/ead09fcb-3c6c-3740-9e36-6de33fd65cca.jpg)

具体可以参考文章 :
[Spring 资源访问剖析和策略模式应用](http://www.ibm.com/developerworks/cn/java/j-lo-spring-resource/index.html)

##2.BeanDefinition
org.springframework.beans.factory.config.BeanDefinition用来抽象和描述一个具体bean对象，是描述一个bean对象的基本数据结构。即是<bean>元素标签在容器内部的表现形式。
![](http://imgur.com/LNrN1Zi.png)

BeanDefinition的类继承结构图如上图所示，RootBeanDefinition是最常用的实现类，对应一般性的<bean>元素标签。配置文件中可以定义父<bean>和子<bean>，顾名思义其中父<bean>就是用RootBeanDefinition表示，子<bean>就用ChildBeanDefinition表示，而没有父<bean>的<bean>就使用RootBeanDefinition表示。AbstractBeanDefinition对两者共同的类信息进行抽象。
###创建BeanDefinition主要的两个步骤:
 - 利用BeanDefinitionReader对配置信息Resource进行读取，通过XML解析器解析配置信息的DOM对象，简单地为每个<bean>生成对应的BeanDefinition对象。但是这里生成的BeanDefinition是个半成品，因为在配置文件中，我们可能通过占位符变量引用外部属性文件的属性，这些占位符变量在这一步还没被解析出来。
 - 利用容器中注册的BeanFactoryPostProcessor对半成品的BeanDefinition进行加工处理，将以占位符表示的配置解析为最终的实际值，这样就成了成品的BeanDefinition

##3.BeanDefinitionReader
BeanDefinitionReader 将外部资源对象描述的 bean 定义统一转化为统一的内部数据结构 BeanDefinition 。对应不同的描述需要有不同的 Reader 。如 XmlBeanDefinitionReader 用来读取 xml 描述配置的 bean 对象。
![](http://dl.iteye.com/upload/attachment/536179/c4143d16-02e2-3d7b-9734-4d19a9a984dd.jpg)

##4.ApplicationContext
 BeanFactory 实现了一个容器基本结构和功能，但是与外部环境隔离。那么读取配置文件，并将配置文件解析成 BeanDefinition ，然后注册到 BeanFactory 的这一个过程的封装自然就需要 ApplicationContext 。 ApplicationContext 和应用环境细细相关，常见实现有 ClasspathXmlApplicationContext,FileSystemXmlApplicationContext,WebApplicationContext 等。
![](http://dl.iteye.com/upload/attachment/536183/c456f949-7b9c-34db-ad1a-ca3141219b6d.jpg)

AbstractApplicationContext是ApplicationContext的抽象实现类，该抽象类的`refresh()`方法定义了Spring容器在加载配置文件后的各项处理过程，这些处理过程清晰刻画了Spring容器启动时的各项操作。
refresh()的基本步骤为:
 - 1.把配置xml文件转换成resource。resource的转换是先通过ResourcePatternResolver来解析可识别格式的配置文件的路径(如"classpath*:"等)，如果没有指定格式，默认会按照类路径的资源来处理。 
 - 2.利用XmlBeanDefinitionReader完成对xml的解析，将xml Resource里定义的bean对象转换成统一的BeanDefinition。
 - 3.将BeanDefinition注册到BeanFactory，完成对BeanFactory的初始化。BeanFactory里将会维护一个BeanDefinition的Map。
    

    public void refresh() throws BeansException, IllegalStateException {
	    synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// Prepare the bean factory for use in this context.
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
				postProcessBeanFactory(beanFactory);

				// Invoke factory processors registered as beans in the context.
				invokeBeanFactoryPostProcessors(beanFactory);

				// Register bean processors that intercept bean creation.
				registerBeanPostProcessors(beanFactory);

				// Initialize message source for this context.
				initMessageSource();

				// Initialize event multicaster for this context.
				initApplicationEventMulticaster();

				// Initialize other special beans in specific context subclasses.
				onRefresh();

				// Check for listener beans and register them.
				registerListeners();

				// Instantiate all remaining (non-lazy-init) singletons.
				finishBeanFactoryInitialization(beanFactory);

				// Last step: publish corresponding event.
				finishRefresh();
			}

			catch (BeansException ex) {
				// Destroy already created singletons to avoid dangling resources.
				beanFactory.destroySingletons();

				// Reset 'active' flag.
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			}
		}
	}

 以上的obtainFreshBeanFactory是很关键的一个方法，里面会调用loadBeanDefinition方法，如下：
 
     	protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws IOException {
    		// Create a new XmlBeanDefinitionReader for the given BeanFactory.
    		XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);
    		
		// Configure the bean definition reader with this context's
		// resource loading environment.
		beanDefinitionReader.setResourceLoader(this);
		beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

		// Allow a subclass to provide custom initialization of the reader,
		// then proceed with actually loading the bean definitions.
		initBeanDefinitionReader(beanDefinitionReader);
		loadBeanDefinitions(beanDefinitionReader);
	}

 LoadBeanDifinition方法很关键，这里特定于整个IOC容器，实例化了一个XmlBeanDefinitionReader来解析Resource文件。关于Resource文件如何初始化和xml文件如何解析都在
 
    loadBeanDefinitions(beanDefinitionReader);
    
 里面的层层调用完成，这里不在累述。
 
 refresh()方法初始化容器的过程可以用下图解释:
 ![](http://dl.iteye.com/upload/attachment/558068/2187e288-5c0b-313b-8053-82992267fab6.jpg)

##5.BeanFactory

用来定义一个很纯粹的 bean 容器。它是一个 bean 容器的必备结构。同时和外部应用环境等隔离。 BeanDefinition 是它的基本数据结构。它维护一个 BeanDefinitions Map, 并可根据 BeanDefinition 的描述进行 bean 的创建和管理。

![](http://dl.iteye.com/upload/attachment/536181/24095923-75cd-363b-bc2f-9fbc603c341f.jpg)

#小结
本篇主要介绍了Spring IoC主要的5个接口，以及初始化容器的过程。知道了IoC的主要接口，可以根据这几个接口为主要路线去阅读源代码。
