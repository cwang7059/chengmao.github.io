# 06 - 轮廓与几何测量（从“看见”到“量化”）

## 轮廓是什么

轮廓（contour）通常来自二值图：前景/背景分开后，提取前景的边界点集合。

## findContours 你要知道的

- 输入一般是 **二值图**（或边缘图）
- 会返回：
  - `contours`：点集列表
  - `hierarchy`：层级关系（外轮廓/内孔洞）

## 常用几何量

- **面积**：`contourArea`
- **周长**：`arcLength`
- **外接矩形**：`boundingRect`
- **最小外接旋转矩形**：`minAreaRect`
- **最小外接圆**：`minEnclosingCircle`
- **多边形逼近**：`approxPolyDP`

## 推荐练习（1~2 小时）

1. 选一个“背景干净”的物体图，做阈值 → 形态学 → 轮廓
2. 过滤掉小轮廓（按面积阈值）
3. 给每个轮廓画：外接矩形、最小外接圆，并把面积/周长写到图上

下一章：`opencv/07-features-and-matching.md`

