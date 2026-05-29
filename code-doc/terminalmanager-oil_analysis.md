# terminalmanager-oil 终端管理（油单）模块代码分析报告

> 分析日期：2026-02-04
> 项目路径：e:\work\code\oil\pudong\terminalmanager-oil\
> 文档版本：1.0

---

## 一、项目概述

### 1.1 基本信息

本模块是浦东机场供油调度系统中专门针对油单业务设计的终端管理模块，采用Maven多模块架构构建。与通用的terminalmanager模块相比，terminalmanager-oil模块专注于油品供应全流程的终端管理功能，包括油单生成、任务分配、油品化验、压力差监控、航班油单登记等核心业务。该模块在原有终端管理架构的基础上，针对油品供应场景进行了深度定制和扩展，集成了油品业务特有的数据模型和服务调用，实现了从任务下发到油品交付的完整业务闭环。

```
┌─────────────────────────────────────────────────────────────────────────────┐
│               terminalmanager-oil 业务定位                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                    油品供应终端管理层                                 │   │
│   │                                                                      │   │
│   │   核心职责：                                                         │   │
│   │   ① 油单全生命周期管理                                              │   │
│   │   ② 油品化验数据采集                                                │   │
│   │   ③ 压力差监控与预警                                                │   │
│   │   ④ 航班油单登记                                                    │   │
│   │   ⑤ 车辆信息管理                                                    │   │
│   │   ⑥ 电子签名与文件上传                                              │   │
│   │                                                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                              │                                               │
│                              ▼                                               │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                      业务服务层                                       │   │
│   │                                                                      │   │
│   │   ┌─────────────┬─────────────┬─────────────┬─────────────┐        │   │
│   │   │  油单管理   │ 化验管理    │ 压力监控    │ 任务管理    │        │   │
│   │   │  Service    │  Service   │  Service   │  Service   │        │   │
│   │   └─────────────┴─────────────┴─────────────┴─────────────┘        │   │
│   │                                                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                              │                                               │
│                              ▼                                               │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                      外部服务集成                                       │   │
│   │                                                                      │   │
│   │   · flight-oil-service  (油品业务服务)                               │   │
│   │   · bill-rpc           (油单RPC服务)                                │   │
│   │   · TerminalModelService (航班模型服务)                              │   │
│   │   · MIS服务           (用户/车辆/部门信息)                          │   │
│   │                                                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 模块结构

本模块采用Maven多模块设计，包含3个子模块，各自承担特定的功能职责。父模块terminalmanager-oil定义了模块的版本标识（5.0.0.4）和子模块声明，版本号低于通用terminalmanager模块（7.0.1.0），表明这是针对特定业务场景的定制版本。terminal子模块是主模块，包含214个Java源文件，涵盖了业务服务、REST API、数据模型等核心功能。messagedispatch模块负责消息分发和路由处理。bsod模块定义了基础数据对象，包括消息实体、配置实体等核心数据结构。

```
terminalmanager-oil/
├── pom.xml                                          # 父POM，共13行
│
├── terminal/                                        # 主业务模块
│   ├── pom.xml                                     # 共1046行
│   ├── Dockerfile
│   ├── src/main/
│   │   ├── java/                                   # 214个Java源文件
│   │   ├── assembly/                               # 打包配置
│   │   │   ├── assembly.xml
│   │   │   ├── bin/                                # 启动脚本
│   │   │   └── conf/                               # 配置文件
│   │   └── resources/
│   │       ├── config.properties                   # 主配置
│   │       ├── jdbc_mysql.properties               # 数据库配置
│   │       ├── logback.xml                        # 日志配置
│   │       ├── dozer/                              # Dozer映射配置
│   │       ├── fdfs_client.conf                   # FastDFS配置
│   │       ├── META-INF/spring/                   # Spring配置
│   │       └── templates/                          # 消息模板（85个文件）
│   │           ├── bill/                          # 油单模板
│   │           ├── gwsm/                          # 工作短语模板
│   │           ├── ttud/                          # 任务更新模板
│   │           └── ...                            # 其他业务模板
│   └── src/test/
│       └── java/
│
├── messagedispatch/                                # 消息分发模块
│   ├── pom.xml                                     # 共258行
│   └── src/main/java/                              # 15个Java源文件
│       └── META-INF/spring/
│           └── applicationContext.xml
│
└── bsod/                                           # 基础数据对象模块
    ├── pom.xml                                     # 共93行
    └── src/main/java/                              # 14个Java源文件
        └── resources/
            ├── dbmigrate/                         # 数据库迁移脚本
            ├── mapper/                             # MyBatis映射文件
            └── META-INF/spring/                   # Spring配置
