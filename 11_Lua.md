# 十一、Lua

> 本章涵盖问题：
> - Lua的基本数据类型有哪些？
> - Lua的table是什么？如何实现数组和字典？
> - Lua的元表（metatable）是什么？
> - Lua如何实现面向对象？
> - Lua的闭包是什么？
> - Lua与C#的交互原理是什么？
> - Lua的GC机制是怎样的？
> - Lua的协程与Unity协程有什么区别？
> - xLua和ToLua的区别是什么？
> - Lua性能优化有哪些方法？
> - Lua如何调试？
> - Lua和C#交互怎么减少GC？

---

## 11.1 Lua基本数据类型

### 八种基本类型

```lua
-- 1. nil：空值
local a = nil
print(type(a))  -- "nil"

-- 2. boolean：布尔值
local b = true
local c = false

-- 3. number：数字（双精度浮点）
local num1 = 42
local num2 = 3.14
local num3 = 1e10

-- 4. string：字符串
local str1 = "hello"
local str2 = 'world'
local str3 = [[
    多行
    字符串
]]

-- 5. table：表（唯一的数据结构）
local t = {1, 2, 3}
local dict = {name = "player", level = 10}

-- 6. function：函数
local func = function(x) return x * 2 end

-- 7. userdata：用户数据（C数据）
-- 用于存储C语言的数据

-- 8. thread：协程
local co = coroutine.create(function() end)
```

### 类型特点

| 类型 | 特点 |
|------|------|
| nil | 表示"无"，未初始化变量默认为nil |
| boolean | 只有false和nil为假，其他都为真（包括0） |
| number | Lua 5.3前只有双精度浮点，5.3+支持整数 |
| string | 不可变，相同字符串共享内存 |
| table | 万能数据结构，可实现数组、字典、对象 |
| function | 一等公民，可作为参数和返回值 |

---

## 11.2 Table详解

### Table作为数组

```lua
-- 数组（索引从1开始）
local arr = {10, 20, 30, 40, 50}

print(arr[1])  -- 10（不是arr[0]！）
print(#arr)    -- 5（长度）

-- 遍历数组
for i = 1, #arr do
    print(arr[i])
end

-- 或使用ipairs
for i, v in ipairs(arr) do
    print(i, v)
end

-- 数组操作
table.insert(arr, 60)       -- 末尾添加
table.insert(arr, 1, 0)     -- 指定位置插入
table.remove(arr, 1)        -- 删除指定位置
table.sort(arr)             -- 排序
```

### Table作为字典

```lua
-- 字典
local dict = {
    name = "Player",
    level = 10,
    ["hp"] = 100,           -- 等价于 hp = 100
    ["special-key"] = 200   -- 特殊字符需要[]
}

print(dict.name)            -- "Player"
print(dict["level"])        -- 10

-- 遍历字典
for k, v in pairs(dict) do
    print(k, v)
end

-- 添加/修改
dict.newKey = "value"
dict["another"] = 123

-- 删除
dict.name = nil
```

### Table的内部结构

```
Lua Table内部结构：

┌─────────────────────────────────────────────────────┐
│ Table                                               │
├─────────────────────────────────────────────────────┤
│                                                     │
│  数组部分（Array Part）                              │
│  ┌─────┬─────┬─────┬─────┬─────┐                   │
│  │  1  │  2  │  3  │  4  │  5  │  连续整数索引     │
│  │ 10  │ 20  │ 30  │ 40  │ 50  │                   │
│  └─────┴─────┴─────┴─────┴─────┘                   │
│                                                     │
│  哈希部分（Hash Part）                               │
│  ┌─────────────────────────────────────────────┐   │
│  │ "name" → "Player"                           │   │
│  │ "level" → 10                                │   │
│  │ "hp" → 100                                  │   │
│  └─────────────────────────────────────────────┘   │
│                                                     │
└─────────────────────────────────────────────────────┘

优化：
• 连续整数索引存储在数组部分（O(1)访问）
• 其他键存储在哈希部分
• 预分配可以提高性能
```

---

## 11.3 元表（Metatable）

### 什么是元表

```lua
-- 元表定义了table的特殊行为
-- 类似于C#的运算符重载

local t = {1, 2, 3}
local mt = {}  -- 元表

setmetatable(t, mt)  -- 设置元表
getmetatable(t)      -- 获取元表
```

