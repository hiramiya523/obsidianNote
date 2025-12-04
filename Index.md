---
title: ナレッジベース ホーム
created: 2025-11-28
tags:
  - index
  - moc
aliases:
  - ホーム
  - 目次
---

# 🏠 ナレッジベース

技術メモ・ナレッジベースのホームページです。

---

## 📚 カテゴリ別ドキュメント

### 🐳 Docker

| ドキュメント | 概要 |
|-------------|------|
| [[Docker/FluentBit導入検討|Fluent Bit 導入検討]] | 導入すべきか？メリット・デメリット比較 |
| [[Docker/FluentBit推奨構成|Fluent Bit 推奨構成]] | 具体的な実装計画と設定ファイル |
| [[Docker/FluentBitFAQ|Fluent Bit FAQ]] | Grafana構成・Loki・logrotate連携 |

### 🐧 Linux

| ドキュメント | 概要 |
|-------------|------|
| [[Linux/コマンドチートシート|コマンドチートシート]] | よく使うLinuxコマンド集 |
| [[Linux/Postfix設定|Postfix設定]] | メールサーバーの設定 |
| [[Linux/サーバー設定|サーバー設定]] | SSH, sudo, sysctl等 |
| [[Linux/rc.local設定|rc.local]] | 起動スクリプト設定 |
| [[Linux/SetUID設定|SetUID]] | 特殊権限について |

### 🪟 Windows

| ドキュメント | 概要 |
|-------------|------|
| [[Windows/hyper-v-nat-setup|Hyper-V NAT構築]] | VM用の内部NATネットワーク設定 |

### 🗄️ Database

| ドキュメント | 概要 |
|-------------|------|
| [[Database/SQLチートシート|SQLチートシート]] | SQL構文・パフォーマンス最適化 |

### 🚀 Laravel

| ドキュメント | 概要 |
|-------------|------|
| [[Laravel/Laravel11ベストプラクティス|Laravel 11 ベストプラクティス]] | 2025年12月現在のLaravel 11のベストプラクティス |
| [[Laravel/Laravel_コンストラクタインジェクションによるDIメリット_まとめ|コンストラクタインジェクションによるDIのメリット]] | DIの本質とメリット、実装パターン |
| [[Laravel/推奨アプローチ|推奨アプローチ]] | DIコンテナ登録とコンストラクタインジェクションの推奨実装順序 |
| [[Laravel/サービスプロバイダ登録順序|ServiceProvider登録順序]] | サービスプロバイダの登録順序の重要性と循環参照の解決方法 |
| [[Laravel/サービス依存関係リファクタリング計画|サービス依存関係リファクタリング計画]] | 循環参照問題の解決計画と実装手順 |
| [[Laravel/サービスディレクトリ構造|サービスディレクトリ構造]] | ドメイン分離戦略とディレクトリ構造の提案 |
| [[Laravel/Coordinatorパターンの実装とコントローラーへの影響|Coordinatorパターンの実装とコントローラーへの影響]] | Coordinatorパターンの実装パターンとコントローラーへの影響 |
| [[Laravel/CoordinatorとMediatorパターンの違い|CoordinatorとMediatorパターンの違い]] | CoordinatorとMediatorパターンの詳細な比較 |
| [[Laravel/一方向依存関係のルール強制方法|一方向依存関係のルール強制方法]] | Deptracによる依存関係の検証とルール強制 |

### 🎨 Frontend

| ドキュメント | 概要 |
|-------------|------|
| [[Frontend/SvelteKitメモ|SvelteKitメモ]] | SSR、$state、$derived、$bindable等 |
| [[Frontend/svelte/Q&A|SvelteKit Q&A]] | Sanctum認証・ユニバーサルfetchの実装 |
| [[Frontend/sveltekit/検索ボックス設計|検索ボックス設計]] | SvelteKitでの検索機能の設計 |
| [[Frontend/sveltekit/検索ボックス実装|検索ボックス実装]] | SvelteKitでの検索機能の実装 |

### 📋 Projects

| ドキュメント | 概要 |
|-------------|------|
| [[Projects/Sveltekit導入計画|Sveltekit導入計画]] | SvelteKitへのフロントエンド移行計画 |
| [[Projects/AI開発移行計画|AI開発移行計画]] | SDD/IDDによる自律型進化システムへの移行計画 |
| [[Projects/テストカバレッジ改善計画|テストカバレッジ改善計画]] | テスト戦略の改善計画 |
| [[Projects/テストカバレッジ指標の説明|テストカバレッジ指標の説明]] | テストカバレッジ指標の詳細説明 |
| [[Projects/予約ドメインの循環参照解決とサブドメイン分割戦略|予約ドメインの循環参照解決とサブドメイン分割戦略]] | 予約ドメインの循環参照問題の解決戦略とサブドメイン分割 |

