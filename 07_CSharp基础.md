# 七、C#基础

> 本章涵盖问题：
> - 值类型和引用类型的区别是什么？
> - 装箱和拆箱是什么？有什么性能影响？
> - struct和class的区别是什么？
> - 接口和抽象类的区别是什么？
> - 委托和事件的区别是什么？
> - ref、out、in关键字的区别是什么？
> - 什么是闭包？有什么注意事项？

---

## 7.1 值类型和引用类型

### 基本分类

| 值类型 | 引用类型 |
|--------|----------|
| int, float, double, bool | class |
| char, byte, short, long | string（特殊） |
| struct | 数组 |
| enum | interface |
| decimal | delegate |

### 内存分配

```csharp
// 值类型
int a = 10;
int b = a;  // 复制值
b = 20;     // a仍然是10

// 引用类型
class Player { public int hp; }
Player p1 = new Player { hp = 100 };
Player p2 = p1;  // 复制引用
p2.hp = 50;      // p1.hp也变成50
```

```
值类型（栈上分配）：
┌─────────────────────────────────────┐
│              栈内存                  │
├─────────────────────────────────────┤
│  a: │  10  │                        │
│  b: │  20  │  ← 独��的副本          │
└─────────────────────────────────────┘

引用类型（堆上分配）：
┌─────────────────────────────────────┐
│              栈内存                  │
├─────────────────────────────────────┤
│  p1: │ 0x1000 │ ──┐                 │
│  p2: │ 0x1000 │ ──┼─► 指向同一对象  │
└─────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────┐
│              堆内存                  │
├─────────────────────────────────────┤
│  0x1000: │ Player { hp = 50 } │     │
└─────────────────────────────────────┘
```

### 参数传递

```csharp
void ModifyValue(int x)
{
    x = 100;  // 不影响原值
}

void ModifyReference(Player p)
{
    p.hp = 100;  // 影响原对象
}

void ReplaceReference(Player p)
{
    p = new Player();  // 不影响原引用
}

// 测试
int num = 10;
ModifyValue(num);  // num仍然是10

Player player = new Player { hp = 50 };
ModifyReference(player);  // player.hp变成100
ReplaceReference(player); // player仍指向原对象
```

---

## 7.2 装箱和拆箱

### 概念

```csharp
// 装箱：值类型 → 引用类型
int i = 10;
object obj = i;  // 装箱

// 拆箱：引用类型 → 值类型
int j = (int)obj;  // 拆箱
```

### 装箱过程

```
装箱 int i = 10; object obj = i;

1. 在堆上分配内存（对象头 + 值）
2. 将值复制到堆上
3. 返回堆上对象的引用

栈:                    堆:
┌─────────┐           ┌─────────────────┐
│ i = 10  │           │ 对象头          │
├─────────┤           ├─────────────────┤
│ obj ────┼──────────►│ 值 = 10         │
└─────────┘           └─────────────────┘
```

### 性能影响

```csharp
// 错误示例：频繁装箱
void BadExample()
{
    ArrayList list = new ArrayList();  // 非泛型集合
    for (int i = 0; i < 10000; i++)
    {
        list.Add(i);  // 每次Add都装箱！
    }
}

// 正确示例：使用泛型避免装箱
void GoodExample()
{
    List<int> list = new List<int>();  // 泛型集合
    for (int i = 0; i < 10000; i++)
    {
        list.Add(i);  // 无装箱
    }
}
```

### 常见装箱场景

```csharp
// 1. 非泛型集合
ArrayList list = new ArrayList();
list.Add(10);  // 装箱

// 2. string.Format
string s = string.Format("{0}", 10);  // 装箱

// 3. 字符串拼接
string s = "Value: " + 10;  // 装箱

// 4. 调用object方法
int i = 10;
i.GetType();  // 装箱（GetType是object的方法）

// 5. 接口调用（值类型实现接口）
interface IValue { int Value { get; } }
struct MyStruct : IValue { public int Value => 10; }
IValue v = new MyStruct();  // 装箱
```

### 避免装箱的方法

```csharp
// 1. 使用泛型
List<int> list = new List<int>();

// 2. 使用字符串插值（编译器优化）
string s = $"Value: {10}";  // 某些情况下避免装箱

// 3. 使用ToString()
string s = "Value: " + 10.ToString();  // 避免装箱

// 4. 使用Span<T>和stackalloc
Span<int> span = stackalloc int[10];

// 5. 使用泛型约束
void Process<T>(T value) where T : struct { }
```

---

## 7.3 struct和class

### 核心区别

