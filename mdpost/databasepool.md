#DBCP数据连接池
![](./source/images/38410531_xl.jpg "")
DBCP数据连接池参数介绍，相关配置，和DBCP是如何创建数据源，如何缓存和管理连接池的；最核心的部分，一个是连接池，第二个是连接，第三个则是连接池和连接的关系，在此表现为一对多的互相引用。
<!--more-->
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
自动检查连接的可用性，dbcp定时检测连接，dbcp自动重连的配置说明：
maxIdle值与maxActive值应配置的接近。
因为，当连接数超过maxIdle值后，刚刚使用完的连接（刚刚空闲下来）会立即被销毁。而不是我想要的空闲M秒后再销毁起一个缓冲作用。这一点DBCP做的可能与你想像的不一样。
若maxIdle与maxActive相差较大，在高负载的系统中会导致频繁的创建、销毁连接，连接数在maxIdle与maxActive间快速频繁波动，这不是我想要的。
高负载系统的maxIdle值可以设置为与maxActive相同或设置为-1(-1表示不限制)，让连接数量在minIdle与maxIdle间缓冲慢速波动。

timeBetweenEvictionRunsMillis建议设置值
initialSize="5"，会在tomcat一启动时，创建5条连接，效果很理想。
但同时我们还配置了minIdle="10"，也就是说，最少要保持10条连接，那现在只有5条连接，哪什么时候再创建少的5条连接呢？
1、等业务压力上来了， DBCP就会创建新的连接。
2、配置timeBetweenEvictionRunsMillis=“时间”,DBCP会启用独立的工作线程定时检查，补上少的5条连接。销毁多余的连接也是同理。
下面就是DBCP连接池，同时使用了以上两个方案的配置配置
    validationQuery = "SELECT 1"  验证连接是否可用，使用的SQL语句
    testWhileIdle = "true"      指明连接是否被空闲连接回收器(如果有)进行检验.如果检测失败,则连接将被从池中去除.
    testOnBorrow = "false"   借出连接时不要测试，否则很影响性能
    timeBetweenEvictionRunsMillis = "30000"  每30秒运行一次空闲连接回收器
    minEvictableIdleTimeMillis = "1800000"  池中的连接空闲30分钟后被回收,默认值就是30分钟。
    numTestsPerEvictionRun="3" 在每次空闲连接回收器线程(如果有)运行时检查的连接数量，默认值就是3.

    解释：
    配置timeBetweenEvictionRunsMillis = "30000"后，每30秒运行一次空闲连接回收器（独立线程）。并每次检查3个连接，如果连接空闲时间超过30分钟就销毁。销毁连接后，连接数量就少了，如果小于minIdle数量，就新建连接，维护数量不少于minIdle，过行了新老更替。
    testWhileIdle = "true" 表示每30秒，取出3条连接，使用validationQuery = "SELECT 1" 中的SQL进行测试 ，测试不成功就销毁连接。销毁连接后，连接数量就少了，如果小于minIdle数量，就新建连接。
    testOnBorrow = "false" 一定要配置，因为它的默认值是true。false表示每次从连接池中取出连接时，不需要执行validationQuery = "SELECT 1" 中的SQL进行测试。若配置为true,对性能有非常大的影响，性能会下降7-10倍。所在一定要配置为false.
    每30秒，取出numTestsPerEvictionRun条连接（本例是3，也是默认值），发出"SELECT 1" SQL语句进行测试 ，测试过的连接不算是“被使用”了，还算是空闲的。连接空闲30分钟后会被销毁。

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

        // 创建返回driver原始物理连接的工厂，使用数据库驱动来创建最底层的JDBC连接
        ConnectionFactory driverConnectionFactory = createConnectionFactory();

        // 为我们的连接创建一个池，缓存JDBC连接
        createConnectionPool();

        // 设置statement pool，如果需要就设置statement的缓存池，这个一般不需要设置
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

        /**
         * 创建连接池工厂
         * 这个工厂使用上述driverConnectionFactory来创建底层JDBC连接，然后包装出一个PoolableConnection，
         * 这个PoolableConnection与连接池设置了一对多的关系，也就是说，连接池中存在多个PoolableConnection，
         * 每个PoolableConnection都关联同一个连接池，这样的好处是便于该表PoolableConnection的close方法的行为，具体会在后面详细分析。
            并将PoolableConnectionFactory赋值到连接池GenericObjectPool：_pool.setFactory(this);
            后面就可以进行调addObject()
         */
        createPoolableConnectionFactory(driverConnectionFactory, statementPoolFactory, abandonedConfig);

        // 创建并返回数据源管理实例
        createDataSourceInstance();

        // 初始化 为连接池中添加PoolableConnection
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
初始化添加连接：
```
 public void addObject() throws Exception {
        assertOpen();
        if (_factory == null) {
            throw new IllegalStateException("Cannot add objects without a factory.");
        }
        Object obj = _factory.makeObject();
        try {
            assertOpen();
            addObjectToPool(obj, false);
        } catch (IllegalStateException ex) { // Pool closed
            try {
                _factory.destroyObject(obj);
            } catch (Exception ex2) {
                // swallow
            }
            throw ex;
        }
    }

    public Object makeObject() throws Exception {
            Connection conn = _connFactory.createConnection();
            if (conn == null) {
                throw new IllegalStateException("Connection factory returned null from createConnection");
            }
            initializeConnection(conn);
            if(null != _stmtPoolFactory) {
                KeyedObjectPool stmtpool = _stmtPoolFactory.createPool();
                conn = new PoolingConnection(conn,stmtpool);
                stmtpool.setFactory((PoolingConnection)conn);
            }
            return new PoolableConnection(conn,_pool,_config);
        }
```
保存链接的数据结构：CursorableLinkedList _pool
```
   synchronized (this) {
               if (isClosed()) {
                   shouldDestroy = true;
               } else {
                   if((_maxIdle >= 0) && (_pool.size() >= _maxIdle)) {
                       shouldDestroy = true;
                   } else if(success) {
                       // borrowObject always takes the first element from the queue,
                       // so for LIFO, push on top, FIFO add to end
                       if (_lifo) {
                           _pool.addFirst(new ObjectTimestampPair(obj));
                       } else {
                           _pool.addLast(new ObjectTimestampPair(obj));
                       }
                       if (decrementNumActive) {
                           _numActive--;
                       }
                       allocate();
                   }
               }
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
borrowObject方法：
```

