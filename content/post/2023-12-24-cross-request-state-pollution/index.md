+++
title = "cross-request state pollution とは"
date = "2023-12-19"
tags = [
    "security",
]
+++

## これは
最近 cross-request state pollution という脆弱性を知りました。この脆弱性について理解を深めるため、コードを書いてこの脆弱性の理解を目指します。

## cross-request state pollution とは

これはVue.jsのSSRにおいてサーバー側で作成したStateを他のリクエストにも利用してしまう脆弱性です。
この脆弱性によって、別のリクエストで使われたStateに含まれる情報が他のリクエストにも伝わってしまう可能性があります。

cf. https://ja.vuejs.org/guide/scaling-up/ssr#cross-request-state-pollution

## 実例

どのように発生するのか理解するために、実際にコードを書いてみました。

- https://github.com/mute1997/crsp-demo-for-advent-calendar

<!-- アプリケーションの構成説明 -->
このアプリケーションには大きく2つのページがあります。

- `/` : カウンタ機能のあるページ
- `/store` : Stateを返すだけのページ

このアプリケーションを使って、cross request state pollution が発生する状況を再現してみます。
以下はブラウザでの操作をコードで示したものです。

```python
user1 = requests.Session()
user2 = requests.Session()

# ユーザー1でインクリメント操作を1回行う
...
user1.get("{}{}".format(target, "/store")) # => {'count': 1}

# ユーザー2でインクリメント操作を2回行う
...
user2.get("{}{}".format(target, "/store")) # => {'count': 2}

# ユーザー1がアクセスした後、ユーザー2のセッションでユーザー1の結果が返却されている
user1.get(target)
res = user2.get("{}{}".format(target, "/store")) # => {'count': 1}
```

cross state request pollution が発生しているため、ユーザー2のレスポンスに意図しない結果が含まれていることが確認できます。

<!-- なぜそれが起こっているのかについて解説 -->
これは、以下のようなサーバー側のコードが原因で発生しています。

```JavaScript
// グローバルスコープに作成されているため、一度しか初期化されない。
export const store = createVuexState()
```

## 対策
対策としては以下の2つがありそうです。

1. リクエスト毎にStateを再作成する
2. SSRをやめる

#### 1. リクエスト毎にStateを再作成する
以下のようにすることで、Stateがリクエスト毎に再作成されるため、問題は解決します。

```JavaScript
// store.js
export function createVuexState() {
    return createState({
        ...
    });
}

// app.js
export function createApp(initValue) {
    // このアプリケーションではcreateApp関数はリクエスト毎に呼ばれる
    // そのためStateはリクエスト毎に初期化される
    const store = createVuexState()

    const app = createSSRApp(...)
    app.use(store)

    return { app, store }
}
```

#### 2. SSRをやめる

この解決策は単純です。
Stateの作成が各クライアントで行われるようになれば、この問題は発生しなくなります。

## 検出
そもそもこの脆弱性にどうやって気づくことができるか、についても少し考えてみます。

#### ブラックボックス 
SSRを行うのであれば、Stateをサーバーからクライアントへ何らかの方法で渡す必要があります。
ドキュメントやフレームワークを見る限り、Vue.jsでは以下のようにしてStateの受け渡しをすることが多いようです。

```JavaScript
window.__INITIAL_STATE__ = ${state}
```

このようなStateを発見できた場合、いろんなページにアクセスしながらStateを観察することで、この脆弱性を発見できる可能性はあります。

ただし、この方法ではサーバー側でStateが書き変わるタイミングによって再現したりしなかったり、確実に検知できるような方法ではなさそうです。
そのような事情もあって、ブラックボックス的な検知は難しそうに感じました。

#### ホワイトボックス
Stateがリクエスト毎に再作成されているかをコードを読んで評価する、というのが一番簡単で確実な検出方法に思えました。

## まとめ

この脆弱性の原理は非常に簡単に理解できるものでした。余談になりますが、この脆弱性を調べているときにJSON.stringifyを使ってStateをHTMLに埋め込むサンプルコードをいくつか見かけました。Stateにユーザーが入力した文字列がある場合、XSSができそうなので、こちらも気を付けて実装した方が良さそうです。