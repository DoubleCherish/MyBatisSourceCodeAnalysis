### Mybatis源码分析一（初始化过程）

​		最近学习MyBatis源码部分，现在准备一边学习一边记录下自己的理解过程。本篇文章主要记录下Mybatis初始过程，主要有以下几个点：

​		1、初始化做了什么

​		2、如何解析mybatis-config.xml各个参数的

​		3、涉及到的设计模式

***

##### 1、初始化

首先我们看一个简易的使用Mybatis的例子。

假设我们有一张用户表如下:

```mysql
create table user(
    id int primary key auto_increment,
   	name char(10) not null,
    password char(20) not null,
    age int not null,
);
```

创建一个实体类User如下：

```java
package com.alibaba.entity;

public class User{
    private Integer id;
    private String name;
    private String password;
    private Integer age;
    // 省略getter、setter
}
```

声明一个接口如下：

```java
package com.alibaba.dao;
import com.alibaba.entity.User;
public interface UserDao{
    int insert(User user);
}
```

再编写一个跟接口对应的mapper.xml文件

```xml
<?xml version="1.0" encoding="UTF-8" ?>   
<!DOCTYPE mapper   
PUBLIC "-//ibatis.apache.org//DTD Mapper 3.0//EN"  
"http://ibatis.apache.org/dtd/ibatis-3-mapper.dtd"> 
<mapper namespace="com.alibaba.dao.UserDao">
   <insert id="insert" ParameterType="com.alibaba.entity.User" > 
      insert into user(name,password,age) values(#{name},#{password},#{age})
   </insert>
</mapper>
```

最后编写下mybatis-config.xml

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
  PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
  "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>

  <!-- 指定properties配置文件， 我这里面配置的是数据库相关 -->
  <properties resource="dbConfig.properties"></properties>
  
  <!-- 指定Mybatis使用log4j -->
  <settings>
     <setting name="logImpl" value="LOG4J"/>
  </settings>
      
  <environments default="development">
    <environment id="development">
      <transactionManager type="JDBC"/>
      <dataSource type="POOLED">
          <!--
          如果上面没有指定数据库配置的properties文件，那么此处可以这样直接配置 
        <property name="driver" value="com.mysql.jdbc.Driver"/>
        <property name="url" value="jdbc:mysql://localhost:3306/test1"/>
        <property name="username" value="root"/>
        <property name="password" value="root"/>
         -->
         
         <!-- 上面指定了数据库配置文件， 配置文件里面也是对应的这四个属性 -->
         <property name="driver" value="${driver}"/>
         <property name="url" value="${url}"/>
         <property name="username" value="${username}"/>
         <property name="password" value="${password}"/>
      </dataSource>
    </environment>
  </environments>
  
  <!-- 映射文件，mybatis精髓， 后面才会细讲 -->
  <mappers>
    <mapper resource="com/alibaba/dao/user-mapper.xml"/>
  </mappers>
</configuration>
```

接下来写测试类如下：

```java
package com.alibaba.test;
// 省略import
public class TestMain(){
    public static void main(String [] args){
        String resource = "mybatis-config.xml";
        SqlSessionFactory sqlSessionFactory = null;
        
        try{
            sqlSessionFactory = new SqlSessionFactoryBuilder().build(Resources.getResourcesAsReader(resource));
        }catch(Exception e){
            e.printStackTrace();
        }
        SqlSession sqlSession = sqlSessionFactory.openSession();
        UserDao userDao = sqlSession.getMapper(UserDao.class);
        User user = new User();
        user.setName("double");
        user.setPassword("double");
        user.setAge(25);
        userDao.insert(user);
    }
}

