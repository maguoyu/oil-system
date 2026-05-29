# 浦东航油数据共享平台（oil-pvg-data-share）代码分析报告

> 分析日期：2026-02-04
> 项目版本：2.0.0（后端）
> 前端版本：Angular 15

---

## 1. 项目概述

### 1.1 项目定位

浦东航油数据共享平台是一个**企业级航空燃油信息管理系统**，提供航班信息管理、客户管理、加油任务跟踪、发票管理、SAP系统集成等完整的业务功能。

### 1.2 项目架构

```
oil-pvg-data-share/                      # 数据共享平台父目录
├── base-framework-test/                 # 后端Spring Boot框架
│   ├── CCF-COMMON/                     # 公共模块
│   ├── CCF-BASECODE/                  # 基础业务模块
│   ├── CCF-SYS/                       # 系统模块
│   └── CCF-CERTIFICATION/             # 认证模块
├── pudong-web-angular/                 # 前端Angular应用（浦東）
└── web-angualr/                        # 前端Angular应用
```

---

## 2. 后端架构分析（Spring Boot）

### 2.1 技术栈

| 类别 | 技术 | 版本 | 说明 |
|------|------|------|------|
| 框架 | Spring Boot | 2.7.14 | 应用框架 |
| 云原生 | Spring Cloud | 2021.0.8 | 微服务框架 |
| 云原生 | Spring Cloud Alibaba | 2021.0.5.0 | 阿里云组件 |
| 注册中心 | Nacos | - | 服务发现/配置中心 |
| ORM | MyBatis | 2.3.1 | 数据持久化 |
| 数据库 | MySQL | 8.0.31 | 主数据库 |
| 数据库 | Oracle | 12.2.0.1 | 业务数据库 |
| 连接池 | Druid | 1.2.19 | 数据库连接池 |
| 消息队列 | Kafka | - | 消息队列 |
| 定时任务 | XXL-JOB | 2.0.2 | 定时调度 |
| 工作流 | Activiti | 5.22.0 | 流程引擎 |
| 文档 | Swagger | 1.6.14 | API文档 |
| 日志 | Logback | - | 日志框架 |
| 缓存 | Caffeine | - | 本地缓存 |
| 缓存 | Redis | - | 分布式缓存 |
| 链路追踪 | Zipkin | - | 分布式追踪 |

### 2.2 模块划分

