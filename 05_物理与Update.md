# 五、物理与Update

> 本章涵盖问题：
> - FixedUpdate和Update的区别是什么？
> - 物理更新为什么要放在FixedUpdate中？

---

## 5.1 Unity生命周期

### 完整执行顺序

```
┌─────────────────────────────────────────────────────────────┐
│                    Unity 帧循环                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  初始化阶段（仅执行一次）                                    │
│  ├─ Awake()                                                 │
│  ├─ OnEnable()                                              │
│  └─ Start()                                                 │
│                                                             │
│  ════════════════════════════════════════════════════════  │
│                                                             │
│  每帧循环：                                                  │
│  │                                                          │
│  ├─► 物理循环（可能执行0次、1次或多次）                      │
│  │   ├─ FixedUpdate()                                       │
│  │   ├─ 内部物理更新                                        │
│  │   │   ├─ 碰撞检测                                        │
│  │   │   ├─ 刚体模拟                                        │
│  │   │   └─ 触发器检测                                      │
│  │   ├─ OnTriggerXXX()                                      │
│  │   └─ OnCollisionXXX()                                    │
│  │                                                          │
│  ├─► 输入事件                                               │
│  │                                                          │
│  ├─► Update()                                               │
│  │                                                          │
│  ├─► 动画更新                                               │
│  │                                                          │
│  ├─► LateUpdate()                                           │
│  │                                                          │
│  ├─► 渲染                                                   │
│  │   ├─ OnWillRenderObject()                                │
│  │   ├─ OnPreRender()                                       │
│  │   ├─ OnRenderObject()                                    │
│  │   ├─ OnPostRender()                                      │
│  │   └─ OnRenderImage()                                     │
│  │                                                          │
│  └─► OnGUI()（可能多次）                                    │
│                                                             │
│  ════════════════════════════════════════════════════════  │
│                                                             │
│  销毁阶段                                                    │
│  ├─ OnDisable()                                             │
│  └─ OnDestroy()                                             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 5.2 Update vs FixedUpdate

### 核心区别

| 特性 | Update | FixedUpdate |
|------|--------|-------------|
| **调用频率** | 每帧一次 | 固定时间间隔 |
| **时间间隔** | Time.deltaTime（不固定） | Time.fixedDeltaTime（固定，默认0.02s） |
| **帧率影响** | 受帧率影响 | 不受帧率影响 |
| **每帧调用次数** | 恰好1次 | 0次、1次或多次 |
| **适用场景** | 输入、渲染相关 | 物理计算 |

### 时间线对比

```
假设：fixedDeltaTime = 20ms，帧率不稳定

时间轴(ms):  0    20    40    60    80    100   120   140
            │     │     │     │     │     │     │     │
FixedUpdate:├──F──├──F──├──F──├──F──├──F──├──F──├──F──┤
            │     │     │     │     │     │     │     │

帧渲染:     ├─────────U─────────────U───────U─────────U─┤
            │    30ms  │    50ms    │ 20ms  │   40ms   │
            │  (帧1)   │   (帧2)    │(帧3)  │  (帧4)   │

帧1 (0-30ms):   FixedUpdate调用2次 (0ms, 20ms)，然后Update
帧2 (30-80ms):  FixedUpdate调用2次 (40ms, 60ms)，然后Update  
帧3 (80-100ms): FixedUpdate调用1次 (80ms)，然后Update
帧4 (100-140ms):FixedUpdate调用2次 (100ms, 120ms)，然后Update
```

### 代码验证

```csharp
public class UpdateTest : MonoBehaviour
{
    private int fixedUpdateCount = 0;
    private int updateCount = 0;
    
    void FixedUpdate()
    {
        fixedUpdateCount++;
        Debug.Log($"FixedUpdate #{fixedUpdateCount}, Time: {Time.time:F3}");
    }
    
    void Update()
    {
        updateCount++;
        Debug.Log($"Update #{updateCount}, DeltaTime: {Time.deltaTime:F3}");
    }
}