```

​		上面就是一个简单使用mybatis的示例，上述代码经历了mybatis初始化-->创建sqlSession-->使用sqlSession获取映射器对象-->使用映射器对象执行语句。

下面介绍下mybatis-config.xml文件中大致包含哪些元素（可以看看上面mybatis-config.xml回顾下）：

​	**Configuration**

​		-- properties 属性值配置

​		--settings 设置

​		--typeAliases 类型别名配置

​		--typeHandlers 类型处理器

​		--objectFactory 对象工厂

​		--plugins 插件配置

​		--enviroments 数据源等信息配置

​		--mapper 映射器配置

![](https://github.com/DoubleCherish/MyBatisSourceCodeAnalysis/blob/master/configuration.png)

最后Mybatis会将配置中的所有信息组装成为一个Configuration对象供给整个框架使用，可以说初始化就是为了构造Configuration对象。

##### 2、mybatis-config.xml配置文件如何解析？

首先回忆一下上面示例中最主要的一条语句 :

​		  **`new SqlSessionFactoryBuilder().build(Resources.getResourcesAsReader(resource));`**

下面先用语言描述下mybatis初始化的步骤：

​		1、调用SqlSessionFactoryBuilder的bulid()方法，传入配置文件

​		2、SqlSessionFacoryBuilder会将传入的配置信息交由XMLConfigBuilder对象处理

​		3、调用XMLConfigBuilder的parse()方法解析配置文件中各个节点的数据

​		4、将解析的结果组装成一个Configuration对象

​		5、SqlSessionFactoryBuilder创建DefaultSqlSessionFactory并将Configuration传入其构造函数

​		6、SqlSessionFactoryBuilder返回DefaultSqlSessionFactory交由用户使用

图示如下：

![](https://github.com/DoubleCherish/MyBatisSourceCodeAnalysis/blob/master/step-construct-defaultsqlsessionfactory.png)

下面看看具体代码的分析：

```java
// 下面抠出示例代码中所用到的方法源码
public SqlSessionFactory build(Reader reader) {
    return build(reader, null, null);
}

public SqlSessionFactory build(Reader reader, String environment, Properties properties) {
    try {
      // 将配置数据源交给这个对象去处理，详细处理步骤忽略，暂时理解为将配置文件
      // 中的每个节点配置封装成Node对象
      XMLConfigBuilder parser = new XMLConfigBuilder(reader, environment, properties);
      // 上面在将配置文件分节点封装以后，下面调用parse()方法开始解析每个节点具体内容
      // 解析完的内容组装为一个Configuration对象传入build方法构建一个
      // DefaultSqlSessionFaction
      return build(parser.parse());
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error building SqlSession.", e);
    } finally {
      ErrorContext.instance().reset();
      try {
        reader.close();
      } catch (IOException e) {
        // Intentionally ignore. Prefer previous error.
      }
    }
}
// 最后调用这个方法返回DefaultSqlSessionFactory供用户使用
public SqlSessionFactory build(Configuration config) {
    return new DefaultSqlSessionFactory(config);
}
```

那么parse()方法具体做了什么呢？ 开始分析此方法

```java
public Configuration parse() {
    if (parsed) {
      throw new BuilderException("Each XMLConfigBuilder can only be used once.");
    }
    parsed = true;
    // 调用此方法，从配置文件的根节点configuration下开始解析
    parseConfiguration(parser.evalNode("/configuration"));
    return configuration;
}

