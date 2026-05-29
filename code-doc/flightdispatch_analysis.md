# Flightdispatch 模块详细分析报告

> 分析时间: 2026-02-03  
> 分析范围: flightdispatch/src/test/java/flightdispatch/  
> 用途: 航班数据对接功能分析、Bug排查参考

---

## 1. 项目概述

### 1.1 基本信息

| 属性 | 值 |
|------|-----|
| 模块名称 | flightdispatch |
| 模块功能 | 航班数据分发 |
| 版本 | 5.0.15.4-pvg |
| 父级POM | flightparent (5.0.0.1) |
| Java版本 | 1.7 |
| Spring版本 | 4.3.3.RELEASE |
| Spring Boot版本 | 1.3.5.RELEASE |

### 1.2 核心职责

flightdispatch 模块是整个航油系统的**航班数据分发中心**，主要负责：
1. 接收并处理外部系统推送的航班数据
2. 航班数据转换与标准化
3. 联程航班匹配与关联
4. 航班数据推送至ActiveMQ消息队列
5. 机位信息管理
6. 航班变更记录追踪

### 1.3 依赖关系

**外部依赖**:
- Spring Framework 4.3.3
- Apache Camel 2.17.0 (路由/集成)
- ActiveMQ 5.12.0 (消息队列)
- Spring Data JPA 1.10.2
- MyBatis 3.x
- Dubbo (RPC框架)

**内部依赖**:
- flight-common (航班公共模块 v5.1.2.5-pvg)
- security (安全模块 v5.0.0.1)
- misgws (MIS网关模块 pvg-5.0.3.0)

---

## 2. 核心实体模型分析

### 2.1 FlightStroeData (航班存储数据)

**功能**: 核心航班数据实体，包含航班的所有动态信息

```java
// 主要字段分类
├── 基础信息
│   ├── flseq      // 航班唯一标识 (UUID)
│   ├── flop       // 航班日期 (yyyyMMdd格式)
│   ├── flno       // 航班号
│   ├── adid       // 进离港标识 (A=进港, D=离港)
│   └── ftyp       // 航班状态 (S=正常, X=取消, Y=延误)
│
├── 航班信息
│   ├── flti       // 国内/国际标志 (D/I/M/R)
│   ├── al2c       // 航空公司2码
│   ├── alcname    // 航空公司名称
│   ├── ac3c       // 机型3码
│   ├── acname     // 机型名称
│   └── regnumber  // 飞机注册号
│
├── 时间信息
│   ├── stot       // 计划时间
│   ├── etot       // 变更时间
│   ├── atot       // 实际时间
│   ├── osod       // 前站计划起飞
│   ├── oetd       // 前站变更起飞
│   └── otkf       // 前站实际起飞
│
├── 位置信息
│   ├── org3/-orgnm // 始发站
│   ├── des3/desnm  // 目的站
│   └── placecode   // 机位
│
└── 联程航班字段
    ├── isrl       // 是否联班 (Y/N)
    ├── rkey       // 关联航班标识
    └── rlop       // 联班航班日
```

### 2.2 ServiceFlightInfo (服务航班信息)

**功能**: 用于推送的航班数据结构，是FlightStroeData的简化版本

```java
// 字段数量: 40+ 字段
// 主要用途: ActiveMQ消息推送
// 与FlightStroeData的转换关系: 
//   MyBeanUtils.copyProperties() 实现属性拷贝
```

### 2.3 UnitedFlight (联程航班)

**功能**: 管理联程航班的关联关系

```java
// 主要字段
├── afleq         // 进港航班flseq
├── dfleq         // 出港航班flseq
├── aflop         // 进港航班日
├── dflop         // 出港航班日
└── tenant        // 租户标识
```

### 2.4 Place (机位实体)

**功能**: 管理机位信息

```java
// 主要字段
├── placecode     // 机位编号
├── ap3c          // 机场三字码
├── flno          // 关联航班号
├── flop          // 航班日
├── status        // 机位状态 (seating/occupied等)
└── tenant        // 租户
```

### 2.5 其他实体

