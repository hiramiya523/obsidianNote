# SvelteKit Form Actions: 複数フォームと通信最適化のFAQ

タグ: #sveltekit #form-actions #performance #best-practices #optimization #use-enhance

---

## 📋 目次

- [[#Q1. 一画面に複数の `<form>` を置いても大丈夫？]]
- [[#Q2. 一覧画面で100個の削除フォームができても平気？]]
- [[#Q3. Form Action後の「無駄なデータ再取得」を防ぐには？]]
- [[#Q4. `use:enhance` って付けないとまずいの？]]
- [[#4. まとめと実装指針]]

---

## Q1. 一画面に複数の `<form>` を置いても大丈夫？

**A. 全く問題ありません。むしろ推奨される設計パターンです。**

これまでのSPA開発（JSONを1つにまとめて送信）とは異なり、SvelteKitでは「意図（Intent）」ごとにフォームを分けるのが自然です。

### 複数のフォームに分けるメリット

1.  **意図の分離**: 「プロフィールの更新」「パスワード変更」「アカウント削除」など、目的が明確になります。
2.  **バックエンドのシンプル化**: Laravel側のControllerもメソッド単位で分離しやすくなります。
3.  **バリデーション**: フォームごとのエラー状態 (`form` オブジェクト) が混ざらず、管理が楽になります。

### 実装例：設定画面（Named Actions）

URLを変えずに複数の処理をさばくには、**名前付きアクション (Named Actions)** を使います。

**Server (`+page.server.ts`)**
```typescript
export const actions = {
    updateProfile: async ({ request }) => { /* ... */ },
    changePassword: async ({ request }) => { /* ... */ },
    deleteAccount: async ({ request }) => { /* ... */ }
};
```

**Client (`+page.svelte`)**
```svelte
<script>
    import { enhance } from '$app/forms';
</script>

<!-- action="?/名前" で呼び分ける -->
<section>
    <h2>プロフィール</h2>
    <form method="POST" action="?/updateProfile" use:enhance>
        <input name="name" />
        <button>更新</button>
    </form>
</section>

<section>
    <h2>パスワード</h2>
    <form method="POST" action="?/changePassword" use:enhance>
        <!-- ... -->
        <button>変更</button>
    </form>
</section>
```

---

## Q2. 一覧画面で100個の削除フォームができても平気？

**A. はい、現代のブラウザでは全く負荷になりません。**

DOM上に100個程度の `<form>` 要素があることは正常です。

### パターンA：個別にフォームを置く（標準的・推奨）

最も直感的で実装しやすいパターンです。

```svelte
{#each data.todos as todo}
    <div class="row">
        <span>{todo.title}</span>
        <!-- 各行に独立したフォーム -->
        <form method="POST" action="?/delete" use:enhance>
            <input type="hidden" name="id" value={todo.id} />
            <button aria-label="削除">🗑️</button>
        </form>
    </div>
{/each}
```

### パターンB：`formaction` を使う（最適化）

`<table>` タグの中など、HTML構造的にフォームを大量に置くのが厳しい場合のテクニックです。

```svelte
<!-- フォームは1つだけ -->
<form method="POST" use:enhance>
    {#each data.todos as todo}
        <!-- ボタン自体に送信先のアクションと値を持たせる -->
        <button formaction="?/delete&id={todo.id}">
            削除
        </button>
    {/each}
</form>
```

> **⚠️ 注意:** HTMLのルールとして、**`<form>` の入れ子（ネスト）は禁止**されています。必ず並列（兄弟要素）に配置するか、HTML5の `form` 属性を使ってください。

---

## Q3. Form Action後の「無駄なデータ再取得」を防ぐには？

**A. デフォルトでは全データが再取得されますが、`depends` と `invalidate` で最適化できます。**

デフォルトの挙動（Action成功 → 全 `load` 関数再実行）は、**「データの整合性」を最優先**しているためです。しかし、サイドバーやヘッダーなど無関係なデータまで再取得してしまうのがボトルネックになる場合、以下の方法で制御します。

### 解決策A：依存関係の定義（推奨）

「このデータに関連するアクションが起きた時だけ再取得する」と明示します。

**Server (`+page.server.ts`)**
```typescript
import { type PageServerLoad } from './$types';

export const load: PageServerLoad = async ({ fetch, depends }) => {
    // 1. このデータ取得に依存キーを付ける
    depends('app:todos'); 
    
    const res = await fetch('http://laravel-api/todos');
    return { todos: await res.json() };
};
```

**Client (`+page.svelte`)**
```svelte
<script lang="ts">
    import { enhance } from '$app/forms';
    import { invalidate } from '$app/navigation';
</script>

<form 
    method="POST" 
    action="?/create" 
    use:enhance={() => {
        return async ({ result }) => {
            if (result.type === 'success') {
                // 2. 成功時、全更新ではなく特定キーのみ更新する
                await invalidate('app:todos'); 
            }
        };
    }}
>
    <!-- ... -->
</form>
```

### 解決策B：楽観的UI更新（通信ゼロ）

結果が自明な操作（削除やいいね）の場合、再取得自体を行わない選択肢もあります。

```svelte
<form 
    method="POST" 
    action="?/delete" 
    use:enhance={() => {
        // 送信直後に画面から即座に消す（楽観的更新）
        data.todos = data.todos.filter(t => t.id !== todo.id);

        return async ({ update }) => {
            // サーバー処理完了後、何もリロードしない
            // update() を呼ばなければ load は再実行されない
        };
    }}
>
```

---

## Q4. `use:enhance` って付けないとまずいの？

**A. 機能はしますが、SPA（シングルページアプリケーション）としての体験が損なわれます。**

「JS無効環境用」というのは誤解で、`use:enhance` は **「リロードなしで画面を書き換えるための必須機能」** です。

### 付けない場合（全ページリロード）の3大デメリット

1.  **アプリの状態（State）が全て消える（致命的）**
    -   入力中のモーダルが閉じる、スクロール位置がトップに戻る、`$state` 変数がリセットされるなど、UXが崩壊します。
2.  **レイアウトの再描画**
    -   サイドバーやヘッダー、再生中の動画プレーヤーなどが一瞬消えて再生成されます。
3.  **体感速度の低下**
    -   画面が白くなる（ホワイトフラッシュ）、JSの再読み込みとハイドレーションが走るため、動作が重く感じられます。

### 比較まとめ

| 項目 | `use:enhance` あり (SPA動作) | `use:enhance` なし (MPA動作) |
| :--- | :--- | :--- |
| **画面遷移** | **なし**（差分更新のみ） | **あり**（ブラウザの再読み込み） |
| **コンポーネントの状態** | **維持される** | **すべて破棄・初期化される** |
| **スクロール位置** | **維持される** (制御可能) | ガタつく / リセットされる |
| **通信量** | 必要なJSONデータのみ | HTML全体 + アセット再検証 |
| **体験** | ネイティブアプリに近い | 昔ながらのWebサイト |

### 結論：いつ外すべき？

**「ファイルのダウンロード」を行う場合のみ外してください。**
それ以外の全てのデータ更新・保存処理においては、**`use:enhance` は必須**と考えてください。

---

## 4. まとめと実装指針

1.  **複数フォームは正義:** 画面上のアクションごとに、遠慮なく小さな `<form>` を作ってください。
2.  **まずはデフォルトで作る:** 全データ再取得のオーバーヘッドは、多くの場合許容範囲内です。「整合性」を優先し、バグを防ぎましょう。
3.  **後から最適化:** パフォーマンスが気になった箇所だけ、`depends` / `invalidate` を追加して通信を絞りましょう。SvelteKitはこの「段階的な最適化」に適しています。
4.  **`use:enhance` は常時ON:** ファイルダウンロード以外は必ず付けましょう。
