# resourceparent 资源模块代码分析报告

> 分析日期：2026-02-04
> 项目路径：e:\work\code\oil\pudong\resourceparent\
> 文档版本：1.0

---

## 1. 项目概述

### 1.1 基本信息

| 属性 | 值 |
|------|-----|
| 项目名称 | resourceparent (资源父模块) |
| GroupId | com.ias.lib.resource |
| 版本 | 5.0.3.0 (parent) |
| 打包方式 | pom (父模块) |
| 框架 | Spring Boot 1.5.3.RELEASE |
| Java版本 | 1.7 |

### 1.2 模块结构

```
resourceparent/                          # 父模块 (v5.0.3.0)
├── pom.xml                            # 父POM配置
├── resourceapi/                       # API模块 (v5.0.3.1)
│   ├── pom.xml
│   └── src/main/java/com/ias/lib/resource/
│       ├── api/                       # 服务接口定义
│       │   ├── ILoginWebservice.java
│       │   ├── IDispatchLoginService.java
│       │   ├── IUserService.java
│       │   ├── ILoginDetail.java
│       │   ├── TaskAllot.java
│       │   ├── VehicleInterface.java
│       │   └── IDataStore.java
│       └── model/                     # 数据模型
│           ├── bean/
│           │   ├── resource/          # 资源实体
│           │   ├── terminal/        # 终端相关
│           │   ├── vehicle/        # 车辆相关
│           │   └── util/           # 工具类
│           └── resultbean/          # 结果模型
│
└── login/                            # 实现模块 (v5.0.3.5)
    ├── pom.xml
    └── src/main/java/com/ias/server/resource/
        ├── ResourceApp.java        # Spring Boot入口
        ├── rpc/                     # Dubbo服务实现
        ├── service/                 # 业务逻辑实现
        ├── dao/                     # 数据访问层
        │   ├── entity/             # JPA实体
        │   └── repository/         # Spring Data JPA
        ├── job/                    # 定时任务
        └── util/                   # 工具类
```

### 1.3 业务定位

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         resourceparent 业务定位                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                         资源管理核心服务                              │   │
│   │                                                                      │   │
│   │   ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │   │
│   │   │  用户认证   │  │  终端管理   │  │  车辆绑定                │  │   │
│   │   │  login()   │  │  register() │  │  bind/unbind            │  │   │
│   │   └─────────────┘  └─────────────┘  └─────────────────────────┘  │   │
│   │                                                                      │   │
│   │   ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │   │
│   │   │  任务分配   │  │  在线状态   │  │  消息通知                │  │   │
│   │   │  findUser()│  │  timeout()  │  │  Camel/ActiveMQ          │  │   │
│   │   └─────────────┘  └─────────────┘  └─────────────────────────┘  │   │
│   │                                                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                              │                                              │
│                              ▼                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                         外部服务依赖                                  │   │
│   │                                                                      │   │
│   │   ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │   │
│   │   │  MIS用户    │  │  Security  │  │  GlobalSequence          │  │   │
│   │   │  服务       │  │  认证      │  │  序列号生成              │  │   │
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
| Spring Boot | 1.5.3.RELEASE | 应用框架 |
| Dubbo | 2.5.3 | RPC框架 |
| Zookeeper | 3.4.6 | 服务注册 |
| ActiveMQ | 5.12.0 | 消息队列 |
| Apache Camel | 2.17.0 | 消息路由 |
| Jackson | 1.9.11 | JSON处理 |
| MySQL | 5.1.35 | 数据库驱动 |
| Spring Data JPA | - | ORM框架 |
| Logstash Logback | 4.11 | 日志 |

### 2.2 外部服务依赖

| 服务 | GroupId | ArtifactId | 版本 | 用途 |
|------|----------|------------|------|------|
| MISGWSAPI | com.ias.lib.mis | MISGWSAPI | 5.0.3.0 | 用户、部门、岗位服务 |
| Security API | com.ias.lib | security-api | 5.0.2.0 | 安全认证服务 |
| GlobalSequence | com.ias.agdis | globalsequence-api | 5.0.0.1 | 序列号生成 |

