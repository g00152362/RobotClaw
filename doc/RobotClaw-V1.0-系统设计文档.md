# RobotClaw V1.0 系统设计文档

> **版本:** 1.0
> **日期:** 2026-06-20
> **状态:** DRAFT
> **基于:** RobotClaw-V1.0-需求列表.md / RobotClaw-V1.0-宣传稿.md / RoboHarness Design Doc

---

## 一、系统总览

### 1.1 设计目标

RobotClaw是Physical AI Harness平台，核心使命：**理解人类标准操作（SOP），让机器人正确执行，并在执行中持续进化**。

**MVP验证场景：**
- **第一阶段（首要）：医院护士送药** — 定制送药机器人（轮式底盘+药箱+推杆），单楼层，涵盖导航、语音交互、力感知、物理开门、视觉识别
- **第二阶段：电力巡检** — 四足/轮式巡检机器人，涵盖导航、热成像、异常检测、告警、报告生成
- 两个场景在P0阶段并行推进，资源冲突时送药优先

系统设计围绕三个核心能力展开：

- **理解层（Understand）**：将自然语言SOP编译为结构化Skill DAG，匹配机器人能力
- **执行层（Execute）**：运行时智能判断和调整，保障每一步正确完成
- **进化层（Evolve）**：从每次执行中学习，越用越正确、越用越快

### 1.2 设计原则

1. **改SOP不改代码** -- 变更传播成本从"数周开发"降为"修改文档"
2. **运行时判断优于预设路径** -- 正确执行的核心是执行中的智能决策，不是死板按计划走
3. **可预测失败优于黑盒成功** -- 不承诺100%成功，但保证100%可解释、100%可处理
4. **数据飞轮驱动进化** -- 每次执行产生的数据回流优化整个系统
5. **云边协同** -- 重计算在云端，实时控制在边缘，兼顾智能与实时性
6. **仿真后置** -- Phase 1-2聚焦真实执行保障，Phase 3引入仿真环境用于Skill验证和回归测试

### 1.3 系统边界

**RobotClaw负责：**
- SOP到Skill DAG的智能编译
- DAG执行调度与状态管理
- 运行时异常检测与恢复
- 机器人能力抽象与状态管理
- 三层记忆系统与数据飞轮
- 执行过程可观测性（Dashboard）
- 行为审计与合规
- Skill生命周期管理

**RobotClaw不负责：**
- 底层运动控制（由机器人SDK负责，通过RML接入）
- 视觉/感知算法实现（由Provider模型负责）
- 硬件驱动和通信协议（由RML Adapter屏蔽）

### 1.4 平台使用角色

RobotClaw作为开放平台，围绕四类核心角色构建生态：

```
┌─────────────────────────────────────────────────────────────────┐
│                     RobotClaw 平台                               │
│                                                                   │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐    │
│  │  代理商    │  │ 本体提供商 │  │ 模型提供商 │  │ Skill开发者│    │
│  │ (Agent)   │  │ (OEM)     │  │ (Model)   │  │ (SkillDev) │    │
│  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘    │
│        │              │              │              │            │
│        ▼              ▼              ▼              ▼            │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
│  │场景模板   │  │e-URDF    │  │Provider  │  │Skill     │        │
│  │SOP编排   │  │能力声明   │  │模型注册   │  │ClawHub   │        │
│  │参数配置   │  │硬件适配   │  │推理服务   │  │交易结算   │        │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘        │
└─────────────────────────────────────────────────────────────────┘
```

#### 1.4.1 代理商（Agent / 集成商）

**定位：** 面向终端客户的方案交付者，利用平台能力开发场景模板，满足客户业务需求。

| 维度 | 说明 |
|------|------|
| **核心工作** | 理解客户业务SOP → 选择/组合Skill → 配置场景模板 → 部署交付 |
| **使用的平台能力** | SOP Compiler、场景模板系统、Dashboard、Skill Registry（选型）、记忆管理 |
| **产出** | 场景模板（含SOP、Skill组合、参数配置、失败处理方案） |
| **典型用户** | 机器人集成商、行业解决方案商、系统集成SI |
| **价值主张** | 从"80人月写代码"变为"配置模板+少量定制"，首个项目10人月以内 |

**代理商工作流：**

```
客户需求（业务SOP）
    │
    ├── 1. 从模板库选择最接近的场景模板
    ├── 2. 配置模板参数（点位、阈值、告警规则等）
    ├── 3. 从Skill Registry选择/购买所需Skill
    ├── 4. 补充定制Skill（如有）
    ├── 5. 联调验证（Dashboard监控+参数微调）
    └── 6. 交付客户，持续运维
```

#### 1.4.2 机器人本体提供商（OEM）

**定位：** 提供机器人硬件及其e-URDF能力描述，使机器人能够被平台识别和调度。

| 维度 | 说明 |
|------|------|
| **核心工作** | 定义e-URDF（能力声明+物理约束） → 实现RML Adapter → 验证Skill兼容性 |
| **使用的平台能力** | e-URDF定义工具、Forge机器人接入、Skill兼容性验证、能力抽象层 |
| **产出** | e-URDF文件、RML Adapter实现、Skill兼容性报告 |
| **典型用户** | 宇树科技、优必选、大疆、UR（优傲）、自研机器人团队 |
| **价值主张** | 一次接入，即可复用平台全部Skill和场景模板（Teach Once, Embody Anywhere） |

**本体接入流程：**

```
机器人硬件
    │
    ├── 1. 编写e-URDF（声明能力、物理约束、传感器参数）
    ├── 2. 实现RML Adapter（对接ROS2/DDS/自定义协议）
    ├── 3. 能力校验（平台自动校验e-URDF与Skill需求的匹配度）
    ├── 4. Skill兼容性测试（在目标Skill上运行验证）
    └── 5. 发布至平台，供代理商选用
```

#### 1.4.3 模型提供商（Model Provider）

**定位：** 提供AI模型能力（视觉、语音、决策等），通过Provider接口注册到平台，供Skill调用。

| 维度 | 说明 |
|------|------|
| **核心工作** | 封装模型为Provider接口 → 注册能力声明 → 提供推理服务（远端API或边缘模型包） |
| **使用的平台能力** | Provider Registry、Provider抽象接口、边缘模型分发管线、模型版本管理 |
| **产出** | Provider实现（远端API或边缘模型包）、能力声明、性能基线 |
| **典型用户** | AI模型公司、视觉算法团队、大模型厂商（通义千问、智谱等）、垂直领域算法供应商 |
| **价值主张** | 模型能力标准化接入，一次封装即可被多个场景多个Skill复用 |

**模型提供方式：**

| 部署位置 | 提供形式 | 适用场景 |
|---------|---------|---------|
| **远端** | API服务（HTTP/gRPC） | 大模型推理（SOP编译、复杂决策、VLM分析） |
| **边缘** | ONNX模型包 + 平台自动编译为目标芯片格式 | 实时推理（目标检测、ASR、TTS、异常识别） |
| **混合** | 远端API + 边缘蒸馏版 | 高可用场景（边缘优先，远端兜底） |

#### 1.4.4 Skill开发者（Skill Developer）

**定位：** 开发可复用的Skill，通过ClawHub市场上架交易，获取收益。

| 维度 | 说明 |
|------|------|
| **核心工作** | 开发Skill（接口定义+执行逻辑） → 声明能力需求 → 测试验证 → 上架ClawHub |
| **使用的平台能力** | Skill接口标准、ClawHub技能市场、Darwin评测体系、Forge测试环境 |
| **产出** | Skill实现（含接口定义、前后置条件、失败模式、恢复策略） |
| **典型用户** | 机器人算法工程师、ROS2开发者、垂直行业技术团队、独立开发者 |
| **价值主张** | 开发一次，多场景多本体复用；通过ClawHub交易获取收益 |

**Skill上架与交易流程：**

```
Skill开发
    │
    ├── 1. 按Skill接口标准开发（输入/输出/前后置条件/失败模式/恢复策略）
    ├── 2. 声明所需能力（required_capabilities）和适用本体
    ├── 3. 本地测试 + Darwin评测（六道关卡）
    ├── 4. 上架ClawHub（版本管理、能力标签、适用场景标注）
    ├── 5. 代理商/集成商购买使用
    └── 6. 按调用量/订阅结算收益
```

**Skill交易模式：**

| 模式 | 说明 | 适用场景 |
|------|------|---------|
| **按次付费** | 每次Skill调用计费 | 低频使用的专业Skill |
| **订阅制** | 按月/年订阅，不限调用次数 | 高频使用的基础Skill |
| **买断制** | 一次性购买，永久使用 | 核心Skill、大客户定制 |
| **开源免费** | 社区贡献，免费使用 | 基础通用Skill，扩大生态 |

#### 1.4.5 角色协作关系

```
                    终端客户（医院、电厂、矿山...）
                              │
                              │ 业务需求
                              ▼
                    ┌──────────────────┐
                    │     代理商        │ ← 选择场景模板 + 配置参数
                    │  (方案交付)       │ ← 选购Skill组合
                    └────────┬─────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
    ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
    │ 本体提供商    │ │ 模型提供商    │ │ Skill开发者   │
    │              │ │              │ │              │
    │ 提供机器人    │ │ 提供AI模型    │ │ 提供Skill     │
    │ + e-URDF     │ │ + Provider   │ │ + ClawHub    │
    └──────┬───────┘ └──────┬───────┘ └──────┬───────┘
           │                │                │
           └────────────────┼────────────────┘
                            ▼
                  ┌──────────────────┐
                  │  RobotClaw 平台   │
                  │                   │
                  │ SOP Compiler      │
                  │ 执行引擎          │
                  │ 记忆系统          │
                  │ 数据飞轮          │
                  └──────────────────┘
```

**平台在角色间的价值：**

| 角色对 | 平台提供的连接价值 |
|--------|------------------|
| 代理商 ↔ 本体提供商 | e-URDF标准化机器人能力，代理商无需关心硬件差异 |
| 代理商 ↔ Skill开发者 | ClawHub市场提供Skill发现、评测、交易的标准化流程 |
| 代理商 ↔ 模型提供商 | Provider接口屏蔽模型差异，代理商只关心能力而非模型实现 |
| 本体提供商 ↔ Skill开发者 | 能力抽象层实现"Skill写一次，多本体运行"，双方独立演进 |
| 模型提供商 ↔ Skill开发者 | Provider接口解耦模型与Skill，模型升级不影响Skill逻辑 |

---

## 二、整体架构

### 2.1 架构总图

```
                    ┌─────────────────────────────────────────────────┐
                    │              远端（Cloud/Edge Server）           │
                    │                                                  │
                    │  ┌──────────────────────────────────────────┐    │
                    │  │            用户交互层                     │    │
                    │  │  [SOP Editor] [Dashboard] [Skill Browser]│    │
                    │  │  [审计日志查看] [记忆管理] [模板配置]       │    │
                    │  └──────────────┬───────────────────────────┘    │
                    │                 │                                │
                    │  ┌──────────────▼───────────────────────────┐    │
                    │  │          SOP Compiler（编译层）           │    │
                    │  │  步骤提取 → 意图映射 → DAG构建 →           │    │
                    │  │  静态校验 → 人工审核                       │    │
                    │  │  [Task Memory缓存] [LLM Provider调用]     │    │
                    │  └──────────────┬───────────────────────────┘    │
                    │                 │                                │
                    │  ┌──────────────▼───────────────────────────┐    │
                    │  │          任务调度层（Task Scheduler）     │    │
                    │  │  DAG实例化 → 参数注入 → 记忆合并 →         │    │
                    │  │  下发到机器人端                           │    │
                    │  └──────────────┬───────────────────────────┘    │
                    │                 │                                │
                    │  ┌──────────────▼───────────────────────────┐    │
                    │  │          数据服务层                       │    │
                    │  │  [Memory Store] [Practice Store]         │    │
                    │  │  [Audit Log] [Skill Registry]            │    │
                    │  │  [Provider Registry] [模板库]            │    │
                    │  └──────────────────────────────────────────┘    │
                    └─────────────────┬───────────────────────────────┘
                                      │
                         ─ ─ ─ ─ ─ ─ ─│─ ─ ─ ─ ─ ─ ─  网络边界
                                      │  (gRPC)
                                      │
                    ┌─────────────────▼───────────────────────────────┐
                    │            机器人端（Robot Runtime）              │
                    │                                                   │
                    │  ┌──────────────────────────────────────────┐    │
                    │  │        执行引擎（Execution Engine）        │    │
                    │  │  [DAG遍历器] [状态机] [异常检测与恢复]     │    │
                    │  │  [运行时环境感知] [动态执行调整]           │    │
                    │  │  [级联熔断] [断点恢复]                     │    │
                    │  └──────────────┬───────────────────────────┘    │
                    │                 │                                  │
                    │  ┌──────────────▼───────────────────────────┐    │
                    │  │        Skill执行层                         │    │
                    │  │  [Skill Runner] [前置/后置条件校验]        │    │
                    │  │  [超时控制] [重试策略] [执行数据采集]      │    │
                    │  └──────────────┬───────────────────────────┘    │
                    │                 │                                  │
                    │  ┌──────────────▼───────────────────────────┐    │
                    │  │    边缘推理层（Edge Inference Runtime）    │    │
                    │  │  [边缘VLM] [目标检测] [异常识别]           │    │
                    │  │  [语音ASR/TTS] [小型决策模型]              │    │
                    │  │  [模型生命周期管理] [推理调度器]           │    │
                    │  └──────────────┬───────────────────────────┘    │
                    │                 │                                  │
                    │  ┌──────────────▼───────────────────────────┐    │
                    │  │    能力抽象层（Capability Abstraction）    │    │
                    │  │  [视觉] [听觉] [力觉] [行走] [抓取]       │    │
                    │  │  [语音] [定位] [环境感知] [显示] [通信]    │    │
                    │  │  [能力状态管理器] [能力降级/故障检测]      │    │
                    │  └──────────────┬───────────────────────────┘    │
                    │                 │                                  │
                    │  ┌──────────────▼───────────────────────────┐    │
                    │  │  机器人消息层（Robot Messaging Layer）     │    │
                    │  │  pub/sub · service · param · action       │    │
                    │  │  [Phase 1: ROS2 Adapter (rclcpp)]         │    │
                    │  │  [Phase 2+: DDS/自定义协议 Adapter]       │    │
                    │  └──────────────┬───────────────────────────┘    │
                    │                 │                                  │
                    │  ┌──────────────▼───────────────────────────┐    │
                    │  │        硬件抽象层（HAL / Drivers）         │    │
                    │  │  [运动控制] [传感器] [SLAM/导航] [相机]    │    │
                    │  │  机器人SDK / 硬件驱动 (C++)                │    │
                    │  └──────────────────────────────────────────┘    │
                    └─────────────────────────────────────────────────┘
```

### 2.2 远端架构（Cloud/Edge Server）

远端承担所有**重计算、重存储、需要人工交互**的工作：

| 模块 | 职责 | 技术选型 |
|------|------|----------|
| **SOP Compiler** | 自然语言SOP编译为Skill DAG | Python + LLM API |
| **Task Scheduler** | DAG实例化、参数注入、任务下发 | Python + gRPC |
| **Memory Store** | 三层记忆存储与查询 | SQLite(Phase 1) → PostgreSQL+PostGIS(Phase 3+) |
| **Practice Store** | 执行实践记录存储 | 时序数据库(TimescaleDB) |
| **Skill Registry** | Skill注册、版本管理、能力声明 | Python + SQLite/PostgreSQL |
| **Provider Registry** | LLM/VLM模型注册与路由 | Python |
| **Audit Service** | 行为审计日志 | Append-only存储 + 哈希链 |
| **Dashboard** | Web可视化界面 | React + WebSocket |
| **模板库** | 场景应用模板管理 | 文件系统 + 数据库索引 |

**部署方式：** Docker容器化，支持完全离线运行。可部署在：
- 工厂内边缘服务器（满足数据不出厂区要求）
- 云端服务器（适合多站点管理）
- 机器人随行计算单元（小型部署场景）

### 2.3 机器人端架构（Robot Runtime）

机器人端承担所有**实时性要求高、需要硬件交互**的工作：

| 模块 | 职责 | 实时性要求 |
|------|------|-----------|
| **Execution Engine** | DAG遍历、状态管理、异常恢复 | 软实时（100ms级决策） |
| **Skill Runner** | Skill实际执行、条件校验 | 与Skill本身一致 |
| **Edge Inference Runtime** | 边缘模型加载、推理调度、模型热切换 | 软实时（50-200ms级推理） |
| **Capability Manager** | 能力状态监控、能力查询、降级检测 | 软实时（200ms级状态更新） |
| **Robot Messaging Layer** | 通信原语抽象（pub/sub/service/param/action），屏蔽底层中间件差异 | 与底层中间件一致 |
| **数据采集Agent** | 执行数据实时采集、上报 | 非实时（异步上报） |
| **本地缓存** | 关键记忆数据的本地副本 | 启动时同步 |
| **ROS2 Bridge** | 通过RML与底层中间件通信（Phase 1: ROS2 Adapter） | 硬实时（DDS层） |

**运行环境：** 机器人端运行C++原生Runtime，通过Robot Messaging Layer（RML）与底层通信中间件交互：
- 开发语言：C++17（确定性延迟，无GIL/GC开销）
- 操作系统：Ubuntu 22.04 LTS（JetPack 6.x / 国产芯片BSP适配）
- 通信抽象：Robot Messaging Layer（RML），Phase 1通过ROS2 Adapter（rclcpp）对接ROS2，Phase 2+可切换DDS/自定义协议
- 构建系统：CMake + colcon
- 计算平台：详见"物理运行环境"章节

### 2.4 物理运行环境

机器人端的计算平台按项目阶段分两步走：**Phase 1在NVIDIA Jetson AGX上验证全链路，Phase 2迁移到国产算力芯片实现自主可控。**

#### 2.4.1 Phase 1 — NVIDIA Jetson AGX Orin（Month 1-6）

Phase 1-3选择Jetson AGX Orin作为首选开发和验证平台，原因：

| 维度 | Jetson AGX Orin优势 |
|------|-------------------|
| **算力** | 275 TOPS (INT8)，可同时运行多个边缘模型 + 执行引擎 |
| **GPU** | Ampere架构 2048 CUDA cores，TensorRT深度优化 |
| **内存** | 32/64GB LPDDR5，统一内存架构（CPU/GPU共享，零拷贝） |
| **ROS2生态** | NVIDIA Isaac ROS加速库原生支持，社区成熟 |
| **开发效率** | JetPack SDK + CUDA工具链完善，调试工具丰富 |
| **行业惯例** | 四足机器人（宇树Go2/G1）、巡检机器人主流搭载平台 |

**Phase 1硬件配置基线：**

```
NVIDIA Jetson AGX Orin 64GB Developer Kit
├── SoC: Orin (12核 ARM Cortex-A78AE + 2048 CUDA cores)
├── 内存: 64GB LPDDR5 (统一内存，CPU/GPU共享)
├── 存储: 64GB eMMC + NVMe SSD扩展（模型+数据存储）
├── AI算力: 275 TOPS (INT8) / 138 TOPS (FP16)
├── 操作系统: JetPack 6.x (Ubuntu 22.04 + CUDA 12.x + TensorRT 10.x)
├── 接口: USB3.2, GbE, PCIe, CAN, GPIO, CSI相机接口
├── 功耗: 15W-60W可调（巡检场景典型30W）
└── 价格: ~$1999 (开发套件)
```

**软件栈映射：**