```
┌─────────────────────────────────────────────────────────────────┐
│                   base-framework-test 后端架构                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                    CCF-CERTIFICATION                       │  │
│  │  认证授权模块                                               │  │
│  │  - 用户认证                                                 │  │
│  │  - 权限管理                                                 │  │
│  │  - Token管理                                               │  │
│  └───────────────────────────────────────────────────────────┘  │
│                              ↑                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                      CCF-SYS                             │  │
│  │  系统管理模块                                               │  │
│  │  - 用户管理                                                 │  │
│  │  - 角色管理                                                 │  │
│  │  - 模块管理                                                 │  │
│  │  - 权限配置                                                 │  │
│  └───────────────────────────────────────────────────────────┘  │
│                              ↑                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                      CCF-BASECODE                        │  │
│  │  基础业务模块（核心）                                       │  │
│  │  ┌───────────────────────────────────────────────────┐  │  │
│  │  │  controller/           - 控制器层                   │  │  │
│  │  │  service/             - 业务服务层                 │  │  │
│  │  │  mo/                  - 消息对象                   │  │  │
│  │  │  persist/             - 持久化层                   │  │  │
│  │  │  util/                - 工具类                    │  │  │
│  │  │  edi/                 - EDI数据交换                │  │  │
│  │  └───────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────┘  │
│                              ↑                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                      CCF-COMMON                          │  │
│  │  公共基础模块                                               │  │
│  │  - 公共常量                                                 │  │
│  │  - 公共工具                                                 │  │
│  │  - 消息模板                                                 │  │
│  │  - 基础服务                                                 │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.3 核心服务清单

#### 2.3.1 基础代码服务（basecode）

| 服务 | 功能 | 主要表 |
|------|------|--------|
| CustomService | 客户信息管理 | T_SPA_CUSTOMER_INFORMATION_HEADER |
| CustomerGroupService | 客户组管理 | T_BAS_CUSTOMER_GROUP |
| PriceService | 价格管理 | - |
| InvoiceService | 发票管理 | - |
| BankbillService | 银行票据 | - |
| AdvanceService | 预付款管理 | - |
| EC | 油任务单服务 | T_SPA_ECC_OIL_TASKSHEET |
| PumpfuelService | 泵油服务 | - |
| TaskEnsureOilNodeService | 加油节点任务 | T_TASK_ENSURE_OIL_NODE |
| PublicQueryService | 公共查询 | - |
| FileFromFTPService | FTP文件服务 | - |

#### 2.3.2 任务服务（task）

| 服务 | 功能 | 说明 |
|------|------|------|
| TaskService | 加油任务管理 | 任务增删改查 |
| RefuelingTaskService | 加油任务 | 加油流程管理 |
| ReceiveTaskService | 任务接收 | 接收外部任务 |

#### 2.3.3 数据集成服务（dataIntegration）

| 服务 | 功能 | 集成系统 |
|------|------|----------|
| SAP系统集成 | | |
| SalesOrderService | 销售订单 | SAP |
| PurchaseOrderService | 采购订单 | SAP |
| InvoiceBillingInformationService | 发票账单 | SAP |
| CustomerInformationService | 客户信息 | SAP |
| PriceInformationService | 价格信息 | SAP |
| InventoryService | 库存管理 | SAP |
| StatementOfAccountService | 对账单 | SAP |
| ARBalanceService | 应收账款 | SAP |
| AuditInformationService | 审计信息 | SAP |
| 统计分析 | | |
| DayFuelChargeStatisticService | 日加油统计 | - |
| SingleFlightFuelChargeStatisticsService | 单班加油统计 | - |
| UndergroundWellFuelChargeStatisticsService | 地下井加油统计 | - |
| AirlineRefuelingStatisticsService | 航空公司加油统计 | - |
| FuellingTruckStatisticsService | 加油车统计 | - |
| FuelFillerStatisticsService | 加油员统计 | - |

#### 2.3.4 其他服务

| 服务 | 功能 |
|------|------|
| FlightInfoService | 航班信息服务 |
| InvoiceService | 发票服务 |
| EmailConfigService | 邮件配置 |
| PushConfigService | 推送配置 |
| PullConfigService | 拉取配置 |
| ReceiveConfigService | 接收配置 |
| PageConfigService | 页面配置 |
| DataMaintenanceService | 数据维护 |

### 2.4 核心业务代码示例

#### 客户查询服务

```java
@CCFService("Basecode.Cust.CustomerService")
public class CustomService extends BaseService {
    
    public IMessageObject query(IMessageObject mo) {
        StringBuilder sql = new StringBuilder();
        List<Object> args = new ArrayList<>();
        IRow condition = this.getConditionByReqMo(mo);
        
        sql.append(" SELECT TT1.* FROM ( ");
        sql.append(" SELECT T2.CUSTOMER_GROUP_NAME AS KTOKD_CN, T1.* FROM T_SPA_CUSTOMER_INFORMATION_HEADER AS T1, T_BAS_CUSTOMER_GROUP AS T2 ");
        sql.append("WHERE 1 = 1 AND T1.DELETE_FLG = 'N' AND T2.DELETE_FLG = 'N' AND T1.KTOKD = T2.CUSTOMER_GROUP_CODE  ");
        // 条件拼接
        if (!ObjectHelper.isEmpty(condition)) {
            addConByCondition(condition, sql, args);
        }
        
        sql.append(" ) AS TT1 ");
        sql.append(" ORDER BY ");
        // 排序逻辑...
        
        IPagination page = this.paginationUtil.pagination(mo);
        List<IRow> list = this.getBaseModule().queryBySQL(sql.toString(), args.toArray(), page);
        this.paginationUtil.setPageCount(mo, page);
        mo.setRowsOfResp("T_SPA_CUSTOMER_INFORMATION_HEADER", list);
        return this.moTemplate.serviceSuccessfulEnd(mo);
    }
}
```

#### 加油任务服务

```java
@CCFService("ccf.service.task.TaskService")
public class TaskService extends BaseService {
    
    public IMessageObject query(IMessageObject mo) throws Throwable {
        IRow cond = this.getConditionByReqMo(mo);
        StringBuffer sql = new StringBuffer();
        sql.append("SELECT * FROM T_TASK_ENSURE_OIL_NODE WHERE DELETE_FLG='N' ");
        
        // 条件拼接
        if (!ObjectHelper.isEmpty(cond.getValue("TNB"))) {
            sql.append(" AND TNB = '" + cond.getValue("TNB") + "'");
        }
        // ...更多条件
        
        sql.append(" ORDER BY MADE_DT DESC ");
        IPagination pag = this.paginationUtil.pagination(mo);
        List<IRow> orgList = baseModule.queryBySQL(sql.toString(), null, pag);
        mo.setRowsOfResp("T_TASK_ENSURE_OIL_NODE", orgList);
        this.paginationUtil.setPageCount(mo, pag);
        return mo;
    }
}
```

### 2.5 控制器层设计

采用统一的网关控制器模式：

```java
@Controller
public class CustomController extends GatewayServiceBaseController {
    
