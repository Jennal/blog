# 在MacOSX下用cmake编译cocos2dx的笔记

最近项目想用[Qt](http://qt-project.org/)来做[cocos2d-x](http://cocos2d-x.org)的跨平台编辑器。由于对Qt和OpenGL都是新手，所以想用相对低成本的方法来做，参考了不少项目。发现好多人有用Qt来做cocos2d-x的编辑器的想法，包括cocos2d-x官方，但不知道什么原因，几乎所有的项目都中途流产了。大部分项目使用的是相对较旧的cocos2d-x版本，有些甚至无法编译成功。不过这些项目也给了我不少帮助，参考了很多有价值的信息。

最后，我决定自己做一个QtPort，选用[CMake](http://www.cmake.org/)来做项目管理，因为新版的cocos2d-x已经带了CMakeLists.txt，如果直接用Qt的pro/pri/prf等项目文件，那工程量就太大了。

## 工具

### otool

这个工具帮了很大忙，因为使用CMake，所有需要链接的库都需要自己设置。而我不知道需要哪些库，利用`otool -L xxx`来查看用XCode编译出来的xxx使用了哪些库，也让我顺利把所需的库都加进来。

## 安装组件

首先要用到的就是[HomeBrew](http://brew.sh/)了。安装完HomeBrew，就可以用它安装各种东西了。

<pre class="bash">
> brew update
> brew install glew jpeg webp libtiff freetype libwebsockets glew
</pre>

这里教大家一个小技巧，当编译提示

<pre class="bash">
library not found for -lfreetype
</pre>

这里的`-l`后面就是缺失的库的名字，可以把这个名字直接放到`brew install`后面进行安装。
如果名字不对的话，也可以使用`brew search freetype`进行搜索。

## 问题

- Spine库的`CCSkeleton.cpp`文件会编译不过，我们也用不到Spine库，所以暂时不编译它。
- Audio相关的库编译不过，我在CMakeLists里加了个`AUDIO_BUILD`的`option`的选项，也暂时关闭它。
- 编译lua时遇到了比较大的问题
	- 首先，因为我们禁用了Spine和Audio的编译，但是cocos2dx的tolua代码中，并没有根据这个条件编译进行判断，是否包含这些文件。为了防止使用这些文件出问题，我加了`#ifdef`来判断，当定义了不要Spine或者Audio时，把相关的lua函数都注册成空函数。
	- 然后是luasocket的问题，cocos2dx官方根本就没有写luasocket的CMakeLists.txt，不过写起来还算顺利，唯一需要注意的是luasocket的wsocket.c和usocket.c，是需要分平台编译的，wsocket.c是针对Win平台写的，usocket.c是针对unix-like系统写的。

## 关于CMakeLists的代码

好像没有特别需要说明的，基本就是CMake的规则，唯一值得一提的是，MacOSX下面的编译，记得这么做

<pre class="bash">
# add the executable
add_executable(${APP_NAME}
  MACOSX_BUNDLE
  WIN32
  ${SAMPLE_SRC}
)
</pre>

这样生成的目录结构才会是这样的

<pre class="bash">
AppName.app/
AppName.app/Contents
AppName.app/Contents/Info.plist
AppName.app/Contents/MacOS
AppName.app/Contents/MacOS/AppExecutable
</pre>

## 最后

最后给出我的项目地址：[https://github.com/Jennal/cocos2dx-3.2-qt](https://github.com/Jennal/cocos2dx-3.2-qt)