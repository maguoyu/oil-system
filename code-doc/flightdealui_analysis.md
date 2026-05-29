# flightdealui 项目分析报告

## 1. 项目概述

### 1.1 基本信息
| 属性 | 值 |
|------|-----|
| 项目名称 | flightdealui-pvg |
| 版本 | 5.0.6.7-pvg |
| 类型 | Spring Boot Web 应用 |
| 端口 | 8889 |
| Java版本 | 1.7 |
| 打包方式 | jar (可执行) |

### 1.2 项目定位
`flightdealui` 是一个航班信息处理系统的用户界面服务，提供航班数据的增删改查、Excel导入导出、权限管理等功能，是整个航班数据处理系统的前端交互层。

### 1.3 核心功能模块
- **航班数据管理**: 航班信息的增删改查
- **Excel导入**: 支持批量导入航班数据
- **用户认证**: Spring Security 安全认证
- **权限控制**: 基于租户和角色的细粒度权限管理
- **消息推送**: ActiveMQ 消息队列集成
- **远程服务调用**: Dubbo 分布式服务调用

---

## 2. 技术架构

### 2.1 技术栈
```
Spring Boot 1.5.6.RELEASE
├── Web框架: Spring MVC + Spring Boot Web
├── 安全框架: Spring Security
├── 模板引擎: Thymeleaf
├── 数据访问: Spring Data JPA
├── 数据库: MySQL
├── ORM框架: Hibernate
├── 消息队列: ActiveMQ
├── 分布式框架: Dubbo + Zookeeper
└── 前端框架: Vue.js 2.x + Element UI
```

### 2.2 系统架构图
```
┌─────────────────────────────────────────────────────────────────┐
│                         客户端 (Browser)                         │
│                   Vue.js + Element UI + jQuery                   │
└─────────────────────────┬───────────────────────────────────────┘
                          │ HTTP/HTTPS
┌─────────────────────────▼───────────────────────────────────────┐
│                    flightdealui 服务 (8889)                      │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │                     Spring Boot 应用                         ││
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────────────┐    ││
│  │  │  控制器层   │ │  业务层     │ │     过滤器/安全      │    ││
│  │  │Controller  │ │  Service    │ │  Filter/Security    │    ││
│  │  └──────┬──────┘ └──────┬──────┘ └──────────┬──────────┘    ││
│  │         │               │                    │               ││
│  │  ┌──────▼──────┐ ┌──────▼──────┐ ┌──────────▼──────────┐    ││
│  │  │   视图层    │ │  数据访问层 │ │    基础数据加载      │    ││
│  │  │ Thymeleaf  │ │    JPA      │ │     BaseData        │    ││
│  │  └─────────────┘ └──────┬──────┘ └─────────────────────┘    ││
│  └─────────────────────────┼────────────────────────────────────┘│
└────────────────────────────┼────────────────────────────────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
   ┌────▼────┐         ┌─────▼─────┐        ┌─────▼─────┐
   │  MySQL  │         │ ActiveMQ  │        │ Zookeeper │
   │  8066   │         │   61616   │        │   2181    │
   └─────────┘         └───────────┘        └───────────┘
```

### 2.3 Maven 依赖分析

#### 核心依赖
| 依赖 | 版本 | 用途 |
|------|------|------|
| spring-boot-starter-web | 1.5.6 | Spring MVC Web 应用 |
| spring-boot-starter-security | 1.5.6 | 安全认证 |
| spring-boot-starter-data-jpa | 1.5.6 | JPA 数据访问 |
| spring-boot-starter-thymeleaf | 1.5.6 | 模板引擎 |
| dubbo | 2.5.3 | 分布式服务框架 |
| zookeeper | 3.4.6 | 服务注册发现 |
| activemq-client | 5.12.0 | 消息队列客户端 |
| fastjson | 1.2.6 | JSON 处理 |
| poi / poi-ooxml | 3.15/3.14 | Excel 文件处理 |
| commons-beanutils | 1.8.3 | Bean 操作工具 |
| commons-lang3 | 3.6 | Apache 工具库 |

