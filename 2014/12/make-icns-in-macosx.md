以下内容转载自：[http://www.cocoachina.com/bbs/read.php?tid=113763](http://www.cocoachina.com/bbs/read.php?tid=113763)

### 步骤1

在任何一个目录下面创建一个文件夹，命名为  [name].iconset 例如 icon.iconset 

### 步骤2

在该文件里面放入以下图片文件，并核对尺寸是否正确 
 
|图片名称|尺寸|
|:------|:---|
|icon_512x512@2x.png|1024×1024|
|icon_512x512.png|512×512|
|icon_256x256@2x.png|512×512|
|icon_256x256.png|256×256|
|icon_128x128@2x.png|256×256|
|icon_128x128.png|128×128|
|icon_32x32@2x.png|64×64|
|icon_32x32.png|32×32|
|icon_16x16@2x.png|32×32|
|icon_16x16.png|16×16|
 
### 步骤3

打开终端，在里面输入以下命令 
 
```iconutil –c icns **/[name].iconset```
 
例如: 
```iconutil –c icns /Users/zhudongyong/Desktop/icon.iconset```

注：可以先输入`iconutil –c icns `（后面需带一个空格），再把步骤1所创建的文件夹拖到终端，则会自动把该文件夹的路径添加到刚输入的命令后面。 
 
### 步骤4

在步骤3后，系统会自动生成`[name].icns`（icon.icns）文件,将该icns文件导入工程，并设置为Icon即可。 

[Apple官方原文由此进](http://developer.apple.com/library/Mac/#documentation/General/Conceptual/MOSXAppProgrammingGuide/CommonAppBehaviors/CommonAppBehaviors.html)