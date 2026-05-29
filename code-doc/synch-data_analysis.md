# synch-data 浦东同步模块代码分析报告

> 分析日期：2026-02-04
> 项目路径：e:\work\code\oil\pudong\synch-data\
> 文档版本：1.0

---

## 1. 项目概述

### 1.1 基本信息

| 属性 | 值 |
|------|-----|
| 项目名称 | synch-data-pvg (浦东同步模块) |
| GroupId | com.ias.oil.synch |
| ArtifactId | synch-data-pvg |
| 版本 | 1.0.0.9 |
| 名称 | 浦东同步模块 |
| 描述 | 浦东航油数据同步至总部平台 |
| Java版本 | 1.8 |
| 框架 | Spring Boot 2.0.8.RELEASE |

### 1.2 模块结构

```
synch-data-pvg/
├── pom.xml                                    # Maven配置 (315行)
├── src/main/java/com/ias/oil/
│   ├── IASStartApplication.java               # 启动类
│   ├── config/                               # 配置类
│   │   ├── OILServiceConfig.java             # RestTemplate配置
│   │   ├── OILServiceExceptionAdvice.java     # 异常处理
│   │   └── IASTransferConfig.java            # ActiveMQ配置
│   ├── controller/                           # REST控制器
│   │   └── SynchController.java              # 同步控制器
│   ├── service/                              # 服务层
│   │   ├── BILLUploadManager.java            # 油单上传接口
│   │   ├── BILLSettlementManager.java         # 结算同步接口
│   │   ├── TaskUploadManager.java            # 任务上传接口
│   │   └── impl/                             # 服务实现
│   │       ├── BILLUploadManagerImpl.java    # 油单上传实现 (490行)
│   │       ├── BILLSettlementManagerImpl.java # 结算同步实现
│   │       ├── TaskUploadManagerImpl.java    # 任务上传实现 (531行)
│   │       ├── ToolManager.java              # 工具管理
│   │       └── CommonService.java            # 公共服务
│   ├── dao/                                  # 数据访问
│   │   ├── UploadRecordRepository.java        # 上传记录DAO
│   │   ├── UploadTaskRecordRepository.java    # 任务上传记录DAO
│   │   ├── TimeRecordRepository.java         # 时间记录DAO
│   │   └── OilConfigRepository.java         # 配置DAO
│   ├── model/                                # 数据模型
│   │   ├── Bill.java                        # 油单实体
│   │   ├── SynTask.java                      # 同步任务实体
│   │   ├── UploadRecord.java                 # 上传记录
│   │   ├── UploadTaskRecord.java            # 任务上传记录
│   │   ├── TimeRecord.java                  # 时间戳记录
│   │   ├── OilConfig.java                   # 配置实体
│   │   ├── CustomerCredit.java               # 客户信用
│   │   └── entity/                          # 扩展实体
│   ├── timer/                                # 定时任务
│   │   └── SynchTimer.java                   # 同步定时器 (112行)
│   ├── transfer/                             # 消息传输
│   │   └── IASMessageTransfer.java           # JMS消息传输 (67行)
│   ├── util/                                 # 工具类
│   │   └── DateUtil.java                     # 日期工具
│   └── exception/                            # 异常定义
│       └── OILException.java                 # 业务异常
│
├── src/main/resources/
│   ├── mappers/                              # MyBatis映射
│   │   ├── UploadRecordMapper.xml            # 上传记录映射
│   │   ├── UploadTaskRecordMapper.xml        # 任务上传映射
│   │   ├── TimeRecordMapper.xml             # 时间记录映射
│   │   └── OilConfigMapper.xml              # 配置映射
│   ├── logback-spring.xml                    # 日志配置
│   └── application.yml                       # 应用配置 (外部化)
│
├── src/main/script/
│   └── assembly.xml                          # Maven打包配置
│
└── target/
    └── synch-data-pvg.jar                   # 打包输出
```

### 1.3 业务定位

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     synch-data-pvg 业务定位                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                         浦东数据同步中心                              │   │
│   │                                                                      │   │
│   │   ┌─────────────┐  ┌─────────────┐  ┌───────────────────────────┐  │   │
│   │   │  油单上传   │  │  任务上传   │  │  结算状态同步           │  │   │
│   │   │ BILL Upload │  │ Task Upload │  │  Settlement Sync         │  │   │
│   │   └─────────────┘  └─────────────┘  └───────────────────────────┘  │   │
│   │                                                                      │   │
│   │   ┌─────────────┐  ┌─────────────┐  ┌───────────────────────────┐  │   │
│   │   │  失败重试   │  │  记录管理   │  │  数据清理               │  │   │
│   │   │  Retry     │  │  Records    │  │  Cleanup                 │  │   │
│   │   └─────────────┘  └─────────────┘  └───────────────────────────┘  │   │
│   │                                                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                              │                                               │
│                              ▼                                               │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                         数据流向                                     │   │
│   │                                                                      │   │
│   │   本地系统                    总部平台                              │   │
│   │   ┌──────────┐              ┌──────────┐                          │   │
│   │   │  db_oil  │ ──────────▶  │ 总部数据 │                          │   │
│   │   │ _pvg    │  REST API    │  管理平台 │                          │   │
│   │   └──────────┘              └──────────┘                          │   │
│   │                                                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. 技术架构

### 2.1 核心依赖

| 依赖 | 版本 | 用途 |
|------|------|------|
| **Spring Boot** | 2.0.8.RELEASE | 应用框架 |
| **Spring Web** | 内置 | REST服务 |
| **MyBatis** | 3.4.5 | ORM框架 |
| **MyBatis-Spring-Boot** | 1.3.1 | MyBatis集成 |
| **PageHelper** | 1.2.3 | 分页插件 |
| **Druid** | 1.1.0 | 数据库连接池 |
| **Dubbo** | 0.2.0 (boot) | RPC框架 |
| **Zookeeper** | 3.4.14 | 服务注册 |
| **ActiveMQ** | 5.7.0 | 消息队列 |
| **Quartz** | 2.2.1 | 定时任务 |
| **HttpClient** | 4.5.3 | HTTP客户端 |
| **FastJSON** | 1.2.83 | JSON处理 |
| **FreeMarker** | 内置 | 模板引擎 |
| **iTextPDF** | 5.5.9 | PDF生成 |
| **Dom4j** | 2.0.1 | XML处理 |