### 常用元方法

```lua
local mt = {
    -- 索引：访问不存在的键时调用
    __index = function(t, key)
        print("访问不存在的键: " .. key)
        return nil
    end,
    
    -- 新索引：设置不存在的键时调用
    __newindex = function(t, key, value)
        print("设置新键: " .. key .. " = " .. tostring(value))
        rawset(t, key, value)  -- 使用rawset避免递归
    end,
    
    -- 调用：把table当函数调用时
    __call = function(t, ...)
        print("Table被调用")
    end,
    
    -- 字符串转换
    __tostring = function(t)
        return "MyTable"
    end,
    
    -- 算术运算
    __add = function(a, b) return a.value + b.value end,
    __sub = function(a, b) return a.value - b.value end,
    __mul = function(a, b) return a.value * b.value end,
    __div = function(a, b) return a.value / b.value end,
    
    -- 比较运算
    __eq = function(a, b) return a.value == b.value end,
    __lt = function(a, b) return a.value < b.value end,
    __le = function(a, b) return a.value <= b.value end,
    
    -- 长度
    __len = function(t) return 0 end,
    
    -- 连接
    __concat = function(a, b) return tostring(a) .. tostring(b) end,
}
```

### __index的两种用法

```lua
-- 用法1：函数
local mt1 = {
    __index = function(t, key)
        return "default"
    end
}

-- 用法2：表（常用于继承）
local base = {
    name = "base",
    sayHello = function(self)
        print("Hello from " .. self.name)
    end
}

local mt2 = {
    __index = base  -- 找不到的键去base里找
}

local derived = {}
setmetatable(derived, mt2)

print(derived.name)      -- "base"（从base继承）
derived:sayHello()       -- "Hello from base"
```

---

## 11.4 Lua面向对象

### 基本实现

```lua
-- 定义类
local Player = {}
Player.__index = Player

-- 构造函数
function Player.new(name, level)
    local self = setmetatable({}, Player)
    self.name = name
    self.level = level
    self.hp = 100
    return self
end

-- 方法
function Player:TakeDamage(amount)
    self.hp = self.hp - amount
    print(self.name .. " takes " .. amount .. " damage, HP: " .. self.hp)
end

function Player:Heal(amount)
    self.hp = self.hp + amount
end

-- 使用
local player = Player.new("Hero", 10)
player:TakeDamage(30)  -- 注意使用冒号调用
```

### 继承实现

```lua
-- 基类
local Entity = {}
Entity.__index = Entity

function Entity.new(name)
    local self = setmetatable({}, Entity)
    self.name = name
    return self
end

function Entity:GetName()
    return self.name
end

-- 子类
local Player = setmetatable({}, {__index = Entity})
Player.__index = Player

function Player.new(name, level)
    local self = setmetatable(Entity.new(name), Player)
    self.level = level
    return self
end

function Player:GetInfo()
    return self.name .. " Lv." .. self.level
end

-- 使用
local player = Player.new("Hero", 10)
print(player:GetName())   -- "Hero"（继承自Entity）
print(player:GetInfo())   -- "Hero Lv.10"
```

### 完整的类系统

```lua
-- ���用类创建函数
function class(base)
    local cls = {}
    
    if base then
        setmetatable(cls, {__index = base})
    end
    
    cls.__index = cls
    cls.super = base
    
    function cls.new(...)
        local instance = setmetatable({}, cls)
        if instance.ctor then
            instance:ctor(...)
        end
        return instance
    end
    
    return cls
end

-- 使用
local Animal = class()

function Animal:ctor(name)
    self.name = name
end

function Animal:speak()
    print(self.name .. " makes a sound")
end

local Dog = class(Animal)

function Dog:ctor(name, breed)
    Animal.ctor(self, name)  -- 调用父类构造
    self.breed = breed
end

function Dog:speak()
    print(self.name .. " barks!")
end

local dog = Dog.new("Buddy", "Labrador")
dog:speak()  -- "Buddy barks!"
```

---

## 11.5 闭包

### 闭包的概念

```lua
-- 闭包：函数 + 它引用的外部变量

function createCounter()
    local count = 0  -- 外部变量
    
    return function()  -- 返回的函数引用了count
        count = count + 1
        return count
    end
end

local counter1 = createCounter()
print(counter1())  -- 1
print(counter1())  -- 2
print(counter1())  -- 3

local counter2 = createCounter()
print(counter2())  -- 1（独立的count）
```

