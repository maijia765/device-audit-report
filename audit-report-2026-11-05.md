# 设备审查报告

**审查对象：** OPPO PFZM10（Reno8 Pro+），Android 15 / ColorOS 15
**adb 采集时间：** 2026-11-06 01:13 — 02:30（CST）
**流量数据覆盖：** 2026-11-05 20:00 — 2026-11-06 04:00（CST，2 小时一桶，最后一桶在采集时仍在累积）
**报告生成时间：** 2026-11-06 01:25（CST）
**审查方式：** 仅通过 USB 连接的本机 adb，未 root，未安装任何应用到手机

---

## 0. 适用范围声明

本报告的全部结论，**只对这一台手机、这一个系统版本、这一段时间有效**。

- 设备型号：`PFZM10`
- Android 版本：`15`（API 35）
- 版本号：`PFZM10_15.0.0.2002(CN01)`
- 指纹：`OPPO/PFZM10/OP5281:15/AP3A.240617.008/S.25c134b-4311e-683b1:user/release-keys`
- 系统编译时间：`2026-09-08 16:05:29 CST`
- 本次开机时间：`2026-11-05 21:42`（adb 采集前约 3 小时 31 分）
- 已安装应用总数：410（其中第三方应用 60）

**不适用于：** 同型号的其他机器、其他 ColorOS 版本、OTA 升级后的状态、审查窗口之外的时段的任何结论。软件包内容与云端配置会随更新变化，本报告的静态结论在升级后需重新验证。

**结论的边界（客观限制）：**

| 手段 | 能看到 | 看不到 |
|---|---|---|
| APK 静态分析 | 代码里的域名、路径、表结构、权限声明 | 运行时的实际请求、加密后的内容、动态拼接的地址 |
| `dumpsys netstats` | 按 uid 的字节数（仅蜂窝网） | Wi-Fi 流量、共享 uid 内的分应用拆分、目标主机 |
| `cmd appops` | 权限的使用/拒绝记录与时间 | 具体调用的参数 |
| `dumpsys alarm` | 已排队的定时任务 | 已取消或已执行的定时任务 |
| `logcat` | 近期日志 | 无 root 无法读取的 `/data/data`、已被轮转删除的日志 |

未 root，因此 `/data/data/*` 下的 SQLite 队列、缓存、加密密钥均**未读取**。本报告所有"数据去向"的判断来自代码与配置，不代表已观测到实际发出的报文。

---

## 1. 联通自注册组件 `com.mobiletools.systemhelper`

### 1.1 身份标识

| 项目 | 值 |
|---|---|
| 包名 | `com.mobiletools.systemhelper` |
| uid | `10209` |
| APK 路径 | `/product/priv-app/SystemHelper-oppo-20240717/SystemHelper-oppo-20240717.apk` |
| 版本 | `3.0.1-20240717`（versionCode 31，targetSdk 33） |
| 首次安装 | `2022-05-09 00:12:32` |
| 最后更新 | `2024-10-22 03:12:28` |
| 安装来源 | `installerPackageName=null` |
| 所在分区 | `/product/priv-app`（系统分区，非 root 不可卸载） |

### 1.2 权限

**已授予（granted=true）：**

```
android.permission.INTERNET
android.permission.FOREGROUND_SERVICE
android.permission.RECEIVE_BOOT_COMPLETED
android.permission.READ_PRECISE_PHONE_STATE
android.permission.READ_PRIVILEGED_PHONE_STATE
android.permission.PACKAGE_USAGE_STATS
android.permission.ACCESS_LOCATION_EXTRA_COMMANDS
android.permission.CHANGE_WIFI_STATE
android.permission.ACCESS_WIFI_STATE
android.permission.ACCESS_NETWORK_STATE
android.permission.SCHEDULE_EXACT_ALARM
android.permission.QUERY_ALL_PACKAGES
```

**定位权限（默认授予，未经用户交互）：**

```
ACCESS_FINE_LOCATION        granted=true  flags=[GRANTED_BY_DEFAULT]
ACCESS_COARSE_LOCATION      granted=true  flags=[GRANTED_BY_DEFAULT]
ACCESS_BACKGROUND_LOCATION  granted=true  flags=[GRANTED_BY_DEFAULT|RESTRICTION_SYSTEM_EXEMPT|RESTRICTION_UPGRADE_EXEMPT]
```

注意 `RESTRICTION_SYSTEM_EXEMPT`：后台定位的系统限制被显式豁免。

**声明但运行时 appops 为 ignore：**

```
READ_CONTACTS / WRITE_CONTACTS / READ_CALL_LOG / WRITE_CALL_LOG
READ_SMS / RECEIVE_SMS / SEND_SMS / RECEIVE_MMS
CAMERA / RECORD_AUDIO / READ_CALENDAR
```
（`cmd appops get com.mobiletools.systemhelper` 全部返回 `ignore`）

### 1.3 组件与触发条件

`dumpsys package` 中声明的接收器与服务：

