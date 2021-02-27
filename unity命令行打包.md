---
title: Command Line打包unity项目
categories:
- shengjunjie
tags:
- Unity 
---

由于神华项目总是在Scene场景进行调试，导致文件在最后打包成exe时才发现打包问题，我们因此需要通过命令行的方式，在上传到git后在后台自动打包。

<!--more-->

## 解决方案
### 前期准备
1. 批处理文件的编辑。
2. 在unity的Asset目录下新建一个文件夹Editor来存放脚本。

### 核心思路
通过批处理文件中的命令行调用unity中Editor的脚本实现后台打包功能。

### 详细过程

1. 在桌面上新建Text文件，输入
```shell
@echo off
@set unity="C:\Program Files\Unity\Editor\Unity.exe"
echo loading...
%unity%  -batchmode -quit -nographics -executeMethod BuildEditor.PerformBuild  -logFile "D:\package\Editor.log" -projectPath "D:\ShenhuaProjectNew2.0\shenhua_Project_main" 
echo Finishing!
pause
```
其中@set unity=""需要更改为当前电脑unity所安装的位置。logFile "D:\package\Editor.log"为日志所输出的位置。projectPath "D:\ShenhuaProjectNew2.0\shenhua_Project_main" 为所需要打包工程文件为位置。编辑完成后将文件保存为.bat格式。
![2](https://i.loli.net/2019/10/24/3PGpoJQxi9b1tlN.png)

2. 打开untiy，在之前新建的Editor文件中新建脚本BuildEditor，在脚本中添加以下代码：
```cs
public class BuildEditor
{

    static List<string> _scenelevels_ = new List<string>();
    public static void PerformBuild()
    {

        foreach (EditorBuildSettingsScene scene in EditorBuildSettings.scenes)
        {
            if (!scene.enabled) continue;
            _scenelevels_.Add(scene.path);

        }
              BuildPipeline.BuildPlayer(_scenelevels_.ToArray(), "D:/package/shenhua_Project_main.exe", BuildTarget.StandaloneWindows64, BuildOptions.None);
    }
 }
```
代码中"D:/package/shenhua_Project_main.exe"为打包文件所输出的位置。
最后只要双击批处理文件就可以实现Command line的打包功能。
![Uploading 1.jpg… (6qtq61a1i)](https://i.loli.net/2019/10/24/vEXykAJ78VW59rN.png)


## 总结
在实现该功能过程前，首先应确保unity许可证是激活的，其次应注意_scenelevels_的命名，如果在整个项目中存在同名的变量，将会导致项目打包失败，在测试过程中就发现在新建项目可以实现打包，但在神华项目中始终无法实现。此外还要注意项目在输出时所在磁盘有足够大的内存，否则也会打包失败。