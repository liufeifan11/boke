---
title: "目标检测深度解析：从 R-CNN 到 DETR 的演进与技术细节"
date: 2026-07-31
tags: [object-detection, deep-learning, yolo, detr, cv]
description: "深度解析目标检测技术：从传统方法到两阶段/单阶段检测器，再到 DETR 等 Transformer 架构的演进，覆盖 anchor、Focal Loss、集合预测与损失函数进化等关键细节"
author: "Hermes Writer Profile"
---

# 目标检测的"减法"哲学：从 R-CNN 到 DETR 的演进与关键细节

你能用多少行代码描述一个目标检测器？

2014 年的答案是三千行 C++，外加 Selective Search 区域生成和四个独立训练阶段。2020 年 DETR 的回答是：一个标准 Transformer encoder-decoder，不到 500 行 Python，没有 anchor、没有 NMS、没有 region proposal。六年时间，目标检测社区持续做同一件事：**把越多越多的手工设计模块，替换成可学习的神经网络组件**。

本文以这条"减法"主线，梳理从 R-CNN 到 DETR 的核心技术演进，并聚焦每个阶段的代表性数字和设计思想。

## 两阶段时代：从手工 proposal 到可学习的 RPN

2014 年，Girshick 用 **R-CNN** 开启深度学习检测时代。流程很直接却极慢：Selective Search 从每张图提取约 2000 个候选区域，每个区域缩放到固定尺寸后分别送入 CNN（AlexNet/VGG）提取特征，再接 SVM 分类和线性回归微调边框。VOC 2007 上达到 **58.5% mAP**，比传统 DPM 的 ~33% 提升 25 个点以上，但单张推理约 **47 秒**——2000 次 CNN 前向是致命瓶颈。

**Fast R-CNN（2015）** 用一个简单翻转改变了格局：整图只过一次 CNN，在共享特征图上用 **RoI Pooling** 将每个 proposal 映射为固定尺寸（7×7 像素）的特征块，再用全连接层同时输出分类和边框回归。VOC 2007 达到 **70.0% mAP**，推理从 47 秒降到 0.3 秒——快了 200 倍。但 proposal 生成仍依赖外部的 Selective Search，与网络分离、不可学习。

**Faster R-CNN（2015）** 补上了最后一块拼图。它引入 **Region Proposal Network（RPN）**，这是整个检测史的转折点。RPN 是一个小型的全卷积网络，在共享特征图上滑动一个小窗口（3×3 卷积），每个位置同时预测两个输出：分类分支输出该位置是否为前景（objectness），回归分支输出多个预设 anchor 的偏移量。关键创新是 **Anchor（锚框）**——特征图每个位置预设 3 种尺度 × 3 种宽高比共 9 个先验框，RPN 不直接回归绝对坐标，而是回归相对锚框的偏移 `(dx, dy, dw, dh)`，这极大地降低了回归难度。RPN 与下游 Fast R-CNN 共享卷积层，整个检测流程首次真正端到端可训练。VOC 2007 达到 **73.2% mAP**，推理速度约 5 FPS（VGG-16 主干）。后续 FPN（特征金字塔，2017）和 Mask R-CNN（添加 RoIAlign + 实例分割分支，2017）进一步巩固了两阶段范式的精度统治。

## 单阶段突破：Focal Loss 解决了什么问题

两阶段方法的 proposal 筛选天然限制速度。**YOLOv1（2016，Redmon）** 开创性地将检测视为单次回归：图像划分为 7×7 网格，每格直接预测 B 个边界框和各类别概率，不需要 proposal，不需要 region refinement。**63.4% mAP @ 45 FPS** 的速度震撼了社区，但定位粗糙、小目标差。YOLOv2 引入 anchor boxes 和维度聚类，YOLOv3 采用 Darknet-53 主干和多尺度预测（三种分辨率特征图），成为工业界最普及的基线。

