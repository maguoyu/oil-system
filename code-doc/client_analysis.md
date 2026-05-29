# Client 调度配置模块（多租户）代码分析报告

> 分析日期：2026-02-04
> 项目版本：5.0.11.0

---

## 1. 项目概述

### 1.1 项目定位

Client 是**机场调度配置管理模块**，为 AGDIS（机场调度信息系统）提供多租户的客户端配置服务。核心功能包括：

| 功能领域 | 说明 |
|----------|------|
| 过滤配置 | 航班列表过滤规则管理 |
| 字段管理 | 调度界面字段配置 |
| 分组管理 | 用户分组及资源分配 |
| 颜色配置 | 航班颜色主题配置 |
| 区域配置 | 机位区域划分配置 |
| 席位配置 | 调度席位域名配置 |
| 排序配置 | 航班/任务排序规则 |
| 闹钟配置 | 航班提醒闹钟管理 |
| 备注模板 | 任务备注模板管理 |
| 车辆预约 | 预约车辆信息管理 |
| 除冰管理 | 除冰池航班管理 |

### 1.2 项目结构

```
client/                          # 父模块
├── pom.xml                     # 父POM（版本：5.0.11.0）
├── client-rpc/                 # RPC接口模块
│   ├── pom.xml                # RPC POM（版本：5.0.8.7）
│   └── src/main/java/com/ias/agdis/client/
│       ├── ClientService.java # 主服务接口
│       ├── ClientServiceDubbo.java
│       ├── ClientProvider.java
│       ├── ClientException.java
│       ├── FlightClock.java
│       ├── FlightClockRule.java
│       ├── UserConfig.java
│       ├── AgdisFilterConfig.java
│       ├── AgdisFields.java
│       ├── AgdisColorConfigVo.java
│       ├── AgdisGroupVo.java
│       ├── AgdisGroupResourceVo.java
│       ├── AgdisSeatDomainConfigVo.java
│       ├── AgdisPlaceRegionConfigVo.java
│       ├── AgdisFlightSortConfigVo.java
│       ├── AgdisFlightGroupSortConfigVo.java
│       ├── AgdisFlightClockConfigVo.java
│       ├── AgdisFlightClockRemindContentVo.java
│       ├── AgdisTaskMinuteVo.java
│       ├── AgdisChangeTaskStateVo.java
│       ├── AgdisAppointmentCarVo.java
│       ├── AgdisRemarkTemplateVo.java
│       └── AgdisSortConfigVo.java
│
└── client-impl/               # 实现模块
    ├── pom.xml
    ├── Dockerfile
    └── src/main/
        ├── java/com/ias/agdis/client/
        │   ├── service/
        │   │   ├── ClientServiceImpl.java    # 主服务实现
        │   │   ├── ClientServiceDubboImpl.java
        │   │   └── BeanMapper.java
        │   │
        │   ├── impl/persistence/            # 数据访问层
        │   │   ├── ManagerSupport.java
        │   │   ├── UserConfigManager.java
        │   │   ├── FlightClockManager.java
        │   │   ├── FlightClockRuleManager.java
        │   │   ├── AgdisFilterConfigManager.java
        │   │   ├── AgdisFieldsManager.java
        │   │   ├── AgdisColorConfigManager.java
        │   │   ├── AgdisPlaceRegionConfigManager.java
        │   │   ├── AgdisSeatDomainConfigManager.java
        │   │   ├── AgdisFlightSortConfigManager.java
        │   │   ├── AgdisFlightGroupSortConfigManager.java
        │   │   ├── AgdisTaskGroupSortConfigManager.java
        │   │   ├── AgdisSortConfigManager.java
        │   │   ├── AgdisFlightClockConfigManager.java
        │   │   ├── AgdisFlightClockRuleManager.java
        │   │   ├── AgdisFlightClockRemindContentManager.java
        │   │   ├── AgdisAppointmentCarManager.java
        │   │   ├── AgdisTaskMinuteManager.java
        │   │   ├── AgdisDeicingManager.java
        │   │   ├── AgdisChangeTaskStateManager.java
        │   │   ├── AgdisTaskRemarkTemplateConfigManager.java
        │   │   ├── AgdisRemarkTemplateConfigManager.java
        │   │   ├── AgdisGroupManager.java
        │   │   ├── AgdisFilterConfigEntity.java
        │   │   ├── AgdisFieldsEntity.java
        │   │   └── AgdisDeicingManager.java
        │   │
        │   └── impl/dbmigrate/
        │       ├── ModuleSpecification.java
        │       ├── DatabaseMigrator.java
        │       └── ClientModuleSpecification.java
        │
        ├── resources/
        │   ├── META-INF/spring/
        │   │   ├── applicationContext.xml         # Spring主配置
        │   │   ├── applicationContext_dubbo-provider.xml
        │   │   └── applicationContext_dubbo-consumer.xml
        │   │
        │   ├── config.properties                  # Dubbo配置
        │   ├── jdbc_mysql.properties              # 数据库配置
        │   ├── dbmigrate.properties               # 迁移配置
        │   ├── logback.xml                       # 日志配置
        │   ├── MyBatis-Configuration.xml         # MyBatis配置
        │   ├── dozer/data-mapping.xml             # Dozer映射
        │   │
        │   └── com/ias/agdis/mapper/             # MyBatis映射文件
        │       ├── UserConfigMapper.xml
        │       ├── FlightClockMapper.xml
        │       ├── AgdisFilterConfigMapper.xml
        │       ├── AgdisFieldsMapper.xml
        │       ├── AgdisColorConfig.xml
        │       ├── AgdisGroupMapper.xml
        │       ├── AgdisPlaceRegionConfig.xml
        │       ├── AgdisSeatDomainConfig.xml
        │       ├── AgdisFlightSortConfig.xml
        │       ├── AgdisFlightGroupSortConfig.xml
        │       ├── AgdisTaskGroupSortConfig.xml
        │       ├── AgdisSortConfig.xml
        │       ├── AgdisFlightClockConfig.xml
        │       ├── AgdisFlightClockRule.xml
        │       ├── AgdisFlightClockRemindContentMapper.xml
        │       ├── AgdisAppointmentCarMapper.xml
        │       ├── AgdisTaskMinute.xml
        │       ├── AgdisChangeTaskStateMapper.xml
        │       ├── AgdisTaskRemarkTemplateConfig.xml
        │       └── AgdisRemarkTemplateConfig.xml
        │
        │   └── dbmigrate/mysql/client/           # 数据库迁移脚本
        │       ├── V0_0_1__table.sql
        │       ├── V0_0_2__table.sql
        │       ├── V0_0_3__table.sql
        │       ├── V0_0_4__table.sql
        │       └── V0_0_5_table.sql
        │
        └── assembly/                             # 部署配置
            ├── assembly.xml
            ├── bin/
            │   ├── start.sh / start.bat
            │   ├── stop.sh
            │   ├── restart.sh
            │   └── run.sh
            └── conf/
                ├── app.properties
                ├── config.properties
                ├── jdbc_mysql.properties
                └── log4j.xml
```

