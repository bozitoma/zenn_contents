---
title: "第3章：フロントエンド(Dashboard/Graphics)の実装 〜スコアボードを表示しよう〜"
---

前章ではテンプレートの環境構築が完了しました。
本章では、NodeCGのフロントエンドの領域である**Dashboard**と**Graphics**を使って、対戦ゲームなどで使用する**スコアボード**を作成します。
NodeCG開発の基本となる、DashboardとGraphicsを連携させる仕組みを、ReactとTypeScriptを使って実装していきます。

## 1. 今回作るもの

対戦ゲームの配信でよく見かける**スコアボード**を作成します。

**管理する情報：**
- Player 1 の名前とスコア
- Player 2 の名前とスコア

**実現する機能：**
- Dashboardから各項目を入力・編集できる
- 「更新」ボタンを押すと、配信画面にリアルタイム反映される
- 入力値を初期化するリセットボタン
- Player 1/Player 2を入れ替えるスワップボタン

![Dashboard 完成イメージ](/images/nodecg-react-overlay/03-dashboard-complete.png)
*Dashboard 完成イメージ*

![Graphics 完成イメージ](/images/nodecg-react-overlay/03-graphic-complete.png)
*Graphics 完成イメージ(背景は透過です)*

---

具体的には、以下の流れで実装します。

1. Zodスキーマによるデータ定義
2. Replicantのデータ定義
3. Dashboard（操作画面）の実装
4. Graphics（表示画面）の実装
5. NodeCGへの登録（package.json）
6. 動作確認 / OBSへの組み込み

## 2. Zodスキーマによるデータ定義

このスコアボードで扱うデータを定義します。
NodeCGでは **Replicant（レプリカント）** という仕組みを使って、DashboardとGraphicsの間でデータを同期します。
本テンプレートでは、**Zod** というライブラリを使って、Replicantのデータの型とデフォルト値を安全に管理しています。

### スキーマファイルの作成
`src/schemas/scoreboard.ts` を作成します。
今回のスコアボードでは2人のプレイヤーの名前とスコアを管理するので、以下のように記述します。

```typescript:src/schemas/scoreboard.ts
import { z } from 'zod';

const playerSchema = z
  .object({
    name: z.string().default('Player Name'),
    score: z.number().default(0),
  })

export const scoreboardSchema = z
  .object({
    player1: playerSchema,
    player2: playerSchema,
  })

export type Scoreboard = z.infer<typeof scoreboardSchema>;
```

:::details 使用しているZodのメソッド解説
簡単にですが、Zodのメソッドについて説明します。
以下のように記述することで、データの型とデフォルト値を安全に管理できます。

| メソッド | 説明 |
|---------|------|
| `z.object({...})` | オブジェクト型のスキーマを定義 |
| `z.string()` | 文字列型のスキーマ |
| `z.number()` | 数値型のスキーマ |
| `.default(value)` | 値が`undefined`の場合に使用するデフォルト値を設定 |
| `z.infer<typeof schema>` | スキーマからTypeScript型を抽出 |

