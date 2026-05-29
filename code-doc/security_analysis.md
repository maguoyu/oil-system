# security 安全模块代码分析报告

> 分析日期：2026-02-04
> 项目路径：e:\work\code\oil\pudong\security\
> 文档版本：1.0

---

## 1. 项目概述

### 1.1 基本信息

| 属性 | 值 |
|------|-----|
| 项目名称 | security (安全模块) |
| GroupId | com.ias.lib |
| ArtifactId | security |
| 版本 | 5.0.2.0 (parent) |
| 打包方式 | pom (父模块) |
| Java版本 | 1.7 |
| Spring版本 | 3.2.4.RELEASE |

### 1.2 模块结构

```
security/                                    # 父模块 (v5.0.2.0)
├── pom.xml                                # 父POM配置
├── security-api/                           # API接口模块 (v5.0.2.1)
│   ├── pom.xml
│   └── src/main/java/com/ias/security/
│       ├── webservice/                    # WebService接口
│       │   ├── IAuthService.java          # 认证服务接口
│       │   ├── IUserService.java         # 用户服务接口
│       │   ├── IResourceService.java     # 资源服务接口
│       │   ├── ISecurityService.java      # 安全服务接口
│       │   └── model/                    # 数据模型
│       │       ├── B2BRequest.java       # 请求模型
│       │       ├── B2BResponse.java      # 响应模型
│       │       ├── B2BCustomResponse.java # 自定义响应
│       │       ├── WSUser.java           # Web服务用户
│       │       ├── WSRole.java           # Web服务角色
│       │       ├── WSResource.java       # Web服务资源
│       │       ├── LoginInfo.java         # 登录信息
│       │       └── ...
│       └── util/                         # 工具类
│
├── security-impl/                          # 服务实现模块 (vPVG.5.0.2.3)
│   ├── pom.xml
│   └── src/main/java/com/ias/security/
│       ├── webservice/                    # WebService实现
│       │   ├── AuthService.java          # 认证服务实现
│       │   ├── UserService.java          # 用户服务实现
│       │   ├── ResourceService.java      # 资源服务实现
│       │   └── BaseService.java          # 服务基类
│       ├── shiro/                        # Shiro安全框架
│       │   ├── SecurityManager.java      # 安全管理器
│       │   ├── SecurityRealm.java        # 安全域
│       │   ├── SecuritySessionListener.java
│       │   ├── UniquePrincipalSecurityManager.java
│       │   └── cluster/                  # 集群支持
│       ├── persistence/                   # 数据访问层
│       │   ├── dao/                      # DAO接口
│       │   ├── mapper/                   # MyBatis Mapper
│       │   └── provider/                 # 数据提供者
│       ├── domain/                       # 领域模型
│       │   ├── User.java                # 用户实体
│       │   ├── Role.java                # 角色实体
│       │   ├── Resource.java            # 资源实体
│       │   ├── Group.java               # 组实体
│       │   └── QueryResult.java         # 查询结果
│       ├── exception/                    # 异常定义
│       │   ├── AuthenticationFailedException.java
│       │   ├── SessionTimeoutException.java
│       │   ├── MissingAuthorizationException.java
│       │   ├── UnauthenticatedException.java
│       │   └── base/                    # 异常基类
│       ├── util/                         # 工具类
│       │   ├── AssertUtil.java          # 断言工具
│       │   ├── ConfigUtil.java          # 配置工具
│       │   ├── PermissionUtil.java      # 权限工具
│       │   ├── MessageUtil.java         # 消息工具
│       │   └── XMLUtil.java            # XML工具
│       ├── main/                         # 启动类
│       │   ├── SecurityManagerMain.java
│       │   └── DatabaseInitMain.java
│       └── log/                          # 日志
│
└── security-web/                           # Web模块
    └── (前端资源)
```

### 1.3 业务定位

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         security 业务定位                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                         安全认证核心服务                              │   │
│   │                                                                      │   │
│   │   ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │   │
│   │   │  用户认证   │  │  权限管理   │  │  会话管理               │  │   │
│   │   │  login()  │  │  permission │  │  Shiro Session          │  │   │
│   │   └─────────────┘  └─────────────┘  └─────────────────────────┘  │   │
│   │                                                                      │   │
│   │   ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │   │
│   │   │  角色管理   │  │  资源管理   │  │  多租户支持             │  │   │
│   │   │  Role      │  │  Resource   │  │  Tenant isolation       │  │   │
│   │   └─────────────┘  └─────────────┘  └─────────────────────────┘  │   │
│   │                                                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                              │                                              │
│                              ▼                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                         调用方                                        │   │
│   │                                                                      │   │
│   │   ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │   │
│   │   │  resource   │  │  mis       │  │  flight-oil             │  │   │
│   │   │  资源模块    │  │  MIS系统   │  │  航班供油系统           │  │   │
│   │   └─────────────┘  └─────────────┘  └─────────────────────────┘  │   │
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
| **Apache Shiro** | 1.2.2 | 安全框架 (认证/授权) |
| **Spring** | 3.2.4.RELEASE | 应用框架 |
| **MyBatis** | 3.2.3 | ORM框架 |
| **Dubbo** | 2.5.3 | RPC框架 |
| **Zookeeper** | 3.4.6 | 服务注册 |
| **Jedis** | 2.7.3 | Redis客户端 |
| **MySQL** | 5.1.26 | 数据库驱动 |
| **Oracle** | 10.2.0.1.0 | Oracle驱动 |
| **Logback** | 1.2.3 | 日志框架 |

