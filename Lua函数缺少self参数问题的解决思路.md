# Lua函数缺少self参数问题的解决思路

## Lua的Self

在Lua中，提供了一定的面向对象机制，我们可以通过冒号运算符来提供隐藏参数`self`，如同`Java`、`C++`、`C#`中的`this`一样，例如：
```lua
local t = {
    a = 1,
    b = 2,
}
function t:Sum()
    return self.a + self.b
end

print(t:Sum())
```

冒号的作用有两个：
1. 对于方法定义来说，会增加一个额外的隐藏形参（`self`）
2. 对于方法调用来说，会增加一个额外的实参（表自身）

## 传入回调函数时缺少self参数的问题

有时，一个函数需要传入回调函数，用来在一定情况下，调用这个回调函数来完成功能，如观察者模式等设计模式都深刻需要回调函数的支持。

但对于需要self参数的冒号函数而言，我们无法通过`table:functionName`来获取成员函数，这是不符合Lua语法的，我们只能通过`table.functionName`来获取成员函数。

如下面代码所示，通过`table.functionName`方式获取的成员函数会缺少`self`参数：

```lua
local data = {}
function data:set(response)
    return self.dataInternal = response.data
end

request:sendTo("https://xx.com/xx?yy=zz"):whenOk(data.set) -- Error: self is nil!!!
```

## 高阶的函数

Lua中，支持高阶函数，也就是说Lua的函数可以作为参数传递给另一个函数，也可以作为另一个函数的返回值。

我们可以通过高阶函数的特性，来封装self参数到成员函数中：
```lua
function currying(fn, param)
    return function(...)
        fn(param, ...)
    end
end
```

这样我们就可以改造原有的代码为：
```lua
local data = {}
function data:set(response)
    self.dataInternal = response.data
end

request
    -- send request to backend
    :sendTo("https://xx.com/xx?yy=zz")
    -- when request receive response set response data to data variable
    :whenOk(currying(data.set)(data)) -- Ok: self is not nil
```

## 函数的柯里化

在上文中，我们创造了一种函数：他是一个函数部分传参的结果，我们只需要传递剩下的参数就能启动函数。

实际上，这是在实践函数柯里化的思想。

柯里化是一种函数的转换，它是指将一个函数从可调用的 f(a, b, c) 转换为可调用的 f(a)(b)(c)。

其实柯里化就是将函数从不可以部分传参数，转换为允许传一部分参数，剩余参数未来再传参的函数转换。在C++中有一个类似的函数叫bind，实现的是一样的功能。