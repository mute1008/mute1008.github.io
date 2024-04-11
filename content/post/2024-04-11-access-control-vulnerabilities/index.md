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
この分類は独自のものであること、正しく全てを網羅していないことに注意が必要です。

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

この脆弱性の悪用により、190万ドルの損害が発生しました。関数の可視性を間違えるだけでそれだけの損害が発生してしまうのは怖いですね。

- [knownsec Blockchain Lab | Creat future was tragically transferred coins at will, who is the mastermind behind the scenes!](https://medium.com/@Knownsec_Blockchain_Lab/creat-future-was-tragically-transferred-coins-at-will-who-is-the-mastermind-behind-the-scenes-8ad42a7af814)

##### 管理者用の関数

これは管理者用に用意していた関数が外部からアクセスされてしまう脆弱性です。攻撃者にトークンをburnさせられたり、コントラクトのオーナーを切り替えた上でトークンを窃取することが可能になります。

- [erc20-public-burn.yaml](https://github.com/Decurity/semgrep-smart-contracts/blob/fb57672c3dbee3fc1417e95034d80a7a62401c4c/solidity/security/erc20-public-burn.yaml)
- [accessible-selfdestruct.yaml](https://github.com/Decurity/semgrep-smart-contracts/blob/fb57672c3dbee3fc1417e95034d80a7a62401c4c/solidity/security/accessible-selfdestruct.yaml)
- [oracle-price-update-not-restricted.yaml](https://github.com/Decurity/semgrep-smart-contracts/blob/fb57672c3dbee3fc1417e95034d80a7a62401c4c/solidity/security/oracle-price-update-not-restricted.yaml)
- [unrestricted-transferownership.yaml](https://github.com/Decurity/semgrep-smart-contracts/blob/fb57672c3dbee3fc1417e95034d80a7a62401c4c/solidity/security/unrestricted-transferownership.yaml)

##### オラクル更新用の関数

これは同じく、オラクル更新用の関数が外部からアクセスされてしまう脆弱性です。
スマートコントラクトの文脈においてのオラクルとは、ブロックチェーン外の情報をコントラクトに提供することを指します。攻撃者がこのオラクルを書き換えることで、基準としている値が信用できなくなり、トークンを不当に安く購入することが可能になります。

- [sense-missing-oracle-access-control.yaml](https://github.com/Decurity/semgrep-smart-contracts/blob/fb57672c3dbee3fc1417e95034d80a7a62401c4c/solidity/security/sense-missing-oracle-access-control.yaml)

バグバウンティで指摘されたようですが、こちらもトークンをほぼ全て盗むことが可能であったことを考えると怖い話です。

- [Sense Finance Access Control Issue Bugfix Review](https://medium.com/immunefi/sense-finance-access-control-issue-bugfix-review-32e0c806b1a0)