    @Autowired
    @Qualifier("JSON_RCVER")
    protected IEdiRcver jsonClientEdiRcver;
    
    @Operation(summary = "query")
    @RequestMapping(
        method = {RequestMethod.POST},
        value = {"/Basecode.Cust.CustomerService/query"},
        consumes = {SystemConstants.CONTENT_TYPE.CONTENT_TYPE_JSON},
        produces = {SystemConstants.CONTENT_TYPE.CONTENT_TYPE_JSON}
    )
    public void query(HttpServletRequest request, HttpServletResponse response, @RequestBody byte[] reqBytes) {
        this.gatewaybase(request, response, reqBytes, jsonClientEdiRcver);
    }
}
```

---

## 3. 前端架构分析（Angular）

### 3.1 技术栈

| 类别 | 技术 | 版本 | 说明 |
|------|------|------|------|
| 框架 | Angular | 15.2.9 | 前端框架 |
| UI库 | NG-ZORRO | 15.1.1 | Ant Design Angular版 |
| 图表 | ECharts | 5.5.1 | 数据可视化 |
| 富文本 | TinyMCE | 13.0.0 | 富文本编辑器 |
| PDF | jsPDF | 2.4.0 | PDF生成 |
| Excel | xlsx | 0.15.6 | Excel处理 |
| HTTP | @angular/common | - | HTTP客户端 |
| 状态管理 | RxJS | 7.5.5 | 响应式编程 |
| 国际化 | @ngx-translate | 13.0.0 | 国际化 |
| 打包 | Angular CLI | 15.2.9 | 构建工具 |
| 混淆 | JavaScript Obfuscator | 4.0.2 | 代码混淆 |

### 3.2 项目结构

```
pudong-web-angular/                      # 浦东前端应用
├── src/
│   ├── app/
│   │   ├── core/                       # 核心模块
│   │   │   ├── core.module.ts          # 核心模块定义
│   │   │   ├── shared-zorro.module.ts  # Zorro共享模块
│   │   │   ├── shared-delon.module.ts  # Delon共享模块
│   │   │   └── ...
│   │   ├── layout/                     # 布局模块
│   │   │   ├── header/                # 头部组件
│   │   │   ├── sidebar/               # 侧边栏组件
│   │   │   └── layout.module.ts       # 布局模块
│   │   ├── business/                   # 业务模块
│   │   │   ├── basecode/              # 基础数据模块
│   │   │   │   ├── customer/          # 客户管理
│   │   │   │   │   ├── custom/        # 客户信息
│   │   │   │   │   ├── customerGroup/ # 客户组
│   │   │   │   │   └── taskEnsureOilNode/ # 加油任务
│   │   │   │   ├── inquiry/           # 查询模块
│   │   │   │   │   ├── bill/          # 账单查询
│   │   │   │   │   ├── invoice/       # 发票查询
│   │   │   │   │   ├── balance/       # 余额查询
│   │   │   │   │   └── ...
│   │   │   │   └── statistic/          # 统计模块
│   │   │   ├── layoutQueryPage/       # 布局查询页
│   │   │   │   ├── HomePage/          # 首页
│   │   │   │   ├── Ensureprogress/     # 进度查询
│   │   │   │   ├── InvoiceinQuiry/    # 发票查询
│   │   │   │   └── ...
│   │   │   ├── approvalcenter/         # 审批中心
│   │   │   └── ...
│   │   ├── business-modal/             # 业务弹窗模块
│   │   ├── pipe/                      # 管道
│   │   ├── service/                   # 服务
│   │   └── store/                     # 状态管理
│   ├── assets/                         # 静态资源
│   │   ├── i18n/                      # 国际化资源
│   │   ├── images/                    # 图片
│   │   └── pdf/                       # PDF模板
│   ├── environments/                   # 环境配置
│   └── styles/                        # 样式文件
```

### 3.3 功能模块清单

#### 3.3.1 基础数据管理（basecode）

| 模块 | 功能 |
|------|------|
| 客户管理 | |
| custom | 客户信息维护 |
| customerGroup | 客户组管理 |
| taskEnsureOilNode | 加油节点 |
| fuelcustomprice | 客户燃油价格 |
| fuelprice | 燃油价格 |
| pumpfuel | 泵油信息 |
| 查询模块 | |
| bill | 账单查询 |
| invoice | 发票查询 |
| balance | 余额查询 |
| accountStatementQuery | 对账单查询 |
| price | 价格查询 |
| oilList | 油单查询 |
| 统计模块 | |
| day | 日统计 |
| airlineCompany | 航司统计 |
| tanker | 油车统计 |
| summary | 汇总统计 |
| topUpBoy | 加油员统计 |
| manhole | 井口统计 |

#### 3.3.2 布局查询页（layoutQueryPage）

| 模块 | 功能 |
|------|------|
| HomePage | 首页仪表盘 |
| Ensureprogress | 保障进度查询 |
| InvoiceinQuiry | 发票查询 |
| Billinquiry | 账单查询 |
| Priceinquiry | 价格查询 |
| Oilinquiry | 油单查询 |
| Contractinquiry | 合同查询 |
| Balanceinquiry | 余额查询 |
| Reconciliationinquiry | 对账查询 |
| OilSummary | 油量汇总 |

### 3.4 前端核心代码示例

#### 主模块配置

```typescript
// app.module.ts
import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { BrowserAnimationsModule } from '@angular/platform-browser/animations';
import { SharedModule } from './shared/shared.module';
import { CoreModule } from './core/core.module';
import { LayoutModule } from './layout/layout.module';
import { BusinessModule } from './business/business.module';

