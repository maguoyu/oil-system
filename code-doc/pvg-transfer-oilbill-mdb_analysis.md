# PVG-Transfer-Oilbill-MDB 油单同步至MDB项目代码分析报告

> 分析日期：2026-02-04
> 项目版本：1.0.1.18

---

## 1. 项目概述

### 1.1 项目定位

PVG-Transfer-Oilbill-MDB 是一个**油单数据分发系统**，负责将油库系统（db_oil_pvg）中的油单数据同步到MDB（移动数据库）。同时支持从MIS系统同步航空公司和机号信息到MDB。

### 1.2 项目特点

| 特点 | 说明 |
|------|------|
| 多数据源 | 4个数据源（master、db_oil_pvg、mis_mutil、mdb） |
| 定时任务 | 5个定时任务（油单抽取、同步、航班、同步航班、航空公司、机号） |
| Dubbo服务 | 支持Dubbo RPC调用 |
| ActiveMQ | 支持消息队列 |

### 1.3 项目结构

```
pvg-transfer-oilbill-mdb/
├── pom.xml                           # Maven配置（1.0.1.18）
├── src/main/
│   ├── java/
│   │   └── com/ias/oil/
│   │       ├── IASStartApplication.java    # 启动类
│   │       ├── cache/RedisCache.java       # Redis缓存
│   │       ├── config/                     # 配置类
│   │       │   ├── FastJson2JsonRedisSerializer.java
│   │       │   └── RedisConfig.java
│   │       ├── controller/OILBaseController.java # 基础控制器
│   │       ├── mapper/                     # MyBatis Mapper（7个）
│   │       │   ├── AirlineMapper.java
│   │       │   ├── ConfigMapper.java
│   │       │   ├── FlightMapper.java
│   │       │   ├── OilBillMapper.java
│   │       │   ├── OilTaskSheetMapper.java
│   │       │   ├── RegistrationMapper.java
│   │       │   └── TimeRecordMapper.java
│   │       ├── model/                      # 数据模型（34个类）
│   │       │   ├── OilBill.java           # 油单实体
│   │       │   ├── OilTaskSheet.java      # MDB油单实体
│   │       │   ├── Flight.java            # 航班实体
│   │       │   ├── MisAirline.java        # 航空公司
│   │       │   ├── Registration.java      # 机号实体
│   │       │   ├── TimeRecord.java        # 时间戳记录
│   │       │   └── ...
│   │       └── service/                    # 服务层
│   │           ├── ConfigManager.java
│   │           ├── ExtractManager.java
│   │           ├── FlightManager.java
│   │           ├── MISSynch.java          # MIS同步接口
│   │           ├── MISMDBManager.java     # MDB管理
│   │           ├── MISSourceManager.java  # MIS数据源
│   │           ├── OILSourceManager.java  # OIL数据源
│   │           ├── OilBillManager.java    # 油单管理
│   │           ├── OilTaskSheetService.java
│   │           ├── SendManager.java
│   │           ├── TimerManager.java
│   │           └── impl/                  # 服务实现（13个）
│   └── resources/
│       ├── application.yml
│       ├── application-prod.yml         # 生产配置
│       ├── logback-spring.xml
│       └── mappers/                     # MyBatis映射（7个XML）
└── target/                             # 构建输出
```

---

## 2. 技术栈分析

### 2.1 核心技术

| 类别 | 技术 | 版本 | 说明 |
|------|------|------|------|
| 框架 | Spring Boot | 2.7.15 | 应用框架 |
| ORM | MyBatis | 2.2.0 | 数据持久化 |
| 连接池 | Druid | 1.2.8 | 数据库连接池 |
| 多数据源 | dynamic-datasource | 3.5.0 | 多数据源切换 |
| RPC | Dubbo | 0.2.0 | 分布式服务 |
| 消息队列 | ActiveMQ | 5.7.0 | 消息中间件 |
| 数据库 | MySQL | 5.1.35 | 主数据库 |
| 定时任务 | Spring Schedule + Quartz | 2.2.1 | 任务调度 |
| 文档生成 | Apache POI | 4.1.2 | Excel生成 |
| PDF生成 | iTextPDF | 5.5.9 | PDF处理 |

