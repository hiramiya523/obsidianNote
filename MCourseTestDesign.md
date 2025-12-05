# MCourse モデル テスト設計書

## 1. 概要

本ドキュメントは、`App\Models\Master\MCourse` モデルのユニットテスト設計方針を定義します。
チーム全体で共有するテスト設計のガイドラインとして、5年後、10年後の長期運用を見据えた包括的なテスト戦略を提供します。

### 1.1 対象モデル
- **モデル名**: `MCourse`
- **テーブル**: `m_courses`
- **主要な責務**:
  - コースマスタデータの管理
  - キャンセルポリシーの取得（コース/主催会社の切り替え）
  - 決済方法情報の取得（コース/主催会社の切り替え）
  - 多言語対応データの管理

### 1.2 設計の基本方針

1. **保守性**: 5年後、10年後も理解しやすく、拡張しやすいテストコード
2. **網羅性**: 正常系・異常系・境界値を適切にカバー
3. **実用性**: テストファースト実装とのバランスを考慮
4. **一貫性**: 既存のテストパターンとの整合性を保つ

---

## 2. テスト範囲とカバレッジ

### 2.1 テスト対象メソッド

#### 必須テスト（Critical Path）
以下のメソッドはビジネスロジックの核心であり、必ずテストを実装します：

1. **`getCurrentCancelPolicies()`**
   - コースのキャンセルポリシー取得
   - 主催会社のキャンセルポリシー取得
   - フラグによる切り替えロジック

2. **`currentPaymentFreeText()`**
   - コースの決済方法フリーテキスト取得
   - 主催会社の決済方法フリーテキスト取得
   - 多言語データの取得

3. **`currentCreditPaymentFlag()`**
   - コースのクレジット決済フラグ取得
   - 主催会社のクレジット決済フラグ取得

4. **`currentCashPaymentFlg()`**
   - コースの現金決済フラグ取得
   - 主催会社の現金決済フラグ取得

#### 推奨テスト（Recommended）
リレーション定義は、必要に応じてテストを追加します：

- リレーションメソッド（`plan()`, `courseCancelPolicies()`, `languages()` など）
  - 基本的にはEloquentの標準機能のため、テストは任意
  - ただし、カスタムロジック（`orderBy`, `withPivot` など）がある場合はテスト推奨

### 2.2 カバレッジ目標

- **必須メソッド**: 100% カバレッジ
- **推奨メソッド**: 80% 以上
- **全体**: 80% 以上を目標

---

## 3. 正常系・異常系の扱い

### 3.1 正常系テスト（必須）

すべてのメソッドについて、正常系のテストは必須です。

#### テストパターン

1. **フラグによる分岐テスト**
   - `i_use_host_cxl_policy_flg = 0` の場合
   - `i_use_host_cxl_policy_flg = 1` の場合
   - `i_use_host_payment_free_text_flg = 0` の場合
   - `i_use_host_payment_free_text_flg = 1` の場合
   - `i_use_host_payment_method_flg = 0` の場合
   - `i_use_host_payment_method_flg = 1` の場合

2. **リレーションのロード確認**
   - 必要なリレーションが正しくロードされているか
   - 多言語データが正しく取得できているか

3. **データの整合性確認**
   - 返却値の型が正しいか
   - 返却値の構造が期待通りか

### 3.2 異常系テスト（必須）

異常系テストは、**ビジネスロジックに影響を与える可能性がある場合**に必須です。

#### 必須テストケース

1. **リレーションが存在しない場合**
   - `plan` が `null` の場合（`i_use_host_*_flg = 1` の時）
   - `hostCompany` が `null` の場合
   - `courseCancelPolicies` が空の場合

2. **データ不整合の場合**
   - フラグが期待値以外（0, 1以外）の場合
   - 多言語データが存在しない場合

3. **例外処理**
   - `match` 式で未定義の値が渡された場合（将来的な拡張への備え）

#### 実装方針

```php
// 例: リレーションが存在しない場合のテスト
function test_getCurrentCancelPolicies_throws_when_plan_is_null()
{
    $course = MCourse::factory()->create([
        'i_plan_cd' => 99999, // 存在しないプランID
        'i_use_host_cxl_policy_flg' => 1
    ]);
    
    $this->expectException(\Illuminate\Database\Eloquent\ModelNotFoundException::class);
    $course->getCurrentCancelPolicies();
}
```

### 3.3 異常系テストの優先度

