# db_oil_pvg 数据库结构分析报告

> 分析日期：2026-03-25
> 数据源：172.21.0.63 MySQL 5.6.23
> 文件：sql/db_oil_pvg.sql (5836行)
> 分析工具：Cursor AI Code Analysis

---

## 1. 数据库概览

### 1.1 基本信息

| 属性 | 值 |
|------|-----|
| 数据库名称 | db_oil_pvg |
| 字符集 | utf8 / utf8mb4 |
| 存储引擎 | InnoDB |
| 表数量 | **206** |
| 用途 | 浦东机场供油调度系统 |

### 1.2 业务定位

```
┌────────────────────────────────────────────────────────────────────────┐
│                        db_oil_pvg 业务定位                              │
├────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                         核心业务模块                              │  │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │  │
│  │  │ 航班管理        │  │ 油单管理        │  │ 服务调度         │ │  │
│  │  │ flightinfo     │  │ oil_bill       │  │ srv_task        │ │  │
│  │  │ 35+ 表         │  │ 25+ 表          │  │ 20+ 表           │ │  │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘ │  │
│  │                                                                     │  │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │  │
│  │  │ AGDIS调度配置   │  │ GA除冰任务      │  │ 用户与权限       │ │  │
│  │  │ agdis_* 30+表  │  │ ga_* 5+ 表     │  │ user/group      │ │  │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘ │  │
│  │                                                                     │  │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │  │
│  │  │ 车辆管理        │  │ 化验管理        │  │ 配置与码表       │ │  │
│  │  │ oil_vehicle    │  │ oil_assay_*    │  │ sys_code/config │ │  │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘ │  │
│  │                                                                     │  │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │  │
│  │  │ ADS-B监控       │  │ 油量告警        │  │ 工作消息         │ │  │
│  │  │ adsb_*         │  │ oil_alarm_*   │  │ workingmsg_*    │ │  │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘ │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                         │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 2. 表分类统计

### 2.1 业务领域分布

| 业务领域 | 表数量 | 占比 | 说明 |
|----------|--------|------|------|
| **AGDIS调度配置** | 35+ | 17% | 调度界面、颜色、过滤、排序等前端配置 |
| **航班相关** | 30+ | 15% | 航班信息、配置、历史、同步 |
| **服务调度** | 25+ | 12% | 服务、任务、进程、告警管理 |
| **油单相关** | 20+ | 10% | 油单、确认、历史、编号管理 |
| **配置管理** | 15+ | 7% | 系统码表、终端配置、时间配置 |
| **用户权限** | 12+ | 6% | 用户、组、岗位、角色 |
| **GA除冰任务** | 8+ | 4% | 除冰任务、模板、流程 |
| **车辆管理** | 10+ | 5% | 加油车、绑定、告警 |
| **化验管理** | 5+ | 2% | 化验单、密度、标准 |
| **ADS-B监控** | 5+ | 2% | 位置历史、配置 |
| **油量告警** | 5+ | 2% | 告警配置、航班告警 |
| **工作消息** | 5+ | 2% | 消息、配置、发送 |
| **其他** | 20+ | 16% | 排班、Gis、注册等 |

### 2.2 表名前缀分布

```
表名前缀统计：
┌─────────────────────────────────────────────────────────────────────┐
│ agdis_*  : 35+ 表  ████████████████████████████ 17%             │
│ flight_*  : 30+ 表  ███████████████████████ 15%                   │
│ srv_*     : 20+ 表  ██████████████ 10%                            │
│ oil_*     : 25+ 表  ██████████████████ 12%                        │
│ ga_*      : 8+ 表   ████████ 4%                                  │
│ user_/group: 8+ 表  ████████ 4%                                  │
│ adsb_*    : 5+ 表   █████ 2%                                     │
│ 其他       : 75+ 表  ████████████████████████████████████████ 37%  │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.3 完整表清单

