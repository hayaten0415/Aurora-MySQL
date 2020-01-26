# ロック関連のsysまたはperformance_schema調査
------------------------------------

## 目次
* [背景](#背景)
* [実行環境](#実行環境)
* [前提知識](#必要な前提知識)
  * [メタデータロック](#メタデータロック)
  * [全行ロック](#全行ロック)
* [ロック関連のperformace_schemaあるいはsys](#ロック関連のperformace_schemaあるいはsys)
  * [sys.innodb_lock_waits](#sys.innodb_lock_waits)
  * [performance_schema.metadata_locks](#performance_schema.metadata_locks)
  * [performance_schema.table_handles](#performance_schema.table_handles)
  * [performance_schema.table_lock_waits_summary_by_table](#performance_schema.table_lock_waits_summary_by_table)
    * [内部ロック](#内部ロックについて)
    * [外部ロック](#外部ロックについて)
  * [performance_schema.events_waits_current](#performance_schema.events_waits_current)
  * [sys.schema_table_lock_waits](#sys.schema_table_lock_waits)
* [ロックとは無関連だが原因究明に必要なテーブル](#LOCKとは無関係だがロックの原因を探すために必要なテーブル)
  * [sys.processlist](#sys.processlist)
  * [information_schema.innodb_locks](#information_schema.innodb_locks)
* [参考リンク](#参考リンク)


## 背景
Aurora MySQLのAuroraバージョンをバージョンアップすることにより<br/>
performance_schemaの利用が可能になった。このことに際し、我々がチューニングにおいて特に重要だと<br/>
考えているperformances_schemaあるいはsysの調査を行う。

## 実行環境
- Amazon Aurora MySQL
- aurora_version:2.04.8
- innodb_version:5.7.12

## 必要な前提知識
今回の資料を読むに当たり必要な事前知識を以下に示す。
- MySQLにはテーブルロックと行ロックがある。
- MySQLの**InnoDB**におけるテーブルロックは2種類に分けられる、メタデータロックと全行ロック(公式の表現ではないがここではこう呼ぶことにする)

### メタデータロック
メタデータロックを引き起こすクエリの例として以下のものがある
- LOCK TABLE 
- DROP TABLE
- ALTER TABLE
- TRUNCATE
- SLEEP()

### 全行ロック
SELECT、UPDATE、DELETEなどによるクエリによってテーブルロックが実行された場合、
InnoDBは文字通り全行ロックという形でテーブルロックを実現する。

## ロック関連のperformace_schemaあるいはsys
ロック関連のperformance_schemaとsysでは以下のものが存在する。
- sys.innodb_lock_waits
- performance_schema.metadata_locks
- performance_schema.table_handles
- performance_schema.table_lock_waits_summary_by_table
- sys.schema_table_lock_waits

これらについて以下にそれぞれ述べる。

### sys.innodb_lock_waits
このビューはInnoDBトランザクションが待機しているロック待ちの情報を示すビューである。<br/>
以下がこのVIEWの定義となっている。
```SQL
VIEW_DEFINITION: SELECT `r`.`trx_wait_started` AS `wait_started`,
       timediff(now(), `r`.`trx_wait_started`) AS `wait_age`,
       timestampdiff(SECOND, `r`.`trx_wait_started`, now()) AS `wait_age_secs`,
       `rl`.`lock_table` AS `locked_table`,
       `rl`.`lock_index` AS `locked_index`,
       `rl`.`lock_type` AS `locked_type`,
       `r`.`trx_id` AS `waiting_trx_id`,
       `r`.`trx_started` AS `waiting_trx_started`,
       timediff(now(), `r`.`trx_started`) AS `waiting_trx_age`,
       `r`.`trx_rows_locked` AS `waiting_trx_rows_locked`,
       `r`.`trx_rows_modified` AS `waiting_trx_rows_modified`,
       `r`.`trx_mysql_thread_id` AS `waiting_pid`,
       `sys`.`format_statement`(`r`.`trx_query`) AS `waiting_query`,
       `rl`.`lock_id` AS `waiting_lock_id`,
       `rl`.`lock_mode` AS `waiting_lock_mode`,
       `b`.`trx_id` AS `blocking_trx_id`,
       `b`.`trx_mysql_thread_id` AS `blocking_pid`,
       `sys`.`format_statement`(`b`.`trx_query`) AS `blocking_query`,
       `bl`.`lock_id` AS `blocking_lock_id`,
       `bl`.`lock_mode` AS `blocking_lock_mode`,
       `b`.`trx_started` AS `blocking_trx_started`,
       timediff(now(), `b`.`trx_started`) AS `blocking_trx_age`,
       `b`.`trx_rows_locked` AS `blocking_trx_rows_locked`,
       `b`.`trx_rows_modified` AS `blocking_trx_rows_modified`,
       concat('KILL QUERY ', `b`.`trx_mysql_thread_id`) AS `sql_kill_blocking_query`,
       concat('KILL ', `b`.`trx_mysql_thread_id`) AS `sql_kill_blocking_connection`
FROM ((((`information_schema`.`innodb_lock_waits` `w`
         JOIN `information_schema`.`innodb_trx` `b` on((`b`.`trx_id` = `w`.`blocking_trx_id`)))
        JOIN `information_schema`.`innodb_trx` `r` on((`r`.`trx_id` = `w`.`requesting_trx_id`)))
       JOIN `information_schema`.`innodb_locks` `bl` on((`bl`.`lock_id` = `w`.`blocking_lock_id`)))
      JOIN `information_schema`.`innodb_locks` `rl` on((`rl`.`lock_id` = `w`.`requested_lock_id`)))
ORDER BY `r`.`trx_wait_started`
```
この定義からこのビューはinformation_schemaのinnodb_lock_waits、innodb_lock_waits、innodb_locksの<br/>
3つのテーブルをJOINしたものだと分かる。
このテーブルはクエリを実行した際に、ロック待ちだと思われる場合に見るべきテーブルの一つである。<br/>
もう一つの見るべきテーブルは``sys.schema_table_lock_waits``


### performance_schema.metadata_locks
メタデータロックの情報がが公開されるテーブル。
メタデータロックについて詳しい情報は[こちら](#https://dev.mysql.com/doc/refman/5.7/en/metadata-locking.html)<br/>

このテーブルにメタデータロックの情報を公開するために以下のクエリを実行する。
```SQL
UPDATE performance_schema.setup_instruments
    -> SET ENABLED = 'YES', TIMED = 'YES'
    -> WHERE NAME = 'wait/lock/metadata/sql/mdl';
```

### performance_schema.table_handles
- テーブルロック情報が公開されているテーブル。
- 読み取り専用で、テーブルサイズはデフォルトで自動サイズが設定される。
  - このテーブルのテーブルサイズは``performance_schema_max_table_handles``というシステム変数で設定する。
- このテーブルをTRUNCATEすることは出来ない
- ロックの測定項目には``wait/lock/table/sql/handler``というinstrumentを使用し、これはデフォルトで有効化されている

サーバーの起動時にテーブルロックの計測状態を制御するには、my.cnfファイルで次のような行を使用する
```
[mysqld]
performance-schema-instrument='wait/lock/table/sql/handler=ON'
```
実行時にテーブルロックの計測状態を制御するには、以下のようにsetup_instrumentsテーブルを更新する
```SQL
UPDATE performance_schema.setup_instruments
SET ENABLED = 'YES', TIMED = 'YES'
WHERE NAME = 'wait/lock/table/sql/handler';
```

**※** AWSでのパフォーマンスインサイトを有効にした際にパラメータを以下のように設定しているので現在(2019/11/29時点)は問題ない

```
performance-schema-instrument='wait/%=ON'
```
**※**そもそも今回のロックの調査結果から見る必要がない可能性あり

### performance_schema.table_lock_waits_summary_by_table
- ``wait/lock/table/sql/handler``というinstrumentにより生成されるように全てのテーブルロック待ちのイベントを集計するテーブル。
- このテーブルには内部ロックと外部ロックに関する情報が含まれる
- このテーブルと関連するテーブルとしてperformance_schema.events_waits_currentテーブルが存在する

有効にする際は以下のクエリを実行する
```SQL
UPDATE performance_schema.setup_instruments
SET ENABLED = 'YES', TIMED = 'YES'
WHERE NAME = 'wait/lock/table/sql/handler';
```

#### 内部ロックについて
内部ロックはSQLレイヤーのロックに対応している。``thr_lock()``によって実装されている。OPERATIONカラムでは以下のうちどれか一つに区別される。
- read normal
- read with shared locks
- read high priority
- read no insert
- write allow write
- write concurrent insert
- write delayed
- write low priority
- write normal

#### 外部ロックについて
外部ロックは、ストレージエンジンレイヤーのロックに対応している。<br/>
``handler::external_lock()``によって実装され、OPERATIONカラムでは以下のうちどれか一つに区別される。
- read external
- write external


メタデータロックの待機中にブロックされているセッションと、それらをブロックしているものが表示される。<br/>
以下がこのVIEWの定義となっている。
```SQL
VIEW_DEFINITION: SELECT `g`.`OBJECT_SCHEMA` AS `object_schema`,
       `g`.`OBJECT_NAME` AS `object_name`,
       `pt`.`THREAD_ID` AS `waiting_thread_id`,
       `pt`.`PROCESSLIST_ID` AS `waiting_pid`,
       `sys`.`ps_thread_account`(`p`.`OWNER_THREAD_ID`) AS `waiting_account`,
       `p`.`LOCK_TYPE` AS `waiting_lock_type`,
       `p`.`LOCK_DURATION` AS `waiting_lock_duration`,
       `sys`.`format_statement`(`pt`.`PROCESSLIST_INFO`) AS `waiting_query`,
       `pt`.`PROCESSLIST_TIME` AS `waiting_query_secs`,
       `ps`.`ROWS_AFFECTED` AS `waiting_query_rows_affected`,
       `ps`.`ROWS_EXAMINED` AS `waiting_query_rows_examined`,
       `gt`.`THREAD_ID` AS `blocking_thread_id`,
       `gt`.`PROCESSLIST_ID` AS `blocking_pid`,
       `sys`.`ps_thread_account`(`g`.`OWNER_THREAD_ID`) AS `blocking_account`,
       `g`.`LOCK_TYPE` AS `blocking_lock_type`,
       `g`.`LOCK_DURATION` AS `blocking_lock_duration`,
       concat('KILL QUERY ', `gt`.`PROCESSLIST_ID`) AS `sql_kill_blocking_query`,
       concat('KILL ', `gt`.`PROCESSLIST_ID`) AS `sql_kill_blocking_connection`
FROM (((((`performance_schema`.`metadata_locks` `g`
          JOIN `performance_schema`.`metadata_locks` `p` on(((`g`.`OBJECT_TYPE` = `p`.`OBJECT_TYPE`)
                                                             AND (`g`.`OBJECT_SCHEMA` = `p`.`OBJECT_SCHEMA`)
                                                             AND (`g`.`OBJECT_NAME` = `p`.`OBJECT_NAME`)
                                                             AND (`g`.`LOCK_STATUS` = 'GRANTED')
                                                             AND (`p`.`LOCK_STATUS` = 'PENDING'))))
         JOIN `performance_schema`.`threads` `gt` on((`g`.`OWNER_THREAD_ID` = `gt`.`THREAD_ID`)))
        JOIN `performance_schema`.`threads` `pt` on((`p`.`OWNER_THREAD_ID` = `pt`.`THREAD_ID`)))
       LEFT JOIN `performance_schema`.`events_statements_current` `gs` on((`g`.`OWNER_THREAD_ID` = `gs`.`THREAD_ID`)))
      LEFT JOIN `performance_schema`.`events_statements_current` `ps` on((`p`.`OWNER_THREAD_ID` = `ps`.`THREAD_ID`)))
WHERE (`g`.`OBJECT_TYPE` = 'TABLE');
```

### performance_schema.events_waits_current
- 現在待機中となっているイベントの情報を公開するテーブル。
- ``TRUNCATE TABLE``が可能

### sys.schema_table_lock_waits
- このテーブルはクエリを実行した際に、ロック待ちだと思われる場合に見るべきテーブルである
- メタデータロックそのものを確認したい場合、performance_schema.metadata_locksを見るべきである

## LOCKとは無関係だがロックの原因を探すために必要なテーブル
以下にロックの原因を追う際に直接ロックの情報が公開されてはいないが原因を突き止めるにあたって見るべきテーブルを示す。

### sys.processlist
いわゆる``SHOW PROCESSLIST``を簡略化したものが入っているテーブル

### information_schema.innodb_locks
- 行ロックあるいは全行ロックが起きているときに見るべきテーブル
- 行ロックあるいは全行ロックの情報が公開されている
- このテーブルのカラムにlock_typeというものがあり、ここには``RECORD``あるいは``TABLE``が当てはまるが、
innodbはテーブルロックを全行ロックで実現しているためlock_typeには``RECORD``のみ入り得る

## 参考リンク
- https://dev.mysql.com/doc/refman/5.7/en/metadata-locking.html
- https://dev.mysql.com/doc/refman/5.7/en/sys-innodb-lock-waits.html
- https://dev.mysql.com/doc/refman/5.7/en/metadata-locks-table.html
- https://dev.mysql.com/doc/refman/5.7/en/table-handles-table.html
- https://dev.mysql.com/doc/refman/5.7/en/table-waits-summary-tables.html#table-lock-waits-summary-by-table-table
- https://dev.mysql.com/doc/refman/5.7/en/sys-schema-table-lock-waits.html
- https://dev.mysql.com/doc/refman/5.7/en/events-waits-current-table.html
- https://dev.mysql.com/doc/refman/5.7/en/sys-processlist.html
- https://dev.mysql.com/doc/refman/5.7/en/innodb-locks-table.html
- https://dev.mysql.com/doc/refman/5.7/en/performance-schema-instrument-naming.html
- http://bluerabbit.hatenablog.com/entry/2013/12/07/075759
- https://ja.programqa.com/