### 2.2 内部依赖

| 依赖 | 版本 | 说明 |
|------|------|------|
| IAS Utils | 7.0.4.0 | 内部工具库 |
| MISGWSAPI | 5.0.2.3 | MIS网关API |
| security-api | 4.0.0.1 | 安全认证 |

### 2.3 Maven配置

```xml
<!-- Dubbo -->
<dependency>
    <groupId>com.alibaba.boot</groupId>
    <artifactId>dubbo-spring-boot-starter</artifactId>
    <version>0.2.0</version>
</dependency>

<!-- ActiveMQ -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-activemq</artifactId>
</dependency>

<!-- MyBatis -->
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>2.2.0</version>
</dependency>

<!-- 多数据源 -->
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>dynamic-datasource-spring-boot-starter</artifactId>
    <version>3.5.0</version>
</dependency>

<!-- MySQL -->
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>5.1.35</version>
</dependency>

<!-- SQL Server -->
<dependency>
    <groupId>net.sourceforge.jtds</groupId>
    <artifactId>jtds</artifactId>
    <version>1.3.1</version>
</dependency>
```

---

## 3. 核心业务流程分析

### 3.1 数据同步架构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        数据同步架构图                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   ┌─────────────────────┐              ┌─────────────────────┐          │
│   │  db_oil_pvg        │              │      mis_mutil      │          │
│   │  (油库业务数据库)   │              │   (MIS主数据系统)   │          │
│   └─────────┬───────────┘              └──────────┬──────────┘          │
│             │                                      │                      │
│             ▼                                      ▼                      │
│   ┌─────────────────────┐              ┌─────────────────────┐          │
│   │  OILSourceManager   │              │  MISSourceManager   │          │
│   │  (数据抽取服务)     │              │  (MIS数据源服务)    │          │
│   └─────────┬───────────┘              └──────────┬──────────┘          │
│             │                                      │                      │
│             │         ┌─────────────────────┐       │                      │
│             │         │  ExtractManager    │       │                      │
│             │         │  (抽取管理服务)     │       │                      │
│             │         └─────────┬───────────┘       │                      │
│             │                   │                 │                      │
│             ▼                   ▼                 ▼                      │
│   ┌─────────────────────────────────────────────────────────────────┐  │
│   │                     TimerManagerImpl                            │  │
│   │  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐   │  │
│   │  │ 抽取油单   │ │ 同步油单   │ │ 抽取航班   │ │ 同步航空公司│   │  │
│   │  │ (每10秒)  │ │ (每15秒)  │ │ (每分钟)  │ │ (每1分钟) │   │  │
│   │  └────────────┘ └────────────┘ └────────────┘ └────────────┘   │  │
│   └─────────────────────────────────────────────────────────────────┘  │
│                           │                                            │
│             ┌─────────────┼─────────────┐                              │
│             ▼             ▼             ▼                              │
│   ┌────────────────┐ ┌────────────────┐ ┌────────────────┐           │
│   │  OilBillManager│ │ FlightManager  │ │   MISSynch     │           │
│   │  (油单管理)    │ │  (航班管理)    │ │  (MIS同步)     │           │
│   └───────┬────────┘ └───────┬────────┘ └───────┬────────┘           │
│           │                  │                  │                      │
│           ▼                  ▼                  ▼                      │
│   ┌─────────────────────────────────────────────────────────────────┐  │
│   │                         MDB数据库                               │  │
│   │  ecc_oil_tasksheet  │  airline  │  registration  │  flight   │  │
│   └─────────────────────────────────────────────────────────────────┘  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.2 核心定时任务

