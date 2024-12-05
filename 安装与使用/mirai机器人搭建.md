# Mirai搭建

## 配置mirai环境

`apt install default-jre`安装Java，版本需要≥11（可通过`java -version`查看）。

下载`mcl`（实际是电脑已经下载好的文件复制过去，包括http插件和QQ配置），解压后在目录使用`.\mcl`启动（需要给执行权限）。

## 配置graia环境

使用pip安装graia，需要注意的是最好安装如下指定版本：

```
graia-application-mirai==0.19.0
graia-broadcast=0.8.11
graia-scheduler=0.0.4
```

## 后台运行MCL

使用`tmux new -s mirai`命令新建一个tmux窗口，启动MCL。

# Sagiri机器人

项目地址https://github.com/SAGIRI-kawaii/sagiri-bot

## 下载对应python包

`pip install -r requirements.txt`安装需要的库。经过测试，至少`opencv-python`和`graiax-silkcoder`两个包是不需要的。

## 运行与管理

新建一个tmux窗口，运行目录下的`main.py`即可。

可用功能可查看https://github.com/SAGIRI-kawaii/sagiri-bot/blob/master/docs/functions.md

开关功能可使用`setting -set %command%=%new_value%`命令，需要注意的是需要相关权限才能使用，可以在`config.yaml`中将`HostQQ`设为自己的QQ。

# graia-python机器人开发

## 变量类型

MessageChain

- MessageChain.create：创建一个消息链。
- MessageChain.has：判断特定元素是否存在在消息链。
- MessageChain.get：获取特定元素
    - msg.get(Image) ：消息中的图片，List[Image]
    - msg.get(Source)[0]：获取消息的发送者ID
- MessageChain.asDisplay：获取字符串消息

Element

- Plain：字符串，可以`[Plain('xxx')]`形式创建字符串
- Image：`Image.fromLoacalFile(filepath)`从本地获取图片

## 函数

app.sendGroupMessage(Group, MessageChain (, quote))：向群里发送消息

app.messageFromId(Source)：根据Source从缓存中获取历史消息
