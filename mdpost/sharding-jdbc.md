# my-sharding-jdbc
![](/images/12564740_xl.jpg "")
ShardingJDBC001简单的例子，配置信息的初始化,以及源码解析
<!--more-->
## 使用示例
ShardingJDBC
```
package com.zz.sharding.jdbc.test;

import io.shardingjdbc.core.api.ShardingDataSourceFactory;
import io.shardingjdbc.core.api.config.ShardingRuleConfiguration;
import io.shardingjdbc.core.api.config.TableRuleConfiguration;
import io.shardingjdbc.core.api.config.strategy.InlineShardingStrategyConfiguration;
import org.apache.commons.dbcp.BasicDataSource;

import javax.sql.DataSource;
import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.util.HashMap;
import java.util.Map;
import java.util.Properties;
import java.util.concurrent.ConcurrentHashMap;


/**
 * 当当分库分表组件实践
 *
 * @author zhangzuizui
 * @date 2018/1/9
 */
public class ShardingJDBC001 {

    public static void main(String[] args) throws SQLException {
        Map<String, DataSource> dataSourceMap = new HashMap<>();
        // 配置第一个数据源
        dataSourceMap.put("my_shard_01",createDataSource("my_shard_01"));

        // 配置第二个数据源
        dataSourceMap.put("my_shard_02", createDataSource("my_shard_02"));

        // 配置Order表规则
        TableRuleConfiguration orderTableRuleConfig = new TableRuleConfiguration();
        orderTableRuleConfig.setLogicTable("my_order");
        orderTableRuleConfig.setActualDataNodes("my_shard_0${1..2}.my_order_00${1..2}");

        // 配置分库策略
        orderTableRuleConfig.setDatabaseShardingStrategyConfig(new InlineShardingStrategyConfiguration("order_id", "my_shard_0${1}"));

        // 配置分表策略
        orderTableRuleConfig.setTableShardingStrategyConfig(new InlineShardingStrategyConfiguration("order_id", "my_order_00${1}"));

        // 配置分片规则
        ShardingRuleConfiguration shardingRuleConfig = new ShardingRuleConfiguration();
        shardingRuleConfig.getTableRuleConfigs().add(orderTableRuleConfig);

        // 省略配置order_item表规则...

        // 获取数据源对象
        DataSource dataSource = ShardingDataSourceFactory.createDataSource(dataSourceMap, shardingRuleConfig, new ConcurrentHashMap(), new Properties());

        String sql = "SELECT * FROM my_order WHERE order_id = ?";
        //2.获取连接
        Connection conn = dataSource.getConnection();
        /**
         * PrepareStatement接口是Statement接口的子接口，他继承了Statement接口的所有功能。它主要是拿来解决我们使用Stateme
         * ParperStatement接口的机制是在数据库支持预编译的情况下预先将SQL语句编译，当多次执行这条SQL语句时，可以直接执行编译好的SQL语句，
         * sql预处理
         */
        PreparedStatement pstmt = conn.prepareStatement(sql);
        pstmt.setString(1, "001");
        //4.SQL执行和结果归并
        ResultSet rs = pstmt.executeQuery();
        //5.获取结果
        while(rs.next()) {
            System.out.println("id="+rs.getInt(1)+",order_id="+rs.getString(2));
        }
    }

    /**
     * 创建数据源
     * @param dataSourceName
     * @return
     */
    private static DataSource createDataSource(String dataSourceName) {
        BasicDataSource result = new BasicDataSource();
        result.setDriverClassName(com.mysql.jdbc.Driver.class.getName());
        result.setUrl(String.format("jdbc:mysql://localhost:3306/%s", dataSourceName));
        result.setUsername("root");
        result.setPassword("123456");
        return result;
    }
}
```
##一、构建ShardingDataSource数据源：初始化配置信息
```
DataSource dataSource = ShardingDataSourceFactory.createDataSource()
```
###详细
```
1.public static DataSource createDataSource(final Map<String, DataSource> dataSourceMap, final ShardingRuleConfiguration shardingRuleConfig,
                                              final Map<String, Object> configMap, final Properties props) throws SQLException {
        return new ShardingDataSource(shardingRuleConfig.build(dataSourceMap), configMap, props);
    }
其中shardingRuleConfig.build(dataSourceMap)构建分库和分表的规则配置

2.public ShardingDataSource(final ShardingRule shardingRule, final Map<String, Object> configMap, final Properties props) throws SQLException {
        super(shardingRule.getDataSourceMap().values());
        if (!configMap.isEmpty()) {
            ConfigMapContext.getInstance().getShardingConfig().putAll(configMap);
        }
        shardingProperties = new ShardingProperties(null == props ? new Properties() : props);
        int executorSize = shardingProperties.getValue(ShardingPropertiesConstant.EXECUTOR_SIZE);
        executorEngine = new ExecutorEngine(executorSize);
        boolean showSQL = shardingProperties.getValue(ShardingPropertiesConstant.SQL_SHOW);
        shardingContext = new ShardingContext(shardingRule, getDatabaseType(), executorEngine, showSQL);
    }
3.初始化ShardingContext
```
二、获取数据库连接
```
Connection conn = dataSource.getConnection();
new ShardingConnection(shardingContext);
```
三、sql预处理
```
PreparedStatement pstmt = conn.prepareStatement(sql);

1.初始化sql预处理器：ShardingPreparedStatement
public ShardingPreparedStatement(final ShardingConnection connection, final String sql, final int resultSetType, final int resultSetConcurrency, final int resultSetHoldability) {
        this.connection = connection;
        this.resultSetType = resultSetType;
        this.resultSetConcurrency = resultSetConcurrency;
        this.resultSetHoldability = resultSetHoldability;
        routingEngine = new PreparedStatementRoutingEngine(sql, connection.getShardingContext());
    }
2.构建路由引擎：PreparedStatementRoutingEngine
routingEngine = new PreparedStatementRoutingEngine(sql, connection.getShardingContext());
public static SQLRouter createSQLRouter(final ShardingContext shardingContext) {
        return HintManagerHolder.isDatabaseShardingOnly() ? new DatabaseHintSQLRouter(shardingContext) : new ParsingSQLRouter(shardingContext);
    }
3.根据isDatabaseShardingOnly判断构造那个sqlRouter
 提示sqlRouter:DatabaseHintSQLRouter
 解释sqlRouter:ParsingSQLRouter
该步骤构造了sql执行所需的路由引擎和
```
##一、路由库和表
```
@Override
public ResultSet executeQuery() throws SQLException {
        ResultSet result;
        try {
            Collection<PreparedStatementUnit> preparedStatementUnits = route();
            List<ResultSet> resultSets = new PreparedStatementExecutor(
                    getConnection().getShardingContext().getExecutorEngine(), routeResult.getSqlStatement().getType(), preparedStatementUnits, getParameters()).executeQuery();
            result = new ShardingResultSet(resultSets, new MergeEngine(resultSets, (SelectStatement) routeResult.getSqlStatement()).merge(), this);
        } finally {
            clearBatch();
        }
        currentResultSet = result;
        return result;
    }
1.构建路由preparedStatementUnits
   routeResult = routingEngine.route(getParameters());

    @Override
    public SQLRouteResult route(final String logicSQL, final List<Object> parameters, final SQLStatement sqlStatement) {
        SQLRouteResult result = new SQLRouteResult(sqlStatement);
        if (sqlStatement instanceof InsertStatement && null != ((InsertStatement) sqlStatement).getGeneratedKey()) {
            processGeneratedKey(parameters, (InsertStatement) sqlStatement, result);
        }
        RoutingResult routingResult = route(parameters, sqlStatement);
        SQLRewriteEngine rewriteEngine = new SQLRewriteEngine(shardingRule, logicSQL, databaseType, sqlStatement);
        boolean isSingleRouting = routingResult.isSingleRouting();
        if (sqlStatement instanceof SelectStatement && null != ((SelectStatement) sqlStatement).getLimit()) {
            processLimit(parameters, (SelectStatement) sqlStatement, isSingleRouting);
        }
        SQLBuilder sqlBuilder = rewriteEngine.rewrite(!isSingleRouting);
        if (routingResult instanceof CartesianRoutingResult) {
            for (CartesianDataSource cartesianDataSource : ((CartesianRoutingResult) routingResult).getRoutingDataSources()) {
                for (CartesianTableReference cartesianTableReference : cartesianDataSource.getRoutingTableReferences()) {
                    result.getExecutionUnits().add(new SQLExecutionUnit(cartesianDataSource.getDataSource(), rewriteEngine.generateSQL(cartesianTableReference, sqlBuilder)));
                }
            }
        } else {
            for (TableUnit each : routingResult.getTableUnits().getTableUnits()) {
                result.getExecutionUnits().add(new SQLExecutionUnit(each.getDataSourceName(), rewriteEngine.generateSQL(each, sqlBuilder)));
            }
        }
        if (showSQL) {
            SQLLogger.logSQL(logicSQL, sqlStatement, result.getExecutionUnits(), parameters);
        }
        return result;
    }
3.
//sql声明
sqlStatement = sqlRouter.parse(logicSQL, parameters.size());
//构造SQL解析引擎
SQLParsingEngine parsingEngine = new SQLParsingEngine(databaseType, logicSQL, shardingRule);
//SQL解析引擎调用Lexer进行sql词法解析
SQLStatement result = parsingEngine.parse();
public SQLStatement parse() {
        //构建LexerEngine,构造Lexer(String input, Dictionary dictionary)
        LexerEngine lexerEngine = LexerEngineFactory.newInstance(dbType, sql);
        //解析
        lexerEngine.nextToken();

        return SQLParserFactory.newInstance(dbType, lexerEngine.getCurrentToken().getType(), shardingRule, lexerEngine).parse();
    }

创建数据库：SQLParserFactory.newInstance
根据数据库类型创建 ：例如：MySQLSelectParser
然后调用
    @Override
    public final SelectStatement parse() {
        SelectStatement result = parseInternal();
        if (result.containsSubQuery()) {
            result = result.mergeSubQueryStatement();
        }
        // TODO move to rewrite
        appendDerivedColumns(result);
        appendDerivedOrderBy(result);
        return result;
    }
    //具体解析sql
    @Override
    protected void parseInternal(final SelectStatement selectStatement) {
        parseDistinct();
        parseSelectOption();
        parseSelectList(selectStatement, getItems());
        parseFrom(selectStatement);
        parseWhere(getShardingRule(), selectStatement, getItems());
        parseGroupBy(selectStatement);
        parseHaving();
        parseOrderBy(selectStatement);
        parseLimit(selectStatement);
        parseSelectRest();
    }
这里大部分调用lexerEngine中的方法进行sql词法解析：
```
##路由之后执行准备sql预处理
```
for (SQLExecutionUnit each : routeResult.getExecutionUnits()) {
   SQLType sqlType = routeResult.getSqlStatement().getType();
   Collection<PreparedStatement> preparedStatements;
   //事务类型的sql，单独处理
   if (SQLType.DDL == sqlType) {
       preparedStatements = generatePreparedStatementForDDL(each);
   } else {
       preparedStatements = Collections.singletonList(generatePreparedStatement(each));
   }
   routedStatements.addAll(preparedStatements);
   for (PreparedStatement preparedStatement : preparedStatements) {
       replaySetParameter(preparedStatement);
       result.add(new PreparedStatementUnit(each, preparedStatement));
   }
}
```
##二、创建sql处理器并执行sql
```
 List<ResultSet> resultSets = new PreparedStatementExecutor(
                    getConnection().getShardingContext().getExecutorEngine(), routeResult.getSqlStatement().getType(), preparedStatementUnits, getParameters())
                    .executeQuery();
```
##二、结果归并处理
```
result = new ShardingResultSet(resultSets, new MergeEngine(resultSets, (SelectStatement) routeResult.getSqlStatement()).merge(), this);
回调，jdbc底层执行sql，并处理结果集
return executorEngine.executePreparedStatement(sqlType, preparedStatementUnits, parameters, new ExecuteCallback<ResultSet>() {
            @Override
            public ResultSet execute(final BaseStatementUnit baseStatementUnit) throws Exception {
                return ((PreparedStatement) baseStatementUnit.getStatement()).executeQuery();
            }
        });
```



