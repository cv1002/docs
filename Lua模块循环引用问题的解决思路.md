# Lua模块循环引用问题的解决思路

## Lua模块

从 Lua 5.1 开始，Lua 加入了标准的模块管理机制，可以把一些公用的代码放在一个文件里，以 API 接口的形式在其他地方调用，有利于代码的重用和降低代码耦合度。

Lua提供了require函数，来加载模块，而模块的语法如下：
```lua
local module = {}

module.data1 = {}

module.function1 = function()
end

return module
```
模块具有如下的特点：
- 模块是一个文件
- 两个模块无法互相依赖
- 重复require一个模块，只会执行一次，最开始的返回值将会被缓存，后续require都使用那个返回值

### 模块无法互相依赖

如果您在Lua中构造了模块1、模块2两个模块，两个模块彼此相互引用：
```lua
------ module1.lua ------

local module2 = require("module2")

return {
    world = "world"
    hello = function()
        print("hello!" .. module2.world())
    end
}
```
```lua
------ module2.lua ------

local module1 = require("module1")

return {
    world = function()
        return module1.world
    end
}
```
在这种情况下require两个模块中的任意一个模块，都将会进行循环require，最终会爆栈，从而导致失败。

换句话说，Lua的模块无法相互依赖。

### 模块是单文件的

或许两个模块如果需要相互依赖，那么他们就应该整合为一个模块？

但在Lua中，一个文件就是一个模块，多文件时，无法将多个文件归属为同一个模块。

这也就意味着，当一个模块不断增长时，我们必须无限扩展一个文件，无法分拆，这是非常不合理的。

## 模块整合方案

我们可以认为，模块是类似函数的东西。我的意思是，模块返回的内容可以是Table，也可以是function，那么我们可以通过一些小技巧，将多个文件整合为一个模块：

```lua
------ master.lua  ------
local master = {}

local slaves = {
    "slave1",
    "slave2"
}

for i = 1， #slaves do
    require(slaves[i])(master)
end

return master

------ slave1.lua ------
return function(master)
    master.slave1.world = "world"
    master.slave1.hello = function()
        print("hello!" .. master.slave2.world())
    end
end

------ slave2.lua ------
return function(master)
    master.slave2.world = function()
        return master.slave1.world
    end
end

------ main.lua ------
require("master").slave1.hello()
```
这样，我们就能将单个大模块分拆为多个互相依赖的小模块了。