| 实体名 | 功能说明 |
|--------|---------|
| FlightUpdateField | 航班更新字段记录 |
| FlseqInterfaceTenant | Flseq接口租户关联 |
| UnitedFlightMessage | 联程航班消息 |
| UnitedFlightChange | 联程航班变更 |
| AgdisFlight | AGDIS格式航班数据 |

---

## 3. 核心业务逻辑分析

### 3.1 航班数据推送流程

```
┌─────────────────────────────────────────────────────────────────────┐
│                     航班数据推送主流程                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. 获取航班数据                                                    │
│     ├── FlightStroeDataService.getByFlseqAndTenant()               │
│     └── FlightStroeDataService.getFlightsByTenant()                │
│                     │                                               │
│                     ▼                                               │
│  2. 数据对象转换                                                    │
│     ├── MyBeanUtils.copyProperties()  // Bean拷贝                   │
│     └── ReflectionUtil.beanToMap()    // 转换为Map                  │
│                     │                                               │
│                     ▼                                               │
│  3. 联程航班处理 (仅test004涉及)                                     │
│     ├── 判断进港/离港航班                                            │
│     ├── 查询关联UnitedFlight                                        │
│     ├── 设置isrl=Y (联班标识)                                       │
│     ├── 设置rkey/rlop (关联信息)                                    │
│     └── 设置preFlightActualLandingTime                              │
│                     │                                               │
│                     ▼                                               │
│  4. 消息发送                                                        │
│     ├── JmsTemplate.send()                                          │
│     └── createObjectMessage()  // 序列化发送                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2 联程航班处理逻辑 (test004)

**代码位置**: `FlightMessageTest.test004()`

```java
// 进港航班处理
if (StringUtils.equalsIgnoreCase("A", service.getAdid())) {
    // 查询对应的离港航班
    UnitedFlight uni = unitedFlightService.getByAfleqAndTenantId(
        service.getFlseq(), service.getTenant());
    
    if (uni != null) {
        // 设置联班标识
        service.setIsrl("Y");
        service.setRkey(flightStroeData.getFlseq());
        service.setRlop(flightStroeData.getFlop());
        service.setPreFlightActualLandingTime(service.getAtot());
        
        // 同时推送离港航班信息
        ServiceFlightInfo serviceD = ...;
        serviceD.setIsrl("Y");
        serviceD.setRkey(service.getFlseq());
        serviceD.setRlop(service.getFlop());
    }
}

// 离港航班处理 (类似逻辑)
if (StringUtils.equalsIgnoreCase("D", service.getAdid())) {
    // 查询对应的进港航班
    // 设置联班标识
    // 设置前序航班落地时间
}
```

### 3.3 航班查询逻辑

**代码位置**: `FlightServiceTest`

```java
// 1. 按航班日查询
flightStroeDataService.getByGreaterThanFlopAndTenant(flop, tenants)

// 2. 按机位查询
flightStroeDataService.getByFlopAndTenantAndPlacecode(flop, tenant, placecode)

// 3. 按航班号查询
flightStroeDataService.getByFlopAndTenantAndPlacecodeAndDelayFlag()

// 4. 特殊条件查询
flightStroeDataService.getFlightAByFlopAndCtotAndRegnumberAndTenant()
```

### 3.4 航班游戏标识 (FlightGame)

```java
// 功能: 标记特殊航班
game.setAdid("A");
game.setFlno("aa11111");
game.setFlop("20220105");
game.setFlseq("123123123");
game.setGame("Y");  // 游戏标识

