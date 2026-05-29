# servicemanager 服务管理模块代码分析报告

> 分析日期：2026-02-04
> 项目路径：e:\work\code\oil\pudong\servicemanager\
> 文档版本：1.0

---

## 1. 项目概述

### 1.1 基本信息

| 属性 | 值 |
|------|-----|
| 项目名称 | servicemanager (服务管理引擎) |
| GroupId | com.ias.lib |
| ArtifactId | servicemanager |
| 版本 | 5.3.2.5 (父模块) |
| 打包方式 | pom (父模块) |
| Java版本 | 1.7 |

### 1.2 模块结构

```
servicemanager/                              # 父模块 (v5.3.2.5)
├── pom.xml                                # 父POM配置
│
├── service-engine/                         # 服务引擎核心 (v5.0.1.1)
│   ├── src/main/java/com/ias/lib/service/engine/
│   │   ├── ServiceEngine.java             # 服务引擎接口
│   │   ├── ServiceEngineImpl.java        # 服务引擎实现
│   │   ├── InstanceService.java          # 实例服务接口
│   │   ├── InstanceServiceImpl.java      # 实例服务实现 (1973行)
│   │   ├── RepositoryService.java       # 仓库服务接口
│   │   ├── ManagerService.java          # 管理服务接口
│   │   ├── EngineServices.java          # 引擎服务基接口
│   │   │
│   │   ├── instance/                   # 业务实体
│   │   │   ├── Service.java            # 服务接口 (174方法)
│   │   │   ├── Task.java               # 任务接口 (212方法)
│   │   │   ├── TaskProcess.java        # 任务进程
│   │   │   ├── ServiceProcess.java     # 服务进程
│   │   │   ├── ServiceMark.java        # 服务标注
│   │   │   ├── TaskMark.java          # 任务标注
│   │   │   ├── TaskForward.java       # 任务转发
│   │   │   ├── TaskTransfer.java       # 任务移交
│   │   │   ├── ApplyTask.java         # 申请任务
│   │   │   ├── ServiceStandard.java    # 服务标准
│   │   │   ├── PreNotice.java         # 预通知
│   │   │   ├── FlightLock.java        # 航班锁定
│   │   │   └── Alarm.java             # 报警
│   │   │
│   │   ├── impl/                        # 服务实现
│   │   │   ├── cmd/                    # 命令模式实现
│   │   │   │   ├── GenarateTaskCmd.java    # 生成任务命令
│   │   │   │   ├── GenarateServiceCmd.java # 生成服务命令
│   │   │   │   ├── SubmitTaskProcessCmd.java # 提交任务进程
│   │   │   │   ├── SubmitServiceProcessCmd.java # 提交服务进程
│   │   │   │   ├── SubmitTaskMarkCmd.java  # 提交任务标注
│   │   │   │   ├── ForwardTaskCmd.java    # 转发任务
│   │   │   │   ├── TransferTaskCmd.java   # 移交任务
│   │   │   │   └── ... (40+ 命令类)
│   │   │   │
│   │   │   ├── persistence/            # 持久化层
│   │   │   │   ├── deploy/             # 部署管理
│   │   │   │   │   ├── DeploymentManager.java    # 部署管理器
│   │   │   │   │   ├── ServiceStandardDeployer.java
│   │   │   │   │   └── ServiceRuleDeployer.java
│   │   │   │   └── entity/            # 实体管理
│   │   │   │       ├── ServiceEntityManager.java
│   │   │   │       ├── TaskEntityManager.java
│   │   │   │       └── ...
│   │   │   │
│   │   │   ├── query/                  # 查询构建器
│   │   │   │   ├── Query.java          # 查询接口
│   │   │   │   ├── TaskQuery.java     # 任务查询
│   │   │   │   ├── ServiceQuery.java  # 服务查询
│   │   │   │   └── AlarmQuery.java    # 报警查询
│   │   │   │
│   │   │   ├── event/                  # 事件机制
│   │   │   │   ├── ServiceEvent.java      # 服务事件
│   │   │   │   ├── ServiceEventType.java  # 事件类型
│   │   │   │   ├── ServiceEventDispatcher.java
│   │   │   │   └── ServiceEventListener.java
│   │   │   │
│   │   │   ├── push/                   # 消息推送
│   │   │   │   ├── PushMessage.java    # 推送消息接口
│   │   │   │   ├── PushMessageService.java
│   │   │   │   ├── PushMessageUtil.java
│   │   │   │   └── PushMessageSubscriber.java
│   │   │   │
│   │   │   ├── lock/                    # 分布式锁
│   │   │   │   └── DistributLock.java
│   │   │   │
│   │   │   ├── memory/                  # 内存管理
│   │   │   │   ├── GlobalDataMemory.java
│   │   │   │   └── ConstantDefined.java
│   │   │   │
│   │   │   └── interceptor/             # 拦截器
│   │   │       ├── Command.java         # 命令接口
│   │   │       ├── CommandContext.java  # 命令上下文
│   │   │       └── CommandExecutor.java # 命令执行器
│   │   │
│   │   └── impl/
│   │       ├── cfg/                     # 配置类
│   │       │   ├── ServiceEngineConfigurationImpl.java
│   │       │   ├── ServiceEngineFactoryBean.java
│   │       │   └── multitenant/        # 多租户支持
│   │       │
│   │       └── ServiceImpl.java
│   │
│   └── src/main/resources/
│       └── mapping/mappings.xml        # MyBatis映射
│
├── service-converter/                     # 服务转换器 (v5.0.0.1)
│   └── (配置模型转换)
│
├── service-dubbo-rcp/                     # Dubbo RPC模块 (v5.0.2.5)
│   └── (RPC接口定义)
│
├── service-dubbo-impl/                   # Dubbo服务实现 (vPVG.5.3.2.7)
│   ├── pom.xml
│   └── src/main/resources/
│       ├── standard/                     # 服务标准配置
│       │   ├── OIL_PVG/
│       │   │   ├── standard.xml        # 浦东标准配置
│       │   │   └── JY.rule.xml        # 加油服务规则
│       │   ├── WA_XIY/
│       │   ├── CA_PEK/
│       │   └── ... (多机场配置)
│       │
│       └── applicationContext*.xml      # Spring配置
│
├── service-config-validation/             # 配置验证 (v5.0.0.1)
│   └── (配置校验规则)
│
└── service-fw/                            # 服务框架 (v5.3.2.5)
    ├── minilang/                         # 迷你语言解析
    │   ├── MiniLangUtil.java
    │   ├── MiniLangParse.java
    │   └── MethodDeal.java
    │
    └── logging/                          # 日志框架
        ├── Logger.java
        ├── ExPatternLayout.java
        └── logback/                     # Logback扩展
```

