# Flight Tools 代码分析报告

## 1. 项目概述

### 1.1 项目基本信息

| 属性 | 值 |
|------|-----|
| 项目名称 | flight-tools |
| 包含模块 | common, commonparent/model |
| Java版本 | 1.7 |
| 源码编码 | UTF-8 |

### 1.2 项目定位

Flight Tools 是浦东机场航班信息系统的基础工具包模块，主要包含：

1. **公共实体类定义** - 航班数据核心模型
2. **数据模型转换** - 不同系统间的数据格式转换
3. **工具类** - 航班相关工具方法

### 1.3 模块结构

```
flight-tools/
├── common/                              # 航班数据实体模块
│   ├── pom.xml (v5.1.2.5-pvg)
│   └── src/main/java/com/ias/server/flight/common/
│       ├── model/                       # 实体类 (13个)
│       │   ├── FlightStroe.java         # 航班基础模型(父类)
│       │   ├── FlightStroeData.java     # 航班数据实体
│       │   ├── FlightChange.java        # 航班变更记录
│       │   ├── FlightSubscribe.java     # 航班订阅
│       │   ├── FlightUpdateField.java   # 航班更新字段
│       │   ├── FlightInfoChange.java    # 航班信息变更
│       │   ├── FlseqInterfaceTenant.java # 航班序列租户关联
│       │   ├── UnitedFlight.java        # 联程航班
│       │   ├── UnitedFlightChange.java  # 联程航班变更
│       │   ├── UnitedFlightMessage.java # 联程航班消息
│       │   ├── Place.java               # 机位信息
│       │   ├── IntoPort.java            # 进港信息
│       │   └── LongAirlineAirport.java  # 长航线机场
│       ├── until/                       # 工具类 (3个)
│       │   ├── FlightTools.java         # 航班工具
│       │   ├── ModelConvert.java        # 模型转换
│       │   └── Clone.java               # 克隆工具
│       └── extend/model/                # 扩展模型 (1个)
│           └── ImfMessage.java          # IMF消息
│
└── commonparent/                        # 父模块
    ├── pom.xml (v5.0.0.3)
    └── model/
        ├── pom.xml (v5.1.2.2-pvg)
        └── src/main/java/com/ias/common/
            ├── model/
            │   ├── base/
            │   │   └── RalationInterfaceTenant.java
            │   └── flight/               # 通用航班模型 (11个)
            │       ├── AgdisFlight.java      # AGDIS系统航班
            │       ├── Flight.java           # 基础航班模型
            │       ├── FlopData.java         # 航班日数据
            │       ├── ImFlight.java         # 即时消息航班
            │       ├── MobileAppFlight.java  # 移动端航班
            │       ├── ServiceFlight.java    # 服务航班
            │       ├── WebGisFlight.java     # WebGIS航班
            │       ├── UnionAgdisFlight.java # 联程AGDIS航班
            │       ├── UnionFlight.java      # 联程航班
            │       ├── UnionServiceFlight.java # 联程服务航班
            │       └── Flight.java           # 基础航班
            └── service/
                └── FlightBase.java          # 航班服务基类
```

---

## 2. 核心实体类详解

### 2.1 FlightStroe - 航班基础模型

**文件位置**: `common/src/main/java/com/ias/server/flight/common/model/FlightStroe.java`

**类继承关系**: `@MappedSuperclass`

**核心字段**:

| 字段名 | 类型 | 长度 | 说明 |
|--------|------|------|------|
| flseq | String | 50 | 航班信息序号 (唯一标识) |
| flop | String | 8 | 航班日期 (yyyyMMdd) |
| flno | String | 20 | 航班号 |
| adid | String | 1 | 进离港标志 (A=进港, D=离港) |
| regnumber | String | 20 | 飞机注册号 |
| frct | String | 1 | 航班重复标记 |
| ttab | String | 1 | 航班星期数 |
| flti | String | 1 | 国内/国际航班标志 (D/I/M/R) |
| ftyp | String | 1 | 航班状态 (X=取消, Y=延迟, S=正常) |
| ttyp | String | 2 | 航班飞行性质 |
| hbxz | String | 20 | 航班性质 |
| al2c | String | 2 | 航空公司2码 |
| alcname | String | 40 | 航空公司中文名称 |
| ac3c | String | 3 | 机型3码 |
| acname | String | 30 | 机型名称 |
| org3 | String | 3 | 始发站三码 |
| des3 | String | 3 | 目的站三码 |
| orgnm | String | 40 | 始发航站名称 |
| desnm | String | 40 | 目的航站名称 |

