---
title: "第4章：バックエンド（Extension）の実装 〜タイマーやAPI連携を実装しよう〜"
---

前章では、フロントエンド（Dashboard/Graphics）でスコアボードを作りました。
本章では、NodeCGのバックエンドの領域である **Extension（エクステンション）** を使って、ブラウザに依存しない機能や外部サービスとの連携を実装します。

## 1. Extensionを使うメリット
Extensionは、Node.js環境で動作します。以下のような処理に適しています。

* **タイマー機能**: ブラウザを閉じていても処理を実行したい場合。
* **API連携**: 外部サービス（Start.ggやGoogleスプレッドシート）からデータを取得する場合。
    * CORSエラーの回避できる。
    * APIキーなどを隠蔽できる。

**「表示と操作はフロントエンド、重い処理やロジックはバックエンド」**
このように役割分担することで、効率的で安定した **NodeCGアプリ** になります。

本章では、Extensionの機能を活用して、以下の機能を実装します。

1. **タイマー機能**: ブラウザを閉じても動き続けるタイマー機能
2. **外部API連携**: 外部サイトからデータを取得する機能

これらを通じて、**「フロントエンドからバックエンドへの命令（Message）」** や **「外部サービスとの通信」** を体験していきます。

## 2. Extensionの基本とディレクトリ
本テンプレートでは、`src/extension/index.ts` をエントリーポイントとしてコンパイルされるように構成しています。
そのため、開発時は `src/extension/index.ts` にコードを記述します。
ここに記述された内容はNodeCG起動時に自動的に読み込まれ、実行されます。

```typescript:src/extension/index.ts
import type { NodeCG } from './nodecg';

export default (nodecg: NodeCG) => {
  const log = new nodecg.Logger('extension');
  log.info('=====Extension is running=====');
  
  // ここにバックエンドのロジックを書いていきます
};
```

## 3. Message機能の解説
DashboardからExtensionを呼び出すには、**Message** という機能を使います。

* **Dashboard側**: `sendMessage` で命令を送る（例：「データを取得して！」）
* **Extension側**: `listenFor` で命令を待ち受ける（例：「命令が来たからAPIを叩くよ！」）

https://www.nodecg.dev/ja/docs/classes/sendMessage

https://www.nodecg.dev/ja/docs/classes/listenFor

### 型定義による安全な通信
本テンプレートでは、このやり取りを型安全に行うために `src/nodecg/messages.d.ts` で型を定義しています。
これを定義することで、`sendMessage` や `listenFor` の記述で型エラーがあればエディタが教えてくれるようになります。

```typescript:src/nodecg/messages.d.ts
export type MessageMap = {
  // メッセージ名: { データの型 }
};
```

## 4. タイマー機能の実装
まずはタイマーをExtensionで実装してみましょう。

### 4-1. メッセージの定義
タイマー操作の命令の形を定義します。
機能としては、タイマーの開始、停止、リセットを実装します。
`src/nodecg/messages.d.ts` に `startTimer` などのタイマー操作に必要なメッセージを定義します。

```typescript:src/nodecg/messages.d.ts
export type MessageMap = {
  // ...既存の定義
  
  // タイマー開始
  startTimer: {
    result: { success: boolean; error?: string };
  };
  // タイマー停止
  stopTimer: {
    result: { success: boolean; error?: string };
  };
  // タイマーリセット
  resetTimer: {
    data: number; // リセット後の時間（秒）を指定
    result: { success: boolean; error?: string };
  };
};
```

### 4-2. データ（Replicant）の定義
タイマーの残り時間などを保持するReplicantの型を定義します。
`src/schemas/` に `timer.ts` を作成します。

```typescript:src/schemas/timer.ts
import { z } from 'zod';

export const timerSchema = z
  .object({
    time: z.number().default(60), // 残り時間（秒）
    isRunning: z.boolean().default(false), // 起動中かどうか
  })

export type Timer = z.infer<typeof timerSchema>;
```

作成したスキーマを、`src/schemas/index.ts` からエクスポートするように追加します。

