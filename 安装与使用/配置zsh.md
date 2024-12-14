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
### 安装历史自动补全
`git clone --depth=1 https://github.com/zsh-users/zsh-autosuggestions.git ${ZSH_CUSTOM:-${ZSH:-~/.oh-my-zsh}/custom}/plugins/zsh-autosuggestions`

`~/.zshrc`的`plugins`添加项：`zsh-autosuggestions`
### 安装shell高亮提醒
`git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting

`~/.zshrc`的`plugins`添加项：`zsh-syntax-highlighting`