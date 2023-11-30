## 安装命令

- go install：下载第三方**二进制文件**
- go get：下载第三方**依赖包**（相当于**源代码**）

## 路径

- GOROOT：go的安装目录，存放内置包和工具
- GOPATH：go指定的工作空间，保存项目代码和第三方依赖包
    - bin：存放编译后的可执行文件，`go install`命令生成的文件存放处
    - src：存放源代码，`go get`命令下载的代码存放处
    - pkg：存放编译后生成的文件

## Go module

### GO111MODULE

`go env`环境变量中`GO111MODULE`有三个可选值：

- off：禁用模块支持，go从GOPATH和vendor文件夹寻找依赖包
- on：开启模块支持，go根据`go.mod`下载依赖
- auto：项目在`GOPATH/src`外且根目录有`go.mod`文件时，开启模块支持

### 安装依赖

在项目目录下运行`go mod init`，初始化生成`go.mod`文件。此时执行`go get`命令下载的第三方包会保存到`GOPATH/pkg/mod`目录下。

在引用同目录的库时，需要将模块名作为路径前缀。结构树如下

```
.
├── api
│   └── api.go
├── go.mod
├── main.go
```

此时在`main.go`中引用`api`时，需要使用`module_name/api`