**SSD（2016，Liu）** 从另一个角度补强单阶段：在多尺度特征图上每位置预设多个 default boxes（SSD300 共 8732 个），直接从浅层特征检测小目标、深层检测大目标。SSD300 达到 **72.1% mAP @ 58 FPS**，精度逼近 Faster R-CNN（73.2%）却快一个数量级；SSD512 进一步将输入放大到 512×512，在 VOC 2007 拿到 **74.9% mAP**（约 22 FPS），首次让单阶段在精度上全面超越两阶段 SOTA。

但单阶段检测器长期受困于一个根本问题：**极端的前景-背景类别不平衡（class imbalance）**。每张图密集采样产生约 10 万个候选位置，其中 99.9% 是易分类的负样本（背景），梯度被海量简单样本淹没，模型难以学习有效特征。

**RetinaNet + Focal Loss（2017，Lin et al.）** 精准诊断并解决了这个问题：

```
FL(p_t) = −α_t · (1 − p_t)^γ · log(p_t)
```

其中 `p_t` 是模型对正确类别的预测概率。调制因子 `(1−p_t)^γ` 是精髓：对易分类样本（`p_t→1`），因子趋近 0，损失贡献被大幅压制；对难分类样本（`p_t` 接近 0.5），因子接近 1（γ=2 时），损失几乎不变。配合 `α_t=0.25` 平衡正负样本权重，训练自动聚焦 hard examples。

这个简单的损失函数改造让单阶段检测器首次**在精度上全面超越所有两阶段 SOTA**（COCO AP 39.1）。核心启示：精度差距的根源不是 proposal 机制的缺失，而是训练信号被简单负样本淹没。后续 YOLOv8 全面转向 anchor-free，YOLOv10 用一致双分配实现免 NMS 端到端，YOLO26 进一步去除 DFL 分布损失和 NMS，单阶段检测持续精简化。

## 去掉 Anchor：回归检测的原始形态

Anchor 为回归提供了参考坐标，但超参数（尺度、宽高比、每层数量）需要大量调参，跨数据集泛化差，且正负样本分配加剧了不均衡。Anchor-free 运动从两个方向发起攻击：

**关键点路线**：CornerNet（2018）把检测建模为角点配对——分别预测左上角和右下角的 heatmap，通过 embedding 向量距离将同一物体的两个角点配对。配套的 corner pooling 沿对角线方向聚合角点特征。COCO 42.1–42.2% AP，首次让 anchor-free 接近两阶段水平。CenterNet（2019）简化了这个思路：只预测物体中心点的 heatmap，再从中心点回归到四条边的距离，无需配对步骤。

**逐像素路线**：FCOS（2019）最强悍。特征图上每个位置直接输出 4 个距离值 `(l, t, r, b)` 和一个类别，还有精心设计的 **centerness 分支**——预测该位置到目标中心的归一化距离（`centerness = sqrt(min(l,r)/max(l,r) × min(t,b)/max(t,b))`），在推理时乘到分类分数上，压制远离中心的低质量框。COCO 44.7% AP，证明简单直接的 per-pixel 预测可以匹敌甚至超越 anchor-based 方法。

## Transformer 入场：检测就是集合预测

**DETR（2020，Carion et al., FAIR）** 将"减法"推到极致。核心理念：目标检测本质上是集合预测——输入图像，输出一个无序的目标集合。架构极简：CNN backbone 提取特征 → 展平加正弦位置编码 → 标准 Transformer encoder（6 层全局 self-attention）→ decoder（6 层 cross-attention，输入 N=100 个可学习的 **object queries**）→ 两个共享 MLP 头分别输出类别和边框。

训练中最关键的创新是 **匈牙利二分匹配（bipartite matching）**——计算所有预测与所有 GT 之间的代价矩阵（代价 = 分类 cost + L1 bbox cost + GIoU cost，典型权重 1:5:2），用匈牙利算法找到最小代价的一一匹配，只对匹配到的 query 计算回归损失。这保证了每个 GT 恰好对应一个预测，没有 anchor 重叠带来的指派模糊。COCO 上 ResNet-50 达到 42 AP（500 epochs）。

但 DETR 有三大先天缺陷：收敛极慢、小目标差、注意力 O(N²) 复杂度。后续改进环环相扣：

