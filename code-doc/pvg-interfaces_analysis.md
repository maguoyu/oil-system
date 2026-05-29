# PVG-Interfaces 模块详细分析报告

> 分析时间: 2026-02-03  
> 分析范围: pvg-interfaces/  
> 用途: 航班数据接口对接分析、Bug排查参考

---

## 1. 项目概述

### 1.1 模块总览

pvg-interfaces 是航油系统的**外部接口对接层**，负责与各类外部系统进行航班数据交换。主要包括：
- ESB企业服务总线对接
- 第三方航空公司数据对接
- 新老系统数据同步
- 航班数据标准转换

### 1.2 模块清单

| 序号 | 模块名称 | 功能描述 | 技术栈 |
|------|---------|---------|--------|
| 1 | flight-oil-connection | 新老系统数据对接 | Spring 4.2.2 + Camel 2.13.2 |
| 2 | southern-airline-http-amq-pvg-new | 南航HTTP+AMQ对接 | Spring Boot 2.1.13 |
| 3 | esb-flightinfo-pvg | ESB航班信息对接 | Spring Boot 2.6.8 |
| 4 | flightcriterion-pvg | 航班标准管理 | Spring 4.3.3 + JPA |
| 5 | eastern-airline-pvg | 东航数据对接 | - |
| 6 | flight-oil-pvg | 航油数据对接 | - |
| 7 | esb-flightinfo-pvg-listener | ESB监听器 | - |

---

## 2. flight-oil-connection 模块详解

### 2.1 基本信息

| 属性 | 值 |
|------|-----|
| 模块名称 | flight-oil-connection |
| 功能描述 | 浦东航油新老系统数据对接模块 |
| 版本 | 5.0.14.23-test |
| Java版本 | 1.7 |
| Spring版本 | 4.2.2.RELEASE |
| Camel版本 | 2.13.2 |
| ActiveMQ版本 | 5.9.1 |

### 2.2 核心职责

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   新老系统数据对接核心功能                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. 任务数据同步                                                        │
│     ├── 从老系统拉取任务数据                                             │
│     ├── 任务号映射转换 (TNB Mapping)                                    │
│     └── 任务派工到新系统                                                 │
│                                                                         │
│  2. 油单数据同步                                                        │
│     ├── 从老系统拉取油单数据                                             │
│     ├── 油单数据格式转换                                                 │
│     └── 电子油单上传                                                     │
│                                                                         │
│  3. 航班数据同步                                                        │
│     ├── 航班数据拉取                                                     │
│     ├── 航班数据推送                                                     │
│     └── 联程航班处理                                                     │
│                                                                         │
│  4. 数据清理                                                            │
│     ├── 过期数据清理                                                     │
│     └── 油品分析数据同步                                                 │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.3 核心实体

#### 4.1.1 航班相关实体

| 实体类 | 功能说明 |
|-------|---------|
| FlightDataDic | 航班数据字典 |
| Flightoilbean | 航班数据Bean |
| UnitedBean | 联程航班Bean |
| FlightFieldChange | 航班字段变更 |

#### 4.1.2 任务与油单实体

| 实体类 | 功能说明 |
|-------|---------|
| OilTaskSheet | 油单任务表 |
| TaskEntity | 任务实体 |
| OilSheetEntity | 油单实体 |
| AircraftMDB | 飞机信息 |
| AirlinesMDB | 航空公司信息 |
| RegistrationMDB | 注册信息 |

### 2.4 核心定时任务

#### OilSystemConnectJob - 新老系统数据对接

**功能**: 定时同步老系统中的任务信息和油单信息

```java
@Component("oilSystemConnectJob")
public class OilSystemConnectJob extends MethodInvokingJob {
    
    // 主要方法 excute()
    // 1. 检查同步开关 (flightOilConnectFlag)
    // 2. 获取最大时间戳
    // 3. 查询新增油单
    // 4. 处理油单数据
    // 5. 执行任务派工
    // 6. 上传电子油单
}
```

**核心处理流程**:
```
1. 检查同步开关
   └── FlightDataDic.code = "flightOilConnectFlag"
   
2. 获取已同步油单的最大时间戳
   └── oilSheetSynchronizedService.selectMaxUrno()
   
3. 拉取新增油单
   └── oilSheetService.selectOilSheetByURNO(map)
   
4. 任务派工
   ├── 查询老任务对应的航班
   ├── 任务号映射转换
   └── 调用 serviceClientBz.synDispatchTask()
   
5. 油单转换与上传
   ├── convertOilSheetToBill()  // 油单格式转换
   ├── 上报任务节点 (TBEG/TEND)
   └── 上传电子油单
```