| 特性 | struct | class |
|------|--------|-------|
| 类型 | 值类型 | 引用类型 |
| 存储位置 | 栈（通常） | 堆 |
| 默认构造函数 | 不能自定义无参构造（C# 10前） | 可以 |
| 继承 | 不支持（隐式继承ValueType） | 支持 |
| 默认值 | 所有字段为默认值 | null |
| 赋值行为 | 复制整个值 | 复制引用 |
| GC压力 | 无（栈上） | 有（堆上） |

### 使用场景

```csharp
// 适合用struct的情况：
// 1. 小型数据（建议≤16字节）
// 2. 不可变数据
// 3. 频繁创建销毁
// 4. 不需要继承

// Unity中的struct示例
public struct Vector3
{
    public float x, y, z;  // 12字节
}

public struct Color
{
    public float r, g, b, a;  // 16字节
}

// 适合用class的情况：
// 1. 大型对象
// 2. 需要继承
// 3. 需要多态
// 4. 需要null表示"无"

public class Player
{
    public string Name;
    public int Level;
    public List<Item> Inventory;
}
```

### struct的陷阱

```csharp
// 陷阱1：在foreach中修改struct
struct Enemy { public int hp; }
List<Enemy> enemies = new List<Enemy>();

foreach (var enemy in enemies)
{
    enemy.hp -= 10;  // 编译错误！foreach的迭代变量是只读的
}

// 陷阱2：struct作为属性返回
class Game
{
    private Enemy _enemy;
    public Enemy Enemy => _enemy;  // 返回副本
}

game.Enemy.hp = 100;  // 修改的是副本，原值不变！

// 陷阱3：struct装箱
interface IDamageable { void TakeDamage(int amount); }
struct Enemy : IDamageable { ... }

IDamageable target = new Enemy();  // 装箱！
```

### readonly struct

```csharp
// C# 7.2+：不可变struct
public readonly struct Point
{
    public readonly float X;
    public readonly float Y;
    
    public Point(float x, float y)
    {
        X = x;
        Y = y;
    }
    
    // 所有方法隐式readonly
    public float Distance() => MathF.Sqrt(X * X + Y * Y);
}

// 优势：
// 1. 编译器可以优化，避免防御性复制
// 2. 明确表达不可变意图
// 3. 线程安全
```

---

## 7.4 接口和抽象类

### 核心区别

| 特性 | 接口 | 抽象类 |
|------|------|--------|
| 多继承 | 支持 | 不支持 |
| 字段 | 不能有（C# 8前） | 可以有 |
| 构造函数 | 不能有 | 可以有 |
| 访问修饰符 | 默认public | 可以任意 |
| 实现 | 不能有（C# 8前可有默认实现） | 可以有 |
| 设计意图 | "能做什么" | "是什么" |

### 代码示例

```csharp
// 接口：定义能力
public interface IDamageable
{
    void TakeDamage(int amount);
    int CurrentHealth { get; }
}

public interface IMovable
{
    void Move(Vector3 direction);
}

// 抽象类：定义基础实现
public abstract class Character
{
    protected int health;
    protected float speed;
    
    public Character(int health, float speed)
    {
        this.health = health;
        this.speed = speed;
    }
    
    // 抽象方法：子类必须实现
    public abstract void Attack();
    
    // 虚方法：子类可以重写
    public virtual void Die()
    {
        Debug.Log("Character died");
    }
    
    // 普通方法：子类继承
    public void Heal(int amount)
    {
        health += amount;
    }
}

// 组合使用
public class Player : Character, IDamageable, IMovable
{
    public Player() : base(100, 5f) { }
    
    public int CurrentHealth => health;
    
    public override void Attack()
    {
        Debug.Log("Player attacks");
    }
    
    public void TakeDamage(int amount)
    {
        health -= amount;
        if (health <= 0) Die();
    }
    
    public void Move(Vector3 direction)
    {
        transform.position += direction * speed;
    }
}
```

### 选择指南

```
使用接口：
├─ 需要多继承
├─ 定义跨类型的共同能力
├─ 需要解耦（依赖注入）
└─ 不同类层次结构共享行为

使用抽象类：
├─ 有共享的实现代码
├─ 需要非public成员
├─ 需要构造函数逻辑
└─ 定义类型层次结构
```

---

## 7.5 委托和事件

### 委托（Delegate）