[Defining schemas | Zod](https://zod.dev/api)

:::

### エクスポートの追加

`src/schemas/index.ts` にエクスポートを追加します。

```typescript:src/schemas/index.ts
export { scoreboardSchema } from './scoreboard';
export type { Scoreboard } from './scoreboard';
```

## 3. Replicantのデータ定義

データの型を作成したので、次はそれをNodeCGに反映します。

### Replicant（レプリカント）の仕組み
![NodeCGアーキテクチャ図](/images/nodecg-react-overlay/02-nodecg-architecture.png)
*NodeCGの構成要素とデータの流れ[^message-note]*

NodeCGには **Replicant（レプリカント）** という、**NodeCG全体で値を共有する仕組み**があります。
Replicantを更新すると、**即座にDashboard/Graphics/Extension間で値が同期**されます。
また、Replicantは内部的にデータベース（SQLite）で保存されているため、**ブラウザを閉じたり、NodeCGを再起動したりしても値が保持され続けます**。

この「リアルタイム同期」と「データの保持」という特性を活用することで、Dashboardで編集した内容を即座にGraphicsへ反映する、ライブ配信に最適なシステムを構築できます。

https://www.nodecg.dev/ja/docs/classes/replicant/

### useReplicantフック
本テンプレートでは、このReplicantをReactの `useState` と同じ感覚で扱えるようにしたカスタムフック `useReplicant` を提供しています。

このフックは、**Replicantの同期された値をReactのUIに反映させるためのラッパー**です。
Replicantの値はNodeCGによって自動的に同期されますが、Reactがそれを検知して再描画するには、`change` イベントを利用して `useState` を更新する必要があります。
`useReplicant` はその処理をすべて隠蔽しているため、開発者は値の読み書きだけに集中できます。

```tsx:useReplicantの例
const [value, setValue] = useReplicant('replicantName');
// value: undefined（初期ロード中）
// value: 'default value'（Replicantの値が同期された後）

// 値を更新
setValue('new value');
// value: 'new value'（更新後）
```

この `useReplicant` フックは、後述する `ReplicantMap` という全体マップの型定義と連動しています。
引数に Replicant 名（例：`'scoreboard'`）を渡すだけで、その名前に対応する型を自動的に判別します。
これにより、**「どの Replicant にどんなデータが入っているか」をエディターが常に把握できる状態になり、入力補完を最大限に活用しながら安全に開発を進めることができます。**

:::details useReplicantの実装を見る
`src/browser/hooks/useReplicant.ts` の中身は以下のようになっています。

```typescript:src/browser/hooks/useReplicant.ts
import { useCallback, useEffect, useState } from 'react';
import type { ReplicantMap } from '../../nodecg/replicants';

export const useReplicant = <T extends keyof ReplicantMap>(
  name: T
): [ReplicantMap[T] | undefined, (newValue: ReplicantMap[T]) => void] => {
  const [rep] = useState(() => nodecg.Replicant(name));
  const [value, setValue] = useState<ReplicantMap[T] | undefined>(undefined);

  useEffect(() => {
    const handleChange = (newValue: ReplicantMap[T]) => setValue(newValue);
    rep.on('change', handleChange);
    return () => {
      rep.removeListener('change', handleChange);
    };
  }, [rep]);

  return [value, useCallback((newValue) => (rep.value = newValue), [rep])];
};
```

**ポイント:**
- `useState(() => nodecg.Replicant(name))` でReplicantインスタンスを保持
- `useEffect` 内で `change` イベントを購読し、`setValue` でReactの状態を更新
- クリーンアップ関数でリスナーを解除（メモリリーク防止）
- 戻り値の第2要素で `rep.value` への代入をラップ
:::

### 型定義の追加（ReplicantMap）

`useReplicant` の型を推論するために、先程スキーマで定義した型定義を`src/nodecg/replicants.d.ts` に追加します。
この `ReplicantMap` は、Replicantの「名前」と「データの型」を紐付けるための定義です。

```typescript:src/nodecg/replicants.d.ts
import type { Alert, Stopwatch, Scoreboard } from '../schemas';

export type ReplicantMap = {
  alert: Alert;
  stopwatch: Stopwatch;
  scoreboard: Scoreboard;  // 追加
};
```

## 4. NodeCGのセットアップ

データの定義が整いました。
続いて、NodeCGを起動するためのセットアップを済ませます。

本テンプレートでは、`src/browser` ディレクトリ内でDashboardとGraphicsのコードを管理しています。
まずは、NodeCGのビルドと起動ができるところまでファイルを準備していきます。

### 作業ディレクトリの作成
今回は `scoreboard` という名前でディレクトリを作成して、その中にファイルを作成していきます。
`src/browser/dashboard/` ディレクトリと`src/browser/graphics/` ディレクトリに、`scoreboard`ディレクトリを作成します。

```bash
mkdir src/browser/dashboard/scoreboard
mkdir src/browser/graphics/scoreboard
```

### ファイルの作成
本テンプレートでは、`src/browser/dashboard/` と`src/browser/graphics/` で作成されたディレクトリ内の`index.tsx`ファイルを読み込んでビルドします。
ビルドファイルはそれぞれルートの`Dashboard`ディレクトリと`Graphics`ディレクトリにhtmlファイルが生成されます。ここに生成することでNodeCGがファイルを読み込めるようになります。
ビルドが通るように、ベースとなるファイルを作成します。

#### メインコンポーネントファイルの作成
実際のコンポーネントを定義するファイルです。
この中でコンポーネントやロジックを書きます。
```tsx:src/browser/dashboard/scoreboard/App.tsx
export function ScoreboardDashboard() {
  return (
    <div>
      <h1>Hello Dashboard</h1>
    </div>
  );
}
```

```tsx:src/browser/graphics/scoreboard/App.tsx
export function ScoreboardGraphic() {
  return (
    <div>
      Hello Graphics
    </div>
  );
}
```

#### エントリーファイルの作成
ビルド時のエントリーファイルです。
`createRoot` でHTML内の描画エリア（`#root`）を取得して、`root.render` で実装したコンポーネントを配置しています。
また、ブラウザによる表示のズレを防ぐため、テンプレートに含まれる `src/browser/global.css` をここで読み込んでおきます。

```tsx:src/browser/dashboard/scoreboard/index.tsx
import '@/browser/global.css';
import { createRoot } from 'react-dom/client';
import { ScoreboardDashboard } from './App';

const root = createRoot(document.getElementById('root')!);
root.render(<ScoreboardDashboard />);
```

```tsx:src/browser/graphics/scoreboard/index.tsx
import '@/browser/global.css';
import { createRoot } from 'react-dom/client';
import { ScoreboardGraphic } from './App';

const root = createRoot(document.getElementById('root')!);
root.render(<ScoreboardGraphic />);
```

### package.jsonのnodecgセクションを編集

作成したファイルをNodeCGに認識させるため、`package.json` の `nodecg` セクションを編集します。
`dashboardPanels`（ダッシュボード）と `graphics`（グラフィックス）の両方を設定します。

```json:package.json
{
  "nodecg": {
    "compatibleRange": "^2.0.0",
    "dashboardPanels": [
      {
        "name": "scoreboard",
        "title": "Scoreboard",
        "file": "scoreboard.html",
        "fullbleed": true
      }
    ],
    "graphics": [
      {
        "file": "scoreboard.html",
        "width": 1920,
        "height": 1080
      }
    ]
  }
}
```

:::details 設定項目の詳細（dashboardPanels / graphics）

#### dashboardPanels

| プロパティ | 必須 | 説明 |
|-----------|:---:|------|
| `name` | ✅ | パネルの内部名。一意である必要があります |
| `title` | ✅ | NodeCGダッシュボードに表示されるタイトル |
| `file` | ✅ | HTMLファイルへの相対パス（バンドルのルートから） |
| `width` | - | パネルの幅（1〜8の範囲で指定） |
| `workspace` | - | パネルを配置するワークスペース名 |
| `fullbleed` | - | `true`にすると専用ワークスペースで最大サイズ表示 |
| `dialog` | - | `true`にすると他のパネルから開けるダイアログになる |

参考:
- [NodeCG公式ドキュメント - Concepts and Terminology (Dashboard)](https://www.nodecg.dev/ja/docs/concepts-and-terminology#dashboard)
- [NodeCG公式ドキュメント - Manifest (dashboardPanels)](https://www.nodecg.dev/ja/docs/manifest#dashboard-panels)

#### graphics

| プロパティ | 必須 | 説明 |
|-----------|:---:|------|
| `file` | ✅ | HTMLファイルへの相対パス（バンドルのルートから） |
| `width` | ✅ | グラフィックスの幅（ピクセル） |
| `height` | ✅ | グラフィックスの高さ（ピクセル） |
| `singleInstance` | - | `true`にすると同時に1つしか開けなくなる |

参考:
- [NodeCG公式ドキュメント - Concepts and Terminology (Graphics)](https://www.nodecg.dev/ja/docs/concepts-and-terminology#graphics)
- [NodeCG公式ドキュメント - Manifest (graphics)](https://www.nodecg.dev/ja/docs/manifest#graphics)
:::

### 起動確認

準備ができたら、NodeCGを起動して、両方の画面が表示されるか確認しましょう。

```bash
pnpm dev
```

1. **Dashboard**: `http://localhost:9090` にアクセスし、「Scoreboard」パネルに「Hello Dashboard」と表示されているか確認。

![Dashboard 初期表示確認](/images/nodecg-react-overlay/03-nodecg-dashboard-hello.png)
*Dashboard 初期表示確認*


2. **Graphics**: Dashboard上のリンク、または `http://localhost:9090/bundles/nodecg-template-with-vite/graphics/scoreboard.html` にアクセスし、「Hello Graphic」と表示されているか確認。

![NodeCG の Graphics タブ](/images/nodecg-react-overlay/03-nodecg-graphics-list.png)
*NodeCG の Graphics タブ*

![Graphics 初期表示確認](/images/nodecg-react-overlay/03-nodecg-graphics-hello.png)
*Graphics 初期表示確認*


両方確認できたらセットアップ完了です。
ここからは、この画面をベースに機能を実装していきます。
Viteで開発しているので、起動したまま編集をすると自動で編集内容がブラウザに反映されます。

## 5. Dashboardの実装

まずは操作画面であるDashboardから実装します。
先ほど作った「Hello Dashboard」の状態の `src/browser/dashboard/scoreboard/App.tsx` を書き換えていきます。

### Replicantの呼び出し

`useReplicant` フックを使って `scoreboard` の Replicant を読み込みます。
実際の運用を想定した時に、フォームの設計は **「Replicantの値」と「手元の編集用ステート」を分ける** のがおすすめです。
Replicantを直接更新してしまうと、フォームの値を更新するたびに配信画面のReplicantの値も変わってしまいます。
そこで、`localData` というローカルステートを用意し、「更新」ボタンを押したタイミングで初めてReplicantに反映する方式にします。
そうすることで、**「値を確定してから配信画面に反映する」** というフローを実現できます。

加えて、**Replicantの値を初回ロード時のみローカルステートに反映させる**と便利です。
ページをリロードした時やNodeCGを再起動した時に、現在のReplicantの値をフォームに反映できます。
ただし、Replicantが更新されるたびにローカルステートも更新されてしまうと、例えば複数人で管理する場合に勝手にフォームの値が書き換わってしまいます。
そこで `useRef` を使って「初回ロード済みかどうか」を管理し、初回のみ反映するようにしています。

このような設計にすることで、複数人でも管理しやすくなります。

```tsx:src/browser/dashboard/scoreboard/App.tsx
import { useState, useEffect, useCallback, useRef } from 'react';
import { useReplicant } from '../../hooks/useReplicant';
import type { Scoreboard } from '../../../schemas/scoreboard';

// 初期値の定義
const INITIAL_PLAYER = { name: '', score: 0 };
const INITIAL_DATA: Scoreboard = {
  player1: INITIAL_PLAYER,
  player2: INITIAL_PLAYER,
};

export function ScoreboardDashboard() {
  // 1. Replicantのデータを取得・同期
  const [scoreboard, setScoreboard] = useReplicant('scoreboard');

  // 2. Dashboard編集用のローカルステート
  // 入力中の値を一時的に保持します（確定するまでReplicantには反映しない）
  const [localData, setLocalData] = useState(INITIAL_DATA);

  // 初回ロード完了フラグ
  const initialized = useRef(false);

  // 初回のみReplicantの値をローカルステートに反映
  useEffect(() => {
    if (!initialized.current && scoreboard) {
      setLocalData(scoreboard);
      initialized.current = true;
    }
  }, [scoreboard]);

  // とりあえずコンテナだけ返しておく
  return <div className="container">ToDo</div>; 
}
```

### 更新ボタンの導入

ローカルの変更をReplicantに反映する「更新」ボタンを実装します。

```tsx:src/browser/dashboard/scoreboard/App.tsx

// ... (これまでのコード)

export function ScoreboardDashboard() {
  // ... (これまでのコード)

  // ローカルの編集内容をReplicantに反映
  const handleUpdate = useCallback(() => {
    setScoreboard(localData);
  }, [localData, setScoreboard]);

  return (
    <div className="container">
      <div className="button-group">
        <button className="update-button" onClick={handleUpdate}>
          更新
        </button>
      </div>
    </div>
  );
}
```

### PlayerCardコンポーネントの導入

Player1とPlayer2の編集画面は共通しているので、コンポーネントで管理します。
`src/browser/dashboard/scoreboard/PlayerCard.tsx` を作成して、以下のコードを記述してください。

```tsx:src/browser/dashboard/scoreboard/PlayerCard.tsx
import { memo } from 'react';

// プレイヤー情報の型定義
type PlayerData = {
  name: string;
  score: number;
};

// コンポーネントが受け取るPropsの型定義
type PlayerCardProps = {
  label: string;       // "Player 1" などの表示ラベル
  player: PlayerData;  // 名前とスコアのデータ
  onNameChange: (e: React.ChangeEvent<HTMLInputElement>) => void; // 名前変更時の処理
  onDecrement: () => void; // －ボタンを押した時の処理
  onIncrement: () => void; // ＋ボタンを押した時の処理
};

// プレイヤー1人分の操作パネルを表示するコンポーネント
// memo化して、関係ないプレイヤーの更新による再レンダリングを防ぎます
export const PlayerCard = memo(function PlayerCard({
  label,
  player,
  onNameChange,
  onDecrement,
  onIncrement,
}: PlayerCardProps) {
  return (
    <div className="player-card">
      <span className="section-title">{label}</span>
      
      {/* 名前入力エリア */}
      <input
        type="text"
        value={player.name}
        onChange={onNameChange}
        className="input"
        placeholder="プレイヤー名"
      />
      
      {/* スコア表示・操作エリア */}
      <div className="score-card">
        <span className="score-label">スコア</span>
        <span className="score-display">{player.score}</span>
        <div className="score-button-row">
          <button className="button" onClick={onDecrement}>
            −
          </button>
          <button className="button" onClick={onIncrement}>
            +
          </button>
        </div>
      </div>
    </div>
  );
});
```
`src/browser/dashboard/scoreboard/App.tsx`に先ほど作成した `PlayerCard` コンポーネントを2つ配置します。
それぞれのプレイヤー名変更やスコア増減のイベントハンドラを作成し、`localData` を更新するようにします。

```tsx:src/browser/dashboard/scoreboard/App.tsx
import { PlayerCard } from './PlayerCard'; // コンポーネントの読み込み

// ... (これまでのコード)

export function ScoreboardDashboard() {
  // ... (これまでのコード)

  // スコア更新の共通処理
  const updateScore = useCallback(
    (player: keyof Scoreboard, delta: number) => {
      setLocalData((prev) => ({
        ...prev,
        [player]: {
          ...prev[player],
          score: Math.max(0, prev[player].score + delta), // スコアがマイナスにならないように制御
        },
      }));
    },
    []
  );

  // 名前変更の共通処理
  const updateName = useCallback(
    (player: keyof Scoreboard, name: string) => {
      setLocalData((prev) => ({
        ...prev,
        [player]: { ...prev[player], name },
      }));
    },
    []
  );

  // 各PlayerCardに渡すハンドラを生成
  const handleP1NameChange = useCallback(
    (e: React.ChangeEvent<HTMLInputElement>) => updateName('player1', e.target.value),
    [updateName]
  );
  const handleP2NameChange = useCallback(
    (e: React.ChangeEvent<HTMLInputElement>) => updateName('player2', e.target.value),
    [updateName]
  );

  // スコア増減ハンドラ
  const decrementP1 = useCallback(() => updateScore('player1', -1), [updateScore]);
  const incrementP1 = useCallback(() => updateScore('player1', 1), [updateScore]);
  const decrementP2 = useCallback(() => updateScore('player2', -1), [updateScore]);
  const incrementP2 = useCallback(() => updateScore('player2', 1), [updateScore]);

  return (
    <div className="container">
      {/* 追加: プレイヤー情報表示エリア */}
      <div className="players-row">
        <PlayerCard
          label="Player 1"
          player={localData.player1}
          onNameChange={handleP1NameChange}
          onDecrement={decrementP1}
          onIncrement={incrementP1}
        />
        <PlayerCard
          label="Player 2"
          player={localData.player2}
          onNameChange={handleP2NameChange}
          onDecrement={decrementP2}
          onIncrement={incrementP2}
        />
      </div>
      
      <div className="button-group">
        {/* 更新ボタン */}
        <button className="update-button" onClick={handleUpdate}>
          更新
        </button>
      </div>

    </div>
  );
}
```

### リセット・入れ替えボタンの導入

便利機能として、初期状態に戻す「リセット」ボタンと、Player 1/Player 2を入れ替える「スワップ」ボタンを追加します。

```tsx:src/browser/dashboard/scoreboard/App.tsx

// ... (これまでのコード)

export function ScoreboardDashboard() {
  // ... (これまでのコード)

  // 「リセット」ボタン: 初期値に戻して即座に反映
  const handleReset = useCallback(() => {
    setLocalData(INITIAL_DATA);
    setScoreboard(INITIAL_DATA);
  }, [setScoreboard]);

  // 「入れ替え」ボタン: Player1とPlayer2の情報を交換する
  const handleSwap = useCallback(() => {
    setLocalData((prev) => ({
      ...prev,
      player1: prev.player2,
      player2: prev.player1,
    }));
  }, []);

  return (
    <div className="container">
      {/* ... (PlayerCards) ... */}
      <div className="button-group">
        <button className="update-button" onClick={handleUpdate}>
          更新
        </button>
        
        {/* 追加: 入れ替えボタン */}
        <button className="swap-button" onClick={handleSwap}>
          入れ替え
        </button>

        {/* 追加: リセットボタン */}
        <button className="reset-button" onClick={handleReset}>
          リセット
        </button>
      </div>
    </div>
  );
}
```

### スタイルの実装

スタイルを定義します。
`src/browser/dashboard/scoreboard/style.css` を作成して、以下のコードを記述してください。

```css:src/browser/dashboard/scoreboard/style.css
.container {
  min-height: 100vh;
  padding: 24px;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto,
    'Helvetica Neue', Arial, sans-serif;
  background-color: transparent;
  color: #ffffff;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 20px;
}

.players-row {
  display: flex;
  gap: 16px;
  width: 100%;
  max-width: 700px;
}

.player-card {
  flex: 1;
  background-color: rgba(255, 255, 255, 0.03);
  border-radius: 12px;
  border: 1px solid rgba(255, 255, 255, 0.1);
  padding: 20px;
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.section-title {
  font-size: 18px;
  font-weight: 700;
  color: #ffffff;
}

.input {
  padding: 12px 16px;
  font-size: 16px;
  background-color: rgba(0, 0, 0, 0.6);
  border: 1px solid rgba(255, 255, 255, 0.15);
  border-radius: 6px;
  color: white;
  outline: none;
  width: 100%;
  box-sizing: border-box;
}

.score-card {
  background-color: rgba(0, 0, 0, 0.4);
  border-radius: 8px;
  padding: 16px;
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.score-label {
  font-size: 12px;
  font-weight: 600;
  color: #888888;
  text-align: center;
}

.score-display {
  font-size: 48px;
  font-weight: 700;
  color: #ffffff;
  text-align: center;
  line-height: 1;
}

.score-button-row {
  display: flex;
  gap: 8px;
}

.button {
  flex: 1;
  padding: 10px;
  font-size: 18px;
  font-weight: 600;
  background-color: rgba(255, 255, 255, 0.1);
  color: #ffffff;
  border: 1px solid rgba(255, 255, 255, 0.2);
  border-radius: 6px;
  cursor: pointer;
}

.button-group {
  display: flex;
  gap: 12px;
  width: 100%;
  max-width: 700px;
}

.update-button {
  flex: 1;
  padding: 14px 24px;
  font-size: 16px;
  font-weight: 600;
  background-color: #ffffff;
  color: #000000;
  border: none;
  border-radius: 6px;
  cursor: pointer;
}

.swap-button {
  padding: 14px 24px;
  font-size: 16px;
  font-weight: 600;
  background-color: rgba(255, 255, 255, 0.1);
  border: 1px solid rgba(255, 255, 255, 0.3);
  color: #ffffff;
  border-radius: 6px;
  cursor: pointer;
}

.reset-button {
  padding: 14px 24px;
  font-size: 16px;
  font-weight: 600;
  background-color: rgba(255, 50, 50, 0.1);
  border: 1px solid rgba(255, 50, 50, 0.3);
  color: #ff4444;
  border-radius: 6px;
  cursor: pointer;
}
```

最後に、作成したCSSファイルを `App.tsx` で読み込みます。

```tsx:src/browser/dashboard/scoreboard/App.tsx
import { useState, useEffect, useCallback, useRef } from 'react';
import { useReplicant } from '../../hooks/useReplicant';
import type { Scoreboard } from '../../../schemas/scoreboard';
import { PlayerCard } from './PlayerCard';
import './style.css'; // 追加
```

---

完成版のコードは以下の通りです。

```tsx:src/browser/dashboard/scoreboard/App.tsx
import { useState, useEffect, useCallback, useRef } from 'react';
import { useReplicant } from '../../hooks/useReplicant';
import type { Scoreboard } from '../../../schemas/scoreboard';
import { PlayerCard } from './PlayerCard';
import './style.css';

// 初期値の定義（ローカルステートの初期化用）
const INITIAL_PLAYER = { name: '', score: 0 };
const INITIAL_DATA: Scoreboard = {
  player1: INITIAL_PLAYER,
  player2: INITIAL_PLAYER,
};

export function ScoreboardDashboard() {
  // 1. Replicantのデータを取得・同期
  const [scoreboard, setScoreboard] = useReplicant('scoreboard');

  // 2. ダッシュボード編集用のローカルステート
  // 入力中の値を一時的に保持します（確定するまでReplicantには反映しない）
  const [localData, setLocalData] = useState(INITIAL_DATA);

  // 初回ロード完了フラグ
  const initialized = useRef(false);

  // 初回のみReplicantの値をローカルステートに反映
  useEffect(() => {
    if (!initialized.current && scoreboard) {
      setLocalData(scoreboard);
      initialized.current = true;
    }
  }, [scoreboard]);

  // 汎用的なスコア更新処理
  const updateScore = useCallback(
    (player: keyof Scoreboard, delta: number) => {
      setLocalData((prev) => ({
        ...prev,
        [player]: {
          ...prev[player],
          score: Math.max(0, prev[player].score + delta), // スコアがマイナスにならないように制御
        },
      }));
    },
    []
  );

  // 「更新」ボタン: ローカルの編集内容をReplicantに書き込む
  const handleUpdate = useCallback(() => {
    setScoreboard(localData);
  }, [localData, setScoreboard]);

  // 「リセット」ボタン: 初期値に戻して即座に反映
  const handleReset = useCallback(() => {
    setLocalData(INITIAL_DATA);
    setScoreboard(INITIAL_DATA);
  }, [setScoreboard]);

  // 「入れ替え」ボタン: Player1とPlayer2の情報を交換する
  const handleSwap = useCallback(() => {
    setLocalData((prev) => ({
      ...prev,
      player1: prev.player2,
      player2: prev.player1,
    }));
  }, []);

  // 名前変更の共通処理
  const updateName = useCallback(
    (player: keyof Scoreboard, name: string) => {
      setLocalData((prev) => ({
        ...prev,
        [player]: { ...prev[player], name },
      }));
    },
    []
  );

  // プレイヤー名の入力ハンドラ
  const handleP1NameChange = useCallback(
    (e: React.ChangeEvent<HTMLInputElement>) => updateName('player1', e.target.value),
    [updateName]
  );
  const handleP2NameChange = useCallback(
    (e: React.ChangeEvent<HTMLInputElement>) => updateName('player2', e.target.value),
    [updateName]
  );

  // コンポーネントに渡すための関数群を作成
  const decrementP1 = useCallback(() => updateScore('player1', -1), [updateScore]);
  const incrementP1 = useCallback(() => updateScore('player1', 1), [updateScore]);
  const decrementP2 = useCallback(() => updateScore('player2', -1), [updateScore]);
  const incrementP2 = useCallback(() => updateScore('player2', 1), [updateScore]);

  // 4. 画面描画
  return (
    <div className="container">
      <div className="players-row">
        {/* Player 1 のカード */}
        <PlayerCard
          label="Player 1"
          player={localData.player1}
          onNameChange={handleP1NameChange}
          onDecrement={decrementP1}
          onIncrement={incrementP1}
        />
        
        {/* Player 2 のカード */}
        <PlayerCard
          label="Player 2"
          player={localData.player2}
          onNameChange={handleP2NameChange}
          onDecrement={decrementP2}
          onIncrement={incrementP2}
        />
      </div>

      <div className="button-group">
        <button className="update-button" onClick={handleUpdate}>
          更新
        </button>
        <button className="swap-button" onClick={handleSwap}>
          入れ替え
        </button>
        <button className="reset-button" onClick={handleReset}>
          リセット
        </button>
      </div>
    </div>
  );
}
```

これでダッシュボードの実装は完了です。
ブラウザでダッシュボード（`http://localhost:9090`）を確認すると、スタイルが適用され、プレイヤー名の入力やスコアの操作ができるようになっているはずです。
NodeCGのDashboard上で実際に動かして、動作を確認してみてください。

![Dashboard 完成イメージ](/images/nodecg-react-overlay/03-dashboard-complete.png)
*Dashboard 完成イメージ*

## 6. Graphicsの実装

続いて、表示画面（Graphics）を実装します。

### メインコンポーネントの実装

`src/browser/graphics/scoreboard/App.tsx` を更新して、Replicantの値を表示するようにします。
Replicantの仕様上、ページ読み込み直後は必ず `undefined` が返されるため、その場合の処理も記述します。
Graphicsは表示専用にするのがベストで、`useReplicant`からも値だけ取得するようにします。

```tsx:src/browser/graphics/scoreboard/App.tsx
import { useReplicant } from '../../hooks/useReplicant';
import './style.css';

export function ScoreboardGraphic() {
  const [scoreboard] = useReplicant('scoreboard');

  // データ同期前は何も表示しない
  if (!scoreboard) return null;

  return (
    <div className="scoreboard-container">
      {/* スコアボード本体 */}
      <div className="scoreboard">
        {/* Player 1 */}
        <div className="player player-1">
          <span className="player-name">{scoreboard.player1.name}</span>
          <span className="player-score">{scoreboard.player1.score}</span>
        </div>

        {/* VS */}
        <div className="vs">VS</div>

        {/* Player 2 */}
        <div className="player player-2">
          <span className="player-score">{scoreboard.player2.score}</span>
          <span className="player-name">{scoreboard.player2.name}</span>
        </div>
      </div>
    </div>
  );
}
```

### スタイルの実装

Graphicsのスタイルを実装します。
`src/browser/graphics/scoreboard/style.css` を作成し、以下のコードを記述してください。

```css:src/browser/graphics/scoreboard/style.css
.scoreboard-container {
  width: 100vw;
  height: 100vh;
  background-color: transparent;
  
  /* 中央上部に配置 */
  display: flex;
  justify-content: center;
  align-items: flex-start;
  padding-top: 50px;

  font-family: 'Helvetica Neue', Arial, sans-serif;
}

.scoreboard {
  display: flex;
  align-items: center;
  gap: 20px;
  background-color: rgba(0, 0, 0, 0.7);
  padding: 16px 32px;
  border-radius: 12px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.5);
}

.player {
  display: flex;
  align-items: center;
  gap: 16px;
}

.player-name {
  font-size: 28px;
  font-weight: 600;
  color: #ffffff;
  min-width: 200px;
}

.player-1 .player-name {
  text-align: right;
}

.player-2 .player-name {
  text-align: left;
}

.player-score {
  font-size: 48px;
  font-weight: 800;
  color: #ffffff;
  background-color: rgba(255, 255, 255, 0.1);
  padding: 8px 20px;
  border-radius: 8px;
  min-width: 60px;
  text-align: center;
}

.vs {
  font-size: 20px;
  font-weight: 700;
  color: #888888;
}
```

最後に、作成したCSSファイルを `App.tsx` で読み込みます。

```tsx:src/browser/graphics/scoreboard/App.tsx
import { useReplicant } from '../../hooks/useReplicant';
import './style.css'; // 追加
```

これでGraphicsの実装は完了です。
`http://localhost:9090/bundles/nodecg-template-with-vite/graphics/scoreboard.html` にアクセスし、ブラウザで表示してみてください。

![Graphics 完成イメージ](/images/nodecg-react-overlay/03-graphic-complete.png)
*Graphics 完成イメージ(背景は透過です)*

## 7. 動作確認とOBSへの取り込み

NodeCGの実装は完了したので、実際にOBSに読み込ませてみましょう。

### OBSへの設定

1. OBSの「ソース」欄の `+` ボタンから **「ブラウザ」** を選択。

![ソースを追加するために+ボタンを選択](/images/nodecg-react-overlay/02-obs-add-source.png)
*ソース欄の+ボタンをクリック*

![ブラウザを選択](/images/nodecg-react-overlay/02-obs-add-browser.png)
*メニューから「ブラウザ」を選択*

2. 新規作成で適当な名前（デフォルトの「ブラウザ」でもOK）をつけてOKボタンを押す。

3. プロパティ画面で以下を設定してOKボタンを押す：
    * **URL**: `http://localhost:9090/bundles/nodecg-template-with-vite/graphics/scoreboard.html`
        * ※NodeCGのDashboardの「Graphics」タブからURLを確認・コピーできます。
    * **幅**: 1920
    * **高さ**: 1080

![OBSのソース設定](/images/nodecg-react-overlay/03-obs-setting-property.png)
*ブラウザソースのプロパティ設定*

OBSの画面上にオーバーレイが表示されればOKです。

![OBSのオーバーレイ表示](/images/nodecg-react-overlay/03-obs-browser.png)
*OBSにオーバーレイが表示された状態(背景の透過をわかりやすくするために、背景にDashboardの画面を載せています)*

Dashboard側で名前やスコアを更新してみてください。
OBSの画面も更新されていれば、実装が完了です。

![Dashboardの更新後のオーバーレイ](/images/nodecg-react-overlay/03-obs-updated-scoreboard.png)
*Dashboardを更新してOBSの画面も更新されていれば成功*

## おわりに

これで、NodeCGを使った基本的な実装フロー（データ定義 → 入力作成 → 表示作成 → OBSへの取り込み）を一通り体験しました。

今回作成したのは単純なテキストと数値の同期だけですが、これを応用すれば画像の切り替えやアニメーションのON/OFFなども実現可能です。

次回はNodeCGのバックエンド処理を担うExtensionについて解説します。
今回の実装内容だけでもアプリとしては十分機能しますが、Extensionを使うことで、ブラウザを閉じても止まらないタイマーや、外部APIとの連携など、より高度な機能実装が可能になります。さらにNodeCGの機能を活用したい方はぜひご覧ください。

[^message-note]: 技術的にはグラフィックからメッセージを送ることも可能ですが、グラフィックは表示専用とし、操作やロジックの発火はダッシュボードに集約させるのがNodeCG開発のベストプラクティスです。そのため、本書のアーキテクチャ図では意図的に導線を省略しています。