# 04 - 滤波与去噪（把图像“洗干净”）

## 你什么时候需要滤波

- 相机噪声/压缩噪声导致边缘检测不稳
- 二值化出现大量小碎点
- 后续要做轮廓/测量，想让结果更稳定

## 一个“跑完就懂”的 C++ 对比实验（手写 vs OpenCV）

目标：对同一张图片分别做 **手写滤波** 与 **OpenCV 滤波**，把输出结果保存到文件里对比。

### 编译方式（Ubuntu/WSL 常见）

前提：你已安装 `libopencv-dev`（见 `opencv/01-install.md`）。

```bash
g++ -std=c++17 demo_filtering.cpp -o demo_filtering `pkg-config --cflags --libs opencv4`
./demo_filtering input.jpg
```

### 完整示例：`demo_filtering.cpp`

说明：

- 为了更清晰对比，示例统一在 **灰度图** 上做滤波（更适合新手理解）
- 你会看到两类实现：
  - **手写**：均值滤波（卷积思想）、中值滤波（排序取中位数）
  - **OpenCV**：`blur` / `GaussianBlur` / `medianBlur` / `bilateralFilter`

```cpp
#include <opencv2/opencv.hpp>
#include <algorithm>
#include <iostream>
#include <vector>

// 边界处理函数：把越界坐标"夹住"到合法范围
// 策略：BORDER_REPLICATE（复制边缘）- 越界就用最近的边缘像素值
// 例如：clampInt(-1, 0, 99) → 0，clampInt(100, 0, 99) → 99
// 详细原理见下方"边界处理策略详解"章节
static inline int clampInt(int v, int lo, int hi) {
    return std::max(lo, std::min(v, hi));
}

// 手写：均值滤波（box filter）
// - ksize 必须是奇数（3,5,7...）
// - 边界：使用 clamp（等价于 BORDER_REPLICATE 的直觉效果）
// ⚠️ 注意：边界处可能出现"黑边"或"亮边"，因为边界像素被重复计算但分母仍用 ksize²
cv::Mat meanFilterManualU8(const cv::Mat& grayU8, int ksize) {
    CV_Assert(grayU8.type() == CV_8UC1);
    CV_Assert(ksize % 2 == 1 && ksize >= 1);

    int r = ksize / 2;
    cv::Mat out(grayU8.size(), CV_8UC1);

    for (int y = 0; y < grayU8.rows; ++y) {
        for (int x = 0; x < grayU8.cols; ++x) {
            int sum = 0;
            for (int dy = -r; dy <= r; ++dy) {
                for (int dx = -r; dx <= r; ++dx) {
                    int yy = clampInt(y + dy, 0, grayU8.rows - 1);
                    int xx = clampInt(x + dx, 0, grayU8.cols - 1);
                    sum += grayU8.at<uchar>(yy, xx);
                }
            }
            int n = ksize * ksize;
            out.at<uchar>(y, x) = static_cast<uchar>(sum / n);
        }
    }
    return out;
}

// 手写：中值滤波（median）
// - 对每个像素取邻域窗口，排序取中位数
// - 适合椒盐噪声，但实现成本更高
cv::Mat medianFilterManualU8(const cv::Mat& grayU8, int ksize) {
    CV_Assert(grayU8.type() == CV_8UC1);
    CV_Assert(ksize % 2 == 1 && ksize >= 1);

    int r = ksize / 2;
    cv::Mat out(grayU8.size(), CV_8UC1);
    std::vector<uchar> window;
    window.reserve(ksize * ksize);

    for (int y = 0; y < grayU8.rows; ++y) {
        for (int x = 0; x < grayU8.cols; ++x) {
            window.clear();
            for (int dy = -r; dy <= r; ++dy) {
                for (int dx = -r; dx <= r; ++dx) {
                    int yy = clampInt(y + dy, 0, grayU8.rows - 1);
                    int xx = clampInt(x + dx, 0, grayU8.cols - 1);
                    window.push_back(grayU8.at<uchar>(yy, xx));
                }
            }
            // nth_element：找中位数不必全排序（比 sort 更快，且足够用）
            auto midIt = window.begin() + window.size() / 2;
            std::nth_element(window.begin(), midIt, window.end());
            out.at<uchar>(y, x) = *midIt;
        }
    }
    return out;
}

int main(int argc, char** argv) {
    if (argc < 2) {
        std::cerr << "Usage: " << argv[0] << " input.jpg\n";
        return 1;
    }

    cv::Mat bgr = cv::imread(argv[1], cv::IMREAD_COLOR);
    if (bgr.empty()) {
        std::cerr << "Failed to read image: " << argv[1] << "\n";
        return 1;
    }

    cv::Mat gray;
    cv::cvtColor(bgr, gray, cv::COLOR_BGR2GRAY);

    int k = 5; // 改这个试试：3/5/7...

    // 手写
    cv::Mat mean_manual = meanFilterManualU8(gray, k);
    cv::Mat median_manual = medianFilterManualU8(gray, k);

    // OpenCV API
    cv::Mat mean_cv, gauss_cv, median_cv, bilateral_cv;
    cv::blur(gray, mean_cv, cv::Size(k, k));
    cv::GaussianBlur(gray, gauss_cv, cv::Size(k, k), 0 /*sigmaX*/);
    cv::medianBlur(gray, median_cv, k);
    // bilateral 参数：
    // - d：邻域直径（-1 表示由 sigma 推导）
    // - sigmaColor：颜色差异阈值（越大越“敢”跨边缘）
    // - sigmaSpace：空间距离阈值（越大影响范围越大）
    cv::bilateralFilter(gray, bilateral_cv, /*d*/ 9, /*sigmaColor*/ 50, /*sigmaSpace*/ 50);

    cv::imwrite("00_gray.png", gray);
    cv::imwrite("01_mean_manual.png", mean_manual);
    cv::imwrite("02_mean_cv.png", mean_cv);
    cv::imwrite("03_gaussian_cv.png", gauss_cv);
    cv::imwrite("04_median_manual.png", median_manual);
    cv::imwrite("05_median_cv.png", median_cv);
    cv::imwrite("06_bilateral_cv.png", bilateral_cv);

    // Canny 边缘检测对比（理解不同滤波对边缘检测的影响）
    // Canny 参数说明：
    //   threshold1（低阈值）：弱边缘的阈值，低于此值直接丢弃
    //   threshold2（高阈值）：强边缘的阈值，高于此值直接保留
    //   通常 threshold2 ≈ 2-3 × threshold1（例如 50, 150）
    //   调参建议：先用 (50, 150)，边缘太多就调高，边缘太少就调低
    cv::Mat edges_gray, edges_mean_cv, edges_gauss_cv, edges_median_cv, edges_bilateral_cv;
    cv::Canny(gray, edges_gray, 50, 150);
    cv::Canny(mean_cv, edges_mean_cv, 50, 150);
    cv::Canny(gauss_cv, edges_gauss_cv, 50, 150);  // 高斯滤波 + Canny 是最常用组合
    cv::Canny(median_cv, edges_median_cv, 50, 150);
    cv::Canny(bilateral_cv, edges_bilateral_cv, 50, 150);

    cv::imwrite("07_edges_gray.png", edges_gray);
    cv::imwrite("08_edges_mean_cv.png", edges_mean_cv);
    cv::imwrite("09_edges_gauss_cv.png", edges_gauss_cv);
    cv::imwrite("10_edges_median_cv.png", edges_median_cv);
    cv::imwrite("11_edges_bilateral_cv.png", edges_bilateral_cv);

    std::cout << "Done. Wrote output images:\n";
    std::cout << "  Filtered: 00_gray.png .. 06_bilateral_cv.png\n";
    std::cout << "  Edges:    07_edges_gray.png .. 11_edges_bilateral_cv.png\n";
    std::cout << "\n观察建议：\n";
    std::cout << "  - 原图直接 Canny：噪声多，边缘不干净\n";
    std::cout << "  - 高斯滤波 + Canny：边缘最干净（工业常用）\n";
    std::cout << "  - 双边滤波 + Canny：边缘保留好，但可能仍有噪声\n";
    return 0;
}
```