#### 内部依赖
| 依赖 | 版本 | 来源 |
|------|------|------|
| flight-services-facade | 5.1.2.2-pvg | 航班服务接口 |
| flight-services-common | 5.1.2.0 | 通用工具类 |
| common (flight-tools) | 5.1.2.5-pvg | 通用模型 |
| model (commonparent) | 5.1.2.2-pvg | 数据模型 |

---

## 3. 项目结构

### 3.1 目录结构
```
flightdealui/
├── pom.xml                           # Maven配置
├── Dockerfile                        # Docker构建文件
├── src/main/
│   ├── java/com/ias/server/flight/deal/
│   │   ├── Api.java                  # 启动类
│   │   ├── controller/               # 控制器层
│   │   │   └── FlightDealController.java
│   │   ├── service/                  # 服务层
│   │   │   ├── FlightDataService.java
│   │   │   ├── FlightDataConfigService.java
│   │   │   ├── FlightImportConfigService.java
│   │   │   └── FlightDataStoreDealService.java
│   │   ├── service/impl/             # 服务实现
│   │   ├── entity/                   # 实体类
│   │   │   ├── FlightData.java
│   │   │   ├── FlightDataConfig.java
│   │   │   └── FlightImportConfig.java
│   │   ├── bean/                     # 业务对象
│   │   ├── jpa/                      # JPA Repository
│   │   ├── config/                   # 配置类
│   │   │   ├── WebSecurityConfig.java
│   │   │   ├── MyAuthenticationProvider.java
│   │   │   └── WebMvcConfig.java
│   │   ├── filter/                   # 过滤器
│   │   │   ├── RequestFilter.java
│   │   │   └── ResetRequestWrapper.java
│   │   ├── handler/                  # 处理器
│   │   │   └── LoginSuccessHandler.java
│   │   ├── job/                      # 定时任务/初始化
│   │   │   └── BaseData.java
│   │   └── util/                     # 工具类
│   ├── resources/
│   │   ├── application.properties    # 主配置文件
│   │   ├── applicationContext_dubbo_consumer.xml
│   │   ├── applicationContext_mq.xml
│   │   ├── applicationContext-quartz.xml
│   │   ├── logback.xml / logback-server.xml
│   │   ├── static/                   # 静态资源
│   │   │   ├── css/
│   │   │   ├── js/
│   │   │   └── images/
│   │   └── templates/                # HTML模板
│   │       ├── login.html
│   │       ├── flight.html
│   │       ├── edit.html
│   │       └── ...
│   └── script/
│       ├── assembly.xml              # 打包配置
│       └── start.sh                  # 启动脚本
└── src/test/java/                    # 测试代码
```

---

## 4. 核心代码分析

### 4.1 启动类 (Api.java)

```1:24:src/main/java/com/ias/server/flight/deal/Api.java
@SpringBootApplication
@EnableScheduling
@EnableJms
@ComponentScan(basePackages = "com.ias.server.flight.deal")
@EntityScan(basePackages = "com.ias.server.flight.deal")
@ImportResource({ "classpath:applicationContext_dubbo_consumer.xml",
                   "classpath:applicationContext_mq.xml",
                   "classpath:applicationContext-quartz.xml"})
public class Api {
    public static void main(String[] args) {
        SpringApplication.run(Api.class, args);
    }
}
```

**关键特性**:
- `@EnableScheduling`: 启用定时任务
- `@EnableJms`: 启用 ActiveMQ 消息监听
- `@ImportResource`: 导入 Dubbo、MQ、Quartz 配置
- 组件扫描范围: `com.ias.server.flight.deal`

### 4.2 控制器层 (FlightDealController.java)

这是项目的核心控制器，包含约2400行代码，主要功能:

#### 4.2.1 API 接口列表

| 接口路径 | 方法 | 功能描述 |
|---------|------|---------|
| `/getFlightStroeData` | GET/POST | 获取新建航班默认数据 |
| `/getFlights` | GET/POST | 分页查询航班列表 |
| `/addFlightCheck` | GET/POST | 添加航班权限检查 |
| `/addFlight` | POST | 添加新航班 |
| `/pagejump` | GET/POST | 航班管理主页面 |
| `/remoteUpdatePage` | POST | 远程更新页面 |
| `/remoteUpdateFlightPage` | POST | 航班编辑页面 |
| `/updateCheck` | POST | 更新权限检查 |
| `/update` | POST | 更新航班信息 |
| `/remoteUpload` | GET/POST | Excel导入航班 |
| `/other` | GET | 404页面 |

