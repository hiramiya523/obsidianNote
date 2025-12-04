---
title: SQL チートシート・パフォーマンス
created: 2025-11-28
updated: 2025-12-03
category: Database
tags:
  - sql
  - database
  - mysql
  - mariadb
  - performance
aliases:
  - SQL
  - データベース
---

# SQL チートシート・パフォーマンス

> [!abstract] 概要
> SQL（主にMySQL/MariaDB）のチートシートとパフォーマンスに関するメモ。

---

## 基本コマンド

### テーブル情報

```sql
-- テーブル詳細
DESC table_name;

-- 制約確認
SHOW CREATE TABLE table_name \G
```

---

## つまづきポイント

### INSERT文でサブクエリが使えない

> [!bug] エラー: Error 1093
> 同じテーブルへのINSERT/UPDATE時にサブクエリで同じテーブルを参照できない。

**回避策**: テンポラリテーブルを参照するようにする

> [!warning] 古い参考リンク
> 元の参考リンク（2009年のブログ記事）は古いため、公式ドキュメントを参照することを推奨:
> - [MySQL公式: Restrictions on Subqueries](https://dev.mysql.com/doc/refman/8.0/en/subquery-restrictions.html)

---

### アンビギュアスグループ (Ambiguous Group)

> [!danger] 重要
> **GROUP BYを使用した時、グループ内で一意でない値は取得できない！**

#### ❌ 問題のあるクエリ

```sql
SELECT name, MAX(weight), animal
FROM Pets
GROUP BY animal
```

`animal`でグループ化しているのに、`name`が一意に特定できないためエラー。
（`MAX`は集約関数で一意に特定できる）

意図: `MAX(weight)`と同じ行にある`name`を取得したい
→ SQL側にはその意図を認識する手段がない

#### ✅ 改善クエリ

```sql
SELECT p1.name, m.weight, m.animal
FROM pets p1
INNER JOIN (
    SELECT MAX(p2.weight) AS weight, p2.animal
    FROM pets p2
    GROUP BY p2.animal
) AS m
ON m.animal = p1.animal AND m.weight = p1.weight
```

GROUP BYを使わず、サブクエリで結合することで想定の値を取得できる。

---

## JOIN

### INNER JOINの結合順序

複数のテーブルを結合する場合、内部的な処理（結合順序）は **SQL Serverが最速と判断した順番** で行われる。

> [!tip] 考え方
> 複数のJoinの結合法則に悩んだら、**JoinをCross Joinから考える**と理解しやすい。

### JOINのパフォーマンス: WHERE vs ON

#### パターン1: WHERE句で条件指定

```sql
SELECT a.name, b.name
FROM a
INNER JOIN b ON a.b_cd = b.cd
WHERE b.cd = '1'
```

#### パターン2: ON句で条件指定

```sql
SELECT a.name, b.name
FROM a
INNER JOIN b ON a.b_cd = b.cd AND b.cd = '1'
```

> [!info] どちらが良いか
> 一般的にはオプティマイザが最適化するため大差ないが、OUTER JOINでは結果が異なる場合がある。

---

## GROUP BY

### 基本

```sql
SELECT
    c_mail,
    MAX(dt_purchase) AS last_purchase_dt
FROM t_order_info
GROUP BY c_mail;
```

**集約関数**と**groupingカラム**は共存できる。

---

## EXISTS句

> [!warning] エイリアス必須
> FROM句のテーブルにもエイリアスを付けないと正しく取得できない。

```sql
SELECT DISTINCT c_mail
FROM t_order_info M1
WHERE NOT EXISTS (
    SELECT 1
    FROM t_order_info M2
    WHERE M1.c_mail = M2.c_mail
      AND M2.i_follow_mail = 1
)
```

---

## CASE式

### 検索CASE式

`WHEN`の直後に判定式を置き、式が真である場合に`THEN`以降の処理を実行する。

> [!tip] CASEの本質
> CASE式の使い方は、**値を別の値に置き換えることで、グルーピングしたり集約できる**ということ。

---

## パフォーマンス最適化

### ソートの回避

#### DISTINCTをEXISTSで代用

> [!info] 理由
> `DISTINCT`も内部的にソートを行っているため、`EXISTS`で代用するほうが良い場合がある。

#### WHERE句で書ける条件はHAVING句には書かない

> [!tip] 理由
> `GROUP BY`で集約する**前**に絞り込むほうが、それ以降に計算する行数を減らせる。

---

### 中間テーブルを減らす

#### HAVING句を活用する

> [!info] ポイント
> 集約した結果に対する条件は、**HAVING句で設定するのが原則**。
> 慣れていないエンジニアはWHERE句に頼りがち。

---

### サブクエリの利用局面

> [!warning] サブクエリは重い
> サブクエリが実行されるたびにテーブルが作られるイメージで、**そのテーブルにはインデックスが張られていない**。

#### 使用すべき場面

- サブクエリで集計する場合
- サブクエリにORDERとLIMITを入れる場合

#### パフォーマンス比較

| ケース | 外部結合 | サブクエリ |
|--------|----------|-----------|
| 検索条件が1つ | 0.026秒 | 0.025秒 |
| 検索条件が複数 | 0.102秒 | **9.319秒** |

> [!danger] 注意
> 検索条件が複数の場合、サブクエリは大幅に遅くなる可能性がある。
> サブクエリで複数条件をORで結合すると最適化できない。

#### WITH句の推奨

```sql
WITH cte AS (
    SELECT ...
    FROM ...
)
SELECT * FROM cte WHERE ...
```

> [!tip] 可読性優先
> どの道重いなら、**WITH句で可読性を上げる**ほうを優先すべき。

---

## インデックスに関する考え方

### 否定条件でインデックスが効かない理由

> [!info] 考え方
> 「○○以外」という条件は**膨大すぎる**ため、インデックスでは効率的に絞り込めない。

### 暗黙変換でインデックスが効かない理由

> [!info] 考え方
> 変換されているということは、**元のインデックスの型とは違う**ため、インデックスが使えない。

---

## 関連ドキュメント

### データベース関連
- [[Docker/FluentBit導入検討|Fluent Bit導入検討]] - ログ管理（DBログも含む）

### その他
- [[../Index|ホーム]] - 目次に戻る

---

*作成日: 2025年11月*


