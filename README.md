# 疲劳驾驶检测项目说明

本项目使用摄像头实时读取驾驶员画面，通过 dlib 人脸检测和 68 点人脸关键点模型，计算眼睛闭合、嘴巴张开和头部点头状态，最后按阈值判断是否出现疲劳驾驶风险。

## 运行环境

当前指定环境：

```powershell
F:\anaconda\envs\py37
```

建议从项目目录运行：

```powershell
Set-Location "G:\Users\weili\Desktop\fatigue steer_detect"
& "F:\anaconda\envs\py37\python.exe" main.py
```

运行后会打开一个 OpenCV 窗口，窗口名为 `Frame`。按 `q` 退出。

已检查的关键库版本：

| 库 | 版本 |
| --- | --- |
| Python | 3.7.4 |
| opencv-python / cv2 | 4.7.0 |
| dlib | 19.24.0 |
| imutils | 0.5.4 |
| scipy | 1.7.3 |
| numpy | 1.21.6 |

## 文件说明

| 文件 | 作用 |
| --- | --- |
| `main.py` | 主程序，同时检测人脸、眼睛闭合、嘴巴张开、点头姿态，并给出疲劳提示。 |
| `pilao_steer.py` | 较早的眼睛闭合/眨眼疲劳检测示例。 |
| `emotion_detected.py` | 表情识别示例，单独运行才会执行。 |
| `smile_detect.py` | 微笑/开心状态检测示例，单独运行才会执行。 |
| `shape_predictor_68_face_landmarks.dat` | dlib 官方 68 点人脸关键点预测模型。 |

## 为什么没有“弹出人脸、人眼、嘴巴模型”

这里容易误解：`main.py` 不会分别弹出“人脸模型窗口”“人眼模型窗口”“嘴巴模型窗口”。它只打开一个 OpenCV 视频窗口 `Frame`，然后在同一帧画面上叠加识别结果：

- 人脸：绿色矩形框。
- 眼睛：左右眼绿色凸包轮廓。
- 嘴巴：嘴部绿色凸包轮廓。
- 关键点：红色 68 点。
- 指标文字：`Faces`、`EAR`、`MAR`、`Blinks`、`Yawning`、`Nod`。

如果窗口打开了但没有看到人脸、眼睛、嘴巴框，通常不是模型没有加载，而是当前帧没有检测到人脸。原来的 `main.py` 只有在 `rects = detector(gray, 0)` 检测到人脸后才绘制眼睛、嘴巴和 68 个点；没有检测到人脸时，窗口里不会有这些标注。现在代码已经补充了 `Faces: 0` 和 `No face detected` 提示，方便判断。

常见原因如下：

1. 摄像头画面里没有正脸，或者脸太偏、太暗、太远。
2. dlib 的 HOG 正脸检测器对侧脸、低光照、遮挡、强反光比较敏感。
3. 摄像头被其他程序占用，或 Windows 隐私权限不允许 Python 使用摄像头。
4. 从其他工作目录运行脚本时，原代码可能找不到 `shape_predictor_68_face_landmarks.dat`。现在已改为按 `main.py` 所在目录查找模型文件。
5. `emotion_detected.py` 和 `smile_detect.py` 不会被 `main.py` 自动调用；它们是独立示例脚本，需要分别运行。

如果要提高小脸检测率，可以尝试把：

```python
rects = detector(gray, 0)
```

改为：

```python
rects = detector(gray, 1)
```

第二个参数表示上采样次数，`1` 会更容易检测到小脸，但速度会更慢。

## main.py 逻辑原理

主程序按以下流程循环处理摄像头画面：

1. 加载 dlib 人脸检测器：

   ```python
   detector = dlib.get_frontal_face_detector()
   ```

2. 加载 68 点人脸关键点模型：

   ```python
   predictor = dlib.shape_predictor(str(PREDICTOR_PATH))
   ```

3. 打开摄像头：

   ```python
   cap = cv2.VideoCapture(0)
   ```

