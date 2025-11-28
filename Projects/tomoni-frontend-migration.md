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

## 解決案: SvelteKit

### なぜSvelteKitか

#### 歴史

| フレームワーク | 登場時期 |
|---------------|---------|
| Svelte v1 | 2016年11月 |
| SvelteKit | 2020年10月 |
| Alpine.js v1 | 2019年 |

#### State of JavaScript 2024 評価

> [!success] 高評価
> - Svelte: 86%
> - SvelteKit: 89%

> [!info] 最新情報の確認推奨
> State of JSは毎年更新されるため、最新版を確認することを推奨:
> - [State of JS 2024](https://2024.stateofjs.com/en-US/libraries/)

#### 実務での採用実績

仕事でも使われている統計あり。

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

---

## 補足: ドキュメント作成について

> [!tip] ドキュメント形式
> Excelに画像添付と共に書くのが、コスパ的にいいかもしれない。

---

## 関連ドキュメント

- [[Index|ホーム]] - 目次に戻る

---

*作成日: 2025年11月*

