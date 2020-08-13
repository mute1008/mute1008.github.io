+++
title = "preenyを利用したサーバーのファジング"
date = "2020-08-08"
tags = [
    "security",
]
+++

最近AFLを利用していろんなソフトウェアのファジングをしています。
AFLとは、テスト手法であるファジングの一種でgreybox-fuzzingに分類されるものです。
詳しい使い方や詳細は以下を参照してください。

https://github.com/google/AFL

このAFLでは標準入力、もしくは引数からファイルとして受け取り、処理をした後に即座に終了するというアプリケーションが期待されます。
nginxなどのWebサーバーはイベントループがあり、ソケットを利用するためにAFLをそのまま利用することが出来ません。

イベントループは自分で即座にbreakすることですぐに終了することが出来ますが、ソケットの問題はpreenyで解決出来ます。

#### preenyとは

https://github.com/zardus/preeny

preenyとは、LD_PRELOADでライブラリ関数を上書きするものです。
いくつかモジュールがありますが、desockというものを利用することで`socket()`、`bind()`、`listen()`、`accept()`あたりが上書きされるようです。
これを利用することで、ソケット操作はstdin、stdoutを読み書きするようになるので、AFLで利用することができます。

#### shadowsocksで試してみる
nginxとかでやろうと思ったんですが同じことをやっている英語記事を書いている途中に見つけてしまったのでshadowsocksでやります。

shadowsocksは中国で開発されたプロキシツールです。
オリジナルは中国政府から削除を要請されたようで、今は以下のレポジトリで開発が進んでいます。

https://github.com/shadowsocks/shadowsocks-libev

##### イベントループを止める
shadowsocksはlibevを使ってイベントループを処理しています。

`server.c`にエントリーポイントがあるのでここから読み進めていきます。
読み進めていくと, `server_recv_cb()`という関数で受信処理をしているようです。
この関数をラップして, この関数の終了時にイベントループを終わらせるようにします。

```c
static void server_recv_once_cb(EV_P_ ev_io *w, int revents) {
  server_recv_cb(loop, w, revents);
  ev_break(EV_A_ EVBREAK_ONE);
}

static server_t *
new_server(int fd, listen_ctx_t *listener)
{
    ...

    ev_io_init(&server->recv_ctx->io, server_recv_once_cb, fd, EV_READ);
    ...
}
```

こうすることで, リクエストを１つ受け取ったら終了するようになりました。

##### afl-gccでコンパイルするように変更
src/Makefileを編集してgccと書いてあるところをafl-gccにしていきます。
```txt
...
CC = afl-gcc
...
```

コンパイルします。
コンパイル途中に`Instrumented n locations`みたいな表示が出てきたらAFLを利用して, コンパイル出来ています。
``` shell
$ make
afl-cc 2.57b by <lcamtuf@google.com>
afl-as 2.57b by <lcamtuf@google.com>
[+] Instrumented 794 locations (64-bit, non-hardened mode, ratio 100%).
```

##### LD_PRELOADで上書き

ここまで来ると後は実行するだけです。

preenyをセットアップ, 適当なconfigファイルを用意して実行します。

```shell
$ LD_PRELOAD=/path/to/desock.so afl-fuzz -i in -o out ss-server -c config.json
```

終わり。

クラッシュするのを待ちましょう。