| 優先度 | ケース | 理由 |
|--------|--------|------|
| **高** | リレーションが存在しない場合 | データ整合性に直接影響 |
| **高** | フラグ値が不正な場合 | ビジネスロジックの分岐に影響 |
| **中** | 多言語データが存在しない場合 | 機能は動作するが、データ不整合の可能性 |
| **低** | 想定外の型が渡された場合 | PHPの型システムで防げる場合が多い |

---

## 4. 境界値テスト

### 4.1 境界値テストの方針

境界値テストは、**ビジネスロジックに境界値が存在する場合**に実施します。

#### MCourseモデルの境界値

1. **フラグ値の境界値**
   - `i_use_host_cxl_policy_flg`: `0` と `1`（必須）
   - `i_use_host_payment_free_text_flg`: `0` と `1`（必須）
   - `i_use_host_payment_method_flg`: `0` と `1`（必須）

2. **コレクションの境界値**
   - キャンセルポリシーが0件の場合
   - キャンセルポリシーが1件の場合
   - キャンセルポリシーが複数件の場合

3. **多言語データの境界値**
   - 言語データが0件の場合
   - 言語データが1件の場合
   - 言語データが複数件の場合

### 4.2 境界値テストの実装例

```php
/**
 * @dataProvider provide_flag_boundary_values
 */
function test_getCurrentCancelPolicies_with_boundary_flags(int $flag, bool $expectsHostPolicy)
{
    $course = MCourse::factory()->create([
        'i_use_host_cxl_policy_flg' => $flag
    ]);
    
    // テスト実装
}

public static function provide_flag_boundary_values(): array
{
    return [
        'flag_0' => [0, false],
        'flag_1' => [1, true],
    ];
}
```

### 4.3 境界値テストの優先度

| 優先度 | ケース | 理由 |
|--------|--------|------|
| **高** | フラグ値の境界値（0, 1） | ビジネスロジックの分岐点 |
| **中** | コレクションが空の場合 | エッジケースとして重要 |
| **中** | コレクションが1件の場合 | 最小データセット |
| **低** | コレクションが大量の場合 | パフォーマンステストの範疇 |

---

## 5. テストファースト実装とのバランス

### 5.1 TDD（テスト駆動開発）の適用方針

#### 完全なTDDを適用するケース
- **新規メソッドの追加**: 必ずテストファーストで実装
- **ビジネスロジックの変更**: テストを先に書いてから実装

#### 実装後にテストを追加するケース
- **既存コードのリファクタリング**: リファクタリング後にテストを追加・更新
- **バグ修正**: バグ修正後に回帰テストを追加

### 5.2 テストの粒度

#### 細かくテストする（推奨）
- ビジネスロジックが複雑なメソッド
- 複数の分岐があるメソッド
- 外部リレーションに依存するメソッド

#### 簡潔にテストする（許容）
- 単純なgetter/setter
- Eloquentの標準機能のみを使用するリレーション

### 5.3 テストの実行速度

- **ユニットテスト**: 1秒以内（モックを使用）
- **統合テスト**: 5秒以内（必要最小限のDBアクセス）

---

## 6. テストコードの品質基準

### 6.1 命名規則

#### テストメソッド名
```
test_{メソッド名}_{シナリオ}_{期待結果}
```

例:
- `test_getCurrentCancelPolicies_returns_course_policies_when_flag_is_0`
- `test_getCurrentCancelPolicies_returns_host_policies_when_flag_is_1`
- `test_getCurrentCancelPolicies_throws_exception_when_plan_is_null`

#### テストクラス名
```
{ModelName}Test
```

### 6.2 テストの構造

#### AAA パターン（Arrange-Act-Assert）

```php
function test_example()
{
    // Arrange: テストデータの準備
    $course = MCourse::factory()->create(['i_use_host_cxl_policy_flg' => 0]);
    $policy = MCourseCancelPolicy::create([...]);
    
    // Act: テスト対象の実行
    $result = $course->getCurrentCancelPolicies();
    
    // Assert: 結果の検証
    $this->assertCount(1, $result);
    $this->assertTrue($result->contains($policy));
}
```

### 6.3 アサーションのベストプラクティス

1. **明確なメッセージ**: すべてのアサーションにメッセージを付与
2. **適切なアサーションメソッド**: `assertSame()` を優先（型と値の両方をチェック）
3. **1つのテストで1つの概念**: 1つのテストメソッドで複数の概念をテストしない

### 6.4 モックの使用方針

#### モックを使用するケース
- 外部サービスへの依存
- データベースアクセスを避けたい場合
- テストの実行速度を向上させたい場合

#### モックを使用しないケース
- Eloquentモデルのリレーション（実際のDBを使用）
- ビジネスロジックの検証（実際のデータで検証）

#### モックの実装例