### 2.3 架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           技术架构图                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                        客户端层                                       │    │
│  │   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │    │
│  │   │ 调度终端  │  │  PAD终端  │  │ 客户端Web │  │ 移动端   │       │    │
│  │   └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘       │    │
│  └─────────┼─────────────┼─────────────┼─────────────┼───────────────┘    │
│            │             │             │             │                      │
│            └──────┬──────┴─────┬──────┴──────┬──────┘                      │
│                   │             │             │                             │
│                   ▼             ▼             ▼                             │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                        Dubbo RPC 层                                  │    │
│  │                                                                      │    │
│  │   ┌─────────────────────────────────────────────────────────────┐   │    │
│  │   │  ILoginWebservice  │  TaskAllot  │  VehicleInterface     │   │    │
│  │   └─────────────────────────────────────────────────────────────┘   │    │
│  │                              │                                      │    │
│  │                              ▼                                      │    │
│  │   ┌─────────────────────────────────────────────────────────────┐   │    │
│  │   │              LoginWebserviceImpl (Spring Bean)             │   │    │
│  │   └─────────────────────────────────────────────────────────────┘   │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                      │                                      │
│                                      ▼                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                        业务逻辑层 (Service)                         │    │
│  │                                                                      │    │
│  │   ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐  │    │
│  │   │LoginDetailImpl│ │VehicleBindImpl│ │TaskAllotServiceImpl    │  │    │
│  │   └──────────────┘  └──────────────┘  └────────────────────────┘  │    │
│  │                                                                      │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                      │                                      │
│                                      ▼                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                        数据访问层 (DAO)                             │    │
│  │   ┌─────────────────────────────────────────────────────────────┐   │    │
│  │   │              Spring Data JPA Repository                      │   │    │
│  │   └─────────────────────────────────────────────────────────────┘   │    │
│  │                              │                                      │    │
│  │                              ▼                                      │    │
│  │   ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐  │    │
│  │   │LoginDetail    │  │ManCarBinding │  │DispatchLogin           │  │    │
│  │   │Repository     │  │Repository    │  │Repository              │  │    │
│  │   └──────────────┘  └──────────────┘  └──────────────────────────┘  │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                      │                                      │
│                                      ▼                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                        消息队列层 (Camel/ActiveMQ)                   │    │
│  │                                                                      │    │
│  │   ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐  │    │
│  │   │  seda:online │  │seda:offline  │  │seda:datamanager_bind   │  │    │
│  │   └──────────────┘  └──────────────┘  └──────────────────────────┘  │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                      │                                      │
│                                      ▼                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                        外部服务集成                                  │    │
│  │                                                                      │    │
│  │   ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐  │    │
│  │   │  MIS用户服务  │  │  Security认证 │  │  GlobalSequence        │  │    │
│  │   │  (WebService)│  │  (Dubbo)     │  │  (Dubbo)               │  │    │
│  │   └──────────────┘  └──────────────┘  └──────────────────────────┘  │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. 核心API接口

### 3.1 ILoginWebservice (登录服务)

| 方法 | 参数 | 返回 | 说明 |
|------|------|------|------|
| `login(userId, password)` | 用户ID, 密码 | LoginResult | 调度客户端登录 |
| `terminallogin(userId, password, ssn)` | 用户ID, 密码, 终端号 | LoginResult | PAD终端登录 |
| `terminalLoginRestful(userId, password, ssn, userids, usernames)` | 用户ID, 密码, 终端号, 多人信息 | LoginResult | REST终端登录 |
| `logout(userId, session_id)` | 用户ID, 会话ID | LoginResult | 登出 |
| `updateOnlineState(userId, state)` | 用户ID, 状态(Y/N) | LoginResult | 更新在线状态 |
| `getPermissonsBySysname(sysname, userid, session)` | 系统名, 用户ID, 会话 | Permissions | 获取权限 |
| `getUsersPermissons(sysname, managerid, managerpass, userid)` | 系统名, 管理员ID, 密码, 用户ID | Permissions | 获取用户权限 |

### 3.2 IDispatchLoginService (调度登录服务)

| 方法 | 参数 | 返回 | 说明 |
|------|------|------|------|
| `insertDispatchLogin(record)` | 登录记录 | int | 新增登录记录 |
| `getDispatchLogins(record)` | 登录记录 | List\<Dispatchlogin\> | 查询登录记录 |
| `getDispatchLoginByUserid(userid)` | 用户ID | Dispatchlogin | 按用户ID查询 |
| `updateDispatchLoginByUserid(record)` | 登录记录 | int | 更新登录记录 |
| `getClientPositions(psid)` | 岗位ID | List\<Dispatchlogin\> | 获取岗位用户 |
| `dispatchLoginLogout(dispatchlogin)` | 登录记录 | void | 处理登出 |

### 3.3 VehicleInterface (车辆绑定服务)

