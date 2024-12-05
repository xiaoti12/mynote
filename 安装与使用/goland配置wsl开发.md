1. 下载go包（以.122版本为例）`wget https://go.dev/dl/go1.22.9.linux-386.tar.gz`

    官网地址：https://go.dev/dl/

2. 解压到`/usr/local`：`tar -C /usr/local -xzf go1.22.9.linux-386.tar.gz`

3. 在`~/.zshrc`中添加如下环境变量（配置文件视shell而变化）

    ```shell
    export GOPATH=/root/go #或者 /home/username/go
    export GOROOT=/usr/local/go
    export PATH=$PATH:$GOROOT/bin:$GOPATH/bin
    ```
	
	然后`source ~/.zshrc`
	
3. 在goland中新建项目，选择wsl中的目录和GOROOT

## 无法调试
发现在goland中调试程序时，无法进入函数，有如下报错
```
2024-11-15T22:05:17+08:00 warning layer=rpc Listening for remote connections (connections are not authenticated nor encrypted)
```
根据[某博客](https://www.sulinehk.com/post/fix-goland-debug-hangs-on-wsl2-project/)所说，是dlv在wsl的mirrored网络下无法运行，有两种解决方法：
- 取消mirror模式（无法直接使用主机代理）
- 重新编译dlv
结果从2023.2升级到2024.3后没这个问题了
但产生了新问题：
调试时，程序直接运行完毕，而不是停在断点上。
尝试了如下措施：
- 更换wsl网络模式
- 确认GOROOT和GOPATH
- 在edit custom properties中更换dlv.path（wsl中的path会报错，但windows的可以）
仍然无法解决，但vscode可以…