+++
title = "semgrep-smart-contracts #04 (basic-arithmetic-underflow)"
date = "2022-06-07"
tags = [
    "security",
]
+++

[semgrep-smart-contracts](https://github.com/Raz0r/semgrep-smart-contracts) というリポジトリがあります。
これは、スマートコントラクトの脆弱性を発見するためのsemgrepルールをまとめたものです。

今回は、basic-arithmetic-underflowというルールを読んでみます。

- [basic-arithmetic-underflow.yaml](https://github.com/Raz0r/semgrep-smart-contracts/blob/master/solidity/basic-arithmetic-underflow.yaml)
- [basic-arithmetic-underflow.sol](https://github.com/Raz0r/semgrep-smart-contracts/blob/master/solidity/basic-arithmetic-underflow.sol)

#### 検出ルール

検出ルールは、以下の通りです。

このルールはまだBETAであり、まだsemgrepでは検出できません。

```yaml
pattern-sinks:
  - pattern: $Y - $X
pattern-sources:
    - pattern-either:
        - pattern-inside: |
            function $F(..., $X, ...) external { ... }
        - pattern-inside: |
            function $F(..., $X, ...) public { ... }
```

#### underflowとは

solidityの特定のバージョンでは、型の下限値を超えてデクリメントをしてしまった場合、意図しない動作を誘発する恐れがあります。

例として、solidityでuint8の変数を使うことを考えます。

このuint8の変数が0のとき、デクリメントすると255になってしまいます。

```solidity
uint8 balance = 0;
balance--; // balance == 255
```
この挙動は、[solidity >= 0.8](https://github.com/ethereum/solidity/releases/tag/v0.8.0) からデフォルトでチェックされるように修正されたそうです。

このチェックを除外してコンパイルするためには、以下のように`unchecked`という構文を利用できるようです。

```solidity
unchecked {
    balance--;
}
```

#### RemittanceTokenでの事例

この脆弱性が存在した、[RemittanceToken](https://etherscan.io/address/0xbbc3a290c7d2755b48681c87f25f9d7f480ad42f#code)というコントラクトについて解説します。

RemittanceTokenは、ERC20のトークンで、このトークンのburn関数にバグがあります。

burn関数のソースコードは以下の通りです。

```solidity
//Burn tokens from owner account
function burn(uint256 _count) public returns (bool success) {
    balanceOf[msg.sender] -=uint112( _count);
    deleteToken = _count.add(deleteToken).toUINT112();
    _totalSupply = _totalSupply.sub(_count).toUINT112();
    Burn(msg.sender, _count);
    return true;
}
```

2行目の計算にバグがあります。

```solidity
balanceOf[msg.sender] -=uint112( _count);
```

このトークンを持っていない場合、`0-_count`の計算が発生するため、underflowが発生します。

その結果、攻撃者はこのトークンを無限に手に入れることが出来ます。