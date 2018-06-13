#数据连接池DBCP
数据连接池DBCP
<!--more-->
##参数介绍：
```
maxActive：一个时间最大的活跃连接数，负数代表没有上限
maxIdle：池中最大的空闲对象保持数，其他的将会释放，负数代表没有上限
minIdle：池中最小的空闲对象保持数，其他的将会被创建，0代表不保持
maxWait：当池耗尽时等待空闲对象的最长时间
maxWaitMillis：获取连接最大等待时间,-1代表一直等
timeBetweenEvictionRunsMillis：空闲对象被清除的时间间隔
poolPreparedStatements：Statement pool true时包含PreparedStatements和CallableStatements
numTestsPerEvictionRun：每次运行清除空闲对象线程时要检查的对象数
minEvictableIdleTimeMillis：空闲对象在可被清除之前的最小的停留时间
testWhileIdle：指示对象是否由空闲对象（如果有的话）验证。 如果一个对象无法验证，它将从池中删除。默认false
maxOpenPreparedStatements = GenericKeyedObjectPool.DEFAULT_MAX_TOTAL （-1）:
最大开放语句数声明池。 一个连接通常每次只使用一个或两个语句，这是主要用于帮助检测资源泄漏
removeAbandonedTimeout:超时多少秒删除连接
```
BasicDataSource配置：
```
 <!-- datasource setting -->
    <bean id="basicDataSource" class="org.apache.commons.dbcp.BasicDataSource" abstract="true"
          init-method="createDataSource">
        <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
        <!-- 一个时间最大的活跃连接数，负数代表没有上限 -->
        <property name="maxActive" value="10"/>
        <!-- 池中空闲对象的最大数量 -->
        <property name="maxIdle" value="10"/>
        <!-- 池中空闲对象的最小数量 -->
        <property name="minIdle" value="5"/>
        <!-- 当池耗尽时等待空闲对象的最长时间 -->
        <property name="maxWait" value="3000"/>
        <!-- 池启动时创建的初始连接数 -->
        <property name="initialSize" value="20"/>
        <!-- 空闲时间：空闲对象被清除的时间间隔-->
        <property name="timeBetweenEvictionRunsMillis" value="60000"/>
        <!-- Statement pool true时包含PreparedStatements和CallableStatements -->
        <property name="poolPreparedStatements" value="true"/>
        <!-- 最大开放语句数声明池 -->
        <property name="maxOpenPreparedStatements" value="50"/>
        <!-- 超时多少秒删除连接 -->
        <property name="removeAbandonedTimeout" value="180"/>
        <!-- 申请连接时执行validationQuery检测连接是否有效，配置为true会降低性能 -->
        <property name="testOnBorrow" value="false"/>
        <!-- 归还连接时执行validationQuery检测连接是否有效，配置为true会降低性能  -->
        <property name="testOnReturn" value="false"/>
        <!-- 建议配置为true，不影响性能，并且保证安全性。申请连接的时候检测，如果空闲时间大于
             timeBetweenEvictionRunsMillis，执行validationQuery检测连接是否有效。  -->
        <property name="testWhileIdle" value="true"/>
        <!-- 用来检测连接是否有效的sql，要求是一个查询语句,如果validationQuery为
             null，testOnBorrow、testOnReturn、testWhileIdle都不起其作用。 -->
        <property name="validationQuery" value="select 1;"/>
    </bean>
```

##1、创建数据源：createDataSource

    a.创建drive工厂：createConnectionFactory
    b.创建GenericObjectPool connectionPool;
    c.statementPoolFactory:GenericKeyedObjectPoolFactory
    d.创建DataSource实例:
    PoolingDataSource pds = new PoolingDataSource(connectionPool);

```
//创建数据源
protected synchronized DataSource createDataSource()
        throws SQLException {
        //如果closed标识为关闭，直接抛异常
        if (closed) {
            throw new SQLException("Data source is closed");
        }
        //不为空，直接返回
        if (dataSource != null) {
            return (dataSource);
        }

        // 创建返回driver原始物理连接的工厂
        ConnectionFactory driverConnectionFactory = createConnectionFactory();

        // 为我们的连接创建一个池
        createConnectionPool();

        // 设置statement pool
        工厂创建的池使用KeyedPoolableObjectFactory。
              * @param maxActive一次可以从池中借用的最大对象数量。
              * @param whenExhaustedAction在池耗尽时采取的操作。
              * @param maxWait当池耗尽时等待空闲对象的最长时间。
              * @param maxIdle池中空闲对象的最大数量。
              * @param maxTotal一次可以存在的最大对象数量。
        GenericKeyedObjectPoolFactory statementPoolFactory = null;
        if (isPoolPreparedStatements()) {
            statementPoolFactory = new GenericKeyedObjectPoolFactory(null,
                        -1, // unlimited maxActive (per key)
                        GenericKeyedObjectPool.WHEN_EXHAUSTED_FAIL,
                        0, // maxWait：当池耗尽时等待空闲对象的最长时间
                        1, // maxIdle (per key)：池中空闲对象的最大数量。
                        maxOpenPreparedStatements);
        }

        // 创建连接池工厂
        createPoolableConnectionFactory(driverConnectionFactory, statementPoolFactory, abandonedConfig);

        // 创建并返回数据源管理实例 pooling data source to manage the connections
        createDataSourceInstance();

        try {
            for (int i = 0 ; i < initialSize ; i++) {
                connectionPool.addObject();
            }
        } catch (Exception e) {
            throw new SQLNestedException("Error preloading the connection pool", e);
        }

        return dataSource;
    }
```

##二、使用DataSource获取连接：PoolingDataSource
```
public Connection getConnection() throws SQLException {
        try {
            Connection conn = (Connection)(_pool.borrowObject());
            if (conn != null) {
                conn = new PoolGuardConnectionWrapper(conn);
            }
            return conn;
        } catch(SQLException e) {
            throw e;
        } catch(NoSuchElementException e) {
            throw new SQLNestedException("Cannot get a connection, pool error " + e.getMessage(), e);
        } catch(RuntimeException e) {
            throw e;
        } catch(Exception e) {
            throw new SQLNestedException("Cannot get a connection, general error", e);
        }
    }
```
##三、参考资料
BasicDataSource
https://blog.csdn.net/kang389110772/article/details/52572930
深入理解JDBC的timeout
https://blog.csdn.net/kobejayandy/article/details/46916063
mysql参数配置
https://dev.mysql.com/doc/connector-j/5.1/en/connector-j-reference-configuration-properties.html
```