| 方法 | 参数 | 返回 | 说明 |
|------|------|------|------|
| `bindVnbUsno(req)` | 绑定请求 | BindResp | 用户绑定车辆 |
| `unbindVnb(req)` | 解绑请求 | UnbindResp | 用户解绑车辆 |
| `getBindToVnb(usno, rid, vnb)` | 用户/资源/车号 | List\<ManCarBinding\> | 查询绑定关系 |
| `getBindByUsno(usno)` | 用户ID | String | 获取用户绑定的车 |

### 3.4 TaskAllot (任务分配服务)

| 方法 | 参数 | 返回 | 说明 |
|------|------|------|------|
| `findUser(req)` | 查找用户请求 | FindUserResp | 按条件查找用户 |
| `findHandOverUser(req)` | 移交用户请求 | FindUserResp | 查找可移交用户 |

---

## 4. 核心数据模型

### 4.1 Dispatchlogin (调度登录实体)

```java
public class Dispatchlogin implements Serializable {
    private String tenant;          // 租户
    private String userid;         // 用户ID (工号)
    private String username;       // 用户姓名
    private String loginresource;   // 登录资源 (终端号/CLIENT)
    private String psid;            // 岗位ID
    private String psnm;           // 岗位名称
    private String dpid;           // 部门ID
    private String dpnm;           // 部门名称
    private String tmid;          // 班组ID
    private String tmnm;          // 班组名称
    private String onln;           // 在线状态 (Y/N)
    private Date tonl;            // 登录时间
    private Date toff;            // 登出时间
    private Date cdat;            // 创建时间
    private Date udat;            // 更新时间
    private String remk;          // 备注
    private String userids;        // 多人用户ID
    private String usernames;     // 多人用户姓名
}
```

### 4.2 LoginResult (登录结果)

```java
public class LoginResult implements Serializable {
    private String result;            // 结果 (S/F)
    private String ems;               // 错误消息
    private String err;               // 错误码
    private String stat;              // 状态
    private String security_session_id;// 安全会话ID
    private List<Userservicerelation> userserviceList; // 用户服务列表
    private Permissions permissions;   // 权限列表
    private UserInfo userinfo;        // 用户信息
    private String rtl;               // ?
    private String num;               // 数量
    private String txl;               // 权限文本长度
    private String txt;               // 权限JSON文本
    private String ttr;               // ?
    private String vnb;               // 车号
    private String kmt;               // 考勤内容
    private String lct;               // 同组员工
    private Authority authority;     // 权限详情
    private FTPServerInfoBean ftpserverinfo; // FTP服务器信息
}
```

### 4.3 ManCarBinding (人车绑定)

```java
public class ManCarBinding implements Serializable {
    private long mcbnb;           // 绑定ID
    private String usno;          // 用户ID
    private String usnm;          // 用户姓名
    private String venumber;      // 车号
    private String bdat;          // 绑定日期
    private String flag;          // 标记 (binding/unbinding)
    private String tenant;        // 租户
}
```

### 4.4 LoginDetail (登录详情)

```java
public class LoginDetail implements Serializable {
    private long no;             // 序号
    private String enb;          // 员工号
    private String enm;          // 员工姓名
    private String rid;          // 资源ID
    private String vnb;          // 车号
    private String stat;         // 状态
    private String onln;         // 在线状态
    private String addr;         // 地址
    private String litm;         // 登录IP
    private String lotm;         // 登出IP
    private Date tonl;           // 登录时间
    private Date toff;           // 登出时间
    private String tdrp;         // 掉线时间
    private String tote;         // 在线时长
    private String luts;         // 时间戳
    private String remk;         // 备注
    private String tenant;       // 租户
}
```

---

## 5. 核心业务逻辑详解

### 5.1 登录流程 (terminallogin)

