在88机器中的流量捕获系统中，通过Go编写的程序需要make编译生成，因此无法直接使用VSCODE调试，尝试使用GDB调试。

打好断点后，发现无法查看变量的值，报错如下：

![image-20230221174630947](C:\Users\xiaoti\AppData\Roaming\Typora\typora-user-images\image-20230221174630947.png)

尝试修改makefile里的`CFLAGS`，以及使用`make debug -j8`仍然是相同错误，搜索后先安装`yum install yum-utils`，然后按照报错信息里的`debuginfo-install xxx`，此时报错

![image-20230221175527336](C:\Users\xiaoti\AppData\Roaming\Typora\typora-user-images\image-20230221175527336.png)

搜索结果说是要在`/etc/yum.repos.d/CentOS-Debuginfo.repo`中将enable设为1，但尴尬的是没有这个文件…于是自己创建了一个，然后运行`yum install -y kernel-debuginfo-$(uname -r)`，结果此时提示空间不足。运行`df -h`，排查发现`/root/code`这个文件夹占了很大空间（几乎一半），只好挪到`/home`目录下。

重新安装后，此时`debuginfo-install glibc`，然后`debuginfo-install`安装对应依赖，结果还是显示optimized out...

搜索后是需要在go build的参数中加上`-gcflags "all=-N -l"`，在makefile中搜索到

```makefile
ifdef NFF_GO_DEBUG
# Flags to build Go files without optimizations
export GO_COMPILE_FLAGS += -gcflags=all='-N -l'
endif

$(EXECUTABLES) : % : %.go
	go build $(GO_COMPILE_FLAGS) -tags "${GO_BUILD_TAGS}" $< $(COMMON_FILES)
```

但是在make编译时go build未加上参数，于是直接改成

```makefile
$(EXECUTABLES) : % : %.go
# go build $(GO_COMPILE_FLAGS) -tags "${GO_BUILD_TAGS}" $< $(COMMON_FILES)
	go build -gcflags "-N -l" -tags "${GO_BUILD_TAGS}" $< $(COMMON_FILES)
```

此时编译后gdb可以正常调试