@NgModule({
  imports: [
    BrowserModule,
    BrowserAnimationsModule,
    SharedModule,
    CoreModule,
    LayoutModule,
    BusinessModule
  ],
  bootstrap: [AppComponent]
})
export class AppModule { }
```

#### 业务服务

```typescript
// customer.service.ts
@Injectable()
export class CustomerService {
  constructor(private http: HttpClient) {}
  
  query(params: any): Observable<any> {
    return this.http.post('/Basecode.Cust.CustomerService/query', params);
  }
  
  save(data: any): Observable<any> {
    return this.http.post('/Basecode.Cust.CustomerService/save', data);
  }
}
```

---

## 4. 配置文件分析

### 4.1 后端配置（bootstrap.yml）

```yaml
spring:
  profiles:
    active: prod    # 云平台配置（生产）
    # active: local   # 本地开发环境
    # active: test    # 云平台配置（测试）
```

### 4.2 数据库配置

**MySQL**:
```properties
jdbc.driverClassName=com.mysql.jdbc.Driver
jdbc.url=jdbc:mysql://172.21.0.19:3306/mis_mutil?useUnicode=true&characterEncoding=UTF-8
jdbc.username=root
jdbc.password=airport
```

**Oracle**:
```properties
# CCF-SYS使用Oracle数据库
```

### 4.3 Nacos配置

```yaml
# 服务发现
spring.cloud.nacos.discovery.server-addr=127.0.0.1:8848

# 配置中心
spring.cloud.nacos.config.server-addr=127.0.0.1:8848
spring.cloud.nacos.config.prefix=base-framework
spring.cloud.nacos.config.file-extension=yaml
```

### 4.4 日志配置

使用Logback + Logstash集成：

```xml
<appender name="LOGSTASH" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
    <destination>logstash-ip:5044</destination>
    <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
</appender>
```

---

## 5. 业务流程分析

### 5.1 加油任务流程

```
┌─────────────────────────────────────────────────────────────────┐
│                    加油任务业务流程                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────┐    ┌─────────────┐    ┌─────────────┐           │
│  │ 任务创建 │───►│  任务接收    │───►│  任务确认   │           │
│  └─────────┘    └─────────────┘    └─────────────┘           │
│       │                                    │                   │
│       ▼                                    ▼                   │
│  ┌─────────┐    ┌─────────────┐    ┌─────────────┐           │
│  │ SAP同步  │    │  进度更新    │    │  完成状态   │           │
│  └─────────┘    └─────────────┘    └─────────────┘           │
│       │                                    │                   │
│       ▼                                    ▼                   │
│  ┌───────────────────────────────────────────────────────┐     │
│  │                   状态流转                            │     │
│  │  U(新接收) → S(成功) / F(失败)                       │     │
│  └───────────────────────────────────────────────────────┘     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 客户数据同步

