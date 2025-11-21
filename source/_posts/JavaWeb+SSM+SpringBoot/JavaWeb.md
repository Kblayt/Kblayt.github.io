---
title: JavaWeb
date: 2022-10-21 16:01:32
categories:
    - JavaWeb+SSM+SpringBoot
tags:
    - JavaWeb
---
# 1、JavaWeb基础

## 1.1、程序架构和Tomcat的操作

略

## 1.2、部署Web项目、JSP入门

略

# 2、JSP实现数据传递和保存

## 2.1、JSP九大内置对象

JSP 中定义了 9 个内置对象，它们分别是：request、response、session、application、out、pagecontext、config、page 和 exception，这些对象在客户端和服务器端交互的过程中分别完成不同的功能。

| 对  象 | 类型 | 说  明 |
| - | - | - |
| request | javax.servlet.http.HttpServletRequest | 获取用户请求信息 |
| response | javax.servlet.http.HttpServletResponse | 响应客户端请求，并将处理信息返回到客户端 |
| out | javax.servlet.jsp.JspWriter | 输出内容到 HTML 中 |
| session | javax.servlet.http.HttpSession | 用来保存用户信息 |
| application | javax.servlet.ServletContext | 所有用户共享信息 |
| config | javax.servlet.ServletConfig | 这是一个 Servlet 配置对象，用于 Servlet 和页面的初始化参数 |
| pageContext | javax.servlet.jsp.PageContext | JSP 的页面容器，用于访问 page、request、application 和 session 的属性 |
| page | javax.servlet.jsp.HttpJspPage | 类似于 Java 类的 this 关键字，表示当前 JSP 页面 |
| exception | java.lang.Throwable | 该对象用于处理 JSP 文件执行时发生的错误和异常；只有在 JSP 页面的 page 指令中指定 isErrorPage 的取值 true 时，才可以在本页面使用 exception 对象。 |


JSP 的内置对象主要有以下特点：

- 由 JSP 规范提供，不用编写者实例化；

- 通过 Web 容器实现和管理；

- 所有 JSP 页面均可使用；

- 只有在脚本元素的表达式或代码段中才能使用。

## 2.2、四大域对象

在 JSP 九大内置对象中，包含四个域对象，它们分别是：pageContext（page 域对象）、request（request 域对象）、session（session 域对象）、以及 application（application 域对象）。

JSP 中的 4 个域对象都能通过以下 3 个方法，对属性进行保存、获取和移除操作。

| 返回值类型 | 方法 | 作用 |
| - | - | - |
| void | setAttribute(String name, Object o) | 将属性保存到域对象中 |
| Object | getAttribute(String name) | 获取域对象中的属性值 |
| void | removeAttribute(String name) | 将属性从域对象中移除 |


JSP 中的 4 个域对象的作用域各不相同，如下表。

| 作用域 | 描述 | 作用范围 |
| - | - | - |
| page | 如果把属性保存到 pageContext 中，则它的作用域是 page。 | 该作用域中的属性只在当前 JSP 页面有效，跳转页面后失效。 |
| request | 如果把属性保存到 request 中，则它的作用域是 request。 | 该作用域中的属性只在当前请求范围内有效。request作用域是在一次请求当中<br>服务器跳转页面后有效，例如&lt;jsp:forward&gt;；<br>客户端跳转页面后无效，例如超链接。 |
| session | 如果把属性保存到 session 中，则它的作用域是 session。 | 该作用域中的属性只在当前会话范围内有效，网页关闭后失效。 |
| application | 如果把属性保存到 application 中，则它的作用域是 application。 | 该作用域中的属性在整个应用范围内有效，服务器重启后失效。 |


## 2.3、获取input输入框中的值

单值：

getParameter(inputname)

数组：

getParameterValues(inputname)

## 2.4、post方式出现乱码情况的解决方法

在最开头设置request.setcharacterEncoding(“utf-8”)和response.setcharacterEncoding(“utf-8”)

get方式出现乱码情况

方式一：修改server.xml配置文件（不建议使用）

方式二：利用代码方式解决

userName  =  new  string(userName.getParameter(“”)………);

## 2.5、post与get的区别

1. GET提交的数据放在URL中，POST则不会。这是最显而易见的差别。这点意味着GET更不安全（POST也不安全，因为HTTP是明文传输抓包就能获取数据内容，要想安全还得加密）

