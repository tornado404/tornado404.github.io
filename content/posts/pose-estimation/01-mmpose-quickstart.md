---
title: "MMPose 快速入门：人体姿态估计与动作捕捉实战"
date: 2026-03-02
draft: false
tags: ["MMPose", "姿态估计", "人体动作捕捉", "深度学习", "OpenMMLab"]
categories: ["pose-estimation"]
toc: true
---

## 什么是 MMPose

[MMPose](https://github.com/open-mmlab/mmpose) 是 OpenMMLab 推出的开源人体姿态估计工具箱，基于 PyTorch 实现。它提供了丰富的姿态估计算法，支持 2D/3D 姿态估计、手部/面部关键点检测等多种任务。

### 核心特性

- **算法丰富**：支持 80+ 种姿态估计算法
- **高精度**：多个 SOTA 模型
- **易用性**：简洁的 API 设计
- **生产就绪**：支持 ONNX/TensorRT 部署
- **社区活跃**：完善的文档和教程

---

## 快速开始

### 环境配置

```bash
# 创建 Conda 环境
conda create -n mmpose python=3.8 -y
conda activate mmpose

# 安装 PyTorch
pip install torch==2.0.0 torchvision==0.15.0 --index-url https://download.pytorch.org/whl/cu118

# 安装 MMCV
pip install -U openmim
mim install mmcv==2.0.0

# 安装 MMPose
git clone https://github.com/open-mmlab/mmpose.git
cd mmpose
pip install -r requirements.txt
pip install -e .

# 验证安装
python -c "import mmpose; print(mmpose.__version__)"
```

### 下载预训练模型

```bash
# 创建模型目录
mkdir checkpoints

# 下载 RTMPose 模型（推荐）
# RTMPose 是 MMPose 最新的高性能模型
wget https://download.openmmlab.com/mmpose/v1/projects/rtmpose/rtmpose-m_simcc-aic-coco_pt-aic-coco_420e-256x192.py
wget https://download.openmmlab.com/mmpose/v1/projects/rtmpose/rtmpose-m_simcc-aic-coco_pt-aic-coco_420e-256x192-63eb25f7.pth
```

---

## 基础使用

### 1. 使用推理器（推荐）

```python
from mmpose.apis import inference_topdown, init_model
from mmpose.structures import merge_data_samples
import cv2

# 初始化模型
config_file = 'rtmpose-m_simcc-aic-coco_pt-aic-coco_420e-256x192.py'
checkpoint_file = 'rtmpose-m_simcc-aic-coco_pt-aic-coco_420e-256x192-63eb25f7.pth'
model = init_model(config_file, checkpoint_file, device='cuda:0')

# 读取图像
image = cv2.imread('test_image.jpg')

# 推理（需要先检测人体）
from mmdet.apis import init_model as init_det_model
from mmdet.apis import inference_detector

# 初始化检测器
det_config = 'mmdet_configs/rtmdet_tiny_8xb32-300e_coco.py'
det_checkpoint = 'https://download.openmmlab.com/mmdetection/v2.0/rtmdet/rtmdet_tiny_8xb32-300e_coco/rtmdet_tiny_8xb32-300e_coco_20220902_112414-78e30dcc.pth'
detector = init_det_model(det_config, det_checkpoint, device='cuda:0')

# 检测人体
det_result = inference_detector(detector, image)
bbox = det_result.pred_instances.bboxes  # 获取检测框

# 姿态估计
pose_result = inference_topdown(model, image, bbox)
pose_result = merge_data_samples(pose_result)

# 可视化结果
from mmpose.visualization import PoseLocalVisualizer
visualizer = PoseLocalVisualizer()
visualizer.set_dataset_meta(model.dataset_meta)

img_vis = visualizer.add_datasample(
    'result',
    image,
    pose_result,
    draw_gt=False,
    show=False
)

cv2.imwrite('output.jpg', img_vis)
```

### 2. 简化版本（使用 MMPose 3.x）

```python
from mmpose.apis import MMPoseInferencer

# 初始化推理器（自动处理检测）
inferencer = MMPoseInferencer(
    pose2d='rtmpose-m',
    det_model='rtmdet_tiny',
    device='cuda:0'
)

# 推理
result = inferencer('test_image.jpg')

# 获取关键点
keypoints = result['predictions'][0]['keypoints']
print(f"检测到 {len(keypoints)} 个关键点")

# 保存可视化结果
inferencer('test_image.jpg', out_dir='output/', show=True)
```

---

## 关键点说明

### COCO 数据集 17 个关键点

```python
COCO_KEYPOINTS = [
    'nose',           # 0
    'left_eye',       # 1
    'right_eye',      # 2
    'left_ear',       # 3
    'right_ear',      # 4
    'left_shoulder',  # 5
    'right_shoulder', # 6
    'left_elbow',     # 7
    'right_elbow',    # 8
    'left_wrist',     # 9
    'right_wrist',    # 10
    'left_hip',       # 11
    'right_hip',      # 12
    'left_knee',      # 13
    'right_knee',     # 14
    'left_ankle',     # 15
    'right_ankle'     # 16
]
```

### 关键点置信度

```python
# 每个关键点包含 (x, y, confidence)
keypoint = keypoints[0]  # 例如鼻子
x, y, conf = keypoint

# 过滤低置信度关键点
valid_keypoints = [kp for kp in keypoints if kp[2] > 0.5]
```

---

## 实战：人体动作捕捉系统

### 完整代码示例

```python
"""
人体姿态估计与动作捕捉系统
支持实时视频处理和骨骼绘制
"""

import cv2
import numpy as np
from mmpose.apis import MMPoseInferencer
from typing import List, Tuple

class PoseCapture:
    def __init__(self, 
                 pose_model: str = 'rtmpose-m',
                 det_model: str = 'rtmdet_tiny',
                 device: str = 'cuda:0',
                 conf_threshold: float = 0.5):
        """
        初始化姿态捕捉器
        
        Args:
            pose_model: 姿态估计模型
            det_model: 人体检测模型
            device: 计算设备
            conf_threshold: 置信度阈值
        """
        self.inferencer = MMPoseInferencer(
            pose2d=pose_model,
            det_model=det_model,
            device=device
        )
        self.conf_threshold = conf_threshold
        
        # 骨骼连接关系（COCO 格式）
        self.skeleton = [
            (0, 1), (0, 2), (1, 3), (2, 4),    # 头部
            (5, 6),                             # 肩膀
            (5, 7), (7, 9),                    # 左臂
            (6, 8), (8, 10),                   # 右臂
            (5, 11), (6, 12),                  # 躯干
            (11, 13), (13, 15),                # 左腿
            (12, 14), (14, 16)                 # 右腿
        ]
        
        # 颜色映射
        self.colors = [
            (255, 0, 0), (255, 85, 0), (255, 170, 0),
            (255, 255, 0), (170, 255, 0), (85, 255, 0),
            (0, 255, 0), (0, 255, 85), (0, 255, 170),
            (0, 255, 255), (0, 170, 255), (0, 85, 255),
            (0, 0, 255), (85, 0, 255), (170, 0, 255),
            (255, 0, 255), (255, 0, 170)
        ]
    
    def process_frame(self, frame: np.ndarray) -> Tuple[np.ndarray, dict]:
        """
        处理单帧图像
        
        Args:
            frame: 输入图像
            
        Returns:
            绘制骨骼的图像，姿态数据
        """
        # 推理
        result = self.inferencer(frame)
        
        # 提取关键点
        if 'predictions' in result and len(result['predictions']) > 0:
            keypoints = result['predictions'][0]['keypoints']
            scores = result['predictions'][0]['keypoint_scores']
        else:
            keypoints = []
            scores = []
        
        # 绘制骨骼
        output_frame = self.draw_skeleton(frame.copy(), keypoints, scores)
        
        return output_frame, {
            'keypoints': keypoints,
            'scores': scores
        }
    
    def draw_skeleton(self, 
                      frame: np.ndarray, 
                      keypoints: List, 
                      scores: List) -> np.ndarray:
        """
        绘制人体骨骼
        
        Args:
            frame: 输入图像
            keypoints: 关键点列表
            scores: 置信度列表
            
        Returns:
            绘制骨骼后的图像
        """
        for person_idx, (kp, score) in enumerate(zip(keypoints, scores)):
            # 过滤低置信度关键点
            valid_mask = score > self.conf_threshold
            valid_kp = kp[valid_mask]
            
            # 绘制关键点
            for idx, point in enumerate(kp):
                if score[idx] > self.conf_threshold:
                    x, y = int(point[0]), int(point[1])
                    color = self.colors[idx % len(self.colors)]
                    cv2.circle(frame, (x, y), 5, color, -1)
            
            # 绘制骨骼连接
            for bone in self.skeleton:
                idx1, idx2 = bone
                if (idx1 < len(kp) and idx2 < len(kp) and
                    score[idx1] > self.conf_threshold and
                    score[idx2] > self.conf_threshold):
                    
                    pt1 = (int(kp[idx1][0]), int(kp[idx1][1]))
                    pt2 = (int(kp[idx2][0]), int(kp[idx2][1]))
                    color = self.colors[idx1 % len(self.colors)]
                    cv2.line(frame, pt1, pt2, color, 2)
        
        return frame
    
    def process_video(self, 
                      input_path: str, 
                      output_path: str,
                      show: bool = False):
        """
        处理视频文件
        
        Args:
            input_path: 输入视频路径
            output_path: 输出视频路径
            show: 是否实时显示
        """
        cap = cv2.VideoCapture(input_path)
        
        # 获取视频信息
        fps = cap.get(cv2.CAP_PROP_FPS)
        width = int(cap.get(cv2.CAP_PROP_FRAME_WIDTH))
        height = int(cap.get(cv2.CAP_PROP_FRAME_HEIGHT))
        
        # 创建写入器
        fourcc = cv2.VideoWriter_fourcc(*'mp4v')
        out = cv2.VideoWriter(output_path, fourcc, fps, (width, height))
        
        frame_count = 0
        while True:
            ret, frame = cap.read()
            if not ret:
                break
            
            # 处理帧
            output_frame, pose_data = self.process_frame(frame)
            
            # 显示 FPS
            cv2.putText(output_frame, f'Frame: {frame_count}', 
                       (10, 30), cv2.FONT_HERSHEY_SIMPLEX, 1, (0, 255, 0), 2)
            
            # 写入输出
            out.write(output_frame)
            
            # 显示
            if show:
                cv2.imshow('Pose Capture', output_frame)
                if cv2.waitKey(1) & 0xFF == ord('q'):
                    break
            
            frame_count += 1
            if frame_count % 30 == 0:
                print(f"Processed {frame_count} frames")
        
        cap.release()
        out.release()
        cv2.destroyAllWindows()
        print(f"Video processed: {output_path}")


# 使用示例
if __name__ == '__main__':
    # 初始化
    capture = PoseCapture()
    
    # 处理单张图片
    import cv2
    image = cv2.imread('input.jpg')
    output, data = capture.process_frame(image)
    cv2.imwrite('output.jpg', output)
    
    # 处理视频
    # capture.process_video('input.mp4', 'output.mp4', show=True)
    
    # 处理摄像头
    # capture.process_video(0, 'webcam_output.mp4', show=True)
```

---

## 高级应用

### 1. 多人姿态估计

```python
def process_multi_person(frame):
    result = inferencer(frame)
    
    for person_idx, prediction in enumerate(result['predictions'][0]['keypoints']):
        print(f"Person {person_idx}: {len(prediction)} keypoints")
        
        # 计算每个人的人体中心
        center = np.mean(prediction[:, :2], axis=0)
        print(f"Center: {center}")
```

### 2. 动作识别

```python
def recognize_action(keypoints):
    """
    基于关键点识别简单动作
    """
    # 获取关键点
    nose = keypoints[0]
    left_shoulder = keypoints[5]
    right_shoulder = keypoints[6]
    left_hip = keypoints[11]
    right_hip = keypoints[12]
    
    # 计算身体角度
    shoulder_vec = right_shoulder[:2] - left_shoulder[:2]
    hip_vec = right_hip[:2] - left_hip[:2]
    
    # 判断站立/坐下
    body_height = abs(nose[1] - (left_hip[1] + right_hip[1]) / 2)
    
    if body_height < 50:  # 阈值需要根据实际情况调整
        return "sitting"
    else:
        return "standing"
```

### 3. 姿态数据导出

```python
import json

def export_pose_data(results, output_file):
    """导出姿态数据为 JSON"""
    data = {
        'frames': []
    }
    
    for frame_result in results:
        frame_data = {
            'keypoints': frame_result['keypoints'].tolist(),
            'scores': frame_result['keypoint_scores'].tolist()
        }
        data['frames'].append(frame_data)
    
    with open(output_file, 'w') as f:
        json.dump(data, f, indent=2)
    
    print(f"Exported to {output_file}")
```

---

## 模型选择指南

| 模型 | 速度 | 精度 | 适用场景 |
|------|------|------|----------|
| RTMPose-m | 快 | 高 | 实时应用 |
| RTMPose-l | 中 | 很高 | 离线处理 |
| ViTPose-B | 慢 | 最高 | 高精度需求 |
| HRNet-W32 | 中 | 高 | 通用场景 |

---

## 部署优化

### ONNX 导出

```bash
# 导出为 ONNX
python tools/deployment/pytorch2onnx.py \
    configs/body_2d_keypoint/rtmpose/rtmpose-m_simcc-aic-coco.py \
    checkpoints/rtmpose-m.pth \
    --output-file rtmpose-m.onnx \
    --input-shape 1 3 256 192
```

### TensorRT 加速

```python
import tensorrt as trt

# 加载 ONNX 模型
with open('rtmpose-m.onnx', 'rb') as f:
    network = trt.load_network(f)

# 构建 TensorRT 引擎
# ... (详细配置参考 TensorRT 文档)
```

---

## 常见问题

### Q1: 检测不到人体？
- 检查检测模型是否正确加载
- 调整检测置信度阈值
- 确保图像中有人体且清晰

### Q2: 关键点抖动？
- 使用 temporal smoothing
- 增加置信度阈值
- 使用更高质量的模型

### Q3: 实时性不够？
- 使用 RTMPose-tiny
- 降低输入分辨率
- 使用 TensorRT 加速

---

## 总结

MMPose 是功能强大的人体姿态估计工具：

**优势：**
- ✅ 算法丰富
- ✅ 高精度
- ✅ 易于使用
- ✅ 支持部署

**适用场景：**
- 健身动作分析
- 舞蹈教学
- 康复训练
- 体感游戏
- 视频监控

在下一篇文章中，我们将探讨 3D 姿态估计和多视角融合技术。

---

## 参考资源

- [MMPose GitHub](https://github.com/open-mmlab/mmpose)
- [官方文档](https://mmpose.readthedocs.io/)
- [OpenMMLab](https://openmmlab.com/)
- [COCO 数据集](https://cocodataset.org/)
