 让 MetaMask 支持 Mixin 机器人
Mixin
MVM
MTG
字数: 4482
在 MVM 带来的大融合中曾经提到 Mixin 的一个目标是让所有钱包用户都能用上 Mixin Network 的各种功能，比如快速的转账和使用 Mixin 的各种机器人。

在 MVM 带来的大融合 中曾经提到 Mixin 的一个目标是让所有钱包用户都能用上 Mixin Network 的各种功能，比如快速的转账甚至是直接使用 Mixin 上的各种机器人。那篇文章也大概介绍了基本的思路和原理，最近我们着手实现了 相应的代码 并且做了一个 简单的部署 来方便开发者使用。

现在这篇文章会面向 Mixin 机器人开发者，重点讲解一下如何让一个 Mixin 机器人支持 MetaMask 钱包，至于具体的实现原理不再展开介绍，可以直接参考项目代码。文章会使用现有的 API 和工具来进行所有操作，而开发者要做的就是把相应的操作变成代码添加到自己的机器人中。
配置 MetaMask

首先得有一个 MetaMask 钱包，开发调试最方便的是使用一个浏览器插件，然后访问 MVM 浏览器 在页面最底部点击 Add MVM 将 Mixin Virtual Machine 网络添加到钱包里。

将钱包的当前网络切换到 Mixin Virtual Machine 之后，创建或者选择自己之前的任何一个账户，为了方便讲解，这里我选择的账户地址是 0x914DFf811EF12267e1b644d9cb9B65743B98B131。

有了地址之后，需要在我们的跨链桥上进行注册，请使用以下命令，注意需要替换成你自己的 MetaMask 账户地址。

curl -H 'Content-Type: application/json' https://bridge.mvm.dev/users \
    -d '{"public_key":"0x914DFf811EF12267e1b644d9cb9B65743B98B131"}'

这个命令会返回一个跨链代理账户信息，要对 MetaMask 里的账户进行任何资产的转入都可以通过这个代理 Mixin 用户进行操作。

{"user":{"user_id":"3da8327c-aabf-3636-a8ab-3b7990f9bf60","identity_number":"0","full_name":"0x914DFf811EF12267e1b644d9cb9B65743B98B131","mute_until":"0001-01-01T00:00:00Z","created_at":"2022-06-22T11:08:22.48765745Z","session_id":"db3dcd62-4029-4b45-9bd6-a09f88262e1d","pin_token":"Fy95T15E6omn3XIYUz7dXnY2Xbn196nFbRMgpsNxF1A=","receive_message_source":"EVERYBODY","accept_conversation_source":"EVERYBODY","accept_search_source":"EVERYBODY","fiat_currency":"USD","key":{"client_id":"3da8327c-aabf-3636-a8ab-3b7990f9bf60","session_id":"db3dcd62-4029-4b45-9bd6-a09f88262e1d","private_key":"uD4ulS_Tvm1KvO6w4_VZ4oGid9fhczMb1lHV_kbZEo0FK2xntcTELe0khjs5dYdCrGzIy9ttoRIf2wanKTYMXg","pin_token":"Fy95T15E6omn3XIYUz7dXnY2Xbn196nFbRMgpsNxF1A=","scope":""},"contract":""}}

充值资产

有了一个 MetaMask 的 MVM 账户之后，第一步是转入一些 XIN 来作为基本的 Gas 费用。我们使用 FOX 团队提供的 mixin-cli 小工具来得到我们代理用户的相关跨链信息。首先是将上一步 JSON 文件中的 key 部分完整取出保存为一个文件比如 3da8327c.json

{"client_id":"3da8327c-aabf-3636-a8ab-3b7990f9bf60","session_id":"db3dcd62-4029-4b45-9bd6-a09f88262e1d","private_key":"uD4ulS_Tvm1KvO6w4_VZ4oGid9fhczMb1lHV_kbZEo0FK2xntcTELe0khjs5dYdCrGzIy9ttoRIf2wanKTYMXg","pin_token":"Fy95T15E6omn3XIYUz7dXnY2Xbn196nFbRMgpsNxF1A=","scope":""}

