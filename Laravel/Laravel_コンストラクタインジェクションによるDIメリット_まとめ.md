---
title: Laravel コンストラクタインジェクションによるDIのメリット
created: 2025-12-03
updated: 2025-12-03
category: Laravel
tags:
  - laravel
  - php
  - dependency-injection
  - di
  - コンストラクタインジェクション
aliases:
  - DIメリット
  - コンストラクタインジェクション
---

# Laravel コンストラクタインジェクションによるDIのメリット - まとめ

## 📌 結論

**コンストラクタインジェクションによるDI**は、以下の3つを実現するための基礎技術です：

1. **テストしやすさ** - モック注入によるユニットテストの容易化
2. **実装差し替えやすさ** - 環境別・将来のリファクタリングに対応
3. **Laravel依存の薄いドメインコード** - フレームワーク非依存のビジネスロジック構築

---

## 🎯 DIの本質とよくある誤解

### ❌ 誤解
- 「将来DBをMySQLからPostgreSQLに変えるかもしれないからDIする」
- 実務でDBエンジンだけを差し替えるケースは稀で、この理由だけではコストに見合わない

### ✅ 正解
**「外部API通信や現在時刻、ランダム値など、不確定要素を排除してユニットテストを書くためにDIする」**

コンストラクタインジェクションにより、クラスが依存するオブジェクトを「外から渡せる」状態にし、テスト時にモック（偽物）へ置き換え可能にします。

---

## 🔑 主なメリット

### 1. テスト容易性（Testability）
- **外部API通信をモック化**：テスト実行時に実際のAPIを叩かない
- **Laravelを起動せずにテスト可能**：純粋なPHPUnitのTestCaseでテストできる
- **依存関係の制御**：テスト時に任意の実装を注入できる

### 2. 依存関係の明示（Explicitness）
- **コンストラクタの引数で依存が可視化**：クラスが何に依存しているか一目瞭然
- **初期化忘れの防止**：オブジェクト生成時に必要な依存がすべて揃うことが保証される
- **God Classの検出**：引数が多すぎる場合は責任過多の警告サイン

### 3. 実装差し替えの容易さ
- **インターフェースに依存**：具体的な実装ではなく抽象に依存
- **環境別実装**：開発環境と本番環境で異なる実装を差し替え可能
- **将来の変更に対応**：実装変更時に利用側のコードを変更不要

### 4. 疎結合な設計
- **Laravel非依存のドメイン層**：ビジネスロジックがフレームワークに依存しない
- **再利用性**：同じサービスをController、Command、Job、Listenerなど複数の場所で使用可能

---

## 📝 実装パターン

### 悪い例：密結合なコード（DIなし）

```php
class PaymentService
{
    public function charge(int $amount, string $token): void
    {
        // 悪い点: クラス内で依存先を生成している（密結合）
        // テスト時にここをモックに差し替えるのが非常に困難
        $stripe = new StripeClient(config('services.stripe.secret'));
        $stripe->charges->create([...]);
    }
}
```

**問題点：**
- テスト時にStripeのAPIを実際に叩いてしまう
- モックに差し替えるのが困難
- 実装変更時にコード修正が必要

### 良い例：コンストラクタインジェクション（DIあり）

```php
// 1. インターフェース定義
interface PaymentGatewayInterface
{
    public function charge(int $amount, string $token): void;
}

// 2. 実装クラス
readonly class StripeGateway implements PaymentGatewayInterface
{
    public function __construct(
        private StripeClient $client
    ) {}
    
    public function charge(int $amount, string $token): void
    {
        $this->client->charges->create([...]);
    }
}

// 3. 利用側クラス
readonly class PurchaseService
{
    public function __construct(
        private PaymentGatewayInterface $gateway
    ) {}
    
    public function executePurchase(int $amount, string $token): void
    {
        $this->gateway->charge($amount, $token);
    }
}
```

**メリット：**
- インターフェースに依存している（疎結合）
- テスト時にモックを注入可能
- 実装変更時に利用側のコード変更不要

---

## 🧪 テストコードでの違い

### DIありの場合

```php
test('実際に課金せずに購入フローが成功することを確認する', function () {
    // 1. モックを作成
    $mockGateway = Mockery::mock(PaymentGatewayInterface::class);
    
    // 2. モックの振る舞いを定義
    $mockGateway->shouldReceive('charge')
        ->once()
        ->with(1000, 'tok_visa')
        ->andReturnNull();

    // 3. コンストラクタ経由でモックを注入
    $service = new PurchaseService($mockGateway);

    // 4. 実行（外部通信は発生しない）
    $service->executePurchase(1000, 'tok_visa');
});
```

**メリット：**
- Laravelを起動せずにテスト可能
- 外部APIを実際に叩かない
- 純粋なPHPUnitのTestCaseでテストできる

---

## ⚙️ Laravel 11での設定

### Service Providerでのバインディング

```php
// app/Providers/AppServiceProvider.php
class AppServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        // StripeClientの設定
        $this->app->singleton(StripeClient::class, function () {
            return new StripeClient(config('services.stripe.secret'));
        });

        // インターフェースと具象クラスのバインディング
        $this->app->bind(PaymentGatewayInterface::class, StripeGateway::class);
    }
}
```

