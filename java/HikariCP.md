## 连接池测试
- 性能方面 hikariCP>druid>tomcat-jdbc>dbcp>c3p0 。hikariCP的高性能得益于最大限度的避免锁竞争。
- druid功能最为全面，sql拦截等功能，统计数据较为全面，具有良好的扩展性。
- 综合性能，扩展性等方面，可考虑使用druid或者hikariCP连接池。
- 可开启prepareStatement缓存，对性能会有大概20%的提升。
测试：http://blog.csdn.net/qq_31125793/article/details/51241943

功能对比：

|功能|dbcp|druid|c3p0|tomcat-jdbc|HikariCP|
|-|-|-|-|-|-|
|是否支持PSCache	|是	|是	|是	|否	|否|
|监控|jmx|jmx/log/http|jmx,log|	jmx	|jmx|
|扩展性|弱	|好	|弱	|弱	|弱|
|sql拦截及解析|无	|支持	|无	|无	|无
|代码|简单|中等|复杂|简单|简单
|更新时间|2015.8.6|2015.10.10| 2015.12.09||2015.12.3|
|特点|依赖于common-pool|阿里开源，功能全面|历史久远，代码逻辑复杂，且不易维护||优化力度大，功能简单，起源于boneCP|
|连接池管理|LinkedBlockingDequ|数组|	 |FairBlockingQueue|threadlocal+CopyOnWriteArrayList|


## jdbc连接池HikariCP
spring集成了HikariCP
github:https://github.com/brettwooldridge/HikariCP
使用例子

```java 
public class DataSource {  
    private HikariDataSource ds;  
      
    /** 
     * 初始化连接池 
     * @param minimum 
     * @param Maximum 
     */  
    public void init(int minimum,int Maximum){  
        //连接池配置  
        HikariConfig config = new HikariConfig();  
        config.setDriverClassName("com.mysql.jdbc.Driver");  
        config.setJdbcUrl("jdbc:mysql://127.0.0.1:3306/testdb?user=root&password=123456&useUnicode=true&characterEncoding=utf8");  
        config.addDataSourceProperty("cachePrepStmts", true);  
        config.addDataSourceProperty("prepStmtCacheSize", 500);  
        config.addDataSourceProperty("prepStmtCacheSqlLimit", 2048);  
        config.setConnectionTestQuery("SELECT 1");  
        config.setAutoCommit(true);  
        //池中最小空闲链接数量  
        config.setMinimumIdle(minimum);  
        //池中最大链接数量  
        config.setMaximumPoolSize(Maximum);  
        ds = new HikariDataSource(config);      
    }  
      
    /** 
     * 销毁连接池 
     */  
    public void shutdown(){  
        ds.shutdown();  
    }  
      
    /** 
     * 从连接池中获取链接 
     * @return 
     */  
    public Connection getConnection(){  
        try {  
            return ds.getConnection();  
        } catch (SQLException e) {  
            e.printStackTrace();  
            ds.resumePool();  
            return null;  
        }  
    }  
      
    public static void main(String[] args) throws SQLException {  
        DataSource ds = new DataSource();  
        ds.init(10, 50);  
        Connection conn = ds.getConnection();  
        //......  
        //最后关闭链接  
        conn.close();  
    }    
}  
```

## spring配置

```xml
<!-- Hikari Datasource -->  
<bean id="dataSourceHikari" class="com.zaxxer.hikari.HikariDataSource"  destroy-method="shutdown">  
 <!-- <property name="driverClassName" value="${db.driverClass}" /> --> <!-- 无需指定，除非系统无法自动识别 -->  
 <property name="jdbcUrl" value="jdbc:mysql://localhost:3306/test?useUnicode=true&characterEncoding=UTF-8" />  
 <property name="username" value="${db.username}" />  
 <property name="password" value="${db.password}" />  
  <!-- 连接只读数据库时配置为true， 保证安全 -->  
 <property name="readOnly" value="false" />  
 <!-- 等待连接池分配连接的最大时长（毫秒），超过这个时长还没可用的连接则发生SQLException， 缺省:30秒 -->  
 <property name="connectionTimeout" value="30000" />  
 <!-- 一个连接idle状态的最大时长（毫秒），超时则被释放（retired），缺省:10分钟 -->  
 <property name="idleTimeout" value="600000" />  
 <!-- 一个连接的生命时长（毫秒），超时而且没被使用则被释放（retired），缺省:30分钟，建议设置比数据库超时时长少30秒，参考MySQL wait_timeout参数（show variables like '%timeout%';） -->  
 <property name="maxLifetime" value="1800000" />  
 <!-- 连接池中允许的最大连接数。缺省值：10；推荐的公式：((core_count * 2) + effective_spindle_count) -->  
 <property name="maximumPoolSize" value="15" />  
</bean>  
```
maxLifetime、maximumPoolSize需要自己去算一下
其他配置（sqlSessionFactory、MyBatis MapperScannerConfigurer、transactionManager等）使用默认就可以