### 2.2 FlightStroeData - 航班数据实体

**文件位置**: `common/src/main/java/com/ias/server/flight/common/model/FlightStroeData.java`

**数据库表**: `flightinfo`

**继承**: extends `FlightStroe`

**扩展字段类别**:

#### 时间字段 (约50个)

| 字段前缀 | 说明 | 示例 |
|----------|------|------|
| stot | 计划到达/起飞时间 | 航班计划时间 |
| etot | 变更到达/起飞时间 | 延误后的时间 |
| atot | 实际到达/起飞时间 | 真实落地/起飞时间 |
| bost/boet | 登机开始/结束时间 | 登机口时间 |
| onbl/ofbl | 上/下轮档时间 | 飞机落地/起飞时间 |
| fdot/fdct | 开/关客舱门时间 | 上下客时间 |
| pushin/pushout | 推入/推出时间 | 飞机移动时间 |
| dpot/dpbt/doit/dcit | 值机/结柜时间 | 旅客服务时间 |
| jsto/jeto/jato | 接飞航班时间 | 衔接航班时间 |

#### 旅客人数字段 (约30个)

| 字段 | 说明 |
|------|------|
| f_pax | 头等舱人数 |
| c_pax | 商务舱人数 |
| y_pax | 经济舱人数 |
| adult | 成人人数 |
| enfant | 儿童人数 |
| infant | 婴儿人数 |
| vipnum | VIP人数 |
| sum_number | 总人数 |

#### 航班资源字段

| 字段 | 说明 |
|------|------|
| placecode | 机位 |
| gtname1/gtname2 | 登机口 (国内/国际) |
| blt1/blt2 | 行李转盘 |
| dpsn/ipsn | 分拣口 (国内/国际) |
| dcic/icic | 值机柜台号 (国内/国际) |
| areacode | 区域代码 |

#### 航班状态字段

| 字段 | 说明 |
|------|------|
| stat | 航班状态 (登机/客齐/落地/离地) |
| dltime | 延误时间 |
| dlreason | 延误原因 |
| crsn | 取消原因 |
| ctim | 取消时间 |

#### 机组信息字段

| 字段 | 说明 |
|------|------|
| captain | 机长 |
| pilot/fstcopilot/copilot | 副驾驶 |
| steward | 乘务员 |
| guard | 保卫员 |
| aircrewNum | 机组人数 |
| stewardNum | 乘务人数 |
| aircrewCompany | 机组所属单位 |

#### 加油信息字段

| 字段 | 说明 |
|------|------|
| oll | 起飞油量 |
| ol2 | 轮档加油量 |
| ola | 应加油量 |
| isCharging | 是否加油 |
| otat | 机组确认标记 |
| olvr | 油量发布版本 |

### 2.3 FlightChange - 航班变更记录

**数据库表**: `flight_change`

| 字段 | 类型 | 说明 |
|------|------|------|
| flseq | String | 航班唯一号 |
| item变更项名称 |
 | String | | value | String | 变更项值 |
| modifyDate | Date | 变更时间 |
| adid | String | 进离港 |
| flop | String | 航班日 |

### 2.4 FlightSubscribe - 航班订阅

**数据库表**: `flight_subscribe`

| 字段 | 类型 | 说明 |
|------|------|------|
| employId | String | 员工号 |
| flseq | String | 航班唯一号 |
| flno | String | 航班号 |
| subToken | String | 订阅标识 |
| subTime | Date | 订阅时间 |
| tenant | String | 租户ID |
| flop | String | 航班日 |

### 2.5 UnitedFlight - 联程航班

**数据库表**: `flight_united_flight`

| 字段 | 类型 | 说明 |
|------|------|------|
| afleq | String | 进航班唯一号 |
| dfleq | String | 出航班唯一号 |
| aflno | String | 进航班号 |
| dflno | String | 出航班号 |
| flop | String | 进港航班日 |
| dflop | String | 离港航班日 |
| tenant | String | 租户ID |