```typescript:src/schemas/index.ts
// ...既存の定義
export { timerSchema } from './timer';
export type { Timer } from './timer';
```

作成した型を `src/nodecg/replicants.d.ts` に追加して、ReplicantMapに登録します。

```typescript:src/nodecg/replicants.d.ts
import type { Alert, Scoreboard, Stopwatch, Timer } from '../schemas'; // Timerを追加

export type ReplicantMap = {
  alert: Alert;
  stopwatch: Stopwatch;
  scoreboard: Scoreboard;
  timer: Timer; // 追加
};
```

### 4-3. Extensionの実装
次に、Extensionのロジックを書きます。
コードが見やすくなるよう、 `src/extension/timer.ts` というファイルを作成して、そこにロジックを記述します。

```typescript:src/extension/timer.ts
import type { NodeCG } from './nodecg';

export function timer(nodecg: NodeCG) {
  const log = new nodecg.Logger('timer');
  
  // Replicantの初期化: サーバー再起動時も値が保持される
  const timerRep = nodecg.Replicant('timer');

  let intervalId: NodeJS.Timeout | null = null;

  // インターバル停止処理（共通化）
  function stopInterval() {
    if (intervalId) {
      clearInterval(intervalId);
      intervalId = null;
    }
  }

  // 1秒ごとに実行される処理
  function tick() {
    const current = timerRep.value;
    // 値がない、または停止中ならインターバルを止める
    if (!current?.isRunning) {
      stopInterval();
      return;
    }

    if (current.time <= 0) {
      // 0になったら停止して完了ログを出す
      stopInterval();
      timerRep.value = { ...current, isRunning: false };
      log.info('Timer finished');
      return;
    }

    // 1秒減らす
    const nextTime = current.time - 1;
    timerRep.value = { ...current, time: nextTime };
    log.info(`${nextTime}s`);
  }

  // インターバル開始処理（二重起動防止）
  function startInterval() {
    if (intervalId) return;
    intervalId = setInterval(tick, 1000);
  }

  // 起動時にタイマーが勝手に動作しないようにする
  if (timerRep.value?.isRunning) {
    log.info('Timer was running on startup, resetting status to stopped.');
    timerRep.value = { ...timerRep.value, isRunning: false };
  }

  // タイマー開始メッセージ
  nodecg.listenFor('startTimer', (_, ack) => {
    const current = timerRep.value;
    
    // 既に動いている、または値がない、または残り時間が0以下の場合はエラーを返す
    if (current?.isRunning || !current || current.time <= 0) {
      if (!ack?.handled) {
        ack?.(null, { success: false, error: 'Cannot start timer' });
      }
      return;
    }

    // 現在の値を保持しつつ、isRunningをtrueにしてインターバルを開始
    timerRep.value = { ...current, isRunning: true };
    startInterval();
    log.info(`Timer started at: ${current.time}s`);

    // 成功応答
    if (!ack?.handled) {
      ack?.(null, { success: true });
    }
  });

  // タイマー停止メッセージ
  nodecg.listenFor('stopTimer', (_, ack) => {
    stopInterval();
    if (timerRep.value) {
      timerRep.value = { ...timerRep.value, isRunning: false };
    }
    log.info('Timer stopped');
    
    if (!ack?.handled) {
      ack?.(null, { success: true });
    }
  });

  // タイマーリセットメッセージ
  nodecg.listenFor('resetTimer', (duration, ack) => {
    stopInterval();
    // 停止状態で、新しい開始時間をセットする
    timerRep.value = { time: duration, isRunning: false };
    log.info(`Timer reset to: ${duration}s`);

    if (!ack?.handled) {
      ack?.(null, { success: true });
    }
  });
}
```

### 4-4. Extensionの登録
作成した `timer.ts` をエントリーポイントである `src/extension/index.ts` で読み込みます。

```typescript:src/extension/index.ts
import type { NodeCG } from './nodecg';
import { timer } from './timer'; // 追加

export default (nodecg: NodeCG) => {
  // ...既存の処理
  
  // タイマー機能を有効化
  timer(nodecg);
};
```

