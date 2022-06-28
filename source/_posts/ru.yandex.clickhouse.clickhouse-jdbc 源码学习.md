---
title: clickhouse-jdbc 源码学习
date: 2022/06/28 17:24:27
---

#  clickhouse-jdbc 源码学习

<!-- toc -->



## 包介绍

### 依赖版本

本次研究源码依赖的版本如下

```xml
<dependency>
    <groupId>com.clickhouse</groupId>
    <artifactId>clickhouse-jdbc</artifactId>
    <version>0.3.2</version>	<!-- 0.2.4/0.2.5/0.2.6/0.3.0/0.3.2 -->
</dependency>
```

官方github有言，不再推荐使用ru.yandex.clickhouse的groupId依赖，以及ru.yandex.clickhouse.ClickHouseDriver的驱动

> Maven groupId ru.yandex.clickhouse and legacy JDBC driver ru.yandex.clickhouse.ClickHouseDriver are deprecated.
>
> Please use new groupId com.clickhouse and driver com.clickhouse.jdbc.ClickHouseDriver instead. It's highly recommended to upgrade to 0.3.2+ and start to integrate the new JDBC driver for improved performance and stability.
>
> 
>
> New JDBC driver class is com.clickhouse.jdbc.ClickHouseDriver(will remove ru.yandex.clickhouse.ClickHouseDriver starting from 0.4.0)

具体的更新日志可以看如下链接https://github.com/ClickHouse/clickhouse-jdbc/issues/768



### 搭建环境版本如下

| 包                                     | 版本                          |
| -------------------------------------- | ----------------------------- |
| spring-boot-starter-parent             | 2.3.2.RELEASE                 |
| mybatis-plus-boot-starter              | 3.4.3.4                       |
| dynamic-datasource-spring-boot-starter | 3.2.0                         |
| mybatis                                | 3.5.7                         |
| mysql-connector-java                   | 8.0.25                        |
| mybatis-plus-generator                 | 3.5.1                         |
| clickhouse-jdbc                        | 0.2.4/0.2.5/0.2.6/0.3.0/0.3.2 |



## QA

### 1.LocalDate/LocalDateTime不兼容

**Q**

使用mybatis-plus-generator插件生成库表实例，一般来说，日期、时间、时间戳的类型可能使用java.util.Date或者java.time.LocalDate或者java.time.LocalDateTime等类型映射。

在使用java.time.LocalDate和java.time.LocalDateTime时，使用了查询sql在结果集映射时报错类型不支持：Not implemented for type=class java.time.LocalDateTime