### 2.6 Place - 机位信息

**数据库表**: `place`

**枚举类型**: `PlaceStatus` (seating, entering)

| 字段 | 类型 | 说明 |
|------|------|------|
| ap3c | String | 机场三码 |
| placecode | String | 机位号 |
| flno | String | 航班号 |
| tenant | String | 租户ID |
| flop | String | 航班日 |
| status | PlaceStatus | 机位状态 |
| flseq | String | 航班唯一号 |

---

## 3. 通用数据模型 (commonparent/model)

### 3.1 AgdisFlight

**用途**: AGDIS系统航班数据传输对象

**特点**: 与 `FlightStroeData` 字段几乎一致，用于Dubbo服务返回

**扩展字段**:

| 字段 | 说明 |
|------|------|
| ctotdr | CTOT时间来源 (A/E/S) |
| abroad | 国际/国内/离境 |
| fpname | 油井 |
| delete_flag | 删除标识 |
| regaline | 机号航司对应 |
| crew_in_place | 机组到位时间 |
| caculate_ofbl | COBT时间 |
| actual_takeoff | DET时间 |
| far_gate_boarding_gate | 远机位登机口 |

### 3.2 MobileAppFlight

**用途**: 移动端航班数据展示

**字段命名风格**: 使用缩写

| 字段 | 全称 | 说明 |
|------|------|------|
| fke | flseq | 航班唯一号 |
| fln | flno | 航班号 |
| reg | regnumber | 机号 |
| acn | ac3c | 机型 |
| psn | placecode | 机位 |
| sto | stot | 计划时间 |
| ato | atot | 实际时间 |
| eto | etot | 变更时间 |
| gat | gtname1/2 | 登机口 |
| apb | dpbt | 上客时间 |
| apo | dpot | 客齐时间 |
| jst/jet/jat | jsto/jeto/jato | 接飞时间 |
| pui/puo | pushin/pushout | 推入/推出 |

### 3.3 其他模型类

| 类名 | 用途 |
|------|------|
| `Flight` | 基础航班模型 |
| `FlopData` | 航班日数据 |
| `ImFlight` | 即时消息航班 |
| `ServiceFlight` | 服务层航班 |
| `WebGisFlight` | WebGIS航班 |
| `UnionAgdisFlight` | 联程AGDIS航班 |
| `UnionFlight` | 联程航班 |
| `UnionServiceFlight` | 联程服务航班 |

---

## 4. 工具类分析

### 4.1 FlightTools

**功能**: 航班相关工具方法

```java
public class FlightTools {
    // 航班唯一号生成算法 (已注释)
    // public static String creatFlseq(String... strings)
}
```

### 4.2 ModelConvert

**功能**: 数据模型转换工具

```java
public class ModelConvert {
    // 已注释的方法:
    // convertFltToServiceFlight()
    // convertFltToAgdisFlight()
}
```

### 4.3 Clone

**功能**: 对象克隆工具

---

## 5. Maven依赖分析

### 5.1 common模块依赖

```xml
<dependencies>
    <!-- JPA API -->
    <dependency>
        <groupId>org.hibernate.javax.persistence</groupId>
        <artifactId>hibernate-jpa-2.0-api</artifactId>
        <version>1.0.1.Final</version>
    </dependency>
    
    <!-- Bean工具 -->
    <dependency>
        <groupId>commons-beanutils</groupId>
        <artifactId>commons-beanutils</artifactId>
        <version>1.8.3</version>
    </dependency>
</dependencies>
```

### 5.2 commonparent/model模块依赖

- 无外部依赖
- 仅定义POJO类

### 5.3 Maven仓库配置

```xml
<repository>
    <id>Maven Ias</id>
    <url>http://172.16.40.82:8088/nexus/content/groups/public</url>
</repository>
```

---

## 6. 数据字典

### 6.1 枚举值说明

