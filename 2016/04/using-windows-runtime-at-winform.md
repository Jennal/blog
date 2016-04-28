# 在WinForm项目中使用Windows Runtime的方法

最近需要在Winform项目中使用蓝牙，蓝牙模块是同事负责的，找了好多版本的蓝牙库，对BLE的支持都不好。最后发现系统直接提供了`Windows.Devices.Bluetooth`这个库可以用，但是只能在Universal项目中使用。试过在nuget中找到的[Target.WindowsRuntime](https://github.com/jamesqo/Target.WindowsRuntime)，但是根本不能用。经过一番google，发现可以用hack的方法在Winform中使用，特此记录。

PS: 我用的是vs2015，win10，据说win8 vs2013也是可以的，我没有测试过。

## 步骤说明

1. 手工修改`csproj`项目文件
2. 添加对`Windows.XXXXX`库的引用
3. 添加`project.json`配置文件

## 修改项目文件

需要关闭项目工程文件，手工在目标`csproj`文件中添加如下代码

<pre class="xml">
<PropertyGroup>
  <TargetPlatformVersion>10.0</TargetPlatformVersion>
</PropertyGroup>
</pre>

你期望编译的目标操作系统是win10，就写`10.0`，如果是win8，就写`8.0`，以此类推。

上个截图，更容易理解

![](file:///E:/Codes/github/Jennal/blog/2016/04/images/uwraw-1.png)

## 添加对库的引用

这个时候，启动`sln`工程文件，然后右键点击`引用`-`添加引用...`，会发现，左侧的分类，多了一类`Universal Windows`

![](file:///E:/Codes/github/Jennal/blog/2016/04/images/uwraw-2.png)

赶紧把需要的库加进来吧，加进来以后，发现代码中可以正常引用了。

但是会编译不过，提示

<pre>
Error occurred while restoring NuGet packages: Could not find file 'C:\Users\user\Documents\Visual Studio 2015\Projects\TestWindowsRuntimeTarget\TestWindowsRuntimeTarget\project.json'.
</pre>

## 添加project.json文件

上面的错误，提示我们需要`project.json`，在项目中新建这个名称的json文件，然后复制下面的内容

<pre class="json">
{
  "frameworks": {
    ".NETFramework,Version=v4.5": {
      "dependencies": {}
    }
  },
  "runtimes": {
    "win": {}
  }
}
</pre>

其中的`v4.5`可以改成任意你需要的.net版本号。

再编译一次试试，大功告成，这样就可以顺利编译通过了。

## 参考资料

- [Using Windows 8* WinRT API from desktop applications](https://software.intel.com/en-us/articles/using-winrt-apis-from-desktop-applications)