---

## 2. 技术栈分析

### 2.1 核心技术

| 类别 | 技术 | 版本 | 说明 |
|------|------|------|------|
| 框架 | Spring | 4.0.8.RELEASE | 应用框架 |
| ORM | MyBatis | 3.2.3 | 数据持久化 |
| 连接池 | Apache Commons DBCP | 1.4 | 数据库连接池 |
| RPC | Dubbo | 2.5.3 | 分布式服务 |
| 注册中心 | Zookeeper | 3.4.6 | 服务注册发现 |
| 数据迁移 | Flyway | 3.0 | 数据库版本管理 |
| 对象映射 | Dozer | 5.4.0 | 对象转换 |
| 日志 | Logback | 1.2.3 | 日志框架 |
| JSON | JSON-LIB | 2.4 | JSON处理 |

### 2.2 内部依赖

| 依赖 | 版本 | 说明 |
|------|------|------|
| fw-common | 3.0.0.1 | 框架公共组件 |
| globalsequence-api | 5.0.0.1 | 序列号服务 |
| flight-services-facade | 5.0.0.34 | 航班服务 |
| flight-services-common | 5.0.0.27 | 航班公共组件 |

### 2.3 Maven配置

```xml
<!-- 父模块 -->
<groupId>com.ias.agdis</groupId>
<artifactId>client</artifactId>
<version>5.0.11.0</version>
<packaging>pom</packaging>

<!-- 子模块 -->
<modules>
    <module>client-rpc</module>    <!-- RPC接口 -->
    <module>client-impl</module>   <!-- 实现 -->
</modules>

<!-- Spring -->
<spring.version>4.0.8.RELEASE</spring.version>

<!-- Dubbo -->
<dubbo.version>2.5.3</dubbo.version>
```

---

## 3. 核心服务分析

### 3.1 服务接口结构