```
RobotClaw Runtime on Jetson AGX Orin
├── 执行引擎 + Skill Runner        → ARM CPU (rclcpp)
├── 边缘推理运行时
│   ├── 目标检测 (YOLO/RT-DETR)    → GPU + TensorRT (FP16, <20ms)
│   ├── 异常识别 (分类/分割)        → GPU + TensorRT (FP16, <30ms)
│   ├── 语音ASR (Whisper small)    → GPU + TensorRT (<100ms)
│   ├── 语音TTS (VITS)            → GPU + TensorRT (<50ms)
│   ├── 轻量VLM (MobileVLM 3B)    → GPU + TensorRT (INT4, <500ms)
│   └── VLA动作生成                → GPU + TensorRT (<30ms)
├── ROS2 rclcpp节点               → ARM CPU
├── SLAM/导航                      → ARM CPU + GPU加速
└── 传感器驱动                     → ARM CPU + ISP (相机)
```

#### 2.4.2 Phase 2 — 国产算力芯片迁移（Month 5+）

从Phase 2开始启动国产算力芯片适配，目标是**在不改变上层C++应用代码的前提下，通过HAL（硬件抽象层）切换底层推理后端**，实现自主可控部署。

**候选国产芯片平台：**

| 芯片平台 | 厂商 | AI算力 | 推理框架 | 适用场景 | 迁移复杂度 |
|---------|------|--------|---------|---------|-----------|
| **昇腾310B** | 华为 | 8 TOPS (INT8) | CANN (AscendCL) | 轻量巡检（检测+ASR） | 中 |
| **昇腾310P** | 华为 | 24 TOPS (INT8) | CANN (AscendCL) | 中等巡检（+异常识别+TTS） | 中 |
| **昇腾910B** | 华为 | 320 TOPS (INT8) | CANN (AscendCL) | 全功能（含VLM） | 中高 |
| **RK3588** | 瑞芯微 | 6 TOPS (INT8) | RKNN | 轻量巡检 | 低 |
| **BM1684X** | 算能 | 32 TOPS (INT8) | SAIL/BMRuntime | 中等巡检 | 中 |
| **寒武纪MLU270** | 寒武纪 | 128 TOPS (INT8) | CNToolkit | 全功能 | 中高 |
| **地平线J5** | 地平线 | 128 TOPS (INT8) | OpenExplorer | 自动驾驶/巡检 | 中 |

**推荐迁移路径：**

```
Phase 1 (Month 1-4):
  Jetson AGX Orin — 全链路开发验证，建立性能基线

Phase 2 (Month 5-6):
  昇腾310P/BM1684X — 首个国产芯片适配
  ├── 目标：核心模型（检测+ASR+异常识别）在国产芯片上跑通
  ├── 方法：ONNX → 国产框架转换（CANN/SAIL），精度对齐
  └── 验证：与Jetson基线对比推理延迟和精度

Phase 3 (Month 7+):
  昇腾910B/寒武纪MLU — 高算力国产平台
  ├── 目标：全部边缘模型（含VLM）在国产芯片上运行
  └── 验证：端到端巡检任务性能达到Jetson 90%+
```

#### 2.4.3 硬件抽象层（HAL）设计

为实现"上层代码不变，底层芯片可换"，Edge Inference Runtime通过HAL屏蔽硬件差异：

```
Edge Inference Runtime (C++)
│
├── EdgeProvider接口（统一C++ API）
│   invoke(request) → result
│
├── 硬件抽象层（HAL）
│   ├── TensorRT Backend    ← Jetson AGX Orin (Phase 1)
│   ├── CANN Backend        ← 华为昇腾 (Phase 2)
│   ├── RKNN Backend        ← 瑞芯微RK3588 (Phase 2)
│   ├── SAIL Backend        ← 算能BM1684X (Phase 2)
│   ├── OpenVINO Backend    ← Intel (兼容)
│   └── ONNX Runtime Backend ← 纯CPU回退 (通用)
│
└── 统一模型格式流程
    ONNX (交换格式)
      ├── → TensorRT .engine   (Jetson编译)
      ├── → CANN .om           (昇腾编译)
      ├── → RKNN .rknn         (瑞芯微编译)
      ├── → SAIL .bmodel       (算能编译)
      └── → ONNX Runtime       (CPU直接加载)
```

**HAL接口定义（C++）：**

```cpp
// 硬件推理后端抽象接口
class InferenceBackend {
public:
    virtual ~InferenceBackend() = default;

    // 加载模型（平台特定格式）
    virtual bool load_model(const std::string& model_path,
                            const ModelConfig& config) = 0;

    // 同步推理
    virtual InferenceResult infer(const InferenceInput& input) = 0;

    // 查询硬件状态（温度、利用率、内存）
    virtual HardwareStatus get_status() const = 0;

    // 获取后端信息（芯片型号、算力、支持的精度）
    virtual BackendInfo get_info() const = 0;
};

// Phase 1 实现
class TensorRTBackend : public InferenceBackend { /* Jetson AGX Orin */ };

// Phase 2 实现
class CANNBackend : public InferenceBackend { /* 华为昇腾 */ };
class RKNNBackend : public InferenceBackend { /* 瑞芯微 */ };
class SAILBackend : public InferenceBackend { /* 算能 */ };
```

#### 2.4.4 模型跨平台编译管线

远端负责将ONNX模型编译为各平台的优化格式，随模型版本分发：

```yaml
edge_model_manifest:
  models:
    - name: "yolo-inspection-v3"
      version: "3.1.0"
      source_format: "onnx"
      platform_builds:
        # Phase 1 — Jetson AGX Orin
        - platform: "jetson_orin"
          format: "tensorrt_engine"
          file: "yolo_v3.1.0_orin_fp16.engine"
          precision: "fp16"
          target_latency_ms: 18
          validated: true

        # Phase 2 — 华为昇腾310P
        - platform: "ascend_310p"
          format: "cann_om"
          file: "yolo_v3.1.0_ascend310p_int8.om"
          precision: "int8"
          target_latency_ms: 25
          validated: true

        # Phase 2 — 算能BM1684X
        - platform: "bm1684x"
          format: "sail_bmodel"
          file: "yolo_v3.1.0_bm1684x_fp16.bmodel"
          precision: "fp16"
          target_latency_ms: 22
          validated: false           # 待验证

        # 通用回退
        - platform: "cpu"
          format: "onnx"
          file: "yolo_v3.1.0.onnx"
          precision: "fp32"
          target_latency_ms: 200
          validated: true
```

#### 2.4.5 远端服务器运行环境

远端服务器对算力要求不高（不做推理），主要需求是稳定性和存储：

```
远端服务器（Cloud/Edge Server）
├── 部署方式: Docker Compose
├── 最低配置:
│   ├── CPU: x86_64 4核
│   ├── 内存: 8GB RAM
│   ├── 存储: 100GB SSD（数据库+Practice记录+模型仓库）
│   └── 网络: 有线以太网（与机器人端通信）
├── 推荐配置:
│   ├── CPU: x86_64 8核
│   ├── 内存: 16GB RAM
│   ├── 存储: 500GB SSD
│   ├── GPU: 可选（本地私有化大模型部署时需要，如A10/A100）
│   └── 网络: 千兆有线 + 可选4G/5G备份
└── 操作系统: Ubuntu 22.04 LTS / CentOS 8+ / 银河麒麟V10 (国产化)
```

### 2.5 云边协同设计

#### 2.5.1 职责划分原则

```
                    远端（重但不急）              机器人端（轻但快）
                    ──────────────              ──────────────
编译类任务：         SOP → DAG编译               -
                    Skill匹配与校验              -
                    记忆查询与参数推荐            -

模型推理任务：       大模型推理（SOP编译、        边缘模型推理（目标检测、
                     复杂决策、知识检索）          异常识别、语音ASR/TTS、
                    模型版本管理与分发              实时场景理解）
                    离线训练与微调                边缘模型加载与热切换
                                                推理结果本地缓存

执行类任务：         任务监控与可视化              DAG遍历与状态机
                    人工接管指令下发              Skill实际执行
                    执行决策记录                  运行时环境感知
                                                异常检测与恢复
                                                传感器数据获取

数据类任务：         Practice持久化存储            执行数据实时采集
                    记忆更新与分析                本地缓存同步
                    离线分析与优化                数据压缩与上报
                    审计日志归档                  本地审计日志缓存
```

#### 2.5.2 通信协议

**gRPC统一通道架构**

所有云边通信统一使用gRPC，通过不同的RPC方法和流模式覆盖控制和数据两类需求：

```
机器人端 ◄──── gRPC（统一通道）────► 远端
   │        HTTP/2多路复用                │
   │                                      │
   ├── Unary RPC:    任务下发/紧急停止
   ├── Server Stream: 模型版本推送/记忆同步
   ├── Client Stream: 状态上报/Practice数据上传
   └── Bidi Stream:   心跳/人工接管
```

**控制类RPC：**
- 任务下发：远端 → 机器人端，下发DAG实例和执行参数（Unary）
- 状态上报：机器人端 → 远端，上报DAG/Skill执行状态变化（Client Streaming）
- 人工接管：双向，支持紧急停止和手动控制指令（Bidirectional Streaming）
- 心跳：双向，30秒间隔，3次丢失触发离线处理（Bidirectional Streaming）

**数据类RPC：**
- 执行数据上报：机器人端 → 远端，Practice数据批量上传（Client Streaming）
- 传感器快照：机器人端 → 远端，失败时上传诊断数据（Unary，大包分片）
- 记忆同步：远端 → 机器人端，下发更新的推荐参数（Server Streaming）
- 边缘模型分发通知：远端 → 机器人端，新模型版本推送（Server Streaming）

**离线缓冲：** gRPC断连时，机器人端将待上报数据写入本地SQLite队列，网络恢复后按时间序批量重传。通过Client Streaming的流控机制实现背压和重传确认，替代MQTT的QoS保证。

**gRPC接口定义示例：**

```protobuf
service TaskControl {
  // 任务下发（远端 → 机器人端）
  rpc DispatchDAG(DAGRequest) returns (DAGResponse);

  // 状态上报（机器人端 → 远端，client streaming）
  rpc ReportStatus(stream StatusUpdate) returns (Ack);

  // 紧急停止（双向，低延迟）
  rpc EmergencyStop(StopRequest) returns (StopResponse);

  // 心跳（双向 streaming）
  rpc Heartbeat(stream Ping) returns (stream Pong);

  // 人工接管（双向 streaming，持续指令流）
  rpc HumanTakeover(stream ControlCommand) returns (stream RobotFeedback);
}

// 数据通道服务（替代MQTT，统一使用gRPC streaming）
service DataChannel {
  // Practice数据批量上传（机器人端 → 远端）
  rpc UploadPractice(stream PracticeChunk) returns (UploadAck);

  // 传感器快照上传（机器人端 → 远端，大包分片）
  rpc UploadSnapshot(stream SnapshotChunk) returns (UploadAck);

  // 记忆同步（远端 → 机器人端，server streaming推送更新）
  rpc SyncMemory(SyncRequest) returns (stream MemoryUpdate);

  // 模型版本推送（远端 → 机器人端）
  rpc NotifyModelUpdate(ModelQuery) returns (stream ModelVersionInfo);
}
```

#### 2.5.3 通信协议选型论证：gRPC统一 vs gRPC+MQTT双通道

最终选择gRPC统一方案，淘汰了gRPC+MQTT双通道方案：

**候选方案：**

```
方案A（采用）: gRPC统一通道（控制+数据全部走gRPC）
方案B（否决）: gRPC（控制通道）+ MQTT（数据通道）
方案C（否决）: WebSocket + Protobuf（控制通道）+ MQTT（数据通道）
```

**逐维度对比：**

| 维度 | gRPC | WebSocket + Protobuf | 对本项目的影响 |
|------|------|---------------------|--------------|
| **代码生成** | .proto一键生成C++和Python双端RPC代码 | 只生成消息类型，service路由需手写 | **决定性差异**：机器人端C++ + 远端Python，接口频繁迭代时gRPC节省大量跨语言同步工作 |
| **类型安全** | 编译期完整类型检查 | 运行时才能发现类型错误 | 工业场景不容许运行时类型错误 |
| **离线重连** | 内置channel级自动重连、指数退避、状态监控 | 需手写重连逻辑和状态恢复 | 工厂/矿山网络不稳定，自动重连是刚需 |
| **双向流** | 原生bidirectional streaming | 原生全双工 | 两者都满足 |
| **延迟** | ~1-2ms（HTTP/2帧开销） | ~0.5-1ms（裸TCP） | 两者都远低于100ms要求，不构成差异 |
| **浏览器支持** | 不支持（需grpc-web代理） | 原生支持 | Dashboard已有独立WebSocket推送，不影响 |
| **调试** | 二进制，需grpcurl/grpcui | 可在浏览器DevTools观察 | 可通过grpcui解决，非阻塞问题 |
| **多机器人扩展** | HTTP/2多路复用，天然支持多连接 | 需手动管理多WebSocket连接 | Phase 4 Swarm多机协作时gRPC更优 |
| **C++生态** | grpc++（Google官方维护） | Boost.Beast / websocketpp | 两者都成熟 |

**选择gRPC的核心理由：**

1. **跨语言代码生成是决定性优势。** 机器人端C++ + 远端Python的架构下，每次修改接口（新增RPC、修改消息字段），gRPC只需改.proto文件并重新生成。WebSocket方案则需要在C++端手写消息路由分发器，在Python端手写对应处理器，两端同步维护。在快速迭代阶段，这个工程负担是持续的：

```cpp
// WebSocket方案：每加一个RPC都要手写路由（C++端）
void on_message(const std::string& raw) {
    auto envelope = parse_envelope(raw);
    switch (envelope.type) {
        case MSG_DISPATCH_DAG: handle_dispatch(envelope.payload); break;
        case MSG_REPORT_STATUS: handle_status(envelope.payload); break;
        case MSG_EMERGENCY_STOP: handle_stop(envelope.payload); break;
        // ... 每次改接口都要在C++和Python两端同步修改
    }
}

// gRPC方案：protoc自动生成，上层只需实现业务逻辑
class TaskControlServiceImpl final : public TaskControl::Service {
    Status DispatchDAG(ServerContext* ctx, const DAGRequest* req,
                       DAGResponse* resp) override {
        // 只写业务逻辑，路由/序列化/反序列化全部自动生成
    }
};
```

2. **内置重连机制。** 工业场景（电厂/矿山/管廊）网络不稳定是常态。gRPC的channel自动重连和退避策略是开箱即用的，WebSocket需要自己实现断线检测、指数退避、状态恢复逻辑。

3. **多机器人扩展就绪。** Phase 4 Swarm多机协作时，gRPC的HTTP/2多路复用可以高效管理多个机器人连接。WebSocket方案需要额外的连接池管理。

**WebSocket仍然在Dashboard中使用：** Dashboard前端（React）通过WebSocket接收实时推送（DAG进度、机器人状态、告警）。这与机器人端的gRPC控制通道互不干扰——Dashboard是浏览器端，只能用WebSocket；机器人端是C++原生，gRPC更合适。两者共存，各取所长。

**不采用MQTT数据通道的理由：** MQTT虽有离线消息队列和QoS保证，但引入了额外的基础设施依赖（EMQX Broker），且MQTT不适合传输大数据包（如传感器快照、Practice批量数据）。gRPC的Client Streaming + 本地SQLite离线队列可以完全覆盖MQTT的离线缓冲能力，同时保持协议栈统一：一套.proto定义、一个grpc++客户端库，降低机器人端的运维和调试复杂度。

#### 2.5.4 离线容灾

当网络断开时，机器人端可以：
1. 继续执行已下发的DAG（本地有完整DAG副本）
2. 使用本地缓存的记忆数据做参数决策
3. 执行数据暂存本地，网络恢复后批量上报
4. 无法接收人工接管指令（但本地熔断机制仍有效）
5. 无法下发新任务（需等待网络恢复）

---

## 三、机器人能力抽象层（Robot Capability Abstraction）

RobotClaw的核心理念是"理解人类标准操作，让机器人正确执行"。但不同机器人的硬件差异巨大：四足机器人能行走不能抓取，机械臂能操作不能移动，人形机器人什么都能做但每项能力的参数完全不同。

**能力抽象层（Capability Abstraction Layer, CAL）** 解决这个问题：将机器人的物理硬件能力抽象为统一的**原子能力（Atomic Capability）**，让上层（SOP Compiler、Skill、执行引擎）只和抽象能力打交道，不关心底层硬件差异。

### 3.1 原子能力模型

每个机器人的能力被分解为以下原子能力维度：

```
Robot Capability Model
├── 感知类（Perception）
│   ├── 视觉（Vision）        -- 看：RGB相机、深度相机、热成像
│   ├── 听觉（Hearing）       -- 听：麦克风阵列、声音检测
│   ├── 力觉（Force Sensing） -- 力感知：力矩传感器、触觉传感器、碰撞检测
│   ├── 位置感知（Localization）-- 定位：IMU、GPS、SLAM、UWB
│   └── 环境感知（Environment）-- 环境：气体传感器、温湿度、激光雷达
│
├── 运动类（Locomotion）
│   ├── 行走（Walking）       -- 腿式移动：四足步态、双足步态
│   ├── 轮式移动（Wheeling）  -- 轮式移动：差速驱动、全向移动
│   ├── 飞行（Flying）        -- 飞行：旋翼、固定翼
│   └── 混合移动（Hybrid）    -- 轮腿混合等
│
├── 操作类（Manipulation）
│   ├── 抓取（Grasping）      -- 夹爪、吸盘、灵巧手
│   ├── 推拉（Pushing）       -- 力控推拉操作
│   ├── 工具使用（Tool Use）  -- 持握并使用工具
│   └── 精密操作（Fine Motor）-- 高精度操作
│
└── 交互类（Communication）
    ├── 语音输出（Speaking）   -- 说：扬声器、语音合成
    ├── 视觉输出（Display）   -- 显示：屏幕、指示灯、投影
    ├── 网络通信（Network）   -- 数据：WiFi、4G/5G、以太网
    └── 人机交互（HRI）       -- 手势识别、表情、体态语言
```

### 3.2 能力声明标准

每个原子能力在e-URDF中声明，包含完整的物理约束：

```yaml
# 能力声明 Schema
capability:
  name: "vision"                     # 原子能力名称
  type: "perception.vision"          # 能力类型路径

  # 硬件绑定
  hardware:
    sensors:
      - name: "front_rgb_camera"
        type: "rgb_camera"
        resolution: [1920, 1080]
        fps: 30
        fov_h: 69.4                  # 水平视场角（度）
        fov_v: 42.5
        mount_frame: "head_link"
      - name: "thermal_camera"
        type: "thermal_camera"
        resolution: [640, 480]
        fps: 15
        temperature_range: [-20, 350] # 摄氏度
        mount_frame: "head_link"

  # 能力参数边界
  constraints:
    max_detection_range: 10.0        # 米
    min_detection_range: 0.3
    lighting_conditions: ["indoor", "outdoor_day", "outdoor_night_with_ir"]
    ip_rating: "IP54"

  # 能力状态查询接口
  status_topic: "/sensors/camera/status"    # ROS2 topic
  data_topic: "/sensors/camera/image_raw"   # ROS2 topic

  # 依赖的其他能力（可选）
  depends_on: []
```

