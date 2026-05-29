# resourcesub 模块详细分析报告

> 分析时间: 2026-02-03  
> 分析范围: resourcesub/  
> 用途: 航班订阅、通知管理、任务申请功能分析

---

## 1. 项目概述

### 1.1 模块基本信息

| 属性 | 值 |
|------|-----|
| 模块名称 | resourcesub |
| 功能描述 | 航班订阅管理子模块（原5500对接模块）|
| 版本 | 5.0.3.1 |
| 框架 | Dubbo 2.5.3 + Spring 4.3.3 |
| Java版本 | 1.7 |
| 运行环境 | Apache Camel 2.13.2 + ActiveMQ 5.9.1 |
| 入口类 | FlightDataManagerProvider (Dubbo Container) |

### 1.2 核心职责

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        resourcesub 核心功能                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. 航班订阅管理                                                        │
│     ├── 订阅条件上传/解析                                               │
│     ├── 订阅条件查询                                                    │
│     └── 订阅条件与航班匹配                                              │
│                                                                         │
│  2. 航班锁定管理                                                        │
│     ├── 锁定条件上传                                                    │
│     ├── 自动任务申请                                                    │
│     └── 锁定条件与航班匹配                                              │
│                                                                         │
│  3. 通知管理                                                            │
│     ├── 通知创建/更新                                                   │
│     ├── 通知下发                                                        │
│     ├── 通知查询                                                        │
│     └── 通知关闭                                                        │
│                                                                         │
│  4. 定时任务                                                            │
│     ├── 航班自检                                                        │
│     └── 失效时间检查                                                    │
│                                                                         │
│  5. 消息处理                                                            │
│     ├── XML消息解析                                                     │
│     ├── JSON消息转换                                                    │
│     └── 消息下发                                                        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.3 技术架构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           技术架构图                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                      终端设备 (Mobile)                           │   │
│  │  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌─────────────┐  │   │
│  │  │ 航班订阅  │  │ 航班锁定  │  │ 通知查询  │  │ 任务状态    │  │   │
│  │  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘  └──────┬──────┘  │   │
│  └────────┼──────────────┼──────────────┼───────────────┼─────────┘   │
│           │              │              │               │             │
│           └──────────────┴──────────────┴───────────────┘             │
│                                      │                                  │
│                                      ▼                                  │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                     ActiveMQ 消息队列                            │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │   │
│  │  │ RSDY (订阅) │  │ DYIN (查询) │  │ DYRP (通知) │              │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘              │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                      │                                  │
│                                      ▼                                  │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                      Apache Camel 路由                           │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │   │
│  │  │ flightLock  │  │ pnas-down   │  │ pnqs-down   │              │   │
│  │  │ Disposer    │  │             │  │             │              │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘              │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                      │                                  │
│                                      ▼                                  │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                      resourcesub 模块                           │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │   │
│  │  │ Service     │  │ CheckFlight │  │ Pnas/Pnqs   │              │   │
│  │  │ Manager     │  │ Job         │  │ Envelope    │              │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘              │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                      │                                  │
│           ┌──────────────────────────┼──────────────────────────┐      │
│           │                          │                          │      │
│           ▼                          ▼                          ▼      │
│  ┌─────────────────┐  ┌─────────────────────┐  ┌─────────────────┐   │
│  │  MySQL 数据库   │  │   Dubbo 远程调用    │  │  ActiveMQ 推送  │   │
│  │  (订阅/锁定/通知)│  │  (flightdispatch)  │  │  (终端设备)     │   │
│  └─────────────────┘  └─────────────────────┘  └─────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. 核心组件详解

### 2.1 入口类

#### FlightDataManagerProvider

```java
public class FlightDataManagerProvider {
    public static void main(String[] args) {
        com.alibaba.dubbo.container.Main.main(args);
    }
}
```

**说明**: 使用Dubbo容器启动，作为Dubbo服务提供者运行。

### 2.2 核心服务层

#### 2.2.1 ServiceManager - 服务管理器

**功能**: 核心业务逻辑处理，包括航班订阅、锁定条件判定、任务申请、通知管理等。

