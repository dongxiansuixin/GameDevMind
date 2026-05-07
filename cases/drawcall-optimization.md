# Draw Call 优化实战案例：从 22fps 到 58fps 的全流程复盘

> **一句话总结：** 一个开放世界 RPG 手游野外场景，帧率从 60fps 暴跌至 22fps，通过材质合并、LOD、静态批处理、GPU Instancing、遮挡剔除和 UI 优化六管齐下，最终将 Draw Call 从 3000+ 降至 480，帧率恢复到 58fps。

---

## 背景

我们正在开发一款**开放世界 RPG 手游**，引擎为 **Unity 2022 LTS + URP**，目标机型锁定 **iPhone 8（A11）**和**骁龙 845（Adreno 630）**——2018-2019 年的中高端机型，代表了项目要覆盖的底线性能。

问题的触发场景是游戏中的**核心野外区域**：

- 地形面积 **2km × 2km**，包含草地、森林、山地等多种地貌
- 植被极度密集：场景中散落 **1500+ 棵树、800+ 块石头、大量灌木**
- 建筑群：**200+ 栋**可交互和纯装饰建筑（村庄、哨塔、遗迹等）
- 同时存在战斗 HUD、任务追踪、小地图等 UI 系统

QA 提交的 Bug 描述很简单但致命：

> *"角色传送到野外区域后，帧率立刻从 60fps 掉到 22fps，5 分钟后手机烫得拿不住。"*

这个问题直接威胁项目上线进度——野外区域是核心玩法载体，占玩家游戏时长的 60% 以上，性能不达标意味着无法通过发行审查。

---

## 症状

### 1. Unity Profiler 抓帧数据（iPhone 8 真机）

| 指标 | 数值 | 健康值参考（移动端） |
|------|------|---------------------|
| **Draw Call**（不含 UI） | **3,200+** | < 600 |
| **Batches Saved by Batching** | ~200 | 应接近 Draw Call 总数 |
| **SetPass Call** | 180+ | < 80 |
| **Triangles** | 4.2M | < 2M |
| **GPU WaitForPresent** | 占帧时间 **40%** | < 10% |
| CPU 主线程 | 占帧时间 35% | 正常 |
| 帧率 | **22 fps** | 60 fps |

### 2. 关键发现

- **Batches Saved by Batching ≈ 200**，而 Draw Call 3000+，说明**几乎没有合批**——动态批处理和静态批处理几乎完全失效。
- **WaitForPresent 占 40%** 帧时间，这是典型的 **GPU 严重瓶颈**信号：渲染管线的提交队列太长，GPU 根本处理不过来。
- 场景中 1500+ 棵树、800+ 块石头、200+ 栋建筑——**全部使用独立的 MeshRenderer**，每一棵树的材质实例都不同。

### 3. 发热验证

- 红外测温：运行 5 分钟后，iPhone 8 背部温度 **47°C**，屏幕亮度被迫降至 50%（iOS 温控降频）。
- 骁龙 845 机型（小米 8）：10 分钟后触发 **50% CPU 降频**，帧率进一步跌至 15fps。

---

## 排查过程

### 第一步：Unity Frame Debugger —— 混乱的渲染顺序

> 工具：Window → Analysis → Frame Debugger

打开 Frame Debugger 逐 Draw Call 查看，立刻发现：

```
Draw 1-50:   地形（Terrain Base Map）
Draw 51:     一棵树（材质A）
Draw 52:     一块石头（材质B）
Draw 53:     一棵树（材质C）    ← 材质C ≠ 材质A，打断合批！
Draw 54:     建筑墙壁（材质D）
Draw 55:     一棵树（材质A）    ← 材质A 又出现了，但已被打断
Draw 56:     一块石头（材质E）  ← 又是一块不同材质的石头
...
```

**结论：渲染顺序混乱，相同材质的物体被其他材质反复打断。** Unity 的渲染队列是按 "渲染顺序 = 相机距离 + 材质索引" 排列的，但这里材质种类太多，导致同材质物体天然被隔开。

从这个截图级的信息可以直接看出：材质种类太多是头号嫌疑犯。

---

### 第二步：材质审计 —— 60+ 种材质的真相

我们在 Project 视窗搜索 `*.mat`，然后编写了一个编辑器脚本对场景中实际引用的材质做去重分析：

