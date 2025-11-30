---
title: SvelteKit Q&A - Sanctum認証とユニバーサルfetch
created: 2025-11-28
tags:
  - sveltekit
  - svelte
  - frontend
  - typescript
  - sanctum
  - authentication
aliases:
  - SvelteKit Q&A
  - ユニバーサルfetch
---

# SvelteKit Q&A - Sanctum認証とユニバーサルfetch

> [!abstract] 概要
> SvelteKitでLaravel Sanctumを使用した認証と、SSR/クライアント両対応のユニバーサルfetch実装に関するQ&A。

---

## 質問: ユニバーサルfetchの実装方法

### 要件

- Laravel APIサーバーがSanctumを利用している
- フロントからのリクエストはトークンが必要
- サーバー間（NodeとLaravel）の通信も考慮が必要
- Docker環境で名前解決が異なる（URLが変わる）
- セッションcookieも使用している

### 2025年11月現在のベストプラクティス

> [!tip] 推奨アプローチ
> 環境変数によるURL切り替え + SvelteKitの`event.fetch`を使用したユニバーサルfetch関数

---

## 実装方法

### 1. 環境変数を使用したURL設定

#### `src/lib/constants.ts`

```typescript
// ブラウザ環境用（PUBLIC_プレフィックスが必要）
export const API_ORIGIN = 
  typeof window !== 'undefined' 
    ? import.meta.env.PUBLIC_API_ORIGIN || 'http://middle-db.com'
    : import.meta.env.API_ORIGIN || 'http://php';

// サーバー環境用（Dockerコンテナ間通信用）
export const SERVER_API_ORIGIN = import.meta.env.API_ORIGIN || 'http://php';
```

> [!info] 環境変数の違い
> - `PUBLIC_API_ORIGIN`: クライアント側用（ブラウザからアクセスするURL）
> - `API_ORIGIN`: サーバー側用（Dockerコンテナ間通信用）

---

### 2. ユニバーサルfetch関数

#### `src/lib/api/fetch.ts`

```typescript
import { API_ORIGIN } from '$lib/constants';

interface FetchOptions extends RequestInit {
  body?: any;
}

/**
 * SSRとクライアントの両方で動作するユニバーサルfetch関数
 * 
 * @param fetch - SvelteKitのevent.fetch（+page.tsまたは+page.server.tsから取得）
 * @param path - APIエンドポイントのパス（例: '/reservations/dynamic_package/search'）
 * @param options - fetchオプション
 * @param cookies - SvelteKitのcookiesオブジェクト（+page.server.tsの場合）
 */
export async function universalFetch(
  fetch: typeof globalThis.fetch,
  path: string,
  options: FetchOptions = {},
  cookies?: any
): Promise<any> {
  const url = `${API_ORIGIN}${path}`;
  
  // リクエストヘッダーの準備
  const headers: HeadersInit = {
    'Content-Type': 'application/json',
    'Accept': 'application/json',
    ...options.headers,
  };

  // セッションcookieの処理
  if (typeof window !== 'undefined') {
    // ブラウザ環境: credentials: "include"でセッションcookieを自動送信
    options.credentials = 'include';
  } else {
    // サーバー環境: cookiesが提供されている場合は転送
    if (cookies) {
      const sessionCookie = cookies.get('middle_db_session');
      const csrfCookie = cookies.get('XSRF-TOKEN');
      
      if (sessionCookie) {
        headers['Cookie'] = `middle_db_session=${sessionCookie}`;
        if (csrfCookie) {
          headers['Cookie'] += `; XSRF-TOKEN=${csrfCookie}`;
        }
      }
    }
  }

  // CSRFトークンの処理（GET以外のリクエスト）
  if (options.method && options.method !== 'GET') {
    try {
      // CSRFトークンを取得
      const csrfResponse = await fetch(`${API_ORIGIN}/sanctum/csrf-cookie`, {
        method: 'GET',
        credentials: 'include',
        headers: cookies ? {
          'Cookie': cookies.get('middle_db_session') 
            ? `middle_db_session=${cookies.get('middle_db_session')}`
            : ''
        } : {}
      });
      
      // XSRF-TOKENを取得してヘッダーに追加
      const xsrfToken = cookies?.get('XSRF-TOKEN') || 
                       document?.cookie?.match(/XSRF-TOKEN=([^;]+)/)?.[1];
      
      if (xsrfToken) {
        headers['X-XSRF-TOKEN'] = decodeURIComponent(xsrfToken);
      }
    } catch (error) {
      console.warn('CSRFトークンの取得に失敗:', error);
    }
  }

  // リクエストボディの処理
  const body = options.body ? JSON.stringify(options.body) : undefined;

  // fetch実行
  const response = await fetch(url, {
    ...options,
    headers,
    body,
  });

  if (!response.ok) {
    throw new Error(`API Error: ${response.status} ${response.statusText}`);
  }

  return await response.json();
}
```

