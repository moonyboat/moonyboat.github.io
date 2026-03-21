---
title: 05 Habitat-lab 导航任务
date: 2026-03-20T00:00:00+08:00
draft: false
tags:
  - Habitat
  - Navigation
summary:
---
Habitat-Lab 的一个重要目标是让用户能够轻松地在 3D 环境中设置各种具身智能体任务。设置任务的过程包括：使用模拟器提供的环境信息，将这些信息与数据集（例如 PointGoal 目标，或用于 Embodied QA 的问答对）连接起来，并提供可供智能体使用的观测。在牢记这一主要目标的前提下，核心 API 定义了以下几个可以扩展的关键抽象概念：

* `Env`：Habitat 的基础环境概念。使用模拟器进行具身任务研究所需的所有信息都被抽象在 Env 中。该类作为其他派生环境类的基类。Env 由三个主要组件组成：Simulator、Dataset（包含 Episodes）以及 Task，并负责将这三个组件连接在一起。

* `Dataset`：包含来自特定数据划分的一组与任务相关的 episode，以及额外的整个数据集级别的信息。负责从磁盘加载和保存数据集、获取场景列表，以及获取某个特定场景的 episode 列表。

![700](images/Pasted%20image%2020260316201535.png)
* `Episode`：用于描述 episode 规格的类，其中包括 Agent 的初始位置与朝向、场景 ID、目标位置，以及可选的到达目标的最短路径。一个 episode 描述了智能体的一个任务实例。

* `Task`：该类建立在模拟器和数据集之上。episode 终止的条件以及成功度量由 Task 提供。

* `Sensor`：对模拟器所提供物理传感器概念的泛化，能够以指定格式提供与 Task 相关的观测数据。

* `Observation`：表示来自 Sensor 的观测数据。这可以对应于 Agent 上的物理传感器（例如 RGB、深度、语义分割掩码、碰撞传感器），也可以是更抽象的传感器，例如当前智能体状态。

需要注意的是，核心功能定义了基本构建模块，例如用于与模拟器后端交互的 API，以及通过Sensors 接收观测数据的机制。具体的模拟后端、3D 数据集以及具身智能体基线实现都构建在核心 API 之上。

可以先把 Habitat 的任务理解成三层：
1. Simulator  负责场景、机器人、传感器、物理。
2. Dataset  负责一组 episode，也就是“每一条导航任务的数据定义”。
3. Task  负责动作空间、观测、奖励、成功判定、指标。

**Habitat 标准导航任务怎么定义**  
标准做法是“按 episode 写 JSON”。一个 episode 至少会描述：
- scene_id
- start_position
- start_rotation
- goals
如果是 ObjectNav，还会有：
- object_category
- 目标物体对应的 goals/view_points