### 2.2 外部服务依赖

| 服务 | GroupId | ArtifactId | 用途 |
|------|----------|------------|------|
| **MISGWSAPI** | com.ias.lib.mis | MISGWSAPI | 5.0.2.0 | MIS用户/部门服务 |

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
│  │   │ resource模块  │  │ mis模块     │  │ flight-oil模块          │  │    │
│  │   └──────┬───────┘  └──────┬───────┘  └───────────┬──────────────┘  │    │
│  └──────────┼──────────────────┼──────────────────────┼──────────────────┘   │
│             │                  │                      │                        │
│             └──────────────────┼──────────────────────┘                        │
│                                │                                               │
│                                ▼                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                     Dubbo RPC 层 (WebService)                        │    │
│  │                                                                      │    │
│  │   ┌─────────────────────────────────────────────────────────────┐   │    │
│  │   │  @WebService @Service                                      │   │    │
│  │   │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │   │    │
│  │   │  │AuthService  │  │UserService  │  │ResourceService  │  │   │    │
│  │   │  └──────────────┘  └──────────────┘  └──────────────────┘  │   │    │
│  │   └─────────────────────────────────────────────────────────────┘   │    │
│  │                              │                                      │    │
│  │                              ▼                                      │    │
│  │   ┌─────────────────────────────────────────────────────────────┐   │    │
│  │   │                    SecurityManager                           │   │    │
│  │   │     (Shiro集成、会话管理、权限校验)                          │   │    │
│  │   └─────────────────────────────────────────────────────────────┘   │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                      │                                      │
│                                      ▼                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                         Shiro 安全框架                              │    │
│  │                                                                      │    │
│  │   ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐  │    │
│  │   │SecurityRealm │  │ Credentials  │  │ SessionManager           │  │    │
│  │   │(认证授权域)  │  │Matcher(密码) │  │ (会话管理)               │  │    │
│  │   └──────────────┘  └──────────────┘  └──────────────────────────┘  │    │
│  │                                                                      │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                      │                                      │
│                                      ▼                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                        数据访问层 (MyBatis)                          │    │
│  │                                                                      │    │
│  │   ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐  │    │
│  │   │  UsersDao    │  │  AuthDao     │  │  ResourcesDao            │  │    │
│  │   │  (用户)      │  │  (权限)      │  │  (资源)                  │  │    │
│  │   └──────────────┘  └──────────────┘  └──────────────────────────┘  │    │
│  │                                                                      │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                      │                                      │
│                                      ▼                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                         数据存储                                     │    │
│  │                                                                      │    │
│  │   ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐  │    │
│  │   │  MySQL       │  │  Oracle      │  │  Redis                  │  │    │
│  │   │  (用户/权限)  │  │  (历史数据) │  │  (会话缓存)             │  │    │
│  │   └──────────────┘  └──────────────┘  └──────────────────────────┘  │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.4 Shiro集成架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Shiro 认证授权流程                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────┐     ┌─────────────┐     ┌─────────────────────────────┐      │
│   │  请求   │────▶│Subject.Builder│────▶│  SecurityManager           │      │
│   └─────────┘     └─────────────┘     └─────────────────────────────┘      │
│                                                  │                        │
│                                                  ▼                        │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                    Shiro 核心组件                                    │  │
│   │                                                                      │  │
│   │   ┌─────────────────────────────────────────────────────────────┐  │  │
│   │   │                    Authenticator (认证器)                     │  │  │
│   │   │                                                              │  │  │
│   │   │         ┌───────────────────────────────────────────┐       │  │  │
│   │   │         │              SecurityRealm                │       │  │  │
│   │   │         │  ┌─────────────────┐  ┌───────────────┐  │       │  │  │
│   │   │         │  │doGetAuthenticat │  │doGetAuthoriz │  │       │  │  │
│   │   │         │  │ionInfo()        │  │ationInfo()   │  │       │  │  │
│   │   │         │  └────────┬────────┘  └───────┬───────┘  │       │  │  │
│   │   │         │           │                 │          │       │  │  │
│   │   │         │           ▼                 ▼          │       │  │  │
│   │   │         │  ┌─────────────────┐  ┌───────────────┐  │       │  │  │
│   │   │         │  │  AuthDao       │  │  UsersDao     │  │       │  │  │
│   │   │         │  │  (权限查询)     │  │  (用户查询)   │  │       │  │  │
│   │   │         │  └─────────────────┘  └───────────────┘  │       │  │  │
│   │   │         └───────────────────────────────────────────┘       │  │  │
│   │   └─────────────────────────────────────────────────────────────┘  │  │
│   │                              │                                     │  │
│   │                              ▼                                     │  │
│   │   ┌─────────────────────────────────────────────────────────────┐  │  │
│   │   │              SessionManager (会话管理)                      │  │  │
│   │   │                                                              │  │  │
│   │   │   ┌─────────────────────────────────────────────────────┐   │  │  │
│   │   │   │  EhCache / Redis (会话存储)                         │   │  │  │
│   │   │   └─────────────────────────────────────────────────────┘   │  │  │
│   │   └─────────────────────────────────────────────────────────────┘  │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. 核心API接口