| 组件 | 触发 |
|---|---|
| `control.OnBootReceiver` | `BOOT_COMPLETED`、`QUICKBOOT_POWERON` |
| `control.ConnectStateReceiver` | `CONNECTIVITY_CHANGE` |
| `control.SIMStateReceiver` | `SIM_STATE_CHANGED` |
| `control.ScheduleRegisterReceiver` | 定时闹钟 |
| `control.AppInfoRegisterReceiver` | 定时闹钟（应用清单上报） |
| `control.SignalInfoRegisterReceiver` | 定时闹钟（信号/基站上报） |
| `control.GetSignalInfoReceiver` | 定时闹钟 |
| `service.RegisterHelperService` | 绑定 `com.mobiletools.systemhelper.service.IRegisterHelperService` |
| `service.RegisterIntentService` | IntentService |
| `service.RegisterJobService` | JobScheduler |
| `Activity.TestActivity` | 测试入口 |

`dumpsys alarm` 中保留的闹钟快照（本次开机前）：

```
*walarm*:com.mobiletools.systemhelper/.control.SignalInfoRegisterReceiver
*walarm*:com.mobiletools.systemhelper/.control.ScheduleRegisterReceiver
*walarm*:com.mobiletools.systemhelper/.control.AppInfoRegisterReceiver
*walarm*:com.mobiletools.systemhelper/.control.GetSignalInfoReceiver
```

即：开机启动 + 网络变化 + SIM 变化 + 三类周期闹钟，均为 **WAKEUP** 类型（可唤醒休眠）。

### 1.4 上报目标

**代码中仅有的两个 URL（字节级精确匹配）：**

```
http://dm.wo.com.cn:18080/register
https://api.map.baidu.com/sdkcs/verify
```

- 第一个是中国联通的自注册接口，**明文 HTTP，非标准端口 18080**。
- 第二个是百度定位 SDK 的鉴权校验地址，代码中含 `com.baidu.android.lbs.API_KEY` 相关字符串，以及 `LocUploadThreadManager`、`getBaseStationLatitude`、`getBaseStationLongitude`、`openGPS`、`定位成功/定位失败` 等。
- DNS：`dm.wo.com.cn` → `180.97.168.69`。

### 1.5 上报内容（代码字段级证据）

**手机身份字段：** `IMEI`、`IMSI`、`ICCID`（含 `IMEI1/IMEI2`、`ICCID1/ICCID2`、`IMSI1/IMSI2` 双卡槽）

**基站身份字段**（错误日志 `imei/mnc/imsi/tac/cid/pci/networkType/mncch is null!` 反推）：
`mnc`、`mcc`、`tac`、`cid`、`pci`、`networkType`、`mncch`、`signalStrength`

**位置字段：** `LATITUDE`、`LONGITUDE`、`Latitude`、`Longitude`

**应用清单字段：** `AppInfo`、`getInstalledApplications`、`getPackageName`、`packageName`、`isInstalled`、`versionName`

**设备字段：** `model`、`OSVersion`

**上报类型：** 日志中出现 `type A-appinfo` 与 `type R-appinfo` 两种，分别受配置项 `appInfoInterval` / `appInfoInterval2` 控制。

**加密：** 字符串 `AES加密密文`、`RSA加密密文`、`AES/CBC/PKCS5PADDING`、`RSAHelper`、MIME 类型 `application/encrypted-json`。**即上报体在传输前已经过 AES/RSA 加密，报文内容无法从网络层还原。**

### 1.6 本地存储结构

`classes*.dex` 中的 SQLite DDL：

```sql
CREATE TABLE IF NOT EXISTS dmregister (ID integer primary key autoincrement, jsonString text);
CREATE TABLE IF NOT EXISTS signal     (ID integer primary key autoincrement, jsonString text);
```

即存在**本地积压队列**：上报失败或未到时机时先落库，之后再补发。

另有百度定位 SDK 自带的表：

```sql
CREATE TABLE IF NOT EXISTS AP (id LONG PRIMARY KEY, x DOUBLE, y DOUBLE, r INTEGER, cl DOUBLE, timestamp INTEGER, frequency INTEGER DEFAULT 0);
CREATE TABLE IF NOT EXISTS CL (id LONG PRIMARY KEY, x DOUBLE, y DOUBLE, r INTEGER, cl DOUBLE, timestamp INTEGER, frequency INTEGER DEFAULT 0);
CREATE TABLE IF NOT EXISTS BLACK (name VARCHAR(100) PRIMARY KEY);
```

### 1.7 审查窗口内的实际行为

| 观测项 | 结果 |
|---|---|
| `cmd appops get ... FINE_LOCATION` | `allow; time=+3h31m ago` → 最近一次定位访问 **2026-11-05 21:48:50**（开机后约 6 分钟） |
| `rejectTime` | `+47d` → 约 47 天前有过一次定位请求被拒 |
| `dumpsys usagestats` | `appLaunchCount=0`，`lastTimeUsed=1970-01-01` → 从未以前台界面启动过 |
| `ps -A` | 审查时无该包进程 |
| `dumpsys alarm` 当前待触发 | 0 条 |
| 已注册广播接收器 | 0 条 |
| 运行中的服务 | 0 个 |
| 蜂窝流量（20:00–22:00 桶） | 接收 1,492 B / 发送 3,980 B |
| `logcat` | 仅 `2026-11-06 00:41:40` 的后台 dexopt 记录，无进程启动事件（日志已轮转） |

### 1.8 当前包状态

```
User 0: installed=true hidden=false suspended=true stopped=true enabled=3
Suspend params: suspendingPackage=<0>com.android.shell
```