```csharp
// 编辑器工具：场景材质统计
[MenuItem("Tools/性能/统计场景材质")]
static void AuditSceneMaterials()
{
    var renderers = FindObjectsOfType<MeshRenderer>();
    var materialMap = new Dictionary<Material, List<string>>();

    foreach (var r in renderers)
    {
        foreach (var mat in r.sharedMaterials)
        {
            if (mat == null) continue;
            if (!materialMap.ContainsKey(mat))
                materialMap[mat] = new List<string>();
            materialMap[mat].Add(r.name);
        }
    }

    Debug.Log($"场景中唯一材质数量: {materialMap.Count}");
    // 结果输出：场景中唯一材质数量: 67
}
```

结果触目惊心：

- **67 种材质**被实际引用
- 其中很多材质的 Shader 和贴图完全相同，仅仅是颜色或平铺参数不同
- 比如 "Tree_Oak_01"、"Tree_Oak_02" ... "Tree_Oak_12" 是 12 种**完全相同**的材质，只是美术在不同 Prefab 上手动拖拽时 Unity 自动创建了材质实例副本

**问题本质：美术在 Inspector 里改了 Material 的某个属性 → Unity 自动 `Instantiate(material)` 创建了新实例 → 合批彻底失效。**

---

### 第三步：Mesh 审查 —— 没有 LOD！

使用场景视图的 **LOD 可视化模式**（需要在 Quality Settings 中开启 LOD 开关），发现所有物体的 LOD Group 组件的 `LOD 0` 至 `LOD 2` 三个槽位**全部为空**，只挂了一个 Mesh。

进一步检查树的 Mesh 资产：

- 每棵树 5000 面（三角面），合 1500 棵树 = **750 万面**仅来自树木
- 200 米外的树和 2 米外的树使用的是**同一个 5000 面的 Mesh**
- 在 iPhone 8 的 750×1334 分辨率下，200 米外的一棵树在屏幕上仅占 **~15 像素高**，却渲染了 5000 个三角面 = 每个像素 300+ 面，极度浪费

**透视分析：GPU 的顶点着色器阶段被大量不可见的细节拖垮，同时这些三角面对屏幕的贡献几乎为零。**

---

### 第四步：遮挡剔除 —— 从未开启

检查 Window → Rendering → Occlusion Culling：

- Occlusion Culling 面板显示 "No occlusion data"
- **整个场景从未烘焙过遮挡数据**
- 在游戏运行状态下，旋转摄像机 180 度背对建筑群，Draw Call 数量不变

这意味着：**摄像机背后的 200+ 栋建筑、背面的树和石头全都在渲染。** GPU 在做大量无用功。

简单估算被遮挡的物体占比：在野外区域任意位置，摄像机视锥体外的物体约占 **35-40%**，加上被大型建筑遮挡的物体，总计约 **50% 的渲染是浪费的**。

---

### 第五步：UI 审查 —— Canvas 重建的隐形杀手

前面四步集中在场景渲染上，但我们注意到 Profiler 中 `Canvas.BuildBatch` 每帧都在执行，耗时约 **8ms**。

定位到战斗 HUD 的血条系统：

```csharp
// 问题代码（简化）
public class HealthBar : MonoBehaviour
{
    public void Show()
    {
        gameObject.SetActive(true);  // ← 每帧触发 Canvas Rebuild！
    }

    public void Hide()
    {
        gameObject.SetActive(false); // ← 再次触发 Rebuild！
    }

    void Update()
    {
        // 每帧检查是否需要显示/隐藏
        if (ShouldShow())
            Show();
        else
            Hide();
    }
}
```

**`SetActive(true/false)` 会导致该物体所在 Canvas 的整个批次重建**（Canvas.BuildBatch），因为 Canvas 需要重新计算所有 UI 元素的顶点、UV、颜色缓冲区。这个问题在 30 个敌人同时存在时被放大：每帧 30 次 SetActive 调用 → 30 次 Canvas Rebuild → 8ms CPU 时间白白浪费。

---

## 根因分析（多个根因叠加）

这个问题不是单一原因造成的，而是**多个根因叠加**形成的性能风暴：

