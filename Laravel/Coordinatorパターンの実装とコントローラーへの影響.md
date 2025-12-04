---
title: Coordinatorパターンの実装とコントローラーへの影響
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
  - Coordinator実装
  - コントローラー影響
---

# Coordinatorパターンの実装とコントローラーへの影響

## 質問への回答

### 1. CoordinatorとMediatorの違い

詳細は [[Laravel/CoordinatorとMediatorパターンの違い|CoordinatorとMediatorパターンの違い]] を参照してください。

**簡単に言うと：**
- **Coordinator（調整者）**: 複数のサービスを調整して一連の処理を実行（コントローラーから呼ばれる）
- **Mediator（仲介者）**: サービス間の直接的な依存を排除（サービス内部で使用）

### 2. コントローラーから呼ぶサービスが変わるか？

**答え：場合による**

## 実装パターン別の影響

### パターン1: 完全にCoordinatorに置き換える（変更大）

**コントローラーが変わる**: ✅ **はい**

```php
// 変更前
class ReservationApiController extends Controller
{
    public function __construct(private readonly ReservationApiService $o_service) {}
    
    public function reserve(ReserveRequest $o_request)
    {
        return response()->json($this->o_service->reserve($o_request, ...));
    }
}

// 変更後
class ReservationApiController extends Controller
{
    public function __construct(
        private readonly ReservationCoordinatorService $coordinator  // 変更
    ) {}
    
    public function reserve(ReserveRequest $o_request)
    {
        return response()->json($this->coordinator->createReservation($o_request, ...));  // 変更
    }
}
```

**メリット：**
- ✅ コントローラーがシンプル
- ✅ 処理の順序が明確

**デメリット：**
- ❌ すべてのコントローラーを変更する必要がある
- ❌ 既存のサービスメソッドが使えなくなる可能性

### パターン2: 部分的にCoordinatorを導入（変更小）✅ 推奨

**コントローラーは変更なし、または最小限**

```php
// 変更前（そのまま）
class ReservationApiController extends Controller
{
    public function __construct(private readonly ReservationApiService $o_service) {}
    
    public function reserve(ReserveRequest $o_request)
    {
        // ReservationApiService は内部で app()->make() を使用
        return response()->json($this->o_service->reserve($o_request, ...));
    }
}

// 変更後（変更なし、または最小限）
class ReservationApiController extends Controller
{
    public function __construct(private readonly ReservationApiService $o_service) {}  // 変更なし
    
    public function reserve(ReserveRequest $o_request)
    {
        // ReservationApiService の内部実装は変更されるが、コントローラーは変更なし
        return response()->json($this->o_service->reserve($o_request, ...));  // 変更なし
    }
}
```

**ReservationApiService の内部実装：**

```php
// ReservationApiService.php
readonly class ReservationApiService
{
    public function __construct(
        // 循環参照を避けるため、直接依存を削除
        // private readonly ReservationService $reservation_service,  // 削除
        private readonly ReservationCommodityService $commodity_service,
    ) {}

    public function reserve(ReserveRequest $o_request, int $i_agent_cd)
    {
        // 内部で app()->make() を使用（循環参照を回避）
        $s_new_reserve_code = app()->make(ReservationService::class)
            ->createReservationCode($o_request)->first();
        
        // 他の処理...
    }
}
```

**メリット：**
- ✅ コントローラーは変更不要
- ✅ 既存のAPIインターフェースを維持
- ✅ 段階的に導入可能

**デメリット：**
- ❌ サービス内部で `app()->make()` を使用（テストがやや複雑）

### パターン3: 新規エンドポイントのみCoordinatorを使用（段階的導入）

**既存のコントローラーは変更なし、新規のみCoordinatorを使用**

```php
// 既存のコントローラー（変更なし）
class ReservationApiController extends Controller
{
    public function __construct(private readonly ReservationApiService $o_service) {}
    
    public function reserve(ReserveRequest $o_request)
    {
        return response()->json($this->o_service->reserve($o_request, ...));
    }
}

// 新規のコントローラー（Coordinatorを使用）
class ReservationV2ApiController extends Controller
{
    public function __construct(
        private readonly ReservationCoordinatorService $coordinator
    ) {}
    
    public function reserve(ReserveRequest $o_request)
    {
        return response()->json($this->coordinator->createReservation($o_request, ...));
    }
}
```

**メリット：**
- ✅ 既存コードへの影響が最小限
- ✅ 段階的に移行可能
- ✅ リスクが低い

**デメリット：**
- ❌ 一時的に2つの実装が存在

