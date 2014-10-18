# cocos2d-x中lua-binding的面向对象开发的研究

## 起因

网路上已经有很多关于lua面向对象开发的文章，为什么我要自己写一篇呢？

一切都是因[cocos2d-x 3.2][1]的lua绑定而起。我们的新项目打算用lua进行开发，一直在跟进[quick-x][2]项目，从独立、第三方，到现在的被触控收编。[cocos2d-x 3.2][1]是最新的稳定版，把很多[quick-x][2]很多好东西吸纳进来。与其说吸纳，不如说是直接把源代码拿过来了。所以，现在[cocos2d-x 3.2][1]直接就支持lua的面向对象开发，提供了原来只有[quick-x][2]才有的[class](https://github.com/cocos2d/cocos2d-x/blob/v3/cocos/scripting/lua-bindings/script/cocos2d/extern.lua)等方法来方便的创建对象以及做继承。

既然cocos2d-x官方都认为quick-x的东西很好，我也不得不研究一下这套针对lua开发的创建类和继承的方法。不研究不要紧，一研究吓一跳。备受关注的quick-x，把自己宣传的那么好，那么便捷（我刚开始研究quick-x的时候，用它写过一个项目，提供的接口确实很便捷，大大提高了开发效率，但是没有研究实现细节），没想到最最底层，最最基础的部分，创建类、类继承的方案却如此业余，简直是一个刚学lua两三天的“高手”写出来的。说刚学两三天，是因为，从这个方案可以看出，完全没有理解lua语言的真谛，没有做到`thinking in lua`，而“高手”并不是贬义，一般人真的做不出这样的设计，写不出这样的代码。

## 分析

当然，也不是说[这份代码](https://github.com/cocos2d/cocos2d-x/blob/v3/cocos/scripting/lua-bindings/script/cocos2d/extern.lua)一无是处（下个章节会详细分析），我这里先说说这份代码的优点和缺陷，仅代表个人意见，希望能引起共鸣。

### 优点

1. 分解了创建实例(cls.__create)和构造函数的方法(instance:ctor)，这样做的好处很明显，让混沌的思路清晰起来，并且“支持”从function继承，这也是支持从cpp对象继承的基础。
2. 创造了类与类的实例之间的区别
    * 类：cls.class == nil
    * 类的实例：instance.class ~= nil
3. 支持从cpp、lua的table继承，也支持创造新的类
4. 支持默认构造函数
    * 当创建新的类时，有一个什么都不做的构造函数
    * 当lua的table继承时，可以调用父类的构造函数

### 缺陷

1. 不支持多继承，这对于从cpp开发转过来的开发人员来说，可能是最大的问题，因为lua的特性其实是可以支持多继承的
2. 没有充分利用lua的特性（下面review的时候详细展开讲）
    * 从lua的table继承时，不需要做clone，clone会失去很多好处，并且带来额外的内存开销
3. 对于不是使用class创建的父类，支持不好，这个其实问题不大，就是子类必须要写构造函数ctor

### 小结

所以总体上来看，还是优点多于缺点的。但是对于用惯了多继承（即使是不支持多继承的语言，比如Java、C#也是支持多接口实现的）的我来说，这点实在不可忍，所以不得不自己写了一套[继承机制][3]，为了避免这篇文章太长，将在下一篇文章中公开。

## 代码review

下面是重头戏了，让我们来看看class函数的代码。

<pre class="lua">
--Create an class.
function class(classname, super)
    -- Part 1 --
    local superType = type(super)
    local cls

    if superType ~= "function" and superType ~= "table" then
        superType = nil
        super = nil
    end
    -- End of Part 1 --

    if superType == "function" or (super and super.__ctype == 1) then
        -- Part 2 --
        -- inherited from native C++ Object
        cls = {}

        if superType == "table" then
            -- copy fields from super
            for k,v in pairs(super) do cls[k] = v end
            cls.__create = super.__create
            cls.super    = super
        else
            cls.__create = super
        end

        cls.ctor    = function() end
        cls.__cname = classname
        cls.__ctype = 1

        function cls.new(...)
            local instance = cls.__create(...)
            -- copy fields from class to native object
            for k,v in pairs(cls) do instance[k] = v end
            instance.class = cls
            instance:ctor(...)
            return instance
        end
        -- End of Part 2 --
    else
        -- Part 3 --
        -- inherited from Lua Object
        if super then
            cls = clone(super)
            cls.super = super
        else
            cls = {ctor = function() end}
        end

        cls.__cname = classname
        cls.__ctype = 2 -- lua
        cls.__index = cls

        function cls.new(...)
            local instance = setmetatable({}, cls)
            instance.class = cls
            instance:ctor(...)
            return instance
        end
        -- End of Part 3 --
    end

    return cls
end
</pre>

为了让没有使用过这个函数的读者也能读懂，先说一下函数的用法吧

<pre class="lua">
--- Sample1: 创建新的类
-- 创建一个名为ClassA的类
local ClassA = class("ClassA")
-- 创建ClassA类的实例
local a = ClassA.new()

-- ClassA的构造函数
function ClassA:ctor()
    self.name = "a"
end
local aa = ClassA.new()
print(aa.name) -- 输出：a

--- Sample2: 从lua的table继承
-- 创建一个名为ClassB的类，ClassB继承于ClassA
local ClassB = class("ClassB", ClassA)
-- 创建ClassB类的实例
local b = ClassB.new()

--- Sample3: 从cpp对象（在lua中是userdata）继承
-- 创建一个名为ClassC的类，ClassC继承于cc.Scene
local ClassC = class("ClassC", function()
    return cc.Scene:create()
end)
-- 创建ClassC类的实例
local c = ClassC.new()
</pre>

总体上看，这个函数可以分为3个部分（我已经在代码中用注释标明）。

1. 局部变量声明以及参数类型检查
2. 从cpp类继承
3. 从lua的table继承

### 局部变量声明以及参数类型检查

<pre class="lua">
local superType = type(super) -- 父类的类型，用于后面决定要进入哪个分支
local cls -- 这将是最后的返回值

-- 父类的类型必须是function或者table，如果都不是，就当做super传的是nil，也就是说当做创建新类来处理
if superType ~= "function" and superType ~= "table" then
    superType = nil
    super = nil
end
</pre>

### 从cpp类继承

<pre class="lua">
-- inherited from native C++ Object
cls = {} -- 初始化cls

-- 父类是从cpp的类继承的
if superType == "table" then
    -- 从父类拷贝所有的元素，我认为这里是有问题的，详见下面的问题1
    for k,v in pairs(super) do cls[k] = v end
    cls.__create = super.__create -- 我想这个只是用来提醒自己，其实这个赋值已经包含在上面的for循环中了
    cls.super    = super
else
    -- 传进来的super是function，__create用来创建类的实例
    cls.__create = super
end

cls.ctor    = function() end  -- 默认构造函数，我认为这里也是有问题的，详见下面的问题2
cls.__cname = classname  -- 类名
cls.__ctype = 1  -- 类的类型，代表是从cpp的类（userdata）继承的

-- 类的new方法
function cls.new(...)
    local instance = cls.__create(...)  -- 创建类的实例
    -- 拷贝所有的元素到实例中，我认为这里也是有问题的，详见下面的问题3
    for k,v in pairs(cls) do instance[k] = v end
    instance.class = cls  -- 对实例的class进行赋值，可以从实例找到cls
    instance:ctor(...)  -- 调用构造函数
    return instance
end
</pre>

#### 问题1：从父类做深度拷贝

这里的super已经是拷贝过cpp类（userdata）的所有元素，已经是个lua的table了，我认为这里可以完全当做从lua的table继承来处理，而没必要再做一次深度拷贝。

#### 问题2：默认的空构造函数

先理一下代码走到这里的前提

* super是function
* 或者super是从cpp类继承的table

你想到了什么？对，如果super是从cpp类继承的table，那么这里的默认构造函数就屏蔽了父类的构造函数。当然，在构造函数ctor里面有办法调到父类的构造函数的，可以这么做`self.class.super.ctor(self)`。但是回过头来看第一个前提，super是function的时候，super是没有被赋值到`self.class.super`的，所以现在要调用父类的构造函数的话，莫非还要判断一下？

#### 问题3：创建实例时的深度拷贝

这是在干嘛呢？拷贝那么多遍干嘛？都已经拷贝一遍到`cls`里面了，再拷贝一遍到`instance`干嘛呢？完全没有`thinking in lua`嘛！这里只需要`setmetatable`就可以了呀。

### 从lua的table继承

<pre class="lua">
-- inherited from Lua Object
if super then
    cls = clone(super)  -- 从父类做深度拷贝，又来了，问题4
    cls.super = super  -- 标记父类
else
    cls = {ctor = function() end}  -- 父类为空时，创建默认构造函数
end

cls.__cname = classname  -- 保存类名
cls.__ctype = 2 -- lua  -- 标记这是个lua继承来的类
cls.__index = cls  -- 为setmetatable做准备

-- 类的new方法
function cls.new(...)
    local instance = setmetatable({}, cls)  -- “正统的”lua类继承来了
    instance.class = cls  -- 对实例的class进行赋值，可以从实例找到cls
    instance:ctor(...)  -- 调用构造函数
    return instance
end
</pre>

#### 问题4：从父类做深度拷贝

刚刚从`userdata`做深度还是可以理解的，但是现在是从lua的table做继承呀，放着`setmetatable`不用，又`clone`干嘛呀？

## 总结

很多观点可能会有人认为偏颇，因为深度拷贝的时候，对于table、function、userdata等数据来说，也只是拷贝一个引用，并没有太大的代价。但我始终固执地认为，既然要用lua，就应该用lua的思维来解决问题，特别是底层的部分。

不过至少大家可以达成共识，最大的问题就是问题2。

最后，放出我自己写的[Oop库][3]来解决上面的问题，并且有更多惊喜等着你去发现。


[1]: http://cocos2d-x.org/download/version#Cocos2d-x
[2]: https://github.com/chukong/quick-cocos2d-x
[3]: https://github.com/Jennal/LuaUtils/blob/master/src/Oop.lua