+++
title = 'GAMES101 作业学习笔记'
date = 2026-08-28T15:35:19+10:00
draft = false
description = 'GAMES101 课程作业学习记录与知识索引。'
categories = ['计算机图形学']
tags = ['GAMES101', '计算机图形学', '学习笔记']
url = '/games101-assignments/'
+++

这个系列用软件光栅器的实现过程串联 GAMES101 的核心知识。笔记重点记录公式为什么成立、代码在渲染管线中的位置，以及实现时最容易混淆的坐标和边界约定。

| 作业 | 主题 | 核心内容 | 状态 |
| --- | --- | --- | --- |
| [Assignment 1：MVP 变换与透视投影](/post/games101-assignment-1/) | Rotation and Projection | 坐标空间、模型矩阵、观察矩阵、透视投影、透视除法、视口变换 | 已整理 |
| [Assignment 2：三角形光栅化与深度缓冲](/post/games101-assignment-2/) | Triangles and Z-buffering | 包围盒、覆盖测试、重心坐标、深度插值、Z-buffer、2×2 MSAA | 已整理 |

## 学习路径

建议先读 Assignment 1，确认顶点如何从局部空间进入屏幕空间；再读 Assignment 2，继续追踪屏幕空间三角形如何变成最终像素。这两篇合在一起，就是 CPU 软件光栅器最基础的一条渲染管线。

后续作业会在这套骨架上继续加入着色、纹理、曲线、光线追踪和物理模拟。