### 3.1 IAuthService (认证服务接口)

| 方法 | 参数 | 返回 | 说明 |
|------|------|------|------|
| `login(B2BRequest req)` | 请求 | B2BCustomResponse | 用户登录 |
| `loginWithoutPermissions(B2BRequest req)` | 请求 | B2BCustomResponse | 登录(不含权限) |
| `logout(B2BRequest req)` | 请求 | B2BCustomResponse | 用户登出 |
| `grantResources(B2BRequest req)` | 请求 | B2BCustomResponse | 授予资源 |
| `grantPermissions(B2BRequest req)` | 请求 | B2BCustomResponse | 授予权限 |
| `getPermissionList(B2BRequest req)` | 请求 | B2BCustomResponse | 获取权限列表 |
| `getPermissionJson(B2BRequest req)` | 请求 | B2BCustomResponse | 获取权限JSON |
| `checkPermissions(B2BRequest req)` | 请求 | B2BCustomResponse | 检查权限 |
| `getTenantId(B2BRequest req, String sessionId)` | 请求 | B2BGetTenantResponse | 获取租户ID |
| `getRolesByUsername(B2BRequest req, String username)` | 请求 | B2BResponse | 获取用户角色 |
| `getPermissionsByUserName(String username)` | 用户名 | List\<String\> | 获取用户权限 |
| `getWSUserByUserName(String userId, String systemId, String pathSimbol)` | 参数 | WSUser | 获取WS用户 |
| `getEmailByUserId(String userId)` | 用户ID | String | 获取邮箱 |
| `getLoginInfo(String userId, String password)` | 账号密码 | LoginInfo | 获取登录信息 |
| `getLoginInfoWithoutPassword(String userId)` | 用户ID | LoginInfo | 获取登录信息(无密码) |
| `getLoginInfoList(String tenant)` | 租户 | List\<LoginInfo\> | 获取登录信息列表 |

### 3.2 IUserService (用户服务接口)

| 方法 | 参数 | 返回 | 说明 |
|------|------|------|------|
| `getUser(String userId)` | 用户ID | WSUser | 获取用户 |
| `createUser(WSUser user)` | 用户 | void | 创建用户 |
| `updateUser(WSUser user)` | 用户 | void | 更新用户 |
| `deleteUser(String userId)` | 用户ID | void | 删除用户 |

### 3.3 IResourceService (资源服务接口)

| 方法 | 参数 | 返回 | 说明 |
|------|------|------|------|
| `getResources()` | - | List\<WSResource\> | 获取资源列表 |
| `getResource(String resourceId)` | 资源ID | WSResource | 获取资源 |
| `createResource(WSResource resource)` | 资源 | void | 创建资源 |
| `updateResource(WSResource resource)` | 资源 | void | 更新资源 |
| `deleteResource(String resourceId)` | 资源ID | void | 删除资源 |

---

## 4. 核心数据模型

### 4.1 B2BRequest (请求模型)

```java
@XmlAccessorType(XmlAccessType.FIELD)
public class B2BRequest implements Serializable {
    private String systemId;              // 系统ID
    private String userId;              // 用户ID
    private String password;            // 密码
    private String securitySession;     // 安全会话ID
    private List<B2BParameter> parameter; // 参数列表
    private Object payload;             // 负载数据
}
```

### 4.2 B2BCustomResponse (响应模型)

```java
public class B2BCustomResponse extends B2BResponse {
    private WSUser user;                 // 用户信息
    private List<String> permissionList; // 权限列表
}
```

### 4.3 WSUser (Web服务用户)

```java
public class WSUser implements Serializable {
    private String userId;             // 用户ID (登录ID)
    private String tenant;            // 租户ID
    private String employeeId;        // 员工ID
    private String userName;          // 用户姓名
    private String password;          // 密码
    private String email;             // 邮箱
    private String createTime;        // 创建时间
    private String lastModifyTime;    // 最后修改时间
    private String lastLoginTime;     // 最后登录时间
    private short state;              // 状态 (0:激活; 1:锁定)
    private String securitySession;   // 安全会话ID
    private List<String> roleIds;     // 角色ID列表
    private List<String> groupIds;    // 组ID列表
    private List<String> permissionList; // 权限列表
    private String permissionJson;    // 权限JSON
}
```

### 4.4 User (内部用户实体)

```java
public class User implements Serializable {
    private String userId;           // 用户ID
    private String userName;        // 用户名
    private String password;        // 密码 (哈希)
    private String employeeId;      // 员工ID
    private String email;           // 邮箱
    private Date createTime;        // 创建时间
    private Date lastModifyTime;    // 修改时间
    private Date lastLoginTime;     // 最后登录
    private short state;            // 状态 (0:active; 1:blocked)
    private String hashSalt;       // 密码盐
}
```

### 4.5 Resource (资源实体)