```java
[2022-06-13 10:53:45.682] [] [http-nio-8991-exec-1] [ERROR] [] org.apache.juli.logging.DirectJDKLog.log(DirectJDKLog.java:175) - Servlet.service() for servlet [dispatcherServlet] in context with path [] threw exception [Request processing failed; nested exception is org.springframework.jdbc.UncategorizedSQLException: Error attempting to get column 'date_time_str' from result set.  Cause: java.sql.SQLException: Not implemented for type=class java.time.LocalDateTime
; uncategorized SQLException; SQL state [null]; error code [0]; Not implemented for type=class java.time.LocalDateTime; nested exception is java.sql.SQLException: Not implemented for type=class java.time.LocalDateTime] with root cause
java.sql.SQLException: Not implemented for type=class java.time.LocalDateTime
	at ru.yandex.clickhouse.response.ClickHouseResultSet.getObject(ClickHouseResultSet.java:661) ~[clickhouse-jdbc-0.2.6.jar:0.2.6]
	at ru.yandex.clickhouse.response.ClickHouseResultSet.getObject(ClickHouseResultSet.java:666) ~[clickhouse-jdbc-0.2.6.jar:0.2.6]
	at com.alibaba.druid.filter.FilterChainImpl.resultSet_getObject(FilterChainImpl.java:1431) ~[druid-1.2.5.jar:1.2.5]
	at com.alibaba.druid.filter.FilterAdapter.resultSet_getObject(FilterAdapter.java:1719) ~[druid-1.2.5.jar:1.2.5]
	at com.alibaba.druid.filter.FilterChainImpl.resultSet_getObject(FilterChainImpl.java:1427) ~[druid-1.2.5.jar:1.2.5]
	at com.alibaba.druid.filter.stat.StatFilter.resultSet_getObject(StatFilter.java:855) ~[druid-1.2.5.jar:1.2.5]
	at com.alibaba.druid.filter.FilterChainImpl.resultSet_getObject(FilterChainImpl.java:1427) ~[druid-1.2.5.jar:1.2.5]
	at com.alibaba.druid.proxy.jdbc.ResultSetProxyImpl.getObject(ResultSetProxyImpl.java:1561) ~[druid-1.2.5.jar:1.2.5]
	at com.alibaba.druid.pool.DruidPooledResultSet.getObject(DruidPooledResultSet.java:1777) ~[druid-1.2.5.jar:1.2.5]
	at org.apache.ibatis.type.LocalDateTimeTypeHandler.getNullableResult(LocalDateTimeTypeHandler.java:38) ~[mybatis-3.5.7.jar:3.5.7]
	at org.apache.ibatis.type.LocalDateTimeTypeHandler.getNullableResult(LocalDateTimeTypeHandler.java:28) ~[mybatis-3.5.7.jar:3.5.7]
	at org.apache.ibatis.type.BaseTypeHandler.getResult(BaseTypeHandler.java:85) ~[mybatis-3.5.7.jar:3.5.7]
	at org.apache.ibatis.executor.resultset.DefaultResultSetHandler.applyAutomaticMappings(DefaultResultSetHandler.java:561) ~[mybatis-3.5.7.jar:3.5.7]
	at org.apache.ibatis.executor.resultset.DefaultResultSetHandler.getRowValue(DefaultResultSetHandler.java:403) ~[mybatis-3.5.7.jar:3.5.7]
	at org.apache.ibatis.executor.resultset.DefaultResultSetHandler.handleRowValuesForSimpleResultMap(DefaultResultSetHandler.java:355) ~[mybatis-3.5.7.jar:3.5.7]
	at org.apache.ibatis.executor.resultset.DefaultResultSetHandler.handleRowValues(DefaultResultSetHandler.java:329) ~[mybatis-3.5.7.jar:3.5.7]
	at org.apache.ibatis.executor.resultset.DefaultResultSetHandler.handleResultSet(DefaultResultSetHandler.java:302) ~[mybatis-3.5.7.jar:3.5.7]
	at org.apache.ibatis.executor.resultset.DefaultResultSetHandler.handleResultSets(DefaultResultSetHandler.java:195) ~[mybatis-3.5.7.jar:3.5.7]
	at org.apache.ibatis.executor.statement.PreparedStatementHandler.query(PreparedStatementHandler.java:65) ~[mybatis-3.5.7.jar:3.5.7]
	at org.apache.ibatis.executor.statement.RoutingStatementHandler.query(RoutingStatementHandler.java:79) ~[mybatis-3.5.7.jar:3.5.7]
	at org.apache.ibatis.executor.SimpleExecutor.doQuery(SimpleExecutor.java:63) ~[mybatis-3.5.7.jar:3.5.7]
	at org.apache.ibatis.executor.BaseExecutor.queryFromDatabase(BaseExecutor.java:325) ~[mybatis-3.5.7.jar:3.5.7]
	at org.apache.ibatis.executor.BaseExecutor.query(BaseExecutor.java:156) ~[mybatis-3.5.7.jar:3.5.7]
	at org.apache.ibatis.executor.CachingExecutor.query(CachingExecutor.java:109) ~[mybatis-3.5.7.jar:3.5.7]
	at org.apache.ibatis.executor.CachingExecutor.query(CachingExecutor.java:89) ~[mybatis-3.5.7.jar:3.5.7]
	at org.apache.ibatis.session.defaults.DefaultSqlSession.selectList(DefaultSqlSession.java:151) ~[mybatis-3.5.7.jar:3.5.7]
	at org.apache.ibatis.session.defaults.DefaultSqlSession.selectList(DefaultSqlSession.java:145) ~[mybatis-3.5.7.jar:3.5.7]
	at org.apache.ibatis.session.defaults.DefaultSqlSession.selectList(DefaultSqlSession.java:140) ~[mybatis-3.5.7.jar:3.5.7]
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[?:1.8.0_291]
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62) ~[?:1.8.0_291]
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[?:1.8.0_291]
	at java.lang.reflect.Method.invoke(Method.java:498) ~[?:1.8.0_291]
	at org.mybatis.spring.SqlSessionTemplate$SqlSessionInterceptor.invoke(SqlSessionTemplate.java:427) ~[mybatis-spring-2.0.6.jar:2.0.6]
	at com.sun.proxy.$Proxy83.selectList(Unknown Source) ~[?:?]
	at org.mybatis.spring.SqlSessionTemplate.selectList(SqlSessionTemplate.java:224) ~[mybatis-spring-2.0.6.jar:2.0.6]
	at com.baomidou.mybatisplus.core.override.MybatisMapperMethod.executeForMany(MybatisMapperMethod.java:166) ~[mybatis-plus-core-3.4.3.4.jar:3.4.3.4]
	at com.baomidou.mybatisplus.core.override.MybatisMapperMethod.execute(MybatisMapperMethod.java:77) ~[mybatis-plus-core-3.4.3.4.jar:3.4.3.4]
	at com.baomidou.mybatisplus.core.override.MybatisMapperProxy$PlainMethodInvoker.invoke(MybatisMapperProxy.java:148) ~[mybatis-plus-core-3.4.3.4.jar:3.4.3.4]
	at com.baomidou.mybatisplus.core.override.MybatisMapperProxy.invoke(MybatisMapperProxy.java:89) ~[mybatis-plus-core-3.4.3.4.jar:3.4.3.4]
	at com.sun.proxy.$Proxy87.selectList(Unknown Source) ~[?:?]
	at com.baomidou.mybatisplus.core.mapper.BaseMapper.selectOne(BaseMapper.java:174) ~[mybatis-plus-core-3.4.3.4.jar:3.4.3.4]
	at java.lang.invoke.MethodHandle.invokeWithArguments(MethodHandle.java:627) ~[?:1.8.0_291]
	at com.baomidou.mybatisplus.core.override.MybatisMapperProxy$DefaultMethodInvoker.invoke(MybatisMapperProxy.java:162) ~[mybatis-plus-core-3.4.3.4.jar:3.4.3.4]
	at com.baomidou.mybatisplus.core.override.MybatisMapperProxy.invoke(MybatisMapperProxy.java:89) ~[mybatis-plus-core-3.4.3.4.jar:3.4.3.4]
	at com.sun.proxy.$Proxy87.selectOne(Unknown Source) ~[?:?]
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[?:1.8.0_291]
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62) ~[?:1.8.0_291]
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[?:1.8.0_291]
	at java.lang.reflect.Method.invoke(Method.java:498) ~[?:1.8.0_291]
	at org.springframework.aop.support.AopUtils.invokeJoinpointUsingReflection(AopUtils.java:344) ~[spring-aop-5.2.8.RELEASE.jar:5.2.8.RELEASE]
	at org.springframework.aop.framework.ReflectiveMethodInvocation.invokeJoinpoint(ReflectiveMethodInvocation.java:198) ~[spring-aop-5.2.8.RELEASE.jar:5.2.8.RELEASE]
	at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:163) ~[spring-aop-5.2.8.RELEASE.jar:5.2.8.RELEASE]
	at com.baomidou.dynamic.datasource.aop.DynamicDataSourceAnnotationInterceptor.invoke(DynamicDataSourceAnnotationInterceptor.java:46) ~[dynamic-datasource-spring-boot-starter-3.2.0.jar:3.2.0]
	at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:186) ~[spring-aop-5.2.8.RELEASE.jar:5.2.8.RELEASE]
	at org.springframework.aop.framework.JdkDynamicAopProxy.invoke(JdkDynamicAopProxy.java:212) ~[spring-aop-5.2.8.RELEASE.jar:5.2.8.RELEASE]
	at com.sun.proxy.$Proxy88.selectOne(Unknown Source) ~[?:?]
	at com.zeekr.bigdata.data.portal.controller.TestController.querySlave2(TestController.java:63) ~[classes/:?]
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[?:1.8.0_291]
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62) ~[?:1.8.0_291]
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[?:1.8.0_291]
	at java.lang.reflect.Method.invoke(Method.java:498) ~[?:1.8.0_291]
	at org.springframework.web.method.support.InvocableHandlerMethod.doInvoke(InvocableHandlerMethod.java:190) ~[spring-web-5.2.8.RELEASE.jar:5.2.8.RELEASE]
	at org.springframework.web.method.support.InvocableHandlerMethod.invokeForRequest(InvocableHandlerMethod.java:138) ~[spring-web-5.2.8.RELEASE.jar:5.2.8.RELEASE]
	at org.springframework.web.servlet.mvc.method.annotation.ServletInvocableHandlerMethod.invokeAndHandle(ServletInvocableHandlerMethod.java:105) ~[spring-webmvc-5.2.8.RELEASE.jar:5.2.8.RELEASE]
	at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.invokeHandlerMethod(RequestMappingHandlerAdapter.java:878) ~[spring-webmvc-5.2.8.RELEASE.jar:5.2.8.RELEASE]
	at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.handleInternal(RequestMappingHandlerAdapter.java:792) ~[spring-webmvc-5.2.8.RELEASE.jar:5.2.8.RELEASE]
	at org.springframework.web.servlet.mvc.method.AbstractHandlerMethodAdapter.handle(AbstractHandlerMethodAdapter.java:87) ~[spring-webmvc-5.2.8.RELEASE.jar:5.2.8.RELEASE]
	at org.springframework.web.servlet.DispatcherServlet.doDispatch(DispatcherServlet.java:1040) ~[spring-webmvc-5.2.8.RELEASE.jar:5.2.8.RELEASE]
	at org.springframework.web.servlet.DispatcherServlet.doService(DispatcherServlet.java:943) ~[spring-webmvc-5.2.8.RELEASE.jar:5.2.8.RELEASE]
	at org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:1006) ~[spring-webmvc-5.2.8.RELEASE.jar:5.2.8.RELEASE]
	at org.springframework.web.servlet.FrameworkServlet.doGet(FrameworkServlet.java:898) ~[spring-webmvc-5.2.8.RELEASE.jar:5.2.8.RELEASE]
	at javax.servlet.http.HttpServlet.service(HttpServlet.java:626) ~[tomcat-embed-core-9.0.37.jar:4.0.FR]
	at org.springframework.web.servlet.FrameworkServlet.service(FrameworkServlet.java:883) ~[spring-webmvc-5.2.8.RELEASE.jar:5.2.8.RELEASE]
	at javax.servlet.http.HttpServlet.service(HttpServlet.java:733) ~[tomcat-embed-core-9.0.37.jar:4.0.FR]
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:231) ~[tomcat-embed-core-9.0.37.jar:9.0.37]
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166) ~[tomcat-embed-core-9.0.37.jar:9.0.37]
	at org.apache.tomcat.websocket.server.WsFilter.doFilter(WsFilter.java:53) ~[tomcat-embed-websocket-9.0.37.jar:9.0.37]
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193) ~[tomcat-embed-core-9.0.37.jar:9.0.37]
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166) ~[tomcat-embed-core-9.0.37.jar:9.0.37]
	at org.springframework.web.filter.RequestContextFilter.doFilterInternal(RequestContextFilter.java:100) ~[spring-web-5.2.8.RELEASE.jar:5.2.8.RELEASE]
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:119) ~[spring-web-5.2.8.RELEASE.jar:5.2.8.RELEASE]
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193) ~[tomcat-embed-core-9.0.37.jar:9.0.37]
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166) ~[tomcat-embed-core-9.0.37.jar:9.0.37]
	at org.springframework.web.filter.FormContentFilter.doFilterInternal(FormContentFilter.java:93) ~[spring-web-5.2.8.RELEASE.jar:5.2.8.RELEASE]
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:119) ~[spring-web-5.2.8.RELEASE.jar:5.2.8.RELEASE]
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193) ~[tomcat-embed-core-9.0.37.jar:9.0.37]
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166) ~[tomcat-embed-core-9.0.37.jar:9.0.37]
	at org.springframework.web.filter.CharacterEncodingFilter.doFilterInternal(CharacterEncodingFilter.java:201) ~[spring-web-5.2.8.RELEASE.jar:5.2.8.RELEASE]
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:119) ~[spring-web-5.2.8.RELEASE.jar:5.2.8.RELEASE]
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193) ~[tomcat-embed-core-9.0.37.jar:9.0.37]
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166) ~[tomcat-embed-core-9.0.37.jar:9.0.37]
	at org.apache.catalina.core.StandardWrapperValve.invoke(StandardWrapperValve.java:202) [tomcat-embed-core-9.0.37.jar:9.0.37]
	at org.apache.catalina.core.StandardContextValve.invoke(StandardContextValve.java:96) [tomcat-embed-core-9.0.37.jar:9.0.37]
	at org.apache.catalina.authenticator.AuthenticatorBase.invoke(AuthenticatorBase.java:541) [tomcat-embed-core-9.0.37.jar:9.0.37]
	at org.apache.catalina.core.StandardHostValve.invoke(StandardHostValve.java:139) [tomcat-embed-core-9.0.37.jar:9.0.37]
	at org.apache.catalina.valves.ErrorReportValve.invoke(ErrorReportValve.java:92) [tomcat-embed-core-9.0.37.jar:9.0.37]
	at org.apache.catalina.core.StandardEngineValve.invoke(StandardEngineValve.java:74) [tomcat-embed-core-9.0.37.jar:9.0.37]
	at org.apache.catalina.connector.CoyoteAdapter.service(CoyoteAdapter.java:343) [tomcat-embed-core-9.0.37.jar:9.0.37]
	at org.apache.coyote.http11.Http11Processor.service(Http11Processor.java:373) [tomcat-embed-core-9.0.37.jar:9.0.37]
	at org.apache.coyote.AbstractProcessorLight.process(AbstractProcessorLight.java:65) [tomcat-embed-core-9.0.37.jar:9.0.37]
	at org.apache.coyote.AbstractProtocol$ConnectionHandler.process(AbstractProtocol.java:868) [tomcat-embed-core-9.0.37.jar:9.0.37]
	at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.doRun(NioEndpoint.java:1589) [tomcat-embed-core-9.0.37.jar:9.0.37]
	at org.apache.tomcat.util.net.SocketProcessorBase.run(SocketProcessorBase.java:49) [tomcat-embed-core-9.0.37.jar:9.0.37]
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149) [?:1.8.0_291]
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624) [?:1.8.0_291]
	at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61) [tomcat-embed-core-9.0.37.jar:9.0.37]
	at java.lang.Thread.run(Thread.java:748) [?:1.8.0_291]
```