### 闭包的应用

```lua
-- 1. 私有变量
function createPlayer(name)
    local hp = 100  -- 私有
    
    return {
        getName = function() return name end,
        getHP = function() return hp end,
        takeDamage = function(amount)
            hp = hp - amount
            if hp < 0 then hp = 0 end
        end
    }
end

local player = createPlayer("Hero")
print(player.getHP())      -- 100
player.takeDamage(30)
print(player.getHP())      -- 70
-- player.hp = 999         -- 无法直接访问

-- 2. 回调函数
function delayedAction(callback, data)
    -- data被闭包捕获
    return function()
        callback(data)
    end
end

-- 3. 迭代器
function range(from, to)
    local current = from - 1
    return function()
        current = current + 1
        if current <= to then
            return current
        end
    end
end

for num in range(1, 5) do
    print(num)  -- 1, 2, 3, 4, 5
end
```

---

## 11.6 Lua与C#交互

### 交互原理

```
Lua与C#交互原理：

┌─────────────────────────────────────────────────────┐
│                    Unity (C#)                       │
│  ┌───────────────────────────────────────────────┐ │
│  │                 xLua/ToLua                     │ │
│  │  ┌─────────────────────────────────────────┐  │ │
│  │  │           Lua虚拟机 (C)                 │  │ │
│  │  │  ┌───────────────────────────────────┐  │  │ │
│  │  │  │         Lua栈                     │  │  │ │
│  │  │  │  ┌─────────────────────────────┐  │  │  │ │
│  │  │  │  │ 参数和返回值在栈上传递      │  │  │  │ │
│  │  │  │  └─────────────────────────────┘  │  │  │ │
│  │  │  └───────────────────────────────────┘  │  │ │
│  │  └─────────────────────────────────────────┘  │ │
│  │                     ↕                          │ │
│  │  ┌─────────────────────────────────────────┐  │ │
│  │  │         Wrapper/Adapter                 │  │ │
│  │  │  • 类型转换                             │  │ │
│  │  │  • 方法绑定                             │  │ │
│  │  │  • 委托适配                             │  │ │
│  │  └─────────────────────────────────────────┘  │ │
│  └───────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────┘
```

### C#调用Lua

```csharp
// xLua示例
using XLua;

public class CSharpCallLua : MonoBehaviour
{
    private LuaEnv luaEnv;
    
    // 定义委托类型
    [CSharpCallLua]
    public delegate int AddDelegate(int a, int b);
    
    [CSharpCallLua]
    public interface IPlayer
    {
        string Name { get; set; }
        void Attack();
    }
    
    void Start()
    {
        luaEnv = new LuaEnv();
        
        // 执行Lua代码
        luaEnv.DoString(@"
            function add(a, b)
                return a + b
            end
            
            player = {
                Name = 'Hero',
                Attack = function(self)
                    print(self.Name .. ' attacks!')
                end
            }
        ");
        
        // 获取Lua函数
        AddDelegate add = luaEnv.Global.Get<AddDelegate>("add");
        int result = add(10, 20);  // 30
        
        // 获取Lua表作为接口
        IPlayer player = luaEnv.Global.Get<IPlayer>("player");
        player.Attack();
        
        // 获取Lua表作为LuaTable
        LuaTable table = luaEnv.Global.Get<LuaTable>("player");
        string name = table.Get<string>("Name");
    }
}
```

### Lua调用C#

```csharp
// C#类需要添加特性
[LuaCallCSharp]
public class GameManager
{
    public static int Score { get; set; }
    
    public static void AddScore(int amount)
    {
        Score += amount;
        Debug.Log($"Score: {Score}");
    }
}

[LuaCallCSharp]
public class Player
{
    public string Name;
    public int Level;
    
    public void TakeDamage(int amount)
    {
        Debug.Log($"{Name} takes {amount} damage");
    }
}
```

```lua
-- Lua中调用C#
local CS = CS  -- C#命名空间

-- 调用静态方法
CS.GameManager.AddScore(100)
print(CS.GameManager.Score)

-- 创建C#对象
local player = CS.Player()
player.Name = "Hero"
player.Level = 10
player:TakeDamage(50)

-- 调用Unity API
local go = CS.UnityEngine.GameObject("NewObject")
local transform = go.transform
transform.position = CS.UnityEngine.Vector3(1, 2, 3)

-- 使用Unity组件
local rb = go:AddComponent(typeof(CS.UnityEngine.Rigidbody))
rb.mass = 10
```

