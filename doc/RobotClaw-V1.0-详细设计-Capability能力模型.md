# RobotClaw V1.0 详细设计 — Capability 能力模型

> **版本:** 1.0
> **日期:** 2026-06-25
> **状态:** DRAFT
> **上游依赖:** RobotClaw-V1.0-系统设计文档.md §三（机器人能力抽象层）、§六（Dashboard设计）、§十二（e-URDF设计）

---

## 一、文档范围与设计目标

### 1.1 文档范围

本文档是系统设计文档第三章"机器人能力抽象层"和第十二章"e-URDF设计"的详细设计展开，覆盖以下内容：

1. **原子能力完整目录** — 所有能力类型的层级结构、属性Schema、约束边界
2. **传感器/执行器属性字典** — 每种硬件设备的标准属性定义，作为能力声明的数据基础
3. **能力画像模板体系** — 预置模板、用户模板、模板继承与版本管理的完整数据模型
4. **Dashboard能力管理界面** — 能力定义、模板配置、能力浏览的交互设计与页面结构
5. **API接口定义** — 能力管理相关的REST API

### 1.2 设计目标

| 目标 | 量化指标 |
|------|---------|
| 新机器人接入时间 | 使用预置模板：< 30分钟；从零配置：< 2小时 |
| 能力声明完整性 | 覆盖MVP两个场景（送药+巡检）100%所需能力 |
| 模板复用率 | 同厂商同系列机器人接入，配置量减少 >= 70% |
| 能力匹配准确性 | SOP编译时能力匹配误判率 < 5% |

### 1.3 术语约定

| 术语 | 定义 |
|------|------|
| **原子能力（Atomic Capability）** | 机器人的单一、不可再分的物理能力，如"视觉"、"行走" |
| **能力类型（Capability Type）** | 原子能力的层级标识，格式为 `category.name`，如 `perception.vision` |
| **传感器（Sensor）** | 感知类能力绑定的硬件设备，如RGB相机、力矩传感器 |
| **执行器（Actuator）** | 运动/操作类能力绑定的硬件设备，如电机、推杆 |
| **能力画像（Capability Profile）** | 一台机器人所有原子能力的集合，描述"这台机器人能做什么" |
| **能力模板（Profile Template）** | 可复用的能力画像配置，分为预置模板和用户模板 |
| **e-URDF** | 扩展URDF，在标准URDF基础上增加能力声明、Benchmark等RobotClaw专用字段 |

---

## 二、原子能力完整目录

### 2.1 能力类型层级总览

```
capability_type_tree:
│
├── perception                        # 感知类
│   ├── perception.vision             # 视觉
│   ├── perception.hearing            # 听觉
│   ├── perception.force_sensing      # 力觉
│   ├── perception.localization       # 位置感知
│   └── perception.environment        # 环境感知
│
├── locomotion                        # 运动类
│   ├── locomotion.walking            # 腿式行走
│   ├── locomotion.wheeling           # 轮式移动
│   ├── locomotion.flying             # 飞行
│   └── locomotion.hybrid             # 混合移动
│
├── manipulation                      # 操作类
│   ├── manipulation.grasping         # 抓取
│   ├── manipulation.pushing          # 推拉
│   ├── manipulation.tool_use         # 工具使用
│   └── manipulation.fine_motor       # 精密操作
│
└── communication                     # 交互类
    ├── communication.speaking        # 语音输出
    ├── communication.display         # 视觉输出
    ├── communication.network         # 网络通信
    └── communication.hri             # 人机交互
```

每个叶子节点是一个**原子能力类型**。以下逐一定义其完整Schema。

---

### 2.2 感知类能力（Perception）

#### 2.2.1 perception.vision — 视觉

**能力描述：** 通过光学传感器获取环境视觉信息，包括可见光图像、深度图、热成像等。

**能力Schema：**

```yaml
capability:
  type: "perception.vision"
  available: true | false
  reason: "string"                     # available=false时填写原因

  # 传感器列表（一个能力可绑定多个传感器）
  sensors:
    - name: "string"                   # 传感器唯一标识，如 "front_rgb_camera"
      type: "enum"                     # 传感器类型，见下方传感器类型表
      # --- 公共属性 ---
      resolution: "string"            # 分辨率，格式 "WxH"，如 "1920x1080"
      fps: number                     # 帧率 (Hz)
      mount_frame: "string"           # 安装坐标系（对应URDF link名称）
      mount_position:                  # 安装位置（相对mount_frame）
        x: number                     # 米
        y: number
        z: number
      mount_orientation:               # 安装朝向（RPY，弧度）
        roll: number
        pitch: number
        yaw: number
      interface: "enum"               # 接口类型: usb3 | ethernet | mipi_csi | gmsl2
      # --- 类型特有属性，见传感器类型表 ---

  # 能力约束
  constraints:
    max_detection_range_m: number     # 最远检测距离（米）
    min_detection_range_m: number     # 最近检测距离（米）
    lighting_conditions: ["string"]   # 适用光照: indoor | outdoor_day | outdoor_night | outdoor_night_with_ir
    ip_rating: "string"              # 防护等级: IP54 | IP65 | IP67 | ...
    operating_temp_range: [min, max] # 工作温度范围（°C）

  # ROS2/RML通信接口
  topics:
    status: "string"                 # 状态topic，如 "/sensors/camera/status"
    data: "string"                   # 数据topic，如 "/sensors/camera/image_raw"
    info: "string"                   # 相机参数topic，如 "/sensors/camera/camera_info"

  depends_on: []                     # 依赖的其他能力（通常无）
```

**视觉传感器类型表（sensor.type 枚举）：**

| 传感器类型 | type值 | 特有属性 | 说明 |
|-----------|--------|---------|------|
| RGB相机 | `rgb_camera` | `fov_h`, `fov_v`, `pixel_size_um`, `lens_type`, `auto_focus`, `hdr` | 标准彩色相机 |
| 立体相机 | `stereo_camera` | `baseline_mm`, `fov_h`, `fov_v`, `depth_range_m`, `disparity_resolution` | 双目深度相机 |
| RGBD相机 | `rgbd_camera` | `depth_range_m`, `depth_resolution`, `depth_fps`, `fov_h`, `fov_v`, `depth_technology` | RGB+深度（结构光/ToF） |
| 热成像相机 | `thermal_camera` | `temperature_range`, `thermal_sensitivity_mk`, `netd_mk`, `palette` | 红外热成像 |
| PTZ云台相机 | `ptz_camera` | `optical_zoom`, `digital_zoom`, `pan_range_deg`, `tilt_range_deg`, `pan_speed_deg_s`, `tilt_speed_deg_s`, `preset_positions` | 可旋转变焦相机 |
| 鱼眼相机 | `fisheye_camera` | `fov_h`, `fov_v`, `distortion_model`, `distortion_coeffs` | 广角/全景 |
| 事件相机 | `event_camera` | `temporal_resolution_us`, `dynamic_range_db`, `pixel_bandwidth_khz` | 基于事件的视觉 |
| LiDAR | `lidar` | `range_m`, `channels`, `points_per_second`, `horizontal_fov_deg`, `vertical_fov_deg`, `angular_resolution_deg`, `return_mode` | 激光雷达 |

**各传感器类型特有属性详细定义：**

```yaml
# ━━━ RGB相机特有属性 ━━━
rgb_camera:
  fov_h: number                      # 水平视场角（度），如 69.4
  fov_v: number                      # 垂直视场角（度），如 42.5
  pixel_size_um: number              # 像素尺寸（微米），如 1.4
  lens_type: "enum"                  # 镜头类型: fixed | varifocal | motorized_zoom
  focal_length_mm: number            # 焦距（毫米），如 4.0
  aperture: "string"                 # 光圈，如 "f/2.0"
  auto_focus: boolean                # 是否支持自动对焦
  auto_exposure: boolean             # 是否支持自动曝光
  hdr: boolean                       # 是否支持HDR
  white_balance: "enum"              # auto | manual | preset
  image_format: ["string"]           # 输出格式: rgb8 | bgr8 | yuv422 | mjpeg | h264 | h265
  distortion_model: "string"         # 畸变模型: pinhole | equidistant | kannala_brandt
  distortion_coeffs: [number]        # 畸变系数数组

# ━━━ 立体相机特有属性 ━━━
stereo_camera:
  baseline_mm: number                # 基线距离（毫米），如 120
  fov_h: number
  fov_v: number
  depth_range_m: [min, max]          # 深度测量范围，如 [0.3, 20.0]
  disparity_resolution: number       # 视差分辨率（像素），如 1280
  stereo_algorithm: "enum"           # sgbm | bm | neural
  depth_accuracy_percent: number     # 深度精度百分比，如 2（表示2%误差）

# ━━━ RGBD相机特有属性 ━━━
rgbd_camera:
  depth_range_m: [min, max]          # 深度范围，如 [0.1, 10.0]
  depth_resolution: "string"         # 深度图分辨率，如 "640x480"
  depth_fps: number                  # 深度图帧率
  depth_technology: "enum"           # 深度技术: structured_light | tof | active_stereo
  depth_accuracy_mm: number          # 深度精度（毫米）
  fov_h: number
  fov_v: number
  rgb_depth_alignment: boolean       # RGB与深度图是否已对齐

# ━━━ 热成像相机特有属性 ━━━
thermal_camera:
  temperature_range: [min, max]      # 测温范围（°C），如 [-20, 350]
  thermal_sensitivity_mk: number     # 热灵敏度（mK），如 50（表示0.05°C）
  netd_mk: number                   # 噪声等效温差（mK），如 40
  accuracy_deg_c: number             # 测温精度（°C），如 2
  emissivity_adjustable: boolean     # 发射率是否可调
  palette: ["string"]                # 可用伪彩色方案: iron | rainbow | grayscale | ...
  spot_metering: boolean             # 是否支持点测温
  area_metering: boolean             # 是否支持区域测温
  spectral_range_um: [min, max]      # 光谱范围（微米），如 [8, 14]

# ━━━ PTZ云台相机特有属性 ━━━
ptz_camera:
  optical_zoom: number               # 光学变焦倍数，如 30
  digital_zoom: number               # 数字变焦倍数，如 16
  pan_range_deg: [min, max]          # 水平旋转范围（度），如 [-180, 180]
  tilt_range_deg: [min, max]         # 垂直旋转范围（度），如 [-90, 30]
  pan_speed_deg_s: number            # 水平旋转速度（度/秒），如 120
  tilt_speed_deg_s: number           # 垂直旋转速度（度/秒），如 60
  preset_positions: number           # 预置位数量，如 256
  auto_tracking: boolean             # 是否支持自动跟踪
  defog: boolean                     # 是否支持透雾
  stabilization: boolean             # 是否支持光学防抖

# ━━━ LiDAR特有属性 ━━━
lidar:
  range_m: number                    # 最大测距（米），如 120
  min_range_m: number                # 最小测距（米），如 0.1
  channels: number                   # 线束数，如 16 | 32 | 64 | 128
  points_per_second: number          # 每秒点数，如 300000
  horizontal_fov_deg: number         # 水平视场角（度），如 360
  vertical_fov_deg: [min, max]       # 垂直视场角（度），如 [-25, 15]
  angular_resolution_deg: number     # 角分辨率（度），如 0.2
  return_mode: ["string"]            # 回波模式: single | dual | triple
  wavelength_nm: number              # 激光波长（纳米），如 905 | 1550
  accuracy_cm: number                # 测量精度（厘米），如 2
  scan_rate_hz: number               # 扫描频率（Hz），如 10 | 20
```

