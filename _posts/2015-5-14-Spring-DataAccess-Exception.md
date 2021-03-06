---
layout: post
category: spring
title: spring数据访问之异常处理体系
tagline: by flight
tags: spring data access
---
　　Spring对持久化技术的支持可以说是很友好的，支持包括Hibernate、oBatis、JDO、JPA、TOPLink。此外通过Spring JDBC框架对JDBC API进行简化。面向DAO制定了一个通用的异常体系，屏蔽具体持久化技术的异常，使业务层和具体的持久化技术解耦。

<!--more-->

#DAO模式
　　不管是一个逻辑简单的小软件系统，还是一个关系复杂的大型软件系统，都很可能涉及到对数据的访问和存储，而这些对数据的访问和存储往往随着场景的不同而各异。为了统一和简化相关的数据访问操作，J2EE核心模式提出了DAO(Data Access Object，数据访问对象)模式。简单的说就是DAO的存在使得业务层和具体的数据访问解耦。

#JDBC异常处理
    public class JDBCUserDao {
     public User findUserByPK(Integer id)
    {
        Connection conn = null;
        try
        {
            conn = getDataSource().getConnection();
            //....
            User user = new User();
            //........
            return user;
        } 
        catch (Exception e)
        {
            //是抛出异常，还是在当前位置处理。。。
        }
        finally
        {
            releaseConnection(conn);
        }
        return null;
    }
    }
    
　　使用JDBC进行数据库访问，当其间出现问题的时候，JDBCAPI会抛出SQLException来表明问题的发生。而SQLException属于checked     exception，所以，我们的DAO实现类要捕获这种异常并处理。那如何处理DAO中捕获的SQLException呢，直接在DAO实现类处理掉？如果这样的话，客户端代码就无法得知在数据访问期间发生了什么变化？所以只好将SQLException抛给客户端，进而，DAO实现类的相应的签名

    public User findUserByPK(Integer id) throws SQLException
    
　　这种向上抛出异常的方法与DAO模式的思想背道而驰，我们的数据访问接口对客户端应该是通用的，不管数据访问的机制发生了如何的变化，客户端代码都不应该受到牵连，这个方法抛出的是SQLException。

当引入另外一种数据访问的模式的时候，比如，当加入LdapUserDao的时候，会抛出NamingException，如果要实现该接口，那么该方法签名又要发生改变，如下所示

       public User findUserByPK(Integer id) throws SQLException,NamingException;

　　这种解决方法完全是无法接受的。因为数据访问的机制有所不同，我们的数据访问接口的定义现在变成了空中楼阁，我们无法最终确定这个接口。比如，有的数据库提供商采用SQLException的ErrorCode作为具体的错误信息标准，有的数据库提供商则通过SQLException的SqlState来返回相信的错误信息。即使将SQLException封装后抛给客户端对象，当客户端要了解具体的错误信息的时候，依然要根据数据库提供商的不同采取不同的信息提取方式，这种客户端处理起来将是非常的糟糕，我们应该向客户端对象屏蔽这种差异性。

#可行性方法
　　不应该将特定的数据访问异常的错误信息提取工作留给客户端对象，而是应该由DAO实现类，或者某个工具类进行统一的处理。假如我们让具体的DAO实现类来做这个工作，那么，对于JdbcUserDao来说，代码如下： 
　　

    try
    {
        conn = getDataSource().getConnection();
    //....
    User user = new User();
    Statement stmt = conn.createStatement();
    stmt.execute("");
    //........
    return user;
    } 
    catch (SQLException e)
    {
        //是抛出异常，还是在当前位置处理。。。
        if(isMysqlVendor())
        {
            //按照mysql数据库的规则分析错误信息然后抛出
            throw new RuntimeException(e);
        }
        if(isOracleVendor())
        {
            //按照oracle数据库的规则分析错误信息并抛出
            throw new RuntimeException(e);
        }
        throw new RuntimeException(e);
    }
    
　　信息提出出来了，可是，只通过RuntimeException一个异常类型，还不足以区分不同的错误类型，我们需要将数据访问期间发生的错误进行分类，然后为具体的错误分类分配一个对应的异常类型。比如，数据库连接不上、ldap服务器连接失败，他们被认为是资源获取失败；而主键冲突或者是其它的资源冲突，他们被认为是数据访问一致性冲突。针对这些情况，可以为RuntimeException为基准，为获取资源失败这种情况分配一个RuntimeException子类型，称其为ResourceFailerException,而数据一致性冲突对应另外一个子类型DataIntegrityViolationException，其它的分类异常可以加以类推，所以我们需要的只是一套unchecked exception类型的面向数据访问领域的异常层次类型。
　　
#Spring异常体系
　　庆幸的是Spring已经提供了异常访问体系。Srping在org.springframework.dao包中提供了一套完备优雅的DAO异常体系，这些异常继承于DataAccessException,而DataAccessException又继承与Nested Runtime Exception,Nested Runtime Exception异常以嵌套的方式封装了源异常。
![](http://i.imgur.com/1um6FRH.png)

可以看到Spring DAO异常体系非常丰富，上图列出了DataAccessException异常类下的子类。下表对这些异常进行了简单的描述:
![](http://i.imgur.com/vqlnbFg.png)