# ESS (电子签单系统) 详细分析报告

> 分析时间: 2026-02-03  
> 分析范围: ess/ (后端 + 前端)  
> 用途: 电子签单系统功能分析、Bug排查、二次开发参考

---

## 1. 项目概述

### 1.1 系统基本信息

| 属性 | 值 |
|------|-----|
| 系统名称 | 浦东航油电子签单系统 |
| 版本 | 2.0.3.2-SNAPSHOT |
| 前端版本 | 2.1.2 |
| 架构类型 | 微服务架构 (Spring Cloud + Spring Boot) |
| 许可证 | MIT |

### 1.2 技术栈概览

#### 后端技术栈

| 技术 | 版本 | 用途 |
|------|------|------|
| Spring Boot | 2.7.7 | 基础框架 |
| Spring Cloud | 2021.0.5 | 微服务框架 |
| Spring Cloud Alibaba | 2021.0.4.0 | 阿里云组件 |
| MyBatis-Plus | - | ORM框架 |
| Druid | 1.2.15 | 数据库连接池 |
| Redis | - | 缓存/会话存储 |
| JWT | 0.9.1 | Token认证 |
| Swagger | 3.0.0 | API文档 |
| MinIO | 8.2.2 | 对象存储 |
| Apache POI | 4.1.2 | Excel处理 |
| ActiveMQ | 5.14.5 | 消息队列 |
| Forest | 1.5.30 | HTTP客户端 |

#### 前端技术栈

| 技术 | 版本 | 用途 |
|------|------|------|
| Vue | 2.6.12 | 核心框架 |
| Element-UI | 2.15.10 | UI组件库 |
| Vuex | 3.6.0 | 状态管理 |
| Vue-Router | 3.5.1 | 路由管理 |
| Axios | 0.24.0 | HTTP客户端 |
| ag-Grid | 25.1.0 | 数据表格 |
| ECharts | 5.4.0 | 图表库 |
| NProgress | 0.2.0 | 进度条 |
| jsencrypt | 3.0.0 | RSA加密 |

---

## 2. 系统架构

### 2.1 微服务架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              ESS 系统架构                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         用户访问层                                    │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐           │   │
│  │  │ Web浏览器 │  │ 移动APP  │  │ 微信小程序│  │ 第三方系统│           │   │
│  │  └─────┬────┘  └─────┬────┘  └─────┬────┘  └─────┬────┘           │   │
│  └────────┼─────────────┼─────────────┼─────────────┼─────────────────┘   │
│           │             │             │             │                      │
│           └─────────────┴─────────────┴─────────────┘                      │
│                                    │                                        │
│                                    ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                       Nginx 负载均衡                                 │   │
│  │  ┌─────────────────┐  ┌─────────────────┐                           │   │
│  │  │   前端静态资源   │  │   反向代理      │                           │   │
│  │  └─────────────────┘  └────────┬────────┘                           │   │
│  └───────────────────────────────┼─────────────────────────────────────┘   │
│                                  │                                          │
│  ┌──────────────────────────────┼───────────────────────────────────────┐  │
│  │                              ▼                                        │  │
│  │  ┌────────────────────────────────────────────────────────────────┐  │  │
│  │  │                     API Gateway (网关)                          │  │  │
│  │  │  ┌──────────────────────────────────────────────────────────┐  │  │  │
│  │  │  │ • 路由转发  • 限流熔断  • 认证鉴权  • 请求日志            │  │  │  │
│  │  │  └──────────────────────────────────────────────────────────┘  │  │  │
│  │  └────────────────────────────────────────────────────────────────┘  │  │
│  │                              │                                        │  │
│  │          ┌───────────────────┼───────────────────┐                   │  │
│  │          ▼                   ▼                   ▼                   │  │
│  │  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐            │  │
│  │  │   Auth       │   │   System     │   │   其他模块    │            │  │
│  │  │  (认证中心)   │   │  (系统模块)   │   │  (Job/File)  │            │  │
│  │  └──────────────┘   └──────────────┘   └──────────────┘            │  │
│  │                                                                         │  │
│  └─────────────────────────────┬───────────────────────────────────────┘   │
│                                │                                            │
│  ┌─────────────────────────────┼───────────────────────────────────────┐   │
│  │                             ▼                                         │   │
│  │  ┌────────────────────────────────────────────────────────────────┐  │   │
│  │  │                     公共服务层                                   │  │   │
│  │  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐          │  │   │
│  │  │  │  通用Core │ │  Security│ │   Log    │ │   Redis  │          │  │   │
│  │  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘          │  │   │
│  │  └────────────────────────────────────────────────────────────────┘  │   │
│  │                                                                         │  │
│  └─────────────────────────────┬───────────────────────────────────────┘   │
│                                │                                            │
│  ┌─────────────────────────────┼───────────────────────────────────────┐   │
│  │                             ▼                                         │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐             │   │
│  │  │  MySQL   │  │  Redis   │  │ ActiveMQ │  │  MinIO   │             │   │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘             │   │
│  │                                                                         │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 模块清单