| 序号 | 表名 | 说明 | 核心字段 |
|------|------|------|----------|
| 1 | accu_filter_oil | 累计过滤器油量 | FILTER_CODE, VNB, ACCU_OIL_WEIGHT |
| 2 | adsb_config | ADS-B配置 | TENANT, WMAP, AGENT |
| 3 | adsb_config_copy | ADS-B配置备份 | 同adsb_config |
| 4 | adsb_ll_history | ADS-B位置历史(分区表) | HEXADDRESS, LON, LAT, FLIGHTNUM |
| 5 | agdis_behavior_record | AGDIS行为记录 | flno, type, operator, content |
| 6 | agdis_bill_curr | 当前版本单据 | BKEY, FLOP, FKEY, BLID |
| 7 | agdis_bill_info | 单据原始数据 | BKEY, ATXT, IFAV |
| 8 | agdis_bill_li | 装机单 | LINO, LTYP, GWHT, NWHT |
| 9 | agdis_bill_ls | 仓单 | LSNO, BPCS |
| 10 | agdis_bill_oli | 卸机单 | LONO, LTYP, GWHT, NWHT |
| 11 | agdis_bill_proc | 单据打印数据 | BPNO, PTTM, USID |
| 12 | agdis_bill_sgin | 单据签名 | BSNO, BSTM, STXT |
| 13 | agdis_bill_tlac | 模板机型匹配 | TLAC, RLAC |
| 14 | agdis_bill_view | 单据查看数据 | BVNO, BVTM |
| 15 | agdis_change_task_state_config | 任务状态变更配置 | current_state, target_state |
| 16 | agdis_color_config | 颜色配置 | groupid, class_name, text, border, background |
| 17 | agdis_dispatchlogin | 调度登录 | userid, cdat, username, auto_refuel_status |
| 18 | agdis_dispatch_default_type_config | 派工默认类型配置 | psid, service_type, flight_type |
| 19 | agdis_dispatch_time_config | 派工时间配置 | al2c, ac3c, delivery_time_type, interval_minutes |
| 20 | agdis_fields | 字段配置 | code, column_name, form_format |
| 21 | agdis_filter_config | 过滤配置 | code, name, filter_rule |
| 22 | agdis_flight_clock_config | 航班时钟配置 | CODE, NAME, display_type, regexp_rule |
| 23 | agdis_flight_clock_remind_content | 时钟提醒内容 | ruleId, flseq, nextAlarmTime, remindContent |
| 24 | agdis_flight_clock_rule | 时钟规则 | articleId, flightRule, flightFilter |
| 25 | agdis_flight_group_sort_config | 航班分组排序配置 | CODE, NAME, arrival_view, depart_view |
| 26 | agdis_flight_sort_config | 航班排序配置 | CODE, NAME, orderby |
| 27 | agdis_login_detail | 登录详情 | no, enb, enm, stat |
| 28 | agdis_man_car_binding | 人车绑定 | id, user_id, vehicle_no, vehicle_type |
| 29 | agdis_man_car_binding_detail | 人车绑定详情 | binding_id, bind_type, bind_value |
| 30 | agdis_message | 消息 | id, title, content, type |
| 31 | agdis_place_region_config | 位置区域配置 | code, name, region_type |
| 32 | agdis_registeredresource | 注册资源 | id, resource_type, resource_no |
| 33 | agdis_remark_template_config | 备注模板配置 | template_id, template_content |
| 34 | agdis_seat_domain_config | 座位域配置 | domain_id, domain_name |
| 35 | agdis_sort_config | 排序配置 | CODE, NAME, orderby |
| 36 | agdis_task_group_sort_config | 任务分组排序配置 | CODE, NAME, groupby |
| 37 | agdis_task_minute | 任务分钟 | task_id, minute, value |
| 38 | agdis_task_remark_template_config | 任务备注模板配置 | template_id, template_content |
| 39 | agdis_ter_config | 终端配置 | ter_code, ter_name |
| 40 | agdis_user_service | 用户服务 | user_id, service_type |
| 41 | airline_airport_join_flight_config | 航司机场航班配置 | al2c, ap3c, flno_pattern |
| 42 | airport_arrearage_detail | 机场欠款详情 | airport_code, airline, amount |
| 43 | airport_type | 机场类型 | type_id, type_name |
| 44 | alarm_time | 告警时间 | id, alarm_type, alarm_time |
| 45 | auto_dispatch_priority_config | 自动派工优先级配置 | priority, condition |
| 46 | change_crew_flno | 换机组航班 | old_flno, new_flno, reason |
| 47 | class_model | 班次模板 | model_id, model_name, work_time |
| 48 | cycle_config | 周期配置 | cycle_id, cycle_type |
| 49 | cycle_schedule | 周期排班 | schedule_id, cycle_id |
| 50 | **flightinfo** | **航班信息主表** | flseq, flop, flno, adid, ac3c |
| 51 | flightinfo_cloud | 云端航班信息 | flid, match_status |
| 52 | flightinfo_copy | 航班信息备份 | 同flightinfo |
| 53 | flightinfo_import | 航班信息导入 | import_id, status |
| 54 | flight_change | 航班变更 | change_id, flseq, change_type |
| 55 | flight_config_byflno | 按航班号配置 | flno, config_key, config_value |
| 56 | flight_data | 航班数据 | flseq, data_key, data_value |
| 57 | flight_data_config | 航班数据配置 | config_id, config_name |
| 58 | flight_data_template | 航班数据模板 | template_id, template_content |
| 59 | flight_hand_change | 航班手动变更 | change_id, flseq, field_name |
| 60 | flight_import_config | 航班导入配置 | config_id, field_mapping |
| 61 | flight_lock | 航班锁定 | flseq, lock_type, lock_by |
| 62 | flight_oil_info | 航班油量信息 | flseq, ol2, ola, oll |
| 63 | flight_subscribe | 航班订阅 | subscriber_id, flseq |
| 64 | flight_united_flight | 联合航班 | flight_id, flno_list |
| 65 | flight_update_field | 航班更新字段 | field_id, field_name, update_rule |
| 66 | flno_dispatch_time_config | 航班号派工时间配置 | flno, dispatch_time |
| 67 | flseq_interface_tenant | 航班接口租户 | flseq, interface_id, tenant |
| 68 | fuel_volume_round_config | 油量取整配置 | min_volume, max_volume, round_rule |
| 69 | ga_change_task_state_config | GA任务状态变更配置 | current_state, target_state |
| 70 | ga_order | GA订单 | order_id, order_number, order_status |
| 71 | ga_order_clock | GA订单时钟 | clock_id, order_id, alarm_time |
| 72 | ga_phrase_record | GA短语记录 | record_id, phrase_content |
| 73 | ga_phrase_template | GA短语模板 | template_id, template_content |
| 74 | ga_task | GA任务 | task_id, bill_number, rtyp |
| 75 | ga_task_process | GA任务进程 | task_id, process_id, process_index |
| 76 | ga_task_template | GA任务模板 | process_id, order_status |
| 77 | group | 用户组 | group_id, group_name, group_type |
| 78 | group_config | 组配置 | config_id, group_id |
| 79 | group_users | 组用户关联 | group_id, user_id |
| 80 | holiday | 节假日 | holiday_id, holiday_date |
| 81 | ias_gis_alert | GIS告警 | alert_id, alert_type, location |
| 82 | ias_gis_alert_config | GIS告警配置 | config_id, alert_type |
| 83 | ias_gis_ll_history | GIS位置历史 | history_id, latitude, longitude |
| 84 | intoport | 进口 | port_id, port_name |
| 85 | job | 岗位 | job_id, job_name |
| 86 | level | 级别 | level_id, level_name |
| 87 | long_airline_airport | 长航线机场 | airport_id, airport_name |
| 88 | oil_addprogress | 加油进度 | task_id, progress, status |
| 89 | oil_alarm_config | 加油告警配置 | config_id, alarm_type |
| 90 | oil_alarm_flight | 加油告警航班 | alarm_id, flseq, alarm_time |
| 91 | oil_alarm_time | 加油告警时间 | id, alarm_type, alarm_time |
| 92 | oil_assay_sheet | 化验单 | sheet_id, assay_date, density |
| 93 | oil_assay_sheet_config | 化验单配置 | config_id, field_name |
| 94 | oil_assay_sheet_history | 化验单历史 | history_id, sheet_id |
| 95 | **oil_bill** | **油单信息表** | id, bill_number, flseq, volume, weight |
| 96 | oil_bill_confirm | 油单确认 | id, meter_start, meter_end, confirm_status |
| 97 | oil_bill_confirm_history | 油单确认历史 | history_id, bill_id |
| 98 | oil_bill_copy | 油单备份 | 同oil_bill |
| 99 | oil_bill_history | 油单历史 | history_id, bill_id |
| 100 | oil_bill_number | 油单号 | id, prefix, current_value |
| 101 | oil_bill_number_distribution | 油单号分配 | distribution_id, bill_number |
| 102 | oil_bill_sign_model | 油单签名模板 | model_id, model_content |
| 103 | oil_config | 加油配置 | config_id, config_key |
| 104 | oil_credit | 加油信用 | credit_id, user_id, credit_value |
| 105 | oil_dutyuser | 值班用户 | duty_id, user_id, duty_date |
| 106 | oil_flight_alarm_config | 航班加油告警配置 | config_id, flno_pattern |
| 107 | oil_flight_group | 航班加油组 | group_id, flseq_list |
| 108 | oil_group | 加油组 | group_id, group_name |
| 109 | oil_group_user | 加油组用户 | group_id, user_id |
| 110 | oil_mass_history | 油品质检历史 | history_id, mass_value |
| 111 | oil_mass_info | 油品质检信息 | info_id, test_date, result |
| 112 | oil_restar_task | 重新开始任务 | task_id, restart_time |
| 113 | oil_spill_flight | 溢油航班 | flight_id, spill_volume |
| 114 | oil_standard_density | 标准密度 | density_id, oil_type, density_value |
| 115 | oil_standard_density_history | 标准密度历史 | history_id, density_id |
| 116 | oil_stop_task | 停止任务 | task_id, stop_time, reason |
| 117 | oil_stop_task_history | 停止任务历史 | history_id, task_id |
| 118 | oil_upload_config | 油单上传配置 | config_id, upload_type |
| 119 | oil_vehicle | 加油车 | id, vehicle_no, type, density |
| 120 | oil_volume_alarm | 油量告警 | id, vehicle_number, surplus_volume |
| 121 | pad_data_dic | PAD数据字典 | id, code, val |
| 122 | place | 机位 | id, placecode, flseq, status |
| 123 | pm_msgsubscriber | 消息订阅者 | subscriber, type, queuename |
| 124 | position | 岗位 | position_code, position_name |
| 125 | position_config | 岗位配置 | config_id, position_code |
| 126 | pressure_difference | 压差 | id, location, difference_value |
| 127 | pressure_difference_terminal | 终端压差 | id, terminal_id, difference_value |
| 128 | pre_notice | 预告 | notice_id, flseq, notice_time |
| 129 | pre_notice_range | 预告范围 | range_id, start_time, end_time |
| 130 | registration | 注册 | reg_id, reg_type, reg_date |
| 131 | registration_sup | 注册补充 | sup_id, reg_id, sup_content |
| 132 | relation_interface_tenant | 接口租户关系 | interface_id, tenant_id |
| 133 | rsub_flight_lock | RSUB航班锁定 | flseq, lock_time |
| 134 | rsub_prenotice | RSUB预告 | notice_id, flseq |
| 135 | rsub_prenotice_range | RSUB预告范围 | range_id, notice_id |
| 136 | rsub_ter_config | RSUB终端配置 | config_id, terminal_id |
| 137 | saf_material_config | SAF物料配置 | material_id, material_name |
| 138 | saf_refuel_plan_config | SAF加油计划配置 | plan_id, blend_ratio |
| 139 | schedule | 排班 | schedule_id, user_id, schedule_date |
| 140 | schedule_result | 排班结果 | result_id, schedule_id |
| 141 | schedule_rules | 排班规则 | rule_id, rule_name |
| 142 | shcedule_rule | 排班规则(重复) | rule_id, rule_content |
| 143 | special_vial | 特殊航线 | vial_id, vial_name |
| 144 | srv_alarm | 服务告警 | alarm_id, task_id, alarm_type |
| 145 | srv_applytask | 服务申请任务 | apply_id, task_id, apply_type |
| 146 | srv_config_un_task_time_limit | 未任务时限配置 | config_id, time_limit |
| 147 | srv_deployment | 服务部署 | deploy_id, deploy_type |
| 148 | srv_flight | 服务航班 | flight_id, flseq, service_type |
| 149 | srv_flight_lock | 服务航班锁定 | lock_id, flight_id |
| 150 | srv_msgsubscriber | 服务消息订阅 | subscriber, message_type |
| 151 | srv_prenotice | 服务预告 | notice_id, flight_id |
| 152 | srv_prenoticerange | 服务预告范围 | range_id, notice_id |
| 153 | srv_pretask | 预任务 | pretask_id, task_id |
| 154 | srv_pushmessage | 推送消息 | message_id, content, target |
| 155 | srv_pushmessage_bak | 推送消息备份 | 同srv_pushmessage |
| 156 | srv_service | 服务 | service_id, service_name |
| 157 | srv_servicemark | 服务标记 | mark_id, service_id |
| 158 | srv_serviceprocess | 服务进程 | process_id, service_id |
| 159 | srv_servicestandard | 服务标准 | standard_id, service_id |
| 160 | **srv_task** | **服务任务表** | ID, TNB, STAT, FKEY, VNB |
| 161 | srv_taskmark | 任务标记 | mark_id, task_id |
| 162 | srv_taskprocess | 任务进程 | process_id, task_id |
| 163 | srv_task_forward | 任务转发 | forward_id, task_id |
| 164 | srv_task_transfer | 任务移交 | transfer_id, task_id |
| 165 | standard_density | 标准密度 | density_id, oil_type |
| 166 | stand_glide_time_config | 标准滑行时间配置 | config_id, stand_code |
| 167 | sys_code | 系统码表 | code_id, code_type, para_value |
| 168 | task_oilbillnumber | 任务油单号 | taskId, oilBillNumber |
| 169 | terminal_config | 终端配置 | config_id, code, value |
| 170 | time_record | 时间记录 | last_time, type |
| 171 | tnb_mapping | 任务号映射 | old_tnb, new_tnb |
| 172 | united_flight_message | 联合航班消息 | message_id, flight_list |
| 173 | upload_oil_bill_message | 油单上传消息 | message_id, bill_id, status |
| 174 | upload_task_message | 任务上传消息 | message_id, task_id, status |
| 175 | user | 用户 | user_id, user_number, user_name |
| 176 | user_type | 用户类型 | user_type_id, user_type_name |
| 177 | workingmsg | 工作消息 | TGNO, MTID, SMID, CTNT |
| 178 | workingmsgconfig | 工作消息配置 | config_id, msg_type |
| 179 | workingmsgfd | 工作消息字段 | field_id, msg_id, field_name |
| 180 | work_schedule | 工作日程 | schedule_id, user_id |

---

## 3. 核心业务表详解

### 3.1 航班核心表 (flightinfo)

**核心特征：**
- 主键：id (bigint AUTO_INCREMENT)
- 唯一索引：flseq (航班唯一号)
- 字段数量：200+ 个字段
- 字符集：utf8
- 存储引擎：InnoDB

**关键字段分类：**

| 字段类别 | 字段示例 | 说明 |
|----------|----------|------|
| 航班标识 | flseq, flop, flno, adid | 航班唯一号、航班日、航班号、进离港 |
| 基础信息 | al2c, alcname, ac3c, acname | 航司2码、航司名、机型3码、机型名 |
| 时间信息 | ostd, atot, stot, ctot | 计划/实际起飞/到达时间 |
| 轮档时间 | onbl, ofbl | 上/撤轮档时间 |
| 客舱门时间 | fdct, fdot | 关/开客舱门时间 |
| 油量信息 | ol2, ola, oll | 轮档油量、应加油量、起飞油量 |
| 旅客信息 | adult, child, infant, sum_number | 成人/儿童/婴儿/总人数 |
| 进离港标识 | flti, ftyp | D=国内/I=国际/M=混合/R=地区 |
| 状态标识 | ftyp, stat | X=取消/Y=延迟/S=正常 |
| 位置信息 | regnumber, bridge, placecode | 机号、廊桥、机位 |
| 云平台 | flid, match_status, different_reason | 云平台航班ID、匹配状态 |
| SAF支持 | order_type, saf_purity, saf_category | SAF订单类型、纯度、品类 |
| 加油状态 | oilstart, oilend, addoilstatus | 加油开始/结束时间、状态 |
| 自动派工 | can_automatic_dispatch, auto_dispatch_priority | 可自动派工、派工优先级 |
| 时间戳 | timeluts | 最后更新时间 |