### 2.2 内部服务依赖

| 服务 | GroupId:ArtifactId | 版本 | 用途 |
|------|-------------------|------|------|
| **flight-oil-service** | com.ias.flight.oil | 1.2.10.21 | 油单服务 |
| **MISGWSAPI** | com.ias.lib.mis | pvg-5.0.3.0 | 用户服务 |
| **security-api** | com.ias.lib | 5.0.2.1 | 安全认证 |
| **flight-services-facade** | com.ias.server.flight | 5.1.2.0 | 航班服务 |
| **common** | com.ias.server.flight | 5.1.2.0 | 公共模块 |

### 2.3 架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           技术架构图                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                     REST API Layer                                   │    │
│  │                                                                      │    │
│  │   ┌─────────────────────────────────────────────────────────────┐   │    │
│  │   │                  SynchController                          │   │    │
│  │   │   /synchConfigs    - 同步配置                              │   │    │
│  │   │   /synchOILBill   - 同步油单                              │   │    │
│  │   │   /uploadBills    - 上传油单                              │   │    │
│  │   │   /retryUploadFailBills - 重试失败油单                    │   │    │
│  │   └─────────────────────────────────────────────────────────────┘   │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                      │                                      │
│                                      ▼                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                     Service Layer                                    │    │
│  │                                                                      │    │
│  │   ┌───────────────────────────┐  ┌───────────────────────────────┐ │    │
│  │   │    BILLUploadManager     │  │      TaskUploadManager        │ │    │
│  │   │    (油单上传服务)        │  │      (任务上传服务)          │ │    │
│  │   │                         │  │                               │ │    │
│  │   │  - process()           │  │  - process()                 │ │    │
│  │   │  - retryUploadFail()  │  │  - retryUploadFail()        │ │    │
│  │   │  - deleteRecords()    │  │  - deleteRecords()          │ │    │
│  │   └───────────────────────────┘  └───────────────────────────────┘ │    │
│  │                                                                      │    │
│  │   ┌───────────────────────────┐  ┌───────────────────────────────┐ │    │
│  │   │  BILLSettlementManager  │  │        ToolManager           │ │    │
│  │   │   (结算同步服务)        │  │       (工具管理)             │ │    │
│  │   └───────────────────────────┘  └───────────────────────────────┘ │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                      │                                      │
│                                      ▼                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                     Message Transfer (ActiveMQ)                     │    │
│  │                                                                      │    │
│  │   ┌─────────────────────────────────────────────────────────────┐   │    │
│  │   │                 IASMessageTransfer                        │   │    │
│  │   │   @JmsListener - 接收任务更新消息                          │   │    │
│  │   │   send() - 发送消息到队列                                 │   │    │
│  │   └─────────────────────────────────────────────────────────────┘   │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                      │                                      │
│                                      ▼                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                     Timer Layer (Quartz)                            │    │
│  │                                                                      │    │
│  │   ┌─────────────────────────────────────────────────────────────┐   │    │
│  │   │                  SynchTimer                              │   │    │
│  │   │   @Scheduled - 定时执行任务                              │   │    │
│  │   │   @EnableAsync - 异步执行                               │   │    │
│  │   └─────────────────────────────────────────────────────────────┘   │    │
│  │                                                                      │    │
│  │   定时任务:                                                         │    │
│  │   - retryUploadFailBills()    - 重试失败油单                     │    │
│  │   - retryUploadFailTasks()    - 重试失败任务                     │    │
│  │   - deleteUploadedRecords()    - 删除已上传记录                   │    │
│  │   - deleteTaskUploadedRecords() - 删除已上传任务记录             │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                      │                                      │
│                                      ▼                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                     External API (HTTP REST)                        │    │
│  │                                                                      │    │
│  │   ┌─────────────────────────────────────────────────────────────┐   │    │
│  │   │                 RestRequestManager                        │   │    │
│  │   │   - uploadOILBill()    - 上传油单到总部平台              │   │    │
│  │   │   - uploadTask()       - 上传任务到总部平台               │   │    │
│  │   └─────────────────────────────────────────────────────────────┘   │    │
│  │                                                                      │    │
│  │   模板引擎:                                                          │    │
│  │   - billForOilHeadquarters.ftl   - 油单模板                        │    │
│  │   - taskForOilHeadquarters.ftl   - 任务模板                       │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                      │                                      │
│                                      ▼                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                     Dubbo Services (RPC)                            │    │
│  │                                                                      │    │
│  │   ┌───────────────────────────┐  ┌───────────────────────────┐   │    │
│  │   │   TerminalModelService   │  │   OilBillNumberService    │   │    │
│  │   │   (航班数据服务)          │  │   (油单号服务)            │   │    │
│  │   │   - getFlightStroeData() │  │   - getTaskOilBillNum()  │   │    │
│  │   └───────────────────────────┘  └───────────────────────────┘   │    │
│  │                                                                      │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                      │                                      │
│                                      ▼                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                     Data Access Layer (MyBatis)                    │    │
│  │                                                                      │    │
│  │   ┌───────────────────────────┐  ┌───────────────────────────────┐ │    │
│  │   │  UploadRecordRepository  │  │  UploadTaskRecordRepository  │ │    │
│  │   │  (上传记录DAO)          │  │  (任务上传记录DAO)          │ │    │
│  │   └───────────────────────────┘  └───────────────────────────────┘ │    │
│  │                                                                      │    │
│  │   ┌───────────────────────────┐  ┌───────────────────────────────┐ │    │
│  │   │   TimeRecordRepository   │  │      OilConfigRepository     │ │    │
│  │   │   (时间戳记录DAO)        │  │      (配置DAO)               │ │    │
│  │   └───────────────────────────┘  └───────────────────────────────┘ │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                      │                                      │
│                                      ▼                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                     Data Storage                                      │    │
│  │                                                                      │    │
│  │   ┌───────────────────────────┐  ┌───────────────────────────────┐   │    │
│  │   │  MySQL (db_oil_pvg)     │  │  Remote Services             │   │    │
│  │   │  - upload_oil_bill_msg  │  │  - 总部数据管理平台          │   │    │
│  │   │  - upload_task_record   │  │  - 结算系统                  │   │    │
│  │   │  - time_record         │  │                             │   │    │
│  │   │  - oil_config          │  │                             │   │    │
│  │   └───────────────────────────┘  └───────────────────────────────┘   │    │
│  │                                                                      │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. 核心数据模型

