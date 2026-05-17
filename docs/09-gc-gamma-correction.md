# 09. GC: Gamma Correction

## 模块作用

Gamma Correction 是非线性亮度映射模块，用 LUT 把线性或近似线性的 RGB 信号映射到更适合显示和人眼感知的曲线。

人眼对暗部变化更敏感，gamma 可以把更多编码范围分配给暗部和中间调，让图像观感更自然。

## openISP 实现

源码类名为 `GC(img, lut, mode)`。主程序构造 10-bit LUT：

```python
bw = 10
gamma = 0.5
maxval = 2 ** bw
val = round((i / maxval) ** gamma * maxval)
```

在 `mode == "rgb"` 时，R/G/B 分别查表，然后除以 4，把 10-bit 映射到约 8-bit。

## 核心公式

连续形式可写成：

```text
out = (in / maxval) ^ gamma * maxval
```

当 `gamma < 1` 时，暗部会被提亮；当 `gamma > 1` 时，暗部会被压暗。

## LUT 的意义

真实硬件 ISP 常用 LUT，而不是实时算指数函数。LUT 可以快速实现任意曲线，包括 gamma、S-curve、分段 tone curve。

## 读源码注意点

主 pipeline 中计算了 `rgbimg_gc`，但后续 `CSC` 使用的是 `rgbimg_ccm`，不是 `rgbimg_gc`。也就是说当前代码路径中 gamma 结果没有真正进入最终 YUV 输出。

## 学习重点

- Gamma 影响整体影调，不只是“变亮”。
- Gamma 和 bit depth 有强关系，LUT 长度要匹配输入范围。
- Gamma 之前的 clipping 和之后的量化都会影响最终细节。

## 面试问答

### Q1: Gamma 校正为什么需要？

显示设备和人眼感知都不是线性的。Gamma 让编码值更符合视觉敏感度，尤其改善暗部和中间调表现。

### Q2: Gamma 和 tone mapping 有什么区别？

Gamma 是一种简单全局非线性曲线。Tone mapping 更广，可以包含高光压缩、局部对比、HDR 映射、S 曲线等。

### Q3: 为什么 ISP 常用 LUT 做 gamma？

LUT 计算快、硬件友好、可配置性强。复杂曲线也可以通过查表实现，不需要昂贵的实时指数运算。

### Q4: `gamma=0.5` 会产生什么效果？

相当于开平方，会提升暗部和中间调，让图像更亮。但如果过强，会让黑位发灰、噪声更明显。

### Q5: Gamma 放在 RGB 还是 YUV 做？

两种都可见。RGB gamma 会分别影响颜色通道；Y 通道 gamma 更偏亮度曲线控制。具体取决于 pipeline 设计和色彩管理目标。