然后使用如下命令得到这个账户的 XIN 充值地址，其中的 c94ac88f-4671-3976-b60a-09064f1811e8 是 XIN 在 Mixin 中的资产 ID。

mixin-cli -f 3da8327c.json http GET /assets/c94ac88f-4671-3976-b60a-09064f1811e8

命令会返回这个代理用户全部的 XIN 资产状态，包括充值地址 deposit_entries，其中的 detination 和 tag 可以添加到自己的钱包里进行充值操作。

{
  "type": "asset",
  "asset_id": "c94ac88f-4671-3976-b60a-09064f1811e8",
  "chain_id": "43d61dcd-e413-450d-80b8-101d5e903357",
  "symbol": "XIN",
  "name": "Mixin",
  "icon_url": "https://mixin....PCBECc-Ws=s128",
  "balance": "0",
  "deposit_entries": [
    {
      "destination": "0x3AAf7DdF6FF02353deaECF786210205f60D72184",
      "tag": ""
    }
  ],
  "destination": "0x3AAf7DdF6FF02353deaECF786210205f60D72184",
  "tag": "",
  "price_btc": "0.00771532",
  "price_usd": "157.74564135",
  "change_btc": "0.013477576662793707",
  "change_usd": "-0.024294683015992063",
  "asset_key": "0xa974c709cfb4566686553a20790685a47aceaa33",
  "mixin_id": "a99c2e0e2b1da4d648755ef19bd95139acbbe6564cfb06dec7cd34931ca72cdc",
  "reserve": "0",
  "confirmations": 16,
  "capitalization": 0,
  "liquidity": "0"
}

对上述地址充值完毕后，可以再用同样的方式得到 CNB 作为测试币的地址，并转入 100CNB 作为后续测试资产。值得注意的是，这个地址可以通过 Ethereum 原生区块链直接转入，也可以通过 Mixin Network 相关的钱包直接瞬间转入。

转入的 XIN 可以直接在 MetaMask 里显示，但是 CNB 需要手动导入资产，如果不清楚具体操作流程可以参考 MetaMask 的官方文章。其中 MVM 上 CNB 的合约地址是 0x910Fb1751B946C7D691905349eC5dD250EFBF40a。

注意不需要转入过多的 XIN，因为 MVM 的 Gas 价格很低，转入 0.01XIN 就足够使用很久，而且可以随时从 MM 钱包免费转入。
调用机器人

当 MetaMask 的 MVM 账户里有了资产之后，就可以方便的直接调用任何 Mixin 里面的机器人了。每个机器人有不同的操作方式，但是所有操作都是通过接受用户转入资产来实现的。所以首先需要得到机器的收款信息以及相应的 memo 格式，然后编码成下面的 JSON 格式。

type Action struct {
	Receivers   []string `json:"receivers,omitempty"`
	Threshold   int64    `json:"threshold,omitempty"`
	Extra       string   `json:"extra,omitempty"`
}

其中的 receivers 和 threshold 代表收款人的多签信息，而 extra 代表转账的 memo 数据。假设我们的机器人收款是单个用户 fcb87491-4fa0-4c2f-b387-262b63cbc112，而 memo 是 Hello from MVM，那我们可以得到如下 JSON。

{"receivers":["fcb87491-4fa0-4c2f-b387-262b63cbc112"],"threshold":1,"extra":"Hello from MVM"}

有了上面的信息后，还需要跟跨链桥相关的一些数据统一编码，为了方便讲解，我们直接将上面的 JSON 信息发送到跨链桥提供的一个方便的 API。

curl -H 'Content-Type: application/json' https://bridge.mvm.dev/extra \
    -d '{"receivers":["fcb87491-4fa0-4c2f-b387-262b63cbc112"],"threshold":1,"extra":"Hello from MVM"}'

API 会成功返回如下数据，我们需要返回的 extra 数据来调用 MVM 上的智能合约。

