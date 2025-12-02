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

# TOMONIシステム 

> [!abstract] 概要
> TOMONIシステムのフロント画面をSvelteKitを導入する計画

---

## 課題

中・大規模アプリ開発における **Alpine.js、素のJS** での開発が年々限界を感じる。

### 主な問題点

> [!bug] 問題
> **処理の共通化が難しく、処理が分散しがち**
> → 同じ処理を何度も書かなければいけない場面がある

### 問題による影響

- 影響調査のコスト増加
- バグ発生率の上昇
- 改修コスト、修正コストの増加
- メンテナンスコストの増加

### これまでの対策

- Web Componentの活用
- HandlebarsによるHTML分割

> [!warning] 限界
> これらの対策でも問題を解消するには至らない。
> 今後さらに複雑な業務ロジック・複雑なデザイン要望の増加が予想される。
> カバーしきれなくなるのは現状から見て、目に見えている。

---

## 解決案: Svelte、Sveltekitの導入

### なぜSvelte(kit)

#### 紹介

| フレームワーク      | 登場時期     |     |
| ------------ | -------- | --- |
| Svelte v1    | 2016年11月 |     |
| SvelteKit    | 2020年10月 |     |
| Alpine.js v1 | 2019年    |     |


Svelteの生みの親は報道機関で、「インタラクティブなニュース記」を作成していた。

彼が直面していた課題は以下の通りです。

- モバイル端末でのパフォーマンス: 
	ニュース記事はスマホで読まれることが多く、回線が遅い環境でも一瞬で表示される必要があります。
- **「ランタイム」の重さ**:  
	ReactやVueを使うと、記事を表示するために「フレームワーク自体のコード（ランタイム）」も一緒にユーザーにダウンロードさせる必要があり、ほんの少しのグラフを表示したいだけなのに、巨大なライブラリを読み込ませるのは非効率だった

「ブラウザに届く前に、コードを最適化してしまえばいいのではないか？」

この発想から、「フレームワーク」ではなく「コンパイラ」として機能するSvelteが生まれました。


2. Svelteのコンセプト
Svelteの最大のコンセプトは、**「フレームワークはブラウザで動くものではなく、ビルドの段階で消え去るもの」**という点にあります。
主な特徴は以下の3点です。


① コンパイラである（The Compiler）
従来（React/Vueなど）: ブラウザに「アプリのコード」＋「フレームワークのコード」を送り、ブラウザの中でフレームワークが計算して画面を動かします。
Svelte: 開発者の書いたコードを、ビルド（コンパイル）の段階で**「極限まで最適化された素のJavaScript（Vanilla JS）」に変換**します。
メリット: ユーザーのブラウザには余計なフレームワークのコードが届かないため、ファイルサイズが圧倒的に小さく、動作も高速になります。
② 仮想DOMを使わない（No Virtual DOM）
Reactなどが採用している「仮想DOM」は、メモリ上で新旧の画面データの差分を比較（Diff）し、変更点を探す仕組みです。これは汎用的ですが、オーバーヘッド（余計な処理負荷）がかかります。
Svelteはコンパイルの段階で、「変数が変わったら、DOMのどこを書き換えればいいか」を完全に把握したコードを生成します。
比較計算をせず、ピンポイントで直接DOMを操作するため、非常に高速でメモリ効率が良いです。
③ コード量を減らす（Write Less Code）
Rich Harris氏は「コードの量はバグの温床である」という考えを持っています。Svelteは非常に記述量が少なくなるように設計されています。
例えば、変数を宣言するだけでリアクティブ（画面に連動する状態）になったり、HTMLの中に自然な形でロジックを書くことができます。
「ボイラープレート（お決まりの冗長な記述）」を極力排除し、開発者が本質的なロジックに集中できるようにしています。

#### State of JavaScript 2024 評価

> [!success] 高評価
> - Svelte: 86%
> - SvelteKit: 89%
>   
>  [State of JS 2024](https://2024.stateofjs.com/en-US/libraries/)

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


