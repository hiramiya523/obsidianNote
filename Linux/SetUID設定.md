---
title: SetUID 特殊権限
created: 2025-11-28
updated: 2025-12-03
category: Linux
tags:
  - linux
  - security
  - permissions
aliases:
  - 特殊権限
  - SUID
---

# SetUID 特殊権限

> [!abstract] 概要
> 一般ユーザーでroot権限配下へのアクセスを実現するSetUIDについて。

---

## 要件

一般ユーザーで、root権限配下へのアクセスを実現したい。

### 解決策

1. C言語でプログラムを作成（例: ファイルを`/root`配下に移動する処理）
2. コンパイル
3. SetUIDを設定
4. 一般ユーザーでも`/root`配下へ移動できるようになる

---

## SetUIDとは

権限の`s`は、SetUIDが設定されていることを表す。

SetUIDを設定していると、**そのプログラムのプロセスのUIDは所有者になる**。

### 例: passwdコマンド

```bash
$ ls -l /usr/bin/passwd
-rwsr-xr-x 1 root root
```

> [!info] なぜsudoが動くのか
> `sudo`コマンドにSetUIDが設定されているため、どのユーザーでもroot権限でコマンドが実行できる。

---

## 実装方法

### SetUIDの付与

```bash
chmod u+s samplepgm
```

> [!warning] 注意点
> - 所有者はrootである必要がある
> - groupは任意

---

## よくある誤解

> [!danger] シェルスクリプトにSetUIDは効かない
> 「シェルスクリプトにsを付ければ良いのでは？」と思いがちだが、**スクリプトファイルに権限を与えても無効**。
> 
> `bash`コマンド自体にSetUIDを割り当てる必要があるが、これは**極めて危険**なので行わない。
> 
> SetUIDは**コンパイルされたバイナリ**に対してのみ有効に機能する。

---

## 関連ドキュメント

### Linux関連
- [[Linux/サーバー設定|サーバー設定]] - SSH, sudo, sysctl等
- [[Linux/コマンドチートシート|Linuxコマンド]] - 基本コマンド

### セキュリティ関連
- [[Linux/サーバー設定#ssh設定|SSH設定]] - セキュリティ設定

### その他
- [[../Index|ホーム]] - 目次に戻る

---

*作成日: 2025年11月*