#### 其他定时任务

| 任务类 | 功能 |
|-------|------|
| OilAssayConnectJob | 油品分析数据同步 |
| DeleteOverdueDataJob | 删除过期数据 |
| FlightDealJob | 航班数据处理 |
| FlightPushJob | 航班数据推送 |
| CalculateUniteJob | 联程航班计算 |
| UnitedFlightPushJob | 联程航班推送 |

### 2.5 核心服务

#### OilSheetService
- `selectOilSheetByURNO()` - 按时间戳查询油单
- `selectTaskEntityByTNB()` - 按任务号查询任务

#### TaskService
- 任务数据管理服务

#### OilSheetSynchronizedService
- 已同步油单管理
- `selectMaxUrno()` - 获取最大时间戳
- `selectOilSheetSynchronizedByFkeyAndTnb()` - 检查油单是否已同步

### 2.6 消息生产者

| 生产者类 | 功能 | 目标队列 |
|---------|------|---------|
| FlightInfoProducer | 航班信息推送 | ActiveMQ |
| UnitedInfoProducer | 联程航班推送 | ActiveMQ |

---

## 3. southern-airline-http-amq-pvg-new 模块详解

### 3.1 基本信息

| 属性 | 值 |
|------|-----|
| 模块名称 | southern-airline-http-amq-pvg-new |
| 功能描述 | 南航HTTP+ActiveMQ数据对接 |
| 版本 | 1.0.0.12 |
| 框架 | Spring Boot 2.1.13 |
| Java版本 | 8 |

### 3.2 核心职责

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     南航数据对接核心功能                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. HTTP数据拉取                                                        │
│     ├── 南航航班信息API调用                                             │
│     ├── 历史数据交接处理                                                │
│     └── 数据解析与转换                                                  │
│                                                                         │
│  2. 数据处理                                                            │
│     ├── 航班数据标准化                                                  │
│     ├── 航班匹配字典处理                                                │
│     └── 联程航班处理                                                    │
│                                                                         │
│  3. 消息推送                                                            │
│     ├── 航班信息推送                                                    │
│     ├── 联程航班推送                                                    │
│     └── 历史数据交接                                                    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.3 核心组件

#### Application.java

```java
@SpringBootApplication
@EnableScheduling
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

#### BuildHttpClient.java - HTTP客户端构建

```java
@Component
public class BuildHttpClient {
    // HTTP客户端配置与构建
}
```

#### FlightInfoHttp.java - 航班信息HTTP调用

```java
public class FlightInfoHttp {
    // 南航航班信息API调用
}
```

### 3.4 核心定时任务

#### FlightMessageJob - 航班消息处理

```java
@Component
@Configuration
@EnableScheduling
public class FlightMessageJob {
    
    @Scheduled(cron = "${flight.time.luts}")
    public void deal() {
        // 1. 获取上次处理时间
        // 2. 查询新增航班数据
        // 3. 转换为标准格式
        // 4. 发送到ActiveMQ
        // 5. 更新时间戳
    }
}
```

#### HistoricalDataHandoverDealJob - 历史数据交接

```java
// 历史数据交接处理
```

#### UnitedFlightMessageJob - 联程航班消息

```java
// 联程航班消息处理
```

### 3.5 数据模型

#### 核心实体

| 实体类 | 功能说明 |
|-------|---------|
| Flightinfo | 航班信息 |
| FlightMessage | 航班消息 |
| FlightDataDic | 航班数据字典 |
| FlightMatchDic | 航班匹配字典 |
| FlightVial | 航班航线信息 |
| UnitedFlight | 联程航班 |

#### HTTP响应Bean

| Bean类 | 功能 |
|-------|------|
| FlightInfo | 航班信息响应 |
| FlightInfos | 航班信息列表 |
| BDType | 基础数据 |
| HDType | 头部数据 |
| RType | 响应类型 |
| SUBType | 子类型 |

### 3.6 核心服务

| 服务类 | 功能 |
|-------|------|
| FlightinfoService | 航班信息服务 |
| FlightMessageMapper | 航班消息处理 |
| FlightMatchDicService | 匹配字典服务 |
| FlightDataDicService | 数据字典服务 |
| FlightVialService | 航线信息服务 |
| UnitedFlightService | 联程航班服务 |

### 3.7 消息发送

#### SendProducer - ActiveMQ消息发送

```java
@Component
public class SendProducer {
    public void sendFlightMessageBean(FlightStroeData flight);
    // 航班消息发送
}
```

---

## 4. flightcriterion-pvg 模块详解

### 4.1 基本信息

| 属性 | 值 |
|------|-----|
| 模块名称 | flightcriterion-pvg |
| 功能描述 | 航班标准管理模块 |
| 版本 | 7.0.5.3 |
| 父级POM | flightparent 7.0.1.0 |
| Java版本 | 1.7 |
| Spring版本 | 4.3.3.RELEASE |
| JPA版本 | 1.10.2.RELEASE |
| Camel版本 | 2.17.0 |
| ActiveMQ版本 | 5.12.0 |

### 4.2 核心职责

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     航班标准管理核心功能                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. 航班数据管理                                                        │
│     ├── 航班信息存储                                                    │
│     ├── 航班历史记录                                                    │
│     └── 航班变更处理                                                    │
│                                                                         │
│  2. 数据匹配                                                            │
│     ├── 航班匹配规则                                                    │
│     ├── 航班字典维护                                                    │
│     └── 航线信息管理                                                    │
│                                                                         │
│  3. 消息处理                                                            │
│     ├── 航班消息处理                                                    │
│     ├── 航班推送                                                        │
│     └── 联程航班处理                                                    │
│                                                                         │
│  4. 定时任务                                                            │
│     ├── 航班数据推送                                                    │
│     ├── 联程航班计算                                                    │
│     └── 航班日切换                                                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4.3 核心入口

#### App.java

```java
public class App {
    public static void main(String[] args) {
        ClassPathXmlApplicationContext context = 
            new ClassPathXmlApplicationContext("applicationContext.xml");
        context.start();
    }
}
```

### 4.4 核心组件

#### FlightMessageDeal - 航班消息处理

```java
@Component
public class FlightMessageDeal {
    // 航班消息处理逻辑
}
```

#### FlightInfoManager - 航班信息管理

```java
public class FlightInfoManager {
    // 航班信息管理
}

