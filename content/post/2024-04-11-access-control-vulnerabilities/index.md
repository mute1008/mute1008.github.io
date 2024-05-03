+++
title = "semgrep-smart-contracts #05 (Access Control Vulnerabilities)"
date = "2024-03-11"
tags = [
    "security",
]
+++

[semgrep-smart-contracts](https://github.com/Raz0r/semgrep-smart-contracts) というリポジトリがあります。
これは、スマートコントラクトの脆弱性を発見するためのsemgrepルールをまとめたものです。

今回は Access Control Vulnerabilities (CWE-284) に関連するルールを読んでみます。
この脆弱性は事例が非常に多いため分類を試みました。
この分類は独自のものであることに注意が必要です。

#### semgrepルールの分類

スマートコントラクトにおけるアクセス制御の脆弱性において、semgrepルールを見る限りは関数の可視性に問題がある場合が多いようです。

大別してみると、以下のようになりそうです。

- 送金系の関数
- 管理者用の関数
- オラクル更新用の関数

この分類は公式なものではないことに注意してください。

##### 送金系の関数

これは送金に利用される関数が外部からアクセスされてしまう脆弱性です。

- [erc20-public-transfer.yaml](https://github.com/Decurity/semgrep-smart-contracts/blob/fb57672c3dbee3fc1417e95034d80a7a62401c4c/solidity/security/erc20-public-transfer.yaml)
- [public-transfer-fees-supporting-tax-tokens.yaml](https://github.com/Decurity/semgrep-smart-contracts/blob/fb57672c3dbee3fc1417e95034d80a7a62401c4c/solidity/security/public-transfer-fees-supporting-tax-tokens.yaml)

例えば以下のような関数があるとき、`public` が指定されているため、外部のコントラクトからの呼び出しが可能になります。

```solidity
function _transfer(
    address from,
    address to,
    uint256 amount
) public {
    ...
}
```

solidityでは、`external` / `public` / `internal` / `private` の4つのfunction visibilityを設定でき、`external` / `public` の場合は外部から関数の呼び出しが可能となります。
`external` / `public` は内部から呼び出せるか否かに違いがありますが、どちらも外部コントラクトからの呼び出しが可能です。

この脆弱性の悪用事例は以下の通り。190万ドルの損害が発生したそうです。

- [knownsec Blockchain Lab | Creat future was tragically transferred coins at will, who is the mastermind behind the scenes!](https://medium.com/@Knownsec_Blockchain_Lab/creat-future-was-tragically-transferred-coins-at-will-who-is-the-mastermind-behind-the-scenes-8ad42a7af814)

##### 管理者用の関数

これは管理者用に用意していた関数が外部からアクセスされてしまう脆弱性です。攻撃者にトークンをburnさせられたり、コントラクトのオーナーを切り替えた上でトークンを窃取することが可能になります。

- [erc20-public-burn.yaml](https://github.com/Decurity/semgrep-smart-contracts/blob/fb57672c3dbee3fc1417e95034d80a7a62401c4c/solidity/security/erc20-public-burn.yaml)
- [accessible-selfdestruct.yaml](https://github.com/Decurity/semgrep-smart-contracts/blob/fb57672c3dbee3fc1417e95034d80a7a62401c4c/solidity/security/accessible-selfdestruct.yaml)
- [unrestricted-transferownership.yaml](https://github.com/Decurity/semgrep-smart-contracts/blob/fb57672c3dbee3fc1417e95034d80a7a62401c4c/solidity/security/unrestricted-transferownership.yaml)

以下のように、ownerの移譲を行う関数が誰でも呼び出し可能な状態になっている場合、攻撃者はそのコントラクトを制御することが可能になります。

```solidity
function transferOwnership(address newOwner) public {
    ...
    _owner = newOwner;
}
```

本来、この関数には `onlyOwner` という修飾子を付与する必要があります。

この脆弱性の悪用事例は以下の通り。

- [Decoding Ragnarok Online Invasion $44,222 Exploit| QuillAudits](https://medium.com/quillhash/decoding-ragnarok-online-invasion-44k-exploit-quillaudits-261b7e23b55)

##### オラクル更新用の関数

これは同じく、オラクル更新用の関数が外部からアクセスされてしまう脆弱性です。
スマートコントラクトの文脈においてのオラクルとは、ブロックチェーン外の情報をコントラクトに提供することを指します。攻撃者がこのオラクルを書き換えることで、基準としている値が信用できなくなり、トークンを不当に安く購入することが可能になります。

- [oracle-price-update-not-restricted.yaml](https://github.com/Decurity/semgrep-smart-contracts/blob/fb57672c3dbee3fc1417e95034d80a7a62401c4c/solidity/security/oracle-price-update-not-restricted.yaml)
- [sense-missing-oracle-access-control.yaml](https://github.com/Decurity/semgrep-smart-contracts/blob/fb57672c3dbee3fc1417e95034d80a7a62401c4c/solidity/security/sense-missing-oracle-access-control.yaml)

以下のようにオラクル更新用の関数が外部コントラクトから呼び出すことが可能になっているとき、このオラクルをベースにしたトークンを不当に安く購入することが可能になります。

```solidity
function setOracleData(address rToken, oracleChainlink _oracle) external {
    oracleData[rToken] = _oracle;
}
```

この脆弱性の発見の経緯は以下の通り。

- [Sense Finance Access Control Issue Bugfix Review](https://medium.com/immunefi/sense-finance-access-control-issue-bugfix-review-32e0c806b1a0)