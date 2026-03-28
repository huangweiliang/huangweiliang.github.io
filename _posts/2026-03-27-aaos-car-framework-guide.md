---
layout: post
title:  "AAOS (Android Automotive OS) study guide (chn)"
subtitle: "android Automotive SDV"
header-img: img/post-bg-coffee.jpeg
date:   2026-03-28 08:55:59
author: half cup coffee
catalog: true
tags:	
    - Android
---

# AAOS (Android Automotive OS) 深度指南

## 目录

- [一、AAOS 整体概述](#一aaos-整体概述)
- [二、系统架构全景](#二系统架构全景)
- [三、Android Framework - Car APIs](#三android-framework---car-apis)
- [四、CarService](#四carservice)
- [五、CarAudioService](#五caraudioservice)
- [六、HVAC (车载空调系统)](#六hvac-车载空调系统)
- [七、CarUserManager](#七carusermanager)
- [八、VehiclePropertyManager (CarPropertyManager)](#八vehiclepropertymanager-carpropertymanager)
- [九、Vehicle HAL (VHAL)](#九vehicle-hal-vhal)
- [十、各模块代码位置速查表](#十各模块代码位置速查表)
- [十一、典型数据流示例](#十一典型数据流示例)
- [十二、Google SDV 战略与 AAOS 演进](#十二google-sdv-战略与-aaos-演进)

---

## 一、AAOS 整体概述

### 1.1 什么是 AAOS

AAOS（Android Automotive OS）是 Google 推出的**嵌入式车载信息娱乐系统 (IVI) 操作系统**，直接运行在车辆的车机硬件上。随着软件定义汽车（SDV）趋势深化，AAOS 从 Android 15/16 起正在向更广泛的车辆域（座舱、ADAS、车身控制）扩展，不再局限于 IVI。

> **与 Android Auto 的区别：**
> - Android Auto：手机投屏方案，车机只做渲染，核心逻辑在手机上
> - AAOS：完整 Android 系统运行在车机上，不依赖手机

### 1.2 与标准 Android 的核心差异

| 方面 | 标准 Android | AAOS |
|------|-------------|------|
| **目标设备** | 手机/平板 | 车机 IVI |
| **多用户** | 基础多用户 | 驾驶员配置文件 (Driver Profile)，支持切换座椅/个性化 |
| **音频** | AudioFlinger | **CarAudioService** — 支持音区 (Audio Zone)、优先级路由 |
| **输入方式** | 触屏为主 | 旋钮、方向盘按键、触屏、语音 |
| **Vehicle HAL** | 无 | 核心组件，抽象车辆总线信号 |
| **安全等级** | 消费级 | 需满足车规 (驾驶分心指南 / Driver Distraction) |
| **系统 UI** | Launcher3 | CarLauncher、CarSystemUI、分屏地图优先显示 |
| **多屏** | 有限支持 | 原生多屏（仪表/中控/后排） |

### 1.3 典型硬件平台

| SoC | 厂商 | 说明 |
|-----|------|------|
| SA8155P / SA8295P | Qualcomm | 上一代主流高端车机平台，支持 QNX + AAOS 虚拟化 |
| SA8775P | Qualcomm | 新一代中高端座舱平台（2023+），多屏 + AI 算力增强 |
| SA8650P | Qualcomm | 成本优化车机平台（2024+） |
| Snapdragon Ride Flex SoC | Qualcomm | 可扩展 SDV 平台，支持座舱 + ADAS 域融合（2025+） |
| Exynos Auto V9 | Samsung | 现代/起亚使用 |
| R-Car H3/M3/S4 | Renesas | 参考平台，S4 支持 IEC 61508 车规安全等级 |
| Intel Atom / i3 | Intel | 部分 Volvo 车型 |

### 1.4 虚拟化架构（常见量产方案）

```
┌──────────────────┐  ┌──────────────────┐
│   QNX (安全域)    │  │   AAOS (IVI域)    │
│  仪表盘 / ADAS   │  │  导航 / 娱乐      │
└────────┬─────────┘  └────────┬─────────┘
         │                     │
    ┌────┴─────────────────────┴────┐
    │    Hypervisor (QNX / Gunyah)  │
    ├───────────────────────────────┤
    │          Hardware (SoC)       │
    └───────────────────────────────┘
```

### 1.5 量产车型（部分）

- **Volvo / Polestar**：最早量产 AAOS（内置 Google 服务）
- **GM（通用）**：Chevrolet、Buick、GMC，2023+ 车型
- **Ford**：2023+ 部分车型
- **BMW**：iDrive 基于 AAOS
- **Honda / Acura**
- **Renault / Nissan**
- **中国自主品牌**：多基于 AAOS 定制（不含 GMS）

### 1.6 Google SDV 战略定位

Google 将 AAOS 定位为 **软件定义汽车（SDV）的平台基层**，不再局限于 IVI。核心理念：

- **开放（Open）**：AOSP 为基础，OEM 可自由定制，不锁定來商
- **可更新（Updatable）**：模块化设计支持 OTA 全车升级，包括底层驱动和 HAL
- **可扩展（Scalable）**：从中控 IVI 到座舱域控制器，单一平台覆盖多个车辆圭
- **智能（Intelligent）**：内置 Google 服务（Assistant、Maps、Cloud AI），赋能车内智能体验

---

## 二、系统架构全景

```
┌─────────────────────────────────────────────────────────────────┐
│                        应用层 (Apps)                             │
│  CarLauncher · CarSettings · CarRadio · 地图 · 音乐 · 第三方App │
├─────────────────────────────────────────────────────────────────┤
│                    Car App API / car-lib                        │
│  android.car.* SDK 接口 (App 侧可调用的 API)                    │
├─────────────────────────────────────────────────────────────────┤
│                  CarService (系统服务层)                         │
│  ┌──────────────┬───────────────┬──────────────┬──────────────┐ │
│  │CarPropertySvc│CarAudioService│CarHvacService│CarUserService│ │
│  ├──────────────┼───────────────┼──────────────┼──────────────┤ │
│  │CarPowerMgmtSv│CarInputService│CarWatchdog   │CarUxRestrSvc │ │
│  ├──────────────┼───────────────┼──────────────┼──────────────┤ │
│  │CarEvsService │CarClusterSvc  │CarTelemetrySv│CarOccupantSvc│ │
│  └──────────────┴───────────────┴──────────────┴──────────────┘ │
├─────────────────────────────────────────────────────────────────┤
│                Vehicle HAL (AIDL 接口)                          │
│  IVehicle.aidl · IVehicleCallback.aidl                         │
├─────────────────────────────────────────────────────────────────┤
│              OEM VHAL 实现 (Vendor 层)                          │
│  CAN 总线驱动 · LIN · SOME/IP · 私有协议                       │
├─────────────────────────────────────────────────────────────────┤
│              Linux Kernel + 设备驱动                            │
│  CAN driver · I2C · SPI · GPU · Display · Audio codec          │
└─────────────────────────────────────────────────────────────────┘
```

---

## 三、Android Framework - Car APIs

### 3.1 概述

Car APIs 是 AAOS 提供给应用层的 **Java/Kotlin SDK 接口**，封装在 `android.car` 包中。应用通过这些 API 访问车辆功能，而不需要直接与 HAL 或底层总线通信。

### 3.2 代码位置

```
packages/services/Car/car-lib/
├── src/android/car/
│   ├── Car.java                          # 核心入口类
│   ├── hardware/
│   │   ├── CarPropertyManager.java       # 车辆属性读写
│   │   ├── CarPropertyValue.java         # 属性值封装
│   │   └── property/
│   │       └── VehiclePropertyIds.java   # 所有属性 ID 常量
│   ├── media/
│   │   └── CarMediaManager.java          # 媒体控制
│   ├── drivingstate/
│   │   ├── CarDrivingStateManager.java   # 驾驶状态
│   │   └── CarUxRestrictionsManager.java # UX 限制
│   ├── user/
│   │   └── CarUserManager.java           # 用户管理
│   ├── input/
│   │   └── CarInputManager.java          # 输入事件
│   ├── cluster/
│   │   └── ClusterHomeManager.java       # 仪表盘
│   ├── watchdog/
│   │   └── CarWatchdogManager.java       # 系统健康监控
│   ├── evs/
│   │   └── CarEvsManager.java            # 外部摄像头
│   ├── telemetry/
│   │   └── CarTelemetryManager.java      # 遥测数据
│   ├── remoteaccess/
│   │   └── CarRemoteAccessManager.java   # 车辆远程唤醒/远程任务 (Android 14+)
│   └── occupantconnection/
│       └── CarOccupantConnectionManager.java # 跨乘员区通信 (Android 14+)
```

### 3.3 核心入口 —— `Car` 类

```java
// 创建 Car 实例（同步方式）
Car car = Car.createCar(context);

// 获取各个 Manager
CarPropertyManager   propMgr  = (CarPropertyManager)   car.getCarManager(Car.PROPERTY_SERVICE);
CarAudioManager      audioMgr = (CarAudioManager)      car.getCarManager(Car.AUDIO_SERVICE);
CarUxRestrictionsManager uxMgr = (CarUxRestrictionsManager) car.getCarManager(Car.CAR_UX_RESTRICTION_SERVICE);
CarUserManager       userMgr  = (CarUserManager)        car.getCarManager(Car.CAR_USER_SERVICE);

// 使用完毕
car.disconnect();
```

### 3.4 API 包一览

| 包名 | 核心类 | 功能 |
|------|--------|------|
| `android.car` | `Car` | SDK 入口，获取各 Manager |
| `android.car.hardware` | `CarPropertyManager` | 读写车辆属性 |
| `android.car.hardware.property` | `VehiclePropertyIds` | 属性 ID 常量（车速、挡位等） |
| `android.car.media` | `CarMediaManager` | 车载媒体源管理 |
| `android.car.drivingstate` | `CarUxRestrictionsManager` | 驾驶分心限制 |
| `android.car.user` | `CarUserManager` | 驾驶员/乘客用户切换 |
| `android.car.input` | `CarInputManager` | 方向盘按键/旋钮事件 |
| `android.car.cluster` | `ClusterHomeManager` | 仪表盘 Cluster 渲染 |
| `android.car.diagnostic` | `CarDiagnosticManager` | OBD-II 诊断数据 |
| `android.car.watchdog` | `CarWatchdogManager` | 进程健康检查 |
| `android.car.evs` | `CarEvsManager` | 环视/倒车摄像头 |
| `android.car.telemetry` | `CarTelemetryManager` | 遥测数据采集 |
| `android.car.occupantawareness` | `OccupantAwarenessManager` | DMS 驾驶员监测 |
| `android.car.content` | `CarAppFocusManager` | App 焦点仲裁 |
| `android.car.remoteaccess` | `CarRemoteAccessManager` | 远程唤醒车辆 / 下发远程任务（Android 14+） |
| `android.car.occupantconnection` | `CarOccupantConnectionManager` | 跨乘员区（主驾/后排）数据通道（Android 14+） |
| `android.car` | `CarOccupantZoneManager` | 多乘员区域管理，Android 16 新增跨区域权限 API |

### 3.5 App 如何引用

**Android.bp（AOSP 内部 App）：**
```
android_app {
    name: "MyCarApp",
    libs: ["android.car"],
    sdk_version: "current",
}
```

**Gradle（外部开发者）：**
```groovy
dependencies {
    implementation "androidx.car.app:app:1.7.0"  // Car App Library (Android 16 时代推荐版本)
}
```

---

## 四、CarService

### 4.1 概述

CarService 是 AAOS 的**核心系统服务**，运行在 `system_server` 进程中（实际是独立进程 `com.android.car`）。它是所有 `android.car.*` API 的服务端实现，充当应用层与 Vehicle HAL 之间的桥梁。

### 4.2 代码位置

```
packages/services/Car/
├── car-lib/                    # 客户端 SDK（android.car.* API）
├── service/                    # CarService 服务端实现
│   └── src/com/android/car/
│       ├── CarService.java                  # 服务入口，初始化所有子服务
│       ├── ICarImpl.java                    # ICar.aidl 的实现
│       ├── CarPropertyService.java          # 车辆属性服务
│       ├── CarAudioService.java             # 音频服务
│       ├── CarHvacService.java              # 空调服务 (旧版)
│       ├── CarUserService.java              # 用户服务
│       ├── CarPowerManagementService.java   # 电源管理
│       ├── CarInputService.java             # 输入服务
│       ├── CarUxRestrictionsManagerService.java # UX限制
│       ├── CarWatchdogService.java          # 看门狗
│       ├── CarEvsService.java               # 摄像头
│       ├── CarDrivingStateService.java      # 驾驶状态
│       ├── CarRemoteAccessService.java      # 远程唤醒服务 (Android 14+)
│       ├── CarOccupantConnectionService.java # 跨区域通信服务 (Android 14+)
│       └── ...
├── tests/                      # 单元测试 + CTS
├── car-builtin-lib/            # System API 内部库
└── framework/                  # 部分 framework 扩展
```

### 4.3 架构详解

```
          App 进程                          CarService 进程
  ┌─────────────────────┐           ┌──────────────────────────┐
  │ CarPropertyManager  │──Binder──▶│ CarPropertyService       │
  │ CarAudioManager     │──Binder──▶│ CarAudioService          │
  │ CarUserManager      │──Binder──▶│ CarUserService           │
  │ ...                 │           │ ...                      │
  └─────────────────────┘           └────────────┬─────────────┘
                                                 │
                                          AIDL/HIDL Binder
                                                 │
                                    ┌────────────▼─────────────┐
                                    │   Vehicle HAL (VHAL)     │
                                    │  (独立进程 / vendor 分区)  │
                                    └──────────────────────────┘
```

### 4.4 启动流程

1. `system_server` 启动后，`CarServiceHelperService` 被加载
2. `CarServiceHelperService` 启动 `CarService`（独立进程 `com.android.car`）
3. `CarService.onCreate()` → 创建 `ICarImpl`
4. `ICarImpl` 构造函数中初始化所有子服务：
   - `VehicleHal` → 连接 VHAL
   - `CarPropertyService`
   - `CarAudioService`
   - `CarUserService`
   - `CarPowerManagementService`
   - ... 共 30+ 个子服务

```java
// ICarImpl.java (简化)
public class ICarImpl extends ICar.Stub {
    ICarImpl(Context context, IVehicle vehicle, ...) {
        mVehicleHal = new VehicleHal(vehicle);
        mCarPropertyService = new CarPropertyService(context, mVehicleHal.getPropertyHal());
        mCarAudioService = new CarAudioService(context);
        mCarUserService = new CarUserService(context, mVehicleHal);
        mCarHvacService = new CarHvacService(context, mVehicleHal.getPropertyHal());
        // ...
    }
}
```

### 4.5 关键接口

```
// AIDL 接口
packages/services/Car/car-lib/src/android/car/ICar.aidl
packages/services/Car/car-lib/src/android/car/hardware/ICarProperty.aidl
```

---

## 五、CarAudioService

### 5.1 概述

CarAudioService 是 AAOS 的**车载音频管理服务**，在标准 Android `AudioFlinger` / `AudioPolicyService` 之上增加了车载特有功能：

- **多音区 (Audio Zone)**：主驾、副驾、后排各有独立音频路由
- **音频焦点仲裁**：比手机更复杂的焦点策略（导航不能被音乐打断）
- **鸭音 (Ducking)**：导航播报时音乐降低音量
- **音量组 (Volume Group)**：按用途分组（媒体、导航、电话、报警）

### 5.2 代码位置

```
packages/services/Car/service/src/com/android/car/audio/
├── CarAudioService.java              # 入口，实现 ICarAudio.aidl
├── CarAudioZonesHelper.java          # 音区配置解析
├── CarAudioZone.java                 # 音区抽象
├── CarVolumeGroup.java               # 音量组
├── CarAudioFocus.java                # 音频焦点仲裁逻辑
├── CarDucking.java                   # 鸭音策略
├── CarAudioDeviceInfo.java           # 音频设备信息
├── CarAudioPlaybackCallback.java     # 播放状态回调
└── CarAudioPolicyVolumeCallback.java # 音量策略回调

# 配置文件
device/<OEM>/<product>/car_audio_configuration.xml   # OEM 音区配置

# SDK 接口
packages/services/Car/car-lib/src/android/car/media/
├── CarAudioManager.java              # App 侧 API
├── CarAudioPatchHandle.java
└── ICarAudio.aidl                    # Binder 接口
```

### 5.3 架构

```
  App (MediaPlayer)           CarAudioManager
        │                         │
        ▼                         ▼ (Binder)
  AudioFlinger            CarAudioService
        │                    │          │
        │              音区路由     焦点仲裁
        │                    │          │
        ▼                    ▼          ▼
  AudioPolicyService ◀── CarAudioPolicy
        │
        ▼
  Audio HAL (AIDL)
        │
        ▼
  Audio Codec / Amplifier (车载功放)
```

### 5.4 多音区概念

```xml
<!-- car_audio_configuration.xml 示例 -->
<carAudioConfiguration version="2">
  <zones>
    <!-- 主驾音区 -->
    <zone name="primary" isPrimary="true" audioZoneId="0">
      <volumeGroups>
        <group name="media">
          <device address="bus0_media_out">
            <context context="music"/>
            <context context="notification"/>
          </device>
        </group>
        <group name="navigation">
          <device address="bus1_nav_out">
            <context context="navigation"/>
          </device>
        </group>
        <group name="voice">
          <device address="bus2_voice_out">
            <context context="call"/>
          </device>
        </group>
      </volumeGroups>
    </zone>

    <!-- 后排音区 -->
    <zone name="rear_seat" audioZoneId="1">
      <volumeGroups>
        <group name="rear_media">
          <device address="bus3_rear_media">
            <context context="music"/>
          </device>
        </group>
      </volumeGroups>
    </zone>
  </zones>
</carAudioConfiguration>
```

### 5.5 音频焦点仲裁规则

| 请求类型 | 当前持有者 | 结果 |
|----------|-----------|------|
| 导航播报 | 音乐 | 授予焦点，音乐鸭音 |
| 电话 | 音乐 | 授予焦点，音乐暂停 |
| 音乐 | 导航 | 延迟授予，导航结束后恢复 |
| 紧急警报 | 任何 | 强制授予，其他静音 |

### 5.6 App 侧使用

```java
CarAudioManager audioMgr = (CarAudioManager) car.getCarManager(Car.AUDIO_SERVICE);

// 获取音区列表
List<Integer> zoneIds = audioMgr.getAudioZoneIds();

// 获取主驾音区的音量组
int groupCount = audioMgr.getVolumeGroupCount(PRIMARY_AUDIO_ZONE);

// 设置某音量组的音量
audioMgr.setGroupVolume(PRIMARY_AUDIO_ZONE, groupId, volume, 0);

// 获取当前音量
int currentVol = audioMgr.getGroupVolume(PRIMARY_AUDIO_ZONE, groupId);
```

---

## 六、HVAC (车载空调系统)

### 6.1 概述

HVAC（Heating, Ventilation and Air Conditioning）即暖通空调系统。在 AAOS 中，HVAC **没有独立的 Manager 类**（旧版有 `CarHvacManager`，现已废弃），而是通过 `CarPropertyManager` 读写 HVAC 相关的 Vehicle Property 来控制空调。

### 6.2 代码位置

```
# HVAC 属性定义
hardware/interfaces/automotive/vehicle/aidl/
  android/hardware/automotive/vehicle/VehicleProperty.aidl
  # 搜索 HVAC_ 开头的属性

# CarService 中的处理（旧版独立服务，新版合并到 CarPropertyService）
packages/services/Car/service/src/com/android/car/
  CarHvacService.java         # 旧版（Android 11 之前）
  CarPropertyService.java     # 新版（统一通过属性服务处理）

# App 侧 SDK
packages/services/Car/car-lib/src/android/car/hardware/
  CarPropertyManager.java
  property/VehiclePropertyIds.java    # HVAC 属性 ID

# HVAC App UI
packages/apps/Car/Hvac/              # AOSP 参考 HVAC 应用
├── src/com/android/car/hvac/
│   ├── HvacController.java          # 控制逻辑
│   ├── HvacPanelActivity.java       # UI 界面（浮层/面板）
│   └── ...
```

### 6.3 HVAC 相关属性

| 属性 ID | 名称 | 类型 | 说明 |
|---------|------|------|------|
| `HVAC_TEMPERATURE_SET` | 0x15600503 | Float | 设定温度 (°C) |
| `HVAC_TEMPERATURE_CURRENT` | 0x15600502 | Float | 当前温度 |
| `HVAC_FAN_SPEED` | 0x12400500 | Int32 | 风扇转速 (1-7) |
| `HVAC_FAN_DIRECTION` | 0x12400501 | Int32 | 吹风方向 (面/脚/除雾) |
| `HVAC_AC_ON` | 0x11200505 | Bool | A/C 开关 |
| `HVAC_AUTO_ON` | 0x11200506 | Bool | 自动模式 |
| `HVAC_RECIRC_ON` | 0x11200508 | Bool | 内循环开关 |
| `HVAC_DEFROSTER` | 0x11200504 | Bool | 除霜/除雾 |
| `HVAC_SEAT_TEMPERATURE` | 0x1540050b | Int32 | 座椅加热/通风 (-3~3) |
| `HVAC_STEERING_WHEEL_HEAT` | 0x1140050c | Int32 | 方向盘加热 |
| `HVAC_POWER_ON` | 0x11200510 | Bool | HVAC 系统电源 |
| `HVAC_DUAL_ON` | 0x11200509 | Bool | 双区同步 |

### 6.4 Area ID (分区概念)

HVAC 属性使用 **Area ID** 来区分不同座位区域：

```
┌───────────────┬───────────────┐
│ SEAT_ROW_1    │ SEAT_ROW_1    │
│ _LEFT (0x01)  │ _RIGHT (0x04) │  ← 前排（主驾/副驾）
├───────────────┼───────────────┤
│ SEAT_ROW_2    │ SEAT_ROW_2    │
│ _LEFT (0x10)  │ _RIGHT (0x40) │  ← 后排
└───────────────┴───────────────┘
```

### 6.5 App 侧使用

```java
CarPropertyManager propMgr = (CarPropertyManager) car.getCarManager(Car.PROPERTY_SERVICE);

// 读取主驾设定温度
float driverTemp = propMgr.getFloatProperty(
    VehiclePropertyIds.HVAC_TEMPERATURE_SET,
    VehicleAreaSeat.SEAT_ROW_1_LEFT   // areaId = 主驾
);

// 设置副驾温度为 23.5°C
propMgr.setFloatProperty(
    VehiclePropertyIds.HVAC_TEMPERATURE_SET,
    VehicleAreaSeat.SEAT_ROW_1_RIGHT, // areaId = 副驾
    23.5f
);

// 开启 A/C
propMgr.setBooleanProperty(
    VehiclePropertyIds.HVAC_AC_ON,
    HVAC_ALL,                         // areaId = 全车
    true
);

// 监听温度变化
propMgr.registerCallback(new CarPropertyManager.CarPropertyEventCallback() {
    @Override
    public void onChangeEvent(CarPropertyValue value) {
        float temp = (float) value.getValue();
        int area = value.getAreaId();
        Log.d(TAG, "Zone " + area + " temp changed to " + temp);
    }
    @Override
    public void onErrorEvent(int propId, int zone) {}
}, VehiclePropertyIds.HVAC_TEMPERATURE_SET, CarPropertyManager.SENSOR_RATE_ONCHANGE);
```

### 6.6 数据流

```
HVAC App UI
    │ setFloatProperty(HVAC_TEMPERATURE_SET, SEAT_ROW_1_LEFT, 24.0)
    ▼
CarPropertyManager (car-lib, App 进程)
    │ Binder IPC
    ▼
CarPropertyService (CarService 进程)
    │ VehicleHal.set()
    ▼
Vehicle HAL (VHAL 进程)
    │ 转换为 CAN 帧
    ▼
CAN Bus → 空调 ECU → 实际调节温度
```

---

## 七、CarUserManager

### 7.1 概述

CarUserManager 管理 AAOS 中的**多用户系统**。车载场景下的多用户比手机复杂：

- 驾驶员切换（座椅记忆、后视镜、音乐偏好）
- 与车钥匙 / 蓝牙设备绑定
- Guest 模式
- 后排乘客用户

### 7.2 代码位置

```
# SDK 接口
packages/services/Car/car-lib/src/android/car/user/
├── CarUserManager.java           # App 侧 API
├── UserCreationResult.java       # 创建用户结果
├── UserSwitchResult.java         # 切换用户结果
├── UserIdentificationAssociationResponse.java
└── ICarUserService.aidl          # Binder 接口

# 服务端实现
packages/services/Car/service/src/com/android/car/user/
├── CarUserService.java           # 核心服务实现
├── UserHandleHelper.java
├── InitialUserSetter.java        # 开机初始用户选择
├── UserPreCreator.java           # 用户预创建（加速切换）
└── ExperimentalCarUserManager.java

# VHAL 中的用户相关属性
hardware/interfaces/automotive/vehicle/aidl/
  # INITIAL_USER_INFO
  # SWITCH_USER
  # CREATE_USER
  # REMOVE_USER
  # USER_IDENTIFICATION_ASSOCIATION
```

### 7.3 架构

```
CarSettings App                CarUserManager
      │                             │
      ├── switchUser(userId) ───────┤
      │                             │ Binder
      ▼                             ▼
                              CarUserService
                                    │
                     ┌──────────────┼──────────────┐
                     ▼              ▼              ▼
              UserManager     VehicleHal      ActivityManager
           (Android 原生)   (通知 VHAL)     (切换前台用户)
                                    │
                                    ▼
                              Vehicle HAL
                                    │
                                    ▼
                        车辆 ECU（座椅记忆等）
```

### 7.4 用户生命周期

```
开机
 │
 ▼
VHAL 发送 INITIAL_USER_INFO 请求
 │
 ▼
CarUserService 决定初始用户
 │  (根据上次驾驶员 / 车钥匙 / 默认值)
 ▼
启动初始用户 → 加载个人设置
 │
 ├── 用户手动切换 → SWITCH_USER → VHAL 通知 ECU
 ├── 创建新用户   → CREATE_USER → VHAL 同步
 └── 删除用户     → REMOVE_USER → VHAL 清理
```

### 7.5 App 侧使用

```java
CarUserManager userMgr = (CarUserManager) car.getCarManager(Car.CAR_USER_SERVICE);

// 切换用户
UserSwitchResult result = userMgr.switchUser(userId).get();
if (result.isSuccess()) {
    Log.d(TAG, "User switched successfully");
}

// 监听用户切换
userMgr.addListener(Runnable::run, (event) -> {
    if (event.getEventType() == CarUserManager.USER_LIFECYCLE_EVENT_TYPE_SWITCHING) {
        int prevUser = event.getPreviousUserId();
        int newUser = event.getUserId();
        Log.d(TAG, "Switching from user " + prevUser + " to " + newUser);
    }
});

// 用户生命周期事件类型
// USER_LIFECYCLE_EVENT_TYPE_STARTING      - 用户开始启动
// USER_LIFECYCLE_EVENT_TYPE_SWITCHING      - 正在切换
// USER_LIFECYCLE_EVENT_TYPE_UNLOCKING      - 解锁中
// USER_LIFECYCLE_EVENT_TYPE_UNLOCKED       - 已解锁
// USER_LIFECYCLE_EVENT_TYPE_STOPPING       - 停止中
// USER_LIFECYCLE_EVENT_TYPE_STOPPED        - 已停止
```

### 7.6 与 Vehicle HAL 的交互属性

| 属性 | 说明 |
|------|------|
| `INITIAL_USER_INFO` | 开机时 VHAL 告知推荐的初始用户 |
| `SWITCH_USER` | 用户切换通知（Android ↔ VHAL 双向） |
| `CREATE_USER` | 创建用户时通知 VHAL |
| `REMOVE_USER` | 删除用户时通知 VHAL |
| `USER_IDENTIFICATION_ASSOCIATION` | 用户与车钥匙/蓝牙设备的绑定关系 |

---

## 八、VehiclePropertyManager (CarPropertyManager)

### 8.1 概述

`CarPropertyManager`（对应旧名 `VehiclePropertyManager`）是 AAOS 中**最核心的 API**，提供对车辆所有属性的统一读写接口。车速、挡位、油量、空调、车灯、车门……所有车辆信号都抽象为 **Property**。

### 8.2 代码位置

```
# SDK 接口 (App 侧)
packages/services/Car/car-lib/src/android/car/hardware/
├── CarPropertyManager.java       # 核心 API 类
├── CarPropertyValue.java         # 属性值 (泛型，含 timestamp, areaId, status)
├── CarPropertyConfig.java        # 属性配置（类型、范围、权限）
├── property/
│   ├── VehiclePropertyIds.java   # 所有属性 ID 常量 (数百个)
│   ├── AreaIdConfig.java
│   └── CarPropertyHelper.java
└── ICarProperty.aidl              # Binder 接口

# 服务端
packages/services/Car/service/src/com/android/car/
├── CarPropertyService.java        # 服务实现
├── hal/
│   ├── VehicleHal.java            # HAL 通信层
│   ├── PropertyHalService.java    # 属性 HAL 服务
│   ├── HalPropConfig.java
│   └── HalPropValue.java

# VHAL 属性定义
hardware/interfaces/automotive/vehicle/aidl/
  android/hardware/automotive/vehicle/
  ├── VehicleProperty.aidl         # 属性枚举（权威定义）
  ├── VehiclePropertyType.aidl     # 类型：INT32, FLOAT, BOOLEAN, STRING...
  ├── VehicleArea.aidl             # 区域类型：GLOBAL, SEAT, DOOR, WHEEL...
  ├── VehiclePropertyAccess.aidl   # 访问权限：READ, WRITE, READ_WRITE
  └── VehiclePropertyChangeMode.aidl # 变化模式：STATIC, ON_CHANGE, CONTINUOUS
```

### 8.3 属性模型

每个 Vehicle Property 由以下要素组成：

```
Property ID (int32)
  ├── 高 4 位: VehiclePropertyGroup (SYSTEM / VENDOR)
  ├── 次 4 位: VehiclePropertyType  (INT32 / FLOAT / BOOLEAN / STRING / MIXED ...)
  ├── 次 4 位: VehicleArea          (GLOBAL / SEAT / DOOR / WHEEL / WINDOW ...)
  └── 低 16 位: 属性编号

例: HVAC_TEMPERATURE_SET = 0x15600503
     1 = SYSTEM
      5 = FLOAT
       6 = SEAT (按座位分区)
        00503 = 编号
```

### 8.4 常用属性速查

| 属性 | ID | 类型 | 区域 | 访问 | 变化模式 |
|------|----|------|------|------|----------|
| `PERF_VEHICLE_SPEED` | 0x11600207 | FLOAT | GLOBAL | READ | CONTINUOUS |
| `GEAR_SELECTION` | 0x11400400 | INT32 | GLOBAL | READ | ON_CHANGE |
| `CURRENT_GEAR` | 0x11400401 | INT32 | GLOBAL | READ | ON_CHANGE |
| `FUEL_LEVEL` | 0x11600307 | FLOAT | GLOBAL | READ | CONTINUOUS |
| `EV_BATTERY_LEVEL` | 0x11600309 | FLOAT | GLOBAL | READ | CONTINUOUS |
| `TURN_SIGNAL_STATE` | 0x11400402 | INT32 | GLOBAL | READ | ON_CHANGE |
| `NIGHT_MODE` | 0x11200407 | BOOL | GLOBAL | READ | ON_CHANGE |
| `DOOR_LOCK` | 0x11200b05 | BOOL | DOOR | READ_WRITE | ON_CHANGE |
| `TIRE_PRESSURE` | 0x11600306 | FLOAT | WHEEL | READ | CONTINUOUS |
| `PARKING_BRAKE_ON` | 0x11200402 | BOOL | GLOBAL | READ | ON_CHANGE |
| `EV_CHARGE_PERCENT_LIMIT` | 0x11600310 | FLOAT | GLOBAL | READ_WRITE | ON_CHANGE |
| `EV_CHARGE_STATE` | 0x11400311 | INT32 | GLOBAL | READ | ON_CHANGE |
| `EV_CHARGE_CURRENT_DRAW_LIMIT` | 0x11600312 | FLOAT | GLOBAL | READ_WRITE | ON_CHANGE |
| `RANGE_REMAINING` | 0x11600308 | FLOAT | GLOBAL | READ_WRITE | CONTINUOUS |
| `EV_STOPPING_MODE` | 0x11400320 | INT32 | GLOBAL | READ_WRITE | ON_CHANGE |

### 8.5 架构

```
App 进程                                 CarService 进程
┌─────────────────────┐          ┌───────────────────────────┐
│ CarPropertyManager  │          │ CarPropertyService        │
│  ├── getProperty()  │─Binder──▶│  ├── getProperty()        │
│  ├── setProperty()  │─Binder──▶│  ├── setProperty()        │
│  └── registerCb()   │─Binder──▶│  └── registerListener()   │
└─────────────────────┘          │          │                 │
                                 │  PropertyHalService        │
                                 │          │                 │
                                 │  VehicleHal                │
                                 └──────────┬────────────────┘
                                            │ AIDL Binder
                                 ┌──────────▼────────────────┐
                                 │ IVehicle (VHAL 进程)       │
                                 │  ├── getValues()           │
                                 │  ├── setValues()           │
                                 │  └── subscribe()           │
                                 └───────────────────────────┘
```

### 8.6 完整使用示例

```java
CarPropertyManager pm = (CarPropertyManager) car.getCarManager(Car.PROPERTY_SERVICE);

// ===== 读取 =====
// 读取车速
float speed = pm.getFloatProperty(VehiclePropertyIds.PERF_VEHICLE_SPEED, 0 /* areaId */);

// 读取挡位
int gear = pm.getIntProperty(VehiclePropertyIds.GEAR_SELECTION, 0);

// 读取完整属性值（含时间戳、状态）
CarPropertyValue<Float> speedVal = pm.getProperty(
    Float.class, VehiclePropertyIds.PERF_VEHICLE_SPEED, 0);
float val = speedVal.getValue();
long timestamp = speedVal.getTimestamp(); // 纳秒
int status = speedVal.getStatus();        // AVAILABLE / UNAVAILABLE / ERROR

// ===== 写入 =====
pm.setFloatProperty(VehiclePropertyIds.HVAC_TEMPERATURE_SET,
    VehicleAreaSeat.SEAT_ROW_1_LEFT, 24.0f);

// ===== 订阅（传统方式，仍受支持）=====
pm.registerCallback(new CarPropertyManager.CarPropertyEventCallback() {
    @Override
    public void onChangeEvent(CarPropertyValue value) {
        int propId = value.getPropertyId();
        Object val = value.getValue();
        Log.d(TAG, "Property " + propId + " = " + val);
    }

    @Override
    public void onErrorEvent(int propId, int zone) {
        Log.e(TAG, "Error on property " + propId);
    }
}, VehiclePropertyIds.PERF_VEHICLE_SPEED, CarPropertyManager.SENSOR_RATE_NORMAL);

// ===== 订阅（Android 16 新方式：Subscription Builder，支持精细化采样率控制）=====
pm.subscribePropertyEvents(
    List.of(
        new CarPropertyManager.Subscription.Builder(VehiclePropertyIds.PERF_VEHICLE_SPEED)
            .setUpdateRateHz(10f)          // 10 Hz 采样
            .setVariableUpdateRateEnabled(true) // 允许 VHAL 动态调整
            .build(),
        new CarPropertyManager.Subscription.Builder(VehiclePropertyIds.EV_BATTERY_LEVEL)
            .setUpdateRateHz(1f)
            .build()
    ),
    Runnable::run,
    new CarPropertyManager.CarPropertyEventCallback() {
        @Override
        public void onChangeEvent(CarPropertyValue value) { /* ... */ }
        @Override
        public void onErrorEvent(int propId, int zone) { /* ... */ }
    }
);

// ===== 查询属性配置 =====
CarPropertyConfig<?> config = pm.getCarPropertyConfig(VehiclePropertyIds.HVAC_TEMPERATURE_SET);
// 获取温度范围
for (int areaId : config.getAreaIds()) {
    float minTemp = config.getMinValue(areaId); // e.g., 16.0
    float maxTemp = config.getMaxValue(areaId); // e.g., 32.0
}

// ===== 获取所有支持的属性 =====
List<CarPropertyConfig> allProps = pm.getPropertyList();
for (CarPropertyConfig cfg : allProps) {
    Log.d(TAG, "Supported property: " + cfg.getPropertyId());
}
```

### 8.7 Vendor 自定义属性

OEM 可以定义私有属性（不在标准列表中）：

```
Property ID 高 4 位 = 0x2 (VENDOR)

例: 0x21401234
     2 = VENDOR group
      1 = INT32
       4 = GLOBAL
        01234 = OEM 自定义编号
```

---

## 九、Vehicle HAL (VHAL)

### 9.1 概述

Vehicle HAL 是 AAOS 架构中**最底层的抽象接口**，定义了 Android 与车辆硬件之间的通信协议。VHAL 运行在独立进程中（vendor 分区），由 OEM 实现。

**核心职责：**
- 将车辆总线信号（CAN/LIN/SOME-IP/DoIP）转换为标准 Vehicle Property
- 向 CarService 上报属性变化
- 接收 CarService 的属性写入请求并转发给 ECU

### 9.2 代码位置

```
# AIDL 接口定义 (Android 13+，Android 16 对应 aidl-v4)
hardware/interfaces/automotive/vehicle/aidl/
├── android/hardware/automotive/vehicle/
│   ├── IVehicle.aidl                # 核心 HAL 接口
│   ├── IVehicleCallback.aidl        # 回调接口（属性变化通知）
│   ├── VehicleProperty.aidl         # 所有标准属性枚举（Android 16 新增 EV/SDV 属性）
│   ├── VehiclePropertyType.aidl     # 属性值类型
│   ├── VehiclePropertyAccess.aidl   # 读写权限
│   ├── VehiclePropertyChangeMode.aidl # 变化模式
│   ├── VehiclePropValue.aidl        # 属性值结构体
│   ├── VehiclePropConfig.aidl       # 属性配置（aidl-v3+ 新增 AreaIdConfig 精细范围）
│   ├── VehicleArea.aidl             # 区域类型
│   ├── StatusCode.aidl              # 返回状态码
│   └── ...
├── vts/                             # VTS 测试
└── default/                         # 参考默认实现

# AIDL 版本与 Android 版本对应关系
# aidl-v1 → Android 13 (T)
# aidl-v2 → Android 14 (U)
# aidl-v3 → Android 15 (V)  新增：MinMaxSupportedValue, SupportedValuesListForAreaId
# aidl-v4 → Android 16 (B)  新增：变量采样率、SDV 扩展属性、OBD 增强

# 注意：HIDL 接口 (2.0) 已于 Android 16 正式移除，不再受支持
# hardware/interfaces/automotive/vehicle/2.0/ 已废弃

# 参考实现 (模拟器用)
hardware/interfaces/automotive/vehicle/aidl/default/
├── DefaultVehicleHal.cpp            # 默认 HAL 实现
├── FakeVehicleHardware.cpp          # 模拟器用假数据源
├── VehiclePropertyStore.cpp         # 属性存储
└── ...

# OEM 具体实现 (通常在 vendor 仓库)
vendor/<oem>/interfaces/automotive/vehicle/
└── <version>/
    └── <OemVehicleHal>.cpp          # OEM 私有实现
```

### 9.3 AIDL 接口详解 (Android 13+ / aidl-v1 起，Android 16 为 aidl-v4)

```aidl
// IVehicle.aidl
interface IVehicle {
    // 获取所有支持的属性配置
    VehiclePropConfigs getAllPropConfigs();

    // 获取指定属性的配置
    VehiclePropConfigs getPropConfigs(in int[] props);

    // 批量读取属性值
    GetValueResults getValues(IVehicleCallback callback, in GetValueRequests requests);

    // 批量写入属性值
    SetValueResults setValues(IVehicleCallback callback, in SetValueRequests requests);

    // 订阅属性变化
    void subscribe(IVehicleCallback callback, in SubscribeOptions[] options, int maxSharedMemoryFileCount);

    // 取消订阅
    void unsubscribe(IVehicleCallback callback, in int[] propIds);
}

// IVehicleCallback.aidl
interface IVehicleCallback {
    // 属性值变化回调
    oneway void onPropertyEvent(in VehiclePropValues propValues, int sharedMemoryFileCount);

    // 属性设置错误回调
    oneway void onPropertySetError(in VehiclePropErrors errors);

    // getValues 异步结果
    oneway void onGetValues(in GetValueResults responses);

    // setValues 异步结果
    oneway void onSetValues(in SetValueResults responses);
}
```

### 9.4 属性值结构

```aidl
// VehiclePropValue.aidl
parcelable VehiclePropValue {
    int prop;            // 属性 ID
    int areaId;          // 区域 ID
    long timestamp;      // 纳秒时间戳
    VehiclePropertyStatus status;  // AVAILABLE / UNAVAILABLE / ERROR
    RawPropValues value;  // 实际值
}

parcelable RawPropValues {
    int[] int32Values;
    long[] int64Values;
    float[] floatValues;
    byte[] byteValues;
    @utf8InCpp String stringValue;
}
```

### 9.5 架构

```
CarService 进程
┌────────────────────────┐
│ VehicleHal.java        │
│   ├── get()            │
│   ├── set()            │
│   └── subscribe()       │
└───────────┬────────────┘
            │ AIDL Binder (跨进程)
┌───────────▼────────────┐
│ VHAL 进程 (vendor)     │
│ ┌────────────────────┐ │
│ │ DefaultVehicleHal  │ │  ← HAL 层框架
│ │ (或 OEM 实现)       │ │
│ ├────────────────────┤ │
│ │ VehicleHardware    │ │  ← 硬件抽象接口
│ │ (OEM 实现)          │ │
│ ├────────────────────┤ │
│ │ CAN Transceiver    │ │  ← 总线驱动
│ │ SocketCAN / j1939  │ │
│ │ SOME/IP gateway    │ │
│ └─────────┬──────────┘ │
└───────────┼────────────┘
            │
     ┌──────▼──────┐
     │  CAN Bus /  │
     │  LIN / ETH  │
     │  (车辆总线)  │
     └──────┬──────┘
            │
   ┌────────▼────────┐
   │  ECU (引擎/空调/ │
   │  车身/底盘...)    │
   └─────────────────┘
```

### 9.6 OEM 实现要点

OEM 需要实现 `IVehicleHardware` 接口（C++）：

```cpp
// 简化示例
class OemVehicleHardware : public IVehicleHardware {
public:
    // 获取支持的属性配置
    std::vector<VehiclePropConfig> getAllPropertyConfigs() override {
        return {
            // 车速
            {.prop = toInt(VehicleProperty::PERF_VEHICLE_SPEED),
             .access = VehiclePropertyAccess::READ,
             .changeMode = VehiclePropertyChangeMode::CONTINUOUS,
             .minSampleRate = 1.0f,
             .maxSampleRate = 100.0f},
            // 挡位
            {.prop = toInt(VehicleProperty::GEAR_SELECTION),
             .access = VehiclePropertyAccess::READ,
             .changeMode = VehiclePropertyChangeMode::ON_CHANGE},
            // HVAC 温度
            {.prop = toInt(VehicleProperty::HVAC_TEMPERATURE_SET),
             .access = VehiclePropertyAccess::READ_WRITE,
             .changeMode = VehiclePropertyChangeMode::ON_CHANGE,
             .areaConfigs = {
                 {.areaId = SEAT_ROW_1_LEFT, .minFloatValue = 16.0, .maxFloatValue = 32.0},
                 {.areaId = SEAT_ROW_1_RIGHT, .minFloatValue = 16.0, .maxFloatValue = 32.0},
             }},
            // ... 更多属性
        };
    }

    // 读取属性
    StatusCode getValues(std::vector<GetValueRequest>& requests,
                         std::vector<GetValueResult>& results) override {
        for (auto& req : requests) {
            switch (req.prop.prop) {
                case toInt(VehicleProperty::PERF_VEHICLE_SPEED):
                    // 从 CAN 总线读取车速
                    results.push_back({.requestId = req.requestId,
                                       .status = StatusCode::OK,
                                       .prop = {.floatValues = {readSpeedFromCan()}}});
                    break;
                // ...
            }
        }
        return StatusCode::OK;
    }

    // 写入属性
    StatusCode setValues(std::vector<SetValueRequest>& requests,
                         std::vector<SetValueResult>& results) override {
        for (auto& req : requests) {
            switch (req.value.prop) {
                case toInt(VehicleProperty::HVAC_TEMPERATURE_SET):
                    // 发送 CAN 帧给空调 ECU
                    sendTempToCan(req.value.areaId, req.value.value.floatValues[0]);
                    break;
                // ...
            }
        }
        return StatusCode::OK;
    }

    // 订阅属性变化
    StatusCode subscribe(SubscribeOptions options) override {
        // 注册 CAN 信号监听
        registerCanSignalListener(options.propId, [this](auto propValue) {
            // 上报变化给 CarService
            mCallback->onPropertyEvent({propValue});
        });
        return StatusCode::OK;
    }
};
```

### 9.7 HIDL 与 AIDL 对比

> ⚠️ **Android 16 重要变更**：VHAL HIDL 接口 (2.0) 已于 Android 16 **正式移除**。所有 OEM 必须迁移到 AIDL VHAL。新设备须使用 aidl-v4 或更高版本。

| 方面 | HIDL (Android 11/12) | AIDL (Android 13 ~ 16+) |
|------|---------------------|-------------------------|
| 接口文件 | `.hal` | `.aidl` |
| 传输 | hwbinder | binder (vndbinder) |
| 异步 | 回调 | 回调 + 共享内存 |
| 性能 | 一般 | 优化（批量操作、共享内存大负载） |
| 版本 | 2.0（已移除） | aidl-v1 (T) / v2 (U) / v3 (V) / v4 (B) |
| 状态 | **Android 16 起已移除** | 当前唯一受支持接口 |

### 9.8 调试命令

```bash
# 列出 VHAL 支持的所有属性
adb shell dumpsys car_service --hal

# 通过命令行读取属性
adb shell cmd car_service get-property 0x11600207  # PERF_VEHICLE_SPEED

# 通过命令行设置属性
adb shell cmd car_service set-property 0x15600503 -a 0x01 -f 24.0  # 设温度

# 查看 VHAL 进程
adb shell ps -A | grep vehicle

# VHAL 日志
adb logcat -s VehicleHal CarPropertyService

# 注入属性（测试用, 模拟器）
adb shell cmd car_service inject-property 0x11600207 -f 60.0
```

---

## 十、各模块代码位置速查表

| 模块 | 代码路径 | 说明 |
|------|---------|------|
| **car-lib (SDK)** | `packages/services/Car/car-lib/` | App 侧 `android.car.*` API |
| **CarService** | `packages/services/Car/service/` | 核心系统服务 |
| **CarPropertyService** | `.../service/src/com/android/car/CarPropertyService.java` | 车辆属性服务端 |
| **CarAudioService** | `.../service/src/com/android/car/audio/` | 音频服务 |
| **CarUserService** | `.../service/src/com/android/car/user/` | 用户管理服务 |
| **CarHvacService** | `.../service/src/com/android/car/CarHvacService.java` | HVAC 服务（旧版） |
| **CarPowerMgmt** | `.../service/src/com/android/car/CarPowerManagementService.java` | 电源管理 |
| **CarInputService** | `.../service/src/com/android/car/CarInputService.java` | 输入事件 |
| **CarWatchdog** | `.../service/src/com/android/car/watchdog/` | 看门狗 |
| **CarUxRestrictions** | `.../service/src/com/android/car/CarUxRestrictionsManagerService.java` | 驾驶分心限制 |
| **VHAL 接口 (AIDL)** | `hardware/interfaces/automotive/vehicle/aidl/` | HAL 接口定义 |
| **VHAL 接口 (HIDL)** | `hardware/interfaces/automotive/vehicle/2.0/` | 旧版 HAL |
| **VHAL 参考实现** | `hardware/interfaces/automotive/vehicle/aidl/default/` | 模拟器默认实现 |
| **VTS 测试** | `hardware/interfaces/automotive/vehicle/aidl/vts/` | 供应商测试 |
| **CTS 测试** | `packages/services/Car/tests/` | 兼容性测试 |
| **CarLauncher** | `packages/apps/Car/Launcher/` | 车载桌面 |
| **CarSettings** | `packages/apps/Car/Settings/` | 车载设置 |
| **CarSystemUI** | `frameworks/base/packages/CarSystemUI/` | 车载状态栏/导航栏 |
| **HVAC App** | `packages/apps/Car/Hvac/` | 空调控制 UI |
| **Audio HAL** | `hardware/interfaces/audio/` | 音频 HAL |
| **EVS HAL** | `hardware/interfaces/automotive/evs/` | 外部摄像头 HAL |
| **AudioControl HAL** | `hardware/interfaces/automotive/audiocontrol/` | 音频控制 HAL |

---

## 十一、典型数据流示例

### 11.1 读取车速 (App → ECU)

```
App: carPropertyManager.getFloatProperty(PERF_VEHICLE_SPEED, 0)
 │
 ▼ Binder IPC
CarPropertyService.getProperty()
 │
 ▼
PropertyHalService.getProperty()
 │
 ▼
VehicleHal.get()
 │
 ▼ AIDL Binder (跨进程到 vendor)
IVehicle.getValues()
 │
 ▼
OemVehicleHardware.getValues()
 │
 ▼ SocketCAN / SPI
CAN Bus → 读取车速 CAN 信号 (e.g., CAN ID 0x123, byte[2:3])
 │
 ▼ 解析 + 单位转换 (km/h → m/s)
返回 VehiclePropValue { floatValues = [16.67] }  // 60 km/h
 │
 ▼ 逐层返回
App 收到 speed = 16.67f (m/s)
```

### 11.2 设置空调温度 (App → ECU)

```
App: carPropertyManager.setFloatProperty(HVAC_TEMPERATURE_SET, SEAT_ROW_1_LEFT, 24.0f)
 │
 ▼ Binder
CarPropertyService.setProperty()
 │
 ▼ 权限检查 (android.car.permission.CONTROL_CAR_CLIMATE)
PropertyHalService.setProperty()
 │
 ▼
VehicleHal.set()
 │
 ▼ AIDL Binder
IVehicle.setValues()
 │
 ▼
OemVehicleHardware.setValues()
 │
 ▼ 构造 CAN 帧: ID=0x456, data=[0x18, 0x00, ...]  (24°C 编码)
CAN Bus → 空调 ECU 接收 → 调节温度
 │
 ▼ ECU 回复新状态 CAN 信号
OemVehicleHardware 收到变化 → IVehicleCallback.onPropertyEvent()
 │
 ▼
CarPropertyService → 通知所有注册了该属性的 App
 │
 ▼
App 的 onChangeEvent() 回调被触发
```

### 11.3 驾驶状态变化 → UX 限制

```
车辆开始行驶 → CAN 信号: 车速 > 0 + 挡位 = D
 │
 ▼ VHAL 上报
PERF_VEHICLE_SPEED 变化 + GEAR_SELECTION = DRIVE
 │
 ▼
CarDrivingStateService 计算驾驶状态 = MOVING
 │
 ▼
CarUxRestrictionsManagerService 生成限制:
  - isRequiresDistractionOptimization = true
  - NO_VIDEO | NO_TEXT_INPUT | LIMIT_STRING_LENGTH
 │
 ▼ 通知所有注册的 App
App 收到 UxRestrictions → 隐藏视频、简化列表、禁止打字
```

---

---

## 十二、Google SDV 战略与 AAOS 演进

### 12.1 SDV 是什么

**软件定义汽车（Software-Defined Vehicle, SDV）** 指车辆的功能、行为和用户体验主要由软件决定和演进，而非硬件。与传统汽车相比：

| 方面 | 传统汽车 | SDV |
|------|----------|-----|
| 功能定义 | 硬件 ECU 固定 | 软件 OTA 升级 |
| ECU 数量 | 70~150+ 个分散 ECU | 少量集中式域控制器（Domain Controller）|
| 数据架构 | CAN 为主 | Ethernet / SOME-IP / 100BASE-T1 |
| 更新方式 | 4S 店刺线刷写 | OTA 无线升级 |
| 开发周期 | 年频定型 | 软件迭代，持续交付 |

### 12.2 Google 的 SDV 战略

Google 于 2023~2024 年明确将 AAOS 定位为 **SDV 全车软件平台**，主要方向：

#### 12.2.1 AAOS 向多域扩展

```
传统 AAOS 覆盖范围:
  • 中控主机 IVI

Android 16 起 SDV 目标覆盖范围:
  • 中控主机 IVI
  • 座舱域控制器 (Cockpit Domain Controller)
  • 仪表盘背板平台 (与表层 QNX 协同工作)
  • 车内互联（TCU，配合 Minimal Telephony HAL）
  • ADAS 敏感层周边辅助处理（非安全功能性部分）
```

#### 12.2.2 核心公开协议与合作

| 伙伴 | 内容 |
|------|------|
| **Qualcomm** | Snapdragon Ride Flex SoC 全面支持 AAOS 多域，2025+ 量产 |
| **Renesas** | R-Car S4 + AAOS 座舱域控制器参考平台 |
| **NXP** | S32G + AAOS 几个山1开源移植方案 |
| **GM / Chevrolet** | Ultifi 软件平台采用 AAOS，2024+ 车型支持车载应用商店 |
| **Volvo / Polestar** | 与 Google 深度合作，SCS (Software Defined) 路线基于 AAOS |
| **Renault / AMPERE** | 全系列 SDV 平台基于 AAOS |
| **大众 / 汦众** | 外操软件定义车融入 AAOS |

### 12.3 Android 14~16 关键 SDV 特性清单

| Android 版本 | 特性 | SDV 意义 |
|-----------|------|----------|
| **Android 14 (U)** | `CarRemoteAccessManager` | 车辆熔下唤醒，支持远程任务下发（OTA 基础） |
| **Android 14 (U)** | `CarOccupantConnectionManager` | 跨乘员区数据通道，支持差异制乘客体验 |
| **Android 14 (U)** | VHAL aidl-v2 | 自定义 Vendor 属性增强，截面打印 API |
| **Android 15 (V)** | VHAL aidl-v3 | `MinMaxSupportedValue` / `SupportedValuesListForAreaId`，属性动态范围 |
| **Android 15 (V)** | Scalable UI 初版 | 多屏布局可配置，支持座舱多屏 SDV 场景 |
| **Android 16 (B)** | VHAL aidl-v4 | 变量采样率、SDV 扩展属性、OBD 增强 |
| **Android 16 (B)** | 动态车辆属性配置 | 支持运行时动态修改 min/max，适应 SDV 灵活配置 |
| **Android 16 (B)** | 第三方可访车辆属性 | 导航/语音/天气/行车状态属性对第三方 App 开放 |
| **Android 16 (B)** | Minimal Telephony HAL | TCU 扛载，车戚内联一体化 |
| **Android 16 (B)** | HIDL VHAL 移除 | 强制全面切换至 AIDL，为下一代模块化 OTA 铺路 |

### 12.4 SDV 架构：多域 + AAOS 塔报

```
整车 SDV 软件层次
┌───────────────────────────────────────────────────────────────┐
│  应用层: CarLauncher  OEM Apps  Google GAS  第三方 App       │
├───────────────────────────────────────────────────────────────┤
│  AAOS (座舱域 / IVI 域):                                           │
│   CarService  · VHAL  · CarAudioService  · CarPowerMgmt         │
│   CarRemoteAccess  ·  CarOccupantConnection  ·  Scalable UI       │
├───────────────────────────────────────────────────────────────┤
│  Hypervisor (Gunyah / QNX Hypervisor / Xen):                        │
│   座舱域 (AAOS)  |  安全域 (QNX / RTOS)  |  ADAS 域 (Linux) │
├───────────────────────────────────────────────────────────────┤
│  SoC (Qualcomm SA8775P / Renesas R-Car S4 / …)                     │
│  Ethernet (SOME-IP)  ·  CAN FD  ·  LIN  ·  FlexRay               │
│  ECU / Gateway / TCU / ADAS 处理器                              │
└───────────────────────────────────────────────────────────────┘
```

### 12.5 OTA 升级架构

SDV 的核心能力之一是 **全车 OTA**。AAOS 在 Android 14~16 持续完善相关模块：

```
OTA 升级流程
                          ╒════════════════╕
                          ║ OEM 云端服务器   ║
                          ║  (OTA Package)    ║
                          ╚════════════════╝
                                   │
               ┌──────────────┴─────────────┐
               │ CarRemoteAccessManager    │  ← 车辆熟眠中也可唤醒接收
               │ (AAOS 远程唯醒层)         │
               └──────────────┬─────────────┘
                              │
          ┌───────────────┴───────────────┐
          │ Android Update Engine          │  ← A/B 分区升级
          │ + Updatable APEX Modules       │  ← 模块化假升级不封锁设备
          └───────────────┬───────────────┘
                         │
       ┌───────────────┴───────────────┐
       │ VHAL OTA (aidl-v4)               │  ← ECU 固件升级通道解耦
       │ ECU Flash (via MCU gateway)      │
       └───────────────────────────────┘
```

**AAOS OTA 升级层次：**

| 层次 | 机制 | 说明 |
|------|------|---------|
| Framework / Mainline | APEX 模块升级 | CarService、Media、Connectivity 都可独立升级 |
| OS 镜像 | A/B 无缝升级 | 升级期间车辆可正常使用，升级失败可回滚 |
| VHAL / HAL | HAL 动态加载 + AIDL 稳定接口 | AIDL 稳定性保证前向兼容 |
| ECU 固件 | VHAL 中继传 OTA 包 | 通过 OEM 网关 ECU 刷写（非 Android 原生） |

### 12.6 CarRemoteAccessManager 详解

`CarRemoteAccessManager` 是 Android 14 引入的 SDV 关键 组件，允许在**车辆熟眠状态**下远程唤醒并执行任务。

```java
CarRemoteAccessManager remoteAccessMgr =
    (CarRemoteAccessManager) car.getCarManager(Car.CAR_REMOTE_ACCESS_SERVICE);

// 注册远程任务处理器
remoteAccessMgr.setRemoteTaskClient(
    Executors.newSingleThreadExecutor(),
    new CarRemoteAccessManager.RemoteTaskClient() {
        @Override
        public void onRegistrationUpdated(RemoteTaskClientRegistrationInfo info) {
            // 获取车辆唐 (vehicleId)、提供商 ID、服务 ID
            // 上报至 OEM 云端，云端日后用于下发远程任务
            Log.d(TAG, "VehicleId: " + info.getVehicleId());
        }

        @Override
        public void onRemoteTaskRequested(String taskId, byte[] data) {
            // 车辆已被唤醒，在 allowedTimeMs 内完成任务
            // 典型任务: 下载 OTA 包、剔除軿霸、推送车辆统计数据
            processRemoteTask(taskId, data);
        }

        @Override
        public void onShutdownStarting(CarRemoteAccessManager.CompletableRemoteTaskFuture future) {
            // 车辆即将重新进入熟眠，尽快完成正在执行的任务
            future.complete();
        }
    }
);
```

**底层流程：**

```
OEM 云端
 │ 下发远程任务指令 (包含 vehicleId)
 ▼
TCU (车载通信单元)
 │ 通过准运建通道将指令发至 ECU
 ▼
ECU / 电源管理模块
 │ 为 AAOS SoC 供电并唤醒
 ▼
AAOS 启动 (CPM 进入 ON_REMOTE_ACCESS 状态)
 │ 通知 CarRemoteAccessManager
 ▼
App RemoteTaskClient.onRemoteTaskRequested() 被回调
 │ 执行任务 (下载 / 清洗 / 上报数据)
 ▼
任务完成 → CPM 进入省电狶态
```

### 12.7 Scalable UI (Android 16 正式引入)

**Scalable UI** 是 AAOS 在 Android 15~16 周期引入的**可配置式车辆窗口管理框架**，为 OEM 提供灵活的多屏布局能力，是 SDV 多屏座舱的锁层技术。

```
典型 SDV 座舱布局 (三屏一体)
┌──────────────┬─────────────────────────┬──────────────┐
│ 仪表盘         │ 中控屏 主显示区          │ 副驾/后排屏     │
│ Instrument      │ (CarLauncher / 导航)   │ (RSE)           │
│ Cluster         │                         │                 │
│ (QNX 或 AAOS)   │                         │                 │
└──────────────┴────────├─────────────────┴──────────────┘
                          │ 辅助显示区 (空调/音乐)│
                          └─────────────────┘
    └──────────── 全部由 Scalable UI 统一管理窗口分配 + 布局 ────────────┘
```

**Scalable UI 关键能力：**
- OEM 通过 XML 配置定义屏幕分区、应用占相关系，无需修改框架代码
- 支持动态调整布局（如导航全屏 / 小窗模式切换）
- 多用户、多乘客区屏幕独立管理
- 配合 `CarOccupantZoneManager` 实现不同乘客区独立操作界面

### 12.8 Google 车辆服务 (GAS) 与 AAOS 的关系

**Google Automotive Services (GAS)** 是 Google 面向 OEM 授权的应用集，运行在 AAOS 之上。12.8 详解 GAS 与 SDV 的关系：

```
GAS 包含的车辆内置 App：
  • Google Maps for Automotive    ← 导航 (SDV 地图服务)
  • Google Assistant              ← 语音交互 (SDV 人车交互层)
  • Google Play Store (Automotive) ← 应用分发 (SDV 应用生态)
  • Google Play Protect            ← 安全 (SDV 安全防护层)
  • Cast / YouTube Music           ← 娱乐 (SDV 应用层)

GAS 与 AAOS 的层位关系：
┌───────────────────────────┐
│ GAS (Google 授权应用集)      │ ← 可选，需 OEM 与 Google 签订 CDD
├───────────────────────────┤
│ AAOS (AOSP 开源基础)          │ ← 免费开源，平台层
├───────────────────────────┤
│ OEM 定制层 + Vendor HAL      │ ← OEM 自研，负责硬件适配
└───────────────────────────┘
```

### 12.9 SDV 开发资源速查

| 资源 | 地址 |
|------|------|
| AOSP Automotive 文档 | https://source.android.com/docs/automotive |
| AAOS 发布记录 | https://source.android.com/docs/automotive/start/releases |
| VHAL 属性和 API 参考 | https://developer.android.com/reference/android/car/VehiclePropertyIds |
| Car App Library | https://developer.android.com/reference/androidx/car/app/package-summary |
| AAOS 公开 Issue Tracker | https://issuetracker.google.com/components/191553 |
| CarRemoteAccessManager API | https://developer.android.com/reference/android/car/remoteaccess/CarRemoteAccessManager |
| Scalable UI 文档 | https://source.android.com/docs/automotive/displays/scalable-ui |

---

> **文档版本**: v2.0  
> **适用 Android 版本**: Android 13 (T) - Android 16 (Baklava / API 36)  
> **最后更新**: 2026-03-27
