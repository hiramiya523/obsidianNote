# Laravel API → SvelteKit TypeScript型定義 自動生成ガイド

タグ: #laravel #sveltekit #openapi #typescript #api #automation #ci-cd

---

## 📋 目次

- [[#全体アーキテクチャ]]
- [[#Step 1: Laravel側でOpenAPIスキーマを自動生成]]
- [[#Step 2: OpenAPIからTypeScript型を自動生成]]
- [[#Step 3: SvelteKitでの型定義活用]]
- [[#Step 4: CI/CD統合]]
- [[#メリット・デメリット分析]]
- [[#将来展望]]
- [[#推奨実装ロードマップ]]
- [[#認識修正・補足]]

---

## 全体アーキテクチャ

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    【推奨アーキテクチャ】                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Laravel API                    OpenAPI Spec              SvelteKit     │
│  ┌─────────────┐               ┌──────────────┐         ┌───────────┐ │
│  │ Controller  │───自動生成───▶│ openapi.yaml  │──生成──▶│ types.ts  │ │
│  │ Request     │               │ (3.1)         │         │           │ │
│  │ Resource    │               └──────────────┘         │ api.ts    │ │
│  └─────────────┘                      │                 │ (client)  │ │
│         │                             │                 └───────────┘ │
│         ▼                             ▼                       │        │
│  ┌─────────────┐               ┌──────────────┐               ▼        │
│  │ Scramble    │               │ openapi-ts   │         ┌───────────┐ │
│  │ (dedoc)     │               │ v7.x         │         │ SvelteKit │ │
│  └─────────────┘               └──────────────┘         │ Component │ │
│                                                          └───────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Step 1: Laravel側でOpenAPIスキーマを自動生成

### 選択肢比較（2025年12月現在）

| ツール | Laravel 11対応 | 特徴 | 推奨度 |
| :--- | :--- | :--- | :--- |
| **Scramble (dedoc)** | ✅ | コードベース自動解析、アノテーション不要 | ⭐⭐⭐⭐⭐ |
| **l5-swagger** | ✅ | PHPDocアノテーション必須 | ⭐⭐⭐ |
| **手動管理** | - | 既存yaml利用 | ⭐⭐ |

### 推奨: Scramble（dedoc/scramble）の導入

#### インストール

```bash
# Laravelプロジェクトで実行
composer require dedoc/scramble
```

#### 設定ファイル作成

`config/scramble.php`:

```php
<?php

return [
    'info' => [
        'version' => '1.0.0',
        'title' => 'プロジェクト API',
    ],
    // APIルートのプレフィックス
    'api_path' => 'api',
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
            'url' => env('APP_URL') . '/api',
            'description' => 'API Server',
        ],
    ],
];
```

#### Controller例（アノテーション不要で自動解析）

```php
<?php

namespace App\Http\Controllers;

use App\Http\Requests\Users\StoreRequest;
use App\Http\Resources\UserResource;
use App\Models\User;
use Illuminate\Http\Resources\Json\AnonymousResourceCollection;

class UserController extends Controller
{
    /**
     * ユーザー一覧を取得
     * 
     * @return AnonymousResourceCollection<UserResource>
     */
    public function list(): AnonymousResourceCollection
    {
        return UserResource::collection(User::paginate());
    }

    /**
     * ユーザー詳細を取得
     */
    public function detail(int $id): UserResource
    {
        return new UserResource(User::findOrFail($id));
    }

    /**
     * ユーザーを作成
     */
    public function store(StoreRequest $request): UserResource
    {
        $user = User::create($request->validated());
        return new UserResource($user);
    }
}
```

#### FormRequest例

```php
<?php

namespace App\Http\Requests\Users;

use Illuminate\Foundation\Http\FormRequest;

class StoreRequest extends FormRequest
{
    /**
     * @return array<string, mixed>
     */
    public function rules(): array
    {
        return [
            'name' => ['required', 'string', 'max:255'],
            'email' => ['required', 'email', 'unique:users,email'],
            'password' => ['required', 'string', 'min:8'],
        ];
    }
}
```

#### API Resource例

```php
<?php

namespace App\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

/**
 * @property int $id
 * @property string $name
 * @property string $email
 * @property \Carbon\Carbon $created_at
 */
class UserResource extends JsonResource
{
    /**
     * @return array<string, mixed>
     */
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'email' => $this->email,
            'created_at' => $this->created_at->toISOString(),
        ];
    }
}
```

#### スキーマ生成コマンド

```bash
# JSONで出力
php artisan scramble:export --path=../docs/api/openapi.json

# または、ブラウザでリアルタイム確認
# http://localhost/docs/api
```

---

## Step 2: OpenAPIからTypeScript型を自動生成

### 推奨: openapi-typescript v7.x + openapi-fetch

#### インストール

```bash
# SvelteKitプロジェクトで実行
pnpm add -D openapi-typescript
pnpm add openapi-fetch
```

#### package.json設定

```json
{
  "scripts": {
    "generate:types": "openapi-typescript ../docs/api/openapi.yaml -o src/lib/types/generated/api.d.ts",
    "generate:types:watch": "openapi-typescript ../docs/api/openapi.yaml -o src/lib/types/generated/api.d.ts --watch",
    "predev": "pnpm run generate:types",
    "prebuild": "pnpm run generate:types"
  }
}
```

#### openapi-ts.config.ts（高度な設定）

```typescript
// openapi-ts.config.ts
import { defineConfig } from '@hey-api/openapi-ts';

export default defineConfig({
  client: '@hey-api/client-fetch',
  input: '../docs/api/openapi.yaml',
  output: {
    path: 'src/lib/api/generated',
    format: 'prettier',
    lint: 'eslint',
  },
  plugins: [
    '@hey-api/typescript',
    '@hey-api/sdk',
    {
      name: '@hey-api/transformers',
      dates: true,
    },
  ],
});
```

---

## Step 3: SvelteKitでの型定義活用

### APIクライアント設定

`src/lib/api/client.ts`:

```typescript
import createClient from 'openapi-fetch';
import type { paths } from '$lib/types/generated/api';

// 型安全なAPIクライアント
export const api = createClient<paths>({
  baseUrl: import.meta.env.VITE_API_BASE_URL || 'http://localhost/api',
  headers: {
    'Content-Type': 'application/json',
    'Accept': 'application/json',
  },
});

// 認証付きクライアント（BFFパターン用）
export const createAuthenticatedClient = (token: string) => {
  return createClient<paths>({
    baseUrl: import.meta.env.VITE_API_BASE_URL,
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json',
    },
  });
};
```

### サーバーサイド（+page.server.ts）での使用

`src/routes/users/+page.server.ts`:

```typescript
import type { PageServerLoad } from './$types';
import { api } from '$lib/api/client';
import { error } from '@sveltejs/kit';

export const load: PageServerLoad = async ({ cookies }) => {
  const token = cookies.get('auth_token');
  const { data, error: apiError } = await api.GET('/users', {
    headers: token ? { Authorization: `Bearer ${token}` } : {},
  });

  if (apiError) {
    throw error(500, 'ユーザー一覧の取得に失敗しました');
  }

  return {
    users: data,
  };
};
```

### コンポーネントでの使用（Svelte 5 Runes）

`src/routes/users/+page.svelte`:

```svelte
<script lang="ts">
  import type { PageData } from './$types';
  import type { components } from '$lib/types/generated/api';

  // 型をエイリアスとして定義
  type User = components['schemas']['UserResource'];

  let { data } = $props<{ data: PageData }>();

  // 型安全なユーザーリスト
  const users: User[] = $derived(data.users ?? []);
</script>

<ul>
  {#each users as user (user.id)}
    <li>{user.name} - {user.email}</li>
  {/each}
</ul>
```

---

## Step 4: CI/CD統合

### GitHub Actions例

`.github/workflows/generate-types.yml`:

```yaml
name: Generate API Types

on:
  push:
    paths:
      - 'docs/api/openapi.yaml'
      - 'controller/app/Http/**'
  workflow_dispatch:

jobs:
  generate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.3'
      
      - name: Install Composer dependencies
        working-directory: ./controller
        run: composer install --no-dev
      
      - name: Generate OpenAPI spec
        working-directory: ./controller
        run: php artisan scramble:export --path=../docs/api/openapi.yaml
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
      
      - name: Setup pnpm
        uses: pnpm/action-setup@v2
        with:
          version: 10
      
      - name: Generate TypeScript types
        working-directory: ./front-sveltekit
        run: |
          pnpm install
          pnpm run generate:types
      
      - name: Commit changes
        uses: stefanzweifel/git-auto-commit-action@v5
        with:
          commit_message: 'chore: auto-generate API types'
          file_pattern: 'docs/api/*.yaml front-sveltekit/src/lib/types/generated/*'
```

---

## メリット・デメリット分析

### 慎重派の経営者視点

| 観点 | メリット | デメリット |
| :--- | :--- | :--- |
| **コスト** | 長期的な保守コスト削減（型不整合バグ-70%見込み） | 初期導入工数 2-3人日 |
| **リスク** | 型安全性によるランタイムエラー削減 | ツール依存による学習曲線 |
| **品質** | APIドキュメントの自動整備 | 既存コードへのアノテーション追加が必要な場合あり |
| **判断** | 段階的導入を推奨 | 新規エンドポイントから適用開始 |

### 躍進的な経営者視点

| 観点 | メリット | デメリット |
| :--- | :--- | :--- |
| **スピード** | 開発速度30%向上（型補完・エラー早期検出） | 初期セットアップ時間 |
| **採用** | モダン技術スタック（エンジニア採用優位性） | チーム全体の学習が必要 |
| **拡張性** | マイクロサービス化への布石 | 過度な自動化への依存リスク |
| **判断** | 即時全面導入を推奨 | 競合他社に対する技術優位性確保 |

---

## 将来展望

### 3年後（2028年）

- Contract-First Development の標準化
- OpenAPI 4.0 への移行
- AI支援によるスキーマ提案・最適化

### 5年後（2030年）

- API自動テスト生成（スキーマからE2Eテスト自動生成）
- GraphQL Federation との統合オプション
- リアルタイム型同期（ホットリロード）

### 10年後（2035年）

- AI駆動API設計（自然言語からAPI定義生成）
- 自己修復型スキーマ（破壊的変更の自動検出・マイグレーション）
- ゼロコード統合プラットフォーム

---

## 推奨実装ロードマップ

### Phase 1 (2週間)

- Scramble導入
- 既存openapi.yamlとの整合性確認
- 型生成パイプライン構築

### Phase 2 (1ヶ月)

- openapi-fetch統合
- BFFパターンでの型活用
- CI/CD自動化

### Phase 3 (継続)

- Zodによる実行時バリデーション追加
- APIモック自動生成（MSW統合）
- E2Eテストとの連携

---

## 認識修正・補足

1. **既存のopenapi.yaml**: `/workspace/docs/api/openapi.yaml` が存在しますが、OpenAPI 3.0.2形式です。3.1.0への更新を推奨します（JSON Schemaとの完全互換性のため）

2. **l5-swagger vs Scramble**: l5-swaggerはアノテーション必須で保守負担が高いため、Scramble（コード自動解析）を推奨します

3. **openapi-typescript vs openapi-generator-cli**:
   - **openapi-typescript**: 軽量、型定義のみ（推奨）
   - **openapi-generator-cli**: フルクライアント生成、重い