### 1.3 业务定位

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         servicemanager 业务定位                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                         航班服务调度核心引擎                           │   │
│   │                                                                      │   │
│   │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────────┐  │   │
│   │  │  服务标准   │  │  任务生成   │  │  进程管理                 │  │   │
│   │  │Standard    │  │  Task      │  │  Process                  │  │   │
│   │  └─────────────┘  └─────────────┘  └─────────────────────────────┘  │   │
│   │                                                                      │   │
│   │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────────┐  │   │
│   │  │  报警监控   │  │  消息推送   │  │  资源锁定                 │  │   │
│   │  │  Alarm     │  │  Push      │  │  Lock                     │  │   │
│   │  └─────────────┘  └─────────────┘  └─────────────────────────────┘  │   │
│   │                                                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                              │                                              │
│                              ▼                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                         调用方                                         │   │
│   │                                                                      │   │
│   │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────────┐  │   │
│   │  │  flight-    │  │  mis/      │  │  flightdealui            │  │   │
│   │  │  dispatch   │  │  security  │  │  调度界面                │  │   │
│   │  └─────────────┘  └─────────────┘  └─────────────────────────────┘  │   │
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
| **Spring** | 4.0.8.RELEASE | 应用框架 |
| **MyBatis** | 3.4.5 | ORM框架 |
| **Dubbo** | 2.5.3 | RPC框架 |
| **Zookeeper** | 3.4.6 | 服务注册 |
| **ActiveMQ** | 5.12.0 | 消息队列 |
| **Apache Camel** | 2.16.1 | 路由/集成 |
| **Redis (Redisson)** | 2.7.2 | 分布式锁/缓存 |
| **Freemarker** | 2.3.20 | 模板引擎 |
| **Dozer** | 5.4.0 | 对象映射 |
| **Guava** | 20.0 | 工具库 |
| **LambdaJ** | 2.3.3 | 函数式编程 |
| **Flyway** | 3.0 | 数据库迁移 |

### 2.2 外部服务依赖

| 服务 | GroupId:ArtifactId | 用途 |
|------|-------------------|------|
| **flight-services-facade** | com.ias.server.flight:flight-services-facade | 航班服务 |
| **flight-services-common** | com.ias.server.flight:flight-services-common | 航班公共 |
| **common** | com.ias.server.flight:common | 公共模块 |
| **model** | com.ias.common:model | 公共模型 |
| **resourceapi** | com.ias.lib.resource:resourceapi | 资源API |
| **MISGWSAPI** | com.ias.lib.mis:MISGWSAPI | MIS用户服务 |
| **security-api** | com.ias.lib:security-api | 安全认证 |

### 2.3 架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           技术架构图                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                        调用方 (Dubbo Consumer)                       │    │
│  │                                                                      │    │
│  │   ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐  │    │
│  │   │ flight-     │  │ flightdealui │  │  terminalmanager-oil     │  │    │
│  │   │ dispatch    │  │  调度界面    │  │  终端管理              │  │    │
│  │   └──────┬───────┘  └──────┬───────┘  └───────────┬──────────────┘  │    │
│  └──────────┼──────────────────┼──────────────────────┼──────────────────┘   │
│             │                  │                      │                        │
│             └──────────────────┼──────────────────────┘                        │
│                                │                                               │
│                                ▼                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                     Dubbo RPC 层                                     │    │
│  │                                                                      │    │
│  │   ┌─────────────────────────────────────────────────────────────┐   │    │
│  │   │                  InstanceService                          │   │    │
│  │   │       (1973行核心实现 - 任务/服务全生命周期管理)            │   │    │
│  │   └─────────────────────────────────────────────────────────────┘   │    │
│  │                              │                                      │    │
│  │                              ▼                                      │    │
│  │   ┌─────────────────────────────────────────────────────────────┐   │    │
│  │   │                 Command模式实现                              │   │    │
│  │   │                                                              │   │    │
│  │   │   ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │   │    │
│  │   │   │GenarateTaskCmd│  │SubmitTaskCmd│  │ForwardTaskCmd  │  │   │    │
│  │   │   └──────────────┘  └──────────────┘  └──────────────────┘  │   │    │
│  │   └─────────────────────────────────────────────────────────────┘   │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                      │                                      │
│                                      ▼                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                     DeploymentManager (部署管理器)                    │    │
│  │                                                                      │    │
│  │   ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐  │    │
│  │   │ServiceStandard│  │ ServiceRule  │  │ TaskRule               │  │    │
│  │   │Model         │  │ Model        │  │                        │  │    │
│  │   │(服务标准)    │  │(服务规则)    │  │                        │  │    │
│  │   └──────────────┘  └──────────────┘  └──────────────────────────┘  │    │
│  │                                                                      │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                      │                                      │
│                                      ▼                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                         消息推送层                                    │    │
│  │                                                                      │    │
│  │   ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐  │    │
│  │   │   ActiveMQ   │  │  Apache     │  │  Redis/Redisson        │  │    │
│  │   │   (JMS)      │  │  Camel      │  │  (Pub/Sub)            │  │    │
│  │   └──────────────┘  └──────────────┘  └──────────────────────────┘  │    │
│  │                                                                      │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                      │                                      │
│                                      ▼                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                        数据访问层 (MyBatis)                           │    │
│  │                                                                      │    │
│  │   ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐  │    │
│  │   │EntityManager │  │  Mapper     │  │  Flyway                 │  │    │
│  │   │(实体管理)    │  │  (SQL映射)  │  │  (数据库迁移)          │  │    │
│  │   └──────────────┘  └──────────────┘  └──────────────────────────┘  │    │
│  │                                                                      │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                      │                                      │
│                                      ▼                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                         数据存储                                      │    │
│  │                                                                      │    │
│  │   ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐  │    │
│  │   │  MySQL      │  │  Redis       │  │  Zookeeper              │  │    │
│  │   │  (持久化)   │  │  (缓存/锁)  │  │  (服务注册)            │  │    │
│  │   └──────────────┘  └──────────────┘  └──────────────────────────┘  │    │
│  │                                                                      │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.4 命令模式架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Command模式实现架构                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐ │
│   │                      CommandExecutor                                   │ │
│   │   (命令执行器 - 负责事务、日志、拦截器)                              │ │
│   └─────────────────────────────────────────────────────────────────────┘ │
│                                      │                                      │
│                                      ▼                                      │
│   ┌─────────────────────────────────────────────────────────────────────┐ │
│   │                     Command<T> 接口                                  │ │
│   │   ┌─────────────────────────────────────────────────────────────┐ │ │
│   │   │  T execute(CommandContext context);                          │ │ │
│   │   └─────────────────────────────────────────────────────────────┘ │ │
│   └─────────────────────────────────────────────────────────────────────┘ │
│                                      │                                      │
│                          ┌────────────┴────────────┐                        │
│                          ▼                         ▼                        │
│   ┌──────────────────────────────┐  ┌──────────────────────────────┐      │
│   │      查询命令 (Read)          │  │      操作命令 (Write)        │      │
│   │                              │  │                              │      │
│   │  ┌────────────────────────┐ │  │  ┌────────────────────────┐ │      │
│   │  │ GetTaskCmd           │ │  │  │ GenarateTaskCmd        │ │      │
│   │  │ GetServiceCmd        │ │  │  │ SubmitTaskProcessCmd   │ │      │
│   │  │ TaskQuery            │ │  │  │ ForwardTaskCmd        │ │      │
│   │  │ ServiceQuery          │ │  │  │ TransferTaskCmd       │ │      │
│   │  └────────────────────────┘ │  │  └────────────────────────┘ │      │
│   └──────────────────────────────┘  └──────────────────────────────┘      │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐ │
│   │                    CommandContext                                    │ │
│   │   (命令上下文 - 持有所有EntityManager引用)                          │ │
│   │   ┌─────────────────────────────────────────────────────────────┐ │ │
│   │   │   serviceEntityManager   │   taskEntityManager            │ │ │
│   │   │   alarmEntityManager    │   deploymentManager            │ │ │
│   │   │   sessionFactory        │   dataManager                  │ │ │
│   │   └─────────────────────────────────────────────────────────────┘ │ │
│   └─────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. 核心数据模型