#### 4.2.2 核心业务逻辑

**添加航班流程** (`addFlight` 方法):
```
1. 参数校验: 检查 flno, flop, adid, tenant 是否完整
2. 重复检查: 调用 flightDataDealService 检查航班是否已存在
3. 生成 flseq: 使用 UUID 生成32位唯一序列号
4. 字段默认值设置: 设置 mssource, frct, depute 等默认值
5. 飞机数据映射: 根据 acname 自动获取 ac3c 和 category
6. 时间处理: 对特定租户进行时间格式转换
7. 保存数据: 同时保存到 FlightData 和 FlightStroeData
8. 消息推送: 通过 ActiveMQ 推送航班变更消息
```

**更新航班流程** (`update` 方法):
```
1. 变更检测: 使用 ToolUtil.isChanged() 比较新旧数据
2. 权限检查: 根据租户和航班号检查更新权限
3. 时间格式处理: 支持自定义时间格式转换
4. 字段变更追踪: 记录修改历史到 modification 字段
5. 消息推送: 推送航班变更消息到 MQ
```

#### 4.2.3 Excel 导入功能

```1912:2040:src/main/java/com/ias/server/flight/deal/controller/FlightDealController.java
private FlightImportCheckResult fileCheck(MultipartFile file, String tenant) {
    // 1. 校验文件格式 (.xls 或 .xlsx)
    // 2. 读取 Excel 数据
    // 3. 根据 FlightImportConfig 配置进行字段映射
    // 4. 时间字段特殊处理 (+/- 偏移)
    // 5. 返回解析结果
}
```

**导入校验规则**:
- 文件格式校验
- 列配置校验
- 数据类型校验
- 时间格式校验 (支持 HHmm、HHmm+、HHmm-)
- 航班日格式校验

### 4.3 基础数据加载 (BaseData.java)

```112:139:src/main/java/com/ias/server/flight/deal/job/BaseData.java
@PostConstruct
public void memoryDeal(){
    this.getTenants();      // 加载租户信息
    this.getAirCrafts();    // 加载飞机信息
    this.getAirLines();     // 加载航空公司
    this.getplaces();       // 加载机位信息
    this.getGates();        // 加载登机口
    this.getAirports();     // 加载机场信息
    this.getFlightTimeFields();  // 加载时间字段配置
    this.getAppends();      // 加载添加权限配置
    this.getUploads();      // 加载导入权限配置
    this.getUpdates();      // 加载更新权限配置
    this.getUnwantedfields();    // 加载隐藏字段配置
}
```

**静态数据缓存**:
- `tenants`: 租户信息
- `airCrafts`: 飞机型号
- `airLines`: 航空公司
- `places`: 机位
- `gates`: 登机口
- `airports`: 机场
- `updates/appends/uploads`: 权限规则

### 4.4 安全认证 (WebSecurityConfig.java)

```30:47:src/main/java/com/ias/server/flight/deal/config/WebSecurityConfig.java
protected void configure(HttpSecurity http) throws Exception {
    http.authorizeRequests()
        .antMatchers("/login", "/addFlightCheck", "/remoteUpload", 
                     "/remoteUpdatePage", "/remoteUpdateFlightPage",
                     "/getFlights", "/addFlight", "/update", 
                     "/updateCheck", "/getFlightStroeData")
        .permitAll()
        .anyRequest().authenticated()
        .and()
        .formLogin()
        .loginPage("/login")
        .successHandler(loginSuccessHandler())
        .usernameParameter("username")
        .passwordParameter("password")
        .and()
        .logout()
        .logoutSuccessUrl("/login")
        .permitAll()
        .and()
        .csrf()
        .disable();
    http.headers().frameOptions().disable();
}
```

**认证特点**:
- 自定义登录页面
- 登录成功处理器
- 允许特定 API 匿名访问
- 禁用 CSRF 保护 (用于 API 调用)