| 序号 | 模块名称 | 功能描述 | 类型 |
|------|---------|---------|------|
| 1 | ess-gateway | API网关 | 微服务 |
| 2 | ess-auth | 认证授权中心 | 微服务 |
| 3 | ess-system | 系统管理模块 | 微服务 |
| 4 | ess-job | 定时任务模块 | 微服务 |
| 5 | ess-gen | 代码生成模块 | 微服务 |
| 6 | ess-file | 文件管理模块 | 微服务 |
| 7 | ess-api | API接口定义 | 公共依赖 |
| 8 | ess-common | 公共组件 | 公共依赖 |

### 2.3 Maven 模块结构

```
ess (父POM)
├── ess-api (API接口定义)
│   └── ess-api-system (系统模块API)
├── ess-common (公共组件)
│   ├── ess-common-core (核心工具)
│   ├── ess-common-security (安全组件)
│   ├── ess-common-log (日志组件)
│   ├── ess-common-redis (Redis组件)
│   ├── ess-common-datascope (数据权限)
│   └── ess-common-datasource (多数据源)
├── ess-gateway (网关服务)
├── ess-auth (认证服务)
└── ess-modules (业务模块)
    ├── ess-system (系统管理)
    ├── ess-job (定时任务)
    ├── ess-gen (代码生成)
    └── ess-file (文件管理)
```

---

## 3. 后端架构详解

### 3.1 核心启动类

#### 3.1.1 EssAuthApplication - 认证授权中心

```java
@EnableRyFeignClients
@SpringBootApplication(exclude = {DataSourceAutoConfiguration.class})
@ForestScan(basePackages = "com.ias.ess.auth.client")
public class EssAuthApplication {
    public static void main(String[] args) {
        SpringApplication.run(EssAuthApplication.class, args);
    }
}
```

**功能**:
- 用户认证 (登录/注册/登出)
- Token生成与刷新
- 验证码管理
- 微信登录集成

#### 3.1.2 EssGatewayApplication - 网关服务

```java
@SpringBootApplication(exclude = {DataSourceAutoConfiguration.class})
public class EssGatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(EssGatewayApplication.class, args);
    }
}
```

**功能**:
- 请求路由转发
- 限流熔断
- 认证鉴权
- 请求日志

#### 3.1.3 EssSystemApplication - 系统模块

```java
@EnableCustomConfig
@EnableScheduling
@EnableRyFeignClients
@SpringBootApplication(exclude = DruidDataSourceAutoConfigure.class)
public class EssSystemApplication {
    public static void main(String[] args) {
        SpringApplication.run(EssSystemApplication.class, args);
    }
}
```

### 3.2 核心组件

#### 3.2.1 TokenService - Token管理服务

**位置**: `ess-common-security/service/TokenService.java`

| 属性 | 说明 |
|-----|------|
| 过期时间 | CacheConstants.EXPIRATION |
| 刷新时间 | CacheConstants.REFRESH_TIME (分钟) |
| Token前缀 | CacheConstants.LOGIN_TOKEN_KEY |

**核心方法**:

| 方法名 | 功能 |
|-------|------|
| `createToken(LoginUser)` | 创建JWT Token |
| `getLoginUser()` | 获取当前登录用户 |
| `refreshToken(LoginUser)` | 刷新Token有效期 |
| `deleteToken(String)` | 删除Token |

**Token结构**:

```java
// JWT Claims中存储的信息
claimsMap.put(SecurityConstants.USER_KEY, token);           // Token标识
claimsMap.put(SecurityConstants.DETAILS_USER_ID, userId);   // 用户ID
claimsMap.put(SecurityConstants.DETAILS_USERNAME, userName);// 用户名
claimsMap.put(SecurityConstants.DETAILS_TENANT, tenantId);  // 租户ID
```