1. GET回退浏览器无害，POST会再次提交请求（GET方法回退后浏览器再缓存中拿结果，POST每次都会创建新资源）

1. GET提交的数据大小有限制（是因为浏览器对URL的长度有限制，GET本身没有限制），POST也有但是大小很大有几个G。

1. GET可以被保存为书签，POST不可以。这一点也能感受到。

1. GET能被缓存，POST不能

1. GET只允许ASCII字符，POST没有限制

1. GET会保存再浏览器历史记录中，POST不会。这点也能感受到。

1. URL可传播

一般情况下使用post方式

## 2.6、转发和重定向

转发a—>b—>转发—>c(一次请求)

request.setAttribute(“msg”,“注册失败”)

request.getRequestDispatcher(filename).forward(request,response)

request.getAttribute(“msg”,“注册失败”)

重定向a—>b重定向b—>c(两次请求)

response.sendRedirect()

# 3、使用JDBC(Java DataBase Connectivity)操作数据库

Connection	数据库连接

PreparedStatement		sql语句

ResultSet	执行查询时，返回的结果集

DriverManager	管理驱动包中的连接

## 3.1、JDBC访问数据库的步骤

1. Class.forName()加载驱动

1. DriverManager获取Connection丽娜姐

1. 创建Statement执行SQL语句

1. 返回ResultSet

1. 释放资源

## 3.2、JDCB公共部分的解耦