**A**

跟踪源码，在mybatis执行类型映射的时候，会调用到这样一段自动映射的方法；

```java
private boolean applyAutomaticMappings(ResultSetWrapper rsw, ResultMap resultMap, MetaObject metaObject, String columnPrefix) throws SQLException {
    List<UnMappedColumnAutoMapping> autoMapping = createAutomaticMappings(rsw, resultMap, metaObject, columnPrefix);
    boolean foundValues = false;
    if (!autoMapping.isEmpty()) {
      for (UnMappedColumnAutoMapping mapping : autoMapping) {
          // 在这里会根据结果集和列名称，找到结果集中的列，并获取该类型的handler，在这里的场景时LocalDateTime，因此获取的是LocalDateTimeTypeHandler，如果是java.util.Date类型，则获取的是DateTypeHandler
        final Object value = mapping.typeHandler.getResult(rsw.getResultSet(), mapping.column);
        if (value != null) {
          foundValues = true;
        }
        if (value != null || (configuration.isCallSettersOnNulls() && !mapping.primitive)) {
          // gcode issue #377, call setter on nulls (value is not 'found')
          metaObject.setValue(mapping.property, value);
        }
      }
    }
    return foundValues;
  }
```

LocalDateTimeTypeHandler会根据当前数据库连接的类型，从指定类型的resultset中调用getObject方法，在这里调用的是ClickHouseResultSet的public <T> T getObject(int columnIndex, Class<T> type) throws SQLException方法；