> [!info] 主な機能
> - **ブラウザ環境**: `credentials: "include"`でセッションcookieを自動送信
> - **サーバー環境**: リクエストのcookieを転送（cookiesが提供されている場合）
> - **SanctumのCSRFトークン**: 自動処理
> - **GET以外のリクエスト**: 事前にCSRFトークンを取得

---

### 3. 使用例

#### `+page.ts`（ユニバーサルload）で使用

```typescript
import { universalFetch } from '$lib/api/fetch';
import type { PageLoad } from './$types';

export const load: PageLoad = async ({ fetch }) => {
  const data = await universalFetch(
    fetch,
    '/reservations/dynamic_package/search',
    {
      method: 'POST',
      body: {
        // 検索パラメータ
        checkin_date: '2025-12-01',
        checkout_date: '2025-12-05',
      }
    }
  );

  return { data };
};
```

> [!warning] 制限
> `+page.ts`では`event.cookies`にアクセスできないため、`event.fetch`が自動的にcookieを転送することを期待します。

#### `+page.server.ts`（サーバーload）で使用（推奨）

```typescript
import { universalFetch } from '$lib/api/fetch';
import type { PageServerLoad } from './$types';

export const load: PageServerLoad = async ({ fetch, cookies }) => {
  const data = await universalFetch(
    fetch,
    '/reservations/dynamic_package/search',
    {
      method: 'POST',
      body: {
        // 検索パラメータ
        checkin_date: '2025-12-01',
        checkout_date: '2025-12-05',
      }
    },
    cookies // cookiesを渡すことで、より確実に動作
  );

  return { data };
};
```

> [!tip] 推奨
> サーバー側でcookieを確実に処理する場合は、`+page.server.ts`を使用してください。

---

## 環境変数の設定

### `.env`ファイルまたは`docker-compose.yml`

```bash
# クライアント側用（ブラウザからアクセスするURL）
PUBLIC_API_ORIGIN=http://middle-db.com

# サーバー側用（Dockerコンテナ間通信用）
API_ORIGIN=http://php
```

> [!info] SvelteKitの環境変数
> - `PUBLIC_`プレフィックス: クライアント側でも使用可能
> - プレフィックスなし: サーバー側のみ使用可能

---

## 注意事項

### 1. Cookie名の設定

Sanctumのセッションcookie名とCSRFトークンのcookie名を確認:

- **セッションcookie**: `middle_db_session`（デフォルト）
- **CSRFトークン**: `XSRF-TOKEN`（デフォルト）

必要に応じて`fetch.ts`内で変更してください。

### 2. Docker環境でのサーバー間通信

> [!warning] 注意
> Docker環境では、サーバー間通信でcookieを明示的に設定する必要がある場合があります。
> その場合は`+page.server.ts`を使用し、`cookies`パラメータを渡してください。

### 3. +page.ts vs +page.server.ts

| 特徴              | +page.ts     | +page.server.ts  |
| --------------- | ------------ | ---------------- |
| **実行環境**        | SSR + クライアント | SSRのみ            |
| **cookiesアクセス** | ❌ 不可         | ✅ 可能             |
| **セキュリティ**      | 中            | 高（機密情報をサーバー側で処理） |
| **推奨ケース**       | 公開データの取得     | 認証が必要なデータの取得     |

---


# +page.tsか、+page.server.tsに記述するかで悩んでます。  
apiはログインセッションが無いとアクセス出来ないのと、laravelがsunctomによるトークン認証を実装しています。

今回のケース（ログインセッション必須、トークン認証、Laravel API）では、**+page.server.ts を使用することを強く推奨します**。


- SvelteKitは「サーバーでもクライアントでも同じコードが動くこと」を保証するため、サーバー実行時であっても +page.ts にはあえてCookieへのアクセス権限を与えていません。


+page.tsはcsr,ssrでも同じようなコードで動かす想定なので、httpOnlyであることがほとんどであるcookieへのアクセスができない。
ssr実行時に、session無し状態でのリクエストとなるため、失敗可能性が高い。

---


## 関連ドキュメント

- [[../sveltekit-notes|SvelteKitメモ]] - SSR、リアクティビティ等の詳細
- [[../../Projects/frontend-migration|TOMONI移行計画]] - フロントエンド移行プロジェクト
- [[../../Index|ホーム]] - 目次に戻る

---

## 参考リンク

- [SvelteKit公式: Loading data](https://kit.svelte.dev/docs/load)
- [Laravel Sanctum公式ドキュメント](https://laravel.com/docs/sanctum)
- [SvelteKit環境変数](https://kit.svelte.dev/docs/modules#$env)

---

*作成日: 2025年11月*
