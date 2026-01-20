# 00 - OpenCV 新手怎么学（最短路径）

## 你最终要达到什么能力

- **能读懂**：图像/矩阵、像素、通道、类型（`uint8`/`float`）、BGR/RGB/Gray
- **能搭 pipeline**：读图 → 预处理 → 分割/检测 → 几何测量 → 可视化/导出
- **能排坑**：颜色顺序、坐标系、数据类型溢出、ROI 是否拷贝、性能瓶颈

## 学习方式（强烈建议）

- **每章只做 2 件事**：
  - 先跑通“最小可运行代码”
  - 再自己改 3 个参数/做 3 个变体（写到笔记里）
- **每次练习都存结果图**：输入、输出、关键中间结果（比如二值图、边缘图）
- **不要一上来学一堆 API**：先把 10 个最常用函数学到“闭眼能用”

## 必练清单（10 个最常用）

1. 读写：`imread` / `imwrite`
2. 尺寸：`resize`
3. 颜色：`cvtColor`
4. 模糊：`GaussianBlur`
5. 边缘：`Canny`
6. 二值：`threshold` / `adaptiveThreshold`
7. 形态学：`erode` / `dilate` / `morphologyEx`
8. 轮廓：`findContours`
9. 几何：`boundingRect` / `minAreaRect` / `minEnclosingCircle`
10. 可视化：`imshow` / `waitKey` / `drawContours` / `rectangle` / `circle`

## 你会反复遇到的 4 个坑

- **颜色顺序**：OpenCV 默认 BGR，不是 RGB
- **坐标系**：`(x, y)` 中，`x` 是列（横向），`y` 是行（纵向）
- **类型溢出**：`uint8` 做加减会溢出/截断，算之前 `convertTo` 或转 `float`
- **ROI 语义**：很多 ROI 是“视图”（不拷贝），修改会影响原图

下一章：`opencv/01-install.md`