## 推奨アプローチ

### 現時点では：パターン2（部分的導入）

**理由：**
1. コントローラーへの影響が最小限
2. 既存のAPIインターフェースを維持
3. 段階的に導入可能
4. リスクが低い

**実装例：**

```php
// コントローラー（変更なし）
class ReservationApiController extends Controller
{
    public function __construct(private readonly ReservationApiService $o_service) {}
    
    public function reserve(ReserveRequest $o_request)
    {
        // インターフェースは変更なし
        return response()->json($this->o_service->reserve($o_request, ...));
    }
}

// ReservationApiService（内部実装のみ変更）
readonly class ReservationApiService
{
    public function __construct(
        // 循環参照を避けるため、直接依存を削除
        private readonly ReservationCommodityService $commodity_service,
        private readonly SnapshotService $snapshot_service,
        private readonly ReservationChargeTypeService $charge_type_service,
    ) {}

    public function reserve(ReserveRequest $o_request, int $i_agent_cd)
    {
        // 内部で app()->make() を使用（循環参照を回避）
        $reservation_service = app()->make(ReservationService::class);
        $s_new_reserve_code = $reservation_service->createReservationCode($o_request)->first();
        
        // 他の処理...
    }
}
```

### 将来的には：パターン1（完全なCoordinator導入）

**段階的に移行：**
1. パターン2で循環参照を解決
2. テストを充実させる
3. パターン1に段階的に移行

## 現在のプロジェクトへの適用

### 現在のコントローラー

```php
// TReservationController.php
class TReservationController extends Controller
{
    public function __construct(private readonly ReservationService $o_service) {}
    
    public function search(SearchRequest $o_request)
    {
        list($i_count, $r_rows) = $this->o_service->search($o_request);
        return response()->json(["count" => $i_count, "rows" => $r_rows]);
    }
}

// ReservationApiController.php
class ReservationApiController extends Controller
{
    public function __construct(private readonly ReservationApiService $o_service) {}
    
    public function reserve(ReserveRequest $o_request)
    {
        return response()->json($this->o_service->reserve($o_request, ...));
    }
}
```

### 推奨される変更

**コントローラーは変更不要**（パターン2）

```php
// コントローラー（変更なし）
class ReservationApiController extends Controller
{
    public function __construct(private readonly ReservationApiService $o_service) {}  // 変更なし
    
    public function reserve(ReserveRequest $o_request)
    {
        // インターフェースは変更なし
        return response()->json($this->o_service->reserve($o_request, ...));  // 変更なし
    }
}
```

**サービス内部のみ変更**（循環参照を回避）

```php
// ReservationApiService.php（内部実装のみ変更）
readonly class ReservationApiService
{
    // 循環参照を避けるため、直接依存を削除
    // private readonly ReservationService $reservation_service,  // 削除
    
    public function reserve(ReserveRequest $o_request, int $i_agent_cd)
    {
        // 内部で app()->make() を使用（循環参照を回避）
        $reservation_service = app()->make(ReservationService::class);
        $s_new_reserve_code = $reservation_service->createReservationCode($o_request)->first();
        
        // 他の処理...
    }
}
```

## まとめ

### 質問への回答

1. **CoordinatorとMediatorの違い**
   - Coordinator: コントローラーから呼ばれる調整サービス
   - Mediator: サービス内部で使用される仲介サービス

2. **コントローラーから呼ぶサービスが変わるか？**
   - **パターン1（完全置き換え）**: ✅ はい、変わる
   - **パターン2（部分導入）**: ❌ いいえ、変わらない（推奨）
   - **パターン3（段階的導入）**: 新規のみ変わる

### 推奨

**現時点ではパターン2（部分導入）を推奨**
- コントローラーは変更不要
- サービス内部のみ変更
- 既存のAPIインターフェースを維持
- リスクが低い

## 関連ドキュメント

- [[Laravel/CoordinatorとMediatorパターンの違い|CoordinatorとMediatorパターンの違い]] - パターンの詳細な比較
- [[Laravel/一方向依存関係のルール強制方法|一方向依存関係のルール強制方法]] - Deptracによる依存関係の検証
- [[Laravel/予約ドメインの循環参照解決とサブドメイン分割戦略|予約ドメインの循環参照解決とサブドメイン分割戦略]] - 循環参照解決の全体戦略
- [[Laravel/サービス依存関係リファクタリング計画|サービス依存関係リファクタリング計画]] - リファクタリング計画
- [[Index|ホーム]] - 目次に戻る