```

### 1.3 版本与依赖关系

模块的版本管理体现了清晰的演进策略。父模块版本为5.0.0.4，子模块版本各异：terminal模块版本为5.1.1.20，bsod模块版本为5.0.2.2，messagedispatch模块版本为5.0.0.3。这种版本差异反映了各模块独立演进的实际情况。与通用terminalmanager模块相比，terminalmanager-oil模块引入了更多与油品业务相关的依赖，包括flight-oil-service（1.2.10.30）和针对油品业务的专用API。版本号中常见的后缀如".20"、".4"等表明模块经过多次迭代和功能增强。

---

## 二、技术架构

### 2.1 核心技术栈

本模块的技术栈与通用terminalmanager模块基本一致，但在某些方面进行了升级和扩展。Spring Framework版本升级到4.2.2.RELEASE，提供了更新的框架特性和bug修复。REST服务采用JBoss RESTEasy 3.0.7.Final框架，支持JAX-RS标准的注解开发方式。消息中间件使用Apache ActiveMQ 5.12.0，支持可靠的JMS消息传输。网络通信方面采用Netty 3.7.0和Apache Mina 1.1.7两种NIO框架。数据存储使用MyBatis 3.2.3进行ORM映射，结合MySQL 5.1.26作为持久化存储。缓存层使用Jedis 2.4.1和Spring Data Redis 1.4.2.RELEASE，支持分布式会话管理。FastDFS分布式文件系统用于存储上传的图片和签名文件。Kryo 2.24.0序列化库提供了高性能的缓存序列化能力。

| 依赖类别 | 技术名称 | 版本 | 主要用途 |
|----------|----------|------|----------|
| **应用框架** | Spring Framework | 4.2.2.RELEASE | IoC容器、AOP |
| **REST框架** | JBoss RESTEasy | 3.0.7.Final | RESTful API |
| **消息路由** | Apache Camel | 2.16.1 | 消息路由、转换 |
| **消息中间件** | Apache ActiveMQ | 5.12.0 | JMS消息传输 |
| **ORM框架** | MyBatis | 3.2.3 | 数据持久化 |
| **缓存** | Redis (Jedis) | 2.4.1 | 分布式缓存 |
| **RPC框架** | Alibaba Dubbo | 2.5.4-IAS | 分布式服务 |
| **文件系统** | FastDFS | 1.27.0.0 | 分布式文件存储 |
| **序列化** | Kryo | 2.24.0 | 高性能序列化 |
| **JSON处理** | FastJSON | 1.2.83 | JSON解析 |
| **JSON处理** | Gson | 2.3.1 | JSON序列化 |
| **日志框架** | Logback | 1.2.3 | 日志记录 |
| **网络通信** | Netty / Apache Mina | 3.7.0 / 1.1.7 | NIO通信 |

### 2.2 架构层次图

本模块的技术架构遵循分层设计原则，从底向上依次为：基础设施层提供数据库访问、缓存管理和文件系统支持；数据持久层通过MyBatis实现数据持久化；业务服务层封装了油品管理的核心业务逻辑；REST API层提供对外的HTTP接口服务；消息处理层通过Apache Camel实现消息路由和分发。各层之间通过Spring IoC容器进行依赖注入，保持了良好的解耦性。

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           技术架构图                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                        REST API层                                       │    │
│  │                                                                      │    │
│  │   ┌─────────────────────────────────────────────────────────────┐   │    │
│  │   │   REST Endpoints (JAX-RS注解)                               │   │    │
│  │   │   · TaskService      (任务服务)                             │   │    │
│  │   │   · OilBillService   (油单服务)                             │   │    │
│  │   │   · FuelPipeOilService (管线油品服务)                       │   │    │
│  │   │   · SecurityService   (安全认证服务)                          │   │    │
│  │   │   · FTPServiceManager (文件上传服务)                         │   │    │
│  │   └─────────────────────────────────────────────────────────────┘   │    │
│  │                                                                      │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                    │                                        │
│                                    ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                       业务服务层                                       │    │
│  │                                                                      │    │
│  │   ┌─────────────────────────────────────────────────────────────┐   │    │
│  │   │                  核心业务服务实现                             │   │    │
│  │   │   · TaskServiceImpl         (任务处理)                     │   │    │
│  │   │   · FuelPipeOilServiceImpl  (油品管输)                     │   │    │
│  │   │   · SecurityServiceImpl      (安全认证)                      │   │    │
│  │   │   · FTPServiceManagerImpl   (文件管理)                      │   │    │
│  │   │   · ConfigManagerImpl       (配置管理)                      │   │    │
│  │   └─────────────────────────────────────────────────────────────┘   │    │
│  │                                                                      │    │
│  │   ┌─────────────────────────────────────────────────────────────┐   │    │
│  │   │                  消息模板处理器                               │   │    │
│  │   │   · ReliProcessor      (登录响应)                           │   │    │
│  │   │   · TtsdProcessor     (任务下发)                           │   │    │
│  │   │   · TrptProcessor     (节点报)                             │   │    │
│  │   │   · GwsmProcessor     (工作短语)                            │   │    │
│  │   │   · BillProcessor     (油单处理)                            │   │    │
│  │   └─────────────────────────────────────────────────────────────┘   │    │
│  │                                                                      │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                    │                                        │
│                                    ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                       数据持久层                                       │    │
│  │                                                                      │    │
│  │   ┌─────────────────────────────────────────────────────────────┐   │    │
│  │   │                    MyBatis Mapper                            │   │    │
│  │   │   · MessageMapper       (消息记录)                          │   │    │
│  │   │   · TerConfigMapper     (终端配置)                          │   │    │
│  │   │   · InfoConfirmMapper   (信息确认)                          │   │    │
│  │   │   · PressureDiffMapper  (压力差数据)                         │   │    │
│  │   └─────────────────────────────────────────────────────────────┘   │    │
│  │                                                                      │    │
│  │   ┌─────────────────────────────────────────────────────────────┐   │    │
│  │   │                    数据模型 (BSOD)                            │   │    │
│  │   │   · MessageEntity      (消息实体)                           │   │    │
│  │   │   · Registeredresource (注册资源)                           │   │    │
│  │   │   · Dispatchlogin     (调度登录)                            │   │    │
│  │   │   · LoginDetail      (登录详情)                             │   │    │
│  │   └─────────────────────────────────────────────────────────────┘   │    │
│  │                                                                      │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                    │                                        │
│                                    ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                       基础设施层                                       │    │
│  │                                                                      │    │
│  │   ┌───────────────┐ ┌───────────────┐ ┌───────────────┐              │    │
│  │   │    MySQL     │ │    Redis     │ │   FastDFS    │              │    │
│  │   │  (持久化)    │ │   (缓存)     │ │  (文件存储)  │              │    │
│  │   └───────────────┘ └───────────────┘ └───────────────┘              │    │
│  │                                                                      │    │
│  │   ┌───────────────┐ ┌───────────────┐                               │    │
│  │   │  ActiveMQ    │ │   Zookeeper  │                               │    │
│  │   │  (消息队列)  │ │  (服务注册)  │                               │    │
│  │   └───────────────┘ └───────────────┘                               │    │
│  │                                                                      │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.3 模块依赖关系

模块间的依赖关系遵循从底层到上层的合理设计。父模块terminalmanager-oil定义了公共依赖和构建配置。bsod模块作为数据模型层，被其他所有模块所依赖。messagedispatch模块提供了消息分发的基础设施。terminal模块是核心业务模块，依赖messagedispatch和bsod模块，同时集成了丰富的外部服务，包括flight-oil-service（油品业务服务）、bill-rpc（油单RPC服务）、TerminalModelService（航班模型服务）、MIS服务（用户/车辆/部门信息）等。

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         模块依赖关系图                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                          terminalmanager-oil (父模块)                          │
│                                    │                                         │
│                                    ▼                                         │
│   ┌───────────────────────────────────────────────────────────────────┐    │
│   │                        bsod (数据模型层)                            │    │
│   │   · MessageEntity        (消息实体)                               │    │
│   │   · Registeredresource   (注册资源)                               │    │
│   │   · Dispatchlogin       (调度登录)                               │    │
│   │   · TerConfigMapper     (终端配置)                               │    │
│   └───────────────────────────────────────────────────────────────────┘    │
│                                    │                                         │
│                                    ▼                                         │
│   ┌───────────────────────────────────────────────────────────────────┐    │
│   │                      messagedispatch (消息分发)                      │    │
│   │   · Camel Routes       (消息路由)                                │    │
│   │   · JMS Integration   (JMS集成)                                 │    │
│   │   · FreeMarker Templates(消息模板)                                 │    │
│   └───────────────────────────────────────────────────────────────────┘    │
│                                    │                                         │
│                                    ▼                                         │
│   ┌───────────────────────────────────────────────────────────────────┐    │
│   │                       terminal (核心业务层)                           │    │
│   │                                                                      │    │
│   │   内部依赖:                                                          │    │
│   │   · terminal-mode         (终端模式)                              │    │
│   │   · messagedispatch       (消息分发)                              │    │
│   │   · bsod                  (数据模型)                             │    │
│   │   · terminal-core         (核心逻辑)                              │    │
│   │   · terminal-rest         (REST API)                             │    │
│   │                                                                      │    │
│   │   外部依赖:                                                          │    │
│   │   · flight-oil-service    (油品业务服务)                            │    │
│   │   · bill-rpc            (油单RPC)                                │    │
│   │   · TerminalModelService (航班模型)                                │    │
│   │   · MIS服务            (用户/车辆/部门)                           │    │
│   │   · VDAPI               (车辆信息)                                │    │
│   │   · GlobalSequence      (全局序列)                                │    │
│   │   · Security-API        (安全认证)                                │    │
│   │                                                                      │    │
│   └───────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 三、核心业务功能详解

### 3.1 任务与油单关联管理

任务与油单的关联管理是本模块的核心功能之一，由TaskServiceImpl类实现。该服务负责根据任务号查询完整的任务信息，并将任务与油单进行关联绑定。当终端请求任务信息时，系统首先通过ServiceTerminalBz获取完整的任务记录，然后调用TerminalModelService获取航班信息，最后通过OilBillNumberService管理油单号与任务的绑定关系。这种设计确保了每个任务都有唯一的油单号对应，实现了油品供应业务的精确追踪。

```java
// TaskServiceImpl.java - 任务油单关联管理
@Service("taskService01")
public class TaskServiceImpl implements TaskService {
    
