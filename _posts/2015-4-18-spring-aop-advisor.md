---
layout: post
category: spring
title: spring学习笔记之aop增强、切面
tagline: by flight
tags: spring aop
---
前面说到Spring AOP通过Pointcut(切点)指定哪些类的哪些方法上织入横切逻辑，通过Advice(增强)描述横切逻辑和方法的具体植入点(方法前、方法后、方法的两端等)。此外，Spring通过Advisor(切面)将Pointcut和Advice两者组装起来。切面Advisor就是切点和增强的复合体。理解了这三者之间的关系后看一下这三个有哪些分类。

<!--more-->
#增强类型
Spring提供4种类型的方法增强，分别是前置增强、后置增强、环绕增强和异常抛出增强，此外还有一种特殊的引介增强，引介增强是类级别的，它为目标类织入新的接口实现。
##异常抛出增强
前三个异常见名知义，异常抛出增强最合适的应用场景就是事务管理，当参与事务的某个Dao发出异常时，事务管理器就必须回滚事务。下面业务代码模拟异常抛出：

	import java.sql.SQLException;

	public class ForumService {
		public void removeForum(int forumId){
			//do sth..
			throw new RuntimeException("运行时异常");
		}
		public void update() throws SQLException{
			//do sth..
			throw new SQLException("数据更新异常");
		}
	}

然后通过下面这个异常抛出增强对业务方法进行增强处理，统一捕捉异常并回滚事务。

	import java.lang.reflect.Method;
	import org.springframework.aop.ThrowsAdvice;

	public class TransactionManager implements ThrowsAdvice {
		public void afterThrowing(Method method,Object[] args,Object targets,Exception ex){
			System.out.println("---------");
			System.out.println("method" + method.getName());
			System.out.println("抛出异常： " + ex.getMessage());
			System.out.println("成功回滚事务。");
		}
	}

ThrowsAdivce异常抛出增强接口没有定义任何方法，只是一个标识接口，运行期Spring使用反射机制自行判断，必须使用代码中的方法和参数形式，其中前三个可选，同时提供或者不提供，最后一个参数必须是Throwable及其子类。
配置代码：

	<bean id="transactionManager" class="cn.hdu.advisor.TransactionManager"/>
	<bean id="forumServiceTarget" class="cn.hdu.advisor.ForumService"/>
	<bean id="forumService" class="org.springframework.aop.framework.ProxyFactoryBean"
		p:interceptorNames="transactionManager"
		p:target-ref="forumServiceTarget"
		p:proxyTargetClass="true"
	/>

测试：

	import java.sql.SQLException;
	import org.springframework.context.ApplicationContext;
	import org.springframework.context.support.ClassPathXmlApplicationContext;

	public class TestAdvisor {
		public static void main(String []args){
			ApplicationContext ctx = new ClassPathXmlApplicationContext("bean.xml");
			ForumService service = (ForumService) ctx.getBean("forumService");
			try {
				service.update();
			} catch (SQLException e) {
				e.printStackTrace();
			}
			service.removeForum(0);
		}
	}

两个方法测试结果为

	---------
	methodupdate
	抛出异常： 数据更新异常
	成功回滚事务。
	---------
	methodremoveForum
	抛出异常： 运行时异常
	成功回滚事务。

#切点类型
主要有
`静态方法切点`:静态方法切点的抽象基类，默认匹配所有的类	 org.springfranmework.aop.support.StaticMethodMacherPointcut

`动态方法切点`:动态方法切点的抽象基类，默认匹配所有的类	 org.springfranmework.aop.support.DynamicMethodMacherPointcut

`注解切点`:支持通过JDK5.0注解标签定义的切点  org.springfranmework.aop.support.annotation.AnnotationMachingPointcut

`表达式切点`:支持AspectJ切点表达式	 org.springfranmework.aop.support.ExpressionPointcut

`流程切点`:根据程序执行堆栈的信息查看目标方法是否由某一个方法直接或间接发起调用，以此判断是否为匹配的连接点。	 org.springfranmework.aop.support.ControlFlowPointcut

`复合切点`:支持创建多个切点	  org.springfranmework.aop.support.ComposablePointcut

#切面类型
切面类型分为一般切面(Advisor)，仅包含一个Advice
带切点切面（Pointcut Advisor）,包含Advice和Pointcut两个类
引介切面(IntroductionAdvisor),对应引介增强的特殊切面，应用于类层面，使用ClassFilter进行定义

##切点切面

###静态普通方法名匹配切面

静态普通方法名匹配切面StaticMethodMatcherPointcutAdvisor，Advisor通过类过滤和方法名匹配定义切点。切面一般定义形式为：
	public class XxxAdvisor extends StaticMethodMatcherPointcutAdvisor{
	public boolean matches(Method method,Class clazz){
			return "xxxName".equals(method.getName());//切点方法匹配股则：方法名为xxxName
		}
	public ClassFilter getClassFilter(){		//切点类匹配规则：xxxClass的类或其子类
			return new ClassFilter(){
				public boolean matches(Class clazz){
					return xxxClass.class.isAssignableFrom(clazz);
				}
			}
		}
	}

###静态正则表达式方法匹配切面
静态正则表达式方法匹配切面，上面的静态方法名描述不够灵活便有了使用正则表达式的描述。无外乎在配置方法名时不再指定某个特定名，而是选择正则匹配一个或多个。

###动态切面
上面的静态切面对类和方法名进行过滤/限定，那么动态切面限定了什么呢？我的理解就是动态切面对指定类的指定方法的调用者或者类的实例化对象进行了限定。增强形式如下：

	public class XxxAdvisor extends DynamicMethodMatcherPointcutAdvisor｛
		public static List<String> specialClientList = new ArrayList<String>();   //指定对象集合
		//可以加入指定对象
		static {
			specialClientList.add("xxx");
			specialClientList.add("yyy");
		}
		public ClassFilter getClassFilter(){		
				return new ClassFilter(){
					public boolean matches(Class clazz){
						return xxxClass.class.isAssignableFrom(clazz);
					}
				}
			}
		public boolean matches(Method method,Class clazz){
				return "xxxName".equals(method.getName());
		}
		//上述为静态检查
		public boolean matches(Method method,Class clazz,Object[] args){     //对方法进行动态切点检查
			String clientName = (String) args[0];
			return specialClientList.contains(clientName);
		}
	｝

动态检查和静态检查一个很重要的差别就是，静态切点检查知会在第一次调用代理类时做静态检查，之后就不再检查；而动态检查在每次调用代理对象的任何一个方法都会执行动态切点检查，这将会导致很大的性能问题。所以一般在定义动态切点时会先覆盖getClassFliter()和matches(Method method,Class clazz)方法做静态切点检查排除大部分方法。


###流程切面
一句话“指定类的指定方法直接或间接发起的调用匹配的切点”，比如指定waiter类的Service()方法发起的、或是其调用的方法的切点与某Advice形成的切面。

###复合切面
多个切面∪与∩形成的切面