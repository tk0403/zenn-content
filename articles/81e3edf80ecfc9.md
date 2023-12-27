---
title: "MySQLからTiDBへ移行〜データの差分チェック&修正ツール：sync-diff-inspectorの使い方"
emoji: "📚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["TiDB","MySQL"]
published: false
---
# はじめに
２つの同じデータを持つはずのDB同士を実際に比較してみたいなぁーということはないでしょうか？

一番考えられるケースとしては、（何らかの理由で）DBを移行することを考える場合、移行"元"DBと移行"先"DBでデータが一致しているか？をデータを突き合わせ確認したくなると思います。

この記事では、MySQL（もしくは互換性のあるDB）同士を比較する際に利用可能なsync-diff-inspectorの使いかたを紹介します。
※sync-diff-inspectorはTiDBのコミュニティツールでもあるため、実際には移行元がMySQL、移行先がTiDBのときのデータチェック目的で一番利用されています（おそらく）

# 先にまとめ：sync-diff-inspectorとは？
sync-diff-inspectorにまつわる全体の概観は以下です。
![](/images/81e3edf80ecfc9/overview-sync-diff-inspector.png)

sync-diff-inspectorは、
* MySQLとTiDB双方で保存されているデータが一致しているかを確認する、つまりはデータの差分チェックツールです。  
  ※MySQL-MySQL、TiDB-TiDBでも利用することが可能です。
  ※比較対象のデータに更新がない状態で実行することを前提としています（ある静止点での比較を見るイメージ）

* チェックだけでなく差分を解消するためのSQLステートメントも生成してくれます。
  ※（だったらそもそも”移行ツール”としても使えるのでは？と思うかもしれませんが、さすがに差分があまりに大量だとしんどいと思うので、基本的には少量の修復が必要な場合の用途に限ったほうが良さそうです　※天井は未確認）