---

#### 2.2.2 perception.hearing — 听觉

**能力描述：** 通过麦克风采集音频信号，支持语音识别、关键词检测、声源定位、声学异常检测。

**能力Schema：**

```yaml
capability:
  type: "perception.hearing"
  available: true | false
  reason: "string"

  sensors:
    - name: "string"                   # 如 "mic_array"
      type: "enum"                     # 传感器类型

      # --- 公共属性 ---
      mount_frame: "string"
      interface: "enum"               # usb | i2s | analog | ethernet

      # --- 类型特有属性 ---
      # （见下方传感器类型表）

  constraints:
    speech_recognition_languages: ["string"]  # 支持的ASR语言: zh-CN | en-US | ja-JP | ...
    keyword_detection: boolean                # 支持唤醒词检测
    noise_cancellation: boolean               # 支持噪声抑制
    effective_range_m: number                 # 有效拾音距离（米）
    max_ambient_noise_db: number              # 可工作的最大环境噪音（dB）
    operating_temp_range: [min, max]

  topics:
    status: "string"                 # 如 "/audio/mic/status"
    data: "string"                   # 如 "/audio/mic/raw"
    asr_result: "string"             # 如 "/audio/asr/result"（可选，若边缘ASR可用）

  depends_on: []
```

**听觉传感器类型表：**

| 传感器类型 | type值 | 特有属性 | 说明 |
|-----------|--------|---------|------|
| 单麦克风 | `microphone` | `channels`, `sample_rate`, `bit_depth`, `sensitivity_dbv`, `snr_db`, `frequency_range_hz` | 单通道音频采集 |
| 麦克风阵列 | `microphone_array` | `channels`, `sample_rate`, `bit_depth`, `array_geometry`, `beam_forming`, `doa_accuracy_deg`, `sensitivity_dbv`, `snr_db` | 多通道阵列，支持波束成形和声源定位 |
| 超声麦克风 | `ultrasonic_mic` | `frequency_range_hz`, `sensitivity_dbv`, `sample_rate` | 超声波段，用于局放检测等工业场景 |

```yaml
# ━━━ 麦克风特有属性 ━━━
microphone:
  channels: number                   # 通道数，如 1
  sample_rate: number                # 采样率（Hz），如 16000 | 44100 | 48000
  bit_depth: number                  # 位深，如 16 | 24
  sensitivity_dbv: number            # 灵敏度（dBV），如 -26
  snr_db: number                     # 信噪比（dB），如 65
  frequency_range_hz: [min, max]     # 频率响应范围（Hz），如 [20, 20000]
  aec: boolean                       # 是否支持回声消除（Acoustic Echo Cancellation）

# ━━━ 麦克风阵列特有属性 ━━━
microphone_array:
  channels: number                   # 通道数（麦克风个数），如 4 | 6 | 8
  sample_rate: number
  bit_depth: number
  array_geometry: "enum"             # 阵列几何: linear | circular | planar | spherical
  array_diameter_mm: number          # 阵列直径/间距（毫米），如 70
  beam_forming: boolean              # 是否支持波束成形
  beam_count: number                 # 同时可形成的波束数
  doa_accuracy_deg: number           # 声源定位精度（度），如 5
  doa_range_deg: number              # 声源定位范围（度），如 360
  sensitivity_dbv: number
  snr_db: number
  wake_word_engine: "enum"           # 唤醒词引擎: built_in | external | none
  aec: boolean

# ━━━ 超声麦克风特有属性 ━━━
ultrasonic_mic:
  frequency_range_hz: [min, max]     # 如 [20000, 200000]（超声波段）
  sensitivity_dbv: number
  sample_rate: number                # 高采样率，如 500000
  application: ["string"]            # 应用场景: partial_discharge | leak_detection | ...
```

---

#### 2.2.3 perception.force_sensing — 力觉

**能力描述：** 通过力/力矩传感器或触觉传感器感知接触力，用于物品称重、碰撞检测、力控操作等。

**能力Schema：**

```yaml
capability:
  type: "perception.force_sensing"
  available: true | false
  reason: "string"

  sensors:
    - name: "string"                   # 如 "tray_load_cell"
      type: "enum"

      mount_frame: "string"
      location: "string"              # 安装位置语义描述，如 "front_left_foot" | "medicine_box"
      interface: "enum"               # analog | spi | i2c | ethernet | usb

  constraints:
    sensitivity_n: number             # 最小可检测力（N），如 0.1
    collision_detection_threshold_n: number  # 碰撞检测阈值（N）
    overload_protection_n: number     # 过载保护值（N）
    operating_temp_range: [min, max]

  topics:
    status: "string"
    data: "string"

  depends_on: []
```

**力觉传感器类型表：**

| 传感器类型 | type值 | 特有属性 | 说明 |
|-----------|--------|---------|------|
| 六维力矩传感器 | `force_torque_6dof` | `range_force`, `range_torque`, `resolution_force`, `resolution_torque`, `frequency`, `axes` | 六自由度力矩测量 |
| 称重传感器 | `load_cell` | `range_kg`, `resolution_g`, `accuracy_class`, `tare_function`, `overload_percent` | 重量测量 |
| 单轴力传感器 | `force_sensor_1d` | `range_n`, `resolution_n`, `frequency`, `direction` | 单方向力测量 |
| 触觉传感器阵列 | `tactile_array` | `taxels`, `spatial_resolution_mm`, `range_kpa`, `frequency`, `area_mm` | 面分布式触觉 |
| 碰撞检测传感器 | `collision_sensor` | `threshold_n`, `response_time_ms`, `coverage_area` | 二值碰撞检测 |

```yaml
# ━━━ 六维力矩传感器属性 ━━━
force_torque_6dof:
  range_force: [min, max]            # 力测量范围（N），如 [0, 200]
  range_torque: [min, max]           # 力矩测量范围（Nm），如 [0, 10]
  resolution_force: number           # 力分辨率（N），如 0.05
  resolution_torque: number          # 力矩分辨率（Nm），如 0.001
  frequency: number                  # 采样频率（Hz），如 500 | 1000
  axes: ["string"]                   # 测量轴: ["fx","fy","fz","tx","ty","tz"]
  overload_force_n: number           # 力过载保护（N）
  overload_torque_nm: number         # 力矩过载保护（Nm）
  temperature_compensation: boolean  # 温度补偿
  calibration_date: "string"         # 最近校准日期

# ━━━ 称重传感器属性 ━━━
load_cell:
  range_kg: number                   # 量程（千克），如 10
  resolution_g: number               # 分辨率（克），如 1
  accuracy_class: "string"           # 精度等级，如 "C3"（OIML标准）
  tare_function: boolean             # 是否支持去皮
  overload_percent: number           # 过载容许百分比，如 150（表示150%量程）
  frequency: number                  # 采样频率（Hz）
  zero_drift_per_hour: number        # 零点漂移（克/小时）
  temperature_effect: number         # 温度影响系数（%/°C）

# ━━━ 触觉传感器阵列属性 ━━━
tactile_array:
  taxels: number                     # 触觉单元数量，如 256
  spatial_resolution_mm: number      # 空间分辨率（毫米），如 3
  range_kpa: [min, max]              # 压力范围（kPa），如 [0, 100]
  frequency: number                  # 采样频率（Hz），如 100
  area_mm: [width, height]           # 传感面积（毫米），如 [40, 40]
```

---

#### 2.2.4 perception.localization — 位置感知

**能力描述：** 获取机器人在环境中的位置和姿态，支持定位、建图、路径规划。

**能力Schema：**

```yaml
capability:
  type: "perception.localization"
  available: true | false
  reason: "string"

  # 定位方法列表（通常多方法融合）
  method: ["enum"]                    # visual_slam | lidar_slam | imu | gps | rtk_gps | uwb | wheel_odometry | ...

  sensors:
    - name: "string"
      type: "enum"
      mount_frame: "string"
      interface: "enum"

  constraints:
    position_accuracy_m: number       # 定位精度（米），如 0.05
    heading_accuracy_deg: number      # 航向精度（度），如 1
    update_rate_hz: number            # 定位更新频率（Hz），如 50
    max_speed_m_s: number             # 最大可跟踪速度（m/s）
    environment: ["enum"]             # 适用环境: indoor | outdoor | mixed
    initialization_time_s: number     # 定位初始化时间（秒）
    relocalization: boolean           # 是否支持重定位（丢失后恢复）

  topics:
    status: "string"
    pose: "string"                   # 位姿topic，如 "/localization/pose"
    map: "string"                    # 地图topic（可选），如 "/map"

  depends_on: []                     # 可能依赖 perception.vision（视觉SLAM场景）
```

**定位传感器类型表：**

| 传感器类型 | type值 | 特有属性 | 说明 |
|-----------|--------|---------|------|
| IMU | `imu` | `accel_range_g`, `gyro_range_dps`, `accel_resolution`, `gyro_resolution`, `frequency`, `drift_deg_per_hour` | 惯性测量单元 |
| GPS接收器 | `gps` | `constellation`, `channels`, `cold_start_s`, `horizontal_accuracy_m` | 标准GNSS定位 |
| RTK GPS | `rtk_gps` | `constellation`, `rtk_accuracy_cm`, `correction_source`, `convergence_time_s` | 厘米级高精度定位 |
| UWB定位标签 | `uwb_tag` | `range_m`, `accuracy_cm`, `update_rate_hz`, `anchor_count` | 超宽带室内定位 |
| 轮式编码器 | `wheel_encoder` | `resolution_ppr`, `wheel_diameter_mm`, `gear_ratio` | 轮式里程计 |

