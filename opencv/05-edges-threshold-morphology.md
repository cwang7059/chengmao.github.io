# 05 - 边缘 / 阈值 / 形态学（二值处理的核心）

二值处理=把图像变成“前景/背景”两类。核心是：**先做合适的预处理 → 二值化 → 形态学清理 → 轮廓/测量**。

## 一张流程图（新手先照做）

1) 输入原图  
2) 去噪/平滑：`GaussianBlur` / `medianBlur`（视噪声类型选择）  
3) 转灰度（如必要：`cvtColor` → `COLOR_BGR2GRAY`）  
4) **阈值**：固定阈值 / Otsu / 自适应  
5) **形态学**：开运算去小噪点，闭运算补小洞（核大小 3~5 起步）  
6) 轮廓：`findContours`（到下一章继续）  
7) 可视化/统计：面积、周长、外接矩形

## 边缘检测（常用：Canny）

边缘 = 像素强度变化剧烈的位置。

### Canny 你要懂的点

- 前面最好先高斯去噪
- 两个阈值决定“边缘保留多少”
- 输出是“细边缘”，适合做轮廓前置步骤（但不是唯一方案）

## 阈值分割

### 什么时候用阈值

- 背景与前景有明显亮度差（或在某个颜色通道能分离）
- 想要一个“干净的黑白图”做面积/数量/形状测量

### 固定阈值 `threshold`

- **优点**：简单
- **缺点**：光照变化就不稳

**参数怎么调**：  
- `type` 选 `THRESH_BINARY` 或 `THRESH_BINARY_INV`（常用）  
- `thresh` 先从 128 试起，根据结果调高/调低  

**常见坑**：  
- 光照不均匀时会失败（见自适应阈值）

### 自适应阈值 `adaptiveThreshold`

- **适合**：光照不均、表面纹理

**它在做什么**：每个小块区域算自己的阈值，所以局部亮暗变化不会影响全局。  
**关键参数**：  
- `blockSize`（奇数，推荐 11/15/21 试起）  
- `C`：整体偏移，想让结果更“白”就减小 C，想更“黑”就增大 C  

### Otsu（大津法）

- **适合**：直方图双峰明显的场景

**白话**：自动找到能让前景/背景方差差异最大的阈值。  
**用法**：灰度图 + `THRESH_OTSU`，无需手动给 `thresh`。  
**失败场景**：直方图不是双峰（例如光照极不均）时效果差。

### 如何选哪一种

- 背景均匀 → 固定阈值 / Otsu  
- 背景不均匀 → 自适应阈值  
- 有明显双峰直方图 → Otsu 常有效  

## 形态学（工业视觉常用）

### 腐蚀 / 膨胀

- **腐蚀**：去小噪点、让前景“变瘦”
- **膨胀**：补小洞、让前景“变胖”

**结构元素（核）怎么选**：  
- 形状：`MORPH_RECT`（矩形，常用）、`MORPH_ELLIPSE`（圆润）、`MORPH_CROSS`（十字）  
- 尺寸：3×3 起步，过大易吞掉细节或填平细缝  

### 开运算 / 闭运算

- **开运算**（先腐蚀后膨胀）：去小噪点、保结构
- **闭运算**（先膨胀后腐蚀）：补洞、连断裂

**怎么用**：  
- 二值图先用 **开运算** 去小颗粒噪声  
- 再用 **闭运算** 把前景里的小洞补起来  
- 参数：核 3×3/5×5 先试，效果不好再调  

**常见坑**：  
- 核太大 → 细小结构被抹掉或连在一起  
- 顺序错误 → 先闭后开可能把缺陷填平，丢掉目标

## 推荐练习（1 小时）

1. 用不同阈值方法得到二值图（固定/自适应/Otsu）
2. 对二值图做开运算/闭运算，比较轮廓是否更“干净”
3. 把最终二值图交给 `findContours`（下一章）

### 练习参考解答/提示

- 练习 1：固定阈值 → 先试 `128`；自适应 → `blockSize=15, C=5` 起步；Otsu → 直接 `THRESH_OTSU`。对比：光照不均时固定阈值容易大片错分，自适应能保持局部细节，Otsu 在双峰直方图时最稳。
- 练习 2：开运算（去噪）先用核 `3×3`，若残留颗粒再试 `5×5`；闭运算（补洞）同样从 `3×3` 起步，洞大再增大核。观察：开后小噪点减少，闭后前景内部空洞被填平、断裂连上。
- 练习 3：`findContours` 后按面积过滤掉小轮廓（例如 `area < 50` 丢弃），再可视化外接矩形/面积。你应该能看到：经过合适阈值 + 形态学后，轮廓数量更少、更干净，面积/位置更稳定。

### 二值处理核心示例（C++，可直接跑）

```cpp
#include <opencv2/opencv.hpp>
int main(int argc, char** argv) {
    if (argc < 2) return 1;
    cv::Mat bgr = cv::imread(argv[1], cv::IMREAD_COLOR);
    if (bgr.empty()) return 1;
    cv::Mat gray; cv::cvtColor(bgr, gray, cv::COLOR_BGR2GRAY);

    // 1) 固定阈值
    cv::Mat bin_fixed;
    cv::threshold(gray, bin_fixed, 128, 255, cv::THRESH_BINARY);

    // 2) Otsu（自动阈值）
    cv::Mat bin_otsu;
    cv::threshold(gray, bin_otsu, 0, 255, cv::THRESH_BINARY | cv::THRESH_OTSU);

    // 3) 自适应阈值（光照不均匀时用）
    cv::Mat bin_adapt;
    cv::adaptiveThreshold(gray, bin_adapt, 255,
        cv::ADAPTIVE_THRESH_MEAN_C, cv::THRESH_BINARY, /*blockSize=*/15, /*C=*/5);

    // 4) 形态学：开运算去小噪点 + 闭运算补小洞
    cv::Mat kernel = cv::getStructuringElement(cv::MORPH_RECT, cv::Size(3, 3));
    cv::Mat bin_clean;
    cv::morphologyEx(bin_adapt, bin_clean, cv::MORPH_OPEN, kernel);  // 去噪
    cv::morphologyEx(bin_clean, bin_clean, cv::MORPH_CLOSE, kernel); // 补洞

    cv::imwrite("bin_fixed.png", bin_fixed);
    cv::imwrite("bin_otsu.png", bin_otsu);
    cv::imwrite("bin_adapt.png", bin_adapt);
    cv::imwrite("bin_clean.png", bin_clean);
    return 0;
}
```

运行方式：
```bash
g++ -std=c++17 demo_binary.cpp -o demo_binary `pkg-config --cflags --libs opencv4`
./demo_binary input.jpg
```

观察要点：
- `bin_fixed`：光照均匀时最简单，光照不均时容易大片错分
- `bin_otsu`：直方图双峰时自动找阈值，省调参
- `bin_adapt`：光照不均匀时仍能把局部分开
- `bin_clean`：经过开/闭后噪点减少、小洞填平，适合后续轮廓检测

下一章：`opencv/06-contours-and-geometry.md`

