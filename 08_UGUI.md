# 八、UGUI

> 本章涵盖问题：
> - UGUI的渲染原理是什么？
> - Canvas的三种渲染模式有什么区别？
> - 什么是Canvas重建？如何优化？
> - Image和RawImage的区别是什么？
> - UGUI的事件系统是如何工作的？
> - 如何优化UGUI的性能？
> - ScrollView的优化方案有哪些？
> - Mask和RectMask2D的区别是什么？
> - 如何实现UI的层级管理？
> - UGUI的锚点和轴心是什么？

---

## 8.1 UGUI渲染原理

### 渲染流程

```
UGUI渲染流程：

1. Canvas收集所有UI元素
   ┌─────────────────────────────────────┐
   │ Canvas                              │
   │ ├─ Image1                           │
   │ ├─ Text1                            │
   │ ├─ Button1                          │
   │ │   ├─ Image2                       │
   │ │   └─ Text2                        │
   │ └─ Panel1                           │
   │     └─ Image3                       │
   └─────────────────────────────────────┘

2. 按深度排序，合并相同材质的元素
   ┌─────────────────────────────────────┐
   │ Batch 1: Image1, Image2, Image3    │ ← 相同材质
   │ Batch 2: Text1, Text2              │ ← 相同字体
   └─────────────────────────────────────┘

3. 生成网格数据
   ┌─────────────────────────────────────┐
   │ 顶点数据、UV、颜色 → Mesh          │
   └─────────────────────────────────────┘

4. 提交渲染
   ┌─────────────────────────────────────┐
   │ Canvas Renderer → GPU              │
   └─────────────────────────────────────┘
```

### Canvas组件

```csharp
// Canvas是UGUI的根组件
// 所有UI元素必须是Canvas的子物体

Canvas canvas = GetComponent<Canvas>();

// 重要属性
canvas.renderMode;      // 渲染模式
canvas.sortingOrder;    // 排序顺序
canvas.overrideSorting; // 是否覆盖父Canvas排序
canvas.pixelPerfect;    // 像素完美
```

---

## 8.2 Canvas渲染模式

### 三种模式对比

| 模式 | 说明 | 适用场景 |
|------|------|----------|
| Screen Space - Overlay | 覆盖在所有3D物体上 | 常规UI |
| Screen Space - Camera | 由指定相机渲染 | 需要3D效果的UI |
| World Space | 作为3D物体存在 | 血条、名字板 |

### Screen Space - Overlay

```
┌─────────────────────────────────────┐
│           屏幕                       │
│  ┌─────────────────────────────┐   │
│  │      3D场景                  │   │
│  │  ┌─────┐                    │   │
│  │  │物体 │                    │   │
│  │  └─────┘                    │   │
│  └─────────────────────────────┘   │
│  ┌─────────────────────────────┐   │
│  │      UI层（最上层）          │   │ ← Overlay
│  │  [按钮] [文本] [图片]        │   │
│  └─────────────────────────────┘   │
└─────────────────────────────────────┘

特点：
• 始终在最上层
• 不受相机影响
• 不能与3D物体穿插
• 性能最好
```

### Screen Space - Camera

```csharp
// 设置渲染相机
canvas.renderMode = RenderMode.ScreenSpaceCamera;
canvas.worldCamera = uiCamera;
canvas.planeDistance = 10f;  // UI平面距离相机的距离
```

```
┌─────────────────────────────────────┐
│           屏幕                       │
│  ┌─────────────────────────────┐   │
│  │      相机视野                │   │
│  │                              │   │
│  │  ┌─────┐    ┌─────────┐    │   │
│  │  │3D物体│    │ UI平面  │    │   │ ← 可以调整深度
│  │  └─────┘    │ [按钮]  │    │   │
│  │             └─────────┘    │   │
│  └─────────────────────────────┘   │
└─────────────────────────────────────┘

特点：
• 可以与3D物体穿插
• 可以应用后处理效果
• 可以使用粒子特效
```

### World Space

```csharp
// 世界空间UI
canvas.renderMode = RenderMode.WorldSpace;
canvas.GetComponent<RectTransform>().sizeDelta = new Vector2(200, 50);
```