```
Draw Call 3000+ 的根因分解：

├── 材质未复用 (40%)
│   └── 67 种材质 → 只有 ~10 种是真正不同的
│   └── 美术拖拽 Prefab 时 Unity 自动创建材质实例副本
│   └── 直接后果：动态批处理/静态批处理全部失效
│
├── 无 LOD (25%)
│   └── 远处物体与近处使用相同 Mesh
│   └── 750 万面仅来自树木，远超移动端 GPU 处理能力
│   └── 直接后果：顶点处理成为 GPU 瓶颈
│
├── 静态批处理未标记 (15%)
│   └── 建筑、石头等静态物体未勾选 Static
│   └── 直接后果：失去了最廉价的合批手段
│
├── GPU Instancing 未启用 (10%)
│   └── 大量重复物体（树、石头）逐物体提交
│   └── 直接后果：Draw Call 数量随物体数量线性增长
│
└── UI 刷新机制错误 (10%)
    └── setActive 每帧触发 Canvas.BuildBatch
    └── 直接后果：CPU 白白浪费 8ms 每帧
```

**核心教训：性能问题很少是单点故障，通常是多个"小问题"叠加，在每个环节都吃掉一点性能余量，最终压垮整个系统。**

---

## 解决方案（按优先级实施）

### 1. 材质合并 —— 从 67 种到 12 种

**目标：让相同 Shader + 相同贴图的物体真正共享材质。**

#### 操作步骤

**Step A：材质审计和归类**

编写编辑器脚本，扫描所有 MeshRenderer，按 (Shader GUID, MainTexture GUID) 分组：

```csharp
[MenuItem("Tools/性能/材质去重合并")]
static void MergeMaterials()
{
    var renderers = FindObjectsOfType<MeshRenderer>();
    var groupMap = new Dictionary<string, Material>();

    foreach (var r in renderers)
    {
        var mats = r.sharedMaterials;
        for (int i = 0; i < mats.Length; i++)
        {
            var mat = mats[i];
            if (mat == null) continue;

            // 按 Shader + 主贴图 分组合并
            string key = $"{mat.shader.name}|{mat.mainTexture?.GetInstanceID()}";

            if (!groupMap.ContainsKey(key))
            {
                groupMap[key] = mat; // 保留第一个作为标准引用
            }

            var newMats = r.sharedMaterials;
            newMats[i] = groupMap[key];
            r.sharedMaterials = newMats;
        }
    }

    Debug.Log($"材质合并完成：{groupMap.Count} 种去重后材质");
}
```

运行后材质种类从 67 降到了 **12 种**：
- 植被类：3 种（树皮、树叶、灌木）
- 石头类：2 种（花岗岩、砂岩）
- 建筑类：4 种（木墙、石墙、屋顶、地板）
- 地形类：2 种（草地、泥土）
- 其他：1 种（特效/粒子）

**Step B：用 MaterialPropertyBlock 处理差异化**

合并后，原来通过不同材质实例实现的颜色/参数差异需要用 MaterialPropertyBlock 替代：

```csharp
// 使用 MaterialPropertyBlock 替代材质实例副本
public class TreeVariation : MonoBehaviour
{
    [SerializeField] private Color barkColor = Color.white;
    [SerializeField] private float leafScale = 1.0f;

    private static MaterialPropertyBlock _mpb;
    private MeshRenderer _renderer;

    private static readonly int ColorID = Shader.PropertyToID("_BaseColor");
    private static readonly int LeafScaleID = Shader.PropertyToID("_LeafScale");

    void Awake()
    {
        _renderer = GetComponent<MeshRenderer>();
        _mpb ??= new MaterialPropertyBlock();

        _renderer.GetPropertyBlock(_mpb);
        _mpb.SetColor(ColorID, barkColor);
        _mpb.SetFloat(LeafScaleID, leafScale);
        _renderer.SetPropertyBlock(_mpb);
    }
}
```

**收益：这一步让原本因材质不同而无法合批的物体重新获得合批能力，为后续的静态批处理和 GPU Instancing 铺平道路。**

---

### 2. 静态批处理 —— Draw Call -600

**目标：将场景中永不移动的物体标记为 Static，让 Unity 在构建时将它们合并。**

操作：
1. 选中场景中所有不会移动的物体（建筑、石头、栅栏、路标等）
2. 在 Inspector 右上角勾选 **Static**（或只勾选 `Batching Static`）
3. 注意：**只标记 `Batching Static`**，不要勾选 `Occluder Static` 和 `Occludee Static` 直到烘焙遮挡剔除时再处理

关键注意事项：

