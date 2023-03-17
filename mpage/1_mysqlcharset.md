# MySQL字符集引发的错误

前几天帮同事处理一个棘手的事情，问题是这样的：  
无论在客户机用哪个版本的mysql客户端连接服务器，发现只要服务器端设置了
```properties
character-set-server = utf8
```
然后
```
character_set_client、 character_set_connection、character_set_results
```
就始终都是和服务器端保持一致了，即便在mysql客户端加上选项
```properties
--default-character-set=utf8
```

还是不行，除非连接进去后，再手工执行命令

```properties
set names latin1
```

这样才会将client、connection、results的字符集改过来。  
经过仔细对比，最终发现引起错误的的地方是，服务器端设置了另一个选项：
```properties
skip-character-set-client-handshake
```

文档上关于这个选项的解释是这样的：

```properties
--character-set-client-handshake

Don't ignore character set information sent by the client. To ignore client information and use the default server character set, use --skip-character-set-client-handshake; this makes MySQL behave like MySQL 4.0
```

这么看来，其实也是有好处的。比如启用 skip-character-set-client-handshake 选项后，就可以避免客户端程序误操作，使用其他字符集连接并写入数据，从而引发错误。