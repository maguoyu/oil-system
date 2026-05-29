# PVG-Transfer-Bill-MDB-DMP 油单同步数据管理平台 代码分析报告

> 分析日期：2026-02-04
> 项目版本：1.0.0.12

---

## 1. 项目概述

### 1.1 项目定位

PVG-Transfer-Bill-MDB-DMP 是一个**MDB油单数据同步管理平台**，负责将机场移动数据库（MDB）中的油单数据同步到数据管理平台（DMP）。主要功能包括：

- **数据同步**：从 SQL Server（MDB）同步油单数据到 MySQL（DMP）
- **定时任务**：自动化的增量数据同步机制
- **数据转换**：油单数据格式转换与映射
- **时间戳管理**：基于时间戳的增量同步策略

### 1.2 项目结构

```
pvg-transfer-bill-mdb-dmp/
├── pom.xml                           # Maven配置
├── src/main/
│   ├── java/
│   │   └── com/ias/oil/
│   │       ├── IASStartApplication.java    # 启动类
│   │       ├── cache/
│   │       │   └── RedisCache.java       # Redis缓存
│   │       ├── config/
│   │       │   ├── FastJson2JsonRedisSerializer.java
│   │       │   └── RedisConfig.java      # Redis配置
│   │       ├── controller/
│   │       │   └── OILBaseController.java # 基础控制器
│   │       ├── mapper/
│   │       │   ├── OilTaskSheetDmpMapper.java
│   │       │   ├── OilTaskSheetMapper.java
│   │       │   └── TimeRecordMapper.java
│   │       ├── model/                    # 数据模型（33个类）
│   │       │   ├── OilTaskSheet.java    # MDB油单实体
│   │       │   ├── OilTaskSheetDmp.java # DMP油单实体
│   │       │   ├── OilBill.java         # 油单实体
│   │       │   ├── TimeRecord.java      # 时间记录
│   │       │   └── ...
│   │       ├── service/
│   │       │   ├── OilBillManager.java
│   │       │   ├── OilTaskSheetService.java
│   │       │   ├── OilTaskSheetDmpService.java
│   │       │   ├── TimeRecordService.java
│   │       │   ├── TimerManager.java
│   │       │   └── impl/                # 服务实现
│   │       │       ├── OilBillManagermpl.java
│   │       │       ├── OilTaskSheetServiceImpl.java
│   │       │       ├── OilTaskSheetDmpServiceImpl.java
│   │       │       ├── TimeRecordServiceImpl.java
│   │       │       └── TimerManagerImpl.java
│   │       └── utils/                  # 工具类
│   │           ├── constant/Constants.java
│   │           ├── excel/              # Excel处理
│   │           └── text/              # 文本处理
│   └── resources/
│       ├── application.yml             # 应用配置
│       ├── application-prod.yml       # 生产环境配置
│       ├── logback-spring.xml        # 日志配置
│       └── mappers/                   # MyBatis映射
│           ├── OilTaskSheetMapper.xml
│           ├── OilTaskSheetDmpMapper.xml
│           └── TimeRecordMapper.xml
└── target/                           # 构建输出
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
| 数据库 | MySQL | 8.0.31 | 目标数据库 |
| 数据库 | SQL Server | - | 源数据库（MDB） |
| 定时任务 | Spring Task | - | 定时调度 |
| 文档生成 | Apache POI | 4.1.2 | Excel生成 |
| PDF生成 | iTextPDF | 5.5.9 | PDF处理 |
| 日志 | Logback | - | 日志框架 |
| HTTP | HttpClient | 4.5.2 | HTTP客户端 |
| 工具库 | IAS Utils | 7.0.4.0 | 内部工具库 |

### 2.2 依赖管理

```xml
<!-- Spring Boot -->
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.7.15</version>
</parent>

<!-- 数据库连接 -->
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid-spring-boot-starter</artifactId>
    <version>1.2.8</version>
</dependency>

<!-- 多数据源 -->
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>dynamic-datasource-spring-boot-starter</artifactId>
    <version>3.5.0</version>
</dependency>

<!-- MySQL驱动 -->
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.31</version>
</dependency>

<!-- SQL Server驱动 -->
<dependency>
    <groupId>net.sourceforge.jtds</groupId>
    <artifactId>jtds</artifactId>
    <version>1.3.1</version>