| 属性 | 值 |
|------|-----|
| 类名 | `com.ias.lib.flightdata.resourcesub.process.ServiceManager` |
| Bean名称 | `serviceManager` |
| 职责 | 业务逻辑编排与协调 |

**核心方法**:

| 方法名 | 功能 | 调用方 |
|-------|------|-------|
| `applyTask(Map<String,String> flightMap)` | 满足锁定条件时自动申请任务 | FlightMessageService |
| `decideFlightLockEntitys()` | 判定航班满足的订阅/锁定条件 | subscribe, applyTask |
| `subscribe()` | 订阅：创建/更新/下发通知 | FlightTaskService |
| `combineFlightLockEntitys()` | 合并相同员工号和服务类型的订阅条件 | subscribe |
| `convertFlightMap()` | 转换航班字段为终端格式 | subscribe |
| `autoCheckFlights()` | 定时自检航班 | CheckFlightJob |

**订阅处理流程**:

```
1. 判定航班满足的订阅条件
   └── decideFlightLockEntitys(flightMap, "1", null)
   
2. 合并订阅条件（员工号+服务类型去重）
   └── combineFlightLockEntitys(flightLockEntitiys)
   
3. 遍历订阅条件
   ├── 判断是否已有通知
   ├── 如有则更新通知
   ├── 如无则创建通知
   └── 通知员工在线状态检查
       └── 获取员工设备号
           └── 下发通知消息 (PnasEnvelope)
```

**任务申请流程**:

```
1. 判定航班满足的锁定条件
   └── decideFlightLockEntitys(flightMap, "0", null)
   
2. 遍历锁定条件
   ├── 获取员工号
   ├── 查询员工是否有该类型任务
   └── 如无任务则申请
       └── serviceBz.applyTask()
```

#### 2.2.2 FlightMessageService - 航班消息服务

**功能**: 处理航班消息订阅请求。

| 属性 | 值 |
|------|-----|
| 类名 | `com.ias.lib.flightdata.resourcesub.process.FlightMessageService` |
| Bean名称 | `flightMessageService` |
| 实现的接口 | `Processor` |

**核心方法**:

| 方法名 | 功能 |
|-------|------|
| `process()` | 航班消息处理（当前未使用） |
| `flightSubscribe()` | 航班订阅入口 |
| `applyTask()` | 锁定条件匹配与任务申请 |

#### 2.2.3 FlightTaskService - 航班任务服务

**功能**: 处理航班任务消息。

| 属性 | 值 |
|------|-----|
| 类名 | `com.ias.lib.flightdata.resourcesub.process.FlightTaskService` |
| Bean名称 | `flightTaskService` |
| 实现的接口 | `Processor`, `ApplicationContextAware` |

**核心方法**:

| 方法名 | 功能 |
|-------|------|
| `process()` | 任务消息JSON转Bean |
| `flightSubscribe()` | 任务关联的航班订阅 |
| `dispatchPreNotice()` | 任务通知派发 |

#### 2.2.4 OilService - 航油服务

**功能**: 处理航油相关信息（代码已注释）。

| 属性 | 值 |
|------|-----|
| 类名 | `com.ias.lib.flightdata.resourcesub.process.OilService` |
| Bean名称 | `oilService` |
| 状态 | 代码已注释，待恢复 |

#### 2.2.5 PreNoticeModelService - 通知模型服务

**功能**: 处理通知消息模型。

| 属性 | 值 |
|------|-----|
| 类名 | `com.ias.lib.flightdata.resourcesub.process.PreNoticeModelService` |
| Bean名称 | `preNoticeModelService` |
| 实现的接口 | `Processor` |

---

## 3. 消息处理层

### 3.1 FlightLockDisposer - 订阅/锁定条件处理器

**功能**: 解析终端上传的订阅/锁定条件XML消息。

| 属性 | 值 |
|------|-----|
| 类名 | `com.ias.lib.flightdata.resourcesub.dispose.FlightLockDisposer` |
| Bean名称 | `flightLockDisposer` |

**支持的消息类型**:

| 消息类型 | 功能 | 处理方法 |
|---------|------|---------|
| RSDY | 上传/更新订阅/锁定条件 | `processConditionsUpload()` |
| DYIN | 查询订阅条件 | `processConditionsQuery()` |
| DYRP | 查询通知 | `processPreNoticeQuery()` |