```yaml
capability:
  name: "walking"
  type: "locomotion.walking"

  hardware:
    actuators:
      - group: "front_left_leg"
        joints: ["hip", "thigh", "calf"]
        max_torque: [40, 40, 40]     # Nm
      - group: "front_right_leg"
        joints: ["hip", "thigh", "calf"]
        max_torque: [40, 40, 40]
      # ... 后腿省略

  constraints:
    max_speed: 1.2                   # m/s
    max_slope: 25                    # 度
    max_step_height: 0.15            # 米
    min_turn_radius: 0.0             # 原地转向
    terrain_types: ["flat", "grass", "gravel", "stairs"]
    battery_impact: 0.8              # 满功率行走时电池消耗率(Ah/km)

  gaits:                             # 可用步态
    - name: "trot"
      speed_range: [0.3, 1.2]
      stability: "high"
    - name: "crawl"
      speed_range: [0.1, 0.3]
      stability: "very_high"
      use_case: "rough_terrain"

  status_topic: "/locomotion/status"
  command_topic: "/cmd_vel"

capability:
  name: "force_sensing"
  type: "perception.force_sensing"

  hardware:
    sensors:
      - name: "foot_force_fl"
        type: "force_torque_6dof"
        location: "front_left_foot"
        range_force: [0, 200]        # N
        range_torque: [0, 10]        # Nm
        frequency: 500               # Hz
      # ... 其他足端力传感器

  constraints:
    sensitivity: 0.1                 # N
    collision_detection_threshold: 50 # N
    overload_protection: 300         # N

  status_topic: "/sensors/force/status"
  data_topic: "/sensors/force/readings"

capability:
  name: "speaking"
  type: "communication.speaking"

  hardware:
    actuators:
      - name: "speaker"
        type: "audio_output"
        channels: 1
        sample_rate: 44100
        max_volume_db: 85

  constraints:
    tts_languages: ["zh-CN", "en-US"]
    max_ambient_noise_db: 70         # 在此噪音水平下仍可清晰传达
    latency_ms: 200                  # 语音合成延迟

  status_topic: "/audio/speaker/status"
  command_topic: "/audio/speaker/play"

capability:
  name: "hearing"
  type: "perception.hearing"

  hardware:
    sensors:
      - name: "mic_array"
        type: "microphone_array"
        channels: 4
        sample_rate: 16000
        beam_forming: true

  constraints:
    speech_recognition_languages: ["zh-CN", "en-US"]
    keyword_detection: true          # 支持唤醒词
    noise_cancellation: true
    effective_range: 5.0             # 米

  status_topic: "/audio/mic/status"
  data_topic: "/audio/mic/raw"
```

```yaml
capability:
  name: "pushing"
  type: "manipulation.pushing"

  hardware:
    actuators:
      - name: "door_pusher"
        type: "linear_actuator"
        stroke_mm: 300
        max_force_n: 50
        speed_mm_s: 100
        mount_frame: "front_link"

  constraints:
    max_safe_force_n: 50           # 安全力上限
    force_control_hz: 100          # 力控频率
    collision_detection_n: 30      # 碰撞检测阈值
    position_accuracy_mm: 5        # 定位精度

  status_topic: "/actuators/pusher/status"
  command_topic: "/actuators/pusher/command"
  force_topic: "/actuators/pusher/force"
```

### 3.3 能力抽象层在系统中的位置

```
    SOP Compiler                Skill定义                 执行引擎
    "前往3号设备拍照"           capture_thermal_image     Skill Runner
         │                           │                       │
         │ 需要哪些能力？             │ 声明所需能力           │ 检查能力就绪
         ▼                           ▼                       ▼
    ┌─────────────────────────────────────────────────────────────┐
    │              能力抽象层（Capability Abstraction Layer）       │
    │                                                              │
    │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐      │
    │  │  Vision   │ │ Walking  │ │ Grasping │ │ Hearing  │ ...  │
    │  │  (看)     │ │  (走)    │ │  (抓)    │ │  (听)    │      │
    │  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘      │
    │       │            │            │             │              │
    │  ┌────▼────────────▼────────────▼─────────────▼─────────┐  │
    │  │        能力状态管理器（Capability State Manager）       │  │
    │  │  - 能力就绪状态监控                                    │  │
    │  │  - 能力降级/故障检测                                   │  │
    │  │  - 能力组合可行性判断                                  │  │
    │  └──────────────────┬────────────────────────────────────┘  │
    └─────────────────────┼──────────────────────────────────────┘
                          │
    ┌─────────────────────▼──────────────────────────────────────┐
    │                  e-URDF + RML (Robot Messaging Layer)           │
    │  物理硬件：相机、麦克风、电机、力传感器、扬声器...          │
    └────────────────────────────────────────────────────────────┘
```

### 3.4 能力与Skill的关系

Skill是面向任务的高级操作，Capability是面向硬件的原子能力。一个Skill通常需要多个Capability组合：

| Skill | 所需Capability组合 | 适用场景 |
|-------|-------------------|---------|
| navigate_to_waypoint | walking/wheeling + localization | 送药/巡检 |
| speak_text | speaking | 送药/巡检 |
| wait_for_weight_change | force_sensing | 送药 |
| check_weight | force_sensing | 送药 |
| detect_target | vision(rgb) + localization | 送药（门/床位） |
| open_door | manipulation.pushing + vision + force_sensing | 送药 |
| wait_for_condition | vision / force_sensing（按条件类型） | 送药 |
| capture_thermal_image | vision(thermal) + localization | 巡检 |
| detect_anomaly | vision(rgb) + environment | 巡检 |
| alert_operator | speaking + network | 送药/巡检 |
| return_to_base | walking/wheeling + localization | 巡检 |
| log_result | network | 送药/巡检 |
| verbal_interaction | hearing + speaking | 送药（扩展） |
| force_guided_insert | grasping + force_sensing + vision | 通用 |
| patrol_and_report | walking + vision + hearing + speaking + network | 巡检 |

Skill接口声明中的`required_sensors`字段升级为**required_capabilities**：

```yaml
skill:
  name: "patrol_and_report"
  version: "1.0.0"

  # 能力需求声明（替代原有的 required_sensors）
  required_capabilities:
    - type: "locomotion.walking"
      min_speed: 0.5                # 最低速度要求
    - type: "perception.vision"
      sensor_type: "rgb_camera"     # 需要RGB相机
    - type: "perception.hearing"
      keyword_detection: true       # 需要关键词检测
      optional: true                # 可选能力：没有也能执行，但功能降级
    - type: "communication.speaking"
      optional: true                # 可选：无扬声器则改为文字告警
    - type: "communication.network"
      min_bandwidth_kbps: 500       # 最低带宽要求

  # 能力降级策略：当可选能力不可用时的替代方案
  capability_fallbacks:
    - when_missing: "perception.hearing"
      fallback: "skip_audio_monitoring"
    - when_missing: "communication.speaking"
      fallback: "use_text_alert_via_network"
```

### 3.5 能力状态管理

能力抽象层在运行时持续监控每个能力的状态：

```cpp
// 能力状态枚举
enum class CapabilityState {
    READY,        // 就绪可用
    BUSY,         // 正在被占用
    DEGRADED,     // 降级（部分功能可用）
    FAULT,        // 故障
    UNAVAILABLE   // 不可用（硬件缺失）
};

// 能力状态管理器 -- 运行在机器人端（C++ rclcpp节点）
class CapabilityManager : public rclcpp::Node {
public:
    // 查询某项能力的当前状态和参数
    CapabilityInfo query_capability(const std::string& cap_type);

    // 检查一组能力需求是否可满足（Skill执行前调用）
    // 返回: all_met / partial(列出缺失) / not_met
    CheckResult check_requirements(const std::vector<CapabilityReq>& requirements);

    // 订阅能力状态变化（用于执行引擎的运行时监控）
    void subscribe_state_change(const std::string& cap_type,
                                std::function<void(CapabilityState)> callback);

    // 返回机器人完整能力画像（供远端SOP Compiler查询）
    RobotCapabilityProfile get_robot_profile();
};
```

**能力状态变化触发的执行引擎响应：**

| 状态变化 | 执行引擎响应 |
|---------|-------------|
| READY → FAULT | 暂停当前依赖该能力的Skill，触发恢复策略 |
| READY → DEGRADED | 通知Skill降级执行（如相机分辨率下降时降低检测精度要求） |
| FAULT → READY | 恢复被暂停的Skill，从断点继续 |
| BUSY → READY | 调度队列中等待该能力的下一个Skill |

### 3.6 能力画像与SOP编译

SOP Compiler在编译阶段，基于机器人的能力画像做智能匹配：

```
SOP步骤: "到3号设备前，用红外相机拍摄设备温度分布图，如发现温度异常则语音播报告警"

能力分析:
  ├── "到3号设备前"  → 需要 locomotion.* (任一移动能力)
  ├── "用红外相机拍摄" → 需要 perception.vision (thermal_camera)
  ├── "发现温度异常"   → 需要 perception.vision (thermal分析能力)
  └── "语音播报告警"   → 需要 communication.speaking
                          或降级为 communication.network (文字告警)

目标机器人: 宇树Go2
能力画像:
  ✓ locomotion.walking (四足, max 1.2m/s)
  ✓ perception.vision (RGB + Thermal)
  ✗ communication.speaking (无扬声器)
  ✓ communication.network (WiFi)

编译结果:
  步骤1: navigate_to_waypoint(target="device_3") -- 使用walking能力
  步骤2: capture_thermal_image(device="device_3") -- 使用vision.thermal能力
  步骤3: detect_anomaly(type="temperature") -- 使用vision分析
  步骤4: alert_operator(method="network") -- 降级：无speaking，使用network文字告警
```

**送药场景能力画像与编译示例：**

```
SOP步骤: "到药房取药，送到301病房张三，开门进入，确认取药后返回护士站"

能力分析:
  ├── "到药房"          → 需要 locomotion.* (任一移动能力)
  ├── "取药/感知药品"    → 需要 perception.force_sensing (药箱力传感器)
  ├── "开门进入"         → 需要 manipulation.pushing + perception.vision
  ├── "识别床位"         → 需要 perception.vision (RGB相机)
  ├── "语音播报"         → 需要 communication.speaking
  └── "返回护士站"       → 需要 locomotion.* + perception.localization

目标机器人: 定制送药机器人
能力画像:
  ✓ locomotion.wheeling (轮式差速, max 0.8m/s)
  ✓ perception.vision (RGB)
  ✓ perception.force_sensing (药箱力传感器)
  ✓ manipulation.pushing (推杆, max 50N)
  ✓ communication.speaking (扬声器)
  ✓ perception.localization (激光雷达+IMU)
  ✗ perception.hearing (无麦克风)

编译结果:
  步骤1: navigate_to_waypoint(target="pharmacy") -- 使用wheeling能力
  步骤2: speak_text(text="请取301病房张三的药品") -- 使用speaking能力
  步骤3: wait_for_weight_change(direction="increase") -- 使用force_sensing能力
  步骤4: navigate_to_waypoint(target="room_301_door") -- 使用wheeling能力
  步骤5: open_door(target="room_301_door", type="push") -- 使用pushing+vision+force_sensing
  步骤6: detect_target(target="bed_3") -- 使用vision能力
  步骤7-N: ... (后续类推)
```

### 3.7 e-URDF中的能力声明

能力声明作为e-URDF扩展的核心部分：

```xml
<eurdf:extensions>
  <!-- 能力声明 -->
  <eurdf:capabilities>
    <eurdf:capability type="perception.vision">
      <eurdf:sensor name="front_rgb_camera" type="rgb_camera"
                    resolution="1920x1080" fps="30" fov_h="69.4"
                    mount_frame="head_link"/>
      <eurdf:sensor name="thermal_camera" type="thermal_camera"
                    resolution="640x480" fps="15"
                    temperature_range="-20,350"
                    mount_frame="head_link"/>
    </eurdf:capability>

    <eurdf:capability type="locomotion.walking">
      <eurdf:actuator_group name="quadruped_legs" joints="12"
                            max_speed="1.2" max_slope="25"
                            terrain="flat,grass,gravel,stairs"/>
      <eurdf:gait name="trot" speed_range="0.3,1.2" stability="high"/>
      <eurdf:gait name="crawl" speed_range="0.1,0.3" stability="very_high"/>
    </eurdf:capability>

    <eurdf:capability type="perception.force_sensing">
      <eurdf:sensor name="foot_force_fl" type="force_torque_6dof"
                    location="front_left_foot"
                    range_force="0,200" frequency="500"/>
      <!-- ... 其他足端 -->
    </eurdf:capability>

    <eurdf:capability type="perception.hearing">
      <eurdf:sensor name="mic_array" type="microphone_array"
                    channels="4" sample_rate="16000"
                    beam_forming="true"/>
    </eurdf:capability>

    <!-- 未具备的能力显式声明为不可用 -->
    <eurdf:capability type="manipulation.grasping" available="false"
                      reason="no_manipulator"/>
    <eurdf:capability type="communication.speaking" available="false"
                      reason="no_speaker"/>
  </eurdf:capabilities>
</eurdf:extensions>
```

**送药机器人e-URDF能力声明示例（MVP首要验证机器人）：**

```xml
<eurdf:extensions>
  <eurdf:robot_type>medication_delivery</eurdf:robot_type>
  <eurdf:chassis>wheeled</eurdf:chassis>

  <eurdf:capabilities>
    <eurdf:capability type="locomotion.wheeling">
      <eurdf:actuator_group name="diff_drive" joints="2"
                            max_speed="0.8" min_turn_radius="0.0"/>
      <eurdf:constraint terrain="flat,indoor" max_slope="5"/>
    </eurdf:capability>

    <eurdf:capability type="perception.vision">
      <eurdf:sensor name="front_rgb_camera" type="rgb_camera"
                    resolution="1920x1080" fps="30" fov_h="69.4"
                    mount_frame="body_link"/>
    </eurdf:capability>

    <eurdf:capability type="perception.force_sensing">
      <eurdf:sensor name="medicine_box_scale" type="load_cell"
                    location="medicine_box" range_force="0,5000"
                    resolution_g="1" frequency="100"/>
    </eurdf:capability>

    <eurdf:capability type="manipulation.pushing">
      <eurdf:actuator name="door_pusher" type="linear_actuator"
                      max_force_n="50" stroke_mm="300"
                      mount_frame="front_link"/>
      <eurdf:constraint max_safe_force_n="50"
                        force_control_hz="100"/>
    </eurdf:capability>

    <eurdf:capability type="communication.speaking">
      <eurdf:actuator name="speaker" type="audio_output"
                      channels="1" max_volume_db="85"/>
    </eurdf:capability>

    <eurdf:capability type="perception.localization">
      <eurdf:sensor name="lidar" type="2d_lidar"
                    range="12.0" frequency="10" mount_frame="body_link"/>
      <eurdf:sensor name="imu" type="imu" frequency="200"/>
    </eurdf:capability>

    <eurdf:capability type="communication.network">
      <eurdf:interface name="wifi" type="wifi" band="5GHz"/>
    </eurdf:capability>

    <!-- 未具备的能力 -->
    <eurdf:capability type="perception.hearing" available="false"
                      reason="no_microphone"/>
    <eurdf:capability type="locomotion.walking" available="false"
                      reason="wheeled_chassis"/>
  </eurdf:capabilities>
</eurdf:extensions>
```

### 3.8 Robot Messaging Layer（机器人消息层）

RobotClaw的机器人端不直接依赖ROS2，而是通过**Robot Messaging Layer（RML）** 抽象通信原语，使上层代码（执行引擎、Skill Runner、CAL）与底层通信中间件解耦。

#### 3.8.1 设计动机

| 问题 | RML解决方案 |
|------|------------|
| ROS2版本碎片化（Humble/Iron/Jazzy） | 上层代码只依赖RML接口，ROS2版本差异封装在Adapter内部 |
| 非ROS2机器人接入困难 | 实现对应的RML Adapter即可接入，无需改动上层逻辑 |
| 部分客户要求纯DDS方案（无ROS2依赖） | Phase 2提供DDS Direct Adapter，编译时不链接ROS2 |
| ROS2的重量级依赖（100+包） | 轻量化场景可选用精简Adapter |

#### 3.8.2 通信原语

RML定义四种通信原语，覆盖机器人通信的所有模式：

```cpp
// Robot Messaging Layer 抽象接口
namespace rml {

// 1. 发布/订阅（Pub/Sub）— 传感器数据、状态广播
template<typename MsgT>
class Publisher {
public:
    virtual ~Publisher() = default;
    virtual void publish(const MsgT& msg) = 0;
};

template<typename MsgT>
class Subscription {
public:
    using Callback = std::function<void(const MsgT&)>;
    virtual ~Subscription() = default;
};

// 2. 服务（Service）— 请求/响应，如查询状态、触发单次动作
template<typename ReqT, typename RespT>
class ServiceClient {
public:
    virtual ~ServiceClient() = default;
    virtual std::future<RespT> call(const ReqT& request) = 0;
};

template<typename ReqT, typename RespT>
class ServiceServer {
public:
    using Handler = std::function<RespT(const ReqT&)>;
    virtual ~ServiceServer() = default;
};

// 3. 参数（Param）— 运行时配置，如速度限制、检测阈值
class ParamClient {
public:
    virtual ~ParamClient() = default;
    virtual std::string get(const std::string& key) = 0;
    virtual void set(const std::string& key, const std::string& value) = 0;
};

// 4. 动作（Action）— 长时间运行的任务，如导航、机械臂运动
template<typename GoalT, typename FeedbackT, typename ResultT>
class ActionClient {
public:
    using FeedbackCallback = std::function<void(const FeedbackT&)>;
    virtual ~ActionClient() = default;
    virtual std::future<ResultT> send_goal(const GoalT& goal,
                                            FeedbackCallback fb_cb = nullptr) = 0;
    virtual void cancel_goal() = 0;
};

// RML工厂 — 创建通信实体
class MessagingFactory {
public:
    virtual ~MessagingFactory() = default;

    template<typename MsgT>
    virtual std::unique_ptr<Publisher<MsgT>>
        create_publisher(const std::string& topic) = 0;

    template<typename MsgT>
    virtual std::unique_ptr<Subscription<MsgT>>
        create_subscription(const std::string& topic,
                            typename Subscription<MsgT>::Callback cb) = 0;

    template<typename ReqT, typename RespT>
    virtual std::unique_ptr<ServiceClient<ReqT, RespT>>
        create_service_client(const std::string& service_name) = 0;

    template<typename GoalT, typename FeedbackT, typename ResultT>
    virtual std::unique_ptr<ActionClient<GoalT, FeedbackT, ResultT>>
        create_action_client(const std::string& action_name) = 0;

    virtual std::unique_ptr<ParamClient>
        create_param_client(const std::string& node_name) = 0;
};

} // namespace rml
```

#### 3.8.3 Adapter实现策略

```
┌─────────────────────────────────────────────┐
│          上层代码（执行引擎/Skill/CAL）       │
│          只依赖 rml:: 接口                   │
└──────────────────┬──────────────────────────┘
                   │
         ┌─────────▼─────────┐
         │   RML Interface   │
         └─────────┬─────────┘
                   │
    ┌──────────────┼──────────────┐
    │              │              │
┌───▼───┐    ┌────▼────┐   ┌────▼────┐
│ ROS2  │    │  DDS    │   │ Custom  │
│Adapter│    │ Direct  │   │Protocol │
│(rclcpp│    │(CycloneDDS│  │ Adapter │
│ Phase1)│   │ Phase2+) │  │(Phase3+)│
└───────┘    └─────────┘   └─────────┘
```

| Adapter | 阶段 | 底层依赖 | 适用场景 |
|---------|------|----------|---------|
| **ROS2 Adapter** | Phase 1（默认） | rclcpp + ROS2 Humble/Iron/Jazzy | 主流ROS2机器人（宇树、UR等） |
| **DDS Direct Adapter** | Phase 2+ | CycloneDDS / FastDDS（无ROS2） | 客户要求无ROS2依赖的纯DDS部署 |
| **Custom Protocol Adapter** | Phase 3+ | 私有协议SDK | 非标机器人、工业PLC等 |

**Phase 1实现原则：** ROS2 Adapter是唯一实现，所有topic/service/action名称与ROS2生态保持一致。上层代码通过RML接口调用，编译时链接ROS2 Adapter。后续新增Adapter时，上层代码零改动，只需切换编译配置。

---

## 四、关键技术分析

### 4.1 DAG vs 行为树

#### 4.1.1 技术对比