---

## 11.7 Lua与C#交互的GC优化

### GC产生的原因

```
Lua与C#交互时GC产生的主要来源：

┌─────────────────────────────────────────────────────────────┐
│                    GC产生的场景                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. 值类型装箱（Boxing）                                     │
│     ┌─────────────────────────────────────────────────┐     │
│     │ Lua传递int/float/bool到C# → 装箱为object        │     │
│     │ C#返回值类型到Lua → 装箱                         │     │
│     └─────────────────────────────────────────────────┘     │
│                                                             │
│  2. 字符串转换                                               │
│     ┌─────────────────────────────────────────────────┐     │
│     │ Lua string ↔ C# string 每次都创建新对象          │     │
│     └─────────────────────────────────────────────────┘     │
│                                                             │
│  3. 委托/闭包创建                                            │
│     ┌─────────────────────────────────────────────────┐     │
│     │ 每次传递Lua函数到C#都创建新的委托对象             │     │
│     └─────────────────────────────────────────────────┘     │
│                                                             │
│  4. 数组/表转换                                              │
│     ┌─────────────────────────────────────────────────┐     │
│     │ Lua table ↔ C# array/List 需要创建新集合         │     │
│     └─────────────────────────────────────────────────┘     │
│                                                             │
│  5. 参数传递                                                 │
│     ┌─────────────────────────────────────────────────┐     │
│     │ 可变参数(params)会创建数组                        │     │
│     │ 多返回值需要创建临时对象                          │     │
│     └─────────────────────────────────────────────────┘     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 优化策略1：避免频繁的跨语言调用

```csharp
// ❌ 不好：每帧多次跨语言调用
public class BadExample : MonoBehaviour
{
    private LuaFunction luaUpdate;
    
    void Update()
    {
        // 每帧调用多次Lua函数，每次都有GC开销
        luaUpdate.Call(transform.position.x);  // GC
        luaUpdate.Call(transform.position.y);  // GC
        luaUpdate.Call(transform.position.z);  // GC
    }
}

// ✅ 好：批量传递数据，减少调用次数
public class GoodExample : MonoBehaviour
{
    private LuaFunction luaUpdate;
    private float[] positionArray = new float[3];  // 复用数组
    
    void Update()
    {
        // 一次调用传递所有数据
        positionArray[0] = transform.position.x;
        positionArray[1] = transform.position.y;
        positionArray[2] = transform.position.z;
        luaUpdate.Call(positionArray);  // 只有一次GC
    }
}
```

### 优化策略2：缓存委托和LuaFunction

```csharp
// ❌ 不好：每次都获取Lua函数
public class BadDelegate : MonoBehaviour
{
    private LuaEnv luaEnv;
    
    void Update()
    {
        // 每帧都创建新的委托对象
        var func = luaEnv.Global.Get<Action<int>>("OnUpdate");
        func(Time.frameCount);
    }
}

// ✅ 好：缓存委托
public class GoodDelegate : MonoBehaviour
{
    private LuaEnv luaEnv;
    private Action<int> cachedOnUpdate;  // 缓存委托
    
    void Start()
    {
        // 只获取一次
        cachedOnUpdate = luaEnv.Global.Get<Action<int>>("OnUpdate");
    }
    
    void Update()
    {
        cachedOnUpdate?.Invoke(Time.frameCount);  // 无GC
    }
    
    void OnDestroy()
    {
        cachedOnUpdate = null;  // 释放引用
    }
}
```

### 优化策略3：使用struct代替class传递数据

```csharp
// ❌ 不好：使用class传递数据
[LuaCallCSharp]
public class PlayerData  // class会在堆上分配
{
    public float x, y, z;
    public int hp;
}

// ✅ 好：使用struct（需要xLua的GCOptimize特性）
[LuaCallCSharp]
[GCOptimize]  // xLua特性，优化struct的传递
public struct PlayerDataStruct
{
    public float x, y, z;
    public int hp;
}