```
package com.company.JDBC;

import java.sql.*;

/**
 * @description:
 * @author: Kblayt
 * @time: 2022/5/13 19:20
 */
public class NewsDao {

    private static Connection connection = null;
    PreparedStatement preparedStatement = null;
    ResultSet resultSet = null;

    String url = "jdbc:mysql://localhost:3306/javaweb?characterEncoding=utf8";
    String user = "root";
    String password = "123456";

    /**
     *@description:
        获取连接
      * @return: java.sql.Connection
      * @author: Kblayt
      * @time: 2022/5/14 14:22
      */
    private Connection getConnection(){
        try {
            if (connection==null||connection.isClosed()){
                //1、建立连接
                Class.forName("com.mysql.jdbc.Driver");
                //2、获取链接
                connection = DriverManager.getConnection(url,user,password);
            }
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } catch (SQLException e) {
            e.printStackTrace();
        }
        return connection;
    }

    /**
     *@description:
        关闭连接
      * @return: void
      * @author: Kblayt
      * @time: 2022/5/14 14:24
      */
    private void closeConnection(){
        try {
            if (resultSet!=null){
                resultSet.close();
            }
            if (preparedStatement!=null){
                preparedStatement.close();
            }
            if (connection!=null){
                connection.close();
            }
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }

    /**
     *@description:
        添加
      * @return: int
      * @author: Kblayt
      * @time: 2022/5/14 14:36
      */
    public int addNews(int categoryId,String title){
        int addNewsDetail = 0;
        try {
            connection = this.getConnection();
            String sql = "insert into news_detail(categoryId, title) values (?,?)";
            preparedStatement = connection.prepareStatement(sql);
            preparedStatement.setInt(1,categoryId);
            preparedStatement.setString(2,title);
            System.out.println("addNews param categoryId:"+categoryId+",title:"+title);
            addNewsDetail = preparedStatement.executeUpdate();
            System.out.println("addNews param categoryId:"+categoryId+",title:"+title+",result:"+addNewsDetail);
        } catch (SQLException e) {
            e.printStackTrace();
            System.out.println("queryNewsByTitle is error,param categoryId："+categoryId+",title:"+title);
        }
        return addNewsDetail;
    }

    /**
     *@description:
        删除
     * @return: int
     * @author: Kblayt
     * @time: 2022/5/14 14:36
     */
    public int delNews(int id){
        int delNewsDetail = 0;
        try {
            connection = this.getConnection();
            String sql = "delete from news_detail where id=?";
            preparedStatement = connection.prepareStatement(sql);
            preparedStatement.setInt(1,id);
            System.out.println("addNews param id:"+id);
            delNewsDetail = preparedStatement.executeUpdate();
            System.out.println("addNews param id:"+id+",result:"+delNewsDetail);
        } catch (SQLException e) {
            e.printStackTrace();
            System.out.println("delNews is error,param id:"+id);
        }
        return delNewsDetail;
    }

    /**
     *@description:
        修改
      * @return: int
      * @author: Kblayt
      * @time: 2022/5/14 14:59
      */
    public int updateNews(int id,String title){
        int updateNewsDetail = 0;
        try {
            connection = this.getConnection();
            String sql = "update news_detail set title=? where id=?;";
            preparedStatement = connection.prepareStatement(sql);
            preparedStatement.setString(1,title);
            preparedStatement.setInt(2,id);
            System.out.println("addNews param id:"+id+",title:"+title);
            updateNewsDetail = preparedStatement.executeUpdate();
            System.out.println("addNews param id:"+id+",title:"+title+",result:"+updateNewsDetail);
        } catch (SQLException e) {
            e.printStackTrace();
            System.out.println("updateNewsDetail is error,param id:"+id+",title:"+title);
        }
        return updateNewsDetail;
    }
    
    /**
     *@description:
        查询
      * @return: void
      * @author: Kblayt
      * @time: 2022/5/14 14:25
      */
    public void queryNewsByTitle(String queryTitle) {
        try {
            connection = this.getConnection();
            //3、发送指令(sql)
            String sql = "select id,title from news_detail where title like ?";
            System.out.println("NewsDao sql:"+sql);
            //4、数据库执行指令
            preparedStatement = connection.prepareStatement(sql);
            preparedStatement.setString(1,"%"+queryTitle+"%");
            resultSet = preparedStatement.executeQuery();
            while (resultSet.next()){
                int id = resultSet.getInt("id");//使用columnLabel来获取列
                String title = resultSet.getString("title");//使用columnIndex来获取列
                System.out.println("id:"+id+",title:"+title);
            }
        } catch (SQLException e) {
            e.printStackTrace();
            System.out.println("queryNewsByTitle is error,param:"+queryTitle);
        }finally {
            closeConnection();
        }
    }

    /**
     *@description:
        入口
      * @return: void
      * @author: Kblayt
      * @time: 2022/5/14 14:52
      */
    public static void main(String[] args) {
        NewsDao newsDao = new NewsDao();

//        //添加
//        int addNewCount = newsDao.addNews(3,"爱上服务器二7345375");
//        System.out.println("main method para categoryId:3,title=爱上服务器二7345375,"+"result:"+addNewCount);
//        if (addNewCount<0){
//            System.out.println("添加失败");
//        }else {
//            System.out.println("添加成功");
//        }

//        //删除
//        int delNewCount = newsDao.delNews(2);
//        System.out.println("main method para id:2"+"result:"+delNewCount);
//        if (delNewCount<0){
//            System.out.println("添加失败");
//        }else {
//            System.out.println("添加成功");
//        }

//        //修改
//        int updateNewCount = newsDao.updateNews(4,"哈哈哈哈哈哈哈哈哈哈哈");
//        System.out.println("main method para id:4"+"result:"+updateNewCount);
//        if (updateNewCount<0){
//            System.out.println("添加失败");
//        }else {
//            System.out.println("添加成功");
//        }
        //查询
        newsDao.queryNewsByTitle("哈哈哈哈哈哈哈哈哈");



    }
}

```

## 3.3、Statement与PreparedStatement区别

Statement：Statement对象不使用SQL参数，不会解析并编译SQL语句，每次调用执行SQL语句时都要进行SQL语句的解析和编译操作，效率低。

PreparedStatement：创建PreparedStatement对象时使用SQL语句做参数，会解析并编译SQL语句（预编译）。也可以使用带占位符“?”的SQL语句做参数，在通过setXxx()方法给占位符赋值后执行SQL语句时无需再解析和编译SQL语句，直接执行的。当进行批处理（多次执行相同操作）时，效率高。

安全性相对来说PreparedStatement比Statement高一些。

## 3.4、JDBC工作原理

![](WEBRESOURCE75396f683d74401cb2c2ff4c39f793b9.png)

![](WEBRESOURCE57a9f12c9fc1a7848b54ca07a475cf9f.png)

# 4、DAO模式及单例模式

## 设计模式：https://www.runoob.com/design-pattern/singleton-pattern.html

## 4.1、DAO模式

DAO 模式