### 你应该从这个实验看懂什么（关键解释）

- **“滤波=邻域运算”**：每个输出像素都由它周围一个窗口（核）计算得到。
- **核大小 ksize 越大**：
  - 平滑更强、噪声更少
  - 细节损失更多、边缘更糊
  - 计算更慢（手写循环尤其明显）
- **为什么手写要特别注意边界**：
  - 到边缘时窗口会"越界"，必须决定怎么取值
  - 示例用了最简单的策略：把越界坐标 clamp 到边界（直觉上像"复制边缘像素"）
  - OpenCV 里这是一个可配置项（不同边界策略会影响边缘附近效果）
- **⚠️ 为什么边界处可能出现"黑边"或"亮边"**：
  - 在边界处（例如 `(0,0)` 用 5×5 窗口），实际只有 3×3=9 个有效像素
  - 但 `clampInt` 会把越界坐标拉回边界，导致边界像素（如 `(0,0)`）被**重复计算多次**
  - 代码用 `n = ksize * ksize = 25` 做除法，平均值会被拉偏
  - 如果边界像素值较小 → 平均值偏低 → 看起来像"黑边"
  - 如果边界像素值较大 → 平均值偏高 → 看起来像"亮边"
  - **这是手写简化版的常见问题**，OpenCV 的实现会更精确地处理边界