**消息解析流程**:

```xml
<!-- 消息结构示例 -->
<R>
    <HD>
        <MT>消息类型</MT>           <!-- RSDY/DYIN/DYRP -->
        <MS>消息号</MS>             <!-- 消息唯一标识 -->
        <SSN>设备号</SSN>           <!-- 发送设备号 -->
        <SEN>员工号</SEN>           <!-- 发送员工号 -->
        <STM>发送时间</STM>         <!-- 发送时间戳 -->
    </HD>
    <BD>
        <!-- 订阅条件内容 -->
        <RTP>服务类型</RTP>         <!-- 如: JY(加油) -->
        <TYP>处理类型</TYP>         <!-- 0:锁定, 1:订阅 -->
        <LTM>过期时间</LTM>         <!-- 失效时间 -->
        <!-- 锁定条件 -->
        <PSN>机位</PSN>             <!-- 如: 101|102 -->
        <FLN>航班号</FLN>           <!-- 如: MU5183 -->
        <REG>机号</REG>             <!-- 如: B-1234 -->
        <ARE>区域</ARE>             <!-- 如: T1 -->
        <GTN>登机口</GTN>           <!-- 如: A01 -->
        <BLT>转盘口</BLT>           <!-- 如: 10 -->
    </BD>
</R>
```

**条件解析逻辑**:

```java
// 支持的锁定类型
parseLockType("PSN", document, fEntity, lists);  // 机位
parseLockType("FLN", document, fEntity, lists);  // 航班号
parseLockType("REG", document, fEntity, lists);  // 机号
parseLockType("ARE", document, fEntity, lists);  // 区域
parseLockType("GTN", document, fEntity, lists);  // 登机口
parseLockType("BLT", document, fEntity, lists);  // 转盘口
```

### 3.2 消息包结构 (Envelope)

| 包类型 | 功能 | 消息类型 |
|-------|------|---------|
| PnasEnvelope | 通知主动下发 | DYRP |
| PnleEnvelope | 通知失效 | TZSX |
| PnqsEnvelope | 通知查询应答 | DYRP |
| LcqsEnvelope | 订阅条件查询应答 | DYIN |
| LcqfEnvelope | 订阅条件查询失败 | DYIN |
| SlrcEnvelope | 订阅/锁定条件应答 | RSDY |

#### PnasEnvelope - 通知下发消息

```java
public class PnasEnvelope implements Serializable {
    // 消息头
    private String mt;      // 消息类型: DYRP
    private String ms;      // 消息号(由terminal添加)
    private String rsn;     // 接收设备号
    private String ren;     // 接收员工号
    private String stm;     // 消息发送时间
    
    // 消息体
    private String nid;     // 通知ID
    private String txl;     // JSON数据长度
    private String txt;     // JSON数据内容
}
```

---

## 4. 定时任务

### 4.1 CheckFlightJob - 航班自检任务

**功能**: 定时从航班模块获取航班，检查是否满足订阅条件。

| 属性 | 值 |
|------|-----|
| 类名 | `com.ias.lib.flightdata.resourcesub.job.CheckFlightJob` |
| Bean名称 | `checkFlightJob` |
| 配置参数 | `flight.timeRangeConfig` (时间范围) |

**执行流程**:

```java
public void execute() {
    // 1. 获取配置的时间范围
    float timeRange = Float.parseFloat(timeRangeConfig);
    
    // 2. 计算当前时间范围
    String nowTime = CommonUtil.getCurrentTime(new Date());
    String minTime = CommonUtil.getMinTime(nowTime, timeRange);
    String maxTime = CommonUtil.getMaxTime(nowTime, timeRange);
    
    // 3. 远程调用航班模块查询航班
    List<ServiceFlight> flightLists = 
        serviceModelService.getServiceFlightsByStartCtotAndEndCtotAndTenant(
            minTime, maxTime, tenant);
    
    // 4. 遍历航班进行订阅检查
    for(ServiceFlight serviceFlight : flightLists) {
        serviceManager.subscribe(flightMap, null, callFlag);
    }
}
```

