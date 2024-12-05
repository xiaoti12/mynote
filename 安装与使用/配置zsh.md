如果没有zsh，安装：apt install zsh

切换当前用户默认shell为zsh：chsh -s `which zsh`

安装oh-my-zsh：

```shell
sh -c "$(curl -fsSL https://raw.github.com/robbyrussell/oh-my-zsh/master/tools/install.sh)" # curl
# o
sh -c "$(wget https://raw.github.com/robbyrussell/oh-my-zsh/master/tools/install.sh -O -)" # wget
```

配置完成：

![image-20240602202520599](C:\Users\xiaoti\AppData\Roaming\Typora\typora-user-images\image-20240602202520599.png)

## 插件
安装历史自动补全：`git clone --depth=1 https://github.com/zsh-users/zsh-autosuggestions.git ${ZSH_CUSTOM:-${ZSH:-~/.oh-my-zsh}/custom}/plugins/zsh-autosuggestions`

在`~/.zshrc`中修改：

```shell
plugins=(
	git
	zsh-autosuggestions // 添加
)
```