// 在xLua中配置
public static class XLuaConfig
{
    [GCOptimize]
    public static List<Type> GCOptimizeList = new List<Type>
    {
        typeof(Vector3),
        typeof(Vector2),
        typeof(Quaternion),
        typeof(Color),
        typeof(PlayerDataStruct),
    };
}
```

### 优化策略4：避免字符串频繁转换

```lua
-- ❌ 不好：频繁传递字符串
function Update()
    -- 每帧都创建新字符串
    CS.Debug.Log("Player position: " .. tostring(x) .. ", " .. tostring(y))
end

-- ✅ 好：使用ID或枚举代替字符串
local LogType = {
    Position = 1,
    Health = 2,
    Damage = 3,
}

function Update()
    -- 传递数字，无字符串GC
    CS.GameLogger.Log(LogType.Position, x, y)
end
```

```csharp
// C#端使用枚举或ID
[LuaCallCSharp]
public static class GameLogger
{
    private static readonly string[] LogFormats = {
        "",
        "Position: {0}, {1}",
        "Health: {0}",
        "Damage: {0}",
    };
    
    public static void Log(int type, params object[] args)
    {
        // 使用预定义格式，减少字符串创建
        Debug.LogFormat(LogFormats[type], args);
    }
}
```

### 优化策略5：使用对象池

```csharp
// C#端对象池
[LuaCallCSharp]
public class LuaObjectPool<T> where T : class, new()
{
    private static Stack<T> pool = new Stack<T>();
    
    public static T Get()
    {
        return pool.Count > 0 ? pool.Pop() : new T();
    }
    
    public static void Return(T obj)
    {
        pool.Push(obj);
    }
}
```

```lua
-- Lua端使用对象池
local Vector3Pool = {}
local poolSize = 0

function Vector3Pool.Get()
    if poolSize > 0 then
        poolSize = poolSize - 1
        return table.remove(Vector3Pool, poolSize + 1)
    end
    return {x = 0, y = 0, z = 0}
end

function Vector3Pool.Return(v)
    v.x, v.y, v.z = 0, 0, 0
    poolSize = poolSize + 1
    Vector3Pool[poolSize] = v
end

-- 使用
local pos = Vector3Pool.Get()
pos.x, pos.y, pos.z = 1, 2, 3
-- 使用完毕
Vector3Pool.Return(pos)
```

### 优化策略6：减少返回值数量

```csharp
// ❌ 不好：多返回值会创建临时对象
[LuaCallCSharp]
public class BadReturn
{
    public static (int, int, int) GetPosition()  // 元组会装箱
    {
        return (1, 2, 3);
    }
}

// ✅ 好：使用out参数或传入引用
[LuaCallCSharp]
public class GoodReturn
{
    // 方法1：使用数组（预分配）
    private static int[] resultArray = new int[3];
    
    public static int[] GetPosition()
    {
        resultArray[0] = 1;
        resultArray[1] = 2;
        resultArray[2] = 3;
        return resultArray;  // 复用数组
    }
    
    // 方法2：传入Lua table填充
    public static void FillPosition(LuaTable table)
    {
        table.Set("x", 1);
        table.Set("y", 2);
        table.Set("z", 3);
    }
}
```

### 优化策略7：使用LuaTable代替Dictionary

```csharp
// ❌ 不好：每次转换都创建新Dictionary
public void ProcessData(Dictionary<string, object> data)
{
    // Dictionary创建有GC
}

// ✅ 好：直接使用LuaTable
public void ProcessData(LuaTable data)
{
    // 直接操作Lua表，无需转换
    int value = data.Get<int>("key");
}
```

### 优化策略8：xLua的GCOptimize配置

```csharp
// xLua配置文件
public static class XLuaGenConfig
{
    // 需要GC优化的值类型
    [GCOptimize]
    public static List<Type> GCOptimizeList = new List<Type>
    {
        // Unity常用值类型
        typeof(Vector2),
        typeof(Vector3),
        typeof(Vector4),
        typeof(Quaternion),
        typeof(Color),
        typeof(Color32),
        typeof(Rect),
        typeof(Bounds),
        typeof(Ray),
        typeof(RaycastHit),
        
        // 自定义值类型
        typeof(MyCustomStruct),
    };
    
