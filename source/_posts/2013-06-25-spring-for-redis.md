---
layout: post
title: "spring对redis的支持"
date: 2013-06-25 07:27
comments: true
categories: 
---

spring-data项目提供了多种数据操作的包，其中spring-data-redis就提供了与redis交互的实现，现在最新的版本是1.0.4。

spring-data-redis提供了对多种redis客户端的集成和抽象，比如jedis, jRedis, RJC, SRP等。并且提供了以下几个方面的支持：

1. 底层抽象：主要封装了配置和连接过程，以及异常的转换和翻译。
2. 高层抽象：以RedisTemplate的形式提供。
3. 可复用的工具类。

<!--more-->

##引用spring-data-redis

可以在pom.xml中加入如下依赖：

``` xml
<dependency>
	<groupId>org.springframework.data</groupId>
	<artifactId>spring-data-redis</artifactId>
	<version>1.0.4.RELEASE</version>
</dependency>
```
引入该依赖后会自动引入org.slf4j:jcl-over-slf4j，如果不想使用jcl可以使用excludes排除。

##连接redis

连接redis的逻辑主要在org.springframework.data.redis.connection包中，里面提供了一些针对不同client的connection实现和一些公共类。本文都以jedis为客户端举例。

连接过程主要涉及RedisConnection和RedisConnectionFactory两个类，RedisConnectionFactory接受连接配置并可以创建具体的连接RedisConnection；RedisConnection封装了底层客户端真实的链接，并且可以自动翻译底层连接抛出的异常，有点类似spring jdbc的异常机制。根据不同的配置每次从Factory获得的connection可以是一个新创建的或者是一个缓存的(pooled)。RedisConnection.getNativeConnection()可以得到底层真正的Connection。

不同客户端的connection并不一定提供了所有redis所支持的特性，因此，如果在connection上使用了一个并不支持的操作，那么RedisConnection会抛出UnsupportedOperationException。

##连接配置

``` xml
<?xml version="1.0" encoding="UTF-8" ?>
<beans xmlns="http://www.springframework.org/schema/beans" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
	xmlns:context="http://www.springframework.org/schema/context"
	xsi:schemaLocation="
	http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
	http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-3.0.xsd">
	
	<context:component-scan base-package="redis.spring" />
	
	<bean id="connectionFactory" class="org.springframework.data.redis.connection.jedis.JedisConnectionFactory">
		<property name="hostName" value="redisserver"/>
		<property name="port" value="6379" />
		<property name="usePool" value="true" />
	</bean>
</beans>
```

##RedisTemplate
RedisTemplate对Redis操作的高层次封装，主要提供了两种操作方式：

###1.Operation
Redis现在所支持的value类型主要有五种：String，Hash，Set，List，Sorted Set(ZSet)。因此RedisTemplate可以获取针对这五种类型的Operation，并且这五种类型的Operation又分为普通的KV Operation和与Key已经绑定的value Operation：

* **String:** ValueOperations/BoundValueOperations
* **Hash:** HashOperations/BoundHashOperations
* **List:** ListOperations/BoundListOperations
* **Set:** SetOperations/BoundSetOperations
* **ZSet:** ZSetOperations/BoundZSetOperations

KV Operation和Value Operation的不同，拿set name 'fish'举例：

* **KV Operation:** template.opsForValue().set("name", "fish");
* **Value Operation:** template.boundValueOps("name").set("fish");  // 获得的Operation可以复用，但是所有操作都是绑定到name这个key上了。

###2.Callback
可以使用Callback的方式直接拿到RedisConnection，并操作Redis

``` java
@Test
public void testCallback() {
    template.execute(new RedisCallback<Object>() {
        @Override
        public Object doInRedis(RedisConnection connection) throws DataAccessException {
	   // use connection to op redis	
           return null;
        }
    });
}
```

spring还提供了专门针对String类型的模版：StringRedisTemplate，要求Key和Value都是String类型。

##序列化

由于redis本质上是存的二进制字节，因此在用java与redis交互的时候需要将java对象（包括String）序列化和反序列化。spring-data-redis的序列化支持在org.springframework.data.redis.serializer包中，所有的序列化机制都需要实现RedisSerializer接口：

``` java
public interface RedisSerializer<T> {
	/**
	 * Serialize the given object to binary data.
	 * 
	 * @param t object to serialize
	 * @return the equivalent binary data
	 */
	byte[] serialize(T t) throws SerializationException;

	/**
	 * Deserialize an object from the given binary data.
	 * 
	 * @param bytes object binary representation
	 * @return the equivalent object instance
	 */
	T deserialize(byte[] bytes) throws SerializationException;
}
```

已经提供的序列化器:

* **StringRedisSerializer:** 专门用于String对象的序列化器，可以配置编码格式
* **JdkSerializationRedisSerializer:** 使用jdk的二进制序列化机制 （RedisTemplate的默认序列化器）
* **OxmSerializer:** Object/XML序列化
* **JacksonJsonRedisSerializer:** JSON序列化

我们可以在配置RedisTemplate的时候配置需要的序列化器。

##Redis事务

Redis的事务虽然不像RDBMS一样支持回滚，但是可以保证在事务中的操作以原子的方式执行（中间不会被其他操作打断）。Redis提供了multi, exec, discard来实现事务的原子性。

在spring-data-redis中将这种事务机制和spring的事务整合在了一起，一般使用下面的模版来操作：

``` java
//execute a transaction
redisTemplate.execute(new SessionCallback<Object>() {
    public Object execute(RedisOperations operations) throws DataAccessException {
        operations.multi();
        operations.opsForValue().set("key", "value");
        List<Object> results =  operations.exec();
        return null;
    }
});
```