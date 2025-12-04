---
title: Laravel 11 ベストプラクティス
created: 2025-12-03
updated: 2025-12-03
category: Laravel
tags:
  - laravel
  - php
  - best-practices
  - di
  - dependency-injection
aliases:
  - Laravel11
  - Laravelベストプラクティス
---

# Laravel 11 ベストプラクティス（2025年12月現在）

## 概要

このドキュメントは、Laravel 11を使用したサービスクラスの設計における、2025年12月現在のベストプラクティスをまとめたものです。

## 1. 依存性注入（DI）のベストプラクティス

### 1.1 コンストラクタインジェクションの徹底

**推奨:**
```php
readonly class ReservationService
{
    public function __construct(
        private readonly PaymentBusinessService $paymentBusinessService,
        private readonly CancelPolicyService $cancelPolicyService,
    ) {}
}
```

**非推奨:**
```php
class ReservationService
{
    public function __construct()
    {
        $this->paymentBusinessService = new PaymentBusinessService();
    }
}
```

### 1.2 サービスコンテナの適切な使用

#### singleton() vs bind() の使い分け

**singleton() を使用する場合:**
- ステートレスなサービス
- 設定やキャッシュなど、アプリケーション全体で1つのインスタンスで十分な場合
- パフォーマンスが重要な場合

```php
$this->app->singleton(ReservationApiService::class);
$this->app->singleton(LanguageService::class);
```

**bind() を使用する場合:**
- ステートフルなサービス
- リクエストごとに新しいインスタンスが必要な場合
- テストでモックを注入しやすいようにする場合

```php
$this->app->bind(ReservationService::class);
$this->app->bind(ReservationCommodityService::class);
```

### 1.3 インターフェースの活用

依存関係を抽象化することで、テストが容易になります：

```php
interface PaymentServiceInterface
{
    public function processPayment(array $data): bool;
}

class PaymentBusinessService implements PaymentServiceInterface
{
    // 実装
}

// ServiceProviderで登録
$this->app->bind(
    PaymentServiceInterface::class,
    PaymentBusinessService::class
);
```

## 2. 循環参照の解決パターン

### 2.1 イベント駆動アーキテクチャ（推奨）

サービス間の直接的な依存を避ける：

```php
// イベントクラス
class CommodityReservationCanceled
{
    public function __construct(
        public readonly TReservationCommodity $commodity,
        public readonly bool $paymentResult,
    ) {}
}

// サービスA
class ReservationCommodityService
{
    public function cancel(int $id, FormRequest $request): void
    {
        $commodity = TReservationCommodity::findOrFail($id);
        $this->cancelReserve($commodity, $request);
        
        // イベント発火
        event(new CommodityReservationCanceled($commodity, true));
    }
}

// サービスB（イベントリスナー）
class PaymentBusinessService
{
    public function handle(CommodityReservationCanceled $event): void
    {
        // 決済処理
        $this->processRefund($event->commodity);
    }
}
```

### 2.2 中間サービスの導入

共通の処理を中間サービスに切り出す：

```php
class ReservationPaymentCoordinatorService
{
    public function __construct(
        private readonly PaymentBusinessService $paymentService,
        private readonly ReservationCommodityService $commodityService,
    ) {}
    
    public function cancelCommodityReservation(int $id, FormRequest $request): void
    {
        $commodity = TReservationCommodity::findOrFail($id);
        
        $paymentResult = $this->paymentService
            ->onCommodityReservationCancel($commodity, $request);
        
        try {
            $this->commodityService->cancelReserve($commodity, $request);
        } catch (Throwable $ex) {
            $this->paymentService->onCommodityReservationCancelFailed(
                $commodity, 
                $paymentResult
            );
            throw $ex;
        }
    }
}
```

## 3. テストのベストプラクティス

### 3.1 Laravel 11の新しいテストヘルパー

