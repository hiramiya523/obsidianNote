---
title: rc.local 起動スクリプト
created: 2025-11-28
tags:
  - linux
  - systemd
  - startup
aliases:
  - 起動スクリプト
---

# rc.local 起動スクリプト

> [!abstract] 概要
> systemd環境での起動時スクリプト実行について。

---

## 背景

systemdになって、起動時に実行したいプログラムは**スクリプトを書いてサービスとして登録**するのが本来の流儀。

しかし、`/etc/rc.local` を使った従来の方法も可能。

---

## 設定方法

1. `/etc/rc.local` をrootで作成・編集

2. 実行権限を付与

```bash
chmod 700 /etc/rc.local
```

これにより、`rc-local` というサービスによって自動的に実行される。

---

## リダイレクト記法

| 記法 | 説明 |
|------|------|
| `>` | 標準出力を上書き |
| `>>` | 標準出力を追記 |

---

## 関連ドキュメント

### Linux関連
- [[server-settings|サーバー設定]] - SSH, sudo, sysctl等
- [[commands-cheatsheet|Linuxコマンド]] - 基本コマンド

### その他
- [[../Index|ホーム]] - 目次に戻る

---

*作成日: 2025年11月*