### 3.1 Service (服务)

```java
public interface Service {
    // 基本属性
    String getSvno();           // 服务号
    String getRtyp();          // 资源类型
    String getRtnm();          // 资源名称
    String getSrid();          // 服务规则ID
    String getSvnm();          // 服务名称
    String getPcid();          // 当前进程ID
    String getPcnm();          // 当前进程名称
    Integer getPidx();         // 当前进程顺序号
    String getStat();          // 当前状态 (PLN/DWN/ACP/BEG/ARV/WRK/END)
    String getTstn();          // 状态名称
    
    // 航班关联
    String getFkey();          // 航班唯一号
    String getFlop();          // 航班日
    String getFlno();          // 航班号
    String getAdid();          // 进离港 (A/D)
    
    // 时间属性
    String getDpbe();         // 计划开始时间
    String getDpen();         // 计划结束时间
    String getDabe();         // 实际开始时间
    String getDaen();         // 实际结束时间
    String getPftm();         // 计划服务总时间
    String getSftm();         // 标准服务总时间
    String getAftm();         // 实际服务总时间
    
    // 关联数据
    List<ServiceProcess> getServiceProcesses();   // 服务进程列表
    List<ServiceMark> getServiceMarks();          // 服务标注列表
}
```

### 3.2 Task (任务)

```java
public interface Task {
    // 基本属性
    String getTnb();           // 任务号
    String getSvno();          // 服务号
    String getTnam();          // 任务名称
    String getTrid();          // 任务规则ID
    String getTrnm();          // 任务规则名称
    String getRtyp();          // 资源类型
    String getPcid();          // 当前进程ID
    String getPcnm();          // 当前进程名称
    Integer getPidx();         // 当前进程顺序号
    String getStat();          // 当前状态 (PLN/DWN/ACP/BEG/ARV/WRK/END/CXX)
    String getTstn();          // 状态名称
    
    // 资源分配
    String getTenb();         // 任务执行人
    String getTenm();          // 执行人姓名
    String getRlid();          // 岗位代码
    String getDepid();         // 部门编号
    String getVnb();          // 车号
    String getSnb();          // 资源号
    
    // 航班关联
    String getFkey();         // 航班唯一号
    String getFlop();          // 航班日
    String getFlno();          // 航班号
    String getAdid();          // 进离港
    String getRegn();          // 机号
    String getActn();          // 机型
    
    // 时间属性
    String getDpbe();         // 计划开始时间
    String getDpen();         // 计划结束时间
    String getDabe();         // 实际开始时间
    String getDaen();         // 实际结束时间
    String getDacx();         // 实际取消时间
    String getPftm();         // 计划服务总时间
    String getSftm();         // 标准服务总时间
    String getAftm();         // 实际服务总时间
    String getCoef();         // 服务效率
    
    // 特殊标记
    String getPmod();         // 派工模式 (A-自动/M-非自动)
    String getFrwd();          // 允许转发
    String getTrsf();         // 允许移交
    String getIsd();          // 限制唯一
    
    // 关联数据
    List<TaskProcess> getTaskProcesss();  // 任务进程
    List<TaskMark> getTaskMarks();         // 任务标注
}
```

