# MySQL文字セットエラー

数日前、私は同僚に厄介なことを手伝っていました. 問題は次のとおりです:
サーバーに接続するためにクライアントでどのバージョンのmysqlクライアントを使用しても、サーバー側が設定されている限り、
```properties
character-set-server = utf8
```
それで
```
character_set_client、 character_set_connection、character_set_results
```
mysql クライアントにオプションを追加しても、常にサーバー側と一貫性があります。

```properties
--default-character-set=utf8
```

接続してから手動でコマンドを実行しない限り、まだそうではありません

```properties
set names latin1
```

これにより、client、connection、results の文字セットが変更されます。
慎重に比較した結果、エラーの原因はサーバー側で別のオプションが設定されていることが最終的に判明しました。

```properties
skip-character-set-client-handshake
```

ドキュメントでは、このオプションを次のように説明しています。

```properties
--character-set-client-handshake

Don't ignore character set information sent by the client. To ignore client information and use the default server character set, use --skip-character-set-client-handshake; this makes MySQL behave like MySQL 4.0
```

このように、実際には有益です。 たとえば、skip-character-set-client-handshake オプションを有効にすると、クライアント プログラムが他の文字セットを使用して接続し、データを書き込むことによってエラーが発生するという誤動作を防ぐことができます。