### 3.1 Bill (油单实体)

```java
public class Bill extends OilBillEntity {
    // 消息头部 (HD)
    private String ms;        // 消息编号, 用于识别每条消息
    private String stm;        // 消息发送时间, 格式: YYYY-MM-DD HH:MI:SS
    
    // 业务数据 (BD)
    private String iud;       // 数据更新类型: I-完整数据, U-更新数据, D-删除数据
    private String luts;       // 最后更新时间戳, 格式: YYYY-MM-DD HH:MI:SS.nnn
    
    // 扩展字段
    private String efs;       // 电子油单(PDF)数据, BASE64编码
    private String stot;      // 航班计划到达/起飞时间
    private String etot;      // 航班变更到达/起飞时间
    private String fsop;      // 油单生成时间
    private String wkop;     // 任务开始时间
    
    // 关联信息
    private String al4c;     // 航空公司4码
    private String orgnm;    // 始发站名称
    private String desnm;    // 目的站名称
    private String via3;     // 经停站3码
    
    // 内部字段
    private String message;  // 原始消息JSON
    private int replyCount;  // 重试次数
}
```

### 3.2 SynTask (同步任务实体)

```java
public class SynTask extends TaskEntity {
    // 消息头部
    private String ms;        // 消息编号
    private String stm;        // 消息发送时间
    
    // 业务数据
    private String iud;       // 数据更新类型
    
    // 航班信息
    private String ap3c;      // 机场3码
    private String ac3c;      // 机型3码
    private String acname;    // 机型名称
    private String al2c;      // 航空公司2码
    private String alcname;   // 航空公司名称
    private String placecode; // 机位
    
    // 时间信息
    private String etot;      // 计划到达/起飞时间
    private String stot;      // 计划时间
    private String atot;      // 实际到达/起飞时间
    
    // 关联信息
    private String message;    // 原始消息
    private int replyCount;   // 重试次数
}
```

### 3.3 UploadRecord (上传记录)

```java
public class UploadRecord {
    // 上传状态常量
    public static final String UPLOAD_STATUS_SUCCESS = "S";  // 上传成功
    public static final String UPLOAD_STATUS_FAIL = "F";      // 上传失败
    public static final String UPLOAD_STATUS_NO = "N";        // 禁止上传
    
    public static final String SIGN_UPLOADED = "2";          // 签名已上传
    public static final String SIGN_NOT_UPLOADED = "1";      // 签名未上传
    
    // 核心字段
    private String id;          // 主键
    private String billNumber;  // 油单号
    private String version;     // 版本号
    private String message;     // 消息内容
    private String replyCount;  // 重试次数
    private String tenant;      // 租户
    private String createTime;  // 创建时间
    private String modifyTime;  // 修改时间
    
    // 状态字段
    private String uploadStatus;  // 上传状态 (S/F/N)
    private String signStatus;    // 签名状态 (0/1/2)
    private String signFile;      // 签名文件路径
    private String failReason;   // 失败原因
    
    // 清理字段
    private String starDate;     // 开始日期
}
```

### 3.4 UploadTaskRecord (任务上传记录)

```java
public class UploadTaskRecord {
    // 核心字段
    private String id;         // 主键
    private String tnb;        // 任务号
    private String tenant;     // 租户
    private String fkey;       // 航班唯一号
    private String pcid;       // 进程ID
    
    // 航班信息
    private String flop;       // 航班日
    private String flno;       // 航班号
    private String adid;       // 进离港
    private String regn;       // 机号
    private String ac3c;       // 机型
    private String acname;     // 机型名称
    
    // 服务信息
    private String rtyp;      // 服务类型
    private String rtnm;      // 服务名称
    
    // 资源信息
    private String vnb;       // 车号
    private String ap3c;      // 机场3码
    private String al2c;      // 航司2码
    private String alcname;   // 航司名称
    
    // 油单信息
    private String billnumber; // 油单号
    private String placecode;   // 机位
    
    // 时间节点
    private String tdwn;       // 派发时间
    private String tget;       // 接收时间
    private String tacp;       // 接受时间
    private String tarv;       // 到位时间
    private String tbeg;       // 开始时间
    private String tprt;       // 作业时间
    private String tend;       // 结束时间
    private String tcal;       // 取消时间
    private String tdec;       // 延误时间
    private String tsto;       // 停止时间
    
    // 其他
    private String etot;      // 计划时间
    private String stot;       // 实际时间
    private String atot;      // 到达时间
    private String refuelSchedule; // 加油计划
    private String addorupdate;   // 添加/更新标识
    private String replyCount;    // 重试次数
    private String uploadStatus;  // 上传状态
    private String failReason;   // 失败原因
}
```