public class FlightInfoManager2 {
    // 航班信息管理v2
}
```

### 4.5 核心定时任务

| 任务类 | 功能 |
|-------|------|
| FlightPushJob | 航班数据推送 |
| CalculateUniteJob | 联程航班计算 |
| SwitchFlop | 航班日切换 |
| UnitedFlightPushJob | 联程航班推送 |

### 4.6 实体模型

| 实体类 | 功能说明 |
|-------|---------|
| FlightInfo | 航班信息 |
| FlightInfoHis | 航班历史 |
| FlightDataDic | 航班数据字典 |
| FlightMatchData | 航班匹配数据 |
| FlightVial | 航班航线 |
| Message | 消息 |
| Replacement | 替换记录 |
| UnitedFlightExe | 联程航班执行 |

### 4.7 消息生产者

| 生产者类 | 功能 |
|-------|------|
| FlightProducer | 航班消息推送 |
| UnitedFlightProducer | 联程航班推送 |

### 4.8 内存数据管理

#### DataMemory - 内存数据缓存

```java
public class DataMemory {
    // 航班数据内存缓存
    // 提高查询性能
}
```

---

## 5. esb-flightinfo-pvg 模块详解

### 5.1 基本信息

| 属性 | 值 |
|------|-----|
| 模块名称 | esb-flightinfo-pvg |
| 功能描述 | ESB航班信息对接 |
| 版本 | 1.0.0.24 |
| 框架 | Spring Boot 2.6.8 |
| Java版本 | 1.7 |

### 5.2 核心职责

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     ESB企业服务总线对接核心功能                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. ESB消息接收                                                        │
│     ├── 航班信息监听                                                    │
│     ├── 任务GPS监听                                                     │
│     └── 消息解析                                                        │
│                                                                         │
│  2. 数据处理                                                            │
│     ├── 航班数据处理                                                    │
│     ├── 数据转换                                                        │
│     └── 错误处理                                                        │
│                                                                         │
│  3. 消息推送                                                            │
│     ├── 航班推送                                                        │
│     ├── 联程航班推送                                                    │
│     └── 历史数据交接                                                    │
│                                                                         │
│  4. 定时任务                                                            │
│     ├── ESB数据请求                                                     │
│     ├── 航班消息处理                                                    │
│     └── 数据同步                                                        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.3 核心入口

```java
@SpringBootApplication
@EnableScheduling
public class EsbFlightinfoPvgApplication {
    public static void main(String[] args) {
        SpringApplication.run(EsbFlightinfoPvgApplication.class, args);
    }
}
```

### 5.4 核心监听器

#### FlightInfoListener - 航班信息监听

```java
public class FlightInfoListener {
    // 监听ESB航班信息
}
```

#### TaskAndGpsListener - 任务与GPS监听

```java
public class TaskAndGpsListener {
    // 监听任务和GPS数据
}
```

### 5.5 核心定时任务

| 任务类 | 功能 |
|-------|------|
| FlightInfoForEsbReceive | ESB航班数据接收 |
| FlightMessageJob | 航班消息处理 |
| HistoricalDataHandoverDealJob | 历史数据交接 |
| UnitedFlightMessageJob | 联程航班消息 |
| RequestTimer | ESB请求定时器 |

### 5.6 数据模型

| 实体类 | 功能说明 |
|-------|---------|
| FlightInfo | 航班信息 |
| FlightInfoMessage | 航班消息 |
| FlightDataDic | 航班数据字典 |
| FlightMatchDic | 航班匹配字典 |
| FlightVial | 航班航线 |
| UnitedFlight | 联程航班 |

### 5.7 ESB配置

#### AMQConfig - ActiveMQ配置

```java
public class AMQConfig {
    // ActiveMQ连接配置
}
```

#### AMQConnection - AMQ连接管理

```java
public class AMQConnection {
    // ActiveMQ连接管理
}
```

### 5.8 常量定义

#### EsbConstance - ESB常量

```java
public class EsbConstance {
    // ESB相关常量
}