- `enabled=3` = `COMPONENT_ENABLED_STATE_DISABLED_USER`
- 出现在 `pm list packages -d` 的禁用列表中
- `suspended=true`，记录的执行方为 `com.android.shell`（即通过 adb 施加）
- 该状态在重启后是否保持，本次开机周期内尚未观测（本周期内未再重启）

### 1.9 先前测试记录（本次窗口之外）

在此前的同一台设备测试中：在系统设置界面停用该应用后重启，服务被重新拉起。该记录的日期在 2026-11-04 至 2026-11-05 之间，具体时间未留存日志。

---

## 2. 全机检索结果

### 2.1 域名检索

对全部 410 个已安装 APK 的 `classes*.dex` 与 `resources.arsc` 解压后逐字节检索（不对压缩包直接 grep，避免漏检）：

| 域名 | 结果 | 命中包 |
|---|---|---|
| `dm.wo.com.cn` | **1 个包** | `com.mobiletools.systemhelper` |
| `cloud.risk.envparam.net` | **0 个包**，DNS 返回 NXDOMAIN | — |
| `safecollector.com` | **0 个包**；该域名的 A 记录为 37.9.175.18，NS 为 `ns1/ns2.loopiagroup.com`，属停放域名，443 端口无 TLS 响应 | — |

方法说明：APK 是 zip 压缩包，直接对 `.apk` 文件执行 `grep` 会因压缩而漏检。对 SystemHelper 的 APK 直接 `grep -c dm.wo.com.cn` 返回 0，而 `unzip -p "$apk" "classes*.dex" | grep` 可命中——本报告全部域名结论均采用后者。

`com.heytap.mcs`（OPPO 推送服务）的 dex 中含若干其他厂商的连通性探测地址（`connectivitycheck.platform.hicloud.com`、`connectivitycheck.platform.hihonorcloud.com`、`wifi.vivo.com.cn` 等），位于 **captive portal（门户探测）列表**上下文中，用于判断网络是否可用，不是数据回传地址。该判断与 `settings get global captive_portal_*` 返回 `null` 相符。

### 2.2 录屏与画面采集

`dumpsys media_projection` 返回 `null`，当前**无活动的屏幕投射（录屏）会话**。`CAPTURE_VIDEO_OUTPUT` 的唯一持有者为 `com.android.shell`（见第 3 节）。

### 2.3 本地数据库

手机管家（`com.coloros.phonemanager`）的 DDL 中有

```sql
CREATE TABLE IF NOT EXISTS `fraud_detect_record_table` (
  `_id` INTEGER PRIMARY KEY AUTOINCREMENT NOT NULL,
  `is_synthetic` INTEGER NOT NULL,
  `is_voip` INTEGER NOT NULL,
  `code` INTEGER NOT NULL,
  `phone_num` TEXT,
  `pack_name` TEXT,
  `is_ai_voice` INTEGER NOT NULL,
  `fraud_risk_type` INTEGER NOT NULL,
  `time` INTEGER NOT NULL
);
```

同库另有 `ignoredRecord`、`questionnaires` 表。该数据库位于 `/data/data/com.coloros.phonemanager/`，无 root 无法读取。

---

## 3. 采集类权限的实际持有者

对 410 个包逐一执行 `dumpsys package | grep '<权限>: granted=true'`，去重后的结果：

| 权限 | 持有包数 | 持有者 |
|---|---|---|
| `CAPTURE_VIDEO_OUTPUT` | **1** | `com.android.shell` |
| `CAPTURE_AUDIO_OUTPUT` | 22 | `com.android.incallui`、`com.android.phone`、`com.android.systemui`、`com.android.shell`、`com.android.vendors.bridge.softsim`、`com.mediatek.ims`、`com.mediatek.telephony`、`com.heytap.speechassist`、`com.oplus.cast`、`com.oplus.nhs`、`com.oplus.vdc`、`com.coloros.translate.engine`、`com.google.android.gms/gsf`、`com.microsoft.deviceintegrationservice`、`vendor.mediatek.iwlanservice` 等 |
| `MANAGE_MEDIA_PROJECTION` | 18 | `com.android.incallui`、`com.android.systemui`、`com.android.phone`、`com.android.stk`、`com.android.vendors.bridge.softsim`、`com.oplus.linker`、`com.oplus.deepthinker`、`com.oplus.ocar`、`com.oplus.nhs`、`com.microsoftsdk.crossdeviceservicebroker` 等 |
| `ACCESS_SURFACE_FLINGER` | 14 | `com.android.incallui`、`com.android.systemui`、`com.android.phone`、`com.android.shell`、`com.android.vendors.bridge.softsim`、`com.oplus.beaconlink` 等 |
| `READ_FRAME_BUFFER` | 57 | 含 `systemui`、`launcher`、`settings`、`pscanvas`、`atlas`、`obrain`、`logkit`、`trafficmonitor`、`healthservice`、`heytap.mcs` 等 |

**当前无活动的 MediaProjection 会话**（`dumpsys media_projection` 返回 `null`）。

---

## 4. OPPO 侧遥测/风控端点

### 4.1 归属：三个域名族，三家持证主体

同一批端点分属三个域名族，TLS 证书主体不同：

