---
title: "3D Gaussian Splatting 入门：新一代实时辐射场渲染技术"
date: 2026-03-02
draft: false
tags: ["3D Gaussian Splatting", "NeRF", "计算机视觉", "3D 重建", "实时渲染"]
categories: ["3d-gaussian"]
toc: true
---

## 什么是 3D Gaussian Splatting

**3D Gaussian Splatting (3DGS)** 是 2023 年由 Inria（法国国家数字科学与技术研究所）提出的一种革命性的 3D 场景表示和渲染技术。它能够在保持高质量渲染效果的同时，实现**实时渲染**（100+ FPS），远超传统的 NeRF 方法。

### 核心论文

> **3D Gaussian Splatting for Real-Time Radiance Field Rendering**
> - Bernhard Kerbl, Georgios Kopanas, et al.
> - SIGGRAPH 2023
> - [论文链接](https://repo-sam.inria.fr/fungraph/3d-gaussian-splatting/)

---

## 技术背景：从 NeRF 到 3DGS

### NeRF 的局限性

NeRF (Neural Radiance Fields) 虽然能生成高质量的 3D 场景，但存在以下问题：

| 问题 | NeRF | 3DGS |
|------|------|------|
| 渲染速度 | 慢 (0.1-1 FPS) | 快 (100+ FPS) |
| 训练时间 | 长 (数小时) | 短 (数分钟) |
| 存储需求 | 大 (数百 MB) | 小 (数十 MB) |
| 编辑能力 | 困难 | 相对容易 |

### 3DGS 的核心创新

3DGS 使用**显式的 3D 高斯分布**来表示场景，而非 NeRF 的隐式神经网络：

```
NeRF:  MLP 网络 → 隐式表示 → 光线追踪渲染
3DGS:  3D 高斯点云 → 显式表示 → 光栅化渲染
```

---

## 3D Gaussian Splatting 原理

### 1. 高斯分布表示

每个 3D 高斯由以下参数定义：

```python
# 每个高斯点的参数
position: Vector3      # 3D 位置 (x, y, z)
covariance: Matrix3x3  # 协方差矩阵（形状和方向）
alpha: float           # 不透明度
sh_coefficients: []    # 球谐系数（颜色）
```

### 2. 从 SfM 点云初始化

```
输入图像 → COLMAP SfM → 稀疏点云 → 高斯初始化 → 优化
```

```python
# 初始化流程
1. 使用 COLMAP 进行运动恢复结构 (SfM)
2. 从 SfM 点云创建初始高斯分布
3. 每个点赋予：
   - 位置：来自 SfM
   - 协方差：基于邻域点
   - 颜色：来自观测
   - 不透明度：初始值
```

### 3. 优化过程

```python
# 优化参数
- 位置 (position)
- 旋转 (rotation, 用四元数表示)
- 缩放 (scale)
- 不透明度 (opacity)
- 球谐系数 (spherical harmonics)

# 优化策略
1. 自适应密度控制
   - 分裂过大高斯
   - 删除过小/透明高斯
   - 克隆欠重建区域

2. 损失函数
   - L1 损失
   - D-SSIM (结构相似性)
```

### 4. 光栅化渲染

```python
# 渲染流程
for each pixel:
    # 1. 将 3D 高斯投影到 2D
    2D_gaussian = project_3D_to_2D(gaussian, camera)
    
    # 2. 按深度排序
    sorted_gaussians = sort_by_depth(gaussians)
    
    # 3. Alpha 混合
    color = 0
    alpha_accum = 0
    for g in sorted_gaussians:
        weight = g.alpha * (1 - alpha_accum)
        color += weight * g.color
        alpha_accum += weight
        if alpha_accum > 0.999:
            break
    
    return color
```

---

## 快速开始

### 环境配置

```bash
# 克隆官方仓库
git clone https://github.com/graphdeco-inria/gaussian-splatting.git
cd gaussian-splatting

# 创建 Conda 环境
conda create -n gaussian_splatting python=3.8
conda activate gaussian_splatting

# 安装依赖
pip install torch torchvision --index-url https://download.pytorch.org/whl/cu118
pip install -r requirements.txt

# 编译 CUDA 扩展
pip install submodules/diff-gaussian-rasterization
pip install submodules/simple-knn
```

### 训练自己的场景

```bash
# 准备数据（COLMAP 格式）
python convert.py --source_path <path> --camera <COLMAP/Blender>

# 训练
python train.py -s <path>

# 评估
python train.py -s <path> --eval
```

### 实时查看器

```bash
# 启动实时查看器
python -m http.server 8080

# 浏览器访问
http://localhost:8080
```

---

## 代码解析

### 高斯分布类

```python
class GaussianModel:
    def __init__(self, sh_degree: int = 3):
        self.active_sh_degree = 0
        self.max_sh_degree = sh_degree
        
        # 高斯参数
        self._xyz = torch.empty(0)           # 位置
        self._features_dc = torch.empty(0)   # 基础颜色
        self._features_rest = torch.empty(0) # 高阶球谐
        self._scaling = torch.empty(0)       # 缩放
        self._rotation = torch.empty(0)      # 旋转
        self._opacity = torch.empty(0)       # 不透明度
        
    def get_covariance(self, scaling_modifier=1):
        """计算协方差矩阵"""
        L = build_scaling_rotation(self._scaling * scaling_modifier, self._rotation)
        actual_covariance = L @ L.transpose(1, 2)
        return actual_covariance
    
    def render(self, viewpoint_camera, pipe, bg_color):
        """渲染高斯场景"""
        return gaussian_rasterization(
            means3D=self._xyz,
            means2D=torch.zeros_like(self._xyz),
            shs=self.get_features,
            colors_precomp=None,
            opacities=self._opacity,
            scales=self._scaling,
            rotations=self._rotation,
            cov3D_precomp=None,
            camera=viewpoint_camera,
            pipe=pipe,
            bg_color=bg_color
        )
```

### 自适应密度控制

```python
def densify_and_prune(self, max_grad, min_opacity, extent, max_screen_size):
    """自适应密度控制"""
    
    # 1. 基于梯度分裂
    grads = self.xyz_gradient_accum / self.denom
    grads[grads.isnan()] = 0.0
    
    # 克隆：小高斯且梯度大
    if len(self.get_xyz) > 0:
        clone_mask = (
            (grads < max_grad).all(dim=1) &
            (self.get_scaling.max(dim=1).values < self.percent_dense * extent)
        )
        self.densify(clone_mask, "clone")
        
        # 分裂：大高斯且梯度大
        split_mask = (
            (grads >= max_grad).all(dim=1) &
            (self.get_scaling.max(dim=1).values >= self.percent_dense * extent)
        )
        self.densify(split_mask, "split")
    
    # 2. 剪枝：透明或过大的高斯
    self.prune(min_opacity)
    
    # 3. 重置统计
    self.reset_densification_stats()
```

---

## 应用场景

### 1. 3D 场景重建

```bash
# 输入：多视角照片
# 输出：可实时浏览的 3D 场景

# 适用场景
- 室内场景扫描
- 文物数字化
- 房地产 VR 看房
- 旅游景点重建
```

### 2. 自动驾驶仿真

```python
# 使用 3DGS 重建真实场景
# 在重建场景中生成合成数据

优势：
- 高保真度
- 实时渲染
- 可编辑光照/天气
```

### 3. AR/VR 应用

```python
# 在 VR 中浏览重建场景
# 支持 6DoF 交互

技术栈：
- 3DGS + WebXR
- 3DGS + Unity/Unreal
```

### 4. 影视特效

```python
# 实拍场景快速重建
# 与 CG 元素合成

优势：
- 快速场景获取
- 真实光照信息
- 无缝合成
```

---

## 开源项目推荐

| 项目 | 链接 | 特点 |
|------|------|------|
| 官方实现 | [graphdeco-inria/gaussian-splatting](https://github.com/graphdeco-inria/gaussian-splatting) | 原始论文代码 |
| Web 查看器 | [antimatter15/splat](https://github.com/antimatter15/splat) | 纯 Web 实现 |
| Unity 集成 | [aras-p/UnityGaussianSplatting](https://github.com/aras-p/UnityGaussianSplatting) | Unity 引擎支持 |
| Unreal 集成 | [danielepanato/UE5-GaussianSplatting](https://github.com/danielepanato/UE5-GaussianSplatting) | UE5 插件 |
| COLMAP GUI | [colmap/colmap](https://github.com/colmap/colmap) | SfM 工具 |

---

## 性能优化技巧

### 1. 减少高斯数量

```python
# 剪枝策略
- 删除不透明度 < 0.005 的高斯
- 限制最大高斯数量
- 定期合并相邻高斯
```

### 2. LOD (Level of Detail)

```python
# 根据距离使用不同精度
if distance < 5:
    use_full_sh_degree = 3
elif distance < 15:
    use_full_sh_degree = 1
else:
    use_full_sh_degree = 0  # 只使用基础颜色
```

### 3. 批量渲染

```python
# 使用实例化渲染
for batch in gaussian_batches:
    render_batch(batch)
```

---

## 与其他技术对比

| 技术 | 训练速度 | 渲染速度 | 质量 | 存储 |
|------|----------|----------|------|------|
| NeRF | 慢 | 很慢 | 高 | 中 |
| Instant NGP | 快 | 中 | 高 | 中 |
| **3DGS** | **中** | **很快** | **高** | **小** |
| Photogrammetry | 很慢 | 快 | 中 | 大 |

---

## 总结

3D Gaussian Splatting 是 3D 重建领域的突破性技术：

**优势：**
- ✅ 实时渲染 (100+ FPS)
- ✅ 高质量输出
- ✅ 较小存储需求
- ✅ 易于部署

**局限：**
- ❌ 需要多视角图像
- ❌ 动态场景支持有限
- ❌ 对弱纹理区域敏感

**适用场景：**
- 静态场景重建
- 需要实时交互的应用
- Web/移动端部署

在下一篇文章中，我们将深入探讨 3DGS 的高级应用，包括动态场景处理和与神经网络的结合。

---

## 参考资源

- [官方论文](https://repo-sam.inria.fr/fungraph/3d-gaussian-splatting/)
- [官方代码](https://github.com/graphdeco-inria/gaussian-splatting)
- [技术解读](https://www.bilibili.com/video/BV1Ru411c7mG)
- [COLMAP 文档](https://colmap.github.io/)