public class IASConstance {
    // IAS相关常量
}
```

---

## 6. 数据流转分析

### 6.1 整体数据流转图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           PVG-Interfaces 数据流转                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────┐                                                        │
│  │   南航API       │                                                        │
│  │ (HTTP接口)      │                                                        │
│  └────────┬────────┘                                                        │
│           │ HTTP请求                                                        │
│           ▼                                                                │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │             southern-airline-http-amq-pvg-new                         │  │
│  │  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐               │  │
│  │  │ HTTP数据拉取 │───▶│ 数据处理    │───▶│ ActiveMQ    │               │  │
│  │  │             │    │             │    │ 消息推送    │               │  │
│  │  └─────────────┘    └─────────────┘    └──────┬──────┘               │  │
│  └────────────────────────────────────────────────┼──────────────────────┘  │
│                                                   │                          │
│                                                   ▼                          │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                        flightdispatch (核心处理)                      │  │
│  │  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐               │  │
│  │  │ 数据存储    │◀───│ 数据标准化  │◀───│ 消息接收    │               │  │
│  │  │             │    │             │    │             │               │  │
│  │  └─────────────┘    └─────────────┘    └──────┬──────┘               │  │
│  └────────────────────────────────────────────────┼──────────────────────┘  │
│                                                   │                          │
│           ┌───────────────────────────────────────┼───────────────────────┐  │
│           │                                       │                       │  │
│           ▼                                       ▼                       │  │
│  ┌─────────────────┐                 ┌─────────────────┐                 │  │
│  │ flight-oil-connection │           │ esb-flightinfo-pvg │                 │  │
│  │ (新老系统同步)   │                 │ (ESB对接)        │                 │  │
│  └────────┬────────┘                 └────────┬────────┘                 │  │
│           │                                   │                           │  │
│           ▼                                   ▼                           │  │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                         ActiveMQ 消息队列                            │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌────────────┐  │  │
│  │  │ 航班消息    │  │ 联程航班    │  │ 任务消息    │  │ GPS消息    │  │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └────────────┘  │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 消息类型汇总

| 消息类型 | 来源模块 | 目标 | 说明 |
|---------|---------|------|------|
| FlightStroeData | southern-airline | flightdispatch | 航班数据 |
| FlightMessage | esb-flightinfo | flightdispatch | ESB航班消息 |
| UnitedFlight | 各模块 | flightdispatch | 联程航班 |
| TaskInfo | flight-oil-connection | 新系统 | 任务派工 |
| OilSheetInfo | flight-oil-connection | 新系统 | 油单数据 |
| GPSInfo | esb-flightinfo | 终端 | GPS定位信息 |

---

## 7. 技术架构对比

### 7.1 技术栈对比

| 模块 | 框架 | Java | 数据库 | 消息队列 |
|------|------|------|--------|---------|
| flight-oil-connection | Spring 4.2 | 1.7 | MyBatis | ActiveMQ 5.9 |
| southern-airline-http-amq | Spring Boot 2.1 | 8 | MyBatis | ActiveMQ |
| esb-flightinfo | Spring Boot 2.6 | 1.7 | MyBatis | ActiveMQ |
| flightcriterion | Spring 4.3 | 1.7 | JPA | ActiveMQ 5.12 |

### 7.2 差异分析

| 维度 | 旧架构 | 新架构 |
|-----|-------|-------|
| 框架 | Spring传统配置 | Spring Boot |
| 数据库 | MyBatis | MyBatis/JPA混合 |
| Java版本 | 1.7 | 8+ |
| 配置 | XML配置 | yml配置 |
| 部署 | WAR包 | JAR包 |

---

## 8. 潜在Bug风险点分析

### 8.1 高风险问题

#### 8.1.1 数据同步一致性问题

**位置**: `OilSystemConnectJob.dealOilSheetEntity()`

```java
// 问题: 油单同步过程中的事务处理
if(list2 == null || list2.size() == 0) {
    // 任务派工
    TaskBzResp taskBzResp = taskDispatch(task);
    
    // 油单转换
    this.saveOilBill(entity, tnb_new, fkey_new, task.getVNB());
    
    // 记录同步状态
    oilSheetSynchronizedService.insertOilSheetSynchronizedEntity(sEntity);
}
```

**风险**: 
- 派工成功但油单上传失败时，系统状态不一致
- 没有事务回滚机制

**建议**: 添加事务控制和补偿机制

#### 8.1.2 时间戳处理问题

**位置**: `FlightMessageJob.deal()`

```java
long start = Timestamp.valueOf(flightMatchDic.getValue()).getTime();
Long end = flightInfoService.getMaxTimeLuts();
```

**风险**: 
- 时间格式异常会导致转换失败
- 时区问题可能导致数据遗漏

**建议**: 添加时间格式校验和日志记录

### 8.2 中等风险问题

#### 8.2.1 循环发送消息

**位置**: 多模块的Job类

```java
for (Flightinfo flightInfo : flightInfos) {
    // 逐条发送
    producer.sendFlightMessageBean(flight);
}
```

**影响**: 大量数据时性能问题

**建议**: 批量发送

#### 8.2.2 异常处理不完整

**位置**: 多个Job类

```java
} catch (Exception e) {
    StringWriter sw = new StringWriter();
    e.printStackTrace(pw);
    logger.error(sw.toString());
    // 仅记录日志，无其他处理
}
```

**影响**: 异常数据可能被遗漏

**建议**: 添加重试机制和死信队列

### 8.3 低风险问题

#### 8.3.1 硬编码配置

```java
@Value("#{configProperties['tenant']}")
private String tenant;
```

**建议**: 统一配置管理

---

## 9. 数据库表关联

### 9.1 核心表清单

| 表名 | 模块 | 说明 |
|-----|------|------|
| flight_data_dic | 多模块 | 航班数据字典 |
| flight_info | 多模块 | 航班信息 |
| flight_vial | 多模块 | 航班航线 |
| flight_match_dic | 多模块 | 航班匹配字典 |
| united_flight | 多模块 | 联程航班 |
| oil_sheet | flight-oil-connection | 油单表 |
| task | flight-oil-connection | 任务表 |
| oil_sheet_synchronized | flight-oil-connection | 油单同步记录 |

### 9.2 表关联关系

```
flight_info (航班主表)
    ├── flight_vial (航线信息) 1:1
    ├── flight_match_dic (匹配字典) 1:N
    └── united_flight (联程航班) 1:N

