# sysbenchインストール、使用、結果の解釈

sysbench は、さまざまなシステム パラメーターの下でデータベースの負荷を評価およびテストするために主に使用される、モジュール式、クロスプラットフォーム、マルチスレッドのベンチマーク ツールです。
現在、sysbench コードは launchpad、プロジェクト アドレス: https://launchpad.net/sysbench でホストされ、ソース コードは bazaar によって管理されています。

## ソースパッケージをダウンロード
bzr クライアントをインストールします。

	yum install bzr
その後、bzr クライアントで tpcc-mysql ソース コードのダウンロードを開始できます。

	cd /tmp
	bzr branch lp:sysbench

sysbench は、次のテスト モードをサポートしています。

1. CPU性能
2. ディスク IO パフォーマンス
3. スケジューラのパフォーマンス
4. メモリ割り当てと転送速度
5. POSIX スレッドのパフォーマンス
6. データベースのパフォーマンス (OLTP ベンチマーク)
   
現在、sysbench は主に mysql、drizzle、pgsql、oracle などのいくつかのデータベースをサポートしています。

## コンパイルしてインストールする
コンパイルは非常に簡単です。README ドキュメントを参照できます。簡単な手順は次のとおりです。

	cd /tmp/sysbench-0.4.12-1.1
	./autogen.sh
	./configure --with-mysql-includes=/usr/local/mysql/include --with-mysql-libs=/usr/local/mysql/lib && make

	# make がエラーを報告しない場合、バイナリ コマンド ライン ツール sysbench が sysbench ディレクトリに生成されます。
	ls -l sysbench
	-rwxr-xr-x 1 root root 3293186 Sep 21 16:24 sysbench

## OLTP テストの準備
テスト ライブラリ環境を初期化します (合計 10 個のテスト テーブル、それぞれ 100,000 レコード、ランダムに生成されたデータが入力されます)：

	cd /tmp/sysbench-0.4.12-1.1/sysbench
	mysqladmin create sbtest

	./sysbench --mysql-host=1.2.3.4 --mysql-port=3317 --mysql-user=tpcc --mysql-password=tpcc \
	--test=tests/db/oltp.lua --oltp_tables_count=10 --oltp-table-size=100000 --rand-init=on prepare
これらのパラメータの説明:

	--test=tests/db/oltp.lua oltp モードのテストのために tests/db/oltp.lua スクリプトが呼び出されることを示します
	--oltp_tables_count=10 10 個のテスト テーブルが生成されることを示します
	--oltp-table-size=100000 各テスト テーブルに入力されたデータの量が 100000 であることを示します
	--rand-init=on 各テスト テーブルにランダム データが入力されていることを示します
このマシンを使用している場合は、–mysql-socket を使用して、接続する socketxxxt ファイルを指定することもできます。 テストデータの読み込みにかかる時間はデータ量によって異なりますので、時間がかかる場合は我慢が必要です。

実際のテスト シナリオでは、10 個以上のデータ テーブルがあり、1 つのテーブルのデータ量が 500 万行以上であることをお勧めします (もちろん、サーバーのハードウェア構成によって異なります)。 SSD や PCIE SSD などの高 IOPS デバイスを搭載している場合は、1 つのテーブルのデータ量が 1 億行以上であることをお勧めします。

## OLTP テストを実施する

上記の初期化データ パラメータに基づいて、さらにパラメータを追加すると、テストを開始できます。

	./sysbench --mysql-host=1.2.3.4. --mysql-port=3306 --mysql-user=tpcc \
	--mysql-password=tpcc --test=tests/db/oltp.lua --oltp_tables_count=10 \
	--oltp-table-size=10000000 --num-threads=8 --oltp-read-only=off \
	--report-interval=10 --rand-type=uniform --max-time=3600 \
	--max-requests=0 --percentile=99 run >> ./log/sysbench_oltpX_8_20140921.log
