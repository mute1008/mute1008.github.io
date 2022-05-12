+++
title = "semgrep-smart-contractsをすべて読む #03 (basic-reentrancy)"
date = "2022-05-12"
tags = [
    "security",
]
draft = true
+++

[semgrep-smart-contracts](https://github.com/Raz0r/semgrep-smart-contracts) というリポジトリがあります。
これは、スマートコントラクトの脆弱性を発見するための、semgrepルールをまとめたものです。

今回は、arbitrary-low-level-callというルールを読んでみます。

- [arbitrary-low-level-call.yaml](https://github.com/Raz0r/semgrep-smart-contracts/blob/master/solidity/arbitrary-low-level-call.yaml)
- [arbitrary-low-level-call.sol](https://github.com/Raz0r/semgrep-smart-contracts/blob/master/solidity/arbitrary-low-level-call.sol)

#### 検出ルール

検出ルールは、以下の通りです。

```yaml
```


#### 事例
