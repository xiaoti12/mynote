# 底层数据类型

## Ziplist（压缩列表）

在6.0以前，是列表和Hashmap的底层实现之一。在6.0以后，是listpack的组成部分。

### 包含字段

![img](https://pdai.tech/images/db/redis/db-redis-ds-x-6.png)

- zlbytes：4字节，整个ziplist所占内存字节数
- zltail：4字节，ziplist最后一个entry的偏移量
- zllen：2字节，整个ziplist中entry的数量。如果超过2字节范围（65535），则需要实际遍历得到长度
- zlend：终止字符，值为`0xFF`。ziplist保证entry的首字节不会是`0xFF`

## entry结构

一般结构：`prevlen`+`encoding`+`entry-data`。当存储int型时，不使用`entry-data`

- prevlen：前一个entry的长度，向前遍历时使用

    - 长度<254时，prevlen为1字节
    - 长度>=254时，prevlen为5个字节：`0xFE`+实际长度

- encoding：根据data类型为string还是int，以及数据长度表示

    - 前两位为11：表示存储int，后跟的字节表示int的值
    - 前两位为00/01/10：表示存储string，后跟的字节/比特表示string长度

    