### 3.3 任务状态机

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           任务状态流转图                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────┐                                                             │
│   │   PLN   │  计划                                                        │
│   └────┬────┘                                                             │
│        │                                                                   │
│        ▼                                                                   │
│   ┌─────────┐                                                             │
│   │   DWN   │  派发                                                        │
│   └────┬────┘                                                             │
│        │                                                                   │
│        ▼                                                                   │
│   ┌─────────┐     ┌─────────┐     ┌─────────┐                              │
│   │   ACP   │────▶│   BEG   │────▶│   ARV   │                              │
│   │  接受   │     │  开始   │     │  到位   │                              │
│   └─────────┘     └────┬────┘     └────┬────┘                              │
│                       │               │                                     │
│                       ▼               ▼                                     │
│                 ┌─────────┐     ┌─────────┐                                │
│                 │   WRK   │     │   CXX   │                                │
│                 │  作业   │     │  取消   │                                │
│                 └────┬────┘     └─────────┘                                │
│                      │                                                     │
│                      ▼                                                     │
│               ┌─────────┐                                                  │
│               │   END   │  结束                                               │
│               └─────────┘                                                   │
│                                                                              │
│   状态说明:                                                                 │
│   ┌────────┬────────────────────────────────────────────────────────┐     │
│   │  PLN   │  计划 - 任务已创建，未派发                              │     │
│   ├────────┼────────────────────────────────────────────────────────┤     │
│   │  DWN   │  派发 - 任务已分配给执行人                             │     │
│   ├────────┼────────────────────────────────────────────────────────┤     │
│   │  ACP   │  接受 - 执行人已接受任务                              │     │
│   ├────────┼────────────────────────────────────────────────────────┤     │
│   │  BEG   │  开始 - 开始执行任务                                  │     │
│   ├────────┼────────────────────────────────────────────────────────┤     │
│   │  ARV   │  到位 - 资源已到达作业点                             │     │
│   ├────────┼────────────────────────────────────────────────────────┤     │
│   │  WRK   │  作业 - 正在作业中                                  │     │
│   ├────────┼────────────────────────────────────────────────────────┤     │
│   │  END   │  结束 - 任务已完成                                   │     │
│   ├────────┼────────────────────────────────────────────────────────┤     │
│   │  CXX   │  取消 - 任务已取消                                   │     │
│   └────────┴────────────────────────────────────────────────────────┘     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.4 Alarm (报警)

```java
public interface Alarm {
    String getAlno();          // 报警号
    String getFkey();          // 航班唯一号
    String getTenantId();     // 租户ID
    String getStat();         // 状态 (Y-已报警/N-未报警)
    String getPmid();         // 报警类型
    String getPmnm();         // 报警名称
    String getAltm();         // 报警时间
    String getAitm();         // 实际时间
    String getPltm();         // 计划时间
    String getUnit();         // 时间差单位
    String getAlcd();         // 报警代码
    String getAlvl();         // 报警级别
}
```

### 3.5 PushMessage (推送消息)

```java
public interface PushMessage {
    // 消息组件类型
    String COMP_STANDARD = "STANDARD_";      // 服务标准
    String COMP_SERVICE = "SERVICE_";       // 服务
    String COMP_SERVICE_PROCESS = "SERVICE_PROCESS_"; // 服务进程
    String COMP_SERVICE_MARK = "SERVICE_MARK_"; // 服务标注
    String COMP_TASK = "TASK_";            // 任务
    String COMP_TASK_PROCESS = "TASK_PROCESS_"; // 任务进程
    String COMP_TASK_MARK = "TASK_MARK_";   // 任务标注
    String COMP_ALARM = "ALARM_";           // 报警
    
    // 消息类型
    String TYPE_STANDARD_ADD = "STANDARD_ADD";
    String TYPE_STANDARD_UPD = "STANDARD_UPD";
    String TYPE_STANDARD_DEL = "STANDARD_DEL";
    
    String TYPE_SERVICE_ADD = "SERVICE_ADD";
    String TYPE_SERVICE_UPD = "SERVICE_UPD";
    String TYPE_SERVICE_FLUSH = "SERVICE_FLUSH";
    
    String TYPE_TASK_ADD = "TASK_ADD";
    String TYPE_TASK_UPD = "TASK_UPD";
    String TYPE_TASK_DWN = "TASK_DWN";      // 任务派发
    String TYPE_TASK_TRSF = "TASK_TRSF";   // 任务移交
    
    String TYPE_TASK_PROCESS_ADD = "TASK_PROCESS_ADD";
    String TYPE_TASK_PROCESS_UPD = "TASK_PROCESS_UPD";
    
    String TYPE_TASK_ALARM_UPD = "ALARM_TASK_UPD";
    String TYPE_SERVICE_ALARM_UPD = "ALARM_SERVICE_UPD";
}
```

---

## 4. 核心业务逻辑详解

### 4.1 任务生成流程 (GenarateTaskCmd)

```java
public class GenarateTaskCmd implements Command<Task> {
    private String rtyp;           // 服务类型
    private String usid;           // 任务执行者
    private String tenant;         // 租户
    private String fkey;          // 航班唯一号
    private String dispatchUsid;  // 派发者
    private String car;           // 车号
    
    private String tnam;          // 任务名称
    private String remk;          // 备注
    private String fromTerminal;  // 是否来自终端
    private String pmod;          // 派工模式
    
    @Override
    public Task execute(CommandContext context) {
        // 1. 校验航班存在
        Flight flight = context.getDataManager().getFlightByFkey(fkey, tenant);
        if (flight == null) {
            throw new NotFindFlightException("航班不存在");
        }
        
        // 2. 校验执行用户存在
        User user = context.getDataManager().getUserByUsid(usid, tenant);
        if (user == null) {
            throw new NotFindUsrException("用户不存在");
        }
        
        // 3. 获取服务规则模型
        ServiceRuleModel ruleModel = context.getDeploymentManager()
            .getServiceRuleModelWithTenantRtypDimension(
                tenant, rtyp, dimension, match);
        
        // 4. 创建任务
        TaskEntity task = new TaskEntity();
        task.setTnb(generateTnb());  // 生成任务号
        task.setFkey(fkey);
        task.setTenant(tenant);
        task.setUsid(usid);
        task.setStat("DWN");  // 初始状态为派发
        
        // 5. 生成任务进程
        List<TaskProcess> processes = generateTaskProcesses(ruleModel, flight);
        task.setTaskProcesss(processes);
        
        // 6. 保存任务
        TaskEntity savedTask = context.getTaskEntityManager().save(task);
        
        // 7. 发送推送消息
        pushMessageService.sendTaskMessage(savedTask, "ADD");
        
        return savedTask;
    }
}
```

### 4.2 任务派发流程