    // 需要适配的委托类型
    [CSharpCallLua]
    public static List<Type> CSharpCallLuaList = new List<Type>
    {
        typeof(Action),
        typeof(Action<int>),
        typeof(Action<float>),
        typeof(Action<string>),
        typeof(Func<int>),
        typeof(Func<float>),
        // 预定义常用委托，避免运行时生成
    };
}
```

### 优化策略9：避免在热点代码中跨语言

```lua
-- ❌ 不好：在Update中频繁调用C#
function Update()
    local pos = CS.UnityEngine.Input.mousePosition  -- 每帧GC
    local ray = CS.UnityEngine.Camera.main:ScreenPointToRay(pos)  -- 每帧GC
end

-- ✅ 好：在C#中处理热点逻辑，只传递结果
-- C#端
[LuaCallCSharp]
public class InputHelper
{
    private static Vector3 mouseWorldPos;
    
    public static void UpdateMousePosition()
    {
        // 在C#中计算，避免跨语言
        var ray = Camera.main.ScreenPointToRay(Input.mousePosition);
        if (Physics.Raycast(ray, out var hit))
        {
            mouseWorldPos = hit.point;
        }
    }
    
    public static float MouseX => mouseWorldPos.x;
    public static float MouseY => mouseWorldPos.y;
    public static float MouseZ => mouseWorldPos.z;
}

-- Lua端
function Update()
    CS.InputHelper.UpdateMousePosition()  -- 一次调用
    local x = CS.InputHelper.MouseX  -- 简单属性访问，GC很小
    local y = CS.InputHelper.MouseY
end
```

### 优化策略10：使用静态方法代替实例方法

```csharp
// ❌ 不好：实例方法需要传递this
[LuaCallCSharp]
public class Calculator
{
    public int Add(int a, int b) => a + b;
}

// Lua中：
// local calc = CS.Calculator()  -- 创建实例，有GC
// calc:Add(1, 2)

// ✅ 好：使用静态方法
[LuaCallCSharp]
public static class Calculator
{
    public static int Add(int a, int b) => a + b;
}

// Lua中：
// CS.Calculator.Add(1, 2)  -- 无需创建实例
```

### GC优化检测工具

```csharp
// 简单的GC监控
public class GCMonitor : MonoBehaviour
{
    private int lastGCCount;
    private float checkInterval = 1f;
    private float timer;
    
    void Update()
    {
        timer += Time.deltaTime;
        if (timer >= checkInterval)
        {
            timer = 0;
            int currentGC = GC.CollectionCount(0);
            if (currentGC > lastGCCount)
            {
                Debug.LogWarning($"GC发生了 {currentGC - lastGCCount} 次");
            }
            lastGCCount = currentGC;
        }
    }
}
```

### GC优化总结表

| 优化策略 | 效果 | 实现难度 |
|----------|------|----------|
| 缓存委托/LuaFunction | ⭐⭐⭐⭐⭐ | 低 |
| 减少跨语言调用次数 | ⭐⭐⭐⭐⭐ | 中 |
| 使用GCOptimize特性 | ⭐⭐⭐⭐ | 低 |
| 对象池 | ⭐⭐⭐⭐ | 中 |
| 避免字符串转换 | ⭐⭐⭐ | 中 |
| 使用静态方法 | ⭐⭐⭐ | 低 |
| 批量传递数据 | ⭐⭐⭐ | 中 |
| 热点代码放C#端 | ⭐⭐⭐⭐⭐ | 高 |

---

## 11.8 Lua GC机制

### GC原理

```
Lua GC采用增量式标记-清除算法：

1. 标记阶段（Mark）
   ┌─────────────────────────────────────┐
   │ 从根对象开始，标记所有可达对象      │
   │ 根对象：全局表、注册表、栈上的对象  │
   └─────────────────────────────────────┘

2. 清除阶段（Sweep）
   ┌─────────────────────────────────────┐
   │ 遍历所有对象，回收未标记的对象      │
   └─────────────────────────────────────┘

增量式：
• 不是一次性完成，分多步执行
• 避免长时间停顿
• 通过GC步进控制
```

### GC控制

```lua
-- 手动控制GC
collectgarbage("collect")    -- 完整GC
collectgarbage("stop")       -- 停止GC
collectgarbage("restart")    -- 重启GC
collectgarbage("step", 100)  -- 执行一步GC

-- 获取内存使用
local mem = collectgarbage("count")  -- KB
print("Memory: " .. mem .. " KB")