また、TiDBはNewSQL（シャーディング設計、実装不要でWriteスケール可能）を謳っていることもあるため、シャーディングされた複数のMySQLをひとつのTiDBへ移行する（まとめる）という発想もあり、
そういったケースにも応えられるように双方のDB間で
* 双方のDBで[異なるスキーマ/テーブル名のデータチェック](https://docs.pingcap.com/tidb/stable/route-diff)
* [MySQLではシャーディングされたテーブルをTiDBではひとつのテーブルにまとめる、という前提でのデータチェック](https://docs.pingcap.com/tidb/stable/shard-diff)
といったパターンもひろってくれる設計になっています。

詳しくは以下のDocsを確認してください。
https://docs.pingcap.com/tidb/stable/sync-diff-inspector-overview


# 使い方
## インストール
以下のDocsに記載の通り、tidb-community-toolkitに含まれています。

以下でダウンロードします。
M1 Mac/macOS Ventura v13.1 
```
https://download.pingcap.org/tidb-community-toolkit-{version}-linux-{arch}.tar.gz
```
`{version}`と`{arch}`はお使いの環境に合わせて適切なモノを指定してください。
例えば今回は以下のような指定をします。
```
https://download.pingcap.org/tidb-community-toolkit-v7.5.0-linux-arm64.tar.gz
```
ダウンロードしたら展開し、その中に`sync_diff_inspector`があることを確認します。
以降は展開したディレクトリで作業をしていきます。

ちなみにDocker Imageもあります。
```
docker pull pingcap/tidb-tools:latest
```

## 事前準備
事前に比較対象のDBであるMySQLとTiDBはすでにある前提で話しを進めます。
なお、ここでは同じローカルにそれぞれ用意しています。
※TiDBについてはtiup playgroundを利用しています

その他に用意すべきは以下の２つとなります。
### 1. Outputファイルの出力先ディレクトリの作成
冒頭で示した通り、いくつかのoutputファイルが生成されますので、その出力先ディレクトリを用意しておきます。

### 2. Inputファイルの作成
`config.toml`ファイルを用意します。
ファイルの中身は以下の内容にします。
```yaml
# Diff Configuration.
######################### Global config #########################
export-fix-sql = true
# check data also
check-struct-only = false

######################### Datasource config #########################
[data-sources]
[data-sources.mysql1]
    host = localhost
    port = 3306
    user = "root"
    password = ""
    route-rules = ["rule1"]

[data-sources.tidb0]
    host = "127.0.0.1"
    port = 4000
    user = "root"
    password = ""
    # snapshot = "386902609362944000"

########################### Routes ###########################
# mapping rules
[routes]
[routes.rule1]
schema-pattern = "test"      # source db
target-schema = "test"         # target db

######################### Task config #########################
[task]
    output-dir = "./sync_diff_inspector_files/output/syncdifftest"
    source-instances = ["mysql1"]
    target-instance = "tidb0"
    # target tables to be checked
    target-check-tables = ["test.test*"]
```

## 実行
以下のコマンドを実行します。事前に設定した`config.toml`ファイルを指定します。
```
./sync_diff_inspector --config=./sync_diff_inspector_files/config.toml
```
## 結果確認


# sync-diff-inspector動作検証
## ケース①：テーブル構造が同じ/違う場合
MySQLとTiDBそれぞれに２つのテーブルを作成し、それぞれを比較します。

一方は（ここでは`test05`テーブル）テーブル構造として主キーのカラムは一致、別カラムも一致ケース、
もう一方は（ここでは`test06`テーブル）テーブル構造として主キーのカラムは一致、別カラムは不一致ケースとします。

どちらもcharset/collationは同じにします。（`CHARSET=utf8mb4`、`COLLATE=utf8mb4_bin`）
※TiDBはデフォルトが上記のパターンになります

### MySQL
MySQLに以下のテーブルとデータを挿入
```sql
mysql> create table test05(id int not null primary key auto_increment, name varchar(64) )DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin;
Query OK, 0 rows affected (0.02 sec)

mysql> create table test06(id int not null primary key auto_increment, name varchar(32) )DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin;
Query OK, 0 rows affected (0.01 sec)
```
挿入データ
```sql
mysql> insert into test05(name) values('AAA'),('bbb'),('CCC'),('ddd'),('eee');
Query OK, 5 rows affected (0.01 sec)
Records: 5  Duplicates: 0  Warnings: 0

mysql> insert into test06(name) values('AAA'),('bbb'),('CCC'),('ddd'),('eee');
Query OK, 5 rows affected (0.00 sec)
Records: 5  Duplicates: 0  Warnings: 0
```
### TiDB
TiDBに以下のテーブルとデータを挿入
```sql
mysql> create table test05(id int not null primary key auto_increment, name varchar(64) );
Query OK, 0 rows affected (0.10 sec)

mysql> create table test06(id int not null primary key auto_increment, name varchar(64) );
Query OK, 0 rows affected (0.10 sec)
```
挿入データ
```sql
mysql> insert into test05(name) values('AAA'),('bbb'),('ccc'),('DDD'),('fff');
Query OK, 5 rows affected (0.01 sec)
Records: 5  Duplicates: 0  Warnings: 0

mysql> insert into test06(name) values('AAA'),('bbb'),('ccc'),('DDD'),('fff');
Query OK, 5 rows affected (0.01 sec)
Records: 5  Duplicates: 0  Warnings: 0
```
### データを確認（差分の目視チェック）
挿入するデータは`test05`、`test06`は同一とします。
ただし、MySQLとTiDBでは少し違うデータを挿入します。

ではデータを確認。MySQL。
```sql
mysql> select * from test05;
+----+------+
| id | name |
+----+------+
|  1 | AAA  |
|  2 | bbb  |
|  3 | CCC  |
|  4 | ddd  |
|  5 | eee  |
+----+------+
5 rows in set (0.00 sec)

mysql> select * from test06;
+----+------+
| id | name |
+----+------+
|  1 | AAA  |
|  2 | bbb  |
|  3 | CCC  |
|  4 | ddd  |
|  5 | eee  |
+----+------+
5 rows in set (0.00 sec)
```

TiDB
```sql
mysql> select * from test05;
+----+------+
| id | name |
+----+------+
|  1 | AAA  |
|  2 | bbb  |
|  3 | ccc  |
|  4 | DDD  |
|  5 | fff  |
+----+------+
5 rows in set (0.01 sec)

mysql> select * from test06;
+----+------+
| id | name |
+----+------+
|  1 | AAA  |
|  2 | bbb  |
|  3 | ccc  |
|  4 | DDD  |
|  5 | fff  |
+----+------+
5 rows in set (0.00 sec)
```

### sync-diff-inspector実行、結果
実行時に読み込ませるtomlファイルの確認（`sync-diff-config.toml`）です。
```yaml
# Diff Configuration.
######################### Global config #########################
export-fix-sql = true
# check data also
check-struct-only = false

######################### Datasource config #########################
[data-sources]
[data-sources.mysql1]
    host = <hostname>
    port = 3306
    user = "root"
    password = <password>
    route-rules = ["rule1"]

[data-sources.tidb0]
    host = "127.0.0.1"
    port = 4000
    user = "root"
    password = ""
    # snapshot = "386902609362944000"

########################### Routes ###########################
# mapping rules
[routes]
[routes.rule1]
schema-pattern = "test"      # source db
target-schema = "test"         # target db

######################### Task config #########################
[task]
    output-dir = "./output/syncdifftest10"
    source-instances = ["mysql1"]
    target-instance = "tidb0"
    # target tables to be checked
    target-check-tables = ["test.test*"]
```
プロンプトで実行
```
./sync_diff_inspector --config=../sync-diff-config.toml
```
以下のプロンプト上での出力結果を確認すると、`test05`はテーブル構造チェックを`PASS`し、データチェックで`failure`（差分あり）となっています。
一方で`test06`はテーブル構造チェックで`failure`となっており、後続のデータチェックプロセスはスキップされています。

```
A total of 2 tables need to be compared

Comparing the table structure of ``test`.`test06`` ... failure
Comparing the table structure of ``test`.`test05`` ... equivalent
Comparing the table data of ``test`.`test05`` ... failure
_____________________________________________________________________________
Progress [============================================================>] 100% 0/0
The structure of `test`.`test06` is not equal, and data-check is skipped
The data of `test`.`test05` is not equal

The rest of tables are all equal.
The patch file has been generated in 
        'output/syncdifftest10/fix-on-tidb0/'
You can view the comparision details through './output/syncdifftest10/sync_diff.log'
```

それでは、生成されたアウトプットファイルを確認していきます。
アウトプット１：`summary.txt`
```
Comparison Result

The table structure and data in following tables are equivalent

The following tables contains inconsistent data

+-----------------+--------------------+----------------+---------+-----------+
|      TABLE      | STRUCTURE EQUALITY | DATA DIFF ROWS | UPCOUNT | DOWNCOUNT |
+-----------------+--------------------+----------------+---------+-----------+
| `test`.`test06` | false              | +0/-0          |       0 |         0 |
| `test`.`test05` | true               | +3/-3          |       5 |         5 |
+-----------------+--------------------+----------------+---------+-----------+

Time Cost: 21.710874ms
Average Speed: 0.005271MB/s
```
アウトプット２：`sync_diff.log`　※`ERROR`と`WARN`のみ抜粋
```
[2023/12/20 14:21:56.546 +00:00] [ERROR] [utils.go:368] ["column properties not compatible"] ["upstream table"=test06] ["column name"=name] ["column type"=15] ["downstream table"=test06] ["column name"=name] ["column type"=15] [stack="github.com/pingcap/tidb-tools/sync_diff_inspector/utils.CompareStruct\n\t/home/jenkins/agent/workspace/build-common/go/src/github.com/pingcap/tidb-tools/sync_diff_inspector/utils/utils.go:368\nmain.(*Diff).compareStruct\n\t/home/jenkins/agent/workspace/build-common/go/src/github.com/pingcap/tidb-tools/sync_diff_inspector/diff.go:318\nmain.(*Diff).StructEqual\n\t/home/jenkins/agent/workspace/build-common/go/src/github.com/pingcap/tidb-tools/sync_diff_inspector/diff.go:302\nmain.checkSyncState\n\t/home/jenkins/agent/workspace/build-common/go/src/github.com/pingcap/tidb-tools/sync_diff_inspector/main.go:125\nmain.main\n\t/home/jenkins/agent/workspace/build-common/go/src/github.com/pingcap/tidb-tools/sync_diff_inspector/main.go:104\nruntime.main\n\t/usr/local/go/src/runtime/proc.go:250"]
・
・
・
[2023/12/20 14:21:56.555 +00:00] [WARN] [utils.go:503] ["find different row"] [column=name] [row1="{ id: 3, name: CCC,  }"] [row2="{ id: 3, name: ccc,  }"]
[2023/12/20 14:21:56.555 +00:00] [WARN] [utils.go:503] ["find different row"] [column=name] [row1="{ id: 4, name: ddd,  }"] [row2="{ id: 4, name: DDD,  }"]
[2023/12/20 14:21:56.555 +00:00] [WARN] [utils.go:503] ["find different row"] [column=name] [row1="{ name: eee, id: 5,  }"] [row2="{ id: 5, name: fff,  }"]
・
・
・
[2023/12/20 14:21:56.565 +00:00] [WARN] [main.go:105] ["check failed!!!"]
```
アウトプット３：`test:test05:1:0-0:0.sql`
```sql
-- table: test.test05
-- range in sequence: Full
/*
  DIFF COLUMNS ╏ `NAME`  
╍╍╍╍╍╍╍╍╍╍╍╍╍╍╍╋╍╍╍╍╍╍╍╍╍
  source data  ╏ 'CCC'   
╍╍╍╍╍╍╍╍╍╍╍╍╍╍╍╋╍╍╍╍╍╍╍╍╍
  target data  ╏ 'ccc'   
╍╍╍╍╍╍╍╍╍╍╍╍╍╍╍╋╍╍╍╍╍╍╍╍╍
*/
REPLACE INTO `test`.`test05`(`id`,`name`) VALUES (3,'CCC');
/*
  DIFF COLUMNS ╏ `NAME`  
╍╍╍╍╍╍╍╍╍╍╍╍╍╍╍╋╍╍╍╍╍╍╍╍╍
  source data  ╏ 'ddd'   
╍╍╍╍╍╍╍╍╍╍╍╍╍╍╍╋╍╍╍╍╍╍╍╍╍
  target data  ╏ 'DDD'   
╍╍╍╍╍╍╍╍╍╍╍╍╍╍╍╋╍╍╍╍╍╍╍╍╍
*/
REPLACE INTO `test`.`test05`(`id`,`name`) VALUES (4,'ddd');
/*
  DIFF COLUMNS ╏ `NAME`  
╍╍╍╍╍╍╍╍╍╍╍╍╍╍╍╋╍╍╍╍╍╍╍╍╍
  source data  ╏ 'eee'   
╍╍╍╍╍╍╍╍╍╍╍╍╍╍╍╋╍╍╍╍╍╍╍╍╍
  target data  ╏ 'fff'   
╍╍╍╍╍╍╍╍╍╍╍╍╍╍╍╋╍╍╍╍╍╍╍╍╍
*/
REPLACE INTO `test`.`test05`(`id`,`name`) VALUES (5,'eee');
```

## ケース②：スプレッドシートのパターン test10とtest11
※スプレッドシートのパターン

テーブル構造は同じ(charsetも同じだが/collationは違う)

### MySQL
```sql
mysql> create table test10(id varchar(64) not null primary key)DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci;
Query OK, 0 rows affected (0.00 sec)

mysql> create table test11(id varchar(64) not null primary key)DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin;
Query OK, 0 rows affected (0.01 sec)
```
```sql
mysql> insert into test10(id) values('a'),('B');
Query OK, 2 rows affected (0.00 sec)
Records: 2  Duplicates: 0  Warnings: 0

mysql> insert into test11(id) values('A'),('a'),('B'),('b');
Query OK, 4 rows affected (0.00 sec)
Records: 4  Duplicates: 0  Warnings: 0
```
当たり前ですが、`test10`に以下のレコードは入りません。
```sql
mysql> insert into test10(id) values('A');
ERROR 1062 (23000): Duplicate entry 'A' for key 'PRIMARY'
mysql> insert into test10(id) values('b');
ERROR 1062 (23000): Duplicate entry 'b' for key 'PRIMARY'
```

### TiDB
```sql
mysql> create table test10(id varchar(64) not null primary key)DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin;
Query OK, 0 rows affected (0.09 sec)

mysql> create table test11(id varchar(64) not null primary key)DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci;
Query OK, 0 rows affected (0.09 sec)
```
```sql
mysql> insert into test10(id) values('A'),('a'),('B'),('b');
Query OK, 4 rows affected (0.00 sec)
Records: 4  Duplicates: 0  Warnings: 0

mysql> insert into test11(id) values('a'),('B');
Query OK, 2 rows affected (0.01 sec)
Records: 2  Duplicates: 0  Warnings: 0
```
もちろんTiDBでもMySQLと同様に以下のようなレコードは入りません。
```sql
mysql> insert into test11(id) values('A'),('b');
ERROR 1062 (23000): Duplicate entry 'A' for key 'test11.PRIMARY'
mysql> insert into test11(id) values('b');
ERROR 1062 (23000): Duplicate entry 'b' for key 'test11.PRIMARY'
```
### データを確認（差分の目視チェック）
挿入するデータは`test10`、`test11`

ではデータを確認。MySQL。
```sql
mysql> select * from test10;
+----+
| id |
+----+
| a  |
| B  |
+----+
2 rows in set (0.00 sec)

mysql> select * from test11;
+----+
| id |
+----+
| A  |
| B  |
| a  |
| b  |
+----+
4 rows in set (0.00 sec)
```

TiDB
```sql
mysql> select * from test10;
+----+
| id |
+----+
| A  |
| B  |
| a  |
| b  |
+----+
4 rows in set (0.00 sec)

mysql> select * from test11;
+----+
| id |
+----+
| a  |
| B  |
+----+
2 rows in set (0.00 sec)
```

### sync-diff-inspector実行、結果
実行時に読み込ませるtomlファイルの確認（`sync-diff-config.toml`）です。
```
```
プロンプトで実行
```
./sync_diff_inspector --config=../sync-diff-config.toml
```
以下のプロンプト上での出力結果を確認すると、`test10`は

```
A total of 2 tables need to be compared

Comparing the table structure of ``test`.`test11`` ... equivalent
Comparing the table structure of ``test`.`test10`` ... equivalent
Comparing the table data of ``test`.`test11`` ... failure
Comparing the table data of ``test`.`test10`` ... failure
_____________________________________________________________________________
Progress [============================================================>] 100% 0/0
The data of `test`.`test11` is not equal
The data of `test`.`test10` is not equal

The rest of tables are all equal.
The patch file has been generated in 
        'output/syncdifftest10and11/fix-on-tidb0/'
You can view the comparision details through './output/syncdifftest10and11/sync_diff.log'
```
アウトプット１：`summary.txt`
```
Comparison Result

The table structure and data in following tables are equivalent

The following tables contains inconsistent data

+-----------------+--------------------+----------------+---------+-----------+
|      TABLE      | STRUCTURE EQUALITY | DATA DIFF ROWS | UPCOUNT | DOWNCOUNT |
+-----------------+--------------------+----------------+---------+-----------+
| `test`.`test11` | true               | +3/-1          |       4 |         2 |
| `test`.`test10` | true               | +1/-3          |       2 |         4 |
+-----------------+--------------------+----------------+---------+-----------+

Time Cost: 20.6737ms
Average Speed: 0.000554MB/s
```
アウトプット２：`sync_diff.log`　※`ERROR`と`WARN`のみ抜粋
```
[2023/12/20 15:43:27.097 +00:00] [WARN] [utils.go:311] ["Ignoring collation differences"] ["column name"=id] ["collation source"=utf8mb4_bin] ["collation target"=utf8mb4_general_ci]
・
・
・
[2023/12/20 15:43:27.099 +00:00] [INFO] [tidb.go:57] ["failed to build bucket iterator, fall back to use random iterator"] [error="primary key on id in buckets info not found"] [errorVerbose="primary key on id in buckets info not found\ngithub.com/pingcap/errors.NotFoundf\n\t/go/pkg/mod/github.com/pingcap/errors@v0.11.5-0.20211224045212-9687c2b0f87c/juju_adaptor.go:117\ngithub.com/pingcap/tidb-tools/pkg/dbutil.GetBucketsInfo\n\t/home/jenkins/agent/workspace/build-common/go/src/github.com/pingcap/tidb-tools/pkg/dbutil/common.go:576\ngithub.com/pingcap/tidb-tools/sync_diff_inspector/splitter.(*BucketIterator).init\n\t/home/jenkins/agent/workspace/build-common/go/src/github.com/pingcap/tidb-tools/sync_diff_inspector/splitter/bucket.go:139\ngithub.com/pingcap/tidb-tools/sync_diff_inspector/splitter.NewBucketIteratorWithCheckpoint\n\t/home/jenkins/agent/workspace/build-common/go/src/github.com/pingcap/tidb-tools/sync_diff_inspector/splitter/bucket.go:80\ngithub.com/pingcap/tidb-tools/sync_diff_inspector/source.(*TiDBTableAnalyzer).AnalyzeSplitter\n\t/home/jenkins/agent/workspace/build-common/go/src/github.com/pingcap/tidb-tools/sync_diff_inspector/source/tidb.go:53\ngithub.com/pingcap/tidb-tools/sync_diff_inspector/source.(*ChunksIterator).produceChunks.func3\n\t/home/jenkins/agent/workspace/build-common/go/src/github.com/pingcap/tidb-tools/sync_diff_inspector/source/chunks_iter.go:133\ngithub.com/pingcap/tidb-tools/sync_diff_inspector/utils.(*WorkerPool).Apply.func1\n\t/home/jenkins/agent/workspace/build-common/go/src/github.com/pingcap/tidb-tools/sync_diff_inspector/utils/utils.go:75\nruntime.goexit\n\t/usr/local/go/src/runtime/asm_amd64.s:1571"]
[2023/12/20 15:43:27.100 +00:00] [INFO] [tidb.go:57] ["failed to build bucket iterator, fall back to use random iterator"] [error="primary key on id in buckets info not found"] [errorVerbose="primary key on id in buckets info not found\ngithub.com/pingcap/errors.NotFoundf\n\t/go/pkg/mod/github.com/pingcap/errors@v0.11.5-0.20211224045212-9687c2b0f87c/juju_adaptor.go:117\ngithub.com/pingcap/tidb-tools/pkg/dbutil.GetBucketsInfo\n\t/home/jenkins/agent/workspace/build-common/go/src/github.com/pingcap/tidb-tools/pkg/dbutil/common.go:576\ngithub.com/pingcap/tidb-tools/sync_diff_inspector/splitter.(*BucketIterator).init\n\t/home/jenkins/agent/workspace/build-common/go/src/github.com/pingcap/tidb-tools/sync_diff_inspector/splitter/bucket.go:139\ngithub.com/pingcap/tidb-tools/sync_diff_inspector/splitter.NewBucketIteratorWithCheckpoint\n\t/home/jenkins/agent/workspace/build-common/go/src/github.com/pingcap/tidb-tools/sync_diff_inspector/splitter/bucket.go:80\ngithub.com/pingcap/tidb-tools/sync_diff_inspector/source.(*TiDBTableAnalyzer).AnalyzeSplitter\n\t/home/jenkins/agent/workspace/build-common/go/src/github.com/pingcap/tidb-tools/sync_diff_inspector/source/tidb.go:53\ngithub.com/pingcap/tidb-tools/sync_diff_inspector/source.(*ChunksIterator).produceChunks.func3\n\t/home/jenkins/agent/workspace/build-common/go/src/github.com/pingcap/tidb-tools/sync_diff_inspector/source/chunks_iter.go:133\ngithub.com/pingcap/tidb-tools/sync_diff_inspector/utils.(*WorkerPool).Apply.func1\n\t/home/jenkins/agent/workspace/build-common/go/src/github.com/pingcap/tidb-tools/sync_diff_inspector/utils/utils.go:75\nruntime.goexit\n\t/usr/local/go/src/runtime/asm_amd64.s:1571"]
・
・
・
[2023/12/20 15:43:27.103 +00:00] [WARN] [utils.go:507] ["target lack data"] [row="{ id: A,  }"]
[2023/12/20 15:43:27.103 +00:00] [WARN] [utils.go:507] ["target lack data"] [row="{ id: B,  }"]
[2023/12/20 15:43:27.103 +00:00] [WARN] [utils.go:505] ["target had superfluous data"] [row="{ id: B,  }"]
[2023/12/20 15:43:27.104 +00:00] [WARN] [utils.go:505] ["target had superfluous data"] [row="{ id: A,  }"]
[2023/12/20 15:43:27.104 +00:00] [WARN] [utils.go:505] ["target had superfluous data"] [row="{ id: B,  }"]
[2023/12/20 15:43:27.104 +00:00] [WARN] [utils.go:507] ["target lack data"] [row="{ id: B,  }"]
・
・
・
[2023/12/20 15:43:27.116 +00:00] [WARN] [main.go:105] ["check failed!!!"]
```
アウトプット３-1：`test:test10:1:0-0:0.sql`
```sql
-- table: test.test10
-- range in sequence: Full
DELETE FROM `test`.`test10` WHERE `id` = 'A' LIMIT 1;
DELETE FROM `test`.`test10` WHERE `id` = 'B' LIMIT 1;
REPLACE INTO `test`.`test10`(`id`) VALUES ('B');
DELETE FROM `test`.`test10` WHERE `id` = 'b' LIMIT 1;
```
アウトプット３-2：`test:test11:0:0-0:0.sql`
```sql
-- table: test.test11
-- range in sequence: Full
REPLACE INTO `test`.`test11`(`id`) VALUES ('A');
REPLACE INTO `test`.`test11`(`id`) VALUES ('B');
DELETE FROM `test`.`test11` WHERE `id` = 'B' LIMIT 1;
REPLACE INTO `test`.`test11`(`id`) VALUES ('b');
```


## ケース②：スプレッドシートのパターン test20
※スプレッドシートのパターン

テーブル構造は同じ(charsetも同じだが/collationは違う)

### MySQL
```sql
mysql> create table test20(id varchar(64) not null primary key)DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci;
Query OK, 0 rows affected (0.01 sec)

mysql> insert into test20(id) values('a'),('B');
Query OK, 2 rows affected (0.00 sec)
Records: 2  Duplicates: 0  Warnings: 0
```

### TiDB
```sql
mysql> create table test20(id varchar(64) not null primary key)DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin;
Query OK, 0 rows affected (0.10 sec)

mysql> insert into test20(id) values('A'),('a'),('B'),('b');
Query OK, 4 rows affected (0.00 sec)
Records: 4  Duplicates: 0  Warnings: 0
```
### データを確認（差分の目視チェック）
挿入するデータは`test10`、`test11`

ではデータを確認。MySQL。
```sql
mysql> select * from test20;
+----+
| id |
+----+
| a  |
| B  |
+----+
2 rows in set (0.00 sec)
```

TiDB
```sql
mysql> select * from test20;
+----+
| id |
+----+
| A  |
| B  |
| a  |
| b  |
+----+
4 rows in set (0.00 sec)
```

### sync-diff-inspector実行、結果
実行時に読み込ませるtomlファイルの確認（`sync-diff-config_with_collation_20.toml`）です。
```yaml
# Diff Configuration.
######################### Global config #########################
export-fix-sql = true
# check data also
check-struct-only = false

######################### Datasource config #########################
[data-sources]
[data-sources.mysql1]
    host = "172.10.60.39"
    port = 3306
    user = "root"
    password = "T@ichi0108"
    route-rules = ["rule1"]

[data-sources.tidb0]
    host = "127.0.0.1"
    port = 4000
    user = "root"
    password = ""
    # snapshot = "386902609362944000"

########################### Routes ###########################
# mapping rules
[routes]
[routes.rule1]
schema-pattern = "test"      # source db
target-schema = "test"         # target db
######################### Task config #########################
[task]
    output-dir = "./output/syncdifftest20"
    source-instances = ["mysql1"]
    target-instance = "tidb0"
    # target tables to be checked
    target-check-tables = ["test.test20"]
######################### Table config #########################
# Special configurations for specific tables. The tables to be configured must be in `task.target-check-tables`.
[table-configs.config1]
# The name of the target table, you can use regular expressions to match multiple tables, but one table is not allowed to be matched by multiple special configurations at the same time.
target-tables = ["test.test20"]
# (optional) Specifies the "collation" for the table. If not specified, this configuration can be deleted or be set as an empty string.
collation = "utf8mb4_general_ci"
```
プロンプトで実行
```
./sync_diff_inspector --config=../sync-diff-config_with_collation_20.toml
```
以下のプロンプト上での出力結果を確認すると、`test20`は

```
A total of 1 tables need to be compared

Comparing the table structure of ``test`.`test20`` ... equivalent
Comparing the table data of ``test`.`test20`` ... failure
_____________________________________________________________________________
Progress [============================================================>] 100% 0/0
The data of `test`.`test20` is not equal

The rest of tables are all equal.
The patch file has been generated in 
        'output/syncdifftest20/fix-on-tidb0/'
You can view the comparision details through './output/syncdifftest20/sync_diff.log'
```
アウトプット１：`summary.txt`
```
Comparison Result

The table structure and data in following tables are equivalent

The following tables contains inconsistent data

+-----------------+--------------------+----------------+---------+-----------+
|      TABLE      | STRUCTURE EQUALITY | DATA DIFF ROWS | UPCOUNT | DOWNCOUNT |
+-----------------+--------------------+----------------+---------+-----------+
| `test`.`test20` | true               | +1/-3          |       2 |         4 |
+-----------------+--------------------+----------------+---------+-----------+

Time Cost: 12.374444ms
Average Speed: 0.000617MB/s
```
アウトプット２：`sync_diff.log`　※`ERROR`と`WARN`のみ抜粋
```
[2023/12/20 16:32:23.797 +00:00] [WARN] [utils.go:505] ["target had superfluous data"] [row="{ id: A,  }"]
[2023/12/20 16:32:23.797 +00:00] [WARN] [utils.go:505] ["target had superfluous data"] [row="{ id: B,  }"]
[2023/12/20 16:32:23.797 +00:00] [WARN] [utils.go:507] ["target lack data"] [row="{ id: B,  }"]
・
・
・
[2023/12/20 16:32:23.804 +00:00] [WARN] [main.go:105] ["check failed!!!"]
```
アウトプット３：`test:test20:0:0-0:0.sql`
```sql
-- table: test.test20
-- range in sequence: Full
DELETE FROM `test`.`test20` WHERE `id` = 'A' LIMIT 1;
DELETE FROM `test`.`test20` WHERE `id` = 'B' LIMIT 1;
REPLACE INTO `test`.`test20`(`id`) VALUES ('B');
DELETE FROM `test`.`test20` WHERE `id` = 'b' LIMIT 1;
```

## ケース③：スプレッドシートのパターン test21
※スプレッドシートのパターン

テーブル構造は同じ(charsetも同じだが/collationは違う)

### MySQL
```sql
mysql> create table test21(id varchar(64) not null primary key)DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin;
Query OK, 0 rows affected (0.01 sec)

mysql> insert into test21(id) values('A'),('a'),('B'),('b');
Query OK, 4 rows affected (0.00 sec)
Records: 4  Duplicates: 0  Warnings: 0
```

### TiDB
```sql
mysql> create table test21(id varchar(64) not null primary key)DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci;
Query OK, 0 rows affected (0.10 sec)

mysql> insert into test21(id) values('a'),('B');
Query OK, 2 rows affected (0.00 sec)
Records: 2  Duplicates: 0  Warnings: 0
```
### データを確認（差分の目視チェック）
挿入するデータは`test21`

ではデータを確認。MySQL。
```sql
mysql> select * from test21;
+----+
| id |
+----+
| A  |
| B  |
| a  |
| b  |
+----+
4 rows in set (0.00 sec)
```

TiDB
```sql
mysql> select * from test21;
+----+
| id |
+----+
| a  |
| B  |
+----+
2 rows in set (0.00 sec)
```

### sync-diff-inspector実行、結果
実行時に読み込ませるtomlファイルの確認（`sync-diff-config_with_collation_21.toml`）です。
```yaml
# Diff Configuration.
######################### Global config #########################
export-fix-sql = true
# check data also
check-struct-only = false

######################### Datasource config #########################
[data-sources]
[data-sources.mysql1]
    host = "172.10.60.39"
    port = 3306
    user = "root"
    password = "T@ichi0108"
    route-rules = ["rule1"]

[data-sources.tidb0]
    host = "127.0.0.1"
    port = 4000
    user = "root"
    password = ""
    # snapshot = "386902609362944000"

########################### Routes ###########################
# mapping rules
[routes]
[routes.rule1]
schema-pattern = "test"      # source db
target-schema = "test"         # target db
######################### Task config #########################
[task]
    output-dir = "./output/syncdifftest21"
    source-instances = ["mysql1"]
    target-instance = "tidb0"
    # target tables to be checked
    target-check-tables = ["test.test21"]
######################### Table config #########################
# Special configurations for specific tables. The tables to be configured must be in `task.target-check-tables`.
[table-configs.config1]
# The name of the target table, you can use regular expressions to match multiple tables, but one table is not allowed to be matched by multiple special configurations at the same time.
target-tables = ["test.test21"]
# (optional) Specifies the "collation" for the table. If not specified, this configuration can be deleted or be set as an empty string.
collation = "utf8mb4_bin"
```
プロンプトで実行
```
./sync_diff_inspector --config=../sync-diff-config_with_collation_21.toml
```
以下のプロンプト上での出力結果を確認すると、`test20`は

```
A total of 1 tables need to be compared

Comparing the table structure of ``test`.`test21`` ... equivalent
Comparing the table data of ``test`.`test21`` ... failure
_____________________________________________________________________________
Progress [============================================================>] 100% 0/0
The data of `test`.`test21` is not equal

The rest of tables are all equal.
The patch file has been generated in 
        'output/syncdifftest21/fix-on-tidb0/'
You can view the comparision details through './output/syncdifftest21/sync_diff.log'
```
アウトプット１：`summary.txt`
```
Comparison Result

The table structure and data in following tables are equivalent

The following tables contains inconsistent data

+-----------------+--------------------+----------------+---------+-----------+
|      TABLE      | STRUCTURE EQUALITY | DATA DIFF ROWS | UPCOUNT | DOWNCOUNT |
+-----------------+--------------------+----------------+---------+-----------+
| `test`.`test21` | true               | +3/-1          |       4 |         2 |
+-----------------+--------------------+----------------+---------+-----------+

Time Cost: 12.816402ms
Average Speed: 0.000298MB/s
```
アウトプット２：`sync_diff.log`　※`ERROR`と`WARN`のみ抜粋
```
[2023/12/20 16:43:36.040 +00:00] [WARN] [utils.go:311] ["Ignoring collation differences"] ["column name"=id] ["collation source"=utf8mb4_bin] ["collation target"=utf8mb4_general_ci]
・
・
・
[2023/12/20 16:43:36.044 +00:00] [WARN] [utils.go:507] ["target lack data"] [row="{ id: A,  }"]
[2023/12/20 16:43:36.044 +00:00] [WARN] [utils.go:507] ["target lack data"] [row="{ id: B,  }"]
[2023/12/20 16:43:36.044 +00:00] [WARN] [utils.go:505] ["target had superfluous data"] [row="{ id: B,  }"]
[2023/12/20 16:43:36.044 +00:00] [INFO] [diff.go:722] ["write sql channel closed"]
[2023/12/20 16:43:36.044 +00:00] [INFO] [diff.go:713] ["close writeSQLs goroutine"]
[2023/12/20 16:43:36.044 +00:00] [INFO] [diff.go:403] ["Stop do checkpoint"]
[2023/12/20 16:43:36.045 +00:00] [INFO] [checkpoints.go:225] ["save checkpoint"] [chunk="{\"state\":\"failed\",\"chunk-range\":{\"index\":{\"table-index\":0,\"bucket-index-left\":0,\"bucket-index-right\":0,\"chunk-index\":0,\"chunk-count\":1},\"type\":2,\"bounds\":[],\"is-first\":false,\"is-last\":false,\"where\":\"((TRUE) AND (TRUE))\",\"args\":null},\"index-id\":0}"] [state=failed]
[2023/12/20 16:43:36.045 +00:00] [INFO] [diff.go:377] ["close handleCheckpoint goroutine"]
[2023/12/20 16:43:36.050 +00:00] [INFO] [main.go:114] ["check data finished"] [cost=23.280609ms]
[2023/12/20 16:43:36.050 +00:00] [WARN] [main.go:105] ["check failed!!!"]
```
アウトプット３：`test:test21:0:0-0:0.sql`
```sql
-- table: test.test21
-- range in sequence: Full
REPLACE INTO `test`.`test21`(`id`) VALUES ('A');
REPLACE INTO `test`.`test21`(`id`) VALUES ('B');
DELETE FROM `test`.`test21` WHERE `id` = 'B' LIMIT 1;
REPLACE INTO `test`.`test21`(`id`) VALUES ('b');
```
