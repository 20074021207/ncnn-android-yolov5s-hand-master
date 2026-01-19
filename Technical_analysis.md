# ncnn-android-yolov5s-hand-master 技术解析

本文档基于开源项目 `ncnn-android-yolov5s-hand-master` 源码编写。

**重要提示**：本项目与常见的 Android AI Demo 不同，它采用了 **全 Native 链路** 设计。相机采集、预处理、推理、结果绘制均在 C++ 层完成，Java 层仅充当“外壳”。这种架构避免了 Java 与 C++ 之间昂贵的图像数据拷贝（JNI 开销），能显著提升推理帧率。

---

## 一、总体架构与 API 调用流程

### 1.1 总体架构

*   **Java 层 (`MainActivity.java`)**：
    *   **UI 容器**：提供 `SurfaceView` 用于显示。
    *   **生命周期管理**：处理 Activity 的 `onResume`/`onPause`，申请相机权限。
    *   **资源传递**：将 `AssetManager`（用于读取模型）和 `Surface`（用于渲染）传递给 Native 层。
*   **JNI 层 (`yoloncnn.cpp`)**：
    *   **接口桥接**：导出 `loadModel`, `openCamera` 等接口。
    *   **对象管理**：持有 `Yolo` 推理实例和 `MyNdkCamera` 相机实例。
*   **C++ 核心层 (`ndkcamera.cpp`, `yolo.cpp`)**：
    *   **NDK Camera**：直接在 Native 层调用 Android Camera2 NDK API 获取图像流。
    *   **推理引擎**：集成 ncnn，运行 YOLOv5s 模型。
    *   **直接渲染**：使用 OpenCV 直接修改相机帧数据（画框），并通过 `ANativeWindow` 将处理后的帧直接渲染到屏幕。

### 1.2 关键 API 调用流程

#### (1) 初始化与模型加载
Java 层调用 `ncnnyolov7.loadModel(...)` -> JNI `Java_..._loadModel`：
1.  **AAssetManager**：将 Java 的 AssetManager 对象转换为 C++ 指针。
2.  **Yolo::load**：直接从 Assets 资源中读取 `.param` 和 `.bin` 文件（无需解压到本地存储）。
3.  **配置 ncnn**：设置线程数、开启 Vulkan (GPU) 加速。

#### (2) 启动相机
Java 层调用 `ncnnyolov7.openCamera(facing)` -> JNI `Java_..._openCamera`：
1.  调用 `g_camera->open(facing)`。
2.  初始化 NDK Camera，设置回调函数。

#### (3) 设置显示窗口
Java 层 `surfaceChanged` -> `ncnnyolov7.setOutputWindow(surface)`：
1.  JNI 使用 `ANativeWindow_fromSurface` 获取原生窗口句柄。
2.  C++ 层绑定该窗口，用于后续的画面渲染。

#### (4) 核心循环（帧回调）
**这是本项目的核心路径**，完全在 C++ 层闭环：
1.  **Frame Capture**：NDK Camera 捕获一帧图像。
2.  **Callback**：触发 `MyNdkCamera::on_image_render(cv::Mat& rgb)` 回调。
3.  **Inference**：
    *   调用 `g_yolo->detect(rgb, objects)`。
    *   图像预处理（Letterbox Resize + Normalize）。
    *   ncnn 前向计算。
    *   后处理（Decode + NMS）。
4.  **Render**：
    *   调用 `g_yolo->draw(rgb, objects)` 直接在 `rgb` 图像上绘制检测框和类别文字。
    *   NDK Camera 模块将绘制好的 `rgb` 数据提交给 `ANativeWindow` 显示。

---

## 二、JNI 层与 C++ 关键代码讲解

### 2.1 JNI 接口 (`yoloncnn.cpp`)
JNI 层非常薄，主要用于转发 Java 命令：
```cpp
// 示例：打开相机
JNIEXPORT jboolean JNICALL Java_com_tencent_ncnnyolov7_NcnnYolov7_openCamera(JNIEnv* env, jobject thiz, jint facing)
{
    g_camera->open((int)facing); // 调用 C++ 对象方法
    return JNI_TRUE;
}
```

### 2.2 推理核心类 (`Yolo` in `yolo.cpp`)