```php
use Tests\TestCase;

class ReservationServiceTest extends TestCase
{
    public function test_some_method(): void
    {
        // Laravel 11の新しいモックヘルパー
        $this->mock(PaymentBusinessService::class, function ($mock) {
            $mock->shouldReceive('someMethod')
                ->once()
                ->andReturn(['result' => 'success']);
        });
        
        $service = app(ReservationService::class);
        $result = $service->someMethod();
        
        $this->assertNotNull($result);
    }
}
```

### 3.2 依存関係のモック

```php
use Mockery;

class ReservationServiceTest extends TestCase
{
    public function test_with_mockery(): void
    {
        $mockPaymentService = Mockery::mock(PaymentBusinessService::class);
        $mockPaymentService->shouldReceive('processPayment')
            ->once()
            ->with(Mockery::type('array'))
            ->andReturn(true);
        
        $this->app->instance(PaymentBusinessService::class, $mockPaymentService);
        
        $service = app(ReservationService::class);
        $result = $service->createReservation($data);
        
        $this->assertTrue($result);
    }
}
```

## 4. パフォーマンス最適化

### 4.1 サービス解決の最適化

- 不要なシングルトン登録を避ける（メモリ使用量の増加）
- 依存関係の解決順序を最適化
- 遅延ローディングを適切に使用

### 4.2 キャッシュの活用

```php
// サービス解決のキャッシュ（Laravel 11では自動的に最適化）
$this->app->singleton(ExpensiveService::class, function ($app) {
    return new ExpensiveService(
        $app->make(HeavyDependency::class)
    );
});
```

## 5. コード品質の向上

### 5.1 型安全性（PHP 8.3対応）

```php
readonly class ReservationService
{
    public function __construct(
        private readonly PaymentBusinessService $paymentBusinessService,
        // 型を明示的に指定
    ) {}
    
    public function createReservation(array $data): Reservation
    {
        // 型安全な処理
    }
}
```

### 5.2 属性（Attributes）の活用

```php
use Illuminate\Contracts\Container\BindingResolutionException;

class ReservationService
{
    public function __construct(
        #[Inject]
        private readonly PaymentBusinessService $paymentBusinessService,
    ) {}
}
```

## 6. アーキテクチャパターン

### 6.1 レイヤードアーキテクチャ

```
Controllers
    ↓
Services (Business Logic)
    ↓
Repositories / Models (Data Access)
    ↓
Database
```

### 6.2 ドメイン駆動設計（DDD）の要素

- エンティティ: モデル
- 値オブジェクト: 値の型
- リポジトリ: データアクセス
- サービス: ビジネスロジック

## 7. よくある間違いと回避方法

### 7.1 サービスロケーターパターンの使用

**非推奨:**
```php
class ReservationService
{
    public function someMethod()
    {
        $service = app(PaymentBusinessService::class); // サービスロケーター
    }
}
```

**推奨:**
```php
class ReservationService
{
    public function __construct(
        private readonly PaymentBusinessService $paymentService,
    ) {}
    
    public function someMethod()
    {
        $this->paymentService->process(); // 依存性注入
    }
}
```

### 7.2 グローバル関数の使用

**非推奨:**
```php
$service = resolve(ReservationService::class);
```

**推奨:**
```php
$service = app(ReservationService::class);
// または
$service = $this->app->make(ReservationService::class);
```

## 8. まとめ

2025年12月現在のLaravel 11ベストプラクティス：

1. ✅ コンストラクタインジェクションの徹底
2. ✅ サービスコンテナの適切な使用（singleton vs bind）
3. ✅ インターフェースの活用
4. ✅ イベント駆動アーキテクチャで循環参照を回避
5. ✅ 型安全性の向上（PHP 8.3対応）
6. ✅ テストしやすい設計
7. ✅ パフォーマンス最適化

これらのベストプラクティスを適用することで、保守性が高く、テストしやすい、拡張可能なコードベースを構築できます。

