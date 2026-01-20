# 01 - 安装与环境（Python / C++）

## 方案选择（新手推荐）

- **只想快速做图像处理练习**：选 **Python + opencv-python**
- **要做高性能/工程项目**：选 **C++ + OpenCV + CMake**

## Python（推荐从这里开始）

### 安装

```bash
python -m pip install -U pip
python -m pip install opencv-python numpy
```

如果你需要一些扩展算法（如 SIFT 在某些发行方式里、或额外模块），可以尝试：

```bash
python -m pip install opencv-contrib-python
```

### 验证安装

```python
import cv2
print(cv2.__version__)
```

### 常见坑

- `import cv2` 报错：通常是环境不对（装在另一个 Python/venv 里）
- `imshow` 无窗口：WSL/无桌面环境需要额外配置显示；也可以先用 `imwrite` 输出结果图验证

## C++（需要一点工程配置）

### Ubuntu/WSL（系统包方式）

```bash
sudo apt update
sudo apt install -y libopencv-dev pkg-config
pkg-config --modversion opencv4
```

### CMake 最小示例（仅示意）

`CMakeLists.txt` 关键点通常是：

- `find_package(OpenCV REQUIRED)`
- `target_link_libraries(your_target ${OpenCV_LIBS})`

## 你下一步该做什么

去 `opencv/03-read-write-show.md` 跑通“读图→显示/保存”，这是所有练习的起点。

