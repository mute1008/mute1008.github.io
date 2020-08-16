+++
title = "REST-ler: Automatic Intelligent REST API Fuzzing"
date = "2020-08-13"
tags = [
    "security",
    "paper",
]
+++

fuzzingみたいな感じで脆弱性発見の自動化をWebアプリとかにはうまく適用できないんだろうかと思って調べていた所見つけたのでちょっと読んでみようと思います。

今回はアルゴリズム部分に注目して読んでみます。

この論文ではswaggerの静的分析を行いテストケースを生成しているらしい。

pythonっぽく書かれたアルゴリズムの疑似コードは以下の通り。

```python
Inputs: swaggerspec, maxLength
# Swaggerから作成したリクエストの集合
reqSet = PROCESS(swaggerspec)
# リクエストシーケンスの集合 (最初は空)
seqSet = {}
# メインループ
n = 1
while (n <= maxLength):
  seqSet = EXTEND(seqSet, reqSet)
  seqSet = RENDER(seqSet)
  n = n + 1

# 依存関係が満たされている新しいリクエストを追加することで、
# seqSet内のすべてのシーケンスを拡張する
def EXTEND(seqSet, reqSet):
  newSeqSet = {}
  for seq in seqSet:
    for req in reqSet:
      if DEPENDENCIES(seq, req):
        newSeqSet = newSeqSet + concat(seq, req)
  return newSeqSet

# 辞書を利用して新たに追加されたすべてのリクエストを具体化し、
# それぞれの新しいリクエストシーケンスを実行し、有効なものを保持する
def RENDER(seqSet):
  newSeqSet = {}
  for seq in seqSet:
    req = last_request_in(seq)
    ~V = tuple_of_fuzzable_types_in(req)
    for ~v in ~V:
      newReq = concretize(req,~v)
      newSeq = concat(seq, newReq)
      response = EXECUTE(newSeq)
      if response has a valid code:
        newSeqSet = newSeqSet + newSeq
      else:
        log error
  return newSeqSet

# リクエストで参照されるすべてのオブジェクトが、
# 以前のリクエストシーケンスの何らかのレスポンスによって生成されたものであることをチェックします。
def DEPENDENCIES(seq, req):
  if CONSUMES(req)⊆PRODUCES(seq):
    return True
  else:
    return False

# リクエストに必要なオブジェクト
def CONSUMES(req):
  return object_types_required_in(req)

# 一連のリクエストのレスポンスで生成されたオブジェクト
def PRODUCES(seq):
  dynamicObjects ={}
  for req in seq:
    newObjs = objects_produced_in_response_of(req)
    dynamicObjects = dynamicObjects + newObjs
    return dynamicObjects
```

パパっと見てみると、
1. 依存関係がある場合にはseqSetにreqを追加
2. seqSetの最後のAPIをfuzzing
3. 200が帰ってきたものをseqSetに追加
を繰り返しているっぽい。

REST-lerを利用する場合には以下のコードを用いるっぽい。

```python
from restler import requests
from restler import dependencies

def parse_posts(data):
  post_id = data["id"]
  dependencies.set_var(post_id)

request = requests.Request(
  restler_static("POST"),
  restler_static("/api/blog/posts/"),
  restler_static("HTTP/1.1"),
  restler_static("{"),
  restler_static("body"),
  restler_fuzzable("string"),
  restler_static("POST"),
  restler_static("{"),
  'post_send': {
    'parser': parse_posts,
    'dependencies': [
      post_id.writer(),
    ],
  }
)
```

依存関係があるっていうのはどうやって把握してるんだろうと思ってよく読んで見ると, シーケンスの最後に返されたレスポンスを含むリクエストは依存関係があると判断されるらしい。

例えばシーケンスの最後が`POST /api/blog/posts`とかだとレスポンスにはidが入っている、リクエストボディにidが入っているリクエストは依存関係があると判断されるのかな。

これは実際のコード読んでみないとよくわからないなぁと思って調べていたらソースコードの公開はないらしく、悲しい。

https://twitter.com/vatlidak/status/1097099793514590208?s=21

結果この手法でGitLabのバグを22個発見したということを言っている。

GitLabのようなプロジェクト作成してから操作がたくさんあるようなWebアプリについてはこの手法は有効そうに見える。

クラッシュのグルーピングや探索方法についても記述があったがそのへんはちゃんと読んでないです。
気になる人は読んでみてください。