DAO (DataAccessobjects 数据存取对象)是指位于业务逻辑和持久化数据之间实现对持久化数据的访问。通俗来讲，就是将数据库操作都封装起来。

对外提供相应的接口在面向对象设计过程中，有一些"套路”用于解决特定问题称为模式。

DAO 模式提供了访问关系型数据库系统所需操作的接口，将数据访问和业务逻辑分离对上层提供面向对象的数据访问接口。

从以上 DAO 模式使用可以看出，DAO 模式的优势就在于它实现了两次隔离。

- 1、隔离了数据访问代码和业务逻辑代码。业务逻辑代码直接调用DAO方法即可，完全感觉不到数据库表的存在。分工明确，数据访问层代码变化不影响业务逻辑代码,这符合单一职能原则，降低了藕合性，提高了可复用性。

- 2、隔离了不同数据库实现。采用面向接口编程，如果底层数据库变化，如由 MySQL 变成 Oracle 只要增加 DAO 接口的新实现类即可，原有 MySQ 实现不用修改。这符合 "开-闭" 原则。该原则降低了代码的藕合性，提高了代码扩展性和系统的可移植性。

一个典型的DAO 模式主要由以下几部分组成。

- 1、DAO接口： 把对数据库的所有操作定义成抽象方法，可以提供多种实现。

- 2、DAO 实现类： 针对不同数据库给出DAO接口定义方法的具体实现。

- 3、实体类：用于存放与传输对象数据。

- 4、数据库连接和关闭工具类： 避免了数据库连接和关闭代码的重复使用，方便修改。

## 4.2、DAO模式封装JDBC再解耦

![](WEBRESOURCEe2e6fa1394521576f61580ca14646764.png)

## 4.3、单列模式

单例模式（Singleton Pattern）是 Java 中最简单的设计模式之一。这种类型的设计模式属于创建型模式，它提供了一种创建对象的最佳方式。

这种模式涉及到一个单一的类，该类负责创建自己的对象，同时确保只有单个对象被创建。这个类提供了一种访问其唯一的对象的方式，可以直接访问，不需要实例化该类的对象。

- 1、单例类只能有一个实例。

- 2、单例类必须自己创建自己的唯一实例。

- 3、单例类必须给所有其他对象提供这一实例。

## 4.4、懒汉模式

```
package com.company.JDBC.Utils;

import java.io.IOException;
import java.io.InputStream;
import java.io.InputStreamReader;
import java.util.Properties;

/**
 * @description:懒汉模式（多线程问题）
 * @author: Kblayt
 * @time: 2022/5/15 16:57
 */
public class ConfigUtil_Lazy {

    private  Properties properties;

    public static ConfigUtil_Lazy configUtilLazy;

    public static synchronized ConfigUtil_Lazy getInstance(){
        if (configUtilLazy ==null){
            configUtilLazy =new ConfigUtil_Lazy();
        }
        return configUtilLazy;
    }

    private ConfigUtil_Lazy(){
        InputStream inputStream = null;
        try {
            String filePath="database.properties";
            inputStream = ConfigUtil_Lazy.class.getClassLoader().getResourceAsStream(filePath);
            properties = new Properties();
            properties.load(inputStream);
        } catch (IOException e) {
            e.printStackTrace();
        }finally {
            try {
                if (inputStream!=null){
                    inputStream.close();
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    public String getValueByKey(String key){
        System.out.println("getValueByKey key:"+key);
        String value = (String) properties.get(key);
        System.out.println("getValueByKey key:"+key+",value:"+value);
        return value;
    }

}

```

## 4.5、饿汉模式

```
package com.company.JDBC.Utils;

import java.io.IOException;
import java.io.InputStream;
import java.util.Properties;

/**
 * @description:饿汉模式
 * @author: Kblayt
 * @time: 2022/5/15 18:59
 */
public class ConfigUtil_Hunger {

    private Properties properties;

    //方式1:直接创建实体类对象
    public static ConfigUtil_Hunger configUtil_hunger = new ConfigUtil_Hunger();

//    //方式2：在静态代码块中创建实体类对象
//    public static ConfigUtil_Hunger configUtil_hunger;
//    static {
//        configUtil_hunger = new ConfigUtil_Hunger();
//    }

    public static ConfigUtil_Hunger getInstance(){
        return configUtil_hunger;
    }

    private ConfigUtil_Hunger(){
        InputStream inputStream = null;
        try {
            String filePath="database.properties";
            inputStream = ConfigUtil_Hunger.class.getClassLoader().getResourceAsStream(filePath);
            properties = new Properties();
            properties.load(inputStream);
        } catch (IOException e) {
            e.printStackTrace();
        }finally {
            try {
                if (inputStream!=null){
                    inputStream.close();
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    public String getValueByKey(String key){
        System.out.println("getValueByKey key:"+key);
        String value = (String) properties.get(key);
        System.out.println("getValueByKey key:"+key+",value:"+value);
        return value;
    }

}

```