```php
// Mockeryを使用したモック
$mockPlan = Mockery::mock(MPlan::class);
$mockPlan->shouldReceive('hostCompany')->andReturn($hostCompany);
$course->setRelation('plan', $mockPlan);
```

---

## 7. データプロバイダーの活用

### 7.1 データプロバイダーの使用推奨ケース

1. **複数の入力パターンをテストする場合**
2. **境界値テスト**
3. **同じロジックを異なるデータで検証する場合**

### 7.2 実装例

```php
/**
 * @dataProvider provide_payment_flag_combinations
 */
function test_currentCreditPaymentFlag_with_various_combinations(
    int $useHostFlag,
    int $courseFlag,
    int $hostFlag,
    int $expected
) {
    // テスト実装
}

public static function provide_payment_flag_combinations(): array
{
    return [
        'use_course_flag_0' => [0, 0, 1, 0],
        'use_course_flag_1' => [0, 1, 0, 1],
        'use_host_flag_0' => [1, 1, 0, 0],
        'use_host_flag_1' => [1, 0, 1, 1],
    ];
}
```

---

## 8. リレーションのテスト

### 8.1 リレーションのテスト方針

#### テスト必須
- カスタムロジックを含むリレーション（`orderBy`, `withPivot` など）
- ビジネスロジックに影響するリレーション

#### テスト任意
- Eloquentの標準機能のみを使用するリレーション

### 8.2 リレーションテストの実装例

```php
function test_courseCancelPolicies_relation_is_ordered_by_id()
{
    $course = MCourse::factory()->create();
    
    $policy1 = MCourseCancelPolicy::create(['i_course_cd' => $course->id, ...]);
    $policy2 = MCourseCancelPolicy::create(['i_course_cd' => $course->id, ...]);
    
    $policies = $course->courseCancelPolicies;
    
    $this->assertSame($policy1->id, $policies->first()->id);
    $this->assertSame($policy2->id, $policies->last()->id);
}
```

---

## 9. 多言語データのテスト

### 9.1 多言語データのテスト方針

1. **基本ケース**: 日本語と英語の2言語
2. **エッジケース**: 言語データが存在しない場合
3. **複数言語**: 3言語以上の場合（将来的な拡張への備え）

### 9.2 実装例

```php
function test_currentPaymentFreeText_returns_multilingual_data()
{
    $langJa = MLanguage::firstOrCreate(['s_key' => 'ja'], [...]);
    $langEn = MLanguage::firstOrCreate(['s_key' => 'en'], [...]);
    
    $course = MCourse::factory()->create([
        'i_use_host_payment_free_text_flg' => 0,
        's_payment_free_text' => 'コースの決済方法テキスト'
    ]);
    
    $course->languages()->attach($langJa->id, [
        's_payment_free_text' => 'コースの決済方法テキスト（日本語）',
        ...
    ]);
    $course->languages()->attach($langEn->id, [
        's_payment_free_text' => 'Course payment text (English)',
        ...
    ]);
    
    $result = $course->currentPaymentFreeText();
    
    $this->assertArrayHasKey('r_languages', $result);
    $this->assertCount(2, $result['r_languages']);
    $this->assertSame('コースの決済方法テキスト（日本語）', 
        $result['r_languages'][$langJa->id]['s_payment_free_text']);
}
```

---

## 10. 長期運用を考慮した設計

### 10.1 テストの保守性

#### テストコードの可読性
- **明確なテスト名**: テストの意図が一目で分かる
- **適切なコメント**: 複雑なロジックにはコメントを追加
- **テストデータの明確化**: Factoryを使用し、テストデータの意図を明確に

#### テストの独立性
- 各テストは独立して実行可能であること
- テストの実行順序に依存しないこと
- データベースの状態に依存しないこと（必要に応じてトランザクションを使用）

### 10.2 拡張性への備え

#### 将来の変更に対応できる設計
- **データプロバイダーの活用**: 新しいテストケースを追加しやすい
- **テストヘルパーメソッド**: 共通処理をメソッド化
- **テストの構造化**: 関連するテストをグループ化

#### 実装例

```php
class MCourseTest extends TestCase
{
    // テストヘルパーメソッド
    private function createCourseWithHostCompany(int $useHostFlag): MCourse
    {
        $hostCompany = MHostCompany::factory()->create();
        $plan = MPlan::factory()->create(['i_host_company_cd' => $hostCompany->id]);
        return MCourse::factory()->create([
            'i_plan_cd' => $plan->id,
            'i_use_host_cxl_policy_flg' => $useHostFlag
        ]);
    }
    
    // テストメソッド
    function test_getCurrentCancelPolicies_with_host_company()
    {
        $course = $this->createCourseWithHostCompany(1);
        // テスト実装
    }
}
```

