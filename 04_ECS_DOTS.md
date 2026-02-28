# 四、ECS/DOTS

> 本章涵盖问题：
> - ECS是什么？有什么优势？
> - DOTS包含哪些技术？
> - Job System和Burst Compiler是什么？

---

## 4.1 传统OOP vs ECS

### 传统面向对象（OOP）

```csharp
// 传统Unity组件方式
public class Enemy : MonoBehaviour
{
    public float health;
    public float speed;
    public float damage;
    
    void Update()
    {
        // 移动逻辑
        transform.position += transform.forward * speed * Time.deltaTime;
    }
    
    public void TakeDamage(float amount)
    {
        health -= amount;
        if (health <= 0) Die();
    }
}
```

**OOP的问题：**

| 问题 | 说明 |
|------|------|
| 缓存不友好 | 数据分散在内存各处，缓存命中率低 |
| 继承复杂 | 深层继承导致代码难以维护 |
| 难以并行 | 对象间依赖复杂，难以多线程 |
| 性能瓶颈 | 大量对象时Update调用开销大 |

### ECS架构

**ECS = Entity + Component + System**

```
┌─────────────────────────────────────────────────────────────┐
│                         ECS 架构                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Entity（实体）                                              │
│  ├─ 只是一个ID（整数）                                       │
│  ├─ 没有数据，没有行为                                       │
│  └─ 是Component的容器                                        │
│                                                             │
│  Component（组件）                                           │
│  ├─ 只有数据，没有行为                                       │
│  ├─ 纯数据结构（struct）                                     │
│  └─ 连续内存存储                                             │
│                                                             │
│  System（系统）                                              │
│  ├─ 只有行为，没有数据                                       │
│  ├─ 处理特定Component组合的Entity                            │
│  └─ 可以并行执行                                             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 4.2 ECS代码示例

### 定义Component

```csharp
using Unity.Entities;
using Unity.Mathematics;

// Component只是数据结构
public struct Position : IComponentData
{
    public float3 Value;
}

public struct Velocity : IComponentData
{
    public float3 Value;
}

public struct Health : IComponentData
{
    public float Value;
    public float MaxValue;
}

public struct Enemy : IComponentData
{
    // 标签组件，没有数据，只用于标识
}
```

### 定义System

```csharp
using Unity.Entities;
using Unity.Transforms;
using Unity.Burst;

// 移动系统：处理所有有Position和Velocity的Entity
[BurstCompile]
public partial struct MovementSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        float deltaTime = SystemAPI.Time.DeltaTime;
        
        // 遍历所有有Position和Velocity的Entity
        foreach (var (position, velocity) in 
            SystemAPI.Query<RefRW<Position>, RefRO<Velocity>>())
        {
            position.ValueRW.Value += velocity.ValueRO.Value * deltaTime;
        }
    }
}

// 使用Job并行处理
[BurstCompile]
public partial struct MovementSystemParallel : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        var job = new MovementJob
        {
            DeltaTime = SystemAPI.Time.DeltaTime
        };
        job.ScheduleParallel();
    }
}

[BurstCompile]
public partial struct MovementJob : IJobEntity
{
    public float DeltaTime;
    
    void Execute(ref Position position, in Velocity velocity)
    {
        position.Value += velocity.Value * DeltaTime;
    }
}
```

### 创建Entity

```csharp
using Unity.Entities;

public partial class SpawnSystem : SystemBase
{
    protected override void OnUpdate()
    {
        if (Input.GetKeyDown(KeyCode.Space))
        {
            var entityManager = World.DefaultGameObjectInjectionWorld.EntityManager;
            
            // 创建Entity
            Entity entity = entityManager.CreateEntity();
            
            // 添加Component
            entityManager.AddComponentData(entity, new Position { Value = float3.zero });
            entityManager.AddComponentData(entity, new Velocity { Value = new float3(1, 0, 0) });
            entityManager.AddComponentData(entity, new Enemy());
        }
    }
}
```

---

## 4.3 ECS内存布局

### OOP内存布局（AoS - Array of Structures）

```
传统方式：对象分散在堆内存中

内存地址:  0x1000        0x2000        0x3000        0x4000
          ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
          │ Enemy1   │  │ Enemy2   │  │ Enemy3   │  │ Enemy4   │
          │ ──────── │  │ ──────── │  │ ──────── │  │ ──────── │
          │ health   │  │ health   │  │ health   │  │ health   │
          │ speed    │  │ speed    │  │ speed    │  │ speed    │
          │ damage   │  │ damage   │  │ damage   │  │ damage   │
          │ position │  │ position │  │ position │  │ position │
          │ ...      │  │ ...      │  │ ...      │  │ ...      │
          └──────────┘  └──────────┘  └──────────┘  └──────────┘