{"extra":"bd67087276ce3263b9333aa337e212a4ef241988d19892fe4eff4935256087f4fdc5ecaa65034ffb618e84db0ba8da744ef9e8e4de63637893444e12e1b318c670dfec527b22726563656976657273223a5b2266636238373439312d346661302d346332662d623338372d323632623633636263313132225d2c227468726573686f6c64223a312c226578747261223a2248656c6c6f2066726f6d204d564d227d"}

同时我们需要再次调用之前注册用户的命令，来得到用户在 MVM 上注册对应的合约地址。

curl -H 'Content-Type: application/json' https://bridge.mvm.dev/users \
    -d '{"public_key":"0x914DFf811EF12267e1b644d9cb9B65743B98B131"}'

返回的数据如下，可以看到跟上次调用的数据几乎一致，只是 contract 字段多出了一个地址 0x5b6e9BdeD656F70314f9CE818aFF2Af83a2D2f0D。

{"user":{"user_id":"3da8327c-aabf-3636-a8ab-3b7990f9bf60","identity_number":"0","full_name":"0x914DFf811EF12267e1b644d9cb9B65743B98B131","mute_until":"0001-01-01T00:00:00Z","created_at":"2022-06-22T11:08:22.48765745Z","session_id":"db3dcd62-4029-4b45-9bd6-a09f88262e1d","pin_token":"Fy95T15E6omn3XIYUz7dXnY2Xbn196nFbRMgpsNxF1A=","has_pin":true,"receive_message_source":"EVERYBODY","accept_conversation_source":"EVERYBODY","accept_search_source":"EVERYBODY","fiat_currency":"USD","key":{"client_id":"3da8327c-aabf-3636-a8ab-3b7990f9bf60","session_id":"db3dcd62-4029-4b45-9bd6-a09f88262e1d","private_key":"uD4ulS_Tvm1KvO6w4_VZ4oGid9fhczMb1lHV_kbZEo0FK2xntcTELe0khjs5dYdCrGzIy9ttoRIf2wanKTYMXg","pin_token":"Fy95T15E6omn3XIYUz7dXnY2Xbn196nFbRMgpsNxF1A=","scope":""},"contract":"0x5b6e9BdeD656F70314f9CE818aFF2Af83a2D2f0D"}}

然后我们打开浏览器上对应的 CNB 智能合约地址，然后切换到 Write Contract 页面，找到最后的 transferWithExtra 方法。方法有 3 个参数，其中每个参数如下：

    to(address): 0x5b6e9BdeD656F70314f9CE818aFF2Af83a2D2f0D，也就是我们前面 JSON 里得到的 contract 字段。
    value(uint256): 123456789，因为所有跨链资产的小数点精度都是 8，所以这个数字代表 1.23456789CNB。
    extra(bytes): 0xbd67087276ce3263b9333aa337e212a4ef241988d19892fe4eff4935256087f4fdc5ecaa65034ffb618e84db0ba8da744ef9e8e4de63637893444e12e1b318c670dfec527b22726563656976657273223a5b2266636238373439312d346661302d346332662d623338372d323632623633636263313132225d2c227468726573686f6c64223a312c226578747261223a2248656c6c6f2066726f6d204d564d227d，这也就是上面 extra API 返回的数据，注意返回的数据前面没有 0x，我们需要添加上。

当所有数据填写完毕，就可以点击 write 按钮唤起 MetaMask 钱包进行签名了。这里需要注意的一点是，如果有多个 MetaMask 账户，要确保在 MVM 浏览器写入合约界面显示连接的账号是最开始使用的 MVM 账号地址。

最后检查自己的机器人账号应该收到了一笔 1.23456789CNB 的转账，并且 memo 恰好是 Hello from MVM。
正式开发

上面的流程展示了如何让 MetaMask 通过 MVM 直接访问 Mixin 内现存的各种机器人，但是这个流程不可能让普通用户来一步步操作，所以需要相关机器人的开发者在自己的机器人前端界面添加相应的逻辑来方便用户一键连接 MetaMask 钱包登录，并通过页面上的点击来直接操作机器人。

当然开发者只需要关注调用机器人这一步的内容的开发，前面的 MetaMask 钱包配置与资产充值，我们会开发相应的界面来让用户方便操作和管理。
