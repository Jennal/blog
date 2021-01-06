# Unity2018 设置 Android 程序名称的多语言

如果了解Android开发的话，应该很清楚Android的App名称，是可以多语言的，因为一般会在`AndroidManifest.xml`中，有如下配置

```
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android" xmlns:tools="http://schemas.android.com/tools" package="com.veewo.alienland">
  ...
  <application android:icon="@drawable/app_icon" android:label="@string/app_name" android:hardwareAccelerated="true">
    ...
  </application>
</manifest>
```

其中 `android:label="@string/app_name"` 就是设置App在Android桌面显示的名称。而这里使用的 `@string/app_name` 代表，这个名称，是设置在 `res/values/strings.xml` 中的。`res/values/strings.xml` 中的内容如下

```
<?xml version="1.0" encoding="utf-8"?>
<resources>
  <string name="app_name">This is a App</string>
</resources>
```

而我们可以很方便的进行多语言的配置，就是新建对应各种语言的 `values` 目录，例如：英文对应 `values-en`，中文对应 `values-zh`。甚至可以更精细地设置大陆地区使用简体中文 `values-zh-rCN`，台湾地区使用繁体中文 `values-zh-rTW`。只需要修改这些目录下对应的 `strings.xml` 的内容，就可以在不同语言的系统下，显示不同的内容。

由于Unity中Android对应的文件，都放在 `Plugins/Android` 目录下，并且看了一些文章，也都说在这个目录下建立一个 `res` 目录，然后把 `values` 目录放在 `res` 目录下就可以了。但是经过测试，发现这样并不生效。

然后又找到一篇Unity官方问答平台的回答 [Overriding App icon string for Android builds](https://answers.unity.com/questions/1300617/overriding-app-icon-string-for-android-builds.html?childToView=1311496#answer-1311496)，里面说 `values-en` 目录应该这样命名 `values-b+en`。我用这个方案测试了，一样不行。

最后，我不直接打包了，而是 `Export Project`，才发现了问题。导出的目录结构是这样的

```
.
 \ProjectName.iml
 \build
 \build.gradle
 \gradle
 \gradle.properties
 \gradlew
 \gradlew.bat
 \libs
 \local.properties
 \proguard-unity.txt
 \settings.gradle
 \src
  \main
   \assets
   \java
   \jniLibs
   \res
   \AndroidManifest.xml
 \unity-android-resources
  \AndroidManifest.xml
  \build
  \build.gradle
  \project.properties
  \res
  \unity-android-resources.iml
```

我期望 `Plugins/Android/res` 目录中的内容被拷贝到 `src/main/res`，而实际上，内容被拷贝到了 `unity-android-resources/res` 中。 其中 `unity-android-resources` 是一个Module的存在，所以代码是可以被正常访问的，但是我需要 `values` 被放到主项目的目录下。

Unity是支持自定义 `gradle` 脚本模板的，于是我想在 `build.gradle` 上做文章。

1. 进入 `Player Settings/Android/Publishing Settings` ，把 `Custom Gradle Template` 勾起来。此时，Unity会自动生成文件 `Assets/Plugins/Android/mainTemplate.gradle` 。
2. 打开 `mainTemplate.gradle` 文件，在末尾添加

```
task localizeAppName(type: Copy) {
    from("${project.rootDir}/unity-android-resources/res/") {
        include "**"
    }
    into "${project.rootDir}/src/main/res"
}

preBuild.dependsOn(localizeAppName)
```

大功告成。