// 通过FlightGameProducer发送到ActiveMQ
flightGameProducer.sendFlightGameMessage(game);
```

---

## 4. 测试用例详细分析

### 4.1 FlightMessageTest (航班消息测试)

| 测试方法 | 功能 | 关键逻辑 | 风险点 |
|----------|------|---------|--------|
| test001 | 单航班推送 | 查询2个航班 → 转换为Map → 发送MQ | 无 |
| test002 | 联程航班推送 | 设置isrl=Y, rkey, rlop关联 | ⚠️ 关联字段设置 |
| test003 | 全量推送 | 遍历所有航班 → 逐条发送 | ⚠️ 性能问题(循环发送) |
| test004 | 联程航班完整流程 | 进港+离港配对推送 | ⚠️ 业务逻辑复杂 |

### 4.2 FlightServiceTest (航班服务测试)

| 测试方法 | 功能 | 关键逻辑 | 风险点 |
|----------|------|---------|--------|
| test001 | 航班日计算 | 根据小时h计算flop | 无 |
| test002 | 延误航班查询 | 查询条件组合 | 无 |
| test003 | 特殊航班查询 | 多条件精确匹配 | 无 |
| test005 | FlightGame发送 | 发送游戏标识 | ⚠️ 字段有效性 |
| text004 | 数据转换 | FlightStroeData → AgdisFlight | ⚠️ 属性拷贝完整性 |

### 4.3 OtherTest (其他功能测试)

| 测试方法 | 功能 | 涉及类 | 风险点 |
|----------|------|--------|--------|
| test001 | Bean属性拷贝 | MyBeanUtils.copyProperties() | ⚠️ 源对象覆盖目标对象 |
| test002 | 反射调用 | ReflectionUtil.invokeGetterMethod() | 无 |
| test003 | 属性排除 | BeanUtils.copyProperties(target, source, strs) | 无 |
| test004 | 字段名获取 | CommonUtil.getFieldNames() | 无 |
| test007 | 变更记录 | CommonUtil.changedInfo() | 无 |
| test015 | 字段空值处理 | CommonUtil.transferFlightFieldNull() | 无 |
| test017 | JSON序列化 | FastJsonUtil.toJsonString() | 无 |
| test020 | 配置解析 | FastJsonUtil.convertJsonToMap() | ⚠️ JSON格式解析 |
| test023 | 进港配置解析 | IntoPortCommon | 无 |
| test024 | Bean拷贝测试 | MyBeanUtils.copyProperties() | ⚠️ 覆盖问题 |

### 4.4 PlaceServiceTest (机位服务测试)

| 测试方法 | 功能 | 风险点 |
|----------|------|--------|
| test001 | 机位更新 | 无 |
| test002 | FlseqInterfaceTenant保存 | 无 |
| test003 | FlseqInterfaceTenant保存(另一个) | 无 |
| test004 | 机位状态查询 | 无 |

---

## 5. 核心服务接口分析

### 5.1 FlightStroeDataService

| 方法名 | 功能 | 返回类型 |
|--------|------|---------|
| getByFlseqAndTenant | 按flseq查询 | FlightStroeData |
| getFlightsByTenant | 按租户查询 | List<FlightStroeData> |
| getByGreaterThanFlopAndTenant | 按航班日查询 | Map<String, List<FlightStroeData>> |
| getByFlopAndTenantAndPlacecode | 按机位查询 | List<FlightStroeData> |
| getByFlopAndTenantAndPlacecodeAndDelayFlag | 延误航班查询 | List<FlightStroeData> |
| getFlightAByFlopAndCtotAndRegnumberAndTenant | 特殊条件查询 | FlightStroeData |

### 5.2 UnitedFlightService

```java
// 联程航班服务
UnitedFlight getByAfleqAndTenantId(String afleq, String tenant);  // 按进港flseq查询
UnitedFlight getByDfleqAndTenantId(String dfleq, String tenant);  // 按离港flseq查询
```

### 5.3 PlaceService

```java
Place getByAp3cAndPlaceCodeAndTenant(String ap3c, String placeCode, String tenant);
void update(Place place);
```

### 5.4 其他服务

| 服务名 | 功能 |
|--------|------|
| FlightUpdateFieldService | 航班更新字段服务 |
| FlseqInterfaceTenantService | Flseq租户关联服务 |
| UnitedFlightMessageService | 联程消息服务 |

---

## 6. 工具类分析

### 6.1 MyBeanUtils

**功能**: Bean属性拷贝

```java
// 内部实现可能调用Spring的BeanUtils
// 问题: 源对象会覆盖目标对象的同名属性
MyBeanUtils.copyProperties(source, target);  // target被source覆盖
```

**风险点**: 
- 如果需要保留目标对象的某些字段，需要先备份
- 建议使用BeanUtils.copyProperties(source, target, ignoreProperties)

### 6.2 ReflectionUtil

**功能**: 反射工具

```java
// 主要方法
Map<String, String> beanToMap(Object bean);       // Bean转Map
Object invokeGetterMethod(Object obj, String field); // 调用getter方法
```

### 6.3 FastJsonUtil

**功能**: JSON序列化/反序列化

```java
// 主要方法
String toJsonString(Object obj);                          // 对象转JSON
Map<String, UnitFlightRule> convertJsonToMap(String json); // JSON转Map
Map<String, IntoPortCommon> convertIntoPortCommonJsonToMap(String json);
```

### 6.4 CommonUtil

**功能**: 通用工具

```java
// 主要方法
Map<String, Object> changedRecord(Object source, Object target);  // 变更记录
Map<String, Object> changedInfo(Object source, Object target);    // 变更信息
List<String> getFieldNames(Object obj);                            // 获取字段名
FlightStroeData transferFlightFieldNull(FlightStroeData flight);  // 空值处理
```

---

## 7. 潜在Bug风险点分析

### 7.1 高风险问题

#### 7.1.1 Bean属性覆盖问题

**位置**: `MyBeanUtils.copyProperties(source, target)`

```java
// 问题代码 (OtherTest.test024)
FlightStroeData flight1 = new FlightStroeData();
flight1.setDes3("345");
FlightStroeData flight2 = new FlightStroeData();
flight2.setDes3("");
MyBeanUtils.copyProperties(flight2, flight1);  // flight1的Des3被覆盖为空!