```java
// InstanceServiceImpl.java

/**
 * 派发任务
 */
public Task dispatchTask(String dispatchUsid, String fkey, String tenant, 
                          String usid, String rtyp, String tnam, String remk) {
    GenarateTaskCmd cmd = new GenarateTaskCmd(tenant, fkey, rtyp, usid);
    cmd.dispatchUsid(dispatchUsid);
    cmd.tnam(tnam);
    cmd.remk(remk);
    return commandExecutor.execute(cmd);
}

/**
 * 终端申请任务
 */
public Task applyTask(String fkey, String tenant, String usid, String rtyp) {
    GenarateTaskCmd cmd = new GenarateTaskCmd(tenant, fkey, rtyp, usid);
    cmd.fromTerminal();
    Task task = commandExecutor.execute(cmd);
    
    // 检查是否需要调度审批
    if (GenarateTaskCmd.RESP_APPLY_TASK_M.equals(cmd.getRespCode())) {
        throw new ApplyTaskMException();  // 需要审批
    }
    
    return task;
}

/**
 * 同步派发任务 (浦东专用)
 */
public Task syndispatchTask(String dispatchUsid, String fkey, String tenant, 
                           String usid, String rtyp, String tnam, 
                           List<TaskMark> taskMarks, String remk, String car) {
    GenarateTaskCmd cmd = new GenarateTaskCmd(tenant, fkey, rtyp, usid, car);
    cmd.dispatchUsid(dispatchUsid);
    cmd.tnam(tnam);
    cmd.setTaskMarks(taskMarks);
    cmd.remk(remk);
    return commandExecutor.execute(cmd);
}
```

### 4.3 任务进程提交流程

```java
/**
 * 上报任务节点
 */
public TaskProcess submitTaskProcess(String tnb, String pcid, String atim, 
                                     String usid, String tenant) {
    SubmitTaskProcessCmd cmd = new SubmitTaskProcessCmd(tnb, pcid, atim, usid, tenant);
    return commandExecutor.execute(cmd);
}

/**
 * 终端上报任务节点
 */
public Service submitTaskProcessByTerminal(String tenantId, String tnb, String pcid, 
                                          String atim, String senb, String remk) {
    // 1. 检查任务是否已结束
    Task task = this.getFullTaskByTnb(tenantId, tnb);
    if ("END".equals(task.getStat())) {
        throw new EndTaskException("任务已结束，不能再次提交");
    }
    
    // 2. 提交任务进程
    TaskProcess taskProcess = submitTaskProcess(tnb, pcid, atim, senb, tenantId, remk);
    
    // 3. 处理事件任务
    if (isEventTask(task)) {
        handleEventTask(task, taskProcess);
    }
    
    // 4. 返回关联的服务
    return getServiceBySvno(tenantId, task.getSvno());
}
```

### 4.4 任务转发与移交

```java
/**
 * 转发任务
 */
public List<Task> forwardTask(String tenantId, String tftnb, String tftenb, 
                              List<String> tenbs) {
    ForwardTaskCmd cmd = new ForwardTaskCmd(tenantId, tftnb, tftenb, tenbs);
    return commandExecutor.execute(cmd);
}

/**
 * 移交任务
 */
public List<Task> transferTask(String tenantId, List<String> tnbs, 
                               String dtenb, String otenb) {
    TransferTaskCmd cmd = new TransferTaskCmd(tenantId, tnbs, dtenb, otenb);
    return commandExecutor.execute(cmd);
}

/**
 * 获取待移交任务
 */
public List<TaskTransfer> getUnfinishedTaskTransfersByDtenb(String tenantId, String dtenb) {
    GetTaskTransfersCmd cmd = new GetTaskTransfersCmd(tenantId, dtenb);
    return commandExecutor.execute(cmd);
}
```

### 4.5 航班锁定与任务生成

```java
/**
 * 根据航班锁定资源生成任务
 */
public void dispatchTaskByFlightLock() {
    FlightLockQuery query = new FlightLockQueryImpl(commandExecutor);
    List<FlightLockRef> lockRefs = query.list();
    
    for (FlightLockRef ref : lockRefs) {
        try {
            if (FlightLockVo.TASKMODE_TASK.equals(ref.getTaskMode())) {
                // 1. 检查用户是否在线
                User user = dataManager.getUserByUsid(ref.getUserId(), ref.getTenantId());
                if (user != null && "Y".equals(user.getOnln())) {
                    
                    // 2. 获取服务标准
                    ServiceStandard standard = commandExecutor.execute(
                        new GetServiceStandardCmd(ref.getTenantId(), 
                                                  ref.getFlseq(), 
                                                  ref.getServiceType()));
                    
                    if (standard != null) {
                        // 3. 生成任务
                        GenarateTaskCmd cmd = new GenarateTaskCmd(
                            ref.getTenantId(), ref.getFlseq(), 
                            ref.getServiceType(), ref.getUserId());
                        commandExecutor.execute(cmd);
                    }
                }
            }
        } catch (Exception e) {
            logger.error("资源锁定任务生成异常: " + e.getMessage());
        }
    }
}
```

---

## 5. 服务标准与规则配置

### 5.1 标准配置结构 (standard.xml)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<serviceStandards>
    <!-- 进港服务标准 -->
    <serviceStandard id="adid_A" name="进港">
        <group key="adid" value="A">进港</group>
        <standard id="JY" type="JY" name="加油">
            <!-- 开始时间: 计划时间=预计到达时间-30分钟 -->
            <startTime planTime="stdTime(STOT, -1800)" 
                       ifAlarm="N"  
                       alarmTime="stdTime(CTOT, -1800)"/>
            <!-- 结束时间 -->
            <endTime planTime="stdTime(STOT, -1800)" 
                     ifAlarm="N" 
                     alarmTime="stdTime(CTOT, -1800)"/>
        </standard>
    </serviceStandard>
    
    <!-- 离港服务标准 -->
    <serviceStandard id="adid_D" name="离港">
        <group key="adid" value="D">离港</group>
        <standard id="JY" type="JY" name="加油">
            <startTime planTime="stdTime(STOT, -1800)" 
                       ifAlarm="N" 
                       alarmTime="stdTime(CTOT, -1800)"/>
            <endTime planTime="stdTime(STOT, -1800)" 
                     ifAlarm="N" 
                     alarmTime="stdTime(CTOT, -1800)"/>
        </standard>
    </serviceStandard>
</serviceStandards>
```

### 5.2 部署管理器 (DeploymentManager)

```java
public class DeploymentManager {
    // 按租户存储的服务标准
    private Map<String, List<ServiceStandardModel>> serviceStandardModelsWithTenant;
    