```
┌─────────────────────────────────────┐
│           3D世界                     │
│                                     │
│      ┌─────────┐                   │
│      │ 敌人    │                   │
│      │  ┌───┐  │ ← 血条（World UI）│
│      │  │HP │  │                   │
│      │  └───┘  │                   │
│      └─────────┘                   │
│                                     │
│  ┌─────┐                           │
│  │玩家 │                           │
│  └─────┘                           │
└─────────────────────────────────────┘

特点：
• 作为3D物体存在
• 有透视效果
• 需要处理朝向（Billboard）
```

---

## 8.3 Canvas重建

### 什么是Canvas重建

```
Canvas重建分为两种：

1. Rebuild（重建）
   ├─ Layout Rebuild：布局重建
   │   └─ 当RectTransform改变时触发
   └─ Graphic Rebuild：图形重建
       └─ 当颜色、材质、网格改变时触发

2. Rebatch（重新合批）
   └─ 当Canvas下任何元素改变时触发
   └─ 重新计算合批，生成新的Mesh
```

### 触发重建的操作

```csharp
// 触发Layout Rebuild
rectTransform.sizeDelta = newSize;
rectTransform.anchoredPosition = newPos;
layoutGroup.padding = newPadding;

// 触发Graphic Rebuild
image.color = newColor;
image.sprite = newSprite;
text.text = newText;

// 触发Rebatch
// 上述任何操作都会触发
// 以及：SetActive、层级改变等
```

### 重建的性能影响

```
Canvas重建流程：

1. 标记脏（Dirty）
   └─ 元素改变时标记

2. 等待帧末尾
   └─ 所有脏元素一起处理

3. 重建
   ├─ 遍历所有脏元素
   ├─ 重新计算布局
   ├─ 重新生成网格
   └─ 重新合批

性能问题：
• 一个元素改变，整个Canvas重新合批
• 复杂Canvas重建开销大
• 频繁改变导致每帧重建
```

---

## 8.4 UGUI优化策略

### 1. 分离动态和静态Canvas

```csharp
// 不好：所有UI在一个Canvas
Canvas
├─ 背景图（静态）
├─ 按钮（静态）
├─ 血条（动态，每帧更新）
└─ 伤害数字（动态）

// 好：分离Canvas
Canvas_Static（静态UI）
├─ 背景图
└─ 按钮

Canvas_Dynamic（动态UI）
├─ 血条
└─ 伤害数字

// 动态Canvas改变不影响静态Canvas
```

### 2. 减少Raycast Target

```csharp
// 不需要交互的元素关闭Raycast
image.raycastTarget = false;
text.raycastTarget = false;

// 只有需要点击的元素开启
button.GetComponent<Image>().raycastTarget = true;
```

### 3. 避免空Image做点击区域

```csharp
// 不好：使用透明Image
Image clickArea = new Image();
clickArea.color = new Color(0, 0, 0, 0);  // 仍然参与渲染

// 好：使用自定义Raycast组件
public class EmptyRaycast : Graphic
{
    protected override void OnPopulateMesh(VertexHelper vh)
    {
        vh.Clear();  // 不生成任何网格
    }
}
```

### 4. 合理使用Layout组件

```csharp
// Layout组件会触发频繁重建
// 静态布局：设置好后禁用Layout组件
void Start()
{
    layoutGroup.enabled = true;
    LayoutRebuilder.ForceRebuildLayoutImmediate(rectTransform);
    layoutGroup.enabled = false;  // 禁用，避免后续重建
}

// 或者使用Canvas.ForceUpdateCanvases()后禁用
```

### 5. 对象池管理UI元素

```csharp
public class UIPool : MonoBehaviour
{
    private Queue<GameObject> pool = new Queue<GameObject>();
    public GameObject prefab;
    
    public GameObject Get()
    {
        if (pool.Count > 0)
        {
            var obj = pool.Dequeue();
            obj.SetActive(true);
            return obj;
        }
        return Instantiate(prefab, transform);
    }
    
    public void Return(GameObject obj)
    {
        obj.SetActive(false);
        pool.Enqueue(obj);
    }
}
```

### 6. 文本优化

```csharp
// 使用TextMeshPro代替Text
// TextMeshPro使用SDF渲染，更清晰，性能更好

// 避免频繁修改文本
// 不好
void Update()
{
    scoreText.text = "Score: " + score;  // 每帧字符串拼接
}

// 好
private int lastScore = -1;
void Update()
{
    if (score != lastScore)
    {
        lastScore = score;
        scoreText.text = $"Score: {score}";
    }
}
```