| 维度 | DAG（有向无环图） | 行为树（Behavior Tree） |
|------|------------------|------------------------|
| **表达能力** | 顺序、并行、条件分支、循环 | 顺序、选择、并行、装饰器 |
| **数据流** | 节点间显式数据引用（n1.output.x） | 通过Blackboard共享状态 |
| **静态分析** | 容易做类型检查、依赖分析、环路检测 | 较难做全局静态分析 |
| **可视化** | 直观展示数据流和执行顺序 | 直观展示决策逻辑 |
| **LLM生成** | LLM容易生成结构化DAG（JSON/YAML） | LLM生成行为树更复杂 |
| **动态修改** | 节点级替换，不影响其他节点 | 子树替换可能影响执行语义 |
| **行业惯例** | 工作流引擎（Airflow/Temporal） | 游戏AI、机器人决策 |
| **失败恢复** | 节点级重试/跳过/回退，语义清晰 | 通过Fallback节点，嵌套复杂 |

#### 4.1.2 设计决策：采用DAG

**选择DAG作为核心执行模型，理由：**

1. **SOP到DAG的映射更自然。** 人类的标准操作流程天然是"先做A，再做B，如果条件C则做D"的结构，这正是DAG的表达形式。行为树更适合"持续运行、实时响应"的场景（如游戏NPC），而非"按步骤执行一次性任务"。

2. **LLM编译输出更可靠。** LLM生成结构化JSON/YAML（DAG节点列表+依赖关系）比生成行为树的嵌套结构更准确、更容易做静态校验。这直接影响SOP Compiler的编译准确率目标（>=80%）。

3. **数据流追踪是刚需。** 巡检任务中，上一步的输出是下一步的输入（如"拍照 → 分析图片 → 决定是否告警"）。DAG的显式数据引用（`n2.output.temperature_map`）比行为树的Blackboard更清晰、更可审计。

4. **静态校验能力。** DAG可以在执行前做完整的类型检查、依赖完整性检查、前置条件可满足性检查。这对"可预测失败"的设计目标至关重要。

5. **与记忆系统的集成更直接。** Task Memory缓存的是DAG编译结果，Execution Memory记录的是DAG节点级别的执行数据。DAG的结构化特性使记忆的读写和匹配更精确。

**行为树的适用场景保留为未来扩展：** 在Skill内部实现中，单个Skill可以内部使用行为树来实现复杂的感知-决策-执行循环（如导航Skill内部的避障决策）。但Skill之间的编排统一使用DAG。

#### 4.1.3 DAG节点类型

```yaml
# Phase 1 支持
sequential:     # 顺序执行，前一节点完成后执行下一节点
conditional:    # 条件分支，基于前一节点输出决定走哪个分支

# Phase 2 增加
parallel:       # 并行fork/join，多个节点同时执行，全部完成后继续
loop:           # 循环，对列表迭代执行同一Skill序列
```

**条件分支的汇合（Merge）语义：**

条件节点（conditional）执行时只走一个分支。后续节点通过 `depends_on` 引用条件节点ID时，语义为"等待条件节点所选分支的最后一个节点完成后继续"。具体规则：

```
规则1: depends_on引用conditional节点ID = 等待被选中分支执行完毕
       （未被选中的分支不会执行，不会阻塞后续节点）

规则2: 条件节点的每个分支内部是独立的子DAG，
       分支内节点通过depends_on相互引用

规则3: 分支外节点不可直接depends_on分支内节点
       （因为该分支可能未被选中，导致永久阻塞）

示例:
  n12 (conditional: 药品是否取完?)
    ├── true分支:  n12a (祝福)
    └── false分支: n12b→n12c→n12d (提醒→再等→祝福)

  n13 (depends_on: ["n12"])
    含义: 等待n12的被选中分支完成后执行
    - 若走true分支: n12a完成后执行n13
    - 若走false分支: n12d完成后执行n13
```

#### 4.1.4 DAG Schema定义

```yaml
dag:
  id: "inspection-substation-001"
  version: "1.0"
  compiled_from: "变电站日常巡检SOP v2.3"
  compile_hash: "sha256:abc123..."        # SOP文本+Skill Registry版本+机器人能力画像的hash

  metadata:
    scene_template: "电力巡检"
    robot_type: "unitree-go2"
    estimated_duration_s: 1800

  nodes:
    - id: "n1"
      skill: "navigate_to_waypoint"
      params:
        target: {x: 10.5, y: 3.2, z: 0.0}
        speed: 0.8
      depends_on: []
      timeout_s: 60
      retry_policy: {max: 3, backoff: "exponential"}

    - id: "n2"
      skill: "capture_thermal_image"
      params:
        target_location: {ref: "n1.output.final_position"}
        capture_params: {resolution: "640x480", mode: "continuous"}
      depends_on: ["n1"]
      timeout_s: 30

    - id: "n3"
      type: "conditional"
      condition: "n2.output.temperature_map.max_temp > 80"
      branches:
        true:
          - id: "n3a"
            skill: "alert_operator"
            params:
              level: "warning"
              message: "高温异常: {n2.output.temperature_map.max_temp}度"
              evidence: {ref: "n2.output.thermal_image"}
        false:
          - id: "n3b"
            skill: "log_result"
            params:
              status: "normal"
              data: {ref: "n2.output.temperature_map"}
      depends_on: ["n2"]
```

### 4.2 LLM驱动的SOP编译

#### 4.2.1 编译管线（5阶段）

```
SOP自然语言文本
    │
    ▼
[0. 缓存查询] ─── 命中 ──→ 直接返回已编译DAG（0ms）
    │ 未命中
    ▼
[1. 步骤提取] LLM将SOP拆分为结构化步骤列表
    │           每步包含: 动作、目标、条件、参数
    │           输出: JSON结构化步骤数组
    ▼
[2. 意图映射] 将每步动作映射到Skill Registry中的Skill
    │           方法: LLM推理 + Skill能力声明embedding匹配
    │           输出: 每步绑定一个Skill + 置信度分数
    ▼
[3. DAG构建]  基于步骤间的数据依赖和控制依赖构建DAG
    │           Phase 1: 顺序 + 条件分支
    │           Phase 2: 增加并行和循环
    ▼
[4. 静态校验] 自动检查:
    │           - 所有引用的Skill在Registry中存在
    │           - 输入/输出类型匹配
    │           - 前置条件可满足
    │           - DAG合法性（无环路）
    │           - 数据引用有效性
    ▼
[5. 人工审核] UI展示SOP原文 ↔ DAG映射
    │           高亮低置信度映射（<0.8）
    │           审核者确认或手动修正
    ▼
最终Skill DAG → 写入Task Memory缓存
```

#### 4.2.2 LLM Prompt策略

**结构化输出约束：**
```json
{
  "steps": [
    {
      "step_id": 1,
      "action": "导航到3号变压器",
      "target": "transformer-T3",
      "condition": null,
      "params": {"speed": "normal"},
      "depends_on": []
    },
    {
      "step_id": 2,
      "action": "拍摄红外热成像",
      "target": "transformer-T3",
      "condition": "到达3号变压器后",
      "params": {"mode": "连续拍摄"},
      "depends_on": [1]
    }
  ]
}
```

**Prompt工程要点：**

1. **Few-shot示例：** 提供3-5个标注好的SOP → 步骤列表示例，覆盖顺序、条件分支、异常处理等模式
2. **Chain-of-thought：** 对含条件分支的SOP，要求LLM先列出判断条件再输出分支结构
3. **幻觉防护：**
   - Stage 2匹配时，使用embedding相似度 + 阈值过滤（相似度 < 0.7 标记为低置信度）
   - Stage 4静态校验捕获引用不存在的Skill名称
   - 对LLM输出做JSON Schema严格校验
4. **编译准确率目标：** 标准工业SOP >= 80%自动正确映射（步骤级）。低于60%回退到"LLM辅助 + 人工确认每步"模式

#### 4.2.3 编译缓存机制

```
缓存Key = SHA256(SOP文本 + Skill Registry版本号 + 机器人能力画像hash)

说明：
- SOP文本：自然语言操作规程原文
- Skill Registry版本号：可用Skill集合的版本
- 机器人能力画像hash：目标机器人的Capability Profile摘要
  同一SOP在不同能力的机器人上编译结果不同（如有/无扬声器会导致不同的Skill映射）

查询流程:
1. 计算当前SOP的缓存Key（含机器人能力画像）
2. 查询Task Memory
3. 命中: 检查success_rate >= 0.9 → 直接返回
4. 命中但success_rate < 0.9 → 标记stale，走重新编译
5. 未命中: 走完整编译流程

失效条件:
- Skill Registry版本变更 → 关联缓存标记stale
- DAG执行成功率低于阈值 → 触发重编译
- 人工标记失效（Dashboard操作）
- TTL过期（30天未使用）
- 机器人能力画像变更（硬件升级/降级/传感器更换）
- 场景模板版本变更（模板Skill组合或参数模板更新）
```

### 4.3 MVP场景一：医院护士送药流程（首要验证场景）

以下以医院护士送药场景为MVP首要验证目标，完整展示SOP从输入到DAG执行的全链路分解。该场景使用定制送药机器人（轮式底盘+药箱+推杆/机械臂），单楼层运行，涵盖导航、语音交互、力感知、视觉识别、物理开门等核心能力。

#### 4.3.1 原始SOP输入

```
医院护理机器人送药标准操作规程（SOP）

1. 到药房窗口，向药剂师语音播报："请取301病房张三的药品"
2. 等待药剂师将药品放入机器人药箱，感知药箱重量变化确认药品已放入
3. 向药剂师语音确认："药品已接收，正在配送至301病房"
4. 导航至301病房门口
5. 到达后语音播报："301病房送药服务，请开门"
6. 推开病房门进入（门锁定时呼叫护士站）
7. 进入病房，识别患者张三的床位（3号床）
8. 导航至3号床旁
9. 语音播报："张三您好，这是您本次的药品，请核对"
10. 等待患者取药，感知药箱重量变化确认药品已取出
11. 如果药品未全部取出，语音提醒："还有药品未取，请检查"
12. 药品全部取出后，语音播报："祝您早日康复，如需帮助请呼叫护士"
13. 导航返回护士站
14. 到达护士站，语音上报："301病房张三送药完成"
```

#### 4.3.2 Step 1 - 步骤提取（LLM输出）

```json
{
  "sop_id": "hospital_medication_delivery_001",
  "total_steps": 14,
  "steps": [
    {
      "step_id": 1,
      "action": "导航到药房窗口并语音播报取药请求",
      "target": "pharmacy_window",
      "params": {"speech": "请取301病房张三的药品"},
      "depends_on": []
    },
    {
      "step_id": 2,
      "action": "等待药品放入，通过力传感器确认",
      "target": "medicine_box",
      "condition": "药箱重量增加 > 阈值",
      "params": {"timeout_s": 120, "weight_threshold_g": 10},
      "depends_on": [1]
    },
    {
      "step_id": 3,
      "action": "语音确认药品已接收",
      "target": "pharmacist",
      "params": {"speech": "药品已接收，正在配送至301病房"},
      "depends_on": [2]
    },
    {
      "step_id": 4,
      "action": "导航至目标病房",
      "target": "room_301_door",
      "params": {"speed": "normal"},
      "depends_on": [3]
    },
    {
      "step_id": 5,
      "action": "语音播报到达通知",
      "target": "room_301",
      "params": {"speech": "301病房送药服务，请开门"},
      "depends_on": [4]
    },
    {
      "step_id": 6,
      "action": "推开病房门进入",
      "target": "room_301_door",
      "condition": "门未锁定",
      "params": {"door_type": "push"},
      "depends_on": [5],
      "on_failure": "呼叫护士站"
    },
    {
      "step_id": 7,
      "action": "进入病房并识别目标床位",
      "target": "bed_3",
      "params": {"patient_name": "张三", "bed_number": 3},
      "depends_on": [6]
    },
    {
      "step_id": 8,
      "action": "导航至目标床位旁",
      "target": "bed_3_side",
      "depends_on": [7]
    },
    {
      "step_id": 9,
      "action": "语音播报送药信息",
      "target": "patient",
      "params": {"speech": "张三您好，这是您本次的药品，请核对"},
      "depends_on": [8]
    },
    {
      "step_id": 10,
      "action": "等待患者取药，通过力传感器确认",
      "target": "medicine_box",
      "condition": "药箱重量减少",
      "params": {"timeout_s": 120},
      "depends_on": [9]
    },
    {
      "step_id": 11,
      "action": "检查药品是否全部取出",
      "target": "medicine_box",
      "condition": "药箱重量 ≈ 空箱重量",
      "params": {},
      "depends_on": [10],
      "on_false": "语音提醒：还有药品未取"
    },
    {
      "step_id": 12,
      "action": "语音播报送药完成",
      "target": "patient",
      "params": {"speech": "祝您早日康复，如需帮助请呼叫护士"},
      "depends_on": [11]
    },
    {
      "step_id": 13,
      "action": "导航返回护士站",
      "target": "nurse_station",
      "depends_on": [12]
    },
    {
      "step_id": 14,
      "action": "语音上报送药完成",
      "target": "nurse_station",
      "params": {"speech": "301病房张三送药完成"},
      "depends_on": [13]
    }
  ]
}
```

#### 4.3.3 Step 2 - 意图映射（Skill匹配）

| 步骤 | SOP动作 | 映射Skill | 置信度 | 所需能力 |
|------|---------|-----------|--------|---------|
| 1 | 导航到药房+语音播报 | navigate_to_waypoint + speak_text | 0.95 | walking + speaking |
| 2 | 等待药品放入（力感知） | wait_for_weight_change | 0.92 | force_sensing |
| 3 | 语音确认 | speak_text | 0.98 | speaking |
| 4 | 导航至301病房 | navigate_to_waypoint | 0.97 | walking + localization |
| 5 | 语音播报到达 | speak_text | 0.98 | speaking |
| 6 | 推开病房门进入 | open_door | 0.93 | manipulation.pushing + vision + force_sensing |
| 7 | 识别床位 | detect_target | 0.90 | vision |
| 8 | 导航至床旁 | navigate_to_waypoint | 0.97 | walking + localization |
| 9 | 语音播报 | speak_text | 0.98 | speaking |
| 10 | 等待取药（力感知） | wait_for_weight_change | 0.92 | force_sensing |
| 11 | 检查药品取完 | check_weight | 0.90 | force_sensing |
| 12 | 语音播报 | speak_text | 0.98 | speaking |
| 13 | 导航返回 | navigate_to_waypoint | 0.97 | walking + localization |
| 14 | 语音上报 | speak_text + log_result | 0.95 | speaking + network |

#### 4.3.4 Step 3 - DAG构建

```yaml
dag:
  id: "med_delivery_301_zhangsan"
  name: "301病房张三送药"
  created_by: "sop_compiler"

  nodes:
    # Phase 1: 药房取药
    - id: "n1_nav_pharmacy"
      skill: "navigate_to_waypoint"
      params:
        target: {name: "pharmacy_window", map_id: "hospital_1f"}
        speed: 0.6
      depends_on: []

    - id: "n2_request_medicine"
      skill: "speak_text"
      params:
        text: "请取301病房张三的药品"
        volume: "normal"
        wait_for_finish: true
      depends_on: ["n1_nav_pharmacy"]

    - id: "n3_wait_load"
      skill: "wait_for_weight_change"
      params:
        sensor: "medicine_box_scale"
        direction: "increase"
        threshold_g: 10
        timeout_s: 120
      failure_modes:
        - code: "TIMEOUT"
          recovery: "escalate_to_human({reason: '药剂师未放药，请检查'})"
      depends_on: ["n2_request_medicine"]

    - id: "n4_confirm_received"
      skill: "speak_text"
      params:
        text: "药品已接收，正在配送至301病房"
      depends_on: ["n3_wait_load"]

    # Phase 2: 导航至病房
    - id: "n5_nav_room"
      skill: "navigate_to_waypoint"
      params:
        target: {name: "room_301_door", map_id: "hospital_3f"}
        speed: 0.6
      failure_modes:
        - code: "PATH_BLOCKED"
          recovery: "invoke_skill('navigate_to_waypoint', {avoid_area: blocked_area})"
        - code: "ELEVATOR_UNAVAILABLE"
          recovery: "retry_after(60, max=5)"
      depends_on: ["n4_confirm_received"]

    - id: "n6_announce_arrival"
      skill: "speak_text"
      params:
        text: "301病房送药服务，请开门"
        volume: "loud"
        repeat: 2
        repeat_interval_s: 10
      depends_on: ["n5_nav_room"]

    - id: "n7_open_door"
      skill: "open_door"
      params:
        target_door: "room_301_door"
        door_type: "push"
      failure_modes:
        - code: "HANDLE_NOT_FOUND"
          recovery: "retry_after(2, max=3)"
        - code: "DOOR_LOCKED"
          recovery: "invoke_skill('alert_operator', {target: 'nurse_station', reason: '301病房门锁定，请开门'})"
      depends_on: ["n6_announce_arrival"]

    # Phase 3: 病房内送药
    - id: "n8_detect_bed"
      skill: "detect_target"
      params:
        target_type: "hospital_bed"
        target_label: "3号床"
        detection_model: "hospital_furniture_v2"
      depends_on: ["n7_open_door"]

    - id: "n9_nav_bedside"
      skill: "navigate_to_waypoint"
      params:
        target: {ref: "n8_detect_bed.output.target_pose"}
        speed: 0.3
        approach_distance: 0.5
      depends_on: ["n8_detect_bed"]

    - id: "n10_greet_patient"
      skill: "speak_text"
      params:
        text: "张三您好，这是您本次的药品，请核对"
      depends_on: ["n9_nav_bedside"]

    - id: "n11_wait_pickup"
      skill: "wait_for_weight_change"
      params:
        sensor: "medicine_box_scale"
        direction: "decrease"
        threshold_g: 5
        timeout_s: 120
      failure_modes:
        - code: "TIMEOUT"
          recovery: "invoke_skill('speak_text', {text: '请取出您的药品'})"
      depends_on: ["n10_greet_patient"]

    - id: "n12_check_empty"
      skill: "check_weight"
      type: "conditional"
      params:
        sensor: "medicine_box_scale"
        condition: "weight <= empty_threshold_g + 5"
      branches:
        true:
          - id: "n12a_farewell"
            skill: "speak_text"
            params:
              text: "祝您早日康复，如需帮助请呼叫护士"
        false:
          - id: "n12b_remind"
            skill: "speak_text"
            params:
              text: "还有药品未取，请检查"
          - id: "n12c_wait_again"
            skill: "wait_for_weight_change"
            params:
              sensor: "medicine_box_scale"
              direction: "decrease"
              timeout_s: 60
            depends_on: ["n12b_remind"]
          - id: "n12d_farewell"
            skill: "speak_text"
            params:
              text: "祝您早日康复，如需帮助请呼叫护士"
            depends_on: ["n12c_wait_again"]
      depends_on: ["n11_wait_pickup"]

    # Phase 4: 返回与上报
    - id: "n13_nav_return"
      skill: "navigate_to_waypoint"
      params:
        target: {name: "nurse_station", map_id: "hospital_1f"}
        speed: 0.6
      depends_on: ["n12_check_empty"]

    - id: "n14_report"
      skill: "speak_text"
      params:
        text: "301病房张三送药完成"
      depends_on: ["n13_nav_return"]

    - id: "n15_log"
      skill: "log_result"
      params:
        task_type: "medication_delivery"
        patient: "张三"
        room: "301"
        status: "completed"
        details:
          load_time: {ref: "n3_wait_load.output.duration_s"}
          delivery_time: {ref: "n5_nav_room.output.duration_s"}
          pickup_time: {ref: "n11_wait_pickup.output.duration_s"}
      depends_on: ["n14_report"]
```

#### 4.3.5 DAG可视化