    // 按租户+纬度存储的服务标准
    private Map<String, ServiceStandardModel> serviceStandardModelWithTenantDimension;
    
    // 按租户+类型存储的服务规则
    private Map<String, List<ServiceRuleModel>> serviceRuleModelsWithTenantRtyp;
    
    // 参考时间影响维度
    private Map<String, Set<RefTimeVariable>> refTimeForFlightDimension;
    
    /**
     * 添加服务标准模型
     */
    protected void addServiceStandardModel(String tenant, List<ServiceStandardModel> models) {
        for (ServiceStandardModel ssm : models) {
            // 1. 添加到租户列表
            serviceStandardModelsWithTenant.get(tenant).add(ssm);
            
            // 2. 计算纬度维度键
            String tenantDimension = buildTenantDimension(tenant, ssm);
            
            // 3. 存储到精确匹配Map
            serviceStandardModelWithTenantDimension.put(tenantDimension, ssm);
            
            // 4. 提取参考时间维度
            Map<String, Set<RefTimeVariable>> refTimeDimension = ssm.getRefTimeDimension();
            for (String refTimeKey : refTimeDimension.keySet()) {
                if (refTimeKey.startsWith("flight.")) {
                    // 处理航班参考时间
                    String keyWithTenant = tenant + ":" + refTimeKey;
                    refTimeForFlightDimension.put(keyWithTenant, refTimeDimension.get(refTimeKey));
                }
            }
        }
    }
    
    /**
     * 获取服务标准模型 (带纬度匹配)
     */
    public ServiceStandardModel getServiceStandardModelWithTenantDimension(
            String tenant, Set<String> dimension, Map<String, String> match) {
        // 构建维度键
        String objDimensionKey = buildDimensionKey(dimension, match);
        objDimensionKey = tenant + ":" + objDimensionKey;
        
        return serviceStandardModelWithTenantDimension.get(objDimensionKey);
    }
}
```

---

## 6. 报警机制

### 6.1 报警类型

| 报警类型 | 说明 | 触发条件 |
|----------|------|----------|
| **ServiceProcessAlarm** | 服务进程报警 | 服务节点超时 |
| **TaskProcessAlarm** | 任务进程报警 | 任务节点超时 |
| **ServiceMonitorAlarm** | 服务监控项报警 | 监控项超时 |
| **TaskMonitorAlarm** | 任务监控项报警 | 监控项超时 |
| **ServiceMarkAlarm** | 服务标注项报警 | 标注项超时 |
| **TaskMarkAlarm** | 任务标注项报警 | 标注项超时 |
| **StandardAlarm** | 服务标准报警 | 超出服务时间 |

### 6.2 报警查询示例

```java
// InstanceServiceImpl.java

/**
 * 获取服务进程报警
 */
public List<Alarm> getServiceProcessAlarm(String tenantId) {
    String egtFlop = dataManager.getEFlopByTenant(tenantId);
    
    AlarmQuery alarmQuery = new AlarmQueryImpl(commandExecutor);
    alarmQuery.egtFlop(egtFlop);
    alarmQuery.tenantId(tenantId);
    alarmQuery.serviceProcessAlarm();  // 服务进程报警
    alarmQuery.alarmed();               // 已报警
    alarmQuery.neqEvent();             // 非通用任务
    alarmQuery.enable("Y");            // 启用
    
    return alarmQuery.list();
}

/**
 * 获取当前未报警的超时报警
 */
public List<Alarm> getUnAlarmLimitNow() {
    AlarmQuery alarmQuery = new AlarmQueryImpl(commandExecutor);
    alarmQuery.unAlarm();              // 未报警
    alarmQuery.limitNow();             // 当前超时
    alarmQuery.enable("Y");
    return alarmQuery.list();
}

/**
 * 推迟报警
 */
public void deferAlarm(String tenantId, String alno, Integer deferTime, String dfus) {
    DeferAlarmCmd cmd = new DeferAlarmCmd(tenantId, alno, deferTime, dfus);
    commandExecutor.execute(cmd);
}
```

---

## 7. 消息推送机制

### 7.1 推送协议

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         推送消息协议分类                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────┐    │
│  │                     服务标准消息 (STANDARD_*)                        │    │
│  │   TYPE_STANDARD_ADD  - 新增服务标准                                 │    │
│  │   TYPE_STANDARD_UPD  - 更新服务标准                                 │    │
│  │   TYPE_STANDARD_DEL  - 删除服务标准                                 │    │
│  └───────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────┐    │
│  │                       服务消息 (SERVICE_*)                         │    │
│  │   TYPE_SERVICE_ADD   - 新增服务                                    │    │
│  │   TYPE_SERVICE_UPD   - 更新服务                                    │    │
│  │   TYPE_SERVICE_FLUSH  - 刷新服务时间                                │    │
│  └───────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────┐    │
│  │                       任务消息 (TASK_*)                            │    │
│  │   TYPE_TASK_ADD      - 新增任务                                    │    │
│  │   TYPE_TASK_UPD      - 更新任务                                    │    │
│  │   TYPE_TASK_DWN      - 任务派发                                    │    │
│  │   TYPE_TASK_TRSF     - 任务移交                                    │    │
│  │   TYPE_TASK_ALARM    - 任务报警                                    │    │
│  └───────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────┐    │
│  │                       进程消息 (PROCESS_*)                          │    │
│  │   TYPE_TASK_PROCESS_ADD    - 新增任务进程                          │    │
│  │   TYPE_TASK_PROCESS_UPD    - 更新任务进程                          │    │
│  │   TYPE_SERVICE_PROCESS_ADD  - 新增服务进程                          │    │
│  │   TYPE_SERVICE_PROCESS_UPD  - 更新服务进程                          │    │
│  └───────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────┐    │
│  │                     标注消息 (MARK_*)                              │    │
│  │   TYPE_TASK_MARK_ADD     - 新增任务标注                            │    │
│  │   TYPE_TASK_MARK_UPD     - 更新任务标注                            │    │
│  │   TYPE_SERVICE_MARK_ADD  - 新增服务标注                            │    │
│  │   TYPE_SERVICE_MARK_UPD  - 更新服务标注                            │    │
│  └───────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7.2 推送实现

```java
// PushMessageService.java
public class PushMessageService {
    