#### 3.2.2 认证流程

```
用户登录
    │
    ▼
┌──────────────┐
│  提交用户名密码  │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  验证码校验    │ ←── 可选
└──────┬───────┘
       │
       ▼
┌──────────────┐     ┌─────────────────┐
│ Auth服务验证  │────▶│ 查询用户信息    │
└──────┬───────┘     │ 查询角色权限    │
       │             └─────────────────┘
       ▼
┌──────────────┐
│  生成JWT Token │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  存入Redis    │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  返回Token    │
└──────────────┘
```

### 3.3 系统管理模块 (ess-system)

#### 3.3.1 Controller清单

| Controller | 功能 | 路径 |
|-----------|------|------|
| BaseOilBillController | 油单信息管理 | /oilBill |
| BaseOilBillSignController | 油单签单管理 | /oilBillSign |
| BaseOilBillHandelController | 油单处理记录 | /oilBillHandel |
| BaseAirlineGroupController | 航空公司组 | /airlineGroup |
| BaseAirplaceController | 机场/机位管理 | /airplace |
| BaseContactsController | 联系人管理 | /contacts |
| BaseRegistrationController | 飞机注册号管理 | /registration |
| BaseVehicleController | 车辆管理 | /vehicle |
| BaseProxyCompanyController | 代理公司管理 | /proxyCompany |
| BaseSignAirlineController | 签单航空公司 | /signAirline |
| BaseSrvTaskController | 服务任务管理 | /srvTask |
| BaseUserConfigController | 用户配置管理 | /userConfig |
| BaseFieldConfigController | 字段配置管理 | /fieldConfig |
| PushInterfaceConfigController | 推送接口配置 | /pushInterface |
| SysUserController | 用户管理 | /system/user |
| SysRoleController | 角色管理 | /system/role |
| SysMenuController | 菜单管理 | /system/menu |
| SysDeptController | 部门管理 | /system/dept |
| SysConfigController | 参数配置 | /system/config |
| SysDictDataController | 字典数据 | /system/dict/data |
| SysDictTypeController | 字典类型 | /system/dict/type |
| SysNoticeController | 通知公告 | /system/notice |
| SysLogininforController | 登录日志 | /system/logininfor |
| SysOperlogController | 操作日志 | /system/operlog |
| SysProfileController | 个人中心 | /system/profile |

#### 3.3.2 核心业务Controller示例

**BaseOilBillController - 油单管理**:

```java
@RestController
@RequestMapping("/oilBill")
public class BaseOilBillController extends BaseController {
    
    @Autowired
    private IBaseOilBillService baseOilBillService;
    
    // 查询油单列表
    @GetMapping("/list")
    public TableDataInfo list(BaseOilBill baseOilBill) {
        startPage();
        List<BaseOilBill> list = baseOilBillService.selectBaseOilBillList(baseOilBill);
        return getDataTable(list);
    }
    
    // 导出油单
    @Log(title = "油单信息", businessType = BusinessType.EXPORT)
    @PostMapping("/export")
    public void export(HttpServletResponse response, BaseOilBill baseOilBill) {
        ExcelUtil<BaseOilBill> util = new ExcelUtil<BaseOilBill>(BaseOilBill.class);
        util.exportExcel(response, list, "油单信息数据");
    }
    
    // 新增油单
    @Log(title = "油单信息", businessType = BusinessType.INSERT)
    @PostMapping
    public AjaxResult add(@RequestBody BaseOilBill baseOilBill) {
        return toAjax(baseOilBillService.insertBaseOilBill(baseOilBill));
    }
    
    // 修改油单
    @Log(title = "油单信息", businessType = BusinessType.UPDATE)
    @PutMapping
    public AjaxResult edit(@RequestBody BaseOilBill baseOilBill) {
        return toAjax(baseOilBillService.updateBaseOilBill(baseOilBill));
    }
}
```

### 3.4 数据模型

#### 3.4.1 核心实体类

| 实体类 | 功能说明 |
|-------|---------|
| BaseOilBill | 油单信息 |
| BaseOilBillSign | 油单签单记录 |
| BaseOilBillHandel | 油单处理日志 |
| BaseAirlineGroup | 航空公司组 |
| BaseAirplace | 机场/机位 |
| BaseRegistration | 飞机注册号 |
| BaseVehicle | 车辆信息 |
| BaseProxyCompany | 代理公司 |
| BaseSignAirline | 签单航空公司 |
| BaseSrvTask | 服务任务 |
| BaseUserConfig | 用户配置 |
| BaseFieldConfig | 字段配置 |
| SysUser | 系统用户 |
| SysRole | 系统角色 |
| SysMenu | 系统菜单 |
| SysDept | 部门 |

