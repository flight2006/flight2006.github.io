---
layout: post
category: spring
title: spring学习笔记之aop动态代理
tagline: by flight
tags: spring aop
---
Spring AOP基础——动态代理

<!--more-->

# JDK动态代理
JDK的动态代理主要涉及到java.lang.reflect包中的两个Proxy和InvocationHandler。InvocationHandler是一个接口，可以通过实现该接口定义横切逻辑，并通过反射机制调用目标类的代码，动态将横切逻辑和业务逻辑编织在一起。
业务逻辑：
	package cn.hdu.proxy;

	public class ForumServiceImpl implements ForumService {

		@Override
		public void removeTopic(int topicid){
			System.out.println("模拟删除topic记录:" + topicid);
			try{
				Thread.currentThread().sleep(20);
			}catch(Exception e){
				throw new RuntimeException();
			}
		}
		
		@Override
		public void removeForum(int forumid){
			System.out.println("模拟删除forum记录:" + forumid);
			try{
				Thread.currentThread().sleep(40);
			}catch(Exception e){
				throw new RuntimeException();
			}
		}
	}
假设在业务逻辑中需要加入性能监控代码，监控代码PerformanceHandler代码如下
	package cn.hdu.proxy;
	import java.lang.reflect.InvocationHandler;
	import java.lang.reflect.Method;

	public class PerformanceHandler implements InvocationHandler {

		private Object target;
		public PerformanceHandler(Object target){
			this.target = target;
		}
		@Override
		public Object invoke(Object proxy, Method method, Object[] args)
				throws Throwable {
			PerformanceMonitor.begin(target.getClass().getName()+"."+method.getName());
			Object obj = method.invoke(target, args);
			PerformanceMonitor.end();
			return obj;
		}

	}
begin和end方法为性能监控的横切代码，method.invoke()方法通过java反射机制间接调用目标对象的方法，这样InvocationHandler的invoke()方法就将横切逻辑代码和业务类方法的逻辑代码编织在一起了。
其中invoke(Object proxy, Method method, Object[] args)方法中，proxy是最终生成的代理实例；method是被代理目标实例的某个具体方法，args是通过被代理实例某个方法的参数。在构造函数中通过target传入希望被代理的目标对象,并将该实例传给method.invoke()方法。
下面通过Proxy结合PerformanceHandler创建ForumService接口的代理实例：
	package cn.hdu.proxy;
	import java.lang.reflect.Proxy;

	public class TestFormService {
		public static void main(String[] args){
			ForumService target = new ForumServiceImpl();//①
			PerformanceHandler handler = new PerformanceHandler(target);//②
			ForumService proxy = (ForumService) Proxy.newProxyInstance(  //③
					target.getClass().getClassLoader(),
					target.getClass().getInterfaces(),
					handler);
			proxy.removeForum(10);
			proxy.removeTopic(1024);

		}

	}
上面代码完成业务逻辑代码和横切代码的编织工作并生成了代理实例。②处将目标业务类和横切代码编织在一起，③处根据编织完成后的InvocationHandler实例创建实例，该方法第一个参数为目标类加载器，第二个是创建实例所需要实现的一组接口，第三个是编织器对象。
输出信息如下：
	begin monitor...
	模拟删除forum记录:10
	end monitor
	cn.hdu.proxy.ForumServiceImpl.removeForum花费40毫秒
	begin monitor...
	模拟删除topic记录:1024
	end monitor
	cn.hdu.proxy.ForumServiceImpl.removeTopic花费20毫秒


#CGLib动态代理
JDK动态代理的一个限制为，只能为接口创建代理实例，而CGlib可以弥补这个缺陷，CGLib采用字节码技术，可以为一个类创建子类，并在子类采用方法拦截的技术拦截所有方法的调用，并顺势织入横切逻辑。下面是一个可以为任何类创建织入性能监视横切逻辑代理对象的代理器：