- **Deformable DETR（2021）** 用可变形注意力取代全局 attention——每个 reference point 只在多尺度特征图上采样 K 个关键点（典型值 4 尺度 × 4 点 × 8 头），复杂度从 O(N²) 降到 O(N·K)。**收敛加速 10 倍**：50 epochs 即达 43.8 AP，超过原版 500 epochs 的 42 AP。
- **DN-DETR（2022）** 加入去噪训练：额外输入 GT 坐标加噪声的 noisy queries，绕过早期匹配不稳定，1/10 的 epoch 达到原版水平。
- **DINO（2023，ICLR）** 集所有技法于一身——动态 anchor query + 对比去噪 + 混合 query 选择（用 encoder 输出 Top-K 初始化 decoder query），**12 epochs 即 49.4 AP，24 epochs 达 51.3 AP**，Swin-L 大模型在 COCO 上登顶 **63.2 AP**。
- **RT-DETR（2024，百度）** 面向实时部署：高效混合编码器（只在最高层做 transformer 交互，其余用 CNN 跨尺度融合） + IoU-aware query selection，**53.0% AP @ 114 FPS（T4）**，首次在实时区间打赢同档 YOLO。

## 损失函数的进化：从 IoU 到 CIoU

边框回归损失的演进同样体现"让优化目标贴近评估指标"的逻辑。原始 L1/L2 loss 对尺度敏感且与 IoU 不直接相关。**IoU Loss（2016）** 直接用 `L=1−IoU` 作为损失，但预测框与 GT 无重叠时 IoU=0，梯度消失。**GIoU（2019）** 引入最小外接框：`L = 1−IoU + |C−A∪B|/|C|`，无重叠时仍有梯度。**DIoU（2020）** 进一步加中心距离惩罚 `ρ²(b, b_gt)/c²`，收敛更快。**CIoU（2020）** 再补上宽高比一致性项 `αv`，`v = 4/π² · (arctan(w_gt/h_gt) − arctan(w/h))²`。YOLOv4 起，CIoU 成为工业界边框回归的标准配置。

## 评估：COCO AP 的三个维度

COCO 的 AP 远比 VOC 的 mAP@0.5 严格：在 IoU 阈值 [0.5, 0.55, ..., 0.95] 共 **10 个阈值**上分别算 AP 再取平均（记作 AP@[0.5:0.95]），对定位精度极其严苛。此外 COCO 还将目标按面积分档：**APS**（<32² 像素）、**APM**（32²–96²）、**APL**（>96²）。这两个维度叠加形成了评估的目标检测三坐标：**定位精度（IoU 轴）、尺度覆盖（面积轴）、类别泛化（80 类均值）**。一篇论文号称"SOTA"，通常需在这三轴上全面对比，不能只看 AP 单一数字。

## 前沿：免 NMS 与开放词汇

最新趋势延续"减法"方向并向外扩展。**免 NMS 端到端**：YOLOv10 用一致双分配训练（推理用 one-to-one 匹配，训练时额外 one-to-many 分支提供丰富监督），YOLO26 原生去除 NMS 和 DFL，推理图更干净、量化部署更友好。**开放词汇检测（Open-vocabulary Detection）**：Grounding DINO（2023）基于 DINO 架构融合文本分支，用户用自然语言指定任意类别，COCO 零样本 **52.5 AP**（从未见过 COCO 训练数据），微调后 63.0 AP。检测正从"识别预定义 80 类"走向"理解任意文本描述的目标"。

## 结语

从 2014 年 R-CNN 的三千行多阶段流水线到 2024 年 RT-DETR 的百行端到端模型，目标检测的进化是一部"持续做减法"的历史：去掉手工特征、去掉独立 proposal、去掉 anchor 先验、去掉 NMS 后处理、去掉闭集类别限制。每一次"去掉"都不是简化，而是对检测问题本质的更深理解——用更强的表征学习、更优雅的匹配策略和更好的损失设计，取代那些过去看似不可或缺的脚手架。当模型不再需要这些外部组件时，端到端学习的真正威力才开始显现。
