# Auroraバージョンアップに伴う変更点調査
----------------

## 調査背景
hoge環境のAurora MySQLのバージョンが古いものを使っており、その結果<br/>
再起動を繰り返しており緊急でバージョンアップを行うこととなったため。

## バージョン確認
現行使用しているバージョン
Aurora Version:2.04.8

## Aurora MySQLバージョン変更による影響
今回のバージョンアップで大きな影響になりそうな部分を以下に述べる
- **パフォーマンススキーマが利用可能**
- SELECT(*) を使用した特定のクエリのパフォーマンスがセカンダリインデックスを持つテーブルに影響を及ぼす可能性があるという Aurora レプリカの問題を修正

また以下に今回のバージョンアップによる影響をすべて記載した。
### サポートされなくなったこと
Aurora MySQL 2.04.4では以下のMySQL 5.7 の以下の機能はサポートされていない
- グループのレプリケーションプラグイン
- ページサイズの増加
- 起動時の InnoDB バッファープールのロード
- InnoDB フルテキストパーサープラグイン
- マルチソースレプリケーション
- オンラインバッファープールのサイズ変更
- パスワード検証プラグイン
- クエリ書き換えプラグイン
- レプリケーションフィルター処理
- CREATE TABLESPACESQL ステートメント

### 改良点
#### Aurora MySQL 2.03からの改良
- **パフォーマンススキーマの利用**
- 「強制終了」状態のゾンビセッションがより多くの CPU を消費するような問題の修正
- 読み取り専用トランザクションが Aurora ライター上のレコードのロックを取得しているときのデッドラッチの問題の修正
- 顧客のワークロードがない Aurora レプリカで CPU 使用率が高くなるような問題の修正
- Aurora レプリカまたは Aurora ライターの再起動を発生させるような問題に関する複数の修正
- ディスクのスループット制限に達したときに診断ログをスキップする機能が追加
- Aurora ライターでバイナリログが有効になっているときのメモリリークの問題の修正
#### Aurora MySQL 2.03.1からの改良
- トランザクションデッドロック検出の実行中に Aurora ライターが再起動する問題を修正
#### Aurora MySQL 2.03.3からの改良
- インデックスに対してバックワードスキャンを実行すると Aurora Replica がデッドラッチになる問題を修正
- Aurora プライマリインスタンスのパーティションテーブルでインプレース DDL オペレーションが実行されると、Aurora Replica が再起動する問題を修正
- Aurora プライマリインスタンスで DDL オペレーションを実行後、クエリキャッシュが無効化されている間に Aurora Replica が再起動する問題を修正
- Aurora プライマリインスタンスのそのテーブルで切り捨てが行われている間に、Aurora Replica がテーブルの SELECT クエリ中に再起動する問題を修正
- インデックス作成された列のみアクセスされる MyISAM の一部テーブルに関する誤った結果の問題を修正
- 約 40,000 回のクエリ後に query_time と lock_time で大きな値が定期的に誤って生成される低速ログの問題を修正
- "tmp" という名前のスキーマが原因で RDS MySQL から Aurora MySQL への移行が停止する問題を修正
- ログの更新中に、イベントが監査ログに見つからない問題を修正
- ラボモードで高速オンラインデータ定義言語 (DDL) 機能が有効な場合に、Aurora 5.6 スナップショットから復元された Aurora プライマリインスタンスが再起動する問題を修正
- ディクショナリの統計スレッドが原因で CPU 使用率が 100% になる問題を修正
- CHECK TABLE ステートメントの実行中に Aurora Replica が再起動する問題を修正
#### Aurora MySQL 2.03.4からの改良
- UTF8MB 4 Unicode 9.0 のアクセントを区別し、大文字と小文字を区別しない照合 utf8mb4_0900_as_ci をサポート
#### Aurora MySQL 2.04からの改良
- GTID ベースのレプリケーションをサポート
- 一時テーブル内の行を削除または更新するステートメントに InnoDB サブクエリが含まれている場合に Aurora Replica で Running in read-only mode エラーが誤ってスローされる問題を修正
#### Aurora MySQL 2.04.1からの改良
- 1.16 以前のバージョン用の Aurora MySQL 5.6 スナップショットを最新の Aurora MySQL 5.7 クラスターに復元できなかった問題を修正。
#### Aurora MySQL 2.04.2からの改良
- カスタム証明書を使用した SSL binlog レプリケーションのサポートを追加
- 全文検索インデックスを持つテーブルが最適化されている場合に発生する Aurora プライマリインスタンスのデッドロックを修正
- SELECT(*) を使用した特定のクエリのパフォーマンスがセカンダリインデックスを持つテーブルに影響を及ぼす可能性があるという Aurora レプリカの問題を修正
- エラー 1032 が投稿される原因となった条件を修正
- Aurora Replica の安定性を向上させるために複数のデッドロックを修正
#### Aurora MySQL 2.04.3からの改良
- binlog スレーブとして設定された Aurora インスタンスで問題が生じる可能性のある binlog レプリケーションのバグを修正
- 大規模なストアドルーチンを処理する際のメモリ不足の問題を修正
- 特定の種類の ALTER TABLE コマンドを処理する際のエラーを修正
- ネットワークプロトコル管理のエラーによる接続の失敗に関する問題を修正
#### Aurora MySQL 2.04.4からの改良
- データを S3 から Aurora にロードする場合にエラーが発生する問題を修正
- データを Aurora から S3 にアップロードする場合にエラーが発生する問題を修正
- ネットワークプロトコル管理のエラーの処理中に接続が中止される問題を修正
- パーティション分割されたテーブルを使用する際にクラッシュする問題を修正
- 一部のリージョンで Performance Insights 機能が利用できない問題を修正
#### Aurora MySQL 2.04.5からの改良
- データベースを再起動する原因となった、ストレージボリュームの拡大中の競合状態を修正
- データベースを再起動する原因となったボリュームオープン中の内部通信障害を修正
- パーティションされたテーブルの ALTER TABLE ALGORITHM=INPLACE の DDL 復元サポートを追加
- データベースを再起動する原因となった ALTER TABLE ALGORITHM=COPY の DDL 復元の問題を修正
- ライターの重度な削除ワークロードの Aurora レプリカの安定性が向上
- 全文検索インデックスの同期を実行しているスレッドと、辞書キャッシュから全文検索テーブルの削除を実行しているスレッドとの間のデッドラッチが原因で<br/>デタベースが再起動される問題を修正
- binlog マスターへの接続が不安定なときの DDL レプリケーション中の binlog スレーブの安定性の問題を修正
- 全文検索コードでデータベースが再起動する原因となったメモリ不足の問題を修正
- 64 TiB のボリューム全体を使用したときに再起動する原因となった Aurora ライターの問題を修正
- データベースを再起動させるパフォーマンススキーマ機能の競合状態を修正
- ネットワークプロトコル管理のエラーの処理中に接続が中止される問題を修正
#### Aurora MySQL 2.04.6からの改良
- パラメータ ``sync_binlog`` の値が 1 に設定されていなかった場合に、マスターの現在のバイナリログファイルのイベントがスレーブでレプリケートされなかった問題を修正
- バイナリログのマスターのフォアグラウンドクエリのパフォーマンスを優先してレプリケーションラグの増加を防ぐために、パラメータ ``aurora_binlog_replication_max_yield_seconds`` のデフォルト値がゼロに変更
#### Aurora MySQL 2.04.7からの改良
- データベースの可用性が改善され、1つ以上のDDLの実行中にクライアント接続の急増に対応可能。必要に応じて一時的に追加のスレッドを作成することで処理される
- エンジンの再起動中に長時間利用できない問題を修正。これは、バッファプールの初期化の問題に対処
- キャッシュされていないデータにアクセスするクエリが通常より遅くなる可能性がある改善
#### Aurora MySQL 2.04.8からの改良
- Aurora DBクラスター内のリーダーインスタンスにデータを効率的に送信することにより、ライターインスタンスからのネットワークトラフィックを削減。<br/>この改善は、レプリカが遅れて再起動するのを防ぐのに役立つため、デフォルトで有効になっています。この機能のパラメーターは``aurora_enable_repl_bin_log_filtering``です。
- 圧縮を使用して、Aurora DBクラスター内のライターインスタンスからリーダーインスタンスへのネットワークトラフィックを削減しました。<br/>この改善は、8xlargeおよび16xlargeインスタンスクラスに対してのみデフォルトで有効になっています。これらのインスタンスは、圧縮のための追加のCPUオーバーヘッドを許容できるためです。この機能のパラメーターは``aurora_enable_replica_log_compression``です。
- Aurora DBクラスター内にリーダーインスタンスが存在する負荷の重い作業中に、<br/>メモリ不足状態によるライターの再起動を防ぐ、Auroraライターインスタンスのメモリ管理の改善。

### Aurora MySQL 2系ではサポートされていないこと
- **Asynchronous Key Prefetch (AKP)**
- ハッシュ結合
- AWS Lambda 関数を同期的に呼び出すためのネイティブ関数。
- スキャンバッチ処理
- Amazon S3 バケットを使用した MySQL からのデータ移行


## 参考リンク
- https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Updates.2025.html
- https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Updates.203.html
- https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Updates.2031.html
- https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Updates.2032.html
- https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Updates.2033.html
- https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Updates.2034.html
- https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Updates.204.html
- https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Updates.2041.html
- https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Updates.2042.html
- https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Updates.2043.html
- https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Updates.2044.html
- https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Updates.2045.html
- https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Updates.2046.html
- https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Updates.2047.html
- https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Updates.2048.html

作成日 2019/11/25
