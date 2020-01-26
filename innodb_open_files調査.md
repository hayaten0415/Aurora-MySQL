# innodb_open_files
-------------------
複数のInnoDBテーブルスペースを使用する場合にのみ関連するパラメータ。<br/>
MySQLで一度に開いたままにできる.ibdファイルの最大数が指定される。<br/>
innodb_file_per_tableがOFFの場合デフォルト値は300<br/>
ONの場合は300よりも大きい値もしくはtable_open_cahceの値。<br/>
**以下に示すのは前提としてMySQL5.7を使用する場合**

## 現在の状況（パラメータ）
ディスクIOにに関連するパラメータの現在の状況とAurora MySQLで変更可能な値かどうかについて表で示した。<br/>
※20191006 16:00での調査時

|パラメータ|値|変更可能であるか|備考|
|----|----|----|----|
|innodb_open_files|6000|●||
|table_open_cache|6000|●||
|innodb_file_per_table|ON|●||
|Opened_tables|5604356|-|ステータス変数|
|open_files_limit|65535|おそらく変更不可能|https://www.s-style.co.jp/blog/2019/02/3314/
|table_definition_cache|20000|●||


証跡
```
MYSQL [xx] > show global variables like 'innodb_open_files';
+-------------------+-------+
| Variable_name     | Value |
+-------------------+-------+
| innodb_open_files | 6000  |
+-------------------+-------+
1 row in set (0.00 sec)

MYSQL [xx] > show global variables like 'table_open_cache';
+------------------+-------+
| Variable_name    | Value |
+------------------+-------+
| table_open_cache | 6000  |
+------------------+-------+
1 row in set (0.00 sec)

MYSQL [xx] > show global variables like 'innodb_file_per_table';
+-----------------------+-------+
| Variable_name         | Value |
+-----------------------+-------+
| innodb_file_per_table | ON    |
+-----------------------+-------+
1 row in set (0.00 sec)

MYSQL [xx] > show global status like 'Opened_tables';
+---------------+---------+
| Variable_name | Value   |
+---------------+---------+
| Opened_tables | 5604356 |
+---------------+---------+
1 row in set (0.00 sec)

MYSQL [xx] > show global variables like 'open_files_limit';
+------------------+-------+
| Variable_name    | Value |
+------------------+-------+
| open_files_limit | 65535 |
+------------------+-------+
1 row in set (0.00 sec)

MYSQL [xx] > show global variables like 'table_definition_cache';
+------------------------+-------+
| Variable_name          | Value |
+------------------------+-------+
| table_definition_cache | 20000 |
+------------------------+-------+
1 row in set (0.00 sec)

```

## table_open_cahe
max_connectionsと同様、サーバーが開いたままにするファイルの最大数に影響する。<br/>
基本的な計算式を以下に示す。<br/>

```
table_open_cahce = max_connections * 実行するクエリの結合あたりのテーブルの最大数
※本来はこれに一時テーブルとファイル用のいくつかの追加のファイルディスクリプタを予約する必要あり。
```


OSでtable_open_cacheの設定に示されたオープンファイルディスクリプタの数を処理できることを確認する。<br/>
大きすぎるとMySQLが接続を拒否し、クエリの実行に失敗して信頼性が大幅に低下する。<br/>

MySQLが使用できるファイルディスクリプタの数を変更するためにはopen_files_limit変数を設定する。


## open_files_limit
OSでmysqldが開くことを許可するファイル数。<br/>
実際のopen_files_limitの値は、システム起動時に指定された値（ある場合）とmax_connections および table_open_cache の値に基づき、次の式を使用する

```
1) 10 + max_connections + (table_open_cache * 2)
2) max_connections * 5
3) operating system limit if positive
4) if operating system limit is Infinity:
   open_files_limit value specified at startup, 5000 if none
```

サーバーはこれらの３つの値の最大値を使用して、ファイルディスクリプタの数を取得しようとする。<br/>
その数のディスクリプタが取得できない場合、サーバーはシステムに許可されるできるだけ多くの数を取得しようとする。<br/>

MySQLサーバーを構築しているOSがUNIXであれば``` ulimit -n```より大きい値は設定できない。

## table_definition_cache
定義キャッシュに格納可能な (.frm ファイルからの) テーブル定義の数。<br/>
多数のテーブルを使用する場合、大きいテーブル定義キャッシュを作成して、テーブルを開くことを高速化可能。<br/>
標準のテーブルキャッシュと異なり、テーブル定義キャッシュは占有スペースが少なくファイルディスクリプタを使用しない。<br/>

