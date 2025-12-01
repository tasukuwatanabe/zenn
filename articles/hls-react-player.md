---
title: "HLSストリーミングをreact-playerで実装する"
emoji: "🎥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['react', 'typescript']
published: false
publication_name: "hrbrain"
---

この記事はアドベントカレンダー4日目の記事です🎄🎅
[HRBrain Advent Calendar 2025](https://adventar.org/calendars/12091)

## はじめに

こんにちは。HRBrainで学習管理サービス[「HRBrain ラーニング」](https://www.hrbrain.jp/lms)を開発している渡邉です。

最近、動画コンテンツの配信機能をHLS（HTTP Live Streaming）で実装しました。「ライブラリを使えばすぐできるだろう」と思っていたら、Cookie認証が効かなかったり、状態管理で詰まったり。この記事では、そんな試行錯誤の末にたどり着いた「react-player」を使ったHLS動画プレーヤーの実装方法を紹介します。

なお、この記事ではReactのフロントエンド実装だけを扱います。HLS動画ファイルの生成方法やサーバー側の配信設定については触れません。

https://github.com/cookpete/react-player

*執筆時点（2025/12/4）での最新バージョン： react-player@3.4.0*

## HLSとは

HLS（HTTP Live Streaming）は、動画配信の仕組みのひとつです。

通常、動画ファイルを配信しようとすると、ファイルサイズが大きく読み込みに時間がかかります。HLSは動画を数秒単位の小さなファイル（セグメント）に分割し、視聴者が見ている部分から順番に配信します。これにより、最初の読み込みを待たずに再生を始められます。

さらにHLSは、ネットワークの速度に合わせて自動で画質を変えてくれる点も便利です。電車などで動画を観ている時に、動画が自動的に低画質に切り替わった経験があるかと思います。

この機能のおかげで、WiFiでも4G回線でも、それぞれの環境で快適に視聴できます。YouTubeやNetflixなどの動画プラットフォームでも、HLSや類似した技術（DASH）が使われています。

ちなみに、react-playerにはデモサイトが用意されているので、「HLS」を選択して再生することで、HLSストリーミングを動かしてみることができます。

[ReactPlayer Demo](https://cookpete.github.io/react-player/)

![](/images/hls-react-player/react-player-demo.png)

## react-playerを選んだ理由

HLS動画を実装する際、使用できそうなライブラリを調査したところ、以下の3つが候補に上がりました。

- **[hls.js](https://github.com/video-dev/hls.js)**: HLS再生の本格的なライブラリ。GitHubスター数も多く、細かい制御が可能
- **[react-player](https://github.com/cookpete/react-player)**: hls.jsをラップし、ReactコンポーネントとしてHLS機能を提供している
- **[video.js](https://github.com/videojs/video.js)**: 老舗の動画プレーヤー。npmダウンロード数も多く、プラグインが豊富

npm trendsで人気を比較すると、人気傾向は以下のようになっています。

![](/images/hls-react-player/npm-trends.png)

どれも人気があって実績のあるライブラリですが、最終的にreact-playerを選んだのは、次の3つの理由からです。

### 動画プレーヤーに必要な機能が揃っている

動画プレーヤーに必要な基本機能が、react-playerにはすべて揃っています。

react-playerは内部でHTMLの`<video>`要素を使用しており、`playbackRate`、`volume`、`muted`といった標準的なプロパティを同名でそのまま使えます。

例えば、再生速度を変更したい場合は`playbackRate`プロパティに値を渡すだけです。ブラウザ標準の`<video>`要素と同じプロパティ名なので、学習コストがほとんどかかりません。

https://developer.mozilla.org/ja/docs/Web/API/HTMLMediaElement

### ReactコンポーネントとしてHLSが使える

react-playerは内部で`hls.js`を使用していますが、それをReactで使いやすい形にラップしています。もし`hls.js`を直接使おうとすると、Reactのライフサイクルに合わせた実装を自分で書く必要があり、面倒です。

react-playerを使えば、こうした統合処理をすべて任せることができ、開発者は動画プレーヤーのUI/UXや機能実装に集中できます。

### 高頻度でメンテナンスが行われている

react-playerは更新頻度も高く、適切にメンテナンスされているライブラリです。issueへの対応も比較的早く、実績も十分だと判断しました。

## react-playerを動かす

ここからは、react-playerの具体的な使い方を見ていきます。まずはインストールから。

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
            src={props.src} // 動画ファイルを渡す
            width="100%"
            height="100%"
            controls={true} // コントロール系のUIを有効化
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

react-playerでは、再生速度やボリュームなどの状態を親コンポーネント側で管理し、それをプロパティとして渡します。プレーヤーコンポーネント自体は内部で状態を保持せず、外部から渡された値に従って動作します。

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
        setSettings(prev => ({
            ...prev,
            muted: event.currentTarget.muted,
            volume: event.currentTarget.volume
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

`onStart`（再生開始）、`onPause`（一時停止）、`onEnded`（再生終了）などさまざまなイベントが用意されているため、イベントを組み合わせて拡張的な実装を行いたいケースでは重宝します。例えば、視聴時間を計測したり、特定の地点まで視聴した場合の処理などを実装する際に、こうしたイベントハンドラーの活用が有効です。

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

react-playerに用意されたイベントについても、「ReactPlayer Demo」のコンソールで確認することができます。

![](/images/hls-react-player/react-player-event.png)

## Cookie認証を有効にする設定

動画データの取得にCookie認証が必要な場合、そのままでは認証が通らず、動画ファイルを取得できませんでした。

HLSプレーヤーが動画ファイルを取得するとき、デフォルトでは`XMLHttpRequest`の`withCredentials`が`false`になっています。つまり、Cookieが送られないため、認証エラーになるというわけです。

リクエストでCookieを送るには、`config.hls.xhrSetup`というオプション内で`withCredentials = true`を設定することで解決できました。

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

また、クライアントで`withCredentials = true`に設定した場合、サーバー側もCORSの設定が必要です。こちらも併せて対応しましょう。

```
Access-Control-Allow-Origin: https://your-domain.com
Access-Control-Allow-Credentials: true
```

`withCredentials`を使う場合、`Access-Control-Allow-Origin`にワイルドカード（`*`）は使えません。適切にドメインを指定してください。

https://developer.mozilla.org/ja/docs/Web/API/XMLHttpRequest/withCredentials

## まとめ

react-playerを使ったHLS動画プレーヤーの実装について、HLSの基本的な仕組みからライブラリの実装方法について紹介しました。ReactでHLS動画を実装するときの参考になれば嬉しいです。

- HLSは動画を小さな単位に分割して順次配信することで、素早い再生開始とネットワーク状況に応じた適応的な画質切り替えを実現する動画配信方式
- react-playerはReactコンポーネントでHLSを簡単に扱えて便利
- ReactPlayerコンポーネント外で状態を管理し、状態とハンドラーをprops経由で渡す
- Cookie認証を使うなら`xhr.withCredentials = true`設定が必要

### PR

株式会社HRBrainでは、一緒に働く仲間を募集しています！
興味を持っていただけた方はぜひ弊社の採用ページをご確認ください。

https://www.hrbrain.co.jp/recruit

### 参考資料

https://github.com/cookpete/react-player
https://cookpete.github.io/react-player
https://github.com/video-dev/hls.js
https://github.com/videojs/video.js
https://developer.mozilla.org/ja/docs/Web/HTML/Reference/Elements/video
https://developer.mozilla.org/ja/docs/Web/API/XMLHttpRequest/withCredentials