これでバックエンドの実装は完了です。
続いて、フロントエンド（DashboardとGraphics）からタイマーを使えるようにしましょう。

### 4-5. NodeCGの設定（DashboardとGraphics）
まずは `package.json` で、今回使用するDashboardパネルとGraphicsの設定を追加します。
Dashboardには新しいパネル「Timer」を追加し、Graphicsにもタイマー用のページを追加します。

また、今回のダッシュボードパネルは `workspace` プロパティを指定して、「Extension」というワークスペース（タブ）の中に表示させてみます。
第3章のようにページをフルに使って表示することもできますが、このようにワークスペース内でパネルを区切って管理することもできます。

```json:package.json
"nodecg": {
  "compatibleRange": "^2.0.0",
  "dashboardPanels": [
    // ...既存のパネル
    {
      "name": "timer",
      "title": "Timer",
      "file": "timer.html",
      "width": 3,
      "workspace": "Extension"
    }
  ],
  "graphics": [
    // ...既存のグラフィックス
    {
      "file": "timer.html",
      "width": 1920,
      "height": 1080
    }
  ]
},
```

### 4-6. Dashboardの実装
タイマーを操作するためのDashboardを作成します。
ここでは、タイマーの時間を入力するState管理と、バックエンド（Extension）へメッセージを送信する処理を実装します。

`src/browser/dashboard/timer/` ディレクトリを作成し、以下のファイルを配置します。

```css:src/browser/dashboard/timer/style.css
.container {
  padding: 20px;
  color: white;
  display: flex;
  flex-direction: column;
  gap: 16px;
}
.input {
  padding: 8px;
  border-radius: 4px;
  border: 1px solid #444;
  background: rgba(0, 0, 0, 0.5);
  color: white;
  width: 100%;
}
.button-group {
  display: flex;
  gap: 8px;
}
.button {
  flex: 1;
  padding: 8px;
  cursor: pointer;
  border-radius: 4px;
  border: none;
  font-weight: bold;
}
.start { background: #4caf50; color: white; }
.stop { background: #f44336; color: white; }
.reset { background: #666; color: white; }
```

```tsx:src/browser/dashboard/timer/App.tsx
import { useState } from 'react';
import './style.css';

export function App() {
  const [duration, setDuration] = useState(60);

  // タイマー開始
  const handleStart = async () => {
    try {
      // payloadなしで開始（現在の残り時間を使用）
      await nodecg.sendMessage('startTimer');
    } catch (error) {
      console.error('Failed to start timer:', error);
    }
  };
  
  // タイマー停止
  const handleStop = async () => {
    try {
      await nodecg.sendMessage('stopTimer');
    } catch (error) {
      console.error('Failed to stop timer:', error);
    }
  };

  // タイマーリセット
  const handleReset = async () => {
    try {
      // 現在入力されているフォームの値をセット
      await nodecg.sendMessage('resetTimer', duration);
    } catch (error) {
      console.error('Failed to reset timer:', error);
    }
  };

  return (
    <div className="container">
      {/* 時間設定フォーム */}
      <div>
        <label>時間（秒）</label>
        <input
          type="number"
          className="input"
          value={duration}
          onChange={(e) => setDuration(Number(e.target.value))}
        />
      </div>

      {/* 操作ボタン */}
      <div className="button-group">
        <button className="button start" onClick={handleStart}>開始</button>
        <button className="button stop" onClick={handleStop}>停止</button>
        <button className="button reset" onClick={handleReset}>リセット</button>
      </div>
    </div>
  );
}
```

```tsx:src/browser/dashboard/timer/index.tsx
import '@/browser/global.css';
import { createRoot } from 'react-dom/client';
import { App } from './App';

const root = createRoot(document.getElementById('root')!);
root.render(<App />);
```

### 4-7. Graphicsの実装
次に、タイマーを表示するGraphicsを作成します。
ExtensionでReplicantの値を更新しているので、Graphicsの値もリアルタイムに更新されます。

`src/browser/graphics/timer/` ディレクトリを作成し、以下のファイルを配置します。