#### (1) 模型加载
利用 ncnn 对 Android Assets 的原生支持：
```cpp
yolo.load_param(mgr, parampath);
yolo.load_model(mgr, modelpath);
```

#### (2) 输入预处理 (Letterbox)
YOLOv5 需要保持长宽比的输入，代码实现了标准的 Letterbox：
*   **Resize**：按比例缩放长边到 `target_size` (如 320)。
*   **Padding**：计算剩余空间，填充灰色像素 (114)，且填充到 `64` 的倍数（适应硬件对齐）。
*   **Normalize**：归一化到 `[0, 1]`。
```cpp
ncnn::Mat in_pad;
ncnn::copy_make_border(in, in_pad, ...); // Padding
in_pad.substract_mean_normalize(0, norm_vals); // Normalize
```

#### (3) 后处理 (Decode & NMS)
*   **Anchors**：硬编码了 YOLOv5s 的 Anchor 尺寸（针对 Stride 8, 16, 32）。
*   **解码**：
    *   `sigmoid` 激活：`x, y, w, h, score`。
    *   坐标还原：应用 YOLOv5 的坐标公式，并反算回原图尺寸（减去 Padding，除以 Scale）。
*   **NMS (Non-Maximum Suppression, 非极大值抑制)**：在目标检测中，模型常会对同一个物体预测出多个略微偏移的框。NMS 的作用是计算框之间的 IoU (交并比)，去除重叠度高且置信度较低的冗余框，只保留最佳的一个。代码中使用 `nms_sorted_bboxes` 实现。

### 2.3 相机类 (`MyNdkCamera` in `yoloncnn.cpp`)
继承自 `NdkCameraWindow`，重写 `on_image_render`：
```cpp
void MyNdkCamera::on_image_render(cv::Mat& rgb) const
{
    // 1. 加锁，防止模型正在重载时推理
    ncnn::MutexLockGuard g(lock);
    
    // 2. 检测
    if (g_yolo) g_yolo->detect(rgb, objects);
    
    // 3. 绘制 (直接修改 rgb 数据)
    g_yolo->draw(rgb, objects);
    
    // 4. 绘制 FPS
    draw_fps(rgb);
}
```

---

## 三、模型运行与转换（PyTorch → ncnn）

### 3.1 数据集准备（HaGRID）
本项目主要针对手势识别，使用 **HaGRID** 数据集，包含 18 种手势（call, like, fist, palm, peace 等）+ no_gesture，共 19 类。

### 3.2 YOLOv5 训练
1.  **环境准备**：Python 3.8+, PyTorch 1.10+, YOLOv5 v6.0/v7.0 源码。
2.  **配置文件**：修改 `models/yolov5s.yaml`，将 `nc` (number of classes) 改为 19。
3.  **训练**：
    ```bash
    python train.py --img 640 --batch 64 --epochs 100 --data hagrid.yaml --weights yolov5s.pt
    ```

### 3.3 导出 ONNX
使用 `export.py` 导出模型。对于 ncnn，通常建议导出纯净的结构或包含简单后处理的结构。
```bash
python export.py --weights best.pt --include onnx --simplify
```
*注意：导出的 ONNX 输出节点名称可能会变化，这会影响 `yolo.cpp` 中 `ex.extract(...)` 的参数，需要同步修改。*

### 3.4 ONNX → ncnn
1.  **转换**：
    ```bash
    ./onnx2ncnn best.onnx best.param best.bin
    ```
2.  **优化 (FP16)**：
    使用 `ncnnoptimize` 转换模型为 FP16 格式，这对移动端性能至关重要。
    ```bash
    ./ncnnoptimize best.param best.bin best-opt.param best-opt.bin 65536
    ```

### 3.5 Android 部署
1.  将生成的 `best-opt.param` 和 `best-opt.bin` 重命名并放入 `app/src/main/assets/`。
2.  如果文件名变更，需修改 `yoloncnn.cpp` 中的 `modeltypes` 数组。
3.  如果模型输出层名称变更，需修改 `yolo.cpp` 中的 `ex.extract(...)` 节点名。

---

## 四、总结
本项目展示了 Android 平台高性能 AI 应用的最佳实践：
1.  **ncnn**：极致优化的移动端推理框架。
2.  **NDK Camera**：绕过 Java 层，实现零拷贝的图像采集与渲染。
3.  **YOLOv5**：成熟且高效的目标检测算法。