```
##三、连接池：
```
protected volatile GenericObjectPool connectionPool = null;
```
apache的common-pool工具库中有5种对象池:GenericObjectPool和GenericKeyedObjectPool,
SoftReferenceObjectPool, StackObjectPool, StackKeyedObjectPool.
五种对象池可分为两类, 一类是无key的:
前面两种用CursorableLinkedList来做容器, SoftReferenceObjectPool用ArrayList做容器, 一次性创建所有池化对象,
 并对容器中的对象进行了软引用(SoftReference)处理, 从而保证在内存充足的时候池对象不会轻易被jvm垃圾回收,
 从而具有很强的缓存能力. 最后两种用Stack做容器. 不带key的对象池是对前面池技术原理的一种简单实现, 带key的相对复杂一些,
  它会将池对象按照key来进行分类, 具有相同的key被划分到一组类别中, 因此有多少个key, 就会有多少个容器.
   之所以需要带key的这种对象池, 是因为普通的对象池通过makeObject()方法创建的对象基本上都是一模一样的, 因为没法传递参数来对池对象进行定制.
   因此四种池对象的区别主要体现在内部的容器的区别, Stack遵循"后进先出"的原则并能保证线程安全,
    CursorableLinkedList是一个内部用游标(cursor)来定位当前元素的双向链表, 是非线程安全的, 但是能满足对容器的并发修改.
    ArrayList是非线程安全的, 遍历方便的容器.

使用对象池的一般步骤:创建一个池对象工厂, 将该工厂注入到对象池中, 当要取池对象, 调用borrowObject,
当要归还池对象时, 调用returnObject, 销毁池对象调用clear(), 如果要连池对象工厂也一起销毁, 则调用close().

而对应的对象池前者采用的是ObjectPool, 后者是KeyedObjectPool, 因为一个数据库只对应一个连接, 而执行操作的Statement却根据Sql的不同会分很多种.
 因此需要根据sql语句的不同多次进行缓存
在对连接池的管理上, common-dbcp主要采用两种对象:
一个是PoolingDriver, 另一个是PoolingDataSource, 二者的区别是PoolingDriver是一个更底层的操作类, 它持有一个连接池映射列表,
一般针对在一个jvm中要连接多个数据库, 而后者相对简单一些. 内部只能持有一个连接池, 即一个数据源对应一个连接池.

整个数据源最核心的其实就三个东西：
一个是连接池，在这里体现为common-pool中的GenericObjectPool，它负责缓存和管理连接，所有的配置策略都是由它管理。
第二个是连接，这里的连接就是PoolableConnection，当然它是对底层连接进行了封装。
第三个则是连接池和连接的关系，在此表现为一对多的互相引用。对数据源的构建则是对连接池，连接以及连接池与连接的关系的构建。
##三、参考资料
BasicDataSource
https://blog.csdn.net/kang389110772/article/details/52572930
深入理解JDBC的timeout
https://blog.csdn.net/kobejayandy/article/details/46916063
对象池相关
https://blog.csdn.net/suixinm/article/details/41761019
mysql参数配置
https://dev.mysql.com/doc/connector-j/5.1/en/connector-j-reference-configuration-properties.html
```