    /**
     * 发送任务消息
     */
    public void sendTaskMessage(Task task, String type) {
        PushMessage message = createPushMessage();
        message.setType(type);  // ADD/UPD
        message.setData(serializeTask(task));
        message.setTenantId(task.getTenantId());
        message.setFkey(task.getFkey());
        
        // 发送到ActiveMQ
        jmsTemplate.convertAndSend("taskTopic", message);
        
        // 同时发送到Redis Pub/Sub
        redissonClient.getTopic("taskChannel").publish(message);
    }
    
    /**
     * 发送任务派发消息
     */
    public void sendTaskDispatchMessage(Task task, String targetUser) {
        PushMessage message = createPushMessage();
        message.setType(PushMessage.TYPE_TASK_DWN);
        message.setData(serializeTask(task));
        message.setTenantId(task.getTenantId());
        message.setBatch(targetUser);  // 推送到特定用户
        
        // 通过Camel路由发送
        camelTemplate.sendBody("activemq:queue:task.dispatch", message);
    }
}
```

---

## 8. 数据访问层

### 8.1 EntityManager架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         EntityManager架构                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                      CommandContext                                  │  │
│   │   (命令上下文 - 持有所有EntityManager)                                │  │
│   │                                                                      │  │
│   │   ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐  │  │
│   │   │ ServiceEntity│  │ TaskEntity   │  │ AlarmEntityManager     │  │  │
│   │   │ Manager      │  │ Manager      │  │                       │  │  │
│   │   │(服务实体管理)│  │(任务实体管理)│  │                       │  │  │
│   │   └──────────────┘  └──────────────┘  └──────────────────────────┘  │  │
│   │                                                                      │  │
│   │   ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐  │  │
│   │   │ ServiceProcess│  │ ServiceMark  │  │ DeploymentManager      │  │  │
│   │   │ EntityManager │  │ EntityManager│  │                       │  │  │
│   │   └──────────────┘  └──────────────┘  └──────────────────────────┘  │  │
│   │                                                                      │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                      │                                      │
│                                      ▼                                      │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                    MyBatis Mapper                                   │  │
│   │                                                                      │  │
│   │   ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐  │  │
│   │   │ServiceMapper │  │ TaskMapper   │  │ AlarmMapper            │  │  │
│   │   │(SQL映射)     │  │(SQL映射)     │  │(SQL映射)              │  │  │
│   │   └──────────────┘  └──────────────┘  └──────────────────────────┘  │  │
│   │                                                                      │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 8.2 Query查询构建器

```java
// TaskQuery.java
public interface TaskQuery extends Query<TaskQuery, Task> {
    TaskQuery tenantId(String tenantId);
    TaskQuery tnb(String tnb);
    TaskQuery fkey(String fkey);
    TaskQuery flno(String flno);
    TaskQuery tenb(String tenb);
    TaskQuery vnb(String vnb);
    TaskQuery rtyp(String rtyp);
    TaskQuery stat(String stat);
    TaskQuery eqFlop(String flop);
    TaskQuery egtFlop(String flop);  // >=
    TaskQuery fullTask();
    TaskQuery standardProcess();
    TaskQuery neqEvent();             // != 通用任务
    TaskQuery event();                // = 通用任务
    TaskQuery statNewTask();
    TaskQuery statUnfinishedTask();
    TaskQuery normalTask();
    TaskQuery orderByRtyp();
}

// 使用示例
TaskQuery query = new TaskQueryImpl(commandExecutor);
query.tenantId("PVG")
      .eqFlop("20240204")
      .tenb("U0001")
      .statUnfinishedTask()
      .neqEvent();
List<Task> tasks = query.list();
```

---

## 9. 业务场景分析

### 9.1 任务全生命周期

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         任务全生命周期流程                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────┐                                                               │
│  │ 航班到达 │  航班数据从外部系统同步到本系统                                    │
│  └────┬────┘                                                               │
│       │                                                                      │
│       ▼                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                     InstanceServiceImpl                              │   │
│  │                                                                      │   │
│  │   ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐  │   │
│  │   │genarateService│ │genarateTask  │  │ dispatchTask           │  │   │
│  │   │  生成服务    │  │  生成任务    │  │  派发任务              │  │   │
│  │   └──────────────┘  └──────────────┘  └──────────────────────────┘  │   │
│  │                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│       │                                                                      │
│       ▼                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                        任务状态流转                                   │   │
│  │                                                                      │   │
│  │   PLN(计划) → DWN(派发) → ACP(接受) → BEG(开始) → ARV(到位)          │   │
│  │                                              ↓                       │   │
│  │                                              WRK(作业)               │   │
│  │                                              ↓                       │   │
│  │                                              END(结束)               │   │
│  │                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│       │                                                                      │
│       ▼                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                     消息推送机制                                     │   │
│  │                                                                      │   │
│  │   终端订阅 ← PushMessageService ← ActiveMQ/Camel/Redis             │   │
│  │                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│       │                                                                      │
│       ▼                                                                      │
│  ┌─────────┐                                                               │
│  │ 任务结束 │  更新油单、生成报表、结算                                      │
│  └─────────┘                                                               │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 9.2 报警处理流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         报警处理流程图                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                     AlarmChecker (定时任务)                          │  │
│   │   @Scheduled(cron = "0 0/5 * * * ?")  每5分钟执行一次              │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                      │                                      │
│                                      ▼                                      │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                    1. 查询未报警的超时记录                          │  │
│   │                                                                      │  │
│   │   List<Alarm> alarms = getUnAlarmLimitNow();                       │  │
│   │                                                                      │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                      │                                      │
│                          ┌────────────┴────────────┐                       │
│                          ▼                         ▼                       │
│   ┌────────────────────────────────┐  ┌────────────────────────────────┐      │
│   │  ServiceProcessAlarm          │  │ TaskProcessAlarm              │      │
│   │  服务进程超时报警             │  │ 任务进程超时报警               │      │
│   └────────────────────────────────┘  └────────────────────────────────┘      │
│                          │                         │                       │
│                          ▼                         ▼                       │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                    2. 创建报警记录                                  │  │
│   │                                                                      │  │
│   │   Alarm alarm = new Alarm();                                       │  │
│   │   alarm.setAlno(generateAlno());                                   │  │
│   │   alarm.setStat("Y");  // 已报警                                   │  │
│   │   alarmEntityManager.save(alarm);                                  │  │
│   │                                                                      │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                      │                                      │
│                                      ▼                                      │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                    3. 发送报警推送消息                               │  │
│   │                                                                      │  │
│   │   pushMessageService.sendAlarmMessage(alarm);                       │  │
│   │                                                                      │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                      │                                      │
│                                      ▼                                      │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                    4. 终端接收并展示                                 │  │
│   │                                                                      │  │
│   │   调度界面显示红色报警标记                                           │  │
│   │   终端设备震动/声音提醒                                              │  │
│   │                                                                      │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 10. 问题与优化建议

### 10.1 技术债务

| 问题 | 严重程度 | 说明 | 建议 |
|------|----------|------|------|
| **Java 1.7 已停止支持** | 🔴 高 | 无法使用新特性，存在安全风险 | 升级到 1.8+ |
| **Spring 4.0.8 过时** | 🟡 中 | 建议升级到 4.3.x 或 5.x | 升级到 5.x |
| **Dubbo 2.5.3 过时** | 🟡 中 | 建议升级到 2.7.x | 考虑升级或迁移到Spring Cloud |
| **ActiveMQ 5.12.0** | 🟡 中 | 建议升级到 5.17.x | 升级或迁移到Kafka |
| **Apache Camel 2.16** | 🟡 中 | 版本较老 | 升级到 3.x |
| **依赖版本不一致** | 🟡 中 | 不同模块使用不同版本 | 统一版本管理 |

### 10.2 性能问题

| 问题 | 描述 | 建议 |
|------|------|------|
| **命令执行器性能** | 每次操作都通过CommandExecutor，可能有额外开销 | 评估是否需要拆分为读写分离 |
| **DeploymentManager内存** | 所有配置加载到内存，配置多时内存占用大 | 考虑配置懒加载或压缩存储 |
| **报警检查定时任务** | 每5分钟全表扫描检查超时 | 优化为增量检查或使用时间索引 |
| **Query查询构建** | 大量使用Builder模式，代码冗长 | 使用Lambda表达式简化 |

```java
// 问题：每次查询都创建新的Query对象
TaskQuery query = new TaskQueryImpl(commandExecutor);
query.tenantId(tenantId);
query.eqFlop(flop);
query.statUnfinishedTask();
// ...
List<Task> tasks = query.list();