```csharp
// 定义委托类型
public delegate void DamageHandler(int amount);

// 使用委托
public class Weapon
{
    public DamageHandler OnDamage;
    
    public void Fire()
    {
        OnDamage?.Invoke(10);
    }
}

// 订阅
Weapon weapon = new Weapon();
weapon.OnDamage += (amount) => Debug.Log($"Damage: {amount}");
weapon.OnDamage += HandleDamage;

void HandleDamage(int amount) { }
```

### 事件（Event）

```csharp
public class Player
{
    // 事件：对委托的封装
    public event Action<int> OnHealthChanged;
    
    private int health;
    public int Health
    {
        get => health;
        set
        {
            health = value;
            OnHealthChanged?.Invoke(health);
        }
    }
}

// 订阅
player.OnHealthChanged += UpdateHealthUI;

// 事件只能在类外部 += 或 -=，不能直接赋值或调用
player.OnHealthChanged = null;  // 编译错误！
player.OnHealthChanged?.Invoke(100);  // 编译错误！
```

### 委托 vs 事件

| 特性 | 委托 | 事件 |
|------|------|------|
| 外部赋值 | 可以 | 不可以 |
| 外部调用 | 可以 | 不可以 |
| 封装性 | 差 | 好 |
| 用途 | 回调、策略模式 | 发布-订阅模式 |

```csharp
// 委托的问题
public class BadExample
{
    public Action OnEvent;  // 委托字段
}

bad.OnEvent = null;  // 可以清空所有订阅者！
bad.OnEvent();       // 可以从外部触发！

// 事件的保护
public class GoodExample
{
    public event Action OnEvent;  // 事件
}

good.OnEvent = null;  // 编译错误
good.OnEvent();       // 编译错误
```

### 内置委托类型

```csharp
// Action：无返回值
Action action = () => Debug.Log("No params");
Action<int> action1 = (x) => Debug.Log(x);
Action<int, string> action2 = (x, s) => Debug.Log($"{x}: {s}");

// Func：有返回值
Func<int> func = () => 10;
Func<int, int> func1 = (x) => x * 2;
Func<int, int, int> func2 = (x, y) => x + y;

// Predicate：返回bool
Predicate<int> isPositive = (x) => x > 0;
```

---

## 7.6 ref、out、in关键字

### ref：引用传递

```csharp
void Swap(ref int a, ref int b)
{
    int temp = a;
    a = b;
    b = temp;
}

int x = 1, y = 2;
Swap(ref x, ref y);  // x=2, y=1

// ref要求：调用前必须初始化
int z;
Swap(ref z, ref x);  // 编译错误：z未初始化
```

### out：输出参数

```csharp
bool TryParse(string s, out int result)
{
    if (int.TryParse(s, out result))
        return true;
    result = 0;  // out参数必须在方法内赋值
    return false;
}

// out不要求调用前初始化
if (TryParse("123", out int value))
{
    Debug.Log(value);
}

// C# 7.0+：内联声明
if (int.TryParse("123", out var num))
{
    Debug.Log(num);
}
```

### in：只读引用

```csharp
// in：传递引用但不能修改
void PrintVector(in Vector3 v)
{
    Debug.Log(v);
    // v.x = 10;  // 编译错误：不能修改
}

Vector3 pos = new Vector3(1, 2, 3);
PrintVector(in pos);  // 传递引用，避免复制
PrintVector(pos);     // in可以省略
```

### 对比

| 关键字 | 调用前初始化 | 方法内可修改 | 方法内必须赋值 | 用途 |
|--------|-------------|-------------|---------------|------|
| ref | 必须 | 可以 | 不必须 | 双向传递 |
| out | 不必须 | 可以 | 必须 | 返回多个值 |
| in | 必须 | 不可以 | - | 避免大struct复制 |

### 性能优化示例

```csharp
// 大struct传递优化
public readonly struct Matrix4x4
{
    // 64字节的数据
    public readonly float M11, M12, M13, M14;
    public readonly float M21, M22, M23, M24;
    public readonly float M31, M32, M33, M34;
    public readonly float M41, M42, M43, M44;
}

// 不好：每次调用复制64字节
void ProcessMatrix(Matrix4x4 matrix) { }

// 好：只传递引用（8字节）
void ProcessMatrix(in Matrix4x4 matrix) { }
```

---

## 7.7 闭包

### 什么是闭包

```csharp
// 闭包：函数捕获外部变量
void ClosureExample()
{
    int counter = 0;
    
    Action increment = () =>
    {
        counter++;  // 捕获外部变量counter
        Debug.Log(counter);
    };
    
    increment();  // 输出1
    increment();  // 输出2
    increment();  // 输出3
}
```

### 编译器如何处理闭包

