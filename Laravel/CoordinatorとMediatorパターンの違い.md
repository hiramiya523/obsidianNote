---
title: CoordinatorとMediatorパターンの違い
created: 2025-12-03
updated: 2025-12-03
category: Laravel
tags:
  - laravel
  - php
  - coordinator
  - mediator
  - デザインパターン
  - 循環参照
aliases:
  - Coordinator vs Mediator
  - パターン比較
---

# Coordinator vs Mediator パターンの違い

## 概要

Coordinator（調整者）とMediator（仲介者）は似ていますが、**目的と責務が異なる**デザインパターンです。

## 1. Coordinator（調整者）パターン

### 目的
**複数のサービスを調整して、一連の処理を実行する**

### 特徴
- **複数のサービスを呼び出す**
- **処理の順序を制御する**
- **トランザクション管理を行う**
- **エラーハンドリングを統一する**

### 実装例

```php
// ReservationCoordinatorService.php
readonly class ReservationCoordinatorService
{
    public function __construct(
        private readonly ReservationService $reservation_service,
        private readonly ReservationApiService $api_service,
        private readonly ReservationVerificationService $verification_service,
    ) {}

    /**
     * 予約作成の一連の処理を調整
     */
    public function createReservationWithVerification(ReserveRequest $request): Reservation
    {
        // 1. 検証
        $this->verification_service->validateRequestCourse(...);
        
        // 2. 予約作成
        $reservation = $this->reservation_service->create(...);
        
        // 3. API処理（料金区分の同期など）
        $this->api_service->syncReservationChargeTypes(...);
        
        return $reservation;
    }
}
```

### 使用例（コントローラー）

```php
// コントローラー
public function reserve(ReserveRequest $request, ReservationCoordinatorService $coordinator)
{
    // Coordinatorを呼び出す（内部で複数のサービスを調整）
    $reservation = $coordinator->createReservationWithVerification($request);
    return response()->json($reservation);
}
```

### メリット
- ✅ コントローラーがシンプルになる
- ✅ 処理の順序が明確
- ✅ トランザクション管理が容易
- ✅ エラーハンドリングが統一される

### デメリット
- ❌ 新しいクラスが必要
- ❌ 調整ロジックが集中する
- ❌ コントローラーから直接サービスを呼べなくなる

## 2. Mediator（仲介者）パターン

### 目的
**サービス間の直接的な依存を排除し、間接的な通信を実現する**

### 特徴
- **サービス間の直接的な依存を削除**
- **イベントやメッセージで通信**
- **疎結合を実現**
- **サービスはMediatorを知らない**

### 実装例

```php
// ReservationMediatorService.php
readonly class ReservationMediatorService
{
    public function __construct(
        private readonly ReservationService $reservation_service,
        private readonly ReservationApiService $api_service,
    ) {}

    /**
     * ReservationServiceから呼ばれる（ReservationServiceはMediatorを知っている）
     */
    public function notifyReservationCreated(Reservation $reservation): void
    {
        // ReservationApiServiceに通知（ReservationServiceは直接呼ばない）
        $this->api_service->onReservationCreated($reservation);
    }

    /**
     * ReservationApiServiceから呼ばれる
     */
    public function notifyReservationUpdated(Reservation $reservation): void
    {
        // ReservationServiceに通知（ReservationApiServiceは直接呼ばない）
        $this->reservation_service->onReservationUpdated($reservation);
    }
}
```

### 使用例（サービス）

```php
// ReservationService.php
readonly class ReservationService
{
    public function __construct(
        private readonly ReservationMediatorService $mediator,  // Mediatorに依存
        // ReservationApiService への直接依存は削除
    ) {}

    public function create(...): Reservation
    {
        $reservation = $this->createReservation(...);
        
        // Mediator経由で通知（直接呼ばない）
        $this->mediator->notifyReservationCreated($reservation);
        
        return $reservation;
    }
}
```

### メリット
- ✅ サービス間の直接的な依存を排除
- ✅ 循環参照を完全に回避
- ✅ 疎結合を実現
- ✅ 拡張性が高い

### デメリット
- ❌ 実装が複雑
- ❌ デバッグが難しい
- ❌ 既存コードの大幅な変更が必要

## 3. 比較表

| 項目 | Coordinator | Mediator |
|------|-------------|----------|
| **目的** | 複数サービスの調整 | サービス間の仲介 |
| **呼び出し元** | コントローラー | サービス内部 |
| **依存関係** | Coordinatorがサービスに依存 | サービスがMediatorに依存 |
| **循環参照** | 解決しない（調整のみ） | 解決する（依存を排除） |
| **実装の複雑さ** | 低 | 高 |
| **既存コードへの影響** | 中（コントローラー変更） | 大（サービス変更） |

