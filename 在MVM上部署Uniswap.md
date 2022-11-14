 在 MVM 上部署 Uniswap
MVM
MTG
Mixin
字数: 3560
MVM 的测试网上线了一段时间，经过不断优化，已经可以无需任何修改，直接部署 Ethereum 平台的代码。本文会用 Uniswap 为例介绍如何直接部署原生 Solidity 代码并让 MM 用户可以直接使用。

Ethereum 上比较活跃的应用大概有几百个，抛开 Ethereum 平台本身的性能问题，这大量的应用代码是非常有价值的，尤其是很多代码都经过了大量金钱和时间的考验，安全性上具有一定的保障。这也是为什么我们有了性能极强的 MTG 技术后，还基于 MTG 开发了 MVM，来方便开发者直接移植使用 Ethereum 上的智能合约。

在 MVM 测试网络上线后，我曾经演示过使用 Solidity 智能合约语言开发基于 MVM 的去中心化机器人 ，开发者不需要自己的服务器，直接编写智能合约部署就能完成非常复杂的去中心化应用，同时还能使用大量的 Solidity 代码。从那之后，我们就一直在尝试如何迁移一个成熟的 Ethereum 应用到 MVM，这种迁移一定不能修改原来的合约代码，这样可以让开发者的迁移成本最低，同时又能避免代码修改可能带来的安全隐患。

基于 Mixin Kernel 的强大性能，我们提出了 MTG 方案，然后基于 MTG 方案，我们开发了 MVM，然后在 MVM 上我们开发了一个 Registry 进程来直接支持 Ethereum 原生应用迁移。这个 Registry 只是 MVM 上的一个普通应用，这篇文章会用 Uniswap 的直接迁移部署为例来介绍如何使用 Registry。

我们会选择 Uniswap V2 版本来部署，这也是众多想要兼容 Ethereum 的平台经常拿来证明平台兼容性的最常见的例子。不同于直接复制 Ethereum 代码的其他项目，要实现 Uniswap 的直接部署还是需要一定的深入研究的，这也是为什么在 MVM 测试上线之后一个多月我们才有了现在的 Registry 方案，完美的解决了 Ethereum 原生应用的移植问题。

第一步首先还是参考上一篇文章配置好 MVM 测试网的 MetaMask，然后获取 Uniswap 的合约代码，第一份是 v2-core，这是 Uniswap 最核心的功能，另一份是 v2-periphery，这是在核心功能之上的一层简单的封装，给开发者提供更易用的接口。
UniswapV2Factory

这个合约是 Uniswap 的核心，也是我们第一步要部署的代码。首先将所有的 v2-core 合约代码导入到 Remix IDE，然后对代码里面的几处参数作一下修改。注意这些修改不是因为要部署到 MVM 才修改的，这只是一些针对不同网络部署的配置文件性质的修改。