---

## 8.5 Image和RawImage

### 区别

| 特性 | Image | RawImage |
|------|-------|----------|
| 纹理类型 | Sprite | Texture |
| 图集支持 | 支持 | 不支持 |
| 合批 | 同图集可合批 | 每个单独DC |
| 九宫格 | 支持 | 不支持 |
| 填充模式 | 支持 | 不支持 |
| 适用场景 | 常规UI | 视频、RenderTexture |

### 使用建议

```csharp
// Image：常规UI图片
// 打入图集，可以合批
Image image = GetComponent<Image>();
image.sprite = atlas.GetSprite("icon");

// RawImage：特殊纹理
// 视频播放
RawImage rawImage = GetComponent<RawImage>();
rawImage.texture = videoPlayer.texture;

// RenderTexture（小地图、镜子）
rawImage.texture = renderTexture;

// 外部加载的图片
rawImage.texture = downloadedTexture;
```

---

## 8.6 事件系统

### EventSystem工作原理

```
事件处理流程：

1. 输入检测
   └─ Input Module检测输入（鼠标、触摸、键盘）

2. 射线检测
   └─ Raycaster发射射线，检测UI元素
   
3. 事件分发
   └─ EventSystem将事件分发给目标对象

4. 事件处理
   └─ 目标对象的事件处理器响应
```

### 事件接口

```csharp
public class MyButton : MonoBehaviour, 
    IPointerClickHandler,
    IPointerEnterHandler,
    IPointerExitHandler,
    IPointerDownHandler,
    IPointerUpHandler,
    IDragHandler,
    IBeginDragHandler,
    IEndDragHandler
{
    public void OnPointerClick(PointerEventData eventData)
    {
        Debug.Log("Clicked");
    }
    
    public void OnPointerEnter(PointerEventData eventData)
    {
        Debug.Log("Mouse Enter");
    }
    
    public void OnPointerExit(PointerEventData eventData)
    {
        Debug.Log("Mouse Exit");
    }
    
    public void OnPointerDown(PointerEventData eventData)
    {
        Debug.Log("Pointer Down");
    }
    
    public void OnPointerUp(PointerEventData eventData)
    {
        Debug.Log("Pointer Up");
    }
    
    public void OnBeginDrag(PointerEventData eventData)
    {
        Debug.Log("Begin Drag");
    }
    
    public void OnDrag(PointerEventData eventData)
    {
        Debug.Log($"Dragging: {eventData.delta}");
    }
    
    public void OnEndDrag(PointerEventData eventData)
    {
        Debug.Log("End Drag");
    }
}
```

### 事件穿透

```csharp
// 阻止事件穿透
public class BlockRaycast : MonoBehaviour, IPointerClickHandler
{
    public void OnPointerClick(PointerEventData eventData)
    {
        // 处理点击，不传递给下层
    }
}

// 允许事件穿透
public class PassThrough : MonoBehaviour, IPointerClickHandler
{
    public void OnPointerClick(PointerEventData eventData)
    {
        // 处理后传递给下层
        PassEvent(eventData, ExecuteEvents.pointerClickHandler);
    }
    
    void PassEvent<T>(PointerEventData data, ExecuteEvents.EventFunction<T> function)
        where T : IEventSystemHandler
    {
        var results = new List<RaycastResult>();
        EventSystem.current.RaycastAll(data, results);
        
        foreach (var result in results)
        {
            if (result.gameObject != gameObject)
            {
                ExecuteEvents.Execute(result.gameObject, data, function);
                break;
            }
        }
    }
}
```

---

## 8.7 ScrollView优化

### 问题：大量元素的ScrollView

```
问题场景：
┌─────────────────────────────────────┐
│ ScrollView（1000个Item）            │
│ ┌─────────────────────────────────┐ │
│ │ Item 1                          │ │ ← 可见
│ │ Item 2                          │ │ ← 可见
│ │ Item 3                          │ │ ← 可见
│ ├─────────────────────────────────┤ │
│ │ Item 4                          │ │ ← 不可见但存在
│ │ Item 5                          │ │
│ │ ...                             │ │
│ │ Item 1000                       │ │
│ └─────────────────────────────────┘ │
└─────────────────────────────────────┘

问题：
• 1000个GameObject
• 1000个UI组件
• 大量内存占用
• 初始化慢
```