```yaml
# ━━━ IMU属性 ━━━
imu:
  accel_range_g: number              # 加速度量程（g），如 16
  gyro_range_dps: number             # 角速度量程（°/s），如 2000
  accel_resolution: number           # 加速度分辨率（mg），如 0.5
  gyro_resolution: number            # 角速度分辨率（°/s），如 0.01
  frequency: number                  # 输出频率（Hz），如 200 | 400 | 1000
  drift_deg_per_hour: number         # 陀螺零偏漂移（°/h），如 5
  axes: number                       # 轴数: 6（6轴）| 9（含磁力计）
  temperature_compensation: boolean

# ━━━ RTK GPS属性 ━━━
rtk_gps:
  constellation: ["string"]          # 卫星系统: gps | glonass | galileo | beidou
  rtk_accuracy_cm: number            # RTK定位精度（厘米），如 2
  correction_source: "enum"          # 差分源: ntrip | radio | satellite_l_band
  convergence_time_s: number         # 收敛时间（秒），如 30
  channels: number                   # 通道数，如 184
  update_rate_hz: number             # 更新频率（Hz），如 10 | 20
  heading_output: boolean            # 是否支持双天线测向

# ━━━ UWB定位标签属性 ━━━
uwb_tag:
  range_m: number                    # 最大测距距离（米），如 50
  accuracy_cm: number                # 定位精度（厘米），如 10
  update_rate_hz: number             # 更新频率（Hz），如 50
  anchor_count: number               # 需要的基站数量（最少），如 4
  tdoa_support: boolean              # 是否支持TDOA模式
  two_way_ranging: boolean           # 是否支持双向测距
```

---

#### 2.2.5 perception.environment — 环境感知

**能力描述：** 通过专用传感器感知温度、湿度、气体浓度等环境参数，主要用于工业巡检场景。

**能力Schema：**

```yaml
capability:
  type: "perception.environment"
  available: true | false
  reason: "string"

  sensors:
    - name: "string"
      type: "enum"
      mount_frame: "string"
      interface: "enum"

  constraints:
    operating_temp_range: [min, max]
    ip_rating: "string"

  topics:
    status: "string"
    data: "string"

  depends_on: []
```

**环境传感器类型表：**

| 传感器类型 | type值 | 特有属性 | 说明 |
|-----------|--------|---------|------|
| 温湿度传感器 | `temp_humidity_sensor` | `temp_range`, `temp_accuracy`, `humidity_range`, `humidity_accuracy`, `frequency` | 温度+湿度 |
| 气体传感器 | `gas_sensor` | `gases`, `range_ppm`, `accuracy_ppm`, `response_time_s`, `alarm_thresholds` | 气体浓度检测 |
| 气压传感器 | `barometer` | `range_hpa`, `accuracy_hpa`, `frequency` | 大气压力 |
| 照度传感器 | `light_sensor` | `range_lux`, `accuracy_percent`, `spectral_response` | 环境光强度 |
| 辐射检测器 | `radiation_detector` | `type`, `range_usv_h`, `alarm_threshold`, `energy_range_kev` | 辐射剂量 |
| 粉尘传感器 | `particle_sensor` | `particle_sizes_um`, `range_ug_m3`, `accuracy_percent` | PM2.5/PM10等 |

```yaml
# ━━━ 气体传感器属性 ━━━
gas_sensor:
  gases: ["string"]                  # 可检测气体种类: SF6 | H2S | CO | CH4 | O2 | CO2 | NH3 | ...
  detection_principle: "enum"        # 检测原理: electrochemical | catalytic | infrared | pid | semiconductor
  range_ppm:                         # 每种气体的检测范围
    SF6: [min, max]                  # 如 [0, 1000]
    H2S: [min, max]
  accuracy_ppm:                      # 每种气体的检测精度
    SF6: number
    H2S: number
  response_time_s: number            # 响应时间（秒），如 30（T90）
  recovery_time_s: number            # 恢复时间（秒）
  alarm_thresholds:                  # 告警阈值
    SF6: {warning: number, danger: number}
    H2S: {warning: number, danger: number}
  calibration_interval_days: number  # 校准周期（天），如 180
  sensor_life_years: number          # 传感器寿命（年），如 2

# ━━━ 温湿度传感器属性 ━━━
temp_humidity_sensor:
  temp_range: [min, max]             # 温度范围（°C），如 [-40, 85]
  temp_accuracy: number              # 温度精度（°C），如 0.3
  temp_resolution: number            # 温度分辨率（°C），如 0.01
  humidity_range: [min, max]         # 湿度范围（%RH），如 [0, 100]
  humidity_accuracy: number          # 湿度精度（%RH），如 2
  frequency: number                  # 采样频率（Hz），如 1
```

---

### 2.3 运动类能力（Locomotion）

#### 2.3.1 locomotion.walking — 腿式行走

**能力Schema：**

```yaml
capability:
  type: "locomotion.walking"
  available: true | false
  reason: "string"

  # 执行器组
  actuators:
    - group: "string"                 # 腿组名称，如 "front_left_leg"
      joints: ["string"]             # 关节名称列表，如 ["hip", "thigh", "calf"]
      max_torque_nm: [number]         # 每关节最大力矩（Nm），如 [40, 40, 40]
      joint_range_deg: [[min, max]]  # 每关节角度范围（度）

  # 步态
  gaits:
    - name: "string"                  # 步态名称: trot | crawl | bound | pace | gallop | biped_walk | biped_run
      speed_range_m_s: [min, max]    # 速度范围（m/s），如 [0.3, 1.2]
      stability: "enum"              # 稳定性: low | medium | high | very_high
      use_case: "string"             # 推荐场景，如 "rough_terrain" | "normal" | "high_speed"
      energy_consumption: "enum"     # 能耗等级: low | medium | high

  constraints:
    max_speed_m_s: number            # 最大移动速度（m/s）
    max_slope_deg: number            # 最大爬坡角度（度）
    max_step_height_m: number        # 最大跨越台阶高度（米）
    min_turn_radius_m: number        # 最小转弯半径（米），0=原地转
    terrain_types: ["string"]        # 适用地形: flat | grass | gravel | stairs | sand | snow
    payload_kg: number               # 载荷能力（千克）
    battery_impact_ah_per_km: number # 电池消耗率（Ah/km）
    leg_count: number                # 腿数: 2 | 4 | 6
    dof_per_leg: number              # 每腿自由度数

  topics:
    status: "string"                 # 如 "/locomotion/status"
    command: "string"                # 如 "/cmd_vel"
    gait_state: "string"             # 如 "/locomotion/gait_state"
    foot_contact: "string"           # 如 "/locomotion/foot_contact"

  depends_on: ["perception.localization"]  # 行走通常需要定位
```

#### 2.3.2 locomotion.wheeling — 轮式移动

**能力Schema：**

```yaml
capability:
  type: "locomotion.wheeling"
  available: true | false
  reason: "string"

  actuators:
    - group: "string"                 # 驱动组名称，如 "diff_drive"
      type: "enum"                   # 驱动类型: differential | omnidirectional | ackermann | skid_steer | mecanum
      motors: number                 # 电机数量
      wheel_diameter_mm: number      # 轮径（毫米）
      max_rpm: number                # 最大转速
      encoder_ppr: number            # 编码器脉冲数

  constraints:
    max_speed_m_s: number            # 最大速度（m/s）
    max_acceleration_m_s2: number    # 最大加速度（m/s²）
    min_turn_radius_m: number        # 最小转弯半径（米），0=原地旋转
    max_slope_deg: number            # 最大爬坡角度（度）
    terrain_types: ["string"]        # 适用地形: flat | indoor | outdoor_paved | outdoor_unpaved
    payload_kg: number
    ground_clearance_mm: number      # 离地间隙（毫米）
    battery_impact_ah_per_km: number

  topics:
    status: "string"
    command: "string"                # 如 "/cmd_vel"
    odometry: "string"              # 如 "/odom"

  depends_on: ["perception.localization"]
```

#### 2.3.3 locomotion.flying — 飞行

**能力Schema：**

```yaml
capability:
  type: "locomotion.flying"
  available: true | false
  reason: "string"

  actuators:
    - group: "string"                 # 如 "quadrotor_motors"
      type: "enum"                   # 飞行器类型: multirotor | fixed_wing | vtol
      motor_count: number            # 电机/旋翼数
      max_thrust_n: number           # 单电机最大推力（N）
      prop_diameter_inch: number     # 螺旋桨直径（英寸）

  constraints:
    max_speed_m_s: number
    max_altitude_m: number           # 最大飞行高度（米）
    max_wind_speed_m_s: number       # 最大抗风速度（m/s）
    max_flight_time_min: number      # 最大续航时间（分钟）
    hover_accuracy_m: number         # 悬停精度（米）
    payload_kg: number
    geofence_support: boolean        # 是否支持电子围栏

  topics:
    status: "string"
    command: "string"
    altitude: "string"

  depends_on: ["perception.localization"]
```

#### 2.3.4 locomotion.hybrid — 混合移动

**能力Schema：**

```yaml
capability:
  type: "locomotion.hybrid"
  available: true | false
  reason: "string"

  modes: ["enum"]                    # 可用移动模式: wheeling | walking | tracked | ...
  mode_switch_time_s: number         # 模式切换时间（秒）

  # 每种模式的参数引用对应的locomotion.*能力
  mode_capabilities:
    wheeling: {ref: "locomotion.wheeling"}
    walking: {ref: "locomotion.walking"}

  constraints:
    # 综合约束
    payload_kg: number
    terrain_types: ["string"]

  topics:
    status: "string"
    command: "string"
    current_mode: "string"

  depends_on: ["perception.localization"]
```

---

### 2.4 操作类能力（Manipulation）

#### 2.4.1 manipulation.grasping — 抓取

**能力Schema：**

```yaml
capability:
  type: "manipulation.grasping"
  available: true | false
  reason: "string"

  actuators:
    - name: "string"                   # 如 "left_arm" 或 "gripper"
      type: "enum"                     # 执行器类型
      # --- 类型特有属性 ---

  constraints:
    reach_m: number                  # 最大工作半径（米）
    max_grip_force_n: number         # 最大夹持力（N）
    max_payload_kg: number           # 最大抓取载荷（千克）
    grip_accuracy_mm: number         # 抓取定位精度（毫米）
    repeat_accuracy_mm: number       # 重复定位精度（毫米）
    grip_open_range_mm: [min, max]  # 夹爪开合范围（毫米）

  topics:
    status: "string"
    command: "string"
    joint_states: "string"

  depends_on: ["perception.vision"]  # 抓取通常需要视觉引导
```

**抓取执行器类型表：**

