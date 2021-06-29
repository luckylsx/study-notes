## vscode golang 环境搭建 

在 vscode 的使用过程中,自己也是从小白到慢慢熟练vscode的日常开发，现在已经很少使用goland 等IDE，平时基本都在使用vscode。本篇文章主要记录自己vscode等一些探索，方便大家快速使用vscode做golang 开发。

## golang 插件安装

首先需要安装golang 相关插件包，Mac 环境下 按快捷键 command+shift+p（windows 环境下 ctrl+shift+p）打开控制面板，输入 "GO: Install/Update" 选中 "GO: Install/Update Tools" 点击进入，全选，点击ok 自动安装。主意此方法部分包需要科学上网，否则会出现安装失败问题。

**vscode 配置科学上网 proxy**

进入设置（Settings），搜索 "Http: Proxy"，配置代理即可。

如果无法科学上网，安装失败，可更改go mod 走国内代理即可，详细方法，可参照此文：https://blog.csdn.net/qq_41065919/article/details/107710144

## code-runner 运行项目

Code Runner支持了 Node.js, Python, C++, Java, PHP, Perl, Ruby, Go等超过40种的语言。下面，我们就来看看如何来玩转Code Runner，提高你的效率。

首先需要安装code-runner插件，在插件库搜索 code-runner，并安装。

**运行golang的几种情况**

### 单文件运行和无需指定配置文件项目

直接找到main文件所在目录，点击main文件，无需任何配置，直接点击vscode右上角运行按钮即可直接运行。

### 运行需要指定配置的项目 - 单main文件

需要在项目目录下添加.vscode的文件夹，并添加settings.json配置文件，添加配置信息如下：

#### 相对路径配置

```
{
    "code-runner.executorMap": {
        "go": "cd  $dir && go run . -conf ../../configs",
    }
}
```

可指定如上配置：$dir 为系统变量，当打开main文件之后，点击运行按钮时，vscode 会自动进入main 文件所在目录，并根据 -conf 指定的配置文件，启动项目。

此方式也适合多项目启动，如果我们工作区下面有多个项目，每个服务都需要启动，这种方式也比较方便，只是每次启动服务时，都需要打开相应的main文件。

这种相对路径的配置方法每次启动都需要打开相应的main文件，多次一举，也会很不方便，下面的绝对目录方式就可以完美解决。单不适合多服务都需要启动模式。

#### 绝对路径配置

```
{
    "code-runner.executorMap": {
        "go": "cd  /Users/limuzi/go/src/{grogram}/cmd/graphql-layout && go run . -conf ../../configs",
    },
    "code-runner.fileDirectoryAsCwd": true
}
```

grogram 替换成自己真实项目名称即可。此种方式比较简单，在打开任意go文件时，都可直接点击右上角运行按钮，即可启动程序运行。

## debug

### 单main文件

可直接点击 Run and Debug 按钮，开启调试

### 调试项目

首先需要安装 div依赖包

安装方式：

```
go get -u github.com/go-delve/delve/cmd/dlv
```

调试项目是，需要在项目目录.vscode文件夹下，添加launch.json 配置。

```
{
    // Use IntelliSense to learn about possible attributes.
    // Hover to view descriptions of existing attributes.
    // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "name": "debug order",
            "type": "go",
            "request": "launch",
            "mode": "debug",
            "program": "${workspaceFolder}/app/order/cmd/server",
            "cwd": ".",
            "args": ["-conf", "${workspaceFolder}/app/order/configs/config.yaml"],
            "env": {
                "RUN_ENV":"dev"
            },
            "dlvLoadConfig": {
                "followPointers": true,
                "maxVariableRecurse": 1,
                "maxStringLen": 5000,
                "maxArrayValues": 64,
                "maxStructFields": -1
            }
        },
        {
            "name": "debug admin",
            "type": "go",
            "request": "launch",
            "mode": "debug",
            "program": "${workspaceFolder}/app/admin/cmd/server",
            "cwd": ".",
            "args": ["-conf", "${workspaceFolder}/app/admin/configs/config.yaml"],
            "env": {
                "RUN_ENV":"dev"
            },
            "dlvLoadConfig": {
                "followPointers": true,
                "maxVariableRecurse": 1,
                "maxStringLen": 5000,
                "maxArrayValues": 64,
                "maxStructFields": -1
            }
        }
    ]
}
```

相关属性介绍：

* name: 调试界面下拉选择项的名称
* type: 设置为go无需改动，是 vs code 用于计算调试代码需要用哪个扩展
* mode: 可以设置为 auto, debug, remote, test, exec 中的一个
* program: 调试程序的路径（绝对路径）
* env: 调试时使用的环境变量。例如:{ "ENVNAME": "ENVVALUE" }
* envFile: 包含环境变量文件的绝对路径，在 env 中设置的属性会覆盖 envFile 中的配置
* args: 传给正在调试程序命令行参数数组 可指定配置文件 如：["-conf", "${workspaceFolder}/app/admin/configs/config.yaml"]
* showLog: 布尔值，是否将调试信息输出
* logOutput: 配置调试输出的组件（debugger, gdbwire, lldbout, debuglineerr, rpc）,使用,分隔， showLog 设置为 true 时，此项配置生效
* buildFlags: 构建 go 程序时传给 go 编译器的标志
* remotePath: 远程调试程序的绝对路径，当 mode 设置为 remote 时有效
* dlvLoadConfig: 调试打印信息过长无法显示全部时，可添加此配置

dlvLoadConfig 相关说明

* maxStringLen: 显示的字符串最大长度



## 提高开发效率的小方法：

### 变量命名方式转换

安装 Change-case插件

打开控制面板, 输入change 即可快速将变量转换为选择类型：

* Change Case snake: 转换为下划线类型
* Change Case camel: 转换为驼峰命名
....

也可以选择 Change Case Commands 来选择需要的类型。

### struct 快速填充

mac 环境下 command+. 快速填充struct

### struct 快速添加tag

mac 环境下 command+shift+p 打开控制面板，输入 Add Tags,选择 Add Tags To Struct Fields

### struct 快速删除tag

mac 环境下 command+shift+p 打开控制面板，输入 Remove Tag,选择 remove Tags From Struct Fields

### 自动生成单元测试

mac 环境下 command+shift+p 打开控制面板，输入 Generate Unit Tests,选择 Generate Unit Tests For File;即可自动为打开的文件生成单元测试。


### 快速实现接口实现

* impl 命令

使用方法：

进入到类型实现所在目录

```
impl 'r *queryResolver' generated.QueryResolver
```

* r 为类型接收到形参
* queryResolver 为类型名
* generated.QueryResolver 接口 interface, generated 为包名，QueryResolver 为 interface 名称

方法会自动生成到终端：

```
func (r *queryResolver) GetOrder(ctx context.Context, id int) (*common.OrderInfo, error) {
        panic("not implemented") // TODO: Implement
}

func (r *queryResolver) GetAddress(ctx context.Context, id int) (*common.AddressInfo, error) {
        panic("not implemented") // TODO: Implement
}
```

## 好用的插件

个人在使用过程中总结了以下好用的插件：

* Change-case 变量类型转换
* Remote-ssh ssh客户端
* Gitlens
* Git Graph
* Git History
* Todo Tree
* Docker
* Live Share 很好的一个团队协作工具
* MySQL cweijan.vscode-mysql-client2 好用的数据库客户端
* CodeStream: GitHub, GitLab, Bitbucket PRs and Code Review 协作工具