```java
public interface ClientService {
    
    // ========== 过滤配置 ==========
    List<AgdisFilterConfig> getAgdisFilterConfig(String tenantId, String param);
    
    // ========== 字段管理 ==========
    List<AgdisFields> getAgdisFields(String tenantId, List<String> column_types);
    String getAgdisFieldsColumnName(String tenantId, String code, String column_type);
    AgdisFields getAgdisFieldsColumn(String tenantId, String code, String column_type);
    
    // ========== 分组管理 ==========
    List<AgdisGroupVo> getGroupResource(String owner);
    void addGroupResource(String owner, List<AgdisGroupVo> groupResource);
    
    // ========== 用户配置 ==========
    int addUserConfig(UserConfig record);
    int addUserConfigMany(UserConfig record);
    List<UserConfig> selectUserConfigByType(String usid, String type);
    List<UserConfig> selectUserConfigByType(String type);
    
    // ========== 航班闹钟 ==========
    int insertFlightClock(FlightClock flightClock);
    int deleteFlightClock(String id);
    List<FlightClock> selectFlightClock(String usid, String status);
    int updateFlightClock(FlightClock record);
    
    // ========== 颜色配置 ==========
    List<AgdisColorConfigVo> getAgdisColorConfig(String tenantId);
    
    // ========== 区域配置 ==========
    List<AgdisPlaceRegionConfigVo> getAgdisPlaceRegionConfig(String tenantId);
    int deleteAgdisPlaceRegionConfig(String tenantId);
    int insertAgdisPlaceRegionConfig(List<AgdisPlaceRegionConfigVo> voList, String tenantId);
    
    // ========== 席位配置 ==========
    List<AgdisSeatDomainConfigVo> getAgdisSeatDomainConfig(String tenantId);
    Map<Integer, List<AgdisSeatDomainConfigVo>> getAgdisSeatDomainConfigMap(String tenantId);
    int deleteAgdisSeatDomainConfig(String tenantId);
    int insertAgdisSeatDomainConfig(List<AgdisSeatDomainConfigVo> voList, String tenantId);
    
    // ========== 排序配置 ==========
    List<AgdisFlightGroupSortConfigVo> getAgdisFlightGroupSortConfig(String tenantId);
    List<AgdisFlightGroupSortConfigVo> getAgdisTaskGroupSortConfig(String tenantId);
    List<AgdisFlightSortConfigVo> getAgdisFlightSortConfig(String tenantId);
    List<AgdisSortConfigVo> getAgdisSortConfig(String tenantId);
    
    // ========== 闹钟规则 ==========
    List<AgdisFlightClockConfigVo> getAgdisFlightClockConfig(String type, String tenantId);
    int insertFlightClockRule(FlightClockRule flightClockRule);
    int deleteFlightClockRule(String id);
    int updateFlightClockRule(FlightClockRule flightClockRule);
    List<FlightClockRule> getFlightClockRuleList(String tenantId);
    FlightClockRule getFlightClockRuleById(String id);
    
    // ========== 提醒内容 ==========
    List<AgdisFlightClockRemindContentVo> getClockAllRemindContent(String tenantId);
    List<AgdisFlightClockRemindContentVo> getClockDelayRemindContent(String tenantId);
    List<AgdisFlightClockRemindContentVo> getClockNotRemindContent(String userId, String tenantId);
    int updataClockDelayTime(String ruleId, String flseq, String nextAlarmTime, String tenantId);
    int savaClockRemindContent(AgdisFlightClockRemindContentVo vo);
    int deleteClockRemindContent(String ruleId, String flseq, String tenantId);
    int deleteClockRemindContentByFlop(String flop, String tenantId);
    
    // ========== 车辆预约 ==========
    List<AgdisAppointmentCarVo> getAllCarInfo(String tenantId);
    int updateCarInfo(int id, String carStatus);
    
    // ========== 任务配置 ==========
    List<AgdisTaskMinuteVo> getAllTaskMinute(String tenantId);
    String getDeicingPoolFlightKeys();
    List<AgdisChangeTaskStateVo> getChangeTaskStateConfig(String tenantId);
    
    // ========== 备注模板 ==========
    List<Map<String, String>> getTaskRemarkTemplate(String tenantId);
    List<String> getRemarkTemplate(String keyType, String keyValue, String tenantId);
    List<AgdisRemarkTemplateVo> getRemarkTemplates(String keyType, String tenantId);
    int modifyRemarkTemplate(AgdisRemarkTemplateVo vo);
    int addRemarkTemplate(AgdisRemarkTemplateVo vo);
    int deleteRemarkTemplate(String id);
    
    // ========== 长航线配置 ==========
    String getLongAirlineConfig(String tenantId);
}
```

### 3.2 服务实现架构

