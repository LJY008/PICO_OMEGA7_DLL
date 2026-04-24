# ChaiTeaLib — CHAI3D 触觉设备 Unity 插件

## 项目简介

本项目基于开源项目 **[ChaiTea](https://github.com/Mads-MA/ChaiTea)**（由 [Mads Mørch Antonsen](https://github.com/Mads-MA) 开发，MIT 许可证）进行扩展开发。ChaiTea 的核心思路是将 [CHAI3D](https://www.chai3d.org/) 触觉仿真框架桥接到 Unity3D 游戏引擎，使 Unity 开发者能够以 C# 脚本直接操控触觉力反馈设备。

本项目在原版基础上新增了：
- **ODE（Open Dynamics Engine）物理引擎**集成，支持刚体动力学仿真；
- 触觉时钟（PrecisionClock）、仿真控制（Simulation）等扩展模块；
- 针对工程实际场景的接口调整与优化。

ChaiTeaLib 由一个 C++ 原生动态链接库（`ChaiTeaLib.dll`）和一个 C# 托管封装层（`ChaiTeaPlugin`）组成，通过 P/Invoke（平台调用）机制将底层触觉设备接口暴露给 Unity 脚本环境。

---

## 项目结构

```
CHAI3D.sln
├── Dll1/                        # C++ 原生 DLL 项目（输出 ChaiTeaLib.dll）
│   ├── Types.h / Types.cpp      # 核心数据类型（向量、变换、设备信息等）
│   ├── World.h / World.cpp      # 触觉场景根容器
│   ├── GenericObject.*          # 场景对象基类
│   ├── Mesh.* / MultiMesh.*     # 多边形网格与碰撞检测
│   ├── Shapes.*                 # 基本几何体（球体、盒体、圆柱体）
│   ├── HapticDevice.*           # 触觉设备通信接口
│   ├── HapticMaterial.*         # 触觉材质属性
│   ├── GenericTool.*            # 触觉工具基类
│   ├── ToolCursor.*             # 笔型工具（光标模式）
│   ├── ToolGripper.*            # 夹持工具（夹爪模式）
│   ├── ODEworld.*               # ODE 物理引擎集成
│   ├── ODEGenericBody.*         # ODE 刚体定义
│   ├── precisionclock.*         # 高精度计时器
│   ├── Simulation.*             # 仿真控制
│   └── Util.*                   # 坐标转换、日志等工具函数
│
├── ChaiTeaPlugin/               # C# 托管封装层（.NET Standard 2.1，当前版本）
│   └── src/
│       ├── Types.cs             # 数据类型定义（对应 C++ Types）
│       ├── Util.cs              # 工具函数封装
│       ├── HapticMaterial.cs    # 触觉材质封装
│       ├── clock/clock.cs       # 计时器封装
│       ├── Tools/               # 触觉工具封装
│       │   ├── HapticDevice.cs
│       │   ├── GenericTool.cs
│       │   ├── HapticPoint.cs
│       │   ├── ToolCursor.cs
│       │   ├── ToolGripper.cs
│       │   └── CollisionEvent.cs
│       ├── World/               # 场景对象封装
│       │   ├── World.cs
│       │   ├── GenericObject.cs
│       │   ├── Mesh.cs
│       │   ├── MultiMesh.cs
│       │   ├── ShapeBox.cs
│       │   ├── ShapeSphere.cs
│       │   └── ShapeCylinder.cs
│       └── ODEworld/            # 物理引擎封装
│           ├── ODEworld.cs
│           └── ODEgenericbody.cs
│
└── chai3dplugin/                # C# 封装层（旧版，已废弃）
```

---

## 技术架构

```
Unity 应用层
    ↓  (C# 脚本调用)
ChaiTeaPlugin  (.NET Standard 2.1 / C#)
    ↓  (P/Invoke DllImport)
ChaiTeaLib.dll  (C++ 原生，x64)
    ↓  (链接)
CHAI3D 库  (触觉渲染 + 3D 图形)
    ↓
触觉设备硬件
```

---

## 核心模块说明

### 数据类型（Types）

| 类型 | 说明 |
|------|------|
| `Vec3` | 三维向量（x, y, z） |
| `Vec4` | 四元数（x, y, z, w） |
| `Transform` | 位姿（位置 + 旋转） |
| `HapticDeviceInfo` | 设备规格信息（最大力、刚度、阻尼、工作空间半径等） |
| `EffectType` | 触觉效果枚举：Magnet / StickSlip / Surface / Vibration / Viscosity |

### 场景管理（World & Objects）

- **World**：所有触觉对象的根容器，负责场景初始化与更新。
- **GenericObject**：场景对象基类，支持层级结构、位置旋转、材质、触觉效果。
- **Mesh / MultiMesh**：多边形网格，支持 AABB 碰撞检测。
- **ShapeSphere / ShapeBox / ShapeCylinder**：基本几何体图元。

### 触觉工具（Tools）

- **ToolCursor**：笔型工具，提供代理位置（Proxy Position）与设备实际位置。
- **ToolGripper**：夹持工具，支持夹爪姿态（GripperPose）与工作空间缩放。
- 关键方法：`ComputeInteractionForces()`、`ApplyToDevice()`、`UpdateFromDevice()`

### 触觉设备（HapticDevice）

设备支持能力来源于底层 CHAI3D 库（版本 3.3.0）。CHAI3D 的 `cHapticDeviceModel` 枚举共定义 **25 个设备标识**（含 Virtual 虚拟设备和 Custom 自定义接口），涵盖以下硬件型号：

| 系列 | 型号 |
|------|------|
| Phantom（3D Systems） | Touch、Omni、Desktop、15-6DOF、30-6DOF、Other |
| Delta（Force Dimension） | Delta 3、Delta 6 |
| Omega（Force Dimension） | Omega 3、Omega 6L/6R、Omega 7L/7R |
| Sigma（Force Dimension） | Sigma 6L/6R、Sigma 7L/7R |
| Lambda（Force Dimension） | Lambda 7L/7R |
| 其他 | Falcon（Novint）、MPR、Sixense（追踪器）、Leap Motion（追踪器） |

### 触觉材质（HapticMaterial）

可配置以下触觉属性：
- 刚度（Stiffness）、阻尼（Damping）、黏滞（Viscosity）
- 静/动摩擦（Static / Dynamic Friction）
- 振动（Vibration）、磁力（Magnet）、粘滑（Stick-Slip）

### 物理引擎（ODE）

- **ODEworld**：Open Dynamics Engine 物理世界，支持重力、阻尼、动力学更新。
- **ODEGenericBody**：动态/静态刚体，支持盒体、平面、质量、力、位置配置。

---

## 开发环境

| 项目 | 要求 |
|------|------|
| C++ DLL | Visual Studio 2022+，Windows x64 |
| C# 封装 | .NET Standard 2.1 |
| Unity 集成 | Unity 2019.4+（需引用 UnityEngine 包） |
| 依赖库 | CHAI3D、ODE（Open Dynamics Engine） |

---

## 构建方式

1. 使用 Visual Studio 打开 `CHAI3D.sln`。
2. 选择 **x64 / Debug** 或 **x64 / Release** 配置。
3. 先生成 `Dll1` 项目，输出 `ChaiTeaLib.dll`。
4. 再生成 `ChaiTeaPlugin` 项目，输出托管封装程序集。
5. 将 `ChaiTeaLib.dll` 和 `ChaiTeaPlugin.dll` 复制到 Unity 项目的 `Assets/Plugins/` 目录下。

---

## 上游项目：ChaiTea

本项目 fork 自 **[Mads-MA/ChaiTea](https://github.com/Mads-MA/ChaiTea)**，该项目的基本结构如下：

| 目录 | 说明 |
|------|------|
| `ChaiTeaLib/` | C++ 原生库，封装 CHAI3D 接口 |
| `ChaiTeaPlugin/` | C# 托管封装，供 Unity 调用 |
| `external/chai3d` | CHAI3D 框架（git 子模块） |
| `external/UnityCsReference` | Unity C# Reference（git 子模块） |

原始项目提供的示例场景（见 [ChaiTea-Examples](https://github.com/Mads-MA/ChaiTea-Examples)）包含：

- **Effects**：展示振动、黏滞、粘滑、磁力等触觉效果
- **HapticMesh**：加载自定义网格用于触觉反馈
- **SolidBox**：实心盒体触觉交互
- **PrimitiveTypes**：球体、盒体、平面、圆柱等基本几何体交互

---

## 许可与致谢

- 本项目基于 **[ChaiTea](https://github.com/Mads-MA/ChaiTea)**（MIT 许可证，作者：Mads Mørch Antonsen）进行二次开发。
- 底层触觉仿真框架：**[CHAI3D](https://www.chai3d.org/)**（BSD 3-Clause 许可证，版权所有 © 2003-2024 CHAI3D）。
- 物理仿真引擎：**[ODE（Open Dynamics Engine）](https://www.ode.org/)**。
- 商业技术支持：**[Force Dimension](https://www.forcedimension.com/)**。