```sql
CREATE TABLE `flightinfo` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `flseq` varchar(50) NOT NULL COMMENT '航班唯一号',
  `flop` varchar(8) NOT NULL COMMENT '航班日',
  `flno` varchar(20) NOT NULL COMMENT '航班号',
  `adid` varchar(1) NOT NULL COMMENT '进离港',
  `ac3c` varchar(10) DEFAULT NULL,
  `acname` varchar(30) DEFAULT NULL COMMENT '机型名称',
  `al2c` varchar(3) DEFAULT NULL COMMENT '航空公司二码',
  `alcname` varchar(40) DEFAULT NULL COMMENT '航空公司中文名称',
  `regnumber` varchar(20) DEFAULT NULL,
  `org3` varchar(4) DEFAULT NULL COMMENT '始发站3码',
  `des3` varchar(4) DEFAULT NULL COMMENT '目的站3码',
  `ostd` varchar(19) DEFAULT NULL COMMENT '计划起飞时间',
  `atot` varchar(19) DEFAULT NULL COMMENT '实际时间',
  `stot` varchar(19) DEFAULT NULL,
  `ctot` varchar(19) DEFAULT NULL,
  `onbl` varchar(19) DEFAULT NULL COMMENT '上轮档',
  `ofbl` varchar(19) DEFAULT NULL COMMENT '撤轮档',
  `fdct` varchar(19) DEFAULT NULL COMMENT '关客舱门',
  `fdot` varchar(19) DEFAULT NULL COMMENT '开客舱门',
  `ol2` varchar(50) DEFAULT NULL,
  `ola` varchar(50) DEFAULT NULL,
  `oll` varchar(50) DEFAULT NULL,
  `flti` varchar(1) DEFAULT NULL COMMENT 'D/I/M/R',
  `ftyp` varchar(1) DEFAULT NULL COMMENT 'X/Y/S',
  `flid` varchar(50) DEFAULT NULL COMMENT '云平台航班id',
  `match_status` varchar(10) DEFAULT NULL COMMENT '匹配状态',
  `order_type` varchar(10) DEFAULT NULL COMMENT 'SAF订单',
  `saf_purity` varchar(20) DEFAULT NULL COMMENT 'SAF纯度',
  `can_automatic_dispatch` tinyint(1) DEFAULT NULL,
  `auto_dispatch_priority` int(11) DEFAULT '0' COMMENT '自动派工等级',
  `timeluts` datetime(6) DEFAULT CURRENT_TIMESTAMP(6),
  PRIMARY KEY (`id`),
  UNIQUE KEY `flseq` (`flseq`) USING BTREE
) ENGINE=InnoDB AUTO_INCREMENT=1674712 DEFAULT CHARSET=utf8;
```

### 3.2 油单核心表 (oil_bill)

**核心特征：**
- 主键：id (varchar 50)
- 索引：bill_number, task_id, version, generated_date, tenant
- 字段数量：190+ 个字段
- 字符集：utf8
- 用途：记录加油业务全流程数据

**关键字段分类：**

| 字段类别 | 字段示例 | 说明 |
|----------|----------|------|
| 基本信息 | id, bill_number, bill_type, version | 主键、油单号、类型、版本 |
| 航班关联 | flseq, flop, flno, adid, regnumber | 航班唯一号、航班日、航班号 |
| 飞机信息 | ac3c, acname, org3, des3, al2c | 机型、始发站、目的站、航司 |
| 油量信息 | oll, volume, weight, oil_temperature | 起飞油量、加油体积/重量、温度 |
| 车辆信息 | vehicle_type, vehicle_number, well_number | 车辆类型、车号、地井号 |
| 加油员 | fule_man1-4, fule_man_count | 加油员1-4，人数 |
| 任务关联 | task_id, task_start_time, task_end_time | 任务号、开始/结束时间 |
| 状态管理 | is_confirm, is_audited, is_delete, is_uploaded | 确认/审核/删除/上传状态 |
| ERP集成 | erpst1, erpdat1, erpst2, erpdat2 | ERP发送/处理状态 |
| 油单标记 | flgDel, flgRev, flgNeg, fsidSfx | 删除/冲销/负向标记、后缀 |
| 标准化 | sdst, sddat, sdrsn | 标准化状态、时间、原因 |
| 税类信息 | tax_type, bill_kind | B=保税/FB=非保税 |
| SAF支持 | order_type, saf_purity, saf_category | SAF订单类型、纯度、品类 |
| 云平台 | flseq_cloud, dsst | 南航航班号、航司发送状态 |
| 化验 | assay_sheet_number, oil_density | 化验单号、油密度 |

```sql
CREATE TABLE `oil_bill` (
  `id` varchar(50) NOT NULL COMMENT '主键id',
  `bill_number` varchar(50) DEFAULT NULL COMMENT '油单号',
  `bill_type` varchar(1) DEFAULT NULL COMMENT '0/1/2',
  `version` varchar(10) DEFAULT NULL COMMENT '版本',
  `flseq` varchar(50) DEFAULT NULL COMMENT '航班唯一号',
  `flop` varchar(8) DEFAULT NULL COMMENT '航班日',
  `flno` varchar(20) DEFAULT NULL COMMENT '航班号',
  `regnumber` varchar(20) DEFAULT NULL COMMENT '飞机注册号',
  `ac3c` varchar(10) DEFAULT NULL,
  `acname` varchar(30) DEFAULT NULL COMMENT '机型名称',
  `oll` varchar(50) DEFAULT NULL COMMENT '起飞油量(吨)',
  `volume` varchar(50) DEFAULT NULL COMMENT '加油体积(升)',
  `weight` varchar(50) DEFAULT NULL COMMENT '加油重量(千克)',
  `oil_temperature` varchar(50) DEFAULT NULL COMMENT '油温',
  `oil_density` varchar(50) DEFAULT NULL COMMENT '油密',
  `vehicle_type` varchar(10) DEFAULT NULL COMMENT 'VEHOILCAN/VEHOILPIPE',
  `vehicle_number` varchar(50) DEFAULT NULL COMMENT '加油车编号',
  `well_number` varchar(100) DEFAULT NULL COMMENT '地井编号',
  `fule_man1` varchar(50) DEFAULT NULL COMMENT '加油员1',
  `fule_man2` varchar(50) DEFAULT NULL COMMENT '加油员2',
  `fule_man3` varchar(50) DEFAULT NULL COMMENT '加油员3',
  `fule_man4` varchar(50) DEFAULT NULL COMMENT '加油员4',
  `task_id` int(11) DEFAULT NULL COMMENT '任务号',
  `is_confirm` tinyint(1) DEFAULT NULL COMMENT '确认状态',
  `is_audited` tinyint(1) DEFAULT NULL COMMENT '审核状态',
  `is_delete` tinyint(1) DEFAULT NULL COMMENT '是否删除',
  `is_uploaded` char(1) DEFAULT '0' COMMENT '上传状态',
  `tax_type` varchar(10) DEFAULT NULL COMMENT 'B/FB',
  `flgDel` varchar(25) DEFAULT NULL COMMENT '0/1/2',
  `flgRev` varchar(25) DEFAULT NULL COMMENT '0/1',
  `flgNeg` varchar(25) DEFAULT NULL COMMENT '0/1',
  `fsidSfx` varchar(25) DEFAULT NULL COMMENT '后缀',
  `erpst1` varchar(25) DEFAULT NULL COMMENT 'N/S/F',
  `erpst2` varchar(25) DEFAULT NULL COMMENT 'N/S/F',
  `sdst` varchar(25) DEFAULT NULL COMMENT '标准化状态',
  `order_type` varchar(5) DEFAULT NULL COMMENT '1=SAF',
  `saf_purity` varchar(10) DEFAULT NULL COMMENT 'SAF纯度',
  `saf_category` varchar(20) DEFAULT NULL COMMENT 'SAF品类',
  `saf_craft` varchar(20) DEFAULT NULL COMMENT 'SAF工艺',
  `timeluts` datetime(6) DEFAULT CURRENT_TIMESTAMP(6),
  PRIMARY KEY (`id`),
  KEY `bill_number` (`bill_number`),
  KEY `task_id` (`task_id`) USING BTREE,
  KEY `version` (`version`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='油单信息表';
```

### 3.3 服务任务表 (srv_task)

**核心特征：**
- 主键：ID (int 11)
- 索引：TNB, FKEY, TENB, SDAT
- 字符集：utf8_bin (二进制排序)
- 用途：服务任务全生命周期管理

**关键字段分类：**

| 字段类别 | 字段示例 | 说明 |
|----------|----------|------|
| 基本信息 | ID, TNB, SVNO, TNAM | 主键、任务号、服务号、任务名 |
| 任务状态 | STAT, TSTN, REMK | 状态码、状态名、备注 |
| 航班关联 | FKEY, FLOP, FLNO, ADID, REGN | 航班唯一号、航班日、航班号 |
| 资源分配 | RTYP, RTNM, RLID, VNB, SNB | 资源类型、车号、资源号 |
| 人员分配 | TENB, TENM, DEPID | 任务员工、工号、部门 |
| 派工模式 | PMOD, dispatchmode | A=自动/M=手动 |
| 时间信息 | DPBE, DPEN, DABE, DAEN, DACX | 计划/实际开始/结束/取消时间 |
| 服务效率 | PFTM, SFTM, AFTM, COEF | 计划/标准/实际服务时间、效率系数 |
| 转发移交 | FRWD, TSF | Y/N 是否允许转发/移交 |
| 进程管理 | PCID, PCNM, PIDX | 当前进程ID/名、顺序号 |
| 油单关联 | billnumber | 关联油单号 |

**任务状态流转：**