```java
@Service("clientService")
public class ClientServiceImpl implements ClientService {

    @Resource
    AgdisFilterConfigManager agdisFilterConfigManager;
    @Resource
    AgdisFieldsManager agdisFieldsManager;
    @Resource
    AgdisGroupManager agdisGroupManager;
    @Resource
    FlightClockManager flightClockManager;
    @Resource
    UserConfigManager userConfigManager;
    @Resource
    AgdisColorConfigManager agdisColorConfigManager;
    @Resource
    AgdisPlaceRegionConfigManager agdisPlaceRegionConfigManager;
    @Resource
    AgdisSeatDomainConfigManager agdisSeatDomainConfigManager;
    @Resource
    AgdisFlightGroupSortConfigManager agdisFlightGroupSortConfigManager;
    @Resource
    AgdisFlightSortConfigManager agdisFlightSortConfigManager;
    @Resource
    AgdisTaskGroupSortConfigManager agdisTaskGroupSortConfigManager;
    @Resource
    AgdisSortConfigManager agdisSortConfigManager;
    @Resource
    AgdisFlightClockConfigManager agdisFlightClockConfigManager;
    @Resource
    AgdisFlightClockRuleManager agdisFlightClockRuleManager;
    @Resource
    AgdisAppointmentCarManager agdisAppointmentCarManager;
    @Resource
    AgdisTaskMinuteManager agdisTaskMinuteManager;
    @Resource
    AgdisDeicingManager agdisDeicingManager;
    @Resource
    AgdisChangeTaskStateManager agdisChangeTaskStateManager;
    @Resource
    AgdisFlightClockRemindContentManager agdisFlightClockRemindContentManager;
    @Resource
    AgdisTaskRemarkTemplateConfigManager agdisTaskRemarkTemplateConfigManager;
    @Resource
    AgdisRemarkTemplateConfigManager agdisRemarkTemplateConfigManager;
    
    @Resource
    ISequenceService tableSequenceService;
}
```

---

## 4. 数据模型分析

### 4.1 核心实体

#### AgdisFilterConfig（过滤配置）

```java
public interface AgdisFilterConfig {
    String getId();
    void setId(String id);
    
    String getArrival_view();      // 到达视图
    String getDepart_view();      // 出发视图
    String getCode();             // 过滤代码
    String getName();             // 过滤名称
    String getOriginal_value();   // 原始值
    String getMapping_value();    // 映射值
    String getDefault_filter_F2();// 默认过滤F2
    int getOrderby();             // 排序
    int getEnable();              // 是否启用
    String getFilter_rule();      // 过滤规则
    String getFilter_desc();      // 过滤描述
    String getTenantId();         // 租户ID
}
```

#### AgdisFields（字段配置）

```java
public class AgdisFields {
    private String code;           // 字段代码
    private String column_name;    // 列名
    private String column_type;    // 列类型
    private String column_value;   // 列值
    private String description;    // 描述
    private int orderby;           // 排序
    private int enable;           // 启用状态
    private String tenantId;      // 租户ID
}
```

#### AgdisGroupVo（分组配置）

```java
public class AgdisGroupVo implements java.io.Serializable {
    private String grid;                          // 组ID
    private String grnm;                         // 组名称
    private String grac;                         // 激活组
    private String owner;                         // 持有人
    private List<AgdisGroupResourceVo> resources; // 资源列表
}
```

#### AgdisColorConfigVo（颜色配置）

```java
public class AgdisColorConfigVo {
    private Integer groupId;           // 组ID
    private String groupDescription;  // 组描述
    private String className;         // CSS类名
    private String classDescription;  // 类描述
    private Integer enable;           // 启用状态
    private Integer text;             // 文字颜色
    private Integer border;           // 边框颜色
    private Integer background;       // 背景颜色
    private String tenantId;          // 租户ID
    private Integer orderby;          // 排序
}
```

#### FlightClock（航班闹钟）

```java
public class FlightClock {
    private String id;              // 闹钟ID
    private String usid;           // 用户ID
    private String flseq;          // 航班序列
    private String alarmTime;      // 提醒时间
    private String status;         // 状态
    private String snoozeTime;     // 延迟时间
    private String createTime;     // 创建时间
}
```

#### FlightClockRule（闹钟规则）

```java
public class FlightClockRule {
    private String id;              // 规则ID
    private String ruleName;       // 规则名称
    private String type;           // 类型
    private String fieldName;      // 字段名称
    private String fieldValue;     // 字段值
    private int advanceTime;       // 提前时间
    private String tenantId;       // 租户ID
    private String enable;         // 是否启用
}
```

### 4.2 配置数据表

| 表名 | 说明 | 主要用途 |
|------|------|----------|
| agdis_filter_config | 过滤配置 | 航班列表过滤 |
| agdis_fields | 字段配置 | 界面字段管理 |
| agdis_group | 分组表 | 用户分组 |
| agdis_group_resource | 分组资源 | 资源分配 |
| agdis_color_config | 颜色配置 | 航班颜色 |
| agdis_place_region_config | 区域配置 | 机位区域 |
| agdis_seat_domain_config | 席位配置 | 调度席位 |
| agdis_flight_sort_config | 航班排序 | 排序规则 |
| agdis_flight_group_sort_config | 分组排序 | 分组航班排序 |
| agdis_task_group_sort_config | 任务排序 | 任务排序 |
| agdis_sort_config | 排序配置 | 通用排序 |
| agdis_flight_clock | 航班闹钟 | 闹钟管理 |
| agdis_flight_clock_rule | 闹钟规则 | 规则配置 |
| agdis_flight_clock_remind_content | 提醒内容 | 提醒详情 |
| agdis_appointment_car | 预约车辆 | 车辆管理 |
| agdis_task_minute | 任务分钟 | 时间配置 |
| agdis_change_task_state | 状态变更 | 任务状态 |
| agdis_task_remark_template | 任务备注 | 备注模板 |
| agdis_remark_template_config | 备注配置 | 备注管理 |
| user_config | 用户配置 | 通用配置 |

