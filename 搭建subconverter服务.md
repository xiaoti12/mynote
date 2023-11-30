运行如下命令

```shell
cd /root #可自行选择路径
wget https://github.com/tindy2013/subconverter/releases/xxx #可自行选择最新release
tar -zxvf subconverter_linux64.tar.gz # 解压
cd subconverter # 进入文件夹
```

此时已经下载好程序，开始进行配置

测试发现0.7.1版本无法正常提供服务，回退到0.6.4后可顺利运行

在配置文件`pref.ini`中较为重要的有如下项，具体可见https://github.com/tindy2013/subconverter/blob/master/README-cn.md

```ini
;根据关键词或正则排除的节点
exclude_remarks=(到期|剩余流量|时间|官网|产品|) 

;规则集，将IP和域名指向策略组
[rulesets]
;表示引用本地的rulesets.txt文件（文件格式见下条规则）
ruleset=!!import:snippets/rulesets.txt

;表示引用本地rules/Download.list规则。并将此规则指向[proxy_group]所设置的🎯全球直连 策略组
ruleset=🎯全球直连,rules/Download.list

;创建策略组，[] 前缀后的文字将被当作引用策略组
[proxy_groups]
;使用本地的snippets/groups.txt文件
custom_proxy_group=!!import:snippets/groups.txt

;表示创建一个叫日本延迟最低的url-test策略组,并向其中添加名字含'日','JP'的节点，每隔300秒测试一次，测速超时为5s
custom_proxy_group=日本延迟最低`url-test`(日|JP)`http://www.gstatic.com/generate_204`300,5

;表示创建一个叫JP的select策略组，并向其中添加名字含'沪日','日本'的节点，引用上述日本规则组
custom_proxy_group=JP`select`沪日`日本`[]日本延迟最低

;设置接口别名
[aliases]
;可将/sub?target=clash&url=简化为/clash?url=
/clash=/sub?target=clash

```

