---
title: "如何生成compile_commands.json"
date: 2024-07-11 15:42:00 +0800  # 注意时区
categories: [博客, 教程]  # 分类（可多个）
tags: [tool, script]      # 标签（可多个）
---

## 如何生成compile_commands.json

生成compile_commands.json的主要目的就是在vscode中配合插件clangd，提供智能提示，代码导航等IDE的功能
毕竟我的主要编码流程都在vscode中，如果要切换回visual studio的话会觉得比较麻烦

### 经过多次的探索，终于找到一篇合适的使用办法

使用bat脚本的方法

为了以防丢失，将网页内容全部拷贝下来。

[船长的资料室 - 花活：在Windows+VSCode+CMake+Ninja环境下设定MSVC工具集版本](https://www.tiger2doudou.com/doku.php/windows:gnu:configure_vs_code_cmake_msvc_toolset_ninja_version)

要素过多，完美死锁
这个花活儿是这样来的：目标平台限制，要用Windows+MSVC，然后希望用VS Code（方便，远程开发），有希望用Clangd诊断工具——于是就必须用ninja，因为MSVC工具链下是不能生成Clangd所需的compile_commands.json文件的。
要光是这样也就罢了，还有最后一项问题：由于MSVC 17.3的某个神秘bug，我的代码编译不过去（某些嵌套模板附近编译器崩溃，不是报错，正常报错就改了），三年没改，于是必须停留在17.2，于是就必须设定工具链版本。
如果是直接用CMake + Visual Studio，可以在执行cmake的configure时传递 -T host=x64,version=14.32.17.2这样的参数来指定工具链，但ninja则自成一套，你在cmake时可以传递上述-T参数，但是似乎只在configure时有效，下次cmake build时，ninja仍然是从环境变量里面自动检测VC工具链——默认的是最新的，所以只要机器上有17.3的编译器，ninja是不会去找17.2版本的。
cmake自带的-T参数失效之后，没有其他什么命令行参数可以直接指定MSVC工具链版本了。新版的Visual Studio中，手动使用时是通过先调用vcvarsall.bat再调用编辑器来指定MSVC版本的。在本地使用时，可以通过先调用vcvarsall.bat，再从同一个命令行启动vscode的方法变通实现，但如果是通过ssh远程开发，就连这个途径都没有了。
完美死锁.jpg
仔细分析的话，最稳定的途径仍然是通过vcvarsall.bat实现版本选择，但难就难在这玩意儿是个交互式脚本，它的生效不是通过额外的参数或者配置文件，而是依赖于它运行完了之后帮你处理过的那个cmd环境，这就让这玩意儿难以跟别的程序形成后台配合。
最后，我找到的解决方案是：自己写一个cmd脚本，假装自己是cmake.exe，然后在每次执行前给自己先加一个vcvarsall.bat的buff……
粗暴而有效.jpg
具体地：新建一个类似下面的bat文件：
msvc172-cmake-wrapper.bat
@echo off
set VSLANG=1033
set VCINSTALLDIR=C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Auxiliary\Build\
call "%VCINSTALLDIR%\vcvarsall.bat" x64 10.0.22000.0 -vcvars_ver=14.32.17.2 > NUL
 
call C:\APP\cmake-3.25.2-windows-x86_64\bin\cmake.exe %*
第一行不解释，第二行是给ninja准备的——ninja通过解析并拦截命令行输出来工作的，而中文版的vc编译器输出的是中文信息，ninja不能识别也就不拦截，得用这个环境变量设定让vc编译器输出英文信息。
第3-4行就是给自己加vcvarsall.batbuff，参数很自由，注意 > NUL是必须要有的，因为vscode的CmakeTools插件用过解析CMake的命令行输出来工作，因此必须把额外的输出屏蔽掉，否则vscode的CmakeTools插件会报错。
也就是假装自己是cmake要装得足够像。
最后一行，把所有的命令转发给真正的cmake.exe
然后，在vscode的CmakeTools插件设置中，找到“Cmake Path”设置项，将之修改为这个msvc172-cmake-wrapper.bat文件的全路径。接下来其他的东西都跟正常用cmake+ninja一样了，-T参数也按原来的样子传递，确保cmake和ninja都满意。
我没试过故意传递不一样的会怎么样。

### bat脚本的方式

[5分钟掌握cmake(15): 使用Ninja替代MSBuild - 知乎](https://zhuanlan.zhihu.com/p/667238877)

哇，大佬当天就回复我了，但是我没有看见，大佬自己写了一个脚本

[亚瑟的微风 ninja脚本](https://github.com/zchrissirhcz/cmake_examples/blob/main/9-cross-build/scripts/build/vs2022-x64-ninja.ps1)

我发现真的可以，很好用，哈哈哈哈哈，舒服了。
然后就是在这个仓库对应的目录下的readme.md
记载了一个微软的网址，里面详细介绍了如何使用PowerShell来提供Visual Studio Developer Command Prompt的环境的办法

[微软Developer PowerShell](https://learn.microsoft.com/en-us/visualstudio/ide/reference/command-prompt-powershell?view=vs-2022)

因此，我这边使用的脚本config.bat内容如下

```bat
call "D:\Program Files\Microsoft Visual Studio\2022\Professional\VC\Auxiliary\Build\vcvarsall.bat" x64

set BUILD_DIR=build\build\x64-windows-clang-debug
set CC=cl
set CXX=cl
cmake ^
    -S . ^
    -B %BUILD_DIR% ^
    -G "Ninja Multi-Config" ^
    -D CMAKE_EXPORT_COMPILE_COMMANDS=ON
```

当然，根据以上分析可以使用如下powershell脚本

### pwsh脚本的方式

```powershell

#启动 Visual Studio 开发者命令行环境
. "D:\Program Files\Microsoft Visual Studio\2022\Professional\Common7\Tools\Launch-VsDevShell.ps1" `
    -SkipAutomaticLocation -Arch amd64 -HostArch amd64

#配置 CMake 构建环境
cmake -S . -B build\build\x64-windows-clang-debug `
    -G "Ninja Multi-Config" `
    -D CMAKE_EXPORT_COMPILE_COMMANDS=ON

```

> 其实第一行设置了环境变量之后，那么之后再跑这个脚本也不用设置了
{: .prompt-tip }

### 一些注意事项



至于我为什么把compile_commands.json放在build\build\x64-windows-clang-debug下，是因为这样跑脚本的时候，不会污染build目录，build目录是vscode的插件cmake tools命令面板的cmake:Configure默认输出目录

当然，也可以修改cmake tools插件的设置，generator直接改成ninja，那么跑上面的命令来生成json的时候，就不会再生成sln这种解决方案文件了，我由于还需要用到sln打开解决方案附加到进程调试。所以最后选择了这样使用脚本的方式

另外，generator改成ninja之后，cmake:build命令也是可以编译通过的，而且编译器依旧是msvc的cl.exe

### 使用Clang Power Tool 在visual studio中导出compile_commands.json

这个需要使用最新版的visual studio，但是项目使用cuda导致不能使用最新的visual studio 所以目前用不了
