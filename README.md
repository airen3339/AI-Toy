# AI Toy · 县域儿童智能陪伴玩具

一个从硬件到云端的完整 AI 玩具项目：ESP32-S3 固件 + Spring Boot 云平台 + 家长端接口。

**不是 SDK 封装，不是贴牌方案。** 板子自己画、固件自己写、云端自己部署。

[![Java](https://img.shields.io/badge/Java-8-blue)](https://openjdk.org/)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-2.7.18-green)](https://spring.io/projects/spring-boot)
[![ESP-IDF](https://img.shields.io/badge/ESP--IDF-5.1.7-orange)](https://docs.espressif.com/projects/esp-idf/)
[![Tests](https://img.shields.io/badge/tests-42%20unit%20%2B%2055%20e2e%20%2B%2020%20firmware-brightgreen)](#测试)
[![Status](https://img.shields.io/badge/status-pre--release-yellow)](#当前状态)

---

## 当前状态

| 层 | 状态 | 说明 |
|---|---|---|
| 云平台 | 🟢 可运行 | 42 单元/集成用例 + 55 端到端用例全绿 |
| 固件 | 🟡 可编译 | 产出 1.16MB 可烧录二进制，**从未上真机** |
| 硬件 | 🔴 未验证 | EVT 板未回，74 条判据 0 条实测 |
| 语音链路 | 🟡 本地已跑通 | 编排服务已实现并端到端验证（T11 3/3）；首响 2.16s，未稳定达 2.0s 预算 |
| 短信登录 | 🟡 本地已跑通 | 出口可配置 + mock-sms 验证 55/55；生产仍需服务商账号/签名/模板 |
| MQTT 接入 | 🟢 本地已打通 | EMQX 5.8.6 + 规则桥接，status/event 真实落库 |

> **一句话**：软件骨架完整、可运行、可回归；三条外部依赖均已在本地用 Docker 打通，
> 产品仍不能直接交付（语音首响性能、生产级供应商账号、硬件实测）。
> 详见 [功能说明文档](03_功能说明文档.md#8-当前完成度必须如实了解)。

---

## 架构

```
┌──────────────────────────────────────────────────────────────┐
│                        家长端 APP                             │
│              Bearer token · 绑定/控制/报告/会员                │
└───────────────────────────┬──────────────────────────────────┘
                            │ HTTPS
┌───────────────────────────▼──────────────────────────────────┐
│                    AI_Toy_Cloud (Spring Boot)                │
│                                                              │
│  auth  user  bind  device  session  parent  report  member   │
│  ota   admin  mqtt  maintenance  security  common            │
│                                                              │
│  MySQL 5.7/8.0 · 18 tables · utf8mb4 · 全表审计              │
└───────┬──────────────────────────────────┬───────────────────┘
        │ X-Device-Token (HTTP)            │ X-Internal-Token
        │                                  │ (EMQX webhook)
┌───────▼──────────────────────────────────▼───────────────────┐
│                   ESP32-S3 (AI_Toy_FW)                       │
│                                                              │
│  bsp  audio  alg  imu  ui  session_fsm  net_link             │
│  ota  power_mgr  diag  fct  app_info                         │
│                                                              │
│  16MB Flash (dual OTA) · 8MB PSRAM · Wi-Fi + BLE             │
└──────────────────────────────────────────────────────────────┘
```

设计决策（不是默认选项，是权衡后的选择）：

- **单体而非微服务**：V1.0 设备量级下，单体 + 包边界比拆微服务少 80% 运维负担。
  包边界已按未来拆分位置画好，需要时按包拆即可。
- **MQTT 用 broker webhook 而非自研客户端**：EMQX 规则引擎把 MQTT 载荷 POST 到
  `/internal/mqtt/*`，云端不必再维护一份 MQTT 连接。
- **设备双凭据体系**：设备 `X-Device-Token`、用户 `Bearer`、后台 `X-Admin-Token`
  三套互不通用。混用即越权，两个拦截器各自只认自己那一类。

---

## 快速开始

### 云端

```bash
cd 02_软件/02_云平台/AI_Toy_Cloud

# 1. 准备 MySQL（5.7 或 8.0）
mysql -h127.0.0.1 -uroot -p -e "
  CREATE DATABASE aitoy_test DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"

# 2. 跑测试（H2 内存库，无需 MySQL）
mvn test                        # 期望 42 用例 0 失败

# 3. 打包并启动
mvn -DskipTests package
java -jar target/ai-toy-cloud.jar \
  --server.port=8091 --spring.profiles.active=test

# 4. 验证
curl http://127.0.0.1:8091/actuator/health
# {"status":"UP","groups":["liveness","readiness"]}
```

### 固件

```bash
# 需要 ESP-IDF v5.1.7 + xtensa-esp32s3-elf-gcc 12.2.0
# Windows 上必须用 ASCII 路径（junction），否则中文路径会被按 GBK 解码
cmake -G Ninja -S E:/fw_ascii -B E:/fw_ascii/build_ascii \
  -DIDF_TARGET=esp32s3 \
  -DSDKCONFIG_DEFAULTS=sdkconfig.defaults \
  -DPYTHON_DEPS_CHECKED=1

cmake --build E:/fw_ascii/build_ascii
# → build_ascii/ai_toy.bin  1,215,056 B  (app 分区余 71%)
```

⚠️ 完整的踩坑记录（Python 3.14 的 `cryptography` 死结、子模块空壳、
`idf-component-manager` 版本、中文路径三重编码）见
[环境搭建文档](04_环境搭建文档.md)。

---

## 目录结构

```
AI机器人 SaaS 平台/
├── 00_计划与管控/         项目计划、任务台账
├── 01_硬件/
│   ├── 01_原理图/          HRS 需求规格书、原理图
│   ├── 02_PCB/            PCB 设计文件
│   ├── 03_外形结构/        毛绒外壳、结构件
│   ├── 04_物料BOM/         BOM 与成本
│   └── 05_数据手册/        芯片手册
├── 02_软件/
│   ├── 01_固件/AI_Toy_FW/  ESP-IDF 工程（4676 行 C）
│   ├── 02_云平台/
│   │   ├── AI_Toy_Cloud/  Spring Boot 服务
│   │   └── CL-001..017/   各模块设计与契约
│   ├── 04_接口契约/
│   ├── 05_部署运维/
│   └── 06_测试/           QA-001..009
└── docs/                  ← 本文档目录
    ├── README.md
    ├── 01_测试报告.md
    ├── 02_修复计划与进度.md
    ├── 03_功能说明文档.md
    ├── 04_环境搭建文档.md
    └── 05_微信公众号宣传文.md
```

---

## 测试

```bash
# 云端单元/集成
mvn test                                    # 42 用例

# 端到端黑盒（需先启动服务）
python _tools/e2e_smoke.py --base http://127.0.0.1:8091 --profile test

# 并发 / 限流 / 幂等
python _tools/verify_load.py

# 固件主机侧算法
bash test/host/run_host_tests.sh            # 20 断言
# 或 Windows：
powershell -File test/host/run_host_tests_msvc.ps1
```

| 测试集 | 用例 | 结果 |
|---|---|---|
| 云端单元/集成 | 42 | 全部通过 |
| 端到端黑盒 | 55 | 全部通过 |
| 固件算法（主机侧） | 20 | 全部通过 |
| 并发 / 限流 / 幂等 | 3 | 3 通过 |
| staging 红线 | 3 | 3 通过 |

覆盖范围包括：认证全链路、设备生命周期、会话记账、会员支付、OTA 元数据、
家长控制、成长报告、跨用户越权（403）、跨 did 越权（401）、双凭据隔离、
限流（429）、幂等（重复 turnId）、emoji 4 字节落库。

---

## 关键设计

### 安全

| 面 | 做法 |
|---|---|
| 设备凭据 | 一机一密换 24h token；校验失败不区分「不存在」与「密钥错」，防枚举 |
| 越权 | 设备 token 内 did 必须与路径一致；用户侧先校验设备归属再放行 |
| 支付 | 金额只在服务端定义，客户端传金额的接口不存在 |
| 内网接口 | `X-Internal-Token` + **未配置即拒绝**（fail-closed，不是放行） |
| 限流 | 单设备 120 事件/分钟；绑定码错误次数窗口限流 |
| 日志 | 验证码明文不进生产日志 |

### 隐私

| 对象 | 保留期 | 动作 |
|---|---|---|
| 对话正文 | 7 天 | 清空字段，行保留 |
| 对话整行 | 30 天 | 删除 |
| 接口审计 | 90 天 | 删除 |
| 手机号 | — | 脱敏展示（`139****1111`） |

成长报告只读 `usage_daily` 汇总表，**不含任何对话原文**。

### 幂等

MQTT QoS1 是"至少一次"，重复投递是常态。`session_turn` 有
`uk_turn (did, turn_id)` 唯一键，重复的 `turn start` 返回成功且**不改变原状态**——
避免把已完成轮次打回 `listening`，进而被超时任务误判。

---

## 已知限制

1. **语音首响未达预算**。编排服务已实现并端到端跑通，但纯 CPU 本机首响均值 2.16s（CL-006 预算 2.0s）；生产建议 GPU 或云厂商 ASR/TTS。
2. **生产环境登录不了**。短信出口已可配置（`AITOY_SMS_GATEWAY_URL`），真实服务商账号/签名/模板尚未申请。
3. **固件从未上板**。只有编译产物，没有真机运行数据。
4. **74 条硬件判据未实测**。EVT 板未回，充电温升、待机电流等只能实测不能算。
5. **未做压测**。528 qps 是 50 请求的瞬时值，不代表容量。
6. **MySQL 5.7 与 8.0 只验证了 5.7**。

这些限制在 [01_测试报告.md](01_测试报告.md) 中逐条记录了证据与影响面。

---

## 文档

| 文档 | 内容 |
|---|---|
| [01_测试报告](01_测试报告.md) | 122 个用例的执行结果、12 项缺陷的证据与位置 |
| [02_修复计划与进度](02_修复计划与进度.md) | 13 项任务，9 项已修并附实测输出，4 项阻塞原因 |
| [03_功能说明文档](03_功能说明文档.md) | 完整功能清单、接口一览、数据模型、业务流程 |
| [04_环境搭建文档](04_环境搭建文档.md) | 从零搭建，含全部踩坑记录 |
| [05_微信公众号宣传文](05_微信公众号宣传文.md) | 对外预热文案 |

---

## 技术栈

| 层 | 选型 | 版本 |
|---|---|---|
| 主控 | ESP32-S3 (N16R8) | 16MB Flash / 8MB PSRAM |
| 固件框架 | ESP-IDF | 5.1.7 |
| 编译器 | xtensa-esp32s3-elf-gcc | 12.2.0 |
| 云端框架 | Spring Boot | 2.7.18 |
| 数据库 | MySQL | 5.7 / 8.0 |
| 构建 | Maven | 3.6+ |
| 字节码目标 | Java | 8 |

---

## 许可

内部项目，未开源。文档与代码版权归项目方所有。

---

## 🏢 关于我们

<p align="center">
  <a href="http://www.net188.net">
    <img src="http://www.net188.net/images/logo1.png" alt="Net188 Logo" width="200" />
  </a>
</p>

<p align="center">
  <strong>Net188 · 互联网技术服务</strong>
</p>

<p align="center">
  专注于跨平台应用开发、AI Agent 集成与大模型应用落地。<br/>
  提供从产品设计、开发实施到部署运维的全栈技术解决方案。
</p>

<p align="center">
  🌐 <a href="http://www.net188.net"><strong>www.net188.net</strong></a>
</p>

---