| 执行器类型 | type值 | 特有属性 | 说明 |
|-----------|--------|---------|------|
| 平行夹爪 | `parallel_jaw` | `jaw_width_mm`, `stroke_mm`, `max_force_n`, `speed_mm_s`, `finger_count` | 两指平行开合 |
| 吸盘 | `suction_cup` | `suction_force_n`, `cup_diameter_mm`, `vacuum_kpa`, `cup_count`, `surface_types` | 真空吸附 |
| 灵巧手 | `dexterous_hand` | `finger_count`, `dof_per_finger`, `max_grip_force_n`, `tactile_sensors`, `tendon_driven` | 多指灵巧操作 |
| 软体夹爪 | `soft_gripper` | `max_object_diameter_mm`, `grip_force_n`, `adaptability`, `material` | 柔性自适应 |
| 机械臂 | `robot_arm` | `dof`, `reach_m`, `max_payload_kg`, `repeat_accuracy_mm`, `joint_ranges`, `max_joint_speeds` | 多关节机械臂 |

```yaml
# ━━━ 机械臂属性 ━━━
robot_arm:
  dof: number                        # 自由度，如 6 | 7
  reach_m: number                    # 最大工作半径（米），如 0.85
  max_payload_kg: number             # 末端最大负载（千克），如 5
  repeat_accuracy_mm: number         # 重复定位精度（毫米），如 0.05
  joint_ranges:                      # 各关节角度范围
    - joint: "string"
      range_deg: [min, max]
      max_speed_deg_s: number
      max_torque_nm: number
  max_tcp_speed_m_s: number          # 末端最大线速度（m/s）
  max_tcp_acceleration_m_s2: number  # 末端最大加速度（m/s²）
  collision_detection: boolean       # 是否支持碰撞检测
  force_control: boolean             # 是否支持力控
  gravity_compensation: boolean      # 是否支持重力补偿

# ━━━ 平行夹爪属性 ━━━
parallel_jaw:
  jaw_width_mm: number               # 夹爪宽度（毫米），如 85
  stroke_mm: number                  # 行程（毫米），如 80
  max_force_n: number                # 最大夹持力（N），如 140
  speed_mm_s: number                 # 开合速度（mm/s），如 100
  finger_count: number               # 指数: 2 | 3
  finger_material: "string"          # 指尖材料: rubber | metal | silicone
  position_feedback: boolean         # 是否有位置反馈
  force_feedback: boolean            # 是否有力反馈

# ━━━ 灵巧手属性 ━━━
dexterous_hand:
  finger_count: number               # 手指数，如 5
  dof_per_finger: number             # 每指自由度，如 3 | 4
  dof_total: number                  # 总自由度，如 16
  max_grip_force_n: number           # 最大握力（N），如 30
  tactile_sensors: boolean           # 指尖是否有触觉传感器
  tendon_driven: boolean             # 是否腱驱动
  object_size_range_mm: [min, max]  # 可操作物体尺寸范围（毫米）
```

#### 2.4.2 manipulation.pushing — 推拉

**能力Schema：**

```yaml
capability:
  type: "manipulation.pushing"
  available: true | false
  reason: "string"

  actuators:
    - name: "string"
      type: "enum"                   # linear_actuator | pneumatic_cylinder | arm_end_effector

      # ━━━ 线性推杆属性 ━━━
      stroke_mm: number              # 行程（毫米），如 300
      max_force_n: number            # 最大推力（N），如 50
      speed_mm_s: number             # 速度（mm/s），如 100
      mount_frame: "string"          # 安装坐标系

  constraints:
    max_safe_force_n: number         # 安全力上限（N）
    force_control_hz: number         # 力控频率（Hz），如 100
    collision_detection_n: number    # 碰撞检测阈值（N）
    position_accuracy_mm: number     # 定位精度（毫米）

  topics:
    status: "string"
    command: "string"
    force: "string"

  depends_on: ["perception.vision", "perception.force_sensing"]
```

#### 2.4.3 manipulation.tool_use — 工具使用

**能力Schema：**

```yaml
capability:
  type: "manipulation.tool_use"
  available: true | false
  reason: "string"

  tools:
    - name: "string"                 # 工具名称，如 "screwdriver" | "wrench"
      tool_type: "enum"             # 工具类型: rotary | linear | pneumatic | custom
      max_torque_nm: number          # 最大力矩（旋转类）
      max_force_n: number            # 最大力（线性类）
      tool_change: boolean           # 是否支持自动换刀

  constraints:
    compatible_tools: ["string"]     # 兼容工具列表
    tool_change_time_s: number       # 换刀时间（秒）

  topics:
    status: "string"
    command: "string"

  depends_on: ["manipulation.grasping", "perception.vision"]
```

#### 2.4.4 manipulation.fine_motor — 精密操作

**能力Schema：**

```yaml
capability:
  type: "manipulation.fine_motor"
  available: true | false
  reason: "string"

  actuators:
    - name: "string"
      type: "enum"                   # precision_stage | dexterous_hand | micro_manipulator
      dof: number
      position_accuracy_um: number   # 定位精度（微米）
      force_resolution_mn: number    # 力分辨率（毫牛）

  constraints:
    workspace_mm: [x, y, z]          # 工作空间范围（毫米）
    max_payload_g: number            # 最大负载（克）
    vibration_isolation: boolean     # 是否有隔振

  topics:
    status: "string"
    command: "string"

  depends_on: ["perception.vision"]
```

---

### 2.5 交互类能力（Communication）

#### 2.5.1 communication.speaking — 语音输出

**能力Schema：**

```yaml
capability:
  type: "communication.speaking"
  available: true | false
  reason: "string"

  sensors:                           # 此处复用sensors字段描述输出设备
    - name: "string"                 # 如 "speaker"
      type: "enum"                   # speaker | speaker_array | bone_conduction

      channels: number               # 声道数: 1（单声道）| 2（立体声）
      sample_rate: number            # 采样率（Hz），如 44100
      max_power_w: number            # 最大功率（瓦），如 5
      max_volume_db: number          # 最大音量（dB SPL），如 85
      frequency_range_hz: [min, max] # 频率响应（Hz），如 [200, 15000]
      mount_frame: "string"

  constraints:
    tts_languages: ["string"]        # TTS支持语言: zh-CN | en-US | ja-JP | ...
    tts_engines: ["string"]          # TTS引擎: edge_tts | cloud_tts | piper | ...
    max_ambient_noise_db: number     # 在该噪音水平下仍可清晰听到
    latency_ms: number               # TTS合成延迟（毫秒），如 200
    voice_count: number              # 可用音色数
    ssml_support: boolean            # 是否支持SSML标记语言
    audio_format: ["string"]         # 支持格式: wav | mp3 | ogg | pcm

  topics:
    status: "string"
    command: "string"                # 如 "/audio/speaker/play"
    tts: "string"                    # 如 "/audio/tts/speak"

  depends_on: []
```

#### 2.5.2 communication.display — 视觉输出

**能力Schema：**

```yaml
capability:
  type: "communication.display"
  available: true | false
  reason: "string"

  sensors:
    - name: "string"
      type: "enum"                   # screen | led_matrix | led_indicator | projector

      # ━━━ 屏幕属性 ━━━
      resolution: "string"          # 分辨率，如 "800x480"
      size_inch: number             # 尺寸（英寸），如 7
      touch: boolean                # 是否触摸屏
      brightness_nit: number        # 亮度（尼特）

      # ━━━ LED指示灯属性 ━━━
      led_count: number             # LED数量
      colors: ["string"]            # 可显示颜色
      programmable: boolean         # 是否可编程

  constraints:
    refresh_rate_hz: number
    outdoor_readable: boolean

  topics:
    status: "string"
    command: "string"

  depends_on: []
```

#### 2.5.3 communication.network — 网络通信

**能力Schema：**

```yaml
capability:
  type: "communication.network"
  available: true | false
  reason: "string"

  interfaces:
    - name: "string"                 # 接口名称，如 "wifi_module"
      type: "enum"                   # wifi | wifi_6 | ethernet | 4g | 5g | lora | bluetooth | zigbee

      # ━━━ WiFi属性 ━━━
      standard: "string"            # 802.11ac | 802.11ax | ...
      frequency_ghz: [number]       # 频段: [2.4, 5.0]
      max_bandwidth_mbps: number    # 最大带宽（Mbps）
      antenna_count: number         # 天线数（MIMO）

      # ━━━ 蜂窝网络属性 ━━━
      bands: ["string"]             # 频段列表
      sim_slots: number             # SIM卡槽数
      max_bandwidth_mbps: number

      # ━━━ 以太网属性 ━━━
      speed_mbps: number            # 速率: 100 | 1000 | 10000
      poe: boolean                  # 是否支持PoE供电

  constraints:
    min_bandwidth_kbps: number       # 最低可用带宽（kbps），低于此值标记DEGRADED
    max_latency_ms: number           # 最大延迟容忍（毫秒）
    offline_capable: boolean         # 是否支持离线运行

  topics:
    status: "string"                 # 如 "/network/status"

  depends_on: []
```

#### 2.5.4 communication.hri — 人机交互

**能力Schema：**

```yaml
capability:
  type: "communication.hri"
  available: true | false
  reason: "string"

  modalities:
    - type: "enum"                   # gesture_recognition | facial_expression | body_language | emotion_display

      # ━━━ 手势识别属性 ━━━
      gesture_count: number          # 可识别手势数量
      recognition_range_m: number    # 识别距离

      # ━━━ 表情显示属性 ━━━
      expression_count: number       # 可显示表情数量
      display_type: "enum"           # screen | led_matrix | mechanical

  constraints:
    recognition_accuracy: number     # 识别准确率
    response_time_ms: number

  topics:
    status: "string"
    data: "string"

  depends_on: ["perception.vision"]  # 通常需要视觉来做手势/人脸识别
```

---

### 2.6 能力类型注册表（Capability Type Registry）

所有能力类型在系统中注册为元数据，供Dashboard渲染表单、SOP Compiler查询、Skill匹配使用：