### 解决方案：虚拟列表

```csharp
public class VirtualScrollView : MonoBehaviour
{
    public RectTransform content;
    public RectTransform viewport;
    public GameObject itemPrefab;
    public float itemHeight = 100f;
    
    private List<ItemData> dataList;
    private List<GameObject> itemPool = new List<GameObject>();
    private int visibleCount;
    private int startIndex;
    
    void Start()
    {
        // 计算可见数量（多创建几个用于缓冲）
        visibleCount = Mathf.CeilToInt(viewport.rect.height / itemHeight) + 2;
        
        // 只创建可见数量的Item
        for (int i = 0; i < visibleCount; i++)
        {
            var item = Instantiate(itemPrefab, content);
            itemPool.Add(item);
        }
    }
    
    public void SetData(List<ItemData> data)
    {
        dataList = data;
        
        // 设置Content高度
        content.sizeDelta = new Vector2(content.sizeDelta.x, data.Count * itemHeight);
        
        UpdateVisibleItems();
    }
    
    void OnScroll(Vector2 pos)
    {
        UpdateVisibleItems();
    }
    
    void UpdateVisibleItems()
    {
        // 计算当前应该显示的起始索引
        int newStartIndex = Mathf.FloorToInt(content.anchoredPosition.y / itemHeight);
        newStartIndex = Mathf.Max(0, newStartIndex);
        
        if (newStartIndex != startIndex)
        {
            startIndex = newStartIndex;
            
            // 更新每个Item的位置和数据
            for (int i = 0; i < itemPool.Count; i++)
            {
                int dataIndex = startIndex + i;
                if (dataIndex < dataList.Count)
                {
                    itemPool[i].SetActive(true);
                    itemPool[i].GetComponent<RectTransform>().anchoredPosition = 
                        new Vector2(0, -dataIndex * itemHeight);
                    UpdateItemData(itemPool[i], dataList[dataIndex]);
                }
                else
                {
                    itemPool[i].SetActive(false);
                }
            }
        }
    }
    
    void UpdateItemData(GameObject item, ItemData data)
    {
        // 更新Item显示的数据
        item.GetComponentInChildren<Text>().text = data.name;
    }
}
```

```
虚拟列表原理：
┌─────────────────────────────────────┐
│ ScrollView                          │
│ ┌─────────────────────────────────┐ │
│ │ Item A (显示Data[0])            │ │ ← 复用
│ │ Item B (显示Data[1])            │ │ ← 复用
│ │ Item C (显示Data[2])            │ │ ← 复用
│ │ Item D (显示Data[3])            │ │ ← 复用
│ └─────────────────────────────────┘ │
│                                     │
│ 滚动后：                            │
│ ┌─────────────────────────────────┐ │
│ │ Item A (显示Data[5])            │ │ ← 移动到底部
│ │ Item B (显示Data[2])            │ │
│ │ Item C (显示Data[3])            │ │
│ │ Item D (显示Data[4])            │ │
│ └─────────────────────────────────┘ │
└─────────────────────────────────────┘

只有4个Item对象，但可以显示1000条数据
```

---

## 8.8 Mask和RectMask2D

### Mask

```csharp
// Mask使用模板缓冲实现
// 需要Image组件

Mask特点：
• 使用Stencil Buffer
• 支持任意形状遮罩
• 打断合批（增加Draw Call）
• 被遮罩的元素仍然渲染（只是不显示）
```

### RectMask2D

```csharp
// RectMask2D使用裁剪实现
// 不需要Image组件

RectMask2D特点：
• 使用Shader裁剪
• 只支持矩形遮罩
• 不打断合批
• 被遮罩的元素不渲染（性能更好）
```

### 对比

| 特性 | Mask | RectMask2D |
|------|------|------------|
| 形状 | 任意 | 仅矩形 |
| 实现 | Stencil Buffer | Shader裁剪 |
| 合批 | 打断 | 不打断 |
| 性能 | 较差 | 较好 |
| 嵌套 | 支持（有限制） | 支持 |
| 适用 | 圆形、不规则遮罩 | 矩形裁剪、ScrollView |

---

## 8.9 锚点和轴心

### 锚点（Anchor）

