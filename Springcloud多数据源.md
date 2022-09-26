前段时间在做会员中心和中间件系统开发时，多次碰到多数据源配置问题，主要用到分包方式、参数化切换、注解+AOP、动态添加 这四种方式。这里做一下总结，分享下使用心得以及踩过的坑。

# 四种方式对比

文章比较长，首先给出四种实现方式的对比，大家可以根据自身需要，选择阅读。

|  | 分包方式 | 参数化切换 | 注解方式 | 动态添加方式 |
| --- | --- | --- | --- | --- |
| 适用场景 | 编码时便知道用哪个数据源 | 运行时才能确定用哪个数据源 | 编码时便知道用哪个数据源 | 运行时动态添加新数据源 |
| 切换模式 | 自动 | 手动 | 自动 | 手动 |
| mapper路径 | 需要分包 | 无要求 | 无要求 | 无要求 |
| 增加数据源是否需要修改配置类 | 需要 | 不需要 | 不需要 | \\ |
| 实现复杂度 | 简单 | 复杂 | 复杂 | 复杂 |

# 分包方式

## 数据源配置文件

在yml中，配置两个数据源，id分别为master和s1。

```yaml
spring:
  datasource:
    master:
      jdbcUrl: jdbc:mysql://192.168.xxx.xxx:xxxx/db1?.........
      username: xxx
      password: xxx
      driverClassName: com.mysql.cj.jdbc.Driver
    s1:
      jdbcUrl: jdbc:mysql://192.168.xxx.xxx:xxxx/db2?........
      username: xxx
      password: xxx
      driverClassName: com.mysql.cj.jdbc.Driver
```

## 数据源配置类

### master数据源配置类

注意点：

需要用@Primary注解指定默认数据源，否则spring不知道哪个是主数据源；

```java
@Configuration
@MapperScan(basePackages = "com.hosjoy.xxx.xxx.xxx.xxx.mapper.master", sqlSessionFactoryRef = "masterSqlSessionFactory")
public class MasterDataSourceConfig {

    //默认数据源
    @Bean(name = "masterDataSource")
    @Primary
    @ConfigurationProperties(prefix = "spring.datasource.master")
    public HikariDataSource masterDataSource() {
        return DataSourceBuilder.create().type(HikariDataSource.class).build();
    }

    @Bean(name = "masterSqlSessionFactory")
    @Primary
    public SqlSessionFactory masterSqlSessionFactory(@Qualifier("masterDataSource") DataSource datasource, PaginationInterceptor paginationInterceptor)
            throws Exception {
        MybatisSqlSessionFactoryBean bean = new MybatisSqlSessionFactoryBean();
        bean.setDataSource(datasource);
        bean.setMapperLocations(
                // 设置mybatis的xml所在位置
                new PathMatchingResourcePatternResolver().getResources("classpath*:mapper/master/**/**.xml"));
        bean.setPlugins(new Interceptor[]{paginationInterceptor});
        return bean.getObject();
    }

    @Bean(name = "masterSqlSessionTemplate")
    @Primary
    public SqlSessionTemplate masterSqlSessionTemplate(
            @Qualifier("masterSqlSessionFactory") SqlSessionFactory sessionfactory) {
        return new SqlSessionTemplate(sessionfactory);
    }
}
```

### s1数据源配置类

```java
@Configuration
@MapperScan(basePackages = "com.hosjoy.xxx.xxx.xxx.xxx.mapper.s1", sqlSessionFactoryRef = "s1SqlSessionFactory")
public class S1DataSourceConfig {

    @Bean(name = "s1DataSource")
    @ConfigurationProperties(prefix = "spring.datasource.s1")
    public HikariDataSource s1DataSource() {
        return DataSourceBuilder.create().type(HikariDataSource.class).build();
    }

    @Bean(name = "s1SqlSessionFactory")
    public SqlSessionFactory s1SqlSessionFactory(@Qualifier("s1DataSource") DataSource datasource
            , PaginationInterceptor paginationInterceptor)
            throws Exception {
        MybatisSqlSessionFactoryBean bean = new MybatisSqlSessionFactoryBean();
        bean.setDataSource(datasource);
        bean.setMapperLocations(
                // 设置mybatis的xml所在位置
                new PathMatchingResourcePatternResolver().getResources("classpath*:mapper/s1/**/**.xml"));
        bean.setPlugins(new Interceptor[]{paginationInterceptor});
        return bean.getObject();
    }

    @Bean(name = "s1SqlSessionTemplate")
    public SqlSessionTemplate s1SqlSessionTemplate(
            @Qualifier("s1SqlSessionFactory") SqlSessionFactory sessionfactory) {
        return new SqlSessionTemplate(sessionfactory);
    }
}
```