```
  [n1 导航药房] → [n2 语音取药] → [n3 等待放药(力)] → [n4 语音确认]
                                                          │
                                                          ▼
  [n7 开门进入(力+视)] ← [n6 语音到达] ← [n5 导航301病房]
        │
        ▼
  [n8 识别床位(视)] → [n9 导航床旁] → [n10 语音送药]
                                          │
                                          ▼
                                   [n11 等待取药(力)]
                                          │
                                          ▼
                                   [n12 检查取完(力)]
                                    ╱            ╲
                              全取出              未取完
                                │                   │
                          [n12a 祝福]         [n12b 提醒]
                                │                   │
                                │             [n12c 再等]
                                │                   │
                                │             [n12d 祝福]
                                ╲                 ╱
                                  ╲             ╱
                                    ▼         ▼
                              [n13 导航返回护士站]
                                      │
                                      ▼
                              [n14 语音上报]
                                      │
                                      ▼
                              [n15 日志记录]
```

#### 4.3.6 能力需求分析

该DAG涉及的原子能力：

| 原子能力 | 使用节点 | 必需/可选 | 降级方案 |
|---------|---------|----------|---------|
| locomotion.wheeling | n1,n5,n9,n13 | 必需 | 无（核心功能） |
| perception.localization | n1,n5,n9,n13 | 必需 | 无 |
| communication.speaking | n2,n4,n6,n10,n12a,n14 | 必需 | 降级为屏幕显示文字 |
| perception.force_sensing | n3,n7,n11,n12 | 必需 | 降级为视觉确认（精度下降） |
| perception.vision | n7,n8 | 必需 | 降级为固定坐标（需预配置） |
| manipulation.pushing | n7 | 必需 | 降级为语音通知人工开门 |
| communication.network | n15 | 必需 | 本地缓存，网络恢复后上报 |

**最低机器人配置：** 轮式移动底盘 + 扬声器 + 药箱力传感器 + RGB相机 + 推杆/机械臂（开门） + 定位系统 + WiFi

#### 4.3.7 记忆系统对该流程的优化

随着送药任务反复执行，三层记忆持续优化该流程：

| 记忆层 | 优化内容 | 效果 |
|--------|---------|------|
| **Task Memory** | 缓存该SOP的编译DAG | 第2次起编译耗时0ms |
| **Execution Memory** | 记录各路段最优速度 | 药房→301最优路径从3min降至2min |
| | 记录等待开门平均时长 | 自动调整timeout从60s→35s |
| | 记录药箱力传感器最优阈值 | 降低误判率 |
| **Spatial Memory** | 记住电梯等待时间规律 | 高峰期自动选择楼梯 |
| | 记住301病房门朝向 | 到达后自动转向面对门 |
| | 记住3号床精确位置 | 跳过视觉识别直接导航（置信度高时） |
| | 记住门把手精确位置和开门力度 | 开门成功率从85%提升至98% |

#### 4.3.8 开门Skill设计（open_door）

送药场景的核心差异化Skill。机器人需要物理操作门把手并推/拉开病房门。

**Skill接口定义：**

```yaml
skill:
  name: "open_door"
  version: "1.0.0"
  description: "物理操作门把手开门，支持推门和拉门"

  required_capabilities:
    - type: "manipulation.pushing"
      min_force_n: 30              # 最低推力
    - type: "perception.vision"
      sensor_type: "rgb_camera"    # 视觉定位门把手
    - type: "perception.force_sensing"
      sensitivity_n: 0.5           # 力控精度

  inputs:
    target_door:
      type: "string"
      description: "目标门标识（如 room_301_door）"
    door_type:
      type: "enum"
      values: ["push", "pull", "sliding"]
      default: "push"
      description: "门的开启方式"

  outputs:
    success:
      type: "bool"
    door_angle_deg:
      type: "float"
      description: "门打开角度"
    force_applied_n:
      type: "float"
      description: "实际施加力度"

  preconditions:
    - "robot.position.distance_to(target_door) < 1.0"
    - "robot.capability('manipulation.pushing').status == 'ready'"

  postconditions:
    - "door.state == 'open'"
    - "door.angle >= 60"           # 至少开60度供机器人通过

  failure_modes:
    - code: "HANDLE_NOT_FOUND"
      description: "视觉未能定位到门把手"
      recovery: "retry_after(2, max=3)"
    - code: "DOOR_LOCKED"
      description: "门被锁定，施力后无法打开"
      recovery: "escalate_to_human({reason: '病房门锁定，请开门'})"
    - code: "FORCE_EXCEEDED"
      description: "施力超过安全阈值仍未打开"
      recovery: "abort_dag({reason: '开门力度异常，可能被卡住'})"
    - code: "DOOR_TYPE_MISMATCH"
      description: "实际门类型与预期不匹配"
      recovery: "invoke_skill('open_door', {door_type: detected_type})"

  execution_steps:
    1_approach: "导航至门前0.5m，正对门面"
    2_detect_handle: "视觉检测门把手位置和类型（推/拉/滑动）"
    3_reach: "机械臂/推杆伸出到门把手位置"
    4_grip_or_push: "根据门类型执行：推门直接推、拉门先握把手再拉"
    5_force_control: "力控模式开门，监控力度不超过安全阈值（50N）"
    6_verify: "视觉确认门已打开至目标角度"
    7_retract: "收回机械臂/推杆"
```

**力控安全策略：**

| 参数 | 值 | 说明 |
|------|-----|------|
| 最大推力 | 50N | 超过则停止并告警 |
| 力控频率 | 100Hz | 力传感器采样率 |
| 碰撞检测阈值 | 30N突增 | 0.1s内力变化>30N视为碰撞 |
| 门把手定位精度 | ±2cm | 视觉+力反馈联合定位 |
| 开门超时 | 15s | 超时未打开则触发恢复策略 |

### 4.4 执行引擎设计

#### 4.4.1 核心状态机

每个Skill执行节点的生命周期：

```
                          ┌────────────────┐
                          │    PENDING      │
                          │  (等待依赖完成)  │
                          └───────┬────────┘
                                  │ 依赖全部完成
                                  ▼
                ┌──────── CHECKING_PRECONDITIONS ────────┐
                │         (校验前置条件)                   │
                │                                         │
           条件不满足                                 条件满足
           (等待/超时)                                    │
                │                                         ▼
                │                                    RUNNING
                │                                   (执行中)
                │                                    │    │
                │                              成功 ─┘    └─ 失败
                │                               │              │
                ▼                               ▼              ▼
            BLOCKED                         SUCCESS        FAILED
          (阻塞/超时)                       (完成)        (执行失败)
                │                               │              │
                │                               │         有恢复策略?
                │                               │         │        │
                │                               │        是        否
                │                               │         │        │
                │                               │         ▼        ▼
                │                               │     RECOVERING  TERMINAL_FAILED
                │                               │    (恢复中)     (最终失败)
                │                               │         │              │
                └───── 人工介入 ─────────────────┤    恢复成功?           │
                                                │    │      │           │
                                                │   是      否          │
                                                │    │      │           │
                                                │    ▼      ▼           │
                                                │ RUNNING  连续失败?     │
                                                │          │    │       │
                                                │         是    否      │
                                                │          │    │       │
                                                │          ▼    └→FAILED│
                                                │     CIRCUIT_BREAK     │
                                                │    (熔断,暂停DAG)     │
                                                │          │            │
                                                ▼          ▼            ▼
                                          ┌─────────────────────────────┐
                                          │  → 写入Practice记录          │
                                          │  → 更新Execution Memory     │
                                          │  → 上报审计日志              │
                                          └─────────────────────────────┘
```

#### 4.4.2 DAG遍历器

```cpp
// 伪代码：DAG遍历核心逻辑（C++实现，运行在机器人端）
class DAGExecutor {
public:
    void execute(const DAG& dag, ExecutionContext& context) {
        // 拓扑排序，确定执行顺序
        auto execution_order = topological_sort(dag.nodes);

        for (const auto& node : execution_order) {
            // 等待依赖节点完成
            wait_for_dependencies(node.depends_on);

            // 使用任务下发时已注入的推荐参数（远端Task Scheduler预合并）
            auto params = node.params;  // 已包含Memory推荐参数

            // 处理条件分支
            if (node.type == NodeType::CONDITIONAL) {
                auto branch = evaluate_condition(node.condition, context);
                execute_branch(node.branches.at(branch));
                continue;
            }

            // 执行Skill
            auto result = skill_runner_.execute(
                node.skill, params, node.timeout_s
            );

            // 状态转换与恢复
            if (result.failed()) {
                auto recovery_result = attempt_recovery(node, result);
                if (!recovery_result.success()) {
                    handle_failure(node, result, recovery_result);
                }
            }

            // 记录执行数据（异步上报远端）
            practice_collector_.record(node, result, context);

            // 更新上下文（供后续节点引用）
            context.set_output(node.id, result.output());
        }
    }
};
```

#### 4.4.3 异常恢复策略

每个Skill定义了失败模式和对应的恢复策略：

```yaml
recovery_actions:
  retry_after:
    description: "延迟后重试"
    params: {delay_s: 5, max_retries: 3}
    backoff: "exponential"    # fixed / linear / exponential

  invoke_skill:
    description: "调用另一个Skill来恢复"
    params: {skill_name: "navigate_to_waypoint", params: {...}}

  escalate_to_human:
    description: "上报人工处理"
    params: {context: "完整诊断上下文"}

  skip_and_log:
    description: "跳过当前节点，继续执行"
    params: {reason: "非关键节点，可跳过"}

  abort_dag:
    description: "终止整个DAG执行"
    params: {reason: "安全风险，必须停止"}
```

**级联熔断机制：**
- 连续3个Skill失败 → 触发熔断，暂停DAG执行
- 熔断后记录断点状态（已完成/未完成节点）
- 支持从断点恢复（人工确认后从失败点继续执行）
- 熔断事件立即通知Dashboard，附完整诊断上下文

#### 4.4.4 运行时环境感知与动态调整

执行引擎在运行过程中持续获取环境数据，根据实际情况调整：

```
传感器数据 ──→ 环境状态评估 ──→ 与预期对比 ──→ 调整决策
                                                  │
                              ┌────────────────────┤
                              ▼                    ▼
                         参数微调              路径变更
                    (速度/力度/精度)       (避障/绕行/重规划)
```

**动态调整示例：**
- 导航时发现障碍物 → 调用避障子策略，动态调整路径
- 拍照时光线不足 → 调整曝光参数或更换拍照位置
- 温度读数异常 → 多次测量取均值，排除传感器噪声

### 4.5 WorldState模型

#### 4.5.1 概念定义

WorldState是执行引擎维护的**对物理世界的实时认知模型**——机器人在执行任务时对"当前世界是什么样的"的结构化理解。

它在系统中扮演三个核心角色：

1. **Skill前置条件的求值上下文。** 前置条件如 `robot.position.distance_to(target_location) < 0.5` 就是对WorldState求值。没有WorldState，前置条件就无法判断"当前是否满足执行条件"。

2. **运行时动态决策的依据。** 执行引擎在DAG遍历过程中，根据WorldState的实时变化判断是否需要调整执行策略（如降速、绕行、等待）。这是"运行时判断优于预设路径"设计原则的技术基础。

3. **失败诊断的现场快照。** Skill失败时，WorldState快照作为诊断上下文的一部分写入审计日志和Practice记录，使事后分析可以还原"失败那一刻，世界是什么状态"。

**类比理解：** 如果DAG是机器人的"行动计划"，WorldState就是机器人的"实时感知"。执行引擎的工作就是在感知（WorldState）和计划（DAG）之间做动态协调。

#### 4.5.2 数据结构

```yaml
world_state:
  robot:                                     # 机器人自身状态
    position: {x: 10.5, y: 3.2, z: 0.0}
    heading: 1.57                            # 弧度
    battery_level: 0.85
    joint_states: {...}
    velocity: {linear: 0.5, angular: 0.0}

  sensors:                                   # 传感器状态
    thermal_camera:
      status: "ready"                        # ready / busy / error / offline
      last_capture_at: "2026-06-20T10:02:30Z"
    lidar:
      status: "ready"
      point_cloud_rate_hz: 10
    imu:
      status: "ready"

  environment:                               # 环境感知
    current_location: "substation-A3"
    detected_obstacles: [...]
    ambient_temperature: 25.0
    lighting_condition: "indoor_normal"

  task:                                      # 当前任务上下文
    current_dag_id: "inspection-substation-001"
    current_node_id: "n2"
    completed_nodes: ["n1"]
    elapsed_time_s: 120
```

#### 4.5.3 数据来源与更新策略

WorldState的各字段通过ROS2 topic订阅实时更新。不同字段的更新频率根据实时性要求分为三级：

| 更新级别 | 频率 | 字段 | ROS2 Topic (示例) |
|---------|------|------|-------------------|
| **高频（实时控制）** | 50-100Hz | robot.position, robot.heading, robot.velocity, robot.joint_states | `/odom`, `/joint_states` |
| **中频（状态监控）** | 1-10Hz | sensors.*.status, environment.detected_obstacles, robot.battery_level | `/sensors/*/status`, `/scan`, `/battery_state` |
| **低频（环境感知）** | 0.1-1Hz | environment.current_location, environment.ambient_temperature, environment.lighting_condition | `/localization/pose`, `/env/temperature`, `/env/light` |
| **事件驱动** | 按需 | task.* (由执行引擎内部更新，不来自ROS2) | 内部状态机事件 |

**更新机制：**
- **流式更新：** robot和sensors字段通过ROS2 subscriber持续更新，保持最新值
- **快照采集：** Skill执行前/失败时，对当前WorldState做一次完整快照（深拷贝），用于前置条件校验和诊断记录
- **本地维护：** WorldState运行在机器人端，不依赖远端网络，保证离线场景下仍可用

#### 4.5.4 WorldState与其他模块的交互

```
ROS2 Topics ──订阅──→ WorldState ──求值──→ Skill前置/后置条件
                          │
                          ├──快照──→ 审计日志（失败诊断上下文）
                          ├──快照──→ Practice记录（执行现场还原）
                          ├──读取──→ 执行引擎（动态调整决策）
                          └──同步──→ Spatial Memory（环境认知更新）
```

---

## 五、Skill管理

### 5.1 Skill接口标准

```yaml
skill:
  name: "capture_thermal_image"
  version: "1.0.0"
  category: "perception"          # navigation / perception / manipulation / communication

  # 能力声明（供SOP Compiler匹配）
  capabilities:
    tags: ["热成像", "温度检测", "红外拍摄"]
    applicable_robots: ["unitree-go2", "unitree-g1"]
    environment: ["indoor", "outdoor"]

  # 能力需求声明（与CAL层对齐，替代原有required_sensors）
  required_capabilities:
    - type: "perception.vision"
      sensor_type: "thermal_camera"
      min_resolution: [640, 480]
    - type: "perception.localization"

  # 输入输出类型定义
  inputs:
    - name: "target_location"
      type: "Pose3D"
      required: true
    - name: "capture_params"
      type: "ThermalCaptureConfig"
      required: false
      default: {resolution: "640x480", mode: "single"}

  outputs:
    - name: "thermal_image"
      type: "Image"
    - name: "temperature_map"
      type: "HeatMap"

  # 前置条件（JSON规则表达式，机器人端C++求值引擎解析）
  preconditions:
    - "robot.position.distance_to(target_location) < 0.5"
    - "capability('perception.vision').status == 'ready'"

  # 后置条件
  postconditions:
    - "outputs.thermal_image.width > 0"
    - "outputs.temperature_map.max_temp is not None"

  # 失败模式与恢复策略
  failure_modes:
    - code: "CAMERA_NOT_READY"
      severity: "recoverable"
      recovery: "retry_after(5, max=3)"
    - code: "POSITION_DRIFT"
      severity: "recoverable"
      recovery: "invoke_skill('navigate_to_waypoint', {target: target_location})"
    - code: "HARDWARE_FAULT"
      severity: "unrecoverable"
      recovery: "escalate_to_human(context)"

  timeout_s: 30
  retry_policy:
    max_retries: 3
    backoff: "exponential"
```

### 5.2 前置/后置条件表达式

条件表达式以字符串形式存储在Skill定义和DAG中，由机器人端C++求值引擎解析执行。

**表达式语法（与语言无关的DSL）：**
- 属性访问：`robot.position.x`
- 比较运算：`==`, `!=`, `<`, `>`, `<=`, `>=`
- 布尔运算：`and`, `or`, `not`
- 内建函数：`distance_to()`, `contains()`, `len()`, `is_valid()`

**C++求值引擎实现策略：**

| 方案 | 适用阶段 | 说明 |
|------|---------|------|
| **JSON规则引擎** | Phase 1（推荐） | 条件编译为JSON规则树 `{"op":"<", "left":"robot.position.distance_to(target)", "right":0.5}`，C++端实现有限的操作符和内建函数集合，安全且实现简单 |
| **Lua嵌入** | Phase 2+（可选） | 表达力更强，嵌入成本低（LuaJIT单头文件），游戏/机器人行业成熟方案。适合条件逻辑复杂化后的扩展 |

**安全保障：** 无论采用哪种方案，求值引擎只能读取WorldState，不能修改状态或调用外部资源。Phase 1的JSON规则引擎天然不存在代码注入风险。

### 5.3 Skill Registry

```
Skill Registry
├── 注册与发现
│   ├── Skill注册（名称、版本、能力声明、接口定义）
│   ├── Skill查询（按名称、能力标签、适用机器人筛选）
│   └── Skill能力匹配（SOP Compiler调用，返回匹配度分数）
│
├── 版本管理（Phase 1+）
│   ├── 多版本共存
│   ├── 版本依赖解析
│   └── 向后兼容性检查
│
├── 生命周期（Phase 3+）
│   ├── 发布/下架/回滚
│   ├── Champion标记（通过Darwin评测）
│   └── 质量评分（基于执行数据）
│
└── MVP核心Skill桩
    ├── navigate_to_waypoint    # 导航到指定点位
    ├── capture_thermal_image   # 热成像拍摄
    ├── detect_anomaly          # 异常检测
    ├── alert_operator          # 告警通知
    ├── return_to_base          # 回基站
    └── log_result              # 日志记录
```

### 5.4 Skill跨本体迁移（Phase 2+）

同一Skill意图在不同机器人上执行时，基于e-URDF进行约束适配：

```
Skill意图: "导航到目标点"
    │
    ├── 宇树Go2（四足）: 使用四足步态导航，速度上限1.2m/s
    ├── 宇树G1（人形）: 使用双足步态，速度上限0.8m/s
    └── UR5e（机械臂）: 不支持导航，标记为"不可用"
```

适配依据：e-URDF中声明的关节范围、力矩限制、工作空间边界、可执行Skill清单。

---

## 六、Dashboard设计

### 6.1 功能模块

```
Dashboard（Web UI）
│
├── 任务管理
│   ├── 任务列表（全部/运行中/已完成/失败）
│   ├── 任务创建（选择SOP/模板 → 编译 → 审核 → 下发）
│   └── 任务详情（DAG可视化、执行进度、耗时统计）
│
├── 实时监控
│   ├── DAG执行进度（节点状态实时更新）
│   ├── 机器人状态（位姿、电量、传感器状态）
│   ├── 当前Skill状态（输入/输出/耗时）
│   ├── 风险等级指示（正常/注意/警告/危险）
│   └── 环境视图（如可用，展示相机画面）
│
├── 人工接管
│   ├── 一键接管按钮
│   ├── 紧急停止按钮
│   ├── 手动控制面板（方向/速度/动作）
│   ├── 断点恢复操作
│   └── 接管日志记录
│
├── SOP编译
│   ├── SOP文本编辑器
│   ├── 编译结果预览（DAG可视化）
│   ├── SOP原文 ↔ DAG映射对照
│   ├── 低置信度映射高亮
│   └── 一键确认/手动修正
│
├── 告警中心
│   ├── 实时告警列表
│   ├── 告警详情（诊断上下文、推荐操作）
│   ├── 告警确认/处理
│   └── 历史告警查询
│
├── 记忆管理（Phase 3+）
│   ├── 任务记忆查看/失效
│   ├── 执行记忆查看/校准
│   ├── 空间记忆查看/编辑
│   └── 记忆命中统计
│
├── Practice回放（Phase 2+）
│   ├── 历史任务列表
│   ├── 逐帧回放（传感器数据流+决策过程+Skill调用链）
│   └── 执行对比分析
│
└── 审计日志
    ├── 操作日志查询
    ├── 执行日志查询
    ├── 系统事件日志
    └── 日志导出

```