System.out.println("flight1 des3: "+flight1.getDes3());  // 输出空字符串
```

**影响**: 重要数据被空值覆盖，导致数据丢失

**修复建议**:
```java
// 方案1: 使用Spring BeanUtils的排除功能
String[] ignoreProperties = {"des3", "otherField"};
BeanUtils.copyProperties(source, target, ignoreProperties);

// 方案2: 手动设置非空值
if (StringUtils.isNotBlank(source.getDes3())) {
    target.setDes3(source.getDes3());
}
```

#### 7.1.2 循环发送消息性能问题

**位置**: `FlightMessageTest.test003()`

```java
// 问题代码
for(FlightStroeData flight:flights){
    List<Map<String, String>> targetPushList = new ArrayList<>();
    // ... 转换逻辑
    this.sendServiceFlightMessage(targetPushList);  // 每条消息单独发送
}
```

**影响**: 
- 大量航班时会导致MQ压力过大
- 消息发送时间长 (test003中打印了时间间隔)

**修复建议**:
```java
// 批量发送
List<Map<String, String>> allFlights = new ArrayList<>();
for(FlightStroeData flight:flights){
    // 转换后添加到列表
    allFlights.add(map);
}
// 批量发送
this.sendServiceFlightMessage(allFlights);
```

#### 7.1.3 联程航班关联字段设置错误

**位置**: `FlightMessageTest.test002()` 和 `test004()`

```java
// 问题: rlop设置的是对方的flop，但字段名暗示是"航班日"
// 实际上rlop应该也是flseq (航班唯一标识)

