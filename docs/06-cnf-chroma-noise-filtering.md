# 06. CNF: Chroma Noise Filtering

## 模块作用

CNF 是 Bayer 域色度噪声过滤模块，位于 AWB 之后、CFA 之前。它主要处理 R/B 采样点上的异常彩色噪声，避免这些彩噪在 demosaic 中扩散。

## openISP 实现

源码类名为 `CNF(img, bayer_pattern, thres, gain, clip)`，包含三步：

- `cnd()`：Chroma Noise Detection，检测色噪。
- `cnc()`：Chroma Noise Correction，修正色噪。
- `cnf()`：组合检测和修正。

## 检测思想

`cnd()` 在中心周围 8x8 区域统计：

- `avgG`：绿色采样平均值。
- `avgC1`：一种色彩采样平均值。
- `avgC2`：另一种色彩采样平均值。

若中心点明显高于 `avgG + thres` 和 `avgC2 + thres`，并且同类色彩均值也高，则认为可能是色噪。

## 修正思想

`cnc()` 计算中心点相对邻域的色彩突起：

```text
signalGap = center - max(avgG, avgC2)
chromaCorrected = max(avgG, avgC2) + dampFactor * signalGap
```

`dampFactor` 与 R/B gain 有关。gain 越大，说明该通道更可能放大噪声，修正越强。

最后用 `fadeTot` 在原值和修正值之间混合：

```text
center_out = (1 - fadeTot) * center + fadeTot * chromaCorrected
```

## 参数说明

| 参数 | 含义 |
| --- | --- |
| `thres` | 色噪检测阈值 |
| `gain` | AWB gain，用于决定阻尼强度 |
| `clip` | 输出上限 |
| `bayer_pattern` | 决定 R/B 位置 |

## 读源码注意点

- openISP 主程序传入 `thres=0`，检测会很敏感。
- `grbg` 分支中出现三维索引 `cnf_img[y, x, :]`，但 `cnf_img` 是二维数组，疑似源码问题。
- 该模块是经验规则，适合理解色噪抑制，不是通用最优算法。

## 面试问答

### Q1: 为什么 CNF 要放在 CFA 前？

因为 CFA 会把单个 R/B 异常点插值扩散到周围多个 RGB 像素。提前在 Bayer 域抑制彩噪，可以减少后续伪彩和色斑。

### Q2: 色度噪声为什么常见于 R/B 通道？

很多传感器绿色通道响应更强，R/B 通道在某些光源或低照下信号更弱，需要更大 AWB gain，因此噪声更容易被放大。

### Q3: CNF 过强有什么副作用？

会降低色彩细节和饱和度，可能把真实小面积彩色纹理当作噪声抹掉。

### Q4: CNF 和 FCS 有什么区别？

CNF 在 Bayer 域处理 R/B 异常采样点，主要防止彩噪扩散。FCS 在 YUV 域根据边缘图压低 UV，主要抑制 demosaic 后的边缘伪彩。

### Q5: 如何验证 CNF 效果？

可以在 RAW 的 R/B 位置人工加入孤立高值，观察 CFA 前后彩点是否减少；也可以比较关闭 CNF 后暗部彩噪变化。
