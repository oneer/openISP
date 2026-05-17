# 05. AWB/WBGC: White Balance Gain Control

## 模块作用

openISP 的 `WBGC` 是白平衡增益应用模块，不是完整自动白平衡算法。它在 Bayer RAW 域对 R、Gr、Gb、B 采样点分别乘以 gain，让不同光源下的中性物体恢复接近中性。

## openISP 实现

源码类名为 `WBGC(img, parameter, bayer_pattern, clip)`。`parameter` 包含：

```text
[r_gain, gr_gain, gb_gain, b_gain]
```

每种 Bayer pattern 都会按坐标切出四类采样点，乘增益后写回二维 Bayer 图。

## 核心公式

```text
R'  = R  * r_gain
Gr' = Gr * gr_gain
Gb' = Gb * gb_gain
B'  = B  * b_gain
```

最后裁剪到 `0..clip`。

## AWB 完整系统缺失的部分

真实 AWB 通常包含：

- 统计模块：收集 R/G/B 均值、直方图、白点候选。
- 光源估计：灰世界、白点检测、色温估计、场景识别。
- 决策和平滑：防止画面跳变。
- gain 应用：openISP 实现的是这一部分。

## 参数说明

| 参数 | 含义 | 现象 |
| --- | --- | --- |
| `r_gain` | 红通道增益 | 增大后画面更暖 |
| `gr_gain` | Gr 增益 | 影响亮度和绿色平衡 |
| `gb_gain` | Gb 增益 | 影响亮度和绿色平衡 |
| `b_gain` | 蓝通道增益 | 增大后画面更冷 |
| `clip` | RAW 上限 | 防止溢出 |

## 学习重点

- AWB gain 会放大噪声，尤其是暗部 R/B 通道。
- AWB 放在 CFA 前，可以在原始通道上操作，避免插值后耦合。
- Gr/Gb 分开有助于处理 green imbalance。

## 面试问答

### Q1: AWB 为什么常在 Bayer 域做？

Bayer 域的 R/G/B 采样还未被插值混合，通道关系更直接。先做 gain 可以让 demosaic 在更合理的颜色比例上工作。

### Q2: 白平衡 gain 过大会有什么副作用？

会放大该通道噪声，并可能造成高光 clipping。例如蓝光不足的暖光场景中，`b_gain` 大会让蓝通道暗部彩噪更明显。

### Q3: openISP 的 AWB 为什么不算真正的自动白平衡？

因为它不估计光源，也不自动计算 gain，只是应用配置文件或主程序给定的 gain。

### Q4: 灰世界 AWB 的基本假设是什么？

假设一幅自然图像的平均颜色应该接近灰色，因此可以根据 R/G/B 均值比例估计通道 gain。缺点是遇到大面积单色场景会失效。

### Q5: AWB 和 CCM 谁先谁后？

通常 AWB 先，CCM 后。AWB 先校正光源色温，CCM 再把传感器颜色空间映射到目标颜色空间。