### 4.5 Dubbo 服务消费者配置

```1:34:src/main/resources/applicationContext_dubbo_consumer.xml
<dubbo:application name="flight-deal-service-consumer" />
<dubbo:consumer check="false" timeout="10000" />
<dubbo:registry protocol="zookeeper" address="${dubbo.address}" />
<dubbo:protocol name="dubbo" port="${dubbo.port}" />

<!-- 服务引用 -->
<dubbo:reference id="authService" interface="com.ias.security.webservice.IAuthService" />
<dubbo:reference id="tenantService" interface="com.misgws.service.TenantService" />
<dubbo:reference id="airCraftService" interface="com.misgws.service.AirCraftService" />
<dubbo:reference id="flightDealService" interface="com.ias.server.flight.service.FlightDataDealService" />
```

**服务列表**:
- `authService`: 认证服务
- `tenantService`: 租户服务
- `airCraftService`: 飞机服务
- `airLineService`: 航空公司服务
- `gateService`: 登机口服务
- `placeService`: 机位服务
- `airPortService`: 机场服务
- `flightDealService`: 航班数据服务

---

## 5. 数据库配置

### 5.1 数据源配置 (application.properties)
```properties
spring.datasource.driverClassName=com.mysql.jdbc.Driver
spring.datasource.url=jdbc:mysql://172.21.0.19:8066/db_flightdeal
spring.datasource.username=root
spring.datasource.password=Airport@123
spring.datasource.dbcp.max-active=100
spring.datasource.dbcp.max-idle=30
spring.datasource.dbcp.min-idle=10
```

### 5.2 JPA 配置
```properties
spring.jpa.database=MYSQL
spring.jpa.show-sql=false
spring.jpa.hibernate.ddl-auto=update
spring.jpa.hibernate.naming-strategy=org.hibernate.cfg.ImprovedNamingStrategy
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQL5Dialect
```

---

## 6. 前端页面分析

### 6.1 页面结构
```
templates/
├── login.html        # 登录页面
├── flight.html       # 航班管理主页面
├── edit.html         # 航班编辑页面
├── flightshow.html   # 航班展示页面
├── 404.html          # 404错误页面
├── 4041.html         # 404备用页面
└── prompt.html       # 提示页面
```

### 6.2 flight.html 主页面功能

**Vue.js 组件架构**:
```javascript
// 主应用
var app = new Vue({
    el: "#appmain",
    data: {
        flights: [],           // 航班列表
        tenants: {},           // 租户信息
        schemaadd: [],         // 添加表单配置
        schemaupdate: [],      // 更新表单配置
        schemasearch: [],      // 查询表单配置
        // ... 其他配置
    }
});

// 子组件
Vue.component('data-search', ...)   // 航班详情查看
Vue.component('data-edit', ...)     // 航班编辑
Vue.component('data-add', ...)      // 航班添加
Vue.component('data-import', ...)   // 航班导入
```

**页面功能**:
- 航班列表分页展示
- 多条件组合查询
- 航班添加、编辑、查看
- Excel 批量导入
- 权限控制的按钮显示

---

## 7. 配置属性说明

### 7.1 系统配置
| 属性 | 值 | 说明 |
|------|-----|------|
| server.port | 8889 | 服务端口 |
| server.tomcat.max-threads | 800 | 最大线程数 |
| mssource | flightdeal | 消息来源标识 |
| time.trigger | 10 | 触发时间配置 |

### 7.2 安全配置
| 属性 | 值 | 说明 |
|------|-----|------|
| security.add | add | 添加权限标识 |
| security.update | update | 更新权限标识 |
| security.upload | upload | 导入权限标识 |
| can.not.update.tenants | OIL_PVG | 不可更新的租户 |

### 7.3 权限规则配置

```properties
# 更新检查规则
update.check.rule={"CA_CTU":{"key":"CA_CTU","al2cs":"CA,MU,MF,BK,GS","depute":"1"}}

# 导入检查规则  
upload.check.rule={"CA_CTU":{"key":"CA_CTU","al2cs":"CA,MU,MF,BK,GS","depute":""}}

# 添加检查规则
append.check.rule={"CA_CTU":{"key":"CA_CTU","al2cs":"CA,MU,MF,CQ,MR,CL,MZ,MO,CD,MH,MN","depute":""}}
```