---

## 4. 前端架构详解

### 4.1 项目结构

```
ess-ui/
├── src/
│   ├── api/                    # API接口定义
│   │   ├── login.js            # 登录相关接口
│   │   ├── menu.js             # 菜单接口
│   │   └── system/             # 系统管理接口
│   │       ├── user.js
│   │       ├── role.js
│   │       ├── dept.js
│   │       ├── menu.js
│   │       └── ...
│   ├── assets/                 # 静态资源
│   │   ├── icons/              # 图标
│   │   ├── images/             # 图片
│   │   └── styles/             # 样式
│   ├── components/             # 公共组件
│   │   ├── Pagination/         # 分页组件
│   │   ├── RightToolbar/       # 右侧工具栏
│   │   ├── IconSelect/         # 图标选择器
│   │   ├── FileUpload/         # 文件上传
│   │   ├── ImageUpload/        # 图片上传
│   │   ├── DictTag/            # 字典标签
│   │   └── ...
│   ├── directive/              # 自定义指令
│   │   ├── permission/         # 权限指令
│   │   └── dialog/             # 对话框指令
│   ├── layout/                 # 布局组件
│   ├── plugins/                # 插件
│   │   ├── auth.js             # 认证插件
│   │   ├── cache.js            # 缓存插件
│   │   ├── modal.js            # 模态框插件
│   │   └── tab.js              # 标签页插件
│   ├── router/                 # 路由配置
│   ├── store/                  # Vuex状态管理
│   │   ├── index.js
│   │   └── modules/
│   │       ├── user.js         # 用户状态
│   │       ├── app.js          # 应用状态
│   │       └── permission.js   # 权限状态
│   ├── utils/                  # 工具函数
│   │   ├── request.js          # HTTP请求封装
│   │   ├── auth.js             # 认证工具
│   │   ├── validate.js         # 表单验证
│   │   └── ...
│   └── views/                  # 页面视图
│       ├── login.vue           # 登录页
│       ├── dashboard/          # 首页
│       ├── system/             # 系统管理
│       ├── monitor/            # 监控管理
│       └── tool/               # 工具类
├── main.js                     # 入口文件
├── App.vue                     # 根组件
├── router/index.js             # 路由配置
└── store/index.js              # Vuex配置
```

### 4.2 核心组件详解

#### 4.2.1 HTTP请求封装 (request.js)

**功能**: Axios请求拦截与响应处理

```javascript
// 创建axios实例
const service = axios.create({
  baseURL: process.env.VUE_APP_BASE_API,
  timeout: 30000
});

// 请求拦截器
service.interceptors.request.use(config => {
  // Token注入
  if (getToken() && !isToken) {
    config.headers['Authorization'] = 'Bearer ' + getToken();
  }
  
  // 防重复提交
  if (!isRepeatSubmit && (config.method === 'post' || config.method === 'put')) {
    const requestObj = {
      url: config.url,
      data: typeof config.data === 'object' ? JSON.stringify(config.data) : config.data,
      time: new Date().getTime()
    }
    // 1秒内相同请求视为重复提交
  }
  
  return config;
});

// 响应拦截器
service.interceptors.response.use(res => {
  const code = res.data.code || 200;
  
  if (code === 401) {
    // Token过期，重新登录
    MessageBox.confirm('登录状态已过期...', '系统提示', {...});
    return Promise.reject('无效的会话');
  } else if (code === 500) {
    Message({ message: msg, type: 'error' });
    return Promise.reject(new Error(msg));
  } else if (code === 601) {
    Message({ message: msg, type: 'warning' });
    return Promise.reject('error');
  }
  
  return res.data;
});
```

#### 4.2.2 权限控制 (permission.js)

**功能**: 路由守卫与权限验证

