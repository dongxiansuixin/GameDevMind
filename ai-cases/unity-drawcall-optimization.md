# 用 Cursor 优化 Unity 开放世界 Draw Call

> **一句话总结：** Unity 项目 Draw Call 3000+、帧率 22fps，Cursor 帮我从「方向建议」到「批量修复脚本」，但最终发现问题根因是靠我自己的领域经验——AI 是超级实习生，架构师还是我自己。

---

## 背景

我们在做一款**开放世界 RPG 手游**，Unity 2022 LTS + URP，目标机 iPhone 8。

核心野外场景：2km × 2km 的地形，1500+ 棵树，800+ 块石头，200+ 栋建筑。QA 报告传送到野外后帧率从 60 fps 暴跌到 22 fps，手机 5 分钟烫手。

我打开 Unity Profiler，贴脸开大：

```
Draw Call（不含UI）: 3200+
Batches Saved by Batching: ~200
SetPass Call: 180+
Triangles: 4.2M
FPS: 22
```

GPU wait 占了帧时间的 40%。明确是渲染瓶颈。

---

## 对话记录

### 第一轮：贴 Profiler，Cursor 给方向

> **我：**
> ```
> （贴了 Profiler 截图）
> 这个 Unity 场景 Draw Call 3200+，帧率只有 22fps。
> GPU WaitForPresent 占 40%。帮我分析原因并给出优化方案。
> ```
>
> **Cursor（摘要）：**
>
> Cursor 分析得很专业：
> 1. **Batches Saved by Batching 只有 ~200**，说明动态批处理和静态批处理几乎没生效
> 2. **SetPass Call 180+**，说明材质种类太多（每次切换材质就是一个 SetPass）
> 3. 建议方向：
>    - 合并相同 Shader 的材质 → 创建材质图集（Texture Atlas）
>    - 检查 Static Batching 是否启用
>    - 植被用 GPU Instancing
>    - LOD Group 减少远处三角面
>
> 方向都对，但我试了材质合并后，Draw Call 只降到 2800——还差得远。

**我的感受：** Cursor 像一个有经验的 Unity 开发者，知道所有标准优化手段。但它只是「列举可能原因」，没有深入分析「为什么你的场景合批率这么低」。

---

### 第二轮：追问根因，Cursor 发现关键遗漏

> **我：**
> ```
> 我合并了材质，Draw Call 只从 3200 降到 2800。
> 为什么静态物体没有合批？场景里 200 栋建筑理论上都是静态的，
> Batches Saved 却只有 ~200。
> ```

> **Cursor（摘要）：**
>
> Cursor 让我检查一个我完全忽略的东西——**Static 标记**：
>
> ```
> 请在 Hierarchy 中选中一栋建筑，查看 Inspector 右上角的 Static 复选框。
> 如果只勾选了部分子选项（比如只勾了 Navigation Static），
> Unity 不会把它纳入 Static Batching。
> 你需要确认 Batching Static 单独被勾选。
> ```
>
> 我检查后倒吸一口凉气：**200 栋建筑里，有 160 栋的 Static 标记不完整。**
> 美术同学在建模时只勾了 `Navigation Static` 和 `Occluder Static`，关键的 `Batching Static` 是空的。
>
> 批量修正 160 个 GameObject 是个体力活。我让 Cursor 写脚本。

**我的感受：** 这才是 AI 协作的关键时刻——不是 AI 自己发现了问题，而是**我的追问指向了正确方向**，AI 在那个方向上做了深度诊断。如果我不知道「合批需要 Static 标记」，就不会问这个问题。

---

### 第三轮：批量修复脚本，Cursor 的真正强项

> **我：**
> ```
> 帮我写一个 Unity Editor 脚本，遍历场景中所有带有 MeshRenderer 的 GameObject，
> 如果它属于 "Building" 或 "Prop" Layer，自动设置：
> 1. StaticEditorFlags.BatchingStatic = true
> 2. 同时把 MaterialPropertyBlock 应用到所有使用相同材质的对象上，
>    让它们在 GPU Instancing 时能合并（但保持不同颜色/参数）
> 3. 输出处理了多少个对象
> ```