### 3.5 数据库表结构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         同步模块数据库表                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────┐    │
│  │                     upload_oil_bill_message                       │    │
│  │   (油单上传消息记录表)                                            │    │
│  ├───────────────────────────────────────────────────────────────────┤    │
│  │   id          VARCHAR(50)    主键                                │    │
│  │   bill_number VARCHAR(50)    油单号                              │    │
│  │   version     VARCHAR(10)    版本号                              │    │
│  │   message     TEXT           消息内容                            │    │
│  │   reply_count INT            重试次数                          │    │
│  │   tenant     VARCHAR(20)     租户ID                             │    │
│  │   create_time DATETIME       创建时间                            │    │
│  │   modify_time DATETIME       修改时间                            │    │
│  │   upload_status VARCHAR(10)  上传状态 (S/F/N)                  │    │
│  │   sign_status VARCHAR(10)    签名状态 (0/1/2)                   │    │
│  │   sign_file   VARCHAR(255)   签名文件路径                        │    │
│  │   fail_reason VARCHAR(500)   失败原因                            │    │
│  └───────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────┐    │
│  │                     upload_task_record                            │    │
│  │   (任务上传记录表)                                               │    │
│  ├───────────────────────────────────────────────────────────────────┤    │
│  │   id          VARCHAR(50)    主键                                │    │
│  │   tnb         VARCHAR(50)    任务号                              │    │
│  │   tenant     VARCHAR(20)     租户ID                             │    │
│  │   fkey       VARCHAR(50)     航班唯一号                          │    │
│  │   ... (航班、任务、资源信息字段)                                 │    │
│  │   upload_status VARCHAR(10)  上传状态                           │    │
│  │   fail_reason VARCHAR(500)   失败原因                            │    │
│  └───────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────┐    │
│  │                     time_record                                  │    │
│  │   (时间戳记录表)                                                 │    │
│  ├───────────────────────────────────────────────────────────────────┤    │
│  │   id          BIGINT         主键                                │    │
│  │   type        VARCHAR(20)     记录类型                           │    │
│  │   tenant     VARCHAR(20)     租户ID                             │    │
│  │   last_time   DATETIME       最后时间戳                          │    │
│  │   modifier   VARCHAR(50)     修改人                             │    │
│  │   modify_time DATETIME       修改时间                            │    │
│  │   remark     VARCHAR(255)   备注                                │    │
│  └───────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────┐    │
│  │                     oil_config                                   │    │
│  │   (配置表)                                                       │    │
│  ├───────────────────────────────────────────────────────────────────┤    │
│  │   id          BIGINT         主键                                │    │
│  │   config_key  VARCHAR(100)   配置键                              │    │
│  │   config_value VARCHAR(500)   配置值                             │    │
│  │   tenant     VARCHAR(20)     租户ID                             │    │
│  │   remark     VARCHAR(255)   备注                                │    │
│  └───────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. 核心业务逻辑详解

### 4.1 油单上传流程 (BILLUploadManager)

```java
@Service
public class BILLUploadManagerImpl extends CommonService implements BILLUploadManager {
    
    @Value("${bill.upload.host}")
    private String host;           // 总部平台地址
    
    @Value("${bill.upload.tenants}")
    private String tenants;       // 租户列表
    
    @Value("${bill.upload.maxRetries}")
    private String maxRetries;     // 最大重试次数
    
    /**
     * 处理油单上传请求
     * 流程: 解析消息 -> 填充数据 -> 发送到总部平台 -> 更新状态
     */
    @Override
    public String process(String message) throws OILException {
        // 1. 解析JSON消息为Bill对象
        Bill bill = OILUtils.convertBean(Bill.class, message);
        bill.setMessage(message);
        
        // 2. 获取上传记录
        UploadRecord record = getUploadedRecord(bill);
        
        // 3. 执行上传
        return upload(bill, record);
    }
    
    /**
     * 核心上传逻辑
     */
    private String upload(Bill bill, UploadRecord record) {
        try {
            // 1. 填充油单数据 (机场名称、航司名称等)
            fillOILBill(bill);
            
            // 2. 发送到总部平台
            uploadOILBillToPlatform(bill);
            
            // 3. 更新上传成功状态
            updateRecordAfterUpload(record);
            
            return "success";
        } catch (OILException e) {
            logger.error("上传油单失败: {}", e.getMessage());
            // 4. 更新上传失败状态
            updateRecordAfterFail(record);
            return "error";
        }
    }
    
    /**
     * 填充油单数据
     * - 根据机场代码获取机场名称
     * - 根据航司代码获取航司名称
     * - 格式化数据字段
     */
    private void fillOILBill(Bill bill) {
        // 1. 获取始发站名称
        if (StringUtils.isNotEmpty(bill.getOrg3())) {
            CAirPort org = toolManager.AIRPORTS3.get(bill.getOrg3());
            if (org != null) {
                bill.setOrgnm(org.getCity());
            }
        }
        
        // 2. 获取目的站名称
        if (StringUtils.isNotEmpty(bill.getDes3())) {
            CAirPort des = toolManager.AIRPORTS3.get(bill.getDes3());
            if (des != null) {
                bill.setDesnm(des.getCity());
            }
        }
        
        // 3. 获取经停站信息
        if (StringUtils.isNotEmpty(bill.getStopOver())) {
            CAirPort sto = toolManager.AIRPORTSCITY.get(bill.getStopOver());
            if (sto != null) {
                bill.setVia3(sto.getAp3c());
            }
        }
        
        // 4. 格式化字段
        if (StringUtils.isEmpty(bill.getFlno())) {
            bill.setFlno("0000");
        }
        if (StringUtils.isEmpty(bill.getAcname())) {
            bill.setAcname("0000");
        }
        if (StringUtils.isEmpty(bill.getRegnumber())) {
            bill.setRegnumber("0000");
        }
        
        // 5. 设置消息元数据
        bill.setIud("I");  // I-完整数据
        bill.setMs(OILUtils.getId());  // 消息编号
        bill.setStm(getDateHelper().getCurrentDateTime());
        bill.setLuts(String.format("%s.000", bill.getModifyTime()));
    }
    
    /**
     * 发送到总部数据管理平台
     */
    private void uploadOILBillToPlatform(Bill bill) throws OILException {
        logger.info("上传油单到总部平台: number={}, version={}", 
                   bill.getBillNumber(), bill.getVersion());
        
        // 1. 使用FreeMarker模板生成XML
        String xml = toolManager.getMessageTemplate(bill, "billForOilHeadquarters.ftl");
        
        // 2. 发送HTTP请求
        String response = restRequestManager.uploadOILBill(host, xml);
        
        // 3. 解析响应
        BillUploadXMLDocument document = new BillUploadXMLDocument(response);
        if (!"S".equals(document.getStatus())) {
            throw new OILException("上传失败: " + document.getReason());
        }
    }
}
```

