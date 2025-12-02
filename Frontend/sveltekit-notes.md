---
title: SvelteKit メモ
created: 2025-11-28
tags:
  - sveltekit
  - svelte
  - frontend
  - typescript
aliases:
  - Svelte
  - フロントエンド
---

# SvelteKit メモ

> [!abstract] 概要
> SvelteKit/Svelte 5のリアクティビティ、SSR、コンポーネント設計に関するメモ。

---

## SSR（Server-Side Rendering）

### SSRの「レンダリング」はどこで行われる？

> [!info] 結論
> SSRの「レンダリング」は **サーバー側** で行われる。
> 
> ブラウザ側では:
> 1. サーバーから受け取った **完成済みHTML** を描画
> 2. その後 **ハイドレーション**（クライアントJSがくっついてインタラクティブになる）

---

## 本番コンテナの構築

### Adapter設定

> [!warning] 重要
> SvelteKitの初期設定では `adapter-auto` が使われているが、
> Dockerで明示的にNodeサーバーとして動かすには **`@sveltejs/adapter-node`** への変更が必須。

```bash
npm install @sveltejs/adapter-node
```

```js
// svelte.config.js
import adapter from '@sveltejs/adapter-node';

export default {
  kit: {
    adapter: adapter()
  }
};
```

> [!tip] SSGの場合
> 完全SSG（Static Site Generation）なら、Nodeサーバーは不要。
> `adapter-static` を使用。

---

## パフォーマンス

### コンポーネント分割のオーバーヘッド

> [!success] 結論
> DOMノード数が **10万単位** になるような極端なケースで、
> かつ計測してコンポーネント境界のオーバーヘッドが効いていることが分かっている場合のみ問題。
> 
> **通常の業務システム規模であれば、共通化を避ける理由はほぼない。**

---

## ページ遷移のローディング表示

### クライアントサイドナビゲーション

`$app/stores` の `navigating` を使用:

```svelte
<script>
  import { navigating } from "$app/state";
  const isNavigating = $derived(Boolean(navigating.to));
</script>

{#if isNavigating}
  <div>Loading...</div>
{/if}
```

> [!warning] 制限
> - クライアント画面上での **ページ遷移なら表示可能**
> - **リロードでは表示されない**

> [!tip] 本格的な対応
> 「SSRで骨組みだけ返す + クライアントで部分的に遅延ロード（クライアントfetch）」
> という設計への切り替えを検討。

---

## Svelte 5 リアクティビティ

### $derived

```svelte
<script>
  // 式の場合
  let doubled = $derived(count * 2);
  
  // 関数の場合
  let complex = $derived.by(() => {
    // 複雑な計算
    return result;
  });
</script>
```

> [!info] $derivedの性質
> - ✅ `$derived` で定義する変数は、**const で宣言する読み取り専用のリアクティブ値**
> - ✅ 値そのものは、依存する `$state` が変わるたびに **再計算されて変わる**
> - ❌ `const` だから一度計算した値が完全に固定されて変わらない → **これは違う**
> 
> 参照先は固定だが、中の「見える値」は依存stateに応じて変動する。

---

### $bindable - 親子間のバインド

#### 子コンポーネント

```svelte
<script lang="ts">
  interface Props {
    items: DateStock[];
    borderColor?: string;
  }
  
  // 親と子の値をバインドしたいなら、$bindableで受け取ってDOMにbindするだけ
  let { items = $bindable<DateStock[]>(), borderColor = "border-slate-500" }: Props = $props();
</script>
```

#### 親コンポーネント

```svelte
<!-- bind:あり - 双方向バインド -->
<StockReserveRow bind:items={row.r_common_date_stocks} borderColor="border-blue-500" />

<!-- bind:なし - 一方通行（子の変更は親に反映されない...と思いきや） -->
<StockReserveRow items={agent.r_date_stocks} borderColor="border-red-600" />
```

### $bindableと$stateの関係

> [!question] 子で`$bindable`で受け取る値は、親で`$state`で定義されている必要がある？

**回答: 必須ではないが、実質必要な場合が多い**

| 条件 | 挙動 |
|------|------|
| 親が `$state` + `bind:` あり | 完全な双方向バインド |
| 親が `$state` + `bind:` なし | **中身の変更は共有される**（Proxyを共有） |
| 親が通常の値 + `bind:` なし | 一方通行、子の変更は親に反映されない |

> [!info] なぜbind:なしでも動くように見えるのか
> `$state` は **deep Proxy** を親子で共有している。
> 
> - 子で `items[i].xxx` を変えると、親のstateも変わり、UIも更新される
> - **配列自体を差し替える**（`items = [...]`）場合は `bind:` の有無で挙動が変わる

#### 結論

> [!tip] まとめ
> - 「`$state` のおかげで **たまたま bind なしでも動いている**」
> - 「`bind:` は **別のレイヤの機構**（代入・所有関係の宣言）」

---

## 関連ドキュメント

- [[svelte/QA|SvelteKit Q&A]] - Sanctum認証・ユニバーサルfetch実装
- [[Sveltekit導入計画|TOMONI移行計画]] - SvelteKit移行プロジェクト
- [[../Index|ホーム]] - 目次に戻る

---

*作成日: 2025年11月*