```java
public <T> T getObject(int columnIndex, Class<T> type) throws SQLException {
    if(type.equals(UUID.class)) {	//这里传入的type类型是java.time.LocalDateTime
    	return (T) UUID.fromString(getString(columnIndex));
    } else {
    	throw new SQLException("Not implemented for type=" + type.toString());
    }
}
```

可以看出，如果类型不是UUID.class，就会走进else分支而报错。这里可能是作者的一个bug或者说是作者还没有在这里做到多类型的getObject兼容；

于是想到升级版本或许问题可以得到解决。

到maven官网找到clickhouse-jdbc的包，查询了changelog，从0.3.0的日志中得到如下情报：

* support more data types: IPv4, IPv6, Int128, UInt128, Int256, UInt256, Decimal256, DateTime*, and Map

这里没有提到LocalDateTime，但是决定还是一试；

```
0.3.0
  * BREAKING CHANGE - dropped JDK 7 support
  * BREAKING CHANGE - removed Guava dependency(and so is UnsignedLong)
  * JDBC 4.2 support
  * add connection setting client_name for load-balancing and troubleshooting
  * add writeBytes & writeUUIDArray and remove UnsignedLong related methods in ClickHouseRowBinaryStream
  * support more data types: IPv4, IPv6, Int128, UInt128, Int256, UInt256, Decimal256, DateTime*, and Map
  * support ORC/Parquet streaming
  * support read/write Bitmap from/into AggregateFunction(groupBitmap, UInt[8-64]) column
  * throw SQLException instead of RuntimeException when instantiating ClickHouseConnectionImpl
  * fix error when using ClickHouseCompression.none against 19.16
  * fix NegativeArraySizeException when dealing with large array
  * fix datetime/date display issue caused by timezone differences(between client and column/server)
```

