# インデックス関連のsysあるいはperformance_schemaの調査

## 変更履歴
|変更日|変更内容|
|-----|--------|
|2019/12/12|新規作成|

## 目次
* [背景](#背景)
* [前提条件](#前提条件)
* [インデックス監視関連のテーブル](#インデックス監視関連のテーブル)
  * [sys.schema_index_statistics](#sys.schema_index_statistics)
  * [sys.schema_unused_indexes](#sys.schema_unused_indexes)
  * [performance_schema.table_io_waits_summary_by_index_usage](#performance_schema.table_io_waits_summary_by_index_usage)
* [使用されていないインデックスを特定する手順の例](#使用されていないインデックスを特定する手順の例)
  * [実行環境](#実行環境)
  * [解析手順](#解析手順)
* [参考リンク](#参考リンク)

## 背景
インデックスの二重管理が起きてしまっているが、<br/>
この度これらをマージすることになった。マージする際に消すかどうかの判断をするのにsysあるいはperformance_schemaから<br/>
読み取れる情報を根拠に判断するためこれらの使用方法を調査する

## 前提条件
- performance_schemaが有効になっている
- 以下の状態になっていること
```SQL
MYSQL [hoge] > SELECT * FROM performance_schema.setup_instruments
    -> WHERE name = 'wait/io/table/sql/handler';
+---------------------------+---------+-------+
| NAME                      | ENABLED | TIMED |
+---------------------------+---------+-------+
| wait/io/table/sql/handler | YES     | YES   |
+---------------------------+---------+-------+
1 row in set (0.00 sec)
```

## インデックス監視関連のテーブル
以下がインデックス関連のテーブルである。
- sys.schema_index_statistics
- sys.schema_unused_indexes
- performance_schema.table_io_waits_summary_by_index_usage

これらのテーブルについて以下に述べる

### sys.schema_index_statistics
VIEW定義
```SQL
MYSQL [hoge] > SELECT VIEW_DEFINITION FROM information_schema.views WHERE table_schema = 'sys' AND TABLE_NAME = 'schema_index_statistics'\G;
*************************** 1. row ***************************
VIEW_DEFINITION: select `performance_schema`.`table_io_waits_summary_by_index_usage`.`OBJECT_SCHEMA` AS `table_schema`,`performance_schema`.`table_io_waits_summary_by_index_usage`.`OBJECT_NAME` AS `table_name`,`performance_schema`.`table_io_waits_summary_by_index_usage`.`INDEX_NAME` AS `index_name`,`performance_schema`.`table_io_waits_summary_by_index_usage`.`COUNT_FETCH` AS `rows_selected`,`sys`.`format_time`(`performance_schema`.`table_io_waits_summary_by_index_usage`.`SUM_TIMER_FETCH`) AS `select_latency`,`performance_schema`.`table_io_waits_summary_by_index_usage`.`COUNT_INSERT` AS `rows_inserted`,`sys`.`format_time`(`performance_schema`.`table_io_waits_summary_by_index_usage`.`SUM_TIMER_INSERT`) AS `insert_latency`,`performance_schema`.`table_io_waits_summary_by_index_usage`.`COUNT_UPDATE` AS `rows_updated`,`sys`.`format_time`(`performance_schema`.`table_io_waits_summary_by_index_usage`.`SUM_TIMER_UPDATE`) AS `update_latency`,`performance_schema`.`table_io_waits_summary_by_index_usage`.`COUNT_DELETE` AS `rows_deleted`,`sys`.`format_time`(`performance_schema`.`table_io_waits_summary_by_index_usage`.`SUM_TIMER_INSERT`) AS `delete_latency` from `performance_schema`.`table_io_waits_summary_by_index_usage` where (`performance_schema`.`table_io_waits_summary_by_index_usage`.`INDEX_NAME` is not null) order by `performance_schema`.`table_io_waits_summary_by_index_usage`.`SUM_TIMER_WAIT` desc
```
インデックスの統計情報が公開されているVIEW<br/>
しかしこのVIEWの大元になっている``performance_schema.table_io_waits_summary_by_index_usage``を見た方が<br/>
より詳しく調べることが出来る。


### sys.schema_unused_indexes
VIEW定義
```SQL
MYSQL [hoge] > SELECT VIEW_DEFINITION FROM information_schema.views WHERE table_schema = 'sys' AND TABLE_NAME = 'schema_unused_indexes'\G;
*************************** 1. row ***************************
VIEW_DEFINITION: select `performance_schema`.`table_io_waits_summary_by_index_usage`.`OBJECT_SCHEMA` AS `object_schema`,`performance_schema`.`table_io_waits_summary_by_index_usage`.`OBJECT_NAME` AS `object_name`,`performance_schema`.`table_io_waits_summary_by_index_usage`.`INDEX_NAME` AS `index_name` from `performance_schema`.`table_io_waits_summary_by_index_usage` where ((`performance_schema`.`table_io_waits_summary_by_index_usage`.`INDEX_NAME` is not null) and (`performance_schema`.`table_io_waits_summary_by_index_usage`.`COUNT_STAR` = 0) and (`performance_schema`.`table_io_waits_summary_by_index_usage`.`OBJECT_SCHEMA` <> 'mysql') and (`performance_schema`.`table_io_waits_summary_by_index_usage`.`INDEX_NAME` <> 'PRIMARY')) order by `performance_schema`.`table_io_waits_summary_by_index_usage`.`OBJECT_SCHEMA`,`performance_schema`.`table_io_waits_summary_by_index_usage`.`OBJECT_NAME`
1 row in set (0.00 sec)
```

イベントが無いインデックスが表示されるもの。イベントが無い、すなわち使用されていないということである。<br/>
しかしこのVIEWはインデックス使用時のI/Oを集計したテーブルから情報を取得しているため、<br/>
長く稼働しているかつ、ワークロードが十分に長く処理されている場合に最も役立つ。<br/>
そうではない場合このVIEWにインデックスが存在していても意味がない可能性がある。<br/>
以下のクエリはdmmcore_poc5で使用されていないインデックスを示すクエリ

```SQL
MYSQL [hoge] > SELECT * FROM sys.schema_unused_indexes
    -> WHERE object_schema = database();
```

### performance_schema.table_io_waits_summary_by_index_usage
- インデックスのI/Oイベントを集計するテーブル
- ``TRUNCATE TABLE``が可能で集計をリセットすることが可能
- ``performance_schema.table_io_waits_summary_by_table``と構成がほとんど同じであり、違いはINDEX_NAMEというカラムがあるだけである。
- 以下のクエリは現在いるスキーマのうち主キー以外のインデックスで使用されているものを参照するクエリである。
```SQL
MYSQL [hoge] > SELECT * FROM performance_schema.table_io_waits_summary_by_index_usage WHERE index_name <> 'PRIMARY' AND OBJECT_SCHEMA = database(); AND COUNT_STAR <> 0\G;

ERROR:
No query specified
```

## 使用されていないインデックスを特定する手順の例
以下に使用されていないインデックスを特定する手順の例を示す。<br/>
**ただし今から示す手順は特定のクエリに関してのインデックスが使用されるかの判断であり**<br/>
**全体的に必要か否かは長いスパンでシステムを稼働させないと判断することは出来ない**<br/>

### 解析手順
のクエリを参考に解析を行った。

1. 以下のクエリを実行し集計結果をリセットする
```SQL
MYSQL [hoge] > TRUNCATE TABLE performance_schema.table_io_waits_summary_by_index_usage;
Query OK, 0 rows affected (0.02 sec)
```

2. インデックス比較対象のクエリを実行する
```SQL

```

3. インデックスが使用されたかを確認する。今回の例ではhogeでテーブル名Thogeについて確認する。
```SQL
MYSQL [hoge] > SELECT * FROM sys.schema_unused_indexes
    -> WHERE object_schema = database()
    -> AND object_name = テーブル名;
```

よって今回のクエリでは``IX_T_PRODUCT_SM_FEATURES_PAGE_LINK_PVN_SID``は使用されていないということが判明した。

4. 念のため使用されたインデックスである``IX_T_PRODUCT_SM_FEATURES_PAGE_LINK``についても確認する。
```SQL
MYSQL [hoge] > SELECT * FROM performance_schema.table_io_waits_summary_by_index_usage
    -> WHERE index_name = 'インデックス名'
    -> AND OBJECT_SCHEMA = database()\G;
```


## 参考リンク
- https://dev.mysql.com/doc/refman/5.7/en/sys-schema-index-statistics.html
- https://dev.mysql.com/doc/refman/5.7/en/sys-schema-unused-indexes.html
- https://dev.mysql.com/doc/mysql-perfschema-excerpt/5.7/en/table-io-waits-summary-by-index-usage-table.html
- https://dev.mysql.com/doc/mysql-perfschema-excerpt/5.7/en/table-io-waits-summary-by-table-table.html