> **Cursor（摘要）：**
>
> Cursor 生成一个完整 Editor 窗口脚本（约 80 行）：
>
> ```csharp
> [MenuItem("Tools/Batch Fix Static Batching")]
> public static void FixStaticBatching()
> {
>     int fixedCount = 0;
>     var renderers = FindObjectsOfType<MeshRenderer>();
>     
>     foreach (var r in renderers)
>     {
>         if (r.gameObject.layer == LayerMask.NameToLayer("Building") ||
>             r.gameObject.layer == LayerMask.NameToLayer("Prop"))
>         {
>             var flags = GameObjectUtility.GetStaticEditorFlags(r.gameObject);
>             flags |= StaticEditorFlags.BatchingStatic;
>             GameObjectUtility.SetStaticEditorFlags(r.gameObject, flags);
>             
>             // 设置 MaterialPropertyBlock 用于 GPU Instancing
>             var mpb = new MaterialPropertyBlock();
>             r.GetPropertyBlock(mpb);
>             r.SetPropertyBlock(mpb);
>             
>             fixedCount++;
>         }
>     }
>     Debug.Log($"Fixed {fixedCount} GameObjects for batching.");
> }
> ```
>
> 还贴心地加了进度条（`EditorUtility.DisplayProgressBar`），因为 800+ 个对象处理需要几秒钟。

**我的感受：** 这就是 AI 的甜点区——**重复性的工具脚本**。逻辑清晰、API 调用准确、边界条件处理合理。我自己写大概要 20 分钟查文档，AI 30 秒就生成好了。

---

## 最终成果

执行批量修复后，再加上之前的手动优化的累积效果：

| 指标 | 优化前 | 优化后 | 改善 |
|------|--------|--------|------|
| Draw Call | 3,200 | 480 | **-85%** |
| SetPass Call | 180 | 42 | **-77%** |
| Batches Saved | ~200 | 1,850 | **+825%** |
| Triangles | 4.2M | 1.6M | **-62%** |
| GPU Wait | 40% | 12% | **-70%** |
| 帧率 | 22 fps | 58 fps | ✅ |

最终方案组合拳：
1. ✅ Static 标记批量修复（Cursor 脚本）— 贡献最大的单步优化
2. ✅ 材质图集合并 — 减少 SetPass
3. ✅ GPU Instancing 植被 — 树木和灌木
4. ✅ LOD Group — 远处用低模
5. ✅ Occlusion Culling 烘焙 — 建筑遮挡

---

## 关键收获

### 1. AI 擅长「执行」，但「诊断」需要人的领域知识

Cursor 第一轮给了标准优化 checklist，方向都对但不够精准。只有当我说出「为什么静态物体没合批」时，它才沿着这个方向深挖到 Static 标记问题。

**如果我不具备「合批需要 Static 标记」这个领域知识，我根本不会问出这个问题。**

### 2. AI 的超级能力：工具脚本

Editor 脚本、批处理工具、数据迁移脚本——这类「逻辑清晰 + API 明确 + 可验证」的代码，AI 几乎是零成本生成。人工写 20 分钟，AI 写 30 秒，还带进度条。

### 3. 把 AI 当「超级实习生」

Cursor 像极了一个聪明但缺乏经验的实习生：
- 你说「优化性能」，它给你 checklist
- 你说「查为什么 XX 没生效」，它能追踪到根因
- 你说「写个脚本批量处理」，它写得又快又好

但**架构决策、方向判断、优先级排序**，最终还是我来。

---

## 图谱知识点映射

- [3.1.3 客户端优化](../mds/3.研发能力/3.1.3.客户端优化.md) — Draw Call、GPU Instancing、Static Batching
- [3.1.2 客户端3D场景开发](../mds/3.研发能力/3.1.2.客户端3D场景开发.md) — 场景管理、LOD
- [1.1.3 C#语言](../mds/1.基础能力/1.1.3.C%23语言.md) — Editor 脚本、反射

> 🏥 相关实战案例：[开放世界卡顿 — Draw Call 3000+ 到 480 的优化实战](../cases/drawcall-optimization.md)