```css:src/browser/graphics/timer/style.css
.container {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100vh;
  background-color: transparent;
}

.timer-box {
  background-color: rgba(0, 0, 0, 0.8);
  padding: 20px 40px;
  border-radius: 8px;
  color: white;
  font-weight: bold;
  font-family: monospace;
  font-size: 64px;
  border: 2px solid rgba(255, 255, 255, 0.2);
}
```

```tsx:src/browser/graphics/timer/App.tsx
import { useReplicant } from '../../hooks/useReplicant';
import './style.css';

// 秒数を mm:ss 形式に変換するヘルパー関数
const formatTime = (seconds: number) => {
  const m = Math.floor(seconds / 60).toString().padStart(2, '0');
  const s = (seconds % 60).toString().padStart(2, '0');
  return `${m}:${s}`;
};

export function App() {
  const [timer] = useReplicant('timer');

  return (
    <div className="container">
      {timer && (
        <div className="timer-box">
          {formatTime(timer.time)}
        </div>
      )}
    </div>
  );
}
```

```tsx:src/browser/graphics/timer/index.tsx
import '@/browser/global.css';
import { createRoot } from 'react-dom/client';
import { App } from './App';

const root = createRoot(document.getElementById('root')!);
root.render(<App />);
```

### 動作確認

NodeCGを再起動して動作を確認します。
既にNodeCGを起動している場合は、`Ctnl + C` でNodeCGを停止してから、`pnpm dev` で再起動してください。
Dashboardでタイマーを操作すると、Graphics上のタイマーも連動して動くようになります。

![タイマー機能 Dashboard](/images/nodecg-react-overlay/04-nodecg-dashboard-timer.png)
*タイマー機能 Dashboard*

![タイマー機能 Graphics](/images/nodecg-react-overlay/04-nodecg-graphics-timer.png)
*タイマー機能 Graphics*

また、バックエンドでReplicantを更新しているので、ダッシュボードのブラウザを閉じても、タイマーは動き続けます。
実際にブラウザを閉じてみても、バックエンドのログは動作し続けます。

```
2026-02-07 13:30:28 - info: [timer] Timer started at: 60s
2026-02-07 13:30:29 - info: [timer] 59s
2026-02-07 13:30:30 - info: [timer] 58s
2026-02-07 13:30:31 - info: [timer] 57s
2026-02-07 13:30:32 - info: [timer] 56s
```

このように、**バックエンド（Extension）でReplicantを更新することで、フロントエンド（Dashboard/Graphics）でも同期されたReplicantの値を使うことができる** のがNodeCGの強力な点です。



## 5. 外部APIとの連携
もう一つのExtensionの使用パターンとして、APIからデータを取得する機能を実装しましょう。