oil_sheet (油单表)
    └── task (任务表) 1:1
```

---

## 10. 改进建议

### 10.1 架构优化

1. **统一技术栈**
   - 迁移到Spring Boot
   - 统一Java版本
   - 统一配置管理

2. **消息可靠性**
   - 添加消息确认机制
   - 实现死信队列
   - 添加消息重试

3. **性能优化**
   - 批量处理消息
   - 添加缓存层
   - 数据库索引优化

### 10.2 代码规范

1. **异常处理**
   - 统一异常处理
   - 添加业务补偿
   - 完善日志记录

2. **测试覆盖**
   - 添加单元测试
   - 添加集成测试
   - 添加性能测试

---

## 附录

### A. 模块依赖关系

```
flight-oil-connection
├── flight-oil-service
├── MISGWSAPI
└── service-dubbo-rcp

southern-airline-http-amq-pvg-new
├── MySQL
├── ActiveMQ
└── MyBatis

esb-flightinfo-pvg
├── MySQL
├── ActiveMQ
└── FastJSON

flightcriterion-pvg
├── flightparent
├── JPA
└── ActiveMQ
```

### B. 关键配置项

```properties
# flight-oil-connection
tenant=xxx
ap3c=PVG
canVehicle=xxx
pipeVehicle=xxx
flightOilConnectFlag=Y

# southern-airline-http-amq-pvg-new
flight.time.luts=0/5 * * * * *
spring.datasource.url=jdbc:mysql://...

# esb-flightinfo-pvg
spring.activemq.broker-url=tcp://...
```

---

> **报告完成**  
> 如有问题或需要更详细的分析，请联系技术团队。