| 域名族 | 证书 O 字段 | CN | 系统内承载 |
|---|---|---|---|
| `*.allawntech.com` | OPPO广东移动通信有限公司 | `www.oppo.com` | 推理（Odin）、运营内容 |
| `*.wanyol.com` | 广东欢太科技有限公司 | `www.wanyol.com` | 推理备用通道、部署、测试环境 |
| `*.heytapmobi.com` / `*.heytap.com` / `*.heytapdownload.com` | 广东欢太科技有限公司（`heytapdownload.com` 为「深圳市欢太科技有限公司」，CN `nearme.com.cn`） | `www.heytap.com` / `*.heytapmobi.com` | 统计、风控、验证码、反馈、文档 |

端点不是"每个包各自打各自的域名"，而是**若干公共 SDK 被大量系统应用打包**，同一域名因此出现在几十个包里。按"域名 → 包"列表无法区分这一点，故下面按 SDK 分组。

### 4.2 端点清单（按实际性质分组）

"原列表"列指该主机是否在用户先前给出的 15 个域名中。

| 组 | 端点 | 命中包数 | 代码中的形态 | 原列表 |
|---|---|---|---|---|
| **A 推理上报** | `odin-inference-cn.allawntech.com` | 3 | `phonemanager` 有 3 条完整路径；`speechassist`、`dmp` 仅根路径 | — |
| | `odin-deployment-cn.wanyol.com` | 3 | `phonemanager` 2 条完整路径；`dmp` 为 `/crypt/`；`deepthinker` 仅根路径 | — |
| **B 设备风控** | `omes-sec-stdsrp-cn.heytapmobi.com` | 1 | 6 条完整路径（`std-srp/*`、`std-passkey/login_chk`） | ✓ |
| **C 密钥/票据** | `datasec-kmsex-cn.heytapmobi.com` | 10 | `/v1/` 及 4 条 `getkms*` / `getserviceticket` 路径 | ✓ |
| **D 统计埋点** | `cubefs-norths3-cn.heytapmobi.com` | 1 | 仅根路径；S3 兼容对象存储 | — |
| | `mdp-usertrace-cn.heytapmobi.com` | 9 | 仅根路径，9 个包内均无路径 | ✓ |
| **E OCR** | `aitool-cuiocr-cn.heytapmobi.com` | 6 | `nps/cui/api/v1/ocr_file`（5 个包同一条） | ✓ |
| **F 意图** | `sense-intent-cn-01.heytapmobi.com` | 3 | 仅根路径 | ✓ |
| **G 运营内容** | `traffic-operation-cn.allawntech.com` | 21 | 仅 `ocrscanner` 一条路径 `operate_service/v1/prod/get_content` | — |
| **H 服务转发** | `oms-forward-cn.heytapmobi.com` | 13 | `/api`、`/api/forward/service-forward/appstore-download-token` | — |
| **I 验证码** | `captcha-sec.heytapmobi.com` | 4 | `dx-captcha/index.js`、`libs/greenseer.js`、`udid/m1` 等静态资源 | ✓ |
| | `captcha-uc.heytapmobi.com` | 6 | `/api/captcha/v1.0/captcha-page` | — |
| **J 安全校验** | `virusinfo-sec.heytapmobi.com` | 4 | `viapi/v1/url/check`（`ocrscanner`、`camera`） | ✓ |
| **K 测速** | `security-speed.heytapdownload.com` | 1 | 仅根路径；响应头暴露 S3 风格 bucket `/speed-test-cn-s1` | — |
| **L 用户反馈** | `rest-cn-feedback.heytapmobi.com` | 49 | `feedback-app/*` 5 条，含 `upload-feedback`、`putobjectapp` | ✓ |
| **M URL 白名单** | `fraud-feedback-cn.allawntech.com` | 1 | 仅裸主机名，位于 `resources.arsc` 的白名单条目；`classes*.dex` 中无该主机路径 | ✓ |
| **N 页面跳转** | `sport.health.heytapmobi.com` | 2 | 1 条 `/h5/watch-scancode-connect/index.html` | ✓ |
| **O 协议文档** | `muc.heytap.com` | 25 | 41 条 URL 全部为 `/document/`、`/agreement/` 静态 HTML | ✓ |
| **P 问卷分发** | `wj.heytap.com` | 3 | `/r/<8位ID>` 10 条 | ✓ |
| **Q 非 HTTP** | `carlink.heytap.com` | 4 | 8 条全为 `card://` URI scheme，非 HTTP 主机 | ✓ |
| **不存在** | `cloud.risk.envparam.net` | 0 | NXDOMAIN | ✓ |
| | `safecollector.com` | 0 | 停放域名，443 无 TLS | ✓ |

即：15 个域名中，**A–L 中属于原列表的 9 个是真实存在的网络端点**；`muc`、`wj`、`sport.health` 是静态页面或问卷站，`carlink` 是 URI scheme，`fraud-feedback` 只出现在 ARSC 白名单里，两个域名不存在。

### 4.3 各组代码级依据

**A 组 · 推理上报**

`com.coloros.phonemanager`（uid 10156，17.7.10）内的 `com/oplus/phonemanager/aivoicecalldetect/` 包（含 `antifraudhome/SecurityHomeRepository`、`AiCapabilityStateObserver` 等）：

