+++
title = "論文紹介: Memory-Safety Challenge Considered Solved? An In-DepthExperience Report with All Rust CVEs"
date = "2020-08-09"
tags = [
    "paper",
    "security",
]
+++

ふとRustで書かれたソフトウェアでは絶対にメモリ系のバグが起こらないのか気になりました。
メモリ系のバグが起こらないとしたらファジングで脆弱性を見つけることは出来ないがパニックするパターンは発見できる？
その辺りがちょっと気になっていた所、この論文[1]を見つけたので読みました。
正確な翻訳ではなく、ちょっとしたまとめとただの感想です。

この論文ではRustに存在する脆弱性のリストを調査して, メモリ安全性を達成出来るのかどうかを調査している。
リストは以下の２つ。

[Advisory-db](https://github.com/RustSec/advisory-db)

[Trophy-cases](https://github.com/rust-fuzz/trophy-case)

このリストを調査したところ、Rustによってuse-after-free, double-freeの問題を新たに引き起こしてしまう場合があるということを言っている。

今回はUse-After-Free, Double-Free辺りの問題を見てみる。

#### Use-After-Free
コンストラクタにunsafeを使った場合にUse-After-Freeが発生するということを言ってる。

例として以下のコードを考える。

このコードでは、コンストラクタではポインタの引数をdereferenceして参照を返す。
しかし、関数の終了時にこの参照が開放されてしまうので、mainで参照するとUse-After-Freeが発生する。
```rust
fn test() -> Vec<u8> {
  let mut s = String::from("lifetime_test");
  let ptr = s.as_mut_ptr();
  unsafe {
    let v = Vec::from_raw_parts(ptr, s.len(), s.len());
    v
  }
}

fn main() {
  let v = test();
  assert_eq!('l' as u8, v[0]); /*fail*/
}
```

#### Double-Free
Double-Freeはいろいろ起こりうるシチュエーションはあるみたいだけど、とりあえず以下のコードを考える。
srcがfun1のスコープを抜けるときに開放され、mainのスコープを抜けるときにfooの開放をしようとしてdouble-freeが発生する。
```rust
impl Drop for Foo {
  fn drop(&mut self) { println!("Dropping: {}",self.s); }
}
struct Foo {s: String}
/*fix2:struct Foo {s: mem::ManuallyDrop<String>}*/

fn fun2(mut src: &mut String) -> Foo {
  let s = unsafe{ String::from_raw_parts(src.as_mut_ptr(), src.len(), 32) };
  Foo{s:s}
}

fn fun1() -> Foo {
  let mut src = String::from("Dropped twice!");
  let foo = fun2(&mut src);
  /*fix1:std::mem::forget(src);*/
  foo
}

fn main() {
  let foo = fun1();
}
```

double-freeを発生させるだけならfun1だけでいける気がしたけどなんでfun2に分離されているんだろう。詳しい人いたら教えてください。
```rust
impl Drop for Foo {
    fn drop(&mut self) { println!("Dropping: {}",self.s); }
}

struct Foo {s: String}

fn fun1() -> Foo {
    let mut src = String::from("Dropped twice!");
    let s = unsafe{ String::from_raw_parts(src.as_mut_ptr(), src.len(), 32) };
    Foo{s:s}
}

fn main() {
    let foo = fun1();
}
```

#### まとめ
この論文では現在のRustプロジェクトではunsafeを利用するのが一般的[2]で, unsafeとRustのメモリ管理があわさったときに脆弱性が発生しやすいということを言っている。

1番初めの問いである、Rustプロジェクトではファジングによって脆弱性が見つからないか、というのはNoと言えそう。

---

[1] Xu, Hui, et al. "Memory-Safety Challenge Considered Solved? An Empirical Study with All Rust CVEs." arXiv preprint arXiv:2003.03296 (2020).

[2] Evans, Ana Nora, Bradford Campbell, and Mary Lou Soffa. "Is Rust Used Safely by Software Developers?." arXiv preprint arXiv:2007.00752 (2020).