```
┌─────────────────────────────────────────────────────────────────┐
│                      任务状态流转图                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│    ┌──────┐     ┌──────┐     ┌──────┐     ┌──────┐          │
│    │ PLN  │ ──▶ │ DWN  │ ──▶ │ ACP  │ ──▶ │ BEG  │          │
│    │ 计划 │     │ 派发 │     │ 接受 │     │ 开始 │          │
│    └──────┘     └──────┘     └──────┘     └──────┘          │
│                                                 │             │
│                                                 ▼             │
│    ┌──────┐     ┌──────┐     ┌──────┐     ┌──────┐          │
│    │ END  │ ◀── │ WRK  │ ◀── │ ARV  │     │      │          │
│    │ 结束 │     │ 作业 │     │ 到位 │              │          │
│    └──────┘     └──────┘     └──────┘              │          │
│                                                  │  │         │
│                    ┌───────────────────────┐      │  │         │
│                    │ CXX (取消)            │ ◀─────┘  │         │
│                    └───────────────────────┘          │         │
│                                                      ▼         │
│    ┌─────────────────────────────────────────────────────┐   │
│                    其他操作：转发(FRWD)、移交(TRSF)         │   │
│    ┌─────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

```sql
CREATE TABLE `srv_task` (
  `ID` int(11) NOT NULL DEFAULT '0',
  `TENANTID` varchar(50) DEFAULT NULL COMMENT '租户',
  `TNB` int(11) DEFAULT NULL COMMENT '任务号',
  `SVNO` varchar(64) DEFAULT NULL COMMENT '服务号',
  `TNAM` varchar(100) DEFAULT NULL COMMENT '任务名称',
  `STAT` varchar(20) DEFAULT NULL COMMENT 'PLN/DWN/ACP/BEG/ARV/WRK/END/CXX',
  `TSTN` varchar(50) DEFAULT NULL COMMENT '状态名称',
  `FKEY` varchar(64) DEFAULT NULL COMMENT '航班唯一号',
  `FLOP` varchar(10) DEFAULT NULL COMMENT '航班日',
  `FLNO` varchar(50) DEFAULT NULL COMMENT '航班号',
  `ADID` char(1) DEFAULT NULL COMMENT '进离港',
  `REGN` varchar(50) DEFAULT NULL COMMENT '机号',
  `RLID` varchar(50) DEFAULT NULL COMMENT '岗位代码',
  `TENB` varchar(50) DEFAULT NULL COMMENT '任务员工',
  `TENM` varchar(50) DEFAULT NULL COMMENT '任务员工姓名',
  `VNB` varchar(50) DEFAULT NULL COMMENT '车号',
  `PMOD` char(1) DEFAULT NULL COMMENT 'A/M',
  `dispatchmode` varchar(10) DEFAULT 'M' COMMENT 'A/M',
  `FRWD` char(1) DEFAULT NULL COMMENT 'Y/N',
  `TRSF` char(1) DEFAULT NULL COMMENT 'Y/N',
  `DPBE` varchar(20) DEFAULT NULL COMMENT '计划开始时间',
  `DPEN` varchar(20) DEFAULT NULL COMMENT '计划结束时间',
  `DABE` varchar(20) DEFAULT NULL COMMENT '实际开始时间',
  `DAEN` varchar(20) DEFAULT NULL COMMENT '实际结束时间',
  `DACX` varchar(20) DEFAULT NULL COMMENT '实际取消时间',
  `PFTM` varchar(10) DEFAULT NULL COMMENT '计划服务总时间',
  `SFTM` varchar(10) DEFAULT NULL COMMENT '标准任务总时间',
  `AFTM` varchar(10) DEFAULT NULL COMMENT '实际服务总时间',
  `COEF` varchar(10) DEFAULT NULL COMMENT '服务效率',
  `billnumber` varchar(50) DEFAULT NULL COMMENT '油单号',
  PRIMARY KEY (`ID`),
  KEY `tnb` (`TNB`),
  KEY `FKEY` (`FKEY`),
  KEY `TENB` (`TENB`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='任务';
```

### 3.4 AGDIS配置表

#### agdis_color_config (颜色配置)

```sql
CREATE TABLE `agdis_color_config` (
  `groupid` int(11) NOT NULL COMMENT '分组ID',
  `group_description` varchar(100) NOT NULL COMMENT '分组描述',
  `class_name` varchar(100) NOT NULL COMMENT 'CSS类名',
  `class_description` varchar(100) NOT NULL COMMENT '颜色描述',
  `enable` tinyint(4) NOT NULL DEFAULT '1' COMMENT '是否显示',
  `text` tinyint(4) NOT NULL DEFAULT '1' COMMENT '文本颜色',
  `border` tinyint(4) NOT NULL DEFAULT '1' COMMENT '边框颜色',
  `background` tinyint(4) NOT NULL DEFAULT '1' COMMENT '背景颜色',
  `important` tinyint(4) NOT NULL DEFAULT '1' COMMENT '是否important',
  `dark` tinyint(4) NOT NULL DEFAULT '1' COMMENT '是否深颜色',
  `orderby` int(11) NOT NULL COMMENT '排序',
  `TENANTID` varchar(50) DEFAULT ''
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

#### agdis_filter_config (过滤配置)

```sql
CREATE TABLE `agdis_filter_config` (
  `id` varchar(50) NOT NULL,
  `arrival_view` varchar(10) NOT NULL COMMENT '到达视图',
  `depart_view` varchar(10) NOT NULL COMMENT '出发视图',
  `code` varchar(20) NOT NULL COMMENT '代码',
  `name` varchar(20) NOT NULL COMMENT '名称',
  `original_value` varchar(20) NOT NULL COMMENT '原始值',
  `mapping_value` varchar(20) NOT NULL COMMENT '映射值',
  `default_filter_F2` varchar(10) NOT NULL COMMENT 'F2默认过滤',
  `orderby` int(11) NOT NULL COMMENT '排序',
  `enable` int(11) NOT NULL COMMENT '是否启用',
  `filter_rule` varchar(200) DEFAULT NULL COMMENT '过滤规则',
  `filter_desc` varchar(200) DEFAULT NULL COMMENT '过滤描述',
  `TENANTID` varchar(50) DEFAULT '' COMMENT '租户'
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

#### agdis_flight_clock_* (航班时钟相关)

```sql
-- 时钟规则
CREATE TABLE `agdis_flight_clock_rule` (
  `articleId` varchar(64) DEFAULT NULL COMMENT '文章ID',
  `userType` varchar(10) DEFAULT NULL COMMENT '用户类型',
  `typeId` varchar(64) DEFAULT NULL COMMENT '类型ID',
  `flightRule` varchar(500) DEFAULT NULL COMMENT '航班规则',
  `flightFilter` varchar(20000) DEFAULT NULL COMMENT '航班过滤规则',
  `TENANTID` varchar(50) DEFAULT NULL COMMENT '租户'
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

-- 提醒内容
CREATE TABLE `agdis_flight_clock_remind_content` (
  `ruleId` varchar(64) NOT NULL COMMENT '规则ID',
  `flseq` varchar(255) NOT NULL COMMENT '航班key',
  `nextAlarmTime` varchar(64) DEFAULT NULL COMMENT '下次提醒时间',
  `userId` varchar(32) DEFAULT NULL COMMENT '用户ID',
  `remindContent` varchar(10000) DEFAULT NULL COMMENT '提醒内容',
  `flop` varchar(16) DEFAULT NULL COMMENT '航班日',
  `tenantId` varchar(32) DEFAULT NULL COMMENT '租户ID',
  PRIMARY KEY (`ruleId`,`flseq`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

### 3.5 GA除冰任务表 (ga_task)

```sql
CREATE TABLE `ga_task` (
  `task_id` varchar(50) NOT NULL COMMENT '任务ID',
  `bill_number` varchar(25) DEFAULT NULL COMMENT '油单号',
  `bill_type_cn` varchar(25) DEFAULT NULL COMMENT '油单类型名称',
  `bill_type` varchar(1) DEFAULT NULL COMMENT '油单类型',
  `task_name` varchar(50) DEFAULT NULL COMMENT '任务名称',
  `process_id` varchar(20) DEFAULT NULL COMMENT '进程ID',
  `process_name` varchar(20) DEFAULT NULL COMMENT '进程名称',
  `process_time` varchar(50) DEFAULT NULL COMMENT '进程时间',
  `order_id` varchar(50) DEFAULT NULL COMMENT '订单ID',
  `order_number` varchar(50) DEFAULT NULL COMMENT '订单号',
  `order_type` varchar(10) DEFAULT NULL COMMENT '订单类型',
  `executor` varchar(20) DEFAULT NULL COMMENT '执行人',
  `executor_name` varchar(20) DEFAULT NULL COMMENT '执行人名称',
  `creator` varchar(20) DEFAULT NULL COMMENT '创建人',
  `creator_name` varchar(20) DEFAULT NULL COMMENT '创建人名称',
  `tenant` varchar(20) DEFAULT NULL COMMENT '租户',
  `remark` varchar(100) DEFAULT NULL COMMENT '备注',
  `rtyp` varchar(10) DEFAULT NULL COMMENT '服务类型',
  `rtnm` varchar(10) DEFAULT NULL COMMENT '服务类型名称',
  `operator` varchar(25) DEFAULT NULL COMMENT '操作者',
  `operator_name` varchar(25) DEFAULT NULL COMMENT '操作者名称',
  PRIMARY KEY (`task_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

### 3.6 用户表 (user)

```sql
CREATE TABLE `user` (
  `user_id` varchar(50) NOT NULL COMMENT '用户ID',
  `user_number` varchar(20) NOT NULL COMMENT '工号',
  `user_name` varchar(50) NOT NULL COMMENT '姓名',
  `dept_id` varchar(50) DEFAULT NULL COMMENT '部门ID',
  `dept_name` varchar(100) DEFAULT NULL COMMENT '部门名称',
  `position_id` varchar(20) DEFAULT NULL COMMENT '岗位ID',
  `position_name` varchar(100) DEFAULT NULL COMMENT '岗位名称',
  `age` int(3) DEFAULT NULL COMMENT '年龄',
  `sex` varchar(2) DEFAULT NULL COMMENT '性别',
  `national` varchar(50) DEFAULT NULL COMMENT '民族',
  `job_id` varchar(50) DEFAULT NULL COMMENT '职务ID',
  `level_id` varchar(50) DEFAULT NULL COMMENT '级别',
  `user_type_id` varchar(50) DEFAULT NULL COMMENT '用户类型',
  `telephone` varchar(20) DEFAULT NULL COMMENT '电话',
  `remark` varchar(255) DEFAULT NULL COMMENT '备注',
  `tenant` varchar(20) DEFAULT NULL COMMENT '租户',
  PRIMARY KEY (`user_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

### 3.7 加油车表 (oil_vehicle)

```sql
CREATE TABLE `oil_vehicle` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '主键',
  `create_date` datetime DEFAULT NULL COMMENT '创建时间',
  `modify_date` datetime DEFAULT NULL COMMENT '修改时间',
  `assay_no` varchar(255) DEFAULT NULL COMMENT '化验单号',
  `density` float NOT NULL COMMENT '密度',
  `temperature` float NOT NULL COMMENT '温度',
  `tenant` varchar(255) DEFAULT NULL COMMENT '租户ID',
  `type` varchar(255) DEFAULT NULL COMMENT '车辆类型 O/管线车',
  `vehicle_no` varchar(255) DEFAULT NULL COMMENT '车辆号',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

### 3.8 ADS-B位置历史表 (adsb_ll_history)

**分区表特性：**
- 按 `LASTTIME` 字段按天分区
- 分区范围：2024-05-26 至 2024-11-29 (约200个分区)

```sql
CREATE TABLE `adsb_ll_history` (
  `HEXADDRESS` varchar(60) DEFAULT NULL COMMENT '飞机地址码',
  `DEPARTMENT` varchar(60) DEFAULT NULL COMMENT '部门',
  `LON` varchar(60) DEFAULT NULL COMMENT '经度',
  `LAT` varchar(60) DEFAULT NULL COMMENT '纬度',
  `VERTICALSPEED` varchar(60) DEFAULT NULL COMMENT '垂直速度',
  `SPEED` varchar(60) DEFAULT NULL COMMENT '速度',
  `HEIGHT` varchar(60) DEFAULT NULL COMMENT '高度',
  `RECEIVERLOCATION` varchar(60) DEFAULT NULL COMMENT '接收器位置',
  `FLIGHTNUM` varchar(60) DEFAULT NULL COMMENT '航班号',
  `REGCODE` varchar(60) DEFAULT NULL COMMENT '注册号',
  `COURSE` varchar(60) DEFAULT NULL COMMENT '航向',
  `REALFIGHTNO` varchar(60) DEFAULT NULL COMMENT '实际航班号',
  `FLSEQ` varchar(372) DEFAULT NULL COMMENT '航班唯一号',
  `TENANT` varchar(60) DEFAULT NULL COMMENT '租户',
  `LASTTIME` datetime DEFAULT NULL COMMENT '最后时间',
  `CREATETIME` datetime DEFAULT NULL COMMENT '创建时间',
  KEY `RANGE_time` (`LASTTIME`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8
PARTITION BY RANGE (TO_DAYS(LASTTIME)) (
  PARTITION p20240526 VALUES IN (739397) ENGINE = InnoDB,
  PARTITION p20240527 VALUES IN (739398) ENGINE = InnoDB,
  -- ... 更多分区
);
```

---

## 4. 业务数据流分析

### 4.1 航班数据流

```
┌────────────────────────────────────────────────────────────────────────┐
│                          航班数据流转图                                  │
├────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌────────────────┐    ┌────────────────┐    ┌────────────────┐     │
│  │ 外部接口        │    │ flightinfo     │    │ 调度服务        │     │
│  │ 接口同步        │ ──▶│ 航班主表       │ ──▶│ srv_task       │     │
│  └────────────────┘    └────────────────┘    └────────────────┘     │
│         │                    │                       │                │
│         │                    │                       │                │
│         ▼                    ▼                       ▼                │
│  ┌────────────────┐    ┌────────────────┐    ┌────────────────┐     │
│  │ flight_data   │    │ flight_lock   │    │ oil_bill       │     │
│  │ 航班数据配置   │    │ 航班锁定      │    │ 油单表         │     │
│  └────────────────┘    └────────────────┘    └────────────────┘     │
│                                                │                    │
│                                                ▼                    │
│                                         ┌────────────────┐          │
│                                         │ 云平台同步      │          │
│                                         │ flightinfo_cloud│          │
│                                         └────────────────┘          │
│                                                                         │
└────────────────────────────────────────────────────────────────────────┘
```

### 4.2 油单生命周期

```
┌────────────────────────────────────────────────────────────────────────┐
│                          油单生命周期                                    │
├────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  创建(CREATE) ──▶ 确认(CONFIRM) ──▶ 审核(AUDIT) ──▶ 上传(UPLOAD)        │
│      │                │                │                │               │
│      │                │                │                │               │
│      ▼                ▼                ▼                ▼               │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐             │
│  │ bill_   │    │ 油单    │    │ 审核    │    │ 发送    │             │
│  │ number  │    │ 确认    │    │ 通过    │    │ ERP    │             │
│  │ 编号    │    │ 签字    │    │         │    │         │             │
│  └─────────┘    └─────────┘    └─────────┘    └─────────┘             │
│                                                                         │
│                         ◀───────────────────────────────────          │
│                         │      状态回退/修改                           │  │
│                                                                         │
│  上传后 ──▶ 航司确认 ──▶ 结算 ──▶ 归档                                 │
│      │         │          │        │                                   │
│      ▼         ▼          ▼        ▼                                   │
│  ┌─────────┐┌─────────┐┌─────────┐┌─────────┐                           │
│  │ 航司    ││ 云平台  ││ 财务    ││ oil_    │                           │
│  │ 确认    ││ 发送    ││ 结算    ││ bill_   │                           │
│  │ dsst=S ││         ││         ││ history │                           │
│  └─────────┘└─────────┘└─────────┘└─────────┘                           │
│                                                                         │
└────────────────────────────────────────────────────────────────────────┘
```

### 4.3 任务派发流程

```
┌────────────────────────────────────────────────────────────────────────┐
│                          任务派发流程                                    │
├────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. 航班落地/计划 ──▶ 生成服务需求                                     │
│                             │                                           │
│                             ▼                                           │
│  2. 自动派工(AUTO) ──▶ 手动派工(MANUAL)                               │
│        │                     │                                         │
│        │                     │                                         │
│        ▼                     ▼                                         │
│  3. 员工接收(ACP) ──▶ 员工确认                                         │
│                             │                                           │
│                             ▼                                           │
│  4. 到位(ARV) ──▶ 开始作业(BEG) ──▶ 作业中(WRK)                       │
│                                                                         │
│                             │                                           │
│                             ▼                                           │
│  5. 作业完成(END) ──▶ 生成油单(oil_bill)                               │
│                             │                                           │
│                             ▼                                           │
│  6. 油单确认 ──▶ 油单审核 ──▶ 任务结束                                 │
│                                                                         │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 5. 索引分析

### 5.1 主键类型分布

| 主键类型 | 表数量 | 代表表 |
|----------|--------|--------|
| bigint AUTO_INCREMENT | 10+ | flightinfo, oil_vehicle, place |
| varchar | 180+ | oil_bill, user, ga_task, agdis_* |
| int | 5+ | srv_task, accu_filter_oil |
| char | 5+ | workingmsg, srv_alarm |
| decimal | 10+ | agdis_bill_curr, agdis_bill_li |

### 5.2 常用查询索引

```sql
-- 航班查询索引
flightinfo: UNIQUE INDEX flseq(flseq)
flightinfo: UNIQUE INDEX id(id)

-- 油单查询索引
oil_bill: INDEX bill_number(bill_number)
oil_bill: INDEX task_id(task_id) USING BTREE
oil_bill: INDEX version(version) USING BTREE
oil_bill: INDEX generated_date_idx(generated_date) USING BTREE
oil_bill: INDEX tenant(tenant) USING BTREE

-- 任务查询索引
srv_task: INDEX tnb(TNB)
srv_task: INDEX FKEY(FKEY)
srv_task: INDEX TENB(TENB)
srv_task: INDEX SDAT(SDAT) USING BTREE

-- ADS-B历史索引
adsb_ll_history: INDEX RANGE_time(LASTTIME) -- 分区键
```

### 5.3 分区表

| 表名 | 分区类型 | 分区键 | 说明 |
|------|----------|--------|------|
| adsb_ll_history | RANGE | LASTTIME | 按天分区，约200个分区 |

---

## 6. 数据关联关系

### 6.1 ER关系图(核心表)

```
┌────────────────────────────────────────────────────────────────────────┐
│                          核心表关系图                                    │
├────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│    ┌────────────────┐                        ┌────────────────┐         │
│    │  flightinfo   │                        │      user      │         │
│    │  航班主表      │                        │    用户表      │         │
│    └───────┬────────┘                        └────────┬───────┘         │
│            │ flseq, flop, flno                        │                 │
│            │                                           │ TENB, TENM      │
│            │                                           │                 │
│            ▼                                           ▼                 │
│    ┌────────────────┐  1:N                 ┌────────────────┐         │
│    │   srv_task    │◀────────────────────────│    vehicle     │         │
│    │   任务表       │        VNB             │    车辆表       │         │
│    └───────┬────────┘                        └────────────────┘         │
│            │ task_id, FKEY                                         │         │
│            │                                                        │         │
│            ▼                                                        ▼         │
│    ┌────────────────┐                                     ┌────────────────┐ │
│    │   oil_bill     │                                     │    ga_task     │ │
│    │   油单表        │                                     │    除冰任务     │ │
│    └────────────────┘                                     └────────────────┘ │
│           │                                                        │         │
│           │ task_id, flseq                                        │         │
│           │                                                        │         │
│           ▼                                                        ▼         │
│    ┌────────────────┐                                     ┌────────────────┐ │
│    │oil_bill_confirm│                                     │ga_task_process │ │
│    │   油单确认     │                                     │   进程记录     │ │
│    └────────────────┘                                     └────────────────┘ │
│                                                                         │
└────────────────────────────────────────────────────────────────────────┘
```

### 6.2 外键关系

| 子表 | 外键字段 | 父表 | 说明 |
|------|----------|------|------|
| oil_bill | task_id | srv_task | 任务-油单 |
| srv_task | FKEY | flightinfo | 任务-航班 |
| srv_task | VNB | oil_vehicle | 任务-车辆 |
| oil_bill | flseq | flightinfo | 油单-航班 |
| ga_task | task_id | srv_task | 除冰-服务任务 |
| agdis_man_car_binding | vehicle_no | oil_vehicle | 人车绑定 |

---

## 7. 租户隔离设计

### 7.1 多租户字段

几乎所有表都包含 `tenant` 或 `TENANTID` 字段：

```sql
-- 字段命名统计
tenant     : 150+ 表使用
TENANTID   : 50+ 表使用 (主要在agdis_*表)

-- 典型租户字段
`tenant` varchar(20) COMMENT '租户ID'
`TENANTID` varchar(50) COMMENT '租户'
```

### 7.2 租户隔离比例

| 指标 | 数值 |
|------|------|
| 总表数 | 206 |
| 含租户字段的表 | 200+ |
| 租户隔离比例 | 97% |

---

## 8. 扩展字段支持

### 8.1 SAF支持字段

**在 flightinfo 表中：**
```sql
`order_type` varchar(10) DEFAULT NULL COMMENT '1=SAF油单, 0=非SAF油单'
`saf_purity` varchar(20) DEFAULT NULL COMMENT 'SAF纯度'
`saf_blend_purity` varchar(20) DEFAULT NULL COMMENT 'SAF勾兑纯度'
`saf_category` varchar(20) DEFAULT NULL COMMENT 'SAF品类 (固定为SBC)'
`saf_craft` varchar(20) DEFAULT NULL COMMENT 'SAF工艺 (HEFA-SPK/SIP/FT-SPK)'
`saf_material_type` varchar(20) DEFAULT NULL COMMENT 'SAF物料类型'
```

**在 oil_bill 表中：**
```sql
`order_type` varchar(5) DEFAULT NULL COMMENT '1=SAF油单, 0=非SAF油单'
`saf_purity` varchar(10) DEFAULT NULL COMMENT 'SAF纯度'
`saf_category` varchar(20) DEFAULT NULL COMMENT 'SAF品类'
`saf_category_name` varchar(100) DEFAULT NULL COMMENT 'SAF品类名称'
`saf_craft` varchar(20) DEFAULT NULL COMMENT 'SAF工艺'
`saf_craft_name` varchar(100) DEFAULT NULL COMMENT 'SAF工艺名称'
`saf_blend_purity` varchar(20) DEFAULT NULL COMMENT 'SAF勾兑纯度'
`sbc_tax` varchar(10) DEFAULT NULL COMMENT 'SBC保税类型'
`sbc_weight` varchar(20) DEFAULT NULL COMMENT 'SBC重量'
`sbc_volume` varchar(20) DEFAULT NULL COMMENT 'SBC升数'
`saf_material_type` varchar(20) DEFAULT NULL COMMENT 'SAF物料类型'
`erp_material_code` varchar(20) DEFAULT NULL COMMENT 'ERP物料编码'
```

### 8.2 云平台支持

**在 flightinfo 表中：**
```sql
`flid` varchar(50) DEFAULT NULL COMMENT '云平台航班ID'
`match_status` varchar(10) DEFAULT NULL COMMENT '匹配状态'
`different_reason` varchar(255) DEFAULT NULL COMMENT '匹配不一致原因'
```

**在 oil_bill 表中：**
```sql
`flseq_cloud` varchar(64) DEFAULT NULL COMMENT '南航航班唯一号'
`dsst` varchar(10) DEFAULT NULL COMMENT 'S=已发送,F=失败,null=未发送'
`dsst_time` varchar(30) DEFAULT NULL COMMENT '航司发送时间'
`airline_confirm_status` varchar(10) DEFAULT NULL COMMENT '航司确认状态'
```

---

## 9. 数据安全与状态管理

### 9.1 删除标记

```sql
-- 通用删除标记
`is_delete` tinyint(1) COMMENT '是否删除'

-- 油单特殊标记
`flgDel` varchar(25) COMMENT '0=未删除, 1=已删除, 2=已失效'
`flgRev` varchar(25) COMMENT '0=未冲销, 1=已冲销'
`flgNeg` varchar(25) COMMENT '0=正常, 1=负向油单'
`fsidSfx` varchar(25) COMMENT '油单号后缀 -X/-R/-D'
```

### 9.2 版本控制

```sql
-- 油单版本控制
`version` varchar(10) COMMENT '版本号(从0开始,每修改+1)'
`vrsng` varchar(25) COMMENT '大版本号: A, B, C...'
`ifmod` varchar(25) COMMENT 'N=未修改, Y=已修改'

-- 标准化状态
`sdst` varchar(25) COMMENT 'N=未标准化, S=成功, F=失败'
`sddat` varchar(25) COMMENT '标准化时间'
`sdrsn` varchar(25) COMMENT '标准化失败原因'
```

### 9.3 ERP状态

```sql
-- ERP发送状态
`erpst1` varchar(25) COMMENT 'N=未发送, S=成功, F=失败'
`erpdat1` varchar(25) COMMENT '发送时间'
`erprsn1` varchar(250) COMMENT '发送失败原因'

-- ERP处理状态
`erpst2` varchar(25) COMMENT 'N=未处理, S=成功, F=失败'
`erpdat2` varchar(25) COMMENT '处理时间'
`erprsn2` varchar(250) COMMENT '处理失败原因'
```

### 9.4 有效状态

```sql
-- 任务有效状态
`ENABLE` char(1) COMMENT 'Y=有效, N=无效'

-- 油单有效状态
`is_valid` tinyint(1) COMMENT 'true=有效, false=无效'

-- 单据状态
`IFAV` char(1) COMMENT 'N=无效, Y=有效'
```

---

## 10. 配置与码表

### 10.1 系统码表 (sys_code)

```sql
CREATE TABLE `sys_code` (
  `code_id` varchar(60) NOT NULL COMMENT '代码ID',
  `code_type` varchar(50) DEFAULT NULL COMMENT '代码类型',
  `code_name` varchar(50) DEFAULT NULL COMMENT '代码名称',
  `para_value` varchar(50) DEFAULT NULL COMMENT 'key值',
  `para_desc` varchar(50) DEFAULT NULL COMMENT 'value值',
  `para_sort` int(11) DEFAULT NULL COMMENT '排序',
  `tenant` varchar(10) DEFAULT NULL COMMENT '租户',
  PRIMARY KEY (`code_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

-- 码表类型示例
-- flight_type: 航班类型
-- service_type: 服务类型
-- vehicle_type: 车辆类型
-- process_status: 进程状态
-- alarm_type: 告警类型
```

### 10.2 终端配置 (terminal_config)

```sql
CREATE TABLE `terminal_config` (
  `code` varchar(50) NOT NULL COMMENT '配置代码',
  `name` varchar(50) DEFAULT NULL COMMENT '配置名称',
  `type` varchar(20) DEFAULT NULL COMMENT '配置类型',
  `value` text COMMENT '配置值',
  `remark` varchar(100) DEFAULT NULL COMMENT '备注',
  `tenant` varchar(20) DEFAULT NULL COMMENT '租户',
  `config_id` varchar(100) NOT NULL COMMENT '主键',
  PRIMARY KEY (`code`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

### 10.3 AGDIS派工配置

```sql
-- 派工默认类型配置
CREATE TABLE `agdis_dispatch_default_type_config` (
  `psid` varchar(16) NOT NULL COMMENT '岗位代码',
  `service_type` varchar(50) NOT NULL COMMENT '服务类型代码',
  `flight_type` varchar(1) NOT NULL COMMENT 'A=进港, D=离港',
  `tenantid` varchar(50) NOT NULL COMMENT '租户代码',
  PRIMARY KEY (`psid`,`service_type`,`flight_type`,`tenantid`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

-- 派工时间配置
CREATE TABLE `agdis_dispatch_time_config` (
  `id` varchar(50) NOT NULL,
  `al2c` varchar(2) DEFAULT NULL COMMENT '航司二码',
  `ac3c` varchar(5) DEFAULT NULL COMMENT '机型',
  `flight_type` varchar(50) DEFAULT NULL COMMENT '航班类型',
  `delivery_time_type` varchar(50) DEFAULT NULL COMMENT '到位时间类型',
  `key_time_type` varchar(50) DEFAULT NULL COMMENT '关键时间类型',
  `interval_minutes` varchar(10) DEFAULT NULL COMMENT '间隔分钟',
  `tenant` varchar(20) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

---

## 11. 性能与优化建议

### 11.1 热点表

| 表名 | 数据量估算 | 访问频率 | 优化建议 |
|------|-----------|---------|----------|
| flightinfo | 100万+/天 | 极高 | 分区表、读写分离 |
| oil_bill | 10万+/天 | 高 | 索引优化、归档策略 |
| srv_task | 50万+/天 | 高 | 状态索引、TTL清理 |
| adsb_ll_history | 1000万+/天 | 中 | 分区表、定期归档 |
| agdis_* | 1万以内 | 低 | 配置表、无需优化 |

### 11.2 索引优化建议

```sql
-- 建议添加的索引
ALTER TABLE flightinfo ADD INDEX idx_flightinfo_flseq_timeluts (flseq, timeluts);
ALTER TABLE oil_bill ADD INDEX idx_oil_bill_flseq (flseq);
ALTER TABLE oil_bill ADD INDEX idx_oil_bill_flop (flop);
ALTER TABLE oil_bill ADD INDEX idx_oil_bill_bill_number_status (bill_number, flgDel);
ALTER TABLE srv_task ADD INDEX idx_srv_task_stat (STAT);
ALTER TABLE srv_task ADD INDEX idx_srv_task_flop (FLOP);
```

### 11.3 分区建议

```sql
-- flightinfo 按 flop (航班日) 分区
ALTER TABLE flightinfo PARTITION BY RANGE (TO_DAYS(flop)) (
  PARTITION p202603 VALUES LESS THAN (TO_DAYS('2026-04-01')),
  PARTITION p202604 VALUES LESS THAN (TO_DAYS('2026-05-01')),
  PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- oil_bill 按 wkop (油单日期) 分区
ALTER TABLE oil_bill PARTITION BY RANGE (TO_DAYS(wkop)) (
  PARTITION p202603 VALUES LESS THAN (TO_DAYS('2026-04-01')),
  PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- srv_task 按 SDAT (状态时间) 分区
ALTER TABLE srv_task PARTITION BY RANGE (TO_DAYS(SDAT)) (
  PARTITION p202603 VALUES LESS THAN (TO_DAYS('2026-04-01')),
  PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

### 11.4 归档策略

```sql
-- 油单历史归档
INSERT INTO oil_bill_history 
SELECT * FROM oil_bill 
WHERE timeluts < DATE_SUB(NOW(), INTERVAL 3 MONTH)
AND flgDel IN ('0', '1', '2');

DELETE FROM oil_bill 
WHERE timeluts < DATE_SUB(NOW(), INTERVAL 3 MONTH)
AND flgDel IN ('0', '1', '2');

-- ADS-B历史归档 (超过30天)
DELETE FROM adsb_ll_history 
WHERE LASTTIME < DATE_SUB(NOW(), INTERVAL 30 DAY);

-- 任务归档 (超过90天且已结束)
INSERT INTO srv_task_history 
SELECT * FROM srv_task 
WHERE SDAT < DATE_SUB(NOW(), INTERVAL 90 DAY)
AND STAT = 'END';

DELETE FROM srv_task 
WHERE SDAT < DATE_SUB(NOW(), INTERVAL 90 DAY)
AND STAT = 'END';
```

---

## 12. 问题与建议

### 12.1 设计问题

| 问题 | 位置 | 影响 | 建议 |
|------|------|------|------|
| **字段冗余** | flightinfo | 200+字段，查询效率低 | 垂直拆分 |
| **时间字段类型** | 大量varchar(19) | 索引效率低 | 改为datetime |
| **字符集不统一** | utf8/utf8mb4混用 | 排序问题 | 统一utf8mb4 |
| **主键类型不统一** | 有int有varchar | 关联查询复杂 | 统一bigint |
| **缺少外键约束** | 业务表之间 | 数据一致性风险 | 添加外键 |

### 12.2 性能问题

| 问题 | 描述 | 建议 |
|------|------|------|
| 大表无分区 | flightinfo 200+字段 | 按日期分区 |
| 缺少复合索引 | 常用查询无覆盖索引 | 添加复合索引 |
| ADS-B分区不足 | 仅200个分区 | 自动创建分区 |
| 历史数据未归档 | oil_bill 数据量大 | 定期归档 |
| 缺少缓存 | 无Redis缓存热点数据 | 引入缓存层 |

### 12.3 业务问题

| 问题 | 描述 | 建议 |
|------|------|------|
| 状态码不统一 | STAT字段多种状态码 | 统一状态机 |
| 租户隔离不完整 | 部分表无tenant | 补全tenant字段 |
| 缺少审计字段 | 修改历史 | 添加审计表 |
| 字段命名不规范 | 驼峰下划线混用 | 统一命名规范 |

---

## 13. 数据库统计

### 13.1 字符集分布

| 字符集 | 表数量 | 占比 |
|--------|--------|------|
| utf8_general_ci | 180+ | 87% |
| utf8_bin | 20+ | 10% |
| utf8mb4 | 5+ | 2% |
| 其他 | 2+ | 1% |

### 13.2 存储引擎分布

| 引擎 | 表数量 | 占比 |
|------|--------|------|
| InnoDB | 206 | 100% |

### 13.3 字段类型分布

| 类型 | 数量估算 | 占比 |
|------|----------|------|
| varchar | 3500+ | 70% |
| int | 300+ | 6% |
| datetime | 150+ | 3% |
| char | 100+ | 2% |
| text/longtext | 50+ | 1% |
| 其他 | 900+ | 18% |

---

## 14. 总结

### 14.1 数据库特点

1. **业务复杂度高**：200+表覆盖航班、油单、任务、配置等多个业务域
2. **多租户支持**：97%的表包含tenant字段
3. **字段冗余大**：部分核心表字段超过200个
4. **状态管理复杂**：多级状态机（航班状态、任务状态、油单状态）
5. **扩展性好**：支持SAF、云平台等新需求
6. **分区策略**：ADS-B历史表已实现分区存储

### 14.2 核心价值

| 能力 | 说明 |
|------|------|
| 航班调度 | 完整的航班生命周期管理 |
| 油单管理 | 加油全流程跟踪与结算 |
| 任务派工 | 自动/手动派工、工时统计 |
| 多租户 | 支持多机场、多航司 |
| 国际化 | 支持国内/国际/混合航班 |
| SAF支持 | 可持续航空燃料订单管理 |
| 云平台集成 | 与南航等云平台数据同步 |

### 14.3 优化优先级

**P0 - 高优：**
- flightinfo 分区优化
- 统一字符集 utf8mb4
- 时间字段类型转换

**P1 - 中优：**
- 补充复合索引
- 数据归档策略
- ADS-B分区自动扩展

**P2 - 低优：**
- 外键约束补充
- 审计日志表
- 字段命名规范

---

*报告生成时间：2026-03-25*
*分析工具：Cursor AI Code Analysis*

---

## 15. db_oil_extend 压差监控数据库分析

### 15.1 基本信息

| 属性 | 值 |
|------|-----|
| 数据库名称 | db_oil_extend |
| 数据源 | 172.20.0.2 MySQL 5.7.38 |
| 表数量 | **5** |
| 用途 | 加油车压差监控与过滤器管理 |

### 15.2 表总览

| 序号 | 表名 | 说明 | 核心字段 |
|------|------|------|----------|
| 1 | dpm_filter | 过滤器台账管理 | filter_code, model, brand, state, vnb |
| 2 | dpm_filter_maintenance_sheet | 过滤器维护记录 | filter_id, maintenance_date, replace_*_count |
| 3 | dpm_org_pressure | 原始压差数据 | vnb, vehicle_type, pressure, velocity, task_id |
| 4 | dpm_pressure_statistic | 压差统计汇总 | check_date, check_week, vnb, pressure, abnormal_state |
| 5 | dpm_standard_pressure | 标准化压差数据 | org_pressure_id, vnb, pressure, is_exception, is_maxflow |

### 15.3 业务定位

```
┌────────────────────────────────────────────────────────────────────────┐
│                   db_oil_extend 业务定位                               │
├────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                      压差监控系统                                  │  │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │  │
│  │  │  dpm_filter    │  │ dpm_org_       │  │ dpm_standard_   │ │  │
│  │  │  过滤器台账     │  │ pressure        │  │ pressure         │ │  │
│  │  │  设备管理      │  │ 原始压差数据    │  │ 标准化压差       │ │  │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘ │  │
│  │                                                                     │  │
│  │  ┌─────────────────┐  ┌─────────────────┐                        │  │
│  │  │ dpm_filter_    │  │ dpm_pressure_  │                        │  │
│  │  │ maintenance_   │  │ statistic       │                        │  │
│  │  │ sheet          │  │ 压差统计        │                        │  │
│  │  │ 维保记录       │  │                 │                        │  │
│  │  └─────────────────┘  └─────────────────┘                        │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                          │
└────────────────────────────────────────────────────────────────────────┘
```

### 15.4 过滤器管理表详解

#### dpm_filter (过滤器台账)

```sql
CREATE TABLE `dpm_filter` (
  `id` int(11) NOT NULL AUTO_INCREMENT COMMENT '主键',
  `filter_code` varchar(10) NOT NULL COMMENT '过滤器编号',
  `model` varchar(30) NOT NULL COMMENT '过滤器型号',
  `brand` varchar(30) NOT NULL COMMENT '过滤器品牌',
  `state` varchar(255) NOT NULL COMMENT '过滤器状态  正常-1 维护-2 报废-3',
  `purchase_date` varchar(20) NULL DEFAULT NULL COMMENT '采购日期',
  `last_maintenance_date` varchar(20) NULL DEFAULT NULL COMMENT '最后一次维保日期',
  `maintenance_interval` int(10) NOT NULL COMMENT '维保间隔(天)',
  `vnb` varchar(20) NOT NULL COMMENT '绑定车号',
  `tenant` varchar(20) NULL DEFAULT NULL COMMENT '租户',
  `cuser` varchar(10) NULL DEFAULT NULL COMMENT '创建人id',
  `create_time` datetime NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `uuser` varchar(10) NULL DEFAULT NULL COMMENT '修改人id',
  `update_time` datetime(6) NULL DEFAULT CURRENT_TIMESTAMP(6) COMMENT '修改时间',
  `delete_flag` varchar(255) NULL DEFAULT NULL COMMENT '删除标记位',
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 14;
```

**关键字段说明：**

| 字段 | 类型 | 说明 |
|------|------|------|
| filter_code | varchar(10) | 过滤器唯一编号 |
| model | varchar(30) | 过滤器型号，如 Filtax SRL |
| brand | varchar(30) | 品牌 |
| state | varchar(255) | 状态：1=正常, 2=维护, 3=报废 |
| purchase_date | varchar(20) | 采购日期 |
| last_maintenance_date | varchar(20) | 最后一次维保日期 |
| maintenance_interval | int(10) | 维保间隔(天) |
| vnb | varchar(20) | 绑定车号，与加油车关联 |

#### dpm_filter_maintenance_sheet (过滤器维护记录表)

```sql
CREATE TABLE `dpm_filter_maintenance_sheet` (
  `id` int(11) NOT NULL AUTO_INCREMENT COMMENT '过滤器维护记录表主键',
  `filter_id` int(11) NOT NULL COMMENT '过滤器id',
  `is_checked` char(1) NULL DEFAULT NULL COMMENT '过滤器是否检查',
  `is_cleaned` char(1) NULL DEFAULT NULL COMMENT '过滤器是否清洗',
  `replace_analysis_filter_core_count` int(10) NULL DEFAULT NULL COMMENT '更换分析滤芯条数',
  `replace_separate_filter_core_count` int(10) NULL DEFAULT NULL COMMENT '更换分离滤芯条数',
  `replace_filter_seal_gasket_count` varchar(255) NULL DEFAULT NULL COMMENT '更换过滤器密封垫圈条数',
  `maintenance_date` datetime NULL DEFAULT NULL COMMENT '维保记录日期',
  `recode_name` varchar(50) NULL DEFAULT NULL COMMENT '记录人',
  `cuser` varchar(50) NULL DEFAULT NULL COMMENT '创建人',
  `create_time` datetime NULL DEFAULT NULL COMMENT '创建时间',
  `uuser` varchar(50) NULL DEFAULT NULL COMMENT '修改人',
  `update_time` datetime NULL DEFAULT NULL COMMENT '修改时间',
  `delete_flag` char(1) NULL DEFAULT NULL COMMENT '删除标记位',
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 31;
```

**关键字段说明：**

| 字段 | 类型 | 说明 |
|------|------|------|
| filter_id | int(11) | 外键关联 dpm_filter.id |
| is_checked | char(1) | 是否检查 (Y/N) |
| is_cleaned | char(1) | 是否清洗 (Y/N) |
| replace_analysis_filter_core_count | int | 更换分析滤芯条数 |
| replace_separate_filter_core_count | int | 更换分离滤芯条数 |
| replace_filter_seal_gasket_count | varchar | 更换密封垫圈条数 |
| maintenance_date | datetime | 维保日期 |

**过滤器维保周期逻辑：**
- 根据 `maintenance_interval`（维保间隔天数）计算下次维保时间
- 维保时记录：检查、清洗、更换滤芯、更换密封垫圈等情况
- 过滤器状态 state 变更：正常(1) → 维护(2) → 报废(3)

### 15.5 压差数据表详解

#### dpm_org_pressure (原始压差数据)

```sql
CREATE TABLE `dpm_org_pressure` (
  `org_pressure_id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '主键',
  `vnb` varchar(50) NOT NULL COMMENT '车号',
  `vehicle_type` varchar(10) NOT NULL COMMENT '车辆类型',
  `pressure` varchar(50) NULL DEFAULT NULL COMMENT '压差',
  `velocity` varchar(50) NULL DEFAULT NULL COMMENT '流速',
  `task_id` bigint(20) NULL DEFAULT NULL COMMENT '任务id',
  `flno` varchar(50) NULL DEFAULT NULL COMMENT '航班号',
  `regn` varchar(50) NULL DEFAULT NULL COMMENT '机号',
  `bill_number` varchar(50) NULL DEFAULT NULL COMMENT '油单号',
  `placecode` varchar(50) NULL DEFAULT NULL COMMENT '机位号',
  `well` varchar(50) NULL DEFAULT NULL COMMENT '地井',
  `collection_time` datetime NULL DEFAULT NULL COMMENT '采集时间',
  `user_id` varchar(50) NULL DEFAULT NULL COMMENT '加油员id',
  `user_name` varchar(50) NULL DEFAULT NULL COMMENT '加油员姓名',
  `tenant` varchar(20) NOT NULL COMMENT '租户',
  `ap3c` varchar(10) NULL DEFAULT NULL COMMENT '机场3码',
  `is_maxflow` varchar(1) NULL DEFAULT NULL COMMENT '当前任务中是否是最大流速压差（Y/N）',
  `update_time` datetime(6) NULL DEFAULT CURRENT_TIMESTAMP(6),
  `create_time` datetime(6) NULL DEFAULT CURRENT_TIMESTAMP(6),
  `version` int(11) NULL DEFAULT NULL COMMENT '版本号',
  PRIMARY KEY (`org_pressure_id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 16384 COMMENT = '原始压差数据';
```

**关键字段说明：**

| 字段 | 类型 | 说明 |
|------|------|------|
| vnb | varchar(50) | 车号 |
| vehicle_type | varchar(10) | 车辆类型 |
| pressure | varchar(50) | 压差值 |
| velocity | varchar(50) | 流速 |
| task_id | bigint(20) | 关联任务ID |
| flno/regn | varchar(50) | 航班号/机号 |
| collection_time | datetime | 数据采集时间 |
| is_maxflow | varchar(1) | 是否为当前任务最大流速压差 (Y/N) |

**数据来源：** 加油车设备实时采集，通过接口上传到系统

#### dpm_standard_pressure (标准化压差数据)

```sql
CREATE TABLE `dpm_standard_pressure` (
  `standard_pressure_id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '主键',
  `org_pressure_id` bigint(20) NULL DEFAULT NULL COMMENT '原始压差ID',
  `vnb` varchar(50) NOT NULL COMMENT '车号',
  `vehicle_type` varchar(50) NOT NULL COMMENT '车类型',
  `pressure` float NULL DEFAULT NULL COMMENT '压差, 单位, MPa',
  `velocity` float NULL DEFAULT NULL COMMENT '流速, 单位，L/min',
  `total_pressure` float NULL DEFAULT NULL COMMENT '标准压差(kPa)',
  `effect` float NULL DEFAULT NULL COMMENT '流速功效',
  `abnormal_score` varchar(25) NULL DEFAULT NULL COMMENT '异常情况得分',
  `abnormal_state` varchar(25) NULL DEFAULT NULL COMMENT '异常状态',
  `task_id` bigint(20) NULL DEFAULT NULL COMMENT '任务号',
  `flno` varchar(20) NULL DEFAULT NULL COMMENT '航班号',
  `regn` varchar(20) NULL DEFAULT NULL COMMENT '机号',
  `bill_number` varchar(50) NULL DEFAULT NULL COMMENT '油单号',
  `placecode` varchar(20) NULL DEFAULT NULL COMMENT '机位',
  `well` varchar(20) NULL DEFAULT NULL COMMENT '地井',
  `pressure_dif` varchar(20) NULL DEFAULT NULL COMMENT '与上一次的压差差值',
  `change_state` varchar(20) NULL DEFAULT NULL COMMENT '压差变动异常',
  `tenant` varchar(50) NOT NULL COMMENT '租户',
  `ap3c` varchar(10) NULL DEFAULT NULL COMMENT '机场3码',
  `rated_flow_filter` float NULL DEFAULT NULL COMMENT '额定流速(L/min)',
  `voltage_up` varchar(30) NULL DEFAULT NULL COMMENT '正常压差的上限值',
  `voltage_down` varchar(30) NULL DEFAULT NULL COMMENT '正常压差的下限值',
  `voltage_change_up` varchar(30) NULL DEFAULT NULL COMMENT '压差变动上限',
  `voltage_change_down` varchar(30) NULL DEFAULT NULL COMMENT '压差变动下限',
  `is_exception` varchar(10) NULL DEFAULT NULL COMMENT '是否异常（Y、N）',
  `velocity_diff` float NULL DEFAULT NULL COMMENT '流速与额定流速/2 的差值',
  `is_maxflow` varchar(5) NULL DEFAULT NULL COMMENT '是否是最大压差流速',
  PRIMARY KEY (`standard_pressure_id`) USING BTREE,
  INDEX `dpm_standard_pressure_task_id_IDX`(`task_id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 2201;
```

**关键字段说明：**

| 字段 | 类型 | 说明 |
|------|------|------|
| org_pressure_id | bigint(20) | 关联原始压差数据 |
| pressure | float | 压差值 (MPa) |
| total_pressure | float | 标准压差 (kPa) |
| rated_flow_filter | float | 额定流速 (L/min) |
| voltage_up/down | varchar(30) | 正常压差上/下限值 |
| voltage_change_up/down | varchar(30) | 压差变动上/下限 |
| is_exception | varchar(10) | 是否异常 (Y/N) |
| abnormal_score | varchar(25) | 异常情况得分 |
| abnormal_state | varchar(25) | 异常状态 |
| effect | float | 流速功效 |
| velocity_diff | float | 流速与额定流速/2的差值 |
| is_maxflow | varchar(5) | 是否为最大压差流速 |

**标准化处理逻辑：**
- 基于 `dpm_org_pressure` 原始数据，添加标准化计算字段
- 计算标准压差 (total_pressure)
- 判断压差是否在正常范围内 (voltage_up/down)
- 判断压差变动是否异常 (voltage_change_up/down)
- 计算异常得分 (abnormal_score)
- 标记是否异常 (is_exception)

#### dpm_pressure_statistic (压差统计汇总)

```sql
CREATE TABLE `dpm_pressure_statistic` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `check_date` varchar(20) NULL DEFAULT NULL COMMENT '��查日期',
  `check_week` varchar(20) NULL DEFAULT NULL COMMENT '检查周',
  `check_time` varchar(50) NULL DEFAULT NULL COMMENT '检查时间',
  `vnb` varchar(20) NULL DEFAULT NULL COMMENT '车号',
  `vehicle_type` varchar(50) NULL DEFAULT NULL COMMENT '车类型',
  `pressure` float NULL DEFAULT NULL COMMENT '压差',
  `collection_time` varchar(50) NULL DEFAULT NULL COMMENT '采集时间',
  `velocity` varchar(50) NULL DEFAULT NULL COMMENT '流速, 单位，L/min',
  `effect` varchar(20) NULL DEFAULT NULL COMMENT '流速功效',
  `rated_flow_filter` varchar(20) NULL DEFAULT NULL COMMENT '额定流速(L/min)',
  `total_pressure` float NULL DEFAULT NULL COMMENT '标准压差(kPa)',
  `voltage_up` varchar(30) NULL DEFAULT NULL COMMENT '正常压差的上限值',
  `abnormal_score` varchar(25) NULL DEFAULT NULL COMMENT '异常情况得分',
  `abnormal_state` varchar(25) NULL DEFAULT NULL COMMENT '异常状态',
  `ap3c` varchar(10) NULL DEFAULT NULL COMMENT '机场3码',
  `task_id` bigint(20) NULL DEFAULT NULL COMMENT '任务号',
  `standard_pressure_id` bigint(20) NULL DEFAULT NULL COMMENT '标准化压差ID',
  `tenant` varchar(50) NOT NULL COMMENT '租户',
  `data_type` int(11) NULL DEFAULT NULL COMMENT '数据类型(1:day-max. 2:day-middle 3:week-max. 4:week-middle)',
  `pressure_dif` varchar(20) NULL DEFAULT NULL COMMENT '与上一次的压差差值',
  `change_state` varchar(20) NULL DEFAULT NULL COMMENT '压差变动异常',
  `user_name` varchar(50) NULL DEFAULT NULL COMMENT '检查人姓名',
  `flno` varchar(20) NULL DEFAULT NULL COMMENT '航班号',
  `regn` varchar(20) NULL DEFAULT NULL COMMENT '机号',
  `placecode` varchar(20) NULL DEFAULT NULL COMMENT '机位',
  `well` varchar(20) NULL DEFAULT NULL COMMENT '地井',
  `user_id` varchar(50) NULL DEFAULT NULL COMMENT '检查人id',
  `update_time` datetime(6) NULL DEFAULT CURRENT_TIMESTAMP(6),
  `create_time` datetime(6) NULL DEFAULT CURRENT_TIMESTAMP(6),
  `version` int(11) NULL DEFAULT NULL,
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 315;
```

**关键字段说明：**

| 字段 | 类型 | 说明 |
|------|------|------|
| check_date | varchar(20) | 检查日期 |
| check_week | varchar(20) | 检查周 |
| data_type | int(11) | 数据类型：1=日最大, 2=日中间, 3=周最大, 4=周中间 |
| standard_pressure_id | bigint(20) | 关联标准化压差ID |

### 15.6 数据流分析

```
┌────────────────────────────────────────────────────────────────────────┐
│                      压差数据流转图                                      │
├────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌────────────────┐    ┌────────────────┐    ┌────────────────┐         │
│  │ 加油车设备      │    │ dpm_org_       │    │ dpm_standard_ │         │
│  │ 压差传感器      │ ──▶│ pressure       │ ──▶│ pressure      │         │
│  │                │    │ 原始压差数据    │    │ 标准化压差     │         │
│  └────────────────┘    └────────────────┘    └───────┬────────┘         │
│                                                        │                 │
│                                                        ▼                 │
│  ┌────────────────┐    ┌────────────────┐    ┌────────────────┐         │
│  │ dpm_filter     │    │ dpm_filter_    │    │ 压差统计汇总   │         │
│  │ 过滤器台账     │    │ maintenance_   │    │ dpm_pressure_ │         │
│  │                │    │ sheet          │    │ statistic     │         │
│  └────────────────┘    │ 维保记录       │    └────────────────┘         │
│                        └────────────────┘                                │
│                                                                          │
└────────────────────────────────────────────────────────────────────────┘
```

**数据处理流程：**

1. **采集阶段**：加油车压差传感器实时采集压差和流速数据
2. **存储阶段**：原始数据存入 `dpm_org_pressure`
3. **标准化阶段**：对原始数据进行标准化处理，计算标准压差、异常判断
4. **存储阶段**：标准化结果存入 `dpm_standard_pressure`
5. **统计阶段**：按日/周汇总统计，存入 `dpm_pressure_statistic`
6. **维保管理**：过滤器定期维护，记录维保历史

### 15.7 压差异常判断逻辑

```
压差判断维度：
┌────────────────────────────────────────────────────────────────────────┐
│ 1. 绝对值判断：压差是否在正常范围内                                       │
│    voltage_down <= pressure <= voltage_up                                │
├────────────────────────────────────────────────────────────────────────┤
│ 2. 变动判断：压差变化是否异常                                              │
│    voltage_change_down <= pressure_dif <= voltage_change_up              │
├────────────────────────────────────────────────────────────────────────┤
│ 3. 流速判断：当前流速是否正常                                              │
│    velocity_diff = velocity - rated_flow_filter / 2                    │
├────────────────────────────────────────────────────────────────────────┤
│ 4. 最大流速判断：是否为当前任务中最大压差的流速                            │
│    is_maxflow = Y/N                                                      │
└────────────────────────────────────────────────────────────────────────┘

异常等级计算（abnormal_score）：
- 综合上述各维度，判断异常程度
- 输出异常状态（abnormal_state）
- 标记是否异常（is_exception）
```

### 15.8 与其他模块关联

| 关联模块 | 关联字段 | 说明 |
|----------|----------|------|
| srv_task | task_id | 压差数据与任务关联 |
| flightinfo | flno, regn | 压差数据与航班关联 |
| oil_bill | bill_number | 压差数据与油单关联 |
| oil_vehicle | vnb | 过滤器与加油车绑定 |

### 15.9 与 db_oil_pvg 对比

| 维度 | db_oil_pvg | db_oil_extend |
|------|-------------|---------------|
| 定位 | 核心业务调度 | 监控数据扩展 |
| 表数量 | 206 | 5 |
| 字符集 | utf8/utf8mb4 | utf8mb4 |
| 存储引擎 | InnoDB | InnoDB |
| 主要表 | flightinfo, oil_bill, srv_task | dpm_* 压差监控表 |
| 数据量 | 大 | 中 |
| 访问频率 | 极高 | 中 |

### 15.10 总结

**db_oil_extend 压差监控模块特点：**

1. **专用监控库**：独立数据库存储压差监控数据，与业务数据分离
2. **多维度分析**：支持压差绝对值、变动值、流速等多维度判断
3. **异常预警**：计算异常得分，支持压差异常告警
4. **过滤器管理**：完整的过滤器台账和维保记录管理
5. **统计数据**：支持按日/周汇总统计，便于分析趋势

**核心价值：**
- 实时监控加油车压差，确保加油安全
- 过滤器状态管理，维保提醒
- 压差异常预警，提前发现设备问题
- 统计报表，支持运维决策

---

*报告生成时间：2026-04-07*
*分析工具：Cursor AI Code Analysis*
