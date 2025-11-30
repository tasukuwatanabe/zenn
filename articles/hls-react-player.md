---
title: "HLSストリーミングをreact-playerで実装する"
emoji: "🎥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['react', 'typescript']
published: false
publication_name: "hrbrain"
---

## はじめに

こんにちは。HRBrainで学習管理サービス[「HRBrain ラーニング」](https://www.hrbrain.jp/lms)を開発している渡邉です。

最近、動画コンテンツの配信機能をHLS（HTTP Live Streaming）で実装しましたが、想像以上に苦戦しました。「ライブラリを使えばすぐできるだろう」と思っていたら、Cookie認証が効かなかったり、状態管理で詰まったり。結局、公式ドキュメントだけでは分からない実装の工夫がいくつも必要でした。

この記事では、そんな試行錯誤の末にたどり着いた「react-player」を使ったHLS動画プレーヤーの実装方法を書いていきます。バージョン3系での状態管理の方法やCookie認証の対応方法などについて紹介します。

なお、この記事ではReactのフロントエンド実装だけを扱います。HLS動画ファイルの生成方法やサーバー側の配信設定については触れません。

https://github.com/cookpete/react-player

## HLSとは

HLS（HTTP Live Streaming）は、動画配信の仕組みのひとつです。

通常、動画ファイルを配信しようとすると、ファイルサイズが大きく読み込みに時間がかかります。HLSは動画を数秒単位の小さなファイル（セグメント）に分割し、視聴者が見ている部分から順番に配信します。これにより、最初の読み込みを待たずに再生を始められます。

さらにHLSは、ネットワークの速度に合わせて自動で画質を変えてくれる点も便利です。WiFiでも4G回線でも、それぞれの環境で快適に視聴できます。YouTubeやNetflixも、HLSや似たような技術（DASH）を使っています。

## なぜreact-playerを選んだか

HLS動画を実装する際、使用できそうなライブラリを調査したところ、以下の3つが候補に上がりました。

- **[hls.js](https://github.com/video-dev/hls.js)**: HLS再生の本格的なライブラリ。GitHubスター数も多く、細かい制御ができる
- **[react-player](https://github.com/cookpete/react-player)**: hls.jsをラップし、ReactコンポーネントとしてHLS機能を提供するライブラリ
- **[video.js](https://github.com/videojs/video.js)**: 老舗の動画プレーヤー。npmダウンロード数も多く、プラグインが豊富

npm trendsで人気を比較すると、人気傾向は以下のようになっています。

![](/images/hls-react-player/npm-trends.png)

どれも人気があって実績のあるライブラリですが、最終的にreact-playerを選んだのは、次の3つの理由からです。

### 1. Reactコンポーネントとしてビデオライブラリを提供している

react-playerは裏側で`hls.js`を使っています。`hls.js`は確かに強力なライブラリですが、それを直接Reactで使おうとすると、さまざまな実装が必要になります。

例えば以下のような実装が必要です。

- コンポーネントのマウント時にHLSインスタンスを初期化
- アンマウント時に適切にクリーンアップ
- video要素への参照を管理
- エラーハンドリングの実装
- イベントリスナーの登録と解除

こういったReactアプリケーションとの統合部分は、react-playerがすべて面倒を見てくれます。自分で実装すると地味に大変で、しかもバグが混入しやすい部分です。

### 2. 必要な機能が揃っている

動画プレーヤーに必要な基本機能が、react-playerにはすべて揃っています。

- **再生速度の変更**: `playbackRate`プロパティで0.5倍速から2倍速まで対応
- **音量調整**: `volume`と`muted`プロパティで細かく制御可能
- **全画面表示**: 標準のHTML要素として全画面表示が可能

厳密には、react-playerは内部でHTMLの`<video>`要素をラップしており、これらの設定は`video`要素が持つ`playbackRate`、`volume`、`muted`といったプロパティや`requestFullscreen()`のようなメソッドを通じて制御されています。プロパティを渡すだけで、動画プレーヤーに必要な機能を簡単に利用できます。

https://developer.mozilla.org/ja/docs/Web/HTML/Reference/Elements/video

### 3. 高頻度でメンテナンスが行われている

react-playerは更新頻度も高く、適切にメンテナンスされているライブラリです。issueへの対応も比較的早く、実績も十分だと判断しました。

## react-playerを動かす

ここからは、実際に私たちのプロジェクトで使っている実装をベースに、具体的な使い方を見ていきます。

まずはインストールから。

```bash
npm install react-player
# または
yarn add react-player
```

一番シンプルな実装例は以下のようになります。

```tsx
import ReactPlayer from 'react-player'

type Props = {
    src: string
}

export const VideoPlayer = (props: Props) => {
    return (
        <ReactPlayer
            src={props.src}
            width="100%"
            height="100%"
            controls={true}
        />
    )
}
```

`controls={true}`を設定すると、ブラウザ標準のvideo要素のコントロールUIがそのまま表示されます。

![](/images/hls-react-player/hls-ui-basic.png)

### HLSの再生の仕組み

`src`プロパティには、HLSのマニフェストファイル(`.m3u8`)のURLを指定します。

マニフェストファイルには以下のような情報が記載されています。

- 分割された動画セグメントファイル(`.ts`)のリスト
- それぞれのセグメントの再生時間
- 利用可能な画質の情報

react-playerは、このマニフェストファイルを読み込んだ後、記載されているセグメントファイルを順番に取得していきます。「HLSとは」のセクションで説明したように、セグメントファイルが連続的にダウンロードされることで、動画がストリーミング再生される仕組みです。

Networkタブで通信を可視化すると、`.ts`拡張子のファイル（セグメントファイル）が順番にダウンロードされていることが分かります。

![](/images/hls-react-player/hls-streaming.png)

### 再生設定を管理する

react-playerでは、再生速度やボリュームなどの状態を親コンポーネント側で管理し、それをプロパティとして渡す「Controlled Components」パターンを採用しています。プレーヤーコンポーネント自体は内部で状態を保持せず、外部から渡された値に従って動作します。

以下は、再生速度とボリュームを状態管理する実装例です。

```tsx
import { useState } from 'react'
import ReactPlayer from 'react-player'

type Props = {
    src: string
}

export const VideoPlayer = (props: Props) => {
    // 再生設定をReactPlayerコンポーネント外で管理して渡す
    const [settings, setSettings] = useState({
        playbackRate: 1.0,  // 再生速度
        volume: 1.0,        // ボリューム
        muted: false        // ミュート
    })

    // 再生速度変更ハンドラー
    const handleRateChange = (event: React.SyntheticEvent<HTMLVideoElement>) => {
        setSettings(prev => ({
            ...prev,
            playbackRate: event.currentTarget.playbackRate
        }))
    }

    // ボリューム変更ハンドラー
    const handleVolumeChange = (event: React.SyntheticEvent<HTMLVideoElement>) => {
        const target = event.currentTarget
        setSettings(prev => ({
            ...prev,
            muted: target.muted,
            volume: target.volume
        }))
    }

    return (
        <ReactPlayer
            src={props.src}
            width="100%"
            height="100%"
            controls={true}
            playbackRate={settings.playbackRate}
            volume={settings.volume}
            muted={settings.muted}
            onRateChange={handleRateChange}
            onVolumeChange={handleVolumeChange}
        />
    )
}
```

なお、react-playerはv3系が最新版ですが、v2からv3への移行では、プロパティ名の変更やコールバック関数の仕様変更など、多くの破壊的変更があります。既存のプロジェクトでv2を使用している場合は、公式の[Migration Guide](https://github.com/cookpete/react-player/blob/master/MIGRATING.md)を必ず確認してください。

### 動画の再生タイミングで処理を実行する

動画の再生や一時停止、終了などのタイミングで処理を実行したい場合には、react-playerに用意されているイベントを使用します。

```tsx
<ReactPlayer
    src={src}
    width="100%"
    height="100%"
    controls={true}
    onStart={() => {
        console.log('再生開始')
    }}
    onPlay={() => {
        console.log('再生中')
    }}
    onPause={() => {
        console.log('一時停止')
    }}
    onEnded={() => {
        console.log('再生終了')
    }}
    onSeeking={() => {
        console.log('シーク中')
    }}
    onSeeked={() => {
        console.log('シーク完了')
    }}
/>
```

学習管理システムで「誰がどこまで見たか」を記録したり、視聴時間を計測したりする場合に、こういったイベントハンドラーを活用します。

## Cookie認証で詰まった話

動画データの取得にCookie認証が必要な場合、そのままでは認証が通らず、動画ファイルを取得できません。私たちのプロジェクトでも、最初この問題に気づかず困りました。

### 何が問題だったか

HLSプレーヤーが動画ファイルを取得するとき、デフォルトでは`XMLHttpRequest`の[`withCredentials`](https://developer.mozilla.org/ja/docs/Web/API/XMLHttpRequest/withCredentials)が`false`になっています。つまり、Cookieが送られないため、認証エラーになってしまいます。

### どう解決したか

`config.hls.xhrSetup`で、すべてのリクエストに`withCredentials = true`を設定すればOKです。

```tsx
<ReactPlayer
    src={src}
    width="100%"
    height="100%"
    controls={true}
    config={{
        hls: {
            xhrSetup: (xhr: XMLHttpRequest, url: string) => {
                // Cookieを送信するように設定
                xhr.withCredentials = true
            }
        }
    }}
/>
```

これで、マニフェストファイル(`.m3u8`)もセグメントファイル(`.ts`)も、Cookieが送られるようになります。

ただし、クライアントで`withCredentials = true`にしたら、サーバー側もCORSの設定が必要です。

```
Access-Control-Allow-Origin: https://your-domain.com
Access-Control-Allow-Credentials: true
```

`withCredentials`を使う場合、`Access-Control-Allow-Origin`にワイルドカード（`*`）は使えません。適切にドメインを指定してください。

## まとめ

react-playerを使ったHLS動画プレーヤーの実装について、実際に詰まったところと解決方法を書いてきました。

- react-playerはReactコンポーネントでHLSを簡単に扱えて便利
- ReactPlayerコンポーネント外で状態を管理し、状態とハンドラーをprops経由で渡す
- Cookie認証を使うなら`xhr.withCredentials = true`が必須

ReactでHLS動画を実装するときの参考になれば嬉しいです。

## PR

株式会社HRBrainでは、一緒に働く仲間を募集しています！
興味を持っていただけた方はぜひ弊社の採用ページをご確認ください。

https://www.hrbrain.co.jp/recruit

## 参考

### ライブラリ・リポジトリ

https://github.com/cookpete/react-player
https://github.com/video-dev/hls.js
https://github.com/videojs/video.js

### ドキュメント

https://developer.mozilla.org/ja/docs/Web/HTML/Reference/Elements/video
https://developer.mozilla.org/ja/docs/Web/API/XMLHttpRequest/withCredentials
