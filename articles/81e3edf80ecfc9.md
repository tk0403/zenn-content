---
title: "sync-diff-inspectorの使い方：MySQL（or TiDB）でデータの差分チェック&修正ツール"
emoji: "📚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["TiDB","MySQL"]
published: true
---
２つの同じデータを持つはずのDB同士を実際に比較してみたいなぁということはないでしょうか？
レプリケーションであったり、（何らかの理由で）DBを移行することを考える場合にはたまにあるかと思います（めったにない）。

この記事では、MySQL（もしくはMySQLと互換性のあるDB）同士を比較する際に利用可能なsync-diff-inspectorの使いかたを紹介します。

※sync-diff-inspectorはTiDBのコミュニティツールとして提供されているため、実際には移行元がMySQL、移行先がTiDBのときのデータチェック目的で一番利用されています（おそらく）

https://github.com/pingcap/tidb-tools

# sync-diff-inspectorとは？
以下のイメージのように
1. 比較元（MySQL）と比較先（TiDB）双方のテーブル構造を比較チェック
2. 構造が同一のテーブルのデータを比較チェック
3. （2.の）差分解消のためのSQLを生成する
ツールです。実行後の出力は、上記の比較結果のサマリーと実行ログ、修正用SQLファイルの3つです。
:::message
MySQL-MySQL、TiDB-TiDBでも利用することが可能です。
:::
![](/images/81e3edf80ecfc9/overview-sync-diff-inspector.png)

実行時の前提として、比較対象のデータに更新がない状態が想定されています。（理想的にはある静止点で比較するイメージ）

また比較元を正として、比較先を修正するためのSQLステートメントも生成します
:::message
だったら”移行ツール”としても使えるのでは？と思うかもしれませんが、さすがに差分があまりに大量だとしんどいと思うので、基本的には少量の修復が必要な場合の用途に限ったほうが良さそうです。
:::

