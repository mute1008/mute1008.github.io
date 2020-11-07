+++
title = "イベントハンドラでの実体参照の扱い"
date = "2020-11-07"
tags = [
    "security",
]
+++

以下の記事を読んでいて気になったことがあるので調べてみた。

https://portswigger.net/web-security/cross-site-scripting/preventing#encode-data-on-output

気になったのは以下の部分。

```text
Sometimes you'll need to apply multiple layers of encoding, in the correct order. For example, to safely embed user input inside an event handler, you need to deal with both the JavaScript context and the HTML context. So you need to first Unicode-escape the input, and then HTML-encode it:

<a href="#" onclick="x='This string needs two layers of escaping'">test</a>
```

イベントハンドラにJavaScriptを書いている所にユーザーの入力を埋め込む場合は、unicodeエスケープをしてからHTMLエンコードする必要があると書いてある。

個人的にはunicodeエスケープだけではダメなのか気になったので実験してみた。

アラートを実行するために`';alert(1);'`を入力したことを考える。

以下がunicodeエスケープのみを行って埋め込んだ場合。

```html
<a href="#" onclick="x='&#x0027;&#x003b;alert&#x0028;1&#x0029;&#x003b;&#x0027;';">link</a>
```

以下はunicodeエスケープした後にHTMLエンコードを行ったもの。

```html
<a href="#" onclick="x='&#38;&#35;x0027&#59;&#38;&#35;x003b&#59;alert&#38;&#35;x0028&#59;1&#38;&#35;x0029&#59;&#38;&#35;x003b&#59;&#38;&#35;x0027&#59;';">link</a>
```

unicodeエスケープとHTMLエンコードには以下の関数を用いた。

```javascript
function htmlEncode(str){
  return String(str).replace(/[^\w. ]/gi, function(c){
     return '&#'+c.charCodeAt(0)+';';
  });
}
```

```javascript
function jsEscape(str){
  return String(str).replace(/[^\w. ]/gi, function(c){
     return '&#x'+('0000'+c.charCodeAt(0).toString(16)).slice(-4)+';';
  });
}
```

これらをブラウザで開いてからクリックすると、前者はアラートが実行され、後者は何も起こらない。

これはイベントハンドラでのみ起こるのか調べるために、unicodeエスケープのみを行った`<img src=x onerror=alert(1)>`をページに書き込んで開いてみる。

```html
&#x003c;img src&#x003d;x onerror&#x003d;alert&#x0028;1&#x0029;&#x003e;
```

`<img src=x onerror=alert(1)>`と表示され、アラートは発火しない。

イベントハンドラと同じJavaScriptコードがscriptタグに埋め込まれている場合を考える。

以下のようなHTMLがサーバーによって生成される。

```html
<script>
x='&#x0027;&#x003b;alert&#x0028;1&#x0029;&#x003b;&#x0027;';
</script>
```

これを開いてもアラートは実行されない。

これらからイベントハンドラでのみ実体参照が評価された状態でJavaScriptが実行されることがわかる。
