---
title: "第2章：環境構築とNodeCGの基本構造 〜NodeCGを起動して動かしてみよう〜"
---

前章では、NodeCGを使ってできることや、導入のメリットを紹介しました。
本章では、私が作成した **NodeCGアプリの開発用テンプレート（Vite + React + TypeScript）** を使って、実際に開発環境を構築します。
環境が構築できたら実際に動かしてみて、NodeCGの構造の理解も深めていきます。

-----

:::message
**本書で紹介するテンプレートのリポジトリ**
[nodecg-template-with-vite](https://github.com/bozitoma/nodecg-template-with-vite)
:::
https://github.com/bozitoma/nodecg-template-with-vite/blob/main/README.md

-----

## 1. 開発環境の前提条件
開発を始める前に、以下のツールがインストールされていることを確認してください。

* **Node.js** (推奨: v22以上のLTS版)
* **pnpm** (パッケージマネージャー)
* **Git**
* **コードエディター** (VS Code, Cursor, Antigravityなど何でもOK)
* **OBS Studio** (動作確認用。配信ソフトなら何でもOKですが、解説はOBS Studioで行います)

## 2. テンプレートの技術構成
テンプレートで使用している技術構成を簡単に紹介します。

* **TypeScript**
    * JavaScriptを型安全に記述するために採用しています。このテンプレートではNodeCG独自のメソッド(ReplicantやMessage)にも型定義が簡単にできます。
* **React**
    * 画面（UI）を作るためのライブラリです。コンポーネント指向でパーツを再利用できるため、効率よく開発できます。
* **Vite**
    * ビルドツールです。ReactやTypeScriptで書かれたコードをブラウザが理解できる形に高速変換します。
    * **HMR (Hot Module Replacement)** という機能により、コードを保存した瞬間にブラウザ上の画面が更新されるため、快適に開発が進められます。
* **Zod**
    * TypeScriptファーストなスキーマ宣言・バリデーションライブラリです。Replicant（共有データ）の構造を定義することで、型安全かつ実行時のバリデーションを同時に実現します。
* **ESLint / Prettier**
    * 構文チェックとコードフォーマットの統一を自動で行います。

## 3. 環境構築の手順
それでは、実際に環境を作っていきましょう。

### 3-1. テンプレートをクローンする
ターミナルを開き、任意のディレクトリで以下のコマンドを実行してテンプレートをクローンします。

```bash
git clone git@github.com:bozitoma/nodecg-template-with-vite.git
cd nodecg-template-with-vite
```

### 3-2. 依存関係をインストールする
フォルダを移動したら、必要なライブラリをインストールします。

```bash
pnpm install
```

### 3-3. バンドル名を定義する
NodeCGでは、プロジェクトを識別するための「バンドル名」を定義する必要があります。
このテンプレートでは、バンドル名を `bundleName.ts` というファイルで定義するようにしています。

`bundleName.ts` を開き、値が `package.json` の `name` フィールドと同じであることを確認（または変更）してください。

```typescript:bundleName.ts
// package.json の name と一致させる
export const BUNDLE_NAME = 'nodecg-template-with-vite' as const;
```

:::message
**なぜこの設定が必要？**
NodeCGはバンドル名に基づいて、URLの生成やデータの同期（Replicant）を行います。
この値を間違えると、この後の手順で「Graphicsが表示されない」「エラーが出る」といったトラブルに繋がるため、必ず確認してください。
:::

### 3-4. 開発サーバーを起動する
準備が整ったら、NodeCGを起動します。

```bash
pnpm dev
```

ターミナルに `NodeCG running on http://localhost:9090` のようなログが表示されれば起動成功です。

### 3-5. 動作確認
ブラウザで `http://localhost:9090` にアクセスしてみてください。
NodeCGのDashboard（管理画面）が表示されるはずです。

![テンプレートのDashboard画面](/images/nodecg-react-overlay/02-dashboard-template.png)
*テンプレートのDashboard - テロップ更新とストップウォッチのサンプル機能*

## 4. OBS Studioに映してみよう
次に、このグラフィックをOBS Studioに表示させてみます。

1. NodeCGのDashboardを開き、「Graphics」タブから`EXAMPLE.HTML`の「COPY URL」ボタンをクリックして、URLをコピーします。

![NodeCGのGraphicsタブ](/images/nodecg-react-overlay/02-nodecg-graphics.png)
*NodeCGのGraphicsタブでURLをコピーする*

2. OBS Studioの「ソース」欄の `+` ボタンから **「ブラウザ」** を選択。

![ソースを追加するために+ボタンを選択](/images/nodecg-react-overlay/02-obs-add-source.png)
*ソース欄の+ボタンをクリック*

![ブラウザを選択](/images/nodecg-react-overlay/02-obs-add-browser.png)
*メニューから「ブラウザ」を選択*

2. 新規作成で適当な名前（デフォルトの「ブラウザ」でもOK）をつけてOKボタンを押す。

3. プロパティ画面で以下を設定してOKボタンを押す：
    * **URL**: `http://localhost:9090/bundles/nodecg-template-with-vite/graphics/example.html`
    * **幅**: 1920
    * **高さ**: 1080

![OBS Studioのソース設定](/images/nodecg-react-overlay/02-obs-setting-property.png)
*ブラウザソースのプロパティ設定*

OBS Studioの画面上にグラフィックが表示されればOKです。

![OBS Studioのグラフィック表示](/images/nodecg-react-overlay/02-obs-browser.png)
*OBS Studioにグラフィックが表示された状態*

Dashboard上のテキストボックスを書き換えたり、ストップウォッチのボタンを押してみてください。
OBS Studio上の表示が**リアルタイムに更新**されれば、環境構築は完了です。

![NodeCG Dashboardの操作](/images/nodecg-react-overlay/02-obs-dashboard-update.png)
*Dashboardでテキスト変更やストップウォッチの開始をする*

![OBS Studioの確認](/images/nodecg-react-overlay/02-obs-browser-update.png)
*OBS Studio上の表示がリアルタイムに反映される*

## 5. NodeCGの仕組みを理解しよう
NodeCGの動きを体験できました。次に、その仕組みについて簡単に解説します。
この仕組みを理解しておくと、次章からの開発がスムーズになるはずです。

@[card](https://www.nodecg.dev/ja/docs/concepts-and-terminology)

### 5-1. 構成要素とデータのやり取り

![NodeCGアーキテクチャ図](/images/nodecg-react-overlay/02-nodecg-architecture.png)
*NodeCGの構成要素とデータの流れ[^message-note]*

NodeCGは主に**3つの構成要素**と、それらをつなぐ**2つの通信手段**で成り立っています。

**構成要素:**
1. **Dashboard** - スタッフが操作する管理画面
2. **Graphics** - 視聴者に見せる配信グラフィック
3. **Extension** - サーバーサイドで動くバックエンド

**通信手段:**
- **Replicant** - 各構成要素間で同期される変数
- **Message** - Extensionへ処理を依頼する仕組み

それでは、各要素について詳しく見ていきましょう。

#### Dashboard（ダッシュボード）
**スタッフが操作する管理画面**です。
ブラウザで `http://localhost:9090` にアクセスして表示します。

- テキスト入力フォーム、ボタン、セレクトボックスなど、操作UIを自由に設計可能
- 入力値はReplicantを通じてGraphicsにリアルタイム反映
- 複数の操作パネルを用途別に分割して配置できる

#### Graphics（グラフィックス）
**視聴者に見せる配信グラフィック**です。
OBS Studioなどの配信ソフトでブラウザソースとして読み込ませることで配信に表示します。

- HTMLとCSS（React等）で自由にデザイン可能
- 操作UIは持たせず、「表示専用」として設計するのがベストプラクティス

#### Extension（エクステンション）
**サーバーサイドで動くバックエンド処理**です。
Node.js環境で実行されます。以下のような処理の実装に適しています。

- **ブラウザを閉じても動かしたい処理**（タイマー、定期実行など）
- **外部API呼び出し**（Start.gg、Googleスプレッドシートなど）
- **セキュアな処理**（APIキーなどの機密情報をフロントエンドに露出させない）
- **重い計算処理**（データの集計・加工など）

DashboardやGraphicsはブラウザ上で動くため、これらの処理には限界があります。
Extensionを使えば、Node.jsのエコシステムを活用できます。

#### Replicant（レプリカント）
**Dashboard/Graphics/Extension間で同期される変数**です。
Replicantを更新すると、即座にすべての構成要素で値が反映されます。

- DashboardやExtensionでReplicantを更新 → ブラウザのリロードなしでGraphicsのReplicantも同期して更新
- ブラウザを閉じても値が保持される

このリアルタイム同期とデータの保持という特性がライブ配信にとても適しています。

https://www.nodecg.dev/ja/docs/classes/replicant

#### Message（メッセージ）
**Dashboard/GraphicsからExtensionへ処理を依頼する**仕組みです。

| 依頼内容 | Extension側の処理 |
|----------|-------------------|
| 「タイマーを開始して」 | サーバー側で時間をカウントし、Replicantを更新 |
| 「Start.ggから対戦情報を取得して」 | APIを叩いてデータを取得し、Replicantに保存 |
| 「スプレッドシートを読み込んで」 | Google Sheets APIでデータ取得 |

Messageで処理を依頼し、結果はReplicantで共有する。
この組み合わせが、NodeCGの強力なアーキテクチャを実現しています。

https://www.nodecg.dev/ja/docs/classes/sendMessage/

## 6. おわりに
テンプレートの環境構築と、NodeCGの基本構造について解説しました。
値が即座に更新されるリアルタイム同期と、ブラウザを閉じても値が消えないデータの保持という特性が、ライブ配信にはとても有用です。
これらの仕組みが既に用意されているのが、NodeCGというフレームワークの最大メリットです。

次回は、実際にこのテンプレートを使って開発を進めていきます。

[^message-note]: 技術的にはグラフィックからメッセージを送ることも可能ですが、グラフィックは表示専用とし、操作やロジックの発火はダッシュボードに集約させるのがNodeCG開発のベストプラクティスです。そのため、本書のアーキテクチャ図では意図的に導線を省略しています。