```java
public class Resource implements Serializable {
    private Integer resId;          // 资源ID
    private String resName;         // 资源名称
    private String resCode;         // 资源代码
    private Integer parentId;       // 父资源ID
    private String resType;        // 资源类型
    private String permission;      // 权限字符串
    private Integer nodeLft;        // 左节点 (树形结构)
    private Integer nodeRgt;        // 右节点 (树形结构)
    private String systemId;        // 系统ID
}
```

### 4.6 LoginInfo (登录信息)

```java
public class LoginInfo {
    private String userId;          // 用户ID
    private String userName;       // 用户姓名
    private String dpid;           // 部门ID
    private String psid;          // 岗位ID
    private String tenant;        // 租户ID
    private String dpFullName;    // 部门全称
    private String dpShortName;   // 部门简称
    private String psFullName;    // 岗位全称
    private String psShortName;   // 岗位简称
    private String tenantFullName; // 租户全称
    private String tenantShortName; // 租户简称
}
```

---

## 5. 核心业务逻辑详解

### 5.1 登录流程 (login)

```java
@WebService
@Service
public class AuthService extends BaseService implements IAuthService {

    @Override
    public B2BCustomResponse login(B2BRequest req) {
        String userId = req.getUserId();
        String password = req.getPassword();
        String systemId = req.getSystemId();

        // 1. 调用SecurityManager进行认证
        String securitySession = securityManager.login(userId, password);

        // 2. 获取租户信息并存储到会话
        Subject requestSubject = new Subject.Builder()
            .sessionId(securitySession)
            .buildSubject();
        Session session = requestSubject.getSession();
        String tenant = usersClient.getTenant(userId);
        session.setAttribute("tenant", tenant);

        // 3. 查询用户信息
        User user = usersDao.getMapper().getUser(userId);

        // 4. 检查用户状态 (state > 0 表示锁定)
        if (user.getState() > 0) {
            // 用户被锁定
        }

        // 5. 构建WSUser返回对象
        WSUser wsUser = buildWSUser(user);
        wsUser.setSecuritySession(securitySession);
        wsUser.setTenant(tenant);

        // 6. 获取用户权限列表
        if (CommonUtil.isEmpty(format) || !"JSON".equalsIgnoreCase(format)) {
            wsUser.setPermissionList(getPermissionList(systemId, userId, pathSimbol));
        } else {
            wsUser.setPermissionJson(getPermissionJson(systemId, userId));
        }

        response = new B2BCustomResponse(B2BResponse.SUCCESS);
        response.setUser(wsUser);

        return response;
    }
}
```

### 5.2 权限获取流程 (getPermissionList)

```java
private List<String> getPermissionList(String systemId, String userId, String pathSimbol) {
    // 1. 获取根资源
    Resource rootRes = null;
    if (!CommonUtil.isEmpty(systemId)) {
        // 指定系统：获取系统根资源
        rootRes = resDao.getMapper().getSystemResource(systemId, ResourceService.RES_ID_SYSTEM);
    } else {
        // 未指定：获取全局根资源
        rootRes = resDao.getMapper().getResource(ResourceService.RES_ID_ROOT);
    }

    // 2. 获取所有叶子节点资源
    List<Resource> leaves = resDao.getMapper().getLeafResources(rootRes);

    // 3. 获取所有资源用于构建树
    List<Resource> allResources = resDao.getMapper().getAllResource();

    // 4. 遍历叶子节点，检查用户是否有权限
    for (Resource leaf : leaves) {
        // 构建从根到叶子的完整路径
        List<Resource> parentsWithChild = new ArrayList<>();
        for (Resource resource : allResources) {
            if (leaf.getNodeLft() > resource.getNodeLft() && 
                resource.getNodeRgt() > leaf.getNodeRgt()) {
                parentsWithChild.add(resource);
            }
        }
        parentsWithChild.add(leaf);
        Collections.sort(parentsWithChild, (o1, o2) -> o1.getNodeLft() - o2.getNodeLft());

        // 5. 构建权限字符串
        String permission = PermissionUtil.buildPermission(parentsWithChild);

        // 6. 检查用户是否有该权限
        if (!permission.equals("") && securityManager.hasPermission(userId, permission)) {
            parentsWithChild.remove(0);  // 移除根节点
            if (onlySystem) {
                parentsWithChild.remove(0);  // 移除系统节点
                parentsWithChild.remove(0);  // 移除系统ID节点
            }
            paths.add(PermissionUtil.buildPermissionPath(parentsWithChild, pathSimbol));
        }
    }

    return paths;
}
```

### 5.3 Shiro Realm认证 (SecurityRealm)