// 输出示例（帧率不稳定时）：
// FixedUpdate #1, Time: 0.020
// FixedUpdate #2, Time: 0.040
// Update #1, DeltaTime: 0.045
// FixedUpdate #3, Time: 0.060
// FixedUpdate #4, Time: 0.080
// Update #2, DeltaTime: 0.038
// ...
```

---

## 5.3 为什么物理要用FixedUpdate

### 问题：Update中的物理不稳定

```csharp
// 错误示例：在Update中移动刚体
void Update()
{
    // deltaTime每帧不同，导致物理行为不一致
    rb.velocity = direction * speed;
    
    // 帧率高时：小步长，碰撞检测精确
    // 帧率低时：大步长，可能穿透物体
}
```

```
帧率60fps时：
物体位置: ●──●──●──●──●──●──●──●──●──●──●──●
          每步移动小，碰撞检测精确

帧率15fps时：
物体位置: ●────────●────────●────────●────────●
          每步移动大，可能穿过薄墙
          
          ┌─────┐
          │ 墙  │
    ●─────┼─────┼─────●  ← 穿透！
          │     │
          └─────┘
```

### 解决：FixedUpdate保证一致性

```csharp
// 正确示例：在FixedUpdate中处理物理
void FixedUpdate()
{
    // fixedDeltaTime固定，物理行为一致
    rb.AddForce(direction * force);
    
    // 无论帧率如何，物理模拟步长相同
    // 碰撞检测结果一致
}
```

```
无论帧率如何，物理步长固定：
物体位置: ●──●──●──●──●──●──●──●──●──●──●──●
          每步移动相同，碰撞检测一致
```

### 物理引擎工作原理

```csharp
// Unity内部物理更新（伪代码）
void PhysicsLoop()
{
    // 累积时间
    physicsAccumulator += Time.deltaTime;
    
    // 以固定步长执行物理
    while (physicsAccumulator >= fixedDeltaTime)
    {
        // 1. 调用所有FixedUpdate
        CallFixedUpdate();
        
        // 2. 物理模拟
        PhysicsSimulate(fixedDeltaTime);
        
        // 3. 碰撞检测和回调
        DetectCollisions();
        CallCollisionCallbacks();
        
        physicsAccumulator -= fixedDeltaTime;
    }
}
```

---

## 5.4 输入处理的位置

### 为什么输入要在Update中？

```csharp
// 错误：在FixedUpdate中检测输入
void FixedUpdate()
{
    if (Input.GetKeyDown(KeyCode.Space))  // 可能漏掉！
    {
        Jump();
    }
}
```

**问题分析：**

```
假设玩家在第45ms按下空格键

时间轴:     0    20    40    60    80
            │     │     │     │     │
FixedUpdate:├──F──├──F──├──F──├──F──┤
                        ↑
                    检测时机(40ms)
                    
玩家按键:              ↓ (45ms)
                       ●
                       
下次FixedUpdate:            ↑ (60ms)
                            此时GetKeyDown已经是false了！
                            
结果：按键被漏掉
```

### 正确做法

```csharp
public class PlayerController : MonoBehaviour
{
    private bool jumpRequested = false;
    
    void Update()
    {
        // 在Update中捕获输入
        if (Input.GetKeyDown(KeyCode.Space))
        {
            jumpRequested = true;
        }
    }
    
    void FixedUpdate()
    {
        // 在FixedUpdate中执行物理操作
        if (jumpRequested)
        {
            rb.AddForce(Vector3.up * jumpForce, ForceMode.Impulse);
            jumpRequested = false;
        }
    }
}
```

---

## 5.5 移动物体的正确方式

### 非物理移动（Update）

```csharp
// 适用于：不需要物理交互的物体（UI、特效、相机）
void Update()
{
    // 方式1：直接修改Transform
    transform.position += direction * speed * Time.deltaTime;
    
    // 方式2：使用Translate
    transform.Translate(direction * speed * Time.deltaTime);
}
```

### 物理移动（FixedUpdate）

```csharp
// 适用于：需要物理交互的物体
void FixedUpdate()
{
    // 方式1：设置速度（适合持续移动）
    rb.velocity = new Vector3(horizontal * speed, rb.velocity.y, vertical * speed);
    
    // 方式2：添加力（适合加速效果）
    rb.AddForce(direction * force);
    
    // 方式3：MovePosition（运动学刚体）
    rb.MovePosition(rb.position + direction * speed * Time.fixedDeltaTime);
}
```

### 各种移动方式对比

| 方式 | 位置 | 物理交互 | 适用场景 |
|------|------|----------|----------|
| `transform.position` | Update | 无 | UI、特效 |
| `rb.velocity` | FixedUpdate | 有 | 角色移动 |
| `rb.AddForce` | FixedUpdate | 有 | 物理推动 |
| `rb.MovePosition` | FixedUpdate | 有 | 运动学物体 |
| `CharacterController.Move` | Update | 部分 | 角色控制器 |

---

## 5.6 插值与平滑

### 问题：物理更新和渲染不同步

```
物理更新(50Hz):  ●─────●─────●─────●─────●
渲染(60Hz):      ○──○──○──○──○──○──○──○──○──○──○──○