## 4.6、枚举式

```
package com.company.JDBC.Utils;

import java.io.IOException;
import java.io.InputStream;
import java.util.Properties;

/**
 * @description:枚举单列模式
 * @author: Kblayt
 * @time: 2022/5/15 19:08
 */
public enum ConfigUtil_Enum {

    INSTANCE;

    private Properties properties;

    private ConfigUtil_Enum(){
        InputStream inputStream = null;
        try {
            String filePath="database.properties";
            inputStream = ConfigUtil_Hunger.class.getClassLoader().getResourceAsStream(filePath);
            properties = new Properties();
            properties.load(inputStream);
        } catch (IOException e) {
            e.printStackTrace();
        }finally {
            try {
                if (inputStream!=null){
                    inputStream.close();
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    public String getValueByKey(String key){
        System.out.println("getValueByKey key:"+key);
        String value = (String) properties.get(key);
        System.out.println("getValueByKey key:"+key+",value:"+value);
        return value;
    }

}

```

# 5、数据源及分层开发

## 5.1、理解数据源、连接池的作用

之前的方式访问数据库存在一些缺陷：

- 访问前现需要先获取连接

- 每次操作结束后，需要释放资源

- 频繁的连接导致系统的安全性和稳定性差

使用数据源、连接池可以解决这些问题。

![](WEBRESOURCEe1a20d7470c4336cf0f7ebeb24e5a0aa.png)

![](WEBRESOURCE792286b57b3af693bd16a256c1b768f4.png)

## 5.2、掌握JNDI连接数据库（JNDI会出现严重的漏洞）

通过JNDI--获取-->DataSource--创建好连接对象-->Connection---->连接池（管理连接对象）

1、修改Tomcat中conf的context.xml配置文件内容

```
<Resource auth="Container" driverClassName="com.mysql.jdbc.Driver"
          maxActive="100" maxIdle="30" maxWait="10000"
          name="jdbc/news" password="123456" type="javax.sql.DataSource"
          url="jdbc:mysql://127.0.0.1:3306/javaweb" username="root"/>
```

2、使用JNDI获取数据源，使用数据源获取连接对象

```
/**
 *@description:
    通过数据源获取数据库连接对象
  * @return:
  * @author: Kblayt
  * @time: 2022/5/16 8:48
  */
public boolean getConnection2(){
    try {
        //初始化上下文对象
        Context context = new InitialContext();
        //获取与逻辑名称相关联的数据源对象
        DataSource dataSource = (DataSource) context.lookup("java:comp/env/jdbc/news");
        //通过数据源获取数据库连接
        connection = dataSource.getConnection();
    } catch (NamingException e) {
        e.printStackTrace();
    } catch (SQLException e) {
        e.printStackTrace();
    }
    return true;
}
```

## 5.3、使用JavaBean传递数据

JavaBean（pojo、entity、DTO、VO...）

- 就是一个Java类

- 封装业务逻辑

- 封装数据

列如：

