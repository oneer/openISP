# 02. BLC: Black Level Compensation

## 模块作用

BLC 是黑电平补偿模块，位于 DPC 后面。它修正传感器在无光照时仍然存在的基础读数偏置，让真正的黑接近数字零点或目标黑位。

黑电平如果不准，暗部会发灰或偏色，AWB、CCM、Gamma 都会在错误基线上继续放大问题。

## openISP 实现

源码类名为 `BLC(img, parameter, bayer_pattern, clip)`。`parameter` 包含：

```text
[bl_r, bl_gr, bl_gb, bl_b, alpha, beta]
```

代码支持 `rggb/bggr/gbrg/grbg` 四种 Bayer pattern。以 `rggb` 为例：

```text
R  Gr
Gb B
```

切片如下：

```python
r  = img[::2, ::2]
gr = img[::2, 1::2]
gb = img[1::2, ::2]
b  = img[1::2, 1::2]
```

## 核心公式

openISP 用加法表达通道偏置：

```text
R  = R  + bl_r
B  = B  + bl_b
Gr = Gr + bl_gr + alpha * R / 256
Gb = Gb + bl_gb + beta  * B / 256
```

真实 ISP 中常见形式是减去黑电平，openISP 的默认配置为 0，更像给各通道加可配置 offset。`alpha/beta` 是额外的固定点交叉补偿项。

## 参数说明

| 参数 | 含义 |
| --- | --- |
| `bl_r` | R 通道黑电平或偏置补偿 |
| `bl_gr` | Gr 通道黑电平或偏置补偿 |
| `bl_gb` | Gb 通道黑电平或偏置补偿 |
| `bl_b` | B 通道黑电平或偏置补偿 |
| `alpha` | R 对 Gr 的补偿系数 |
| `beta` | B 对 Gb 的补偿系数 |
| `clip` | RAW 输出上限 |

## 学习重点

- Gr 和 Gb 应该分开处理，因为它们在传感器上的位置不同，响应可能不一致。
- BLC 错误会直接影响白平衡估计和颜色矩阵结果。
- BLC 一般依赖 optical black 区域、传感器标定或温度补偿。

## 面试问答

### Q1: 为什么 BLC 要在 AWB 前？

AWB 依赖通道强度比例。如果黑电平偏置还没去掉，尤其在暗部，R/G/B 比例会被错误基线扭曲，AWB gain 会估错或放大色偏。

### Q2: 黑电平过高或过低有什么现象？

黑电平补偿不足时暗部发灰、对比低。补偿过度时暗部被压死，阴影细节丢失，还可能出现暗部偏色。

### Q3: 为什么 Gr/Gb 要分开？

Bayer 中有两个绿色位置，它们处在不同像素行列，受工艺、微透镜、读出路径影响可能不同。分开处理可以修正 green imbalance，减少棋盘感或绿色纹理。

### Q4: BLC 和 offset correction 是一回事吗？

广义上接近。BLC 主要修正黑位基线，offset correction 可以泛指任意通道偏移修正。ISP 语境里 BLC 更特指 RAW 黑电平。

### Q5: BLC 标定通常怎么做？

常用暗场图或 optical black 像素，统计各通道在无光条件下的平均值或中位数，再生成随 gain、曝光、温度变化的黑电平表。