// 优化建议：使用缓存 + Lambda
// List<Task> tasks = taskRepository.findByTenantAndStat(tenantId, "DWN");
```

### 10.3 代码质量问题

```java
// 问题1: 方法过长
public Service submitTaskProcessByTerminal(String tenantId, String tnb, ...) {
    // 1973行文件，方法过长难以维护
}

// 优化建议：拆分方法
private void validateTaskNotEnded(Task task) { ... }
private void handleEventTask(Task task, TaskProcess process) { ... }
private Service buildResponseService(Task task) { ... }

// 问题2: 异常处理不规范
} catch (Exception e) {
    logger.error("异常: " + e.getMessage());  // 丢失堆栈
}

// 优化建议：保留完整异常信息
} catch (Exception e) {
    logger.error("任务处理异常", e);
    throw new ServiceException("任务处理失败", e);
}

// 问题3: 魔法数字
if ("END".equals(task.getStat())) {
    throw new EndTaskException("...");
}
if ("Y".equals(user.getOnln())) {
    // ...
}

// 优化建议：使用常量
private static final String STAT_END = "END";
private static final String ONLINE_YES = "Y";
```

### 10.4 安全建议

| 问题 | 描述 | 建议 |
|------|------|------|
| **SQL注入风险** | 动态拼接SQL | 使用参数化查询 |
| **命令执行权限** | 所有Command都通过执行器 | 添加权限控制 |
| **数据越权** | 未严格校验租户ID | 强校验多租户隔离 |

---

## 11. 总结

### 11.1 模块特点

| 特点 | 说明 |
|------|------|
| **命令模式** | 所有业务操作通过Command模式执行，便于事务管理 |
| **状态机** | 任务/服务状态机驱动业务流程 |
| **配置化** | 服务标准、规则通过XML配置，灵活可配置 |
| **多租户** | 支持多租户隔离，租户ID作为核心维度 |
| **消息推送** | 集成ActiveMQ/Camel/Redis实现多通道推送 |
| **分布式锁** | 使用Redisson实现分布式锁，防止并发冲突 |
| **报警机制** | 完善的超时报警体系，支持报警推迟 |

### 11.2 核心能力

1. **服务管理**：服务标准定义、服务生成、服务进程跟踪
2. **任务管理**：任务派发、任务跟踪、任务转发/移交
3. **资源管理**：航班锁定、用户资源绑定
4. **报警监控**：超时报警、监控项报警、报警推迟
5. **消息推送**：ActiveMQ/Camel/Redis多通道推送

### 11.3 依赖关系

```
servicemanager
├── service-engine (核心引擎)
│   ├── instance/ (业务实体: Service, Task, Alarm...)
│   ├── cmd/ (命令实现: GenarateTaskCmd, SubmitTaskProcessCmd...)
│   ├── persistence/ (EntityManager, Mapper)
│   ├── query/ (Query构建器)
│   └── push/ (消息推送)
│
├── service-converter (配置转换)
├── service-config-validation (配置验证)
├── service-dubbo-rcp (Dubbo接口)
├── service-dubbo-impl (Dubbo实现)
│   └── standard/ (XML配置: standard.xml, *.rule.xml)
│
└── service-fw (框架工具)
    ├── minilang/ (迷你语言)
    └── logging/ (日志)
```

### 11.4 后续建议

1. **架构升级**：
   - Java 1.7 → 1.8/11
   - Spring 4.0 → 5.x
   - Dubbo → Spring Cloud Alibaba

2. **性能优化**：
   - 添加查询缓存
   - 优化报警检查机制
   - 配置懒加载

3. **可观测性**：
   - 添加链路追踪
   - 完善监控指标
   - 统一日志规范

4. **代码质量**：
   - 拆分长方法
   - 统一异常处理
   - 添加单元测试覆盖

---

*报告生成时间：2026-02-04*
*分析工具：Cursor AI Code Analysis*
