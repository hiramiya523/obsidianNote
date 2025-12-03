---
title: Postfix メールサーバー設定
created: 2025-11-28
tags:
  - linux
  - postfix
  - mail
  - server
aliases:
  - メールサーバー
---

# Postfix メールサーバー設定

> [!abstract] 概要
> Postfixのインストールと設定に関するメモ。

---

## インストール

### RHEL/CentOS系

```bash
# 確認
yum list installed | grep postfix

# インストール
yum -y install postfix
```

### Ubuntu/Debian系

```bash
apt update
apt install mailutils -y
# → `no configurations` を選択
```

---

## 基本コマンド

```bash
# バージョン確認
postconf | grep mail_version

# サービス管理
systemctl status postfix
systemctl start postfix
systemctl stop postfix
systemctl restart postfix
systemctl enable postfix

# 設定再読み込み
postfix reload

# 文法チェック
postfix check
```

> [!note] 警告について
> 以下の警告は無視してOK:
> ```
> warning: symlink leaves directory: /etc/postfix/./makedefs.out
> ```

---

## 設定ファイル

### 場所

```bash
vi /etc/postfix/main.cf
```

### Ubuntu初期設定

```bash
# テンプレートからコピー
cp -a /etc/postfix/main.cf.proto /etc/postfix/main.cf

# パラメータ設定
vi /etc/postfix/main.cf
```

---

## 主要パラメータ

### 必須パラメータ

| パラメータ | 説明 | 設定例 |
|-----------|------|--------|
| `inet_interfaces` | Listenするネットワークアドレス | `localhost`（送信専用） |
| `mydestination` | ローカル配送するドメイン | `mydestination = ` |
| `setgid_group` | set-gidグループ | `postdrop` |
| `mailq_path` | mailqコマンドのパス | `/usr/bin/mailq` |
| `newaliases_path` | newaliasesコマンドのパス | `/usr/bin/newaliases` |
| `sendmail_path` | sendmailコマンドのパス | `/usr/sbin/sendmail` |

### オプションパラメータ

| パラメータ | 説明 |
|-----------|------|
| `myhostname` | メールサーバ自身のホスト名（FQDN） |
| `mydomain` | メールサーバのドメイン名 |
| `append_at_myorigin` | ドメイン情報のないアドレスに `@$myorigin` を付加（デフォルト: yes） |
| `append_dot_mydomain` | `.domain`情報のないアドレスに `.$mydomain` を付加（デフォルト: yes） |

### その他設定

```ini
html_directory = no
manpage_directory = /usr/share/man
sample_directory = /etc/postfix
readme_directory = /usr/share/doc/postfix
```

---

## inet_interfaces について

> [!info] 設定の意味
> - `localhost`: サーバ内部でのメールのみ（**送信専用サーバ向け**）
> - `all`: 外部のSMTPサーバとやり取り可能

送信のみのサーバなら `localhost` を指定。

---

## sendmail コマンド

### -f オプション

```bash
# return-pathの書き換えが可能
sendmail -f sender@example.com recipient@example.com
```

> [!warning] 注意
> 強力な設定だが、Apacheで書き換えを禁止にする設定が存在する。

---

## 参考リンク

- [main.cf の設定について](https://monologu.com/maincf-settings-of-postfix/)

---

## 関連ドキュメント

### Linux関連
- [[commands-cheatsheet|Linuxコマンド]] - 基本コマンド
- [[server-settings|サーバー設定]] - SSH, sudo, sysctl等

### その他
- [[../Index|ホーム]] - 目次に戻る

---

*作成日: 2025年11月*


