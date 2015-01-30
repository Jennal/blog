# CocosBuilder的Cocos2dx-lua绑定方案

## 前情提要

似乎写每一份代码，每一篇文章，都有些理由，所以忍不住要写个前情提要。

[CocosBuilder][]虽然已经2年没有更新了，已经被[SpriteBuilder][]所取代，但是由于我们早期的项目使用的就是[CocosBuilder][]，而[SpriteBuilder][]增加的功能并没有特别出彩，没有足够的理由让我们抛弃[CocosBuilder][]，于是沿用下来。但是如果拿不开源的[CocosStudio][]来比较，我还是推荐使用[CocosBuilder][]。

据我所知，最早让[Cocos2dx][]的lua版本支持[CocosBuilder][]的，是这个项目：[LuaProxy][]。不知道为什么，[CocosBuilder][]官方库只做了对js的支持，甚至做了html5的支持，却完全没有考虑对lua的支持。而[LuaProxy][]就是建立在官方对js支持的基础上，做了lua的实现。但是读完[CocosBuilder][]中js方案的实现，以及[Cocos2dx][]中`CCBReader`对js方案的实现，我居然产生了，这不可能是一个人写的吧，这种想法。因为js方案与cpp方案的思路完全不同，如果是同一个人做的，这也太蛋疼了吧？（不过我没有看js那边的使用方法，只是按照[LuaProxy][]的使用方法去理解的。）

cpp方案用了一种非常优雅的方式，加载的过程中，自定义类需要有一个对应的Loader，使用字符串来索引Loader，以帮助CCBReader生成对应的类的实例。

## LuaProxy的方案的缺陷

1. 首先最不能忍的，就是用了一个全局table `ccb`来替代cpp中的`registerCCNodeLoader`，看起来好像可以解决，但是必须事先把所有需要绑定的Controller的名字写入这个`ccb`里面，也就意味着很可能许多不需要的代码会被提前加载，对于lua这样的脚本语言来说，这完全是不必要的代价。
2. 全局`ccb`也带来了另外一个问题，同屏加载多个相同ccbi时，动态更新数据的问题。
	- 试想这种情况：你用[CocosBuilder][]的粒子编辑器做了一束烟花，但是粒子贴图要在几种里面随机，这个需要在代码里面实现。于是你把这个`CCParticleSystemQuad`绑定到了`ccb`的一个table中。然后需求变了，策划需要同时放多个烟花，于是同时出现的烟花，只有最后一个的贴图可以随机了。因为他们共享同一个全局`ccb`的table，后加载的覆盖了前面加载的。这里只是举一个简单的例子，实际应用可能比这个情况复杂得多。
3. 嵌套`CCBFile`时，无法对子`CCBFile`做操作的问题。可能子`CCBFile`绑定了某个lua对象，有一些函数可以调用，有一些成员变量可以使用。但是全在全局`ccb`里面，父节点根本不知道子`CCBFile`对应全局`ccb`的哪个对象。

## 解决方案

对于lua这样的脚本语言来说，完全可以在加载ccbi的过程中，动态加载需要的lua文件，进行绑定，我们需要做的只是设定一个绑定规则。然后对[CocosBuilder][]做一些小改动，让publish的时候，同时生成lua文件，那整个开发流程将大大被缩短。

So，为了解决上面的3个问题，我做了两件事

1. 修改了`CCBReader`读取的一些方法，提供给lua读取嵌套`CCBReader`的方法。并且重写了lua读取ccbi的代码。
2. 修改了[CocosBuilder][]的源代码，让它在publish的时候，直接生成对应的lua文件。

## 准备工作

### 1.下载以下cpp文件，覆盖掉cocos2dx的相应文件，然后重新编译Runtime

