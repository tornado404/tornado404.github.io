---
title: "Babylon.js 入门教程：Web3D 开发的强大工具"
date: 2026-03-02
draft: false
tags: ["Babylon.js", "Web3D", "WebGL", "Three.js", "3D 渲染"]
categories: ["web3d"]
toc: true
---

## 什么是 Babylon.js

[Babylon.js](https://babylonjs.com/) 是一个功能强大的开源 Web3D 引擎，由微软开发并维护。它使用 WebGL 和 WebGPU 技术在浏览器中渲染 3D 图形，为开发者提供了完整的 3D 开发工具链。

### 为什么选择 Babylon.js

与其他 Web3D 库（如 Three.js）相比，Babylon.js 有以下优势：

| 特性 | Babylon.js | Three.js |
|------|------------|----------|
| 类型支持 | 原生 TypeScript | 社区维护类型定义 |
| 物理引擎 | 内置 Cannon.js 集成 | 需要额外配置 |
| 碰撞检测 | 内置支持 | 需要额外库 |
| 动画系统 | 时间线动画器 | 基础动画支持 |
| 节点材质 | 可视化材质编辑器 | 需要手动编写 |
| 官方文档 | 完整详细 | 相对简单 |

### 核心特性

- **WebGL/WebGPU 渲染**：支持最新的图形 API
- **物理引擎集成**：内置 Cannon.js、Oimo.js 支持
- **碰撞检测系统**：完整的碰撞体和射线检测
- **动画系统**：关键帧动画、骨骼动画、 morph 目标
- **粒子系统**：强大的粒子效果
- **后期处理**：Bloom、景深、色调映射等
- **VR/AR 支持**：WebXR 原生支持
- **加载器**：支持 glTF、OBJ、STL 等格式

---

## 快速开始

### 安装方式

#### 方式一：CDN 引入（推荐初学者）

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Babylon.js 入门</title>
    <style>
        html, body {
            margin: 0;
            padding: 0;
            width: 100%;
            height: 100%;
            overflow: hidden;
        }
        #renderCanvas {
            width: 100%;
            height: 100%;
            touch-action: none;
        }
    </style>
</head>
<body>
    <canvas id="renderCanvas"></canvas>
    
    <!-- 引入 Babylon.js -->
    <script src="https://cdn.babylonjs.com/babylon.js"></script>
    <script src="https://cdn.babylonjs.com/loaders/babylonjs.loaders.min.js"></script>
    
    <script>
        // 获取 canvas 元素
        const canvas = document.getElementById('renderCanvas');
        
        // 创建 3D 引擎
        const engine = new BABYLON.Engine(canvas, true);
        
        // 创建场景
        const createScene = () => {
            const scene = new BABYLON.Scene(engine);
            
            // 设置背景颜色
            scene.clearColor = new BABYLON.Color4(0.1, 0.1, 0.2, 1);
            
            // 添加相机
            const camera = new BABYLON.ArcRotateCamera(
                'camera',
                Math.PI / 2,
                Math.PI / 3,
                10,
                BABYLON.Vector3.Zero(),
                scene
            );
            camera.attachControl(canvas, true);
            
            // 添加光源
            const light = new BABYLON.HemisphericLight(
                'light',
                new BABYLON.Vector3(0, 1, 0),
                scene
            );
            light.intensity = 0.7;
            
            // 创建基本几何体
            const sphere = BABYLON.MeshBuilder.CreateSphere('sphere', {
                diameter: 2,
                segments: 32
            }, scene);
            sphere.position.x = -2;
            
            const box = BABYLON.MeshBuilder.CreateBox('box', {
                size: 2
            }, scene);
            box.position.x = 2;
            
            // 创建材质
            const sphereMaterial = new BABYLON.StandardMaterial('sphereMat', scene);
            sphereMaterial.diffuseColor = new BABYLON.Color3(1, 0, 0);
            sphere.material = sphereMaterial;
            
            const boxMaterial = new BABYLON.StandardMaterial('boxMat', scene);
            boxMaterial.diffuseColor = new BABYLON.Color3(0, 1, 0);
            box.material = boxMaterial;
            
            return scene;
        };
        
        const scene = createScene();
        
        // 注册渲染循环
        engine.runRenderLoop(() => {
            scene.render();
        });
        
        // 处理窗口大小变化
        window.addEventListener('resize', () => {
            engine.resize();
        });
    </script>
</body>
</html>
```

#### 方式二：NPM 安装（推荐项目使用）

```bash
npm install @babylonjs/core @babylonjs/loaders
```

```typescript
// TypeScript 项目
import { Engine, Scene, ArcRotateCamera, Vector3, HemisphericLight, MeshBuilder, StandardMaterial, Color3, Color4 } from '@babylonjs/core';

