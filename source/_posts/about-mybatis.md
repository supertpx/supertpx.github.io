---
title: mybatis工作原理及相关
date: 2019-03-25 11:54:46
tags:
---

### mybatis工作流程

先上一张图：

{% asset_img mybatis工作流程.png mybatis工作流程 %}

由图中可以看出，mybatis最重要的两个配置之处：
- mybatis-config.xml--mybatis全局配置相关

        <?xml version="1.0" encoding="UTF-8"?>
        <!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN" "http://mybatis.org/dtd/mybatis-3-config.dtd">

        <configuration>
            <settings>
                <setting name="cacheEnabled" value="true"/>
                <setting name="lazyLoadingEnabled" value="false"/>
            </settings>

            <typeAliases>
                <typeAlias type="com.supertpx.dao.xxx" alias="User"/>
            </typeAliases>

            <environments default="development">
                <environment id="development">
                    <transactionManager type="JDBC"/> <!--事务管理类型-->
                <dataSource type="POOLED"><!--连接池-->
                    <property name="username" value="xxx"/>
                    <property name="password" value="xxx"/>
                    <property name="driver" value="com.mysql.jdbc.Driver"/>
                    <property name="url" value="jdbc:mysql:192.168.1.150/xxx"/>
                </dataSource>
                </environment>
            </environments>

            <mappers>
                <mapper resource="xxxMapper.xml"/>
            </mappers>

        </configuration>

- xxxMapper.xml--mapper.xml映射文件。用于动态sql的生成和resultSet转化为对应的javaBean。

mybatis应用程序通过SqlSessionFactoryBuilder从mybatis-config.xml配置文件（也可以用Java文件配置的方式，需要添加@Configuration）中构建出SqlSessionFactory（SqlSessionFactory是线程安全的）；然后，SqlSessionFactory的实例直接开启一个SqlSession，再通过SqlSession实例获得Mapper对象并运行Mapper映射的SQL语句，完成对数据库的CRUD和事务提交，之后关闭SqlSession。

#### 详细流程如下：

1、加载mybatis全局配置文件（数据源、mapper映射文件等），解析配置文件，MyBatis基于XML配置文件生成Configuration，和一个个MappedStatement（包括了参数映射配置、动态SQL语句、结果映射配置），其对应着相应的CURD操作。
2、SqlSessionFactoryBuilder通过Configuration对象生成SqlSessionFactory，用来开启SqlSession。
3、SqlSession对象完成和数据库的交互：
a、用户程序调用mybatis接口层api（即Mapper接口中的方法）；
b、SqlSession通过调用api的Statement ID找到对应的MappedStatement对象；
c、通过Executor（负责动态SQL的生成和查询缓存的维护）将MappedStatement对象进行解析，sql参数转化、动态sql拼接，生成jdbc Statement对象；
d、JDBC执行sql；
e、借助MappedStatement中的结果映射关系，将返回结果转化成HashMap、JavaBean等存储结构并返回。

#### 参考文献

[《深入理解mybatis原理》 MyBatis的架构设计以及实例分析](https://blog.csdn.net/luanlouis/article/details/40422941)