```yaml
capability_type_registry:
  # 每个能力类型的元信息
  types:
    - type_id: "perception.vision"
      category: "perception"
      display_name: "视觉"
      display_name_en: "Vision"
      icon: "eye"
      description: "通过光学传感器获取环境视觉信息"
      sensor_types: ["rgb_camera", "stereo_camera", "rgbd_camera", "thermal_camera",
                      "ptz_camera", "fisheye_camera", "event_camera", "lidar"]
      required_fields: ["sensors"]          # 至少需要配置的字段
      optional_fields: ["constraints", "topics"]
      mvp_priority: "P0"                    # 在MVP中的优先级

    - type_id: "perception.hearing"
      category: "perception"
      display_name: "听觉"
      display_name_en: "Hearing"
      icon: "ear"
      description: "通过麦克风采集和处理音频信号"
      sensor_types: ["microphone", "microphone_array", "ultrasonic_mic"]
      required_fields: ["sensors"]
      optional_fields: ["constraints", "topics"]
      mvp_priority: "P1"

    - type_id: "perception.force_sensing"
      category: "perception"
      display_name: "力觉"
      display_name_en: "Force Sensing"
      icon: "hand-press"
      description: "通过力/力矩传感器感知接触力"
      sensor_types: ["force_torque_6dof", "load_cell", "force_sensor_1d",
                      "tactile_array", "collision_sensor"]
      required_fields: ["sensors"]
      mvp_priority: "P0"

    - type_id: "perception.localization"
      category: "perception"
      display_name: "位置感知"
      display_name_en: "Localization"
      icon: "map-pin"
      description: "获取机器人在环境中的位置和姿态"
      sensor_types: ["imu", "gps", "rtk_gps", "uwb_tag", "wheel_encoder"]
      required_fields: ["method"]
      mvp_priority: "P0"

    - type_id: "perception.environment"
      category: "perception"
      display_name: "环境感知"
      display_name_en: "Environment"
      icon: "thermometer"
      description: "感知温度、湿度、气体等环境参数"
      sensor_types: ["temp_humidity_sensor", "gas_sensor", "barometer",
                      "light_sensor", "radiation_detector", "particle_sensor"]
      required_fields: ["sensors"]
      mvp_priority: "P1"

    - type_id: "locomotion.walking"
      category: "locomotion"
      display_name: "腿式行走"
      display_name_en: "Walking"
      icon: "footprints"
      description: "通过腿式结构实现移动"
      actuator_types: ["leg_group"]
      required_fields: ["actuators", "gaits", "constraints"]
      mvp_priority: "P1"

    - type_id: "locomotion.wheeling"
      category: "locomotion"
      display_name: "轮式移动"
      display_name_en: "Wheeling"
      icon: "circle-dot"
      description: "通过轮式底盘移动"
      actuator_types: ["wheel_drive"]
      required_fields: ["actuators", "constraints"]
      mvp_priority: "P0"

    - type_id: "locomotion.flying"
      category: "locomotion"
      display_name: "飞行"
      display_name_en: "Flying"
      icon: "plane"
      description: "通过旋翼或固定翼飞行"
      actuator_types: ["rotor_group"]
      required_fields: ["actuators", "constraints"]
      mvp_priority: "P2"

    - type_id: "locomotion.hybrid"
      category: "locomotion"
      display_name: "混合移动"
      display_name_en: "Hybrid"
      icon: "shuffle"
      description: "支持多种移动模式切换"
      required_fields: ["modes", "mode_capabilities"]
      mvp_priority: "P2"

    - type_id: "manipulation.grasping"
      category: "manipulation"
      display_name: "抓取"
      display_name_en: "Grasping"
      icon: "grab"
      description: "通过夹爪/灵巧手抓取物体"
      actuator_types: ["parallel_jaw", "suction_cup", "dexterous_hand",
                        "soft_gripper", "robot_arm"]
      required_fields: ["actuators", "constraints"]
      mvp_priority: "P1"

    - type_id: "manipulation.pushing"
      category: "manipulation"
      display_name: "推拉"
      display_name_en: "Pushing"
      icon: "arrow-right-from-line"
      description: "力控推拉操作"
      actuator_types: ["linear_actuator", "pneumatic_cylinder", "arm_end_effector"]
      required_fields: ["actuators", "constraints"]
      mvp_priority: "P0"

    - type_id: "manipulation.tool_use"
      category: "manipulation"
      display_name: "工具使用"
      display_name_en: "Tool Use"
      icon: "wrench"
      description: "持握并使用工具"
      required_fields: ["tools"]
      mvp_priority: "P2"

    - type_id: "manipulation.fine_motor"
      category: "manipulation"
      display_name: "精密操作"
      display_name_en: "Fine Motor"
      icon: "microscope"
      description: "高精度微操作"
      required_fields: ["actuators", "constraints"]
      mvp_priority: "P2"

    - type_id: "communication.speaking"
      category: "communication"
      display_name: "语音输出"
      display_name_en: "Speaking"
      icon: "volume-2"
      description: "通过扬声器输出语音"
      sensor_types: ["speaker", "speaker_array", "bone_conduction"]
      required_fields: ["sensors"]
      mvp_priority: "P0"

    - type_id: "communication.display"
      category: "communication"
      display_name: "视觉输出"
      display_name_en: "Display"
      icon: "monitor"
      description: "通过屏幕/LED显示信息"
      sensor_types: ["screen", "led_matrix", "led_indicator", "projector"]
      required_fields: ["sensors"]
      mvp_priority: "P2"

    - type_id: "communication.network"
      category: "communication"
      display_name: "网络通信"
      display_name_en: "Network"
      icon: "wifi"
      description: "无线或有线网络连接"
      required_fields: ["interfaces"]
      mvp_priority: "P0"

    - type_id: "communication.hri"
      category: "communication"
      display_name: "人机交互"
      display_name_en: "HRI"
      icon: "users"
      description: "手势/表情等多模态人机交互"
      required_fields: ["modalities"]
      mvp_priority: "P2"
```

---

## 三、能力画像数据模型

### 3.1 能力画像实体（CapabilityProfile）

```yaml
capability_profile:
  # ─── 基本信息 ───
  id: "string"                        # 全局唯一标识，如 "unitree-go2" 或 "user-tpl-001"
  name: "string"                      # 显示名称
  description: "string"               # 描述
  manufacturer: "string"              # 厂商名称
  form_factor: "enum"                 # 形态: quadruped | biped | humanoid | wheeled | tracked | aerial | hybrid
  thumbnail: "string"                 # 缩略图路径

  # ─── 模板类型 ───
  template_type: "enum"               # builtin（预置）| user（用户自定义）| instance（在线实例）
  owner: "string"                     # 所有者（user模板时为用户/代理商ID，builtin时为 "platform"）

  # ─── 继承关系 ───
  inherits_from: "string | null"      # 父模板ID（null=独立模板）
  overrides: {}                       # 相对父模板的差异字段

  # ─── 能力集合 ───
  capabilities:
    "capability_type":                 # key = 能力类型ID，如 "perception.vision"
      available: boolean
      reason: "string"                 # available=false时的原因
      # ... 该能力类型的完整Schema字段

  # ─── 性能基准 ───
  benchmark:
    battery_capacity_wh: number        # 电池容量（瓦时）
    battery_life_min: number           # 典型工作续航（分钟）
    weight_kg: number                  # 自重（千克）
    height_m: number                   # 高度（米）
    width_m: number                    # 宽度（米）
    length_m: number                   # 长度（米）
    ip_rating: "string"               # 防护等级
    operating_temp_range: [min, max]  # 工作温度范围（°C）
    charging_time_min: number          # 充电时间（分钟）
    payload_kg: number                 # 最大载荷（千克）

  # ─── 用户附加信息 ───
  custom_fields: {}                    # 自定义字段（不参与能力匹配）

  # ─── 版本管理 ───
  version: number                      # 版本号（递增整数）
  version_history:                     # 版本历史摘要
    - version: number
      changed_at: "datetime"
      changed_by: "string"
      change_summary: "string"
  created_at: "datetime"
  updated_at: "datetime"

  # ─── 关联 ───
  eurdf_file_id: "string | null"      # 关联的e-URDF文件ID（可选，有则能力声明从e-URDF同步）
  compatible_skills: ["string"]        # 该画像可执行的Skill列表（系统自动计算）
```

### 3.2 模板继承规则

```
继承解析算法（get_effective_profile）:

1. 加载当前模板的原始定义
2. 如果 inherits_from 不为空:
   a. 递归加载父模板（最多3层嵌套，防止循环）
   b. 以父模板的 capabilities 为基础
   c. 用当前模板的 overrides 逐字段覆盖
   d. overrides 中 available=false 可将父模板的 available=true 关闭
   e. overrides 中 available=true 可将父模板的 available=false 开启（新增硬件）
   f. overrides 中的 sensors/actuators 列表为**替换语义**（不是追加），须完整声明
3. 返回合并后的完整 capabilities
4. benchmark 字段：子模板有则覆盖父模板，无则继承

继承链示例:
  unitree-go1 (base)
    └── unitree-go2 (inherits_from: unitree-go1, overrides: +hearing, +speaking, ↑speed)
        └── unitree-go2-pro (inherits_from: unitree-go2, overrides: +grasping)

解析 unitree-go2-pro 的 effective profile:
  1. 加载 go1 → go2 覆盖 → go2-pro 覆盖
  2. 结果: go1基础能力 + go2增量 + go2-pro增量
```

### 3.3 能力画像与e-URDF的关系

```
两种数据来源，保持同步:

方式A: Dashboard先行（推荐新用户使用）
  Dashboard表单配置 → 保存为 CapabilityProfile → 可选导出为 e-URDF XML

方式B: e-URDF先行（已有URDF的OEM使用）
  上传 URDF/e-URDF → 解析提取能力声明 → 自动生成 CapabilityProfile

同步规则:
  - CapabilityProfile 是系统内部统一数据模型（YAML/JSON）
  - e-URDF 是面向ROS2生态的外部交换格式（XML）
  - 两者通过 eurdf_file_id 关联
  - 修改任一侧，系统提示同步到另一侧
  - 能力匹配和SOP编译使用 CapabilityProfile（非e-URDF）
```

---

## 四、Dashboard能力管理界面设计

### 4.1 页面结构总览

```
Dashboard → 机器人管理
│
├── 4.1 画像库（Profile Library）         # 浏览/搜索所有能力模板
├── 4.2 画像详情（Profile Detail）         # 查看某个模板的完整能力信息
├── 4.3 创建/编辑画像（Profile Editor）    # 配置能力的核心交互页面
├── 4.4 能力对比（Profile Compare）        # 多机器人能力横向对比
└── 4.5 在线机器人（Online Robots）        # 已注册机器人实例列表
```

### 4.2 画像库页面（Profile Library）