// 此方法就是讲配置文件中的所有信息逐个解析的逻辑，下面会分析几个常用的解析点
private void parseConfiguration(XNode root) {
    try {
      //issue #117 read properties first
      propertiesElement(root.evalNode("properties"));
      Properties settings = settingsAsProperties(root.evalNode("settings"));
      loadCustomVfs(settings);
      loadCustomLogImpl(settings);
      typeAliasesElement(root.evalNode("typeAliases"));
      pluginElement(root.evalNode("plugins"));
      objectFactoryElement(root.evalNode("objectFactory"));
      objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
      reflectorFactoryElement(root.evalNode("reflectorFactory"));
      settingsElement(settings);
      // read it after objectFactory and objectWrapperFactory issue #631
      environmentsElement(root.evalNode("environments"));
      databaseIdProviderElement(root.evalNode("databaseIdProvider"));
      typeHandlerElement(root.evalNode("typeHandlers"));
      mapperElement(root.evalNode("mappers"));
    } catch (Exception e) {
      throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
    }
}
```

常用配置解析之`propertiesElement(root.evalNode("properties"))`

```java
// 1、<properties resource="dbConfig.properties"></properties>
// 2、<properties>
//		<property name="" value=""/>
//	</properties>
// 3、<properties url="xxxxx"></properties>
// 这个是解析放在配置里面的一些属性值的方法，配置有三种写法，如上
// 通过方法参数可看到将配置信息封装成一个个XNode对象
private void propertiesElement(XNode context) throws Exception {
  if (context != null) {
    // 首先按照方法2的格式读取默认配置
    Properties defaults = context.getChildrenAsProperties();
    // 获取<properties>中的resource的值
    String resource = context.getStringAttribute("resource");
    // 获取<properties>中url的值
    String url = context.getStringAttribute("url");
    // 不能给<properties>同时配置resource和url，要不然会抛出异常
    if (resource != null && url != null) {
      throw new BuilderException("The properties element cannot specify both a URL and a resource based property file reference.  Please specify one or the other.");
    }
    // 如果resource不为空，将resource指定文件中的配置读进来，这一步外部配置文件会覆盖mybatis
    // 配置文件里配置的属性
    if (resource != null) {
      defaults.putAll(Resources.getResourceAsProperties(resource));
    } else if (url != null) {
      defaults.putAll(Resources.getUrlAsProperties(url));
    }
    // 获取configuration对象已有的属性值
    Properties vars = configuration.getVariables();
    if (vars != null) {
      // 如果已有属性不为空，则将其和现在读到的属性值合并
      defaults.putAll(vars);
    }
    // 最后更新parser和configuration对象的属性值为最新值
    parser.setVariables(defaults);
    configuration.setVariables(defaults);
  }
}
```

剩下的配置解析方式类似，在这里不做过多记录。

##### 3、所用到的设计模式

3.1 **构造模式**

​		应用一：

​				`new SqlSessionFactoryBuilder.bulid(InputStream);`

可以看到第一个应用的地方是构造SqlSessionFactory的时候，Mybatis使用了构造者模式

​		应用二 :

在创建Environment对象时候使用到了构建者模式

```
private void environmentsElement(XNode context) throws Exception {
    if (context != null) {
      if (environment == null) {
        environment = context.getStringAttribute("default");
      }
      for (XNode child : context.getChildren()) {
        String id = child.getStringAttribute("id");
        if (isSpecifiedEnvironment(id)) {
          TransactionFactory txFactory = transactionManagerElement(child.evalNode("transactionManager"));
          DataSourceFactory dsFactory = dataSourceElement(child.evalNode("dataSource"));
          DataSource dataSource = dsFactory.getDataSource();
          // 在此使用构建者模式创建Environment对象
          Environment.Builder environmentBuilder = new Environment.Builder(id)
              .transactionFactory(txFactory)
              .dataSource(dataSource);
          configuration.setEnvironment(environmentBuilder.build());
        }
      }
 }
```

具体Environment内部代码是一个标准的构建者模式

```java
public final class Environment {
  private final String id;
  private final TransactionFactory transactionFactory;
  private final DataSource dataSource;

  public Environment(String id, TransactionFactory transactionFactory, DataSource dataSource) {
    if (id == null) {
      throw new IllegalArgumentException("Parameter 'id' must not be null");
    }
    if (transactionFactory == null) {
      throw new IllegalArgumentException("Parameter 'transactionFactory' must not be null");
    }
    this.id = id;
    if (dataSource == null) {
      throw new IllegalArgumentException("Parameter 'dataSource' must not be null");
    }
    this.transactionFactory = transactionFactory;
    this.dataSource = dataSource;
  }

  public static class Builder {
    private String id;
    private TransactionFactory transactionFactory;
    private DataSource dataSource;

    public Builder(String id) {
      this.id = id;
    }

    public Builder transactionFactory(TransactionFactory transactionFactory) {
      this.transactionFactory = transactionFactory;
      return this;
    }

    public Builder dataSource(DataSource dataSource) {
      this.dataSource = dataSource;
      return this;
    }

    public String id() {
      return this.id;
    }

    public Environment build() {
      return new Environment(this.id, this.transactionFactory, this.dataSource);
    }

  }

  public String getId() {
    return this.id;
  }

  public TransactionFactory getTransactionFactory() {
    return this.transactionFactory;
  }

  public DataSource getDataSource() {
    return this.dataSource;
  }

}
```

以上就是初始化过程的简单记录，随后会由浅入深记录更多Mybatis源码的核心技术。