- 静态批处理会将多个 Mesh 合并为一个大 Mesh，**会占用额外内存**。合并后的顶点数据会被复制到新的 VBO 中。
- 不要将**太大的 Mesh** 标记为 Static——Unity 有 64K 顶点限制。
- 树木不要标记 Static（它们在风中需要摆动动画）。

实施后效果：静态物体被合并，**Draw Call 减少约 600**。

---

### 3. LOD Group —— GPU 负载降 40%

**目标：为所有植被和建筑设置 3 级 LOD，远距离使用低面数模型。**

#### 树（以中等橡树为例）

| LOD 级别 | 面数 | 切换距离（iPhone 8） | 占屏比例 |
|----------|------|---------------------|----------|
| LOD 0 | 5000 面 | 0-30m | > 10% 屏高 |
| LOD 1 | 1500 面 | 30-80m | 3%-10% |
| LOD 2 | 300 面 | 80-150m | < 3% |
| Culled | 0 | > 150m | — |

**LOD 切换距离计算公式：**

```
切换距离 = 物体包围盒高度 × LOD 系数 / (2 × tan(FOV/2) × 目标屏占比)

对于 iPhone 8（FOV≈60°, 屏高 1334px）：
LOD0→LOD1: 包围盒高度 × 0.5 / (2 × tan(30°) × 0.1) ≈ 30m
LOD1→LOD2: 包围盒高度 × 0.5 / (2 × tan(30°) × 0.04) ≈ 80m
```

#### 石头/灌木

| LOD 级别 | 面数 | 切换距离 |
|----------|------|----------|
| LOD 0 | 800 面 | 0-15m |
| LOD 1 | 200 面 | 15-40m |
| LOD 2 | 50 面 | 40-70m |
| Culled | 0 | > 70m |

#### LOD 制作流程

1. 美术在建模软件中制作各 LOD 级别的 Mesh
2. 导入 Unity 后，给 Prefab 添加 **LOD Group** 组件
3. 将各级别 Mesh 拖入对应槽位
4. 设置 Screen Relative Transition Height（使用上面计算的百分比）

#### 实测效果

设置 LOD 后重新在 iPhone 8 上运行：
- 摄像机下总三角面数从 4.2M 降至 **~1.8M**
- GPU 顶点着色器负载降低约 **40%**
- 视觉效果：在手机屏幕上几乎察觉不到 LOD 切换（Unity 的 Screen Percentage 过渡平滑）

---

### 4. GPU Instancing —— Draw Call -800

**目标：让相同 Mesh + 相同材质的物体（树、石头）通过 GPU Instancing 批量提交。**

#### 操作步骤

1. **材质启用 Instancing：**
   - 选中树皮材质 → Inspector → 勾选 **Enable GPU Instancing**
   - 同样操作树叶、灌木、石头材质

2. **Shader 兼容性检查：**
   - URP Lit Shader 默认支持 Instancing
   - 自定义 Shader 需要添加 `#pragma multi_compile_instancing` 并在顶点/片元函数中正确声明 `UNITY_VERTEX_INPUT_INSTANCE_ID`

3. **验证效果：**
   - 打开 Frame Debugger，查看 Draw Call 类型
   - 启用 Instancing 后，Draw Call 显示为 **"Draw Mesh (Instanced)"** 而非 "Draw Mesh"
   - 一次 Instanced Draw Call 可以渲染**数百个**同 Mesh 同材质的物体

#### 注意事项

- GPU Instancing 要求所有实例使用**完全相同的 Mesh 和 Material**
- MaterialPropertyBlock 的属性差异在 Instancing 下通过 `UNITY_INSTANCING_BUFFER` 方式传递，**仍然支持**
- 不要对标记了 Static 的物体同时启用 Instancing——Unity 会优先使用静态批处理
- 移动端 GPU 的 Instancing 支持：A11 (iPhone 8) 和 Adreno 630 (骁龙 845) 均支持

实施后效果：1500 棵树从 1500 个 Draw Call → 约 **12 个 Instanced Draw Call**（按树种分组）；800 块石头 → 约 **6 个 Instanced Draw Call**。**Draw Call 减少约 800。**

---

### 5. 遮挡剔除 —— 释放被遮挡物体的渲染开销

**目标：烘焙遮挡数据，让摄像机背后的和被建筑遮挡的物体自动剔除。**

#### 操作步骤