```
https://odin-inference-cn.allawntech.com/speechAntiFraud/infer
https://odin-inference-cn.allawntech.com/speechSynthDetect/infer
https://odin-inference-cn.allawntech.com/wifiPhotos/infer
https://odin-deployment-cn.wanyol.com/speechAntiFraud/infer
https://odin-deployment-cn.wanyol.com/wifiPhotos/infer
```

同 dex 中另有调试目录字符串 `/sdcard/ai_voice_detect_debug/`、`/sdcard/ai_voice_detect_synthetic_debug/`、`/recordings/`，以及 `/v1/feedback/false-positive`、`/v1/feedback/file/public-key`、`/v1/feedback/file/upload`、密钥别名 `fraud-feedback-file-ecdh`。

`speechAntiFraud` / `speechSynthDetect` / `wifiPhotos` 三个路径在采集时均返回：

```
{"code":3000404,"message":"请求头缺失"}
```

即路径存在、需要请求头鉴权。`odin-deployment-cn.wanyol.com/crypt/` 同一时刻返回 `502 backend connect timeout`。

**B 组 · 设备风控（Osense SRP）**

`com.heytap.htms`（uid 10148，2.3.1_e02f13c_261010_btIn-shop_dom）：

```
/v1/std-srp/async_report_data
/v1/std-srp/periodic_report_data
/v1/std-srp/periodic_report_device_data
/v1/std-srp/raw_report_data
/v1/std-srp/request_biz_token
/v1/std-srp/turing_upload_data
/v1/std-passkey/login_chk
```

同 APK 含 `com/oplus/omes/srp/util/wb/`（`CipherPackage`、`CipherUtil`、`StringUtil`）、`com/oplus/osense/OsenseResEventClient`、`com/tencent/turingfd/sdk/tianyu`（腾讯图灵盾），以及 `getVmApplist`、`hasSandBoxInfo`、`hookRiskNum`、`houseIdentifier`、`asyncPeriodicGather` 等标识符。

**C / D 组 · 统计 SDK（DCS）**

`com.oplus.statistics.rom`（uid 10092，5.9.17.01）的完整 URL 只有：

```
https://datasec-kmsex-cn.heytapmobi.com/v1/
https://cubefs-norths3-cn.heytapmobi.com
```

类名含 `CubeFsStorage` / `CubeFsUpload` / `CubeFsDownload`、`AmazonS3 putObject`、`Bucket`，`cubefs-norths3-cn.heytapmobi.com` 返回 `403 InvalidAccessKeyId` —— 是 S3 兼容对象存储，用于放文件而非收上报。

真正的上报地址**不在静态字符串里**：APK 内有 SQLite 表

```sql
CREATE TABLE IF NOT EXISTS dynamic_url (
  common_appid INTEGER PRIMARY KEY, data_type INTEGER NOT NULL DEFAULT 0, dynamic_url Text);
```

并有 `getDynamicUrlTypeList`、`uploadHost=[`、`BziuploadHost =` 等运行时拼装逻辑。同库还有 `app_list_info`、`preinstall_applist`、`uninstall`、`page_visit_info`、`dau_info`、`common_info`、`period_data_info` 等表，字段包括 `appPackage`、`otaVersion`、`is_new_device`、`is_first_time_report`、`factory_version`。

另有被字符串截断的 `http://apisnd.heytap`、`https://api-snake.`，不完整，无法确定主机名。

**G 组 · 运营内容**

`traffic-operation-cn.allawntech.com` 出现在 21 个包，但只有 `com.coloros.ocrscanner` 带路径：

```
https://traffic-operation-cn.allawntech.com/operate_service/v1/prod/get_content
```

该路径返回 `{"code":100004,"message":"必要参数不能为空！"}`。其余 20 个包只含根路径 `https://traffic-operation-cn.allawntech.com/`。

**E 组 · OCR**

`aitool-cuiocr-cn.heytapmobi.com/nps/cui/api/v1/ocr_file`，5 个包共用（`colordirectservice`、`directui`、`gallery3d`、`ocrscanner`、`phonemanager`）。GET 请求返回：

```
405 — Method Not Allowed / Method GET not allowed for URL /cui/api/v1/ocr_file
```

即只接受上传，不接受查询。

**L 组 · 用户反馈**

`rest-cn-feedback.heytapmobi.com` 的 5 条路径：

```
/feedback-app/api-new/feedback/fms/upload-feedback
/feedback-app/ocs/putobjectapp
/feedback-app/faqcateType/checkFaqCateAndHotFaq
/feedback-app/feedback/queryNoReadCountNew
/feedback-app/ocs/putobjectapp
```

`/feedback-app/ocs/putobjectapp` 返回 `{"msg":"失败","status":"0001","subCode":10001,"subMsg":"未知异常","success":false}`，为活跃服务。

**K 组 · 测速**

`security-speed.heytapdownload.com` 仅被 `phonemanager` 引用，对应类为 `com/oplus/phonemanager/networkdetect/speed/DownloadSpeedTest`（`networkdetect/repository/NetworkDiagnosisRepositoryImpl`）。访问返回：

```
400 MissingSecurityElement ... <Resource>/speed-test-cn-s1</Resource>
```

即下载测速用的对象存储 bucket，不是上报接口。

**实测响应（含 N / O / P / Q 等非上报类主机）**