Node.jsなので、`fetch` 関数を使ってAPIを利用できます。
ここでは例として、ダミーデータを提供してくれるサイト[JSONPlaceholder](https://jsonplaceholder.typicode.com/)から「コメントデータ」を取得してみます。

:::details JSONPlaceholderとは？
プロトタイピングやテストのために、無料で使えるフェイク（ダミー）のREST APIサービスです。
本物のサーバーを用意しなくても、HTTPリクエストの挙動を手軽に確認・テストできます。

**主な特徴:**
- 🆓 **登録不要**: APIキーなしですぐに利用可能
- 📦 **豊富なリソース**: ToDo、記事、ユーザーなどのダミーデータが用意されています
- 🚀 **REST準拠**: GET, POST, PUT, DELETEなどの主要なメソッドに対応

公式サイト: [JSONPlaceholder](https://jsonplaceholder.typicode.com/)
:::

### 5-1. メッセージの定義
`fetchExternalData` というメッセージを定義します。
今回は「どのコメントを取得するか」を指定できるように、コメントのIDを送れるようにします。

```typescript:src/nodecg/messages.d.ts
import type { Comment } from '../schemas';

export type MessageMap = {
  // ...
  fetchExternalData: {
    data: { id: number };
    result: { success: boolean; data?: CommentData; error?: string };
  };
};
```

### 5-2. データ（Replicant）の定義
JSONPlaceholderから取得したコメントデータをReplicantに保存するために、`src/schemas/comment.ts`を作成します。
データから `name` (投稿者)と`body` (本文) を抽出して保存します。

```typescript:src/schemas/comment.ts
import { z } from 'zod';

export const commentSchema = z
  .object({
    id: z.number().default(0),
    name: z.string().default('投稿者名'),
    body: z.string().default('ここにコメントが表示されます'),
  })

export type CommentData = z.infer<typeof commentSchema>;
```

作成したスキーマを、`src/schemas/index.ts` に追記します。

```typescript:src/schemas/index.ts
// ...
export { commentSchema } from './comment';
export type { CommentData } from './comment';
```

作成したスキーマを `src/nodecg/replicants.d.ts` に追加します。

```typescript:src/nodecg/replicants.d.ts
import type { Alert, Scoreboard, Stopwatch, Timer, CommentData } from '../schemas';

export type ReplicantMap = {
  // ...
  comment: CommentData;
};
```

### 5-3. Extensionの実装
`src/extension/comment.ts` を作成します。
送られてきたIDを使ってJSONPlaceholderから特定のコメントデータを取得し、Replicantに保存してから、結果をDashboardに返します。

```typescript:src/extension/comment.ts
import type { NodeCG } from './nodecg';
import type { CommentData } from '../schemas';

export function comment(nodecg: NodeCG) {
  const log = new nodecg.Logger('comment');
  const commentRep = nodecg.Replicant('comment');

  nodecg.listenFor('fetchExternalData', async (payload, ack) => {
    try {
      const { id } = payload;
      log.info(`Fetching comment for ID: ${id}...`);
      
      const response = await fetch(`https://jsonplaceholder.typicode.com/comments/${id}`);
      if (!response.ok) {
        throw new Error(`API Error: ${response.statusText}`);
      }

      const data = await response.json();
      log.info('Data fetched:', data);

      // Replicantを更新
      const sanitizedData: CommentData = {
        id: data.id,
        name: data.name,
        body: data.body,
      };
      commentRep.value = sanitizedData;

      if (ack && !ack.handled) {
        ack(null, { success: true, data: sanitizedData });
      }
    } catch (err) {
      log.error('Failed to fetch data:', err);
      if (ack && !ack.handled) {
        ack(null, { success: false, error: (err as Error).message });
      }
    }
  });
}
```

`src/extension/index.ts` で `comment.ts` を読み込みます。

```typescript:src/extension/index.ts
import { comment } from './comment'; // 追加

export default (nodecg: NodeCG) => {
  // ...
  comment(nodecg); // 追加
};
```

これでバックエンドの準備が整いました。

### 5-4. NodeCGの設定（DashboardとGraphics）
Dashboardからコメント取得機能を呼び出すためのパネルを作成します。
`package.json` に **dashboardPanels** と **graphics** の両方にコメント用のファイルを登録します。

```json:package.json 
"dashboardPanels": [
  // ...
  {
    "name": "comment",
    "title": "Comment",
    "file": "comment.html",
    "width": 3,
    "workspace": "Extension"
  }
],
"graphics": [
  // ...
  {
    "file": "comment.html",
    "width": 1920,
    "height": 1080
  }
]
```

### 5-5. Dashboardの実装
APIのIDを指定する入力フォームと、実行ボタンを持つシンプルなDashboardを作成します。
`src/browser/dashboard/comment/` ディレクトリを作成し、以下のファイルを配置します。

```css:src/browser/dashboard/comment/style.css
.container {
  padding: 24px;
  color: #ffffff;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 20px;
}
.form-group {
  display: flex;
  align-items: center;
  gap: 10px;
  width: 100%;
  max-width: 400px;
}
.form-group input {
  flex: 1;
  padding: 8px;
  border-radius: 4px;
  border: 1px solid rgba(255, 255, 255, 0.2);
  background: rgba(0, 0, 0, 0.4);
  color: white;
}
.button {
  padding: 12px;
  width: 100%;
  max-width: 400px;
  cursor: pointer;
  background: white;
  color: black;
  border: none;
  border-radius: 6px;
  font-weight: bold;
}
.result {
  width: 100%;
  max-width: 400px;
  background: rgba(0, 0, 0, 0.5);
  padding: 16px;
  border-radius: 8px;
  white-space: pre-wrap;
  font-family: monospace;
  font-size: 12px;
  box-sizing: border-box;
}
```

```tsx:src/browser/dashboard/comment/App.tsx
import { useState } from 'react';
import './style.css';