1. **标记遮挡相关 Static 标志：**
   - 建筑、大型石头等大体积物体：勾选 `Occluder Static` 和 `Occludee Static`
   - 小型物体（灌木、小石头）：仅勾选 `Occludee Static`

2. **调整烘焙参数：**
   ```
   Window → Rendering → Occlusion Culling → Bake
   - Smallest Occluder: 1.5 (适合建筑尺度)
   - Smallest Hole: 0.25
   - Backface Threshold: 100
   ```

3. **烘焙：** 点击 Bake，等待完成后可以在场景视图中选择 Occlusion Culling 可视化模式验证。

4. **运行时验证：** 使用 Occlusion Portal 处理室内-室外过渡，确保门洞/窗户等开口正确标记为开放区域。

#### 注意事项

- 遮挡剔除数据是**预烘焙**的，场景修改后需要重新烘焙
- 对于 2km×2km 的大场景，建议分区域烘焙（使用 Occlusion Area）
- 移动端不要设置过小的 `Smallest Occluder`，否则烘焙数据过大且运行时查询变慢

---

### 6. UI 优化 —— 消灭每秒 8ms 的 Canvas 重建

**问题回顾：** `SetActive(true/false)` 每帧触发 Canvas.BuildBatch，30 个血条每帧浪费 8ms。

#### 解决方案：CanvasGroup.alpha + 对象池

```csharp
public class HealthBarOptimized : MonoBehaviour
{
    [SerializeField] private CanvasGroup canvasGroup;
    private bool _isVisible = true;

    // 对象池
    private static Stack<HealthBarOptimized> _pool = new Stack<HealthBarOptimized>();

    public static HealthBarOptimized Get(Transform parent)
    {
        HealthBarOptimized bar;
        if (_pool.Count > 0)
        {
            bar = _pool.Pop();
            bar.gameObject.SetActive(true); // 只在从池中取出时调用一次
        }
        else
        {
            bar = Instantiate(prefab, parent);
        }
        return bar;
    }

    public void ReturnToPool()
    {
        gameObject.SetActive(false);
        _pool.Push(this);
    }

    public void SetVisible(bool visible)
    {
        if (_isVisible == visible) return; // 状态未变化，跳过

        _isVisible = visible;

        // 关键：用 CanvasGroup.alpha 代替 SetActive
        // CanvasGroup.alpha=0 → UI 不可见但不会触发 Rebuild
        // CanvasGroup.blocksRaycasts=false → 同时关闭射线检测
        canvasGroup.alpha = visible ? 1f : 0f;
        canvasGroup.blocksRaycasts = visible;
    }

    // 手动更新血条值 —— 直接操作 CanvasRenderer，完全绕过 Rebuild
    public void SetHealthPercent(float percent)
    {
        // 使用预生成的 Mesh 或直接修改顶点颜色
        // 而不是改动 RectTransform 触发 Layout Rebuild
        fillImage.fillAmount = percent; // Image.fillAmount 不会触发 Rebuild
    }
}
```

#### 关键认知

| 操作 | 是否触发 Canvas Rebuild | 适用场景 |
|------|----------------------|----------|
| `SetActive(true/false)` | ✅ **是！** | 仅用于对象池的 Get/Return |
| `CanvasGroup.alpha` | ❌ 否 | 频繁的显示/隐藏切换 |
| `Image.fillAmount` | ❌ 否 | 血条、进度条更新 |
| `RectTransform.sizeDelta` | ✅ **是！** | 避免每帧修改 |
| `Text.text` | ✅ **是！** | 使用 TextMeshPro + 缓存对比 |

**收益：Canvas.BuildBatch 从每帧 8ms 降至 ~0.3ms。**

---

## 效果

在 iPhone 8 上完成所有优化后，同场景重新抓取 Profiler 数据：

| 指标 | 改前 | 改后 | 改善幅度 |
|------|------|------|----------|
| **Draw Call** | 3,200+ | **480** | ↓ 85% |
| **Batches Saved by Batching** | ~200 | **1,400+** | ↑ 7× |
| **SetPass Call** | 180+ | **42** | ↓ 77% |
| **Triangles** | 4.2M | **1.6M** | ↓ 62% |
| **GPU WaitForPresent** | 40% 帧时间 | **6%** | ↓ 85% |
| **CPU Canvas.BuildBatch** | 8ms | **0.3ms** | ↓ 96% |
| **帧率（iPhone 8）** | **22 fps** | **58 fps** ✅ | ↑ 164% |
| **帧率（骁龙 845）** | 24 fps | **60 fps** ✅ | — |
| **5 分钟后温度** | 47°C（烫手） | **38°C（温热）** | ↓ 9°C |
| **材质种类** | 67 | **12** | ↓ 82% |

