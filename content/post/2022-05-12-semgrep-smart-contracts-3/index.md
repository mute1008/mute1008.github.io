+++
title = "semgrep-smart-contractsをすべて読む #03 (basic-reentrancy)"
date = "2022-05-12"
tags = [
    "security",
]
+++

[semgrep-smart-contracts](https://github.com/Raz0r/semgrep-smart-contracts) というリポジトリがあります。
これは、スマートコントラクトの脆弱性を発見するためのsemgrepルールをまとめたものです。

今回は、basic-reentrancyというルールを読んでみます。

- [basic-reentrancy.yaml](https://github.com/Raz0r/semgrep-smart-contracts/blob/master/solidity/basic-reentrancy.yaml)
- [basic-reentrancy.sol](https://github.com/Raz0r/semgrep-smart-contracts/blob/master/solidity/basic-reentrancy.sol)

#### 検出ルール

検出ルールは、以下の通りです。

このルールはまだBETAであり、まだsemgrepでは検出できません。

```yaml
patterns:
- pattern-either:
        - pattern-inside: |
            function $F(..., $X, ...) external { ... }
        - pattern-inside: |
            function $F(..., $X, ...) public { ... }
- pattern-not-inside: |
    function $F(..., $X, ...) onlyOwner { ... }
- pattern: $X.$M(...)
```

#### Reentrancy attackとは

reentrant (再入可能性) とは、ある関数の実行中に、同じ関数を実行しても安全であるという性質のことを指します。

スマートコントラクトにおいては、攻撃対象から、任意のコントラクトを呼び出し可能な場合にReentrancy attackが発生する恐れがあります。

この攻撃は、以下の論文で詳細に解説されています。

[Reentrancy Vulnerability Identification in Ethereum Smart Contracts](https://arxiv.org/abs/2105.02881)

論文では、
- Single Function Reentrancy attack
- Cross Function Reentrancy attack

という２つの攻撃に分類されています。

##### fallback / receive関数

この攻撃を理解する前に、前提条件としてfallback / receive関数について知る必要があります。

receive関数は、スマートコントラクトが送金を受け取った時に呼び出される関数です。
receive関数が定義されていないなどで呼び出せなかった場合には、fallback関数が呼び出されます。

それぞれ、以下のような特殊な関数として記述されます。

```solidity

contract Sample {
    receive() external payable {
    }
    fallback() external payable {
    }
}
```

##### Single Function Reentrancy attack

Single Function Reentrancy attackは、単一の関数で発生するReentrancy attackです。

具体例として、以下のようなスマートコントラクトを考えます。
これは、預け入れたものをwithdraw関数で引き出すことが可能なスマートコントラクトになります。

```solidity
// 被害者コード
contract ContractA {
    ...

    // 引き出し
    function withdraw() external {
        receiver.transfer(amount);
        balances[receiver] -= amount;
    }

    ...
}
```

攻撃のためのエクスプロイトコードは以下の通りです。

receiveという関数があり、その関数内では他コントラクトが呼び出されています。

```solidity
// 攻撃者コード
contract ContractB {
    receive() external payable {
        contractA.withdraw();
    }
}
```

エクスプロイトの流れを以下に記しました。

```

        1 ) 攻撃者がcontractBを呼び出し

        2 ) contractBがwithdraw関数を呼び出し

        3 ) contractAに制御が移り、送金が行われる

        4 ) 送金により、contractBのreceive関数が呼び出される

        5 ) receive関数内で、再度withdraw関数の呼び出し

        6 ) 繰り返し

        ...
        ...
        ...

```

上記に記載した流れが続くことで、攻撃者は無限に引き出しを行うことが出来ます。

これは、状態を更新する前に送金を行ってしまっているために起こっています。

以下のように修正することで、攻撃を防ぐことが可能になります。

```solidity
contract ContractA {
    ...

    // 引き出し
    function withdraw() external {
        balances[receiver] -= amount;
        receiver.transfer(amount);
    }

    ...
}
```

##### Cross Function Reentrancy attack

Cross Function Reentrancy attackも基本は同じで、状態を共有する2つの関数がある場合に起こります。

具体例として、以下のようなスマートコントラクトがあることを考えます。

```solidity
mapping (address => uint) private balance;

function transfer(address to, uint amount) {
    if (balance[msg.sender] >= amount){
        balance[to] += amount;
        balance[msg.sender] -= amount;
    }
}

function withdraw() public {
    uint amount = balance[msg.sender];
    require(msg.sender.call.value(amount)());
    /* At this point, the caller’s code is executed, and can call transfer() */
    balance[msg.sender] = 0;
}
```

攻撃者がwithdrawを呼び出した段階で残高が更新されていないため、receive関数がtransferを呼び出すことで資金の抜き取りが可能となります。
