# TODOアプリ実装方針（REST API対応版）

タグ: #sveltekit #typescript #todo-app #rest-api

---

## 📋 目次

- [[#1. 概要]]
- [[#2. データモデル設計]]
- [[#3. REST API設計]]
- [[#4. APIクライアント実装]]
- [[#5. ルーティング設計]]
- [[#6. 一覧画面の実装方針]]
- [[#7. 詳細画面の実装方針]]
- [[#8. 型定義と `app.d.ts` について]]
- [[#9. `type` と `interface` の使い分け]]
- [[#10. エラーハンドリング]]
- [[#11. 環境変数の設定]]
- [[#12. スタイリング方針]]
- [[#13. 実装の優先順位]]
- [[#14. 学習ポイントまとめ]]
- [[#15. 注意事項]]
- [[#16. 次のステップ]]
- [[#17. 全体のディレクトリ構成]]
- [[#18. まとめ]]

---

## 1. 概要

### 1.1 目的
SvelteKit + TypeScriptの学習を目的とした、CRUD対応のTODOアプリケーションを実装する。REST APIと連携し、サーバーサイドレンダリング（SSR）を活用し、TypeScriptの型安全性を学習する。

### 1.2 機能要件
- **一覧画面**: TODOアイテムの一覧表示（SSR）
- **詳細画面**: 個別のTODOアイテムの詳細表示（SSR）
- **作成機能**: 新しいTODOアイテムの作成（REST API経由）
- **更新機能**: TODOアイテムのステータス更新（チェックボックス、REST API経由）
- **削除機能**: TODOアイテムの削除（オプション、REST API経由）

### 1.3 技術要件
- SvelteKit 2.x（SSR対応）
- TypeScript 5.x（型安全性）
- Tailwind CSS 4.x（スタイリング）
- REST API（外部APIまたはSvelteKit API Routes）

---

## 2. データモデル設計

### 2.1 Todo型の定義

```typescript
// src/lib/types/todo.ts

/**
 * TODOアイテムのステータス
 */
export type TodoStatus = 'pending' | 'completed';

/**
 * TODOアイテムの型定義
 */
export interface Todo {
  /** 一意の識別子 */
  id: string;
  /** タスクの内容 */
  task: string;
  /** タスクのステータス */
  status: TodoStatus;
  /** 作成日時（ISO 8601形式） */
  created_at: string;
}

/**
 * TODOアイテムの作成用データ（idとcreated_atは自動生成）
 */
export type TodoCreateInput = Omit<Todo, 'id' | 'created_at'>;

/**
 * TODOアイテムの更新用データ（部分更新可能）
 */
export type TodoUpdateInput = Partial<Pick<Todo, 'task' | 'status'>>;
```

### 2.2 型定義の学習ポイント
- `interface` による型定義
- `type` エイリアスの使用
- `Omit` と `Pick` ユーティリティ型の活用
- リテラル型（`'pending' | 'completed'`）の使用

---

## 3. REST API設計

### 3.1 APIエンドポイント設計

| メソッド | エンドポイント | 説明 | リクエストボディ | レスポンス | 実装ファイル |
|---------|--------------|------|----------------|-----------|------------|
| `GET` | `/api/todos` | 一覧取得 | - | `Todo[]` | `src/routes/api/todos/+server.ts` |
| `GET` | `/api/todos/:id` | 詳細取得 | - | `Todo` | `src/routes/api/todos/[id]/+server.ts` |
| `POST` | `/api/todos` | 作成 | `TodoCreateInput` | `Todo` | `src/routes/api/todos/+server.ts` |
| `PUT` | `/api/todos/:id` | 更新 | `TodoUpdateInput` | `Todo` | `src/routes/api/todos/[id]/+server.ts` |
| `DELETE` | `/api/todos/:id` | 削除 | - | `{ success: boolean }` | `src/routes/api/todos/[id]/+server.ts` |

### 3.2 SvelteKit API Routesの実装

**重要**: SvelteKitでAPIエンドポイントを作成するには、`src/routes/api/` ディレクトリに `+server.ts` ファイルが必要です。

#### ファイル構造

```
src/routes/
└── api/
    └── todos/
        ├── +server.ts      # GET /api/todos, POST /api/todos
        └── [id]/
            └── +server.ts  # GET /api/todos/:id, PUT /api/todos/:id,DELETE /todos/:id
```

#### 実装例: 一覧取得・作成（`src/routes/api/todos/+server.ts`）

```typescript
// src/routes/api/todos/+server.ts

import { json } from '@sveltejs/kit';
import type { RequestHandler } from './$types';
import type { Todo, TodoCreateInput } from '$lib/types/todo';

// 仮のデータストア（実際にはデータベースやファイルを使用）
let todos: Todo[] = [
  {
    id: '1',
    task: 'SvelteKitを学習する',
    status: 'pending',
    created_at: new Date().toISOString()
  }
];

/**
 * GET /api/todos - 一覧取得
 */
export const GET: RequestHandler = async () => {
  return json(todos);
};

/**
 * POST /api/todos - 作成
 */
export const POST: RequestHandler = async ({ request }) => {
  try {
    const input: TodoCreateInput = await request.json();
    
    // バリデーション
    if (!input.task || input.task.trim() === '') {
      return json(
        { message: 'タスクを入力してください' },
        { status: 400 }
      );
    }
    
    // 新しいTODOを作成
    const newTodo: Todo = {
      id: Date.now().toString(), // 簡易的なID生成（実際にはUUIDなどを使用）
      task: input.task.trim(),
      status: input.status || 'pending',
      created_at: new Date().toISOString()
    };
    
    todos.push(newTodo);
    
    return json(newTodo, { status: 201 });
  } catch (error) {
    return json(
      { message: 'リクエストの解析に失敗しました' },
      { status: 400 }
    );
  }
};
```

#### 実装例: 詳細取得・更新・削除（`src/routes/api/todos/[id]/+server.ts`）

```typescript
// src/routes/api/todos/[id]/+server.ts

import { json, error } from '@sveltejs/kit';
import type { RequestHandler } from './$types';
import type { Todo, TodoUpdateInput } from '$lib/types/todo';

// 仮のデータストア（実際にはデータベースやファイルを使用）
// 注意: 実際の実装では、データストアを共有する必要があります
// ここでは説明のため、簡易的な実装としています

/**
 * GET /api/todos/:id - 詳細取得
 */
export const GET: RequestHandler = async ({ params }) => {
  // 実際の実装では、データストアから取得
  // const todo = await getTodoById(params.id);
  
  // 仮の実装
  const todo: Todo | undefined = undefined; // 実際にはデータストアから取得
  
  if (!todo) {
    throw error(404, { message: 'TODOアイテムが見つかりません' });
  }
  
  return json(todo);
};

/**
 * PUT /api/todos/:id - 更新
 */
export const PUT: RequestHandler = async ({ params, request }) => {
  try {
    const input: TodoUpdateInput = await request.json();
    
    // 実際の実装では、データストアから取得して更新
    // const todo = await updateTodo(params.id, input);
    
    // 仮の実装
    const todo: Todo | undefined = undefined; // 実際にはデータストアから取得・更新
    
    if (!todo) {
      throw error(404, { message: 'TODOアイテムが見つかりません' });
    }
    
    return json(todo);
  } catch (e) {
    if (e && typeof e === 'object' && 'status' in e) {
      throw e; // SvelteKitのerror()を再スロー
    }
    return json(
      { message: 'リクエストの解析に失敗しました' },
      { status: 400 }
    );
  }
};

/**
 * DELETE /api/todos/:id - 削除
 */
export const DELETE: RequestHandler = async ({ params }) => {
  // 実際の実装では、データストアから削除
  // const success = await deleteTodo(params.id);
  
  // 仮の実装
  const success = true; // 実際にはデータストアから削除
  
  if (!success) {
    throw error(404, { message: 'TODOアイテムが見つかりません' });
  }
  
  return json({ success: true });
};
```

#### データストアの実装（推奨）

実際の実装では、データストアを別ファイルに分離することを推奨します：

```typescript
// src/lib/server/todos-store.ts

import type { Todo, TodoCreateInput, TodoUpdateInput } from '$lib/types/todo';
import { readFile, writeFile } from 'fs/promises';
import { join } from 'path';

const DATA_FILE_PATH = join(process.cwd(), 'src/lib/server/data/todos.json');

export async function getTodos(): Promise<Todo[]> {
  try {
    const data = await readFile(DATA_FILE_PATH, 'utf-8');
    const json = JSON.parse(data);
    return json.todos || [];
  } catch {
    return [];
  }
}

export async function getTodoById(id: string): Promise<Todo | null> {
  const todos = await getTodos();
  return todos.find(t => t.id === id) || null;
}

export async function createTodo(input: TodoCreateInput): Promise<Todo> {
  const todos = await getTodos();
  const newTodo: Todo = {
    id: crypto.randomUUID(),
    task: input.task,
    status: input.status || 'pending',
    created_at: new Date().toISOString()
  };
  todos.push(newTodo);
  await writeFile(DATA_FILE_PATH, JSON.stringify({ todos }, null, 2));
  return newTodo;
}

export async function updateTodo(id: string, input: TodoUpdateInput): Promise<Todo | null> {
  const todos = await getTodos();
  const index = todos.findIndex(t => t.id === id);
  if (index === -1) return null;
  
  todos[index] = { ...todos[index], ...input };
  await writeFile(DATA_FILE_PATH, JSON.stringify({ todos }, null, 2));
  return todos[index];
}

export async function deleteTodo(id: string): Promise<boolean> {
  const todos = await getTodos();
  const index = todos.findIndex(t => t.id === id);
  if (index === -1) return false;
  
  todos.splice(index, 1);
  await writeFile(DATA_FILE_PATH, JSON.stringify({ todos }, null, 2));
  return true;
}
```

そして、`+server.ts` で使用：

```typescript
// src/routes/api/todos/+server.ts

import { json } from '@sveltejs/kit';
import type { RequestHandler } from './$types';
import * as todosStore from '$lib/server/todos-store';

export const GET: RequestHandler = async () => {
  const todos = await todosStore.getTodos();
  return json(todos);
};

export const POST: RequestHandler = async ({ request }) => {
  const input = await request.json();
  const todo = await todosStore.createTodo(input);
  return json(todo, { status: 201 });
};
```

### 3.2 APIレスポンスの型定義

```typescript
// src/lib/types/api.ts

/**
 * APIエラーレスポンス
 */
export interface ApiError {
  message: string;
  code?: string;
  errors?: Record<string, string[]>;
}

/**
 * API成功レスポンス（作成・更新）
 */
export interface ApiSuccess<T> {
  data: T;
  message?: string;
}

/**
 * API一覧レスポンス
 */
export interface ApiListResponse<T> {
  data: T[];
  total?: number;
  page?: number;
  limit?: number;
}
```

### 3.3 APIベースURLの設定

```typescript
// src/lib/constants.ts

import { browser } from '$app/environment';

/**
 * APIのベースURLを取得
 * - ブラウザ環境: PUBLIC_API_ORIGIN環境変数を使用
 * - サーバー環境: API_ORIGIN環境変数を使用
 */
function getApiOrigin(): string {
  if (browser) {
    // クライアント側: PUBLIC_プレフィックス付き環境変数を使用
    return import.meta.env.PUBLIC_API_ORIGIN || 'http://localhost:3000';
  } else {
    // サーバー側: API_ORIGIN環境変数を使用
    return process.env.API_ORIGIN || 'http://localhost:3000';
  }
}

export const API_ORIGIN = getApiOrigin();
export const API_BASE_URL = `${API_ORIGIN}/api`;
```

---

## 4. APIクライアント実装

### 4.1 汎用APIフェッチ関数

```typescript
// src/lib/api/fetch.ts

import { API_BASE_URL } from '$lib/constants';
import type { ApiError } from '$lib/types/api';

/**
 * HTTPメソッド
 */
export type HttpMethod = 'GET' | 'POST' | 'PUT' | 'DELETE' | 'PATCH';

/**
 * APIリクエストオプション
 */
export interface ApiRequestOptions {
  method?: HttpMethod;
  body?: unknown;
  headers?: Record<string, string>;
}

/**
 * 汎用APIフェッチ関数
 * 
 * @param fetchFn - SvelteKitのevent.fetch（サーバー側）またはfetch（クライアント側）
 * @param endpoint - APIエンドポイント（例: '/todos'）
 * @param options - リクエストオプション
 * @returns APIレスポンスのデータ
 */
export async function apiFetch<T>(
  fetchFn: typeof fetch,
  endpoint: string,
  options: ApiRequestOptions = {}
): Promise<T> {
  const { method = 'GET', body, headers = {} } = options;
  
  // エンドポイントが完全なURLでない場合は、API_BASE_URLを付与
  const url = endpoint.startsWith('http') 
    ? endpoint 
    : `${API_BASE_URL}${endpoint}`;

  const requestOptions: RequestInit = {
    method,
    headers: {
      'Content-Type': 'application/json',
      ...headers,
    },
  };

  if (body && method !== 'GET') {
    requestOptions.body = JSON.stringify(body);
  }

  const response = await fetchFn(url, requestOptions);

  if (!response.ok) {
    let errorMessage = `API request failed: ${response.status} ${response.statusText}`;
    
    try {
      const errorData: ApiError = await response.json();
      errorMessage = errorData.message || errorMessage;
    } catch {
      // JSON解析に失敗した場合は、テキストを取得
      try {
        errorMessage = await response.text();
      } catch {
        // テキスト取得にも失敗した場合は、デフォルトメッセージを使用
      }
    }
    
    throw new Error(errorMessage);
  }

  // レスポンスが空の場合は、空オブジェクトを返す
  const contentType = response.headers.get('content-type');
  if (!contentType || !contentType.includes('application/json')) {
    return {} as T;
  }

  return await response.json();
}
```

### 4.2 TODO専用API関数（サーバーサイド用）

```typescript
// src/lib/api/todos.ts

import { apiFetch } from './fetch';
import type { Todo, TodoCreateInput, TodoUpdateInput } from '$lib/types/todo';

/**
 * TODO一覧を取得（サーバーサイド用）
 */
export async function getTodos(fetchFn: typeof fetch): Promise<Todo[]> {
  return apiFetch<Todo[]>(fetchFn, '/todos');
}

/**
 * 指定IDのTODOを取得（サーバーサイド用）
 */
export async function getTodoById(
  fetchFn: typeof fetch,
  id: string
): Promise<Todo> {
  return apiFetch<Todo>(fetchFn, `/todos/${id}`);
}

/**
 * 新しいTODOを作成（サーバーサイド用）
 */
export async function createTodo(
  fetchFn: typeof fetch,
  input: TodoCreateInput
): Promise<Todo> {
  return apiFetch<Todo>(fetchFn, '/todos', {
    method: 'POST',
    body: input,
  });
}

/**
 * TODOを更新（サーバーサイド用）
 */
export async function updateTodo(
  fetchFn: typeof fetch,
  id: string,
  input: TodoUpdateInput
): Promise<Todo> {
  return apiFetch<Todo>(fetchFn, `/todos/${id}`, {
    method: 'PUT',
    body: input,
  });
}

/**
 * TODOを削除（サーバーサイド用）
 */
export async function deleteTodo(
  fetchFn: typeof fetch,
  id: string
): Promise<{ success: boolean }> {
  return apiFetch<{ success: boolean }>(fetchFn, `/todos/${id}`, {
    method: 'DELETE',
  });
}
```

### 4.3 クライアントサイド用API関数（ブラウザから直接呼び出し可能）

```typescript
// src/lib/api/todos.client.ts

import { browser } from '$app/environment';
import { API_BASE_URL } from '$lib/constants';
import type { Todo, TodoCreateInput, TodoUpdateInput } from '$lib/types/todo';
import type { ApiError } from '$lib/types/api';

/**
 * ブラウザ環境でのAPI呼び出し関数
 */
async function clientApiFetch<T>(
  endpoint: string,
  options: RequestInit = {}
): Promise<T> {
  if (!browser) {
    throw new Error('This function can only be called in the browser');
  }

  const url = endpoint.startsWith('http') 
    ? endpoint 
    : `${API_BASE_URL}${endpoint}`;

  const response = await fetch(url, {
    ...options,
    headers: {
      'Content-Type': 'application/json',
      ...options.headers,
    },
  });

  if (!response.ok) {
    let errorMessage = `API request failed: ${response.status} ${response.statusText}`;
    
    try {
      const errorData: ApiError = await response.json();
      errorMessage = errorData.message || errorMessage;
    } catch {
      try {
        errorMessage = await response.text();
      } catch {
        // デフォルトメッセージを使用
      }
    }
    
    throw new Error(errorMessage);
  }

  const contentType = response.headers.get('content-type');
  if (!contentType || !contentType.includes('application/json')) {
    return {} as T;
  }

  return await response.json();
}

/**
 * TODO一覧を取得（ブラウザから直接呼び出し可能）
 */
export async function getTodosClient(): Promise<Todo[]> {
  return clientApiFetch<Todo[]>('/todos');
}

/**
 * 指定IDのTODOを取得（ブラウザから直接呼び出し可能）
 */
export async function getTodoByIdClient(id: string): Promise<Todo> {
  return clientApiFetch<Todo>(`/todos/${id}`);
}

/**
 * 新しいTODOを作成（ブラウザから直接呼び出し可能）
 */
export async function createTodoClient(input: TodoCreateInput): Promise<Todo> {
  return clientApiFetch<Todo>('/todos', {
    method: 'POST',
    body: JSON.stringify(input),
  });
}

/**
 * TODOを更新（ブラウザから直接呼び出し可能）
 */
export async function updateTodoClient(
  id: string,
  input: TodoUpdateInput
): Promise<Todo> {
  return clientApiFetch<Todo>(`/todos/${id}`, {
    method: 'PUT',
    body: JSON.stringify(input),
  });
}

/**
 * TODOを削除（ブラウザから直接呼び出し可能）
 */
export async function deleteTodoClient(id: string): Promise<{ success: boolean }> {
  return clientApiFetch<{ success: boolean }>(`/todos/${id}`, {
    method: 'DELETE',
  });
}
```

### 4.4 クライアントサイドでの使用例

```svelte
<!-- src/routes/+page.svelte -->
<script lang="ts">
  import { onMount } from 'svelte';
  import { getTodosClient, createTodoClient, updateTodoClient } from '$lib/api/todos.client';
  import type { Todo, TodoCreateInput } from '$lib/types/todo';
  
  let todos: Todo[] = [];
  let loading = false;
  let error: string | null = null;
  let newTask = '';

  // コンポーネントマウント時にデータを取得
  onMount(async () => {
    await loadTodos();
  });

  async function loadTodos() {
    loading = true;
    error = null;
    try {
      todos = await getTodosClient();
    } catch (e) {
      error = e instanceof Error ? e.message : 'データの取得に失敗しました';
    } finally {
      loading = false;
    }
  }

  async function handleCreate() {
    if (!newTask.trim()) return;
    
    loading = true;
    error = null;
    try {
      const input: TodoCreateInput = {
        task: newTask.trim(),
        status: 'pending'
      };
      const newTodo = await createTodoClient(input);
      todos = [...todos, newTodo];
      newTask = '';
    } catch (e) {
      error = e instanceof Error ? e.message : 'TODOの作成に失敗しました';
    } finally {
      loading = false;
    }
  }

  async function handleToggleStatus(todo: Todo) {
    const newStatus = todo.status === 'completed' ? 'pending' : 'completed';
    try {
      const updated = await updateTodoClient(todo.id, { status: newStatus });
      todos = todos.map(t => t.id === todo.id ? updated : t);
    } catch (e) {
      error = e instanceof Error ? e.message : 'ステータスの更新に失敗しました';
    }
  }
</script>

{#if loading}
  <p>読み込み中...</p>
{/if}

{#if error}
  <div class="error">{error}</div>
{/if}

<!-- TODO作成フォーム -->
<form on:submit|preventDefault={handleCreate}>
  <input type="text" bind:value={newTask} placeholder="新しいタスクを入力..." />
  <button type="submit" disabled={loading}>作成</button>
</form>

<!-- TODO一覧 -->
<ul>
  {#each todos as todo (todo.id)}
    <li>
      <input 
        type="checkbox" 
        checked={todo.status === 'completed'}
        onchange={() => handleToggleStatus(todo)}
      />
      <span class:completed={todo.status === 'completed'}>
        {todo.task}
      </span>
      <a href="/todos/{todo.id}">詳細</a>
    </li>
  {/each}
</ul>
```

---

## 5. ルーティング設計

### 5.1 ファイル構造

```
src/routes/
├── +layout.svelte          # 共通レイアウト
├── +layout.server.ts       # レイアウトのサーバーサイドロジック（オプション）
├── +page.svelte            # 一覧画面（トップページ）
├── +page.server.ts         # 一覧画面のサーバーサイドロジック
├── +error.svelte           # エラーページ
├── api/                    # APIエンドポイント
│   └── todos/
│       ├── +server.ts      # GET /api/todos, POST /api/todos
│       └── [id]/
│           └── +server.ts  # GET /api/todos/:id, PUT /api/todos/:id, DELETE /api/todos/:id
├── todos/
│   └── [id]/
│       ├── +page.svelte    # 詳細画面
│       └── +page.server.ts # 詳細画面のサーバーサイドロジック
```

### 5.2 ルート定義

| パス | ファイル | 説明 | データ取得方法 |
|------|---------|------|---------------|
| `/` | `+page.svelte` | 一覧画面 | SSR（`+page.server.ts`）またはクライアントサイド（`todos.client.ts`） |
| `/todos/[id]` | `todos/[id]/+page.svelte` | 詳細画面（動的ルート） | SSR（`+page.server.ts`）またはクライアントサイド（`todos.client.ts`） |

### 5.3 データ取得方法の選択

#### パターンA: SSR（サーバーサイドレンダリング） - 推奨
- **メリット**: 初期表示が高速、SEO対応、データが確実に取得される
- **使用場所**: `+page.server.ts` で `event.fetch` を使用
- **実装**: `src/lib/api/todos.ts` の関数を使用

#### パターンB: クライアントサイド
- **メリット**: ページ遷移なしでデータ更新可能、インタラクティブ
- **使用場所**: コンポーネント内で `onMount` やイベントハンドラから呼び出し
- **実装**: `src/lib/api/todos.client.ts` の関数を使用

#### パターンC: ハイブリッド（SSR + クライアント更新）
- **メリット**: 初期表示はSSR、その後はクライアントサイドで更新
- **実装**: `+page.server.ts` で初期データを取得、コンポーネント内で `todos.client.ts` を使用して更新

---

## 6. 一覧画面の実装方針

### 6.1 サーバーサイドロジック（`+page.server.ts`）

```typescript
// src/routes/+page.server.ts

import { getTodos } from '$lib/api/todos';
import type { PageServerLoad } from './$types';

/**
 * 一覧画面のデータをサーバーサイドで取得
 */
export const load: PageServerLoad = async ({ fetch }) => {
  try {
    const todos = await getTodos(fetch);
    
    return {
      todos
    };
  } catch (error) {
    // エラーハンドリング
    console.error('Failed to load todos:', error);
    return {
      todos: [],
      error: error instanceof Error ? error.message : 'Failed to load todos'
    };
  }
};
```

**学習ポイント**:
- `PageServerLoad` 型の使用
- `event.fetch` によるサーバーサイドでのAPI呼び出し
- `$types` による自動型生成
- エラーハンドリング

### 6.2 フォームアクション（作成・更新）

```typescript
// src/routes/+page.server.ts

import type { Actions } from './$types';
import { createTodo, updateTodo } from '$lib/api/todos';

export const actions: Actions = {
  /**
   * 新しいTODOアイテムを作成
   */
  create: async ({ request, fetch }) => {
    const formData = await request.formData();
    const task = formData.get('task')?.toString();
    
    if (!task || task.trim() === '') {
      return {
        error: 'タスクを入力してください'
      };
    }
    
    try {
      const todo = await createTodo(fetch, {
        task: task.trim(),
        status: 'pending'
      });
      
      return {
        success: true,
        todo
      };
    } catch (error) {
      return {
        error: error instanceof Error ? error.message : 'TODOの作成に失敗しました'
      };
    }
  },
  
  /**
   * TODOアイテムのステータスを更新
   */
  updateStatus: async ({ request, fetch }) => {
    const formData = await request.formData();
    const id = formData.get('id')?.toString();
    const status = formData.get('status')?.toString() as 'pending' | 'completed';
    
    if (!id || !status) {
      return {
        error: '無効なリクエストです'
      };
    }
    
    try {
      const todo = await updateTodo(fetch, id, { status });
      
      return {
        success: true,
        todo
      };
    } catch (error) {
      return {
        error: error instanceof Error ? error.message : 'TODOの更新に失敗しました'
      };
    }
  }
};
```

**学習ポイント**:
- `Actions` 型の使用
- `event.fetch` によるサーバーサイドでのAPI呼び出し
- フォームデータの取得とバリデーション
- エラーハンドリング

### 6.3 コンポーネント（`+page.svelte`）

```svelte
<!-- src/routes/+page.svelte -->

<script lang="ts">
  import { enhance } from '$app/forms';
  import type { PageData } from './$types';
  
  let { data }: { data: PageData } = $props();
</script>

{#if data.error}
  <div class="error-message">
    <p>{data.error}</p>
  </div>
{/if}

<!-- TODO作成フォーム -->
<form method="POST" action="?/create" use:enhance>
  <input type="text" name="task" placeholder="新しいタスクを入力..." required />
  <button type="submit">作成</button>
</form>

<!-- TODO一覧 -->
<ul>
  {#each data.todos as todo (todo.id)}
    <li>
      <form method="POST" action="?/updateStatus" use:enhance>
        <input type="hidden" name="id" value={todo.id} />
        <input 
          type="checkbox" 
          name="status" 
          value="completed"
          checked={todo.status === 'completed'}
          onchange={(e) => {
            if (e.currentTarget.checked) {
              e.currentTarget.form?.requestSubmit();
            } else {
              // チェックを外す場合は pending に更新
              const form = e.currentTarget.form;
              if (form) {
                const statusInput = form.querySelector('input[name="status"]') as HTMLInputElement;
                if (statusInput) {
                  statusInput.value = 'pending';
                  form.requestSubmit();
                }
              }
            }
          }}
        />
        <span class:completed={todo.status === 'completed'}>
          {todo.task}
        </span>
      </form>
      <a href="/todos/{todo.id}">詳細</a>
    </li>
  {/each}
</ul>
```

**学習ポイント**:
- `PageData` 型による型安全なデータアクセス
- `use:enhance` による最適化されたフォーム送信
- `{#each}` ブロックとキー指定
- 条件付きクラスバインディング
- エラー表示

---

## 7. 詳細画面の実装方針

### 7.1 サーバーサイドロジック（`todos/[id]/+page.server.ts`）

```typescript
// src/routes/todos/[id]/+page.server.ts

import { getTodoById } from '$lib/api/todos';
import { error } from '@sveltejs/kit';
import type { PageServerLoad } from './$types';

/**
 * 詳細画面のデータをサーバーサイドで取得
 */
export const load: PageServerLoad = async ({ params, fetch }) => {
  try {
    const todo = await getTodoById(fetch, params.id);
    
    return {
      todo,
      // ページタイトルとして使用
      title: todo.task
    };
  } catch (err) {
    // APIエラーの場合、404を返す
    throw error(404, {
      message: 'TODOアイテムが見つかりません'
    });
  }
};
```

**学習ポイント**:
- 動的ルートパラメータ（`params.id`）の取得
- `event.fetch` によるサーバーサイドでのAPI呼び出し
- エラーハンドリング（`error()` 関数）
- ページタイトルの設定

### 7.2 コンポーネント（`todos/[id]/+page.svelte`）

```svelte
<!-- src/routes/todos/[id]/+page.svelte -->

<script lang="ts">
  import type { PageData } from './$types';
  
  let { data }: { data: PageData } = $props();
</script>

<svelte:head>
  <title>{data.title} - TODO詳細</title>
</svelte:head>

<article>
  <h1>{data.todo.task}</h1>
  <p>ステータス: {data.todo.status === 'completed' ? '完了' : '未完了'}</p>
  <p>作成日時: {new Date(data.todo.created_at).toLocaleString('ja-JP')}</p>
  <a href="/">一覧に戻る</a>
</article>
```

**学習ポイント**:
- `<svelte:head>` によるメタデータの設定
- 日付のフォーマット
- 型安全なデータアクセス

---

## 8. 型定義と `app.d.ts` について

### 8.1 `app.d.ts` の拡張は**不要**（今回のTODOアプリの場合）

**理由**:
- SvelteKitは `+page.server.ts` の `load` 関数の戻り値から、自動的に `PageData` 型を生成します
- 各ページで `import type { PageData } from './$types'` として使用できます
- `src/lib/types/todo.ts` で型を定義し、各ページの `PageData` で直接使用すれば十分です

**実際の使い方**:

```typescript
// src/routes/+page.server.ts
export const load: PageServerLoad = async ({ fetch }) => {
  const todos = await getTodos(fetch); // Todo[] 型
  return { todos }; // PageData は自動的に { todos: Todo[] } になる
};
```

```svelte
<!-- src/routes/+page.svelte -->
<script lang="ts">
  import type { PageData } from './$types';
  // PageData は自動的に { todos: Todo[] } 型になっている
  
  let { data }: { data: PageData } = $props();
  // data.todos は型安全にアクセス可能
</script>
```

### 8.2 `app.d.ts` の拡張が**必要**な場面

以下のような場合にのみ、`app.d.ts` の拡張が必要です：

#### ケース1: 全ページで共通のデータ構造を定義したい場合

**例**: 認証情報を全ページで使用する

```typescript
// src/app.d.ts
import type { User } from '$lib/types/user';

declare global {
  namespace App {
    interface PageData {
      // 全ページで共通して使用する認証情報
      user?: User;
    }
  }
}
```

**使用例**:
```typescript
// src/routes/+layout.server.ts
export const load: PageServerLoad = async ({ fetch }) => {
  const user = await getCurrentUser(fetch);
  return { user }; // 全ページで user が使える
};
```

```svelte
<!-- どのページでも user にアクセス可能 -->
<script lang="ts">
  import type { PageData } from './$types';
  let { data }: { data: PageData } = $props();
  
  // data.user は全ページで型安全に使用可能
  {#if data.user}
    <p>こんにちは、{data.user.name}さん</p>
  {/if}
</script>
```

#### ケース2: レイアウトで共通データを提供する場合

**例**: サイト全体の設定情報

```typescript
// src/app.d.ts
interface SiteConfig {
  siteName: string;
  theme: 'light' | 'dark';
}

declare global {
  namespace App {
    interface PageData {
      siteConfig: SiteConfig; // 全ページで必須
    }
  }
}
```

```typescript
// src/routes/+layout.server.ts
export const load: PageServerLoad = async ({ fetch }) => {
  return {
    siteConfig: {
      siteName: 'My App',
      theme: 'light'
    }
  };
};
```

### 8.3 まとめ: どちらを使うべきか？

| 状況 | `app.d.ts` の拡張 | 各ページで個別に定義 |
|------|------------------|---------------------|
| **今回のTODOアプリ** | ❌ 不要 | ✅ 推奨 |
| 全ページで共通データ | ✅ 必要 | ❌ 非効率 |
| ページごとに異なるデータ | ❌ 不要 | ✅ 推奨 |

**今回のTODOアプリでは**:
- 一覧画面: `{ todos: Todo[] }`
- 詳細画面: `{ todo: Todo, title: string }`

各ページで異なるデータ構造なので、`app.d.ts` の拡張は不要です。`src/lib/types/todo.ts` で型を定義し、各ページの `PageData` で直接使用する方が明確で理解しやすいです。

---

## 9. `type` と `interface` の使い分け

### 9.1 使い分けの意図

```typescript
// リテラル型のユニオン → type
export type TodoStatus = 'pending' | 'completed';

// オブジェクトの形状 → interface（拡張可能）
export interface Todo {
  id: string;
  task: string;
  status: TodoStatus;
  created_at: string;
}

// ユーティリティ型を使う → type
export type TodoCreateInput = Omit<Todo, 'id' | 'created_at'>;
export type TodoUpdateInput = Partial<Pick<Todo, 'task' | 'status'>>;
```

**理由**:
1. **`TodoStatus` → `type`**: リテラル型のユニオン（`'pending' | 'completed'`）は `interface` では定義できないため
2. **`Todo` → `interface`**: オブジェクトの形状を定義し、将来の拡張（`extends`、`implements`）を考慮
3. **`TodoCreateInput` / `TodoUpdateInput` → `type`**: ユーティリティ型（`Omit`、`Pick`、`Partial`）を使用するため

### 9.2 使い分けのガイドライン

| 用途 | 推奨 | 理由 |
|------|------|------|
| リテラル型のユニオン | `type` | `interface` では定義できない |
| オブジェクトの形状 | `interface` または `type` | どちらでも可。拡張が必要なら `interface` |
| ユーティリティ型を使う | `type` | `Omit`、`Pick`、`Partial` などは `type` が自然 |
| 交差型（`&`） | `type` | `interface` では直接使えない |
| 関数型 | `type` | `interface` でも可だが、`type` が一般的 |

---

## 10. エラーハンドリング

### 10.1 エラーページの実装

```svelte
<!-- src/routes/+error.svelte -->

<script lang="ts">
  import { page } from '$app/stores';
  import type { Error } from './$types';
  
  let { error }: { error: Error } = $props();
</script>

<h1>{error.status}</h1>
<p>{error.message}</p>
<a href="/">ホームに戻る</a>
```

### 10.2 APIエラーの処理
- 存在しないTODO IDにアクセスした場合 → 404エラー
- API接続エラーの場合 → エラーメッセージを表示
- バリデーションエラーの場合 → フォームにエラーを表示

---

## 11. 環境変数の設定

### 11.1 `.env` ファイル

```bash
# .env（開発環境）
PUBLIC_API_ORIGIN=http://localhost:3000
API_ORIGIN=http://localhost:3000

# .env.production（本番環境）
PUBLIC_API_ORIGIN=https://api.example.com
API_ORIGIN=https://api.example.com
```

### 11.2 環境変数の使い分け
- `PUBLIC_API_ORIGIN`: クライアント側（ブラウザ）で使用可能
- `API_ORIGIN`: サーバー側でのみ使用可能（セキュリティのため）

---

## 12. スタイリング方針

### 12.1 Tailwind CSSの活用
- ユーティリティクラスによるスタイリング
- レスポンシブデザインの実装
- ダークモード対応（オプション）

### 12.2 コンポーネントのスタイル例

```svelte
<!-- チェックボックス付きTODOアイテム -->
<li class="flex items-center gap-2 p-4 border rounded-lg">
  <input type="checkbox" class="w-5 h-5" />
  <span class="flex-1">タスク内容</span>
  <a href="/todos/1" class="text-blue-500 hover:underline">詳細</a>
</li>
```

---

## 13. 実装の優先順位

### フェーズ1: 基本機能
1. ✅ Todo型の定義
2. ✅ APIクライアント関数の実装
3. ✅ 一覧画面のSSR実装
4. ✅ 詳細画面のSSR実装

### フェーズ2: CRUD機能
5. ✅ TODO作成機能
6. ✅ ステータス更新機能（チェックボックス）
7. ✅ 削除機能（オプション）

### フェーズ3: 改善
8. ✅ エラーハンドリング
9. ✅ バリデーション
10. ✅ スタイリング
11. ✅ ローディング状態の表示

---

## 14. 学習ポイントまとめ

### TypeScript関連
- ✅ インターフェースと型エイリアスの定義
- ✅ ユーティリティ型（`Omit`, `Pick`, `Partial`）の使用
- ✅ リテラル型の活用
- ✅ 型安全なデータアクセス
- ✅ `type` と `interface` の使い分け
- ✅ `app.d.ts` の拡張が必要な場面と不要な場面の理解

### SvelteKit関連
- ✅ ファイルベースルーティング
- ✅ サーバーサイドロジック（`+page.server.ts`）
- ✅ フォームアクション（`Actions`）
- ✅ 動的ルート（`[id]`）
- ✅ `use:enhance` による最適化
- ✅ `$types` による自動型生成
- ✅ `event.fetch` によるサーバーサイドでのAPI呼び出し

### REST API関連
- ✅ RESTful APIの設計原則
- ✅ HTTPメソッドの適切な使用
- ✅ エラーハンドリング
- ✅ 型安全なAPIクライアントの実装

### その他
- ✅ 非同期処理（`async/await`）
- ✅ エラーハンドリング
- ✅ 環境変数の管理

---

## 15. 注意事項

### 15.1 API実装について
- **SvelteKit API Routesを使用する場合**: `src/routes/api/todos/+server.ts` に実装
- **外部APIサーバーを使用する場合**: APIエンドポイントのURLを環境変数で設定
- この方針では、SvelteKit API Routesの実装例を記載しています（セクション3.2参照）

### 15.2 認証について
- チュートリアル用途では認証は不要としています
- 実装時には、必要に応じて認証トークンの処理を追加してください

### 15.3 セキュリティ
- バリデーションの実装
- XSS対策（Svelteはデフォルトでエスケープ）
- CSRF対策（SvelteKitのフォームアクションは自動対応）
- APIキーや認証トークンの適切な管理

---

## 16. 次のステップ

実装完了後は、以下の拡張を検討：

1. **削除機能の追加**
2. **編集機能の追加**（タスク内容の変更）
3. **フィルタリング機能**（完了/未完了で絞り込み）
4. **ソート機能**（作成日時順など）
5. **ページネーション**（大量データ対応）
6. **リアルタイム更新**（WebSocketなど）
7. **認証機能の追加**
8. **オプティミスティックUI更新**

---

## 17. 全体のディレクトリ構成

### 17.1 プロジェクト全体の構造

```
front-svelte-tutorial/
├── .env                          # 環境変数（開発環境）
├── .env.production               # 環境変数（本番環境）
├── .gitignore                    # Git除外設定
├── package.json                  # 依存関係とスクリプト
├── pnpm-lock.yaml                # 依存関係のロックファイル
├── pnpm-workspace.yaml           # pnpmワークスペース設定
├── README.md                     # プロジェクト説明
├── svelte.config.js               # SvelteKit設定
├── tsconfig.json                  # TypeScript設定
├── vite.config.ts                # Vite設定
├── TODO_APP_IMPLEMENTATION_PLAN.md # 実装方針（このファイル）
│
├── src/                          # ソースコード
│   ├── app.d.ts                  # SvelteKit型定義拡張
│   ├── app.html                  # HTMLテンプレート
│   │
│   ├── lib/                      # ライブラリコード（再利用可能）
│   │   ├── api/                  # APIクライアント
│   │   │   ├── fetch.ts          # 汎用APIフェッチ関数（サーバーサイド用）
│   │   │   ├── todos.ts          # TODO API関数（サーバーサイド用）
│   │   │   └── todos.client.ts   # TODO API関数（クライアントサイド用）
│   │   │
│   │   ├── assets/               # 静的アセット
│   │   │   └── favicon.svg       # ファビコン
│   │   │
│   │   ├── constants.ts          # 定数定義（API_BASE_URLなど）
│   │   │
│   │   ├── types/                # 型定義
│   │   │   ├── todo.ts           # TODO関連の型定義
│   │   │   └── api.ts            # API関連の型定義
│   │   │
│   │   ├── server/               # サーバー専用コード（ブラウザでは実行不可）
│   │   │   └── data/             # データファイル（JSONファイル方式の場合）
│   │   │       └── todos.json    # TODOデータ（JSONファイル方式の場合のみ）
│   │   │
│   │   └── index.ts              # ライブラリのエクスポート
│   │
│   └── routes/                   # ルーティング（ファイルベース）
│       ├── +layout.svelte         # 共通レイアウト
│       ├── +layout.server.ts      # レイアウトのサーバーサイドロジック（オプション）
│       ├── +page.svelte          # 一覧画面（トップページ）
│       ├── +page.server.ts       # 一覧画面のサーバーサイドロジック
│       ├── +error.svelte         # エラーページ
│       │
│       └── todos/                 # TODO関連のルート
│           └── [id]/             # 動的ルート（IDパラメータ）
│               ├── +page.svelte  # 詳細画面
│               └── +page.server.ts # 詳細画面のサーバーサイドロジック
│
├── static/                       # 静的ファイル（そのまま配信される）
│   └── (画像、フォントなど)
│
└── node_modules/                 # 依存パッケージ（自動生成）
```

### 17.2 主要ディレクトリの説明

#### `src/lib/` - ライブラリコード
- **再利用可能なコード**を配置
- `$lib` エイリアスでインポート可能（例: `import { ... } from '$lib/api/todos'`）

#### `src/lib/api/` - APIクライアント
- **サーバーサイド用**: `fetch.ts`, `todos.ts`（`event.fetch` を使用）
- **クライアントサイド用**: `todos.client.ts`（ブラウザの `fetch` を使用）

#### `src/lib/types/` - 型定義
- TypeScriptの型定義を集約
- `todo.ts`: TODO関連の型
- `api.ts`: API関連の型

#### `src/lib/server/` - サーバー専用コード
- **ブラウザでは実行されない**コード
- ファイルI/O、データベースアクセスなど

#### `src/routes/` - ルーティング
- **ファイルベースルーティング**
- `+page.svelte`: ページコンポーネント
- `+page.server.ts`: ページのサーバーサイドロジック
- `+layout.svelte`: レイアウトコンポーネント
- `+layout.server.ts`: レイアウトのサーバーサイドロジック
- `+server.ts`: **APIエンドポイント**（`src/routes/api/` 配下に配置）
- `[id]`: 動的ルートパラメータ

### 17.3 ファイル命名規則

| ファイル名 | 説明 | 例 |
|-----------|------|-----|
| `+page.svelte` | ページコンポーネント | `src/routes/+page.svelte` |
| `+page.server.ts` | ページのサーバーサイドロジック | `src/routes/+page.server.ts` |
| `+layout.svelte` | レイアウトコンポーネント | `src/routes/+layout.svelte` |
| `+layout.server.ts` | レイアウトのサーバーサイドロジック | `src/routes/+layout.server.ts` |
| `+error.svelte` | エラーページ | `src/routes/+error.svelte` |
| `[id]` | 動的ルートパラメータ | `src/routes/todos/[id]/+page.svelte` |
| `+server.ts` | **APIエンドポイント** | `src/routes/api/todos/+server.ts` |
| `*.client.ts` | クライアントサイド専用 | `src/lib/api/todos.client.ts` |
| `*.server.ts` | サーバーサイド専用 | `src/lib/server/todos.ts` |

### 17.4 インポートパスエイリアス

| エイリアス | 実際のパス | 説明 |
|-----------|----------|------|
| `$lib` | `src/lib` | ライブラリコード |
| `$app` | SvelteKit提供 | SvelteKitのユーティリティ |
| `$env` | SvelteKit提供 | 環境変数（型安全） |

### 17.5 実装後の想定ディレクトリ構成

```
src/
├── app.d.ts
├── app.html
│
├── lib/
│   ├── api/
│   │   ├── fetch.ts              # ✅ 実装
│   │   ├── todos.ts              # ✅ 実装（サーバーサイド用）
│   │   └── todos.client.ts       # ✅ 実装（クライアントサイド用）
│   │
│   ├── assets/
│   │   └── favicon.svg
│   │
│   ├── constants.ts              # ✅ 実装
│   │
│   ├── types/
│   │   ├── todo.ts               # ✅ 実装
│   │   └── api.ts                # ✅ 実装
│   │
│   └── index.ts
│
└── routes/
    ├── +layout.svelte            # ✅ 実装
    ├── +page.svelte              # ✅ 実装
    ├── +page.server.ts           # ✅ 実装
    ├── +error.svelte             # ✅ 実装
    │
    ├── api/                      # ✅ 実装（APIエンドポイント）
    │   └── todos/
    │       ├── +server.ts        # ✅ 実装（GET /api/todos, POST /api/todos）
    │       └── [id]/
    │           └── +server.ts    # ✅ 実装（GET /api/todos/:id, PUT /api/todos/:id, DELETE /api/todos/:id）
    │
    └── todos/
        └── [id]/
            ├── +page.svelte      # ✅ 実装
            └── +page.server.ts   # ✅ 実装
```

---

## 18. まとめ

このTODOアプリは、SvelteKit + TypeScript + REST APIの主要な機能を学習するための最適な教材です。SSR、型安全性、フォーム処理、ルーティング、API連携など、実践的なスキルを身につけることができます。

**重要なポイント**:
- REST APIとの連携方法
- `event.fetch` によるサーバーサイドでのAPI呼び出し
- **ブラウザからの直接API呼び出し**（`todos.client.ts`）
- 型安全なAPIクライアントの実装
- エラーハンドリングの実装
- `type` と `interface` の適切な使い分け
- ファイルベースルーティングの理解