渲染时物体位置可能在两个物理位置之间
如果直接使用物理位置，会出现抖动
```

### 解决：Rigidbody插值

```csharp
// 在Inspector中设置，或代码设置
rb.interpolation = RigidbodyInterpolation.Interpolate;
```

| 插值模式 | 说明 |
|----------|------|
| None | 不插值，可能抖动 |
| Interpolate | 基于上一帧位置插值，有轻微延迟 |
| Extrapolate | 基于速度预测，可能不准确 |

### 手动插值示例

```csharp
public class SmoothFollow : MonoBehaviour
{
    public Transform target;
    public float smoothTime = 0.3f;
    
    private Vector3 velocity = Vector3.zero;
    
    void LateUpdate()
    {
        // 使用SmoothDamp平滑跟随
        transform.position = Vector3.SmoothDamp(
            transform.position,
            target.position,
            ref velocity,
            smoothTime
        );
    }
}
```

---

## 5.7 Time类详解

### 常用时间属性

| 属性 | 说明 |
|------|------|
| `Time.time` | 游戏开始后的时间（受timeScale影响） |
| `Time.unscaledTime` | 游戏开始后的时间（不受timeScale影响） |
| `Time.deltaTime` | 上一帧到这一帧的时间 |
| `Time.unscaledDeltaTime` | 不受timeScale影响的deltaTime |
| `Time.fixedDeltaTime` | 固定更新间隔（默认0.02s） |
| `Time.timeScale` | 时间缩放（0=暂停，1=正常，2=两倍速） |
| `Time.frameCount` | 已渲染的帧数 |
| `Time.realtimeSinceStartup` | 真实时间（不受暂停影响） |

### timeScale的影响

```csharp
// 暂停游戏
Time.timeScale = 0f;

// 此时：
// - Update仍然调用，但Time.deltaTime = 0
// - FixedUpdate不调用
// - 物理暂停
// - 动画暂停（除非使用unscaledTime）

// 暂停时仍需响应的逻辑
void Update()
{
    // 使用unscaledDeltaTime
    if (Time.timeScale == 0)
    {
        // 暂停菜单动画
        pauseMenuAlpha += Time.unscaledDeltaTime;
    }
}
```

---

## 面试要点总结

### 问题21：FixedUpdate和Update的区别是什么？

**答案要点：**

1. **调用频率**：
   - Update：每帧调用一次，频率随帧率变化
   - FixedUpdate：固定时间间隔调用（默认0.02s，即50Hz）

2. **时间间隔**：
   - Update：Time.deltaTime，每帧不同
   - FixedUpdate：Time.fixedDeltaTime，固定值

3. **每帧调用次数**：
   - Update：恰好1次
   - FixedUpdate：可能0次、1次或多次（取决于帧率）

4. **适用场景**：
   - Update：输入检测、非物理移动、渲染相关
   - FixedUpdate：物理计算、刚体操作

### 问题22：物理更新为什么要放在FixedUpdate中？

**答案要点：**

1. **一致性**：固定步长保证物理模拟结果一致，不受帧率影响

2. **碰撞检测**：
   - 固定步长避免高速物体穿透
   - 帧率低时Update步长大，可能错过碰撞

3. **物理引擎设计**：
   - Unity物理引擎在FixedUpdate后执行
   - 物理模拟需要固定时间步长才能稳定

4. **输入处理**：
   - 输入应在Update中检测（避免漏掉）
   - 用标志位传递给FixedUpdate执行物理操作

---

[返回目录](./00_目录.md) | [上一章：ECS/DOTS](./04_ECS_DOTS.md) | [下一章：数据结构](./06_数据结构.md)