```java
@Service
public class TimerManagerImpl implements TimerManager {

    @Autowired
    private OilBillManager oilBillManager;
    @Autowired
    private ExtractManager extractManager;
    @Autowired
    private SendManager sendManager;
    @Autowired
    private MISSynch misSynch;

    /**
     * 任务1：抽取油单数据（每10秒）
     * 从 db_oil_pvg 抽取油单到 master
     */
    @Scheduled(cron = "${config.timer.bill}")
    public void extractOILBills() {
        oilBillManager.addOilBills();
    }

    /**
     * 任务2：同步油单到MDB（每15秒）
     * 从 master 同步油单到 MDB
     */
    @Scheduled(cron = "${config.timer.billtomdb}")
    public void synchOILBills() {
        oilBillManager.synchBillToMdb();
    }

    /**
     * 任务3：抽取航班数据（每分钟）
     * 从 db_oil_pvg 抽取航班到 master
     */
    @Scheduled(cron = "${config.timer.flight}")
    public void extractFlights() {
        extractManager.extractFlights();
    }

    /**
     * 任务4：同步航空公司到MDB（每1分钟）
     * 从 mis_mutil 同步航空公司到 MDB
     */
    @Scheduled(cron = "${config.timer.airlinetomdb}")
    public void synchAirlines() {
        misSynch.synchAirlineToMdb();
    }

    /**
     * 任务5：同步机号到MDB（每1分钟）
     * 从 mis_mutil 同步机号到 MDB
     */
    @Scheduled(cron = "${config.timer.regntomdb}")
    public void synchRegns() {
        misSynch.synchRegistrationToMdb();
    }
}
```

### 3.3 油单抽取流程（db_oil_pvg → master）

```java
@Service
@DS("master")
public class OilBillManagermpl implements OilBillManager {

    @Autowired
    private OILSourceManager oilSourceManager;

    @Override
    public void addOilBills() {
        // 1. 获取上次同步时间戳
        TimeRecord record = getLastTime("oilBill");
        
        // 2. 从油库数据库抽取增量油单
        List<OilBill> oilBills = oilSourceManager.getSourceOilBills(record.getLastTime());
        
        if (oilBills.isEmpty()) {
            logger.info("油单为空");
            return;
        }
        
        // 3. 批量插入或更新到主库
        oilBillMapper.insertOrUpdateBatch(oilBills);
        
        // 4. 更新时间戳
        record.setLastTime(oilBills.get(oilBills.size() - 1).getTimeluts6());
        updateLastTime(record);
    }
}
```

### 3.4 油单同步流程（master → MDB）

```java
@Override
public void synchBillToMdb() {
    // 1. 获取待同步油单
    TimeRecord record = getLastTime("synchbilltomdb");
    List<OilBill> oilBills = oilBillMapper.selectOilBills(record.getLastTime());
    
    if (oilBills.isEmpty()) {
        return;
    }
    
    // 2. 遍历处理每条油单
    for (OilBill bill : oilBills) {
        // 转换为MDB油单格式
        OilTaskSheet oilTaskSheet = convertOilBill(bill);
        
        // 3. 计算NTYP（任务数量）
        List<OilBill> oldSheets = getOilBillByFlnoAndFlopAndRegnumber(oilTaskSheet);
        Set<String> billNumbers = new HashSet<>();
        for (OilBill oldSheet : oldSheets) {
            if (oldSheet.getValid()) {
                billNumbers.add(oldSheet.getBillNumber());
            }
        }
        oilTaskSheet.setNTYP(billNumbers.size());
        
        // 4. 更新NTYP
        oilTaskSheetService.updateNTYP(oilTaskSheet);
        
        // 5. 判断新增或更新
        List<OilTaskSheet> dbTasks = oilTaskSheetService.selectOilTaskSheetsByBillid(oilTaskSheet);
        if (dbTasks != null && !dbTasks.isEmpty()) {
            // 已审核的油单只更新签单状态
            if ("Y".equals(dbTasks.get(0).getIsCheck())) {
                oilTaskSheetService.updateOilTaskSheetSign(oilTaskSheet);
            } else {
                oilTaskSheetService.updateOilTaskSheet(oilTaskSheet);
            }
        } else {
            oilTaskSheetService.saveOilTaskSheet(oilTaskSheet);
        }
    }
    
    // 6. 更新时间戳
    record.setLastTime(oilBills.get(oilBills.size() - 1).getTimeluts6());
    updateLastTime(record);
}
```