### 发热对比

- 改前：5 分钟后 iPhone 8 自动降亮度、降频，游戏体验崩溃
- 改后：30 分钟连续游戏，手机温热但不烫，无降频

### 合批分布（改后）

```
总 Draw Call: 480
├── 地形（内置）          : 18
├── 静态批处理（建筑/石头）: 120
├── GPU Instancing（树）  : 12
├── GPU Instancing（灌木）: 8
├── 动态物体（NPC/特效）   : 45
├── UI Canvas             : 22
├── 阴影/后处理           : 55
└── 其他                  : 200
```

---

## 经验教训

### 1. Draw Call 问题根源往往在美术资源流程，不在代码

这次排查最深刻的教训：**67 种材质没有一种是"故意"创建的。** 它们全是美术在日常工作中反复拖拽 Prefab、在 Inspector 里调色、改贴图 Tiling 时，Unity 静默创建的材质实例副本。

**流程优化建议：**
- 建立**材质命名规范**和**材质审核机制**
- 编写 CI/Meta 检查脚本，自动检测场景中材质种类超过阈值时告警
- 在美术工作流中推广 MaterialPropertyBlock 处理变体需求

### 2. LOD 不是可选项——移动端必须有

很多团队把 LOD 当作"锦上添花"的优化，但在这个案例中，仅 LOD 一项就让三角面数降低 60%+。移动端 GPU 的顶点处理能力远弱于桌面端（A11 约 400 GFLOPS，同时代桌面 GPU 约 5000+ GFLOPS），**LOD 是移动端 3D 游戏的准入门槛，不是可选项。**

### 3. Static 标记和 Occlusion Culling 是"免费"的性能收益

勾选几个 Static 复选框、点一次 Bake 按钮，换来：
- 静态批处理：~600 Draw Call 减少
- 遮挡剔除：额外 15-30% 的渲染量节省

**代价：零运行时开销。** 这是整个优化过程中 ROI 最高的两项操作。

### 4. UI 优化要理解 Canvas 重建机制，setActive 是性能杀手

Unity 的 Canvas 系统是"脏标记 + 全量重建"模型——任何子元素的 Transform、材质、文本变化都会把整个 Canvas 标记为 dirty，下一帧重建整个顶点缓冲区。

`SetActive(false/true)` 之所以昂贵，是因为：
1. GameObject 从 disabled → enabled 会触发 `OnEnable`
2. `OnEnable` 中 Graphic 组件调用 `SetAllDirty()`
3. `SetAllDirty()` 标记 Canvas 为 dirty
4. 下一帧 Canvas.BuildBatch 重建整个批次

**用 CanvasGroup.alpha 代替 SetActive** 可以完全绕过这个流程。

### 5. 用 Frame Debugger 逐帧分析，不要猜

"感觉是 XXX 的问题"是做性能优化的头号陷阱。Frame Debugger 让每一个 Draw Call 的前因后果都透明可见——材质打断合批、LOD 缺失、UI 重建，都不是"猜"出来的，而是**逐帧看出来的**。

**推荐排查流程：**
1. Profiler 定位瓶颈类别（CPU / GPU）
2. Frame Debugger 逐 Draw Call 分析渲染顺序
3. 针对性检查：材质、LOD、遮挡、UI
4. 逐项修复，每项修复后重新抓帧验证

---

## 图谱知识点映射

本案例涉及以下知识图谱节点：

- [客户端优化](../mds/3.研发能力/3.1.3.客户端优化.md) — 性能分析、Profiler 使用、优化方法论
- [UI 系统](../mds/3.研发能力/3.1.6.UI系统.md) — Canvas 重建机制、UI 性能优化
- [图形与渲染](../mds/2.技术能力/2.1.1.图形与渲染.md) — Draw Call、批处理、LOD、遮挡剔除、GPU Instancing

---

> **最后更新：** 2026-05-07
> **适用版本：** Unity 2022 LTS + URP
> **标签：** `#性能优化` `#DrawCall` `#LOD` `#移动端` `#Unity` `#实战案例`