export function App() {
  const [result, setResult] = useState<string>('');
  const [commentId, setCommentId] = useState(1);

  const handleFetch = async () => {
    try {
      setResult('取得中...');
      // IDを送信してコメントを取得
      const response = await nodecg.sendMessage('fetchExternalData', { id: commentId });
      if (response.success && response.data) {
        setResult(JSON.stringify(response.data, null, 2));
      } else {
        setResult(`エラー: ${response.error}`);
      }
    } catch (error) {
      setResult(`エラー: ${(error as Error).message}`);
    }
  };

  return (
    <div className="container">
      <div className="form-group">
        <label>コメントID:</label>
        <input 
          type="number" 
          value={commentId} 
          onChange={(e) => setCommentId(Number(e.target.value))}
          min={1}
        />
      </div>

      <button className="button" onClick={handleFetch}>
        コメントを取得
      </button>
      
      <div className="result-container">
        <div className="label">取得したコメント</div>
        <div className="result">
          {result || 'コメントはまだありません'}
        </div>
      </div>
    </div>
  );
}
```

```tsx:src/browser/dashboard/comment/index.tsx
import '@/browser/global.css';
import { createRoot } from 'react-dom/client';
import { App } from './App';

const root = createRoot(document.getElementById('root')!);
root.render(<App />);
```

### 5-6. Graphicsの実装 (取得したコメントの表示)
取得したコメントをGraphicsに表示します。
ExtensionでReplicantに保存しているので、GraphicsではそのReplicantを監視するだけです。

`src/browser/graphics/comment/` ディレクトリを作成し、以下のファイルを配置します。

```css:src/browser/graphics/comment/style.css
.container {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100vh;
}

.card {
  background: white;
  padding: 24px;
  border-radius: 12px;
  box-shadow: 0 4px 20px rgba(0,0,0,0.5);
  font-family: sans-serif;
  width: 600px;
  text-align: left;
}

.header {
  display: flex;
  flex-direction: column;
  margin-bottom: 12px;
  border-bottom: 1px solid #eee;
  padding-bottom: 12px;
}

.author {
  font-weight: bold;
  font-size: 18px;
  color: #333;
}

.body {
  font-size: 16px;
  line-height: 1.5;
  color: #444;
}

.empty {
  color: white;
  font-size: 24px;
  font-weight: bold;
}
```

```tsx:src/browser/graphics/comment/App.tsx
import { useReplicant } from '../../hooks/useReplicant';
import './style.css';

export function App() {
  const [comment] = useReplicant('comment');

  return (
    <div className="container">
      {comment ? (
        <div className="card">
          <div className="header">
            <span className="author">{comment.name}</span>
          </div>
          <div className="body">
            {comment.body}
          </div>
        </div>
      ) : (
        <div className="empty">コメント待機中...</div>
      )}
    </div>
  );
}
```

```tsx:src/browser/graphics/comment/index.tsx
import '@/browser/global.css';
import { createRoot } from 'react-dom/client';
import { App } from './App';