问题：
- 内存不连续，缓存命中率低
- 遍历时需要跳跃访问
- 每次访问可能导致缓存未命中
```

### ECS内存布局（SoA - Structure of Arrays）

```
ECS方式：相同Component连续存储

Chunk（16KB内存块）:
┌─────────────────────────────────────────────────────────────┐
│ Archetype: [Position, Velocity, Enemy]                      │
├─────────────────────────────────────────────────────────────┤
│ Position数组:  [P1][P2][P3][P4][P5][P6][P7][P8]...          │
│ Velocity数组:  [V1][V2][V3][V4][V5][V6][V7][V8]...          │
│ Enemy数组:     [E1][E2][E3][E4][E5][E6][E7][E8]...          │
└─────────────────────────────────────────────────────────────┘

优势：
- 相同类型数据连续存储
- 缓存友好，预取有效
- 遍历时顺序访问
- SIMD指令可以并行处理
```

### Archetype（原型）

```csharp
// 相同Component组合的Entity属于同一个Archetype
// Archetype决定了Entity存储在哪个Chunk

Archetype A: [Position, Velocity]
┌─────────────────────────────────────┐
│ Chunk 1: Entity 1-100              │
│ Chunk 2: Entity 101-200            │
└─────────────────────────────────────┘

Archetype B: [Position, Velocity, Health]
┌─────────────────────────────────────┐
│ Chunk 3: Entity 201-280            │
└─────────────────────────────────────┘

Archetype C: [Position, Rotation]
┌─────────────────────────────────────┐
│ Chunk 4: Entity 281-400            │
└─────────────────────────────────────┘
```

---

## 4.4 DOTS技术栈

**DOTS = Data-Oriented Technology Stack**

| 技术 | 作用 |
|------|------|
| **Entities** | ECS框架，管理Entity和Component |
| **Jobs** | Job System，多线程任务调度 |
| **Burst** | Burst Compiler，高性能编译器 |
| **Collections** | 原生容器（NativeArray等） |
| **Mathematics** | 数学库（float3, quaternion等） |

### Job System

```csharp
using Unity.Jobs;
using Unity.Collections;
using Unity.Burst;

// 定义Job
[BurstCompile]
public struct MyJob : IJob
{
    public NativeArray<float> data;
    public float multiplier;
    
    public void Execute()
    {
        for (int i = 0; i < data.Length; i++)
        {
            data[i] *= multiplier;
        }
    }
}

// 并行Job（每个元素独立处理）
[BurstCompile]
public struct MyParallelJob : IJobParallelFor
{
    public NativeArray<float> data;
    public float multiplier;
    
    public void Execute(int index)
    {
        data[index] *= multiplier;
    }
}

// 使用Job
public class JobExample : MonoBehaviour
{
    void Update()
    {
        // 创建原生数组
        NativeArray<float> data = new NativeArray<float>(10000, Allocator.TempJob);
        
        // 初始化数据
        for (int i = 0; i < data.Length; i++)
            data[i] = i;
        
        // 创建并调度Job
        var job = new MyParallelJob
        {
            data = data,
            multiplier = 2f
        };
        
        // 调度并行Job，innerloopBatchCount是每批处理的数量
        JobHandle handle = job.Schedule(data.Length, 64);
        
        // 等待完成（或者在后续帧处理）
        handle.Complete();
        
        // 使用结果...
        
        // 释放内存
        data.Dispose();
    }
}
```

### Job依赖

```csharp
// Job可以形成依赖链
JobHandle handle1 = job1.Schedule();
JobHandle handle2 = job2.Schedule(handle1);  // 依赖handle1
JobHandle handle3 = job3.Schedule(handle2);  // 依赖handle2

// 合并多个依赖
JobHandle combined = JobHandle.CombineDependencies(handle1, handle2);
JobHandle handle4 = job4.Schedule(combined);
```

### Burst Compiler

```csharp
using Unity.Burst;

// Burst编译的Job
[BurstCompile]
public struct BurstJob : IJob
{
    public NativeArray<float> input;
    public NativeArray<float> output;
    
