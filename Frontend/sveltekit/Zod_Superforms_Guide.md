# SvelteKit Validation: Zod と Superforms 導入ガイド

タグ: #sveltekit #validation #zod #superforms #svelte-5 #best-practices

---

## 📋 目次

- [[#結論：採用は必須 (Strongly Recommended)]]
- [[#1. Zod採用の判断基準]]
- [[#2. 経営者視点でのメリット・デメリット]]
- [[#3. sveltekit-superforms とは？]]
- [[#4. 実装ステップ (Svelte 5 + Zod + Superforms)]]
- [[#5. コンポーネント化のベストプラクティス (Svelte 5)]]
- [[#6. Laravel × SvelteKit 特有の課題と解決策]]
- [[#7. コーディング規約との適合性]]
- [[#8. 3年・5年・10年先を見据えた展望]]

---

## 結論：採用は必須 (Strongly Recommended)

SvelteKit + Svelte 5 + Laravel 11 の構成において、**Zod（+ sveltekit-superforms）の採用は、開発体験と品質を最大化するための最良の選択**です。

### 誤解を解消：Laravelのバリデーションは不要になる？

**いいえ、なりません。**

- **Laravel (API)**: セキュリティとデータ整合性のための「最終防衛ライン」。絶対に省略できません。
- **Zod (Client/BFF)**: UX向上（即時エラー表示）と型安全（TypeScript統合）のために存在します。

---

## 1. Zod採用の判断基準

| 観点 | 評価 | 理由 |
| :--- | :--- | :--- |
| **学習コスト** | **極めて低い** | メソッドチェーンで直感的に書けるため、Laravelのバリデーションに慣れていれば違和感なし。 |
| **運用コスト** | **導入しない方が高い** | Zodがない場合、型定義を手動で書く必要がある。Zodなら `z.infer` で型を自動生成でき、保守性が劇的に向上する。 |
| **将来性** | **デファクトスタンダード** | TypeScript環境での標準。将来的な「Standard Schema」仕様にも対応予定。 |

---

## 2. 経営者視点でのメリット・デメリット

| 論点 | 🛡️ 慎重派 (リスク・コスト重視) | 🚀 躍進派 (速度・品質重視) |
| :--- | :--- | :--- |
| **二重管理** | 「PHPとJSで同じルールを書くのは無駄では？」 | 「手戻りコストの方が高い。フロントエンドで即時エラーが出せるUXは顧客満足度に直結する。」 |
| **バンドルサイズ** | 「ページ読み込みが遅くなるのでは？」 | 「十数KB程度であり誤差。それより開発速度を優先したい。」 |
| **採用・教育** | 「学習コストを掛けたくない。」 | 「Zodは業界標準。これを使える環境を用意するのが採用ブランディングになる。」 |
| **結論** | **条件付き採用** (自動生成等の仕組みがあれば) | **即採用** (開発速度と品質向上だけで元が取れる) |

---

## 3. sveltekit-superforms とは？

SvelteKit のサーバーサイド (Actions) とクライアントサイド (Form) を、**Zod スキーマ一本で完全同期させるためのミドルウェア**です。

### なぜ Superforms が必要なのか？ (Zod単体との比較)

Zod単体でもバリデーションは可能ですが、SvelteKitと統合するには多くのボイラープレート（定型コード）が必要です。Superformsはそれを自動化します。

| 比較項目 | Zod 単体 (素のSvelteKit) | Zod + Superforms (推奨) |
| :--- | :--- | :--- |
| **バリデーション実行** | `request.formData()` を手動パースし `schema.parse()` | `superValidate(request, zod(schema))` 1行で完了 |
| **型安全性** | フォームデータの型定義を手動で書く必要がある | **完全自動**。クライアント側の `$form` がZodスキーマ通りの型になる。 |
| **エラー処理** | `catch` で捕捉し、`fail(400, { errors })` を手動で返す | **自動**。エラーがあれば自動で捕捉し、フロントエンドの `$errors` ストアにマッピングされる。 |
| **データの書き戻し** | エラー時、手動で `value={form?.name}` と書く必要がある | **自動**。`bind:value={$form.name}` するだけで、エラー時も入力値が維持される。 |
| **クライアント検証** | Submit前のJSチェックロジックを別途書く必要がある | **設定ONにするだけ**。同じZodスキーマを使ってブラウザ上でも即時チェックが走る。 |

### 動作イメージ（ライフサイクル）

1. **Server Load**: `superValidate(zod(schema))` で「空のフォームオブジェクト（初期値と型情報）」を生成し、フロントへ渡す。
2. **Client Render**: フロントは受け取ったオブジェクトを元にストア (`$form`, `$errors`) を初期化。
3. **Submit**:
   - **Client Validation**: ブラウザ上の JS で Zod が走り、エラーなら送信中止（設定可）。
4. **Server Action**:
   - `superValidate(request, zod(schema))` が再び走り、データの検証を行う。
   - エラーなら、「エラー情報 + 入力データ」を自動でクライアントに送り返す (`fail`)。
5. **Response**:
   - フロントはレスポンスを検知し、`$errors` に赤文字を表示し、`$form` に入力値を戻す。
   - これら全てが **ページリロードなし（SPA動作）** で行われる。

---

## 4. 実装ステップ (Svelte 5 + Zod + Superforms)

### Step 1: インストール

```bash
npm install zod sveltekit-superforms
```

### Step 2: スキーマ定義 (`src/lib/schemas/contact.ts`)

```typescript
import { z } from 'zod';

export const contactSchema = z.object({
  name: z.string().min(1, { message: '名前を入力してください' }).max(50),
  email: z.string().email({ message: '正しいメールアドレス形式ではありません' }),
  message: z.string().min(10, { message: 'お問い合わせ内容は10文字以上で入力してください' }),
});

// 型定義の自動生成 (interfaceではなくtypeになる)
export type ContactSchema = z.infer<typeof contactSchema>;
```

### Step 3: Server Side Logic (`src/routes/contact/+page.server.ts`)

```typescript
import { superValidate, message } from 'sveltekit-superforms';
import { zod } from 'sveltekit-superforms/adapters';
import { contactSchema } from '$lib/schemas/contact';
import { fail } from '@sveltejs/kit';
import type { Actions, PageServerLoad } from './$types';

export const load: PageServerLoad = async () => {
  // 初期フォームデータの生成
  const form = await superValidate(zod(contactSchema));
  return { form };
};

export const actions: Actions = {
  default: async ({ request }) => {
    const form = await superValidate(request, zod(contactSchema));

    if (!form.valid) {
      // フロント検証を突破された場合のサーバーサイドチェック
      return fail(400, { form });
    }

    try {
      // ★ ここで Laravel API をコール
      // const response = await fetch('http://laravel-api/api/contact', ...);
      
      // Laravel側で422エラー（バリデーションエラー）が返ってきた場合の処理例
      // if (response.status === 422) {
      //    const errors = await response.json();
      //    // LaravelのエラーをZodのフォームエラーにマッピングする処理が必要
      //    // return setError(form, 'email', 'このメールアドレスは既に登録されています');
      // }

      return message(form, '送信しました');
    } catch (error) {
      return message(form, 'エラーが発生しました', { status: 500 });
    }
  }
};
```

### Step 4: Client Side (`src/routes/contact/+page.svelte`)

```svelte
<script lang="ts">
  import { superForm } from 'sveltekit-superforms';
  import { zodClient } from 'sveltekit-superforms/adapters';
  import { contactSchema } from '$lib/schemas/contact';

  // Svelte 5: $props() でデータを受け取る
  let { data } = $props();

  // Client-side validationを有効にするため validators に zodClient を渡す
  const { form, errors, enhance, message, constraints } = superForm(data.form, {
    validators: zodClient(contactSchema),
    resetForm: true,
    // クライアントサイドでのSPA動作維持
    applyAction: true, 
    invalidateAll: true
  });
</script>

<h1>お問い合わせ</h1>

{#if $message}<div class="alert">{$message}</div>{/if}

<form method="POST" use:enhance>
  <div>
    <label for="name">名前</label>
    <!-- Svelte 5: 双方向バインディングは bind:value -->
    <input 
      type="text" 
      name="name" 
      id="name"
      bind:value={$form.name} 
      aria-invalid={$errors.name ? 'true' : undefined}
      {...$constraints.name}
    />
    {#if $errors.name}<span class="error">{$errors.name}</span>{/if}
  </div>

  <div>
    <label for="email">メールアドレス</label>
    <input 
      type="email" 
      name="email" 
      id="email"
      bind:value={$form.email}
      aria-invalid={$errors.email ? 'true' : undefined}
      {...$constraints.email}
    />
    {#if $errors.email}<span class="error">{$errors.email}</span>{/if}
  </div>

  <button>送信</button>
</form>

<style>
  .error { color: red; font-size: 0.8em; }
</style>
```

---

## 5. コンポーネント化のベストプラクティス (Svelte 5)

フォーム部品をコンポーネント化する場合（例: `TextInput.svelte`）、Superforms の型をどう渡すかが課題になります。Generics を使うのが Svelte 5 + Superforms のベストプラクティスです。

`src/lib/components/TextInput.svelte`:

```svelte
<script lang="ts" generics="T extends Record<string, unknown>">
  import type { FormPath, SuperForm } from 'sveltekit-superforms';

  // 親から superForm のオブジェクトごと受け取る
  let { form, field, label } = $props<{ 
    form: SuperForm<T>, 
    field: FormPath<T>, 
    label: string 
  }>();
</script>

<label>
  {label}
  <input 
    type="text" 
    bind:value={$form.form[field]} 
    aria-invalid={$form.errors[field] ? 'true' : undefined} 
  />
  {#if $form.errors[field]}
    <span class="error">{$form.errors[field]}</span>
  {/if}
</label>
```

使用側 (`+page.svelte`):

```svelte
<TextInput form={superFormObject} field="name" label="名前" />
```

---

## 6. Laravel × SvelteKit 特有の課題と解決策

LaravelとSvelteKitを分離運用する場合の最大の課題は**「型定義とバリデーションルールの二重管理」**です。

### 推奨: OpenAPI (Swagger) を真実のソースにする

Laravel側で `scribe` や `l5-swagger` を用いてOpenAPI仕様書を出力し、フロントエンドで `openapi-typescript` 等を使って型定義を自動生成します。
これにより、Laravel側の変更を検知してSvelteKit側のビルドを落とすことが可能になります。

### 上級者向け: Laravel-to-Zod

「慎重派経営者」が二重管理を強く懸念する場合、LaravelのFormRequestからZodスキーマに近いものを出力するツールの利用を検討してください。ただし、完全な同期は難易度が高いため、まずは「API型定義の同期」を優先すべきです。

---

## 7. コーディング規約との適合性

ご提示いただいたコーディング規約 (Draft v1.0) との親和性は**完璧**です。

| 規約項目 | Superforms / Zod の適合性 |
| :--- | :--- |
| **Type Alias の優先** | **完全適合**: `z.infer` は `type` を返します。`interface` 禁止ルールと合致します。 |
| **BFFパターンの徹底** | **完全適合**: `load` で初期化し `actions` で受け取る設計を強制します。 |
| **Form Actions 第一選択** | **完全適合**: Form Actions のラッパーとして機能します。 |
| **use:enhance 必須** | **完全適合**: `superForm` は `use:enhance` を拡張して利用します。 |
| **Any 禁止** | **完全適合**: Zodスキーマにより型が自動推論され、`any` を排除できます。 |

---

## 8. 3年・5年・10年先を見据えた展望

- **3年後 (2028年)**: Zodは依然デファクト。`Valibot` 等の軽量ライブラリへの移行議論があるかもしれないが、Superformsのアダプターで対応可能。
- **5年後 (2030年)**: 「Standard Schema」の普及により、どのバリデータを使っても同じように動く世界に。今Zodを選んでおけば損はない。
- **10年後 (2035年)**: AIによるスキーマ自動生成や、Wasmによるバリデーションロジックの共通化が進む可能性があるが、現状ではZodが最も安全な投資。