## 4. 実装例の比較

### Coordinatorパターン（提示した実装例）

```php
// コントローラー
public function reserve(ReserveRequest $request, ReservationCoordinatorService $coordinator)
{
    // Coordinatorを呼び出す
    $reservation = $coordinator->createReservationWithVerification($request);
    return response()->json($reservation);
}
```

**コントローラーから呼ぶサービスが変わる**: ✅ **はい**
- 以前: `ReservationService` を直接呼び出し
- 以後: `ReservationCoordinatorService` を呼び出し

### Mediatorパターン

```php
// コントローラー（変更なし）
public function reserve(ReserveRequest $request, ReservationService $service)
{
    // サービスを直接呼び出す（変更なし）
    $reservation = $service->create(...);
    return response()->json($reservation);
}

// ReservationService.php（内部でMediatorを使用）
public function create(...): Reservation
{
    $reservation = $this->createReservation(...);
    $this->mediator->notifyReservationCreated($reservation);  // Mediator経由
    return $reservation;
}
```

**コントローラーから呼ぶサービスが変わる**: ❌ **いいえ**
- コントローラーは変更なし
- サービス内部でMediatorを使用

## 5. どちらを選ぶべきか？

### Coordinatorを選ぶ場合
- ✅ コントローラーをシンプルにしたい
- ✅ 処理の順序を明確にしたい
- ✅ トランザクション管理を統一したい
- ✅ 既存のサービスを変更したくない

### Mediatorを選ぶ場合
- ✅ 循環参照を完全に解決したい
- ✅ サービス間の依存を完全に排除したい
- ✅ イベント駆動アーキテクチャを導入したい
- ✅ 既存コードの大幅な変更が可能

## 6. 推奨アプローチ

### 現時点では：Coordinatorパターン（部分的に）

**理由：**
1. 実装が簡単
2. コントローラーの変更は限定的
3. 既存のサービスを変更せずに済む
4. 循環参照を部分的に解決できる

**実装例：**

```php
// ReservationCoordinatorService.php
readonly class ReservationCoordinatorService
{
    public function __construct(
        private readonly ReservationService $reservation_service,
        private readonly ReservationApiService $api_service,
        private readonly ReservationVerificationService $verification_service,
    ) {}

    /**
     * 予約作成の一連の処理を調整
     * コントローラーからはこのメソッドを呼ぶ
     */
    public function createReservation(ReserveRequest $request): Reservation
    {
        // 検証 → 作成 → 同期 の順序で実行
        $this->verification_service->validateRequestCourse(...);
        $reservation = $this->reservation_service->create(...);
        $this->api_service->syncReservationChargeTypes(...);
        return $reservation;
    }
}
```

**コントローラーの変更：**

```php
// 変更前
public function reserve(ReserveRequest $request, ReservationService $service)
{
    $reservation = $service->create(...);
    return response()->json($reservation);
}

// 変更後
public function reserve(ReserveRequest $request, ReservationCoordinatorService $coordinator)
{
    $reservation = $coordinator->createReservation($request);
    return response()->json($reservation);
}
```

### 将来的には：イベント駆動アーキテクチャ

Mediatorパターンを発展させて、イベント駆動アーキテクチャを導入：

```php
// イベントを発火
event(new ReservationCreated($reservation));

// リスナーで処理
class ReservationApiService
{
    public function handle(ReservationCreated $event)
    {
        $this->syncReservationChargeTypes($event->reservation);
    }
}
```

## 7. まとめ

- **Coordinator**: コントローラーから呼ぶサービスが変わる（調整ロジックを集約）
- **Mediator**: コントローラーは変更なし（サービス内部で使用）
- **現時点の推奨**: Coordinatorパターン（部分的に）
- **将来的な推奨**: イベント駆動アーキテクチャ

## 関連ドキュメント

- [[Laravel/Coordinatorパターンの実装とコントローラーへの影響|Coordinatorパターンの実装とコントローラーへの影響]] - 実装パターンとコントローラーへの影響
- [[Laravel/一方向依存関係のルール強制方法|一方向依存関係のルール強制方法]] - Deptracによる依存関係の検証
- [[Projects/予約ドメインの循環参照解決とサブドメイン分割戦略|予約ドメインの循環参照解決とサブドメイン分割戦略]] - 循環参照解決の全体戦略
- [[Index|ホーム]] - 目次に戻る