### 3.5 数据转换逻辑（OilBill → OilTaskSheet）

```java
public OilTaskSheet convertOilBill(OilBill oilBill) {
    OilTaskSheet oSheet = new OilTaskSheet();
    
    // ===== 基本信息映射 =====
    oSheet.setFLOP(oilBill.getFlop());              // 航班日期
    oSheet.setTASK(oilBill.getBillNumber());         // 油单号
    oSheet.setREGN(oilBill.getRegnumber());          // 机号
    oSheet.setALC2(oilBill.getAl2c());              // 航空公司2码
    oSheet.setALCN(oilBill.getAlcname());            // 航空公司名称
    
    // ===== 航线信息 =====
    oSheet.setFLNO(oilBill.getFlno());               // 航班号
    oSheet.setACT3(oilBill.getAc3c());              // 机型3码
    
    // 国际/国内属性转换
    String flti = oilBill.getFlti();
    if ("D".equals(flti)) {
        oSheet.setFLTI("D");
        oSheet.setDFLT("Z");                        // 国内航司
    } else if ("I".equals(flti) || "M".equals(flti) || "R".equals(flti)) {
        oSheet.setFLTI("I");
        oSheet.setDFLT("W");                        // 国际航司
    }
    
    // ===== 任务信息 =====
    if ("JY".equals(oilBill.getGasorpumptype())) {
        oSheet.setDTYP("FLL");                      // 加油
    } else if ("CY".equals(oilBill.getGasorpumptype())) {
        oSheet.setDTYP("DRW");                      // 抽油
    }
    
    oSheet.setAPLY(oilBill.getValid() ? "Y" : "N");  // 有效性
    
    // ===== 油量信息 =====
    oSheet.setOTMP(convertStringToBigDecimal(oilBill.getOilTemperature(), 2));  // 油温
    oSheet.setODEN(convertStringToBigDecimal(oilBill.getOilDensity(), 4));      // 油密
    oSheet.setOLLT(convertStringToBigDecimal(oilBill.getVolume(), 2));          // 加油体积
    oSheet.setOLKG(convertStringToBigDecimal(oilBill.getWeight(), 2));          // 加油重量
    
    // ===== 车辆信息 =====
    oSheet.setCARN(oilBill.getVehicleNumber());      // 车号
    String vehicleType = oilBill.getVehicleType();
    if ("VEHOILCAN".equals(vehicleType)) {
        oSheet.setCART("罐式加油车");               // 油罐车
    } else if ("VEHOILPIPE".equals(vehicleType)) {
        oSheet.setCART("管线加油车");               // 管线车
    }
    
    // ===== 时间信息 =====
    oSheet.setDABE(oilBill.getTaskStartTime().substring(11, 16));  // 开始时间(HH:mm)
    oSheet.setDAEN(oilBill.getTaskEndTime().substring(11, 16));    // 结束时间(HH:mm)
    
    // ===== 人员信息 =====
    oSheet.setPNUM(Integer.valueOf(oilBill.getFuleManCount()));   // 加油员数量
    oSheet.setPNM1(oilBill.getFuleMan1());                        // 加油员1
    oSheet.setPNM2(oilBill.getFuleMan2());                        // 加油员2
    oSheet.setPNM3(oilBill.getFuleMan3());                        // 加油员3
    oSheet.setPNM4(oilBill.getFuleMan4());                        // 加油员4
    
    // ===== 签名信息 =====
    oSheet.setEleBillNumber(oilBill.getEleBillNumber());           // 电子油单号
    oSheet.setSignPicPath(oilBill.getSignPicPath());               // 签名图片路径
    oSheet.setSignFlag(oilBill.getSignFlag());                     // 是否有签名
    oSheet.setSignSource(oilBill.getSignSource());                 // 签名来源
    oSheet.setSignMan(oilBill.getSignMan());                      // 签单人
    oSheet.setSignTime(oilBill.getSignTime());                     // 签名时间
    
    return oSheet;
}
```

---

## 4. 数据库配置分析

### 4.1 多数据源配置

