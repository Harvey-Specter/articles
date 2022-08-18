# Zabbix データベースを TokuDB に移行する

## 背景紹介
オンラインの Zabbix データベースには、データ量が急増した大規模なテーブルがいくつかあります.単一のテーブルは 500G を超えており、初期段階でパーティション テーブルが作成されておらず、後で維持するのが非常に面倒です. 例えば、期限切れの履歴データを削除したい場合、元のモードでは、`history`、`history_uint` などのいくつかの大きなテーブルを、主キーを 2 つのフィールド (itemid、clock) で結合し、clock フィールドのみを使用します。検索効率が非常に悪いです。

TokuDB は、MySQL および MariaDB 向けの高性能なトランザクション対応ストレージ エンジンです。 TokuDB の主な機能は、高い圧縮率、高い INSERT パフォーマンス、インデックスのほとんどのオンライン変更とフィールドの追加のサポートです。特に、Zabbix のような高い INSERT と少ない UPDATE アプリケーション シナリオに適しています。

## 移行の準備
TokuDBエンジンを利用する場合、サービスレイヤーはMariaDBかPerconaを選べますが、私は以前Perconaをよく使っていたので、今回もPercona版を利用してTokuDBエンジンを統合することにしました。

通常の方法でインストールし、構成ファイルに 3 行を追加します。

```properties
malloc-lib= /usr/local/mysql/lib/mysql/libjemalloc.so
plugin-dir = /usr/local/mysql/lib/mysql/plugin/
plugin-load=ha_tokudb.so
```

jemalloc がロードされていない場合、起動時に次のようなエラーが発生します。

```log
[ERROR] TokuDB not initialized because jemalloc is not loaded
[ERROR] Plugin 'TokuDB' init function returned error.
[ERROR] Plugin 'TokuDB' registration as a STORAGE ENGINE failed.
```

そして、カーネル構成を変更し、transparent_hugepage を無効にします。閉じていない場合、TokuDB のメモリ リークが発生する可能性があります (/etc/rc.local に書き込むことをお勧めします。再起動後も有効になります)。

```bash
echo never > /sys/kernel/mm/redhat_transparent_hugepage/defrag
echo never > /sys/kernel/mm/redhat_transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag
```

カーネル設定を変更しないと、起動時に次のようなエラーが発生します。

```log
Transparent huge pages are enabled, according to /sys/kernel/mm/redhat_transparent_hugepage/enabled
Transparent huge pages are enabled, according to /sys/kernel/mm/transparent_hugepage/enabled
[ERROR] TokuDB will not run with transparent huge pages enabled.
[ERROR] Please disable them to continue.
[ERROR] (echo never > /sys/kernel/mm/transparent_hugepage/enabled)
[ERROR]
[ERROR] ************************************************************
[ERROR] Plugin 'TokuDB' init function returned error.
[ERROR] Plugin 'TokuDB' registration as a STORAGE ENGINE failed.
```

次に、データベースを初期化して起動します。

私のサーバー構成: E5-2620 * 2、64G メモリ、1T の空きディスク容量 (datadir が配置されているパーティションを xfs ファイル システムに設定することをお勧めします)。以下は、参考のためにのみ使用する関連オプションです。

```
[client]
port            = 3306
socket          = mysql.sock
#default-character-set=utf8
 
[mysql]
prompt="\\u@\\h \\D \\R:\\m:\\s [\\d]>
#pager="less -i -n -S"
tee=/home/mysql/query.log
no-auto-rehash
 
[mysqld]
open_files_limit = 8192
max_connect_errors = 100000
 
#buffer & cache
table_open_cache = 2048
table_definition_cache = 2048
max_heap_table_size = 96M
sort_buffer_size = 2M
join_buffer_size = 2M
tmp_table_size = 96M
key_buffer_size = 8M
read_buffer_size = 2M
read_rnd_buffer_size = 16M
bulk_insert_buffer_size = 32M
 
#innodb
#一部の小さなテーブルのみが InnoDB エンジンを保持するため、InnoDB バッファー プールを 1G に設定するだけで十分です。
innodb_buffer_pool_size = 1G
innodb_buffer_pool_instances = 1
innodb_data_file_path = ibdata1:1G:autoextend
innodb_flush_log_at_trx_commit = 1
innodb_log_buffer_size = 64M
innodb_log_file_size = 256M
innodb_log_files_in_group = 2
innodb_file_per_table = 1
innodb_status_file = 1
transaction_isolation = READ-COMMITTED
innodb_flush_method = O_DIRECT

#tokudb
malloc-lib= /usr/local/mysql/lib/mysql/libjemalloc.so
plugin-dir = /usr/local/mysql/lib/mysql/plugin/
plugin-load=ha_tokudb.so
 
#TokuDB の datadir と logdir を MySQL の datadir から分離します.美的観点からも分離できません.この行と次の 2 行をコメントアウトするだけです.
tokudb-data-dir = /data/mysql/zabbix_3306/tokudbData
tokudb-log-dir = /data/mysql/zabbix_3306/tokudbLog
 
#TokuDB の行モードでは、FAST を使用することをお勧めします。ディスク容量が非常に逼迫している場合は、SMALL を使用することをお勧めします。
#tokudb_row_format = tokudb_small
tokudb_row_format = tokudb_fast
tokudb_cache_size = 44G
 
#他の構成のほとんどは実際には変更できません。いくつかの重要な構成のみが必要です。
tokudb_commit_sync = 0
tokudb_directio = 1
tokudb_read_block_size = 128K
tokudb_read_buf_size = 128K

```

## 移行プロセス

Percona (TokuDB) インスタンス プロセスを新しいサーバーで開始し、新しい Zabbix データベースを初期化し、大きなテーブルを直接 TokuDB エンジンに変換し、パーティション モードを有効にすることをお勧めします。これは、オンラインの ALTER TABLE または INSERT...SELECT から直接データをインポートするよりも高速です (単純にテストしたところ、ほぼ 2 ～ 3 倍、またはそれ以上の速度です)。

データの移行を行う場合は、ターゲット サーバーでデータベース テーブル構造を初期化し、ソース サーバーでセグメント単位でエクスポートすることをお勧めします。1 つのテーブルで複数のバックアップ ファイルをエクスポートし、リカバリ中に同時にインポートできるようにします。インポートするときは、binlog を一時的に閉じることを忘れないでください。インポート速度を向上させるために、少なくとも sync_binlog = 0 と tokudb_commit_sync = 0 を設定してください。 mysqldump を使用して -w パラメータを追加し、条件に応じてセグメント化されたエクスポートを実現するか、MySQLDumper を使用します。

外部キーを使用する必要があるテーブルは、引き続き InnoDB エンジンを保持します. 他のテーブルは TokuDB に変換できます. history_str、trends、trend_uint、history、および history_uint などのいくつかの大きなテーブルは、TokuDB に変換する必要があります. イベントは外部キーを使用する必要があるため、したがって、引き続き InnoDB エンジンを維持します。

## 終わり
次に、実行ステータスを観察し、個々の遅いクエリがブロックされているかどうかを確認します。 私の環境ではitemsテーブルも最初にTokuDBに変換したため、描画のためのSQL実行計画が不正確で非常に遅くなりました。 その後、items テーブルでも外部キーを使用する必要があることが判明したため、InnoDB テーブルに戻して、SQL を正常に戻しました。

