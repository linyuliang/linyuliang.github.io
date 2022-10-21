---
title: hardhat本地交易导出到tenderly分析
date: 2022-10-20
excerpt_separator: "<!--more-->"
tags:  
   - tenderly
   - hardhat
categories: 
   - 区块链  
author: linyuliang  
keywords: [hardhat,solidity,本地,debug,调试,tenderly,local]
description: 使用tenderly来debug hardhat本地的事务
toc: true
toc_sticky: true
---
本文主要是介绍使用tenderly来debug hardhat本地的事务。


参考资料：
- [Exporting a Local Transaction](https://docs.tenderly.co/debugger/exporting-a-local-transaction)

# 安装tenderly CLI
根据[tenderly官网的文档](https://github.com/Tenderly/tenderly-cli#installation)，安装tenderly CLI。
## macOS

使用[Homebrew package manager](https://brew.sh/) 来安装Tenderly CLI:

```
brew tap tenderly/tenderly
brew install tenderly
```
或者，如果您也可以使用 cURL 并运行安装脚本进行安装：:

```
curl https://raw.githubusercontent.com/Tenderly/tenderly-cli/master/scripts/install-macos.sh | sh
```

## Linux

您可以使用 cURL 并运行安装脚本来安装 Tenderly CLI：

```
curl https://raw.githubusercontent.com/Tenderly/tenderly-cli/master/scripts/install-linux.sh | sh
```

## Windows

在 [release 页面](https://github.com/Tenderly/tenderly-cli/releases), 下载最新release版本，并将exe文件目录添加到window是环境变量中 `$PATH`.

# 安装[hardhat-tenderly](https://github.com/Tenderly/hardhat-tenderly)插件
## 安装  
在hardhat工程项目根目录下，通过下面命令安装
```
npm install --save-dev @tenderly/hardhat-tenderly
```
## 配置
在 `hardhat.config.js` 或者 `hardhat.config.ts` 配置文件中增加插件的配置
```
const tdly = require("@tenderly/hardhat-tenderly");
tdly.setup();
```
若是使用typescript：
```
import * as tdly from "@tenderly/hardhat-tenderly";
tdly.setup();
```
在export的HardhatUserConfig中增加tenderly配置：
```
module.exports = {
    tenderly: {
        project: "",
        username: "",
    }
};
```
上面的`project`为tenderly中你新建的项目,可以到[tenderly dashboard](https://dashboard.tenderly.co/projects)中查看，  
`username`可以通过控制台，输入以下命令获取：
```
tenderly whoami
```
该命令返回的信息通常有：
```
ID: ***
Email: ***
Username: ***
```
需要记录的信息有`Username`和`ID`,`ID`后续还有用。  

默认hardhat-tenderly是将通过hardhat提供的ethers来发布合约的代码都做了verifyAndPublish，如果手动关闭自动verify，请参考[hardhat-tenderly插件官方文档](https://github.com/Tenderly/hardhat-tenderly)，需要多做配置和代码手动verify。

## 配置Tenderly
要使此插件正常运行，您需要创建一个 `config.yaml` 文件,目录为`$HOME/.tenderly/config.yaml`或者`%HOMEPATH%\.tenderly\config.yaml`，对于windows，以我自己的电脑为例，该配置文件路径为：`C:\Users\Administrator\.tenderly\config.yaml`，文件内容为：
```
access_key: ******
account_id: ******
username: ******
project: ******
```
- access_key 在[tenderly dashboard](https://dashboard.tenderly.co/account/authorization)中获取
- account_id 就是上面让额外记录的`ID`字段的值，通过`tenderly whoami`命令获取
- project和username 就是上面hardhat配置中相同的

# 导出本地transcation示例
## 启动本地hardhat节点
```
npx hardhat node
```
## 执行本地事务脚本
这里我以本地token部署脚本为例，部署完合约，合约token做了一笔mint交易，然后使用tenderly debug该mint交易。  
合约代码如下(这里使用可升级合约及升级合约部署脚本，不可升级合约及脚本同样测试过,OK)：
```soldity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.4;

import "@openzeppelin/contracts-upgradeable/token/ERC20/ERC20Upgradeable.sol";
import "@openzeppelin/contracts-upgradeable/token/ERC20/extensions/ERC20BurnableUpgradeable.sol";
import "@openzeppelin/contracts-upgradeable/access/OwnableUpgradeable.sol";
import "@openzeppelin/contracts-upgradeable/proxy/utils/Initializable.sol";

contract MyToken is Initializable, ERC20Upgradeable, ERC20BurnableUpgradeable, OwnableUpgradeable {
    /// @custom:oz-upgrades-unsafe-allow constructor
    constructor() {
        _disableInitializers();
    }

    function initialize() initializer public {
        __ERC20_init("MyToken", "MTK");
        __ERC20Burnable_init();
        __Ownable_init();
    }

    function mint(address to, uint256 amount) public onlyOwner {
        _mint(to, amount);
    }
}

```
部署测试脚本如下：
```typescript
import { ethers, upgrades } from "hardhat";
const hre = require("hardhat");

async function main() {
  const deployer = (await ethers.getSigners())[0];

  const MyToken = await ethers.getContractFactory("MyToken");

  const mtk = await upgrades.deployProxy(MyToken);

  await mtk.deployed();

  // await hre.tenderly.persistArtifacts({
  //   name: "MyToken",
  //   address:mtk.address
  // });

  const lockedAmount = ethers.utils.parseEther("1");

  await mtk.mint(deployer.address,lockedAmount);

  console.log(`mint ${lockedAmount} deployed to ${deployer.address}`);
}

// We recommend this pattern to be able to use async/await everywhere
// and properly handle errors.
main().catch(error => {
  console.error(error);
  process.exitCode = 1;
});
```
在本地链上执行脚本，命令如下：
```
npx hardhat run scripts/deploy_MyToken.ts --network localhost
```
## 获取事务hash，并导出到tenderly  
在本地hardhat节点的控制台上，可以看见输出事务的交易信息，例如本人执行后，如下显示：
```
eth_chainId
eth_estimateGas
eth_gasPrice
eth_sendTransaction
  Contract deployment: <UnrecognizedContract>
  Contract address:    0x4a84ecfd55fdd6a05d6af7a08e3854668ea4e657
  Transaction:         0x0e3c0b2d2e0aaf95fde333bd7049a057ec67554eaff93461394bc89879141bc1
  From:                0x0797b98884de920620dcd9d84c4f106374c6121c
  Value:               0 ETH
  Gas used:            682271 of 682271
  Block #15574682:     0x3144988904aa158a83c34916cc10007824317afabcb14aabebd47b6109752c17

eth_chainId
eth_getTransactionByHash
eth_chainId
eth_getTransactionReceipt
eth_chainId
eth_estimateGas
eth_gasPrice
eth_sendTransaction
  Contract call:       <UnrecognizedContract>
  Transaction:         0x7ed6b8e9f9a61db8a4f4e5e853f2342c2aeb7ac033923afbadfe2d74717f1c0a
  From:                0x0797b98884de920620dcd9d84c4f106374c6121c
  To:                  0x4a84ecfd55fdd6a05d6af7a08e3854668ea4e657
  Value:               0 ETH
  Gas used:            77873 of 78785
  Block #15574683:     0x6ac51ee9c95215a736b908979a1b6d586c0f1afe4281c419633f494daa8a7267

eth_chainId
eth_getTransactionByHash
```
我想要在tenderly上，debug最后一个事务的堆栈，即mint操作的事务，则，在hardhat工程根目录下，输入如下命令进行导出：
```
tenderly export init
```
在弹出提示后，输入`hardhat`,本地返回已经配置好：
```
E:\projects\hardhat-base-2.11-ts>tenderly export init
v Choose the name for the exported network: hardhat
The network [1;32mhardhat[0m is already configured. If you want to set up the network again, rerun this command with the [1;32m--re-init[0m flag.
```
这里，是我本地导出网络已经配置，若各位报错，可以考虑在hardhat根目录下，新增 `tenderly.yaml` 文件，内容如下：
```
account_id: "********"
exports:
  hardhat:
    project_slug: username/project
    rpc_address: 127.0.0.1:8545
    protocol: ""
    forked_network: Mainnet
    chain_config:
      homestead_block: 0
      eip150_block: 0
      eip150_hash: "0x0000000000000000000000000000000000000000000000000000000000000000"
      eip155_block: 0
      eip158_block: 0
      byzantium_block: 0
      constantinople_block: 0
      petersburg_block: 0
      istanbul_block: 0
      berlin_block: 0
      london_block: 0
project_slug: ""
```
其中 `account_id` project_slug 中的 `username` `project` 前面已经获取过，填写即可，另外网络根据本地实际配置。  

在export init 好之后，就可以执行export了，命令如下：
```
tenderly export 0x7ed6b8e9f9a61db8a4f4e5e853f2342c2aeb7ac033923afbadfe2d74717f1c0a --debug
```
返回如下：
```
E:\projects\hardhat-base-2.11-ts>tenderly export 0x7ed6b8e9f9a61db8a4f4e5e853f2342c2aeb7ac033923afbadfe2d74717f1c0a --debug
Trying OpenZeppelin config path: networks.js
unable to fetch config
 Couldn't read OpenZeppelin config file
couldn't read new OpenzeppelinConfig config file
Trying buidler config path: buidler.config.js
unable to fetch config
 Couldn't read Buidler config file
couldn't read new Buidler config file
Trying hardhat config path: hardhat.config.js
unable to fetch config
 Couldn't read Hardhat config file
Trying hardhat ts config path: hardhat.config.ts
Making request
Got response with body
Collecting network information...

Collecting transaction information...

Collecting contracts...
Trying Hardhat config path: E:\projects\hardhat-base-2.11-ts\hardhat.config.js
Trying Hardhat config path: E:\projects\hardhat-base-2.11-ts\hardhat.config.ts
[          ] Making request
[========= ] Got response with body
Successfully exported transaction with hash [1;32m0x7ed6b8e9f9a61db8a4f4e5e853f2342c2aeb7ac033923afbadfe2d74717f1c0a[0m
Using contracts:
        • [1;32mMyToken[0m with address [1;32m0xd9f9304329451dd31908bc61c0f87e2aa90aacd6[0m
You can view your transaction at [1;32mhttps://dashboard.tenderly.co/Lowell/project/local-transactions/8e577220-7683-4a82-ac8b-2711622e0bba[0m
```
到此，可以看到，我们的本地事务已经导出到tenderly，并且输出了可以查看的链接地址 `https://dashboard.tenderly.co/Lowell/project/local-transactions/8e577220-7683-4a82-ac8b-2711622e0bba`  

## 在tenderly 查看事务堆栈
根据上面的链接地址，查看事务堆栈，以我的交易为例，截图如下：  
![tenderly local transcation](/images/20221020/tenderlyLocalTranscation.png)  
可以看到，tenderly为我们提供了几个功能：
- 概览，事务主要信息，堆栈
- 发布的合约源码查看
  - ![contracts](/images/20221020/contracts.png)  
- 事务的合约事件查看
- 事务的链上数据状态变更查看
- 源码堆栈跟踪
  - ![debugger](/images/20221020/debugger.png)  
- gas消耗跟踪查看
  - ![gasProfiler](/images/20221020/gasProfiler.png)  

可以看到，功能还是很强大的。至此本地事务导出分析完毕，谢谢阅读，欢迎交流。