4. 每一帧读取图像、缩放到宽度 720，并转成灰度图。
5. 用 dlib 检测人脸框 `rects`。
6. 对每个人脸框预测 68 个关键点。
7. 从关键点中取出左右眼和嘴巴区域。
8. 计算眼睛长宽比 `EAR`，判断闭眼和眨眼。
9. 计算嘴巴长宽比 `MAR`，判断张嘴/哈欠。
10. 使用 `solvePnP` 估计头部姿态，判断点头。
11. 在画面上绘制检测框、关键点和统计数值。
12. 当眨眼、哈欠或点头次数达到阈值时显示 `SLEEP!!!`。

## 眼睛 EAR 公式

每只眼睛取 6 个关键点，设为：

```text
p1, p2, p3, p4, p5, p6
```

眼睛长宽比为：

```text
EAR = (||p2 - p6|| + ||p3 - p5||) / (2 * ||p1 - p4||)
```

含义：

- `||p2 - p6||` 和 `||p3 - p5||` 是眼睛上下眼睑的垂直距离。
- `||p1 - p4||` 是眼睛左右眼角的水平距离。
- 眼睛闭合时，垂直距离变小，`EAR` 会下降。

代码中左右眼分别计算后取平均：

```python
ear = (leftEAR + rightEAR) / 2.0
```

当前阈值：

```python
EYE_AR_THRESH = 0.2
EYE_AR_CONSEC_FRAMES = 3
```

判断逻辑：

- 当 `EAR < 0.2` 时，认为当前帧可能闭眼，`COUNTER += 1`。
- 当连续闭眼帧数达到 3 帧后，再睁眼时累计一次眨眼，`TOTAL += 1`。

## 嘴巴 MAR 公式

嘴巴区域从 68 点中的嘴部关键点切片得到：

```python
mouth = shape[mStart:mEnd]
```

代码里使用 3 组距离计算嘴巴长宽比：

```text
MAR = (||m2 - m9|| + ||m4 - m7||) / (2 * ||m0 - m6||)
```

其中：

- 分子是嘴巴上下方向的开合距离。
- 分母是嘴巴左右方向的宽度。
- 张嘴或打哈欠时，垂直距离变大，`MAR` 会升高。

当前阈值：

```python
MAR_THRESH = 0.5
MOUTH_AR_CONSEC_FRAMES = 3
```

判断逻辑：

- 当 `MAR > 0.5` 时，认为当前帧可能张嘴。
- 连续超过 3 帧后，累计一次哈欠。

## 头部姿态估计公式

头部姿态使用 OpenCV 的 `solvePnP`。核心思想是：已知一组人脸 3D 参考点和图像中的 2D 关键点，求解头部相对摄像头的旋转和平移。

投影关系可以写成：

```text
s [u, v, 1]^T = K [R | t] [X, Y, Z, 1]^T
```

其中：

- `(X, Y, Z)` 是 3D 人脸参考点坐标。
- `(u, v)` 是图像中的 2D 像素坐标。
- `K` 是摄像头内参矩阵。
- `R` 是旋转矩阵。
- `t` 是平移向量。
- `s` 是尺度因子。

代码流程：

1. 从 68 点中取出眉毛、眼睛、鼻子、嘴巴、下巴等 14 个 2D 点。
2. 与预设的 14 个 3D 参考点 `object_pts` 对应。
3. 调用：

   ```python
   cv2.solvePnP(object_pts, image_pts, cam_matrix, dist_coeffs)
   ```

4. 用 `cv2.Rodrigues` 将旋转向量转成旋转矩阵。
5. 用 `cv2.decomposeProjectionMatrix` 分解出欧拉角。
6. 取 `pitch` 方向角度作为点头判断依据。

当前代码里：

```python
HAR_THRESH = 0.3
NOD_AR_CONSEC_FRAMES = 3
```