```
┌─────────────────────────────────────────────────────────────────────┐
│  机器人管理 > 画像库                                    [ + 新建画像 ] │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  [搜索机器人名称/厂商...]          筛选: [形态 ▼] [能力 ▼] [来源 ▼]  │
│                                                                      │
│  ┌─ 预置模板 ─────────────────────────────────────────────────────┐ │
│  │                                                                 │ │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐          │ │
│  │  │  [图片]  │  │  [图片]  │  │  [图片]  │  │  [图片]  │          │ │
│  │  │ Go1     │  │ Go2     │  │ Go2 Pro │  │ G1      │          │ │
│  │  │ 四足    │  │ 四足    │  │ 四足+臂  │  │ 人形    │          │ │
│  │  │ 6项能力 │  │ 8项能力 │  │ 9项能力  │  │ 11项能力│          │ │
│  │  │ [派生]  │  │ [派生]  │  │ [派生]   │  │ [派生]  │          │ │
│  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘          │ │
│  │                                                                 │ │
│  │  ┌─────────┐  ┌─────────┐                                     │ │
│  │  │  [图片]  │  │  [图片]  │                                     │ │
│  │  │送药机器人│  │巡检机器人│                                     │ │
│  │  │ 轮式    │  │ 轮式    │                                     │ │
│  │  │ 8项能力 │  │ 9项能力  │                                     │ │
│  │  │ [派生]  │  │ [派生]   │                                     │ │
│  │  └─────────┘  └─────────┘                                     │ │
│  └─────────────────────────────────────────────────────────────────┘ │
│                                                                      │
│  ┌─ 我的模板 ─────────────────────────────────────────────────────┐ │
│  │                                                                 │ │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐                       │ │
│  │  │  [图片]  │  │  [图片]  │  │  [图片]  │                       │ │
│  │  │3楼送药   │  │5楼送药   │  │A区巡检   │                       │ │
│  │  │ 基于:   │  │ 基于:   │  │ 基于:    │                       │ │
│  │  │送药V1   │  │送药V1   │  │巡检V1    │                       │ │
│  │  │ v3      │  │ v1      │  │ v2       │                       │ │
│  │  │[编辑][⋯]│  │[编辑][⋯]│  │[编辑][⋯] │                       │ │
│  │  └─────────┘  └─────────┘  └─────────┘                       │ │
│  └─────────────────────────────────────────────────────────────────┘ │
│                                                                      │
│  [ 对比已选 (0) ]                                                    │
└─────────────────────────────────────────────────────────────────────┘

卡片操作:
  - 预置模板: [派生] → 进入编辑器，基于该模板创建用户模板
  - 用户模板: [编辑] → 进入编辑器修改  |  [⋯] → 克隆/删除/导出e-URDF/查看版本历史
  - 勾选多个卡片 → [对比已选] → 进入能力对比页面
```

### 4.3 画像详情页面（Profile Detail）

```
┌─────────────────────────────────────────────────────────────────────┐
│  机器人管理 > 画像库 > 宇树 Go2                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────┐  宇树 Go2                                [编辑] [派生]│
│  │          │  四足机器人 | Unitree Robotics                         │
│  │  [图片]  │  模板类型: 预置模板                                    │
│  │          │  继承自: 宇树 Go1                                      │
│  └──────────┘  版本: v2 | 更新时间: 2026-06-20                       │
│                                                                      │
│  ── 性能基准 ──────────────────────────────────────────────────────  │
│  🔋 续航120min | ⚖️ 15.0kg | 🛡️ IP54 | 🌡️ -20~50°C              │
│                                                                      │
│  ── 能力清单 ──────────────────────────────────────────────────────  │
│                                                                      │
│  感知类                                                              │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │ ✅ 视觉 (perception.vision)                              [展开]│ │
│  │    传感器: 前置立体相机 (1920x1080@30fps) + 3D LiDAR (16线)    │ │
│  ├────────────────────────────────────────────────────────────────┤ │
│  │ ✅ 听觉 (perception.hearing)                             [展开]│ │
│  │    传感器: 4通道麦克风阵列 (16kHz)                             │ │
│  ├────────────────────────────────────────────────────────────────┤ │
│  │ ✅ 定位 (perception.localization)                         [展开]│ │
│  │    方法: Visual SLAM + IMU                                     │ │
│  ├────────────────────────────────────────────────────────────────┤ │
│  │ ❌ 力觉 (perception.force_sensing)                             │ │
│  │    原因: no_force_sensor                                       │ │
│  ├────────────────────────────────────────────────────────────────┤ │
│  │ ❌ 环境感知 (perception.environment)                           │ │
│  │    原因: no_environment_sensors                                │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                      │
│  运动类                                                              │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │ ✅ 腿式行走 (locomotion.walking)                          [展开]│ │
│  │    最大速度: 5.0m/s | 步态: trot, crawl, bound | 载荷: 5kg    │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                      │
│  操作类                                                              │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │ ❌ 抓取 (manipulation.grasping)  原因: no_manipulator          │ │
│  ├────────────────────────────────────────────────────────────────┤ │
│  │ ❌ 推拉 (manipulation.pushing)   原因: no_manipulator          │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                      │
│  交互类                                                              │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │ ✅ 语音输出 (communication.speaking)                      [展开]│ │
│  │    扬声器: 3W | TTS: zh-CN, en-US                              │ │
│  ├────────────────────────────────────────────────────────────────┤ │
│  │ ✅ 网络 (communication.network)                           [展开]│ │
│  │    接口: WiFi + Ethernet                                       │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                      │
│  ── 可执行Skill ──────────────────────────────────────────────────  │
│  navigate_to_waypoint ✅ | speak_text ✅ | capture_thermal_image ❌ │
│  detect_target ✅ | open_door ❌ | wait_for_weight_change ❌        │
│  alert_operator ✅ | return_to_base ✅ | log_result ✅              │
│                                                                      │
│  ── SOP兼容性 ────────────────────────────────────────────────────  │
│  送药SOP: ❌ 不兼容（缺少: pushing, force_sensing）                 │
│  巡检SOP: ⚠️ 部分兼容（缺少: thermal_camera，可降级）              │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘

[展开] 后显示:
  该能力的全部传感器/执行器参数列表、约束参数、ROS2 topic配置
```

### 4.4 画像编辑器（Profile Editor）— 核心交互页面

```
┌─────────────────────────────────────────────────────────────────────┐
│  机器人管理 > 画像库 > 新建画像                    [保存草稿] [发布]  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  创建方式: ○ 从预置模板派生  ○ 从零创建  ○ 从URDF导入              │
│                                                                      │
│  ═══ 第1步: 基本信息 ═══════════════════════════════════════════════ │
│                                                                      │
│  名称: [________________]     厂商: [________________]              │
│  形态: [轮式 ▼]               描述: [________________________________]│
│  缩略图: [上传图片]                                                   │
│  基于模板: [定制送药机器人 V1 ▼]    (从预置模板派生时显示)           │
│                                                                      │
│  ═══ 第2步: 能力配置 ═══════════════════════════════════════════════ │
│                                                                      │
│  系统按四大类展示所有原子能力。每项能力可独立开启/关闭，              │
│  开启后展开详细配置表单。从模板派生时，继承的能力已预填。              │
│                                                                      │
│  ┌─ 感知类 ──────────────────────────────────────────────────────┐  │
│  │                                                                │  │
│  │  [✅] 视觉 (perception.vision)                        [▼ 展开] │  │
│  │  ┌──────────────────────────────────────────────────────────┐ │  │
│  │  │  传感器列表:                              [ + 添加传感器 ] │ │  │
│  │  │                                                          │ │  │
│  │  │  ┌ 传感器 #1 ─────────────────────────────────── [删除] ┐│ │  │
│  │  │  │ 名称: [front_rgb______]                              ││ │  │
│  │  │  │ 类型: [RGB相机 ▼]                                    ││ │  │
│  │  │  │                                                      ││ │  │
│  │  │  │ ┌─ RGB相机参数 ──────────────────────────────────┐  ││ │  │
│  │  │  │ │ 分辨率: [1920] x [1080]    帧率: [30] fps      │  ││ │  │
│  │  │  │ │ 水平视场角: [69.4]°   垂直视场角: [42.5]°      │  ││ │  │
│  │  │  │ │ 镜头类型: [定焦 ▼]    焦距: [4.0] mm           │  ││ │  │
│  │  │  │ │ 光圈: [f/2.0]        HDR: [✅]                 │  ││ │  │
│  │  │  │ │ 自动对焦: [❌]        自动曝光: [✅]            │  ││ │  │
│  │  │  │ │ 接口: [USB3 ▼]       安装位置: [body_link ▼]   │  ││ │  │
│  │  │  │ └────────────────────────────────────────────────┘  ││ │  │
│  │  │  └──────────────────────────────────────────────────────┘│ │  │
│  │  │                                                          │ │  │
│  │  │  ┌ 传感器 #2 ─────────────────────────────────── [删除] ┐│ │  │
│  │  │  │ 名称: [depth_camera___]                              ││ │  │
│  │  │  │ 类型: [RGBD相机 ▼]                                   ││ │  │
│  │  │  │ (RGBD相机特有参数表单...)                             ││ │  │
│  │  │  └──────────────────────────────────────────────────────┘│ │  │
│  │  │                                                          │ │  │
│  │  │  ─ 约束参数 ─                                           │ │  │
│  │  │  最远检测距离: [10.0] m   最近检测距离: [0.3] m          │ │  │
│  │  │  适用光照: [☑ 室内] [☑ 室外白天] [☐ 室外夜间] [☐ 红外夜视]│ │  │
│  │  │  防护等级: [IP54 ▼]      工作温度: [-20]°C ~ [50]°C     │ │  │
│  │  │                                                          │ │  │
│  │  │  ─ ROS2 Topic ─ (高级，可折叠)                           │ │  │
│  │  │  状态: [/sensors/camera/status___]                       │ │  │
│  │  │  数据: [/sensors/camera/image_raw]                       │ │  │
│  │  └──────────────────────────────────────────────────────────┘ │  │
│  │                                                                │  │
│  │  [✅] 力觉 (perception.force_sensing)                 [▼ 展开] │  │
│  │  (力觉配置表单...)                                             │  │
│  │                                                                │  │
│  │  [✅] 定位 (perception.localization)                   [▼ 展开] │  │
│  │  (定位配置表单，含定位方法多选...)                               │  │
│  │                                                                │  │
│  │  [❌] 听觉 (perception.hearing)                                │  │
│  │       不可用原因: [no_microphone____]                           │  │
│  │                                                                │  │
│  │  [❌] 环境感知 (perception.environment)                        │  │
│  │       不可用原因: [________________]                            │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌─ 运动类 ──────────────────────────────────────────────────────┐  │
│  │  [✅] 轮式移动 (locomotion.wheeling)                  [▼ 展开] │  │
│  │  (轮式移动配置表单...)                                         │  │
│  │                                                                │  │
│  │  [❌] 腿式行走 — 不可用                                       │  │
│  │  [❌] 飞行 — 不可用                                           │  │
│  │  [❌] 混合移动 — 不可用                                       │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌─ 操作类 ──────────────────────────────────────────────────────┐  │
│  │  (同上模式，展开后显示对应执行器配置表单)                       │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌─ 交互类 ──────────────────────────────────────────────────────┐  │
│  │  (同上模式)                                                    │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ═══ 第3步: 性能基准 ══════════════════════════════════════════════ │
│                                                                      │
│  电池容量: [___] Wh    续航时间: [240] min    充电时间: [___] min    │
│  自重: [25.0] kg       载荷: [10.0] kg        防护等级: [IP54 ▼]    │
│  外形尺寸: [___] x [___] x [___] m           工作温度: [__]~[__]°C  │
│                                                                      │
│  ═══ 第4步: 附加信息（可选）══════════════════════════════════════ │
│                                                                      │
│  部署位置: [________________]    联系人: [________________]          │
│  序列号: [________________]     备注: [________________________________]│
│                                                                      │
│  ═══ 校验结果 ═════════════════════════════════════════════════════ │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  ✅ 基本信息完整                                               │ │
│  │  ✅ 至少配置了1项运动能力                                      │ │
│  │  ✅ 至少配置了1项感知能力                                      │ │
│  │  ✅ 网络通信已配置                                             │ │
│  │  ⚠️ 未配置听觉能力，部分需要语音交互的Skill将不可用             │ │
│  │  ✅ 传感器参数无冲突                                           │ │
│  │                                                                │ │
│  │  可执行Skill预估: 9/13 (69%)                                  │ │
│  │  送药SOP兼容: ✅ 完全兼容 | 巡检SOP兼容: ❌ 缺少thermal_camera │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                      │
│                                      [保存草稿]  [发布]  [取消]     │
└─────────────────────────────────────────────────────────────────────┘
```