| 字段 | 枚举值 | 说明 |
|------|--------|------|
| adid | A | 进港航班 |
| adid | D | 离港航班 |
| flti | D | 国内航班 |
| flti | I | 国际航班 |
| flti | M | 混合航班 |
| flti | R | 地区航班 |
| ftyp | X | 取消 |
| ftyp | Y | 延误 |
| ftyp | S | 正常 |
| fnflag | F | 远机位 |
| fnflag | N | 近机位 |
| ifbj | Y | 备降 |

### 6.2 航班状态说明

| 状态值 | 说明 |
|--------|------|
| 登机 | 旅客正在登机 |
| 客齐 | 旅客已登机完毕 |
| 落地 | 航班已到达 |
| 离地 | 航班已起飞 |

### 6.3 航班时间优先级 (CTOT计算)

| 优先级 | 字段 | 说明 |
|--------|------|------|
| 1 | atot | 实际时间 |
| 2 | etot | 变更时间 |
| 3 | stot | 计划时间 |

---

## 7. 代码质量评估

### 7.1 优点

1. **字段命名规范**: 采用驼峰命名 + 数据库列映射
2. **继承结构合理**: `FlightStroe` 作为父类，`FlightStroeData` 继承扩展
3. **多模型支持**: 为不同系统提供不同数据格式 (AgdisFlight, MobileAppFlight)
4. **枚举使用**: `PlaceStatus` 使用枚举定义状态
5. **JPA注解完善**: 使用 `@Table`, `@Column`, `@Id` 等注解

### 7.2 需改进项

1. **缺少文档注释**: 大部分类缺少JavaDoc注释
2. **工具类功能不完善**: `FlightTools`, `ModelConvert` 方法被注释掉
3. **字段类型混乱**: 时间字段统一使用String，建议使用Date/LocalDateTime
4. **命名不一致**: 
   - 部分使用驼峰 (flopStart)
   - 部分使用下划线 (stop_over_station)
5. **代码重复**: `AgdisFlight` 与 `FlightStroeData` 大量字段重复
6. **缺少校验注解**: 未使用 `@NotNull`, `@Size` 等校验注解

---

## 8. 数据库表结构映射

### 8.1 实体与表映射关系

| 实体类 | 数据库表 | 说明 |
|--------|----------|------|
| FlightStroeData | flightinfo | 航班主表 |
| FlightChange | flight_change | 航班变更记录 |
| FlightSubscribe | flight_subscribe | 航班订阅 |
| FlightUpdateField | flight_update_field | 更新字段追踪 |
| FlseqInterfaceTenant | flseq_interface_tenant | 序列租户关联 |
| UnitedFlight | flight_united_flight | 联程航班 |
| Place | place | 机位信息 |

### 8.2 外键关系

```
flightinfo (FlightStroeData)
  ├── flight_change (FlightChange) - flseq
  ├── flight_update_field (FlightUpdateField) - flseq
  ├── flight_subscribe (FlightSubscribe) - flseq
  └── flight_united_flight (UnitedFlight) - afleq / dfleq
```

---

## 9. 总结

### 9.1 项目特点

1. **基础工具模块**: 为航班服务提供数据模型支持
2. **多模型适配**: 支持不同系统/终端的数据格式需求
3. **字段丰富**: 航班模型包含200+字段，覆盖航班全生命周期
4. **多租户支持**: 通过 `tenant` 字段实现数据隔离

### 9.2 适用场景

- 航班数据存储与查询
- 航班状态追踪与变更记录
- 移动端/PC端航班数据展示
- 航班订阅与消息推送

### 9.3 优化建议

1. 升级Java版本至1.8+，使用 `LocalDateTime` 替代String时间字段
2. 添加数据校验注解 (`@NotNull`, `@Size`, `@Pattern`)
3. 完善工具类功能
4. 统一命名规范
5. 增加JavaDoc文档注释

---

## 10. 文件统计

| 模块 | Java文件数 | 主要类 |
|------|------------|--------|
| common/model | 13 | 实体类 |
| common/until | 3 | 工具类 |
| common/extend | 1 | 扩展模型 |
| commonparent/model/flight | 11 | 通用模型 |
| commonparent/model/base | 1 | 基础模型 |
| commonparent/model/service | 1 | 服务基类 |
| **总计** | **30** | - |

---

*报告生成时间: 2026-02-03*
*分析工具: Cursor AI Assistant*
