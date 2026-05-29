# 智慧航空加油系统 (Smart Aviation Fueling System)

<p align="center">
  <img src="https://img.shields.io/badge/Java-1.8-blue" alt="Java">
  <img src="https://img.shields.io/badge/Spring-4.3.3-brightgreen" alt="Spring">
  <img src="https://img.shields.io/badge/Apache_Dubbo-2.5.3-red" alt="Dubbo">
  <img src="https://img.shields.io/badge/MySQL-5.7-yellow" alt="MySQL">
  <img src="https://img.shields.io/badge/Vue-2.6-blue" alt="Vue">
  <img src="https://img.shields.io/badge/Angular-15.x-blue" alt="Angular">
</p>

> 上海浦东机场核心业务系统 - 航油供应全流程数字化管理平台

## 📋 目录

- [项目简介](#项目简介)
- [核心功能](#核心功能)
- [技术架构](#技术架构)
- [项目结构](#项目结构)
- [快速开始](#快速开始)
- [模块说明](#模块说明)
- [API 文档](#api-文档)
- [部署指南](#部署指南)
- [配置说明](#配置说明)
- [常见问题](#常见问题)
- [技术债务](#技术债务)
- [更新日志](#更新日志)
- [许可证](#许可证)

---

## 项目简介

### 1.1 项目背景

**智慧航空加油系统**是服务于上海浦东机场的核心业务系统之一，基于微服务架构设计。系统采用 Apache Dubbo 作为 RPC 框架，Spring Framework 作为应用框架，MySQL 作为主数据库，Redis 作为缓存中间件，Zookeeper 作为服务注册中心，Apache Camel 和 ActiveMQ 作为消息中间件。

系统的核心业务价值在于实现航油供应全流程的数字化管理：
- 从航班落地到加油完成再到电子签单确认
- 处理浦东机场每天 **数百架次** 航班的加油任务
- 服务东航、南航、国航、海航等多家航空公司
- 实时性要求高，数据准确性要求严格

### 1.2 项目规模

| 指标 | 数值 | 说明 |
|------|------|------|
| 总模块数 | 28 个业务模块 | 涵盖前后端、数据、接口等 |
| Java 文件数 | 2500+ | 包含业务代码、配置、工具类等 |
| 数据库表数 | 200+ | 涵盖业务表、配置表、日志表等 |
| Vue 组件数 | 127+ | 智慧航空加油系统前端模块 |
| Angular 组件数 | 581+ | Pudong-Web 模块 |
| Dubbo 服务接口数 | 100+ | 对外暴露的 RPC 服务 |
| REST API 端点数 | 200+ | HTTP 接口 |

---

## 核心功能

### 🚀 航班信息管理

- 通过 ESB（企业服务总线）接收航空公司和机场的航班动态信息
- 支持航班计划、延误信息、变更信息等实时处理
- 多航空公司数据标准化处理（东航、南航、国航等）
- 处理航班号、机型、机号、起降时间、登机口等多种信息

### ⛽ 加油任务调度

- **任务生成**：根据航班信息和油品配置自动生成加油任务
- **任务分配**：按区域、按人员、按车辆等多种分配策略
- **任务执行**：实时监控任务执行状态，支持催办和取消
- **任务归档**：完整的任务生命周期管理

### 📝 电子签单系统

- 加油完成后自动生成正式油单记录
- 支持多种油品类型（航空煤油、汽油等）
- 电子签名确认功能
- 油单审核和作废流程
- 支持油单打印和 PDF 导出

### 📱 终端设备管理

- 支持 5500 终端和 PAD 终端两种设备类型
- RMDP 协议和 REST 协议通信
- 设备注册、状态监控、固件升级
- 任务接收、状态上报、消息推送

### 🔄 数据对接

- 东航、南航等航空公司航班系统对接
- 机场行李系统、登机口系统对接
- 总部数据平台同步
- WebService、REST API、消息队列等多种对接方式

---

## 技术架构

### 整体架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           智慧航空加油系统 - 整体架构                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                        展示层 (Presentation Layer)                   │   │
│  │   ┌──────────────────────┐      ┌──────────────────────┐            │   │
│  │   │  Vue 2.x + Element-UI │      │  Angular 15.x + ng-zorro  │            │   │
│  │   │     智慧航空加油前端     │      │      数据共享平台前端       │            │   │
│  │   └──────────────────────┘      └──────────────────────┘            │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                      │                                     │
│                                      ▼                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                        网关层 (Gateway Layer)                       │   │
│  │                   Spring Cloud Gateway / Nginx                        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                      │                                     │
│                                      ▼                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                        服务层 (Service Layer)                         │   │
│  │                                                                       │   │
│  │   ┌─────────────────────┐  ┌─────────────────────┐                   │   │
│  │   │   核心业务服务       │  │    终端管理服务     │                   │   │
│  │   │   flight-oil        │  │   terminalmanager  │                   │   │
│  │   │   flightdispatch    │  │   terminalmanager-oil│                 │   │
│  │   └─────────────────────┘  └─────────────────────┘                   │   │
│  │                                                                       │   │
│  │   ┌─────────────────────┐  ┌─────────────────────┐                   │   │
│  │   │   数据同步服务       │  │    安全认证服务     │                   │   │
│  │   │   synch-data        │  │   security         │                   │   │
│  │   │   pvg-transfer-*    │  │   resourceparent   │                   │   │
│  │   └─────────────────────┘  └─────────────────────┘                   │   │
│  │                                                                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                      │                                     │
│            ┌─────────────────────────┼─────────────────────────┐         │
│            │                         │                         │         │
│            ▼                         ▼                         ▼         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                        接口层 (Interface Layer)                      │   │
│  │   pvg-interfaces / eastern-airline / southern-airline / esb-flight  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                      │                                     │
│                                      ▼                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                        数据存储层 (Data Layer)                        │   │
│  │   ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐       │   │
│  │   │   MySQL   │  │   Redis   │  │  ActiveMQ │  │ Zookeeper │       │   │
│  │   │  db_oil   │  │   缓存    │  │  消息队列  │  │ 服务注册   │       │   │
│  │   └───────────┘  └───────────┘  └───────────┘  └───────────┘       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 技术栈

#### 后端技术栈

| 类别 | 技术 | 版本 | 说明 |
|------|------|------|------|
| 开发语言 | Java | 1.7 / 1.8 | 后端开发语言 |
| 应用框架 | Spring Framework | 3.1.4 / 4.3.3 | 应用框架 |
| 应用框架 | Spring Boot | 1.5.4 / 2.0.8 / 2.7.x | 微服务框架 |
| RPC 框架 | Apache Dubbo | 2.5.3 / 2.5.4-IAS | 分布式服务框架 |
| ORM 框架 | MyBatis | 3.2.3 / 3.4.5 | 数据持久化框架 |
| 数据库连接 | Druid | 1.0.31 / 1.1.0 | 数据库连接池 |
| 消息队列 | Apache ActiveMQ | 5.7.0 / 5.9.1 / 5.12.0 | 消息中间件 |
| 消息路由 | Apache Camel | 2.16.1 / 2.17.0 | 消息路由引擎 |
| 定时任务 | Quartz | 2.2.x | 定时任务调度 |
| 服务注册 | Apache Zookeeper | 3.4.6 / 3.4.14 | 服务注册与发现 |
| 缓存 | Redis + Jedis | 2.4.1+ | 分布式缓存 |
| 安全框架 | Apache Shiro | 1.2.2 | 安全认证授权 |
| JSON 处理 | FastJSON / Jackson | - | JSON 序列化 |
| WebSocket | - | - | 实时通信 |
| 高并发框架 | LMAX Disruptor | 3.3.2 | 高性能队列 |

#### 前端技术栈

| 类别 | 技术 | 版本 | 说明 |
|------|------|------|------|
| 框架 | Vue.js | 2.6.12 | 前端 MVVM 框架 |
| 框架 | Angular | 15.x | 前端框架 |
| 组件库 | Element-UI | 2.15.10 | Vue 组件库 |
| 组件库 | ng-zorro-antd | 15.1.1-cwf | Angular 组件库 |
| 状态管理 | Vuex | 3.6.0 | 状态管理 |
| HTTP 客户端 | Axios | 0.24.0 | HTTP 客户端 |
| 表格组件 | ag-grid | 25.1.0 | 高性能表格 |
| 图表 | ECharts | 5.4.0 | 图表库 |

#### 数据库

| 数据库 | 版本 | 用途 |
|--------|------|------|
| MySQL | 5.1.26 / 5.7 | 业务数据存储 |
| Redis | - | 缓存、会话、分布式锁 |
| ActiveMQ | 5.x | 异步消息 |
| H2 | 1.3.173 | 单元测试 |

---

## 项目结构

```
pudong/
├── agdis-oil-web/                 # 核心Web应用模块（483个Java文件）
│   ├── agdis-bean/               # 实体类定义
│   ├── agdis-core/               # 核心业务逻辑
│   ├── agdis-controller/          # HTTP控制器
│   ├── agdis-dubbo/              # Dubbo服务
│   ├── agdis-redis/              # Redis缓存
│   ├── agdis-boot/               # Camel消息路由
│   ├── agdis-websocket/          # WebSocket通信
│   └── agdis-web-oil/            # Web应用打包
│
├── flight-oil/                   # 加油业务核心模块
│   ├── flight-oil-master/       # 主服务（70个Java文件）
│   └── flight-oil-service/      # 服务接口
│
├── flightdispatch/               # 航班调度模块（117个Java文件）
├── flightdealui/                 # 签单UI模块（51个Java文件）
│
├── terminalmanager/               # 终端管理模块
│   ├── terminal-core/           # 核心逻辑
│   ├── terminal-rest/           # REST服务
│   ├── terminal-rmdp/           # RMDP协议
│   └── messagedispatch/         # 消息分发
│
├── terminalmanager-oil/          # 油单终端模块
│
├── data-sync/
│   ├── synch-data/              # 同步到总部
│   ├── pvg-transfer-oilbill-mdb/   # 同步到MDB
│   └── pvg-transfer-bill-mdb-dmp/  # 同步到DMP
│
├── security/
│   ├── security/                # 安全认证模块
│   ├── resourceparent/          # 资源父模块
│   └── resourcesub/             # 资源子模块
│
├── interfaces/
│   ├── pvg-interfaces/          # 接口聚合模块
│   ├── eastern-airline-pvg/     # 东航接口
│   ├── southern-airline-http-amq-pvg-new/  # 南航接口
│   └── esb-flightinfo-pvg/      # ESB接口
│
├── oil-pvg-data-share/          # 数据共享平台（Angular 15.x）
│
├── code-doc/                         # 项目文档
│   ├── readme.md               # 项目说明文档
│   ├── code/                   # 代码分析文档
│   └── bug/                    # Bug分析文档
│
└── sql/                         # SQL脚本
```

---

## 快速开始

### 环境要求

- JDK 1.8+
- MySQL 5.7+
- Redis 3.2+
- Apache Zookeeper 3.4.6+
- Apache ActiveMQ 5.9+
- Maven 3.6+
- Node.js 14+（前端开发）

### 后端启动

```bash
# 1. 克隆项目
git clone https://your-repo/smart-aviation-fuel-system.git
cd smart-aviation-fuel-system

# 2. 安装依赖
mvn clean install -DskipTests

# 3. 配置数据库
# 修改各模块的 application.yml 配置数据库连接

# 4. 启动顺序
# 1) 启动 Zookeeper
# 2) 启动 ActiveMQ
# 3) 启动 MySQL Redis
# 4) 启动各服务模块（按依赖顺序）
mvn spring-boot:run -pl flight-oil/flight-oil-master
mvn spring-boot:run -pl flightdispatch
mvn spring-boot:run -pl terminalmanager
```

### 前端启动

```bash
# 智慧航空加油系统前端（Vue）
cd agdis-oil-web/agdis-web-oil
npm install
npm run dev

# 数据共享平台前端（Angular）
cd oil-pvg-data-share/pudong-web-angular
npm install
ng serve
```

---

## 模块说明

### 核心业务模块

| 模块 | 说明 | 核心类 |
|------|------|--------|
| `flight-oil` | 加油任务管理 | OilBillService, OilTaskService |
| `flightdispatch` | 航班调度指挥 | FlightDispatchController, TaskDispatchService |
| `flightdealui` | 签单Web界面 | OilBillController |
| `agdis-oil-web` | 核心Web应用 | 多子模块组成 |

### 终端管理模块

| 模块 | 说明 | 协议 |
|------|------|------|
| `terminalmanager` | 5500/PAD终端通信 | RMDP, REST |
| `terminalmanager-oil` | 油单终端适配 | RMDP, REST |
| `terminal-dispatch` | 消息分发路由 | ActiveMQ, Camel |

### 数据同步模块

| 模块 | 说明 | 目标 |
|------|------|------|
| `synch-data` | 同步到总部平台 | HTTP REST |
| `pvg-transfer-oilbill-mdb` | 同步到MDB数据库 | JDBC |
| `pvg-transfer-bill-mdb-dmp` | 同步到DMP平台 | JDBC |

### 安全与资源模块

| 模块 | 说明 | 核心功能 |
|------|------|----------|
| `security` | 安全认证授权 | Shiro, RBAC |
| `resourceparent` | 资源管理 | 用户、资源、权限 |
| `vdinfo` | 车辆绑定 | 设备-车辆关联 |

---

## API 文档

### 核心 API 端点

| 服务 | 端口 | API 路径 | 说明 |
|------|------|----------|------|
| ess-auth | 8080 | /api/auth/** | 认证服务 |
| ess-modules | 8080 | /api/** | 业务服务 |
| flight-oil | 808x | /api/oil/** | 加油业务 |
| terminalmanager | 808x | /api/terminal/** | 终端管理 |
| resourcesub | 808x | /api/resource/** | 资源API |

### 主要 REST API

```bash
# 航班相关
GET    /api/flight/info/{flightNo}     # 获取航班信息
GET    /api/flight/dynamic/{flightId}  # 获取航班动态

# 任务相关
GET    /api/task/list                   # 任务列表
POST   /api/task/create                # 创建任务
PUT    /api/task/{taskId}/dispatch     # 派发任务
PUT    /api/task/{taskId}/status       # 更新状态

# 油单相关
GET    /api/oilbill/list               # 油单列表
GET    /api/oilbill/{billId}          # 油单详情
POST   /api/oilbill/{billId}/sign     # 油单签名
PUT    /api/oilbill/{billId}/audit    # 油单审核

# 终端相关
GET    /api/terminal/status            # 终端状态
POST   /api/terminal/message           # 推送消息
```

### Dubbo 服务接口

```java
// 油单服务
public interface OilBillService {
    OilBillEntity getById(Long id);
    List<OilBillEntity> list(OilBillQuery query);
    void create(OilBillEntity bill);
    void update(OilBillEntity bill);
    void sign(Long billId, String signature);
}

// 任务服务
public interface OilTaskService {
    List<OilTaskEntity> list(TaskQuery query);
    void createTask(OilTaskEntity task);
    void dispatch(Long taskId, Long userId, Long vehicleId);
    void updateStatus(Long taskId, Integer status);
}

// 航班服务
public interface FlightInfoService {
    FlightInfoEntity getByFlightNo(String flightNo);
    List<FlightInfoEntity> list(FlightQuery query);
    void updateDynamic(FlightDynamicEntity dynamic);
}
```

---

## 部署指南

### 部署拓扑

```
                         ┌─────────────────┐
                         │   Nginx 集群    │
                         │   (负载均衡)     │
                         └────────┬────────┘
                                  │
        ┌─────────────────────────┼─────────────────────────┐
        │                         │                         │
        ▼                         ▼                         ▼
┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐
│  flight-oil     │   │ terminalmanager │   │   ess-* 服务     │
│  集群 (3节点)    │   │  集群 (3节点)   │   │   集群          │
└─────────────────┘   └─────────────────┘   └─────────────────┘
        │                         │                         │
        └─────────────────────────┼─────────────────────────┘
                                  │
           ┌──────────────────────┼──────────────────────┐
           │                      │                      │
           ▼                      ▼                      ▼
   ┌───────────────┐      ┌───────────────┐      ┌───────────────┐
   │   Zookeeper   │      │   ActiveMQ    │      │    Redis      │
   │   集群 (3节点) │      │   集群         │      │   主从集群     │
   └───────────────┘      └───────────────┘      └───────────────┘
                                  │
                                  ▼
                         ┌─────────────────┐
                         │  MySQL 主从集群   │
                         └─────────────────┘
```

### 端口配置

| 服务 | 默认端口 | 协议 | 说明 |
|------|----------|------|------|
| HTTP 服务 | 8080 | TCP | ess-* 服务 |
| Dubbo 服务 | 20880 | TCP | Dubbo 协议 |
| Zookeeper | 2181 | TCP | 服务注册 |
| ActiveMQ | 61616 | TCP | 消息队列 |
| MySQL | 3306 | TCP | 数据库 |
| Redis | 6379 | TCP | 缓存 |

### Docker 部署

```bash
# 构建各模块镜像
cd flight-oil/flight-oil-master
docker build -t flight-oil:latest .

# 运行容器
docker run -d \
  --name flight-oil \
  -p 8080:8080 \
  -e SPRING_PROFILES_ACTIVE=prod \
  flight-oil:latest
```

---

## 配置说明

### 数据库配置

```yaml
# application.yml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/db_oil_pvg?useUnicode=true&characterEncoding=utf8
    username: root
    password: ${DB_PASSWORD}
    driver-class-name: com.mysql.jdbc.Driver
    type: com.alibaba.druid.pool.DruidDataSource
    druid:
      initial-size: 5
      min-idle: 5
      max-active: 20
```

### Redis 配置

```yaml
spring:
  redis:
    host: localhost
    port: 6379
    password: ${REDIS_PASSWORD}
    database: 0
    jedis:
      pool:
        max-active: 20
        max-idle: 10
        min-idle: 5
```

### Dubbo 配置

```yaml
dubbo:
  application:
    name: flight-oil-service
  registry:
    address: zookeeper://localhost:2181
  protocol:
    name: dubbo
    port: 20880
```

### ActiveMQ 配置

```yaml
spring:
  activemq:
    broker-url: tcp://localhost:61616
    user: admin
    password: ${ACTIVEMQ_PASSWORD}
  jms:
    pub-sub-domain: false
```

---

## 常见问题

### Q1: 启动顺序是什么？

**A:** 正确的启动顺序：
1. Zookeeper
2. ActiveMQ
3. MySQL + Redis
4. globalsequence（全局序列）
5. messagemanager（消息管理）
6. security + resourceparent（安全模块）
7. flight-oil（核心业务）
8. flightdispatch（调度模块）
9. terminalmanager（终端管理）

### Q2: 如何添加新的终端类型？

**A:** 
1. 在 `terminal-dispatch` 中添加新的路由配置
2. 创建对应的消息处理器
3. 配置新的 ActiveMQ 队列
4. 在 `terminalmanager` 中添加终端适配逻辑

### Q3: 如何对接新的航空公司？

**A:**
1. 在 `pvg-interfaces` 中创建新的子模块
2. 实现航班数据接收逻辑
3. 配置数据标准化转换规则
4. 添加对应的 Listener 监听器

### Q4: 数据库连接池配置建议

**A:** 推荐配置：
```yaml
druid:
  initial-size: 10
  min-idle: 10
  max-active: 50
  max-wait: 60000
  validation-query: SELECT 1
```

---

## 技术债务

### ⚠️ 已知问题

| 问题 | 严重程度 | 影响 | 建议 |
|------|----------|------|------|
| Java 1.7 停止支持 | 🔴 高 | 安全漏洞 | 升级至 Java 11+ |
| Spring Framework 3.x/4.x 停止维护 | 🔴 高 | 安全风险 | 统一升级至 5.3.x |
| Apache Dubbo 2.5.x 停止维护 | 🔴 高 | 功能受限 | 升级至 3.x |
| SQL 注入风险 | 🔴 高 | 安全漏洞 | `${}` 改为 `#{}` |
| 明文密码存储 | 🟡 中 | 安全风险 | 改为加密存储 |
| 技术栈版本不统一 | 🟡 中 | 维护复杂 | 统一版本 |

### 升级路线图

**短期（1-3个月）**
- [ ] 修复 SQL 注入漏洞
- [ ] 修改明文密码存储
- [ ] 统一 Spring 版本

**中期（3-6个月）**
- [ ] 制定技术栈升级路线图
- [ ] 引入配置中心
- [ ] 完善监控告警体系

**长期（6-12个月）**
- [ ] 微服务化改造
- [ ] 引入 Service Mesh
- [ ] 建立 DevOps 流水线

---

## 更新日志

### v3.1 (2026-02-04)
- ✨ 整合 agdis-oil-web 模块
- ✨ 新增 28 个业务模块文档
- 📝 更新技术栈版本
- 🐛 修复已知问题

### v3.0 (2025-12-01)
- ✨ 新增数据共享平台文档
- ✨ 更新架构图
- 📝 完善 API 文档

### v2.5 (2025-09-15)
- ✨ 新增终端管理模块文档
- ✨ 新增安全模块文档
- 📝 完善部署指南

---

## 许可证

本项目仅供内部使用，禁止对外传播。

---

## 贡献者

- 开发团队：上海浦东机场技术部

---

## 联系方式

- 项目维护者：开发团队
- 问题反馈：请提交 Issue

---

<p align="center">
  <sub>Made with ❤️ by Smart Aviation Fuel System Team</sub>
</p>