### 4.2 任务上传流程 (TaskUploadManager)

```java
@Service
public class TaskUploadManagerImpl extends CommonService implements TaskUploadManager {
    
    @Value("${task.upload.host}")
    private String host;           // 总部平台地址
    
    @Value("${task.upload.sendRule}")
    private String sendRule;       // 发送规则
    
    /**
     * 处理任务上传
     * 支持多种发送规则:
     * 0 - 全不发送
     * 1 - 仅发送任务节点变化
     * 2 - 仅发送加油计划变化
     * 3 - 发送节点和加油计划变化
     */
    @Override
    public void process(String message) throws OILException {
        // 检查发送规则
        if (StringUtils.equals("0", sendRule)) {
            return;  // 不发送
        }
        
        // 1. 解析任务消息
        SynTask synTask = getSynTask(message);
        
        // 2. 获取航班信息
        FlightStroeData flightinfo = terminalModelService
            .getFlightStroeDataByTenantAndFlseq(
                synTask.getTenantId(), synTask.getFkey());
        
        // 3. 填充航班数据
        synTask.setRegn(flightinfo.getRegnumber());
        synTask.setAc3c(flightinfo.getAc3c());
        synTask.setAcname(flightinfo.getAcname());
        synTask.setAp3c(flightinfo.getAp3c());
        synTask.setAl2c(flightinfo.getAl2c());
        synTask.setPlacecode(flightinfo.getPlacecode());
        
        // 4. 获取油单号
        TaskOilBillNumberEntity taskOilBill = 
            oilBillNumberService.getTaskOilBillNumberEntity(
                synTask.getTnb(), synTask.getTenantId());
        if (taskOilBill != null) {
            synTask.setBillNumber(taskOilBill.getOilBillNumber());
        }
        
        // 5. 获取航司信息
        setTaskAlcname(synTask, flightinfo);
        
        // 6. 根据发送规则判断是否上传
        String pcids = configs.get(getCode(synTask.getTenantId(), "task.upload.config"));
        if (StringUtils.isNotEmpty(pcids) && StringUtils.contains(pcids, synTask.getPcid())) {
            uploadTask(synTask);
        }
    }
    
    /**
     * 上传任务
     */
    private void uploadTask(SynTask synTask) throws OILException {
        // 1. 查询已上传记录
        UploadTaskRecord record = getDefaultRecord(synTask);
        UploadTaskRecord target = getUploadedRecordFromDB(record);
        
        if (target != null) {
            // 2. 根据发送规则判断是否需要更新
            if (shouldUpdate(record, target)) {
                updateAndUpload(record);
            }
        } else {
            // 3. 新增并上传
            record.setAddorupdate("A");
            addUploadRecord(record);
            upload(record);
        }
    }
    
    /**
     * 判断是否需要更新上传
     */
    private boolean shouldUpdate(UploadTaskRecord record, UploadTaskRecord target) {
        if (StringUtils.equals("1", sendRule)) {
            // 规则1: 检查节点变化
            return !StringUtils.equals(record.getPcid(), target.getPcid());
        } else if (StringUtils.equals("2", sendRule)) {
            // 规则2: 检查加油计划变化
            return !StringUtils.equals(record.getRefuelSchedule(), target.getRefuelSchedule());
        } else if (StringUtils.equals("3", sendRule)) {
            // 规则3: 检查节点或加油计划变化
            return !StringUtils.equals(record.getPcid(), target.getPcid()) 
                || !StringUtils.equals(record.getRefuelSchedule(), target.getRefuelSchedule());
        }
        return false;
    }
}
```

### 4.3 失败重试机制

```java
@Service
public class BILLUploadManagerImpl extends CommonService implements BILLUploadManager {
    
    /**
     * 重新上传失败的任务 (定时任务调用)
     */
    @Override
    public void retryUploadFailBills() throws OILException {
        time = getCurrentTime();
        
        // 1. 遍历所有租户
        for (String tenant : getTenants()) {
            retryUploadFailBillsByTenant(tenant);
        }
    }
    
    /**
     * 按租户重试失败油单
     */
    private void retryUploadFailBillsByTenant(String tenant) {
        // 1. 查询失败的记录 (重试次数 < 最大重试次数)
        Map<String, List<UploadRecord>> records = getUploadedFailRecordsForMap(tenant);
        
        // 2. 按油单号分组处理
        for (String number : records.keySet()) {
            List<UploadRecord> recordsForBill = records.get(number);
            
            // 3. 重置历史版本状态
            resetHistoryVersionBills(recordsForBill);
            
            // 4. 获取最新版本
            UploadRecord record = getNewestVersionBill(recordsForBill);
            if (record == null) {
                continue;
            }
            
            // 5. 增加重试次数
            int replyCount = Integer.valueOf(record.getReplyCount()) + 1;
            record.setReplyCount(String.valueOf(replyCount));
            
            // 6. 重新上传
            Bill bill = getBill(record.getMessage());
            upload(bill, record);
        }
    }
    
    /**
     * 重置历史版本状态
     * 如果存在多个版本，只保留最新版本继续重试
     */
    private void resetHistoryVersionBills(List<UploadRecord> records) {
        if (records.size() <= 1) {
            return;  // 只有一个版本，无需处理
        }
        
        List<UploadRecord> historyBills = getHistoryVersionBills(records);
        for (UploadRecord bill : historyBills) {
            // 将历史版本标记为"禁止上传"
            bill.setUploadStatus(UploadRecord.UPLOAD_STATUS_NO);
            updateUploadedRecord(bill);
        }
    }
    
    /**
     * 获取历史版本列表
     */
    private List<UploadRecord> getHistoryVersionBills(List<UploadRecord> records) {
        List<UploadRecord> historyBills = new ArrayList<>();
        for (int i = HISTORY_BILL_INDEX; i < records.size(); i++) {
            historyBills.add(records.get(i));
        }
        return historyBills;
    }
}
```