    @Resource
    private OilBillNumberService oilBillNumberService;
    
    @Resource
    private OilBillNumService oilBillNumService;
    
    @Override
    public GetOilBillTaskByTnbResponseVO gettaskbytnb(GetOilBilltaskbytnbRequest request) {
        LinkedHashMap<String, Task> hashMap = new LinkedHashMap<>();
        
        for (String key : requestmap.keySet()) {
            // 1. 查询完整任务信息
            AgdisTaskVo taskRecord = serviceClient.getFullTaskByTnb(tenant, key);
            
            if (taskRecord != null && !"END".equalsIgnoreCase(taskRecord.getStat())) {
                // 2. 查询航班信息
                FlightStroeData flightinfo = terminalModelService
                    .getFlightStroeDataByTenantAndFlseq(tenant, taskRecord.getFkey());
                
                // 3. 构建任务对象
                Task task = dataManager.buildTask(taskRecord);
                
                // 4. 获取或创建油单号绑定
                TaskOilBillNumberEntity taskOilBillNumberEntity = 
                    oilBillNumberService.getTaskOilBillNumberEntity(task.getTNB(), tenant);
                
                if (taskOilBillNumberEntity == null) {
                    // 如果没有绑定油单号，创建新的绑定
                    String elebillnumber = oilBillNumService.getEleBillNumber(
                        taskRecord.getVnb(), taskRecord.getFlop());
                    task.setEleBillNumber(elebillnumber);
                    
                    TaskOilBillNumberEntity newEntity = new TaskOilBillNumberEntity();
                    newEntity.setOilBillNumber(elebillnumber);
                    newEntity.setTaskId(task.getTNB());
                    newEntity.setUserId(taskRecord.getTenb());
                    newEntity.setTenant(taskRecord.getTenantId());
                    
                    // 任务号与油单号绑定
                    oilBillNumberService.insertTaskOilBillNumberService(newEntity);
                } else {
                    task.setEleBillNumber(taskOilBillNumberEntity.getOilBillNumber());
                }
                
                hashMap.put(key, task);
            }
        }
        
        return result;
    }
}
```

### 3.2 签约方式处理

签约方式管理是油品供应业务的重要功能，系统支持根据客户的签约类型进行不同的业务处理。TaskServiceImpl类中实现了签约方式的查询和设置逻辑，通过BaseCustomerRecordService查询客户的签约信息，获取签约方式类型（signOrderType）。这个签约方式会随任务信息下发到终端，指导现场作业人员进行相应的操作处理。

```java
// 签约方式处理逻辑
// 消息推送终端时增加签约方式
String al2c = task.getTaskInfo().getAL2();
if (StringUtils.isNotEmpty(al2c)) {
    BaseCustomerRecord searchrecord = new BaseCustomerRecord();
    searchrecord.setAl2c(al2c);
    
    List<BaseCustomerRecord> dbrecords = baseCustomerRecordService
        .getCustomerRecordByAl2cAlcname(searchrecord);
    
    if (!dbrecords.isEmpty() && dbrecords.size() > 0) {
        // 设置签约方式类型
        task.setSignOrderType(dbrecords.get(0).getSignOrderType());
    }
}
```

### 3.3 一机一位功能

"一机一位"是机场油品供应的特色功能，用于管理加油车辆与停机位之间的匹配关系。系统通过OneMachineOneSeatService查询一机一位的配置信息，并在任务下发时将这些信息推送到终端设备，帮助司机快速定位正确的作业位置。一机一位的查询条件包括机场三字码（ap3c）、停机位代码（placecode）、车辆类型（vtype）和机型（acname）。

```java
// 一机一位功能实现
private void addOneMachineOneSeat(Task task, FlightStroeData flightinfo) {
    String ap3c = flightinfo.getAp3c();
    String placecode = flightinfo.getPlacecode();
    String acname = flightinfo.getAcname();
    String vnb = task.getTaskInfo().getVNB();
    String vtype = getVtype(vnb, flightinfo.getTenant());
    
    if (StringUtils.isNotEmpty(ap3c) && StringUtils.isNotEmpty(placecode) 
            && StringUtils.isNotEmpty(acname)) {
        task.setOneMachineOneSeat(getOneMachineOneSeat(ap3c, placecode, vtype, acname));
    }
}