Laravel 11では、型宣言さえしておけばコンテナが自動でインスタンスを注入してくれます。

---

## 🎯 なぜ「コンストラクタ」インジェクションなのか？

メソッドインジェクションやセッターインジェクションもありますが、**コンストラクタインジェクションが推奨される理由**：

1. **完全性の保証**：オブジェクト生成時に必要な依存がすべて揃うことが保証される
2. **イミュータビリティ**：PHP 8.1以降の`readonly`プロパティと相性が良く、不変なクラスを作れる
3. **依存の可視化**：コンストラクタの引数リストで依存関係が一目瞭然

---

## 📋 実務での導入ステップ

### Step 1: 依存を洗い出す
- 各クラスで`new`している箇所を特定
- Facadeや`app()`ヘルパを直接呼んでいる箇所を列挙

### Step 2: ロジックをサービスクラスに切り出す
- Controllerからビジネスロジックを`App\Application`や`App\Domain`配下へ移す
- サービスクラスはコンストラクタで依存を受け取る

### Step 3: インターフェースを定義
- 外部サービス・永続化・メッセージングなど、差し替えが必要な箇所にインターフェースを定義
- `App\Domain\*`にinterface、`App\Infrastructure\*`に実装を配置

### Step 4: Service Providerでバインド
- `register()`で`bind`/`singleton`を設定
- 専用プロバイダを作ると見通しが良くなる

### Step 5: コントローラではサービスだけをDI
- ControllerはRequest → DTO、Service呼び出し、Response構築に専念
- DB/外部API/メール送信などはすべてサービスの責務

### Step 6: テストを書いて恩恵を体感
- ユニットテストで`new Service(new FakeRepository())`のようにDI
- Laravelをほぼ知らなくてもテストが書けることを実感

---

## ⚠️ アンチパターン

### 避けるべき書き方

1. **Service Locatorパターン**
   ```php
   // 悪い例
   public function __construct(Container $app) {
       $this->service = $app->make(SomeService::class);
   }
   ```
   → 依存関係が見えなくなる

2. **`app()`ヘルパの乱用**
   ```php
   // 悪い例
   $service = app(SomeService::class);
   ```
   → テストや静的解析で追いにくくなる

3. **過度な依存**
   - 1クラスのコンストラクタに10個以上の依存
   - → クラスが太りすぎているサイン。役割を分割する

---

## 💡 PHP 8.3での推奨記法

- **コンストラクタプロパティプロモーション**：記述量を減らす
- **`readonly`プロパティ/クラス**：不変オブジェクトを表現
- **`declare(strict_types=1);`**：厳密な型宣言
- **型宣言の徹底**：静的解析ツールの恩恵を最大限に

---

## 🚀 長期的なメリット（5〜10年スパン）

### 5年後あたりに効くこと
- **レガシー機能のリプレース**：ドメインサービスやUseCaseの差し替えだけで済む
- **フレームワークのメジャーバージョンアップ**：ドメイン層はほぼ無傷で済み、UI/インフラ層の修正に集中できる

### 10年スパンでの話
- **フレームワーク移行**：Laravelから別のフレームワークや別言語への移行が容易
- **マイクロサービス分割**：ポート＆アダプタ/Hexagonalな境界が引けていれば、サービスごとの切り出しが容易
- **ツール/エコシステムの進化**：静的解析ツールや自動リファクタリングツールとの相性が良い

---

## 📚 推奨事項

### 静的解析ツールの導入
- **推奨**：`larastan/larastan`を導入し、レベル8以上で運用
- **効果**：コード実行前にインターフェース不整合や型エラーを検知

### Clean Architecture / Hexagonal Architecture
- ビジネスロジックのコア部分では、Laravel特有の機能（FacadeやHelper）を使わず、インターフェースに依存
- 純粋なPHPクラスとして記述する習慣をつける

### コンテナとオートワイヤリングの進化
- PHP 8.x以降、Attributes（アトリビュート）を活用したDI設定が可能
- 将来的には属性ベースで依存解決を行うスタイルが主流になる可能性

---

## 🎓 まとめ

### コンストラクタインジェクションの主な価値

1. **依存関係の明示**：コンストラクタの引数で依存が可視化される
2. **実装差し替えの容易さ**：インターフェースに依存することで柔軟性が向上
3. **Laravel依存の薄いドメイン/アプリケーション層**：フレームワーク非依存のビジネスロジック
4. **純粋なPHPユニットテストのしやすさ**：Laravelを起動せずにテスト可能

### 重要な認識

- 「Facades / `app()`で十分」に見えるのは、**単一のHTTPエンドポイントしか見ていない視点**
- 実際にはCommand / Job / Listener / 別サービスから再利用し始めた途端に差が出る
- Laravel 11 & PHP 8.3の現在（2025年）では、**コンストラクタインジェクション + interface + Service Container**が、長期運用・スケール・フレームワーク変更まで見据えた標準解

---

## 📌 実践のポイント

1. **外部API通信**や**現在時刻依存**の箇所からDIへリファクタリングを開始
2. **テストを書く場面**を想定して設計する
3. **クラスの再利用**を前提に設計する
4. **インターフェース**を適切に定義し、実装と抽象を分離する

