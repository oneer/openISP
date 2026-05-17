# 14. FCS: False Color Suppression

## 模块作用

FCS 是伪彩抑制模块，作用在 UV 色度通道，同时参考 EE 生成的 edge map。它在强边缘处降低色度强度，以减少 demosaic 或锐化带来的彩边和摩尔纹。

## openISP 实现

源码类名为 `FCS(img, edgemap, fcs_edge, gain, intercept, slope)`。

输入 `img` 是 UV 两通道：

```python
yuvimg_csc[:, :, 1:3]
```

## 核心逻辑

根据 edge map 强度决定 `uvgain`：

```text
abs(edge) <= edge_min:
    uvgain = gain
edge_min < abs(edge) < edge_max:
    uvgain = intercept - slope * edge
else:
    uvgain = 0
```

然后：

```text
UV' = uvgain * UV / 256 + 128
```

边缘越强，越压低色度，让强边缘更接近中性色，减少彩边。

## 参数说明

| 参数 | 含义 |
| --- | --- |
| `fcs_edge_min/max` | 开始和完全抑制的边缘阈值 |
| `fcs_gain` | 平坦区 UV gain |
| `intercept/slope` | 中间过渡区斜率 |

## 学习重点

- FCS 是用亮度边缘指导色度处理。
- 过强会让边缘发灰、颜色变淡。
- 过弱则彩边和伪彩残留。

## 读源码注意点

中间区公式使用 `self.edgemap[y,x]` 而不是 `abs(edge)`，正负边缘可能得到不同 gain。真实系统通常会明确设计正负响应处理。

## 面试问答

### Q1: 什么是 false color？

false color 是图像中不存在于真实场景的错误颜色，常出现在细密纹理、强边缘、重复图案处，多由采样和 demosaic 引起。

### Q2: FCS 为什么参考 edge map？

伪彩常出现在高频亮度边缘。edge map 能指出这些高风险位置，FCS 在这些位置压低色度更有针对性。

### Q3: FCS 和降低饱和度有什么区别？

全局降低饱和度会影响整幅图颜色。FCS 通常只在边缘或高风险区域抑制 UV，更局部、更有选择性。

### Q4: FCS 过强会怎样？

边缘颜色会被压灰，彩色细节丢失，画面可能显得不够鲜艳。

### Q5: 如何测试 FCS？

用细线、织物、栅格、文字边缘等高频图案，比较 FCS 前后的彩边和色度保留。