// 初始化代码...
```

---

## 核心概念详解

### 1. 引擎（Engine）

Engine 是 Babylon.js 的核心，负责管理 WebGL 上下文和渲染循环。

```javascript
const engine = new BABYLON.Engine(canvas, true, {
    preserveDrawingBuffer: false,  // 是否在渲染后保留缓冲区
    stencil: true,                  // 启用模板缓冲区
    antialias: true                 // 启用抗锯齿
});
```

### 2. 场景（Scene）

Scene 是所有 3D 对象的容器。

```javascript
const scene = new BABYLON.Scene(engine);
scene.clearColor = new BABYLON.Color4(0, 0, 0, 1);  // 黑色背景
scene.gravity = new BABYLON.Vector3(0, -9.81, 0);   // 重力
```

### 3. 相机（Camera）

Babylon.js 提供多种相机类型：

```javascript
// 弧形旋转相机（最常用）
const arcCamera = new BABYLON.ArcRotateCamera(
    'arcCamera',
    0,           // alpha: 水平旋转角度
    Math.PI / 2, // beta: 垂直旋转角度
    10,          // radius: 距离目标的距离
    new BABYLON.Vector3(0, 0, 0), // target: 目标点
    scene
);

// 自由飞行相机（FPS 风格）
const freeCamera = new BABYLON.FreeCamera(
    'freeCamera',
    new BABYLON.Vector3(0, 5, -10),
    scene
);

// 跟随相机
const followCamera = new BABYLON.FollowCamera(
    'followCamera',
    new BABYLON.Vector3(0, 5, -10),
    scene
);
followCamera.lockedTarget = playerMesh;
```

### 4. 光源（Light）

```javascript
// 半球光（环境光）
const hemiLight = new BABYLON.HemisphericLight(
    'hemiLight',
    new BABYLON.Vector3(0, 1, 0),
    scene
);
hemiLight.intensity = 0.5;

// 方向光
const dirLight = new BABYLON.DirectionalLight(
    'dirLight',
    new BABYLON.Vector3(-1, -2, -1),
    scene
);
dirLight.position = new BABYLON.Vector3(20, 40, 20);

// 点光源
const pointLight = new BABYLON.PointLight(
    'pointLight',
    new BABYLON.Vector3(0, 10, 0),
    scene
);

// 聚光灯
const spotLight = new BABYLON.SpotLight(
    'spotLight',
    new BABYLON.Vector3(0, 30, -10),
    new BABYLON.Vector3(0, -1, 0),
    Math.PI / 3,
    2,
    scene
);
```

### 5. 网格（Mesh）

```javascript
// 创建立方体
const box = BABYLON.MeshBuilder.CreateBox('box', {
    size: 1,
    width: 2,
    height: 3,
    depth: 1
}, scene);

// 创建球体
const sphere = BABYLON.MeshBuilder.CreateSphere('sphere', {
    diameter: 2,
    segments: 32
}, scene);

// 创建圆柱体
const cylinder = BABYLON.MeshBuilder.CreateCylinder('cylinder', {
    height: 3,
    diameterTop: 1,
    diameterBottom: 1
}, scene);

// 创建平面
const plane = BABYLON.MeshBuilder.CreatePlane('plane', {
    size: 10,
    sideOrientation: BABYLON.Mesh.DOUBLESIDE
}, scene);

// 从外部加载模型
BABYLON.SceneLoader.ImportMesh(
    '',                    // 模型名称（空表示全部）
    'models/',             // 模型路径
    'model.glb',           // 模型文件
    scene,
    (meshes) => {
        console.log('模型加载成功', meshes);
    },
    (progress) => {
        console.log('加载进度', progress);
    },
    (error) => {
        console.error('加载失败', error);
    }
);
```

### 6. 材质（Material）

```javascript
// 标准材质
const standardMat = new BABYLON.StandardMaterial('standardMat', scene);
standardMat.diffuseColor = new BABYLON.Color3(1, 0, 0);  // 红色
standardMat.specularColor = new BABYLON.Color3(0.5, 0.5, 0.5);
standardMat.emissiveColor = new BABYLON.Color3(0.1, 0.1, 0.1);
standardMat.alpha = 0.9;  // 透明度

// PBR 材质（基于物理的渲染）
const pbrMat = new BABYLON.PBRMaterial('pbrMat', scene);
pbrMat.albedoColor = new BABYLON.Color3(0.8, 0.2, 0.2);
pbrMat.metallic = 0.5;
pbrMat.roughness = 0.3;

// 加载纹理
const textureMat = new BABYLON.StandardMaterial('textureMat', scene);
textureMat.diffuseTexture = new BABYLON.Texture('textures/diffuse.png', scene);
textureMat.bumpTexture = new BABYLON.Texture('textures/normal.png', scene);