| 主机 | 响应 |
|---|---|
| `muc.heytap.com` | 静态文档页（`/document/heytap/privacyPolicy/...`、`/agreement/...`） |
| `wj.heytap.com/r/1prTMzN3` | HTML，标题「通知使用体验满意度调研」 |
| `sport.health.heytapmobi.com/h5/watch-scancode-connect/index.html` | HTML，标题「OPPO 健…」 |
| `fraud-feedback-cn.allawntech.com/` | `404 path location is not configured` |
| `carlink.heytap.com` | NXDOMAIN；代码中全部为 `card://carlink.heytap.com/control?operation=...` 形式 |
| `omes-sec-stdsrp-cn.heytapmobi.com/` | `404 path location is not configured` |
| `datasec-kmsex-cn.heytapmobi.com/v1/` | `200`，返回 `{"IP":"10.32.124.106","NumCPU":64,"NumGoroutine":1486}` |

### 4.4 DNS 与 TLS 证书主体

| 主机 | A 记录 | 证书 O | 证书 CN |
|---|---|---|---|
| fraud-feedback-cn.allawntech.com | 14.29.106.224 | OPPO广东移动通信有限公司 | www.oppo.com |
| odin-inference-cn.allawntech.com | 106.3.16.189 | OPPO广东移动通信有限公司 | www.oppo.com |
| traffic-operation-cn.allawntech.com | 106.39.205.81 | OPPO广东移动通信有限公司 | www.oppo.com |
| odin-deployment-cn.wanyol.com | 14.17.89.159 | 广东欢太科技有限公司 | www.wanyol.com |
| omes-sec-stdsrp-cn.heytapmobi.com | 119.147.175.246 | 广东欢太科技有限公司 | www.heytap.com |
| datasec-kmsex-cn.heytapmobi.com | 119.147.175.139 | 广东欢太科技有限公司 | www.heytap.com |
| virusinfo-sec.heytapmobi.com | 106.39.205.196 | 广东欢太科技有限公司 | www.heytap.com |
| security-speed.heytapdownload.com | 125.77.162.172 | 深圳市欢太科技有限公司 | nearme.com.cn |
| captcha-sec.heytapmobi.com | 119.147.175.29 | 广东欢太科技有限公司 | www.heytap.com |
| captcha-uc.heytapmobi.com | 106.3.18.68 | 广东欢太科技有限公司 | www.heytap.com |
| oms-forward-cn.heytapmobi.com | 59.37.130.5 | 广东欢太科技有限公司 | www.heytap.com |
| mdp-usertrace-cn.heytapmobi.com | 119.147.98.71 | 广东欢太科技有限公司 | *.heytapmobi.com |
| aitool-cuiocr-cn.heytapmobi.com | 119.147.98.66 | 广东欢太科技有限公司 | *.heytapmobi.com |
| sense-intent-cn-01.heytapmobi.com | 106.120.154.125 | 广东欢太科技有限公司 | www.heytap.com |
| rest-cn-feedback.heytapmobi.com | 119.147.175.124 | 广东欢太科技有限公司 | www.heytap.com |
| sport.health.heytapmobi.com | 106.120.154.69 | 广东欢太科技有限公司 | www.heytap.com |
| cubefs-norths3-cn.heytapmobi.com | 49.7.118.32 | 广东欢太科技有限公司 | *.heytapmobi.com |
| muc.heytap.com | 36.110.185.46 | 广东欢太科技有限公司 | www.heytap.com |
| wj.heytap.com | 49.7.118.132 | 广东欢太科技有限公司 | www.heytap.com |
| carlink.heytap.com | **NXDOMAIN** | — | — |
| dm.wo.com.cn | 180.97.168.69 | — | — |
| cloud.risk.envparam.net | **NXDOMAIN** | — | — |
| safecollector.com | 37.9.175.18（停放域名，NS `ns1/ns2.loopiagroup.com`，443 无 TLS） | — | — |

### 4.5 敏感权限的默认授予状态

**`com.heytap.htms`（uid 10148）**

| 权限 | 状态 |
|---|---|
| `ACCESS_FINE_LOCATION` | granted=true，`GRANTED_BY_DEFAULT` |
| `ACCESS_COARSE_LOCATION` | granted=true，`GRANTED_BY_DEFAULT` |
| `ACCESS_BACKGROUND_LOCATION` | granted=true，`GRANTED_BY_DEFAULT` + 三类豁免 |
| `QUERY_ALL_PACKAGES` | granted=true |
| `ACCESS_WIFI_STATE` / `CHANGE_WIFI_STATE` | granted=true |
| `POST_NOTIFICATIONS` | granted=false |

**`com.oplus.statistics.rom`（uid 10092）**

| 权限 | 状态 |
|---|---|
| `ACCESS_FINE_LOCATION` / `COARSE` | granted=true，`GRANTED_BY_DEFAULT` |
| `ACCESS_BACKGROUND_LOCATION` | granted=true，`GRANTED_BY_DEFAULT` |
| `QUERY_ALL_PACKAGES` | granted=true |
| `READ_PRIVILEGED_PHONE_STATE` | granted=true |
| `com.oplus.permission.safe.PHONE` / `com.oppo.permission.safe.PHONE` | granted=true |

