# 开发基于 MVM 的去中心化机器人

2021-12-03 04:29
字数:
1900
> 虽然我写代码很牛逼，但是能够 1 小时完成一个完全去中心化无法停止的 Mixin Messenger 机器人在一定程度上还是 MVM 这项技术的功劳。

MVM 在上个月已经正式开发完成，并且上线了完整的测试网，也形成了比较全面的 MVM 开发部署文档 ，但是普通的开发者和用户可能还是不熟悉 MVM 的开发以及到底使用体验如何，于是今天我便顺手写了一个简单的 FOMO 小游戏，点击链接或者 MM 内搜索 7000104292 即可使用。

MVM 现在同时支持 Ethereum 和 EOS 的虚拟机，文档暂时只完善了 Ethereum 相关的细节，EOS 相关的合约开发与部署还在代码审查过程中，会尽快上线。因为 MVM 开发部署文档 已经相对完善，这里不再介绍具体流程，只介绍基于 Ethereum 虚拟机的 Solidity 语言如何开发相关的逻辑。

## 合约接口

MVM 的基本架构非常普通，就是用户付款给 MVM 的多签组后，多签组编码相应的转账信息并发送给相应的 Ethereum 合约。所以使用 MVM 开发部署去中心化机器人异常简单，我只用了大概 50 分钟就完成了整个机器人的开发上线测试。一个完整的 MVM 合约，只需要实现如下事件与接口
```
event MixinTransaction(bytes);

function mixin(bytes calldata raw) public returns (bool) {}
```
其中 mixin 方法用来接受 MVM 的多签组指令，方法参数只有一个字节数组 raw，具体的编码和解析可以参考 MVM 代码里的 encoding/event.go 以及 quorum/contracts/mixin.sol

当 raw 参数在 Solidity 合约中被解析成功后，能得到一些关键参数
```
address sender; // 发起转账的用户
uint64 nonce; // 递增的序列号
uint128 asset; // 转账的 Mixin 资产 ID
uint256 amount; // 转账的 Mixin 资产数量
uint64 timestamp; // Mixin 主网时间戳
bytes extra; // 其他应用相关的信息
```
在有了这些信息后，合约具体的操作逻辑已经变得非常明显，比如我们的测试机器人要做的就是
```
// 判断用户转过来的资产对不对
if (asset != ASSET) {
  return true;
}
// 判断用户转过来的数量是不是比上次增加的
if (amount <= AMOUNT || timestamp <= TIMESTAMP) {
  return true;
}
if (TIMESTAMP == 0) {
  TIMESTAMP = timestamp;
}
// 如果时间还没过 24 小时，更新数量和时间，游戏继续
if (timestamp <= TIMESTAMP + GRACE) {
  AMOUNT = amount;
  TIMESTAMP = timestamp;
  BIDDER = sender;
  return true;
}

// 游戏过了 24 小时，那么游戏结束，把所有资金转给游戏结束前的上一个 sender
uint256 reward = custodian[asset] - amount;
bytes memory receiver = members[BIDDER];
bytes memory log = encodeMixinEvent(nonce, asset, reward, extra, receiver);
emit MixinTransaction(log);

return true;
```
在转给用户资金上，这里就用到了关键的 MixinTransaction 事件，简单来说就是把相应的转账信息编码成与上面 mixin 的参数 raw 相同的结构发布出去，具体的 Solidity 代码可以参考 encodeMixinEvent 方法。当 MVM 的多签组检测到这个事件时，会将相应的资金多签转给相应的接收人，当然这个过程会经过严格的多签组安全检查。

## 机器人交互

与 MVM 机器人的交互并不需要专门的 Ethereum 钱包，都依然是使用大家熟悉的 Mixin Messenger，同样可以使用基于 Web 的界面或者聊天交互模型。如果选择 Web 界面交互，开发者直接可以使用 Ethereum 的 web3.js 库，他们有详细的文档。

在这个测试机器人中，我选用的是聊天交互界面，所以需要写一些针对 Mixin Messenger 的聊天交互代码，并且要能够从 Ethereum 区块链上获取到实时的合约状态。我经常使用的是 Go 语言，所以选择了 FOX 团队开发的 Mixin SDK 以及 Ethereum 官方的 Go Contract Bindings。

首先要参考 abigen 的文档，给我们自己的机器人合约生成相应的绑定代码

```
abigen --abi abi.json --pkg main --type MixinProcess --out abi.go
```
然后我们的 MM 消息处理代码里就可以实时响应合约相关的状态
```
// 获取上一次加注的 XIN 的数量
asset, _ := new(big.Int).SetString(XIN, 0)
pAmount, _ := rw.proc.AMOUNT(nil)
amount := decimal.NewFromBigInt(pAmount, -8)

// 获取上一次加注的时间
pTimestamp, _ := rw.proc.TIMESTAMP(nil)
timestamp := time.Unix(0, int64(pTimestamp))
if pTimestamp == 0 {
	timestamp = time.Now()
}

// 根据加注时间计算相应的游戏结束时间
duration := 24*time.Hour - time.Now().Sub(timestamp)
if duration < 0 {
	duration = 0
}

// 获取合约里总共的 XIN 的数量
pPool, _ := rw.proc.Custodian(nil, asset)
pool := decimal.NewFromBigInt(pPool, -8)
有了相应的合约状态后，在用户发送正确的机器人聊天指令时，就可以针对性的给用户回复相应的付款链接。这个链接需要指定合适的资产类型和数量，最关键的是确保生成正确的交易 memo。具体的 memo 格式需要参考 MVM 代码里的 encoding/operation.go，比如这个测试机器人的代码

op := &encoding.Operation{
	Purpose: encoding.OperationPurposeGroupEvent,
	Process: MVMProcessId,
	Extra:   []byte("GAO"),
}
pr := mixin.TransferInput{
	Memo:    base64.RawURLEncoding.EncodeToString(op.Encode()),
}
```
然后指定正确的 MVM 多签组收款人就可以把链接回复给用户了，用户付款是直接给到了 MVM 多签组，然后多签组将相应的信息转发给相应的智能合约地址。这里的资金因为不经过任何开发者独立的机器人，而是直接到达了 MVM 多签组，在相应的智能合约开源安全的情况下，用户的资金会得到充分保障。

## 风险警告

注意使用这个机器人的时候，不需要打开机器人，只要跟机器人聊天发送任意消息即可。另外这个机器人部署在 MVM 测试网上，使用的是实际有价值的币 XIN。虽然 MVM 测试网会永久运行确保稳定可用，但是这个游戏代码本身还不严谨，大家不要过多的投入资金，测试体验 MVM 本身能够做什么即可。

代码我已经完全开源，也不希望这个项目本身能够有什么实际的作用，主要是给广大开发者一个示例和起点，可以开发出更多有价值的去中心化机器人。