// 应用材质
mesh.material = standardMat;
```

---

## 动画基础

### 1. 简单旋转动画

```javascript
// 在渲染循环中更新
engine.runRenderLoop(() => {
    box.rotation.y += 0.01;
    sphere.rotation.x += 0.005;
    scene.render();
});
```

### 2. 使用 Animation 类

```javascript
// 创建旋转动画
const rotationAnimation = new BABYLON.Animation(
    'rotationAnim',
    'rotation.y',
    30,                          // 帧率
    BABYLON.Animation.ANIMATIONTYPE_FLOAT,
    BABYLON.Animation.ANIMATIONLOOPMODE_CYCLE
);

const keys = [
    { frame: 0, value: 0 },
    { frame: 60, value: Math.PI * 2 }
];

rotationAnimation.setKeys(keys);
box.animations.push(rotationAnimation);

// 播放动画
scene.beginAnimation(box, 0, 60, true);
```

### 3. 使用 Action Manager 交互

```javascript
// 添加点击交互
box.actionManager = new BABYLON.ActionManager(scene);
box.actionManager.registerAction(
    new BABYLON.ExecuteCodeAction(
        BABYLON.ActionManager.OnPickTrigger,
        () => {
            console.log('立方体被点击了！');
            box.material.diffuseColor = new BABYLON.Color3.Random();
        }
    )
);

// 添加悬停交互
box.actionManager.registerAction(
    new BABYLON.ExecuteCodeAction(
        BABYLON.ActionManager.OnPointerOverTrigger,
        () => {
            document.getElementById('renderCanvas').style.cursor = 'pointer';
        }
    )
);

box.actionManager.registerAction(
    new BABYLON.ExecuteCodeAction(
        BABYLON.ActionManager.OnPointerOutTrigger,
        () => {
            document.getElementById('renderCanvas').style.cursor = 'default';
        }
    )
);
```

---

## 物理引擎集成

```javascript
// 启用物理引擎
scene.enablePhysics(
    new BABYLON.Vector3(0, -9.81, 0),  // 重力
    new BABYLON.CannonJSPlugin()       // 使用 Cannon.js
);

// 创建立方体并添加物理
const box = BABYLON.MeshBuilder.CreateBox('box', { size: 1 }, scene);
box.physicsImpostor = new BABYLON.PhysicsImpostor(
    box,
    BABYLON.PhysicsImpostor.BoxImpostor,
    { mass: 1, restitution: 0.7 },  // 质量和弹性
    scene
);