同 APK 含 `DeviceIdContentProvider`、`DeviceIdUtil`、`DuidRatioUtils`、`AppIdDictionary*`、`EVENT_ACTIVITY_ENTER/EXIT` 等类。

**`com.heytap.mcs`（uid 1000）**

| 权限 | 状态 |
|---|---|
| `ACCESS_FINE_LOCATION` / `COARSE` / `BACKGROUND_LOCATION` | granted=true，`SYSTEM_FIXED` + `GRANTED_BY_DEFAULT` |
| `READ_WIFI_CREDENTIAL`、`LOCATION_BYPASS`、`LOCATION_HARDWARE` | granted=true |
| `QUERY_ALL_PACKAGES` | granted=true |

由于运行在 uid 1000（system），其流量与 `system_server` 合并统计，无法单独核算。

**`com.coloros.phonemanager`（uid 10156）appops 全部为 `ignore`：**

```
READ_CALL_LOG / READ_SMS / RECORD_AUDIO / CAMERA
FINE_LOCATION / COARSE_LOCATION / READ_CONTACTS
READ_PHONE_STATE / GET_ACCOUNTS / READ_EXTERNAL_STORAGE
```

同一 APK 中的界面文案（资源字符串）：

```
本应用需要收集您的通话记录
AI通话防诈功能仅保留识别到有风险的通话记录
诈骗举报中读取并让用户上报诈骗通话记录
从本机通话记录中选择被举报号码
设置-隐私与安全-全部权限-通话记录
```

即：该功能的读取权限当前处于关闭状态，上述文案描述的能力在本机配置下不成立。

**`com.coloros.colordirectservice`（uid 10134）**

| 权限 | 状态 |
|---|---|
| `READ_FRAME_BUFFER` | 声明，**granted=false** |
| `MANAGE_EXTERNAL_STORAGE` | granted=true |
| `QUERY_ALL_PACKAGES` | granted=true |
| `CAPTURE_VIDEO_OUTPUT` / `CAPTURE_AUDIO_OUTPUT` / `MANAGE_MEDIA_PROJECTION` / `ACCESS_SURFACE_FLINGER` | **未声明** |

---

## 5. 实际流量（蜂窝网，`dumpsys netstats detail`）

统计方式：`bucketDuration=7200`（2 小时一桶），本地时间。**仅含蜂窝网（ccmni2），不含 Wi-Fi；共享 uid 会合并多个包的流量。**

| uid | 包 | 11-05 20:00 | 11-05 22:00 | 11-06 00:00 | 11-06 02:00（未满） |
|---|---|---|---|---|---|
| 10092 | `com.oplus.statistics.rom` | 17,597 / 32,928 | 153,105 / **247,859** | 203,461 / **358,312** | 23,222 / 47,366 |
| 10148 | `com.heytap.htms` | 35,381 / 23,885 | 102,857 / 18,826 | 45,295 / 39,553 | 4,940 / 3,264 |
| 10156 | `com.coloros.phonemanager` | 5,418 / 3,203 | 26,403 / 11,714 | — | — |
| 10134 | `com.coloros.colordirectservice` | — | 9,213 / 3,493 | — | — |
| 10209 | `com.mobiletools.systemhelper` | 1,492 / 3,980 | — | — | — |
| 10208 | `com.coloros.sceneservice` | 4,511 / 2,824 | — | — | — |
| 1000 | `com.heytap.mcs` | 65,968 / 28,272 | 41,092 / 23,839 | 26,549 / 22,640 | 42,519 / 18,163 |

单元格格式为 **接收 / 发送**（字节）。uid 1000 与 `system_server` 合并统计，无法拆分到 `com.heytap.mcs`。

观测到的特征：

1. `com.oplus.statistics.rom` 的上行持续高于下行，22:00、00:00、02:00 三个桶分别在 248 KB、358 KB、47 KB 量级（最后一桶未满）。其 APK 内没有任何完整的上报 URL（只有 KMS 与对象存储两条），上报地址由 `dynamic_url` 表在运行时下发。**该流量的具体接收方无法从静态分析确定。**
2. `com.heytap.htms` 的流量以**下行**为主（102,857 B 接收 / 18,826 B 发送），与"周期上报设备数据"的方向相反；其上行绝对量在 3.2–39.5 KB 之间。
3. `com.mobiletools.systemhelper` 只在 20:00–22:00 桶有记录（1,492 / 3,980 B）；停用动作发生于 21:48 前后，之后各桶均无记录。
4. `com.coloros.phonemanager` 的 appops 敏感项全部为 `ignore`，其流量为 3–12 KB 级别。
5. `com.oplus.deepthinker`、`com.oplus.metis`、`com.oplus.dmp`、`com.coloros.directui`、`com.heytap.speechassist` 在 `dumpsys netstats` 中**没有任何记录**（该表只列有流量的 uid），即本次窗口内无蜂窝流量。

---

## 6. 证据附录（可复现命令）