```
package com.company.JavaBean.pojo;

import java.io.Serializable;
import java.util.Date;

/**
 * @description:新闻JavaBean：封装数据
 * @author: Kblayt
 * @time: 2022/5/16 10:52
 */
public class News implements Serializable {
    
    private static final long serialVersionUID = -2227279597126347529L;

    private int id;
    private int categoryId;
    private String title;
    private  String summary;
    private  String content;
    private  String picPath;
    private  String author;
    private Date createDate;
    private Date modifyDate;

    public int getId() {
        return id;
    }
    public void setId(int id) {
        this.id = id;
    }
    public int getCategoryId() {
        return categoryId;
    }
    public void setCategoryId(int categoryId) {
        this.categoryId = categoryId;
    }
    public String getTitle() {
        return title;
    }
    public void setTitle(String title) {
        this.title = title;
    }
    public String getSummary() {
        return summary;
    }
    public void setSummary(String summary) {
        this.summary = summary;
    }
    public String getContent() {
        return content;
    }
    public void setContent(String content) {
        this.content = content;
    }
    public String getPicPath() {
        return picPath;
    }
    public void setPicPath(String picPath) {
        this.picPath = picPath;
    }
    public String getAuthor() {
        return author;
    }
    public void setAuthor(String author) {
        this.author = author;
    }
    public Date getCreateDate() {
        return createDate;
    }
    public void setCreateDate(Date createDate) {
        this.createDate = createDate;
    }
    public Date getModifyDate() {
        return modifyDate;
    }
    public void setModifyDate(Date modifyDate) {
        this.modifyDate = modifyDate;
    }
    
    @Override
    public String toString() {
        return "News{" +
                "id=" + id +
                ", categoryId=" + categoryId +
                ", title='" + title + '\'' +
                ", summary='" + summary + '\'' +
                ", content='" + content + '\'' +
                ", picPath='" + picPath + '\'' +
                ", author='" + author + '\'' +
                ", createDate=" + createDate +
                ", modifyDate=" + modifyDate +
                '}';
    }
}
```

idea如何设置实体类自动生成SerialVersionUID：

IDEA的File->Settings->Editor->Inspections,然后在搜索框输入serialV，出现对应的

选择：

- Serializable class without 'serialVersionUID'

- 'serialVersionUID' field not declared 'private static final long'

然后点击需要添加serialVersionUID的实体类名，Alt+Enter，出现Add ‘serialVersionUID’ field，即可给类添加serialVersionUID标识。

![](WEBRESOURCEb4ff26f268f66c89eada8da65f986f0e.png)

## 5.4、对service层的剥离

Controller---->service接口---->serviceImpl---->dao接口---->daoImpl---->mapper---->database

注：现阶段使用JSP直接调用service接口、daoImpl直接调用database，但在实际应用中，这种方法并不适用。各层之间使用JavaBean传递数据，即封装好的实体类。

## 5.5、对MVC（Model、View、Controller）三层模式的初步解析

Model：模型代表一个存取数据的对象或 JAVA POJO。它也可以带有逻辑，在数据变化时更新控制器。

View： 视图代表模型包含的数据的可视化。

Controller： 控制器作用于模型和视图上。它控制数据流向模型对象，并在数据变化时更新视图。它使视图与模型分离开。

例如：

![](WEBRESOURCE73ea99d61d1f3d1b66cee9aa720aec3f.png)

## 5.6、JSTL（JavaServer Pages Standard Tag Library：JSP标准标签库）

暂时了解以下三个标签：

```
<jsp:forward page=""></jsp:forward>
<jsp:useBean id=""></jsp:useBean>
<jsp:include page=""></jsp:include>
```

# 6、第三方控件

## 6.1、CKeditor

略

## 6.2、使用comons-fileupload.jar实现文件上传下载

### 6.2.1、上传