```yaml
spring:
  datasource:
    dynamic:
      druid:
        initial-size: 5
        min-idle: 5
        maxActive: 20
      datasource:
        # 主库 - 统一数据存储
        master:
          driver-class-name: com.mysql.jdbc.Driver
          url: jdbc:mysql://172.21.0.19:3306/dd3500
          username: root
          password: airport
        
        # 油库业务库 - 油单数据来源
        db_oil_pvg:
          url: jdbc:mysql://172.21.0.19:3306/db_oil_pvg
          driver-class-name: com.mysql.jdbc.Driver
          username: root
          password: airport
        
        # MIS主数据库 - 航空公司/机号数据来源
        mis_mutil:
          url: jdbc:mysql://172.21.0.19:3306/mis_mutil
          driver-class-name: com.mysql.jdbc.Driver
          username: root
          password: airport
        
        # MDB - 移动数据库（目标库）
        mdbsqlserver:
          url: jdbc:mysql://172.21.0.19:3306/mdb
          driver-class-name: com.mysql.jdbc.Driver
          username: root
          password: airport
```

### 4.2 数据源汇总

| 数据源 | 类型 | 地址 | 库名 | 用途 |
|--------|------|------|------|------|
| master | MySQL | 172.21.0.19:3306 | dd3500 | 主数据存储 |
| db_oil_pvg | MySQL | 172.21.0.19:3306 | db_oil_pvg | 油库业务库 |
| mis_mutil | MySQL | 172.21.0.19:3306 | mis_mutil | MIS主数据 |
| mdbsqlserver | MySQL | 172.21.0.19:3306 | mdb | MDB目标库 |

### 4.3 数据源切换注解

```java
// 访问油库数据库
@DS("db_oil_pvg")
public class OILSourceManagerImpl implements OILSourceManager {
    // ...
}

// 访问主库
@DS("master")
public class OilBillManagermpl implements OilBillManager {
    // ...
}
```

---

## 5. 定时任务配置

```yaml
config:
  dpid: OIL_PVG_010
  timer:
    bill: 0/10 * * * * ?              # 抽取油单（每10秒）
    billtomdb: 0/15 * * * * ?          # 同步油单到MDB（每15秒）
    sendFlightToOilCloud: "-"           # 发送到云端（禁用）
    flight: 0 * * * * ?                # 抽取航班（每分钟）
    airlinetomdb: 0 0/1 * * * ?        # 同步航空公司（每1分钟）
    regntomdb: 0 0/1 * * * ?           # 同步机号（每1分钟）
```

### 5.1 任务汇总

| 任务 | Cron表达式 | 频率 | 数据流向 |
|------|-----------|------|----------|
| 抽取油单 | `0/10 * * * * ?` | 每10秒 | db_oil_pvg → master |
| 同步油单到MDB | `0/15 * * * * ?` | 每15秒 | master → mdb |
| 抽取航班 | `0 * * * * ?` | 每分钟 | db_oil_pvg → master |
| 同步航空公司 | `0 0/1 * * * ?` | 每1分钟 | mis_mutil → mdb |
| 同步机号 | `0 0/1 * * * ?` | 每1分钟 | mis_mutil → mdb |

---

## 6. 数据模型分析

### 6.1 核心实体

#### OilBill（油库油单）