### 4.2 CheckLostTimeJob - 失效时间检查任务

**功能**: 检查并关闭过期通知。

---

## 5. 数据模型层

### 5.1 核心实体

| 实体类 | 功能说明 | 表名 |
|-------|---------|------|
| FlightLockEntity | 订阅/锁定条件 | flight_lock |
| PreNoticeEntity | 通知 | pre_notice |
| PreNoticeRangeEntity | 通知接收范围 | pre_notice_range |
| TaskEntity | 任务信息 | task |
| TaskMarkEntity | 任务标注 | task_mark |
| TaskProcessEntity | 任务过程 | task_process |
| UserEntity | 用户信息 | user |

### 5.2 FlightLockEntity - 订阅/锁定条件

**字段说明**:

| 字段 | 类型 | 说明 |
|-----|------|------|
| id | String | 主键UUID |
| employeeNumber | String | 员工号 |
| serviceType | String | 服务类型 (如: JY-加油) |
| processType | String | 处理类型 (0:锁定, 1:订阅) |
| lockType | String | 锁定类型 (PSN/FLN/REG/ARE/GTN/BLT) |
| lockValue | String | 锁定值 |
| referenceTime | String | 参考时间字段 |
| timeRange | Float | 时间范围 (小时) |
| timeRangeType | String | 时间范围类型 (0:往前,1:往后,2:前后) |
| lostTime | String | 失效时间 |
| tenantId | String | 租户ID |
| qualifiedFlag | String | 条件满足标志 |

**锁定类型说明**:

| 锁定类型 | 含义 | 示例 |
|---------|------|------|
| PSN | 机位 | 101, 102 |
| FLN | 航班号 | MU5183 |
| REG | 飞机注册号 | B-1234 |
| ARE | 区域 | T1, T2 |
| GTN | 登机口 | A01, B02 |
| BLT | 转盘口 | 10, 11 |

### 5.3 PreNoticeEntity - 通知

**字段说明**:

| 字段 | 类型 | 说明 |
|-----|------|------|
| pnid | String | 通知ID |
| ntyp | String | 通知类型 (服务类型) |
| fkey | String | 航班唯一号 |
| flop | String | 航班日 |
| flno | String | 航班号 |
| adid | String | 进离港 |
| stat | String | 状态 (OPEN/CLOSED) |
| data | String | 通知数据 (JSON格式) |
| luts | Long | 时间戳 |

### 5.4 数据关系图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           数据关系图                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  FlightLockEntity (订阅/锁定条件)                                       │
│  ┌─────────────────────────────────────────────────────────────┐       │
│  │ id, employeeNumber, serviceType, processType,               │       │
│  │ lockType, lockValue, timeRange, tenantId, lostTime          │       │
│  └─────────────────────────────────────────────────────────────┘       │
│           │                                                           │
│           │ 触发                                                       │
│           ▼                                                           │
│  PreNoticeEntity (通知)                                                │
│  ┌─────────────────────────────────────────────────────────────┐       │
│  │ pnid, ntyp, fkey, flop, flno, adid, stat, data, luts        │       │
│  └─────────────────────────────────────────────────────────────┘       │
│           │                                                           │
│           │ 包含                                                       │
│           ▼                                                           │
│  PreNoticeRangeEntity (通知范围)                                       │
│  ┌─────────────────────────────────────────────────────────────┐       │
│  │ id, pnid, nenb (通知接收员工号), tenantId                    │       │
│  └─────────────────────────────────────────────────────────────┘       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 6. 远程服务依赖

### 6.1 Dubbo 服务调用

| 服务接口 | 功能 | 来源模块 |
|---------|------|---------|
| ServiceModelService | 航班信息服务 | flightdispatch |
| ServiceBz | 任务服务 | 服务模块 |
| ServiceTerminalBz | 终端任务服务 | 服务模块 |

### 6.2 API 依赖

| 依赖包 | 功能 |
|-------|------|
| resourceapi | 资源/登录服务 |
| security-api | 安全认证服务 |
| MISGWSAPI | MIS系统接口 |

---

## 7. 业务场景分析

