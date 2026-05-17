# openISP ISP Pipeline 学习文档

本文档基于 GitHub 项目 `cruxopen/openISP` 的 `isp_pipeline.py`、`model/*.py` 和 `config/config.csv` 整理。目标是把 openISP 的每个模块拆成独立学习材料，并补充常见面试问答，方便按 ISP pipeline 系统学习。

## Pipeline 总览

openISP 的主流程可以按数据域分为三段：

```text
RAW Bayer
  -> DPC: Dead Pixel Correction
  -> BLC: Black Level Compensation
  -> LSC: Lens Shading Correction, README 提到但源码未实现
  -> AAF: Anti-aliasing Filter
  -> AWB/WBGC: White Balance Gain Control
  -> CNF: Chroma Noise Filtering
  -> CFA: Demosaic / Color Filter Array Interpolation
  -> RGB
  -> CCM: Color Correction Matrix
  -> GC: Gamma Correction
  -> CSC: Color Space Conversion
  -> YUV
  -> NLM: Non-local Means Denoising, Y channel
  -> BNF: Bilateral Noise Filtering, Y channel
  -> EE: Edge Enhancement, Y channel
  -> FCS: False Color Suppression, UV channel with edge map
  -> HSC: Hue Saturation Control, UV channel
  -> BCC: Brightness Contrast Control, Y channel
  -> Final YUV
```

## 学习顺序建议

1. 先学 Bayer pattern：理解 `RGGB/BGGR/GBRG/GRBG` 的坐标关系。
2. 再学 RAW 域模块：`DPC/BLC/LSC/AAF/AWB/CNF`。
3. 接着学 demosaic：`CFA` 是 RAW 到 RGB 的关键转折点。
4. 然后学颜色模块：`CCM/GC/CSC`。
5. 最后学 YUV 调校模块：`NLM/BNF/EE/FCS/HSC/BCC`。

## 文档目录

| 顺序 | 模块 | 源码 | 文档 |
| --- | --- | --- | --- |
| 01 | DPC | `model/dpc.py` | [01-dpc-dead-pixel-correction.md](01-dpc-dead-pixel-correction.md) |
| 02 | BLC | `model/blc.py` | [02-blc-black-level-compensation.md](02-blc-black-level-compensation.md) |
| 03 | LSC | README 提到，源码未实现 | [03-lsc-lens-shading-correction.md](03-lsc-lens-shading-correction.md) |
| 04 | AAF | `model/aaf.py` | [04-aaf-anti-aliasing-filter.md](04-aaf-anti-aliasing-filter.md) |
| 05 | AWB/WBGC | `model/awb.py` | [05-awb-white-balance-gain-control.md](05-awb-white-balance-gain-control.md) |
| 06 | CNF | `model/cnf.py` | [06-cnf-chroma-noise-filtering.md](06-cnf-chroma-noise-filtering.md) |
| 07 | CFA | `model/cfa.py` | [07-cfa-demosaic.md](07-cfa-demosaic.md) |
| 08 | CCM | `model/ccm.py` | [08-ccm-color-correction-matrix.md](08-ccm-color-correction-matrix.md) |
| 09 | GC | `model/gac.py` | [09-gc-gamma-correction.md](09-gc-gamma-correction.md) |
| 10 | CSC | `model/csc.py` | [10-csc-color-space-conversion.md](10-csc-color-space-conversion.md) |
| 11 | NLM | `model/nlm.py` | [11-nlm-non-local-means.md](11-nlm-non-local-means.md) |
| 12 | BNF | `model/bnf.py` | [12-bnf-bilateral-noise-filter.md](12-bnf-bilateral-noise-filter.md) |
| 13 | EE | `model/eeh.py` | [13-ee-edge-enhancement.md](13-ee-edge-enhancement.md) |
| 14 | FCS | `model/fcs.py` | [14-fcs-false-color-suppression.md](14-fcs-false-color-suppression.md) |
| 15 | HSC | `model/hsc.py` | [15-hsc-hue-saturation-control.md](15-hsc-hue-saturation-control.md) |
| 16 | BCC | `model/bcc.py` | [16-bcc-brightness-contrast-control.md](16-bcc-brightness-contrast-control.md) |

## 全局面试题

### Q1: 为什么 ISP pipeline 前半段要在 Bayer 域处理？

因为 RAW Bayer 是传感器最原始的数据，坏点、黑电平、镜头 shading、白平衡增益和部分色噪都直接发生在采样点上。如果等到 demosaic 之后再处理，异常会被插值扩散到多个颜色通道和多个像素，问题更难定位，也更容易产生伪彩。

### Q2: 为什么常见流程是 BLC 在 AWB 和 CCM 前面？

BLC 修正的是信号基线。若黑电平未校正，AWB 会在错误的通道均值上估计或应用 gain，CCM 也会把错误偏置混到其他通道里。先把黑位拉准，后续颜色处理才有可靠输入。

### Q3: RAW 域、RGB 域和 YUV 域各适合处理什么？

RAW 域适合处理传感器采样问题，如 DPC、BLC、LSC、AWB、Bayer 降噪。RGB 域适合颜色还原和显示前 tone 处理，如 CCM、Gamma。YUV 域适合把亮度和色度分开处理，如 Y 通道降噪锐化、UV 通道饱和度和伪彩抑制。

### Q4: openISP 更像教学项目还是量产 ISP？

更像教学和算法验证项目。它模块齐全、代码直接、便于学习 pipeline，但大量实现是 Python 逐像素循环，缺少真实量产系统中的统计模块、自动控制、分段曲线、硬件约束、性能优化和完整标定工具。

### Q5: 面试中如何介绍一个 ISP pipeline？

可以按“数据域 + 模块目的 + 副作用”讲：RAW 域先修传感器和镜头问题，再 demosaic 到 RGB；RGB 域做颜色校正和 gamma；YUV 域做亮度降噪锐化、色度控制和输出风格。强调每个模块都有权衡，例如降噪会损细节，锐化会放大噪声，饱和度会放大色噪。