また、TiDBはNewSQL（シャーディング設計、実装不要でWriteスケール可能）を謳っていることもあるため、シャーディングされた複数のMySQLをひとつのTiDBへ移行する（まとめる）という発想もあり、
そういったケースにも応えられるように双方のDB間で
* 双方のDBで[異なるスキーマ/テーブル名のデータチェック](https://docs.pingcap.com/tidb/stable/route-diff)
* [MySQLではシャーディングされたテーブルをTiDBではひとつのテーブルにまとめる、という前提でのデータチェック](https://docs.pingcap.com/tidb/stable/shard-diff)
などといったパターンもひろってくれる設計になっています。

詳しくは以下のDocsを確認してください。
https://docs.pingcap.com/tidb/stable/sync-diff-inspector-overview


# 今回の前提
## 環境
sync-diff-inspectorの大枠の挙動や使い方を確認する目的のため、
今回は同一EC2インスタンス上にあるMySQL、TiDB（tiup playground）と同じマシンにsync-diff-inspectorをインストールして確認していきます。
（ちょうど手元にあった環境を利用）

MySQLとTiDBのバージョンは以下の通り。

* MySQL
```sh
% mysql --version
mysql  Ver 14.14 Distrib 5.7.39, for Linux (x86_64) using  EditLine wrapper
```
* TiDB（tiup playgroundを利用）

```sh
% tiup playground v6.5.7  
```
※tiup playgroundについては[こちらの記事](https://zenn.dev/shigeyuki/articles/8663e1598ecfa5)を参考にどうぞ
※ちょっと古いバージョンですが。。。

sync-diff-inspectorのインストール手順は"使い方"の段でまとめます。

## 比較パターン（MySQLとTiDBのデータ準備）
今回はsync-diff-inspectorの挙動をざっくり抑えるために以下のパターンで検証してみます。

| Table Name | 構造 | データ | 期待結果 | 
|----|----|----|----| 
| `test01` | 一致 | 一致 |一致していることがわかる|
| `test02` | 不一致 | 一致 |テーブル構造が不一致であることがわかる|
| `test03` | 一致 | 不一致 |テーブル構造は一致しているが、一致していないデータの対象とその差分を修正するSQLが出力される|


データが不一致となるケースである`test03`というテーブルには以下のようなパターンでMySQL、TiDB双方にレコードを入れておきます。

|シナリオ| MySQL | TiDB |
|----|----|----|
| 完全一致 | `id=1,name="a"` | `id=1,name="a"` |
| PK一致、列不一致 | `id=2,name="b1"` | `id=2,name="b2"` |
| PK不一致（MySQL：あり / TiDB：なし） | `id=3,name="c"` ||
| PK不一致（MySQL：なし / TiDB：あり） || `id=4,name="d"` |


ここからは上記に沿ったデータを作成していきます。
### 比較データの準備
#### テーブル構造（CREATE TABLE）

`test01`：MySQL/TiDB共通
```sql
CREATE TABLE test01(id INT NOT NULL PRIMARY KEY AUTO_INCREMENT, name VARCHAR(64) )DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin;
```
`test02`：MySQL
```sql
CREATE TABLE test02(id INT NOT NULL PRIMARY KEY AUTO_INCREMENT, name VARCHAR(64) )DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin;
```
`test02`：TiDB
```sql
CREATE TABLE test02(id INT NOT NULL PRIMARY KEY AUTO_INCREMENT, name VARCHAR(32) )DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin;
```

`test03`：MySQL/TiDB共通
```sql
CREATE TABLE test03(id INT NOT NULL PRIMARY KEY AUTO_INCREMENT, name VARCHAR(64) )DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin;
```

#### 挿入データ（INSERT）
`test01`：MySQL/TiDB共通
```sql
INSERT INTO test01(name) values('AAA'),('bbb'),('CCC'),('ddd'),('eee');
```

`test02`：MySQL/TiDB共通
```sql
INSERT INTO test02(name) values('AAA'),('bbb'),('CCC'),('ddd'),('eee');
```

`test03`：MySQL
```sql
INSERT INTO test03(name) values('a'),('b1'),('c');
```
`test03`：TiDB
```sql
INSERT INTO test03(name) values('a'),('b2');
INSERT INTO test03(id, name) values(4,'d');
```
### データ確認
若干余談ですが、GUIのSQLクライアントであるDBeaverもTiDBへの接続サポートしています。

他にもあります↓↓↓

https://docs.pingcap.com/tidb/stable/dev-guide-third-party-support#gui

各テーブルはこんな状態です。

#### `test01`：左：MySQL　右：TiDB
![](/images/81e3edf80ecfc9/test01.png)
#### `test02`：左：MySQL　右：TiDB
![](/images/81e3edf80ecfc9/test02.png)
#### `test03`：左：MySQL　右：TiDB
![](/images/81e3edf80ecfc9/test03.png)




# sync-diff-inspectorの使い方
## sync-diff-inspectorのインストール
Docsに記載のある通り、sync-diff-inspectorはtidb-community-toolkitに含まれており、
```
https://download.pingcap.org/tidb-community-toolkit-{version}-linux-{arch}.tar.gz
```
`{version}`と`{arch}`はお使いの環境に合わせて適切なモノを指定してください。

例えば今回は以下のモノを利用しています。※だいぶ前にインストールしていたので少々古いものになっています
```
https://download.pingcap.org/tidb-community-toolkit-v6.2.0-linux-amd64.tar.gz
```

ダウンロードしたら展開し、その中に`sync_diff_inspector`があることを確認します。
以降は展開したディレクトリで作業をしていきます。

ちなみにDocker Imageはこちら。
```
docker pull pingcap/tidb-tools:latest
```

## 実行前の事前準備
### 1. Outputファイルの出力先ディレクトリの作成
冒頭で示した通り、いくつかのoutputファイルが生成されますので、その出力先ディレクトリを用意しておきます。

### 2. Inputファイルの作成
`config.toml`ファイルを用意します。
中身は概ね以下のような構成です。
* `Global config`:一般的な設定。修正用SQLファイルの出力要否やデータチェックスキップ（≒構造のみ比較）など。
* `Databases config`: 比較元、先含め比較対象となるデータベースのインスタンス設定。
* `Routes` (optional): 比較元と先でのテーブルのマッピングを指定する。比較元がシャーディングされているケースで利用。
* `Task config`:比較元と先の指定。テーブルとの関連性など。
* `Table config` (optional):テーブルに関してより詳細な設定（範囲や条件を指定する場合）

なお、`Databases config`に記載する各DBへの接続ユーザーには`SELECT`、`SHOW_DATABASES`、`RELOAD`の権限が必要です。

このファイルの詳細やサンプルとして以下も参考になるかと思います。
https://github.com/pingcap/tidb-tools/tree/master/sync_diff_inspector/config

今回ファイルの中身は以下のように最低限の内容にします。
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

########################### Routes ###########################
# mapping rules
[routes]
[routes.rule1]
schema-pattern = "test"      # source db
target-schema = "test"         # target db

######################### Task config #########################
[task]
    output-dir = "./output/syncdifftest"
    source-instances = ["mysql1"]
    target-instance = "tidb0"
    # target tables to be checked
    target-check-tables = ["test.test*"]
```

## 実行
以下のように実行します。事前に設定した`config.toml`ファイルを指定しています。
```
./sync_diff_inspector --config=./config.toml
```

その他の実行時オプションは以下を確認してください。
https://github.com/pingcap/tidb-tools/tree/master/sync_diff_inspector

## 結果確認
実行結果です。
```sh
A total of 3 tables need to be compared

Comparing the table structure of ``test`.`test02`` ... failure
Comparing the table structure of ``test`.`test03`` ... equivalent
Comparing the table structure of ``test`.`test01`` ... equivalent
Comparing the table data of ``test`.`test01`` ... equivalent
Comparing the table data of ``test`.`test03`` ... failure
_____________________________________________________________________________
Progress [============================================================>] 100% 0/0
The data of `test`.`test03` is not equal
The structure of `test`.`test02` is not equal, and data-check is skipped

The rest of tables are all equal.
The patch file has been generated in 
        'output/syncdifftest20240115/fix-on-tidb0/'
You can view the comparision details through './output/syncdifftest20240115/sync_diff.log'
```
データ構造チェックで`test02`は`failure`（不一致）となっており、後続のデータチェックはskipされています。
また、データ構造チェックをpassした`test01`と`test03`は後続のデータチェック処理が実施されており、`test03`はデータチェックで不一致があるようです。


では、出力されたファイルをそれぞれ見ていきます。
### `summary.txt`
各テーブルごとの比較結果のまとめが出力されています。
見た感じ期待どおりです。
```
Source Database

host = "172.10.60.39"
port = 3306
user = "root"

Target Databases

host = "127.0.0.1"
port = 4000
user = "root"

Comparison Result

The table structure and data in following tables are equivalent

+-----------------+---------+-----------+
|      TABLE      | UPCOUNT | DOWNCOUNT |
+-----------------+---------+-----------+
| `test`.`test01` |       5 |         5 |
+-----------------+---------+-----------+

The following tables contains inconsistent data

+-----------------+--------------------+----------------+---------+-----------+
|      TABLE      | STRUCTURE EQUALITY | DATA DIFF ROWS | UPCOUNT | DOWNCOUNT |
+-----------------+--------------------+----------------+---------+-----------+
| `test`.`test03` | true               | +2/-2          |       3 |         3 |
| `test`.`test02` | false              | +0/-0          |       0 |         0 |
+-----------------+--------------------+----------------+---------+-----------+

Time Cost: 57.703872ms
Average Speed: 0.002496MB/s
```

### `sync_diff.log`
ここでは割愛しますが、実行ログが出力されますので問題があればこちらを参考にします。

### `fix-on-tidb0`：`output`ディレクトリ配下のファイル
今回データ不一致は`test03`テーブルのため、その差分解消用のsqlが出力されています。

考え方として、ソース（今回はMySQL）を正としてターゲット（TiDB）のデータを修正するためのクエリになるため、以下のようにMySQLのデータに合わせるかたちになっています。
```sql
-- table: test.test03
-- range in sequence: Full
/*
  DIFF COLUMNS ╏ `NAME`  
╍╍╍╍╍╍╍╍╍╍╍╍╍╍╍╋╍╍╍╍╍╍╍╍╍
  source data  ╏ 'b1'    
╍╍╍╍╍╍╍╍╍╍╍╍╍╍╍╋╍╍╍╍╍╍╍╍╍
  target data  ╏ 'b2'    
╍╍╍╍╍╍╍╍╍╍╍╍╍╍╍╋╍╍╍╍╍╍╍╍╍
*/
REPLACE INTO `test`.`test03`(`id`,`name`) VALUES (2,'b1');
REPLACE INTO `test`.`test03`(`id`,`name`) VALUES (3,'c');
DELETE FROM `test`.`test03` WHERE `id` = 4 AND `name` = 'd' LIMIT 1;
```

# 最後に
使い方のイメージはついたでしょうか？

また、TiDBのユーザーコミュニティ「TiUG」でも活用事例が紹介されています。

https://speakerdeck.com/taowata/onpuremysqlwotidb-cloudheyi-xing-sitashou-shun-shao-jie-at-tiug-number-0

