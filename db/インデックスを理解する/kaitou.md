## 課題 1

### インデックスの仕組みをわかりやすく説明する

インデックスはテーブル内のレコードを効率よく取得するための索引のこと。
本に例えると
インデックスなしの検索 => 特定の項目を、本の 1 ページ目から順に読んで該当するページを探す。
インデックスありの検索 => 特定の項目が本のどこに載っているか索引を調べることで、項目が記載されているページ番号がわかる

### インデックスを手当たり次第にはってはいけない理由

- INSERT, UPDATE, DELETE などの DML クエリの実行が遅くなるため。インデックスを作成した場合、常にテーブル情報と同期する必要がある。そのため、レコードが増減するたびにインデックスのメンテナンスが行われる。インデックスが増えれば増えるほどメンテナンスコストが高くなり処理が遅くなってしまう。
- 検索時に想定とは違うインデックスをが利用される確率が上がる

### カーディナリティとは

テーブルの同一のカラムに含まれる値のバリエーションのこと。
とりうる値の種類が少ない(性別など)　=> カーディナリティが低い
とりうる値の種類が多い(ID など) => カーディナリティが高い

インデックスは値をグループ分けのうえ紐付けして検索効率を高める仕組みなので、カーディナリティが低い場合はインデックスを貼っても効果が薄い。

カーディナリティについてまとめてみた
参考：https://qiita.com/soyanchu/items/034be19a2e3cb87b2efb

### カバリングインデックスとは

インデックスがクエリに必要な要素をすべてカバーしており、テーブルアクセスしなくても良かった場合のこと。
テーブルアクセスが発生しないので高速である。

### autoincrementid カラムを PK として採用するメリデメ

####　メリット
処理効率が良い
InnoDB の場合、主キーによるインデックスの B ツリーインデックスを採用している。
UUID 等と比べた場合、ヒット率の関係上 autoincremental な id のほうが早い

#### デメリット

- id から情報が読み取られる(データ数、スクレイピング)
- インサートするまで id がわからない
- サービス合併した時に id の振り直しが発生する

参考：id を autoincrement して何が悪いの？
https://zenn.dev/dowanna6/articles/3c84e3818891c3
MySQL でプライマリキーを UUID にする前に知っておいて欲しいこと
https://techblog.raccoon.ne.jp/archives/1627262796.html

## 課題 2

実行クエリ

```sql
SELECT * FROM employees WHERE last_name like "b%";
```

実行結果

```sql
28794 rows in set (0.25 sec)
```

高速化するインデックス

```sql
CREATE INDEX last_name_index ON employees(last_name);
```

インデックス作成後実行結果

```sql
28794 rows in set (0.10 sec)
```

インデックスなし実行結果(5 回)

```sql
28794 rows in set (0.24 sec)
28794 rows in set (0.10 sec)
28794 rows in set (0.11 sec)
28794 rows in set (0.10 sec)
28794 rows in set (0.11 sec)
```

### クエリを複数回実行すると高速化する利用

クエリキャッシュ機能を利用しているため。
クエリキャッシュとは

> データベースを参照(SELECT)した結果をメモリにキャッシュし、次に同一の参照が行われた場合に前回メモリにキャッシュした結果を返却する機能です。 1 回目の参照ではデータをディスクから読み込みますのでディスク I/O が発生しますが、2 回目以降はディスクにアクセスせずにメモリ上にキャッシュされた結果を返却するので、2 回目以降の参照ではマシンに負荷がかからず、かつ高速に応答できます。

データベースに更新があった場合はキャッシュは削除される

```sql
+------------------+-------+
| Variable_name    | Value |
+------------------+-------+
| have_query_cache | YES   |
+------------------+-------+
1 row in set (0.17 sec)
```

参考：MySQL クエリキャッシュの概要と導入・評価方法
https://weblabo.oscasierra.net/mysql-query-cache/

## 課題 3

10 秒以上かかるクエリを考える

```sql
select * from salaries
join employees on employees.emp_no = salaries.emp_no
where salaries.salary > 70000
order by salaries.salary DESC
;
```

実行結果
_問題とは違い、むしろインデックスでクエリが遅くなりますが思った以上に結果が違ったので残します_
参考: 遅いインデックス パート II
https://use-the-index-luke.com/ja/sql/where-clause/the-equals-operator/slow-indexes-part-ii

インデックスなし

```sql
907693 rows in set (4.65 sec)
-- ちなみに salaries.salary > 0 だと
2844047 rows in set (10.03 sec)
```

インデックスあり

```sql
create index salary_index on salaries(salary);
```

```sql
907693 rows in set (27.54 sec)
-- ちなみに salaries.salary > 0 だと
2844047 rows in set (1 min 27.07 sec)
```

実行計画

```sql
mysql> explain select * from salaries join employees on employees.emp_no = salaries.emp_no where salaries.salary > 70000 order by salaries.salary DESC;
+----+-------------+-----------+------------+--------+----------------------+--------------+---------+---------------------------+------+----------+-----------------------+
| id | select_type | table     | partitions | type   | possible_keys        | key          | key_len | ref                       | rows | filtered | Extra                 |
+----+-------------+-----------+------------+--------+----------------------+--------------+---------+---------------------------+------+----------+-----------------------+
|  1 | SIMPLE      | salaries  | NULL       | range  | PRIMARY,salary_index | salary_index | 4       | NULL                      |    1 |   100.00 | Using index condition |
|  1 | SIMPLE      | employees | NULL       | eq_ref | PRIMARY              | PRIMARY      | 4       | employees.salaries.emp_no |    1 |   100.00 | NULL                  |
+----+-------------+-----------+------------+--------+----------------------+--------------+---------+---------------------------+------+----------+-----------------------+
2 rows in set, 1 warning (0.00 sec)
```

## 課題 4

### 複合インデックスのしくみ

複合インデックスとは、複数のカラムを組み合わせたインデックスのこと
たとえば苗字と年齢の組み合わせで複合インデックスを作成した場合以下のようになる

インデックスイメージ
田中 10
田中 20
田中 30
佐藤 11
佐藤 21

先頭のカラムから順にソートした状態でインデックスを作成してくれるので、検索を高速化することができる。

### 順序が違う複合インデックスが NG な理由

先頭のカラムから順にソートされた状態になっているため、検索条件の組み合わせによっては複合インデックスが利用されないため
A、B、C の順で複合インデックスを作成した場合、利用可否は以下のような組み合わせになる
A => OK
B => NG
C => NG
A,B => OK
A,C => NG
B,C => NG

よって姓だけの検索が多い場合、(last_name, first_name)の順にしなければインデックスが効かない

## メモ

便利なコマンド

```sql
-- 結果表示を無くす
pager cat >/dev/null;
-- 結果表示を戻す
nopager;
```