```
┌─────────────────────────────────────────────────────────────────┐
│                    客户数据同步流程                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────┐    ┌─────────────┐    ┌─────────────┐           │
│  │ 数据采集 │───►│  格式转换    │───►│  数据验证   │           │
│  └─────────┘    └─────────────┘    └─────────────┘           │
│                                            │                   │
│                                            ▼                   │
│                                      ┌─────────────┐          │
│                                      │  数据存储    │          │
│                                      │ (MySQL)     │          │
│                                      └─────────────┘          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6. 代码质量分析

### 6.1 优点

1. **架构清晰**：分层明确，Controller → Service → DAO
2. **技术栈现代**：Spring Boot 2.7 + Angular 15
3. **云原生支持**：Nacos服务发现、配置中心
4. **完善的可观测性**：Zipkin链路追踪、Logback日志
5. **定时任务支持**：XXL-JOB
6. **前端工程化**：完整的Angular工程配置

### 6.2 问题与建议

#### 🔴 高优先级

| 问题 | 位置 | 建议 |
|------|------|------|
| SQL注入风险 | TaskService | 使用参数化查询 |
| 硬编码SQL | 多个Service | 移至Mapper XML |
| 缺少统一异常处理 | 全局 | 实现全局异常处理器 |
| 密码明文存储 | 配置 | 使用Nacos密文配置 |

#### 🟡 中优先级

| 问题 | 位置 | 建议 |
|------|------|------|
| 缺少单元测试 | - | 增加测试覆盖率 |
| 代码重复 | 多个Service | 提取公共方法 |
| 缺少API文档 | Controller | 使用Swagger注解 |
| 日志过多 | Service | 优化日志级别 |

#### 🟢 低优先级

| 问题 | 位置 | 建议 |
|------|------|------|
| 魔法值 | 代码中 | 定义常量 |
| 方法过长 | TaskService | 拆分方法 |
| 缺少注释 | 部分代码 | 增加Javadoc |

---

## 7. 部署架构

### 7.1 Docker部署

后端支持Docker容器化部署：

```dockerfile
# Dockerfile (CCF-BASECODE)
FROM 172.16.11.31:5000/ias/os-jvm:centos6-jdk8
# ... 部署配置
```

### 7.2 服务拓扑

```
┌─────────────────────────────────────────────────────────────────┐
│                     系统部署拓扑                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │                    前端服务                              │  │
│   │   pudong-web-angular (4200)                            │  │
│   └─────────────────────────────────────────────────────────┘  │
│                              │                                  │
│                              ▼                                  │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │                   API网关 / Nginx                       │  │
│   └─────────────────────────────────────────────────────────┘  │
│                              │                                  │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │               Spring Boot 后端服务                       │  │
│   │  ┌─────────────────────────────────────────────────┐   │  │
│   │  │  CCF-BASECODE (808x)                          │   │  │
│   │  │  CCF-SYS (808x)                               │   │  │
│   │  └─────────────────────────────────────────────────┘   │  │
│   └─────────────────────────────────────────────────────────┘  │
│                              │                                  │
│          ┌───────────────────┼───────────────────┐             │
│          ▼                   ▼                   ▼             │
│   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐       │
│   │   Nacos     │    │    MySQL    │    │  Oracle    │       │
│   │  :8848     │    │   :3306     │    │   :1521    │       │
│   └─────────────┘    └─────────────┘    └─────────────┘       │
│          │                   │                                  │
│          ▼                   ▼                                  │
│   ┌─────────────┐    ┌─────────────┐                          │
│   │    Kafka    │    │   Redis    │                          │
│   │   :9092    │    │   :6379    │                          │
│   └─────────────┘    └─────────────┘                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 8. 总结

### 8.1 项目定位

浦东航油数据共享平台是一个**完整的B/S架构企业级应用**，涵盖：

- ✅ 客户信息管理
- ✅ 加油任务跟踪
- ✅ 发票账单管理
- ✅ SAP系统集成
- ✅ 数据统计分析
- ✅ 完整的Web管理界面

### 8.2 技术特点

| 特点 | 说明 | 状态 |
|------|------|------|
| 前后端分离 | Angular + Spring Boot | ✅ |
| 微服务架构 | Spring Cloud Alibaba | ✅ |
| 容器化部署 | Docker支持 | ✅ |
| 服务发现 | Nacos | ✅ |
| 配置中心 | Nacos | ✅ |
| 链路追踪 | Zipkin | ✅ |
| 消息队列 | Kafka | ✅ |
| 定时任务 | XXL-JOB | ✅ |

### 8.3 优化建议

1. **短期优化**：
   - 修复SQL注入风险
   - 优化日志输出
   - 增加单元测试

2. **中期重构**：
   - 统一API响应格式
   - 实现全局异常处理
   - 完善Swagger文档

3. **长期升级**：
   - 微服务化改造
   - 前端性能优化
   - 监控告警完善

---

*报告生成时间：2026-02-04*
*分析工具：Cursor AI Code Analysis*
