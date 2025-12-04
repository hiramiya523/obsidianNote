---
title: ログ収集コンテナ導入 FAQ
created: 2025-11-28
updated: 2025-12-03
category: Docker
tags:
  - docker
  - logging
  - fluent-bit
  - loki
  - grafana
  - faq
aliases:
  - FAQ
  - よくある質問
---

# ログ収集コンテナ導入 FAQ

> [!abstract] 概要
> ログ収集コンテナ導入に関するよくある質問と回答です。
> 
> 👉 基本的な検討は [[Docker/FluentBit導入検討|導入検討]] を参照
> 👉 具体的な実装は [[Docker/FluentBit推奨構成|推奨構成]] を参照

---

## 目次

1. [[#1. Grafanaは同一環境に構築するのか？]]
2. [[#2. Lokiとは何か？]]
3. [[#3. logrotate連携の具体的な想定]]

---

## 1. Grafanaは同一環境に構築するのか？

### 回答: どちらでも可能だが、分離を推奨

### パターンA: 同一環境に構築（シンプル）

```
┌──────────────────────────────────────────────────────────────┐
│ 本番サーバー                                                  │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐            │
│  │ Laravel │ │  Nginx  │ │ MariaDB │ │  Redis  │            │
│  └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘            │
│       └───────────┴───────────┴───────────┘                  │
│                       ↓                                      │
│              ┌─────────────┐                                 │
│              │  Fluent Bit │                                 │
│              └──────┬──────┘                                 │
│                     ↓                                        │
│              ┌─────────────┐      ┌─────────────┐           │
│              │    Loki     │ ───→ │   Grafana   │ :3000     │
│              └─────────────┘      └─────────────┘           │
└──────────────────────────────────────────────────────────────┘
```

> [!success] メリット
> - 構成がシンプル
> - ネットワーク設定不要

> [!failure] デメリット
> - 本番サーバーのリソースを消費
> - 本番障害時にログ確認不可

### パターンB: 監視サーバーを分離（推奨）

```
┌────────────────────────────┐    ┌────────────────────────────┐
│ 本番サーバー                │    │ 監視サーバー                │
│  ┌─────────┐ ┌─────────┐  │    │                            │
│  │ Laravel │ │  Nginx  │  │    │  ┌─────────────┐           │
│  └────┬────┘ └────┬────┘  │    │  │    Loki     │           │
│       └───────────┘       │    │  └──────┬──────┘           │
│            ↓              │    │         ↓                  │
│     ┌─────────────┐       │    │  ┌─────────────┐           │
│     │  Fluent Bit │ ──────────→│  │   Grafana   │ :3000     │
│     └─────────────┘   TCP │    │  └─────────────┘           │
│                           │    │                            │
│  + ファイル出力（バックアップ）│    │  + Zabbix も同居可能      │
└────────────────────────────┘    └────────────────────────────┘
```

> [!success] メリット
> - 本番障害時もログ確認可能
> - リソース分離

> [!failure] デメリット
> - サーバー追加コスト
> - ネットワーク設定必要

### 推奨判断マトリクス

| 条件 | 推奨パターン |
|------|-------------|
| 監視サーバー（Zabbix用）が既にある | **パターンB**: そこにLoki+Grafana追加 |
| サーバー追加が難しい | **パターンA**: 同一環境に構築 |
| 高可用性が必要 | **パターンB**: 分離必須 |

---

## 2. Lokiとは何か？

### 一言で言うと

> [!info] 定義
> **「ログ版Prometheus」** = ログを効率的に保存・検索するためのデータベース

### 従来のログスタック vs Loki

#### 従来: ELK/EFK スタック

```
Fluent Bit → Elasticsearch → Kibana
```

> [!warning] 問題点
> - Elasticsearchはメモリを大量消費（最低4GB推奨）
> - 全文検索インデックスでディスク容量も大きい
> - 運用が複雑

#### 軽量: PLG スタック (Promtail/Loki/Grafana)

```
Fluent Bit → Loki → Grafana
```

> [!success] 特徴
> - 軽量（256MB〜512MB RAM で動作）
> - ラベルベースのインデックス（全文検索しない）
> - Grafanaと同じ会社が開発（相性抜群）

### Lokiの仕組み

```
ログの流れ:
                                                      
  {"container":"php", "log":"[ERROR] DB connection failed"}
                          ↓
              ┌───────────────────────┐
              │        Loki          │
              ├───────────────────────┤
              │ ラベル（インデックス）   │ → container=php, level=ERROR
              │ ログ本文（圧縮保存）    │ → "DB connection failed"
              └───────────────────────┘
                          ↓
              Grafanaでクエリ:
              {container="php"} |= "ERROR"
```

### Elasticsearch vs Loki 詳細比較

| 項目 | Elasticsearch | Loki |
|------|--------------|------|
| **メモリ使用量** | 4GB〜 | 256MB〜 |
| **検索方式** | 全文検索（高速） | ラベル+grep（やや遅い） |
| **ディスク使用量** | 大 | 小（圧縮） |
| **学習コスト** | 高 | 低（LogQL ≒ PromQL） |
| **適用ケース** | 大規模・高頻度検索 | 中小規模・コスト重視 |

### LogQL クエリ例

```logql
# コンテナ名でフィルタ
{container="php"}

# エラーログのみ
{container="php"} |= "ERROR"

# 正規表現でフィルタ
{container="php"} |~ "ERROR|CRITICAL"

# JSON解析してフィールドでフィルタ
{container="php"} | json | level="error"

# 直近1時間のエラー数をカウント
count_over_time({container="php"} |= "ERROR" [1h])
```

> [!tip] 結論
> **オンプレ環境でリソースが限られるなら Loki が最適**

---

## 3. logrotate連携の具体的な想定

### 想定シナリオ

> [!bug] 問題
> ```
> Fluent Bit → /var/log/app/laravel/app.log
> 
> ファイルが無限に肥大化する
> → ディスク容量を圧迫 → サービス停止のリスク
> ```

### 解決策: logrotate でローテーション

#### 設定ファイル: `/etc/logrotate.d/app-logs`

```bash
/var/log/app/**/*.log {
    daily              # 毎日ローテーション
    rotate 30          # 30世代保持（= 30日分）
    compress           # gzip圧縮
    delaycompress      # 1世代前から圧縮開始
    missingok          # ファイルがなくてもエラーにしない
    notifempty         # 空ファイルはローテーションしない
    create 0640 root root
    sharedscripts
    postrotate
        # オプション: Fluent Bitに通知
        docker kill --signal=USR1 $(docker ps -qf name=fluent-bit) 2>/dev/null || true
    endscript
}
```

### 動作イメージ

```
Day 1:
  /var/log/app/laravel/app.log      (今日のログ)

Day 2:
  /var/log/app/laravel/app.log      (今日のログ - 新規)
  /var/log/app/laravel/app.log.1    (昨日のログ)

Day 3:
  /var/log/app/laravel/app.log      (今日のログ)
  /var/log/app/laravel/app.log.1    (昨日のログ)
  /var/log/app/laravel/app.log.2.gz (一昨日のログ - 圧縮)

Day 31:
  /var/log/app/laravel/app.log
  /var/log/app/laravel/app.log.1
  /var/log/app/laravel/app.log.2.gz
  ...
  /var/log/app/laravel/app.log.30.gz
  (app.log.31.gz は自動削除される)
```

### Fluent Bit側の設定

```ini
# fluent-bit.conf

[OUTPUT]
    Name          file
    Match         docker.php*
    Path          /var/log/app/laravel/
    File          app.log
    Format        json_lines
    # ファイルローテーションに対応するオプション
    Mkdir         On           # ディレクトリ自動作成
```

### logrotate 設定オプション解説

| オプション | 説明 |
|-----------|------|
| `daily` | 毎日ローテーション（weekly, monthlyも可） |
| `rotate 30` | 30世代保持 |
| `compress` | gzip圧縮（約1/10になる） |
| `delaycompress` | 直近1ファイルは圧縮しない（grep可能に） |
| `missingok` | ファイルがなくてもエラーにしない |
| `notifempty` | 空ファイルはスキップ |
| `create 0640 root root` | 新規ファイルのパーミッション |
| `copytruncate` | ファイルをコピー後に元ファイルを空にする |

### copytruncate版（より安全）

> [!tip] 推奨
> Fluent Bitはファイルを開き続けるため、`copytruncate`が安全な場合あり：

```bash
/var/log/app/**/*.log {
    daily
    rotate 30
    compress
    delaycompress
    missingok
    notifempty
    copytruncate      # ファイルをコピー後に元ファイルを空にする
                      # → Fluent Bitがファイルを開き直す必要なし
}
```

### 注意点

> [!warning] 注意
> | 項目 | 説明 |
> |------|------|
> | **copytruncate vs create** | Fluent Bitはファイルを開き続けるため、`copytruncate`が安全 |
> | **ディスク監視** | Zabbixでディスク使用率80%でアラート推奨 |
> | **圧縮率** | gzipで約1/10になる（10MBログ → 1MB） |

### ディスク容量の計算例

```
前提:
- 1日あたりのログ量: 100MB
- 保持期間: 30日
- 圧縮率: 1/10

計算:
- 非圧縮（直近2日）: 100MB × 2 = 200MB
- 圧縮（残り28日）: 100MB × 28 × 0.1 = 280MB
- 合計: 約500MB

→ 14コンテナ × 500MB = 約7GB（余裕を持って10GB確保推奨）
```

---

## まとめ

| 質問 | 回答 |
|------|------|
| [[#1. Grafanaは同一環境に構築するのか？\|Grafanaは同一環境？]] | **どちらでも可**。監視サーバーがあれば分離推奨 |
| [[#2. Lokiとは何か？\|Lokiとは？]] | **軽量ログDB**。Elasticsearchの代替。オンプレ向き |
| [[#3. logrotate連携の具体的な想定\|logrotate連携とは？]] | **ログファイルの自動ローテーション**。ディスク容量管理 |

### 実際の構成例（最小構成）

```yaml
# オンプレ本番の最小構成
services:
  fluent-bit:
    # → /var/log/app/ にファイル出力
    # → logrotate で30日ローテーション
    # → Lokiは後から追加可能（最初は不要）
```

---

## 関連ドキュメント

### Docker関連
- [[Docker/FluentBit導入検討|導入検討]] - メリット・デメリット比較
- [[Docker/FluentBit推奨構成|推奨構成]] - 具体的な実装計画

### その他
- [[../Index|ホーム]] - 目次に戻る

---

*作成日: 2025年11月*


