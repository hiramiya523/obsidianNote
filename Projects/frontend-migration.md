---
title: フロントエンド移行計画
created: 2025-11-28
tags:
  - project
  - sveltekit
  - typescript
  - frontend
  - migration
aliases:
  - SvelteKit移行
  - フロントエンド移行
---

# TOMONIシステム フロントエンド移行計画

> [!abstract] 概要
> TOMONIシステムのフロント画面をSvelteKitへ移行する計画。

---

## 課題

中・大規模アプリ開発における **Alpine.js、素のJS** での開発が年々厳しくなっている。

### 主な問題点

> [!bug] 問題
> **処理の共通化が難しく、処理が分散しがち**
> → 同じ処理を何度も書かなければいけない場面がある

### 影響

- 影響調査の漏れ
- 調査時間の増加
- 改修コストの増加
- メンテナンスコストの増加

### これまでの対策

- Web Componentの活用
- HandlebarsによるHTML分割

> [!warning] 限界
> これらの対策でも問題を解消するには至らない。
> 今後さらに複雑な業務ロジック・画面増加が予想される。

---

## 解決案: Svelte、Sveltekitを使います

### なぜSvelteか

#### 歴史

| フレームワーク      | 登場時期     |     |
| ------------ | -------- | --- |
| Svelte v1    | 2016年11月 |     |
| SvelteKit    | 2020年10月 |     |
| Alpine.js v1 | 2019年    |     |

#### State of JavaScript 2024 評価

> [!success] 高評価
> - Svelte: 86%
> - SvelteKit: 89%

> [!info] 最新情報の確認推奨
> State of JSは毎年更新されるため、最新版を確認することを推奨:
> - [State of JS 2024](https://2024.stateofjs.com/en-US/libraries/)

#### 実務での採用実績

> [!info] 採用事例
> 仕事でも使われている統計あり。以下のような実績がある:
> - Eコマースサイト: [Arial Shop](https://arialshop.com)
> - コンテンツ豊富なサイト: [Beatbump](https://beatbump.io/home)（YouTube Musicの代替フロントエンド）
> - [日本企業の採用事例まとめ](https://github.com/svelte-jp/japanese-svelte-companies)（随時更新）

---

## Alpine.jsの制限とSvelteでの解決

### Alpine.jsの独特な記述制限

> [!bug] 問題1: template直下は1つのタグのみ
> Alpine.jsでは、`x-for`ディレクティブを使用する際、`template`直下は1つのタグである必要がある。

```html
<!-- ✅ 正しい -->
<template x-for="user in users">
  <div></div>
</template>

<!-- ❌ エラー: 複数のタグは不可 -->
<template x-for="user in users">
  <div></div>
  <span></span>
</template>
```

> [!bug] 問題2: ネストしたループでDOM順序が逆転
> ネストした`x-for`を使用すると、**描画されるDOMの順番が逆転する**ケースがある。

```html
<!-- Alpine.js: 順序が保証されない場合がある -->
<template x-for="group in groups">
  <template x-for="item in group.items">
    <div x-text="item.name"></div>
  </template>
</template>
```

### Svelteでの解決

```svelte
<!-- Svelte: 自然な記述で問題なし -->
{#each groups as group}
  {#each group.items as item}
    <div>{item.name}</div>
  {/each}
{/each}
```

> [!success] メリット
> - 複数のタグを自由に配置可能
> - DOM順序が保証される
> - より自然な記述が可能

---

## 現状の問題（コード例）

> [!example] 繰り返しの多いコード
> 以下のような同じ構造のHTMLを十数回書いている:

```html
<div class="h-auto w-full border-t-[2.23px] border-[#666666]">
    <div
        class="flex items-center border-b-[2.23px] border-[#666666] bg-[#F2F2F2] py-2 pl-4"
        x-text="MULTI_LANGUAGE_TEXTS.ItineraryNumber[LANG_CODE]"></div>
    <div class="col-span-2 flex items-center break-all bg-white py-3 pl-4" 
         x-text="`${o_reserve.itinerary_id}`"></div>
</div>
<!-- Hotel name  -->
<div class="h-auto w-full border-t-[2.23px] border-[#666666]">
    <div
        class="flex items-center border-b-[2.23px] border-[#666666] bg-[#F2F2F2] py-2 pl-4"
        x-text="MULTI_LANGUAGE_TEXTS.HotelName[LANG_CODE]"></div>
    <div class="col-span-2 flex items-center break-all bg-white py-3 pl-4" 
         x-text="`${o_reserve.s_plan_name}` "></div>
</div>
<!-- 以降、同じ記述を十数回書く… -->
```

### SvelteKitでの改善イメージ

```svelte
<!-- InfoRow.svelte コンポーネント -->
<script lang="ts">
  export let label: string;
  export let value: string;
</script>

<div class="h-auto w-full border-t-[2.23px] border-[#666666]">
  <div class="flex items-center border-b-[2.23px] border-[#666666] bg-[#F2F2F2] py-2 pl-4">
    {label}
  </div>
  <div class="col-span-2 flex items-center break-all bg-white py-3 pl-4">
    {value}
  </div>
</div>
```

```svelte
<!-- 使用側 -->
<InfoRow label={$t('ItineraryNumber')} value={o_reserve.itinerary_id} />
<InfoRow label={$t('HotelName')} value={o_reserve.s_plan_name} />
<InfoRow label={$t('RoomType')} value={o_reserve.s_course_name} />
```

> [!success] 改善効果
> - **コード量の削減**: 十数回の繰り返し → 1つのコンポーネント呼び出し
> - **保守性の向上**: 変更が1箇所で済む
> - **型安全性**: TypeScriptで型チェック可能
> - **再利用性**: 他の画面でも同じコンポーネントを使用可能

---

## 移行計画の概要

> [!note] 段階的移行
> 既存システムを一気に置き換えるのではなく、段階的に移行することを推奨:
> 1. 新規機能からSvelteKitで実装
> 2. 既存機能の改修時にSvelteKitへ移行
> 3. 最終的に全機能をSvelteKit化

---

## 参考リンク

### 採用事例
- [Arial Shop](https://arialshop.com) - Eコマースサイト
- [Beatbump](https://beatbump.io/home) - YouTube Musicの代替フロントエンド
- [日本企業の採用事例まとめ](https://github.com/svelte-jp/japanese-svelte-companies)（随時更新）

### 公式ドキュメント
- [Svelte公式](https://svelte.dev/)
- [SvelteKit公式](https://kit.svelte.dev/)
- [State of JS 2024](https://2024.stateofjs.com/en-US/libraries/)

## 関連ドキュメント

- [[../Frontend/sveltekit-notes|SvelteKitメモ]] - SSR、リアクティビティ等の詳細
- [[../Frontend/svelte/QA|SvelteKit Q&A]] - Sanctum認証・ユニバーサルfetch実装
- [[../Index|ホーム]] - 目次に戻る

---