```java
public class SecurityRealm extends AuthorizingRealm {

    @Override
    protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken token) {
        UsernamePasswordToken upToken = (UsernamePasswordToken) token;
        String userId = upToken.getUsername();

        // 获取密码和盐
        String password = null;
        String salt = null;
        switch (saltStyle) {
            case NO_SALT:
                password = getUserPassword(userId)[0];
                break;
            case COLUMN:
                String[] queryResults = getUserPassword(userId);
                password = queryResults[0];
                salt = queryResults[1];
                break;
            case EXTERNAL:
                password = getUserPassword(userId)[0];
                salt = getSaltForUser(userId);
        }

        if (password == null) {
            throw new UnknownAccountException("No password found for user [" + userId + "]");
        }

        // 构建认证信息
        SimpleAuthenticationInfo info = new SimpleAuthenticationInfo(
            userId, 
            password.toCharArray(), 
            getName()
        );
        
        if (salt != null) {
            info.setCredentialsSalt(ByteSource.Util.bytes(salt));
        }

        return info;
    }

    @Override
    protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principals) {
        String userId = (String) getAvailablePrincipal(principals);

        // 获取用户角色
        List<String> roleIds = getUserRoles(userId);

        // 获取角色对应的权限
        List<String> permissions = new ArrayList<>();
        if (!roleIds.isEmpty()) {
            permissions.addAll(getRolePermissions(roleIds));
        }

        SimpleAuthorizationInfo info = new SimpleAuthorizationInfo(
            new HashSet<>(roleIds)
        );
        info.setStringPermissions(new HashSet<>(permissions));

        return info;
    }
}
```

### 5.4 SecurityManager (安全管理器)

```java
@Service
public class SecurityManager {

    public String login(String userId, String password) throws AuthenticationFailedException {
        // 1. 创建Subject
        Subject requestSubject = new Subject.Builder().buildSubject();
        
        // 2. 获取会话ID作为安全会话
        String session = String.valueOf(requestSubject.getSession().getId());
        
        // 3. 执行登录
        loginSubject(requestSubject, userId, password);
        
        return session;
    }

    public boolean checkPermission(String session, String permission) {
        if (session.length() > 0) {
            // 1. 通过会话ID重建Subject
            Subject requestSubject = new Subject.Builder()
                .sessionId(session)
                .buildSubject();
            
            if (requestSubject.isAuthenticated()) {
                try {
                    // 2. 检查权限
                    requestSubject.checkPermission(permission);
                    return true;
                } catch (AuthorizationException e) {
                    // 无权限
                }
            }
        }
        return false;
    }

    public boolean hasPermission(String userId, String permission) {
        // 1. 创建用户PrincipalCollection
        SimplePrincipalCollection c = new SimplePrincipalCollection(
            userId, 
            securityRealm.getName()
        );
        
        // 2. 检查权限
        return securityRealm.isPermitted(c, permission);
    }

    public String hash(String source, String salt) {
        // 密码哈希处理
        CredentialsMatcher matcher = this.securityRealm.getCredentialsMatcher();
        if (matcher instanceof HashedCredentialsMatcher) {
            HashedCredentialsMatcher hashedMatcher = (HashedCredentialsMatcher) matcher;
            SimpleHash hs = new SimpleHash(
                hashedMatcher.getHashAlgorithmName(), 
                source, 
                null,
                hashedMatcher.getHashIterations()
            );
            return hs.toHex();
        }
        return source;
    }

    private void loginSubject(Subject subject, String userId, String password) {
        UsernamePasswordToken token = new UsernamePasswordToken(userId, password);
        try {
            subject.login(token);
            // 1. 更新最后登录时间
            securityRealm.updateLastLoginTime(userId);
            // 2. 绑定Subject到当前线程
            ThreadContext.bind(subject);
        } catch (AuthenticationException e) {
            throw new AuthenticationFailedException(userId);
        }
    }
}
```

### 5.5 密码盐策略 (SaltStyle枚举)

```java
public enum SaltStyle {
    NO_SALT,      // 不使用盐
    COLUMN,       // 盐存储在数据库独立列
    EXTERNAL      // 盐从外部获取
};

// 使用示例
switch (saltStyle) {
    case NO_SALT:
        password = getUserPassword(userId)[0];
        break;
    case COLUMN:
        // 从数据库获取盐
        String[] queryResults = getUserPassword(userId);
        password = queryResults[0];
        salt = queryResults[1];
        break;
    case EXTERNAL:
        // 从外部系统获取盐
        password = getUserPassword(userId)[0];
        salt = getSaltForUser(userId);  // 默认返回用户名
}
```

---

## 6. 权限模型设计

### 6.1 树形资源结构

```
Root (根节点)
├── System:TERM (调度终端系统)
│   ├── Menu (菜单)
│   │   ├── Dispatch (调度管理)
│   │   │   ├── TaskDispatch (任务派发)
│   │   │   └── TaskMonitor (任务监控)
│   │   └── Report (报表)
│   └── Button (按钮)
│       ├── Dispatch.Start (开始作业)
│       ├── Dispatch.End (结束作业)
│       └── Dispatch.Cancel (取消任务)
│
└── System:MIS (MIS管理系统)
    ├── UserManagement (用户管理)
    └── RoleManagement (角色管理)
```

### 6.2 权限字符串构建

```java
// 权限字符串格式: Root:System:TERM:Menu:Dispatch:TaskDispatch
// 或使用分隔符: Root>System>TERM>Menu>Dispatch>TaskDispatch

public static String buildPermission(List<Resource> resources) {
    StringBuilder sb = new StringBuilder();
    for (Resource res : resources) {
        if (sb.length() > 0) {
            sb.append(":");
        }
        sb.append(res.getResName());
    }
    return sb.toString();
}

// 权限路径构建
public static String buildPermissionPath(List<Resource> resources, String symbol) {
    StringBuilder sb = new StringBuilder();
    for (Resource res : resources) {
        if (sb.length() > 0) {
            sb.append(symbol);
        }
        sb.append(res.getResName());
    }
    return sb.toString();
}
```