第一个是将 contracts/UniswapV2ERC20.sol 中的 chainId 修改成我们测试网的网络 ID，注意这里去掉了对 assembly 的调用，是因为 Uniswap 的代码非常古老，很多新的特性在新的网络上支持不好。

     constructor() public {
-        uint chainId;
-        assembly {
-            chainId := chainid
-        }
+        uint chainId = 82397;
         DOMAIN_SEPARATOR = keccak256(

第二处修改是在 contracts/UniswapV2Factory.sol 的 constructor 中添加一个简单的 event 来方便知道 UniswapV2Pair 这个合约字节码的 keccak256 值，方便对后续的文件修改做装备。

+    event InitCode(bytes32 indexed hash);
 
     constructor(address _feeToSetter) public {
         feeToSetter = _feeToSetter;
+        bytes memory bytecode = type(UniswapV2Pair).creationCode;
+        emit InitCode(keccak256(bytecode));
     }

然后就可以直接通过 Remix 来部署 UniswapV2Factory 了，部署时就一个参数，可以直接输入自己的测试网的地址即可，等部署成功后，会得到这个合约的地址。在 MVM 测试网浏览器 搜索这个合约地址打开后，查看 logs 会得到我们添加的这个 InitCode 事件的输出结果。

像图中所示，这里我们需要的就是最下面的 0x649f 开头的那串数字，在后面部署中需要用到。
UniswapV2Router02

这个合约是辅助合约，不需要它 Uniswap 其实就已经可以运行了，但是有这个合约会让操作变得更简单。这个合约在 v2-periphery 项目中，同样的在 Remix IDE 中新建立一个项目，并把所有的合约文件导入，然后修改 contracts/libraries/UniswapV2Libary.sol 的 pairFor 方法。

    keccak256(abi.encodePacked(token0, token1)),
-   hex'96e8ac4277198ff8b6f785478aa9a39f403cb768dd02cbee326c3e7da348845f' // init code hash
+   hex'649fd46fce5aaf216609a840d1be8982f16c52f43dc9d5066901e13ed8e5d74d' // init code hash

其实就是把那一串数字换成我们上一个步骤部署 UniswapV2Factory 时得到的那个 0x649f 开头的数字，只不过要去掉 0x。

然后部署 UniswapV2Router02 这个合约，需要两个参数，factory 就是我们上一个步骤部署后的合约地址，另一个 WETH 参数可以随便使用一个 ETH 地址即可，因为我们 MVM 中并不需要 ETH 相关的操作。

至此 Uniswap V2 的所有代码都已经成功部署在了 MVM 测试网，接下来的关键是如何让 Mixin Messenger 的用户直接使用。因为 MM 一直以来做的事情是只有用户输入密码才能动用资金，不会出现像其他网络的钱包扫个码授权一下钱就都没了的情况。
UniswapMVMRouter

这个合约是我们基于 Uniswap 做了一个非常简单的封装，来让 MM 用户调用更方便，代码非常简单，直接放在 v2-periphery 项目的 contracts/UniswapMVMRouter.sol 文件里即可。

pragma solidity =0.6.6;

import '@uniswap/v2-core/contracts/interfaces/IUniswapV2Factory.sol';

import './interfaces/IUniswapV2Router02.sol';
import './interfaces/IERC20.sol';
import './libraries/UniswapV2Library.sol';

contract UniswapMVMRouter {
    address immutable router;
    uint256 constant AGE = 300;
    mapping(address => Operation) public operations;

    struct Operation {
        address asset;
        uint256 amount;
        uint256 deadline;
    }

    constructor(address _router) public {
        router = _router;
    }

    function addLiquidity(address asset, uint256 amount) public {        
        IERC20(asset).transferFrom(msg.sender, address(this), amount);
        
        Operation memory op = operations[msg.sender];
        if (op.asset == address(0)) {
            op.asset = asset;
            op.amount = amount;
            op.deadline = block.timestamp + AGE;
            operations[msg.sender] = op;
            return;
        }

        if (op.deadline < block.timestamp || op.asset == asset) {
            IERC20(op.asset).transfer(msg.sender, op.amount);
            IERC20(asset).transfer(msg.sender, amount);
            operations[msg.sender].asset = address(0);
            return;
        }

        IERC20(op.asset).approve(router, op.amount);
        IERC20(asset).approve(router, amount);
        
        uint256 amountA = op.amount;
        uint256 amountB = amount;
        uint256 liquidity;
        (amountA, amountB, liquidity) = IUniswapV2Router02(router).addLiquidity(
            op.asset, asset,
            amountA, amountB,
            amountA / 2, amountB / 2,
            msg.sender,
            block.timestamp + AGE
        );
        
        if (op.amount > amountA) {
            IERC20(op.asset).transfer(msg.sender, op.amount - amountA);
        }
        if (amount > amountB) {
            IERC20(asset).transfer(msg.sender, amount - amountB);
        }
        operations[msg.sender].asset = address(0);
    }

    function claim() public {
        Operation memory op = operations[msg.sender];
        if (op.asset == address(0)) {
            return;
        }
        IERC20(op.asset).transfer(msg.sender, op.amount);
    }
}

然后部署这个合约，参数只有一个，就是上一步骤中我们部署的 UniswapV2Router02 的合约地址。在部署成功之后，我们会得到一个新的合约地址，有了这个地址我们就可以通过 MVM 命令发起最简单的 Registry 指令了。
Registry

Registry 作为一个 MVM 进程，整个的操作调用流程是跟普通的 MVM 程序一样的。下载好 mvm 的代码编译后，随便准备一个机器人的 keystore.json 文件，就可以对我们的 Uniswap 发起操作了。

mvm invoke -m config/config.toml -k keystore.json \
	-p 60e17d47-fa70-3f3f-984d-716313fe838a \
	-asset c6d0c728-2624-429b-8e0d-d9d19b6592fa \
	-amount 0.00002 \
	-extra 7c15d0d2faa1b63862880bed982bd3020e1f1a9a5668870000000000000000000000000099cfc3d0c229d03c5a712b158a29ff186b294ab300000000000000000000000000000000000000000000000000000000000007d0

上面这个指定中的 60e17d47-fa70-3f3f-984d-716313fe838a 是我部署的测试 Registry 的进程 ID，大家可以直接使用，或者也可以自己部署 registry.sol 代码到测试网，效果是一样的。

然后的关键就是 extra 里面的内容，可以看到开头的 7c15 部分是我们上一步部署的 UniswapMVMRouter 的合约地址 0x7c15d0D2faA1b63862880Bed982bd3020e1f1A9A 去掉 0x 后全部小写，而后面 从 566887 开始则是 addLiquidity(address asset, uint256 amount) 方法的详细参数。

这其中的一个关键是 asset 参数对应的 c6d0c728-2624-429b-8e0d-d9d19b6592fa 是 BTC 在 Mixin 网络里的资产 ID，所以在 extra 参数里也是有相对应的 BTC 的合约地址，同时 amount 0.00002 也对应 extra 里的相应数字。

执行这个命令后，会得到一个二维码，然后 MM 扫码支付 0.00002BTC 即可。因为添加流动性需要两个币，下面的命令会让你添加 0.00002XIN 来配对。

mvm invoke -m config/config.toml -k keystore.json \
	-p 60e17d47-fa70-3f3f-984d-716313fe838a \
	-asset c94ac88f-4671-3976-b60a-09064f1811e8 \
	-amount 0.00002 \
	-extra 7c15d0d2faa1b63862880bed982bd3020e1f1a9a56688700000000000000000000000000bd6efc2e2cb99aef928433209c0a3be09a34f11400000000000000000000000000000000000000000000000000000000000007d0

同样的扫码支付后，就能看到流动性添加成功了。通过 MVM 测试网浏览器（https://testnet.mvmscan.com/address/0x5aD700bd8B28C55a2Cac14DCc9FBc4b3bf37679B）可以方便的查看 Registry 进程相关的所有的操作结果。

可以看到，在 MVM 上部署 Uniswap 非常简单，完全不需要修改代码，只是几个必须的配置参数的修改。同时对 Uniswap 的使用也完全符合 MM 的用户体验原则，不会出现随便扣钱的情况，添加流动性这种操作也跟 4swap 一样，只需要转账两次即可。

当然我们在 MVM 上部署的 Uniswap 还没有一个方便普通用户使用的前端界面，但是核心的智能合约代码部署后，前端其实本身并不关键，开发者可以很方便的做出适用 MM 用户体验的操作界面，而不用从头开发智能合约。