```
锚点定义UI元素相对于父元素的位置参考点

┌─────────────────────────────────────┐
│ 父元素                              │
│                                     │
│  (0,1)────────(0.5,1)────────(1,1) │
│    │            │            │     │
│    │            │            │     │
│  (0,0.5)────(0.5,0.5)────(1,0.5)  │
│    │            │            │     │
│    │            │            │     │
│  (0,0)────────(0.5,0)────────(1,0) │
│                                     │
└─────────────────────────────────────┘

锚点类型：
• 单点锚点：anchorMin == anchorMax
  └─ 固定大小，相对锚点定位
  
• 拉伸锚点：anchorMin != anchorMax
  └─ 随父元素拉伸
```

### 轴心（Pivot）

```
轴心是UI元素自身的中心点

┌─────────────────┐
│                 │
│  (0,1)   (1,1)  │
│    ┌───────┐    │
│    │ Pivot │    │
│    │(0.5,0.5)   │
│    └───────┘    │
│  (0,0)   (1,0)  │
│                 │
└─────────────────┘

轴心影响：
• 旋转中心
• 缩放中心
• 位置参考点
```

### 代码设置

```csharp
RectTransform rect = GetComponent<RectTransform>();

// 设置锚点
rect.anchorMin = new Vector2(0, 0);
rect.anchorMax = new Vector2(1, 1);  // 拉伸填充父元素

// 设置轴心
rect.pivot = new Vector2(0.5f, 0.5f);  // 中心

// 设置位置（相对于锚点）
rect.anchoredPosition = new Vector2(100, 50);

// 设置大小
rect.sizeDelta = new Vector2(200, 100);

// 设置边距（拉伸模式下）
rect.offsetMin = new Vector2(10, 10);  // 左下边距
rect.offsetMax = new Vector2(-10, -10); // 右上边距（负值）
```

---

## 面试要点总结

### 问题35：UGUI的渲染原理是什么？

**答案要点：**
1. Canvas收集所有UI元素
2. 按深度排序，合并相同材质
3. 生成网格数据（Mesh）
4. 提交给GPU渲染

### 问题36：Canvas的三种渲染模式有什么区别？

**答案要点：**
- **Overlay**：覆盖在所有3D物体上，性能最好
- **Camera**：由指定相机渲染，可与3D穿插
- **World**：作为3D物体存在，有透视效果

### 问题37：什么是Canvas重建？如何优化？

**答案要点：**
1. **重建类型**：Layout Rebuild、Graphic Rebuild、Rebatch
2. **优化方法**：
   - 分离动态和静态Canvas
   - 减少不必要的Raycast Target
   - 避免频繁修改UI属性
   - 使用对象池

### 问题38：Image和RawImage的区别是什么？

**答案要点：**
- **Image**：使用Sprite，支持图集合批，支持九宫格
- **RawImage**：使用Texture，不支持合批，用于视频/RenderTexture

### 问题39：UGUI的事件系统是如何工作的？

**答案要点：**
1. Input Module检测输入
2. Raycaster发射射线检测UI
3. EventSystem分发事件
4. 实现IPointerXXXHandler接口处理事件

### 问题40：如何优化UGUI的性能？

**答案要点：**
1. 分离动态/静态Canvas
2. 关闭不需要的Raycast Target
3. 使用对象池
4. 使用TextMeshPro
5. 合理使用图集

### 问题41：ScrollView的优化方案有哪些？

**答案要点：**
- 使用虚拟列表（只创建可见数量的Item）
- 复用Item对象
- 使用RectMask2D代替Mask

### 问题42：Mask和RectMask2D的区别是什么？

**答案要点：**
- **Mask**：Stencil实现，支持任意形状，打断合批
- **RectMask2D**：Shader裁剪，仅矩形，不打断合批，性能更好

### 问题43：如何实现UI的层级管理？

**答案要点：**
1. 使用多个Canvas，设置不同sortingOrder
2. 使用Canvas.overrideSorting
3. 调整Hierarchy中的顺序
4. 使用Transform.SetSiblingIndex()

### 问题44：UGUI的锚点和轴心是什么？

**答案要点：**
- **锚点**：定义相对于父元素的位置参考点
- **轴心**：定义自身的中心点，影响旋转和缩放

---

[返回目录](./00_目录.md) | [上一章：C#基础](./07_CSharp基础.md) | [下一章：资源管理](./09_资源管理.md)