table_definition_cacheについて他の重要点
1. table_definition_cacheは、一度に開くことができる、InnoDB file-per-table テーブルスペースの数のソフト制限を定義。これは innodb_open_files によっても制御される。
2. table_definition_cache および innodb_open_files の両方が設定される場合、高い方の設定値が使用される
3. どちらの変数も設定されない場合、デフォルト値が高い table_definition_cache が使用される。
4. オープンテーブルスペースファイルハンドルの数が、table_definition_cache または innodb_open_files によって定義された制限を超える場合、LRU メカニズムは、テーブルスペースファイル LRU リストを検索して、完全にフラッシュされて現在延長されていないファイルを探す。
5. 手順4の処理は、新しいテーブルスペースが開くたびに実行される。「非アクティブな」テーブルスペースがない場合、テーブルスペースファイルはクローズされない。


## AuroraのOSについて
以下のパラメータから判明する。
- version_compile_os MySQLが構築されているOSの種類
- version_compile_machine サーバーバイナリのタイプ

以下現在のパラメータ
|パラメータ|値|
|----|----|
|version_compile_os|Linux|
|version_compile_machine|x86_64|


証跡
```

MYSQL [xx] > show global variables like 'version_compile_os';
+--------------------+-------+
| Variable_name      | Value |
+--------------------+-------+
| version_compile_os | Linux |
+--------------------+-------+
1 row in set (0.01 sec)


MYSQL [xx] > show global variables like 'version_compile_machine';
+-------------------------+--------+
| Variable_name           | Value  |
+-------------------------+--------+
| version_compile_machine | x86_64 |
+-------------------------+--------+
1 row in set (0.01 sec)


```


## その他

### Aurora MySQLのインスタンスタイプごとのメモリ、CPUなど
|インスタンスクラス|vCPU|ECU|メモリ (GiB)|
|----|----|----|----|
|db.t2.small|1|1|2|
|db.t2.medium|2|2|4|
|db.r3.large|2|6.5|15.25|
|db.r3.xlarge|4|13|30.5|
|db.r3.2xlarge|8|26|61|
|db.r3.4xlarge|16|52|122|
|db.r3.8xlarge|32|104|244
|db.r4.large|2|7|15.25|
|db.r4.xlarge|4|13.5|30.5
|db.r4.2xlarge|8|27|61|
|db.r4.4xlarge|16|53|122|
|db.r4.8xlarge|32|99|244|
|db.r4.16xlarge|64|195|488|
|db.r5.large|2|10|16|
|db.r5.xlarge|4|19|32|
|db.r5.2xlarge|8|38|64|
|db.r5.4xlarge|16|71|128|
|db.r5.12xlarge|48|173|384|

### Aurora MySQL DB インスタンスへの最大接続数

|インスタンスクラス|max_connections デフォルト値|
|----|----|
|db.t2.small|45|
|db.t2.medium|90|
|db.r3.large|1,000|
|db.r3.xlarge|2000|
|db.r3.2xlarge|3000|
|db.r3.4xlarge|4000|
|db.r3.8xlarge|5000|
|db.r4.large|1,000|
|db.r4.xlarge|2000|
|db.r4.2xlarge|3000|
|db.r4.4xlarge|4000|
|db.r4.8xlarge|5000|
|db.r4.16xlarge|6000|
|db.r5.large|1,000|
|db.r5.xlarge|2000|
|db.r5.2xlarge|3000|
|db.r5.4xlarge|4000|
|db.r5.12xlarge|6000|

**※max_connectionsを増やしたい場合、メモリ量の多いインスタンスクラスに変更するか、<br/>max_connections パラメータの設定値を最大 16,000 まで大きくすること**


### 参考リンク
- https://dev.mysql.com/doc/refman/5.6/ja/innodb-parameters.html#sysvar_innodb_open_files
- https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_open_files
- https://dev.mysql.com/doc/refman/5.6/ja/table-cache.html
- https://dev.mysql.com/doc/refman/5.7/en/table-cache.html
- https://dev.mysql.com/doc/refman/5.6/ja/server-system-variables.html#sysvar_table_definition_cache
- https://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html#sysvar_table_definition_cache
- https://dev.mysql.com/doc/refman/5.6/ja/not-enough-file-handles.html
- https://dev.mysql.com/doc/refman/5.7/en/not-enough-file-handles.html
- https://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html#sysvar_version_compile_os
- https://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html#sysvar_version_compile_machine
- https://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html#sysvar_open_files_limit
- https://dev.mysql.com/doc/refman/5.6/ja/optimizing-innodb-diskio.html
- https://dev.mysql.com/doc/refman/5.7/en/optimizing-innodb-diskio.html
- https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Reference.html#AuroraMySQL.Reference.Parameters.Instance
- https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Managing.Performance.html