// 创建地面
const ground = BABYLON.MeshBuilder.CreateGround('ground', {
    width: 20,
    height: 20
}, scene);
ground.physicsImpostor = new BABYLON.PhysicsImpostor(
    ground,
    BABYLON.PhysicsImpostor.BoxImpostor,
    { mass: 0 },  // 质量为 0 表示静态物体
    scene
);
```

---

## 实战：创建一个可交互的 3D 场景

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <title>Babylon.js 交互场景</title>
    <style>
        html, body { margin: 0; padding: 0; width: 100%; height: 100%; }
        #renderCanvas { width: 100%; height: 100%; touch-action: none; }
        #info {
            position: absolute;
            top: 10px;
            left: 10px;
            color: white;
            font-family: Arial;
            background: rgba(0,0,0,0.5);
            padding: 10px;
            border-radius: 5px;
        }
    </style>
</head>
<body>
    <canvas id="renderCanvas"></canvas>
    <div id="info">
        <h3>Babylon.js 交互场景</h3>
        <p>左键旋转 | 右键平移 | 滚轮缩放</p>
        <p>点击物体改变颜色</p>
    </div>
    
    <script src="https://cdn.babylonjs.com/babylon.js"></script>
    
    <script>
        const canvas = document.getElementById('renderCanvas');
        const engine = new BABYLON.Engine(canvas, true);
        
        const createScene = () => {
            const scene = new BABYLON.Scene(engine);
            scene.clearColor = new BABYLON.Color4(0.05, 0.05, 0.1, 1);
            
            // 相机
            const camera = new BABYLON.ArcRotateCamera(
                'camera',
                -Math.PI / 2,
                Math.PI / 2.5,
                15,
                BABYLON.Vector3.Zero(),
                scene
            );
            camera.attachControl(canvas, true);
            camera.lowerBetaLimit = 0.1;
            camera.upperBetaLimit = (Math.PI / 2) * 0.99;
            camera.lowerRadiusLimit = 5;
            camera.upperRadiusLimit = 30;
            
            // 光源
            const hemiLight = new BABYLON.HemisphericLight(
                'hemiLight',
                new BABYLON.Vector3(0, 1, 0),
                scene
            );
            hemiLight.intensity = 0.6;
            
            const dirLight = new BABYLON.DirectionalLight(
                'dirLight',
                new BABYLON.Vector3(-1, -2, -1),
                scene
            );
            dirLight.position = new BABYLON.Vector3(20, 40, 20);
            dirLight.intensity = 0.5;
            
            // 地面
            const ground = BABYLON.MeshBuilder.CreateGround('ground', {
                width: 30,
                height: 30
            }, scene);
            const groundMat = new BABYLON.StandardMaterial('groundMat', scene);
            groundMat.diffuseColor = new BABYLON.Color3(0.2, 0.3, 0.2);
            ground.material = groundMat;
            
            // 创建多个几何体
            const meshes = [];
            
            // 立方体
            const box = BABYLON.MeshBuilder.CreateBox('box', { size: 2 }, scene);
            box.position = new BABYLON.Vector3(-4, 1, 0);
            const boxMat = new BABYLON.StandardMaterial('boxMat', scene);
            boxMat.diffuseColor = new BABYLON.Color3(1, 0.3, 0.3);
            box.material = boxMat;
            meshes.push(box);
            
            // 球体
            const sphere = BABYLON.MeshBuilder.CreateSphere('sphere', {
                diameter: 2
            }, scene);
            sphere.position = new BABYLON.Vector3(0, 1, 0);
            const sphereMat = new BABYLON.StandardMaterial('sphereMat', scene);
            sphereMat.diffuseColor = new BABYLON.Color3(0.3, 1, 0.3);
            sphere.material = sphereMat;
            meshes.push(sphere);
            
            // 圆柱体
            const cylinder = BABYLON.MeshBuilder.CreateCylinder('cylinder', {
                height: 3,
                diameterTop: 1,
                diameterBottom: 1
            }, scene);
            cylinder.position = new BABYLON.Vector3(4, 1.5, 0);
            const cylinderMat = new BABYLON.StandardMaterial('cylinderMat', scene);
            cylinderMat.diffuseColor = new BABYLON.Color3(0.3, 0.3, 1);
            cylinder.material = cylinderMat;
            meshes.push(cylinder);
            
            // 为每个物体添加点击交互
            meshes.forEach(mesh => {
                mesh.actionManager = new BABYLON.ActionManager(scene);
                mesh.actionManager.registerAction(
                    new BABYLON.ExecuteCodeAction(
                        BABYLON.ActionManager.OnPickTrigger,
                        () => {
                            mesh.material.diffuseColor = new BABYLON.Color3(
                                Math.random(),
                                Math.random(),
                                Math.random()
                            );
                        }
                    )
                );
            });
            
            // 添加动画
            scene.registerBeforeRender(() => {
                box.rotation.y += 0.01;
                sphere.rotation.x += 0.005;
                sphere.rotation.z += 0.005;
                cylinder.rotation.y -= 0.01;
            });
            
            return scene;
        };
        
        const scene = createScene();
        
        engine.runRenderLoop(() => {
            scene.render();
        });
        
        window.addEventListener('resize', () => {
            engine.resize();
        });
    </script>
</body>
</html>
```

---

## 性能优化建议

1. **使用实例化（Instancing）**
   ```javascript
   const instances = [];
   for (let i = 0; i < 100; i++) {
       const instance = mesh.createInstance(`instance_${i}`);
       instance.position = new BABYLON.Vector3(
           Math.random() * 20 - 10,
           0,
           Math.random() * 20 - 10
       );
       instances.push(instance);
   }
   ```

2. **合理使用 LOD（Level of Detail）**
   ```javascript
   const lodNode = new BABYLON.Mesh('lodNode', scene);
   lodNode.addLODLevel(0, highDetailMesh);
   lodNode.addLODLevel(20, mediumDetailMesh);
   lodNode.addLODLevel(40, lowDetailMesh);
   ```

3. **合并网格**
   ```javascript
   const mergedMesh = BABYLON.Mesh.MergeMeshes(
       meshesToMerge,
       true,   // disposeSource
       true,   // allow32BitsIndices
       undefined,
       false,  // disposeSourceOnMergeFail
       true    // subdivideWithSubMeshes
   );
   ```

---

## 学习资源

- [Babylon.js 官网](https://babylonjs.com/)
- [官方文档](https://doc.babylonjs.com/)
- [Playground 在线编辑器](https://playground.babylonjs.com/)
- [GitHub 仓库](https://github.com/BabylonJS/Babylon.js)
- [官方论坛](https://forum.babylonjs.com/)

---

## 总结

Babylon.js 是一个功能全面、文档完善的 Web3D 引擎，特别适合：

- 需要 TypeScript 支持的项目
- 需要物理引擎和碰撞检测的游戏
- 企业级 3D 可视化应用
- VR/AR 应用开发

在下一篇文章中，我们将深入探讨 Babylon.js 的高级特性，包括节点材质、粒子系统和后期处理效果。
