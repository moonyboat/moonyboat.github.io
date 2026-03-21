---
title: 01 Habitat 介绍
date: 2025-12-05T21:27:17+08:00
draft: false
tags:
  - Habitat
summary: "  "
---
# 内容来源
Habitat官方发布的视频教程以及colab例程代码，学习记录。视频教程详见A-star的[Habitat Tutorials](https://www.youtube.com/watch?v=L9GuINYhmZI&list=PLGywud_-HlCORC0c4uj97oppQrGiB6JNy&index=2)。

# 核心目标
 
 核心目标是为具身 AI (embodied AI)提供一个 **高性能、真实感、可扩展的 3D 模拟平台**，让研究者可以在虚拟室内环境中训练/测试智能体 (agent) — 包括视觉 (RGB / 深度 / 语义等 sensors)、导航 (navigation)、感知 (perception)、动作 (movement) 等。 

架构分为两部分 (即 “底层 + 上层”)：
- **Habitat‑Sim** — 高性能 3D 模拟器 ，负责场景渲染、sensor 模拟 (RGB, depth, semantics…)、agent 配置 (相机、碰撞、运动…) 以及对多种 3D 数据集 (e.g. Matterport3D, Gibson 等) 的支持。
- **Habitat‑Lab (也称 Habitat-API)** — 高层接口 (framework / library)，用于定义任务 (navigation, exploration, embodied-AI tasks)、配置 agent、训练 & 评估、benchmark。

![](images/截图%202025-12-05%2021-16-03.png)

传统的仿真工具大多建立在unity，再嵌套Python的基础api，再结合PyTorch的高级api实现仿真，仿真速度慢，效率太低，帧率只有10-60 FPS。

通过 Habitat，你能够以非常高的效率 (fast rendering, many fps) 运行大规模实验 — 相比传统真实机器人实验，它大幅降低了成本与复杂性，同时加速研究进展。 

开发本质并不是向人类渲染丰富的场景环境，而是直接对人工智能或者具身智能开发，关注不同的机器人形态，设计多种传感器，如音频、视觉等，habitat诞生于对具身智能的架构需求。

![697](images/截图%202025-12-05%2021-21-47.png)
![](images/截图%202025-12-05%2021-23-42.png)

介绍 Tutorial outline

![](images/截图%202025-12-05%2021-27-17.png)