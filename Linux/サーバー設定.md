---
title: サーバー設定
created: 2025-11-28
tags:
  - linux
  - server
  - security
  - ssh
  - sudo
aliases:
  - SSH設定
  - sudo設定
---

# サーバー設定

> [!abstract] 概要
> Linuxサーバーの基本設定（sysctl, sudo, SSH等）に関するメモ。

---

## カーネルパラメータ

### /etc/sysctl.conf

`/proc/sys/` ディレクトリに存在するファイルの初期値を記述するファイル。

> [!info] 説明
> `/proc/sys/` ディレクトリには、カーネルの挙動を決める値が格納されたファイルが存在。
> - 参照: `cat` コマンド
> - 書き込み: `echo` コマンド

---

## sudo設定

### パスワードなしでsudo実行

#### 方法1: visudoで編集

```bash
sudo visudo
```

以下を追加:

```
# Allow members of group sudo to execute any command
%sudo   ALL=(ALL:ALL) ALL
shirakawa ALL=NOPASSWD: /sbin/su
```

#### 方法2: wheelグループに追加

> [!warning] 注意
> ユーザーを既存の`wheel`グループに追加する方法もあるが、**全コマンド許可**になる。

---

## SSH設定

### 設定ファイル

```bash
vi /etc/ssh/sshd_config
```

### 推奨設定

```diff
# rootログインを拒否
- #PermitRootLogin yes
+ PermitRootLogin no

# パスワード無しユーザのログインを拒否
- #PermitEmptyPasswords no
+ PermitEmptyPasswords no
```

### 構文確認

```bash
sshd -t
```

> [!tip] 構文チェック
> 設定変更後、サービス再起動前に必ず `sshd -t` で構文チェックを行うこと。

---

## 関連ドキュメント

### Linux関連
- [[commands-cheatsheet|Linuxコマンド]] - 基本コマンド
- [[setuid|SetUID]] - 特殊権限
- [[rc-local|rc.local]] - 起動スクリプト
- [[postfix|Postfix設定]] - メールサーバー

### その他
- [[../Index|ホーム]] - 目次に戻る

---

*作成日: 2025年11月*