```
package com.company.Javaweb.servlet;

import com.company.Javaweb.pojo.News;
import com.company.Javaweb.service.Impl.NewsServiceImpl;
import com.company.Javaweb.service.NewsService;
import org.apache.commons.fileupload.FileItem;
import org.apache.commons.fileupload.FileItemFactory;
import org.apache.commons.fileupload.FileUploadException;
import org.apache.commons.fileupload.disk.DiskFileItemFactory;
import org.apache.commons.fileupload.servlet.ServletFileUpload;
import org.apache.commons.io.FilenameUtils;

import javax.servlet.ServletException;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.File;
import java.io.IOException;
import java.util.Date;
import java.util.List;
import java.util.UUID;

/**
 * @description:用户增加新闻的处理Servlet，接收用户表单提交的新闻数据，调用Service的方法将新闻保存大哦数据库，跳转到新闻列表页面
 * @author: Kblayt
 * @time: 2022/5/23 18:49
 */
public class AddServlet extends HttpServlet {

    @Override
    protected void doGet(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        this.doPost(request, response);
    }

    @Override
    protected void doPost(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        request.setCharacterEncoding("utf-8");
        News news=new News();
        news.setCreateDate(new Date());
        NewsService newsService=new NewsServiceImpl();
        //判断类型
        boolean isM= ServletFileUpload.isMultipartContent(request);
        if(isM){
            //from 表单提示正确
            FileItemFactory fi=new DiskFileItemFactory();
            ServletFileUpload sfu=new ServletFileUpload(fi);
            List<FileItem> list= null;
            try {
                list = sfu.parseRequest(request);
            } catch (FileUploadException e) {
                e.printStackTrace();
            }
            for(FileItem item:list){
                //判断类型，选择存值
                if(item.isFormField()){
                    if("title".equals(item.getFieldName())){
                        news.setTitle(item.getString("utf-8"));
                    }else if("summary".equals(item.getFieldName())){
                        news.setSummary(item.getString("utf-8"));
                    }else if("newscontent".equals(item.getFieldName())){
                        news.setContent(item.getString("utf-8"));
                    }else if("picPath".equals(item.getFieldName())){
                        news.setPicPath(item.getString("utf-8"));
                    }else if("author".equals(item.getFieldName())){
                        news.setAuthor(item.getString("utf-8"));
                    }else if("categoryId".equals(item.getFieldName())){
                        String s=item.getString("utf-8");
                        news.setCategoryId(Integer.parseInt(s));
                    }
                }else{
                    //文件的上传
                    String filename=item.getName();
                    if(filename!=null &&!"".equals(filename)){
                        //分割后缀名
                        String endName="."+ FilenameUtils.getExtension(filename);
                        //拼接完整路径
                        String picPath="d:/"+ UUID.randomUUID().toString()+endName;
                        news.setPicPath(picPath);
                        //根据路径封装文件
                        File file=new File(picPath);
                        //上传
                        try {
                            item.write(file);
                        } catch (Exception e) {
                            e.printStackTrace();
                        }
                    }
                }
            }
        }
        boolean flag=newsService.addNews(news);
        if(flag){
            response.sendRedirect("/admin/newsDetailList.jsp");
        }
    }
}

```

### 6.2.2、下载

```
package com.company.Javaweb.servlet;

import javax.servlet.ServletException;
import javax.servlet.ServletOutputStream;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.File;
import java.io.FileInputStream;
import java.io.IOException;
import java.io.InputStream;

/**
 * @description:
 * @author: Kblayt
 * @time: 2022/5/24 10:48
 */
public class DownloadServlet extends HttpServlet {

    @Override
    protected void doGet(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        this.doPost(request, response);
    }

    @Override
    protected void doPost(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
//        JSP页面中实现需要关闭 out
//        out.clear();
//        out=pageContext.pushBody();
//        //根据路径 获取文件
        String picPath=request.getParameter("picPath");
        File file=new File(picPath);
        if(file.exists()){
            //设置相应类型
            response.setContentType("application/x-msdownload");
            //设置相应头
            response.setHeader("Content-Disposition", "attachment;filename="+picPath);
            //获取输入输出流
            InputStream is=new FileInputStream(file);
            ServletOutputStream sos=response.getOutputStream();
            //使用字节流输出
            byte []b=new byte[1024];
            int len=0;
            while((len=is.read(b))!=-1){
                sos.write(b, 0, len);
            }
            sos.close();
            is.close();
        }
    }
}

```

# 7、分页查询



# 8、EL(Expression Language)与JSTL(JavaServerPages Standard Tag Library)

## 8.1、EL表达式

为什么要使用EL表达式

- JSP脚本有哪些不足

- 代码结构混乱

- 脚本与HTML混合，容易出错

- 代码不宜与维护

- 使用EL表达式来优化程序代码，增加程序可读性

## 8.2、EL语法

EL表达式

- ${EL 表达式}例如：${username}

EL操作符

- 操作符“.”

- 获取对象的属性，例如：${news.title}

- 操作符“[]”

- 获取对象的属性，例如：${news["title"]}

- 获取集合中的对象，例如：newsList[0]

![](WEBRESOURCE49988252cc330d20f8b4642a6b94ec11.png)

![](WEBRESOURCE444b5dbab8a3186b87439ffb875e0bc6.png)

## 8.3、JSTL介绍

- JSTL

- JSP标准标签库

- 实现JSP页面中的逻辑控制

- JSTL使用步骤