```csharp
// 原始代码
void Original()
{
    int x = 10;
    Action action = () => Debug.Log(x);
    action();
}

// 编译器生成的代码（简化）
void Compiled()
{
    // 生成一个类来存储捕获的变量
    var closure = new <>c__DisplayClass0_0();
    closure.x = 10;
    
    Action action = closure.<Original>b__0;
    action();
}

class <>c__DisplayClass0_0
{
    public int x;
    
    public void <Original>b__0()
    {
        Debug.Log(x);
    }
}
```

### 闭包的陷阱

```csharp
// 陷阱1：循环中的闭包
void LoopTrap()
{
    var actions = new List<Action>();
    
    for (int i = 0; i < 5; i++)
    {
        actions.Add(() => Debug.Log(i));  // 捕获的是同一个i
    }
    
    foreach (var action in actions)
    {
        action();  // 全部输出5！
    }
}

// 解决方法：创建局部副本
void LoopFixed()
{
    var actions = new List<Action>();
    
    for (int i = 0; i < 5; i++)
    {
        int localI = i;  // 每次循环创建新变量
        actions.Add(() => Debug.Log(localI));
    }
    
    foreach (var action in actions)
    {
        action();  // 输出0, 1, 2, 3, 4
    }
}
```

### 闭包的GC问题

```csharp
// 问题：闭包导致对象无法回收
class Enemy
{
    public void StartBehavior()
    {
        // 闭包捕获this，导致Enemy无法被GC
        SomeManager.OnUpdate += () =>
        {
            this.Move();  // 捕获this
        };
    }
}

// 解决：记得取消订阅
class Enemy
{
    private Action updateAction;
    
    public void StartBehavior()
    {
        updateAction = () => this.Move();
        SomeManager.OnUpdate += updateAction;
    }
    
    public void OnDestroy()
    {
        SomeManager.OnUpdate -= updateAction;
    }
}
```

### Unity中的闭包注意事项

```csharp
// 协程中的闭包
IEnumerator CoroutineWithClosure()
{
    int count = 0;
    
    while (count < 10)
    {
        // 闭包捕获count，协程运行期间count不会被回收
        yield return new WaitForSeconds(1f);
        count++;
    }
}

// 避免在Update中创建闭包
void Update()
{
    // 不好：每帧创建新的委托对象
    list.ForEach(item => item.Update());
    
    // 好：使用普通循环
    foreach (var item in list)
    {
        item.Update();
    }
}
```

---

## 面试要点总结

### 问题28：值类型和引用类型的区别是什么？

**答案要点：**
1. **存储位置**：值类型通常在栈，引用类型在堆
2. **赋值行为**：值类型复制值，引用类型复制引用
3. **默认值**：值类型有默认值，引用类型默认null
4. **GC**：值类型无GC压力，引用类型有

### 问题29：装箱和拆箱是什么？有什么性能影响？

**答案要点：**
1. **装箱**：值类型→引用类型，在堆上分配内存
2. **拆箱**：引用类型→值类型，从堆复制到栈
3. **性能影响**：产生GC、内存分配、复制开销
4. **避免方法**：使用泛型、避免非泛型集合

### 问题30：struct和class的区别是什么？

**答案要点：**
1. **类型**：struct是值类型，class是引用类型
2. **继承**：struct不支持继承
3. **适用场景**：struct适合小型、不可变、频繁创建的数据

### 问题31：接口和抽象类的区别是什么？

**答案要点：**
1. **多继承**：接口支持，抽象类不支持
2. **实现**：接口不能有实现（C# 8前），抽象类可以
3. **设计意图**：接口定义"能做什么"，抽象类定义"是什么"

### 问题32：委托和事件的区别是什么？

**答案要点：**
1. **封装性**：事件只能+=/-=，委托可以直接赋值和调用
2. **安全性**：事件防止外部清空订阅者或触发
3. **用途**：事件用于发布-订阅模式

### 问题33：ref、out、in关键字的区别是什么？

**答案要点：**
1. **ref**：双向传递，调用前必须初始化
2. **out**：输出参数，方法内必须赋值
3. **in**：只读引用，避免大struct复制

### 问题34：什么是闭包？有什么注意事项？

**答案要点：**
1. **定义**：函数捕获外部变量
2. **实现**：编译器生成类存储捕获的变量
3. **陷阱**：循环中的闭包、GC问题
4. **注意**：避免在Update中创建闭包，记得取消事件订阅

---

[返回目录](./00_目录.md) | [上一章：数据结构](./06_数据结构.md) | [下一章：UGUI](./08_UGUI.md)