### 4.4 数据清理机制

```java
@Service
public class BILLUploadManagerImpl extends CommonService implements BILLUploadManager {
    
    /**
     * 清理已上传成功的历史记录
     * 定时任务调用
     */
    @Override
    public void deleteUploadedRecords() throws OILException {
        for (String tenant : getTenants()) {
            // 清理指定天数前的记录
            removeHistoryUploadBills(tenant, day);
        }
    }
    
    /**
     * 删除历史上传记录
     */
    private void removeHistoryUploadBills(String tenant, int day) {
        UploadRecord record = new UploadRecord();
        record.setTenant(tenant);
        
        // 计算清理截止日期 (当前日期 - 配置天数)
        record.setStarDate(getDateHelper().getNextDate(getCurrentDate(), -day));
        record.setUploadStatus(UploadRecord.UPLOAD_STATUS_SUCCESS);
        
        logger.info("清理已上传成功的历史油单: tenant={}, date={}", 
                   tenant, record.getStarDate());
        
        try {
            uploadRecordRepository.deleteUploadedRecords(record);
        } catch (Exception e) {
            logger.error("删除历史油单失败: {}", e.getMessage());
        }
    }
}
```

### 4.5 定时任务调度 (SynchTimer)

```java
@Component
@EnableScheduling
@EnableAsync
public class SynchTimer {
    
    /**
     * 重试上传失败油单
     * 定时调用: bill.upload.option.replytimer
     */
    @Async
    @Scheduled(cron = "${bill.upload.option.replytimer}")
    public void retryUploadFailBills() {
        if (!Boolean.valueOf(isUploadBill)) {
            return;
        }
        billUploadManager.retryUploadFailBills();
    }
    
    /**
     * 重试上传失败任务
     * 定时调用: task.upload.option.replytimer
     */
    @Async
    @Scheduled(cron = "${task.upload.option.replytimer}")
    public void retryUploadFailTasks() {
        if (!Boolean.valueOf(isUploadTask)) {
            return;
        }
        taskUploadManager.retryUploadFailTasks();
    }
    
    /**
     * 清理已上传的油单记录
     * 定时调用: bill.upload.option.deletetimer
     */
    @Async
    @Scheduled(cron = "${bill.upload.option.deletetimer}")
    public void deleteUploadedRecords() {
        if (!Boolean.valueOf(isUploadBill)) {
            return;
        }
        billUploadManager.deleteUploadedRecords();
    }
    
    /**
     * 清理已上传的任务记录
     * 定时调用: task.upload.option.deletetimer
     */
    @Async
    @Scheduled(cron = "${task.upload.option.deletetimer}")
    public void deleteTaskUploadedRecords() {
        if (!Boolean.valueOf(isUploadTask)) {
            return;
        }
        taskUploadManager.deleteUploadedTaskRecords();
    }
}
```

---

## 5. REST API 接口

### 5.1 SynchController

```java
@RestController
public class SynchController {
    
    /**
     * GET /synchConfigs
     * 手动同步系统配置信息
     */
    @GetMapping(value = "/synchConfigs")
    public Map<String, String> synchConfigs() {
        toolManager.loadConfigs();
        return toolManager.getConfigs();
    }
    
    /**
     * GET /synchOILBill
     * 手动同步油单确认结果
     */
    @GetMapping(value = "/synchOILBill")
    public void synchOILBill() {
        BILLSettlementManager.synchOILBills();
    }
    
    /**
     * POST /uploadBills
     * 上传油单到总部平台
     * 
     * @param billjson 油单JSON数据
     * @return { "code": "200/500", "msg": "success/error" }
     */
    @PostMapping(value = "/uploadBills")
    public String uploadBills(@RequestBody String billjson) {
        String msg = billUploadManager.process(billjson);
        
        JSONObject result = new JSONObject();
        result.put("msg", msg);
        result.put("code", "success".equals(msg) ? "200" : "500");
        
        return result.toJSONString();
    }
    
    /**
     * GET /retryUploadFailBills
     * 手动触发重试上传失败油单
     */
    @GetMapping(value = "/retryUploadFailBills")
    public void retryUploadFailBills() {
        billUploadManager.retryUploadFailBills();
    }
}
```

---

## 6. 消息传输机制

### 6.1 ActiveMQ 消息监听