在升级0.3.0后再看源码，发现有变化

```java
	public <T> T getObject(int columnIndex, Class<T> type) throws SQLException {
        if (String.class.equals(type)) {
            return (T) getString(columnIndex);
        }
		// 这里if匹配不到的情况不再直接抛出异常，二十走下面的分支，在ClickHouseValueParser.getParser(type).parse(getValue(columnIndex), columnInfo, tz)中进行解析
        ClickHouseColumnInfo columnInfo = getColumnInfo(columnIndex);
        TimeZone tz = getEffectiveTimeZone(columnInfo);
        return columnInfo.isArray()
            ? (Array.class.isAssignableFrom(type) ? (T) getArray(columnIndex) : (T) getArray(columnIndex).getArray())
            : ClickHouseValueParser.getParser(type).parse(getValue(columnIndex), columnInfo, tz);
    }
```

进入ClickHouseValueParser.getParser源码，终于看到了类型支持的代码。

```java
	static Map<Class<?>, ClickHouseValueParser<?>> parsers;

    static {
        parsers = new HashMap<>();
        register(Array.class, ClickHouseArrayParser.getInstance());
        register(BigDecimal.class, BigDecimal::new);
        register(BigInteger.class, BigInteger::new);
        register(Boolean.class,
            s -> Boolean.valueOf("1".equals(s) || Boolean.parseBoolean(s)),
            Boolean.FALSE);
        register(Date.class, ClickHouseSQLDateParser.getInstance());
        register(Double.class, ClickHouseDoubleParser.getInstance());
        register(Float.class,
            Float::valueOf,
            Float.valueOf(0f),
            Float.valueOf(Float.NaN));
        register(Instant.class, ClickHouseInstantParser.getInstance());
        register(Integer.class, Integer::decode, Integer.valueOf(0));
        register(LocalDate.class, ClickHouseLocalDateParser.getInstance());
        register(LocalDateTime.class, ClickHouseLocalDateTimeParser.getInstance());
        register(LocalTime.class, ClickHouseLocalTimeParser.getInstance());
        register(Long.class, Long::decode, Long.valueOf(0L));
        register(ClickHouseBitmap.class, ClickHouseBitmapParser.getInstance());
        register(Map.class, ClickHouseMapParser.getInstance());
        register(Object.class, s -> s);
        register(OffsetDateTime.class, ClickHouseOffsetDateTimeParser.getInstance());
        register(OffsetTime.class, ClickHouseOffsetTimeParser.getInstance());
        register(Short.class, Short::decode, Short.valueOf((short) 0));
        register(String.class, ClickHouseStringParser.getInstance());
        register(Time.class, ClickHouseSQLTimeParser.getInstance());
        register(Timestamp.class, ClickHouseSQLTimestampParser.getInstance());
        register(UUID.class, UUID::fromString);
        register(ZonedDateTime.class, ClickHouseZonedDateTimeParser.getInstance());
    }

	public static <T> ClickHouseValueParser<T> getParser(Class<T> clazz)
        throws SQLException
    {
        // 这里parsers是一个静态的map对象，在类初始化时就做好的类型注册
        ClickHouseValueParser<T> p = (ClickHouseValueParser<T>) parsers.get(clazz);
        if (p == null) {
            throw new ClickHouseUnknownException(
                "No value parser for class '" + clazz.getName() + "'", null);
        }
        return p;
    }
```

