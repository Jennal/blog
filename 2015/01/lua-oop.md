# lua的面向对象库

接上一篇文章[cocos2d-x中lua-binding的面向对象开发的研究](http://jennal.com/2014/10/18/cocos2dx-lua-oop/)，终于有时间整理一下我自己写的面向对象方案。
先上github地址：[https://github.com/Jennal/LuaUtils/blob/master/src/Oop.lua](https://github.com/Jennal/LuaUtils/blob/master/src/Oop.lua)

## 为什么要写这个库？

我们一直使用[cocos2d-x][1]来作为底层框架进行开发。之前用的是cpp，最近转到lua，所以难免保留一些cpp的习惯，包括饱受诟病的多继承。我不想讨论多继承有多好，或者有多糟。我认为，作为库来说，应该尽量提供符合大家习惯的机制，至于是否使用，如何使用，取决于用户。

我们知道lua在语言级别并没有支持类，而是提供了更灵活的metatable，我们可以利用metatable来实现类似cpp的类。[cocos2d-x][1]的lua绑定提供了类的支持，支持创建lua级别的类，也支持从cpp继承，但是对于面向对象机制来说，只提供继承，远远不够。

## 我想要的面向对象机制

- 多层级继承
	- 从lua继承
	- 从lua继承后的lua继承
	- 从cpp继承
	- 从cpp继承后的lua继承
- 单继承
- 多继承
	- 调用不同父类的构造函数
	- 调用不同父类的同名函数
	- 多继承不允许从2个或者以上的cpp继承
- 接口继承
	- 接口用来约束必须实现的成员函数
- 判断是否是某个类的实例
	- 包括是否是某个父类的子类的实例

## 如何使用我的库

### 单继承

#### 从lua创建对象

<pre class="lua">
local Oop = require("Oop")

-- 从lua创建对象
local A = Oop.class("A")
-- 构造函数
function A:ctor(arg)
	self.arg = arg
	print("create A")
end

-- new的参数将传递到ctor中
local a = A:new("hello") -- create A
print("a.arg", a.arg) -- a.arg	hello

-- 从lua创建的类继承
local B = Oop.class("B", A)
function B:ctor(arg, brg)
	-- 调用父类的构造函数
	B.super.A.ctor(self, arg)
	self.brg = brg
	print("create B")
end

local b = B:new("hello", "world") -- create A \n create B
print("b.arg", "b.brg", b.arg, b.brg) -- b.arg	b.brg	hello	world
</pre>

#### 从cpp创建对象

<pre class="lua">
-- 从cpp创建对象
local SpriteEx = Oop.class("SpriteEx", function(...)
	return cc.Sprite:create(...)
end)
-- 构造函数
function SpriteEx:ctor(...)
	self.arg = {...}
	print("create SpriteEx")
end

-- new的参数将传递到上面的匿名function中，以及ctor中
local sprite = SpriteEx:new("hello.png") -- create SpriteEx
print("sprite.arg", unpack(sprite.arg)) -- sprite.arg	hello.png

-- 从cpp创建的类继承
local SpecialSprite = Oop.class("SpecialSprite", SpriteEx)
function SpecialSprite:ctor(...)
	-- 调用父类的构造函数
	B.super.A.ctor(self, ...)
	print("create SpecialSprite")
end

local ss = SpecialSprite:new("hello.png") -- create SpriteEx \n create SpecialSprite
print("ss.arg", unpack(ss.arg)) -- ss.arg	hello.png
</pre>

### 多继承

#### 纯lua多继承

<pre class="lua">
local A = Oop.class("A")
function A:ctor(arg)
	self.arg = arg
	print("create A")
end

function A:show()
	print("A show", self.arg)
end

local B = Oop.class("B")
function B:ctor(arg)
	self.arg = arg
	print("create B")
end

function B:show()
	print("B show", self.arg)
end

local C = Oop.class("C", A, B)
function C:ctor(arg)
	-- 调用不同父类的构造函数
	C.super.A.ctor(self, arg)
	C.super.B.ctor(self, arg)
end

function C:showA()
	-- 调用不同父类的同名函数
	C.super.A.show(self)
end

function C:showB()
	-- 调用不同父类的同名函数
	C.super.B.show(self)
end

local c = C:new("Hello") -- create A \n create B
c:showA() -- A show	Hello
c:showB() -- B show	Hello
</pre>

#### lua与cpp的多继承

<pre class="lua">
local Oop = require("init")

local A = Oop.class("A")
function A:ctor(arg)
	self.arg = arg
	print("create A")
end

function A:show()
	print("A show", self.arg)
end

local B = Oop.class("B", function(...)
	return cc.Sprite:create(...)
end)
function B:ctor(arg)
	self.arg = arg
	print("create B")
end

function B:show()
	print("B show", self.arg)
end

local C = Oop.class("C", A, B)
function C:ctor(arg)
	C.super.A.ctor(self, arg)
	C.super.B.ctor(self)
end

function C:showA()
	C.super.A.show(self)
end

function C:showB()
	C.super.B.show(self)
end

local c = C:new("Hello.png") -- create A \n create B
c:showA() -- A show	Hello.png
c:showB() -- B show	Hello.png
-- layer:addChild(c)
</pre>

### 接口

<pre class="lua">
-- 定义一个接口
local IA = Oop.interface("IA", {
    {"func1", "(param1, param2)", "description"}, -- 成员函数1
    {"func2", ""}, -- 成员函数2
},{
    {"member", ""} -- 成员变量
})

IA.func1() -- 抛出异常: func1(param1, param2) is not implemented
local test = IA.member -- 抛出异常: member is not defined

local IAImpl = Oop.class("IAImpl", Oop.Obj, IA)

function IAImpl:func2()
    return true
end
IAImpl.member = 1

-- 实现接口
local o = IAImpl:new()
o:func1() -- 抛出异常: func1(param1, param2) is not implemented
print(o:func2()) -- true
print(o.member) -- 1
</pre>

### 判断是否某个类或接口的实例

<pre class="lua">
Oop.isClass(obj, "A") -- false
Oop.checkClass(obj, "A") -- 抛出异常：param should be A type
Oop.checkClass(obj, "A", 2) -- 抛出异常：param2 should be A type
</pre>

[1]: http://cocos2d-x.org/download/version#Cocos2d-x