### 6.2 技术架构

```
Browser ──WebSocket──→ Dashboard Backend ──→ 数据服务层
                         │
                         ├── React前端（SPA）
                         │   ├── DAG可视化（D3.js / React Flow）
                         │   ├── 实时状态面板（WebSocket推送）
                         │   ├── SOP编辑器（Monaco Editor）
                         │   └── 告警通知（浏览器通知API）
                         │
                         └── 后端（Python FastAPI）
                             ├── WebSocket服务（实时推送）
                             ├── REST API（CRUD操作）
                             ├── gRPC客户端（与机器人端通信）
                             └── 认证与授权（JWT）
```

### 6.3 实时推送机制

```
机器人端状态变化 ──gRPC──→ Task Scheduler ──WebSocket──→ Dashboard前端

推送事件类型:
- skill_state_changed:  Skill状态变化（pending/running/success/failed）
- dag_progress_updated: DAG整体进度更新
- robot_state_updated:  机器人位姿/电量/传感器更新（1Hz）
- alert_triggered:      告警触发
- human_takeover:       人工接管事件
- memory_hit:           记忆命中事件（展示哪条记忆影响了决策）
```

---

## 七、行为审计

### 7.1 审计目标

工业场景（电力、矿山、石化）对可追溯性有强合规要求。审计系统确保：
- **100%可追溯：** 每一步执行决策都有记录
- **防篡改：** 日志写入后不可修改
- **可回放：** 任何历史任务都可以完整回放

### 7.2 审计数据模型

```yaml
audit_event:
  event_id: "evt-20260620-001-042"
  timestamp: "2026-06-20T10:02:30.123Z"

  # 事件分类
  category: "skill_execution"          # sop_compilation / skill_execution /
                                       # state_transition / human_intervention /
                                       # system_event / memory_operation
  severity: "info"                     # debug / info / warning / error / critical

  # 事件上下文
  task_run_id: "run-20260620-001"
  dag_id: "inspection-substation-001"
  node_id: "n2"
  skill: "capture_thermal_image"

  # 事件详情
  action: "skill_started"
  details:
    input_params: {target_location: {x: 10.5, y: 3.2}, ...}
    param_source: "memory_recommended"   # dag_default / memory_recommended / human_override
    precondition_check: "passed"

  # 机器人状态快照
  robot_snapshot:
    position: {x: 10.4, y: 3.1, z: 0.0}
    battery: 0.85
    sensor_status: {thermal_camera: "ready", lidar: "ready"}

  # 审计链
  prev_event_hash: "sha256:def456..."
  event_hash: "sha256:abc123..."       # SHA256(prev_hash + event_content)
```

### 7.3 审计事件类型

| 类别 | 事件 | 触发时机 |
|------|------|----------|
| **SOP编译** | sop_compiled | SOP编译完成 |
| | sop_cache_hit | 命中编译缓存 |
| | sop_human_reviewed | 人工审核通过/修正 |
| **Skill执行** | skill_started | Skill开始执行 |
| | skill_completed | Skill执行完成（含结果） |
| | skill_failed | Skill执行失败（含错误码和诊断） |
| | skill_timeout | Skill执行超时 |
| **状态转换** | precondition_checked | 前置条件校验（含结果） |
| | recovery_attempted | 恢复策略执行 |
| | circuit_breaker_triggered | 熔断触发 |
| | checkpoint_saved | 断点保存 |
| **人工介入** | human_takeover_started | 人工接管开始 |
| | human_takeover_ended | 人工接管结束 |
| | human_override_param | 人工覆盖参数 |
| | emergency_stop | 紧急停止 |
| **记忆操作** | memory_read | 读取记忆（含命中/未命中） |
| | memory_write | 写入/更新记忆 |
| | memory_invalidated | 记忆失效 |
| **系统事件** | connection_lost | 网络断开 |
| | connection_restored | 网络恢复 |
| | dag_started | DAG执行开始 |
| | dag_completed | DAG执行完成 |

### 7.4 存储与防篡改

```
审计日志存储架构:

机器人端:
  本地缓冲区（SQLite WAL模式）
  ├── 实时写入（每个事件即时写入）
  ├── 批量上报（每30秒或缓冲区满时上报远端）
  └── 本地保留7天（用于离线场景）

远端:
  Append-only存储
  ├── PostgreSQL + 审计表（只INSERT，不UPDATE/DELETE）
  ├── 哈希链校验（每条记录包含前一条的hash）
  ├── 定期完整性校验（每小时验证哈希链）
  └── 归档（90天后归档到冷存储）

防篡改机制:
  1. Append-only：数据库层面禁止UPDATE/DELETE
  2. 哈希链：每条记录 = SHA256(上一条hash + 当前内容)
  3. 时间戳校验：事件时间戳单调递增
  4. 数字签名：机器人端使用设备密钥签名（Phase 2+）
```

### 7.5 审计查询API

```python
# 查询某次任务的完整审计轨迹
audit.query(task_run_id="run-20260620-001")

# 查询某个Skill的所有失败记录
audit.query(skill="navigate_to_waypoint", action="skill_failed",
            time_range=("2026-06-01", "2026-06-20"))

# 查询所有人工介入事件
audit.query(category="human_intervention")

# 验证审计链完整性
audit.verify_chain(task_run_id="run-20260620-001")  # → True/False
```

---

## 八、记忆系统设计

### 8.1 三层记忆架构

```
┌──────────────────────────────────────────────────────────┐
│                     Memory Store                          │
│                                                           │
│  ┌────────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │  Task Memory   │  │  Execution   │  │   Spatial    │ │
│  │  (任务记忆)    │  │   Memory     │  │   Memory     │ │
│  │               │  │ (执行记忆)    │  │  (空间记忆)  │ │
│  │ SOP→DAG缓存   │  │ Skill最优参数│  │ 环境认知图   │ │
│  │ KV Store      │  │ 时序数据      │  │ 空间索引     │ │
│  └───────┬────────┘  └──────┬───────┘  └──────┬───────┘ │
│          │                  │                  │          │
│  ────────┴──────────────────┴──────────────────┴──────── │
│                    统一Memory API                         │
│  memory.task.get/put/invalidate                          │
│  memory.exec.query/record/recommend                      │
│  memory.spatial.get/update/find_nearest                  │
└──────────────────────────────────────────────────────────┘
         │                │                │
    存储后端             存储后端          存储后端
  Phase 1: SQLite     Phase 1: SQLite   Phase 1: SQLite
  Phase 3+: PG        Phase 3+: PG     Phase 3+: PG+PostGIS
```

### 8.2 Task Memory（任务记忆）

**用途：** 缓存SOP编译结果，避免重复LLM调用。

```yaml
task_memory_entry:
  key: "sha256:hash(SOP文本+Skill_Registry_v1.2+Robot_Profile_unitree-go2)"
  value:
    compiled_dag: {完整DAG YAML}
    compiler_version: "1.0.0"
    llm_provider: "gpt-4o"
    compilation_time_ms: 3200
    human_review_status: "approved"
    human_corrections:
      - {step: 3, original_skill: "log_result", corrected_skill: "alert_operator"}
  metadata:
    created_at: "2026-06-20T10:00:00Z"
    last_used: "2026-06-20T15:30:00Z"
    use_count: 47
    success_rate: 0.96
```

**查询流程：**
1. SOP Compiler在编译前计算缓存Key
2. 查询Task Memory：命中且success_rate >= 0.9 → 直接返回（0ms）
3. 未命中 → 走完整编译流程 → 结果写入Task Memory

### 8.3 Execution Memory（执行记忆）

**用途：** 记录Skill在特定环境+机器人型号下的最优参数。

```yaml
execution_memory_entry:
  skill: "navigate_to_waypoint"
  context:
    environment: "substation-A3"
    robot_model: "unitree-go2"

  entries:                           # 历史执行记录
    - params: {speed: 0.6}
      outcome: "success"
      duration_ms: 12000
      timestamp: "2026-06-20T10:05:00Z"
    - params: {speed: 0.8}
      outcome: "success"
      duration_ms: 8500
      timestamp: "2026-06-20T15:35:00Z"

  derived:                           # 自动推导的推荐
    optimal_speed: 0.8
    avg_duration_ms: 10250
    success_rate: 1.0
    recommended_params: {speed: 0.8}
```

**参数合并优先级：**
1. 人工覆盖参数（最高优先级）
2. DAG中显式指定的参数
3. Execution Memory推荐参数
4. Skill定义的默认参数（最低优先级）

### 8.4 Spatial Memory（空间记忆）

**用途：** 持久化机器人对环境的认知。

```yaml
spatial_memory:
  environment: "substation-A3"
  last_full_scan: "2026-06-20T10:00:00Z"

  landmarks:                         # 环境中的设备/物体
    - id: "transformer-T1"
      type: "equipment"
      position: {x: 5.2, y: 1.8, z: 0.0}
      properties:
        model: "SZ11-31500/110"
        normal_temp_range: [35, 65]
        last_observed_temp: 52
        last_observed_at: "2026-06-20T10:02:30Z"
      status: "normal"

      # 视觉参考（Visual Reference）
      visual_references:
        - state: "normal"
          image_path: "spatial/substation-A3/transformer-T1/normal_ref.jpg"
          captured_at: "2026-06-15T10:00:00Z"
          description: "正常状态：表面无锈蚀，散热风扇运转正常，接线端无放电痕迹"
        - state: "anomaly_overheat"
          image_path: "spatial/substation-A3/transformer-T1/anomaly_overheat_ref.jpg"
          captured_at: "2026-06-18T14:30:00Z"
          description: "过热异常：红外热成像显示顶部热点>85°C"
        - state: "anomaly_rust"
          image_path: "spatial/substation-A3/transformer-T1/anomaly_rust_ref.jpg"
          captured_at: null  # 尚未观测到，由人工上传
          description: "锈蚀异常：外壳表面出现明显锈蚀斑块"

  paths:                             # 已探索路径
    - from: "entrance"
      to: "transformer-T1"
      distance_m: 8.5
      avg_traverse_time_ms: 15000
      obstacles: ["cable_tray_at_3m"]

  changes_detected:                  # 环境变化记录
    - timestamp: "2026-06-20T15:35:00Z"
      landmark: "transformer-T1"
      field: "last_observed_temp"
      old_value: 52
      new_value: 78
      alert_generated: true
```

**空间记忆的价值：**
- 导航优化：已知路径复用，避免盲目规划
- 异常检测基线：基于历史数据对比，而非固定阈值
- 增量巡检：对稳定设备降低频率，对异常设备增加关注

**视觉参考（Visual Reference）：**

Spatial Memory中的每个landmark可携带**视觉参考图片**，记录该对象在不同状态下的外观。这使异常检测从"纯数值阈值"进化为"图像对比+数值阈值"双重判断：

| 场景 | 数值检测 | 视觉对比 | 联合判断 |
|------|---------|---------|---------|
| 变压器表面锈蚀 | 温度正常，无法检测 | 与正常参考图对比发现外观变化 | 触发锈蚀告警 |
| 配电柜门未关 | 无对应传感器 | 与正常状态图对比发现柜门开启 | 触发安全告警 |
| 仪表读数异常 | 仪表无联网输出 | OCR读数与参考图对比发现偏差 | 触发读数异常告警 |

视觉参考图片来源：
1. **自动采集**：巡检过程中自动拍摄并更新（状态为normal时覆盖旧图）
2. **人工上传**：运维人员通过Dashboard上传标准参考图（如"正常状态""已知异常"）
3. **异常存档**：检测到异常时自动保存当次图片作为异常参考

### 8.5 记忆原子（Memory Atom）

Practice自动分析后提取的结构化记忆单元：

```yaml
memory_atom:
  id: "atom-20260620-001"
  type: "success_pattern"              # success_pattern / failure_lesson /
                                       # env_feature / optimal_param
  source_practice: "practice-20260620-001"

  content:
    skill: "navigate_to_waypoint"
    insight: "在substation-A3的入口走廊，速度0.8比0.6快30%且同样安全"
    evidence:
      success_count: 15
      fail_count: 0
      avg_duration_improvement: "30%"

  weight: 0.85                         # 权重，随新Practice注入持续更新
  created_at: "2026-06-20T10:00:00Z"
  last_updated: "2026-06-20T15:30:00Z"
  update_count: 15
```

### 8.6 记忆更新管线

```
Skill执行完成
    │
    ▼
[实时更新 Layer 1]
    ├── 执行成功 → 更新Execution Memory（追加记录，更新推荐参数）
    ├── 新环境观测 → 更新Spatial Memory（更新landmark/path）
    ├── DAG执行成功 → 更新Task Memory（更新success_rate/use_count）
    └── 观测到异常值 → Spatial Memory记录变化
    │
    ▼
[准实时更新 Layer 2]（Phase 1+）
    ├── Skill失败并恢复 → 记录恢复策略效果
    ├── 记忆命中 → 更新记忆权重
    └── 参数推荐被采纳 → 强化推荐置信度
    │
    ▼
[离线分析 Layer 3]（Phase 2+）
    ├── Skill参数调优（每日，基于Execution Memory）
    ├── SOP编译质量评估（每周，基于人工修正记录）
    ├── 异常模式挖掘（每周，基于Spatial Memory变化）
    ├── 失败模式分类（每周，更新recovery策略优先级）
    └── 巡检路径优化（每月，基于Spatial Memory路径数据）
```

---

## 九、Practice记录系统

### 9.1 Practice数据结构

每次任务执行生成一条完整的Practice记录：

```yaml
practice:
  id: "practice-20260620-001"
  task_run_id: "run-20260620-001"
  dag_id: "inspection-substation-001"
  robot_model: "unitree-go2"
  environment: "substation-A3"

  timeline:                          # 统一时间线
    start_time: "2026-06-20T10:00:00Z"
    end_time: "2026-06-20T10:30:00Z"
    duration_s: 1800

  execution_records:                 # 每个节点的执行记录
    - node_id: "n1"
      skill: "navigate_to_waypoint"
      started_at: "2026-06-20T10:00:05Z"
      completed_at: "2026-06-20T10:01:20Z"
      duration_ms: 75000
      input_params: {target: {x: 10.5, y: 3.2}, speed: 0.8}
      param_source: "memory_recommended"
      outcome: "success"
      output_summary: {final_position: {x: 10.4, y: 3.1}}
      retry_count: 0
      robot_pose_at_start: {x: 0.0, y: 0.0, heading: 0.0}
      robot_pose_at_end: {x: 10.4, y: 3.1, heading: 1.57}

  decisions:                         # 执行中的决策记录
    - timestamp: "2026-06-20T10:00:30Z"
      type: "param_adjustment"
      reason: "检测到路径障碍物"
      before: {speed: 0.8}
      after: {speed: 0.5}

  memory_hits:                       # 记忆命中记录
    - memory_type: "execution"
      skill: "navigate_to_waypoint"
      recommended: {speed: 0.8}
      adopted: true

  environment_changes:               # 环境变化记录
    - timestamp: "2026-06-20T10:02:30Z"
      type: "temperature_change"
      landmark: "transformer-T1"
      value: {from: 52, to: 78}

  overall_result:
    status: "completed"              # completed / partial / failed / aborted
    nodes_total: 5
    nodes_completed: 5
    nodes_failed: 0
    autonomy_rate: 1.0               # 本次执行的自主率
```

### 9.2 Practice存储

- **Phase 1：** SQLite，按task_run_id索引
- **Phase 3+：** TimescaleDB，支持时序查询和聚合分析
- **存储策略：** 完整Practice保留90天，之后压缩为摘要（保留统计数据，丢弃传感器原始数据）

---

## 十、Provider模块设计

### 10.1 边云协同推理架构

模型推理不是全部在云端，也不是全部在边缘——**根据延迟要求、模型规模、数据敏感度，将推理任务分配到最合适的位置。**

```
┌──────────────────────────────────────────────────────────────┐
│                     远端 Provider 层                          │
│                                                              │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐              │
│  │ Cloud LLM  │ │ Cloud VLM  │ │ Private LLM│              │
│  │ (GPT-4o /  │ │ (通义千问VL│ │ (本地私有化 │              │
│  │  通义千问)  │ │  / GPT-4V) │ │  大模型)   │              │
│  └──────┬─────┘ └──────┬─────┘ └──────┬─────┘              │
│         └──────────────┼──────────────┘                      │
│                        │                                      │
│  ┌─────────────────────▼──────────────────────────────────┐ │
│  │           Provider Router（远端路由器）                    │ │
│  │  - 根据请求类型、延迟要求、Provider可用性选择最优Provider │ │
│  │  - 模型版本管理与边缘模型分发                             │ │
│  │  - 推理结果审计与成本追踪                                 │ │
│  └─────────────────────┬──────────────────────────────────┘ │
└────────────────────────┼────────────────────────────────────┘
                         │ gRPC
─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─│─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ 网络边界
                         │
┌────────────────────────▼────────────────────────────────────┐
│                   机器人端 Edge Inference                     │
│                                                              │
│  ┌─────────────────────────────────────────────────────────┐│
│  │          Edge Provider Router（边缘路由器）               ││
│  │  - 本地优先：能在边缘推理的不上云                         ││
│  │  - 延迟敏感任务强制走边缘                                 ││
│  │  - 边缘不可用时回退到远端                                 ││
│  └────┬────────────┬────────────┬───────────┬──────────────┘│
│       │            │            │           │                │
│  ┌────▼────┐ ┌─────▼─────┐ ┌───▼───┐ ┌────▼────┐          │
│  │目标检测  │ │ 异常识别   │ │语音    │ │场景理解  │          │
│  │(YOLO/   │ │(分类/分割  │ │ASR/TTS│ │(边缘VLM │          │
│  │ RT-DETR)│ │ 小模型)    │ │       │ │ 小模型)  │          │
│  └─────────┘ └───────────┘ └───────┘ └─────────┘          │
│                                                              │
│  ┌─────────────────────────────────────────────────────────┐│
│  │          推理基础设施                                      ││
│  │  [ONNX Runtime / TensorRT] [模型热加载] [GPU/NPU管理]    ││
│  │  [推理队列] [结果缓存] [模型版本同步]                     ││
│  └─────────────────────────────────────────────────────────┘│
└──────────────────────────────────────────────────────────────┘
```

### 10.2 边缘 vs 远端：推理任务分配

| 推理任务 | 部署位置 | 延迟要求 | 模型规模 | 理由 |
|---------|---------|---------|---------|------|
| **SOP编译** | 远端 | 秒级（可容忍） | 大（>7B参数） | 离线编译，不要求实时；需要强推理能力 |
| **执行决策** | 远端（复杂）/ 边缘（简单） | 100ms-1s | 大/小 | 复杂决策走远端，简单条件判断走边缘规则引擎 |
| **目标检测** | **边缘** | <50ms | 小（<100MB） | 导航、巡检、送药（门/床位/门把手检测）中持续运行，必须低延迟 |
| **异常识别** | **边缘** | <100ms | 小（<200MB） | 检测到温度/外观异常需实时响应 |
| **语音ASR** | **边缘** | <200ms | 小（Whisper tiny/small） | 人机交互实时性要求高，网络不可依赖 |
| **语音TTS** | **边缘** | <100ms | 小（<100MB） | 即时语音播报，不能等网络往返 |
| **图像深度分析** | 远端 | 秒级（可容忍） | 大（VLM >7B） | 复杂场景理解，边缘算力不足 |
| **场景理解(轻量)** | **边缘** | <200ms | 中（1-3B VLM） | 实时环境感知，如门是否打开、床位识别 |
| **知识检索** | 远端 | 秒级 | 大（向量库+LLM） | Know/How引擎，数据量大 |
| **动作生成(VLA)** | **边缘** | <50ms | 中 | 运动控制实时性要求极高 |