---

## 5. 多租户架构

### 5.1 租户隔离设计

```java
// 租户参数传递
public List<AgdisFilterConfig> getAgdisFilterConfig(String tenantId, String param) {
    return agdisFilterConfigManager.getAgdisFilterConfig(tenantId, param);
}

// 多租户查询
public List<AgdisColorConfigVo> getAgdisColorConfig(String tenantId) {
    return agdisColorConfigManager.getAgdisColorConfig(tenantId);
}

// 默认配置fallback
public List<UserConfig> selectUserConfigByType(String usid, String type) {
    if (usid == null || "".equalsIgnoreCase(usid)) {
        usid = "default";  // 默认租户
    }
    List<UserConfig> userConfigs = userConfigManager.getUserConfig(usid, type);
    
    if (userConfigs == null || userConfigs.isEmpty()) {
        usid = "default";  // 回退到默认配置
        userConfigs = userConfigManager.getUserConfig(usid, type);
    }
    return userConfigs;
}
```

### 5.2 序列号生成

```java
@Resource
ISequenceService tableSequenceService;

// 分组序列号
long grid = tableSequenceService.getSequence("AGDIS_GROUP");
ag.setGrid(String.valueOf(grid));

// 分组资源序列号
long grrid = tableSequenceService.getSequence("AGDIS_GROUP_RESOURCE");
agr.setGrrid(String.valueOf(grrid));
```

---

## 6. 数据库配置

### 6.1 数据源配置

```properties
# MySQL
jdbc.driverClassName=com.mysql.jdbc.Driver
jdbc.url=jdbc:mysql://172.16.11.15:8066/client
jdbc.username=root
jdbc.password=Airport@123

# 连接池配置
jdbc.initialSize=10
jdbc.maxActive=100
jdbc.maxIdle=30
jdbc.minIdle=10
jdbc.timeBetweenEvictionRunsMillis=86400
jdbc.testWhileIdle=true
jdbc.validationQuery=SELECT 1
```

### 6.2 Flyway迁移

```properties
# 数据库迁移配置
application.database.type=mysql

# 迁移开关
dbmigrate.enabled=false
dbmigrate.clean=false

client.dbmigrate.enabled=false
client.dbmigrate.initData=false
```

### 6.3 迁移脚本

| 版本 | 文件名 | 说明 |
|------|--------|------|
| V0_0_1 | __table.sql | 初始表结构 |
| V0_0_2 | __table.sql | 新增字段 |
| V0_0_3 | __table.sql | 结构调整 |
| V0_0_4 | __table.sql | 性能优化 |
| V0_0_5 | _table.sql | 功能扩展 |

---

## 7. Dubbo服务配置

### 7.1 服务发布

```xml
<!-- applicationContext_dubbo-provider.xml -->
<dubbo:application name="dubbo-service-provider-client"/>

<dubbo:registry protocol="zookeeper" address="${dubbo.address}"/>

<dubbo:provider delay="-1" timeout="10000" retries="0"/>

<dubbo:protocol name="dubbo" port="${dubbo.port}"/>

<dubbo:service interface="com.ias.agdis.client.ClientServiceDubbo"
               ref="clientServiceDubbo"/>
```

### 7.2 Dubbo配置

```properties
# dubbo.properties
dubbo.address=172.16.10.111:2181
dubbo.port=10881
```

### 7.3 服务架构