オプション 説明

	--num-threads=8 8 つの同時接続が開始されたことを示します
	--oltp-read-only=off 読み取り専用テストを実行しないことを示します。つまり、読み取り/書き込み混合モード テストを使用します。
	--report-interval=10 10秒ごとにテスト進捗レポートを出力することを示します
	--rand-type=uniform ランダム タイプが固定モードであり、その他のいくつかのランダム モード (uniform、gaussian、special、pareto) であることを示します。
	--max-time=120 最大実行時間が 120 秒であることを示します
	--max-requests=0 リクエストの総数が0であることを示します. 総実行時間は上記で定義されているため, 総リクエスト数を0に設定することもできます. 最大実行時間を設定せずに総リクエスト数のみを設定することもできます.
	--percentile=99 サンプリング率を設定することを示します。デフォルトは 95% です。つまり、長いリクエストの 1% を破棄し、残りの 99% で最大値を取ります

つまり、それぞれ 1,000 万行のレコードを持つ 10 個のテーブルで OLTP テストをシミュレートし、連続ストレス テストの時間は 1 時間です。

実際のテスト シナリオでは、継続的なストレス テストの時間を 30 分以上にすることをお勧めします。そうしないと、テスト データが参考にならなくなります。

## テスト結果の解釈:

テスト結果は次のように解釈されます。

	sysbench 0.5:  multi-threaded system evaluation benchmark

	Running the test with following options:
	Number of threads: 8
	Report intermediate results every 10 second(s)
	Random number generator seed is 0 and will be ignored


	Threads started!
	-- 10 秒ごとのテスト結果、tps、1 秒あたりの読み取り、1 秒あたりの書き込み、および 99% を超える応答時間の統計をレポートします
	[  10s] threads: 8, tps: 1111.51, reads/s: 15568.42, writes/s: 4446.13, response time: 9.95ms (99%)
	[  20s] threads: 8, tps: 1121.90, reads/s: 15709.62, writes/s: 4487.80, response time: 9.78ms (99%)
	[  30s] threads: 8, tps: 1120.00, reads/s: 15679.10, writes/s: 4480.20, response time: 9.84ms (99%)
	[  40s] threads: 8, tps: 1114.20, reads/s: 15599.39, writes/s: 4456.30, response time: 9.90ms (99%)
	[  50s] threads: 8, tps: 1114.00, reads/s: 15593.60, writes/s: 4456.70, response time: 9.84ms (99%)
	[  60s] threads: 8, tps: 1119.30, reads/s: 15671.60, writes/s: 4476.50, response time: 9.99ms (99%)
	OLTP test statistics:
		queries performed:
			read:            938224    -- 読み取りの総数
			write:           268064    -- 書き込みの総数
			other:           134032    -- その他の操作の合計(COMMIT など、SELECT、INSERT、UPDATE、DELETE 以外の操作。)
			total:           1340320    -- すべての合計
		transactions:        67016  (1116.83 per sec.)    -- 合計トランザクション (1 秒あたりのトランザクション)
		deadlocks:           0      (0.00 per sec.)    -- デッドロックの総数
		read/write requests: 1206288 (20103.01 per sec.)    -- 読み取りと書き込みの合計数 (1 秒あたりの読み取りと書き込みの数)
		other operations:    134032 (2233.67 per sec.)    -- その他の操作の合計 (1 秒あたりのその他の操作)

	General statistics:    -- いくつかの統計
		total time:          60.0053s    -- 合計時間
		total number of events:              67016    -- 総取引
		total time taken by event execution: 479.8171s    -- 時間のかかるすべてのトランザクションの追加 (同時実行要因を考慮しない)
		response time:    -- 応答時間の統計
			min:             4.27ms    -- 最短時間
			avg:             7.16ms    -- 平均時間
			max:             13.80ms    -- 最長の時間
			approx.  99 percentile: 9.88ms    -- 99%以上の平均時間

	Threads fairness:
		events (avg/stddev):           8377.0000/44.33
		execution time (avg/stddev):   59.9771/0.00