至此，基本可以下结论，**>=0.3.0**的版本，可以支持LocalDateTime类型，**<0.3.0**不能支持；

为保险起见还是将0.2.4到0.3.2的版本的源码都看了一遍，**的确如上面的结论那样**。故本选用最新0.3.2版本；

### 2.一次查询请求的源码追踪

使用springboot+mybatisplus+druid+clickhouse-jdbc的组合，注定能擦出一点火花；

```java
List<Map<String, Object>> results = queryEngineMapper.queryM241TableFieldsConditionsPage(sysDictService.convertTableName(tableName), fields, conditions, dateTimeBeg, dateTimeEnd, (pageNum - 1) * pageSize, pageSize);
```

上面是一段通过mybatis查询clickhouse的调用，mapper写的比较简单，就是普通的动态sql写法；

就用debug的方式进入源码看看

在进入ck源码的最后一个调用栈是在com.alibaba.druid.filter.FilterChainImpl.preparedStatement_execute(PreparedStatementProxy statement)  line 3461

```java
@Override
public boolean preparedStatement_execute(PreparedStatementProxy statement) throws SQLException {
    if (this.pos < filterSize) {
        return nextFilter().preparedStatement_execute(this, statement);
    }
    // 在这里进入
    return statement.getRawObject().execute();
}
```

进入后就调起clickhouse-jdbc的源码部分了，首先进入的是ru.yandex.clickhouse.ClickHousePreparedStatementImpl line 139

```java
@Override
public boolean execute() throws SQLException {
    return executeQueryStatement(buildSql(), null, null, null) != null;
}
```

再来到ClickHouseStatementImpl

```java
protected ResultSet executeQueryStatement(ClickHouseSqlStatement stmt,
                                          Map<ClickHouseQueryParam, String> additionalDBParams, List<ClickHouseExternalData> externalData,
                                          Map<String, String> additionalRequestParams) throws SQLException {
    // 设置db参数
    additionalDBParams = importAdditionalDBParameters(additionalDBParams);
    
    // 设置format参数
    stmt = applyFormat(stmt, ClickHouseFormat.TabSeparatedWithNamesAndTypes);

    // 建立连接，获取输入流
    InputStream is = getInputStream(stmt, additionalDBParams, externalData, additionalRequestParams);
    try {
        //对查询结果封装结果集ResultSet
        return updateResult(stmt, is);
    } catch (Exception e) {
        try {
            is.close();
        } catch (IOException ioe) {
            log.error("can not close stream: %s", ioe.getMessage());
        }
        throw ClickHouseExceptionSpecifier.specify(e, properties.getHost(), properties.getPort());
    }
}
```