private List<OneMachineOneSeat> getOneMachineOneSeat(String ap3c, String placeCode, 
        String vtype, String acname) {
    OneMachineOneSeat oneSeat = new OneMachineOneSeat();
    oneSeat.setAp3c(ap3c);
    oneSeat.setPlaceCode(placeCode);
    oneSeat.setVtype(vtype);
    oneSeat.setAcname(acname);
    
    // 调用MIS服务查询一机一位配置
    B2BResponse oneMachineOneSeat = oneMachineOneSeatService
        .findOneMachineOneSeat(getB2BRequest(), oneSeat);
    
    return (List<OneMachineOneSeat>) oneMachineOneSeat.getPayload();
}
```

### 3.4 终端消息初始化

TerminalInitMessage类定义了终端初始化消息的数据结构，包含了发送工作短语所需的完整信息。这些信息涵盖了航班基础数据、任务信息、发送者信息、消息类型等。终端设备通过解析这些初始化消息，获取当前需要执行的工作任务和操作指引。

```java
// TerminalInitMessage - 终端初始化消息
public class TerminalInitMessage implements java.io.Serializable {
    
    // 短语标识
    private String messageID;        // 短语ID
    private String orderID;          // 短语发送记录唯一号
    
    // 航班信息
    private String flightDate;       // 航班日
    private String flightKey;        // 航班唯一号
    private String flightNumber;     // 航班号
    private String aodFLAG;          // 进离港标志
    private String regNumber;        // 机号
    private String actName;          // 机型
    