### 6.3 用户-角色-权限关系

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         权限模型关系图                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────┐      ┌─────────┐      ┌─────────┐      ┌─────────┐          │
│   │  User   │──────│ UserRole │──────│  Role   │──────│RolePerm │      │
│   │  (用户) │      │  (关联)  │      │ (角色)  │      │(关联)   │      │
│   └─────────┘      └─────────┘      └─────────┘      └─────────┘          │
│       │                │                │                │                   │
│       │                │                │                │                   │
│       │                │                │                ▼                   │
│       │                │                │         ┌─────────┐                │
│       │                │                └────────▶│Resource │                │
│       │                │                          │ (资源)  │                │
│       │                │                          └─────────┘                │
│       │                │                               │                     │
│       │                └───────────────────────────────┘                     │
│       │                                                                │
│       ▼                                                                │
│   ┌─────────┐                                                          │
│   │  Tenant │  (多租户隔离)                                             │
│   │ (租户)  │                                                          │
│   └─────────┘                                                          │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 7. 数据访问层

### 7.1 DAO接口列表

| DAO | 说明 | 主要方法 |
|-----|------|----------|
| **UsersDao** | 用户数据访问 | getUser, getUserPassword, getUserAllRoles, loginUser |
| **AuthDao** | 权限数据访问 | getRolePermissions, setRolePermissions, getRolesByUsername |
| **ResourcesDao** | 资源数据访问 | getResource, getLeafResources, getAncestorResourcesWithChild |

### 7.2 MyBatis Mapper示例

```java
// UsersMapper.xml
public interface UsersMapper {
    User getUser(@Param("userId") String userId);
    String getUserPassword(@Param("userId") String userId);
    String getSalt(@Param("userId") String userId);
    List<String> getUserAllRoles(@Param("userId") String userId);
    void loginUser(@Param("userId") String userId, @Param("loginTime") Date loginTime);
}

// AuthMapper.xml
public interface AuthMapper {
    List<String> getRolePermissions(@Param("roleIds") List<String> roleIds);
    void setRolePermissions(@Param("roleId") String roleId, @Param("permissions") List<String> permissions);
    List<String> getRolesByUsername(@Param("username") String username);
    List<String> getPermissionsByUserName(@Param("username") String username);
}

// ResourcesMapper.xml
public interface ResourcesMapper {
    Resource getResource(@Param("resId") Integer resId);
    Resource getSystemResource(@Param("systemId") String systemId, @Param("resId") Integer resId);
    List<Resource> getLeafResources(@Param("root") Resource root);
    List<Resource> getAllResource();
    List<Resource> getAncestorResourcesWithChild(@Param("resId") Integer resId);
}
```

---

## 8. 异常体系

### 8.1 异常类层次

```
Exception
├── RuntimeException
│   ├── SecurityException
│   │   ├── AuthenticationFailedException    # 认证失败
│   │   ├── UnauthenticatedException        # 未认证
│   │   ├── MissingAuthorizationException   # 缺少授权
│   │   └── AuthorizationException          # 授权异常
│   ├── ParameterException
│   │   ├── MissingParameterException       # 缺少参数
│   │   ├── InvalidParameterException      # 无效参数
│   │   └── ParameterTypeException         # 参数类型错误
│   ├── TargetException
│   │   ├── TargetNotExistException        # 目标不存在
│   │   └── TargetExistingException        # 目标已存在
│   ├── SessionTimeoutException             # 会话超时
│   └── UnexpectedException
│       ├── UnexpectedInternalException     # 内部错误
│       └── UnexpectedPayloadException     # 负载错误
└── UserBlockedException                    # 用户被锁定
```

### 8.2 异常使用示例

```java
// 认证失败
if (!authenticate) {
    throw new AuthenticationFailedException(userId);
}

// 未授权访问
try {
    securityManager.checkPermission(session, permission);
} catch (AuthorizationException e) {
    throw new MissingAuthorizationException(userId, serviceName, methodName);
}

// 目标不存在
if (resource == null) {
    throw new TargetNotExistException(resourceId, "资源");
}

// 会话超时
if (subject.getPrincipal() == null) {
    throw new SessionTimeoutException();
}
```

---

## 9. 配置信息

### 9.1 Spring配置

```xml
<!-- Shiro Realm配置 -->
<bean id="securityRealm" class="com.ias.security.shiro.SecurityRealm">
    <property name="credentialsMatcher">
        <bean class="org.apache.shiro.authc.credential.HashedCredentialsMatcher">
            <property name="hashAlgorithmName" value="md5"/>
            <property name="hashIterations" value="2"/>
            <property name="storedCredentialsHexEncoded" value="true"/>
        </bean>
    </property>
    <property name="authenticationQuery" value="SELECT password FROM users WHERE userId=?"/>
    <property name="saltedAuthenticationQuery" value="SELECT password, password_salt FROM users WHERE userId=?"/>
</bean>

<!-- SecurityManager配置 -->
<bean id="securityManager" class="org.apache.shiro.spring.LifecycleBeanPostProcessor"/>
```

