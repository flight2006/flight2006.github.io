---
layout: post
category: spring
title: spring学习笔记之aop动态代理
tagline: by flight
tags: spring aop
---
Spring AOP基础——动态代理,Spring AOP使用动态代理技术在运行期织入增强的代码，为了揭示Spring AOP底层的工作机理，有必要对涉及到的Java动态代理知识进行学习。Spring AOP底层使用两种代理机制：一种是基于JDK的动态代理；另一种是基于CGLib的动态代理。之所以需要两种代理机制，很大程度上是因为JDK本身只提供接口的代理，而不支持类的代理。

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
JDK动态代理的一个限制为，只能为接口创建代理实例，而CGLib可以弥补这个缺陷，CGLib采用字节码技术，可以为一个类创建子类，并在子类采用方法拦截的技术拦截所有方法的调用，并顺势织入横切逻辑。下面是一个可以为任何类创建织入性能监视横切逻辑代理对象的代理器：

	public class CglibProxy implements MethodInterceptor{
		private Enhancer enhancer = new Enhancer();
		public Object getProxy(Class clazz){
			enhancer.setSuperclass(clazz);   //①
			enhancer.setCallback(this);		
			return enhancer.create();		//②
		}
		public Object intercept(Object obj,Method method,Object[] args,MethodProxy proxy) throws Throwable{     //③
			PerformanceMonitor.begin(obj.getClass().getName()+'.'+method.getName());
			Object result = proxy.invokeSuper(obj,args);
			PerformanceMonitor.end();
			return result;
		}
	}

①处设置需要创建子类的类
②处通过字节码技术动态创建子类实例
③处拦截弗雷所有方法的调用
用户可以通过getProxy(Class clazz)为一个类创建动态代理对象，该代理对象通过扩展class创建代理对象。interceptor方法是CGLib定义的Interceptor接口的方法，它拦截所有目标类方法的调用，obj表示目标类的实例；method为目标类方法的反射对象；args为方法的动态入参；proxy为代理类实例。
下面代码创建代理对象，并测试代理对象的方法：

public class TestForumService(String[] args){
	CglibProxy proxy = new CglibProxy();
	ForumServiceImpl forumService =
						(ForumServiceImpl)proxy.getProxy(ForumServiceImpl.class);
	forumService.removeForum(10);
	forumService.removeTopic(1023);
}

通过CglibProxy为ForumServiceImpl动态创建了一个织入性能监视逻辑的代理对象，并调用代理类的业务方法。运行结果如下

	begin monitor...
	模拟删除forum记录:10
	end monitor
	cn.hdu.proxy.ForumServiceImpl$$EnhancerByCGLIB$$44025b6d.removeForum花费52毫秒
	begin monitor...
	模拟删除topic记录:1023
	end monitor
	cn.hdu.proxy.ForumServiceImpl$$EnhancerByCGLIB$$44025b6d.removeTopic花费20毫秒

上面结果显示代理类的名字是cn.hdu.proxy.ForumServiceImpl$$EnhancerByCGLIB$$44025b6d，这个类就是CGlib为ForumServiceImpl动态创建的子类。

#总结
动态代理实质上就是使用代理对象对被包装对象请求进行'拦截''增强'，然后用代理再转发给这些被包装对象，AOP就是基于该思想为目标Bean织入横切逻辑。Spring AOP通过Pointcut(切点)指定哪些类的哪些方法上织入横切逻辑，通过Advice(增强)描述横切逻辑和方法的具体植入点(方法前、方法后、方法的两端等)。此外，Spring通过Advisor(切面)将Pointcut和Advice两者组装起来。有了Advisor的信息，Spring就可以利用JDK或CGLib的动态代理技术采用同意的方式为目标Bean创建织入切面的代理对象了。