    // 任务信息
    private String taskNumber;       // 任务号
    private String resourceType;     // 资源类型（服务类型）
    private String resourceName;     // 资源名称（服务名称）
    
    // 消息内容
    private String msgTypeCode;      // 短语类型代码
    private String msgTypeName;      // 短语类型名称
    private String msgCode;         // 短语代码
    private String msgTitle;        // 短语标题
    private String msgTxt;          // 短语内容
    
    // 发送者信息
    private String senderResourceNo; // 发送者资源号
    private String senderNo;         // 发送者员工号
    private String senderName;       // 发送者姓名
    private String positionId;      // 发送者岗位ID
    private String positionName;    // 发送者岗位名称
    private String senderRole;      // 发送者角色
    private String departmentID;     // 发送者部门ID
    private String departmentName;   // 发送者部门名称
    
    // 处理标志
    private String replyType;       // 应答类型：N-自动应答，Y-手动确认
    private String isReply;        // 是否已回执
    private String msgTxl;         // 短语长度
    private String msgResource;    // 信息来源
}
```

---

## 四、配置文件详解

### 4.1 父POM配置

父POM文件定义了模块的基本信息和子模块声明。版本号5.0.0.4表明这是针对油品业务的定制版本。模块声明包含terminal（主模块）、messagedispatch（消息分发）和bsod（数据模型）三个子模块。这种简洁的父POM设计将所有构建细节下放到子模块中，保持了配置的清晰性。

```xml
<!-- terminalmanager-oil父POM配置 -->
<project>
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.ias.agdis</groupId>
    <artifactId>terminalmanager</artifactId>
    <packaging>pom</packaging>
    <version>5.0.0.4</version>
    
    <modules>
        <module>terminal</module>
        <module>messagedispatch</module>
        <module>bsod</module>
    </modules>
</project>
```

### 4.2 主模块POM配置

terminal模块的POM配置详细定义了项目构建参数和依赖关系。Spring版本升级到4.2.2.RELEASE，提供了更新的框架特性。依赖管理方面，模块引入了完整的Spring技术栈，包括spring-core、spring-context、spring-jdbc、spring-tx、spring-webmvc等核心模块。RESTEasy框架支持RESTful服务开发。Dubbo和Zookeeper支持分布式服务调用。MyBatis支持数据持久化。Redis和FastDFS支持缓存和文件存储。日志系统采用Logback配合Logstash编码器。

```xml
<!-- terminal模块核心依赖配置 -->
<properties>
    <org.springframework.version>4.2.2.RELEASE</org.springframework.version>
    <camel-version>2.16.1</camel-version>
    <activemq-version>5.12.0</activemq-version>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <maven.compiler.source>1.7</maven.compiler.source>
    <maven.compiler.target>1.7</maven.compiler.target>
</properties>

