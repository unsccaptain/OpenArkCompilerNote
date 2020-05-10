# 一 编译与调试
## 源代码下载
[OpenArkCompiler](https://gitee.com/harmonyos/OpenArkCompiler)

## 编译环境
&emsp;&emsp;在源码目录下doc/cn/DevelopmentPreparation.md有详细描述，经过笔者测试，按照它的流程做一遍基本没有问题。Ubuntu使用的18.04，clang和ninja的版本可以选择最新的版本。jdk最好按照它的说法选择openjdk-8.0，不然java2jar这个脚本可能不兼容。

## 源代码编译
&emsp;&emsp;具体方法在源码目录下的doc/cn/DeveloperGuide.md中有所描述。按照它的做法也基本没有问题。因为很多文件还没有开源，所以编译的时间也比较的短。目前生成的只有maple和maplegen这两个可执行文件。

## 编译HelloWorld.java
&emsp;&emsp;因为目前只有java前端（/src/mplfe），所以只能写一个java文件来测试了。在源码目录的doc/cn/DeveloperGuide.md文件中描述了怎样去用它的脚本去自动编译HelloWorld，不过如果我们想要调试并研究内在的原理的话，还是要一步一步的去手动执行程序的。这一章我们只需要将java文件编译出来，其中生成的文件，执行的程序以及执行的参数是什么含义下一章在说。  
&emsp;&emsp;首先下载libcore的jar，它已经给出了预编译的jar包[https://gitee.com/mirrors/java-core/](https://gitee.com/mirrors/java-core/)。然后我们用如下命令生成它的mplt文件。
```
jbc2mpl -injar java-core.jar -out libjava-core
```
&emsp;&emsp;它会在目录下生成两个文件，一个是libjava-core.mpl，另一个是libjava-core.mplt。然后我们在目录中创建一个HelloWorld.java的文件，随便写点代码。首先执行java2jar脚本：
```
java2jar HelloWorld.jar java-core.jar HelloWorld.java
```
&emsp;&emsp;注意一下，如果下载的是OpenJDK8，那么没有任何问题，如果是OpenJDK11以及更新，会提示bootclasspath参数不对，因为11已经不支持了，需要在java2jar中将bootclasspath改成sourcepath，然后就可以了。执行完后会在当前目录生成HelloWorld.class和HelloWorld.jar这两个文件。下一步执行jbc2mpl这个程序：
```
jbc2mpl -mplt libjava-core.mplt -injar HelloWorld.jar -o HelloWorld.mpl
```
&emsp;&emsp;执行完后会在目录下生成HelloWorld.mpl和HelloWorld.mplt。然后执行maple，
```
maple --infile HelloWorld.mpl --run=me:mpl2mpl:mplcg --option="--O2:--emitVtableImpl:--fpic"
```
&emsp;&emsp;这样目录下就会生成HelloWorld.VtableImpl.S，其中是HelloWorld.java对应的aarch64的机器码。目前还不知道怎样去执行它。

## 调试
&emsp;&emsp;笔者使用的是VSCode来调试。首先下载插件CodeLLDB，然后在源码目录下创建.vscode目录，在其中创建文件launch.json，输入配置参数：
```
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "OpenArk Debug",
            "type": "lldb",
            "request": "launch",
            "program": "${workspaceFolder}/output/bin/maple",
            "args": "--infile ../libjava-core/HelloWorld.mpl --run=me:mpl2mpl:mplcg --option=::--fpic",
            "cwd": "${workspaceFolder}",
            "environment": [],
            "preLaunchTask": "",
        }
    ]
}
```
&emsp;&emsp;然后在代码里下个断点F5就可以开始调试了。