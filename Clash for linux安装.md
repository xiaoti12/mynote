下载二进制文件，服务器为AMD64架构，解压后提权，移动到`/usr/local/bin`目录下

```shell
gunzip clash-linux-amd64-v1.10.6.gz
mv clash-linux-amd64-v1.10.6.gz /usr/local/bin/clash
chmod u+x /usr/local/bin
```

启动clash，第一次启动时会在`~/.config/clash`目录下载`config.yaml`和`Country.mmdb`，可以自行下载并替换

> `Country.mmdb` 文件利用 GeoIP2 服务能识别互联网用户的地点位置，以供规则分流时使用

此外还可以使用`clash-dashboard`服务，下载到特定目录后修改或添加`config.yaml`目录里的`external-ui`和`secret`项，通过URL`ip:external-port/ui`访问。不过实测网页为一片空白，不知道出了什么问题。

由于一些程序和shell不会走系统代理，因此下载`proxychains`，并修改配置文件

```shell
apt install proxychains-ng
vim /etc/proxychains.conf
#####################
[ProxyList]
# add proxy here ...
# meanwile
# defaults set to "tor"
socks5 	127.0.0.1 7891
```

此时在shell里输入命令前加上`proxychains`就能让该命令走代理了

```shell
proxychains curl www.google.com
[proxychains] config file found: /etc/proxychains4.conf
[proxychains] preloading /usr/lib/x86_64-linux-gnu/libproxychains.so.4
[proxychains] DLL init: proxychains-ng 4.14
[proxychains] Strict chain  ...  127.0.0.1:7891  ...  www.google.com:80  ...  OK
<!doctype html><html itemscope="" itemtype="http://schema.org/WebPage" lang="zh-HK"><head><meta content="text/html; charset=UTF-8" http-equiv="Content-Type">
```

同时可以在conf文件中反注释掉quiet_mode ，这样不会每个请求都有提示信息