**表单动态行为说明：**

| 交互 | 行为 |
|------|------|
| 切换能力开关 ✅/❌ | 开启→展开配置表单（从模板派生时预填父模板值）；关闭→收起表单，显示原因输入框 |
| 选择传感器类型下拉框 | 动态渲染该传感器类型的特有属性表单（如选"热成像相机"则出现温度范围、NETD等字段） |
| [+ 添加传感器] | 在当前能力下追加一个新传感器配置块 |
| 修改任一字段 | 实时触发校验，更新底部校验结果区域 |
| [发布] | 执行完整校验→通过→保存为正式版本（版本号+1）→触发Task Memory缓存失效 |
| [保存草稿] | 保存当前编辑状态，不触发版本号递增，不影响已有编译结果 |

### 4.5 能力对比页面（Profile Compare）

```
┌─────────────────────────────────────────────────────────────────────┐
│  机器人管理 > 能力对比                                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  对比对象: [+ 添加机器人]  已选: Go2, Go2 Pro, 送药机器人V1          │
│                                                                      │
│  ┌─────────────────┬───────────┬────────────┬───────────────┐       │
│  │ 能力             │ Go2       │ Go2 Pro    │ 送药机器人V1   │       │
│  ├─────────────────┼───────────┼────────────┼───────────────┤       │
│  │ 形态             │ 四足      │ 四足+臂    │ 轮式           │       │
│  │ 自重             │ 15kg      │ 18kg       │ 25kg           │       │
│  │ 续航             │ 120min    │ 90min      │ 240min         │       │
│  ├─────────────────┼───────────┼────────────┼───────────────┤       │
│  │ 视觉             │ ✅ 立体+LiDAR│ ✅ 同Go2│ ✅ RGB+RGBD    │       │
│  │ 听觉             │ ✅ 4ch阵列│ ✅ 4ch阵列│ ❌              │       │
│  │ 力觉             │ ❌        │ ✅ 腕部FT  │ ✅ 称重传感器  │       │
│  │ 定位             │ ✅ VSLAM  │ ✅ VSLAM   │ ✅ LiDAR SLAM  │       │
│  │ 环境感知         │ ❌        │ ❌         │ ❌              │       │
│  ├─────────────────┼───────────┼────────────┼───────────────┤       │
│  │ 移动方式         │ ✅ 四足行走│ ✅ 四足行走│ ✅ 轮式差速    │       │
│  │ 最大速度         │ 5.0m/s   │ 5.0m/s     │ 0.8m/s         │       │
│  │ 载荷             │ 5kg      │ 5kg        │ 10kg           │       │
│  ├─────────────────┼───────────┼────────────┼───────────────┤       │
│  │ 抓取             │ ❌        │ ✅ 平行夹爪 │ ❌             │       │
│  │ 推拉             │ ❌        │ ✅ 臂推     │ ✅ 线性推杆    │       │
│  ├─────────────────┼───────────┼────────────┼───────────────┤       │
│  │ 语音输出         │ ✅ 3W     │ ✅ 3W      │ ✅ 5W          │       │
│  │ 网络             │ ✅ WiFi+ETH│ ✅ WiFi+ETH│ ✅ WiFi       │       │
│  ├─────────────────┼───────────┼────────────┼───────────────┤       │
│  │ 送药SOP兼容      │ ❌        │ ⚠️ 无称重   │ ✅             │       │
│  │ 巡检SOP兼容      │ ⚠️ 无热像  │ ⚠️ 无热像   │ ❌ 无热像      │       │
│  └─────────────────┴───────────┴────────────┴───────────────┘       │
│                                                                      │
│  差异高亮: ✅=具备  ❌=不具备  ⚠️=部分具备/可降级                    │
│  仅显示差异项: [☐]                                                   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.6 URDF导入流程

```
上传URDF文件
    │
    ▼
┌──────────────────────────────────────────────────┐
│  URDF解析结果                                      │
│                                                    │
│  文件: unitree_go2.urdf                            │
│  机器人名称: unitree_go2                            │
│  Link数量: 21                                      │
│  Joint数量: 20                                     │
│  检测到的传感器:                                     │
│    ✅ front_camera (camera类型, 已自动识别)          │
│    ✅ imu_sensor (imu类型, 已自动识别)              │
│    ⚠️ head_sensor (未知类型, 需手动指定)             │
│                                                    │
│  检测到的执行器:                                     │
│    ✅ 12个关节执行器 (四足结构, 已自动归组)           │
│                                                    │
│  自动推断的能力画像:                                 │
│    ✅ locomotion.walking (基于12关节四足结构)        │
│    ✅ perception.vision (基于camera传感器)          │
│    ✅ perception.localization (基于IMU)             │
│    ❓ 其他能力需手动配置                             │
│                                                    │
│  [ 进入编辑器补充配置 ]    [ 重新上传 ]              │
└──────────────────────────────────────────────────┘
    │
    ▼ 点击"进入编辑器"
打开 Profile Editor（第4.4节），已解析的字段预填
```

---

## 五、REST API设计

### 5.1 能力画像 CRUD

```yaml
# ── 列表查询 ──
GET /api/v1/profiles
  query:
    template_type: "builtin" | "user" | "all"     # 筛选模板类型
    form_factor: "quadruped" | "wheeled" | ...     # 筛选形态
    capability: "perception.vision"                 # 筛选包含某能力的模板
    owner: "string"                                 # 筛选所有者
    search: "string"                                # 模糊搜索名称/厂商
    page: number
    page_size: number
  response: 200
    body:
      total: number
      items:
        - id, name, form_factor, template_type, owner, thumbnail,
          capability_count, version, updated_at

# ── 获取详情（含继承解析后的完整能力） ──
GET /api/v1/profiles/{profile_id}
  query:
    resolve_inheritance: true | false              # 是否解析继承链返回完整能力
  response: 200
    body: CapabilityProfile (完整YAML结构)

# ── 创建 ──
POST /api/v1/profiles
  body:
    name: "string"                                 # 必填
    form_factor: "string"                          # 必填
    inherits_from: "string | null"                 # 可选
    capabilities: {}                               # 能力配置
    benchmark: {}                                  # 性能基准
    custom_fields: {}                              # 附加信息
  response: 201
    body: {id, version}

# ── 更新（创建新版本） ──
PUT /api/v1/profiles/{profile_id}
  body: (同创建，覆盖更新)
  response: 200
    body: {id, version}                            # version自动+1

# ── 删除 ──
DELETE /api/v1/profiles/{profile_id}
  response: 204
  注意: 预置模板不可删除；被在线机器人引用的模板不可删除

# ── 克隆 ──
POST /api/v1/profiles/{profile_id}/clone
  body:
    new_name: "string"
  response: 201
    body: {id, version}
```

### 5.2 版本管理

```yaml
# ── 版本历史 ──
GET /api/v1/profiles/{profile_id}/versions
  response: 200
    body:
      - version, changed_at, changed_by, change_summary

# ── 获取指定版本 ──
GET /api/v1/profiles/{profile_id}/versions/{version}
  response: 200
    body: CapabilityProfile

# ── 版本对比 ──
GET /api/v1/profiles/{profile_id}/versions/diff
  query:
    from_version: number
    to_version: number
  response: 200
    body:
      added: [字段路径列表]
      removed: [字段路径列表]
      changed: [{path, old_value, new_value}]

# ── 回滚 ──
POST /api/v1/profiles/{profile_id}/rollback
  body:
    target_version: number
  response: 200
    body: {id, version}                            # 回滚产生新版本
```

### 5.3 能力匹配查询

```yaml
# ── 检查画像与Skill的兼容性 ──
GET /api/v1/profiles/{profile_id}/skill-compatibility
  response: 200
    body:
      compatible_skills:
        - skill_name: "string"
          compatible: true | false
          missing_capabilities: ["string"]         # 缺失的必需能力
          degraded_capabilities: ["string"]        # 缺失的可选能力

# ── 检查画像与SOP的兼容性 ──
POST /api/v1/profiles/{profile_id}/sop-compatibility
  body:
    sop_text: "string"                             # SOP文本
  response: 200
    body:
      verdict: "EXECUTABLE" | "EXECUTABLE_DEGRADED" | "NOT_EXECUTABLE"
      details: (可执行性报告，见系统设计文档4.2.4节)

# ── 批量兼容性检查（多画像 × 一个SOP） ──
POST /api/v1/profiles/batch-sop-compatibility
  body:
    profile_ids: ["string"]
    sop_text: "string"
  response: 200
    body:
      results:
        - profile_id, profile_name, verdict, missing_count, degraded_count
```

### 5.4 URDF导入

```yaml
# ── 上传并解析URDF ──
POST /api/v1/profiles/import-urdf
  content-type: multipart/form-data
  body:
    file: (URDF文件)
  response: 200
    body:
      parsed_name: "string"
      detected_links: number
      detected_joints: number
      detected_sensors: [{name, type, auto_recognized}]
      detected_actuators: [{name, type, auto_recognized}]
      inferred_capabilities: [{type, confidence}]
      draft_profile: CapabilityProfile              # 自动生成的草稿画像

# ── 确认导入（将草稿保存为正式画像） ──
POST /api/v1/profiles/import-urdf/confirm
  body:
    draft_profile: CapabilityProfile                # 用户修改后的画像
  response: 201
    body: {id, version}
```

### 5.5 能力类型注册表查询

```yaml
# ── 获取所有能力类型元信息（供前端渲染表单） ──
GET /api/v1/capability-types
  response: 200
    body:
      types:
        - type_id, category, display_name, display_name_en, icon,
          description, sensor_types, actuator_types, required_fields,
          optional_fields, mvp_priority

