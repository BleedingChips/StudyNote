可能需要的宏变量或宏
* ModuleRules::ModuleDirectory`.build.cs`的路径
* `UnrealTargetPlatform ReadOnlyTargetRules::Platform`
    ```c#
    public enum UnrealTargetPlatform
	{
		Unknown,
		Win32,
		Win64,
		Mac,
		XboxOne,
		PS4,
		IOS,
		Android,
		HTML5,
		Linux,
		AllDesktop,
		TVOS,
		Switch,
	}
    ```

* `UnrealTargetConfiguration ReadOnlyTargetRules::Configuration`
    ```c#
    public enum UnrealTargetConfiguration
	{
		Unknown,
		Debug,
		DebugGame,
		Development,
		Shipping,
		Test,
	}
    ```
* `PublicAdditionalLibraries`
    ```c#
     PublicAdditionalLibraries.AddRange(
            new string[]
            {
                //libhere
            }
            );
    ```
---

鉴于某种蛋疼的设置，无论你的程序是从Debug编译还是从Develop编译，你的第三方库尽量都要直接使用release下的。

---
编译时遇到的问题：

* `error LNK2038: 检测到“_ITERATOR_DEBUG_LEVEL”的不匹配项: 值“2”不匹配值“0”`
    DEBUG模式下，在命令行中添加 `/D_ITERATOR_DEBUG_LEVEL=0` 

* `error LNK2038: 检测到“RuntimeLibrary”的不匹配项: 值“MDd_DynamicDebug”不匹配值“MD_DynamicRelease”`
    在项目属性->C/C++->代码生成->运行库->多线程DLL/MD

* `warning LNK4075: 忽略“/EDITANDCONTINUE”(由于“/OPT:LBR”规范)`
    在项目属性->C/C++->常规->调试信息格式->程序数据库(/Zi)

* `error LNK2001: 无法解析的外部符号 __CheckForDebuggerJustMyCode`
    项目属性->C/C++->常规->SupportJustMyCodeDebugging->no

* `unrecognized flag '-Ot' in 'p2'`
	项目属性->c/c++->优化->优化->Disabled(/Od)

* `fatal error C1900: Il mismatch between 'P1' version '20080116' and 'P2' version '20070207'`
	项目属性->c/c++->优化->全局程序优化（while Program Optimization）-> No
	