- [cocos/editor-support/cocosbuilder/CCBReader.h](https://raw.githubusercontent.com/Jennal/cocos2dx-3.2-qt/master/cocos/editor-support/cocosbuilder/CCBReader.h)
- [cocos/editor-support/cocosbuilder/CCBReader.cpp](https://raw.githubusercontent.com/Jennal/cocos2dx-3.2-qt/master/cocos/editor-support/cocosbuilder/CCBReader.cpp)
- [cocos/editor-support/cocosbuilder/CCNodeLoader.cpp](https://raw.githubusercontent.com/Jennal/cocos2dx-3.2-qt/master/cocos/editor-support/cocosbuilder/CCNodeLoader.cpp)
- [cocos/scripting/lua-bindings/manual/lua_cocos2dx_extension_manual.cpp](https://raw.githubusercontent.com/Jennal/cocos2dx-3.2-qt/master/cocos/scripting/lua-bindings/manual/lua_cocos2dx_extension_manual.cpp)

PS: 值得注意的是，我修改的是cocos2dx 3.2，不确定覆盖更高或者更低的版本会不会出问题，如果不是3.2的用户，可以查看这个[commit](https://github.com/Jennal/cocos2dx-3.2-qt/commit/a25e42267450a1c04223b7ab688d6a7a58bc7d51)，自行修改相应代码。

### 2.下载以下lua文件，放到lua项目的src对应目录下

- [Oop/init.lua](https://raw.githubusercontent.com/Jennal/CocosBuilder/master/lua-binding/Oop/init.lua)
- [CCB/Loader.lua](https://raw.githubusercontent.com/Jennal/CocosBuilder/master/lua-binding/CCB/Loader.lua)

### 3.获取修改过的CocosBuilder

有2个方法

1. 从Github clone源代码自己编译：<https://github.com/jennal/CocosBuilder>
2. 下载我编译好的版本：<https://github.com/Jennal/CocosBuilder/blob/master/Release/CocosBuilder.app.tar.gz?raw=true>

PS: 值得注意的是，由于我们的项目使用的默认设计尺寸是1200x800，所以代码中的缩放比例都是针对这个分辨率的。修改也不难，在这里：[CocosBuilder/ccBuilder/ResolutionSetting.m](https://github.com/Jennal/CocosBuilder/blob/master/CocosBuilder/ccBuilder/ResolutionSetting.m)，把顶部的两个`define`改成你自己要的设计尺寸就行了。

<pre class="cpp">
#define DESIGN_WIDTH 1200
#define DESIGN_HEIGHT 800
</pre>

## 使用方法

改完Cocos2dx的框架代码，下载了lua代码，有了Cocosbuilder的修改版，准备完成，正式进入使用阶段。

### 1.ccb文件制作

ccb文件的制作基本没有什么特别，只是有几个点需要注意：

1. Publish Settings里面多了个选项，记得勾起来。新建项目，默认是勾的。
![Publish Settings](http://jennal.com/wp-content/uploads/2015/01/1.jpg)
2. 文件名与该文件的Controller名，必须保持一致，这是为了简化接口，也为了让代码与设计统一
![文件名与该文件的Controller名，必须保持一致](http://jennal.com/wp-content/uploads/2015/01/2.jpg)

### 2.代码绑定

生成的lua文件已经包含了一些注释，几乎不用有太多额外的代码，就可以很容易地绑定ccbi文件与lua类。

下面讲解几个需要注意的地方

1. `CCBLoader:setRootPath`的两个参数
<pre class="lua">
--[[
第一个参数表示绑定lua文件的package前缀
第二个参数表示ccb文件的路径
]]
CCBLoader:setRootPath("UI.ccb", "ccb/")
</pre>
2. 生成的lua文件的ctor相当于cpp中使用ccbi的`onNodeLoaded`，换句话说，代码执行到这里的时候，这个节点的子节点以及绑定都应该已经完成，可以放心使用了。可以在这里做一些初始化数据的工作。
3. 创建ccbi节点的方法
<pre class="lua">
local mainLayer = require("UI.ccb.MainLayer"):new()
</pre>

## 示例下载

<https://github.com/Jennal/CocosBuilder/blob/master/Sample/CCBSample-lua-binding.tar.gz?raw=true>

## 感谢

虽然[LuaProxy][]的方案并不完美，但也为我的改进铺平的道路，在这里感谢[LuaProxy][]的作者[shawnclovie][]。








[Cocos2dx]: https://github.com/cocos2d/cocos2d-x "Cocos2dx"
[LuaProxy]: https://github.com/shawnclovie/cocos2dx-LuaProxy "LuaProxy"
[CocosBuilder]: https://github.com/cocos2d/cocosbuilder "CocosBuilder"
[shawnclovie]: https://github.com/shawnclovie "shawnclovie"
[SpriteBuilder]: https://github.com/spritebuilder/SpriteBuilder "SpriteBuilder"
[CocosStudio]: http://cocosstudio.org/