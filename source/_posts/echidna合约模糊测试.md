---
title: echidna智能合约模糊测试  
date: 2022-09-13
excerpt_separator: "<!--more-->"  
tags:  
   - echidna
   - 智能合约  

categories: 
   - 区块链  

author: linyuliang  
description: echidna简单入门  
toc: true  
toc_sticky: true  
---

Echidna是一个快速的智能合约（solidity）模糊测试框架，它是用Haskell语言编写的程序，实现基于以太坊智能合约属性的模糊测试。它使用基于合约ABI的复杂的grammar-based模糊测试来验证用户定义断言或者Solidity断言。  
Echidna在设计时考虑了模块化，可以很容易地被扩展成包含新的变种或者在特定情况下，测试特定合约。  

# 特性：  
- 生成根据实际代码生成定制化输入
- 可选的语料库集合，变种和覆盖指引，以便发现更深层次的错误
- [Slither](https://github.com/crytic/slither) 加持，在模糊测试之前提取有用的信息
- 集成源码，以识别哪些代码行在模糊测试后已经被覆盖
- Curses-based retro UI、text-only或JSON输出
- 快速分类的自动最小化测试用例
- 与开发工作流无缝集成
- 模糊测试的最大gas消耗报告
- 支持使用 [Etheno](https://github.com/crytic/etheno) 和Truffle来初始化复杂的合约  

此文只是Echidna的简单入门介绍，具体学会使用还是要看官方的[Echidna 教程](https://github.com/crytic/building-secure-contracts/tree/master/program-analysis/echidna#echidna-tutorial) 。  

参考资料：
- [Echidna 官方github ](https://github.com/crytic/echidna) （基本翻译自官方github文档）
- [Echidna：功能强大的以太坊模糊测试框架](https://www.freebuf.com/articles/blockchain-articles/211940.html)  
<!-- more -->
# 使用方法
## 执行测试任务
Echidna的核心功能通过一个名叫echidna-test的可执行文件实现，echidna-test以一个智能合约和合约中的一组不变量（不变量：以echidna_开头、没有参数并返回布尔值的实体函数，返回的值应该始终保持为true）作为输入。对于每一个不变量，它将生成一套针对智能合约的随机调用序列，并检查不变量是否保持不变。如果它能够找到某种方法来篡改不变量（即,使返回值为false），它就会打印出这样做（使结果为false）的的调用序列。如果不能找到，则判断为该智能合约是安全的。
## 编写不变量
不变量是以echidna_开头的solidity函数，该函数没有参数，并且返回布尔值。例如，如果你有一个```balance```变量，该变量应当一直不小于```20```，则你可以在你的合约中添加一个额外的函数：
```solidity
function echidna_check_balance() public returns (bool) {
  return(balance >= 20);
}
```
进行不变量的测试，可以执行如下命令：
```shell
$ echidna-test myContract.sol
```
在[tests/solidity/basic/flags.sol.](https://github.com/crytic/echidna/blob/master/tests/solidity/basic/flags.sol) 是一个例子合约。你可以执行如下命令进行测试：
```shell
$ echidna-test tests/solidity/basic/flags.sol
```
在上面这个例子中，Echidna 可以找到一个调用序列，让```echidna_sometimesfalse```的结果为false，并且不可能找到一个伪造输入，让```echidna_alwaystrue```的结果返回false。
## 支持合约的编译系统
Echidna可以使用[crytic-compile](https://github.com/crytic/crytic-compile) 来测试由不同的合约编译系统编译的智能合约，包括[Truffle](https://truffleframework.com/) 或者[hardhat](https://hardhat.org/) 。使用```echidna-test .```可以在当前的编译框架环境下调用echidna。

最重要的是，Echidna 支持两种测试复杂合约的模式。  
1. 可以用[Truffle和Etheno描述合约的初始化过程](https://github.com/crytic/building-secure-contracts/blob/master/program-analysis/echidna/end-to-end-testing.md) ，并将其用作Echidna的基本状态
2. echidna看可以通过在CLI命令中传入对应的Solidity源，来调用已知ABI的任意合约。（在你的配置中使用```multi-abi: true```来打开这个特性）

## Echidna速成
在官方的[Building Secure Smart Contracts](https://github.com/crytic/building-secure-contracts/tree/master/program-analysis/echidna#echidna-tutorial) 仓库，包含了Echidna的速成课程，包含例子，课程文档和练习题。
## 在Github Actions工作流中集成Echidna
Echidna Action可以作为Github Actions工作流的一部分，来执行```echidna-test```。有关的使用说明和示例，可以参考[crytic/echidna-action](https://github.com/crytic/echidna-action) 代码库。
## 配置项
Echidna 的 CLI命令可以用来选择测试合约，并且加载配置文件：
```shell
$ echidna-test contract.sol --contract TEST --config config.yaml
```
配置文件允许用户选择 EVM 和测试生成参数。可以在[tests/solidity/basic/default.yaml](https://github.com/crytic/echidna/blob/master/tests/solidity/basic/default.yaml) 中找到具有完整默认选项且带有注释的配置文件示例。有关配置选项的更详细文档可以查看官方的[wiki](https://github.com/trailofbits/echidna/wiki/Config) 。   

Echidna 支持三种不同的输出模式。有默认的文本模式、json模式和none模式（禁止所有```stdout```输出）。 JSON 模式按如下方式报告整个活动。
```javascript
Campaign = {
  "success"      : bool,
  "error"        : string?,
  "tests"        : [Test],
  "seed"         : number,
  "coverage"     : Coverage,
  "gas_info"     : [GasInfo]
}
Test = {
  "contract"     : string,
  "name"         : string,
  "status"       : string,
  "error"        : string?,
  "testType"     : string,
  "transactions" : [Transaction]?
}
Transaction = {
  "contract"     : string,
  "function"     : string,
  "arguments"    : [string]?,
  "gas"          : number,
  "gasprice"     : number
}
```

其中```Coverage```是一个描述某些覆盖率增加的调用的字典。每个```GasInfo```条目都是一个元组，它描述了如何达到最大的gas使用量，而且也不太重要。这些接口可能会在以后的某个日期进行更改，以便稍微更友好一些。```testType```可以是```property```或```assertion```，而```status```的值是```fuzzing```、```shrinking```、```solved```、```passed```或```error```。

## Echidna调试性能问题  
处理Echidna性能问题的最佳方法是运行```echidna-test```时，开启分析。这样的话，会生成一个文本文件```echidna-test.prof```,该文件显示了哪些函数占用了最多的CPU和内存。  
要构建一个支持分析的```echidna-test```版本，可以使用Stack或者Nix。  
1. 使用Stack，添加标记```--profile```将使构建的版本支持分析；
2. 使用Nix，运行```nix-build --arg profiling true```将使构建的版本支持分析。  

在运行时，要打开分析，需要在执行```echidna-test```命令时，需要增加```+RTS -p```标记。

过去的性能问题是因为函数在可以被缓存时被重复调用，并且内存泄漏与 Haskell 的惰性求值有关；检查这些将是一个很好的起点。  
## 限制和已知问题  
EVM 仿真和测试很难。 Echidna 在最新版本中有许多限制。 其中一些是限制从[hevm](https://github.com/dapphub/dapptools/tree/master/src/hevm) 继承的，一些是处于设计/性能的考量，或者是代码中的bug。在这里将它们列出来，包含对应的issue和状态（"wont fix", "on hold", "in review", "fixed"）。“fixed”状态的Issues将会被纳入到下一个Echidna发布的版本中。  

| 描述 | 问题链接 | 状态 |
| -------- | -------- | -------- |
| Vyper支持存在一些限制 | [#652](https://github.com/crytic/echidna/issues/652) | wont fix |
| 对Library的测试支持存在限制 | [#651](https://github.com/crytic/echidna/issues/651) | wont fix |
| 缺乏对Solidity中对函数指针的支持 | [#798](https://github.com/crytic/echidna/issues/798) | on hold |

# 安装
## 预编译的二进制文件
在开始前，请确保安装了[Slither](https://github.com/crytic/slither) (```pip3 install slither-analyzer --user```)。如果你想在Linux或者MacOS中快速测试Echidna，可以在[官方的release页面](https://github.com/crytic/echidna/releases) 上找到基于Ubuntu构建的linux二进制包的静态链接库和基于MacOS的二进制包。  
## Docker容器
如果选择了使用预编译好的docker容器，可以在[docker包页面](https://github.com/orgs/crytic/packages?repo_name=echidna) 获得，这些包都是通过Github Actions自动编译的。```echidna```容器是基于```ubuntu:focal```镜像，该镜像是一个刚好可以使用Echidna的小而灵活的镜像。该容器提供了```echidna-test```的预构建版本，以及```slither```、```crytic-compile```、```solc-select``` 和 ```nvm```，总大小在200 MB以下。  

请注意，这些容器镜像当前仅构建在 x86 系统上。由于 CPU 仿真会导致性能损失，因此不建议在 ARM 设备（例如 Mac M1 系统）上运行它们。  

Docker容器镜像的不同Tag含义：

| Tag | 含义 |
| -------- | -------- |
| ```vx.y.z``` | 对应版本 ```vx.y.z``` 的构建 |
| ```latest``` |  最新的Echidna release版本 |
| ```edge``` |  默认分支上的最新提交版本 |
| ```testing-foo``` |  基于 ```foo``` 分支测试构建 |

要以交互方式运行最新 Echidna 版本的容器，可以使用类似于以下命令。该命令将当前目录映射到容器内的 ```/src```，并为您提供一个shell，您可以在其中使用```echidna-test```：
```shell
$ docker run --rm -it -v `pwd`:/src ghcr.io/crytic/echidna/echidna
```
如果是windows环境，```pwd```要使用全路径，例如：
```shell
$ docker run --rm -it -v "E:\projects\hardhat-template":/src ghcr.io/crytic/echidna/echidna
```
如果固定solidity版本测试，可以在启动时，就将对应solidity版本安装好（安装solidity可能需要搭梯子）：
```shell
$ docker run --rm -it -v "E:\projects\hardhat-template":/src ghcr.io/crytic/echidna/echidna
$ solc-select install 0.8.9 && solc-select use 0.8.9
```
然后就可以使用```echidna-test```，对映射到容器```/src```目录下的合约进行测试：
```shell
$ echidna-test ./src/contracts/Greeter.sol
```
或者，如果想在本地构建最新版本的Echidna，建议使用 Docker。在该克隆下来的代码库中，运行以下命令来构建 Docker 容器映像：
```shell
$ docker build -t echidna -f docker/Dockerfile --target final-ubuntu .
```
然后，你可以再本地运行```echidna ```镜像。例如，安装solc 0.5.7 并且测试```tests/solidity/basic/flags.sol```,可以执行如下命令：
```shell
$ docker run -it -v `pwd`:/src echidna bash -c "solc-select install 0.5.7 && solc-select use 0.5.7 && echidna-test /src/tests/solidity/basic/flags.sol"
```
## 使用Stack构建
如果您希望从源代码构建，请使用 [Stack](https://docs.haskellstack.org/en/stable/README/) 。应该在 ```~/.local/bin``` 目录下使用```stack install``` 构建和编译```echidna-test```。 您需要链接 libreadline 和 libsecp256k1（在recovery enabled的情况下构建），它们应该用您选择的包管理器安装。还需要安装最新版本的[libff](https://github.com/scipr-lab/libff) ，可以参考官方的[CI测试脚本](https://github.com/crytic/echidna/blob/master/.github/scripts/install-libff.sh) 。  
一些 Linux 发行版没有为 Haskell 需要的某些东西提供静态库，例如。 Arch Linux，这将导致```stack build```执行失败并出现链接错误，因为我们使用了```-static```参数。在这种情况下，可以使用```--flag echidna:-static```参数来生成一个动态链接库。  
如果您遇到与链接相关的构建错误，请尝试修改```--extra-include-dirs```和```--extra-lib-dirs```。
## 使用Nix构建（在Apple M1系统上工作）  
[Nix users](https://nixos.org/download.html) 可以使用如下的命令安装最新的 Echidna：
```shell
$ nix-env -i -f https://github.com/crytic/echidna/tarball/master
```
要为非 Nix的macOS 系统构建独立版本，使用一下命令将Echidna 和所有链接的 dylib 打包到一个 tarball 中：
```shell
$ nix-build macos-release.nix
$ ll result/
bin    echidna-1.7.3-aarch64-darwin.tar.gz
```
可以在```nix-shell```中使用 Cabal 开发 Echidna。Nix 将自动安装开发所需的所有依赖项，包括```crytic-compile```和```solc```。以下命令是一个让```GHCi```与```Echidna```一起工作的快速方法：
```shell
$ git clone https://github.com/crytic/echidna
$ cd echidna
$ nix-shell
[nix-shell]$ cabal new-repl
```
执行测试套件：
```shell
nix-shell --run 'cabal test'
```
# 已公开的使用
## 属性测试使用方
这是使用 Echidna 进行测试的智能合约项目的部分列表：  
* [Uniswap-v3](https://github.com/search?q=org%3AUniswap+echidna&type=commits)
* [Balancer](https://github.com/balancer-labs/balancer-core/tree/master/echidna)
* [MakerDAO vest](https://github.com/makerdao/dss-vest/pull/16)
* [Optimism DAI Bridge](https://github.com/BellwoodStudios/optimism-dai-bridge/blob/master/contracts/test/DaiEchidnaTest.sol)
* [WETH10](https://github.com/WETH10/WETH10/tree/main/contracts/fuzzing)
* [Yield](https://github.com/yieldprotocol/fyDai/pull/312)
* [Convexity Protocol](https://github.com/opynfinance/ConvexityProtocol/tree/dev/contracts/echidna)
* [Aragon Staking](https://github.com/aragon/staking/blob/82bf54a3e11ec4e50d470d66048a2dd3154f940b/packages/protocol/contracts/test/lib/EchidnaStaking.sol)
* [Centre Token](https://github.com/centrehq/centre-tokens/tree/master/echidna_tests)
* [Tokencard](https://github.com/tokencard/contracts/tree/master/tools/echidna)
* [Minimalist USD Stablecoin](https://github.com/usmfum/USM/pull/41)
## 成果
以下安全漏洞都是使用Echidna发现的。如果您使用我们的工具发现了安全漏洞，请提交包含相关信息的 PR。  

| Project | Vulnerability | Date |
| -------- | -------- | -------- |
| [0x Protocol](https://github.com/trailofbits/publications/blob/master/reviews/0x-protocol.pdf) | If an order cannot be filled, then it cannot be canceled | Oct 2019 |
| [0x Protocol](https://github.com/trailofbits/publications/blob/master/reviews/0x-protocol.pdf) | If an order can be partially filled with zero, then it can be partially filled with one token | Oct 2019 |
| [0x Protocol](https://github.com/trailofbits/publications/blob/master/reviews/0x-protocol.pdf) | The cobbdouglas function does not revert when valid input parameters are used | Oct 2019 |
| [Balancer Core](https://github.com/trailofbits/publications/blob/master/reviews/BalancerCore.pdf) | An attacker cannot steal assets from a public pool | Jan 2020 |
| [Balancer Core](https://github.com/trailofbits/publications/blob/master/reviews/BalancerCore.pdf) | An attacker cannot generate free pool tokens with joinPool | Jan 2020 |
| [Balancer Core](https://github.com/trailofbits/publications/blob/master/reviews/BalancerCore.pdf) | Calling joinPool-exitPool does not lead to free pool tokens | Jan 2020 |
| [Balancer Core](https://github.com/trailofbits/publications/blob/master/reviews/BalancerCore.pdf) |  Calling exitswapExternAmountOut does not lead to free assets | Jan 2020 |
| [Liquity Dollar](https://github.com/trailofbits/publications/blob/master/reviews/Liquity.pdf) | [Closing troves require to hold the full amount of LUSD minted](https://github.com/liquity/dev/blob/echidna_ToB_final/packages/contracts/contracts/TestContracts/E2E.sol#L242-L298) | Dec 2020 |
| [Liquity Dollar](https://github.com/trailofbits/publications/blob/master/reviews/Liquity.pdf) | [Troves can be improperly removed](https://github.com/liquity/dev/blob/echidna_ToB_final/packages/contracts/contracts/TestContracts/E2E.sol#L242-L298) | Dec 2020 |
| [Liquity Dollar](https://github.com/trailofbits/publications/blob/master/reviews/Liquity.pdf) | Initial redeem can revert unexpectedly | Dec 2020 |
| [Liquity Dollar](https://github.com/trailofbits/publications/blob/master/reviews/Liquity.pdf) | Redeem without redemptions might still return success | Dec 2020 |
| [Origin Dollar](https://github.com/trailofbits/publications/blob/master/reviews/OriginDollar.pdf) | Users are allowed to transfer more tokens that they have | Nov 2020 |
| [Origin Dollar](https://github.com/trailofbits/publications/blob/master/reviews/OriginDollar.pdf) | User balances can be larger than total supply | Nov 2020 |
| [Yield Protocol](https://github.com/trailofbits/publications/blob/master/reviews/YieldProtocol.pdf) | Arithmetic computation for buying and selling tokens is imprecise | Aug 2020 |

## 研究
还可以使用 Echidna 从智能合约模糊测试论文中复制研究示例，以演示它可以多快找到解决方案。所有这些都可以在笔记本电脑上完成，用时从几秒钟到一两分钟不等。  

| Source | Code |
| -------- | -------- |
|[Using automatic analysis tools with MakerDAO contracts](https://forum.openzeppelin.com/t/using-automatic-analysis-tools-with-makerdao-contracts/1021) | [SimpleDSChief](https://github.com/crytic/echidna/blob/master/tests/solidity/research/vera_dschief.sol) |
|[Integer precision bug in Sigma Prime](https://github.com/b-mueller/sabre#example-2-integer-precision-bug) | [VerifyFunWithNumbers](https://github.com/crytic/echidna/blob/master/tests/solidity/research/solcfuzz_funwithnumbers.sol) |
|[Learning to Fuzz from Symbolic Execution with Application to Smart Contracts](https://files.sri.inf.ethz.ch/website/papers/ccs19-ilf.pdf) | [Crowdsale](https://github.com/crytic/echidna/blob/master/tests/solidity/research/ilf_crowdsale.sol) |
|[Harvey: A Greybox Fuzzer for Smart Contracts](https://arxiv.org/abs/1905.06944) | [Foo](https://github.com/crytic/echidna/blob/master/test/solidity/research/harvey_foo.sol), [Baz](https://github.com/crytic/echidna/blob/master/tests/solidity/research/harvey_baz.sol) |

## 学术刊物

| Paper Title | Venue | Publication Date |
| --- | --- | --- |
| [echidna-parade: Diverse multicore smart contract fuzzing](https://agroce.github.io/issta21.pdf) | [ISSTA 2021](https://conf.researchr.org/home/issta-2021) | July 2021 |
| [Echidna: Effective, usable, and fast fuzzing for smart contracts](https://agroce.github.io/issta20.pdf) | [ISSTA 2020](https://conf.researchr.org/home/issta-2020) | July 2020 |
| [Echidna: A Practical Smart Contract Fuzzer](papers/echidna_fc_poster.pdf) | [FC 2020](https://fc20.ifca.ai/program.html) | Feb 2020 |

如果您使用 Echidna 用于学术工作，可以考虑申请[Crytic的1万美元研究奖](https://blog.trailofbits.com/2019/11/13/announcing-the-crytic-10k-research-prize/) 。

# 寻求帮助
可以随时访问官方在[Empire Hacking](https://empireslacking.herokuapp.com/) 中的#ethereum slack 频道，以获取使用或扩展 Echidna 的帮助。
* 从查看这些简单 [Echidna 不变量](tests/solidity/basic/flags.sol) 开始
* 考虑直接向 Echidna 开发团队发送[电子邮件](mailto:echidna-dev@trailofbits.com) 

# 版权
Echidna 在 [AGPLv3 许可](https://github.com/crytic/echidna/blob/master/LICENSE) 下获得许可和分发。

