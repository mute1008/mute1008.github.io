+++
title = "IERAE CTF 2024 Writeup"
date = "2024-09-22"
tags = [
    "security",
]
+++

IERAE CTF 2024にソロ参加しました。解けた問題は`Welcome`を除いて7問、32位でした。

#### This is warmup (pwn, warmup)

`signal(SIGSEGV, win);` となっており、SIGSEGVを発生させるのが条件。
入力に対して、以下のようなオーバーフローチェックが行われているため、これをバイパスすればSIGSEGVを発生させられる。

```c
if (nrow * ncol < nrow) { // this is integer overflow right?
    puts("Don't hack!");
    exit(1);
}
```

`nrow`と`ncol`はそれぞれ`unsinged long long`で宣言されているため、それに合わせてオーバーフローしたときに良い感じになる値を作成して終わり。

```plain
nrow : 2
ncol : ((1 << 64) + 512)/2
```

#### Copy & Paste (pwn, was_warmup, easy)

ファイルをメモリ上に読み込み、他のメモリにコピーするというアプリケーション。
これも`signal(SIGSEGV, win);`となっているため、メモリ範囲外にコピーを発生させるのが条件。

ファイルサイズの取得は`fseek`で`SEEK_END`までシークしてから、`ftell`を使ってサイズを読み取るという実装になっている。

```c
  // Get file size to allocate buffer
  fseek(fp, 0, SEEK_END);

  int size = ftell(fp);
  char *ptr = malloc(sizeof(char)*(size+1)); // plus 1 for '\0'
  if (!ptr) {
    puts("malloc failed");
    exit(1);
  }
```

この時`/proc/self/fd/0`などを指定すると`ftell`が-1を返す。
この時バッファは`malloc(0)`として確保されることになるが、ファイルサイズは`size_t`で管理されているためサイズが非常に大きなものとして扱われることとなる。

```c
struct buffer {
  size_t buf_size;
  char *buf_ptr;
};
```

コピーされるファイルサイズは以下のように決定されるため、常に-1ではない方のファイルサイズ分だけ`malloc(0)`した領域にコピーされることになる。

```c
size_t copy_size = src->buf_size;
if (dst->buf_size < copy_size) copy_size = dst->buf_size;
```

ここに`/bin/perl`などの比較的大きなバイナリをコピーすることで終わり。

#### Luz Da Lua (rev, was_warmup, easy)

調べてみるとLuaのデコンパイラーがあったので、バイナリを投げる。

- https://luadec.metaworm.site/

こんな感じの命令が並んでいることが分かる。

```
if string.len(input) ~= 28 then
  goto label_301
elseif string.byte(input, 1) ~ 232 ~= 161 then
  goto label_301
elseif string.byte(input, 2) ~ 110 ~= 43 then
  goto label_301
elseif string.byte(input, 3) ~ 178 ~= 224 then
```

上記を整形して以下のような形式にした。

```plain
232,161
110,43
178,224
172,237
212,145
```

XORして終わり。

```python
with open('flag.txt') as f:
    line = f.readline()
    while line:
        l = line.split(",")
        print(chr(int(l[0],10) ^ int(l[1],10)),end='')
        line = f.readline()
```

#### Assignment (rev, warmup)

`ELF 64-bit LSB pie executable`なファイルが渡される。 objdumpをしてみると、以下のような命令が並んでいる。

```plain
    1158:       c6 05 fd 2e 00 00 33    movb   $0x33,0x2efd(%rip)        # 405c <flag+0x1c>
    115f:       c6 05 db 2e 00 00 45    movb   $0x45,0x2edb(%rip)        # 4041 <flag+0x1>
    1166:       c6 05 d5 2e 00 00 52    movb   $0x52,0x2ed5(%rip)        # 4042 <flag+0x2>
    116d:       c6 05 e0 2e 00 00 72    movb   $0x72,0x2ee0(%rip)        # 4054 <flag+0x14>
    1174:       c6 05 df 2e 00 00 61    movb   $0x61,0x2edf(%rip)        # 405a <flag+0x1a>
```

これをvimで以下のように整形。

```plain
0,0x49
1,0x45
2,0x52
3,0x41
```

以下のコードを用いてフラグを作成した。

```python
flag = [""] * 50

with open('./flag.txt') as f:
    line = f.readline()
    while line:
        l = line.strip().split(",")
        print(chr(int(l[1],16)), end='')
        line = f.readline()

print("".join(flag))
```

#### OMG (misc, warmup)
スタートボタンをクリック、コンソールから `window.history.go(-33)` を実行。

#### derangement (crypto, warmup)
ランダムにmagic wordが作成され、ヒントの表示とmagic wordの入力が可能なアプリケーション。
magic wordの生成部分を見ると、以下のようなコードが利用されていることを確認できる。

```python
def is_derangement(perm, original):
    return all(p != o for p, o in zip(perm, original))

...

def output_derangement(magic_word):
    while True:
        deranged = ''.join(random.sample(magic_word, len(magic_word)))
        if is_derangement(deranged, magic_word):
            print('hint:', deranged)
            break
```

ヒントは複数回表示できるため、以下のようなコードを書いてmagic wordを推定した。

```python
from pwn import *

hints = []

io = remote('104.199.135.28', '55555')
# io = process(['python', 'challenge.py'])
io.recvuntil(b'> ')

for i in range(0, 299):
    io.sendline(b'1')
    hint = io.recvuntil(b'> ')
    hints.append(hint[6:6+15].decode())

words = []
for i in range(0, 15):
    charset = []
    for hint in hints:
        charset.append(hint[i])
    set(hints[0]) - set(charset)
    words.append(list(set(hints[0]) - set(charset))[0])
    print('magic word: ', "".join(words))

io.sendline(b'2')
io.recv()
io.sendline("".join(words).encode())

print(io.recv())
```

#### Futari APIs (web, warmup)

内部的な通信で利用されている`apiKey`にフラグがあり、これを取得する問題。

```javascript
const FLAG: string = Deno.env.get("FLAG") || "IERAE{dummy}";
const USER_SEARCH_API: string = Deno.env.get("USER_SEARCH_API") ||
    "http://user-search:3000";
const PORT: number = parseInt(Deno.env.get("PORT") || "3000");

async function searchUser(user: string, userSearchAPI: string) {
    const uri = new URL(`${user}?apiKey=${FLAG}`, userSearchAPI);
    return await fetch(uri);
}

async function handler(req: Request): Promise<Response> {
    const url = new URL(req.url);
    switch (url.pathname) {
        case "/search": {
            const user = url.searchParams.get("user") || "";
            return await searchUser(user, USER_SEARCH_API);
        }
        default:
            return new Response("Not found.");
    }
}

Deno.serve({ port: PORT, handler });
```

`user`パラメータは制御可能な値であるため、こちらにinteract.shなどのURLを指定すればOK。

#### 感想

面白い問題が多く、楽しかったです！
それはそれとして、見返してみると簡単な問題しか解けておらず悲しい。精進します。
