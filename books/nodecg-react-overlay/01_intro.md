---
title: "第1章：導入とNodeCGの概要 〜NodeCGって何？〜"
---

はじめまして、Webエンジニアの**ぼじ**です。
普段は個人でゲームイベントを開催したり、配信周りの技術を触ったりしています。

これまで、[NodeCG](https://www.nodecg.dev/ja/)というフレームワークを使って、**配信用のグラフィックを制御するアプリケーション**をいくつか開発してきました。
本書では、自分用に作成した**NodeCG開発テンプレート（Vite + React + TypeScript）を公開**し、ハンズオン形式でNodeCGを体系的に学べるように解説していきます。

NodeCGについて知りたい人、自作で配信グラフィックのアプリケーションを作りたい人は、ぜひ参考にしてください。

-----

:::message
**本書で紹介するテンプレートのリポジトリ**
[nodecg-template-with-vite](https://github.com/bozitoma/nodecg-template-with-vite)
:::
https://github.com/bozitoma/nodecg-template-with-vite/blob/main/README.md#L1-L23

-----

## 1. NodeCGとは？
[NodeCG](https://www.nodecg.dev/ja/)は、Node.jsとブラウザをベースに、**ライブ配信用のグラフィック（映像に重ねる動的なコンテンツ）と操作パネルの開発に特化したフレームワーク**です。


![NodeCG公式サイト](/images/nodecg-react-overlay/01-nodecg-official-site.png)
*[NodeCG公式サイト](https://www.nodecg.dev/ja/)*

[OBS Studio](https://obsproject.com/ja)のような配信ソフト単体では実装が難しい、グラフィックのリアルタイムな制御を **Webアプリケーション**として開発できます。
例えば、以下のような要素を**ブラウザ上の管理画面からリアルタイムに制御**することが可能です。
- ⚔️ 対戦中のスコア管理
- ⌛️ イベントのタイムスケジュールの更新
- 🖼️ 画像の切り替え

## 2. NodeCGの採用事例
実際にNodeCGが使われているライブ配信の事例をいくつか紹介します。

### 2-1. RTA in Japan
[RTA in Japan](https://rtain.jp/)は、日本で開催されている、ゲームを最速でクリアする遊び方「リアルタイムアタック（RTA）」の大規模イベントです。
RTAにはタイマー機能が必須です。また、複数のプレイヤーとゲームが次々と進行していくため、タイムスケジュールやプレイヤー情報の更新を柔軟に行う必要があります。

NodeCGを使えば、プレイヤー名、解説者名、SNSアカウント情報、そして現在の進行状況などをブラウザの管理画面から配信画面に反映させることができます。

![タイムスケジュールの画面](/images/nodecg-react-overlay/01-rtaij-layout-schedule.png)
*RTA in Japan Winter 2025 スケジュール画面（[アーカイブ 36:10:44〜](https://www.twitch.tv/videos/2655180893?t=36h10m44s)）*

![ゲーム画面](/images/nodecg-react-overlay/01-rtaij-layout-game.png)
*RTA in Japan Winter 2025 ゲーム画面 - 走者や解説者、タイマー、ゲーム名、レギュレーションなどを管理（[アーカイブ 36:23:41〜](https://www.twitch.tv/videos/2655180893?t=36h23m41s)）*

RTA in JapanのNodeCGアプリケーションはGitHubで公開されています。
@[card](https://github.com/RTAinJapan/rtainjapan-layouts)

### 2-2. 若人杯（ストリートファイター6）

「ストリートファイター6」を用いた2名1チームで行われる団体戦トーナメントです。
この大会では、チーム紹介や使用キャラクターの表示、現在のスコアや勝敗状況をNodeCGで管理しています。

チームごとのプレイヤー名や使用キャラクター、一言紹介文を表示したりと、団体戦ならではの複雑な情報管理にもNodeCGは適しています。

![チーム紹介画面](/images/nodecg-react-overlay/01-wakoudo-intro.png)
*チーム紹介画面 - チーム名、プレイヤー名、一言コメント、使用キャラクターを表示（[アーカイブ 10:37:48〜](https://www.youtube.com/live/JiCNf3tt0bo?si=SZ24KYT4WSal0_Oz&t=38268)）*

![試合画面](/images/nodecg-react-overlay/01-wakoudo-game.png)
*試合画面 - スコアや勝敗を管理（[アーカイブ 10:44:33〜](https://www.youtube.com/live/JiCNf3tt0bo?si=xhT-0uTjqAPp4YB4&t=38673)）*

### 2-3. VGBootCamp（日本語配信）
配信クルー「[VGBootCamp](https://www.twitch.tv/vgbootcamp)」の日本語配信用に、NodeCGベースのスコアボード「**OverlayCast**」を開発しました。

VGBootCampでは複数のゲームタイトルを配信するため、タイトルごとに異なるレイアウトを表示する必要があります。
NodeCGを活用することで、入力項目や表示内容をゲームに応じて切り替えています。

また、参加者が1000人を超える大規模大会の配信もあります。
対戦カードを迅速に反映させるため、トーナメントを管理する[Start.gg API](https://developer.start.gg/docs/intro/)と連携して対戦情報を自動取得する機能も実装しています。

![OverlayCast ダッシュボード（スマブラSP）](/images/nodecg-react-overlay/01-vgbootcamp-dashboard.png)
*OverlayCast ダッシュボード（スマブラSP） - 基本的なスコアボード機能とStart.gg API連携*

![VGBootCamp 配信画面（スマブラSP）](/images/nodecg-react-overlay/01-vgbootcamp-battle.png)
*VGBootCamp 配信画面（スマブラSP）（[アーカイブ 4:32:38〜](https://www.twitch.tv/videos/2608360921?t=04h32m38s)）*

![OverlayCast ダッシュボード（スマブラDX）](/images/nodecg-react-overlay/01-vgbootcamp-dashboard-melee.png)
*OverlayCast ダッシュボード（スマブラDX） - ゲームタイトルに応じて入力フォームを切り替え*

![VGBootCamp 配信画面（スマブラDX）](/images/nodecg-react-overlay/01-vgbootcamp-battle-melee.png)
*VGBootCamp 配信画面（スマブラDX） - キャラクターアイコンやポート情報を表示*

### 2-4. TSKaigi（テックカンファレンス）

TypeScriptに関するあらゆるテーマを扱う国内最大級のカンファレンスです。このライブ配信基盤にもNodeCGが採用されていました（[制作者による技術解説](https://zenn.dev/ken7253/articles/tskaigi-streaming-layout)）。

登壇者名や講演タイトル、スポンサーのロゴ切り替え表示などがNodeCGによって管理されています。ゲーム配信だけでなく、こうしたテックカンファレンスでも採用されています。

![TSKaigi配信画面](/images/nodecg-react-overlay/01-tskaigi-session.png)
*TSKaigi 2024 配信画面 - 登壇者情報やスポンサーロゴを管理（[アーカイブ 16:34〜](https://www.youtube.com/live/fbAykLlOTZ4?si=K3VZaUngt5zLM6fe&t=994)）*

## 3. 既存ツールとの比較
スコアやプレイヤー名の管理であれば、[BraceBracket](https://bracebracket.app/)という有用な既存ツールがあり、標準的な用途であればこちらで十分に役割を果たせます。

@[card](https://bracebracket.app/)

では、あえてNodeCGで開発する利点は何か、今一度整理してみます。

### 3-1. 独自の情報を制御したい
既存ツールでは、原則として表示できるレイアウトや項目が決まっています。
「もっと多くのテキストを表示したい」「ライブ配信の状況に合わせて画像を切り替えたい」といった、既存ツールの枠に収まらない独自の制御を行いたい場合、NodeCGであれば実現が可能です。

### 3-2. 運営フローに合わせた機能を作れる
例えばRTA in Japanでは、ボランティアスタッフがオペレーションを行うため、「入力項目にミスがないか」を判定するチェックリスト機能が実装されています。
このように、「全ての項目がOKにならないと画面更新ボタンが押せないようにする」といった、自分たちのイベント運営に特化した安全装置やロジックも組み込めます。

![チェックリスト機能](/images/nodecg-react-overlay/01-rtaij-dashboard-stream.png)
*ダッシュボードの左下にチェックリスト機能 - 出典: [RTA in JapanのNodeCGレイアウトを動かしてみる](https://qiita.com/pasta04/items/d676d9c2fb716176f665)*

### 3-3. 外部APIやデータベースを利用できる
Node.js（バックエンド）が使えるため、外部サービスとの連携が可能です。

* Start.gg（トーナメントサイト）のAPIを利用して対戦カードを自動取得する
* Googleスプレッドシートで管理している参加者リストを読み込む
* 過去の対戦データから「勝率」を計算して表示する

このように、手動入力を減らして自動化したり、データを加工してそのまま表示することが可能です。

### 3-4. デザインを自由にできる
配信画面のデザインはもちろん、スタッフが操作する管理画面のデザインも、使いやすいように自由にカスタマイズできます。

![OverlayCast ダッシュボード（スマブラSP）](/images/nodecg-react-overlay/01-vgbootcamp-dashboard.png)
*自作のNodeCGアプリ「OverlayCast」のダッシュボードのデザイン - VGBootCampのロゴデザインに合わせて赤と黒を基調にしたデザインにしています*

## 4. おわりに
この章では、NodeCGの概要と採用事例、そして開発するメリットを紹介しました。

次章では、テンプレートを使って実際に開発環境を構築して、NodeCGを動かしてみます。

-----

:::message
**本書で紹介するテンプレートのリポジトリ**
[nodecg-template-with-vite](https://github.com/bozitoma/nodecg-template-with-vite)
:::
https://github.com/bozitoma/nodecg-template-with-vite/blob/main/README.md#L1-L23

-----