## 使用

可以看出，mapper接口、xml文件，需要按照不同的数据源分包。在操作数据库时，根据需要在service类中注入dao层。

## 特点分析

### 优点

实现起来简单，只需要编写数据源配置文件和配置类，mapper接口和xml文件注意分包即可。

### 缺点

很明显，如果后面要增加或删除数据源，不仅要修改数据源配置文件，还需要修改配置类。

例如增加一个数据源，同时还需要新写一个该数据源的配置类，同时还要考虑新建mapper接口包、xml包等，没有实现 “热插拔” 效果。

# 参数化切换方式

## 思想

参数化切换数据源，意思是，业务侧需要根据当前业务参数，动态的切换到不同的数据源。

这与分包思想不同。分包的前提是在编写代码的时候，就已经知道当前需要用哪个数据源，而参数化切换数据源需要根据业务参数决定用哪个数据源。

例如，请求参数userType值为1时，需要切换到数据源slave1；请求参数userType值为2时，需要切换到数据源slave2。

```java
/**伪代码**/
int userType = reqUser.getType();
if (userType == 1){
    //切换到数据源slave1
    //数据库操作
} else if(userType == 2){
    //切换到数据源slave2
    //数据库操作
}
```

## 设计思路

### 数据源注册

数据源配置类创建datasource时，从yml配置文件中读取所有数据源配置，自动创建每个数据源，并注册至bean工厂和AbstractRoutingDatasource(后面聊聊这个)，同时返回默认的数据源master。