const root = createRoot(document.getElementById('root')!);
root.render(<App />);
```

### 動作確認

NodeCGを再起動して動作を確認します。
既にNodeCGを起動している場合は、`Ctnl + C` でNodeCGを停止してから、`pnpm dev` で再起動してください。
Dashboardからコメントを取得すると、GraphicsにもAPIで取得したコメントが表示されます。
IDを変えてコメントを取得し、Graphicsに表示される内容が変わることも確認しましょう。

![コメント Dashboard](/images/nodecg-react-overlay/04-nodecg-dashboard-comment.png)
*コメント Dashboard*

![コメント Graphics](/images/nodecg-react-overlay/04-nodecg-graphics-comment.png)
*コメント Graphics*

## 6. Bundle Configで設定を管理

今のコードでは、APIのURLをソースコードの中に直接書いています。
この状態だと、別のAPIに繋ぎたい場合はコード内のURLを書き換えてビルドし直す必要があります。
他にも、認証が必要なAPIを使う場合、APIキーなどの秘密情報をコードに書くのはセキュリティ上も好ましくありません。

NodeCGには、こうした設定値をコードから分離して管理する**Bundle Configuration**という仕組みがあります。

https://www.nodecg.dev/ja/docs/bundle-configuration

Web開発では`.env`ファイル（環境変数）を使うのが一般的ですが、NodeCGではこのBundle Configurationが標準です。
Bundle Configurationには以下のようなメリットがあります。

*   **どこからでも使える**: バックエンド（Extension）だけでなく、フロントエンド（Dashboard/Graphics）からも、追加の設定なしで同じ値を取得できます。
*   **ビルド不要**: 設定値を変えるためにコードを再ビルドする必要はありません。JSONファイルを書き換えてNodeCGを再起動するだけで反映されます。

最後に、この機能を使ってコードをリファクタリングしてみましょう。

### 6-1. 設定の型定義 (Schema)

本テンプレートでは、**Zodを使ってBundle Configurationの型定義を行っています。**
これにより、`ts-nodecg` を通じて `nodecg.bundleConfig` の型が自動的に推論され、TypeScript上で安全に設定値を扱えるようになります。

今回はシンプルに、接続先のURLを設定できるようにします。
`src/schemas/bundleConfig.ts` を以下のように編集します。

```typescript:src/schemas/bundleConfig.ts
import { z } from "zod";

export const bundleConfigSchema = z.object({
  apiUrl: z.string().default(''),
});

export type BundleConfig = z.infer<typeof bundleConfigSchema>;
```

### 6-2. 設定ファイルの作成
次に、実際の設定値を記述するファイルを作成します。

**重要なルールとして、設定ファイルのファイル名はバンドル名と一致させる必要があります。**
バンドル名は `package.json` の `name` フィールドに記述されている名前です。今回は `nodecg-template-with-vite` なので、ファイル名は `cfg/nodecg-template-with-vite.json` となります。

`cfg/nodecg-template-with-vite.json` を作成して、APIのURLを記述します。

```json:cfg/nodecg-template-with-vite.json
{
  "apiUrl": "https://jsonplaceholder.typicode.com/comments"
}
```

### 6-3. Extensionの実装変更
最後に、Extensionのコード (`src/extension/comment.ts`) を変更して、設定ファイルの値を使うするようにします。
`nodecg.bundleConfig` から、定義した設定値に型安全にアクセスできます。

```diff:src/extension/comment.ts
  nodecg.listenFor('fetchExternalData', async (payload, ack) => {
    try {
-     const response = await fetch(`https://jsonplaceholder.typicode.com/comments/${id}`);
+     // Bundle Config から設定値を取得して使用する
+     const { apiUrl } = nodecg.bundleConfig;
+     const response = await fetch(`${apiUrl}/${id}`);
```


### 動作確認
NodeCGを再起動しても先程と同じようにコメントが取得できるか確認してみます。問題なく動作していればOKです。
これで、コードを書き換えなくても、`cfg/nodecg-template-with-vite.json` を書き換えてNodeCGを再起動するだけで、接続先を変更できるようになりました。
今回はURLだけですが、APIキーやアクセストークンなどの **秘密情報や環境変数を管理する場所** としてBundle Configは有用です。

## おわりに
この章では、Extensionを使って**タイマー機能**と**外部データ取得（コメント取得）機能**を実装しました。

Extensionを使えば、「バックエンドならではの処理」が可能になります。色々と応用が効きますので、ぜひ活用してみてください。

次章では、NodeCGで作成したアプリの運用方法について解説します。