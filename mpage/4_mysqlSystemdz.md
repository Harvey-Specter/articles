## 利用systemd管理MySQL单机多实例
> 用systemd代替mysqld_multi管理单机多实例，也很方便。

有时候，我们需要在单机环境下跑多实例。在以前，一般是习惯用mysqld_multi来跑多实例。不过从CentOS 7开始引入systemd作为新的系统管理器后，用它来管理多实例也是很方便的。

本文我们以RPM/YUM方式安装后的MySQL为例，介绍如何用systemd管理多实例。

以RPM/YUM方式安装完后，会生成systemd服务文件 `/usr/lib/systemd/system/mysqld.service`，可以看到其中有两行：

```properties
ExecStartPre=/usr/bin/mysqld_pre_systemd
ExecStart=/usr/sbin/mysqld $MYSQLD_OPTS
```

在编辑 `/usr/bin/mysqld_pre_systemd` 时能看到有 `--defaults-group-suffix` 选项，它就是用于构建多实例的了。接下来，只需要简单的照猫画虎就行。

复制MySQL服务文件 `/usr/lib/systemd/system/mysqld.service` 到一个新文件，例如 `/usr/lib/systemd/system/greatsql@.service`，加上一个 @ 符号，只需修改上述提到的2行内容，其他内容照旧即可：

    # vim /usr/lib/systemd/system/greatsql@.service
    ...
    ExecStartPre=/usr/local/GreatSQL-8.0.25-15-Linux-glibc2.28-x86_64/bin/mysqld_pre_systemd %I
    ExecStart=/usr/local/GreatSQL-8.0.25-15-Linux-glibc2.28-x86_64/bin/mysqld --defaults-group-suffix=@%I $MYSQLD_OPTS

在这里我改成GreatSQL的二进制路径，如果缺少`mysqld_pre_systemd` 文件，可以从 `/usr/bin/mysqld_pre_systemd` 复制一份，然后做些微调即可.

接下来编辑修改 /etc/my.cnf 配置文件，在原来的基础上增加多实例相关的几个片段，例如：

```
[mysqld@mgr01]
datadir=/data/GreatSQL/mgr01
socket=/data/GreatSQL/mgr01/mysql.sock
port=3306
server_id=103306
log-error=/data/GreatSQL/mgr01/error.log
group_replication_local_address= "127.0.0.1:33061"

[mysqld@mgr02]
datadir=/data/GreatSQL/mgr02
socket=/data/GreatSQL/mgr02/mysql.sock
port=3307
server_id=103307
log-error=/data/GreatSQL/mgr02/error.log
group_replication_local_address= "127.0.0.1:33071"

```

然后执行命令 systemctl daemon-reload 重新加载systemd服务，即可识别到这些新增加的服务列表了：

    $ systemctl -l

    ... greatsql@mgr01.service loaded active running GreatSQL Server... greatsql@mgr02.service loaded active running GreatSQL Server... greatsql@mgr03.service loaded active running GreatSQL Server... ... 

现在可以直接执行类似下面的命令启停多实例服务：

    $ systemctl start greatsql@mgr01

现在可以在单机环境下很方便的管理多实例服务了。