**核心原则：与物理世界实时交互的推理放边缘，离线分析和复杂推理放远端。**

### 10.3 Provider抽象接口

Provider接口同时适用于远端和边缘，通过 `deployment` 字段区分：

```python
# 远端Provider定义（Python，运行在远端服务器）
class Provider(ABC):
    """统一模型能力接口"""

    @abstractmethod
    def declare_capabilities(self) -> ProviderCapabilities:
        """声明能力：支持的输入类型、输出类型、延迟范围"""
        pass

    @abstractmethod
    async def invoke(self, request: ProviderRequest) -> ProviderResponse:
        """调用模型"""
        pass

    @abstractmethod
    def health_check(self) -> bool:
        """健康检查"""
        pass


class ProviderCapabilities:
    name: str                    # "gpt-4o", "qwen-vl", "edge-yolo", "edge-whisper"
    model_type: str              # "llm", "vlm", "vla", "vln", "detector", "asr", "tts"
    input_types: list[str]       # ["text", "image", "video", "audio"]
    output_types: list[str]      # ["text", "json", "action", "bbox", "audio"]
    max_tokens: int              # LLM类有效
    avg_latency_ms: int
    supports_streaming: bool
    deployment: str              # "cloud", "private", "edge"
    hardware_requirement: str    # "gpu", "npu", "cpu"（边缘模型需声明）
```

```cpp
// 边缘Provider定义（C++，运行在机器人端）
class EdgeProvider {
public:
    virtual ~EdgeProvider() = default;

    // 声明能力（名称、输入/输出类型、延迟范围、硬件需求）
    virtual ProviderCapabilities declare_capabilities() const = 0;

    // 同步推理（边缘模型延迟低，不需要异步）
    virtual InferenceResult invoke(const InferenceRequest& request) = 0;

    // 健康检查（含GPU/NPU状态）
    virtual bool health_check() const = 0;

    // 模型热加载/卸载（远端下发新版本模型时调用）
    virtual bool load_model(const std::string& model_path) = 0;
    virtual void unload_model() = 0;

    // 获取推理统计（耗时、吞吐量、GPU利用率）
    virtual InferenceStats get_stats() const = 0;
};
```

### 10.4 边缘推理运行时（Edge Inference Runtime）

运行在机器人端的C++推理基础设施：

```
Edge Inference Runtime（C++）
├── 模型管理器（Model Manager）
│   ├── 模型加载/卸载/热切换
│   ├── 模型版本管理（与远端同步）
│   ├── 模型格式支持：ONNX / TensorRT / OpenVINO
│   └── 模型存储：本地磁盘 + LRU缓存
│
├── 推理调度器（Inference Scheduler）
│   ├── 推理请求队列（优先级调度）
│   ├── GPU/NPU资源分配
│   ├── 批量推理优化（多个检测请求合批）
│   └── 推理超时管理
│
├── 硬件加速抽象（HAL）— 详见2.4.3节
│   ├── TensorRT Backend   ← Jetson AGX Orin (Phase 1)
│   ├── CANN Backend       ← 华为昇腾 (Phase 2)
│   ├── RKNN Backend       ← 瑞芯微RK3588 (Phase 2)
│   ├── SAIL Backend       ← 算能BM1684X (Phase 2)
│   ├── OpenVINO Backend   ← Intel (兼容)
│   └── ONNX Runtime Backend ← 纯CPU回退 (通用)
│
└── 边缘Provider实例
    ├── EdgeDetectorProvider    → 目标检测（YOLO/RT-DETR）
    ├── EdgeAnomalyProvider    → 异常识别（分类/分割模型）
    ├── EdgeASRProvider        → 语音识别（Whisper tiny/small）
    ├── EdgeTTSProvider        → 语音合成（VITS/edge-tts）
    └── EdgeVLMProvider        → 轻量场景理解（MobileVLM等）
```

**硬件适配策略（按阶段）：**

| 阶段 | 计算平台 | AI算力 | 可运行的边缘模型 |
|------|---------|--------|----------------|
| **Phase 1** | Jetson AGX Orin (64GB) | 275 TOPS | 全部边缘模型 + VLM(3B)，性能基线 |
| **Phase 2** | 昇腾310P / BM1684X | 24-32 TOPS | 检测+异常+ASR/TTS，VLM需量化 |
| **Phase 2** | RK3588 (8GB) | 6 TOPS | 检测+ASR，其他回退远端 |
| **Phase 3+** | 昇腾910B / 寒武纪MLU | 128-320 TOPS | 全部边缘模型（国产高算力平台） |
| **通用回退** | x86/ARM CPU | — | 仅ONNX Runtime CPU推理，性能降级 |

### 10.5 Provider路由与降级

远端和边缘各有一个路由器，协同工作：

```
Skill执行中产生推理请求
    │
    ▼
[Edge Provider Router]（机器人端C++，本地优先）
    │
    ├── 检查：边缘是否有对应的Provider？
    │   │
    │   ├── 有且健康 → 边缘本地推理 → 返回结果
    │   │
    │   ├── 有但降级（GPU过载/模型未加载）
    │   │   ├── 延迟敏感（<100ms）→ 等待或使用降级结果
    │   │   └── 延迟可容忍 → 转发到远端
    │   │
    │   └── 没有 → 转发到远端Provider Router
    │
    ▼
[Remote Provider Router]（远端Python）
    │
    ├── Cloud LLM/VLM（首选）
    ├── Private LLM（备选，数据不出厂区）
    └── 全部不可用 → 规则引擎兜底 + 人工升级
```

**离线场景：** 网络断开时，所有推理请求仅走边缘。边缘不支持的能力（如SOP编译）排队等待网络恢复。这要求**核心执行链路上的推理必须有边缘方案**。

### 10.6 模型版本管理与分发

远端负责模型的版本管理和向边缘分发：

```yaml
edge_model_manifest:
  models:
    - name: "yolo-inspection-v3"
      type: "detector"
      version: "3.1.0"
      formats:
        tensorrt: "yolo_v3.1.0_orin.engine"     # Jetson预编译
        rknn: "yolo_v3.1.0_rk3588.rknn"         # RK3588预编译
        onnx: "yolo_v3.1.0.onnx"                # 通用格式
      size_mb: 45
      min_hardware: "npu"
      checksum: "sha256:abc123..."

    - name: "whisper-small-zh"
      type: "asr"
      version: "1.0.0"
      formats:
        onnx: "whisper_small_zh.onnx"
      size_mb: 240
      min_hardware: "cpu"                        # CPU即可运行
      checksum: "sha256:def456..."

  sync_policy:
    check_interval_hours: 24                     # 每日检查新版本
    download_window: "02:00-05:00"               # 凌晨下载，不影响白天执行
    rollback_on_failure: true                    # 新模型推理质量下降时自动回滚
```

**分发流程：**
1. 远端训练/优化新版本模型 → 发布到模型仓库
2. 机器人端定期检查（或远端推送通知）
3. 在低负载时段下载新模型（增量更新）
4. 沙箱验证：用本地测试集验证推理质量，通过后替换
5. 审计记录：记录模型版本变更（何时、何版本、验证结果）

### 10.7 Skill与Provider的交互

Skill定义中声明推理需求，由Edge/Remote Provider Router满足：

```yaml
skill:
  name: "detect_anomaly"
  version: "1.0.0"

  # 推理需求声明（新增）
  inference_requirements:
    - name: "object_detection"
      type: "detector"
      preferred_deployment: "edge"           # 优先边缘
      max_latency_ms: 50                     # 延迟硬约束
      input: "rgb_image"
      output: "bounding_boxes"

    - name: "anomaly_classification"
      type: "classifier"
      preferred_deployment: "edge"
      max_latency_ms: 100
      fallback_deployment: "remote"          # 边缘不可用时回退远端
      input: "cropped_image"
      output: "anomaly_label + confidence"

    - name: "deep_analysis"
      type: "vlm"
      preferred_deployment: "remote"         # 优先远端（需要大模型能力）
      max_latency_ms: 3000                   # 可容忍秒级延迟
      input: "image + context_text"
      output: "analysis_report"
      optional: true                         # 可选：没有也能完成基础检测
```

### 10.8 模型输出审计

每次Provider调用记录（远端和边缘统一格式）：

```yaml
provider_call_log:
  call_id: "call-20260620-001"
  provider: "edge-yolo-inspection-v3"        # 或 "gpt-4o"
  deployment: "edge"                         # edge / cloud / private
  purpose: "object_detection"
  skill: "detect_anomaly"
  node_id: "n2"

  # 输入摘要
  input_summary: "RGB image 1920x1080, substation-A3"
  input_size_bytes: 6220800

  # 输出摘要
  output_summary: "3 objects detected, max_conf=0.95"
  output_size_bytes: 1024

  # 性能指标
  latency_ms: 28                             # 边缘推理通常很快
  hardware_used: "gpu"                       # gpu / npu / cpu
  gpu_utilization: 0.45

  # 仅远端LLM类Provider
  input_tokens: null
  output_tokens: null
  cost_estimate: null
  data_sensitivity: null

  timestamp: "2026-06-20T10:02:30.123Z"
```

---

## 十一、场景模板系统

### 11.1 场景模板的角色与定位

场景模板是RobotClaw面向**集成商和行业客户**的交付单元，将平台的通用能力封装为特定行业的"开箱即用"方案。

**核心作用：**

1. **降低接入门槛**：集成商无需理解DAG/Skill/Provider等平台概念，通过模板配置界面即可完成部署
2. **固化行业Know-How**：将经过验证的SOP→Skill映射、失败处理策略、最优参数积累在模板中
3. **加速项目交付**：典型项目从"配置模板参数"开始，而非从"编写SOP"开始，交付时间从数周降至数天
4. **质量基线保障**：模板中的Skill组合和失败处理方案经过验证，避免集成商从零开始试错

**场景模板与Skill的关系：**

```
┌──────────────────────────────────────────────────────┐
│                   场景模板（Scene Template）            │
│                                                       │
│  ┌─────────────────────────────────────────────────┐ │
│  │  预配置SOP（自然语言）                            │ │
│  │  "从基站出发 → 巡检1号变压器 → 热成像检测 → ..."  │ │
│  └──────────────────────┬──────────────────────────┘ │
│                         │ SOP Compiler编译             │
│                         ▼                              │
│  ┌─────────────────────────────────────────────────┐ │
│  │  已验证的Skill DAG                               │ │
│  │                                                   │ │
│  │  [navigate_to_waypoint] ──→ [capture_thermal]    │ │
│  │         │                        │                │ │
│  │         │                   [detect_anomaly]      │ │
│  │         │                   ╱            ╲        │ │
│  │         │         [alert_operator]  [log_result]  │ │
│  │         │                                         │ │
│  │         └──→ ... ──→ [return_to_base]             │ │
│  └─────────────────────────────────────────────────┘ │
│                                                       │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────┐ │
│  │ 可配参数模板  │  │ 失败处理方案  │  │ 适用机器人  │ │
│  │ 点位/阈值/速度│  │ 绕行/重试/跳过│  │ Go2/G1/UR  │ │
│  └──────────────┘  └──────────────┘  └────────────┘ │
└──────────────────────────────────────────────────────┘
```

**模板 ≠ Skill集合。** 模板是Skill的"上层编排"加上行业参数化配置。同一组Skill可以组成不同模板（如同样的导航+检测Skill，在电力巡检模板中按变压器点位编排，在管廊模板中按管道段编排）。模板的价值在于：已经替集成商完成了SOP编写、Skill选型、参数调优和失败处理设计。

### 11.2 模板结构

```yaml
scene_template:
  name: "电力巡检"
  version: "1.0.0"
  category: "power_inspection"

  # 预配置SOP
  default_sop: |
    1. 从基站出发，导航到1号变压器
    2. 拍摄1号变压器红外热成像
    3. 如果温度超过80度，发送高温告警
    4. 导航到2号变压器...
    5. 完成所有点位后返回基站

  # 已验证的Skill组合
  required_skills:
    - navigate_to_waypoint
    - capture_thermal_image
    - detect_anomaly
    - alert_operator
    - return_to_base
    - log_result

  # 可配置参数
  configurable_params:
    waypoints:
      type: "list[Pose3D]"
      description: "巡检点位列表"
    temperature_threshold:
      type: "float"
      default: 80.0
      description: "温度告警阈值(摄氏度)"
    patrol_speed:
      type: "float"
      default: 0.8
      range: [0.3, 1.2]
      description: "巡检速度(m/s)"
    alert_recipients:
      type: "list[string]"
      description: "告警接收人列表"

  # 典型失败处理方案
  failure_handling:
    navigation_blocked: "尝试绕行，3次失败后跳过并告警"
    camera_error: "重试3次，失败后继续下一点位"
    low_battery: "电量<20%时返回基站充电"

  # 适用机器人
  compatible_robots: ["unitree-go2", "unitree-g1"]
```

**医院送药模板（MVP首要验证模板）：**

```yaml
scene_template:
  name: "医院送药"
  version: "1.0.0"
  category: "hospital_medication_delivery"

  # 预配置SOP
  default_sop: |
    1. 到药房窗口，语音播报取药请求
    2. 等待药品放入药箱，力传感器确认
    3. 语音确认药品已接收
    4. 导航至目标病房门口
    5. 语音播报到达通知
    6. 推开病房门进入
    7. 识别目标床位
    8. 导航至床旁，语音播报送药信息
    9. 等待患者取药，力传感器确认
    10. 检查药品是否全部取出
    11. 语音播报送药完成
    12. 导航返回护士站
    13. 语音上报送药结果

  # 已验证的Skill组合
  required_skills:
    - navigate_to_waypoint
    - speak_text
    - wait_for_weight_change
    - detect_target
    - open_door
    - check_weight
    - alert_operator
    - log_result

  # 可配置参数
  configurable_params:
    pharmacy_location:
      type: "Pose3D"
      description: "药房窗口位置"
    nurse_station_location:
      type: "Pose3D"
      description: "护士站位置"
    room_locations:
      type: "map[string, Pose3D]"
      description: "病房位置映射（如 room_301 → 坐标）"
    bed_locations:
      type: "map[string, Pose3D]"
      description: "床位位置映射（可选，有Spatial Memory后自动学习）"
    weight_threshold_g:
      type: "float"
      default: 10.0
      description: "药箱重量变化检测阈值(克)"
    door_type:
      type: "enum"
      values: ["push", "pull", "sliding"]
      default: "push"
      description: "病房门类型"
    delivery_speed:
      type: "float"
      default: 0.6
      range: [0.3, 0.8]
      description: "送药导航速度(m/s)"

  # 典型失败处理方案
  failure_handling:
    door_locked: "语音通知护士站开门，等待60秒"
    navigation_blocked: "尝试绕行，3次失败后通知护士站"
    medicine_not_loaded: "超时120秒未放药，通知药房"
    medicine_not_picked: "语音重复提醒，超时120秒通知护士站"
    door_handle_not_found: "重试3次，失败后通知护士站开门"

  # 适用机器人
  compatible_robots: ["custom-med-delivery-v1"]
```

### 11.3 模板使用流程

```
集成商拿到模板
    │
    ├── 配置（80%工作量）
    │   ├── 导入客户巡检点位坐标
    │   ├── 设置温度/异常阈值
    │   ├── 配置告警接收人
    │   └── 调整巡检速度等参数
    │
    ├── 补充（15%工作量）
    │   ├── 添加客户特有的检测Skill
    │   └── 调整失败处理策略
    │
    └── 定制（5%工作量）
        └── 处理特殊场景需求
```

---

## 十二、e-URDF设计

### 12.1 扩展内容

在标准URDF基础上增加：

```xml
<robot name="unitree_go2">
  <!-- 标准URDF内容：links, joints, sensors -->
  ...

  <!-- e-URDF扩展 -->
  <eurdf:extensions>
    <!-- 关节力矩限制和软限制 -->
    <eurdf:joint_limits joint="front_left_hip">
      <eurdf:torque_limit max="40.0" unit="Nm"/>
      <eurdf:soft_limit lower="-2.5" upper="2.5" k_velocity="10.0"/>
    </eurdf:joint_limits>

    <!-- 工具坐标系 -->
    <eurdf:tool_frame name="thermal_camera_frame">
      <origin xyz="0.15 0.0 0.05" rpy="0 0 0"/>
      <parent link="head_link"/>
    </eurdf:tool_frame>

    <!-- 工作空间边界 -->
    <eurdf:workspace>
      <eurdf:boundary type="cylinder" radius="2.0" height="1.5"/>
      <eurdf:reachable_volume>...</eurdf:reachable_volume>
    </eurdf:workspace>

    <!-- 可执行Skill清单 -->
    <eurdf:skill_manifest>
      <eurdf:skill name="navigate_to_waypoint" supported="true"/>
      <eurdf:skill name="capture_thermal_image" supported="true"
                   sensor_requirement="thermal_camera"/>
      <eurdf:skill name="pick_and_place" supported="false"
                   reason="no_manipulator"/>
    </eurdf:skill_manifest>

    <!-- Benchmark指标 -->
    <eurdf:benchmark>
      <eurdf:max_speed value="1.2" unit="m/s"/>
      <eurdf:battery_life value="120" unit="min"/>
      <eurdf:payload_capacity value="5.0" unit="kg"/>
    </eurdf:benchmark>
  </eurdf:extensions>
</robot>
```

### 12.2 e-URDF的作用

1. **能力抽象层**：e-URDF中的`<eurdf:capabilities>`声明是CAL的数据来源，定义机器人具备哪些原子能力及其物理约束
2. **SOP Compiler**：基于能力画像判断机器人是否支持某Skill，做能力降级决策
3. **执行引擎**：基于关节限制和工作空间校验执行参数
4. **Skill迁移**：基于物理约束为不同本体适配Skill参数
5. **Dashboard**：展示机器人物理能力概览

### 12.3 e-URDF定义工具

为降低新机器人接入的工程量，平台提供**e-URDF定义工具**，支持在后台快速创建和维护e-URDF文件：

#### 12.3.1 工具功能

```
┌──────────────────────────────────────────────────┐
│              e-URDF Definition Tool               │
│                                                   │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐ │
│  │ URDF导入   │  │ 向导式表单  │  │ 模板继承   │ │
│  │ 自动转换   │  │ 逐步填写   │  │ 快速派生   │ │
│  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘ │
│        │               │               │         │
│        └───────────────┼───────────────┘         │
│                        ▼                          │
│  ┌─────────────────────────────────────────────┐ │
│  │  e-URDF编辑器                               │ │
│  │  ├── 关节力矩/软限制配置                      │ │
│  │  ├── 工具坐标系定义                           │ │
│  │  ├── 工作空间边界绘制                         │ │
│  │  ├── 能力声明勾选（从能力清单中选择）          │ │
│  │  ├── Skill清单关联                            │ │
│  │  ├── Benchmark参数填写                        │ │
│  │  └── 传感器配置（型号/安装位/参数）            │ │
│  └──────────────────────┬──────────────────────┘ │
│                         ▼                         │
│  ┌─────────────────────────────────────────────┐ │
│  │  校验与发布                                  │ │
│  │  ├── XML Schema校验                          │ │
│  │  ├── 物理约束一致性检查                       │ │
│  │  ├── 能力与Skill可达性验证                    │ │
│  │  └── 版本化存储 + Skill Registry联动更新      │ │
│  └─────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────┘
```

