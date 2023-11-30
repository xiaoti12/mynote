# Vim

## 编辑类

撤销：按一下`u`

## 搜索类

进入搜索模式：按`/`，回车确定后按`n`跳到下一个

不区分大小写：在命令模式输入`:set ignorecase`

## 浏览类

下一页：`Ctrl + F (Forword)`

上一页：`Ctrl + B (Back)`

移动到最后：`G`

# tmux

新建会话：`tmux new -s [session-name]`

离开会话：`Ctrl+b -> d`或`tmux detach`

查看会话：`tmux ls`

进入会话：`tmux attach -t [session-name]`

删除会话：`tmux kill-session -t [session-name]`

# Crontab

编辑：crontab -e

列出：crontab -l

格式： * * * * * command（前五个操作符分别代表分、时、日、月、周）

其中操作符：

- *：取值范围内所有数字
- /num：每过多少数字
- num1-num2：从num1到num2
- num1,num2：num1和num2

例：

- 每1小时（整点）执行一次	0 */1 * * * command
- 每周三的22时执行一次 0 22 * * 3 command

# netstat

用于显示TCP、UDP的端口、进程使用情况

常用：`netstat -tunlp`（后可跟grep 查询特定端口）

- -t：显示TCP相关
- -u：显示UDP相关
- -n：以网络IP地址代替名称
- -l：列出处于Listen状态的服务
- -p：显示进程名
- -a：显示所有socket

# 文件夹大小

- ll -h：以直观单位（MB/GB）显示目录下文件大小
- du -h --max-depth=1 PATH：显示目录下文件和文件夹大小
- df -h：显示挂载文件系统的可用空间和使用情况

# 显卡工作

- nvidia-smi：显卡使用情况
- watch -n TIME nvismi：每TIME秒刷新一次显卡使用情况

# 解压缩

解压：`tar -xf file.tar` `tar -xzf file.tar.gz`

压缩：`tar -czf file.tar.gz file_folder  `

- -c：压缩 ←→ -x：解压
- -v：显示详细过程
- -f：指定文件
- -z：使用gzip
- -j：使用bzip2
- --use-compress-program=pigz：使用pigz进行多线程处理