-- 设置GC参数
collectgarbage("setpause", 100)      -- 暂停百分比
collectgarbage("setstepmul", 200)    -- 步进倍率
```

### GC优化

```lua
-- 1. 减少临时对象
-- 不好
function update()
    local pos = {x = 0, y = 0}  -- 每帧创建新表
end

-- 好
local pos = {x = 0, y = 0}
function update()
    pos.x = 0
    pos.y = 0  -- 复用表
end

-- 2. 使用对象池
local pool = {}

function getObject()
    if #pool > 0 then
        return table.remove(pool)
    end
    return {}
end

function returnObject(obj)
    -- 清理对象
    for k in pairs(obj) do
        obj[k] = nil
    end
    table.insert(pool, obj)
end

-- 3. 避免字符串拼接
-- 不好
local str = ""
for i = 1, 1000 do
    str = str .. i  -- 每次创建新字符串
end

-- 好
local t = {}
for i = 1, 1000 do
    t[i] = i
end
local str = table.concat(t)
```

---

## 11.9 Lua协程

### 基本使用

```lua
-- 创建协程
local co = coroutine.create(function(a, b)
    print("协程开始", a, b)
    local c = coroutine.yield(a + b)  -- 暂停并返回
    print("协程继续", c)
    return "完成"
end)

-- 运行协程
local status, result = coroutine.resume(co, 1, 2)
print(status, result)  -- true, 3

status, result = coroutine.resume(co, 100)
print(status, result)  -- true, "完成"

-- 协程状态
print(coroutine.status(co))  -- "dead"
```

### 协程状态

| 状态 | 说明 |
|------|------|
| suspended | 暂停（刚创建或yield后） |
| running | 运行中 |
| normal | 正常（调用了其他协程） |
| dead | 结束 |

### Lua协程 vs Unity协程

| 特性 | Lua协程 | Unity协程 |
|------|---------|-----------|
| 实现 | 语言级别 | 基于IEnumerator |
| 调度 | 手动resume | Unity自动调度 |
| 返回值 | 支持 | 不直接支持 |
| 嵌套 | 支持 | 支持 |
| 用途 | 通用 | 主要用于异步 |

### 在Unity中使用Lua协程

```lua
-- 封装Unity风格的协程
local function wait(seconds)
    local startTime = os.clock()
    while os.clock() - startTime < seconds do
        coroutine.yield()
    end
end

local function myCoroutine()
    print("开始")
    wait(1)
    print("1秒后")
    wait(2)
    print("再2秒后")
end

-- 需要在Update中驱动
local co = coroutine.create(myCoroutine)

function update()
    if coroutine.status(co) ~= "dead" then
        coroutine.resume(co)
    end
end
```

---

## 11.10 xLua vs ToLua

### 对比

| 特性 | xLua | ToLua |
|------|------|-------|
| 维护者 | 腾讯 | 开源社区 |
| 热修复 | 支持（Hotfix） | 不直接支持 |
| 代码生成 | 按需生成 | 全量生成 |
| 学习曲线 | 较平缓 | 较陡峭 |
| 性能 | 略低 | 略高 |
| 文档 | 完善 | 一般 |

### xLua特性

```csharp
// xLua热修复
[Hotfix]
public class Player
{
    public int hp = 100;
    
    public void TakeDamage(int amount)
    {
        hp -= amount;
    }
}
```

```lua
-- Lua中修复C#方法
xlua.hotfix(CS.Player, 'TakeDamage', function(self, amount)
    -- 新的实现
    if amount > 50 then
        amount = 50  -- 限制最大伤害
    end
    self.hp = self.hp - amount
end)
```

---

## 11.11 Lua性能优化

### 优化技巧

```lua
-- 1. 局部化全局变量
local pairs = pairs
local ipairs = ipairs
local type = type
local math_sin = math.sin

-- 2. 避免在循环中创建函数
-- 不好
for i = 1, 1000 do
    table.sort(arr, function(a, b) return a < b end)
end

-- 好
local compare = function(a, b) return a < b end
for i = 1, 1000 do
    table.sort(arr, compare)
end

-- 3. 预分配表大小
-- 不好
local t = {}
for i = 1, 1000 do
    t[i] = i
end

-- 好（如果知道大小）
local t = {0,0,0,0,0,0,0,0,0,0}  -- 预分配
-- 或使用table.create（LuaJIT）