#### 12.3.2 三种创建方式

| 方式 | 适用场景 | 流程 |
|------|---------|------|
| **URDF导入** | 已有标准URDF的机器人（如从ROS2包获取） | 上传URDF → 自动解析link/joint → 向导补充e-URDF扩展字段 → 校验 → 保存 |
| **向导式表单** | 全新机器人，无现有描述文件 | 基本信息 → 传感器配置 → 关节约束 → 能力声明 → Skill关联 → Benchmark → 校验 → 保存 |
| **模板继承** | 同系列机器人的变体（如Go2 → Go2 Pro） | 选择父模板 → 覆盖差异字段（如新增传感器、调整限制） → 校验 → 保存 |

#### 12.3.3 版本管理

- 每次修改生成新版本，支持版本对比（diff）和回滚
- e-URDF版本变更自动触发：Skill Registry能力重新校验、Task Memory缓存失效（因为Skill可用性可能改变）
- 版本号与Skill Registry版本联动，确保编译缓存key的一致性

**实现优先级：** Phase 1提供URDF导入+向导表单（Web界面）；Phase 2增加模板继承和版本对比。

---

## 十三、数据飞轮

### 13.1 飞轮循环

```
    执行任务
       │
       ▼
  采集执行数据 ────────→ 写入Practice / Memory
  (每个Skill的                    │
   参数/耗时/结果)                ▼
       │                  记忆驱动优化
       │                  - 推荐最优参数
       │                  - 缓存编译结果
       │                  - 空间基线更新
       │                         │
       ▼                         ▼
  离线分析引擎              下次执行更快更准
  (定时批处理)                    │
       │                         ▼
       ▼                  采集更好的数据
  优化产出                       │
  - SOP编译提示词优化             │
  - Skill参数调优                 │
  - 异常模式库更新                │
  - 失败恢复策略改进              │
       │                         │
       └─────────────────────────┘
              持续正反馈循环
```

### 13.2 飞轮效果量化

| 指标 | 首次执行 | 第10次 | 第100次 |
|------|---------|--------|---------|
| SOP编译耗时 | 3-5秒 | 0秒（缓存命中） | 0秒 |
| 导航路径效率 | 盲目规划 | 已知路径复用 | 最优路径 |
| 异常检测精度 | 固定阈值 | 设备基线对比 | 趋势预测 |
| 执行正确率 | 基线 | +15% | +30% |
| 任务总耗时 | 基线 | -20% | -40% |

---

## 十四、Know/How知识引擎

### 14.1 Know（离线知识炼化）

将外部知识编译为系统可用的工程先验：

```
输入源：
  ├── 论文、技术文档
  ├── 项目失败报告
  ├── Benchmark结果
  ├── 行业标准规范
  └── 代码库文档

    ↓ 知识编译

输出：
  ├── 结构化知识索引（向量化，可语义检索）
  ├── Skill改进建议
  ├── 参数推荐规则
  └── 异常模式库
```

### 14.2 How（在线经验注入）

执行卡住或风险升高时，实时提供精准提示：

```
触发条件：
  ├── Skill执行失败次数 > 阈值
  ├── 执行耗时 > 历史平均值的2倍
  ├── 传感器数据异常
  └── 连续恢复尝试失败

提示内容（最小、精准、带证据）：
  ├── "降低步行速度到0.5m/s（历史数据：此区域0.8m/s失败率12%）"
  ├── "检查热成像相机标定（上次标定距今超过30天）"
  ├── "回退到navigate_to_waypoint重新定位（当前位姿偏差0.3m）"
  └── "切换到备用路径（主路径近期发现新障碍物）"
```

---

## 十五、Auto/Darwin — 自进化引擎

Darwin是RobotClaw的Skill自动进化机制：基于积累的Practice数据，自动生成候选Skill改进版本，经过六道关卡评测后替代现有版本。目标是让Skill"越用越好"，无需人工逐一调优。

### 15.1 自进化总体流程

```
Practice数据积累
    │
    ▼
┌──────────────────────────────────────────────────────┐
│                  候选Skill生成                         │
│                                                       │
│  ┌────────────┐  ┌────────────┐  ┌────────────────┐ │
│  │ 参数变异   │  │ 策略调整   │  │ Know知识注入   │ │
│  │ 基于统计   │  │ 基于失败   │  │ 基于外部知识   │ │
│  │ 推导最优   │  │ 模式改进   │  │ 论文/规范启发  │ │
│  └─────┬──────┘  └─────┬──────┘  └───────┬────────┘ │
│        └───────────────┼─────────────────┘           │
│                        ▼                              │
│              候选Skill (Challenger)                   │
└────────────────────────┬─────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────┐
│              六道关卡评测流水线                         │
│                                                       │
│  Gate 1 ──→ Gate 2 ──→ Gate 3 ──→ Gate 4 ──→        │
│  多任务     扰动       多初始     回归               │
│  验证       测试       状态       测试               │
│                                                       │
│  ──→ Gate 5 ──→ Gate 6                               │
│      安全       性能                                  │
│      验证       基准                                  │
└────────────────────────┬─────────────────────────────┘
                         │
                    全部通过？
                    ╱        ╲
                  是           否
                  │            │
                  ▼            ▼
          Champion晋升     记录失败原因
          替代现有版本     供下轮改进参考
          推送ClawHub
```

### 15.2 候选Skill生成

候选Skill（Challenger）通过三种方式从现有Skill派生：

| 生成方式 | 输入数据 | 生成逻辑 | 示例 |
|---------|---------|---------|------|
| **参数变异** | Execution Memory中的历史执行数据 | 统计分析最优参数区间，生成参数组合候选 | navigate_to_waypoint的速度从0.8调整为0.85（统计显示0.85成功率更高） |
| **策略调整** | Practice中的失败记录 + 恢复策略效果 | 分析高频失败模式，调整恢复策略优先级或新增恢复路径 | 导航遇障碍时，从"重试3次→告警"改为"重试1次→绕行→告警"（历史数据显示绕行成功率80%） |
| **Know知识注入** | Know引擎中的外部知识 | 将论文/规范中的改进方法注入Skill参数或逻辑 | 基于热成像检测论文，将温度异常判定从"固定阈值"改为"基于环境温度的动态阈值" |

**生成约束：**
- 每轮最多生成3个Challenger（避免评测资源浪费）
- Challenger必须与原Skill接口兼容（相同输入/输出类型，可能增加可选参数）
- 参数变异幅度不超过原值的±30%（防止激进变异导致安全问题）

### 15.3 六道关卡评测

每个Challenger必须**顺序通过**全部六道关卡，任一关卡失败即淘汰：

```yaml
evaluation_pipeline:
  - gate: 1
    name: "多任务验证"
    description: "在多个不同SOP任务中执行Challenger，验证通用性"
    pass_criteria:
      min_tasks: 5                    # 至少5个不同任务
      success_rate: ">= 0.95"        # 成功率不低于现有Champion
      max_regression_tasks: 0         # 不允许任何任务出现回归

  - gate: 2
    name: "扰动测试"
    description: "在标准任务上注入环境扰动（光照变化、障碍物、传感器噪声），验证鲁棒性"
    pass_criteria:
      perturbation_types:
        - "lighting_change"           # 光照突变（±50%）
        - "obstacle_injection"        # 随机障碍物出现
        - "sensor_noise"              # 传感器增加高斯噪声
        - "communication_delay"       # 通信延迟注入（100-500ms）
      success_rate_under_perturbation: ">= 0.85"
      degradation_vs_normal: "<= 0.10"  # 扰动下性能下降不超过10%

  - gate: 3
    name: "多初始状态"
    description: "从不同初始状态（位置、姿态、电量）启动执行，验证状态无关性"
    pass_criteria:
      initial_states: 10              # 至少10种不同初始状态
      success_rate: ">= 0.90"

  - gate: 4
    name: "回归测试"
    description: "在历史成功案例上重放，确保Challenger不破坏已有正确行为"
    pass_criteria:
      regression_dataset: "golden_practices"  # 黄金Practice数据集
      regression_count: 0             # 零回归（所有历史成功案例必须仍然成功）

  - gate: 5
    name: "安全验证"
    description: "验证Challenger不违反安全约束（关节力矩限制、速度限制、碰撞检测）"
    pass_criteria:
      joint_limit_violations: 0       # 零关节超限
      collision_events: 0             # 零碰撞
      emergency_stop_triggers: 0      # 零紧急停止
      max_force_overshoot: "<= 5%"    # 力矩超调不超过5%

  - gate: 6
    name: "性能基准"
    description: "Challenger的性能不低于现有Champion（执行时间、资源消耗）"
    pass_criteria:
      execution_time: "<= champion * 1.05"   # 耗时不超过Champion的105%
      cpu_usage: "<= champion * 1.10"        # CPU使用不超过110%
      memory_usage: "<= champion * 1.10"     # 内存使用不超过110%
      improvement_in_any_metric: true        # 至少在一个指标上有改进
```

**评测环境：**
- Phase 1-2：基于Practice数据回放 + 参数化验证（无仿真）
- Phase 3+：接入仿真环境（Gazebo/Isaac Sim），支持扰动注入和多初始状态的自动化测试
- 真机验证：通过全部关卡的Challenger在真机上进行最终确认（人工监督下执行3+次）

### 15.4 Champion晋升与版本管理

```
Challenger通过全部6道关卡
    │
    ▼
┌──────────────────────────────────────┐
│  晋升流程                             │
│                                       │
│  1. 标记Challenger为新Champion        │
│  2. 旧Champion降级为Fallback          │
│  3. 更新Skill Registry版本            │
│  4. 推送至ClawHub（标记Champion徽章） │
│  5. 通知相关SOP重新校验兼容性         │
│  6. Task Memory中依赖此Skill的         │
│     缓存标记为"需重新验证"             │
└──────────────────────────────────────┘
```

**版本策略：**

| 版本状态 | 含义 | 行为 |
|---------|------|------|
| **Champion** | 当前最优版本 | 新任务默认使用 |
| **Fallback** | 上一代Champion | Champion异常时自动回退 |
| **Challenger** | 正在评测中 | 仅在评测环境执行 |
| **Retired** | 已淘汰 | 保留记录，不再执行 |

**安全回滚：** 新Champion上线后，系统持续监控其真实执行数据。如果连续5次执行中出现2次以上失败（且Fallback版本在相同条件下成功），自动触发回滚：
1. Champion降级为Retired（标记失败原因）
2. Fallback晋升为Champion
3. 告警通知运维人员

### 15.5 评测数据集管理

Darwin评测依赖三类数据集：

| 数据集 | 来源 | 更新频率 | 用途 |
|--------|------|---------|------|
| **Golden Practices** | 历史执行中人工标注的"标准正确"Practice | 每月人工审核新增 | Gate 4 回归测试 |
| **扰动参数集** | 预定义 + 真实环境中采集的边界条件 | 每季度更新 | Gate 2 扰动测试 |
| **安全约束集** | e-URDF物理约束 + 安全规范 | 随e-URDF更新 | Gate 5 安全验证 |

**数据集版本化：** 每次Darwin评测记录所用数据集版本，确保评测结果可复现。评测数据集本身纳入版本管理，支持回溯。

---

## 十六、安全与部署

### 16.1 安全架构

| 层面 | 措施 | 优先级 |
|------|------|--------|
| **传输安全** | TLS 1.3加密所有网络通信 | P1 |
| **存储安全** | AES-256加密敏感数据 | P1 |
| **模型安全** | 向外部LLM API发送前脱敏处理 | P1 |
| **审计安全** | 防篡改审计日志（哈希链） | P1 |
| **私有化** | 支持完全离线运行 + 本地模型 | P1 |
| **认证授权** | JWT令牌 + 角色权限控制 | P1 |

### 16.2 部署架构

```
Docker Compose部署:

┌──────────────────────────────────────────┐
│  docker-compose.yml                       │
│                                           │
│  ┌─────────┐ ┌──────────┐ ┌───────────┐ │
│  │ app     │ │ dashboard│ │ db        │ │
│  │ (Python │ │ (React   │ │ (PG/      │ │
│  │  后端)  │ │  前端)   │ │  SQLite)  │ │
│  └────┬────┘ └────┬─────┘ └─────┬─────┘ │
│       │           │             │         │
│  ┌────▼───────────▼─────────────▼───────┐│
│  │         Internal Network              ││
│  └───────────────────────────────────────┘│
│                                           │
│  ┌──────────┐                               │
│  │ redis    │                               │
│  │ (缓存)   │                               │
│  └──────────┘                               │
└──────────────────────────────────────────┘

外部通信:
  ├── gRPC: 机器人端控制+数据统一通道
  ├── HTTPS: Dashboard Web访问
  └── LLM API: 模型调用（可选，离线模式不需要）
```

### 16.3 机器人端部署

```
机器人端Runtime:
  ├── C++原生应用（单进程，静态编译，无Python依赖）
  ├── 通过RML（Robot Messaging Layer）与底层中间件交互
  │   └── Phase 1: ROS2 Adapter (rclcpp)  Phase 2+: DDS/自定义
  ├── 边缘推理运行时（HAL抽象，详见2.4.3节）
  │   ├── Phase 1: TensorRT (Jetson AGX Orin)
  │   ├── Phase 2: CANN/RKNN/SAIL (国产算力芯片)
  │   ├── 通用回退: ONNX Runtime CPU
  │   └── 目标检测、异常识别、语音ASR/TTS、轻量VLM
  ├── 本地SQLite缓存（SQLite C API）
  ├── gRPC客户端（grpc++，统一控制+数据通道）+ 本地SQLite离线队列
  ├── 条件求值引擎（Phase 1: JSON规则引擎, Phase 2+: 可选Lua嵌入）
  ├── 构建: CMake + colcon, 支持交叉编译(ARM/x86)
  └── 计算平台:
      ├── Phase 1: Jetson AGX Orin 64GB (275 TOPS, JetPack 6.x)
      ├── Phase 2: 昇腾310P/BM1684X (24-32 TOPS, 国产自主可控)
      └── Phase 3+: 昇腾910B/寒武纪MLU (128-320 TOPS, 高算力国产)
```

---

## 十七、技术栈总览

| 层面 | 技术选型 | 理由 |
|------|----------|------|
| **远端后端语言** | Python 3.11+ | LLM SDK生态、快速开发、SOP Compiler和数据分析主力 |
| **机器人端语言** | C++17 | 确定性延迟（无GIL/GC）、低资源占用、机器人行业标配，通过RML抽象通信层 |
| **Web框架** | FastAPI | 异步、高性能、自动OpenAPI文档 |
| **前端** | React + TypeScript | 组件化、生态丰富 |
| **DAG可视化** | React Flow | 专业DAG可视化组件 |
| **数据库** | SQLite(Phase 1) → PostgreSQL(Phase 3+) | 简单启动，按需升级 |
| **时序数据** | TimescaleDB (Phase 2+) | Practice和传感器数据 |
| **空间索引** | PostGIS (Phase 3+) | Spatial Memory地理查询 |
| **RPC** | gRPC | 强类型、低延迟、双向流，.proto自动生成C++/Python双端代码，统一控制+数据通道 |
| **容器** | Docker + Docker Compose | 远端统一部署、离线运行 |
| **机器人通信** | Robot Messaging Layer (RML) | 通信原语抽象，Phase 1: ROS2 Adapter (rclcpp)，Phase 2+: DDS/自定义 |
| **LLM** | GPT-4o / 通义千问 / 本地模型 | 灵活接入（远端调用） |
| **条件求值** | JSON规则引擎(Phase 1) / Lua(Phase 2+) | 替代Python eval，C++端安全求值 |
| **机器人端构建** | CMake + colcon | ROS2标准构建工具链，支持交叉编译 |
| **边缘推理框架** | TensorRT(Phase 1) / CANN·RKNN·SAIL(Phase 2) / ONNX Runtime(回退) | Phase 1 Jetson原生加速，Phase 2国产芯片适配，ONNX作为通用回退 |
| **边缘模型格式** | ONNX（交换格式）→ 各平台预编译格式(.engine/.om/.rknn/.bmodel) | ONNX统一训练导出，部署时编译为平台最优格式 |
| **机器人端计算平台** | Phase 1: Jetson AGX Orin / Phase 2: 昇腾·算能·瑞芯微 | Phase 1验证全链路，Phase 2国产自主可控 |

---

## 十八、里程碑与验证标准

### Phase 1 (Month 1-2): MVP验证

**运行平台：** NVIDIA Jetson AGX Orin 64GB

**交付物：**
1. SOP Compiler五步编译流水线
2. e-URDF标准定义和解析器（含送药机器人和巡检机器人示例）
3. Skill接口标准 + 11个核心Skill桩（送药：导航、语音、力感知等待、重量检查、目标检测、开门、条件等待、告警、日志；巡检：热成像、异常检测、回基站。共享：导航、告警、日志）
4. DAG Schema（顺序+条件分支）
5. 执行引擎骨架（C++ + TensorRT，状态机+基础顺序执行+异常恢复）
6. 边缘推理运行时（TensorRT后端 + 首批边缘模型，含门/床位/设备检测 + ASR + TTS）
7. Practice记录基础（数据采集+存储）
8. Task Memory（编译缓存）
9. 医院送药场景模板（**首要**）+ 电力巡检场景模板
10. Docker容器化部署（远端）+ C++原生部署（Jetson AGX Orin）

**验证标准：**
- **送药场景（首要）：** 可演示护士送药全链路 —— 输入送药SOP → 编译为Skill DAG → 机器人执行：药房取药 → 导航 → 开门 → 识别床位 → 语音交互 → 力感知交接 → 返回上报
- **巡检场景：** 可演示"改SOP不改代码"完整流程：输入自然语言SOP → 编译为Skill DAG → 修改SOP → 自动更新执行计划 → 第二次编译缓存命中(0ms)
- 边缘模型在Jetson AGX Orin上推理延迟达标（检测<20ms, ASR<100ms, TTS<50ms）

### Phase 2 (Month 3-4): 引擎完善

**运行平台：** Jetson AGX Orin（主） + 启动国产芯片适配（昇腾310P / BM1684X）

**交付物：**
- 完整执行引擎（并行/循环节点）
- 三层记忆系统
- Provider多模型接入（远端多Provider + 边缘多后端）
- Dashboard驾驶舱
- 人工接管功能
- HAL硬件抽象层实现（TensorRT + 首个国产后端）
- 核心边缘模型在国产芯片上的精度对齐验证

**验证标准：**
- 同一送药/巡检任务第10次执行比第1次快20%+
- 至少一款国产芯片上核心模型（检测+ASR）跑通，精度损失<2%

### Phase 3 (Month 5-6): 真机验证

**运行平台：** Jetson AGX Orin（Design Partner验证） + 国产芯片（并行验证）

**交付物：**
- 联合Design Partner真实场景端到端验证
- Forge机器人接入
- Know/How知识引擎
- Spatial Memory真机验证
- 国产芯片全模型适配完成
- 仿真环境集成（用于Skill验证、回归测试和Darwin评测，不替代真机验证）

**验证标准：**
- 送药场景真机端到端跑通（送药客户优先验证）
- 巡检场景真机端到端跑通
- 国产芯片端到端性能达到Jetson基线的90%+

### Phase 4 (Month 7-9): 产品化

**运行平台：** 国产芯片为主要交付平台，Jetson保留为开发/高端选项

**交付物：**
- ClawHub技能市场
- Darwin评测体系
- Swarm多机协作
- 场景模板库扩展

---

> **文档维护说明：** 本文档随项目迭代持续更新。每个Phase完成后应回顾并修正设计中与实际不符的部分。
