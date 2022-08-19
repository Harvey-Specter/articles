## systemd を使用して MySQL 単一マシンの複数インスタンスを管理する
> mysqld_multi の代わりに systemd を使用して、単一のマシンで複数のインスタンスを管理することも非常に便利です。

場合によっては、単一マシン環境で複数のインスタンスを実行する必要があります。 以前は、mysqld_multi を使用して複数のインスタンスを実行するのが一般的でした。 しかし、CentOS 7 から新しいシステム マネージャーとして systemd が導入された後、それを使用して複数のインスタンスを管理することも非常に便利です。

この記事では、RPM/YUM モードでインストールされた MySQL を例として、systemd を使用して複数のインスタンスを管理する方法を紹介します。

RPM/YUM モードでインストールすると、systemd サービス ファイル `/usr/lib/systemd/system/mysqld.service` が生成されます。

```properties
ExecStartPre=/usr/bin/mysqld_pre_systemd
ExecStart=/usr/sbin/mysqld $MYSQLD_OPTS
```

`/usr/bin/mysqld_pre_systemd` を編集すると、複数のインスタンスを構築するために使用される `--defaults-group-suffix` オプションが表示されます。 次に、猫に続いて虎を描くだけです。

MySQL サービス ファイル `/usr/lib/systemd/system/mysqld.service` を新しいファイル (例: `/usr/lib/systemd/system/greatsql@.service`) にコピーし、@ 記号を追加して、上記の2行で、他のコンテンツは通常どおりです。

    # vim /usr/lib/systemd/system/greatsql@.service
    ...
    ExecStartPre=/usr/local/GreatSQL-8.0.25-15-Linux-glibc2.28-x86_64/bin/mysqld_pre_systemd %I
    ExecStart=/usr/local/GreatSQL-8.0.25-15-Linux-glibc2.28-x86_64/bin/mysqld --defaults-group-suffix=@%I $MYSQLD_OPTS

ここで GreatSQL のバイナリパスに変更しました. `mysqld_pre_systemd` ファイルが見つからない場合は、`/usr/bin/mysqld_pre_systemd` からコピーして微調整してください。

次に、/etc/my.cnf 構成ファイルを編集および変更し、元のベースで複数のインスタンスに関連するいくつかのフラグメントを追加します。次に例を示します。

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

次に、コマンド systemctl daemon-reload を実行して systemd サービスをリロードすると、これらの新しく追加されたサービスのリストを特定できます。

    $ systemctl -l

    ... greatsql@mgr01.service loaded active running GreatSQL Server... greatsql@mgr02.service loaded active running GreatSQL Server... greatsql@mgr03.service loaded active running GreatSQL Server... ... 

次のようなコマンドを実行して、マルチインスタンス サービスを直接開始および停止できるようになりました。

    $ systemctl start greatsql@mgr01

スタンドアロン環境でマルチインスタンス サービスを簡単に管理できるようになりました。