</dependency>
```

---

## 3. 核心功能分析

### 3.1 数据同步流程

```
┌─────────────────────────────────────────────────────────────────┐
│                   MDB → DMP 数据同步流程                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐       │
│  │ 定时触发    │───►│ 读取时间戳   │───►│ 查询增量数据 │       │
│  │ (每15秒)    │    │ (TimeRecord) │    │ (SQL Server)│       │
│  └─────────────┘    └─────────────┘    └─────────────┘       │
│                                              │                 │
│                                              ▼                 │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐       │
│  │ 更新DMP     │◄───│ 数据转换    │◄───│ 遍历油单    │       │
│  │ (MySQL)     │    │ (实体映射)   │    │ (OilTaskSheet)│     │
│  └─────────────┘    └─────────────┘    └─────────────┘       │
│                                              │                 │
│                                              ▼                 │
│  ┌─────────────┐    ┌─────────────┐                         │
│  │ 保存时间戳   │───►│ 同步完成     │                         │
│  │ (TimeRecord) │    │             │                         │
│  └─────────────┘    └─────────────┘                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 定时任务配置

```java
@Service
@DS("master")
public class TimerManagerImpl implements TimerManager {

    @Autowired
    private OilBillManager oilBillManager;

    /**
     * 每15秒执行一次同步任务
     */
    @Scheduled(cron = "${config.timer.mdbtodmp}")
    public void synchOILBillsToDmp() {
        long start = System.currentTimeMillis();
        logger.info("同步MDB油单到DMP，定时任务开始");
        oilBillManager.synchBillMdbToDmp();
        logger.info("同步MDB油单到DMP，定时任务结束，耗时{}ms", 
            System.currentTimeMillis() - start);
    }
}
```

### 3.3 同步服务实现

```java
@Service
public class OilBillManagermpl implements OilBillManager {

    @Override
    public void synchBillMdbToDmp() {
        long start = System.currentTimeMillis();
        logger.info("同步MDB油单到DMP，处理逻辑开始");
        
        // 1. 获取上次同步时间戳
        TimeRecord record = timeRecordService.getLastTime("synchBillMdbToDmp");
        logger.info("同步MDB油单到DMP，时间戳={}", record.getLastTime());
        
        // 2. 从MDB查询增量数据
        List<OilTaskSheet> oilTaskSheets = 
            oilTaskSheetService.selectOilTaskSheetsByLuts(record.getLastTime());
        
        if (oilTaskSheets.isEmpty()) {
            logger.info("同步MDB油单到DMP，待同步油单为空");
            return;
        }
        
        logger.info("同步MDB油单到DMP，待同步油单={}条", oilTaskSheets.size());
        
        // 3. 逐条处理：更新或新增
        for (OilTaskSheet taskSheet : oilTaskSheets) {
            OilTaskSheetDmp oilTaskSheet = convertOilTaskSheetDmp(taskSheet);
            String task = oilTaskSheet.getTASK();
            
            List<OilTaskSheetDmp> dbTasks = 
                oilTaskSheetDmpService.selectOilTaskSheetsByTask(task);
            
            if (dbTasks != null && dbTasks.size() > 0) {
                // 更新
                int count = oilTaskSheetDmpService.updateOilTaskSheet(oilTaskSheet);
                logger.info("同步MDB油单到DMP，更新油单{}条", count);
            } else {
                // 新增
                int count = oilTaskSheetDmpService.saveOilTaskSheet(oilTaskSheet);
                logger.info("同步MDB油单到DMP，新增油单{}条", count);
            }
        }
        
        // 4. 更新时间戳
        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss.SSS");
        String lastDate = sdf.format(
            oilTaskSheets.get(oilTaskSheets.size() - 1).getModifyTime());
        record.setLastTime(lastDate);
        timeRecordService.updateLastTime(record);
        
        logger.info("同步MDB油单到DMP，处理逻辑结束，耗时{}ms", 
            System.currentTimeMillis() - start);
    }
}
```

### 3.4 数据查询（Mapper）

```xml
<!-- OilTaskSheetMapper.xml -->
<select id="selectOilTaskSheetsByLuts" resultType="com.ias.oil.model.OilTaskSheet">
    SELECT top 1000 *
    FROM (
        SELECT t.*,
            (CASE WHEN t.LastDate > t.CheckDate THEN t.LastDate 
                  ELSE t.CheckDate END) modifyTime 
        FROM ecc_oil_tasksheet t
    ) aa
    WHERE aa.isCheck ='Y' 
      AND aa.modifyTime > #{lastDate,jdbcType=VARCHAR} 
    ORDER BY aa.modifyTime
</select>
```

---

## 4. 数据模型分析

### 4.1 核心实体

#### OilTaskSheet（MDB油单）