### 9.2 Dubbo服务暴露

```xml
<!-- Dubbo Provider配置 -->
<dubbo:application name="dubbo-service-provider-security"/>
<dubbo:registry protocol="zookeeper" address="${dubbo.address}"/>
<dubbo:protocol name="dubbo" port="${dubbo.port}"/>

<!-- 服务暴露 -->
<dubbo:service interface="com.ias.security.webservice.IAuthService" 
               ref="authService"/>
<dubbo:service interface="com.ias.security.webservice.IUserService" 
               ref="userService"/>
<dubbo:service interface="com.ias.security.webservice.IResourceService" 
               ref="resourceService"/>
```

---

## 10. 外部服务集成

### 10.1 MIS服务集成

```java
@Resource
private UsersService usersClient;        // 用户服务
@Resource
private DepartmentService departmentService; // 部门服务
@Resource
private PositionService positionService;   // 岗位服务
@Resource
private TenantService tenantService;     // 租户服务

// 调用示例
String tenant = usersClient.getTenant(userId);
CUsers user = usersClient.getUsersList(request, user);

// 获取登录信息
LoginInfo loginInfo = new LoginInfo();
loginInfo.setDpid(users.getDpid());
loginInfo.setPsid(users.getPsid());
loginInfo.setTenant(users.getTenantid());
```

### 10.2 Redis集成

```java
// 使用Jedis进行会话缓存
@Resource
private JedisPool jedisPool;

// 会话存储
public void saveSession(String sessionId, Session session) {
    try (Jedis jedis = jedisPool.getResource()) {
        jedis.setex(sessionId, timeout, serialize(session));
    }
}

// 会话获取
public Session getSession(String sessionId) {
    try (Jedis jedis = jedisPool.getResource()) {
        byte[] data = jedis.get(sessionId.getBytes());
        return deserialize(data);
    }
}
```

---

## 11. 业务场景分析

### 11.1 用户登录序列图

```
┌─────────────────────────────────────────────────────────────────────┐
│                      用户登录序列图                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Client      AuthService    SecurityManager   SecurityRealm  DB      │
│   │              │               │               │          │        │
│   │  1.login()  │               │               │          │        │
│   │─────────────▶│               │               │          │        │
│   │              │ 2.login()    │               │          │        │
│   │              │─────────────▶│               │          │        │
│   │              │               │ 3.login()    │          │        │
│   │              │               │─────────────▶│          │        │
│   │              │               │               │ 4.认证   │        │
│   │              │               │               │─────────▶│        │
│   │              │               │               │◀─────────│        │
│   │              │               │◀──────────────│          │        │
│   │              │ 5.返回session │               │          │        │
│   │              │◀─────────────│               │          │        │
│   │              │               │               │          │        │
│   │              │ 6.getTenant()│               │          │        │
│   │              │───────────────────────────────────────────────▶   │
│   │              │               │               │          │        │
│   │              │ 7.getUser()  │               │          │        │
│   │              │───────────────────────────────────────────────▶   │
│   │              │               │               │          │        │
│   │              │ 8.getPermissionList()        │          │        │
│   │              │─────────────────────────────────────────────────▶│
│   │              │               │               │          │        │
│   │◀─────────────│ 9.loginResp │               │          │        │
│   │              │               │               │          │        │
└─────────────────────────────────────────────────────────────────────┘
```

### 11.2 权限检查流程

```
┌─────────────────────────────────────────────────────────────────────┐
│                      权限检查流程图                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────┐                                                      │
│  │ 请求到达  │                                                      │
│  └────┬─────┘                                                      │
│       │                                                             │
│       ▼                                                             │
│  ┌──────────────────────────┐                                      │
│  │ 检查用户是否已认证        │                                      │
│  └──────────┬───────────────┘                                      │
│       ┌─────┴─────┐                                                │
│       ▼           ▼                                                 │
│    已认证      未认证                                               │
│       │           │                                                 │
│       ▼           ▼                                                 │
│  ┌──────────────────────────┐                                      │
│  │ 获取请求的权限列表        │                                      │
│  └──────────┬───────────────┘                                      │
│       ┌─────┴─────┐                                                │
│       ▼           ▼                                                 │
│    有权限      无权限                                               │
│       │           │                                                 │
│       ▼           ▼                                                 │
│  ┌───────┐   ┌─────────┐                                           │
│  │ 放行  │   │拒绝请求 │                                           │
│  └───────┘   └─────────┘                                           │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 12. 问题与优化建议

### 12.1 技术债务

| 问题 | 严重程度 | 说明 | 建议 |
|------|----------|------|------|
| **Spring 3.2.4 过时** | 🔴 高 | 2012年发布，存在安全漏洞 | 升级到 4.3.x 或 5.x |
| **Shiro 1.2.2 过时** | 🔴 高 | 建议升级到 1.9.x | 升级到 1.9.1 |
| **Java 1.7 已停止支持** | 🟡 中 | 建议升级到 Java 8+ | 升级到 1.8 |
| **Dubbo 2.5.3 过时** | 🟡 中 | 建议升级到 2.7.x | 考虑升级 |
| **Jedis 2.7.3 过时** | 🟡 中 | 建议升级到 3.x | 升级到 3.7.0 |
| **Log4j 1.2.17 存在漏洞** | 🔴 高 | Log4j 1.x有安全漏洞 | 迁移到Logback |

### 12.2 安全问题

| 问题 | 描述 | 建议 |
|------|------|------|
| **密码明文传输** | B2BRequest中password字段明文 | 使用HTTPS或加密传输 |
| **密码哈希不够强** | MD5哈希，可被暴力破解 | 使用SHA-256或BCrypt |
| **无登录防护** | 无验证码、无登录限制 | 添加登录防护机制 |
| **会话固定** | 会话管理存在固定风险 | 使用随机会话ID |

```java
// 问题代码
public void login(B2BRequest req) {
    String password = req.getPassword();  // 明文密码
    // ...
}