```java
public LoginResult terminallogin(String userId, String password, String ssn) {
    // 1. 调用Security服务验证用户
    B2BRequest loginRequest = new B2BRequest();
    loginRequest.setUserId(userId);
    loginRequest.setPassword(password);
    loginRequest.setSystemId("TERM");
    B2BCustomResponse response = authServiceRpc.login(loginRequest);
    
    // 2. 获取设备信息
    CDevice device = getDeviceinfo(ssn);
    
    // 3. 验证设备有效性
    if (response.getReturnCode() == 0 && device != null) {
        result.setStat("success");
        result.setSecurity_session_id(response.getUser().getSecuritySession());
    } else {
        result.setStat("false");
        result.setEms("设备编号【" + ssn + "】不存在，为无效编号。");
        return result;
    }
    
    // 4. 获取用户详细信息
    CUsers cuser = getUserinfo(response.getUser().getEmployeeId());
    // 获取岗位信息
    CPosition position = getPosition(cuser.getPsid());
    // 获取部门信息
    CDepartment department = getDepartment(cuser.getDpid());
    
    // 5. 构建Dispatchlogin记录
    dispatchlogin.setUserid(userId);
    dispatchlogin.setUsername(response.getUser().getUserName());
    dispatchlogin.setPsid(cuser.getPsid());
    dispatchlogin.setPsnm(position.getFullname());
    dispatchlogin.setDpid(cuser.getDpid());
    dispatchlogin.setDpnm(department.getFullname());
    dispatchlogin.setOnln("N");  // PAD默认离线状态
    dispatchlogin.setLoginresource(ssn);
    dispatchlogin.setTonl(new Date());
    
    // 6. 保存登录记录 (存在则更新)
    Dispatchlogin returnLogin = dispatchLogin.getDispatchloginByUserid(userId);
    if (returnLogin == null) {
        dispatchLogin.insertDispatchLogin(dispatchlogin);
    } else {
        dispatchLogin.updateDispatchLoginByUserid(dispatchlogin);
    }
    
    // 7. 获取绑定车辆信息
    BindRequest request = new BindRequest();
    request.setUsno(userId);
    BindResponse bindResponse = vehicleRecordImpl.getBindMessage(request);
    if (bindResponse != null) {
        result.setVnb(bindResponse.getRecords().get(0).getVenumber());
    }
    
    // 8. 返回FTP配置信息
    FTPServerInfoBean ftpserverinfo = new FTPServerInfoBean();
    ftpserverinfo.setPubip(public_ip);
    ftpserverinfo.setPriip(private_ip);
    result.setFtpserverinfo(ftpserverinfo);
    
    return result;
}
```

### 5.2 车辆绑定流程 (bindVnbUsno)

```java
public BindResp bindVnbUsno(BindReq req) {
    String usno = req.getUsno();
    String vnb = req.getVnb();
    
    // 1. 校验车号是否存在
    if (!checkVnb(vnb)) {
        bindResp.setEms("车号不存在");
        return bindResp;
    }
    
    // 2. 查询当前用户已绑定车辆
    List<ManCarBinding> manBounds = getBindToVnb(usno, "", "");
    
    // 3. 判断是否为强制绑定
    if (boundSets.size() > 0 && "N".equalsIgnoreCase(req.getForceFlag())) {
        // 非强制绑定，返回提示信息
        bindResp.setEms("您已经绑定车辆【"+ manBounds.get(0).getVenumber() +"】，是否重新绑定？");
        return bindResp;
    }
    
    // 4. 解绑已绑定的其他车辆
    for (ManCarBinding bound : boundSets) {
        unBindToVnb(bound);
    }
    
    // 5. 添加新绑定关系
    ManCarBinding binding = new ManCarBinding();
    binding.setUsno(usno);
    binding.setUsnm(req.getUsnm());
    binding.setVenumber(vnb);
    bindToVnb(binding);
    
    return bindResp;
}

private void bindToVnb(ManCarBinding binding) {
    // 1. 生成序列号
    long mcbnb = sequenceService.getSequence(SequenceConfig.MAN_CAR_BINDING_MCBNB);
    binding.setMcbnb(mcbnb);
    
    // 2. 获取租户信息
    String tenant = tenantService.getTenantIdByUsno(binding.getUsno());
    binding.setTenant(tenant);
    
    // 3. 保存绑定主表
    manCarBindingRepository.save(entity);
    
    // 4. 生成绑定明细
    long mcsnb = sequenceService.getSequence("MAN_CAR_BINDING_DETAIL_MCSNB");
    ManCarBindingDetailEntity bindDetail = new ManCarBindingDetailEntity();
    bindDetail.setMcsnb(mcsnb);
    bindDetail.setTenant(tenant);
    bindDetail.setUsno(binding.getUsno());
    bindDetail.setVenumber(binding.getVenumber());
    bindDetail.setFlag("binding");
    manCarBindingDetailRepository.save(bindDetail);
    
    // 5. 发送绑定消息到消息队列
    VehicleRecord record = new VehicleRecord();
    record.setFlag("Y");  // Y=绑定
    record.setTenant(tenant);
    record.setUsno(binding.getUsno());
    record.setVnb(binding.getVenumber());
    camelTemplate.sendBody("seda:datamanager_bindtovnb", record);
}
```

