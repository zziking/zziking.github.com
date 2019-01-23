---

layout: post
title:  windows lib 静态库转 dll 动态库
fullview: true
category: windows

---



lib 静态库转 dll 动态库需要有一个链接的过程，将其依赖的外部符号与对应的库链接起来，Visual Studio 中的 `link` 工具提供了此功能。



在 windows 的命令行下，需要先运行 `vcvars64.bat` 初始化编译环境，才能使用 `link` 工具，以 `Visual Studio 2017` 为例， `vcvars64.bat`  位于 `C:\Program Files(x86)\Microsoft Visual Studio\2017\Community\VC\Auxiliary\Build` 目录下，可通过以下命令初始化编译环境：

```bash
set INSTALLPATH=
if exist "%programfiles(x86)%\Microsoft Visual Studio\Installer\vswhere.exe" (
  for /F "tokens=* USEBACKQ" %%F in (`"%programfiles(x86)%\Microsoft Visual Studio\Installer\vswhere.exe" -version 15.0 -property installationPath`) do set INSTALLPATH=%%F
)
echo INSTALLPATH is "%INSTALLPATH%"
REM Save current dir for later
pushd %CD%
if NOT "" == "%INSTALLPATH%" (
  call "%INSTALLPATH%\VC\Auxiliary\Build\vcvars%ARCH:~-2%.bat"
) else (
  goto ERROR_NO_VS15
)
:WORK
REM Retrieve the working dir and proceed
popd
echo Doing work in %CD%
// ......
goto END

:ERROR_NO_VS15
echo Visual Studio 2017 Tools Not Available!

:END
echo Processing ends.
```



编译环境初始化之后，可通过以下命令来链接 lib 库，生成 dll：

```bash
link /OUT:"xxxx.dll" /IMPLIB:"xxxx_imp.lib" /DYNAMICBASE "Ws2_32.lib" "CRYPT32.LIB" /DEBUG:FASTLINK /DLL /MACHINE:x64 /ERRORREPORT:PROMPT  /NOLOGO /LIBPATH:"依赖的第三方库的lib路径1" /LIBPATH:"依赖的第三方库的lib路径2" /DEF:"xxxx.def" xxxx.lib

```



说明：

- `/OUT` : 生成的 dll 路径

- `/IMPLIB` : 生成的 dll 的引入库，相当于头文件，定义了相关导出函数

- `/DYNAMICBASE` : 可选，静态库依赖的三方库和系统库，多个库用空格隔开

- `/MACHINE` : x86 / x64

- `/LIBPATH` : 可选，依赖的第三方库的查找路径

- `xxx.lib` : 要链接的 lib 静态库，可多个

- `/DEF` : dll 导出函数，格式如下

  ```
  LIBRARY xxxx //dll名称
  
  EXPORTS	     //导出的函数名列表
  	function1
  	function2
  	.....
  ```

  

