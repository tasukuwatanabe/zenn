---
title: "useEffect vs refコールバック：DOM操作における適切な選択を実装で比較する"
emoji: "🔀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['react', 'typescript', 'dom']
published: false
publication_name: "hrbrain"
---

## はじめに

こんにちは。HRBrainで学習管理サービス[「HRBrain ラーニング」](https://www.hrbrain.jp/lms)を開発している渡邉です。

DOM要素に対してイベントリスナーを登録したり、要素のサイズを監視したりする際、`useEffect`と`refコールバック`のどちらを使うべきか迷ったことはありませんか？

私も動画プレーヤーの実装中に、条件付きレンダリングされるvideo要素のクリーンアップで問題に遭遇しました。最初は`useEffect`を使っていましたが、うまく動作せず、調査の結果、`refコールバック`を使うべきケースだと気づきました。

この記事では、実際のコード例を通じて、`useEffect`と`refコールバック`の使い分けを解説します。

## 結論：useEffectとrefコールバックの使い分け

まず結論から。DOM操作において、以下の基準で使い分けます。

### refコールバックを選ぶべきケース

以下のいずれかに当てはまる場合は、refコールバックを使いましょう：

- **条件付きレンダリングで要素が出現/消失する**
- **`ref.current`を依存配列に入れるか悩んでいる**
- **DOM APIを直接扱う**（ResizeObserver、IntersectionObserverなど）

### それ以外はuseEffectで

上記に当てはまらない場合は、`useEffect`を使います：

- DOM要素の変化に関係ない副作用（documentやwindowへのイベントリスナー登録など）
- 初回マウント時のみの処理
- propsやstateの変化に応じた処理

:::message
外部の状態（ウィンドウサイズやスクロール位置など）を購読して値として使いたい場合は、`useSyncExternalStore`の使用も検討してください。詳細は[公式ドキュメント](https://react.dev/reference/react/useSyncExternalStore)を参照。
:::

## useEffectとrefコールバックの基本的な違い

両者の最も重要な違いは、**実行タイミング**です。

### useEffect：コンポーネントのライフサイクル

`useEffect`は、コンポーネント全体のマウント/アンマウントや、依存配列の変化に反応します。

```tsx
useEffect(() => {
    // コンポーネントがマウントされた時
    console.log('Component mounted')
    
    return () => {
        // コンポーネントがアンマウントされる時
        console.log('Component unmounted')
    }
}, [])
```

### refコールバック：DOM要素のライフサイクル

`refコールバック`は、**個別のDOM要素**のマウント/アンマウントに反応します。

```tsx
const handleRef = (element: HTMLDivElement | null) => {
    if (element) {
        // この要素がDOMに追加された時
        console.log('Element mounted')
    } else {
        // この要素がDOMから削除される時
        console.log('Element unmounted')
    }
}

return <div ref={handleRef}>Hello</div>
```

### この違いが重要になる場面

条件付きレンダリングで要素だけが消える場合、`useEffect`では検知できません：

```tsx
function Component() {
    const [show, setShow] = useState(true)
    
    useEffect(() => {
        // このクリーンアップは、Componentがアンマウントされる時だけ実行される
        return () => console.log('cleanup')
    }, [])
    
    return (
        <div>
            {show && <div>条件付きで表示</div>}
            <button onClick={() => setShow(!show)}>Toggle</button>
        </div>
    )
}
```

上記のコードで`show`が`false`になっても、`useEffect`のクリーンアップは実行されません。親コンポーネント（`Component`）はまだマウントされているためです。

このような場面で、refコールバックが活躍します。

## refコールバックを使うべき3つのケース

ここからは、具体的なコード例を通じて、refコールバックが適切なケースを見ていきます。

### ケース1：条件付きレンダリング時のイベントリスナー登録

#### 問題：useEffectでは要素の出現/消失に反応できない

video要素の音声を、要素がアンマウントされる時に確実に停止したい場合を考えます。

```tsx
import { useRef, useEffect, useState } from 'react'

const VideoPlayerWithUseEffect = () => {
    const [showVideo, setShowVideo] = useState(true)
    const videoRef = useRef<HTMLVideoElement>(null)

    useEffect(() => {
        const videoElement = videoRef.current
        
        // クリーンアップ関数
        return () => {
            if (videoElement) {
                videoElement.pause()
            }
        }
    }, [])

    return (
        <div>
            <button onClick={() => setShowVideo(!showVideo)}>
                {showVideo ? '動画を非表示' : '動画を表示'}
            </button>
            {showVideo && (
                <video ref={videoRef} controls src="/sample-video.mp4" />
            )}
        </div>
    )
}
```

**この実装の問題点：**

- `useEffect`のクリーンアップ関数は、**コンポーネント全体がアンマウントされる時**に実行される
- しかし、`{showVideo && <video ... />}`により、**video要素だけがDOMから削除される**
- 親コンポーネントはまだマウントされているため、クリーンアップ関数は呼ばれない
- 結果として、音声が再生され続ける

依存配列に`showVideo`を追加しても解決しません。`showVideo`の変更時にエフェクトが再実行されますが、video要素がすでにDOMから削除された後では手遅れです。

#### 解決：refコールバックで実装

refコールバックは、**要素自体のマウント/アンマウントに直接反応**します。

```tsx
import { useState, useRef, useCallback } from 'react'

const VideoPlayer = () => {
    const [showVideo, setShowVideo] = useState(true)
    const videoElementRef = useRef<HTMLVideoElement | null>(null)

    const videoRef = useCallback((element: HTMLVideoElement | null) => {
        if (element) {
            // video要素がマウントされた時
            videoElementRef.current = element
        } else {
            // video要素がアンマウントされる時
            if (videoElementRef.current) {
                videoElementRef.current.pause()
                videoElementRef.current = null
            }
        }
    }, [])

    return (
        <div>
            <button onClick={() => setShowVideo(!showVideo)}>
                {showVideo ? '動画を非表示' : '動画を表示'}
            </button>
            {showVideo && (
                <video
                    ref={videoRef}
                    controls
                    src="/sample-video.mp4"
                />
            )}
        </div>
    )
}
```

この実装により、video要素がDOMから削除される直前に確実に`pause()`が呼び出され、音声が停止します。

:::message alert
**なぜuseRefが必要なのか**
React 18までの仕様では、refコールバックのアンマウント時に`null`が渡されます。そのため、アンマウント時に要素にアクセスするには、マウント時に`useRef`で要素を保存しておく必要があります。React 19では、クリーンアップ関数を返すことでこの問題が解決されます（後述）。

また、refコールバックは`useCallback`でメモ化する必要があります。メモ化しないと、毎回新しい関数が作成され、refコールバックが不要に再実行されます。
:::

### ケース2：ResizeObserverによる要素サイズの監視

#### 問題：useEffectでは要素の入れ替わりを検知できない

要素のサイズ変化を監視したい場合、ResizeObserverを使います。

```tsx
import { useRef, useEffect } from 'react'

const ComponentWithUseEffect = ({ showFirst }: { showFirst: boolean }) => {
    const ref = useRef<HTMLDivElement>(null)

    useEffect(() => {
        if (ref.current) {
            const observer = new ResizeObserver((entries) => {
                console.log('Size changed:', entries[0].contentRect)
            })
            observer.observe(ref.current)

            return () => {
                observer.disconnect()
            }
        }
    }, []) // 依存配列が空

    return (
        <div>
            {showFirst ? (
                <div ref={ref}>First element</div>
            ) : (
                <div ref={ref}>Second element</div>
            )}
        </div>
    )
}
```

**この実装の問題点：**

- `showFirst`が変わると、異なるdiv要素がレンダリングされる
- しかし、`useEffect`の依存配列が空なので、Observerは最初の要素にしか適用されない
- 新しい要素のサイズ変化は監視されない

依存配列に`showFirst`を追加しても、`ref.current`が指す要素は毎回同じrefオブジェクト経由でアクセスされるため、Reactはこれを変更とみなしません。

#### 解決：refコールバックで実装

```tsx
import { useCallback } from 'react'

const ComponentWithRefCallback = ({ showFirst }: { showFirst: boolean }) => {
    const handleRef = useCallback((element: HTMLDivElement | null) => {
        if (element) {
            const observer = new ResizeObserver((entries) => {
                console.log('Size changed:', entries[0].contentRect)
            })
            observer.observe(element)

            return () => {
                observer.disconnect()
            }
        }
    }, [])

    return (
        <div>
            {showFirst ? (
                <div ref={handleRef}>First element</div>
            ) : (
                <div ref={handleRef}>Second element</div>
            )}
        </div>
    )
}
```

refコールバックは、要素が変わるたびに自動的に再実行されます。古い要素のObserverはクリーンアップされ、新しい要素に新しいObserverが適用されます。

### ケース3：IntersectionObserverによる可視性の監視

#### 問題：条件付きレンダリングとの組み合わせで漏れが発生

無限スクロールなどで、要素がビューポートに入った時に処理を実行したい場合：

```tsx
import { useRef, useEffect, useState } from 'react'

const ListItemWithUseEffect = ({ id, show }: { id: number; show: boolean }) => {
    const ref = useRef<HTMLDivElement>(null)

    useEffect(() => {
        if (ref.current) {
            const observer = new IntersectionObserver((entries) => {
                if (entries[0].isIntersecting) {
                    console.log(`Item ${id} is visible`)
                }
            })
            observer.observe(ref.current)

            return () => {
                observer.disconnect()
            }
        }
    }, [id])

    if (!show) return null

    return <div ref={ref}>Item {id}</div>
}
```

**この実装の問題点：**

- `show`が`false`になると、要素はDOMから削除される
- しかし、`useEffect`のクリーンアップは呼ばれない（コンポーネント自体はまだマウントされているため）
- Observerがメモリリークを引き起こす可能性がある

#### 解決：refコールバックで実装

```tsx
import { useCallback } from 'react'

const ListItemWithRefCallback = ({ id, show }: { id: number; show: boolean }) => {
    const handleRef = useCallback((element: HTMLDivElement | null) => {
        if (element) {
            const observer = new IntersectionObserver((entries) => {
                if (entries[0].isIntersecting) {
                    console.log(`Item ${id} is visible`)
                }
            })
            observer.observe(element)

            return () => {
                observer.disconnect()
            }
        }
    }, [id])

    if (!show) return null

    return <div ref={handleRef}>Item {id}</div>
}
```

refコールバックを使うことで、要素がDOMから削除される時に確実にObserverがdisconnectされます。

## refコールバックが不要なケース

refコールバックは強力ですが、すべてのケースで使うべきではありません。**DOM要素の変化に関係ない副作用**の場合は、従来通り`useEffect`を使います。

### useEffectで十分な場合

以下のような場合は、`useEffect`の方が適切です：

**1. documentやwindowへのイベントリスナー登録**

```tsx
import { useEffect } from 'react'

const KeyboardShortcuts = () => {
    useEffect(() => {
        const handleKeyDown = (e: KeyboardEvent) => {
            if (e.key === 'Escape') {
                console.log('Escape pressed')
            }
        }

        document.addEventListener('keydown', handleKeyDown)

        return () => {
            document.removeEventListener('keydown', handleKeyDown)
        }
    }, [])

    return <div>Press Escape key</div>
}
```

この場合、イベントリスナーは`document`に登録されており、特定のDOM要素のライフサイクルとは関係ありません。そのため、`useEffect`が適切です。

**2. 初回マウント時のみの処理**

- API呼び出し
- 外部ライブラリの初期化
- ログ送信

**3. propsやstateの変化に応じた処理**

これらはすべて、DOM要素の変化とは無関係な副作用であり、コンポーネントのライフサイクルやprops/stateの変化に反応すべきものです。

## React 18と19での書き方の違い

React 19では、refコールバックの使い勝手が大幅に向上しました。

### React 18までの書き方

React 18では、refコールバックのアンマウント時に`null`が渡されます。そのため、クリーンアップ処理で要素にアクセスするには、`useRef`で要素を保存しておく必要があります。

```tsx
import { useState, useRef, useCallback } from 'react'

// React 18の書き方
const VideoPlayer = () => {
    const [showVideo, setShowVideo] = useState(true)
    const videoElementRef = useRef<HTMLVideoElement | null>(null)

    const videoRef = useCallback((element: HTMLVideoElement | null) => {
        if (element) {
            // 要素がマウントされた時
            videoElementRef.current = element
            console.log('Video mounted')
        } else {
            // 要素がアンマウントされる時（nullが渡される）
            if (videoElementRef.current) {
                videoElementRef.current.pause()
                console.log('Video unmounted')
                videoElementRef.current = null
            }
        }
    }, [])

    return (
        <div>
            <button onClick={() => setShowVideo(!showVideo)}>
                Toggle
            </button>
            {showVideo && <video ref={videoRef} controls src="/video.mp4" />}
        </div>
    )
}
```

**React 18の課題：**

- アンマウント時に`null`が渡されるため、要素にアクセスできない
- 別途`useRef`で要素を保存する必要があり、やや冗長

### React 19の新しい書き方

React 19では、`useEffect`と同様に、refコールバックから**クリーンアップ関数を返せる**ようになりました。

```tsx
import { useState, useCallback } from 'react'

// React 19の書き方
const VideoPlayer = () => {
    const [showVideo, setShowVideo] = useState(true)

    const videoRef = useCallback((element: HTMLVideoElement) => {
        console.log('Video mounted')

        // クリーンアップ関数を返す
        return () => {
            element.pause()
            console.log('Video unmounted')
        }
    }, [])

    return (
        <div>
            <button onClick={() => setShowVideo(!showVideo)}>
                Toggle
            </button>
            {showVideo && <video ref={videoRef} controls src="/video.mp4" />}
        </div>
    )
}
```

**React 19の改善点：**

- クリーンアップ関数内でクロージャを利用して`element`に直接アクセスできる
- `useRef`が不要になり、コードがシンプルになる
- `useEffect`と同じ考え方が通用するため、直感的

:::message
**重要：後方互換性**
React 19でも後方互換性が保たれています：
- クリーンアップ関数を返した場合 → クリーンアップ関数が呼ばれる（新しい動作）
- クリーンアップ関数を返さない場合 → `null`が渡される（従来の動作）

ただし、将来のバージョンでは`null`を渡す動作は非推奨になる予定です。
:::

### refコールバックのメモ化

React 18でも19でも、refコールバックは`useCallback`でメモ化する必要があります。

```tsx
// ❌ 悪い例：メモ化していない
const videoRef = (element: HTMLVideoElement) => {
    return () => element.pause()
}

// ✅ 良い例：useCallbackでメモ化
const videoRef = useCallback((element: HTMLVideoElement) => {
    return () => element.pause()
}, [])
```

メモ化しない場合、毎レンダリングごとに新しい関数が作成され、refコールバックが不要に再実行されます。

**refコールバックの再実行ルール：**

1. refコールバック関数が変わった場合、Reactは以下を実行：
   - 古いrefコールバックのクリーンアップ（`null`を渡す or クリーンアップ関数を実行）
   - 新しいrefコールバックを呼び出し（要素を渡す）

2. refコールバック関数が変わらない場合：
   - DOM要素自体が変わった時のみ再実行される

この動作により、`useCallback`でのメモ化が重要になります。

## まとめ

この記事では、`useEffect`と`refコールバック`の使い分けを、実装例を通じて解説しました。

### 使い分けの要点

**refコールバックを選ぶべきケース：**
- 条件付きレンダリングで要素が出現/消失する
- `ref.current`を依存配列に入れるか悩んでいる
- DOM APIを直接扱う（ResizeObserver、IntersectionObserverなど）

**useEffectで十分なケース：**
- DOM要素の変化に関係ない副作用
- documentやwindowへのイベントリスナー登録
- 初回マウント時のみの処理

### React 19の改善

React 19では、refコールバックからクリーンアップ関数を返せるようになり、`useEffect`と同じ感覚で使えるようになりました。

### 迷ったときは

「この処理は、**特定のDOM要素のライフサイクル**に紐づいているか？」と考えましょう。Yesならrefコールバック、Noなら`useEffect`です。

### PR

株式会社HRBrainでは、一緒に働く仲間を募集しています！
興味を持っていただけた方はぜひ弊社の採用ページをご確認ください。

https://www.hrbrain.co.jp/recruit

### 参考資料

https://react.dev/reference/react-dom/components/common#ref-callback
https://react.dev/blog/2024/12/05/react-19#cleanup-functions-for-refs
https://react.dev/learn/manipulating-the-dom-with-refs