```java
public class OilBill {
    // ===== 标识信息 =====
    private String id;              // 主键ID
    private String billNumber;     // 油单号
    private String flseq;          // 航班唯一标识
    
    // ===== 航班信息 =====
    private String flop;           // 航班日期
    private String flno;           // 航班号
    private String regnumber;      // 机号
    private String ac3c;           // 机型3码
    private String acname;         // 机型名称
    private String org3;           // 出发机场3码
    private String des3;           // 目的机场3码
    private String flti;           // 航线性质
    private String stopOver;       // 经停机场
    
    // ===== 航空公司信息 =====
    private String al2c;           // 航司2码
    private String al4c;           // 航司4码
    private String alcname;        // 航司名称
    
    // ===== 任务信息 =====
    private String taskId;         // 任务ID
    private String taskStartTime;  // 任务开始时间
    private String taskEndTime;    // 任务结束时间
    private String gasorpumptype;  // 作业类型（JY-加油/CY-抽油）
    
    // ===== 油量信息 =====
    private String volume;         // 加油体积
    private String weight;         // 加油重量
    private String oilTemperature; // 油温
    private String oilDensity;     // 油密
    private String oll;           // 总油量
    private String assaySheetNumber; // 化验单号
    
    // ===== 车辆信息 =====
    private String vehicleType;    // 车辆类型
    private String vehicleNumber;  // 车号
    private String wellNumber;     // 地井编号
    
    // ===== 人员信息 =====
    private String fuleMan1~4;    // 加油员
    private String fuleManCount;  // 加油员数量
    
    // ===== 签名信息 =====
    private String eleBillNumber;   // 电子油单号
    private String signPicPath;    // 签名图片路径
    private String signFlag;       // 是否有签名
    private String signSource;     // 签名来源
    private String signMan;        // 签单人
    private String signTime;       // 签名时间
    private String signRemark;     // 签名备注
    
    // ===== 状态信息 =====
    private Boolean isValid;       // 是否有效
    private Boolean isConfirm;     // 是否确认
    private Boolean isAudited;     // 是否审核
    private Boolean isDelete;     // 是否删除
    private Boolean isUploaded;    // 是否上传
    
    // ===== 同步状态 =====
    private String synchMdbStatus; // 同步MDB状态
    private String synchMdbTime;  // 同步MDB时间
    
    // ===== 时间戳 =====
    private String timeluts6;     // 时间戳
}
```

#### OilTaskSheet（MDB油单）

与 OilBill 结构类似，增加了MDB特定字段。

#### Flight（航班信息）

```java
public class Flight {
    private String id;
    private String flseq;         // 航班唯一标识
    private String flno;          // 航班号
    private String flti;          // 航线性质
    private String regnumber;     // 机号
    private String ac3c;          // 机型
    private String org3;          // 出发机场
    private String des3;          // 目的机场
    private String stopOver;      // 经停
    private String flop;          // 航班日期
    private String std;           // 计划起飞时间
    private String sta;           // 计划到达时间
    private String etd;           // 预计起飞时间
    private String eta;           // 预计到达时间
    private String atd;           // 实际起飞时间
    private String ata;           // 实际到达时间
    private String flightStatus;  // 航班状态
    private String createDate;    // 创建时间
    private String modifyDate;    // 修改时间
}
```

### 6.2 辅助实体

| 实体 | 说明 | 主要用途 |
|------|------|----------|
| MisAirline | MIS航空公司 | 同步到MDB |
| Registration | 机号信息 | 同步到MDB |
| TimeRecord | 时间戳记录 | 增量同步控制 |
| Vehicle | 车辆信息 | 车辆管理 |
| OilBillStatistics | 油单统计 | 报表分析 |
| PlaneRefuellerChecklist | 加油车检查单 | 车辆管理 |
| FlowmeterInspection | 流量计检定 | 设备管理 |

---

## 7. Dubbo服务配置

### 7.1 Dubbo配置

```yaml
dubbo:
  application:
    name: pvg-transfer-oilbill-mdb
  consumer:
    check: false
    timeout: 6000000
  registry:
    id: dubbo-registry
    address: zookeeper://zookeeper:2181
```

### 7.2 Dubbo服务引用

```java
@Service
public class OilBillManagermpl extends CommonManager implements OilBillManager {

    @Reference
    private AirLineService airLineService;  // 航空公司服务
    
    // 使用航空公司服务获取航司信息
    public void someMethod() {
        B2BRequest br = new B2BRequest();
        br.setUserId(mis_user);
        br.setPassword(mis_pwd);
        br.setSystemId(systemid);
        
        CAirLine airLine = new CAirLine();
        airLine.setAl4c(oilBill.getAl4c());
        
        AirLineResponse response = airLineService.getAirLineList(br, airLine);
    }
}
```

---

## 8. 代码质量分析

### 8.1 优点

