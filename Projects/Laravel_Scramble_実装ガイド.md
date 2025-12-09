# Laravel Scramble 実装ガイド: link_intersystem_api 導入記録

タグ: #laravel #scramble #openapi #api-documentation #gitlab-ci #implementation

---

## 📋 目次

- [[#実装概要]]
- [[#Step 1: Scramble導入と基本設定]]
- [[#Step 2: link_intersystem_api 用カスタマイズ]]
- [[#Step 3: Controllerアノテーション追加]]
- [[#Step 4: OpenAPIスキーマ生成]]
- [[#Step 5: GitLab CI統合]]
- [[#Step 6: 複数APIの分離設定]]
- [[#新規API作成時の手順]]
- [[#トラブルシューティング]]

---

## 実装概要

**目的**: Laravel APIからOpenAPIスキーマを自動生成し、型安全なAPIドキュメントを維持する

**使用ツール**:
- **Scramble (dedoc/scramble)** v0.13.8: Laravel 11対応、コード自動解析
- **GitLab CI**: 自動ドキュメント更新

**対象API**: `link_intersystem_api`（外部連携API）

---

## Step 1: Scramble導入と基本設定

### インストール

```bash
composer require dedoc/scramble
```

### 設定ファイル公開

```bash
php artisan vendor:publish --tag=scramble-config
```

これで `config/scramble.php` が生成されます。

---

## Step 2: link_intersystem_api 用カスタマイズ

### config/scramble.php の設定

```php
<?php

return [
    'info' => [
        'version' => '1.0.0',
        'title' => 'プロジェクト システム連携API',
    ],
    // APIルートのプレフィックス
    'api_path' => 'link_intersystem_api',
    // 生成対象のミドルウェア
    'middleware' => [
        'web',
        'api',
    ],
    // OpenAPI 3.1.0 サポート
    'openapi_version' => '3.1.0',
    // サーバー情報
    'servers' => [
        [
            'url' => env('APP_URL') . '/link_intersystem_api',
            'description' => 'API Server',
        ],
    ],
];
```

### ScrambleServiceProvider の作成

`app/Providers/ScrambleServiceProvider.php`:

```php
<?php

namespace App\Providers;

use Dedoc\Scramble\Scramble;
use Dedoc\Scramble\Support\Generator\OpenApi;
use Illuminate\Support\ServiceProvider;

class ScrambleServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        // link_intersystem_api のルートのみを対象にする
        Scramble::afterOpenApiGenerated(function (OpenApi $openApi) {
            // 必要に応じて後処理を追加
        });
    }
}
```

### ServiceProvider の登録

`bootstrap/app.php`:

```php
->withProviders([
    // ... 既存のプロバイダー
    App\Providers\ScrambleServiceProvider::class,
])
```

### composer.json にスクリプト追加

```json
{
  "scripts": {
    "openapi:export": "php artisan scramble:export",
    "openapi:export:json": "php artisan scramble:export --path=storage/api-docs/link_intersystem_api.json"
  }
}
```

---

## Step 3: Controllerアノテーション追加

Scrambleはコードから自動解析しますが、より詳細な情報を提供するためにPHPDocアノテーションを追加します。

### 例: IdempotentController

```php
<?php

namespace App\Http\Controllers\Api\LinkIntersystem;

use App\Traits\Api\ApiBearerAuth;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;

class IdempotentController extends Controller
{
    use ApiBearerAuth;

    /**
     * 冪等性キーを作成
     * 
     * @response array{key: string, expires_at: string}
     */
    public function createKey(IdempotentService $o_service, Request $o_request): JsonResponse
    {
        // 実装...
    }
}
```

### 例: MasterApiController

```php
<?php

namespace App\Http\Controllers\Api\LinkIntersystem;

use App\Traits\Api\ApiBearerAuth;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Response;

class MasterApiController extends Controller
{
    use ApiBearerAuth;

    /**
     * マスターデータを取得
     * 
     * @response array{data: array}
     */
    public function getMaster(): JsonResponse
    {
        // 実装...
    }
}
```

### 重要なポイント

- **PHPDocの `@response`**: レスポンス型を明示的に指定
- **戻り値型**: `JsonResponse` を指定することで型安全性が向上
- **FormRequest**: `rules()` メソッドからリクエストパラメータが自動抽出される

---

## Step 4: OpenAPIスキーマ生成

### 生成コマンド

```bash
# 開発環境で実行
cd /workspace/controller
composer openapi:export

# または直接
php artisan scramble:export

# 特定のパスに出力
php artisan scramble:export --path=storage/api-docs/link_intersystem_api.json
```

### 生成結果

- **OpenAPI 3.1.0 形式**
- **レスポンス型**: コードから自動解析
- **リクエストパラメータ**: FormRequestから自動抽出
- **Bearer認証**: 設定済み

### ブラウザで確認

```
http://localhost/docs/api
```

---

## Step 5: GitLab CI統合

### 方法1: Artifactsとして保存（シンプル）

`.gitlab-ci.yml` に追加:

```yaml
generate-openapi:
  stage: docs
  extends: .php-setup
  variables:
    DB_HOST: "db"
    FF_NETWORK_PER_BUILD: "true"
    HEALTHCHECK_TCP_PORT: "3306"
  services:
    - name: "$CI_DB_IMAGE"
      alias: db
      command:
        - --character-set-server=utf8mb4
        - --collation-server=utf8mb4_unicode_ci
      variables:
        MYSQL_DATABASE: test_db
        MYSQL_USER: test
        MYSQL_PASSWORD: test
        MYSQL_ROOT_PASSWORD: test
  script:
    - cp .env.testing .env || true
    - |
      echo "Waiting for database..."
      for i in $(seq 1 30); do
        if php -r "try { new PDO('mysql:host=db;dbname=test_db', 'test', 'test'); echo 'ok'; exit(0); } catch(Exception \$e) { exit(1); }" 2>/dev/null; then
          echo "Database is ready!"
          break
        fi
        sleep 2
      done
    - php artisan migrate --force
    - php artisan scramble:export
    - echo "OpenAPI schema generated successfully!"
    - cat storage/api-docs/link_intersystem_api.json | head -50
  artifacts:
    paths:
      - controller/storage/api-docs/link_intersystem_api.json
    expire_in: 1 month
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - when: manual
      allow_failure: true
```

### 方法2: リポジトリに自動コミット（推奨）

```yaml
update-openapi-schema:
  stage: docs
  extends: .php-setup
  variables:
    DB_HOST: "db"
    FF_NETWORK_PER_BUILD: "true"
    HEALTHCHECK_TCP_PORT: "3306"
    GIT_STRATEGY: clone
    GIT_DEPTH: 0
  services:
    - name: "$CI_DB_IMAGE"
      alias: db
      command:
        - --character-set-server=utf8mb4
        - --collation-server=utf8mb4_unicode_ci
      variables:
        MYSQL_DATABASE: test_db
        MYSQL_USER: test
        MYSQL_PASSWORD: test
        MYSQL_ROOT_PASSWORD: test
  before_script:
    - cd controller
    - composer install --prefer-dist --no-progress --no-suggest
    # Git設定
    - git config --global user.email "ci@middle-db.com"
    - git config --global user.name "GitLab CI"
    - git remote set-url origin "https://oauth2:${GITLAB_TOKEN}@${CI_SERVER_HOST}/${CI_PROJECT_PATH}.git"
  script:
    - cp .env.testing .env || true
    - |
      echo "Waiting for database..."
      for i in $(seq 1 30); do
        if php -r "try { new PDO('mysql:host=db;dbname=test_db', 'test', 'test'); echo 'ok'; exit(0); } catch(Exception \$e) { exit(1); }" 2>/dev/null; then
          break
        fi
        sleep 2
      done
    - php artisan migrate --force
    - php artisan scramble:export
    # docs/api ディレクトリにもコピー
    - mkdir -p ../docs/api/generated
    - cp storage/api-docs/link_intersystem_api.json ../docs/api/generated/
    - cd ..
    # 変更があればコミット
    - |
      if git diff --quiet docs/api/generated/link_intersystem_api.json; then
        echo "No changes in OpenAPI schema"
      else
        echo "OpenAPI schema changed, committing..."
        git add docs/api/generated/link_intersystem_api.json
        git commit -m "docs: update OpenAPI schema [skip ci]"
        git push origin HEAD:${CI_COMMIT_REF_NAME}
      fi
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      changes:
        - "controller/app/Http/Controllers/**/*"
        - "controller/app/Http/Requests/**/*"
        - "controller/routes/**/*"
    - when: manual
      allow_failure: true
```

**必要な設定**:
1. GitLab → Settings → CI/CD → Variables に `GITLAB_TOKEN` を追加
2. Project Access Token（`write_repository` 権限）を作成して設定

---

## Step 6: 複数APIの分離設定

複数のサービス向けにAPIエンドポイントを分け、ドキュメントも分離する場合。

### ScrambleServiceProvider での設定

```php
<?php

namespace App\Providers;

use Dedoc\Scramble\Scramble;
use Illuminate\Routing\Route;
use Illuminate\Support\Str;
use Illuminate\Support\ServiceProvider;

class ScrambleServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        // 1. 外部連携API（現在の設定）
        Scramble::registerApi('link_intersystem', [
            'api_path' => 'link_intersystem_api',
            'info' => [
                'title' => 'システム連携API',
                'version' => '1.0.0',
            ],
        ])->routes(fn(Route $r) => Str::startsWith($r->uri, 'link_intersystem_api'));

        // 2. 管理画面用API
        Scramble::registerApi('admin', [
            'api_path' => 'api',
            'info' => [
                'title' => '管理画面API',
                'version' => '1.0.0',
            ],
        ])->routes(fn(Route $r) => Str::startsWith($r->uri, 'api/'));

        // 3. 顧客向けAPI
        Scramble::registerApi('customer', [
            'api_path' => 'customer',
            'info' => [
                'title' => '顧客向けAPI',
                'version' => '1.0.0',
            ],
        ])->routes(fn(Route $r) => Str::startsWith($r->uri, 'customer/'));
    }
}
```

### 生成されるドキュメント

| API | ドキュメントUI | エクスポート |
| :--- | :--- | :--- |
| link_intersystem | `/docs/link_intersystem` | `php artisan scramble:export --api=link_intersystem` |
| admin | `/docs/admin` | `php artisan scramble:export --api=admin` |
| customer | `/docs/customer` | `php artisan scramble:export --api=customer` |

### 出力ファイル

```bash
# 個別に出力
php artisan scramble:export --api=link_intersystem --path=storage/api-docs/link_intersystem.json
php artisan scramble:export --api=admin --path=storage/api-docs/admin.json
php artisan scramble:export --api=customer --path=storage/api-docs/customer.json
```

---

## 新規API作成時の手順

新しいAPIエンドポイントを追加する際は以下の手順で型が自動生成されます。

### Step 1: Controller作成

```php
<?php

namespace App\Http\Controllers\Api\LinkIntersystem;

use Illuminate\Http\JsonResponse;

class NewApiController extends Controller
{
    /**
     * サンプルAPIの説明
     *
     * @operationId getSample
     * @tags NewApi
     *
     * @response array{id: int, name: string}
     */
    public function show(int $id): JsonResponse
    {
        return response()->json([
            'id' => $id,
            'name' => 'Sample',
        ]);
    }
}
```

### Step 2: FormRequest作成（バリデーション付き）

```php
<?php

namespace App\Http\Requests\Api\LinkIntersystem;

use Illuminate\Foundation\Http\FormRequest;

class SampleRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'name' => ['required', 'string', 'max:255'],
            'email' => ['required', 'email'],
        ];
    }
}
```

### Step 3: ルート追加

```php
// routes/link_intersystem_api.php
Route::prefix("new-api")->controller(NewApiController::class)->group(function () {
    Route::get('/{id}', 'show');
});
```

### Step 4: スキーマ再生成

```bash
php artisan scramble:export
```

---

## トラブルシューティング

### DB接続エラー

**問題**: DBに接続できない環境でスキーマ生成を実行するとエラーになる

**解決策**:
- 実際のDBに接続できる環境で実行する
- または、SQLiteなどのテスト用DBを使用する
- CI/CD環境では、services でDBコンテナを起動する

### 不要なResourceクラス

**教訓**: Scrambleは既存のコードを自動解析するため、以下で十分です：
- ControllerのPHPDocアノテーション（`@response`）
- FormRequestの `rules()` メソッド
- メソッドの戻り値型

**不要なもの**:
- API Resourceクラスを新規作成する必要はない（既存のコードから自動解析される）

---

## まとめ

- **Scramble導入**: コードから自動でOpenAPIスキーマを生成
- **GitLab CI統合**: 自動でドキュメントを更新
- **複数API対応**: `registerApi()` でサービスごとに分離可能
- **新規API追加**: Controller + FormRequest + ルート追加 → 自動反映

