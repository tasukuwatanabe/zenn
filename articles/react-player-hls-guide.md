---
title: "react-playerで動かす「HLS動画ストリーミング」実装ガイド"
emoji: "🎥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['react', 'typescript']
published: false
---

## はじめに

こんにちは。HRBrainで学習管理サービス[「HRBrain ラーニング」](https://www.hrbrain.jp/lms)の開発に関わっている渡邉です。

最近、学習管理システムにおける動画コンテンツの配信機能を、HLS(HTTP Live Streaming)を用いて実装しました。当初はシンプルに実装できると考えていましたが、実際には認証対応や状態管理など、HLS特有の課題に直面することになりました。

この記事では、`react-player 3.3.3`を使用したHLS動画ストリーミング機能の実装について、実際のコードと設定を交えながら解説します。特に、Cookie認証への対応や再生状態の管理といった、本番環境で必要になる実践的なテクニックにフォーカスします。

なお、本記事は**Reactを使ったフロントエンド実装**に焦点を当てています。HLS動画ファイルの生成やサーバー側の配信設定については扱いませんので、ご了承ください。

https://github.com/cookpete/react-player

## HLSとは

HLS(HTTP Live Streaming)は、動画をスムーズに配信するための技術です。

通常の動画ファイルをそのまま配信すると、ファイルサイズが大きすぎてダウンロードに時間がかかってしまいます。HLSでは、動画を数秒ごとの小さなファイル(セグメント)に分割し、視聴者が見ている部分から順番に配信していきます。これにより、待ち時間なく動画の再生を開始できます。

また、ネットワークの速度に応じて自動的に画質を調整する機能もあるため、WiFi環境でも4G回線でもストレスなく視聴できるのが特徴です。YouTubeやNetflixなどの動画配信サービスでも、HLSや類似の技術(DASH)が使われています。

## react-playerを選んだ理由

HLS動画を再生するためのReactライブラリには、いくつかの選択肢があります。代表的なものとして:

- **[hls.js](https://github.com/video-dev/hls.js)**: 低レベルのHLS再生ライブラリで、細かい制御が可能
- **[video.js](https://github.com/videojs/video.js)**: プラグインシステムを持つ包括的なビデオプレーヤー
- **[react-player](https://github.com/cookpete/react-player)**: 複数の動画プラットフォームに対応した抽象化レイヤー

今回、私たちが`react-player`を選択した主な理由は以下の3点です:

### 1. 統一されたAPIと開発体験

`react-player`は、HLSだけでなくYouTube、Vimeo、通常のMP4ファイルなど、多様な動画ソースに対して**統一されたReactコンポーネントインターフェース**を提供します。

```tsx
// react-playerなら、ソースの種類に関係なく同じインターフェースで扱える
<ReactPlayer
    src={videoUrl} // HLS, YouTube, MP4など何でも対応
    controls={true}
    onPlay={handlePlay}
/>
```

将来的に動画ソースを切り替える可能性がある場合、この抽象化は大きなメリットとなります。実際、私たちのプロジェクトでもYouTube動画とHLS動画を併用しており、コンポーネントの実装を統一できました。

### 2. hls.jsの内部利用と適切な抽象化

`react-player`は内部で`hls.js`を使用しています。つまり、`hls.js`の強力な機能を活用しつつ、React向けに使いやすくラップされた形で利用できます。

重要なのは、必要に応じて`config.hls`オプションを通じて**hls.jsの詳細な設定にもアクセスできる**点です:

```tsx
<ReactPlayer
    config={{
        hls: {
            // hls.jsの設定を直接カスタマイズ可能
            xhrSetup: (xhr, url) => { /* ... */ },
            // その他のhls.js設定も指定可能
        }
    }}
/>
```

純粋な`hls.js`を使う場合は、Reactのライフサイクルに合わせた初期化・クリーンアップ処理を自分で実装する必要がありますが、`react-player`はこれらを内部で適切に処理してくれます。

### 3. メンテナンスとコミュニティ

`react-player`は活発にメンテナンスされており、npm週間ダウンロード数も100万を超える人気ライブラリです。issue対応も比較的早く、多くの実績があることから、本番環境での採用に安心感がありました。

## react-playerの基本的な使い方

ここからは、実際のプロジェクトで使用している実装をベースに、`react-player`を使ったHLS動画プレーヤーの実装方法を解説します。

### パッケージのインストール

まず、`react-player`をインストールします:

```bash
npm install react-player@3.3.3
# または
yarn add react-player@3.3.3
```

本記事執筆時点での最新安定版は`3.3.3`です。バージョンアップ時には[リリースノート](https://github.com/cookpete/react-player/releases)を確認し、破壊的変更がないか注意してください。

#### バージョン2系から3系への移行時の注意点

`react-player`のv2系からv3系への移行では、**状態管理の方式が大きく変更**されています。

v2系では内部状態として管理されていた再生速度やボリュームなどの設定が、**v3系では外部から明示的に渡す必要があります**:

```tsx
// v2系: 内部で状態管理される（設定不要）
<ReactPlayer url={src} controls={true} />

// v3系: 外部から状態を渡す必要がある
<ReactPlayer
    src={src}
    controls={true}
    playbackRate={playbackRate}  // 明示的に指定が必要
    volume={volume}              // 明示的に指定が必要
    muted={muted}                // 明示的に指定が必要
/>
```

この変更により、親コンポーネント側で状態管理を行い、プレーヤーに対して制御的に状態を渡すアーキテクチャに変わりました。これは、Reactの「制御されたコンポーネント(Controlled Components)」のパターンに沿った設計です。

実際のプロジェクトでも、この変更に対応するため、`useVideoPlayer`カスタムフックを実装し、状態管理とイベントハンドラーを集約しています。詳細は後述の「再生速度とボリュームの管理」セクションで解説します。

その他の変更点（プロパティ名の変更、設定構造の変更など）については、公式の[Migration Guide](https://github.com/cookpete/react-player/blob/master/MIGRATION.md)を参照してください。

### 基本的なコンポーネント実装

最もシンプルな実装は以下のようになります:

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

しかし、本番環境で使用するには、これだけでは不十分です。以下、実践的な実装のポイントを解説します。

### 1. HLS設定のカスタマイズ

HLS動画を扱う際は、`config.hls`オプションを使用して詳細な設定を行います:

```tsx
<ReactPlayer
    src={src}
    width="100%"
    height="100%"
    controls={true}
    config={{
        hls: {
            xhrSetup: (xhr: XMLHttpRequest, url: string) => {
                // XMLHttpRequestのセットアップをカスタマイズ
                // 詳細は後述の「認証対応の工夫」セクションで解説
            }
        }
    }}
/>
```

`xhrSetup`関数では、HLSセグメントやマニフェストファイルを取得する際の`XMLHttpRequest`オブジェクトを、リクエスト送信前にカスタマイズできます。これは認証ヘッダーの追加やURLの書き換えなど、本番環境で必須となる処理を実装する場所です。

### 2. イベントハンドリング

動画の状態を追跡するため、各種イベントハンドラーを設定します:

```tsx
<ReactPlayer
    src={src}
    width="100%"
    height="100%"
    controls={true}
    // 再生開始時(初回のみ)
    onStart={() => {
        console.log('動画が開始されました')
    }}
    // 再生中
    onPlay={() => {
        console.log('再生中')
    }}
    // 一時停止
    onPause={() => {
        console.log('一時停止')
    }}
    // 再生終了
    onEnded={() => {
        console.log('再生が終了しました')
    }}
    // シーク操作開始
    onSeeking={() => {
        console.log('シーク中...')
    }}
    // シーク操作完了
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

これらのイベントは、学習管理システムにおける「視聴履歴の記録」「視聴時間の計測」「レポート機能」などを実装する際に不可欠です。

### 3. 再生速度とボリュームの管理

ユーザーが設定した再生速度やボリュームを保持し、次回の動画再生時にも適用するには、状態管理が必要です:

```tsx
import { useState } from 'react'
import ReactPlayer from 'react-player'

export const VideoPlayer = ({ src }: { src: string }) => {
    // 再生設定の状態管理
    const [settings, setSettings] = useState({
        playbackRate: 1.0,  // 再生速度(0.5倍速〜2倍速など)
        volume: 1.0,        // ボリューム(0.0〜1.0)
        muted: false        // ミュート状態
    })

    // 再生速度変更時のハンドラー
    const handleRateChange = (event: React.SyntheticEvent<HTMLVideoElement>) => {
        const target = event.currentTarget
        setSettings(prev => ({
            ...prev,
            playbackRate: target.playbackRate
        }))
        // ローカルストレージに保存するなどの処理も可能
    }

    // ボリューム変更時のハンドラー
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

この実装により、ユーザーが再生速度を変更した際にその値を保持し、次の動画でも同じ設定を適用できます。

## 認証対応の工夫

本番環境でHLS動画を配信する場合、多くのケースで認証が必要になります。私たちのプロジェクトでは、Cookie-based認証を採用しており、APIリクエスト時にCookieを送信する必要がありました。

### 課題: デフォルトではCookieが送信されない

HLSプレーヤーがマニフェストファイルやセグメントファイルを取得する際、デフォルトの設定では`XMLHttpRequest`の[`withCredentials`](https://developer.mozilla.org/ja/docs/Web/API/XMLHttpRequest/withCredentials)が`false`となっており、Cookieが送信されません。これにより、認証エラー(401 Unauthorized)が発生します。

### 解決策: xhrSetupでwithCredentialsを有効化

`config.hls.xhrSetup`を使用して、すべてのHLS関連リクエストで`withCredentials`を`true`に設定します:

```tsx
<ReactPlayer
    src={src}
    width="100%"
    height="100%"
    controls={true}
    config={{
        hls: {
            xhrSetup: (xhr: XMLHttpRequest, url: string) => {
                // APIリクエスト時にCookieを送信しないと認証エラーが発生するため、
                // withCredentialsをtrueに設定
                xhr.withCredentials = true
            }
        }
    }}
/>
```

これにより、マニフェストファイル(`.m3u8`)とセグメントファイル(`.ts`)の両方の取得時にCookieが送信されるようになります。

また、クライアント側で`withCredentials = true`を設定した場合、サーバー側でも対応するCORS設定が必要です。

```
Access-Control-Allow-Origin: https://your-domain.com (ワイルドカード*は使用不可)
Access-Control-Allow-Credentials: true
```

`withCredentials`を使用する場合、`Access-Control-Allow-Origin`にワイルドカード(`*`)は使用できないため、具体的なオリジンを指定する必要があります。

## まとめ

本記事では、`react-player 3.3.3`を使用したHLS動画ストリーミング機能の実装について、実際のプロジェクトでの経験を基に解説しました。

### 重要なポイント

1. **react-playerの選定理由**
   - 統一されたAPIによる開発体験の向上
   - hls.jsの機能を活用しつつ、React向けに最適化
   - 活発なメンテナンスとコミュニティサポート

2. **HLS特有の課題と解決策**
   - Cookie認証への対応: `xhr.withCredentials = true`の設定

3. **本番環境での注意点**
   - 再生速度やボリュームの状態管理

4. **サーバー側の対応**
   - CORSヘッダーの適切な設定
   - `withCredentials`使用時のオリジン指定

### 最終的な実装例

以下が、本記事で解説した要素を全て含む完全な実装例です:

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

本記事が、React環境でのHLS動画実装の参考になれば幸いです。