-- 4. 使用整数索引
-- 数组部分比哈希部分快

-- 5. 避免使用全局变量
-- 全局变量查找比局部变量慢

-- 6. 字符串优化
-- 使用table.concat代替..拼接
-- 短字符串会被内部化（共享）
```

---

## 11.12 Lua调试

### 调试方法

```lua
-- 1. print调试
print("Debug:", variable)

-- 2. 断言
assert(condition, "Error message")

-- 3. 错误处理
local status, err = pcall(function()
    -- 可能出错的代码
    error("Something wrong")
end)

if not status then
    print("Error:", err)
end

-- 4. 调用栈
print(debug.traceback())

-- 5. 变量检查
local info = debug.getinfo(1)
print(info.name, info.source, info.currentline)

-- 6. 使用IDE调试器
-- ZeroBrane Studio
-- VSCode + Lua Debug插件
```

---

## 面试要点总结

### 问题60：Lua的基本数据类型有哪些？

**答案要点：**
8种：nil、boolean、number、string、table、function、userdata、thread

### 问题61：Lua的table是什么？如何实现数组和字典？

**答案要点：**
- table是Lua唯一的数据结构
- 数组：连续整数索引（从1开始）
- 字典：任意键值对
- 内部分为数组部分和哈希部分

### 问题62：Lua的元表（metatable）是什么？

**答案要点：**
- 定义table的特殊行为
- 常用元方法：__index、__newindex、__call、__tostring
- __index用于实现继承

### 问题63：Lua如何实现面向对象？

**答案要点：**
- 使用table + metatable
- __index指向类表实现方法继承
- 冒号语法自动传递self

### 问题64：Lua的闭包是什么？

**答案要点：**
- 函数 + 它引用的外部变量
- 用于实现私有变量、回调、迭代器

### 问题65：Lua与C#的交互原理是什么？

**答案要点：**
- 通过Lua栈传递参数和返回值
- Wrapper/Adapter进行类型转换
- C#调用Lua：获取函数/表，Invoke调用
- Lua调用C#：通过CS命名空间访问

### 问题66：Lua的GC机制是怎样的？

**答案要点：**
- 增量式标记-清除算法
- 分步执行，避免长时间停顿
- 可通过collectgarbage控制

### 问题67：Lua的协程与Unity协程有什么区别？

**答案要点：**
- Lua协程：语言级别，手动resume，支持返回值
- Unity协程：基于IEnumerator，自动调度

### 问题68：xLua和ToLua的区别是什么？

**答案要点：**
- xLua：腾讯维护，支持热修复，按需生成
- ToLua：社区维护，性能略高，全量生成

### 问题69：Lua性能优化有哪些方法？

**答案要点：**
1. 局部化全局变量
2. 避免循环中创建函数
3. 预分配表大小
4. 使用整数索引
5. 使用table.concat拼接字符串

### 问题70：Lua如何调试？

**答案要点：**
- print调试
- pcall错误处理
- debug.traceback调用栈
- IDE调试器（ZeroBrane、VSCode）

### 问题71：Lua和C#交互怎么减少GC？

**答案要点：**

1. **GC产生的主要原因**：
   - 值类型装箱（int/float/bool传递）
   - 字符串转换
   - 委托/闭包创建
   - 数组/表转换
   - 可变参数

2. **核心优化策略**：
   - **缓存委托**：Start时获取，避免每帧创建
   - **减少调用次数**：批量传递数据
   - **GCOptimize特性**：优化Vector3等值类型传递
   - **对象池**：复用Lua table和C#对象
   - **避免字符串**：使用ID/枚举代替
   - **静态方法**：避免创建实例
   - **热点代码放C#**：减少跨语言调用

3. **xLua配置**：
   ```csharp
   [GCOptimize]
   public static List<Type> GCOptimizeList = new List<Type>
   {
       typeof(Vector3),
       typeof(Quaternion),
       // 自定义struct
   };
   ```

4. **最佳实践**：
   - Update中避免跨语言调用
   - 预分配数组复用
   - 使用LuaTable代替Dictionary
   - 监控GC频率定位问题

---

[返回目录](./00_目录.md) | [上一章：热更新](./10_热更新.md) | [下一章：网络协议](./12_网络协议.md)