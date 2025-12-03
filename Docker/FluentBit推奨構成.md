---
title: ログ収集コンテナ導入推奨事項
created: 2025-11-28
tags:
  - docker
  - logging
  - fluent-bit
  - implementation
  - zabbix
aliases:
  - 推奨構成
  - 実装計画
---

# ログ収集コンテナ導入推奨事項

> [!abstract] 概要
> 本番環境へのFluent Bit導入に関する具体的な実装計画と設定ファイルです。
> 
> 👉 導入すべきかの判断は [[fluent-bit-introduction|導入検討]] を参照
> 👉 よくある質問は [[fluent-bit-faq|FAQ]] を参照

---

## 環境情報

| 項目 | 回答 |
|------|------|
| **デプロイ先** | 開発環境 + オンプレ本番（環境別切り替え可） |
| **運用フェーズ** | 本番運用中 |
| **既存の監視ツール** | Zabbix（使用許可は未確定） |
| **ログ保存要件** | 未定 |

---

## 結論: 導入を推奨

> [!success] 推奨理由
> 1. **本番運用中** → 障害調査時のログ横断検索が必要になる場面が必ず発生する
> 2. **オンプレ環境** → クラウドのログサービス（CloudWatch等）が使えないため、自前でのログ管理が必須
> 3. **Zabbixとの連携** → Fluent Bitはログベースのアラート発火が可能で、Zabbixと役割分担できる

---

## 推奨構成

### 環境別構成

```
┌─────────────────────────────────────────────────────────────┐
│ 開発環境                                                     │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ compose.yaml (開発用override)                           │ │
│ │ - logging: json-file（現状維持でOK）                    │ │
│ │ - docker compose logs -f で確認                        │ │
│ └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ オンプレ本番環境                                             │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ compose.yaml (本番用override)                           │ │
│ │ - Fluent Bit → ファイル出力（/var/log/app/）           │ │
│ │ - ログローテーション: logrotate連携                     │ │
│ │ - オプション: Loki + Grafana（可視化）                  │ │
│ └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

---

## 実装計画

### Phase 1: 本番環境にFluent Bit導入（ファイル出力のみ）

**目的**: 最小構成で導入し、ログを永続化

#### ファイル構成

```
/workspace/
├── compose.yaml                    # 共通設定
├── compose.override.yaml           # 開発用（json-file維持）
├── compose.production.yaml         # 本番用（Fluent Bit使用）
└── docker/
    └── fluent-bit/
        ├── fluent-bit.conf
        └── parsers.conf
```

#### compose.production.yaml

```yaml
# 本番環境用オーバーライド
x-common-configs: &x-common-configs
  logging:
    driver: fluentd
    options:
      fluentd-address: "fluent-bit:24224"
      fluentd-async: "true"
      fluentd-retry-wait: "1s"
      fluentd-max-retries: "3"
      tag: "docker.{{.Name}}"

services:
  fluent-bit:
    image: fluent/fluent-bit:3.2
    hostname: fluent-bit
    restart: always
    volumes:
      - ./docker/fluent-bit/fluent-bit.conf:/fluent-bit/etc/fluent-bit.conf:ro
      - ./docker/fluent-bit/parsers.conf:/fluent-bit/etc/parsers.conf:ro
      - /var/log/app:/var/log/app  # ホストのログディレクトリ
    networks:
      - base-net
    healthcheck:
      test: ["CMD-SHELL", "curl -sf http://localhost:2020/api/v1/health || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 5s
```

#### docker/fluent-bit/fluent-bit.conf（本番用・シンプル版）

```ini
[SERVICE]
    Flush         5
    Daemon        Off
    Log_Level     warn
    Parsers_File  parsers.conf
    HTTP_Server   On
    HTTP_Listen   0.0.0.0
    HTTP_Port     2020

# ログ受信
[INPUT]
    Name          forward
    Listen        0.0.0.0
    Port          24224
    Buffer_Chunk_Size  1M
    Buffer_Max_Size    6M

# タイムスタンプ追加
[FILTER]
    Name          record_modifier
    Match         *
    Record        hostname ${HOSTNAME}
    Record        collected_at ${TIMESTAMP}

# Laravelログのパース
[FILTER]
    Name          parser
    Match         docker.php*
    Key_Name      log
    Parser        laravel
    Reserve_Data  On

# コンテナ別にファイル出力
[OUTPUT]
    Name          file
    Match         docker.php*
    Path          /var/log/app/laravel/
    File          app.log
    Format        json_lines

[OUTPUT]
    Name          file
    Match         docker.web*
    Path          /var/log/app/nginx/
    File          access.log
    Format        json_lines

[OUTPUT]
    Name          file
    Match         docker.mariadb*
    Path          /var/log/app/mariadb/
    File          mariadb.log
    Format        json_lines

[OUTPUT]
    Name          file
    Match         docker.redis*
    Path          /var/log/app/redis/
    File          redis.log
    Format        json_lines

# その他のログ
[OUTPUT]
    Name          file
    Match         *
    Path          /var/log/app/other/
    File          other.log
    Format        json_lines
```

#### docker/fluent-bit/parsers.conf

```ini
[PARSER]
    Name          laravel
    Format        regex
    Regex         ^\[(?<time>[^\]]+)\] (?<env>\w+)\.(?<level>\w+): (?<message>.*)$
    Time_Key      time
    Time_Format   %Y-%m-%d %H:%M:%S