### 7.1 场景一：员工订阅航班

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        航班订阅流程                                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. 员工在终端设置订阅条件                                               │
│     └── 订阅类型: 订阅 (processType=1)                                  │
│     └── 锁定类型: 机位 (PSN=101)                                        │
│     └── 服务类型: 加油 (JY)                                             │
│     └── 时间范围: 2小时 (timeRange=2)                                   │
│     └── 时间范围类型: 航班预计到达前 (timeRangeType=1)                  │
│                                                                         │
│  2. 终端发送XML消息到ActiveMQ                                           │
│     └── 消息类型: RSDY                                                  │
│                                                                         │
│  3. FlightLockDisposer接收并解析                                        │
│     └── 保存订阅条件到数据库                                             │
│     └── 触发立即航班检查                                                │
│     └── 下发应答消息 (SlrcEnvelope)                                     │
│                                                                         │
│  4. CheckFlightJob定时检查                                              │
│     └── 查询时间范围内的航班                                            │
│     └── 匹配订阅条件                                                    │
│     └── 创建通知                                                        │
│     └── 下发通知到终端 (PnasEnvelope)                                   │
│                                                                         │
│  5. 终端收到通知                                                        │
│     └── 显示航班信息                                                    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 7.2 场景二：航班锁定自动派工

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        航班锁定自动派工流程                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. 员工在终端设置锁定条件                                               │
│     └── 锁定类型: 锁定 (processType=0)                                  │
│     └── 锁定类型: 登机口 (GTN=A01)                                      │
│     └── 服务类型: 加油 (JY)                                             │
│                                                                         │
│  2. 航班到达触发锁定匹配                                                │
│     └── 调用 decideFlightLockEntitys(flightMap, "0", null)             │
│     └── 匹配到锁定条件                                                   │
│                                                                         │
│  3. 自动任务申请                                                        │
│     └── 查询员工当前任务                                                │
│     └── 如无任务则调用 serviceBz.applyTask()                           │
│                                                                         │
│  4. 任务创建成功                                                        │
│     └── 通知员工                                                        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 8. 潜在Bug风险点分析

### 8.1 高风险问题

#### 8.1.1 通知Map内存泄漏风险

**位置**: `FlightLockDisposer.noticeMap`

```java
Map<String, List<PreNoticeEntity>> noticeMap = new HashMap<String, List<PreNoticeEntity>>();
```

**风险**:
- 员工查询完通知后，如果不再查询，`noticeMap` 中的列表不会被清理
- 长时间运行可能导致内存泄漏

**建议**:
- 添加清理机制
- 使用WeakHashMap
- 设置过期时间

#### 8.1.2 并发安全问题

**位置**: `ServiceManager.subscribe()`

```java
// 多线程环境下可能存在问题
List<PreNoticeEntity> lists = noticeMap.get(employeeNumber);
lists.remove(lists.size() - 1);
```

**风险**: 
- `noticeMap` 和 `lists` 的操作没有同步控制
- 可能导致ConcurrentModificationException

**建议**: 添加synchronized或使用线程安全集合

### 8.2 中等风险问题

#### 8.2.1 远程调用异常处理

**位置**: 多处Dubbo调用

```java
AgdisTaskVo agdisTaskVo = serviceBz.getTaskByFkeyRtypUsid(tenantId, flseq, serviceType, employeeNumber);
// 无try-catch保护
```

**风险**: 
- 远程调用失败可能导致整个流程中断
- 没有重试机制

**建议**: 添加try-catch和重试逻辑

#### 8.2.2 时间范围计算精度

**位置**: `CommonUtil.isInPreviousTimeRange()`

```java
// 使用float计算时间范围，可能存在精度问题
float timeRange = flightLockEntity.getTimeRange();
```

**建议**: 使用long或BigDecimal处理时间

### 8.3 低风险问题

#### 8.3.1 注释代码清理

**位置**: `OilService`、`FlightTaskService` 等

```java
// 大量注释掉的代码
// public void process(Exchange exchange) throws Exception {
//     // ...
// }
```

**建议**: 定期清理无用代码

#### 8.3.2 日志输出

**位置**: 多处使用 `logger.info()`

```java
// 部分日志可能泄露敏感信息
logger.info("员工【" + employeeNumber + "】不在线...");
```

**建议**: 脱敏处理敏感信息