### 5.3 超时登出任务 (TimeoutLogoutJob)

```java
@Service
public class TimeoutLogoutJob {
    
    @Scheduled(cron = "0 0/${logintimeout} * * * ?")
    public void process() {
        // 1. 检查调度客户端登录超时
        dispatchLoginLogout();
        
        // 2. 检查终端注册资源超时
        List<Registeredresource> timeoutregresources = getTriggerTimeoutTerminal();
        for (Registeredresource regresource : timeoutregresources) {
            if ("Y".equalsIgnoreCase(regresource.getOnln())) {
                // 3. 执行离线处理
                this.offline(userid);
                this.offline(userid, rid);
            }
        }
    }
    
    private void dispatchLoginLogout() {
        List<DispatchLoginEntity> timeoutusers = commonRepository.getTimeoutTerminalUsers();
        for (DispatchLoginEntity dispatchlogin : timeoutusers) {
            if ("Y".equalsIgnoreCase(dispatchlogin.getOnln())) {
                // 1. 更新登录状态为离线
                Dispatchlogin record = new Dispatchlogin();
                record.setUserid(dispatchlogin.getUserid());
                record.setOnln("N");
                record.setToff(new Date());
                record.setRemk("用户掉线");
                dispatchLogin.updateDispatchLoginByUserid(record);
                
                // 2. 发送登出消息到Camel
                ReliEnvelope logoutenvelope = new ReliEnvelope();
                logoutenvelope.setMt("RELI");
                logoutenvelope.setRsn(dispatchlogin.getLoginresource());
                logoutenvelope.setRen(dispatchlogin.getUserid());
                logoutenvelope.setPayload(logoutresult);
                camelTemplate.sendBody("seda:offline", logoutenvelope);
            }
        }
    }
}
```

### 5.4 任务分配查询 (findUser)

```java
public FindUserResp findUser(FindUserReq req) {
    String userid = req.getSen();
    
    // 1. 获取用户所属组的成员
    List<CUsers> cusers = misService.getUsersByGroup(userid);
    
    // 2. 遍历成员，检查在线状态
    for (CUsers cu : cusers) {
        Dispatchlogin record = new Dispatchlogin();
        record.setUserid(cu.getUsno());
        record.setOnln("Y");
        List<Dispatchlogin> loginUsers = dispatchloginService.getDispatchlogins(record);
        
        if (loginUsers != null && loginUsers.size() > 0) {
            // 用户在线
            User u = new User();
            u.setEnb(cu.getUsno());        // 员工号
            u.setEnm(cu.getName());         // 姓名
            u.setSnb(one.getLoginresource()); // 终端资源号
            u.setOnl("Y");                  // 在线状态
            usersY.add(u);
        } else {
            // 用户离线
            User u = new User();
            u.setEnb(cu.getUsno());
            u.setOnl("N");
            usersN.add(u);
        }
    }
    
    // 3. 根据请求返回结果
    String onl = req.getOnl();
    if (onl == null) {
        resp.setUsers(users);        // 返回全部
    } else if ("Y".equalsIgnoreCase(onl)) {
        resp.setUsers(usersY);       // 只返回在线
    } else if ("N".equalsIgnoreCase(onl)) {
        resp.setUsers(usersN);       // 只返回离线
    }
    
    return resp;
}
```

---

## 6. 数据访问层

### 6.1 Repository列表

| Repository | 实体 | 说明 |
|------------|------|------|
| LoginDetailRepository | LoginDetailEntity | 登录详情 |
| DispatchloginRepository | DispatchLoginEntity | 调度登录 |
| RegisteredresourceRepository | RegisteredResourceEntity | 注册资源 |
| ManCarBindingRepository | ManCarBindingEntity | 人车绑定 |
| ManCarBindingDetailRepository | ManCarBindingDetailEntity | 绑定明细 |
| UserServiceRelationRepository | UserServiceRelationEntity | 用户服务关联 |
| CommonRepository | - | 通用查询 |

### 6.2 典型查询方法