service1.setIsrl("Y");
service1.setRkey("a3227d123d28446c84f784fa5246cd90");  // 这个是flseq
service1.setRlop(service2.getFlop());  // 这个是flop，但字段名是rlop
```

**建议**: 检查 UnitedFlight 实体中的 rlop 字段定义，确认语义是否正确

### 7.2 中等风险问题

#### 7.2.1 时间计算逻辑重复

**位置**: 多个测试类中的 `calculateFlop()` 方法

```java
// FlightServiceTest.java 和 OtherTest.java 都有相同的方法
public String calculateFlop(String h) {
    // 相同的计算逻辑
}
```

**建议**: 提取到公共工具类

#### 7.2.2 JSON配置解析容错

**位置**: `OtherTest.test020`

```java
String json = "{\"OIL_PVG\":{\"al2c\":\"MU|FM|KN\"}}";
Map<String, UnitFlightRule> map = FastJsonUtil.convertJsonToMap(json);
```

**风险**: 如果JSON格式错误，会抛出解析异常

**建议**: 添加JSON格式校验和异常处理

#### 7.2.3 航班状态硬编码

**位置**: `FlightServiceTest.test005`

```java
game.setAdid("A");
game.setFtyp("S");  // 硬编码
```

**建议**: 使用枚举或常量类定义航班状态

### 7.3 低风险问题

#### 7.3.1 调试代码未清理

**位置**: `OtherTest.test009`

```java
@Test
public void test009(){
    FlightStroeData flight = DefaultFieldUtil.getDefaultField("OIL_PVG");
    System.out.println("123");  // 无意义输出
}
```

#### 7.3.2 注释代码未清理

**位置**: `OtherTest.test021`

```java
//@Test
/*public void test021(){
    // 注释掉的代码
}*/
```

---

## 8. 数据流转分析

### 8.1 航班数据流转图

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  外部系统    │────▶│ flightdispatch│────▶│  ActiveMQ   │
│  (ESB/其他)  │     │    模块       │     │   消息队列   │
└─────────────┘     └─────────────┘     └─────────────┘
                           │
                           ▼
                    ┌─────────────┐
                    │   MySQL     │
                    │   数据库     │
                    └─────────────┘
```

### 8.2 消息推送格式

**消息类型**: ObjectMessage (Serializable)

**消息内容**:
```java
List<Map<String, String>> targetPushList = [
    {
        "flseq": "a1f0551b2cc14f6c98abcf1778854ccb",
        "flno": "MU1234",
        "flop": "20220105",
        "adid": "D",
        // ... 其他字段
    }
]
```

### 8.3 联程航班数据关联

```
┌─────────────────────────────────────────────────────────────┐
│                    联程航班关联关系                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  进港航班 (A)              联程关系              出港航班 (D) │
│  ┌─────────────┐        ┌─────────────┐       ┌─────────────┐│
│  │ flseq: ABC  │───────▶│ unitedFlight│◀──────│ flseq: XYZ  ││
│  │ flop: 0105  │        │ afleq: ABC  │       │ flop: 0105  ││
│  │ atot: 1000  │        │ dfleq: XYZ  │       │             ││
│  └─────────────┘        └─────────────┘       └─────────────┘│
│         │                                            │        │
│         │         ┌──────────────────────┐          │        │
│         └────────▶│ isrl: Y              │◀─────────┘        │
│                   │ rkey: XYZ            │                   │
│                   │ rlop: 0105           │                   │
│                   │ preFlightActual      │                   │
│                   │   LandingTime: 1000  │                   │
│                   └──────────────────────┘                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 9. 配置依赖分析

### 9.1 Spring配置文件

| 配置文件 | 用途 |
|---------|------|
| applicationContext.xml | Spring核心配置 |
| applicationContext_camel.xml | Camel路由配置 |
| applicationContext_dubbo.xml | Dubbo RPC配置 |
| applicationContext_mq.xml | ActiveMQ配置 |

### 9.2 关键Bean依赖

```java
// ActiveMQ
@Resource(name="serviceActiveMqJmsTemplate")
private JmsTemplate serviceActiveMqJmsTemplate;

// 航班数据服务
@Resource
private FlightStroeDataService flightStroeDataService;

// 联程航班服务
@Resource(name="unitedFlightServiceImpl")
private UnitedFlightService unitedFlightService;