```java
public class OilTaskSheet {
    // ===== 航班信息 =====
    private String FLOP;      // 日期（加油开始时间的日期）
    private String FLNO;       // 航班号
    private String FLTI;      // 航线性质（D-国内，I-国际）
    private String REGN;      // 机号
    private String ACT3;       // 机型3码
    
    // ===== 航空公司信息 =====
    private String ALCN;      // 航空公司全称
    private String ALC2;      // 航空公司2码
    private String DFLT;      // 国际国内属性（Z-国内，W-国际）
    
    // ===== 任务信息 =====
    private String TASK;       // 油单号
    private String DTYP;      // 任务类型（FLL-加油，DRW-抽油）
    private String APLY;      // 是否有效标识
    
    // ===== 加油信息 =====
    private BigDecimal OTMP;   // 油量温度
    private BigDecimal ODEN;   // 油量密度
    private BigDecimal OLLT;   // 加油体积（升数）
    private BigDecimal OLKG;   // 加油重量（千克数）
    
    // ===== 车辆信息 =====
    private String CARN;      // 车号
    private String CART;      // 车辆类型
    
    // ===== 时间信息 =====
    private String DABE;      // 实际加油开始时间
    private String DAEN;      // 实际加油结束时间
    
    // ===== 人员信息 =====
    private int PNUM;         // 加油员数量
    private String PNM1~PNM5; // 加油员姓名
    
    // ===== 签名信息 =====
    private String signFlag;      // 是否有签名（Y/N）
    private String signPicPath;   // 签名图片路径
    private String signMan;       // 实际签单人
    private String signTime;      // 签名时间
    
    // ===== 其他信息 =====
    private String DES3;       // 目的机场3码
    private String DESNM;      // 目的地
    private String Remark;     // 备注
    private Timestamp modifyTime; // 修改时间
}
```

#### OilTaskSheetDmp（DMP油单）

与 OilTaskSheet 结构相同，用于存储同步后的数据。

### 4.2 辅助实体

| 实体 | 说明 |
|------|------|
| TimeRecord | 同步时间戳记录 |
| OilBill | 油单统计信息 |
| OilBillStatistics | 油单统计数据 |
| OilBillStatisticsForUser | 用户维度统计 |
| OilBillStatisticsForVnb | 车辆维度统计 |
| Flight | 航班信息 |
| FlightAnalysis | 航班分析 |
| Flightstatics | 航班统计 |
| Vehicle | 车辆信息 |
| PlaneRefuellerChecklist | 加油车检查单 |
| FlowmeterInspection | 流量计检定 |
| LargeScreenBean | 大屏展示数据 |

---

## 5. 数据库配置分析

### 5.1 多数据源配置

```yaml
spring:
  datasource:
    dynamic:
      druid:
        initial-size: 5
        min-idle: 5
        maxActive: 20
        maxWait: 60000
      datasource:
        # 主库（MySQL - DMP）
        master:
          driver-class-name: com.mysql.jdbc.Driver
          url: jdbc:mysql://192.168.200.8:3306/pdsinopdb
          username: afscadmin
          password: Airport@1024
        
        # MDB（SQL Server）
        mdbsqlserver:
          driver-class-name: net.sourceforge.jtds.jdbc.Driver
          url: jdbc:jtds:sqlserver://192.168.200.120:1433/AOTIMSDB
          username: sa
          password: pdahy
```

### 5.2 数据源切换注解

```java
// 访问SQL Server（MDB）
@DS("mdbsqlserver")
public class OilTaskSheetServiceImpl implements OilTaskSheetService {
    // ...
}

// 访问MySQL（主库）
@DS("master")
public class TimerManagerImpl implements TimerManager {
    // ...
}
```

### 5.3 数据库信息

| 数据源 | 类型 | 地址 | 数据库 | 用途 |
|--------|------|------|--------|------|
| master | MySQL | 192.168.200.8:3306 | pdsinopdb | 数据管理平台 |
| mdbsqlserver | SQL Server | 192.168.200.120:1433 | AOTIMSDB | 机场移动数据库 |

---

## 6. 定时任务配置

```yaml
# 定时器配置
config:
  timer:
    # 每15秒执行一次
    mdbtodmp: 0/15 * * * * ?
```

### 6.1 任务特点

| 特点 | 说明 |
|------|------|
| 触发频率 | 每15秒 |
| 执行方式 | 增量同步 |
| 批量大小 | 每次最多1000条 |
| 同步方向 | MDB → DMP |
| 状态判断 | IsCheck = 'Y'（已审核） |

