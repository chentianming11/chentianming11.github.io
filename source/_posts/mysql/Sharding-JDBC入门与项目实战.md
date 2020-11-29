---
title: 数据量大了一定要分表，分库分表组件Sharding-JDBC入门与项目实战
tags:
  - sharing-jdbc
categories:
  - 数据库
abbrlink: 2363983740
date: 2020-06-06 14:36:43
---

最近项目中不少表的数据量越来越大，并且导致了一些数据库的性能问题。因此想借助一些分库分表的中间件，实现自动化分库分表实现。调研下来，发现`Sharding-JDBC`目前成熟度最高并且应用最广的`Java分库分表的客户端组件`。本文主要介绍一些Sharding-JDBC核心概念以及生产环境下的实战指南，旨在帮助组内成员快速了解Sharding-JDBC并且能够快速将其使用起来。[Sharding-JDBC官方文档](https://shardingsphere.apache.org/document/current/cn/overview/)
<!--more-->

## 核心概念

在使用`Sharding-JDBC`之前，一定是先理解清楚下面几个核心概念。

### 逻辑表

水平拆分的数据库（表）的相同逻辑和数据结构表的总称。例：订单数据根据主键尾数拆分为10张表，分别是`t_order_0`到`t_order_9`，他们的逻辑表名为`t_order`。

### 真实表

在分片的数据库中真实存在的物理表。即上个示例中的`t_order_0`到`t_order_9`。

### 数据节点

数据分片的最小单元。由数据源名称和数据表组成，例：`ds_0.t_order_0`。

### 绑定表

指分片规则一致的主表和子表。例如：`t_order`表和`t_order_item`表，均按照`order_id`分片，则此两张表互为绑定表关系。绑定表之间的多表关联查询不会出现笛卡尔积关联，关联查询效率将大大提升。举例说明,如果SQL为：

```sql
SELECT i.* FROM t_order o JOIN t_order_item i ON o.order_id=i.order_id WHERE o.order_id in (10, 11);
```

假设`t_order`和`t_order_item`对应的真实表各有2个，那么真实表就有`t_order_0`、`t_order_1`、`t_order_item_0`、`t_order_item_1`。在不配置绑定表关系时，假设分片键`order_id`将数值10路由至第0片，将数值11路由至第1片，那么路由后的SQL应该为4条，它们呈现为笛卡尔积：

```sql
SELECT i.* FROM t_order_0 o JOIN t_order_item_0 i ON o.order_id=i.order_id WHERE o.order_id in (10, 11);
SELECT i.* FROM t_order_0 o JOIN t_order_item_1 i ON o.order_id=i.order_id WHERE o.order_id in (10, 11);
SELECT i.* FROM t_order_1 o JOIN t_order_item_0 i ON o.order_id=i.order_id WHERE o.order_id in (10, 11);
SELECT i.* FROM t_order_1 o JOIN t_order_item_1 i ON o.order_id=i.order_id WHERE o.order_id in (10, 11);
```

在配置绑定表关系后，路由的SQL应该为2条：

```sql
SELECT i.* FROM t_order_0 o JOIN t_order_item_0 i ON o.order_id=i.order_id WHERE o.order_id in (10, 11);
SELECT i.* FROM t_order_1 o JOIN t_order_item_1 i ON o.order_id=i.order_id WHERE o.order_id in (10, 11);
```

### 广播表

指所有的分片数据源中都存在的表，表结构和表中的数据在每个数据库中均完全一致。适用于数据量不大且需要与海量数据的表进行关联查询的场景，例如：字典表。

## 数据分片

### 分片键

用于分片的数据库字段，是将数据库(表)水平拆分的关键字段。例：将订单表中的订单主键的尾数取模分片，则订单主键为分片字段。 SQL 中如果无分片字段，将执行全路由，性能较差。 除了对单分片字段的支持，Sharding-JDBC 也支持根据多个字段进行分片。

### 分片算法

通过分片算法将数据分片，支持通过`=、>=、<=、>、<、BETWEEN和IN`分片。 分片算法需要应用方开发者自行实现，可实现的灵活度非常高。

目前提供4种分片算法。 由于分片算法和业务实现紧密相关，因此并未提供内置分片算法，而是通过分片策略将各种场景提炼出来，提供更高层级的抽象，并提供接口让应用开发者自行实现分片算法。

#### 精确分片算法

对应 `PreciseShardingAlgorithm`，**用于处理使用单一键作为分片键的 = 与 IN 进行分片的场景**。需要配合 `StandardShardingStrategy` 使用。

#### 范围分片算法

对应 `RangeShardingAlgorithm`，**用于处理使用单一键作为分片键的 BETWEEN AND、>、<、>=、<=进行分片的场景**。需要配合 StandardShardingStrategy 使用。

#### 复合分片算法

对应 `ComplexKeysShardingAlgorithm`，用于处理使用多键作为分片键进行分片的场景，包含多个分片键的逻辑较复杂，需要应用开发者自行处理其中的复杂度。需要配合 `ComplexShardingStrategy` 使用。

#### Hint分片算法

对应 `HintShardingAlgorithm`，**用于处理通过Hint指定分片值而非从SQL中提取分片值的场景**。需要配合 `HintShardingStrategy` 使用。

### 分片策略

包含分片键和分片算法，由于分片算法的独立性，将其独立抽离。真正可用于分片操作的是分片键 + 分片算法，也就是分片策略。目前提供 5 种分片策略。

### 标准分片策略

对应 `StandardShardingStrategy`。提供对 SQ L语句中的 `=, >, <, >=, <=, IN 和 BETWEEN AND` 的分片操作支持。 `StandardShardingStrategy` 只支持单分片键，提供 `PreciseShardingAlgorithm` 和 `RangeShardingAlgorithm` 两个分片算法。 **`PreciseShardingAlgorithm` 是必选的**，用于处理 = 和 IN 的分片。 `RangeShardingAlgorithm` 是可选的，用于处理 `BETWEEN AND, >, <, >=, <=`分片，如果不配置 RangeShardingAlgorithm，SQL 中的 BETWEEN AND 将按照全库路由处理。

### 复合分片策略

对应 `ComplexShardingStrategy`。复合分片策略。提供对 SQL 语句中的 `=, >, <, >=, <=, IN 和 BETWEEN AND` 的分片操作支持。 **`ComplexShardingStrategy` 支持多分片键**，由于多分片键之间的关系复杂，因此并未进行过多的封装，而是直接将分片键值组合以及分片操作符透传至分片算法，完全由应用开发者实现，提供最大的灵活度。

### 行表达式分片策略

对应 `InlineShardingStrategy`。使用 Groovy 的表达式，提供对 SQL 语句中的 `= 和 IN`的分片操作支持，只支持单分片键。 对于简单的分片算法，可以通过简单的配置使用，从而避免繁琐的Java代码开发，如: `t_user_$->{u_id % 8}` 表示 t_user 表根据 u_id 模 8，而分成 8 张表，表名称为 `t_user_0` 到 `t_user_7`。 **可以认为是精确分片算法的简易实现**

### Hint分片策略

对应 HintShardingStrategy。通过 Hint 指定分片值而非从 SQL 中提取分片值的方式进行分片的策略。

### 分布式主键

用于在分布式环境下，生成全局唯一的id。Sharding-JDBC 提供了内置的分布式主键生成器，例如 `UUID`、`SNOWFLAKE`。还抽离出分布式主键生成器的接口，方便用户自行实现自定义的自增主键生成器。**为了保证数据库性能，主键id还必须趋势递增，避免造成频繁的数据页面分裂。**

## 读写分离

提供一主多从的读写分离配置，可独立使用，也可配合分库分表使用。

- 同一线程且同一数据库连接内，如有写入操作，以后的读操作均从主库读取，用于保证数据一致性
- 基于Hint的强制主库路由。
- 主从模型中，事务中读写均用主库。

## 执行流程

Sharding-JDBC 的原理总结起来很简单: 核心由 `SQL解析 => 执行器优化 => SQL路由 => SQL改写 => SQL执行 => 结果归并`的流程组成。
![Sharding-JDBC执行流程](https://shardingsphere.apache.org/document/current/img/sharding/sharding_architecture_cn.png)

## 项目实战

spring-boot项目实战

### 引入依赖

```xml
<dependency>
    <groupId>org.apache.shardingsphere</groupId>
    <artifactId>sharding-jdbc-spring-boot-starter</artifactId>
    <version>4.0.1</version>
</dependency>
```

### 数据源配置

如果使用`sharding-jdbc-spring-boot-starter`, 并且数据源以及数据分片都使用shardingsphere进行配置，对应的数据源会自动创建并注入到spring容器中。

```properties
spring.shardingsphere.datasource.names=ds0,ds1

spring.shardingsphere.datasource.ds0.type=org.apache.commons.dbcp.BasicDataSource
spring.shardingsphere.datasource.ds0.driver-class-name=com.mysql.jdbc.Driver
spring.shardingsphere.datasource.ds0.url=jdbc:mysql://localhost:3306/ds0
spring.shardingsphere.datasource.ds0.username=root
spring.shardingsphere.datasource.ds0.password=

spring.shardingsphere.datasource.ds1.type=org.apache.commons.dbcp.BasicDataSource
spring.shardingsphere.datasource.ds1.driver-class-name=com.mysql.jdbc.Driver
spring.shardingsphere.datasource.ds1.url=jdbc:mysql://localhost:3306/ds1
spring.shardingsphere.datasource.ds1.username=root
spring.shardingsphere.datasource.ds1.password=

# 其它分片配置
```

但是在我们已有的项目中，数据源配置是单独的。**因此要禁用`sharding-jdbc-spring-boot-starter`里面的自动装配，而是参考源码自己重写数据源配置**。需要在启动类上加上`@SpringBootApplication(exclude = {org.apache.shardingsphere.shardingjdbc.spring.boot.SpringBootConfiguration.class})`来排除。然后自定义配置类来装配`DataSource`。

```java
@Configuration
@Slf4j
@EnableConfigurationProperties({
        SpringBootShardingRuleConfigurationProperties.class,
        SpringBootMasterSlaveRuleConfigurationProperties.class, SpringBootEncryptRuleConfigurationProperties.class, SpringBootPropertiesConfigurationProperties.class})
@AutoConfigureBefore(DataSourceConfiguration.class)
public class DataSourceConfig implements ApplicationContextAware {

    @Autowired
    private SpringBootShardingRuleConfigurationProperties shardingRule;

    @Autowired
    private SpringBootPropertiesConfigurationProperties props;

    private ApplicationContext applicationContext;

    @Bean("shardingDataSource")
    @Conditional(ShardingRuleCondition.class)
    public DataSource shardingDataSource() throws SQLException {
        // 获取其它方式配置的数据源
        Map<String, DruidDataSourceWrapper> beans = applicationContext.getBeansOfType(DruidDataSourceWrapper.class);
        Map<String, DataSource> dataSourceMap = new HashMap<>(4);
        beans.forEach(dataSourceMap::put);
        // 创建shardingDataSource
        return ShardingDataSourceFactory.createDataSource(dataSourceMap, new ShardingRuleConfigurationYamlSwapper().swap(shardingRule), props.getProps());
    }

    @Bean
    public SqlSessionFactory sqlSessionFactory() throws SQLException {
        SqlSessionFactoryBean sqlSessionFactoryBean = new SqlSessionFactoryBean();
        // 将shardingDataSource设置到SqlSessionFactory中
        sqlSessionFactoryBean.setDataSource(shardingDataSource());
        // 其它设置
        return sqlSessionFactoryBean.getObject();
    }
}
```

### 分布式id生成器配置

Sharding-JDBC提供了`UUID`、`SNOWFLAKE`生成器，还支持用户实现自定义id生成器。比如可以实现了type为`SEQ`的分布式id生成器，调用统一的`分布式id服务`获取id。

```java
@Data
public class SeqShardingKeyGenerator implements ShardingKeyGenerator {

    private Properties properties = new Properties();

    @Override
    public String getType() {
        return "SEQ";
    }

    @Override
    public synchronized Comparable<?> generateKey() {
       // 获取分布式id逻辑
    }
}
```

**由于扩展`ShardingKeyGenerator`是通过JDK的`serviceloader`的SPI机制实现的，因此还需要在`resources/META-INF/services`目录下配置`org.apache.shardingsphere.spi.keygen.ShardingKeyGenerator`文件。** 文件内容就是`SeqShardingKeyGenerator`类的全路径名。这样使用的时候，指定分布式主键生成器的type为`SEQ`就好了。

至此，Sharding-JDBC就整合进spring-boot项目中了，后面就可以进行数据分片相关的配置了。

### 数据分片实战

如果项目初期就能预估出表的数据量级，当然可以一开始就按照这个预估值进行分库分表处理。但是大多数情况下，我们一开始并不能准备预估出数量级。这时候通常的做法是：

1. 线上数据某张表查询性能开始下降，排查下来是因为数据量过大导致的。
2. 根据历史数据量预估出未来的数据量级，并结合具体业务场景确定分库分表策略。
3. 自动分库分表代码实现。

下面就以一个具体事例，阐述具体数据分片实战。比如有张表数据结构如下：

```sql
CREATE TABLE `hc_question_reply_record` (
  `id` bigint NOT NULL AUTO_INCREMENT COMMENT '自增ID',
  `reply_text` varchar(500) NOT NULL DEFAULT '' COMMENT '回复内容',
  `reply_wheel_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '回复时间',

  `ctime` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `mtime` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  PRIMARY KEY (`id`),
  INDEX `idx_reply_wheel_time` (`reply_wheel_time`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci
  COMMENT='回复明细记录';

```

#### 分片方案确定

先查询目前目标表月新增趋势：

```sql
SELECT count(*), date_format(ctime, '%Y-%m') AS `日期`
FROM hc_question_reply_record
GROUP BY date_format(ctime, '%Y-%m');
```

![月新增趋势](https://chentianming11.github.io/images/month-increse.png)

目前月新增在180w左右，预估未来达到300w(基本以2倍计算)以上。期望单表数据量不超过1000w，可使用`reply_wheel_time`作为分片键按季度归档。

#### 分片配置

```yaml
spring:
  # sharing-jdbc配置
  shardingsphere:
    # 数据源名称
    datasource:
      names: defaultDataSource,slaveDataSource
    sharding:
      # 主从节点配置
      master-slave-rules:
        defaultDataSource:
          # maser数据源
          master-data-source-name: defaultDataSource
          # slave数据源
          slave-data-source-names: slaveDataSource
      tables:
        # hc_question_reply_record 分库分表配置
        hc_question_reply_record:
          # 真实数据节点  hc_question_reply_record_2020_q1
          actual-data-nodes: defaultDataSource.hc_question_reply_record_$->{2020..2025}_q$->{1..4}
          # 表分片策略
          table-strategy:
            standard:
              # 分片键
              sharding-column: reply_wheel_time
              # 精确分片算法 全路径名
              preciseAlgorithmClassName: com.xx.QuestionRecordPreciseShardingAlgorithm
              # 范围分片算法，用于BETWEEN，可选。。该类需实现RangeShardingAlgorithm接口并提供无参数的构造器
              rangeAlgorithmClassName: com.xx.QuestionRecordRangeShardingAlgorithm

      # 默认分布式id生成器
      default-key-generator:
        type: SEQ
        column: id
```

#### 分片算法实现

- 精确分片算法：`QuestionRecordPreciseShardingAlgorithm`

  ```java
  public class QuestionRecordPreciseShardingAlgorithm implements PreciseShardingAlgorithm<Date> {
    /**
     * Sharding.
     *
     * @param availableTargetNames available data sources or tables's names
     * @param shardingValue        sharding value
     * @return sharding result for data source or table's name
     */
    @Override
    public String doSharding(Collection<String> availableTargetNames, PreciseShardingValue<Date> shardingValue) {
        return ShardingUtils.quarterPreciseSharding(availableTargetNames, shardingValue);
    }
  }
  ```

- 范围分片算法：`QuestionRecordRangeShardingAlgorithm`

  ```java
  public class QuestionRecordRangeShardingAlgorithm implements RangeShardingAlgorithm<Date> {

    /**
     * Sharding.
     *
     * @param availableTargetNames available data sources or tables's names
     * @param shardingValue        sharding value
     * @return sharding results for data sources or tables's names
     */
    @Override
    public Collection<String> doSharding(Collection<String> availableTargetNames, RangeShardingValue<Date> shardingValue) {
        return ShardingUtils.quarterRangeSharding(availableTargetNames, shardingValue);
    }
  }
  ```

- 具体分片实现逻辑：`ShardingUtils`

  ```java
  @UtilityClass
  public class ShardingUtils {
      public static final String QUARTER_SHARDING_PATTERN = "%s_%d_q%d";

      /**
      * logicTableName_{year}_q{quarter}
      * 按季度范围分片
      * @param availableTargetNames 可用的真实表集合
      * @param shardingValue 分片值
      * @return
      */
      public Collection<String> quarterRangeSharding(Collection<String> availableTargetNames, RangeShardingValue<Date> shardingValue) {
          // 这里就是根据范围查询条件，筛选出匹配的真实表集合
      }

      /**
      * logicTableName_{year}_q{quarter}
      * 按季度精确分片
      * @param availableTargetNames 可用的真实表集合
      * @param shardingValue 分片值
      * @return
      */
      public static String quarterPreciseSharding(Collection<String> availableTargetNames, PreciseShardingValue<Date> shardingValue) {
          // 这里就是根据等值查询条件，计算出匹配的真实表
      }
  }

  ```

到这里，针对`hc_question_reply_record`表，使用`reply_wheel_time`作为分片键，按照季度分片的处理就完成了。还有一点要注意的就是，**分库分表之后，查询的时候最好都带上分片键作为查询条件**，否则就会使用全库路由，性能很低。 还有就是`Sharing-JDBC`对`mysql`的全文索引支持的不是很好，项目有使用到的地方也要注意一下。总结来说整个过程还是比较简单的，后续碰到其它业务场景，相信大家按照这个思路肯定都能解决的。

> 原创不易，觉得文章写得不错的小伙伴，点个赞👍 鼓励一下吧~