```bash
# 设备身份
adb shell getprop ro.product.model
adb shell getprop ro.build.fingerprint
adb shell getprop ro.build.display.id

# 包清单与 uid
adb shell pm list packages -f
adb shell pm list packages -U

# 包详情 / 权限 / 启用状态
adb shell dumpsys package com.mobiletools.systemhelper
adb shell dumpsys package com.coloros.colordirectservice
adb shell cmd appops get com.mobiletools.systemhelper
adb shell pm list packages -d

# 组件与定时任务
adb shell dumpsys alarm
adb shell dumpsys activity broadcasts
adb shell dumpsys activity services com.mobiletools.systemhelper

# 录屏 / 采集权限持有者
adb shell dumpsys media_projection
adb shell 'for p in $(pm list packages | sed "s/package://"); do \
  dumpsys package $p | grep -q "android.permission.CAPTURE_VIDEO_OUTPUT: granted=true" && echo $p; done'

# 流量
adb shell dumpsys netstats detail > /tmp/netstats.txt

# APK 内 URL 与串（在设备上解压后检索，避免对压缩包直接 grep）
adb shell 'APK=$(pm path com.mobiletools.systemhelper | sed "s/package://"); \
  unzip -p "$APK" "classes*.dex" | strings -n 4 | grep -a -E "https?://"'

# 全机字节级域名检索（注意：pm list packages -f 每个包只给出一条 APK 路径，
# 对 htms 这类 split 包必须用 pm path 取全部 APK，否则会漏检）
adb shell 'for pkg in $(pm list packages | sed "s/package://"); do
  for apk in $(pm path $pkg | sed "s/package://"); do
    unzip -p "$apk" "classes*.dex" "resources.arsc" 2>/dev/null | grep -aq "dm.wo.com.cn" && echo "HIT $pkg $apk"
  done
done'

# DNS 与证书归属
getent ahostsv4 <host>
openssl s_client -connect <host>:443 -servername <host> </dev/null 2>/dev/null | openssl x509 -noout -subject
```

---

## 7. 客观事实汇总

1. 设备为**非运营商定制机**，同时预置了联通自注册组件 `com.mobiletools.systemhelper`，位于 `/product/priv-app`，不可卸载。
2. 该组件的上报目标是 `http://dm.wo.com.cn:18080/register`，**明文 HTTP**；上报体为 AES/RSA 加密的 `application/encrypted-json`。
3. 该组件默认获得前台+后台精确定位；本次开机后约 6 分钟有一次定位访问记录；`appLaunchCount=0`。
4. 本次审查窗口内，该组件已处于 `DISABLED_USER` + `stopped` + `suspended` 状态，无进程、无待触发闹钟、无已注册接收器；蜂窝流量仅出现在停用之前的 20:00–22:00 桶。
5. 全部 410 个已安装包的 `classes*.dex` 与 `resources.arsc` 中，`dm.wo.com.cn` 仅出现在 `com.mobiletools.systemhelper`；`cloud.risk.envparam.net` 与 `safecollector.com` 均 0 命中。
6. 手机管家的本地库中有 `fraud_detect_record_table`（含 `phone_num`、`is_ai_voice`、`fraud_risk_type` 等字段），位于不可读的 `/data/data/com.coloros.phonemanager/`。
7. 当前无活动的 MediaProjection 会话；`CAPTURE_VIDEO_OUTPUT` 唯一持有者为 `com.android.shell`；`com.coloros.colordirectservice` 未声明 `CAPTURE_*` / `MANAGE_MEDIA_PROJECTION` / `ACCESS_SURFACE_FLINGER`。
8. 用户列出的 15 个域名中：9 个是真实网络端点（`virusinfo-sec`、`datasec-kmsex`、`captcha-sec`、`omes-sec-stdsrp`、`mdp-usertrace`、`aitool-cuiocr`、`sense-intent`、`rest-cn-feedback`、`fraud-feedback`）；`muc`、`wj`、`sport.health` 为静态页面或问卷站；`carlink` 仅为 `card://` URI scheme 且 DNS 不解析；`cloud.risk.envparam.net` 与 `safecollector.com` 不存在。
9. 证书归属：`*.allawntech.com` → OPPO广东移动通信有限公司（CN `www.oppo.com`）；`*.heytapmobi.com`、`*.heytap.com`、`*.wanyol.com` → 广东欢太科技有限公司；`security-speed.heytapdownload.com` → 深圳市欢太科技有限公司（CN `nearme.com.cn`）。
10. 三处与设备数据的直接接触面：手机管家的 Odin 推理端点（`speechAntiFraud`、`speechSynthDetect`、`wifiPhotos`，测试请求返回「请求头缺失」）；HTMS 的 Osense SRP 设备上报（`periodic_report_device_data` 等 6 条路径）叠加腾讯图灵盾 `turing_upload_data`；统计 SDK 的上行（最大 358 KB/2h，接收方由 `dynamic_url` 表运行时下发，静态无法确定）。
11. 手机管家的通话记录/短信/录音/相机/定位 appops 当前全部为 `ignore`；HTMS、统计 SDK 的精确定位 appops 为 `allow`（`GRANTED_BY_DEFAULT`），最近一次访问分别约在采集前 42 分钟、3 分钟。
12. `com.oplus.deepthinker` 的 `READ_SMS`、`READ_CALL_LOG` appops 为 `allow`，最近访问约在采集前 4 小时 48 分；该包内引用 `odin-deployment-cn.wanyol.com`。`com.oplus.metis`、`com.oplus.dmp`、`com.heytap.speechassist`、`com.coloros.directui` 的短信/通话/相机/录音 appops 均为 `ignore`。

---

*本报告仅记录观测结果与代码/配置证据，不包含处置建议。*