```java
// LoginDetailRepository
List<LoginDetailEntity> findByEnb(String enb);
void deleteByRidAndEnb(String rid, String enb);

// DispatchloginRepository
DispatchLoginEntity findByUserid(String userid);
List<DispatchLoginEntity> findByOnln(String onln);
List<DispatchLoginEntity> findByPsidAndOnln(String psid, String onln);

// ManCarBindingRepository
List<ManCarBindingEntity> findByUsno(String usno);
void deleteByVenumberAndUsno(String venumber, String usno);

// CommonRepository
@Query("SELECT d FROM DispatchLoginEntity d WHERE d.onln = 'Y' AND d.tonl < :timeout")
List<DispatchLoginEntity> getTimeoutTerminalUsers();
```

---

## 7. 消息队列集成

### 7.1 Camel路由配置

```java
// 绑定消息路由
camelTemplate.sendBody("seda:datamanager_bindtovnb", record);

// 解绑消息路由
camelTemplate.sendBody("seda:datamanager_unbindtovnb", record);

// 上线消息路由
camelTemplate.sendBody("seda:online", envelope);

// 离线消息路由
camelTemplate.sendBody("seda:offline", logoutenvelope);
```

### 7.2 消息模板

| 路由名称 | 用途 | 消息类型 |
|----------|------|----------|
| seda:datamanager_bindtovnb | 车辆绑定通知 | VehicleRecord |
| seda:datamanager_unbindtovnb | 车辆解绑通知 | UnBindNotice |
| seda:online | 用户上线 | ReliEnvelope |
| seda:offline | 用户离线 | ReliEnvelope |

### 7.3 FreeMarker消息模板

```
templates/
├── reli/
│   ├── online-pub.ftl      # 上线发布模板
│   ├── offline-pub.ftl     # 离线发布模板
│   ├── reli-pub.ftl        # 可靠消息模板
│   ├── reli-down.ftl      # 下行模板
│   └── relo-pub.ftl       # 重连模板
└── ucbd/
    ├── ucbd-notice.ftl     # UCBD通知
    └── ucub-down.ftl      # 下行模板
```

---

## 8. 配置信息

### 8.1 Dubbo服务暴露

```xml
<!-- 登录服务 -->
<dubbo:service interface="com.ias.lib.resource.api.ILoginWebservice" 
               ref="loginWebservice" />

<!-- 任务分配服务 -->
<dubbo:service interface="com.ias.lib.resource.api.TaskAllot" 
               ref="taskAllot" />

<!-- 车辆绑定服务 -->
<dubbo:service interface="com.ias.lib.resource.api.VehicleInterface" 
               ref="vehicleInterface" />

<!-- 数据存储服务 -->
<dubbo:service interface="com.ias.lib.resource.api.IDataStore" 
               ref="dataStore" />
```

### 8.2 应用配置 (application.properties)

```properties
# MIS配置
mis_user=admin
mis_password=xxxxxx

# Dubbo配置
dubbo.address=172.16.10.111:2181
dubbo.port=10881

# FTP配置
public_network_ip=xxx.xxx.xxx.xxx
private_network_ip=xxx.xxx.xxx.xxx
port_number=21
network_ip_username=ftpuser
network_ip_password=ftppass
guest_content=考勤内容

# 超时配置
terminal.timeout=300  # 秒
logintimeout=5        # 分钟
```

---

## 9. 外部服务集成

### 9.1 MIS服务集成

```java
@Resource
private UsersService misuserService;      // 用户服务
@Resource
private PositionService positionService;  // 岗位服务
@Resource
private DepartmentService departmentService; // 部门服务
@Resource
private DeviceService deviceService;       // 设备服务
@Resource
private VehicleService vehicleService;     // 车辆服务
@Resource
private TenantService tenantService;       // 租户服务
```

### 9.2 Security服务集成

```java
@Resource
private IAuthService authServiceRpc;     // 安全认证服务

// 调用示例
B2BCustomResponse response = authServiceRpc.login(loginRequest);
List<String> permissions = response.getUser().getPermissionList();
```

### 9.3 GlobalSequence服务集成

```java
@Resource
private ISequenceService sequenceService;

// 生成序列号
long no = sequenceService.getSequence(SequenceConfig.Login_Detail);
long mcbnb = sequenceService.getSequence(SequenceConfig.MAN_CAR_BINDING_MCBNB);
```

---

## 10. 业务场景分析

### 10.1 PAD终端登录场景

