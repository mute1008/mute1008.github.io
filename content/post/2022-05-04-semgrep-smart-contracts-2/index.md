+++
title = "semgrep-smart-contractsをすべて読む #02 (arbitrary-low-level-call)"
date = "2022-05-04"
tags = [
    "security",
]
+++

[semgrep-smart-contracts](https://github.com/Raz0r/semgrep-smart-contracts) というリポジトリがあります。
これは、スマートコントラクトの脆弱性を発見するためのsemgrepルールをまとめたものです。

今回は、arbitrary-low-level-callというルールを読んでみます。

- [arbitrary-low-level-call.yaml](https://github.com/Raz0r/semgrep-smart-contracts/blob/master/solidity/arbitrary-low-level-call.yaml)
- [arbitrary-low-level-call.sol](https://github.com/Raz0r/semgrep-smart-contracts/blob/master/solidity/arbitrary-low-level-call.sol)

#### 検出ルール

検出ルールは、以下の通りです。

このルールは、betaであり、semgrepではまだ検出は出来ないようです。

```yaml
patterns:
    - pattern-either:
        - pattern-inside: |
            function $F(..., $ADDR, ...) external { ... }
        - pattern-inside: |
            function $F(..., $ADDR, ...) public { ... }
    - pattern: $ADDR.call($DATA);
```

この脆弱性は、引数で渡されたアドレスに対し、`call`関数が呼ばれた場合に検知するものです。

`call`関数は、アドレス型の関数で、別スマートコントラクトの関数を呼び出すためのものです。
詳細は以下のドキュメントを参照してください。
https://docs.soliditylang.org/en/develop/types.html#members-of-addresses

この脆弱性は、特定のスマートコントラクトやユーザーに対してのみ許可している操作を、攻撃者が任意で呼び出せてしまうことに問題があります。

攻撃者は、transferFromなどの送金用関数を自分のアドレスに対して利用することで、資金を盗むことが可能になります。

#### Li Financeでの事例

この脆弱性を悪用した攻撃が、Li Financeで発生しました。
攻撃トランザクションとエクスプロイトコードは以下の通りです。

- [Transaction](https://versatile.blocksecteam.com/tx/eth/0x4b4143cbe7f5475029cf23d6dcbb56856366d91794426f2e33819b9b1aac4e96)
- [Exploit](https://github.com/PwnedNoMore/postmortem/tree/main/2022/lifi)

問題が起こったのは、`swapAndStartBridgeTokensViaCBridge`関数です。

この関数は、`_swapData`という関数を受け取り、`callData`を引数として`callTo`に対して関数が呼び出されています。

```solidity
function swapAndStartBridgeTokensViaCBridge(
    LiFiData memory _lifiData,
    LibSwap.SwapData[] calldata _swapData,
    CBridgeData memory _cBridgeData
) public payable {
            ...

            LibSwap.swap(_lifiData.transactionId, _swapData[i]);

            ...
}
```

```solidity
    function swap(bytes32 transactionId, SwapData calldata _swapData) internal {
        ...

        (bool success, bytes memory res) = _swapData.callTo.call{ value: msg.value }(_swapData.callData);

        ...
    }
```

攻撃者は、
- `callTo`をスマートコントラクトの呼び出しが許可されているアドレスに設定
- `callData`に`transferFrom`関数を利用する

ことで、自身のアドレスへの送金を行うことが可能でした。

```solidity
_swapData[1].callTo = address(MATIC);
_swapData[1].callData = abi.encodeWithSignature(
    "transferFrom(address,address,uint256)",
    0x445C21166a3Cb20b14FA84Cfc5D122F6bd3fFa17,
    address(this),
    transferAmount(MATIC, 0x445C21166a3Cb20b14FA84Cfc5D122F6bd3fFa17)
);
```
