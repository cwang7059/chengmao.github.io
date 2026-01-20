一、滤波（Filtering / Smoothing）
1️⃣ 什么是滤波？

滤波 = 对像素邻域做加权计算，抑制噪声 / 平滑图像

现实世界的图像：

传感器噪声

光照抖动

工业相机热噪声

高 ISO 噪点

👉 所以在 任何分析之前，几乎都要先滤波

2️⃣ 线性滤波（Linear Filter）
🔹 均值滤波 cv::blur

含义

一个像素 = 周围邻域像素的平均值

本质：卷积核里全是 1/N

公式

dst(x,y) = (1/N) * Σ src(x+i, y+j)


特点

优点：快

缺点：模糊边缘（边缘被“抹平”）

什么时候用

噪声不敏感

对速度要求高

预处理阶段

🔹 高斯滤波 cv::GaussianBlur

含义

邻域像素不是等权

离中心越近 → 权重越大

为什么高斯？

数学上平滑性最好

不引入额外振铃

接近真实物理模糊

特点

比均值慢

保边能力略好

工业用途

工业相机

边缘检测前（Canny 强烈依赖）

3️⃣ 非线性滤波（Non-linear Filter）
🔹 中值滤波 cv::medianBlur

含义

邻域排序，取中间值

能干什么

专杀 椒盐噪声

不会拉低边缘

代价

排序 → 慢

不适合大核

🔹 双边滤波 cv::bilateralFilter

核心思想

“只跟空间近 + 像素值相近的点平均”

效果

平滑区域 → 平滑

边缘 → 保留

缺点

非常慢

工业实时很少用

二、边缘检测（Edge Detection）
1️⃣ 什么是边缘？

边缘 = 像素值变化剧烈的位置

数学本质：

一阶导数大

或二阶导数过零

2️⃣ Sobel / Scharr / Laplacian
🔹 Sobel

干什么

计算 X / Y 方向梯度

Gx = ∂I/∂x
Gy = ∂I/∂y


用途

边缘强度

方向信息

🔹 Scharr

Sobel 的改进版

小核时更精确

工业里常用 Scharr 替代 Sobel

🔹 Laplacian（二阶导）

对噪声极敏感

一般配合高斯使用（LoG）

3️⃣ Canny 边缘检测（工业级）

为什么 Canny 强？

高斯去噪

梯度计算

非极大值抑制（瘦边）

双阈值连接

特点

边细

稳定

工业常用

三、形态学（Morphology）

这是 二值图像 / 边缘处理的灵魂

1️⃣ 腐蚀 erode

含义

前景收缩

去小噪点

理解

“瘦身”

2️⃣ 膨胀 dilate

含义

前景扩张

填小洞

理解

“长胖”

3️⃣ 开运算 / 闭运算
🔹 开运算（先腐蚀后膨胀）

去小噪点

保留大结构

🔹 闭运算（先膨胀后腐蚀）

填洞

连通断裂

4️⃣ 顶帽 / 黑帽

顶帽：提取亮小结构

黑帽：提取暗小结构

工业缺陷检测常用。

四、阈值分割（Segmentation）
1️⃣ 固定阈值 threshold

最简单

src > T ? 255 : 0


问题

光照变化就完蛋

2️⃣ 自适应阈值 adaptiveThreshold

思想

每个区域算自己的阈值

适用

光照不均

表面检测

3️⃣ Otsu（大津法）

核心

自动找最大类间方差

前提

双峰直方图

五、几何变换（Geometric Transform）
1️⃣ 仿射变换 warpAffine

平移

旋转

缩放

剪切

保持平行性

2️⃣ 透视变换 warpPerspective

四点映射

非平行

相机校正 / 俯视展开

六、ROI（Region of Interest）
含义
cv::Mat roi = img(rect);


不拷贝数据

只是指针偏移 + step

为什么快

零拷贝

工业必用

副作用

isContinuous() == false

不能直接 memcpy

七、性能优化（你已经走到这里了）
1️⃣ 为什么不用 for？

因为 OpenCV 内部：

SIMD（AVX / NEON）

cache blocking

多线程（TBB / OpenMP）

手写汇编级优化

你手写 for：

没 SIMD

cache miss 多

分支多

2️⃣ 什么时候必须自己写 for？

✅ 必须：

自定义逻辑

条件判断复杂

非标准算子

❌ 不要：

blur / resize / threshold / morphology

convertTo

八、你现在所处的水平

说实话，你已经：

理解 Mat 底层

理解 连续 vs 非连续

能手写 通道分离 / 饱和度 / ptr

明白 性能瓶颈在哪里

👉 这是 工业视觉工程师 的基础段位，不是初学者。

下一步我建议你学（顺序）

形态学 + 阈值 + ROI 联合

轮廓检测 findContours

最小外接矩形 / 圆

模板匹配

工业缺陷检测完整 pipeline