```
┌─────────────────────────────────────────────────────────────────────┐
│                      PAD终端登录序列图                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  PAD         Resource模块       Security      MIS      数据库         │
│   │               │                │           │          │          │
│   │  1.login()   │                │           │          │          │
│   │──────────────▶│                │           │          │          │
│   │               │ 2.login()      │           │          │          │
│   │               │───────────────▶│           │          │          │
│   │               │                │ 3.验证用户 │          │          │
│   │               │                │──────────▶│          │          │
│   │               │                │           │ 4.查询设备 │          │
│   │               │                │◀──────────│          │          │
│   │               │◀──────────────│           │          │          │
│   │               │                │           │          │          │
│   │               │ 5.getDevice() │           │          │          │
│   │               │───────────────────────────────────────────▶        │
│   │               │                │           │          │          │
│   │               │ 6.getUsers()  │           │          │          │
│   │               │───────────────────────────────────────────▶        │
│   │               │                │           │          │          │
│   │               │ 7.getPosition()            │          │          │
│   │               │───────────────────────────────────────────▶        │
│   │               │                │           │          │          │
│   │               │ 8.getDepartment()          │          │          │
│   │               │───────────────────────────────────────────▶        │
│   │               │                │           │          │          │
│   │               │ 9.save/login() │           │          │          │
│   │               │───────────────────────────────────────────▶        │
│   │               │                │           │          │          │
│   │               │ 10.getBind()  │           │          │          │
│   │               │──────────────────────────────────────────────────▶ │
│   │               │                │           │          │          │
│   │◀──────────────│ 11.loginResp │           │          │          │
│   │               │                │           │          │          │
└─────────────────────────────────────────────────────────────────────┘
```

### 10.2 车辆绑定流程

```
┌─────────────────────────────────────────────────────────────────────┐
│                      车辆绑定流程图                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  用户          Resource模块        MIS        消息队列   终端        │
│   │               │                │           │          │         │
│   │ 1.bind()     │                │           │          │         │
│   │──────────────▶│                │           │          │         │
│   │               │ 2.checkVnb()  │           │          │         │
│   │               │───────────────────────────────────▶│          │
│   │               │                │           │          │         │
│   │               │ 3.getBind()   │           │          │         │
│   │               │───────────────────────────────────────────▶     │
│   │               │                │           │          │         │
│   │               │ 4.unbind() (if exists)  │          │         │
│   │               │───────────────────────────────────────────▶     │
│   │               │                │           │          │         │
│   │               │ 5.save_bind() │           │          │         │
│   │               │───────────────────────────────────────────▶     │
│   │               │                │           │          │         │
│   │               │ 6.send_bind_msg           │          │         │
│   │               │──────────────────────────────────────────▶│      │
│   │               │                │           │          │         │
│   │◀──────────────│ 7.bindResp   │           │          │         │
│   │               │                │           │          │         │
│   │               │                │           │ ▼         │         │
│   │               │                │           │ datamanager_bind    │
│   │               │                │           │──────────▶│         │
│   │               │                │           │          │         │
└─────────────────────────────────────────────────────────────────────┘
```

### 10.3 超时检测与登出

```
┌─────────────────────────────────────────────────────────────────────┐
│                      超时检测定时任务                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────┐                                                   │
│  │TimeoutLogoutJob│                                                  │
│  │  (每5分钟)   │                                                   │
│  └──────┬──────┘                                                   │
│         │                                                          │
│         ▼                                                          │
│  ┌──────────────────────────────┐                                 │
│  │ 1. 查询超时的调度客户端登录记录  │                                 │
│  │    (tonl < 当前时间 - timeout)│                                 │
│  └──────┬───────────────────────┘                                 │
│         │                                                          │
│         ▼                                                          │
│  ┌──────────────────────────────┐                                 │
│  │ 2. 更新登录状态: onln='N'     │                                 │
│  │    toff=当前时间             │                                 │
│  └──────┬───────────────────────┘                                 │
│         │                                                          │
│         ▼                                                          │
│  ┌──────────────────────────────┐                                 │
│  │ 3. 查询超时的终端注册资源      │                                 │
│  │    (luts < 当前时间 - timeout)│                                 │
│  └──────┬───────────────────────┘                                 │
│         │                                                          │
│         ▼                                                          │
│  ┌──────────────────────────────┐                                 │
│  │ 4. 更新资源状态: onln='N'     │                                 │
│  │    tdrp=当前时间              │                                 │
│  └──────┬───────────────────────┘                                 │
│         │                                                          │
│         ▼                                                          │
│  ┌──────────────────────────────┐                                 │
│  │ 5. 发送离线消息 (Camel)       │                                 │
│  │    seda:offline             │                                 │
│  └──────────────────────────────┘                                 │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 11. 问题与优化建议

### 11.1 技术债务

| 问题 | 严重程度 | 说明 | 建议 |
|------|----------|------|------|
| Spring Boot 1.5.3 过时 | 🔴 高 | 2017年发布，存在安全漏洞 | 升级到 2.7.x |
| Jackson 1.9.11 过时 | 🔴 高 | 已被Jackson 2.x替代 | 升级到 jackson-databind |
| Dubbo 2.5.3 过时 | 🟡 中 | 建议升级到 2.7.x 或 3.x | 考虑升级 |
| Java 1.7 已停止支持 | 🟡 中 | 建议升级到 Java 8+ | 升级到 1.8 |
| 代码中有大量注释掉的旧代码 | 🟢 低 | 影响代码可读性 | 清理废弃代码 |

### 11.2 性能问题

| 问题 | 描述 | 建议 |
|------|------|------|
| 循环中调用远程服务 | findUser方法中多次调用misService | 使用批量查询 |
| 超时检测全表扫描 | CommonRepository.getTimeoutTerminalUsers | 添加索引 |
| 无缓存机制 | 用户信息、绑定关系每次都查数据库 | 引入Redis缓存 |

### 11.3 安全问题

| 问题 | 描述 | 建议 |
|------|------|------|
| 密码明文传输 | login方法中密码参数明文 | 使用HTTPS或加密传输 |
| 无登录验证码 | 容易被暴力破解 | 添加验证码或限制登录次数 |
| 序列号生成依赖外部服务 | GlobalSequence服务不可用时影响业务 | 添加本地缓存或降级方案 |

### 11.4 代码质量问题

```java
// 问题1: 魔法数字
if (response.getReturnCode() == 0) { ... }