[PARSER]
    Name          nginx_access
    Format        regex
    Regex         ^(?<remote>[^ ]*) - (?<user>[^ ]*) \[(?<time>[^\]]*)\] "(?<method>\S+) (?<path>\S+) (?<protocol>\S+)" (?<status>[^ ]*) (?<size>[^ ]*)
    Time_Key      time
    Time_Format   %d/%b/%Y:%H:%M:%S %z
```

#### 起動コマンド

```bash
# 開発環境（現状通り）
docker compose up -d

# 本番環境
docker compose -f compose.yaml -f compose.production.yaml up -d
```

---

### Phase 2: ログローテーション設定

**目的**: ディスク容量を管理

> [!note] 詳細
> [[fluent-bit-faq#3-logrotate連携の具体的な想定|FAQ - logrotate連携]] も参照

#### /etc/logrotate.d/app-logs

```
/var/log/app/**/*.log {
    daily
    rotate 30
    compress
    delaycompress
    missingok
    notifempty
    create 0640 root root
    sharedscripts
    postrotate
        # Fluent Bitにログローテーションを通知（オプション）
        docker kill --signal=USR1 $(docker ps -qf name=fluent-bit) 2>/dev/null || true
    endscript
}
```

**ログ保存期間の推奨**:

| ログ種別 | 推奨期間 | 理由 |
|---------|---------|------|
| アプリケーションログ | 30日 | 障害調査に必要 |
| アクセスログ | 90日 | トラフィック分析 |
| エラーログ | 180日 | 再発防止分析 |
| セキュリティログ | 1年 | 監査要件 |

---

### Phase 3: 可視化（オプション）

**目的**: ログ検索・分析の効率化

> [!note] Lokiについて
> [[fluent-bit-faq#2-lokiとは何か|FAQ - Lokiとは？]] も参照

#### 構成追加（Loki + Grafana）

```yaml
# compose.production.yaml に追加
services:
  loki:
    image: grafana/loki:3.2.0
    hostname: loki
    command: -config.file=/etc/loki/local-config.yaml
    volumes:
      - ./docker/loki/config.yaml:/etc/loki/local-config.yaml:ro
      - loki-data:/loki
    networks:
      - base-net
    restart: always

  grafana:
    image: grafana/grafana:11.3.0
    hostname: grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - grafana-data:/var/lib/grafana
      - ./docker/grafana/provisioning:/etc/grafana/provisioning:ro
    networks:
      - base-net
    ports:
      - "3000:3000"
    depends_on:
      - loki
    restart: always

volumes:
  loki-data:
  grafana-data:
```

#### Fluent Bit に Loki 出力を追加

```ini
# fluent-bit.conf に追加
[OUTPUT]
    Name          loki
    Match         *
    Host          loki
    Port          3100
    Labels        job=docker, container=$TAG, env=production
    Auto_Kubernetes_Labels  Off
```

---

## Zabbixとの役割分担

| 機能 | Zabbix | Fluent Bit + Loki |
|------|--------|-------------------|
| **サーバーリソース監視** | ✅ 推奨 | - |
| **サービス死活監視** | ✅ 推奨 | - |
| **ログ収集・保存** | - | ✅ 推奨 |
| **ログ検索・分析** | - | ✅ 推奨 |
| **エラーログアラート** | 連携可能 | ✅ 推奨 |

### Zabbixへのアラート連携（オプション）

```ini
# fluent-bit.conf でエラーログをZabbixに送信
[FILTER]
    Name          grep
    Match         docker.php*
    Regex         level (ERROR|CRITICAL|ALERT|EMERGENCY)

[OUTPUT]
    Name          http
    Match         docker.php*
    Host          zabbix-server
    Port          10051
    URI           /api_jsonrpc.php
    Format        json
```

---

## 導入スケジュール案

| Week | タスク | 成果物 |
|------|--------|--------|
| 1 | Phase 1 環境構築 | compose.production.yaml, fluent-bit設定 |
| 1 | ステージング検証 | 動作確認レポート |
| 2 | Phase 2 logrotate設定 | logrotate設定ファイル |
| 2 | 本番デプロイ | 本番稼働 |
| 3-4 | Phase 3 Loki+Grafana（任意） | ダッシュボード |

---

## 注意事項

> [!warning] 重要な注意点

### 1. 起動順序

```yaml
# Fluent Bitを最初に起動
services:
  fluent-bit:
    # ...
  
  php:
    depends_on:
      fluent-bit:
        condition: service_healthy
```

### 2. fluentd-async オプション

```yaml
logging:
  options:
    fluentd-async: "true"  # Fluent Bit起動前でもコンテナ起動可能
```

これを設定しないと、Fluent Bit起動前に他のコンテナが起動失敗する。

### 3. ディスク容量監視

- Fluent Bitのバッファ溢れを防ぐため、ディスク使用率をZabbixで監視推奨
- 閾値: 80%でアラート

### 4. セキュリティ

- Fluent Bitのポート24224は内部ネットワークのみに公開
- 外部からアクセスできないようにファイアウォール設定

---

## 次のアクション

- [ ] ログ保存期間をチームで決定
- [ ] Zabbixの使用許可を確認
- [ ] compose.production.yaml の作成
- [ ] Fluent Bit設定ファイルの作成
- [ ] ステージング環境での検証
- [ ] 本番デプロイ

---

## 関連ドキュメント

### Docker関連
- [[fluent-bit-introduction|導入検討]] - メリット・デメリット比較
- [[fluent-bit-faq|FAQ]] - よくある質問

### 監視・ログ関連
- [[../Linux/commands-cheatsheet|Linuxコマンド]] - ログ確認コマンド

### その他
- [[../Index|ホーム]] - 目次に戻る

---

*作成日: 2025年11月*