注意：`euler_angle` 是角度值，`0.3` 度非常小，点头判断可能偏敏感。实际使用时建议根据摄像头位置和坐姿重新标定，例如从 `10` 到 `20` 度区间试起。

## 疲劳判断规则

当前主程序的疲劳提示条件是：

```python
if TOTAL >= 50 or mTOTAL >= 15 or hTOTAL >= 15:
    cv2.putText(frame, "SLEEP!!!", ...)
```

也就是满足任一条件就提示疲劳：

- 眨眼次数 `TOTAL >= 50`
- 哈欠次数 `mTOTAL >= 15`
- 点头次数 `hTOTAL >= 15`

这些阈值和摄像头帧率、驾驶员个人习惯、光照环境都有关系，实际应用中建议采集一段自己的数据后再调整。

## 常见问题排查

### 1. 完全没有窗口

请从 PowerShell 直接运行：

```powershell
Set-Location "G:\Users\weili\Desktop\fatigue steer_detect"
& "F:\anaconda\envs\py37\python.exe" main.py
```

如果终端出现摄像头或模型文件错误，优先按错误信息处理。当前代码已经增加摄像头打开失败、摄像头读取失败、模型文件不存在的中文提示。

### 2. 有窗口，但没有人脸框和眼嘴轮廓

看窗口左上角：

- `Faces: 0`：说明当前帧没有检测到人脸。
- `Faces: 1` 或更大：说明已检测到人脸，应该能看到矩形框和关键点。

如果一直是 `Faces: 0`，请尝试：

- 正脸面对摄像头。
- 增强光照，避免背光。
- 靠近摄像头，让脸部更大。
- 去掉口罩、墨镜等遮挡。
- 把 `detector(gray, 0)` 临时改成 `detector(gray, 1)`。

### 3. 摄像头打不开

可以单独测试：

```powershell
& "F:\anaconda\envs\py37\python.exe" -c "import cv2; cap=cv2.VideoCapture(0); print(cap.isOpened()); ret, frame=cap.read(); print(ret, None if frame is None else frame.shape); cap.release()"
```

如果输出不是 `True`，请检查：

- Windows 摄像头隐私权限。
- 摄像头是否被微信、腾讯会议、浏览器等程序占用。
- 外接摄像头是否需要改成 `cv2.VideoCapture(1)` 或其他编号。

### 4. 想运行表情或微笑识别

`main.py` 不会自动调用 `emotion_detected.py` 或 `smile_detect.py`。如果要运行对应示例：

```powershell
& "F:\anaconda\envs\py37\python.exe" emotion_detected.py
& "F:\anaconda\envs\py37\python.exe" smile_detect.py
```

## 调参建议

| 参数 | 当前值 | 作用 | 建议 |
| --- | --- | --- | --- |
| `EYE_AR_THRESH` | `0.2` | 闭眼阈值 | 眼睛小或戴眼镜时可调到 `0.18` 到 `0.25`。 |
| `EYE_AR_CONSEC_FRAMES` | `3` | 连续闭眼帧数 | 帧率高时可适当增大。 |
| `MAR_THRESH` | `0.5` | 张嘴阈值 | 如果普通说话也误判哈欠，可调高。 |
| `MOUTH_AR_CONSEC_FRAMES` | `3` | 连续张嘴帧数 | 想减少误判可调到 `5` 以上。 |
| `HAR_THRESH` | `0.3` | 点头角度阈值 | 当前偏敏感，建议重新标定。 |
| `NOD_AR_CONSEC_FRAMES` | `3` | 连续点头帧数 | 结合帧率调整。 |

## 当前已做的稳定性改进

- `shape_predictor_68_face_landmarks.dat` 改为按 `main.py` 所在目录查找，不依赖运行时工作目录。
- 摄像头无法打开时，直接给出中文错误。
- 摄像头读取失败时，停止循环并提示。
- 创建可调整大小的 `Frame` 窗口。
- 没检测到人脸时，在画面显示 `Faces: 0` 和 `No face detected`。
