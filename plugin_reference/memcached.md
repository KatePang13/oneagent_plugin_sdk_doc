# memcached

采集 memcached stats ，关于 stats 的 具体介绍，请参考 [memcached_protocol](https://github.com/memcached/memcached/blob/master/doc/protocol.txt)

支持以下三种通信协议：

- TCP - 简单文本行
- Unix sockets - 简单文本行
- UDP - 带二进制头的文本行

所有需要的配置信息从解析 进程命令行 中获取。

 `class ruxit_memcached.MemcachedPlugin(**kwargs)`