1. **多数据源管理完善**：使用dynamic-datasource实现灵活切换
2. **定时任务分工明确**：5个定时任务各司其职
3. **增量同步策略**：基于时间戳的增量同步
4. **批量处理**：使用insertOrUpdateBatch提高性能
5. **数据校验**：转换过程中的有效性校验
6. **日志完善**：关键节点都有日志记录

### 8.2 问题与建议

#### 🔴 高优先级

| 问题 | 位置 | 建议 |
|------|------|------|
| 循环处理油单 | OilBillManagermpl | 使用批量插入替代循环 |
| 异常处理不完善 | TimerManagerImpl | 增加异常告警机制 |
| 无重试机制 | 同步逻辑 | 增加失败重试逻辑 |
| 硬编码SQL | Mapper XML | 考虑移至配置 |

#### 🟡 中优先级

| 问题 | 位置 | 建议 |
|------|------|------|
| 缺少幂等处理 | 同步逻辑 | 增加唯一性校验 |
| 无数据校验 | 数据转换 | 增加数据有效性校验 |
| 监控缺失 | 全局 | 接入监控平台 |
| 无同步状态监控 | 定时任务 | 增加同步状态记录 |

#### 🟢 低优先级

| 问题 | 位置 | 建议 |
|------|------|------|
| 方法注释不完整 | 多个Service | 增加JavaDoc |
| 魔法值 | Constants | 集中管理常量 |
| 无单元测试 | - | 增加测试用例 |
| 无API文档 | - | 添加Swagger文档 |

---

## 9. 业务字段说明

### 9.1 油单核心字段映射

| OilBill字段 | OilTaskSheet字段 | 说明 |
|-------------|------------------|------|
| billNumber | TASK | 油单号 |
| flop | FLOP | 航班日期 |
| flno | FLNO | 航班号 |
| regnumber | REGN | 机号 |
| al2c | ALC2 | 航司2码 |
| alcname | ALCN | 航司全称 |
| ac3c | ACT3 | 机型3码 |
| des3 | DES3 | 目的机场3码 |
| stopOver | VIANM | 经停机场 |
| volume | OLLT | 加油体积（升） |
| weight | OLKG | 加油重量（公斤） |
| oilTemperature | OTMP | 油温 |
| oilDensity | ODEN | 油密 |
| vehicleNumber | CARN | 车号 |
| vehicleType | CART | 车辆类型 |
| taskStartTime | DABE | 开始时间 |
| taskEndTime | DAEN | 结束时间 |
| fuleMan1~4 | PNM1~4 | 加油员 |
| eleBillNumber | eleBillNumber | 电子油单号 |
| signPicPath | signPicPath | 签名路径 |
| signFlag | signFlag | 签名标识 |

### 9.2 任务类型

| OilBill值 | OilTaskSheet值 | 说明 |
|-----------|----------------|------|
| JY | FLL | 加油任务 |
| CY | DRW | 抽油任务 |

### 9.3 航线性质

| OilBill值 | OilTaskSheet值 | DFLT值 | 说明 |
|-----------|----------------|--------|------|
| D | D | Z | 国内航线/国内航司 |
| I | I | W | 国际航线/国际航司 |
| M | I | W | 地区航线/国际航司 |
| R | I | W | 跨境航线/国际航司 |

### 9.4 车辆类型

| OilBill值 | OilTaskSheet值 | 说明 |
|-----------|----------------|------|
| VEHOILCAN | 罐式加油车 | 油罐车 |
| VEHOILPIPE | 管线加油车 | 管线车 |

---

## 10. 部署配置

### 10.1 服务端口

```yaml
server:
  port: 8111
```