```
┌─────────────────────────────────────────────────────────────┐
│                     Client服务架构                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   ┌─────────────────────────────────────────────────────┐   │
│   │              Dubbo服务消费者 (各业务模块)             │   │
│   │   flightdispatchr | flightdealuir | 其他模块...      │   │
│   └────────────────────────────┬──────────────────────────┘   │
│                                │                              │
│                                ▼                              │
│   ┌─────────────────────────────────────────────────────┐   │
│   │           Zookeeper注册中心 (172.16.10.111:2181)    │   │
│   │                                                     │   │
│   │   /dubbo/com.ias.agdis.client.ClientServiceDubbo   │   │
│   │   providers: dubbo://172.16.11.15:10881/...        │   │
│   │                                                     │   │
│   └────────────────────────────┬──────────────────────────┘   │
│                                │                              │
│                                ▼                              │
│   ┌─────────────────────────────────────────────────────┐   │
│   │              Client服务提供者                        │   │
│   │   ┌─────────────────────────────────────────────┐   │   │
│   │   │        ClientServiceImpl                    │   │   │
│   │   │  ┌─────────┬─────────┬─────────┬───────┐  │   │   │
│   │   │  │Filter   │Fields   │Group    │Color  │  │   │   │
│   │   │  │Config   │Manager  │Manager  │Config │  │   │   │
│   │   │  └─────────┴─────────┴─────────┴───────┘  │   │   │
│   │   │  ┌─────────┬─────────┬─────────┬───────┐  │   │   │
│   │   │  │Seat     │Flight   │Appointment│Task  │  │   │   │
│   │   │  │Domain   │Clock    │Car      │Minute │  │   │   │
│   │   │  └─────────┴─────────┴─────────┴───────┘  │   │   │
│   │   └─────────────────────────────────────────────┘   │   │
│   │                          │                          │   │
│   │                          ▼                          │   │
│   │   ┌─────────────────────────────────────────────┐   │   │
│   │   │         MySQL (172.16.11.15:8066/client)  │   │   │
│   │   └─────────────────────────────────────────────┘   │   │
│   └─────────────────────────────────────────────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 8. 核心功能详解

### 8.1 过滤配置管理

```java
/**
 * 获取过滤配置
 * 参数说明：
 * - A: 查询到达视图
 * - D: 查询出发视图  
 * - AD: 查询全部
 */
public List<AgdisFilterConfig> getAgdisFilterConfig(String tenantId, String param) {
    if (param == null) {
        throw new ClientException("查询参数不能为空。");
    }
    
    String paramUpper = param.toUpperCase();
    List<String> range = Arrays.asList("A", "D", "AD");
    
    if (!range.contains(paramUpper)) {
        throw new ClientException("参数取值范围：A,D,AD. 输入值：" + param);
    }
    
    return agdisFilterConfigManager.getAgdisFilterConfig(tenantId, paramUpper);
}
```

### 8.2 分组资源管理

```java
/**
 * 添加分组和资源
 * 支持层级结构：Group -> Resources
 */
public void addGroupResource(String owner, List<AgdisGroupVo> groupResource) {
    if (owner == null) {
        throw new ClientException("AgdisGroup中owner不能为空");
    }
    
    // 删除旧的分组和资源
    agdisGroupManager.delAgdisGroup(owner);
    agdisGroupManager.delAgdisGroupResource(owner);
    
    for (AgdisGroupVo ag : groupResource) {
        // 生成组ID
        long grid = tableSequenceService.getSequence("AGDIS_GROUP");
        ag.setGrid(String.valueOf(grid));
        ag.setOwner(owner);
        agdisGroupManager.addAgdisGroup(ag);
        
        // 处理资源
        List<AgdisGroupResourceVo> list = ag.getResources();
        for (AgdisGroupResourceVo agr : list) {
            long grrid = tableSequenceService.getSequence("AGDIS_GROUP_RESOURCE");
            agr.setGrrid(String.valueOf(grrid));
            agr.setGrid(ag.getGrid());
            agr.setGrnm(ag.getGrnm());
            agr.setOwner(ag.getOwner());
            agdisGroupManager.addAgdisGroupResource(agr);
        }
    }
}
```

### 8.3 航班闹钟管理

```java
// 闹钟CRUD操作
public int insertFlightClock(FlightClock flightClock);
public int deleteFlightClock(String id);
public List<FlightClock> selectFlightClock(String usid, String status);
public int updateFlightClock(FlightClock record);

// 闹钟规则管理
public int insertFlightClockRule(FlightClockRule flightClockRule);
public int updateFlightClockRule(FlightClockRule flightClockRule);
public List<FlightClockRule> getFlightClockRuleList(String tenantId);

// 提醒内容管理
public List<AgdisFlightClockRemindContentVo> getClockAllRemindContent(String tenantId);
public List<AgdisFlightClockRemindContentVo> getClockDelayRemindContent(String tenantId);
public int updataClockDelayTime(String ruleId, String flseq, String nextAlarmTime, String tenantId);
```

### 8.4 备注模板管理

```java
/**
 * 获取备注模板
 * @param keyType 关键类型（如：regist_number机号）
 * @param keyValue 关键值
 * @param tenantId 租户ID
 */
public List<String> getRemarkTemplate(String keyType, String keyValue, String tenantId) {
    return agdisRemarkTemplateConfigManager.getRemarkTemplate(keyType, keyValue, tenantId);
}

/**
 * 添加备注模板
 */
public int addRemarkTemplate(AgdisRemarkTemplateVo agdisRemarkTemplate) {
    return agdisRemarkTemplateConfigManager.saveOrUpdateRemarkTemplate(agdisRemarkTemplate);
}