```javascript
const whiteList = ['/login', '/auth-redirect', '/bind', '/register'];

router.beforeEach((to, from, next) => {
  NProgress.start();
  
  if (getToken()) {
    if (to.path === '/login') {
      next({ path: '/' });
    } else {
      // 判断是否已获取用户信息
      if (store.getters.roles.length === 0) {
        store.dispatch('GetInfo').then(() => {
          store.dispatch('GenerateRoutes').then(accessRoutes => {
            router.addRoutes(accessRoutes); // 动态添加路由
            next({ ...to, replace: true });
          });
        });
      } else {
        next();
      }
    }
  } else {
    // 白名单直接访问
    if (whiteList.indexOf(to.path) !== -1) {
      next();
    } else {
      next(`/login?redirect=${to.fullPath}`);
    }
  }
});
```

#### 4.2.3 用户状态管理 (user.js)

```javascript
const user = {
  state: {
    token: getToken(),
    name: '',
    avatar: '',
    roles: [],
    permissions: []
  },
  
  mutations: {
    SET_TOKEN: (state, token) => { state.token = token; },
    SET_NAME: (state, name) => { state.name = name; },
    SET_AVATAR: (state, avatar) => { state.avatar = avatar; },
    SET_ROLES: (state, roles) => { state.roles = roles; },
    SET_PERMISSIONS: (state, permissions) => { state.permissions = permissions; }
  },
  
  actions: {
    // 登录
    Login({ commit }, userInfo) {
      const username = userInfo.username;
      const password = stringToBase64(userInfo.password); // 密码加密
      return new Promise((resolve, reject) => {
        login(username, password, code, uuid).then(res => {
          const data = res.data;
          setToken(data.access_token);
          commit('SET_TOKEN', data.access_token);
          resolve();
        }).catch(error => reject(error));
      });
    },
    
    // 获取用户信息
    GetInfo({ commit }) {
      return new Promise((resolve, reject) => {
        getInfo().then(res => {
          const user = res.user;
          if (res.roles && res.roles.length > 0) {
            commit('SET_ROLES', res.roles);
            commit('SET_PERMISSIONS', res.permissions);
          } else {
            commit('SET_ROLES', ['ROLE_DEFAULT']);
          }
          commit('SET_NAME', user.userName);
          resolve(res);
        });
      });
    },
    
    // 登出
    LogOut({ commit }) {
      return new Promise((resolve) => {
        logout().then(() => {
          commit('SET_TOKEN', '');
          commit('SET_ROLES', []);
          removeToken();
          resolve();
        });
      });
    }
  }
};
```

### 4.3 核心页面

#### 4.3.1 登录页面 (login.vue)

**功能**: 用户登录、验证码、记住密码

```vue
<template>
  <div class="login">
    <el-form :model="loginForm" :rules="loginRules">
      <h3 class="title">浦东航油电子签单系统</h3>
      
      <el-form-item prop="username">
        <el-input v-model="loginForm.username" placeholder="账号" />
      </el-form-item>
      
      <el-form-item prop="password">
        <el-input v-model="loginForm.password" type="password" placeholder="密码" />
      </el-form-item>
      
      <el-form-item prop="code" v-if="captchaEnabled">
        <el-input v-model="loginForm.code" placeholder="验证码" style="width: 63%" />
        <div class="login-code">
          <img :src="codeUrl" @click="getCode" />
        </div>
      </el-form-item>
      
      <el-checkbox v-model="loginForm.rememberMe">记住密码</el-checkbox>
      
      <el-button type="primary" @click.native.prevent="handleLogin">
        登 录
      </el-button>
    </el-form>
  </div>
</template>
```

### 4.4 API接口清单

#### 4.4.1 登录相关接口

| 接口 | 方法 | 功能 |
|------|------|------|
| /auth/login | POST | 用户登录 |
| /auth/register | POST | 用户注册 |
| /auth/refresh | POST | 刷新Token |
| /auth/logout | DELETE | 退出登录 |
| /code | GET | 获取验证码 |
| /system/user/getInfo | GET | 获取用户信息 |

#### 4.4.2 系统管理接口

| 模块 | 接口路径 | 功能 |
|------|---------|------|
| 用户管理 | /system/user | 用户CRUD |
| 角色管理 | /system/role | 角色CRUD |
| 菜单管理 | /system/menu | 菜单CRUD |
| 部门管理 | /system/dept | 部门CRUD |
| 字典管理 | /system/dict | 字典CRUD |
| 配置管理 | /system/config | 参数配置 |
| 油单管理 | /oilBill | 油单CRUD |

---

## 5. 安全机制

