

  
handleFetch は、SvelteKitのサーバーサイド機能の中でも**BFF（Backend For Frontend）パターンを実現するために最も重要なフック**の一つです。

エンジニア向けに、実行タイミング、トリガー、そして「何が起きているのか」の内部挙動を解像度高く解説します。

---

### 1. handleFetch の概要

**サーバーサイドで実行される fetch リクエスト**をインターセプト（横取り）するミドルウェアです。

- **場所:** src/hooks.server.ts
    
- **役割:** SvelteKitサーバーから外部（Laravelなど）へ飛ぶリクエストの「直前」と「直後」に介入します。
    

---

### 2. トリガー：いつ、何がきっかけで動くのか？

handleFetch が発火するトリガーは、**「SvelteKit サーバー上で fetch 関数が呼ばれた時」** です。  
具体的には、以下の3つのファイル・関数内で fetch を実行した瞬間にトリガーされます。

#### トリガーとなる具体的な場所

1. **+page.server.ts / +layout.server.ts**
    
    - load 関数内でデータを取得するために fetch() した時。
        
    - actions (フォーム送信) 内で API を叩くために fetch() した時。
        
2. **+server.ts (API Routes)**
    
    - GET / POST ハンドラ内で、さらに別の外部APIを叩くために fetch() した時。
        
3. **SSR (Server-Side Rendering) 中**
    
    - .svelte コンポーネントの初期レンダリング中に、サーバー上でデータ取得が必要になった時。
        

#### ⚠️ 重要：トリガーされないケース

- **ブラウザ（クライアントサイド）での fetch**
    
    - .svelte ファイル内の <script> タグで、onMount やボタンクリックイベントで実行される fetch は、**handleFetch を通りません**。
        
    - これらはブラウザから直接ネットワークに出るため、SvelteKitサーバー（Node.js）は関与できないからです。