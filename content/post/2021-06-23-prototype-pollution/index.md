+++
title = "prototype pollutionを自動で見つける"
date = "2021-06-23"
draft = true
tags = [
    "security",
]
+++

prototype pollutionについて調べていたところ、単純なパターンであれば自動で検知出来るのではと考えたので実際に実装してみることにしました。

#### prototype pollutionとは

JavaScriptでは、オブジェクトのプロパティにアクセスを試みた際、プロトタイプをたどり、nullに到達するまで探索が続きます。
これはプロトタイプチェーンと呼ばれます。

例として以下のようなコードを考えます。

```javascript
const a = {'p': 1}
const b = Object.create(a)

// bにプロパティはない
console.log(b) // {}

// aまで探索され、1と表示される
console.log(b.p) // 1
```

JavaScriptでは、上記のような挙動を実現するために、各オブジェクトは__proto__プロパティを保持しており、これは他オブジェクトへリンクされています。

```javascript
console.log(b.__proto__ == a) // true
```

一般的なオブジェクトであれば、__proto__プロパティはObject.prototypeにリンクされています。

```javascript
console.log(({}).__proto__ == Object.prototype) // true
```

そのため、Object.prototypeのプロパティを上書きすることが出来れば、各オブジェクトに任意のプロパティを生やすことができます。これはprototype pollutionと呼ばれています。

このようなObject.prototypeを上書きするような挙動は、以下のようなオブジェクトをコピーするコードで発生することがあります。

```javascript
function isObject(obj) {
    return obj !== null && typeof obj === 'object';
}

function merge(a, b) {
    for (let key in b) {
        if (isObject(a[key]) && isObject(b[key])) {
            merge(a[key], b[key]);
        } else {
            a[key] = b[key];
        }
    }
    return a;
}
```

```javascript
const obj1 = {}
const obj2 = JSON.parse('{"__proto__": {"polluted": 1}}')
merge(obj1, obj2)
console.log(Object.prototype.polluted) // == 1
```

先ほど説明したように、一般的なオブジェクトであれば__proto__はObject.prototypeを指すため、__proto__というキーを含むオブジェクトをコピーすると、Object.prototypeを上書きしてしまいます。

今回は、このような`<関数名>(<コピー先>, <コピー元>)`というような関数を検知するコードを書くことを目標とします。

以下は今回のスキャナでの検知を目的としたコードです。

https://github.com/mute1997/prototype-pollution

#### prototype pollutionを見つける

基本方針として、
- コード内のすべての関数定義の取り出し
- __proto__というキーを含むオブジェクトを引数に指定して関数呼び出し
- 関数呼び出し後にObject.prototypeが上書きされているか確認

という方法で確認していきます。

構文解析にはesprimaを利用し、ASTを辿っていくのにestraverseを利用しました。

以下が検出部分のコードです。

```javascript
estraverse.traverse(ast, {
	enter: function (node, _parent) {
		if (node.type !== esprima.Syntax.FunctionDeclaration) {
			return
		}

		try {
			let payload = esprima.parse(`JSON.parse('{"__proto__":{"polluted":1}}')`).body[0].expression
			let arg = esprima.parse('({})').body[0].expression
			let args = node.params.map((_param, idx) => {
				return _.cloneDeep(node.params).map((_args, i) => idx == i ? payload : arg)
			})
			args.forEach((arg) => {
				let declaration = escodegen.generate(node)
				let call = escodegen.generate({
					type: esprima.Syntax.ExpressionStatement,
					expression: {
						type: esprima.Syntax.CallExpression,
						callee: { type: esprima.Syntax.Identifier, name: node.id.name },
						arguments: arg
					}
				})
				let detection = `
				if (Object.prototype.polluted == 1) {
					console.log(\`detected! ${call}\`)
					delete Object.prototype.polluted
				}
				`
				eval(declaration)
				eval(call)
				eval(detection)
			})
		} catch (e) {
			console.log(e)
		}
	}
})
```

上記のコードは、merge関数に対して以下のパターンを試します。

```javascript
merge({}, JSON.parse('{"__proto__":{"polluted":1}}'))
merge(JSON.parse('{"__proto__":{"polluted":1}}'), {})
```

実行すると、`merge({}, JSON.parse('{"__proto__":{"polluted":1}}'))` のパターンが脆弱であると検出されていることが分かります。

```shell
[~/ppfinder]$ npm start

> ppfinder@1.0.0 start /home/mute/ppfinder
> node index.js

[ppfinder] git clone https://github.com/mute1997/prototype-pollution
[ppfinder] command failed: git clone https://github.com/mute1997/prototype-pollution
[ppfinder] npm install: ./prototype-pollution
detected! merge({}, JSON.parse('{"__proto__":{"polluted":1}}'));
finish https://github.com/mute1997/prototype-pollution
```

以下が今回、検出に利用したコードベースです。

- https://github.com/mute1997/find-prototype-pollution

#### 今後の作業

この実装では、テストする関数内で別の関数の呼び出しがあった際にうまく動作しないため、修正が必要です。

さらに、引数に`{"__proto__":{"polluted":1}}`を渡すタイプしか検知が出来ないため、他パターンへの対応が必要です。