### 7.4 时间字段配置
```properties
# 成都租户特殊时间字段处理
flight.time.fields={
  "CA_CTU":{
    "key":"CA_CTU",
    "timefields":"",
    "textfields":"stot,etot,atot,ostd,oetd,otkf,deta,dlnd,dltime,ctim,atax,bbfa,doit,dpbt,dpot,etax,fdct,icit,ipbt,ipot,ptax,pushin,pushout"
  }
}
```

---

## 8. 实体类分析

### 8.1 FlightData 实体
```1:17:src/main/java/com/ias/server/flight/deal/entity/FlightData.java
@Entity
@Table(name = "flight_data")
public class FlightData extends FlightStroe {
    // 继承自 FlightStroe (位于 flight-tools/common 模块)
    // 包含航班的所有核心字段
}
```

### 8.2 FlightDataConfig 实体
- 存储航班字段配置信息
- 支持按租户、按操作类型(add/update/search)配置字段
- 包含字段编码、中文名、类型、默认值

### 8.3 FlightImportConfig 实体
- 存储 Excel 导入配置
- 包含字段映射、格式定义

---

## 9. Docker 部署

```dockerfile:1:15
FROM hub.adsb.vip:5000/ias/os-jvm:centos6-jdk8

WORKDIR /app

COPY ./target/${APP_NAME}.jar ./app.jar

VOLUME /tmp
VOLUME /app/config

ENTRYPOINT exec java -jar -server -Xms1024m -Xmx1024m -Xss256k app.jar
```

**部署特点**:
- 基于 CentOS6 + JDK8
- 内存配置: 最小1G, 最大1G
- 线程栈大小: 256k
- 支持配置目录挂载

---

## 10. 代码质量评估

### 10.1 优点
1. **分层清晰**: Controller -> Service -> Repository 三层架构
2. **配置外部化**: 使用 application.properties 集中管理配置
3. **缓存机制**: 基础数据使用 @PostConstruct 加载到内存
4. **权限控制**: 基于租户和角色的细粒度权限管理
5. **消息驱动**: 集成 ActiveMQ 实现数据变更推送
6. **分布式调用**: Dubbo 消费者配置完善

### 10.2 待改进点
1. **代码复用**: FlightDealController 过长(2400+行)，建议拆分
2. **异常处理**: 部分方法异常处理不够完善
3. **日志规范**: 使用自定义 Logger，建议统一使用 SLF4J
4. **单元测试**: 测试类较少，建议增加覆盖率
5. **安全增强**: 建议启用 CSRF 保护
6. **API 版本管理**: 建议添加 API 版本控制

### 10.3 性能考虑
1. **数据库连接池**: 使用 DBCP，配置合理
2. **缓存策略**: 基础数据已缓存，减少远程调用
3. **Dubbo 超时**: 已配置10秒超时
4. **线程池**: Tomcat 最大800线程

---

## 11. 依赖外部服务

| 服务 | 地址 | 端口 | 用途 |
|------|------|------|------|
| MySQL | 172.21.0.19 | 8066 | 主数据库 |
| Zookeeper | 172.21.0.20 | 2181 | Dubbo 注册中心 |
| ActiveMQ | 172.21.0.20 | 61616 | 消息队列 |
| Nexus 仓库 | 172.16.40.82:8088 | - | Maven 私有仓库 |

---

## 12. 总结

flightdealui 是一个功能完整的航班信息管理前端服务，具有以下特点:

1. **技术成熟**: 基于 Spring Boot + Vue.js 的成熟技术栈
2. **功能完整**: 航班 CRUD、Excel导入、权限控制、消息推送
3. **架构合理**: 分层清晰，配置外部化，易于维护
4. **部署灵活**: 支持 Docker 容器化部署
5. **扩展性强**: Dubbo 框架支持水平扩展

建议后续优化方向:
- 前后端分离改造
- API 接口规范化
- 增加单元测试覆盖率
- 引入 API 网关统一管理
