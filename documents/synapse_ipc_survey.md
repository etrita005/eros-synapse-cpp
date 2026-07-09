# 机器人进程间通信（IPC）技术综述

## 摘要

本文档梳理机器人操作系统 IPC 设计的核心概念、主流机制与对比分析，为 `eros-synapse-cpp` 的设计与实现提供知识基础。内容涵盖 DDS/RMW 抽象、RPC 实现、与 Android IPC 的对比、主流 RPC 框架差异、流式 RPC，以及从第一性原理出发的机器人 IPC 功能需求。

---

## 1. 机器人 IPC 的本质

机器人系统是一种**分布式物理计算系统**：

- 多个进程/线程并行运行（感知、规划、控制、交互）
- 数据具有时间、空间与语义维度
- 通信延迟直接影响控制稳定性与人机安全
- 组件可能分布在 CPU、GPU、NPU、MCU 及云端
- 组件独立失效，系统需优雅降级

因此，机器人 IPC 不是简单的“进程间传数据”，而是**时间敏感、语义化、可容错、跨异构硬件的统一数据总线**。

---

## 2. 通信模式

| 模式 | 特点 | 机器人场景 |
|---|---|---|
| **发布订阅 Pub/Sub** | 多对多、解耦、异步 | 传感器流、状态广播、控制指令 |
| **请求-响应 RPC** | 一对一、同步/异步、有返回 | 获取地图、调用技能、查询参数 |
| **单向通知** | 无返回、低延迟 | 事件告警、状态机触发 |
| **流式 RPC** | 持续双向数据流 | 远程操控、语音对话、视频分析 |
| **共享状态/黑板** | 多读者多写者 | 世界模型、任务上下文 |

---

## 3. DDS 与 RMW 抽象

### 3.1 DDS 核心概念

| DDS 概念 | 含义 |
|---|---|
| **Domain** | 逻辑通信空间，同一 Domain 的 Participant 才能互相发现 |
| **DomainParticipant** | Domain 中的成员身份，所有 DDS 实体的工厂 |
| **Topic** | 发布订阅的逻辑主题 |
| **DataWriter** | 发布端实体，向 Topic 写数据 |
| **DataReader** | 订阅端实体，从 Topic 读数据 |
| **QoS** | 服务质量策略，控制可靠性、延迟、历史等 |
| **Discovery** | 对等发现机制，自动发现 Participant 与 Endpoint |

### 3.2 RMW 对 DDS 的抽象

ROS2 的 RMW（ROS Middleware Interface）层将 DDS 概念映射为 ROS2 语义：

| DDS 概念 | RMW 抽象 | 说明 |
|---|---|---|
| Domain | `rmw_init_options_t` 中的 `domain_id` | 不暴露为独立对象 |
| DomainParticipant | `rmw_node_t` | 一个 ROS 节点对应一个 Participant |
| Topic | 隐式创建 | 由 Publisher/Subscriber 创建时生成 |
| DataWriter | `rmw_publisher_t` | `rmw_publish()` |
| DataReader | `rmw_subscription_t` | `rmw_take()` |
| QoS | `rmw_qos_profile_t` | 统一 QoS 结构体 |
| Discovery | 图查询 API | `rmw_get_node_names`、`rmw_get_topic_names_and_types` 等 |

### 3.3 为什么叫 Domain 和 Participant

- **Domain**：借用“论域/通信域”概念，类似 VLAN。它是一个逻辑边界，不是进程、主机或网络。
- **Participant**：借用“参与者”概念，表示 Domain 中的一个活跃成员。一个 Participant 代表一个完整的 DDS 发现身份。

---

## 4. RMW 的 RPC 实现

### 4.1 基本实现

RMW 的 Service/Client 不依赖 DDS-RPC 标准，而是**用两个 Topic 模拟 RPC**：

- `rq/<service_name>`：Client → Server，发送请求
- `rr/<service_name>`：Server → Client，发送响应

### 4.2 请求-响应匹配

由于 DDS Pub/Sub 是多对多的，RMW 通过以下信息匹配：

- `writer_guid`：标识请求来源客户端
- `sequence_number`：单调递增序列号，服务端回写响应时携带

客户端只接收序列号匹配自己未决请求的响应。

### 4.3 为什么不使用 DDS-RPC

DDS-RPC 是 OMG 定义的 DDS 扩展标准，但 ROS2 RMW 普遍不直接使用，原因包括：

- DDS-RPC 是可选扩展，厂商支持不一致
- ROS2 需要跨多种 DDS 实现保持一致性
- 双 Topic 实现更简单、可控
- DDS-RPC 设计时 ROS2 已采用自有方案

---

## 5. RMW RPC vs Android IPC