// FlightGame Producer
@Resource(name="flightGameProducer")
private FlightGameProducer flightGameProducer;
```

---

## 10. 测试代码统计

### 10.1 测试文件清单

| 文件名 | 行数 | 测试方法数 | 主要功能 |
|--------|------|-----------|---------|
| FlightMessageTest.java | 208 | 4 | 航班消息推送 |
| FlightServiceTest.java | 145 | 5 | 航班服务测试 |
| OtherTest.java | 641 | 30+ | 杂项功能测试 |
| PlaceServiceTest.java | 90 | 4 | 机位服务测试 |
| ServiceFlightInfo.java | 402 | - | 实体类定义 |
| UnitedMessageTest.java | 38 | 1 | 联程消息测试 |
| FlightUpdateFieldServiceTest.java | 34 | 1 | 更新字段测试 |
| testttt.java | 33 | 1 | 用户服务测试 |
| ParseXmlTest.java | 24 | 1 | XML解析测试 |

### 10.2 测试覆盖分析

| 功能模块 | 测试覆盖 | 完整性 |
|---------|---------|--------|
| 航班数据推送 | ✅ 已覆盖 | 完整 |
| 联程航班处理 | ✅ 已覆盖 | 完整 |
| 航班查询 | ✅ 已覆盖 | 完整 |
| Bean转换 | ✅ 已覆盖 | 完整 |
| JSON序列化 | ✅ 已覆盖 | 完整 |
| MQ消息发送 | ✅ 已覆盖 | 完整 |
| 异常处理 | ❌ 未覆盖 | 不完整 |
| 边界条件 | ⚠️ 部分覆盖 | 不完整 |

---

## 11. Bug排查指南

### 11.1 常见问题与解决方案

| 问题现象 | 可能原因 | 排查方向 |
|---------|---------|---------|
| 航班数据未推送 | MQ连接异常 | 检查ActiveMQ配置 |
| 联程航班未关联 | UnitedFlight表数据缺失 | 检查航班匹配逻辑 |
| 数据被覆盖 | MyBeanUtils.copyProperties使用不当 | 检查属性拷贝代码 |
| 推送延迟 | 循环发送消息 | 检查批量发送逻辑 |
| JSON解析失败 | 配置格式错误 | 检查JSON格式 |

### 11.2 关键日志位置

```
// 消息发送日志
flightGameProducer.sendFlightGameMessage(game);

// 航班处理日志
flightStroeDataService.getByFlseqAndTenant();

// 联程处理日志
unitedFlightService.getByAfleqAndTenantId();
```

### 11.3 数据库表检查

| 表名 | 用途 |
|------|------|
| flight_stroe_data | 航班数据表 |
| united_flight | 联程航班表 |
| place | 机位表 |
| flseq_interface_tenant | Flseq租户关联表 |

---

## 12. 改进建议

### 12.1 代码规范改进

1. **统一工具类**: 抽取重复的 `calculateFlop()` 方法到公共模块
2. **使用枚举**: 航班状态、进离港标识使用枚举类型
3. **清理代码**: 删除注释掉的代码和无效的调试输出
4. **完善注释**: 为复杂业务逻辑添加注释

### 12.2 性能优化

1. **批量发送**: 改进test003的循环发送为批量发送
2. **缓存机制**: 对频繁查询的数据添加缓存
3. **索引优化**: 检查数据库表索引是否合理

### 12.3 测试改进

1. **异常测试**: 添加异常场景的测试用例
2. **边界测试**: 添加边界条件的测试用例
3. **Mock测试**: 使用Mock减少对外部依赖的依赖

### 12.4 架构改进

1. **接口标准化**: 统一服务接口定义
2. **配置外置**: 将硬编码的配置提取到配置文件
3. **日志规范**: 统一日志格式和级别

---

## 附录

### A. 测试方法速查

| 测试类 | 测试方法 | 功能 |
|--------|---------|------|
| FlightMessageTest | test001 | 基础航班推送 |
| FlightMessageTest | test002 | 联程航班推送 |
| FlightMessageTest | test003 | 全量推送 |
| FlightMessageTest | test004 | 完整联程流程 |
| FlightServiceTest | test005 | FlightGame发送 |
| PlaceServiceTest | test001 | 机位更新 |
| PlaceServiceTest | test004 | 机位查询 |

### B. 关键配置项

```properties
# ActiveMQ配置
activemq.broker-url=tcp://localhost:61616
activemq.user=admin
activemq.password=admin

# 数据库配置
jdbc.url=jdbc:mysql://localhost:3306/db_oil_pvg
jdbc.username=root
jdbc.password=123456

