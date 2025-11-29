---
title: "react-playerで動かす「HLS動画ストリーミング」実装ガイド"
emoji: "🎥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['react', 'typescript']
published: false
---

## はじめに

こんにちは。HRBrainで学習管理サービス[「HRBrain ラーニング」](https://www.hrbrain.jp/lms)の開発をしている渡邉です。

先日、動画コンテンツの配信機能をHLS(HTTP Live Streaming)で実装しましたが、想像以上に苦戦しました。「react-playerを使えばすぐできるだろう」と思っていたら、Cookie認証が効かなかったり、状態管理で詰まったり。結局、公式ドキュメントだけでは分からない実装の工夫がいくつも必要でした。

この記事では、そんな試行錯誤の末にたどり着いた`react-player 3.3.3`を使ったHLS動画プレーヤーの実装方法を書いていきます。Cookie認証の対応方法や、バージョン3系での状態管理のやり方など、実際に動くコードとセットで紹介します。

なお、この記事ではReactのフロントエンド実装だけを扱います。HLS動画ファイルの生成方法やサーバー側の配信設定については触れません。

https://github.com/cookpete/react-player

## HLSって何?

HLS(HTTP Live Streaming)は、動画配信の仕組みのひとつです。

普通に動画ファイルを配信しようとすると、ファイルサイズが大きくて読み込みに時間がかかります。HLSは動画を数秒単位の小さなファイル(セグメント)に分割して、視聴者が見ている部分から順番に配信します。これにより、最初の読み込みを待たずに再生を始められます。

さらに便利なのが、ネットワークの速度に合わせて自動で画質を変えてくれること。WiFiでも4G回線でも、それぞれの環境で快適に見られます。YouTubeやNetflixも、HLSや似たような技術(DASH)を使っています。

## なぜreact-playerを選んだか

ReactでHLS動画を再生する方法は、実はいくつかあります:

- **[hls.js](https://github.com/video-dev/hls.js)**: HLS再生の本格的なライブラリ。GitHubスター数も多く、細かい制御ができる
- **[video.js](https://github.com/videojs/video.js)**: 老舗の動画プレーヤー。npmダウンロード数も多く、プラグインが豊富
- **[react-player](https://github.com/cookpete/react-player)**: いろんな動画ソース(YouTube、HLSなど)を同じように扱えるラッパーライブラリ

どれも人気があって実績のあるライブラリですが、最終的に`react-player`を選んだのは、次の3つの理由からです:

### 1. Reactとの統合を任せられる

`react-player`は裏側で`hls.js`を使っています。`hls.js`は確かに強力なライブラリですが、それを直接Reactで使おうとすると、いろいろと面倒なことがあります。

例えば:
- コンポーネントのマウント時にHLSインスタンスを初期化
- アンマウント時に適切にクリーンアップ
- video要素への参照を管理
- エラーハンドリングの実装
- イベントリスナーの登録と解除

こういったReactアプリケーションとの統合部分を、`react-player`がすべて面倒を見てくれます。自分で実装すると地味に大変で、しかもバグが混入しやすい部分です。

それでいて、必要なときは`config.hls`オプションで`hls.js`の細かい設定にもアクセスできます:

```tsx
<ReactPlayer
    config={{
        hls: {
            // hls.jsの設定を直接いじれる
            xhrSetup: (xhr, url) => { /* ... */ },
            // 他の設定もいろいろ指定できる
        }
    }}
/>
```

つまり、「Reactとの統合はライブラリに任せて、ビジネスロジックの実装に集中できる」というのが、`react-player`を選んだ一番の理由です。

### 2. 複数の動画フォーマットに対応している

私たちのプロジェクトでは、HLS動画だけでなくYouTube動画も再生したいという要件がありました。`react-player`は、HLS、YouTube、Vimeo、MP4ファイルなど、さまざまな動画フォーマットに対応しており、この点を満たしていました。

```tsx
// HLSでもYouTubeでも同じコンポーネントで扱える
<ReactPlayer
    src={videoUrl} // HLS URL、YouTube URL、どちらもOK
    controls={true}
    onPlay={handlePlay}
/>
```

もし`hls.js`だけを使っていたら、YouTube動画用には別のライブラリを組み合わせる必要があり、コンポーネントの実装も複雑になっていたでしょう。`react-player`なら、動画ソースに関わらず実装を統一できます。

### 3. メンテナンスされてる安心感

`react-player`はちゃんとメンテナンスされてて、週に100万ダウンロードもされてる人気ライブラリです。issueへの対応も割と早いし、実績も十分。本番で使うなら、この安心感は大事です。

## 実際に使ってみる

ここからは、実際に私たちのプロジェクトで使ってる実装をベースに、具体的な使い方を見ていきます。

### パッケージのインストール

まずはインストールから:

```bash
npm install react-player@3.3.3
# または
yarn add react-player@3.3.3
```

この記事を書いてる時点では`3.3.3`が最新です。バージョンを上げるときは[リリースノート](https://github.com/cookpete/react-player/releases)を確認して、破壊的変更がないかチェックしてください。

#### v2からv3への移行で気をつけること

v2からv3にバージョンアップすると、状態管理の仕組みが結構変わります。

v2では再生速度やボリュームをコンポーネント内部で勝手に管理してくれていましたが、v3では外から明示的に渡さないといけなくなりました:

```tsx
// v2: 勝手に管理してくれた
<ReactPlayer url={src} controls={true} />

// v3: 自分で渡す必要がある
<ReactPlayer
    src={src}
    controls={true}
    playbackRate={playbackRate}  // 自分で管理して渡す
    volume={volume}              // 自分で管理して渡す
    muted={muted}                // 自分で管理して渡す
/>
```

つまり、親コンポーネント側で状態を持って、それをプレーヤーに渡すスタイルになったわけです。Reactの「Controlled Components」パターンです。

私たちのプロジェクトでも、この対応で`useVideoPlayer`っていうカスタムフックを作って、状態管理とイベントハンドラーをまとめました。詳しくは後の「再生速度とボリュームの管理」で説明します。

他にもプロパティ名の変更とかいろいろあるので、公式の[Migration Guide](https://github.com/cookpete/react-player/blob/master/MIGRATION.md)も見ておくと良いです。

### まずは最小構成で動かしてみる

一番シンプルに書くとこんな感じです:

```tsx
import ReactPlayer from 'react-player'

export const VideoPlayer = (props) => {
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

ただ、本番で使うにはこれだけじゃ足りません。実際に必要になるポイントを見ていきます。

### 1. HLSの設定をカスタマイズする

HLS動画を扱うときは、`config.hls`で細かい設定ができます:

```tsx
<ReactPlayer
    src={src}
    width="100%"
    height="100%"
    controls={true}
    config={{
        hls: {
            xhrSetup: (xhr: XMLHttpRequest, url: string) => {
                // リクエストする前にXMLHttpRequestをいじれる
                // 詳細は後で説明します
            }
        }
    }}
/>
```

この`xhrSetup`関数で、HLSのファイルを取得する直前に`XMLHttpRequest`をカスタマイズできます。認証ヘッダーを追加したり、URLを書き換えたり、本番環境で必要な処理はここでやります。

### 2. 動画の状態を追跡する

動画で何が起きてるか知りたいときは、イベントハンドラーを設定します:

```tsx
<ReactPlayer
    src={src}
    width="100%"
    height="100%"
    controls={true}
    onStart={() => {
        console.log('動画が開始されました')
    }}
    onPlay={() => {
        console.log('再生中')
    }}
    onPause={() => {
        console.log('一時停止')
    }}
    onEnded={() => {
        console.log('再生が終了しました')
    }}
    onSeeking={() => {
        console.log('シーク中...')
    }}
    onSeeked={() => {
        console.log('シーク完了')
    }}
    config={{
        hls: {
            xhrSetup: (xhr, url) => {
                // HLS設定
            }
        }
    }}
/>
```

学習管理システムで「誰がどこまで見たか」を記録したり、視聴時間を計測したりするなら、こういったイベントが必須です。

### 3. 再生速度とボリュームを覚えておく

ユーザーが設定した再生速度やボリュームを、次の動画でも使えるようにするには、状態管理が必要です:

```tsx
import { useState } from 'react'
import ReactPlayer from 'react-player'

export const VideoPlayer = ({ src }: { src: string }) => {
    // 設定を保持しておく
    const [settings, setSettings] = useState({
        playbackRate: 1.0,  // 再生速度
        volume: 1.0,        // ボリューム
        muted: false        // ミュート
    })

    // 再生速度が変わったら状態を更新
    const handleRateChange = (event: React.SyntheticEvent<HTMLVideoElement>) => {
        const target = event.currentTarget
        setSettings(prev => ({
            ...prev,
            playbackRate: target.playbackRate
        }))
        // ここでlocalStorageに保存とかもできる
    }

    // ボリュームが変わったら状態を更新
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
            src={src}
            width="100%"
            height="100%"
            controls={true}
            playbackRate={settings.playbackRate}
            volume={settings.volume}
            muted={settings.muted}
            onRateChange={handleRateChange}
            onVolumeChange={handleVolumeChange}
            config={{
                hls: {
                    xhrSetup: (xhr, url) => {
                        // HLS設定
                    }
                }
            }}
        />
    )
}
```

こうしておけば、ユーザーが再生速度を変えたら、その設定を覚えておいて次の動画でも使えます。

## Cookie認証で詰まった話

本番環境で動画を配信するなら、だいたい認証が必要です。私たちのプロジェクトではCookie認証を使っていましたが、これが最初うまく動かなくて困りました。

### 何が問題だったか

HLSプレーヤーが動画ファイルを取得するとき、デフォルトでは`XMLHttpRequest`の[`withCredentials`](https://developer.mozilla.org/ja/docs/Web/API/XMLHttpRequest/withCredentials)が`false`になっています。つまり、Cookieが送られません。当然、認証エラー(401)が返ってきます。

### どう解決したか

`config.hls.xhrSetup`で、すべてのリクエストに`withCredentials = true`を設定すればOKです:

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

`withCredentials`を使う場合、`Access-Control-Allow-Origin`にワイルドカード(`*`)は使えません。ちゃんとドメインを指定してください。

## まとめ

`react-player 3.3.3`を使ったHLS動画プレーヤーの実装について、実際に詰まったところと解決方法を書いてきました。

### ポイント

- `react-player`は、HLSもYouTubeも同じように扱えて便利
- 裏で`hls.js`を使ってるから、細かい設定も可能
- Cookie認証を使うなら`xhr.withCredentials = true`が必須
- サーバー側でもCORSの設定を忘れずに
- v3では再生速度やボリュームを自分で管理する必要がある

### 最終的なコード

ここまで説明した内容をまとめた実装がこちらです:

```tsx
import { useState } from 'react'
import ReactPlayer from 'react-player'

type Props = {
    src: string
    events?: {
        onStart: () => void
        onPlay: () => void
        onPause: () => void
        onEnded: () => void
        onSeeked: () => void
        onSeeking: () => void
    }
}

export const VideoPlayer = ({ src, events }: Props) => {
    // 再生設定の状態管理
    const [settings, setSettings] = useState({
        playbackRate: 1.0,
        volume: 1.0,
        muted: false
    })

    // 再生速度変更ハンドラー
    const handleRateChange = (event: React.SyntheticEvent<HTMLVideoElement>) => {
        setSettings((prev) => ({
            ...prev,
            playbackRate: event.currentTarget.playbackRate
        }))
    }

    // ボリューム変更ハンドラー
    const handleVolumeChange = (event: React.SyntheticEvent<HTMLVideoElement>) => {
        const target = event.currentTarget
        setSettings((prev) => ({
            ...prev,
            muted: target.muted,
            volume: target.volume
        }))
    }

    return (
        <ReactPlayer
            src={src}
            width="100%"
            height="100%"
            controls={true}
            muted={settings.muted}
            playbackRate={settings.playbackRate}
            volume={settings.volume}
            onRateChange={handleRateChange}
            onVolumeChange={handleVolumeChange}
            onStart={events?.onStart}
            onPlay={events?.onPlay}
            onPause={events?.onPause}
            onEnded={events?.onEnded}
            onSeeked={events?.onSeeked}
            onSeeking={events?.onSeeking}
            config={{
                hls: {
                    xhrSetup: (xhr: XMLHttpRequest, url: string) => {
                        // Cookie認証のため、withCredentialsを有効化
                        xhr.withCredentials = true
                    }
                }
            }}
        />
    )
}
```

ReactでHLS動画を実装するときの参考になれば嬉しいです。