```java
@Component
public class IASMessageTransfer {
    
    @Autowired
    private JmsMessagingTemplate iasTransferTemplate;
    
    @Autowired
    private BILLUploadManager billUploadManager;
    
    @Autowired
    private TaskUploadManager taskUploadManager;
    
    /**
     * 监听任务更新消息
     * 队列: activemq.ias.option.queues.fromTaskUpd
     */
    @JmsListener(destination = "${activemq.ias.option.queues.fromTaskUpd}")
    private void receiveTaskUpd(final Message message) throws JMSException {
        String text = ((TextMessage) message).getText();
        logger.info("接收任务节点数据: {}", text);
        
        try {
            // 处理任务上传
            taskUploadManager.process(text);
        } catch (OILException e) {
            logger.error("处理任务节点数据失败: {}", e.getMessage());
        }
    }
    
    /**
     * 发送消息到指定队列
     */
    public boolean send(Destination destination, final String message) {
        logger.info("发送消息到队列: {}, 内容: {}", destination, message);
        try {
            iasTransferTemplate.convertAndSend(destination, message);
            return true;
        } catch (MessagingException e) {
            logger.error("发送消息失败: {}", e.getFailedMessage());
            return false;
        }
    }
}
```

---

## 7. 配置文件

### 7.1 application.yml 配置项

```yaml
# 油单上传配置
bill:
  upload:
    host: http://总部平台地址/oil/uploadBill    # 上传地址
    tenants: tenant1,tenant2                  # 租户列表
    maxRetries: 3                               # 最大重试次数
    option:
      enable: true                             # 是否启用上传
      day: 30                                   # 数据保留天数
      replytimer: "0 0 0 * * ?"               # 重试任务Cron (每天0点)
      deletetimer: "0 0 1 * * ?"              # 清理任务Cron (每天1点)

# 任务上传配置
task:
  upload:
    host: http://总部平台地址/task/uploadTask  # 上传地址
    tenants: tenant1,tenant2                    # 租户列表
    maxRetries: 3                              # 最大重试次数
    option:
      enable: true                             # 是否启用上传
      day: 30                                   # 数据保留天数
      replytimer: "0 0 0 * * ?"               # 重试任务Cron
      deletetimer: "0 0 1 * * ?"              # 清理任务Cron
      hourConf: 24                             # 小时配置
      hourConf: 24                             # 忽略时间配置

# ActiveMQ配置
activemq:
  ias:
    option:
      queues:
        fromTaskUpd: queue_task_upd            # 任务更新队列

# REST配置
rest:
  template:
    connectTimeout: 30000                      # 连接超时 (30秒)
```

---

## 8. 数据流分析

### 8.1 油单上传数据流

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         油单上传数据流                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────┐                                                               │
│   │ 油单数据 │  JSON格式                                                       │
│   └────┬────┘                                                               │
│        │                                                                      │
│        ▼                                                                      │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                      SynchController                                 │   │
│   │                 POST /uploadBills                                  │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│        │                                                                      │
│        ▼                                                                      │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                   BILLUploadManager.process()                       │   │
│   │   1. 解析JSON为Bill对象                                              │   │
│   │   2. 获取或创建上传记录                                               │   │
│   │   3. 填充数据(机场名称、航司名称)                                     │   │
│   │   4. 生成消息模板                                                     │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│        │                                                                      │
│        ▼                                                                      │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                   RestRequestManager                                  │   │
│   │              HTTP POST 上传到总部平台                                │   │
│   │                                                                      │   │
│   │   Content-Type: application/xml                                     │   │
│   │   请求体: XML格式的油单数据 (FreeMarker模板生成)                     │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│        │                                                                      │
│        ▼                                                                      │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                   总部平台响应                                        │   │
│   │   <response>                                                         │   │
│   │     <status>S/F</status>    <!-- S-成功, F-失败 -->               │   │
│   │     <reason>失败原因</reason>                                        │   │
│   │   </response>                                                        │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│        │                                                                      │
│        ▼                                                                      │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                   更新上传记录                                        │   │
│   │   upload_oil_bill_message 表                                         │   │
│   │   - upload_status = 'S' (成功) / 'F' (失败)                        │   │
│   │   - fail_reason = 失败原因 (如果失败)                                │   │
│   │   - modify_time = 当前时间                                           │   │
│   │   - reply_count += 1 (重试次数)                                     │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                   定时任务处理                                        │   │
│   │                                                                      │   │
│   │   retryUploadFailBills() - 重试失败油单                              │   │
│   │   deleteUploadedRecords() - 清理历史记录                             │   │
│   │                                                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 8.2 任务上传数据流

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         任务上传数据流                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────┐                                                               │
│   │ 任务数据 │  JSON格式 (从ActiveMQ队列接收)                                │
│   └────┬────┘                                                               │
│        │                                                                      │
│        ▼                                                                      │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │              IASMessageTransfer.receiveTaskUpd()                    │   │
│   │                 @JmsListener 监听队列                               │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│        │                                                                      │
│        ▼                                                                      │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                TaskUploadManager.process()                          │   │
│   │   1. 解析JSON为SynTask对象                                           │   │
│   │   2. 通过Dubbo获取航班信息                                           │   │
│   │   3. 获取油单号                                                       │   │
│   │   4. 获取航司信息                                                     │   │
│   │   5. 根据发送规则判断是否上传                                         │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│        │                                                                      │
│        ▼                                                                      │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                   发送规则判断                                       │   │
│   │                                                                      │   │
│   │   sendRule=0: 不发送                                               │   │
│   │   sendRule=1: 节点变化时发送                                        │   │
│   │   sendRule=2: 加油计划变化时发送                                    │   │
│   │   sendRule=3: 节点或加油计划变化时发送                              │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│        │                                                                      │
│        ▼ (需要上传时)                                                       │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                   RestRequestManager                                  │   │
│   │              HTTP POST 上传到总部平台                                │   │
│   │   模板: taskForOilHeadquarters.ftl                                   │   │
│   │   包含: 任务信息 + 航班信息 + 时间节点                                │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│        │                                                                      │
│        ▼                                                                      │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                   更新上传记录                                        │   │
│   │   upload_task_record 表                                              │   │
│   │   - upload_status = 'S' / 'F'                                       │   │
│   │   - addorupdate = 'A' (新增) / 'U' (更新)                          │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 9. 问题与优化建议

### 9.1 技术债务