// 问题2: 异常吞掉
} catch (Exception e) {
    e.printStackTrace();  // 应该使用logger
}

// 问题3: 字符串拼接
String errmsg = "您已经绑定车辆【"+ manBounds.get(0).getVenumber() +"】...";

// 问题4: 过时的注释
// import com.misgws.base.DepartmentResponse;
// 这个import实际上没有被移除
```

### 11.5 建议优化代码

```java
// 优化1: 使用常量
private static final int RETURN_CODE_SUCCESS = 0;
if (response.getReturnCode() == RETURN_CODE_SUCCESS) { ... }

// 优化2: 使用Logger
catch (Exception e) {
    logger.error("绑定失败", e);
    unbindResp.setErr("20101");
    unbindResp.setEms("未知错误，请稍后重试");
}

// 优化3: 使用StringBuilder
StringBuilder errmsg = new StringBuilder();
errmsg.append("您已经绑定车辆【")
      .append(manBounds.get(0).getVenumber())
      .append("】，是否重新绑定？");
```

---

## 12. 总结

### 12.1 模块特点

| 特点 | 说明 |
|------|------|
| **Dubbo服务提供方** | 提供登录、任务分配、车辆绑定等核心服务 |
| **多系统集成** | 集成MIS用户服务、Security认证、GlobalSequence序列 |
| **消息队列集成** | 使用Camel路由 + ActiveMQ实现异步消息 |
| **定时任务** | 自动检测超时登出、离线通知 |
| **Spring Data JPA** | 使用JPA Repository简化数据访问 |

### 12.2 核心能力

1. **用户认证**：支持调度客户端、PAD终端、移动端等多终端登录
2. **在线状态管理**：实时跟踪用户在线状态，自动处理超时登出
3. **车辆绑定管理**：支持人车绑定/解绑、强制绑定模式
4. **任务分配支持**：按组、按权限查找可分配用户
5. **消息通知**：通过Camel/ActiveMQ推送上下线消息

### 12.3 依赖关系

```
resourceparent
├── resourceapi (API定义)
│   ├── ILoginWebservice ←─┐
│   ├── TaskAllot ←───────┤
│   ├── VehicleInterface ←─┤
│   └── IDispatchLoginService
│
└── login (服务实现)
    ├── Security API (认证)
    ├── MISGWSAPI (用户/部门/岗位)
    └── GlobalSequence API (序列号)
```

### 12.4 后续建议

1. **框架升级**：Spring Boot 1.5 → 2.7，Jackson 1.x → 2.x
2. **缓存优化**：引入Redis缓存用户信息、绑定关系
3. **安全增强**：HTTPS传输、登录限流、密码加密
4. **代码清理**：移除废弃代码、统一错误处理
5. **监控完善**：添加业务指标监控、性能告警

---

*报告生成时间：2026-02-04*
*分析工具：Cursor AI Code Analysis*