# Dubbo配置
dubbo.application.name=flightdispatch
dubbo.registry.address=zookeeper://localhost:2181
```

### C. 相关文档链接

- [项目主文档](../readme.md)
- [数据库设计文档](../sql/db_oil_pvg.sql)

---

## 13. 航班数据来源分析

### 13.1 数据来源总览

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           航班数据来源架构                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐  │
│  │  外部系统   │    │  WebService │    │  Camel路由  │    │  定时任务   │  │
│  │  (ESB/其他) │    │    接口     │    │   消息队列  │    │  主动拉取  │  │
│  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘    └──────┬──────┘  │
│         │                 │                 │                 │           │
│         └─────────────────┼─────────────────┼─────────────────┘           │
│                           │                 │                             │
│                           ▼                 ▼                             │
│                 ┌─────────────────────────────────────┐                   │
│                 │      ActiveMQ 消息队列               │                   │
│                 │      (seda:fltmessagedeal)           │                   │
│                 └──────────────────┬──────────────────┘                   │
│                                    │                                        │
│                                    ▼                                        │
│                 ┌─────────────────────────────────────┐                   │
│                 │  FlightDealCommon.dealFlight()      │                   │
│                 │  (航班数据处理核心入口)               │                   │
│                 └──────────────────┬──────────────────┘                   │
│                                    │                                        │
│         ┌─────────────────────────┼─────────────────────────┐              │
│         │                         │                         │              │
│         ▼                         ▼                         ▼              │
│  ┌─────────────┐          ┌─────────────┐          ┌─────────────┐        │
│  │  数据存储   │          │  联程计算   │          │  消息推送   │        │
│  │FlightStroe  │          │UnitedFlight │          │  各终端系统 │        │
│  │   Data      │          │             │          │             │        │
│  └─────────────┘          └─────────────┘          └─────────────┘        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### 13.2 消息推送 (ActiveMQ/JMS) - **主要来源**

#### 入口文件：`FlightDealCommon.java`

```java
@Override
public void dealFlight(Exchange exchange) {

    Object body = exchange.getIn().getBody();
    logger.info("收到外部系统的航班消息：" + body);
    if (body instanceof FlightStroeData) {
        processFlight((FlightStroeData) body);
    } else if (body instanceof FlightGame) {
        processFlightGame((FlightGame) body);
    }
}
```

#### 消息类型：

| 消息类型 | 处理方法 | 说明 |
|---------|---------|------|
| `FlightStroeData` | `processFlight()` | 完整航班数据 |
| `FlightGame` | `processFlightGame()` | 航班游戏标识 |

#### Camel路由配置：

```java
// 路由端点
camelTemplate.sendBody("seda:fltmessagedeal", fgosMessageSendVO);
camelTemplate.sendBody("seda:luggagemessagedeal", fgosMessageSendVO);
camelTemplate.sendBody("seda:dispatch", flightCommons);
camelTemplate.sendBody("seda:recombinationUnitedFlight", recombinations);
```

---

### 13.3 WebService 接口 - **外部系统对接**

#### 入口文件：`WorkingMsgDisposer.java`

```java
public void dealWorkingMsg(Exchange exchange){

    FgosMessageSendVO fgosMessageSendVO = (FgosMessageSendVO)exchange.getIn().getBody();

    if(fgosMessageSendVO != null){

        fgosMessageSendVO.setTenant(messageTenant);

        // 通过原始flseq查找租户关联
        FlseqInterfaceTenant flseqInterfaceTenant =
            flseqInterfaceTenantService.getFlseqByMssourceflseqAndTenant(
                fgosMessageSendVO.getFlightKey(),
                fgosMessageSendVO.getTenant());

        if(flseqInterfaceTenant != null){

            fgosMessageSendVO.setFlightKey(flseqInterfaceTenant.getFlseq());

            // 发送到Camel路由
            camelTemplate.sendBody("seda:fltmessagedeal",fgosMessageSendVO);
        }
    }
}
```

#### 依赖的WebService接口：

| 接口 | 功能 | 包路径 |
|-----|------|-------|
| `ClientMsgEndpoint` | FGOS消息发送 | `com.ias.agdis.message.webservice.endpoint` |
| `PositionService` | 岗位信息查询 | `com.misgws.service` |
| `MsgSendRuleService` | 消息发送规则 | `com.missws.service` |

---

### 13.4 定时任务 - **主动拉取/计算**

#### 定时任务清单：

| 任务类 | 功能 | 触发方式 |
|-------|------|---------|
| `CalculateUniteDeal` | 联程航班计算 | 定时执行 |
| `CalculateGameDeal` | 航班游戏标识计算 | 定时执行 |
| `CalculateRegisalineJob` | 机号计算 | 定时执行 |
| `CalculateSeatFlightCalJob` | 机位航班计算 | 定时执行 |
| `CalculateUnitedFlightMessageJob` | 联程消息计算 | 定时执行 |

#### 联程航班计算示例 (`CalculateUniteDeal.java`)：

```java
public void calculateUnitedFlight(){

    logger.info("calculateUnitedFlight is start ...");

    try {
        this.unitedFlightDeal();

    } catch (Exception e) {
        // 异常处理
    }

    logger.info("calculateUnitedFlight is end ...");
}