- **为什么手写均值用 `int sum`**：
  - `uint8` 的 0~255 很容易在求和时溢出
  - 先用更大范围的整型累计，再除以 N，最后转回 `uint8`
- **为什么"OpenCV 的结果通常更快/更稳"**：
  - 内部做了 SIMD、多线程、缓存优化
  - 对边界、类型、对齐等细节处理更成熟
  - 手写主要用于：学习原理 / 自定义算子 / 做研究验证
- **为什么要在滤波后做 Canny 边缘检测对比**：
  - **原图直接 Canny**：噪声会被当作边缘，结果很乱
  - **滤波后再 Canny**：噪声被平滑掉，真正的边缘更清晰
  - **高斯滤波 + Canny**：工业最常用组合，边缘干净且稳定
  - **双边滤波 + Canny**：能保留边缘细节，但去噪能力不如高斯
  - **对比不同滤波的 Canny 结果**：帮你理解"哪种滤波最适合你的场景"

## 先把 3 个最核心概念讲明白（新手看完就能用）

### 1) 什么叫“滤波”

滤波本质就是：**对每个像素，看它周围一圈邻居，然后算一个新值替换它**。

- **输入**：原图像素值
- **规则**：一个“窗口/核”（例如 3×3、5×5）
- **输出**：新图像素值

所以滤波一定是“邻域运算”，不会只看一个点。

### 2) 什么叫“核 / 卷积”（理解线性滤波的关键）

对灰度图最直观：给你一个 3×3 的核（每个位置一个权重），它会做：

- **加权求和**：邻域像素 × 权重，然后相加
- **归一化**：有些核还会除以权重和（例如均值核）

你可以把它想成“把邻居按不同重要性混合一下”。

> 你在本章的手写均值滤波里，其实已经在做“卷积”的简化版（权重全相等）。

### 3) 为什么边界需要特殊处理（BORDER）

核在图像边缘会"伸出去"，例如你在 `(0,0)` 做 5×5 邻域时，会访问到 `(-2,-2)` 这种越界坐标。

常见处理策略（先记住含义即可）：

- **复制边缘（replicate）**：越界就用最近的边缘像素（本章手写用的 clamp 就是这个直觉）
- **镜像（reflect）**：把图像当镜子对折映射
- **常数填充（constant）**：越界当成 0 或固定值（可能引入黑边）

工程建议：**如果你看到边缘出现奇怪暗边/亮边，先怀疑是边界策略 + 核大小造成的。**

#### 边界处理策略详解（代码 + 效果对比）

**为什么代码里用 `clampInt`？**

```cpp
// 把访问边界的 (x, y) "夹住"到合法范围（最简单的边界策略：replicate）
static inline int clampInt(int v, int lo, int hi) {
    return std::max(lo, std::min(v, hi));
}
```

这个函数做的事情：
- 如果 `v < lo`，返回 `lo`（下边界）
- 如果 `v > hi`，返回 `hi`（上边界）
- 否则返回 `v`（在范围内）

**在滤波中的效果**：
- 当窗口越界到 `(-1, 5)` 时，`clampInt(-1, 0, rows-1)` 会返回 `0`
- 相当于"把越界坐标拉回到最近的边缘像素位置"
- 这就是 **BORDER_REPLICATE**（复制边缘）的直觉实现

**其他边界策略的实现思路**：

1. **BORDER_REPLICATE（复制边缘）** - 本章用的方法
   ```cpp
   int yy = clampInt(y + dy, 0, grayU8.rows - 1);
   int xx = clampInt(x + dx, 0, grayU8.cols - 1);
   ```
   - 效果：边缘像素会被"复制"到越界区域
   - 适用：大多数情况，简单且稳定

2. **BORDER_REFLECT（镜像）**
   ```cpp
   int reflect(int v, int lo, int hi) {
       if (v < lo) return lo + (lo - v);
       if (v > hi) return hi - (v - hi);
       return v;
   }
   ```
   - 效果：图像边缘像镜子一样反射
   - 适用：希望边界过渡更自然（但实现更复杂）

3. **BORDER_CONSTANT（常数填充）**
   ```cpp
   uchar getPixel(const cv::Mat& img, int y, int x, uchar fillValue = 0) {
       if (y < 0 || y >= img.rows || x < 0 || x >= img.cols)
           return fillValue;  // 越界返回固定值（通常是 0 或 255）
       return img.at<uchar>(y, x);
   }
   ```
   - 效果：越界区域是固定值（黑边或白边）
   - 适用：明确知道边界应该是什么值

**在 OpenCV 中如何指定边界策略**：