- 添加jstl.jar和standard.jar包

- 在JSP页面中添加如下指令

<%@ taglib uri="http://java.sun.com/jsp/jstl/core" prefix="c"%>

<%@ taglib uri="http://java.sun.com/jsp/jstl/fmt" prefix="fmt"%>

![](WEBRESOURCE2489f52d280439c37df16ec02776723c.png)

# 9、Servlet

## 9.1、Servlet生命周期

![](WEBRESOURCE7f1a9492e9fe6eab45a38d07e88d749b.png)

## 9.2、ServletAPI

- javax.servlet.Servlet接口

- 所有Java Servlet的基础接口类，规定了必须由Servlet具体类实现的方法集。

- javax.setvlet.GenericServlet类

- 是Servlet的通用版本，是一种与协议无关的Servlet

- javax.servlet.http.HttpServlet类

- 在GenericServlet基础上扩展的基于Http协议的Servlet

## 9.3、Servlet中主要方法

- init()：Servlet的初始化方法，仅仅会执行一次。

- service()：处理请求和生成响应。（派发器，用户请求类型调用响应的doGet()、doPost()）

- doGet()：

- doPost()：

- destroy：在服务器停止并且程序中的Servlet对象不再使用的时候调用，只执行一次。

## 9.4、Servlet中重定向和转发相对路径和绝对路径问题

HttpServletResponse 重定向response.sendRedirect()

若当前路径为：http://localhost:8080/Javaweb/servlet/AddServlet.class

- 相对路径

- response.sendRedirect("newsDetailList.jsp");

- 实际访问路径为：http://localhost:8080/Javaweb/servlet/newsDetailList.jsp

- 绝对路径

- response.sendRedirect("/Javaweb/jsp/admin/newsDetailList.jsp");

- 实际访问路径为：http://localhost:8080/Javaweb/jsp/admin/newsDetailList.jsp

HttpServletRequest转发request.getRequestDispatcher().forword()

若当前路径为：http://localhost:8080/Javaweb/servlet/AddServlet.class

- 相对路径

- request.getRequestDispatcher("newsDetailList.jsp").forword(request,response)

- 实际访问路径为：http://localhost:8080/Javaweb/servlet/newsDetailList.jsp

- 绝对路径

- request.getRequestDispatcher("/jsp/admin/newsDetailList.jsp").forword(request,response)

- 实际访问路径为：http://localhost:8080/Javaweb/jsp/admin/newsDetailList.jsp

# 10、Filter(过滤器)和Listener(监听器)

## 10.1、过滤器

- 是向Web应用程序的请求和响应添加功能的Web服务组件

- 过滤器可以统一的集中处理请求和响应

- 使用过滤器技术实现对请求数据的过滤

![](WEBRESOURCE277454b3d7603570f51d24dcbf05ccbd.png)



![](WEBRESOURCEe7df8ee34688604773aef094dcd20911.png)



过滤器的生命周期

init()（初始化）

doFilter()（执行过滤器）

destroy()（销毁）

## 10.2、监听器

- Listener是Servlet的监听器

- 监听客户端的请求和服务器端的操作、

- 通过实现Listener接口的类可以在特定事件(Event)发生时，自动激发一些操作

### 10.2.1、HttpSessionBindingListener

- HttpSessionBindingListener

- 当一个实现了该接口的对象被捆绑到session中或从session中被解放的时候启用此监听器()session的值发生改变的时候)

- 关键点

- 创建实现HttpSessionBindingListener接口

- valueBound()

- valueUnbound()

- 不需要再web.xml中配置监听器

- 监听范围：一对一

### 10.2.2、HttpSessionListener

- HttpSessionListener

- 在Web应用中，当一个session被创建或者被销毁时启用这个监听器

- 关键点

- sessionCreate(HttpSessionEvent event)

- 客户第一次和服务器交互时触发

- sessionDestroyed(HttpSessionEvent event)

- 销毁会话的时候触发

- 必须在web.xml中配置监听器

- 监听范围：这只一次就可以监听所有session

## 10.2.2、HttpContextListener

- HttpContextListener

- 在web应用中，当一个容器产生或销毁时启用这个监听器

# 11、Ajax与Jquery的使用

参考：https://note.youdao.com/s/48YEmaJu

# 12、实现在Linux上的项目部署