---

## 7. 代码质量分析

### 7.1 优点

1. **架构简洁**：清晰的Controller → Service → Mapper分层
2. **多数据源支持**：使用dynamic-datasource实现灵活的数据源切换
3. **定时任务完善**：基于Spring Schedule的可靠定时任务
4. **增量同步**：基于时间戳的增量同步策略，避免全量同步
5. **日志完善**：关键节点都有日志记录
6. **工具类丰富**：提供Excel、PDF、文件处理等工具

### 7.2 问题与建议

#### 🔴 高优先级

| 问题 | 位置 | 建议 |
|------|------|------|
| 硬编码SQL | Mapper XML | 考虑移至代码配置 |
| 批量处理缺失 | OilBillManagermpl | 使用批量插入替代循环 |
| 异常处理不完善 | TimerManagerImpl | 添加异常告警机制 |
| 无重试机制 | 同步服务 | 增加失败重试逻辑 |

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

## 8. 业务字段说明

### 8.1 油单核心字段

| 字段 | 说明 | 示例值 |
|------|------|--------|
| URNO | 唯一号 | 12345 |
| FLOP | 加油日期 | 2024-01-15 |
| TASK | 油单号 | TS202401150001 |
| FLNO | 航班号 | MU5183 |
| REGN | 机号 | B-1234 |
| ALCN | 航司全称 | 中国东方航空公司 |
| ALC2 | 航司2码 | MU |
| OLLT | 加油量（升） | 15000 |
| OLKG | 加油量（公斤） | 12150 |
| CARN | 车号 | 油车001 |
| DABE | 加油开始时间 | 2024-01-15 08:30:00 |
| DAEN | 加油结束时间 | 2024-01-15 08:45:00 |
| IsCheck | 审核状态 | Y（已审核） |

### 8.2 任务类型

| 类型 | 说明 |
|------|------|
| FLL | 加油任务 |
| DRW | 抽油任务 |

### 8.3 航线性质

| 类型 | 说明 |
|------|------|
| D | 国内航线 |
| I | 国际/地区航线 |

---

## 9. 部署配置

### 9.1 服务端口

```yaml
server:
  port: 9003
```

### 9.2 构建配置

```xml
<!-- Maven Assembly -->
<plugin>
    <artifactId>maven-assembly-plugin</artifactId>
    <configuration>
        <descriptor>src/main/script/assembly.xml</descriptor>
    </configuration>
</plugin>

<!-- Spring Boot打包 -->
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
</plugin>
```

### 9.3 部署拓扑

```
┌─────────────────────────────────────────────────────────────────┐
│                     系统部署拓扑                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │           pvg-transfer-bill-mdb-dmp (端口: 9003)     │   │
│   │  ┌─────────────────────────────────────────────────┐ │   │
│   │  │  TimerManager (定时任务)                       │ │   │
│   │  │      ↓                                          │ │   │
│   │  │  OilBillManager (同步逻辑)                      │ │   │
│   │  │      ↓                                          │ │   │
│   │  │  OilTaskSheetService (MDB查询) ───┐           │ │   │
│   │  │  OilTaskSheetDmpService (DMP存储) ──┼──┐       │ │   │
│   │  │  TimeRecordService (时间戳管理)   ──┼──┼──┐    │ │   │
│   │  └─────────────────────────────────────────────────┘ │   │
│   └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│          ┌───────────────────┼───────────────────┐             │
│          ▼                   ▼                   ▼             │
│   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    │
│   │   MySQL     │    │ SQL Server  │    │    Redis    │    │
│   │  :3306      │    │   :1433     │    │   :6379     │    │
│   │  (pdsinopdb)│    │ (AOTIMSDB)  │    │   (可选)    │    │
│   └─────────────┘    └─────────────┘    └─────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 10. 总结

### 10.1 项目定位

本项目是一个**专注的油单数据同步服务**，负责将机场作业系统（MDB）中的油单数据同步到数据管理平台（DMP），为后续的数据分析、报表生成提供数据支撑。

### 10.2 核心能力

| 能力 | 实现 | 状态 |
|------|------|------|
| 定时同步 | Spring Schedule（每15秒） | ✅ |
| 增量同步 | 基于时间戳 | ✅ |
| 多数据源 | dynamic-datasource | ✅ |
| 数据转换 | 实体映射 | ✅ |
| 批量处理 | 单条循环（待优化） | ⚠️ |
| 异常处理 | 基础捕获 | ⚠️ |
| 监控告警 | 未实现 | ❌ |

### 10.3 优化建议

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