```cpp
// OpenCV 的滤波函数通常支持 borderType 参数
cv::GaussianBlur(src, dst, cv::Size(5, 5), 0, 0, cv::BORDER_REPLICATE);
cv::blur(src, dst, cv::Size(5, 5), cv::Point(-1, -1), cv::BORDER_REFLECT);

// 常见枚举值：
// - cv::BORDER_REPLICATE：复制边缘（默认，最常用）
// - cv::BORDER_REFLECT：镜像
// - cv::BORDER_CONSTANT：常数填充（需要额外指定 fillValue）
// - cv::BORDER_REFLECT_101：镜像但边界不重复（更平滑）
```

**实际影响**：
- 边界策略主要影响**图像边缘附近约 `ksize/2` 像素宽的区域**
- 如果核很小（3×3），影响范围小，不同策略差异不明显
- 如果核很大（15×15），边缘效果差异会很明显
- **新手建议**：先用 `BORDER_REPLICATE`（OpenCV 默认），遇到边缘问题再换策略

## 线性滤波

### 均值滤波 `blur`

- **优点**：快
- **缺点**：边缘变糊

#### 详细释义（白话 + 原理）

均值滤波就是：**把窗口里所有像素取平均**。等价于用一个“全是 1 的核”做卷积再除以 \(N\)。

它做的事情很朴素：把“局部的小波动”平均掉，所以画面更平滑。

#### 适合什么噪声

- 对“细碎的小幅随机噪声”有效（轻微颗粒感、抖动）
- 对“椒盐噪声”（少量极端黑点/白点）不够好（会把离群点影响扩散到周围）

#### 参数怎么调（ksize）

- `ksize=3`：轻度平滑，细节保留更多
- `ksize=5/7`：平滑更强，但边缘更糊，细小结构更容易消失

### 高斯滤波 `GaussianBlur`

- **特点**：比均值更“自然”，边缘保留略好
- **典型用法**：Canny 前几乎必配

#### 详细释义（白话 + 原理）

高斯滤波也是“加权平均”，但权重不是一样的：

- **离中心越近权重越大**
- **离中心越远权重越小**

这相当于“更相信中心附近的像素”，所以视觉上更自然，也更适合当作边缘检测的前置去噪。

#### 参数怎么调（ksize / sigmaX）

- `ksize`：窗口大小（奇数）
- `sigmaX`：模糊强度（越大越糊）
  - `sigmaX=0`：OpenCV 会根据 `ksize` 自动估一个合理值（新手推荐先用这个）

## 非线性滤波

### 中值滤波 `medianBlur`

- **专治**：椒盐噪声
- **代价**：核大时会慢

#### 详细释义（白话 + 原理）

中值滤波不是平均，而是：

1. 把窗口里的像素值拿出来
2. 找到“中间那个值”（中位数）
3. 用它替换中心像素

#### 为什么它“专治椒盐噪声”

椒盐噪声是“极端离群点”（突然变 0 或 255）。平均会被离群点拉偏；中位数对离群点不敏感，所以非常稳。

#### 参数怎么调（ksize）

- `ksize=3`：轻度去椒盐
- `ksize=5`：更强去噪，但也更容易抹掉细小结构（例如细线、尖角）

### 双边滤波 `bilateralFilter`

- **效果**：平滑区域 + 保边
- **缺点**：很慢（实时项目谨慎）

#### 详细释义（白话 + 原理）

双边滤波做平均时，会同时考虑两件事：

- **空间距离近不近**（离得近才参与）
- **像素值像不像**（颜色/灰度差太大就不参与）

所以在边缘处（两侧像素值差很大），它不会把两侧硬平均到一起，边缘就能保留下来。

#### 参数怎么调（d / sigmaColor / sigmaSpace）

- `sigmaColor`：像素值差异容忍度
  - 小：更保边，但去噪弱
  - 大：更敢跨边缘平均，可能把边缘也抹掉
- `sigmaSpace`：空间距离影响范围
  - 小：只看很近的邻居
  - 大：更大范围参与，平滑更强也更慢

新手建议：先固定 `d=9`，只调 `sigmaColor/sigmaSpace`，看输出变化最直观。

## 学完这一章你应该达到什么水平（验收标准）

- **能解释清楚**：为什么滤波是“邻域窗口运算”，核大小变大意味着什么代价与收益
- **能自己写出**：最基本的均值滤波（含边界处理、类型不溢出）
- **能选对工具**：
  - 椒盐噪声 → 先想 `medianBlur`
  - Canny 前 → 优先 `GaussianBlur`
  - 想保边但不追求速度 → 试 `bilateralFilter`（知道它慢）
- **能做工程对比**：同一张图输出多张结果图并给出“我选哪个/为什么”的判断

## 推荐练习（30 分钟）

1. 给同一张图分别做：均值/高斯/中值/双边
2. 每种滤波后都跑一遍 Canny，把边缘图保存对比
3. 记录你认为“最好用”的参数组合（核大小/σ）

下一章：`opencv/05-edges-threshold-morphology.md`