### 10.3 パフォーマンスへの配慮

#### テストの実行速度
- **モックの活用**: 不要なDBアクセスを避ける
- **Factoryの最適化**: 必要なデータのみを作成
- **バッチ処理**: 複数のテストで同じデータを使用する場合は、`setUp()` で準備

### 10.4 ドキュメント化

#### テストコード自体がドキュメント
- **テスト名**: テストの意図を明確に表現
- **アサーションメッセージ**: 期待する動作を説明
- **コメント**: 複雑なロジックには説明を追加

---

## 11. 具体的なテストケース一覧

### 11.1 `getCurrentCancelPolicies()` のテストケース

| テストケース | 優先度 | 分類 |
|------------|--------|------|
| フラグ=0の時、コースのキャンセルポリシーを返す | 高 | 正常系 |
| フラグ=1の時、主催会社のキャンセルポリシーを返す | 高 | 正常系 |
| コースポリシーが0件の場合、空のコレクションを返す | 中 | 境界値 |
| 主催会社ポリシーが0件の場合、空のコレクションを返す | 中 | 境界値 |
| 複数のポリシーが存在する場合、すべて取得できる | 高 | 正常系 |
| languagesリレーションがロードされている | 高 | 正常系 |
| フラグ=1の時、planが存在しない場合の例外 | 高 | 異常系 |
| フラグ=1の時、hostCompanyが存在しない場合の例外 | 高 | 異常系 |

### 11.2 `currentPaymentFreeText()` のテストケース

| テストケース | 優先度 | 分類 |
|------------|--------|------|
| フラグ=0の時、コースの決済方法テキストを返す | 高 | 正常系 |
| フラグ=1の時、主催会社の決済方法テキストを返す | 高 | 正常系 |
| 多言語データが正しく取得できる | 高 | 正常系 |
| 言語データが0件の場合、空の配列を返す | 中 | 境界値 |
| 複数言語（2言語以上）が正しく取得できる | 中 | 正常系 |
| フラグ=1の時、planが存在しない場合の例外 | 高 | 異常系 |

### 11.3 `currentCreditPaymentFlag()` のテストケース

| テストケース | 優先度 | 分類 |
|------------|--------|------|
| フラグ=0の時、コースのクレジット決済フラグを返す | 高 | 正常系 |
| フラグ=1の時、主催会社のクレジット決済フラグを返す | 高 | 正常系 |
| フラグ値の境界値（0, 1） | 高 | 境界値 |
| フラグ=1の時、planが存在しない場合の例外 | 高 | 異常系 |

### 11.4 `currentCashPaymentFlg()` のテストケース

| テストケース | 優先度 | 分類 |
|------------|--------|------|
| フラグ=0の時、コースの現金決済フラグを返す | 高 | 正常系 |
| フラグ=1の時、主催会社の現金決済フラグを返す | 高 | 正常系 |
| フラグ値の境界値（0, 1） | 高 | 境界値 |
| フラグ=1の時、planが存在しない場合の例外 | 高 | 異常系 |

---

## 12. テスト実行とCI/CD

### 12.1 テストの実行方法

```bash
# すべてのテストを実行
php artisan test tests/Unit/Models/MCourseTest.php

# 特定のテストメソッドを実行
php artisan test --filter test_getCurrentCancelPolicies

# カバレッジレポートを生成
php artisan test --coverage
```

### 12.2 CI/CDでのテスト実行

- **プルリクエスト時**: すべてのテストを実行
- **マージ時**: カバレッジレポートを生成
- **デプロイ前**: 必須テストを実行

### 12.3 テストの失敗時の対応

1. **ローカルで再現**: 失敗したテストをローカルで実行
2. **原因の特定**: エラーメッセージとスタックトレースを確認
3. **修正と検証**: 修正後、関連するテストをすべて実行

---

## 13. まとめ

### 13.1 テスト設計の要点

1. **正常系は必須**: すべてのメソッドについて正常系テストを実装
2. **異常系は選択的**: ビジネスロジックに影響する異常系を優先
3. **境界値は重要**: フラグ値やコレクションの境界値をテスト
4. **保守性を重視**: 5年後、10年後も理解しやすいテストコード

### 13.2 継続的な改善

- **定期的なレビュー**: テストコードの品質を定期的にレビュー
- **リファクタリング**: 重複コードや複雑なロジックをリファクタリング
- **ドキュメント更新**: 設計方針の変更時は本ドキュメントを更新