| 问题 | 严重程度 | 说明 | 建议 |
|------|----------|------|------|
| **Spring Boot 2.0.8 过时** | 🔴 高 | 建议升级到 2.7.x | 升级到 2.7.x LTS |
| **Dubbo 0.2.0 boot** | 🟡 中 | 版本较老 | 升级到 2.7.x |
| **Quartz 2.2.1** | 🟡 中 | 版本较老 | 升级到 2.3.x |
| **iTextPDF 5.5.9** | 🟡 中 | 安全版本升级 | 升级到 5.5.13.x |
| **HttpClient 4.5.3** | 🟡 中 | 建议升级到 4.5.14 | 修复安全漏洞 |
| **FastJSON 1.2.83** | 🟡 中 | 安全版本 | 升级到 1.2.87+ |
| **Dom4j 2.0.1** | 🟡 中 | 版本较新但需要验证兼容性 | 保持当前版本 |

### 9.2 代码质量问题

```java
// 问题1: 大量硬编码的常量
public static final String UPLOAD_STATUS_SUCCESS = "S";
public static final String UPLOAD_STATUS_FAIL = "F";
public static final String UPLOAD_STATUS_NO = "N";

// 优化建议: 移到配置文件中
// upload.status.success=S

// 问题2: 方法过长
// BILLUploadManagerImpl.java - 490行
// TaskUploadManagerImpl.java - 531行

// 优化建议: 拆分为多个私有方法
private void fillAirportInfo(Bill bill) { ... }
private void formatBillFields(Bill bill) { ... }

// 问题3: 异常处理不完整
} catch (Exception e) {
    logger.error(e.getMessage());  // 丢失堆栈信息
}

// 优化建议: 记录完整异常信息
} catch (Exception e) {
    logger.error("上传油单失败", e);
    throw new OILException("上传失败", e);
}

// 问题4: 魔法数字
for (int i = HISTORY_BILL_INDEX; i < records.size(); i++) {
    historyBills.add(records.get(i));
}

// 优化建议: 使用有意义的常量名
private static final int FIRST_HISTORY_INDEX = 1;
```

### 9.3 性能问题

| 问题 | 描述 | 建议 |
|------|------|------|
| **同步HTTP调用** | 上传操作同步等待，可能阻塞 | 考虑使用异步HTTP |
| **定时任务全量扫描** | 重试任务时扫描所有失败记录 | 优化查询条件，添加时间限制 |
| **数据清理策略** | 定时清理可能产生锁表 | 分批删除，使用LIMIT |
| **缺少缓存** | 频繁查询配置数据 | 添加本地缓存或Redis缓存 |

```java
// 问题: 全量扫描失败记录
private List<UploadRecord> getUploadedFailRecords(String tenant) {
    // 缺少时间限制，可能扫描大量数据
    return uploadRecordRepository.getUploadedFailRecords(record);
}

// 优化建议: 添加时间范围限制
<select id="getUploadedFailRecords" ...>
    WHERE create_time >= #{startTime}
    AND reply_count &lt; #{maxRetries}
    ...
</select>
```

### 9.4 安全问题

| 问题 | 描述 | 建议 |
|------|------|------|
| **HTTP明文传输** | 使用HTTP而非HTTPS | 升级为HTTPS |
| **敏感日志** | 油单消息可能包含敏感信息 | 脱敏处理 |
| **缺少限流** | API无频率限制 | 添加限流机制 |
| **错误信息泄露** | 异常信息返回给前端 | 统一错误处理 |

### 9.5 可用性问题

| 问题 | 描述 | 建议 |
|------|------|------|
| **重试机制** | 无人工干预入口 | 添加管理界面 |
| **监控缺失** | 无运行状态监控 | 添加健康检查 |
| **告警缺失** | 失败无告警通知 | 集成告警机制 |

---

## 10. 总结

### 10.1 模块特点

| 特点 | 说明 |
|------|------|
| **数据同步** | 将本地油单/任务数据同步到航油总部平台 |
| **失败重试** | 支持失败任务自动重试，最大重试次数可配置 |
| **定时任务** | 使用Quartz实现定时重试和清理任务 |
| **异步处理** | 使用@Async注解实现异步执行 |
| **消息队列** | 通过ActiveMQ接收任务更新消息 |
| **模板生成** | 使用FreeMarker生成XML/JSON请求体 |
| **多租户** | 支持按租户配置和隔离 |

### 10.2 核心能力

1. **油单上传**：`BILLUploadManager` - 将本地油单数据上传到航油总部平台
2. **任务上传**：`TaskUploadManager` - 将任务执行数据上传到航油总部平台
3. **结算同步**：`BILLSettlementManager` - 同步油单结算状态
4. **失败重试**：定时扫描失败记录并重试上传
5. **数据清理**：定时清理已上传成功的历史记录

### 10.3 依赖关系

```
synch-data-pvg
├── Spring Boot Web (REST服务)
├── MyBatis (数据访问)
├── ActiveMQ (消息接收)
├── Quartz (定时任务)
├── RestTemplate (HTTP客户端)
├── FreeMarker (模板引擎)
│
└── Dubbo Services (RPC调用)
    ├── TerminalModelService (航班数据)
    ├── OilBillNumberService (油单号)
    └── OilBillService (油单服务)
```

### 10.4 后续建议

1. **架构升级**：
   - Spring Boot 2.0.8 → 2.7.x
   - 评估是否迁移到 Spring Cloud

2. **可观测性**：
   - 添加 Prometheus 指标
   - 集成链路追踪
   - 完善日志规范

3. **高可用**：
   - 实现上传任务的幂等性
   - 添加熔断降级机制
   - 考虑引入消息队列做异步缓冲

4. **运维能力**：
   - 添加管理界面
   - 集成告警通知
   - 完善监控大盘

---

*报告生成时间：2026-02-04*
*分析工具：Cursor AI Code Analysis*
