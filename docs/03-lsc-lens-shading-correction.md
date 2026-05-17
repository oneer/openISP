# 03. LSC: Lens Shading Correction

## 模块作用

LSC 是镜头阴影校正。openISP README 把它列为目标模块，但当前源码没有 `model/lsc.py` 实现，主 pipeline 中也只是注释占位。

LSC 解决镜头和传感器组合导致的中心亮、边缘暗、四角色偏问题。它通常工作在 Bayer RAW 域，并且按 R/Gr/Gb/B 分通道校正。

## 问题来源

镜头 shading 主要来自：

- 镜头边缘入射角导致的光量衰减。
- 微透镜和像素井对斜入射光响应不同。
- 红绿蓝不同波长在边缘的衰减不一致。
- 模组装配偏心造成的非对称 shading。

## 常见公式

对每个 Bayer 采样点乘以对应位置和颜色的 gain：

```text
pixel_out(x, y, color) = pixel_in(x, y, color) * gain_map(x, y, color)
```

通常中心 gain 接近 1，边缘和角落 gain 更大。

## gain map 类型

### 全尺寸 gain map

每个像素一个 gain，精度高，但存储量大。

### Mesh gain map

只存低分辨率网格，例如 `17x13` 或 `33x25`，运行时双线性插值到每个像素。真实 ISP 常用这种方式。

### 径向模型

用离光心距离建模：

```text
gain = a0 + a1*r^2 + a2*r^4 + ...
```

参数少，但无法很好处理非对称 shading。

## 标定流程

1. 拍摄均匀光源，例如积分球或均匀白板。
2. 先做 BLC，去掉黑电平。
3. 按 Bayer pattern 分离 R/Gr/Gb/B。
4. 计算每个位置相对中心或目标亮度的增益。
5. 对 gain map 做平滑，避免块状痕迹。
6. 限制最大 gain，防止边角噪声被过度放大。

## 学习重点

- LSC 不只是亮度校正，也是颜色均匀性校正。
- LSC 会放大边缘噪声，因此最大 gain 需要限制。
- LSC 和 AWB 耦合明显，边缘色偏会影响整体观感。

## 如果给 openISP 补实现

可设计接口：

```python
class LSC:
    def __init__(self, img, gain_maps, bayer_pattern, clip):
        ...
```

最小版本使用四张全尺寸 gain map；进阶版本使用 mesh gain map 加双线性插值。

## 面试问答

### Q1: LSC 为什么通常在 RAW 域做？

因为 shading 是传感器采样层面的位置和颜色响应问题。RAW 域可以按 R/Gr/Gb/B 精准校正，demosaic 后颜色已经混合，校正会更不准确。

### Q2: LSC 会带来什么副作用？

边角 gain 较大，会放大边角噪声，也可能放大坏点或暗电流。因此 LSC 需要配合 DPC、BLC、降噪和 gain 限制。

### Q3: 为什么要分 R/Gr/Gb/B 四张表？

不同颜色波长和不同绿色位置的 shading 程度不同。只用一张亮度表可能解决暗角，但无法解决角落偏色。

### Q4: Mesh LSC 为什么常见？

全尺寸表存储太大，径向模型又不够灵活。Mesh 表在存储、计算和校正精度之间比较平衡，适合硬件 ISP。

### Q5: LSC 和 AWB 的关系是什么？

LSC 处理空间维度的颜色和亮度不均，AWB 处理全局光源色温。LSC 没做好时，画面不同位置白点不同，AWB 很难同时兼顾中心和边缘。
