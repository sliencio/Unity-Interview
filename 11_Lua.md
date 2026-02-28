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

## 11.7 Lua GC机制

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

## 11.8 Lua协程

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

## 11.9 xLua vs ToLua

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

## 11.10 Lua性能优化

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

## 11.11 Lua调试

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

---

[返回目录](./00_目录.md) | [上一章：热更新](./10_热更新.md) | [下一章：网络协议](./12_网络协议.md)