<dependencies>
    <!-- 内部模块依赖 -->
    <dependency>
        <groupId>com.ias.agdis</groupId>
        <artifactId>terminal-mode</artifactId>
        <version>5.0.2.1</version>
    </dependency>
    <dependency>
        <groupId>com.ias.agdis</groupId>
        <artifactId>bsod</artifactId>
        <version>5.0.2.2</version>
    </dependency>
    <dependency>
        <groupId>com.ias.agdis</groupId>
        <artifactId>messagedispatch</artifactId>
        <version>5.0.0.3</version>
    </dependency>
    
    <!-- Spring框架 -->
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-core</artifactId>
        <version>${org.springframework.version}</version>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-context</artifactId>
        <version>${org.springframework.version}</version>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-jdbc</artifactId>
        <version>${org.springframework.version}</version>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-tx</artifactId>
        <version>${org.springframework.version}</version>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-webmvc</artifactId>
        <version>${org.springframework.version}</version>
    </dependency>
    
    <!-- RESTEasy框架 -->
    <dependency>
        <groupId>org.jboss.resteasy</groupId>
        <artifactId>resteasy-jaxrs</artifactId>
        <version>3.0.7.Final</version>
    </dependency>
    
    <!-- Apache Camel -->
    <dependency>
        <groupId>org.apache.camel</groupId>
        <artifactId>camel-core</artifactId>
        <version>${camel-version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.camel</groupId>
        <artifactId>camel-spring</artifactId>
        <version>${camel-version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.camel</groupId>
        <artifactId>camel-jms</artifactId>
        <version>${camel-version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.camel</groupId>
        <artifactId>camel-freemarker</artifactId>
        <version>${camel-version}</version>
    </dependency>
    
    <!-- ActiveMQ -->
    <dependency>
        <groupId>org.apache.activemq</groupId>
        <artifactId>activemq-client</artifactId>
        <version>${activemq-version}</version>
    </dependency>
    <dependency>
        <groupId>org.apache.activemq</groupId>
        <artifactId>activemq-pool</artifactId>
        <version>${activemq-version}</version>
    </dependency>
    
    <!-- MyBatis -->
    <dependency>
        <groupId>org.mybatis</groupId>
        <artifactId>mybatis</artifactId>
        <version>3.2.3</version>
    </dependency>
    <dependency>
        <groupId>org.mybatis</groupId>
        <artifactId>mybatis-spring</artifactId>
        <version>1.2.1</version>
    </dependency>
    
    <!-- Redis -->
    <dependency>
        <groupId>redis.clients</groupId>
        <artifactId>jedis</artifactId>
        <version>2.4.1</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.data</groupId>
        <artifactId>spring-data-redis</artifactId>
        <version>1.4.2.RELEASE</version>
    </dependency>
    
    <!-- FastDFS -->
    <dependency>
        <groupId>net.oschina.zcx7878</groupId>
        <artifactId>fastdfs-client-java</artifactId>
        <version>1.27.0.0</version>
    </dependency>
    
    <!-- Dubbo RPC -->
    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>dubbo</artifactId>
        <version>2.5.4-IAS</version>
    </dependency>
    <dependency>
        <groupId>org.apache.zookeeper</groupId>
        <artifactId>zookeeper</artifactId>
        <version>3.4.6</version>
    </dependency>
    
    <!-- 日志框架 -->
    <dependency>
        <groupId>ch.qos.logback</groupId>
        <artifactId>logback-classic</artifactId>
        <version>1.2.3</version>
    </dependency>
    <dependency>
        <groupId>net.logstash.logback</groupId>
        <artifactId>logstash-logback-encoder</artifactId>
        <version>4.11</version>
    </dependency>
    
    <!-- JSON处理 -->
    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>fastjson</artifactId>
        <version>1.2.83</version>
    </dependency>
</dependencies>
```

### 4.3 构建配置

构建配置定义了Maven打包和Docker镜像构建的参数。finalName设置为terminalmanager-oil，将作为Docker镜像名称的基础。maven-compiler-plugin指定Java源码和目标版本为1.7。maven-assembly-plugin将项目打包为包含所有依赖的分发包。Docker构建配置被注释但保留，便于后续启用容器化部署。配置文件排除规则确保logback.xml、config.properties等敏感配置不会被打包进JAR文件。

```xml
<build>
    <finalName>terminalmanager-oil</finalName>
    
    <plugins>
        <!-- 编译器配置 -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <configuration>
                <source>1.7</source>
                <target>1.7</target>
                <encoding>UTF-8</encoding>
            </configuration>
        </plugin>
        
        <!-- 打包配置 -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-assembly-plugin</artifactId>
            <configuration>
                <descriptor>src/main/assembly/assembly.xml</descriptor>
            </configuration>
        </plugin>
        
        <!-- JAR排除配置 -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-jar-plugin</artifactId>
            <configuration>
                <excludes>
                    <exclude>logback.xml</exclude>
                    <exclude>config.properties</exclude>
                    <exclude>jdbc_mysql.properties</exclude>
                </excludes>
            </configuration>
        </plugin>
    </plugins>
</build>
```

---

## 五、REST API接口设计

### 5.1 核心服务接口

模块通过REST API对外提供服务，采用JAX-RS注解定义接口规范。核心服务接口包括TaskService（任务服务）、SecurityService（安全认证）、FuelPipeOilService（油品管输）、FTPServiceManager（文件服务）等。这些服务实现了油品供应业务的完整功能覆盖，包括用户登录登出、任务查询、任务操作、油单管理、化验数据上传、压力差记录、航班登记、车辆绑定等。

| 服务类 | 功能描述 | 主要方法 |
|--------|----------|----------|
| **TaskService** | 任务管理服务 | gettaskbytnb（按任务号查询） |
| **SecurityService** | 安全认证服务 | login（登录）、logout（登出） |
| **FuelPipeOilService** | 油品管输服务 | 油品管输相关操作 |
| **FTPServiceManager** | 文件上传服务 | upload（上传）、download（下载） |
| **ConfigManager** | 配置管理服务 | getConfig（获取配置） |

### 5.2 请求/响应模型

模块定义了丰富的请求和响应VO类，用于API接口的数据交换。这些VO类封装了业务数据，并提供了与内部数据模型之间的转换方法。请求VO类继承自基础请求结构，包含请求头和请求体。响应VO类定义了统一的响应格式，支持成功/失败状态、错误码和错误信息等。

```java
// 任务查询请求模型
public class GetOilBilltaskbytnbRequest {
    private RequestHeader header;
    private GetOilBillTaskByTnbRequestVO getOilBillTaskByTnbRequestVO;
}

public class GetOilBillTaskByTnbRequestVO {
    private LinkedHashMap<String, String> hashmap;  // 任务号映射
}

// 任务查询响应模型
public class GetOilBillTaskByTnbResponseVO {
    private LinkedHashMap<String, Task> hashmap;   // 任务结果映射
}

// 登录请求模型
public class LoginRequestVO {
    private String userId;       // 用户ID
    private String password;     // 密码
    private String deviceId;    // 设备ID
    private String forceLogin;   // 是否强制登录
}

// 油单响应模型
public class OilBillResponseVO {
    private String oilBillNumber;     // 油单号
    private String status;            // 状态
    private String createTime;        // 创建时间
    private String updateTime;        // 更新时间
}
```

---

## 六、消息模板系统

### 6.1 模板目录结构

模块采用FreeMarker模板引擎生成各类业务消息。templates目录下包含了针对不同消息类型的模板文件，按消息类型分组存储。这种设计使得消息格式的配置与代码分离，便于业务人员调整消息内容而无需修改代码。模板文件支持变量替换、条件判断、循环等FreeMarker语法，能够灵活生成各种格式的消息内容。

```
templates/
├── bill/                           # 油单相关模板
│   └── ...
├── gwsm/                           # 工作短语模板
│   ├── gwsm.xml
│   └── gwsm.ftl
├── ttud/                          # 任务更新模板
│   ├── ttud.xml
│   └── ttud.ftl
├── ttrq/                          # 任务请求模板
│   ├── ttrq.xml
│   └── ttrq.ftl
├── ttas/                          # 任务分配模板
│   ├── ttas.xml
│   └── ttas.ftl
├── reli/                          # 登录响应模板
│   ├── reli.xml
│   └── reli.ftl
├── sgrp/                          # 通用应答模板
│   ├── sgrp.xml
│   └── sgrp.ftl
├── inii/                          # 初始化模板
│   └── ...
├── trpt/                          # 节点报模板
│   ├── trpt.xml
│   └── trpt.ftl
└── ...                            # 其他业务模板
```

### 6.2 模板处理流程

消息模板的处理通过Apache Camel的FreeMarker组件实现。当需要生成业务消息时，Camel路由从模板文件读取模板定义，填充相应的数据变量，生成最终的消息内容。模板处理支持两种模式：XML格式（.xml文件）和FreeMarker模板格式（.ftl文件）。XML文件定义模板结构和变量占位，FreeMarker文件使用FreeMarker语法定义输出格式。

```java
// Camel路由中的模板处理配置
// 消息模板生成示例
from("jms:queue:Q_TASK")
    .to("freemarker:templates/task/task.ftl")
    .to("jms:queue:Q_TASK_OUTPUT");
```

---

## 七、问题与优化建议

### 7.1 技术债务分析

本模块继承了通用terminalmanager模块的技术债务，同时存在一些特有的问题。Spring Framework 4.2.2虽然比4.1.8版本新，但仍不是LTS版本。Java 1.7早已停止支持，在JDK 8已成为主流的当下，继续使用会增加维护成本和安全风险。FastDFS客户端版本1.27.0.0存在已知的安全漏洞，建议升级到官方推荐版本。代码中硬编码的B2BRequest参数（如用户ID和密码）存在安全隐患，应改为配置管理。

| 问题类型 | 问题描述 | 严重程度 | 优化建议 |
|----------|----------|----------|----------|
| **框架版本** | Spring 4.2.2非LTS版本 | 中 | 升级至5.3.x LTS |
| **Java版本** | Java 1.7已停止支持 | 高 | 升级至Java 11 |
| **安全问题** | B2BRequest硬编码密码 | 高 | 改为配置管理 |
| **文件存储** | FastDFS客户端版本旧 | 中 | 升级到官方稳定版 |
| **序列化** | Kryo/Kryo-Serializers版本低 | 低 | 升级到2.24.0+ |

### 7.2 代码质量问题

代码中存在一些需要改进的地方。B2BRequest的构建采用硬编码的用户名和密码，不仅存在安全风险，也不便于多环境配置。异常处理采用e.printStackTrace()，不符合生产环境的日志规范。大量使用LinkedHashMap而非泛型集合，增加了代码的不安全性。HTTP和FTP客户端的创建未使用连接池，可能导致连接资源耗尽。

```java
// 问题：硬编码的B2BRequest
private B2BRequest getB2BRequest() {
    B2BRequest request = new B2BRequest();
    request.setUserId("mis_user");  // 硬编码
    request.setPassword("mis_user");  // 硬编码
    request.setSystemId("MIS");
    return request;
}

// 优化建议：使用配置管理
@Value("${b2b.username}")
private String b2bUsername;

@Value("${b2b.password}")
private String b2bPassword;

private B2BRequest getB2BRequest() {
    B2BRequest request = new B2BRequest();
    request.setUserId(b2bUsername);
    request.setPassword(b2bPassword);
    request.setSystemId("MIS");
    return request;
}

// 问题：异常处理不规范
} catch (Exception e) {
    e.printStackTrace();  // 不规范的异常处理
}

// 优化建议：使用日志框架
} catch (Exception e) {
    logger.error("查询一机一位配置失败", e);
}
```

### 7.3 性能优化建议

性能优化方面，建议从以下几个维度入手。数据库查询应增加索引，特别是任务号、航班号、油单号等常用查询字段的索引。缓存使用可以进一步优化，对于不经常变化的基础数据（如一机一位配置）应设置较长的缓存时间。HTTP和FTP客户端应配置连接池，避免频繁创建连接带来的开销。JSON序列化可以升级到更高版本的Jackson或FastJSON，利用其性能优化特性。

```java
// 优化建议：数据库索引优化
// ALTER TABLE task_oil_bill_number ADD INDEX idx_tnb (task_id);
// ALTER TABLE base_customer_record ADD INDEX idx_al2c (al2c);

// 优化建议：连接池配置
// FTPClient配置连接池
// HttpClient配置连接池
```

---

## 八、总结

### 8.1 模块特点总结

terminalmanager-oil模块是针对浦东机场油品供应业务定制的终端管理模块，在通用terminalmanager架构的基础上扩展了丰富的油品业务功能。模块专注于油单全生命周期管理，实现了任务与油单的精确绑定、签约方式处理、一机一位管理、化验数据采集等特色功能。与通用模块相比，terminalmanager-oil更贴近油品供应的实际业务需求，提供了更加专业和深入的解决方案。85个消息模板文件覆盖了油品业务的各类消息场景，214个Java源文件构成了完整的业务功能实现。

### 8.2 核心能力

本模块具备以下核心能力：任务与油单关联管理能力，支持任务号的查询、油单的自动绑定和电子油单号的生成；签约方式处理能力，根据客户签约类型提供不同的业务处理逻辑；一机一位管理能力，帮助现场作业人员快速定位正确的作业位置；油品化验数据采集能力，支持化验结果的电子化录入和上传；压力差监控能力，采集和记录加油过程中的压力数据；航班信息集成能力，整合航班动态和任务信息；文件上传能力，支持签名和图片的FastDFS存储；安全认证能力，提供用户登录和权限验证。

### 8.3 与通用模块的对比

与通用的terminalmanager模块相比，terminalmanager-oil模块具有以下特点：版本较低（5.0.x vs 7.0.x），表明是两个独立演进的产品线；专注于油品业务，扩展了flight-oil-service、bill-rpc等油品相关依赖；保留了核心的终端管理功能，包括消息处理、任务管理、用户认证等；模板系统针对油品业务进行了定制；未集成Redis缓存层，简化了部署复杂度。

---

*报告生成时间：2026-02-04*
*分析工具：Cursor AI Code Analysis*
*代码分析范围：terminalmanager-oil模块全部源代码及配置文件*
