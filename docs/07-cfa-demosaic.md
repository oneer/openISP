# 07. CFA: Color Filter Array Interpolation / Demosaic

## 模块作用

CFA 插值也叫 demosaic，是把单通道 Bayer RAW 转成三通道 RGB 的关键模块。每个传感器像素只采集 R/G/B 之一，CFA 要为每个位置估计缺失的两个颜色通道。

## openISP 实现

源码类名为 `CFA(img, mode, bayer_pattern, clip)`。实际实现的模式是 `malvar`，支持四种 Bayer pattern。

输入：

```text
H x W Bayer RAW
```

输出：

```text
H x W x 3 RGB
```

## Bayer pattern

以 `rggb` 为例：

```text
R  Gr
Gb B
```

代码每次处理一个 2x2 Bayer block，并根据当前位置是 `r/gr/gb/b` 调用不同插值公式。

## Malvar 插值思想

Malvar-He-Cutler 类算法不是简单双线性平均。它会根据中心点和周围像素构造线性滤波公式，把局部梯度信息纳入缺失通道估计。

直觉：

- 平坦区域：接近平滑插值。
- 边缘区域：利用梯度校正，减少跨边缘混色。
- 高频区域：比简单双线性更能抑制伪彩。

## 代码流程

1. 对 RAW 做 2 像素 reflect padding。
2. 转成 `int32`，避免中间乘加溢出。
3. 按 2x2 block 遍历。
4. 根据 `bayer_pattern` 判断每个位置颜色。
5. 调用 `malvar()` 得到 `[r, g, b]`。
6. 写入 `cfa_img` 并裁剪。

## 参数说明

| 参数 | 含义 |
| --- | --- |
| `mode` | 插值模式，openISP 实现 `malvar` |
| `bayer_pattern` | Bayer 排列，必须与 RAW 一致 |
| `clip` | RGB 输出上限，源码按 RAW clip |

## 常见问题

Bayer pattern 错误会导致严重偏色，例如把 `rggb` 当 `bggr`，红蓝会互换。CFA 前的坏点、混叠、色噪也会在 CFA 后被扩散成可见伪彩。

## 面试问答

### Q1: Demosaic 为什么是 ISP 中很重要的模块？

它决定了 RAW 到 RGB 的基本质量。插值不好会造成伪彩、锯齿、拉链边、细节模糊，这些问题后续很难完全修复。

### Q2: 双线性插值和 Malvar 插值有什么区别？

双线性主要用邻域平均，简单但容易模糊和伪彩。Malvar 使用更大的滤波模板和梯度校正，能更好地兼顾细节和颜色准确性。

### Q3: Bayer pattern 配错会怎样？

颜色通道位置错乱，通常表现为严重偏色、红蓝互换、边缘彩色纹理异常。因为从最基础采样位置开始就解释错了。

### Q4: CFA 前为什么要做 DPC/AAF/CNF？

这些模块分别减少坏点、高频混叠和色噪。若不提前处理，CFA 会把局部异常扩散到 RGB 图像中。

### Q5: 如何评价 demosaic 算法好坏？

可以看解析力、伪彩、拉链边、噪声放大、计算复杂度。测试图通常包括斜线、细网格、彩色边缘和真实纹理。