/**
 * 修改备注模板
 */
public int modifyRemarkTemplate(AgdisRemarkTemplateVo agdisRemarkTemplate) {
    return agdisRemarkTemplateConfigManager.saveOrUpdateRemarkTemplate(agdisRemarkTemplate);
}
```

### 8.5 席位域名配置

```java
/**
 * 获取席位配置（按域分组）
 */
public Map<Integer, List<AgdisSeatDomainConfigVo>> getAgdisSeatDomainConfigMap(String tenantId) {
    return agdisSeatDomainConfigManager.getAgdisSeatDomainConfigMap(tenantId);
}

/**
 * 批量更新席位配置
 */
public int insertAgdisSeatDomainConfig(List<AgdisSeatDomainConfigVo> voList, String tenantId) {
    return agdisSeatDomainConfigManager.insertAgdisSeatDomainConfig(voList, tenantId);
}
```

---

## 9. 依赖关系

### 9.1 服务依赖

```
Client (client-impl)
    ├── fw-common (3.0.0.1)
    │   └── 框架公共组件
    ├── globalsequence-api (5.0.0.1)
    │   └── 序列号服务
    ├── flight-services-facade (5.0.0.34)
    │   └── 航班服务接口
    ├── flight-services-common (5.0.0.27)
    │   └── 航班公共组件
    ├── MyBatis (3.2.3)
    ├── Dubbo (2.5.3)
    ├── Zookeeper (3.4.6)
    ├── Flyway (3.0)
    └── Dozer (5.4.0)
```

### 9.2 模块关系

```
                    ┌──────────────────┐
                    │   client-rpc     │ (接口定义)
                    │   5.0.8.7        │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │   client-impl    │ (实现)
                    │   5.0.11.0       │
                    └────────┬─────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
       ┌────────────┐ ┌────────────┐ ┌────────────┐
       │ flight-    │ │ flight-    │ │ 其他业务   │
       │ dispatchr  │ │ dealuir   │ │   模块     │
       └────────────┘ └────────────┘ └────────────┘
```

---

## 10. 部署配置

### 10.1 服务端口

```properties
dubbo.port=10881
```

### 10.2 部署拓扑

```
┌─────────────────────────────────────────────────────────────────┐
│                      部署拓扑架构                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │          应用服务器 (172.16.11.15)                      │  │
│   │  ┌──────────────────────────────────────────────────┐   │  │
│   │  │         client-impl.war                          │   │  │
│   │  │  ┌──────────────────────────────────────────┐   │   │  │
│   │  │  │  ClientServiceImpl                        │   │   │  │
│   │  │  │  ┌──────────────────────────────────────┐ │   │   │  │
│   │  │  │  │  17个 Manager 实现                   │ │   │   │  │
│   │  │  │  └──────────────────────────────────────┘ │   │   │  │
│   │  │  │  ┌──────────────────────────────────────┐ │   │   │  │
│   │  │  │  │  Flyway 数据库迁移                   │ │   │   │  │
│   │  │  │  └──────────────────────────────────────┘ │   │   │  │
│   │  │  └──────────────────────────────────────────┘   │   │  │
│   │  └──────────────────────────────────────────────────┘   │  │
│   └─────────────────────────────────────────────────────────┘  │
│                              │                                  │
│                              ▼                                  │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │                    Zookeeper                            │  │
│   │              (172.16.10.111:2181)                       │  │
│   │                                                         │  │
│   │   /dubbo/com.ias.agdis.client.ClientServiceDubbo       │  │
│   │   consumers: [各业务模块]                               │  │
│   │   providers: client-impl                               │  │
│   │                                                         │  │
│   └─────────────────────────────────────────────────────────┘  │
│                              │                                  │
│                              ▼                                  │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │                   MySQL                                 │  │
│   │            (172.16.11.15:8066/client)                   │  │
│   │                                                         │  │
│   │   20+ 数据表                                              │  │
│   │   SCHEMA_VERSION_CLIENT (Flyway迁移记录)                 │  │
│   │                                                         │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 11. 代码质量分析

### 11.1 优点

1. **清晰的分层架构**：接口 → 实现 → Manager → Mapper
2. **多租户支持**：完整的tenantId参数设计
3. **服务化架构**：基于Dubbo的RPC调用
4. **数据库迁移**：Flyway支持版本管理
5. **对象映射**：Dozer支持VO/Entity转换
6. **统一异常处理**：ClientException统一处理

### 11.2 问题与建议

#### 🔴 高优先级

| 问题 | 位置 | 建议 |
|------|------|------|
| 无服务降级 | Dubbo配置 | 增加Mock降级机制 |
| 无熔断保护 | Dubbo配置 | 集成Sentinel/Hystrix |
| 缺少缓存 | Manager层 | 引入Redis缓存 |
| 无接口限流 | Dubbo配置 | 增加限流配置 |