---

## 9. 数据库表设计

### 9.1 flight_lock 表

```sql
CREATE TABLE flight_lock (
    id VARCHAR(32) PRIMARY KEY,
    employee_number VARCHAR(20),      -- 员工号
    service_type VARCHAR(10),         -- 服务类型
    process_type VARCHAR(1),          -- 0:锁定, 1:订阅
    lock_type VARCHAR(10),            -- 锁定类型
    lock_value VARCHAR(50),           -- 锁定值
    reference_time VARCHAR(20),       -- 参考时间字段
    time_range FLOAT,                 -- 时间范围(小时)
    time_range_type VARCHAR(1),       -- 0:往前, 1:往后, 2:前后
    lost_time VARCHAR(20),            -- 失效时间
    tenant_id VARCHAR(20),            -- 租户ID
    create_time VARCHAR(20),          -- 创建时间
    qualified_flag VARCHAR(1)         -- 条件满足标志
);
```

### 9.2 pre_notice 表

```sql
CREATE TABLE pre_notice (
    id VARCHAR(32) PRIMARY KEY,
    pnid VARCHAR(32),                 -- 通知ID
    ntyp VARCHAR(10),                 -- 通知类型
    fkey VARCHAR(32),                 -- 航班唯一号
    flop VARCHAR(8),                  -- 航班日
    flno VARCHAR(10),                 -- 航班号
    adid VARCHAR(1),                  -- 进离港
    stat VARCHAR(10),                 -- OPEN/CLOSED
    data TEXT,                        -- 通知数据(JSON)
    luts BIGINT,                      -- 时间戳
    tenant_id VARCHAR(20)             -- 租户ID
);
```

---

## 10. 配置参数

### 10.1 核心配置项

| 配置项 | 说明 | 默认值 |
|-------|------|-------|
| resourcesub.timeRange | 订阅时间范围 | - |
| timeRangeType | 时间范围类型 | - |
| referenceTime | 参考时间字段 | - |
| flight.timeRangeConfig | 自检时间范围 | - |

### 10.2 队列配置

| 队列名称 | 功能 |
|---------|------|
| seda:pnas-down | 通知下发 |
| seda:pnqs-down | 通知查询应答 |
| seda:pnle-down | 通知失效 |
| seda:lcqs-down | 订阅条件查询应答 |
| seda:lcqf-down | 订阅条件查询失败 |
| seda:slrc-down | 订阅/锁定条件应答 |

---

## 11. 改进建议

### 11.1 性能优化

1. **批量处理**
   - 航班自检时批量查询
   - 通知批量下发

2. **缓存优化**
   - 订阅条件内存缓存
   - 员工在线状态缓存

3. **异步处理**
   - 通知下发异步化
   - 远程调用异步化

### 11.2 可靠性提升

1. **事务控制**
   - 订阅条件保存添加事务
   - 通知创建添加事务

2. **重试机制**
   - 远程调用重试
   - 消息下发重试

3. **死信队列**
   - 消息发送失败进入死信队列
   - 定时处理死信消息

### 11.3 安全性增强

1. **消息加密**
   - XML/JSON消息加密传输
   - 敏感数据脱敏

2. **权限控制**
   - 订阅条件权限验证
   - 通知下发权限验证

---

## 附录

### A. 消息类型汇总表

| 消息类型 | 方向 | 功能 |
|---------|------|------|
| RSDY | 上行 | 上传订阅/锁定条件 |
| DYIN | 上行 | 查询订阅条件 |
| DYRP | 上行 | 查询通知 |
| SLRC | 下行 | 订阅/锁定条件应答 |
| LCQS | 下行 | 订阅条件查询应答 |
| LCTF | 下行 | 订阅条件查询失败 |
| PNQS | 下行 | 通知查询应答 |
| PNLE | 下行 | 通知失效 |
| PNAS | 下行 | 通知主动下发 |

### B. 错误码说明

| 错误码 | 含义 |
|-------|------|
| 400 | 没有符合要求的订阅条件 |
| STS=F | 处理失败 |
| STS=S | 处理成功 |

---

> **报告完成**  
> 如有问题或需要更详细的分析，请联系技术团队。