### 5.1 认证体系

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          认证体系架构                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. 登录认证                                                            │
│     ├── 用户名密码认证                                                  │
│     ├── 验证码校验 (可选)                                               │
│     ├── Token生成 (JWT)                                                │
│     └── Token存储 (Redis)                                              │
│                                                                         │
│  2. Token认证                                                           │
│     ├── 请求头携带Token                                                 │
│     ├── Gateway拦截验证                                                 │
│     ├── 解析用户信息                                                    │
│     └── Redis有效性校验                                                 │
│                                                                         │
│  3. 权限控制                                                            │
│     ├── 角色权限 (RBAC)                                                │
│     ├── 接口权限                                                        │
│     ├── 数据权限                                                        │
│     └── 按钮权限                                                        │
│                                                                         │
│  4. 安全防护                                                            │
│     ├── 防重复提交                                                      │
│     ├── XSS防护                                                         │
│     ├── SQL注入防护                                                     │
│     └── 请求日志记录                                                     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.2 密码安全

**前端密码加密**:

```javascript
// 使用Base64加密
const password = stringToBase64(userInfo.password);

// 或使用RSA加密
import { encrypt } from "@/utils/jsencrypt";
const encryptedPassword = encrypt(password);
```

**后端密码处理**:
- 使用BCrypt或MD5加密存储
- 密码强度校验
- 登录失败次数限制

### 5.3 接口安全

**请求头配置**:

```javascript
// Token注入
config.headers['Authorization'] = 'Bearer ' + getToken();

// 防重复提交
const requestObj = {
  url: config.url,
  data: typeof config.data === 'object' ? JSON.stringify(config.data) : config.data,
  time: new Date().getTime()
}
// 1秒内相同请求视为重复提交
```

---

## 6. 潜在Bug风险点分析

### 6.1 高风险问题

#### 6.1.1 Token刷新机制

**位置**: `TokenService.refreshToken()`

**风险**: 
- Token过期时间固定
- 刷新窗口期过短
- 并发刷新问题

**建议**:
- 实现Token自动续期
- 添加刷新锁
- 支持多设备登录

#### 6.1.2 前端密码安全

**位置**: `user.js` 登录逻辑

```javascript
// 当前实现
const password = stringToBase64(userInfo.password);

// 风险: Base64编码不是加密，可逆
```

**建议**: 
- 使用RSA非对称加密
- 使用HTTPS传输

### 6.2 中等风险问题

#### 6.2.1 防重复提交限制过严

**位置**: `request.js`

```javascript
const interval = 1000; // 1秒间隔
if (s_data === requestObj.data && requestObj.time - s_time < interval) {
  return Promise.reject(new Error，请勿重复提交('数据正在处理'));
}
```

**影响**: 正常操作可能被误判为重复提交

**建议**:
- 放宽时间间隔
- 或者使用唯一请求ID

#### 6.2.2 错误处理不完整

**位置**: 多处Controller

```java
// 部分接口缺少权限注解
//@RequiresPermissions("system:oilBill:list")
```

**建议**: 统一添加权限注解

#### 6.2.3 缓存穿透风险

**位置**: `TokenService.getLoginUser()`

```java
public LoginUser getLoginUser(String token) {
    LoginUser user = null;
    try {
        user = redisService.getCacheObject(RedisConstants.LOGIN_TOKEN_KEY + token);
    } catch (Exception e) {
        // 仅记录日志
    }
    return user;
}
```

**建议**: 添加缓存空值保护

### 6.3 低风险问题

#### 6.3.1 硬编码配置

**位置**: 多处使用魔法值

```java
private final static long expireTime = CacheConstants.EXPIRATION;
```

**建议**: 配置文件外化

#### 6.3.2 日志敏感信息

**位置**: 日志输出

```java
logger.info("用户【" + employeeNumber + "】登录成功");
// 建议脱敏
```

---

## 7. 部署架构

### 7.1 Docker部署

```
docker/
├── docker-compose.yml          # 编排文件
├── mysql/                      # MySQL配置
│   ├── dockerfile
│   └── db/
├── nacos/                      # Nacos配置
│   ├── dockerfile
│   └── conf/
├── nginx/                      # Nginx配置
│   ├── dockerfile
│   └── conf/
├── redis/                      # Redis配置
│   ├── dockerfile
│   └── conf/
└── ruoyi/                      # 应用镜像
    ├── auth/
    ├── gateway/
    ├── modules/
    └── visual/
```

### 7.2 启动脚本