# ── 获取某传感器类型的属性Schema（供前端动态渲染表单字段） ──
GET /api/v1/capability-types/sensor-schema/{sensor_type}
  response: 200
    body:
      sensor_type: "string"
      display_name: "string"
      fields:
        - name, type, required, default, unit, description, enum_values, range
```

---

## 六、数据存储设计

### 6.1 数据库表结构

```sql
-- 能力画像主表
CREATE TABLE capability_profiles (
    id              VARCHAR(64) PRIMARY KEY,
    name            VARCHAR(128) NOT NULL,
    description     TEXT,
    manufacturer    VARCHAR(128),
    form_factor     VARCHAR(32) NOT NULL,     -- quadruped|wheeled|humanoid|...
    thumbnail_url   VARCHAR(512),
    template_type   VARCHAR(16) NOT NULL,     -- builtin|user
    owner           VARCHAR(64),              -- 用户/代理商ID
    inherits_from   VARCHAR(64) REFERENCES capability_profiles(id),
    overrides_json  JSONB,                    -- 继承覆盖字段
    capabilities_json JSONB NOT NULL,         -- 完整能力配置
    benchmark_json  JSONB,                    -- 性能基准
    custom_fields_json JSONB,                 -- 用户附加信息
    eurdf_file_id   VARCHAR(64),
    version         INTEGER NOT NULL DEFAULT 1,
    created_at      TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMP NOT NULL DEFAULT NOW(),
    created_by      VARCHAR(64),
    status          VARCHAR(16) DEFAULT 'active'  -- active|archived|draft
);

-- 版本历史表
CREATE TABLE profile_versions (
    id              SERIAL PRIMARY KEY,
    profile_id      VARCHAR(64) NOT NULL REFERENCES capability_profiles(id),
    version         INTEGER NOT NULL,
    snapshot_json   JSONB NOT NULL,           -- 该版本的完整快照
    changed_by      VARCHAR(64),
    change_summary  TEXT,
    created_at      TIMESTAMP NOT NULL DEFAULT NOW(),
    UNIQUE(profile_id, version)
);

-- 在线机器人实例表（与画像的绑定关系）
CREATE TABLE robot_instances (
    id              VARCHAR(64) PRIMARY KEY,
    profile_id      VARCHAR(64) NOT NULL REFERENCES capability_profiles(id),
    serial_number   VARCHAR(128),
    deployment_location VARCHAR(256),
    online_status   VARCHAR(16) DEFAULT 'offline',  -- online|offline|fault
    last_heartbeat  TIMESTAMP,
    runtime_capabilities_json JSONB,          -- 运行时能力状态快照
    created_at      TIMESTAMP NOT NULL DEFAULT NOW()
);

-- 能力类型注册表（系统初始化时导入，应用启动时加载到内存）
CREATE TABLE capability_type_registry (
    type_id         VARCHAR(64) PRIMARY KEY,  -- 如 perception.vision
    category        VARCHAR(32) NOT NULL,
    display_name    VARCHAR(64) NOT NULL,
    display_name_en VARCHAR(64),
    icon            VARCHAR(32),
    description     TEXT,
    sensor_types    JSONB,                    -- 该能力可用的传感器类型列表
    actuator_types  JSONB,
    schema_json     JSONB NOT NULL,           -- 该能力的完整JSON Schema
    mvp_priority    VARCHAR(4) DEFAULT 'P2'
);

-- 传感器类型Schema表
CREATE TABLE sensor_type_schemas (
    sensor_type     VARCHAR(64) PRIMARY KEY,  -- 如 rgb_camera
    display_name    VARCHAR(64) NOT NULL,
    category        VARCHAR(32),              -- 所属能力类型
    fields_json     JSONB NOT NULL,           -- 属性字段定义
    created_at      TIMESTAMP NOT NULL DEFAULT NOW()
);

-- 索引
CREATE INDEX idx_profiles_template_type ON capability_profiles(template_type);
CREATE INDEX idx_profiles_owner ON capability_profiles(owner);
CREATE INDEX idx_profiles_form_factor ON capability_profiles(form_factor);
CREATE INDEX idx_profiles_inherits ON capability_profiles(inherits_from);
CREATE INDEX idx_versions_profile ON profile_versions(profile_id);
CREATE INDEX idx_robots_profile ON robot_instances(profile_id);
```

### 6.2 预置模板数据初始化

系统首次部署时，通过数据迁移脚本将预置模板（见系统设计文档3.7.1节）导入 `capability_profiles` 表，`template_type = 'builtin'`，`owner = 'platform'`。

预置模板的更新通过平台版本升级分发，升级脚本自动合并新版本预置模板（不覆盖用户模板）。

---

## 七、校验规则

### 7.1 画像发布前校验

```yaml
validation_rules:
  # ── 基本信息校验 ──
  basic:
    - rule: "名称非空且唯一（同一owner下）"
      level: ERROR
    - rule: "form_factor为有效枚举值"
      level: ERROR

  # ── 能力配置校验 ──
  capability:
    - rule: "至少配置1项运动能力（locomotion.*）"
      level: ERROR
      message: "机器人必须具备至少一种移动方式"
    - rule: "至少配置1项感知能力（perception.*）"
      level: ERROR
      message: "机器人必须具备至少一种感知能力"
    - rule: "必须配置网络通信（communication.network）"
      level: ERROR
      message: "RobotClaw平台要求机器人具备网络连接能力"
    - rule: "每项已开启的能力至少配置1个传感器/执行器"
      level: ERROR
    - rule: "传感器名称在画像内唯一"
      level: ERROR

  # ── 参数合理性校验 ──
  parameter:
    - rule: "分辨率格式符合 WxH 模式"
      level: ERROR
    - rule: "帧率 > 0 且 <= 240"
      level: ERROR
    - rule: "最大速度 > 0 且 <= 20 m/s"
      level: WARNING
      message: "速度超过20m/s，请确认数值正确"
    - rule: "温度范围 min < max"
      level: ERROR
    - rule: "力范围 min <= max 且 >= 0"
      level: ERROR
    - rule: "续航时间 > 0"
      level: WARNING

  # ── 继承一致性校验 ──
  inheritance:
    - rule: "inherits_from引用的模板存在且状态为active"
      level: ERROR
    - rule: "继承链深度 <= 3"
      level: ERROR
      message: "模板继承最多3层"
    - rule: "无循环继承"
      level: ERROR
    - rule: "overrides中的能力类型必须在能力类型注册表中存在"
      level: ERROR

  # ── 关联影响检查 ──
  impact:
    - rule: "发布新版本时，检查是否有在线机器人绑定该画像"
      level: WARNING
      message: "有N台在线机器人使用此画像，更新后将影响其SOP编译结果"
    - rule: "关闭某项原有能力时，检查是否有已编译DAG依赖该能力"
      level: WARNING
      message: "关闭此能力将导致N个已编译DAG的Task Memory缓存失效"
```

---

## 八、与上下游模块的接口关系

```
┌─────────────────────────────────────────────────────────────────────┐
│                          上游模块                                    │
│                                                                      │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐            │
│  │ e-URDF解析器  │   │ URDF导入工具  │   │ Dashboard UI │            │
│  │  → 创建/更新  │   │  → 创建画像  │   │  → CRUD操作  │            │
│  │    能力画像   │   │              │   │              │            │
│  └──────┬───────┘   └──────┬───────┘   └──────┬───────┘            │
└─────────┼──────────────────┼──────────────────┼────────────────────┘
          │                  │                  │
          ▼                  ▼                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   Capability Profile Service                         │
│                                                                      │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐      │
│  │ Profile    │ │ 继承解析   │ │ 版本管理   │ │ 校验引擎   │      │
│  │ CRUD       │ │ Engine     │ │ Manager    │ │ Validator  │      │
│  └────────────┘ └────────────┘ └────────────┘ └────────────┘      │
│  ┌────────────┐ ┌────────────┐                                     │
│  │ 能力匹配   │ │ 兼容性     │                                     │
│  │ Matcher    │ │ Checker    │                                     │
│  └────────────┘ └────────────┘                                     │
└────────┬──────────────┬──────────────┬──────────────┬──────────────┘
         │              │              │              │
         ▼              ▼              ▼              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                          下游模块                                    │
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│  │ SOP Compiler │  │ Skill        │  │ 执行引擎     │              │
│  │  查询画像做  │  │ Registry     │  │  运行时查询  │              │
│  │  能力匹配    │  │  Skill→Cap  │  │  能力状态    │              │
│  │  可执行性验证 │  │  匹配查询    │  │              │              │
│  └──────────────┘  └──────────────┘  └──────────────┘              │
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐                                │
│  │ Task Memory  │  │ 场景模板     │                                │
│  │  缓存Key含   │  │  推荐适用    │                                │
│  │  画像hash    │  │  机器人      │                                │
│  └──────────────┘  └──────────────┘                                │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 九、实现优先级

### Phase 1（Month 1-2）— MVP

| 模块 | 范围 | 优先级 |
|------|------|--------|
| 能力类型注册表 | 全部17种能力类型的Schema定义和元信息 | P0 |
| 预置模板 | 6个预置模板（Go1/Go2/Go2Pro/G1/送药/巡检） | P0 |
| Profile CRUD API | 创建/读取/更新/删除/克隆 | P0 |
| 继承解析引擎 | 支持最多3层继承链解析 | P0 |
| Dashboard画像库页面 | 浏览/搜索预置和用户模板 | P0 |
| Dashboard画像编辑器 | MVP场景涉及的8种能力（vision/force_sensing/localization/wheeling/pushing/speaking/network/hearing）的表单 | P0 |
| 能力匹配查询API | 单画像×Skill兼容性检查 | P0 |
| 画像发布校验 | 基本信息+能力配置+参数合理性校验 | P0 |

### Phase 2（Month 3-4）— 完善

| 模块 | 范围 | 优先级 |
|------|------|--------|
| URDF导入 | 上传→解析→自动推断→编辑器补充 | P1 |
| 版本管理 | 版本历史/对比/回滚 | P1 |
| 能力对比页面 | 多画像横向对比表 | P1 |
| SOP兼容性批量检查 | 多画像×SOP批量检查 | P1 |
| 非MVP能力表单 | walking/flying/hybrid/grasping/tool_use/fine_motor/display/hri | P1 |
| e-URDF双向同步 | Profile↔e-URDF XML互相转换和同步 | P1 |

### Phase 3（Month 5-6）— 扩展

| 模块 | 范围 | 优先级 |
|------|------|--------|
| 在线机器人实例管理 | 机器人注册/心跳/运行时能力状态 | P2 |
| 画像详情中的Skill可执行列表 | 自动计算某画像可执行的Skill清单 | P2 |
| 预置模板自动更新 | 平台升级时自动合并新版本预置模板 | P2 |
| 能力类型扩展机制 | 支持用户自定义能力类型（非标传感器） | P2 |