继续跟踪getInputStream方法

```java
private InputStream getInputStream(ClickHouseSqlStatement parsedStmt,
            Map<ClickHouseQueryParam, String> additionalClickHouseDBParams, List<ClickHouseExternalData> externalData,
            Map<String, String> additionalRequestParams) throws ClickHouseException {
        String sql = parsedStmt.getSQL();
        boolean ignoreDatabase = parsedStmt.isRecognized() && !parsedStmt.isDML()
                && parsedStmt.containsKeyword("DATABASE");
        if (parsedStmt.getStatementType() == StatementType.USE) {
            // 这里大费周章，就是识别user database这种语句
            currentDatabase = parsedStmt.getDatabaseOrDefault(currentDatabase);
        }

        log.debug("Executing SQL: %s", sql);

    	//这里会生成一个queryId并赋值到params里面，additionalClickHouseDBParams外面方法传进来的是个bull值，因此这里会走三目表达式的true分支，由于构造器里的实现是默认值queryId为null，因此这里会生成queryId。clickhouse-jdbc使用的是uuid生成queryId，可见下一份代码块
        additionalClickHouseDBParams = addQueryIdTo(additionalClickHouseDBParams == null
                ? new EnumMap<ClickHouseQueryParam, String>(ClickHouseQueryParam.class)
                : additionalClickHouseDBParams);
		
    	// 参数准备完毕，建立uri连接，后面将使用httpclient进行clickhouse的访问，使用post请求
        URI uri = buildRequestUri(null, externalData, additionalClickHouseDBParams, additionalRequestParams,
                ignoreDatabase);
        log.debug("Request url: %s", uri);

        HttpEntity requestEntity;
        if (externalData == null || externalData.isEmpty()) {
            // 没有额外的参数的话默认走这个分支，创建请求实体；
            requestEntity = new StringEntity(sql, StandardCharsets.UTF_8);
        } else {
            MultipartEntityBuilder entityBuilder = MultipartEntityBuilder.create();

            ContentType queryContentType = ContentType.create(ContentType.TEXT_PLAIN.getMimeType(),
                    StandardCharsets.UTF_8);
            entityBuilder.addTextBody("query", sql, queryContentType);

            try {
                for (ClickHouseExternalData externalDataItem : externalData) {
                    // clickhouse may return 400 (bad request) when chunked encoding is used with
                    // multipart request
                    // so read content to byte array to avoid chunked encoding
                    // TODO do not read stream into memory when this issue is fixed in clickhouse
                    entityBuilder.addBinaryBody(externalDataItem.getName(),
                            Utils.toByteArray(externalDataItem.getContent()), ContentType.APPLICATION_OCTET_STREAM,
                            externalDataItem.getName());
                }
            } catch (IOException e) {
                throw new RuntimeException(e);
            }

            requestEntity = entityBuilder.build();
        }

        requestEntity = applyRequestBodyCompression(requestEntity);

        HttpEntity entity = null;
        try {
            uri = followRedirects(uri);
            // 这里开始创建post请求
            HttpPost post = new HttpPost(uri);
            post.setEntity(requestEntity);

            if (parsedStmt.isIdemponent()) {
                // 这个参数表示是否幂等
                httpContext.setAttribute("is_idempotent", Boolean.TRUE);
            } else {
                httpContext.removeAttribute("is_idempotent");
            }

            // post请求的响应结果进行解析，处理异常情况
            HttpResponse response = client.execute(post, httpContext);
            entity = response.getEntity();
            checkForErrorAndThrow(entity, response);

            InputStream is;
            if (entity.isStreaming()) {
                // 这里用的是流式读取
                is = entity.getContent();
            } else {
                FastByteArrayOutputStream baos = new FastByteArrayOutputStream();
                entity.writeTo(baos);
                is = baos.convertToInputStream();
            }

            // retrieve response summary
            if (isQueryParamSet(ClickHouseQueryParam.SEND_PROGRESS_IN_HTTP_HEADERS, additionalClickHouseDBParams,
                    additionalRequestParams)) {
                Header summaryHeader = response.getFirstHeader("X-ClickHouse-Summary");
                currentSummary = summaryHeader != null
                        ? JsonStreamUtils.readObject(summaryHeader.getValue(), ClickHouseResponseSummary.class)
                        : null;
            }

            // 返回输入流
            return is;
        } catch (ClickHouseException e) {
            throw e;
        } catch (Exception e) {
            log.info("Error during connection to %s, reporting failure to data source, message: %s", properties,
                    e.getMessage());
            EntityUtils.consumeQuietly(entity);
            log.info("Error sql: %s", sql);
            throw ClickHouseExceptionSpecifier.specify(e, properties.getHost(), properties.getPort());
        }
    }
```