### 10.2 部署拓扑

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         系统部署拓扑                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │          pvg-transfer-oilbill-mdb (端口: 8111)                │   │
│   │  ┌─────────────────────────────────────────────────────────┐   │   │
│   │  │  TimerManagerImpl (定时任务调度)                       │   │   │
│   │  └─────────────────────────────────────────────────────────┘   │   │
│   │                          │                                      │
│   │  ┌────────────────────────┼────────────────────────┐          │   │
│   │  ▼                        ▼                        ▼          │   │
│   │ ┌────────────┐    ┌────────────┐    ┌────────────┐           │   │
│   │ │ OilBillMgr │    │FlightMgr   │    │  MISSynch  │           │   │
│   │ │(油单同步)   │    │(航班同步)   │    │(航司/机号) │           │   │
│   │ └─────┬──────┘    └──────┬─────┘    └──────┬─────┘           │   │
│   │       │                 │                │                   │   │
│   │       ▼                 ▼                ▼                   │   │
│   │ ┌─────────────────────────────────────────────────────────┐   │   │
│   │ │              OILSourceManager (db_oil_pvg)            │   │   │
│   │ └─────────────────────────────────────────────────────────┘   │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                              │                                          │
│         ┌────────────────────┼────────────────────┐                    │
│         ▼                    ▼                    ▼                    │
│   ┌────────────┐    ┌────────────┐    ┌────────────┐                  │
│   │   master   │    │ db_oil_pvg │    │ mis_mutil  │                  │
│   │  :3306     │    │  :3306     │    │  :3306     │                  │
│   │ (dd3500)   │    │            │    │            │                  │
│   └────────────┘    └────────────┘    └────────────┘                  │
│                                                                  │
│                          ┌────────────────────┐                 │
│                          │       mdb          │                 │
│                          │      :3306         │                 │
│                          │  (目标数据库)       │                 │
│                          └────────────────────┘                 │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 10.3 Zookeeper配置

```yaml
dubbo:
  registry:
    address: zookeeper://zookeeper:2181
```

---

## 11. 与MDB→DMP项目对比

| 对比项 | pvg-transfer-bill-mdb-dmp | pvg-transfer-oilbill-mdb |
|--------|---------------------------|--------------------------|
| **方向** | MDB → DMP | db_oil_pvg → MDB |
| **源数据库** | SQL Server (AOTIMSDB) | MySQL (db_oil_pvg) |
| **目标数据库** | MySQL (pdsinopdb) | MySQL (mdb) |
| **定时任务** | 1个（每15秒） | 5个（10秒/15秒/1分钟） |
| **数据源数量** | 2个 | 4个 |
| **Dubbo** | 无 | 有 |
| **ActiveMQ** | 无 | 有 |
| **版本** | 1.0.0.12 | 1.0.1.18 |
| **端口** | 9003 | 8111 |
| **核心功能** | 油单同步 | 油单+航班+航司+机号同步 |

### 11.1 功能对比

| 功能 | MDB→DMP | OIL→MDB |
|------|---------|---------|
| 油单同步 | ✅ | ✅ |
| 航班同步 | ❌ | ✅ |
| 航空公司同步 | ❌ | ✅ |
| 机号同步 | ❌ | ✅ |
| 增量同步 | ✅ | ✅ |
| 批量处理 | ❌ | ✅ |
| 数据转换 | ✅ | ✅ |

---

## 12. 总结

### 12.1 项目定位

本项目是一个**多数据源油单数据分发系统**，负责将油库业务系统、航班系统、MIS主数据系统中的数据同步到MDB（移动数据库），为机场移动应用提供数据支撑。

### 12.2 核心能力

| 能力 | 实现 | 状态 |
|------|------|------|
| 多数据源管理 | dynamic-datasource | ✅ |
| 定时任务调度 | Spring Schedule | ✅ |
| 增量同步 | 基于时间戳 | ✅ |
| 数据转换 | 实体映射 | ✅ |
| Dubbo服务 | Zookeeper | ✅ |
| ActiveMQ消息 | JMS | ✅ |
| 批量处理 | insertOrUpdateBatch | ✅ |
| 异常处理 | 基础捕获 | ⚠️ |
| 监控告警 | 未实现 | ❌ |

### 12.3 优化建议

1. **短期优化**：
   - 优化批量插入逻辑
   - 增加同步状态监控
   - 完善异常处理

2. **中期重构**：
   - 引入消息队列解耦
   - 增加重试机制
   - 接入监控系统

3. **长期规划**：
   - 微服务化改造
   - 数据一致性保证
   - 同步性能优化

---

*报告生成时间：2026-02-04*
*分析工具：Cursor AI Code Analysis*