#### 🟡 中优先级

| 问题 | 位置 | 建议 |
|------|------|------|
| XML配置过多 | Spring | 迁移到Java Config |
| 无API文档 | - | 添加Swagger文档 |
| 无单元测试 | Test | 增加测试用例 |
| 硬编码 | 常量类 | 集中管理常量 |

#### 🟢 低优先级

| 问题 | 位置 | 建议 |
|------|------|------|
| Java版本较老 | pom.xml | 升级到1.8 |
| Spring版本 | 4.0.8 | 升级到4.3.x |
| 无监控指标 | - | 接入监控系统 |

---

## 12. 业务场景说明

### 12.1 调度界面配置

```
┌─────────────────────────────────────────────────────────────────┐
│                    AGDIS 调度客户端                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  席位配置 (AgdisSeatDomainConfig)                       │   │
│  │  ┌─────────┬─────────┬─────────┬─────────┐              │   │
│  │  │ 航班席1 │ 航班席2 │ 加油席1 │ 加油席2 │ ...         │   │
│  │  └─────────┴─────────┴─────────┴─────────┘              │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  颜色配置 (AgdisColorConfig)                           │   │
│  │  ┌─────────────────────────────────────────────────┐   │   │
│  │  │ 延误航班: 红底白字 | 正常: 绿底黑字 | ...       │   │   │
│  │  └─────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  过滤规则 (AgdisFilterConfig)                           │   │
│  │  ┌─────────────────────────────────────────────────┐   │   │
│  │  │ 已起飞: FLIGHT_STATUS=DEP | 延误: DELAY=1 ...  │   │   │
│  │  └─────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 12.2 航班提醒流程

```
┌─────────────────────────────────────────────────────────────────┐
│                    航班闹钟提醒流程                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   1. 配置闹钟规则 (FlightClockRule)                              │
│      ┌─────────────────────────────────────────────────────┐     │
│      │ 规则名称: "航班起飞前30分钟提醒"                    │     │
│      │ 类型: BEFORE_TAKE_OFF                              │     │
│      │ 字段: STA (计划到达时间)                           │     │
│      │ 提前时间: 30 (分钟)                                 │     │
│      └─────────────────────────────────────────────────────┘     │
│                              │                                   │
│                              ▼                                   │
│   2. 监控航班时间 (FlightClock)                                  │
│      ┌─────────────────────────────────────────────────────┐     │
│      │ 航班: MU5183                                        │     │
│      │ 提醒时间: 08:00 (STA=08:30 - 30分钟)                │     │
│      │ 状态: PENDING / ACKNOWLEDGED / SNOOZE              │     │
│      └─────────────────────────────────────────────────────┘     │
│                              │                                   │
│                              ▼                                   │
│   3. 发送提醒 (FlightClockRemindContent)                        │
│      ┌─────────────────────────────────────────────────────┐     │
│      │ 提醒内容: "MU5183 即将起飞，请确认准备"              │     │
│      │ 延迟时间: 用户可设置延迟提醒                          │     │
│      └─────────────────────────────────────────────────────┘     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 13. 总结

### 13.1 项目定位

Client 是**机场调度信息系统的配置管理中心**，为AGDIS各客户端提供统一的配置管理服务，支持多租户隔离。

### 13.2 核心能力

| 能力 | 实现 | 状态 |
|------|------|------|
| 多租户支持 | tenantId参数 | ✅ |
| 配置管理 | 20+ 配置表 | ✅ |
| RPC服务 | Dubbo | ✅ |
| 数据库迁移 | Flyway | ✅ |
| 对象映射 | Dozer | ✅ |
| 服务注册发现 | Zookeeper | ✅ |
| 缓存机制 | 未实现 | ❌ |
| 熔断降级 | 未实现 | ❌ |
| API文档 | 未实现 | ❌ |

### 13.3 技术债务

1. **技术栈老旧**：
   - Java 1.7 → 建议升级到1.8
   - Spring 4.0.8 → 建议升级到4.3.x
   - Dubbo 2.5.3 → 建议升级到2.7.x

2. **运维能力缺失**：
   - 无监控指标
   - 无日志追踪
   - 无服务降级

3. **开发效率**：
   - XML配置过多
   - 缺少单元测试
   - 缺少API文档

### 13.4 优化建议

1. **短期优化**：
   - 添加Redis缓存层
   - 增加服务降级机制
   - 完善异常处理

2. **中期重构**：
   - 技术栈升级
   - 引入监控体系
   - 添加API文档

3. **长期规划**：
   - 微服务化改造
   - 多级缓存架构
   - 全链路追踪

---

*报告生成时间：2026-02-04*
*分析工具：Cursor AI Code Analysis*