生成queryId的部分

```javascript
private Map<ClickHouseQueryParam, String> addQueryIdTo(Map<ClickHouseQueryParam, String> parameters) {
    if (this.queryId != null) {
        return parameters;
    }

    String queryId = parameters.get(ClickHouseQueryParam.QUERY_ID);
    if (queryId == null) {
        // TODO perhaps we should use TimeUUID so that it's easy to sort?
        // 这个注释是作者留的，看来作者对这块的queryId的生成也有保留，应该会在后面的版本里去优化
        this.queryId = UUID.randomUUID().toString();
        parameters.put(ClickHouseQueryParam.QUERY_ID, this.queryId);
    } else {
        this.queryId = queryId;
    }

    return parameters;
}
```



updateResult拼装返回结果集部分

```java
protected ResultSet updateResult(ClickHouseSqlStatement stmt, InputStream is)
    throws IOException, ClickHouseException {
    ResultSet rs = null;
    if (stmt.isQuery()) {
        currentUpdateCount = -1;
        // 非常的简单粗暴，new一个返回结果集出来
        currentResult = createResultSet(properties.isCompress() ? new ClickHouseLZ4Stream(is) : is,
                                        properties.getBufferSize(), stmt.getDatabaseOrDefault(properties.getDatabase()), stmt.getTable(),
                                        stmt.hasWithTotals(), this, getConnection().getTimeZone(), properties);
        currentResult.setMaxRows(maxRows);
        rs = currentResult;
    } else {
        currentUpdateCount = 0;
        try {
            is.close();
        } catch (IOException e) {
            log.error("can not close stream: %s", e.getMessage());
        }
    }

    return rs;
}

private ClickHouseResultSet createResultSet(InputStream is, int bufferSize, String db, String table,
                                            boolean usesWithTotals, ClickHouseStatement statement, TimeZone timezone, ClickHouseProperties properties)
    throws IOException {
    // 默认是单向的，因此走else
    if (isResultSetScrollable) {
        return new ClickHouseScrollableResultSet(is, bufferSize, db, table, usesWithTotals, statement, timezone,
                                                 properties);
    } else {
        return new ClickHouseResultSet(is, bufferSize, db, table, usesWithTotals, statement, timezone, properties);
    }
}
```

这里跳转到ClickHouseResultSet的构造器，看看他的构造方法里做了哪些处理

```java
public ClickHouseResultSet(InputStream is, int bufferSize, String db, String table,
                           boolean usesWithTotals, ClickHouseStatement statement, TimeZone timeZone,
                           ClickHouseProperties properties) throws IOException
{
    this.db = db;
    this.table = table;
    this.statement = statement;
    this.properties = properties;
    this.usesWithTotals = usesWithTotals;
    this.dateTimeTimeZone = timeZone;
    this.dateTimeZone = properties.isUseServerTimeZoneForDates()
        ? timeZone
        : TimeZone.getDefault(); // FIXME should be the timezone defined in useTimeZone?
    // 这里用分隔符来获取列名
    bis = new StreamSplitter(is, (byte) 0x0A, bufferSize);  ///   \n
    ByteFragment headerFragment = bis.next();
    if (headerFragment == null) {
        // 如果没有列名会抛出异常
        throw new IllegalArgumentException("ClickHouse response without column names");
    }
    String header = headerFragment.asString(true);
    // 这里判断异常的方式，也挺简单粗暴的- -||
    if (header.startsWith("Code: ") && !header.contains("\t")) {
        is.close();
        throw new IOException("ClickHouse error: " + header);
    }
    // 这里获取列类型，获取不到也会抛出异常
    String[] cols = toStringArray(headerFragment);
    ByteFragment typesFragment = bis.next();
    if (typesFragment == null) {
        throw new IllegalArgumentException("ClickHouse response without column types");
    }
    String[] types = toStringArray(typesFragment);
    columns = new ArrayList<>(cols.length);
    TimeZone tz = null;
    try {
        if (statement != null && statement.getConnection() instanceof ClickHouseConnection) {
            tz = ((ClickHouseConnection)statement.getConnection()).getServerTimeZone();
        }
    } catch (SQLException e) {
        // ignore the error
    }

    if (tz == null) {
        tz = timeZone;
    }
	
    //最终将列信息加到private List<ClickHouseColumnInfo> columns;中
    for (int i = 0; i < cols.length; i++) {
        columns.add(ClickHouseColumnInfo.parse(types[i], cols[i], tz));
    }
}
```

