// 核心计算逻辑
public void unitedFlightDeal(){
    // 1. 获取昨日所有航班
    // 2. 匹配进港/离港航班
    // 3. 建立联程关联
    // 4. 发送联程变更消息
}
```

---

### 13.5 航班数据处理主流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        航班数据处理主流程                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. 消息接收                                                                 │
│     ├── ActiveMQ/JMS 消息                                                   │
│     ├── WebService 调用                                                     │
│     └── 定时任务触发                                                         │
│                    │                                                        │
│                    ▼                                                        │
│  2. 数据校验与转换                                                           │
│     ├── 时间格式转换 (yyyyMMddHHmmss → yyyy-MM-dd HH:mm:ss)                 │
│     ├── 机场三字码标准化                                                     │
│     ├── 航空公司信息补全                                                     │
│     └── 机位信息关联                                                         │
│                    │                                                        │
│                    ▼                                                        │
│  3. Flseq唯一标识生成                                                        │
│     ├── 新航班: 生成UUID作为flseq                                           │
│     ├── 已存在航班: 复用原有flseq                                           │
│     └── 航班变更: 生成新flseq，旧航班标记为取消                              │
│                    │                                                        │
│                    ▼                                                        │
│  4. 数据存储                                                                 │
│     ├── INSERT: 新航班插入                                                   │
│     ├── UPDATE: 已存在航班更新                                               │
│     └── 变更记录: 保存到flight_change表                                      │
│                    │                                                        │
│                    ▼                                                        │
│  5. 联程航班处理                                                             │
│     ├── 匹配进港/离港航班                                                    │
│     ├── 计算预计加油时间                                                     │
│     └── 更新延误标识                                                         │
│                    │                                                        │
│                    ▼                                                        │
│  6. 消息推送                                                                 │
│     ├── AgdisFlightProducer → AGDIS系统                                     │
│     ├── ServiceFlightProducer → 服务终端                                     │
│     ├── TerminalFlightProducer → 终端设备                                    │
│     ├── OasisFlightProducer → OASIS系统                                      │
│     └── WebgisFlightProducer → WebGIS系统                                    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### 13.6 外部数据源标识

代码中识别到的外部数据源：

| 数据源标识 | 说明 | 代码位置 |
|-----------|------|---------|
| `WROCS_PVG` | 老系统航油数据 | `FlightDealCommon.java:98` |
| `EASTERN-AIRLINE-PVG` | 东航数据 | `FlightDealCommon.java:100` |
| 配置动态映射 | 通过 `BaseData.TENANTS` 映射 | `FlightDealCommon.java:183` |

---

### 13.7 数据来源总结

| 数据来源 | 占比 | 实时性 | 说明 |
|---------|------|-------|------|
| **ActiveMQ消息推送** | 高 | 实时 | 主要来源，外部系统主动推送 |
| **WebService接口** | 中 | 实时 | 外部系统通过接口调用 |
| **定时任务计算** | 低 | 定时 | 联程航班、延误计算等 |

**核心入口**：`FlightDealCommon.dealFlight()` 是所有航班数据的统一处理入口。

---

> **报告完成**
> 如有问题或需要更详细的分析，请联系技术团队。