    public void Execute()
    {
        for (int i = 0; i < input.Length; i++)
        {
            // Burst会优化这个循环
            // - 自动向量化（SIMD）
            // - 循环展开
            // - 消除边界检查
            output[i] = math.sqrt(input[i]) * math.sin(input[i]);
        }
    }
}
```

**Burst优化效果：**

| 操作 | 普通C# | Burst编译 | 提升 |
|------|--------|-----------|------|
| 向量运算 | 100ms | 5ms | 20x |
| 数学计算 | 50ms | 2ms | 25x |
| 数组遍历 | 30ms | 3ms | 10x |

**Burst限制：**
- 不能使用托管类型（class、string）
- 不能分配托管内存
- 不能调用托管方法
- 只能使用值类型和NativeContainer

---

## 4.5 ECS的优势

### 1. 性能优势

```
传统方式处理10000个敌人：
┌─────────────────────────────────────���
│ 10000个Update调用                   │
│ 10000次虚函数调用                   │
│ 缓存命中率低                        │
│ 单线程执行                          │
│ 结果：~15ms                         │
└─────────────────────────────────────┘

ECS方式处理10000个敌人：
┌─────────────────────────────────────┐
│ 1个System处理所有Entity             │
│ 连续内存访问                        │
│ 缓存命中率高                        │
│ 多线程并行 + Burst优化              │
│ 结果：~0.5ms                        │
└─────────────────────────────────────┘
```

### 2. 代码组织优势

```csharp
// 传统方式：功能分散在各个类中
class Player : Character { ... }
class Enemy : Character { ... }
class NPC : Character { ... }
// 如果要添加新功能，需要修改继承链

// ECS方式：功能通过Component组合
Entity player = CreateEntity(Position, Velocity, Health, PlayerInput);
Entity enemy = CreateEntity(Position, Velocity, Health, AI);
Entity npc = CreateEntity(Position, Velocity, Health, Dialogue);
// 添加新功能只需添加新Component和System
```

### 3. 可测试性

```csharp
// System是纯函数，易于测试
[Test]
public void MovementSystem_UpdatesPosition()
{
    // Arrange
    var world = new World("Test");
    var entity = world.EntityManager.CreateEntity();
    world.EntityManager.AddComponentData(entity, new Position { Value = float3.zero });
    world.EntityManager.AddComponentData(entity, new Velocity { Value = new float3(1, 0, 0) });
    
    // Act
    var system = world.CreateSystem<MovementSystem>();
    system.Update();
    
    // Assert
    var position = world.EntityManager.GetComponentData<Position>(entity);
    Assert.AreEqual(new float3(1, 0, 0) * deltaTime, position.Value);
}
```

---

## 4.6 ECS vs 传统方式选择

| 场景 | 推荐方式 |
|------|----------|
| 大量相似对象（子弹、粒子、敌人群） | ECS |
| 复杂UI逻辑 | 传统OOP |
| 物理模拟 | ECS |
| 单个复杂角色 | 传统OOP或混合 |
| 需要快速迭代原型 | 传统OOP |
| 性能关键的系统 | ECS + Burst |

---

## 面试要点总结

### 问题18：ECS是什么？有什么优势？

**答案要点：**

1. **ECS定义**：
   - Entity：只是ID，Component的容器
   - Component：只有数据，没有行为
   - System：只有行为，处理特定Component组合

2. **优势**：
   - **缓存友好**：相同Component连续存储，缓存命中率高
   - **易于并行**：System之间独立，可以多线程
   - **组合优于继承**：通过Component组合功能，避免继承复杂性
   - **数据与逻辑分离**：易于测试和维护

### 问题19：DOTS包含哪些技术？

**答案要点：**

1. **Entities**：ECS框架
2. **Jobs**：多线程任务调度系统
3. **Burst**：高性能编译器，生成优化的原生代码
4. **Collections**：原生容器（NativeArray等）
5. **Mathematics**：高性能数学库

### 问题20：Job System和Burst Compiler是什么？

**答案要点：**

1. **Job System**：
   - Unity的多线程任务调度系统
   - 通过IJob、IJobParallelFor等接口定义任务
   - 自动处理线程调度和依赖关系
   - 使用NativeContainer在Job间传递数据

2. **Burst Compiler**：
   - 将C#代码编译为高度优化的原生代码
   - 自动向量化（SIMD）
   - 消除边界检查
   - 限制：只能使用值类型和NativeContainer

---

[返回目录](./00_目录.md) | [上一章：协程](./03_协程.md) | [下一章：物理与Update](./05_物理与Update.md)