![image-20200701112808482](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b7190cf2fa954e0da7fe008feb816a7e~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

### 数据源切换

（1）通过线程池处理请求，每个请求独占一个线程，这样每个线程切换数据源时互不影响。

（2）根据业务参数获取应切换的数据源ID，根据ID从数据源缓存池获取数据源bean；

（3）生成当前线程数据源key；

（4）将key设置到threadLocal;

（5）将key和数据源bean放入数据源缓存池；

（6）在执行mapper方法前，spring会调用determineCurrentLookupKey方法获取key，然后根据key去数据源缓存池取出数据源，然后getConnection获取该数据源连接；

（7）使用该数据源执行数据库操作；

（8）释放当前线程数据源。

![image-20200701112808482](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9ca7d556434c4a7eb6bc7e56c81a70a9~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

## AbstractRoutingDataSource源码分析

spring为我们提供了AbstractRoutingDataSource抽象类，该类就是实现动态切换数据源的关键。

我们看下该类的类图，其实现了DataSource接口。

![image-20200701151212528](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/67bd4867a8a54d748a4069967c6c1bfe~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

我们看下它的getConnection方法的逻辑，其首先调用determineTargetDataSource来获取数据源，再获取数据库连接。很容易猜想到就是这里来决定具体使用哪个数据源的。

![image-20200701151449823](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/92a9f008fe2d4a7bb60820a58ff2e9d9~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

进入到determineTargetDataSource方法，我们可以看到它先是调用determineCurrentLookupKey获取到一个lookupKey，然后根据这个key去resolvedDataSources里去找相应的数据源。

![image-20200701151728497](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/92a3e6de8e0742d7bf0f47349bb82edf~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

看下该类定义的几个对象，defaultTargetDataSource是默认数据源，resolvedDataSources是一个map对象，存储所有主从数据源。

![image-20200701151932020](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c032b0eb499d4aa9b58a4dd80b262f97~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

所以，关键就是这个lookupKey的获取逻辑，决定了当前获取的是哪个数据源，然后执行getConnection等一系列操作。determineCurrentLookupKey是AbstractRoutingDataSource类中的一个抽象方法，而它的返回值是你所要用的数据源dataSource的key值，有了这个key值，resolvedDataSource（这是个map,由配置文件中设置好后存入的）就从中取出对应的DataSource，如果找不到，就用配置默认的数据源。

所以，通过扩展AbstractRoutingDataSource类，并重写其中的determineCurrentLookupKey()方法，可以实现数据源的切换。

## 代码实现

下面贴出关键代码实现。

### 数据源配置文件

这里配了3个数据源，其中主数据源是MySQL，两个从数据源是sqlserver。

```yaml
spring:
  datasource:
    master:
      jdbcUrl: jdbc:mysql://192.168.xx.xxx:xxx/db1?........
      username: xxx
      password: xxx
      driverClassName: com.mysql.cj.jdbc.Driver
    slave1:
      jdbcUrl: jdbc:sqlserver://192.168.xx.xxx:xxx;DatabaseName=db2
      username: xxx
      password: xxx
      driverClassName: com.microsoft.sqlserver.jdbc.SQLServerDriver
    slave2:
      jdbcUrl: jdbc:sqlserver://192.168.xx.xxx:xxx;DatabaseName=db3
      username: xxx
      password: xxx
      driverClassName: com.microsoft.sqlserver.jdbc.SQLServerDriver
```

### 定义动态数据源

主要是继承AbstractRoutingDataSource，实现determineCurrentLookupKey方法。

```java
public class DynamicDataSource extends AbstractRoutingDataSource {
    /*存储所有数据源*/
    private Map<Object, Object> backupTargetDataSources;

    public Map<Object, Object> getBackupTargetDataSources() {
        return backupTargetDataSources;
    }
    /*defaultDataSource为默认数据源*/
    public DynamicDataSource(DataSource defaultDataSource, Map<Object, Object> targetDataSource) {
        backupTargetDataSources = targetDataSource;
        super.setDefaultTargetDataSource(defaultDataSource);
        super.setTargetDataSources(backupTargetDataSources);
        super.afterPropertiesSet();
    }
    public void addDataSource(String key, DataSource dataSource) {
        this.backupTargetDataSources.put(key, dataSource);
        super.setTargetDataSources(this.backupTargetDataSources);
        super.afterPropertiesSet();
    }
    /*返回当前线程的数据源的key*/
    @Override
    protected Object determineCurrentLookupKey() {
        return DynamicDataSourceContextHolder.getContextKey();
    }
}
```

### 定义数据源key线程变量持有

定义一个ThreadLocal静态变量，该变量持有了线程和线程的数据源key之间的关系。当我们要切换数据源时，首先要自己生成一个key，将这个key存入threadLocal线程变量中；同时还应该从DynamicDataSource对象中的backupTargetDataSources属性中获取到数据源对象， 然后将key和数据源对象再put到backupTargetDataSources中。 这样，spring就能根据determineCurrentLookupKey方法返回的key，从backupTargetDataSources中取出我们刚刚设置的数据源对象，进行getConnection等一系列操作了。

```java
public class DynamicDataSourceContextHolder {
    /**
     * 存储线程和数据源key的映射关系
     */
    private static final ThreadLocal<String> DATASOURCE_CONTEXT_KEY_HOLDER = new ThreadLocal<>();

    /***
     * 设置当前线程数据源key
     */
    public static void setContextKey(String key) {
        DATASOURCE_CONTEXT_KEY_HOLDER.set(key);
    }
    /***
     * 获取当前线程数据源key
     */
    public static String getContextKey() {
        String key = DATASOURCE_CONTEXT_KEY_HOLDER.get();
        return key == null ? DataSourceConstants.DS_KEY_MASTER : key;
    }
    /***
     * 删除当前线程数据源key
     */
    public static void removeContextKey() {
        DynamicDataSource dynamicDataSource = RequestHandleMethodRegistry.getContext().getBean(DynamicDataSource.class);
        String currentKey = DATASOURCE_CONTEXT_KEY_HOLDER.get();
        if (StringUtils.isNotBlank(currentKey) && !"master".equals(currentKey)) {
            dynamicDataSource.getBackupTargetDataSources().remove(currentKey);
        }
        DATASOURCE_CONTEXT_KEY_HOLDER.remove();
    }
}
```

### 多数据源自动配置类

这里通过读取yml配置文件中所有数据源的配置，自动为每个数据源创建datasource 对象并注册至bean工厂。同时将这些数据源对象，设置到AbstractRoutingDataSource中。

通过这种方式，后面如果需要添加或修改数据源，都无需新增或修改java配置类，只需去配置中心修改yml文件即可。

```java
@Configuration
@MapperScan(basePackages = "com.hosjoy.xxx.xxx.modules.xxx.mapper")
public class DynamicDataSourceConfig {
    @Autowired
    private BeanFactory beanFactory;
    @Autowired
    private DynamicDataSourceProperty dynamicDataSourceProperty;
    /**
     * 功能描述: <br>
     * 〈动态数据源bean 自动配置注册所有数据源〉
     *
     * @param
     * @return javax.sql.DataSource
     * @Author li.he
     * @Date 2020/6/4 16:47
     * @Modifier
     */
    @Bean
    @Primary
    public DataSource dynamicDataSource() {
        DefaultListableBeanFactory listableBeanFactory = (DefaultListableBeanFactory) beanFactory;
        /*获取yml所有数据源配置*/
        Map<String, Object> datasource = dynamicDataSourceProperty.getDatasource();
        Map<Object, Object> dataSourceMap = new HashMap<>(5);
        Optional.ofNullable(datasource).ifPresent(map -> {
            for (Map.Entry<String, Object> entry : map.entrySet()) {
                //创建数据源对象
                HikariDataSource dataSource = (HikariDataSource) DataSourceBuilder.create().build();
                String dataSourceId = entry.getKey();
                configeDataSource(entry, dataSource);
                /*bean工厂注册每个数据源bean*/
                listableBeanFactory.registerSingleton(dataSourceId, dataSource);
                dataSourceMap.put(dataSourceId, dataSource);
            }
        });
        //AbstractRoutingDataSource设置主从数据源
        return new DynamicDataSource(beanFactory.getBean("master", DataSource.class),          dataSourceMap);
    }

    private void configeDataSource(Map.Entry<String, Object> entry, HikariDataSource dataSource) {
        Map<String, Object> dataSourceConfig = (Map<String, Object>) entry.getValue();
        dataSource.setJdbcUrl(MapUtils.getString(dataSourceConfig, "jdbcUrl"));
        dataSource.setDriverClassName(MapUtils.getString(dataSourceConfig, "driverClassName"));
        dataSource.setUsername(MapUtils.getString(dataSourceConfig, "username"));
        dataSource.setPassword(MapUtils.getString(dataSourceConfig, "password"));
    }

}
```

### 数据源切换工具类

切换逻辑：

（1）生成当前线程数据源key

（2）根据业务条件，获取应切换的数据源ID；

（3）根据ID从数据源缓存池中获取数据源对象，并再次添加到backupTargetDataSources缓存池中；

（4）threadLocal设置当前线程对应的数据源key；

（5）在执行数据库操作前，spring会调用determineCurrentLookupKey方法获取key，然后根据key去数据源缓存池取出数据源，然后getConnection获取该数据源连接；

（6）使用该数据源执行数据库操作；

（7）释放缓存：threadLocal清理当前线程数据源信息、数据源缓存池清理当前线程数据源key和数据源对象，目的是防止内存泄漏。

```java
@Slf4j
@Component
public class DataSourceUtil {
    @Autowired
    private DataSourceConfiger dataSourceConfiger;
    
    /*根据业务条件切换数据源*/
    public void switchDataSource(String key, Predicate<? super Map<String, Object>> predicate) {
        try {
            //生成当前线程数据源key
            String newDsKey = System.currentTimeMillis() + "";
            List<Map<String, Object>> configValues = dataSourceConfiger.getConfigValues(key);
            Map<String, Object> db = configValues.stream().filter(predicate)
                    .findFirst().get();
            String id = MapUtils.getString(db, "id");
            //根据ID从数据源缓存池中获取数据源对象，并再次添加到backupTargetDataSources
            addDataSource(newDsKey, id);
            //设置当前线程对应的数据源key
            DynamicDataSourceContextHolder.setContextKey(newDsKey);
            log.info("当前线程数据源切换成功，当前数据源ID:{}", id);

        }
        catch (Exception e) {
            log.error("切换数据源失败，请检查数据源配置文件。key:{}, 条件:{}", key, predicate.toString());
            throw new ClientException("切换数据源失败，请检查数据源配置", e);
        }
    }
    
    /*将数据源添加至多数据源缓存池中*/
    public static void addDataSource(String key, String dataSourceId) {
        DynamicDataSource dynamicDataSource = RequestHandleMethodRegistry.getContext().getBean(DynamicDataSource.class);
        DataSource dataSource = (DataSource) dynamicDataSource.getBackupTargetDataSources().get(dataSourceId);
        dynamicDataSource.addDataSource(key, dataSource);
    }
}

```

### 使用

```java
public void doExecute(ReqTestParams reqTestParams){
    //构造条件
    Predicate<? super Map<String, Object>> predicate =.........;
    //切换数据源
    dataSourceUtil.switchDataSource("testKey", predicate);
    //数据库操作
    mapper.testQuery();
    //清理缓存，避免内存泄漏
    DynamicDataSourceContextHolder.removeContextKey();
}
```

每次数据源使用后，都要调用removeContextKey方法清理缓存，避免内存泄漏，这里可以考虑用AOP拦截特定方法，利用后置通知为执行方法代理执行缓存清理工作。

```java
@Aspect
@Component
@Slf4j
public class RequestHandleMethodAspect {
    @After("xxxxxxxxxxxxxxExecution表达式xxxxxxxxxxxxxxxxxx")
    public void afterRunning(JoinPoint joinPoint){
        String name = joinPoint.getSignature().toString();
        long id = Thread.currentThread().getId();
        log.info("方法执行完毕，开始清空当前线程数据源，线程id:{},代理方法:{}",id,name);
        DynamicDataSourceContextHolder.removeContextKey();
        log.info("当前线程数据源清空完毕,已返回至默认数据源:{}",id);
    }
}
```

## 特点分析

（1）参数化切换数据源方式，出发点和分包方式不一样，适合于在运行时才能确定用哪个数据源。

（2）需要手动执行切换数据源操作；

（3）无需分包，mapper和xml路径自由定义；

（4）增加数据源，无需修改java配置类，只需修改数据源配置文件即可。

# 注解方式

## 思想

该方式利用注解+AOP思想，为需要切换数据源的方法标记自定义注解，注解属性指定数据源ID，然后利用AOP切面拦截注解标记的方法，在方法执行前，切换至相应数据源；在方法执行结束后，切换至默认数据源。

需要注意的是，自定义切面的优先级需要高于@Transactional注解对应切面的优先级。

否则，在自定义注解和@Transactional同时使用时，@Transactional切面会优先执行，切面在调用getConnection方法时，会去调用AbstractRoutingDataSource.determineCurrentLookupKey方法，此时获取到的是默认数据源master。这时@UsingDataSource对应的切面即使再设置当前线程的数据源key，后面也不会再去调用determineCurrentLookupKey方法来切换数据源了。

## 设计思路

### 数据源注册

同上。

### 数据源切换

利用切面，拦截所有@UsingDataSource注解标记的方法，根据dataSourceId属性，在方法执行前，切换至相应数据源；在方法执行结束后，清理缓存并切换至默认数据源。

![image-20200717094314674](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d4f16c4d102f4602a5a6e74c33377c32~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

## 代码实现

### 数据源配置文件

同上。

### 定义动态数据源

同上。

### 定义数据源key线程变量持有

同上。

### 多数据源自动配置类

同上。

### 数据源切换工具类

切换逻辑：

（1）生成当前线程数据源key

（3）根据ID从数据源缓存池中获取数据源对象，并再次添加到backupTargetDataSources缓存池中；

（4）threadLocal设置当前线程对应的数据源key；

（5）在执行数据库操作前，spring会调用determineCurrentLookupKey方法获取key，然后根据key去数据源缓存池取出数据源，然后getConnection获取该数据源连接；

（6）使用该数据源执行数据库操作；

（7）释放缓存：threadLocal清理当前线程数据源信息、数据源缓存池清理当前线程数据源key和数据源对象。

```java
public static void switchDataSource(String dataSourceId) {
    if (StringUtils.isBlank(dataSourceId)) {
        throw new ClientException("切换数据源失败，数据源ID不能为空");
    }
    try {
        String threadDataSourceKey = UUID.randomUUID().toString();
        DataSourceUtil.addDataSource(threadDataSourceKey, dataSourceId);
        DynamicDataSourceContextHolder.setContextKey(threadDataSourceKey);
    }
    catch (Exception e) {
        log.error("切换数据源失败，未找到指定的数据源，请确保所指定的数据源ID已在配置文件中配置。dataSourceId:{}", dataSourceId);
        throw new ClientException("切换数据源失败，未找到指定的数据源，请确保所指定的数据源ID已在配置文件中配置。dataSourceId:" + dataSourceId, e);
    }
}
```

### 自定义注解

自定义注解标记当前方法所使用的数据源，默认为主数据源。

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface UsingDataSource {

    String dataSourceId() default "master";
}
```

### 切面

主要是定义前置通知和后置通知，拦截UsingDataSource注解标记的方法，方法执行前切换数据源，方法执行后清理数据源缓存。

需要标记切面的优先级比@Transaction注解对应切面的优先级要高。否则，在自定义注解和@Transactional同时使用时，@Transactional切面会优先执行，切面在调用getConnection方法时，会去调用AbstractRoutingDataSource.determineCurrentLookupKey方法，此时获取到的是默认数据源master。这时@UsingDataSource对应的切面即使再设置当前线程的数据源key，后面也不会再去调用determineCurrentLookupKey方法来切换数据源了。

```java
@Aspect
@Component
@Slf4j
@Order(value = 1)
public class DynamicDataSourceAspect {

    //拦截UsingDataSource注解标记的方法，方法执行前切换数据源
    @Before(value = "@annotation(usingDataSource)")
    public void before(JoinPoint joinPoint, UsingDataSource usingDataSource) {
        String dataSourceId = usingDataSource.dataSourceId();
        log.info("执行目标方法前开始切换数据源，目标方法:{}, dataSourceId:{}", joinPoint.getSignature().toString(), dataSourceId);
        try {
            DataSourceUtil.switchDataSource(dataSourceId);
        }
        catch (Exception e) {
            log.error("切换数据源失败！数据源可能未配置或不可用,数据源ID:{}", dataSourceId, e);
            throw new ClientException("切换数据源失败！数据源可能未配置或不可用,数据源ID:" + dataSourceId, e);
        }
        log.info("目标方法:{} , 已切换至数据源:{}", joinPoint.getSignature().toString(), dataSourceId);
    }

    //拦截UsingDataSource注解标记的方法，方法执行后清理数据源，防止内存泄漏
    @After(value = "@annotation(com.hosjoy.hbp.dts.common.annotation.UsingDataSource)")
    public void after(JoinPoint joinPoint) {
        log.info("目标方法执行完毕，执行清理，切换至默认数据源，目标方法:{}", joinPoint.getSignature().toString());
        try {
            DynamicDataSourceContextHolder.removeContextKey();
        }
        catch (Exception e) {
            log.error("清理数据源失败", e);
            throw new ClientException("清理数据源失败", e);
        }
        log.info("目标方法:{} , 数据源清理完毕，已返回默认数据源", joinPoint.getSignature().toString());
    }
}
```

### 使用

```java
@UsingDataSource(dataSourceId = "slave1")
@Transactional
public void test(){
    AddressPo po = new AddressPo();
    po.setMemberCode("asldgjlk");
    po.setName("lihe");
    po.setPhone("13544986666");
    po.setProvince("asdgjwlkgj");
    addressMapper.insert(po);
    int i = 1 / 0;
}
```

# 动态添加方式（非常用）

## 业务场景描述

这种业务场景不是很常见，但肯定是有人遇到过的。

项目里面只配置了1个默认的数据源，而具体运行时需要动态的添加新的数据源，非已配置好的静态的多数据源。例如需要去服务器实时读取数据源配置信息（非配置在本地），然后再执行数据库操作。

这种业务场景，以上3种方式就都不适用了，因为上述的数据源都是提前在yml文件配制好的。

## 实现思路

除了第6步外，利用之前写好的代码就可以实现。

思路是：

（1）创建新数据源；

（2）DynamicDataSource注册新数据源；

（3）切换：设置当前线程数据源key；添加临时数据源；

（4）数据库操作（必须在另一个service实现，否则无法控制事务）；

（5）清理当前线程数据源key、清理临时数据源；

（6）清理刚刚注册的数据源；

（7）此时已返回至默认数据源。

## 代码

代码写的比较粗陋，但是模板大概就是这样子，主要想表达实现的方式。

Service A:

```java
public String testUsingNewDataSource(){
        DynamicDataSource dynamicDataSource = RequestHandleMethodRegistry.getContext().getBean("dynamicDataSource", DynamicDataSource.class);
        try {
            //模拟从服务器读取数据源信息
            //..........................
            //....................
            
            //创建新数据源
            HikariDataSource dataSource = (HikariDataSource)                   DataSourceBuilder.create().build();
            dataSource.setJdbcUrl("jdbc:mysql://192.168.xxx.xxx:xxxx/xxxxx?......");
            dataSource.setDriverClassName("com.mysql.cj.jdbc.Driver");
            dataSource.setUsername("xxx");
            dataSource.setPassword("xxx");
            
            //DynamicDataSource注册新数据源
            dynamicDataSource.addDataSource("test_ds_id", dataSource);

            //设置当前线程数据源key、添加临时数据源
            DataSourceUtil.switchDataSource("test_ds_id");

            //数据库操作（必须在另一个service实现，否则无法控制事务）
            serviceB.testInsert();
        }
        finally {
            //清理当前线程数据源key
            DynamicDataSourceContextHolder.removeContextKey();

            //清理刚刚注册的数据源
            dynamicDataSource.removeDataSource("test_ds_id");

        }
        return "aa";
    }
```

Service B:

```java
@Transactional(rollbackFor = Exception.class)
    public void testInsert() {
        AddressPo po = new AddressPo();
        po.setMemberCode("555555555");
        po.setName("李郃");
        po.setPhone("16651694996");
        po.setProvince("江苏省");
        po.setCity("南京市");
        po.setArea("浦口区");
        po.setAddress("南京市浦口区宁六路219号");
        po.setDef(false);
        po.setCreateBy("23958");
        addressMapper.insert(po);
        //测试事务回滚
        int i = 1 / 0;
    }
```

DynamicDataSource: 增加removeDataSource方法， 清理注册的新数据源。

```java
public class DynamicDataSource extends AbstractRoutingDataSource {
    
            .................
            .................
            .................
    public void removeDataSource(String key){
        this.backupTargetDataSources.remove(key);
        super.setTargetDataSources(this.backupTargetDataSources);
        super.afterPropertiesSet();
    }
    
            .................
            .................
            .................
}
```

# 四种方式对比

|  | 分包方式 | 参数化切换 | 注解方式 | 动态添加方式 |
| --- | --- | --- | --- | --- |
| 适用场景 | 编码时便知道用哪个数据源 | 运行时才能确定用哪个数据源 | 编码时便知道用哪个数据源 | 运行时动态添加新数据源 |
| 切换模式 | 自动 | 手动 | 自动 | 手动 |
| mapper路径 | 需要分包 | 无要求 | 无要求 | 无要求 |
| 增加数据源是否需要修改配置类 | 需要 | 不需要 | 不需要 | \\ |
| 实现复杂度 | 简单 | 复杂 | 复杂 | 复杂 |

# 事务问题

使用上述数据源配置方式，可实现单个数据源事务控制。

例如在一个service方法中，需要操作多个数据源执行CUD时，是可以实现单个数据源事务控制的。方式如下，分别将需要事务控制的方法单独抽取到另一个service，可实现单个事务方法的事务控制。

ServiceA:

```java
public void updateMuilty(){
     serviceB.updateDb1();
     serviceB.updateDb2();
}
```

ServiceB:

```java
@UsingDataSource(dataSourceId = "slave1")
@Transactional
public void updateDb1(){
    //业务逻辑......
}

@UsingDataSource(dataSourceId = "slave2")
@Transactional
public void updateDb2(){
    //业务逻辑......
}
```

但是在同一个方法里控制多个数据源的事务就不是这么简单了，这就属于分布式事务的范围，可以考虑使用atomikos开源项目实现JTA分布式事务处理或者阿里的Fescar框架。

由于涉及到分布式事务控制，实现比较复杂，这里只是引出这个问题，后面抽时间把这块补上来。

# 参考文章

1.[www.liaoxuefeng.com/article/118…](https://www.liaoxuefeng.com/article/1182502273240832 "https://www.liaoxuefeng.com/article/1182502273240832") Spring主从数据库的配置和动态数据源切换原理

2.[blog.csdn.net/hekf2010/ar…](https://blog.csdn.net/hekf2010/article/details/81155778 "https://blog.csdn.net/hekf2010/article/details/81155778") Springcloud 多数库 多数据源整合,查询动态切换数据库

3.[blog.csdn.net/tuesdayma/a…](https://blog.csdn.net/tuesdayma/article/details/81081666 "https://blog.csdn.net/tuesdayma/article/details/81081666") springboot-mybatis多数据源的两种整合方法

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ca39685ef46d4524806019f264ffcf8b~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)