| 维度 | RMW RPC | Android IPC |
|---|---|---|
| 通信模型 | 基于 Pub/Sub 模拟 RPC | 面向对象 RPC |
| 发现机制 | DDS 对等发现，无中心注册表 | ServiceManager 集中注册 |
| 传输范围 | 可跨主机、跨网络 | 仅限同一设备 |
| 调用方式 | 异步为主 | 同步为主 |
| 数据拷贝 | 依赖 DDS 序列化/网络拷贝 | Binder `mmap` 一次拷贝 |
| 安全模型 | DDS-Security | Android UID/SELinux |
| 多服务端 | 请求广播到所有服务端 | 定向到绑定的服务进程 |
| 失败感知 | 较弱，依赖超时 | 强，可监听 DeathRecipient |
| 大数据 | 依赖 DDS 共享内存 | Binder 1MB 限制，需 Ashmem |

---

## 6. 主流 RPC/IPC 框架对比

| 框架 | Pub/Sub | RPC | 流式 RPC | 单向 | 回调 | 跨主机 | 主要场景 |
|---|---|---|---|---|---|---|---|
| **RMW/ROS2** | ✅ Topic | ✅ 双 Topic | ⚠️ Topic 模拟 | ⚠️ Topic | ⚠️ 另建 Service | ✅ | 机器人分布式系统 |
| **Android IPC** | ⚠️ Broadcast | ✅ | ❌ | ✅ | ✅ | ❌ | 设备内系统服务 |
| **D-Bus** | ✅ Signals | ✅ | ❌ | ✅ | ✅ | ⚠️ | Linux 桌面/车载 |
| **gRPC** | ❌ | ✅ | ✅ | ⚠️ Empty | ⚠️ | ✅ | 云原生微服务 |
| **DCOM** | ✅ Events | ✅ | ❌ | ⚠️ | ✅ | ⚠️ | Windows 组件 |

---

## 7. 流式 RPC

流式 RPC 允许在一次调用中持续传输多个消息：

- **服务端流式**：一个请求，多个响应
- **客户端流式**：多个请求，一个响应
- **双向流式**：双方独立发送多个消息

gRPC 原生支持流式 RPC，基于 HTTP/2 多路复用实现。

RMW、Android IPC、D-Bus、DCOM 没有原生流式 RPC，通常用以下方式模拟：

- RMW：Topic 发布订阅
- Android IPC：多次 RPC + Ashmem
- D-Bus：Signals
- DCOM：Connection Points

---

## 8. 机器人 IPC 的第一性原理需求

从移动机器人、具身智能的未来场景出发，机器人 IPC 中间件应具备：

1. **多通信模式**：Pub/Sub、RPC、流式、事件、共享状态
2. **传输抽象**：同进程、跨进程、跨设备、机器人 ↔ MCU、CPU ↔ GPU/NPU
3. **去中心化发现**：动态上下线、语义命名、命名空间隔离
4. **丰富 QoS**：Reliability、Deadline、Durability、History、Liveliness、Ownership
5. **实时性**：优先级调度、Deadline 监控、有界延迟
6. **时间模型**：Wall-time、Mono-time、Sensor-time、跨节点同步
7. **类型系统**：强类型、Schema 演进、跨语言、零拷贝
8. **安全**：认证、加密、访问控制、审计
9. **容错**：心跳、看门狗、优雅降级、幂等
10. **可观测性**：延迟/吞吐指标、拓扑可视化、日志回放
11. **AI 原生**：Tensor 传输、多模态同步、数据飞轮
12. **资源效率**：内存池、背压、带宽自适应

---

## 9. Synapse 的定位

Synapse 作为 EROS 生态的 IPC 中间件，目标是在 Cytoskeleton 基础之上实现上述需求，同时：

- 不重复实现 ROS2 RMW 或 DDS，但可借鉴其设计
- 与 Kinesin 算法库无直接依赖，但为其提供高效数据通道
- 优先满足机器人本体内通信，再扩展跨设备与云边协同
- 核心设计原则：**时间敏感、语义化、AI 原生、安全可信、跨异构硬件**

---

## 10. 关键设计启示

1. **不要强制把所有通信都建模为 RPC**：机器人系统更适合 Pub/Sub + RPC 混合模型。
2. **发现机制决定系统扩展性**：去中心化发现更适合动态机器人系统。
3. **QoS 是核心接口**：IPC 必须暴露丰富的 QoS，而不是隐藏所有策略。
4. **零拷贝对 AI 场景至关重要**：图像、点云、Tensor 等大消息必须避免拷贝。
5. **安全不能事后补**：从一开始就要考虑认证、加密、访问控制。
6. **可观测性是调试分布式系统的关键**：拓扑、延迟、日志缺一不可。

---

## 11. 参考

- OMG DDS Specification
- OMG DDS-RPC Specification
- ROS2 RMW Interface Design
- Android Binder Documentation
- gRPC Core Concepts
- D-Bus Specification
- Microsoft DCOM Documentation
