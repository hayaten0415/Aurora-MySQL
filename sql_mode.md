# sql_modeについて
-----------------

## 背景
Auroraでint型やDATETIME型のカラムに'あああああ'などをINSERTするとなぜかエラーではなく<br/>
そのままINSERTされる現象が起こった

## Auroraでの挙動確認

````
MYSQL [hoge] > desc KW_SALES;
+------------+----------+------+-----+---------+-------+
| Field      | Type     | Null | Key | Default | Extra |
+------------+----------+------+-----+---------+-------+
| SALES_ID   | int(5)   | NO   | PRI | NULL    |       |
| SALES_DATE | datetime | YES  |     | NULL    |       |
| ITEM_NO    | int(5)   | YES  | MUL | NULL    |       |
| SUU        | int(5)   | YES  |     | NULL    |       |
| KINGAKU    | int(10)  | YES  |     | NULL    |       |
+------------+----------+------+-----+---------+-------+
5 rows in set (0.01 sec)

MYSQL [hoge] > SELECT * FROM KW_SALES;
+----------+---------------------+---------+-------+---------+
| SALES_ID | SALES_DATE          | ITEM_NO | SUU   | KINGAKU |
+----------+---------------------+---------+-------+---------+
|        1 | 2019-08-21 08:55:02 |      10 | 10000 |       5 |
|        2 | 2019-08-21 08:55:17 |      10 |   365 |    NULL |
|        3 | 2019-08-21 09:09:18 |      11 |     2 |    NULL |
|        4 | 2019-08-21 09:09:18 |      11 |     2 |    NULL |
+----------+---------------------+---------+-------+---------+
4 rows in set (0.00 sec)

■以下のINSERT文を実行
MYSQL [hoge] > INSERT INTO KW_SALES(SALES_ID, SALES_DATE, ITEM_NO, SUU)
    -> VALUES(0,'2020-11-22 23:25:00', 12, 'aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa');
Query OK, 1 row affected, 1 warning (0.00 sec)

MYSQL [hoge] > commit;
Query OK, 0 rows affected (0.00 sec)

MYSQL [hoge] > SELECT * FROM KW_SALES;
+----------+---------------------+---------+-------+---------+
| SALES_ID | SALES_DATE          | ITEM_NO | SUU   | KINGAKU |
+----------+---------------------+---------+-------+---------+
|        0 | 2020-11-22 23:25:00 |      12 |     0 |    NULL |
|        1 | 2019-08-21 08:55:02 |      10 | 10000 |       5 |
|        2 | 2019-08-21 08:55:17 |      10 |   365 |    NULL |
|        3 | 2019-08-21 09:09:18 |      11 |     2 |    NULL |
|        4 | 2019-08-21 09:09:18 |      11 |     2 |    NULL |
+----------+---------------------+---------+-------+---------+
5 rows in set (0.00 sec)

■別パターン
MYSQL [hoge] > INSERT INTO KW_SALES(SALES_ID, SALES_DATE, ITEM_NO, SUU)
    -> VALUES(0,'ああああああああああああああああああ', 12, 1);
Query OK, 1 row affected, 1 warning (0.01 sec)

MYSQL [hoge] > commit;
Query OK, 0 rows affected (0.00 sec)

MYSQL [hoge] > SELECT * FROM KW_SALES;
+----------+---------------------+---------+-------+---------+
| SALES_ID | SALES_DATE          | ITEM_NO | SUU   | KINGAKU |
+----------+---------------------+---------+-------+---------+
|        0 | 0000-00-00 00:00:00 |      12 |     1 |    NULL |
|        1 | 2019-08-21 08:55:02 |      10 | 10000 |       5 |
|        2 | 2019-08-21 08:55:17 |      10 |   365 |    NULL |
|        3 | 2019-08-21 09:09:18 |      11 |     2 |    NULL |
|        4 | 2019-08-21 09:09:18 |      11 |     2 |    NULL |
+----------+---------------------+---------+-------+---------+
5 rows in set (0.00 sec)
````
このように本来エラーが返ってくるべきところでエラーが返らず<br/>
INSERTされてしまう

## ローカルMySQLでの挙動確認

````
MYSQL [hoge2] > desc KW_SALES;
+------------+----------+------+-----+---------+-------+
| Field      | Type     | Null | Key | Default | Extra |
+------------+----------+------+-----+---------+-------+
| SALES_ID   | int(5)   | NO   | PRI | NULL    |       |
| SALES_DATE | datetime | YES  |     | NULL    |       |
| ITEM_NO    | int(5)   | YES  | MUL | NULL    |       |
| SUU        | int(5)   | YES  |     | NULL    |       |
| KINGAKU    | int(10)  | YES  |     | NULL    |       |
+------------+----------+------+-----+---------+-------+
5 rows in set (0.00 sec)

MYSQL [hoge2] > SELECT * FROM KW_SALES;
Empty set (0.00 sec)

■INSERT文実行
MYSQL [hoge2] > INSERT INTO KW_SALES(SALES_ID, SALES_DATE, ITEM_NO, SUU)
    -> VALUES(1,'2020-11-22 23:25:00', 12, 'あああああああああああああああああああああああああああああ');
ERROR 1366 (HY000): Incorrect integer value: 'あああああああああああああああああああああああああああああ' for column 'SUU' at row 1

エラーが返ってくる

MYSQL [hoge2] > SELECT * FROM KW_SALES;
Empty set (0.00 sec)

■別パターン
MYSQL [hoge2] > INSERT INTO KW_SALES(SALES_ID, SALES_DATE, ITEM_NO, SUU)
    -> VALUES(0,'ああああああああああああああああああ', 12, 1);
ERROR 1292 (22007): Incorrect datetime value: 'ああああああああああああああああああ' for column 'SALES_DATE' at row 1
````

## これらの原因の理由
パラメータの``sql_mode``の設定によるもの

Auroraのsql_mode
````
MYSQL [hoge] > SELECT @@GLOBAL.sql_mode;
+-------------------+
| @@GLOBAL.sql_mode |
+-------------------+
|                   |
+-------------------+
1 row in set (0.00 sec)
````
純MySQLのsql_mode
````
MYSQL [hoge2] > SELECT @@GLOBAL.sql_mode;+-------------------------------------------------------------------------------------------------------------------------------------------+
| @@GLOBAL.sql_mode                                                                                                                         |
+-------------------------------------------------------------------------------------------------------------------------------------------+
| ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION |
+-------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)

````

Auroraではsql_modeがデフォルトでは何も設定されていないため、<br/>
今回のような現象が起こった。

## 設定について
調査の結果、Auroraでもsql_modeの変更が可能であることが公式ドキュメントより判明。

## 現状Auroraで判明していること
- 型の間違いはエラーで返されない
- UNIQUEキーの重複エラーは返される


## 考察
移行のデータの整合性が取れていないこと、以前にpoc5でデータを大量投入した際にDATETIM型のカラムに<br/>
``0000-00-00 00:00:00``が大量に入ったことはこれらの設定によるものではないかと考えられる

## 参考リンク
- https://dev.mysql.com/doc/refman/5.6/ja/sql-mode.html
- https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Reference.html
- https://www.tcmobile.jp/dev_blog/%E6%9C%AA%E5%88%86%E9%A1%9E/rds-aurora-%E3%81%AE-sql_mode-%E3%81%8C%E3%83%87%E3%83%95%E3%82%A9%E3%83%AB%E3%83%88%E5%80%A4%E3%81%AB%E6%88%BB%E3%81%9B%E3%81%AA%E3%81%84/