---

## 🔍 トピック別クイックリンク

### ログ管理
- [[Docker/FluentBit導入検討#代替ツールの比較（2025年現在）|ログ収集ツール比較]]
- [[Docker/FluentBitFAQ#2-lokiとは何か|Lokiとは？]]
- [[Docker/FluentBitFAQ#3-logrotate連携の具体的な想定|logrotate連携]]

### サーバー管理
- [[Linux/コマンドチートシート#ssh鍵|SSH鍵の生成]]
- [[Linux/サーバー設定#ssh設定|SSH設定]]
- [[Linux/コマンドチートシート#ファイアウォール|Firewall設定]]

### 仮想化
- [[Windows/hyper-v-nat-setup|Hyper-V NAT構築]]

### フロントエンド
- [[Frontend/SvelteKitメモ#svelte-5-リアクティビティ|Svelte 5 リアクティビティ]]
- [[Frontend/SvelteKitメモ#bindable---親子間のバインド|$bindable の使い方]]
- [[Frontend/svelte/Q&A|ユニバーサルfetch実装]]
- [[Projects/Sveltekit導入計画|SvelteKit導入計画]] - フロントエンド移行の包括的な計画

### AI開発・仕様駆動開発
- [[Projects/AI開発移行計画|AI開発移行計画]] - SDD/IDDによる自律型進化システムへの移行
- [[Projects/Sveltekit導入計画#5年10年先を見据えたaiとの共同開発への対応|TypeScriptとAI共同開発]] - TypeScriptがAI開発に不可欠な理由

### パフォーマンス
- [[Database/SQLチートシート#パフォーマンス最適化|SQLパフォーマンス]]
- [[Database/SQLチートシート#サブクエリの利用局面|サブクエリの注意点]]

---

## 🏷️ タグ一覧

### インフラ
#docker #linux #windows #hyper-v #server #logging #monitoring

### 開発
#sql #database #mysql #laravel #php #typescript #sveltekit #svelte #frontend

### セキュリティ
#security #ssh #permissions

---

## 📁 ディレクトリ構造

```
📁 Vault
├── 📄 Index.md              # このファイル（MOC）
├── 📄 Readme.md             # Vaultについて
├── 📁 Docker/               # Docker関連
│   ├── FluentBit導入検討.md
│   ├── FluentBit推奨構成.md
│   └── FluentBitFAQ.md
├── 📁 Linux/                # Linux関連
│   ├── コマンドチートシート.md
│   ├── Postfix設定.md
│   ├── サーバー設定.md
│   ├── rc.local設定.md
│   └── SetUID設定.md
├── 📁 Windows/              # Windows関連
│   └── hyper-v-nat-setup.md
├── 📁 Database/             # データベース関連
│   └── SQLチートシート.md
├── 📁 Laravel/              # Laravel関連
│   ├── Laravel11ベストプラクティス.md
│   ├── 推奨アプローチ.md
│   ├── サービスプロバイダ登録順序.md
│   ├── サービス依存関係リファクタリング計画.md
│   └── サービスディレクトリ構造.md
├── 📁 Frontend/             # フロントエンド関連
│   ├── SvelteKitメモ.md
│   ├── sveltekit/
│   │   ├── 検索ボックス設計.md
│   │   └── 検索ボックス実装.md
│   └── svelte/
│       └── Q&A.md
└── 📁 Projects/             # プロジェクト関連
    ├── Sveltekit導入計画.md
    ├── AI開発移行計画.md
    ├── テストカバレッジ改善計画.md
    ├── テストカバレッジ指標の説明.md
    └── 予約ドメインの循環参照解決とサブドメイン分割戦略.md
```

---

---

*最終更新: 2025年12月3日*

## 📊 Graphviewでのカテゴリ表示について

各ドキュメントには`category`フィールドが追加されています。Graphviewでカテゴリ別に表示するには：

1. **Graphviewを開く** (`Ctrl+G` または `Cmd+G`)
2. **フィルターパネルを開く** (右上のフィルターアイコン)
3. **タグでフィルタリング** - カテゴリタグ（例: `#docker`, `#laravel`, `#frontend`）で絞り込み
4. **フロントマターの`category`フィールド** - 各ノードにカテゴリ情報が含まれています

### カテゴリ一覧

- `Database` - データベース関連
- `Docker` - Docker・コンテナ関連
- `Frontend` - フロントエンド関連
- `Laravel` - Laravel・PHP関連
- `Linux` - Linux・サーバー関連
- `Projects` - プロジェクト関連
- `Windows` - Windows関連
