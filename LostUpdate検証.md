# LostUpdate検証
MySQLのMVCCにより、LostUpdateという現象が起きうる可能性が参考リンクから判明した。<br/>
よってAurora環境でLostUpdateが起こるかどうか検証した。

## 検証環境
- Aurora Versoin: 2.04.8
- MySQL Version: 5.7.12

## 検証

````
検証テーブル
CREATE TABLE t (
a INT UNSIGNED NOT NULL PRIMARY KEY,
b INT NOT NULL
) ENGINE=INNODB;
````
2つのトランザクションで確認する

````
T1:                                                          | T2:
MYSQL [xxxxxxxxxxxx] > CREATE TABLE t (                      |
    -> a INT UNSIGNED NOT NULL PRIMARY KEY,                  |
    -> b INT NOT NULL                                        |
    -> ) ENGINE=INNODB;                                      |
                                                             |
MYSQL [xxxxxxxxxxxx] > INSERT INTO t VALUES(1, 100);         |
MYSQL [xxxxxxxxxxxx] > commit;                               |
                                                             |
MYSQL [xxxxxxxxxxxx] > BEGIN;                                |
MYSQL [xxxxxxxxxxxx] > SELECT b INTO @x FROM t WHERE a = 1;  |
                                                             |
                                                             | MYSQL [xxxxxxxxxxxx] > BEGIN;
                                                             | MYSQL [xxxxxxxxxxxx] > SELECT b INTO @x FROM t WHERE a = 1;
                                                             | MYSQL [xxxxxxxxxxxx] > UPDATE t SET b = @x + 1 WHERE a = 1;
                                                             | MYSQL [xxxxxxxxxxxx] > commit;
                                                             |
MYSQL [xxxxxxxxxxxx] > UPDATE t SET b = @x + 10 WHERE a = 1; |
MYSQL [xxxxxxxxxxxx] > commit;                               |
MYSQL [xxxxxxxxxxxx] > SELECT * FROM t;                      |
+---+-----+                                                  |
| a | b   |                                                  |
+---+-----+                                                  |
| 1 | 110 |                                                  |
+---+-----+                                                  |
1 row in set (0.00 sec)                                      |
````

## 結論
本来、bのカラムには111が入ると思われるが、110が入っていることからも<br/>
MySQLのMVCCは最後に書き込んだもの勝ちとしている。

## 参考リンク
http://nippondanji.blogspot.com/2013/12/innodbrepeatable-readlocking-read.html