// 优化建议
// 1. 使用HTTPS传输
// 2. 前端哈希后传输
// 3. 服务端使用BCrypt
// 4. 添加登录失败次数限制
```

### 12.3 性能问题

| 问题 | 描述 | 建议 |
|------|------|------|
| **权限查询效率低** | 每次登录遍历所有叶子节点 | 添加权限缓存 |
| **数据库查询过多** | 多次数据库查询组装用户信息 | 使用JOIN或批量查询 |
| **无缓存机制** | 用户/角色信息无缓存 | 引入Redis缓存 |

```java
// 问题代码
private List<String> getPermissionList(String systemId, String userId, String pathSimbol) {
    // 1. 查询所有资源 - O(n)
    List<Resource> allResources = resDao.getMapper().getAllResource();
    // 2. 查询叶子节点 - O(n)
    List<Resource> leaves = resDao.getMapper().getLeafResources(rootRes);
    // 3. 嵌套循环检查权限 - O(n²)
    for (Resource leaf : leaves) {
        for (Resource resource : allResources) {
            // ...
        }
    }
}

// 优化建议
// 1. 缓存allResources到Redis
// 2. 使用SQL JOIN替代Java层嵌套
// 3. 缓存用户权限列表
```

### 12.4 代码质量问题

```java
// 问题1: 异常处理不当
} catch (DataAccessException e) {
    final String message = "There was a SQL error...";
    log.error(message, e);
    throw new AuthorizationException(message, e);  // 丢失原始异常信息
}

// 优化
} catch (DataAccessException e) {
    throw new AuthorizationException("Authorization query failed for user: " + userId, e);
}

// 问题2: 魔法数字
if (user.getState() > 0) {  // 0表示什么？
    // ...
}

// 优化
private static final short STATE_BLOCKED = 1;
if (user.getState() == STATE_BLOCKED) {
    throw new UserBlockedException(userId);
}

// 问题3: 方法过长
public B2BCustomResponse login(B2BRequest req) {
    // 150+行代码
}

// 优化：拆分为多个私有方法
private WSUser buildWSUser(User user, String tenant) { ... }
private List<String> buildPermissionList(...) { ... }
```

---

## 13. 总结

### 13.1 模块特点

| 特点 | 说明 |
|------|------|
| **Shiro集成** | 使用Shiro 1.2.2实现认证授权 |
| **WebService** | 基于CXF的WebService接口 |
| **多租户支持** | 租户隔离，Tenant ID存储在会话中 |
| **树形权限** | 资源使用嵌套集合模型管理 |
| **密码哈希** | 支持多种盐策略 (NO_SALT/COLUMN/EXTERNAL) |
| **Dubbo服务** | 通过Dubbo暴露服务供其他模块调用 |

### 13.2 核心能力

1. **用户认证**：用户名密码认证、会话管理
2. **权限管理**：基于角色的访问控制 (RBAC)
3. **资源管理**：树形结构资源，动态权限
4. **租户隔离**：多租户用户数据隔离
5. **外部集成**：与MIS系统集成获取用户/部门信息

### 13.3 依赖关系

```
security
├── security-api (接口定义)
│   ├── B2BRequest/Response
│   ├── WSUser/WSRole/WSResource
│   └── LoginInfo
│
└── security-impl (服务实现)
    ├── AuthService (认证服务)
    ├── UserService (用户服务)
    ├── ResourceService (资源服务)
    ├── SecurityRealm (Shiro域)
    ├── SecurityManager (安全管理器)
    ├── MyBatis Mappers (数据访问)
    │
    └── 外部依赖
        ├── MISGWSAPI (用户/部门/岗位)
        ├── Shiro 1.2.2 (安全框架)
        ├── MyBatis 3.2.3 (ORM)
        └── Redis (会话缓存)
```

### 13.4 后续建议

1. **框架升级**：Spring 3.2 → 5.x, Shiro 1.2 → 1.9
2. **安全加固**：HTTPS传输、BCrypt密码、登录防护
3. **性能优化**：权限缓存、会话缓存、批量查询
4. **监控完善**：添加认证授权指标监控
5. **代码重构**：拆分大方法、统一异常处理

---

*报告生成时间：2026-02-04*
*分析工具：Cursor AI Code Analysis*