| 脚本 | 功能 |
|------|------|
| run-auth.bat | 启动认证服务 |
| run-gateway.bat | 启动网关服务 |
| run-modules-system.bat | 启动系统模块 |
| run-modules-job.bat | 启动定时任务 |
| run-modules-file.bat | 启动文件服务 |
| run-modules-gen.bat | 启动代码生成 |

---

## 8. 开发规范

### 8.1 代码规范

#### 8.1.1 Controller规范

```java
@RestController
@RequestMapping("/模块名")
public class XxxController extends BaseController {
    
    @Autowired
    private IXxxService xxxService;
    
    // 权限注解
    //@RequiresPermissions("system:xxx:list")
    @GetMapping("/list")
    public TableDataInfo list(Xxx xxx) {
        startPage();
        List<X xxxService.selectXxx> list =xxList(xxx);
        return getDataTable(list);
    }
    
    // 日志注解
    @Log(title = "模块名称", businessType = BusinessType.EXPORT)
    @PostMapping("/export")
    public void export(HttpServletResponse response, Xxx xxx) {
        // 导出逻辑
    }
}
```

#### 8.1.2 Service规范

```java
public interface IXxxService {
    List<Xxx> selectXxxList(Xxx xxx);
    Xxx selectXxxById(Long id);
    int insertXxx(Xxx xxx);
    int updateXxx(Xxx xxx);
    int deleteXxxByIds(Long[] ids);
}

@Service
public class XxxServiceImpl implements IXxxService {
    // 实现逻辑
}
```

### 8.2 前端组件规范

#### 8.2.1 API定义

```javascript
// api/system/user.js
import request from '@/utils/request'

export function listUser(query) {
  return request({
    url: '/system/user/list',
    method: 'get',
    params: query
  })
}

export function getUser(userId) {
  return request({
    url: '/system/user/' + userId,
    method: 'get'
  })
}
```

---

## 9. 改进建议

### 9.1 安全增强

1. **密码加密升级**
   - 前端使用RSA加密
   - 后端使用BCrypt

2. **Token安全**
   - 增加Token绑定IP
   - 实现Token自动续期
   - 支持Token黑名单

3. **接口安全**
   - 添加请求签名
   - 增强防重复提交
   - 增加限流保护

### 9.2 性能优化

1. **缓存优化**
   - 热点数据缓存
   - 二级缓存支持
   - 缓存预热

2. **数据库优化**
   - 添加索引
   - 慢查询监控
   - 连接池调优

3. **前端优化**
   - 路由懒加载
   - 组件按需引入
   - 静态资源CDN

### 9.3 可维护性

1. **代码规范**
   - 统一编码规范
   - 添加单元测试
   - 代码审查流程

2. **文档完善**
   - API文档Swagger
   - 架构设计文档
   - 部署运维手册

---

## 附录

### A. 依赖版本清单

| 依赖 | 版本 | 说明 |
|------|------|------|
| Spring Boot | 2.7.7 | 基础框架 |
| Spring Cloud | 2021.0.5 | 微服务 |
| Spring Cloud Alibaba | 2021.0.4.0 | 阿里组件 |
| MyBatis-Plus | - | ORM |
| Druid | 1.2.15 | 连接池 |
| JWT | 0.9.1 | Token |
| FastJSON | 2.0.22 | JSON |
| Lombok | 1.18.28 | 简化代码 |
| MinIO | 8.2.2 | 对象存储 |
| POI | 4.1.2 | Excel |

### B. 前端依赖清单

| 依赖 | 版本 | 说明 |
|------|------|------|
| Vue | 2.6.12 | 核心框架 |
| Element-UI | 2.15.10 | UI组件 |
| Axios | 0.24.0 | HTTP客户端 |
| Vuex | 3.6.0 | 状态管理 |
| Vue-Router | 3.5.1 | 路由 |
| ag-Grid | 25.1.0 | 表格 |
| ECharts | 5.4.0 | 图表 |
| jsencrypt | 3.0.0 | 加密 |

### C. 配置参数说明

| 配置项 | 说明 | 默认值 |
|-------|------|-------|
| server.port | 服务端口 | 8080 |
| spring.datasource.url | 数据库URL | - |
| spring.redis.host | Redis地址 | localhost |
| jwt.secret | JWT密钥 | - |
| jwt.expiration | Token过期时间 | 86400000ms |
| logging.level | 日志级别 | INFO |

---

> **报告完成**  
> 如有问题或需要更详细的分析，请联系技术团队。
