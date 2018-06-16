**************
NNS开发者手册
**************

现阶段NNS在测试网已经部署完成，主网部署也即将进行，开发者可以根据本指导手册完成在自己开发的NEO钱包中对NNS的接入。

测试网NNS系统合约地址:

1. 域名中心******************** 0xd30348a37fb57b7a61f6637ecfc0da6b4eb08dd0
#. 域名中心跳板**************** 0x537758fbe85505801faa7d7d7b75b37686ad7e2d
#. 标准解析器****************** 0xd7bb680b4318f0823e968a5a067fc8bef35d13a1
#. 先到先得注册器************** 0x4e02db8842aa400a4abd77e4bdf23c54ba4aa90e
#. 拍卖注册器****************** 0xd90d82bf64083312b0b7b8dc668d633cf56899ec
#. SGAS*********************** 0xc7816d11287c08135f4e5f907af9e39754910ba3
#. nnc（包含资金池）************ 0xd8fa0cfdd54493dfc9e908b26ba165605363137b

.. toctree::
   :maxdepth: 2
   :caption: Contents:

    nns_dev_guide_nnc
    nns_dev_guide_nns
    nns_dev_guide_sgas
    nns_dev_guide_config


先到先得注册器
===================

先到先得注册器将只在测试网部署，采取先到先得的形式，用户注册后就可以直接使用，不需要支付手续费。

域名注册
---------------



域名查询解析
-------------

我们在测试网中部署的顶级域名为 **test** 顶级域名，在用户注册域名时，需要获取顶级域名的哈希值，然后拼接上用户输入的域名，详情参考:ref:`domainhash`。
获取域名注册信息的脚本构造如下：

::

    var sb = new ThinNeo.ScriptBuilder();
    sb.EmitParamJson([ "(bytes)" + domain.toHexString() ]);//第二个参数是个数组
    sb.EmitPushString("getOwnerInfo");
    sb.EmitAppCall(address);

::
    
    appCall:DAPP_REG_FIFO
    method:getOwnerInfo
    params:[domain_hash]

交易类型：InvocationTransaction

签名：无

交易执行结果：

::

   {
	"jsonrpc": "2.0",
	"id": 1,
	"result": [{
		"script": "208563abfb9556bd12e5485bb873ea4860b6c047429e580a7bb36e9e2264c12f9f51c10c6765744f776e6572496e666f6750591a2f81a506786a39d9aeb4d7ee935a284f95",
		"state": "HALT, BREAK",
		"gas_consumed": "0.968",
		"stack": [{
			"type": "Array",
			"value": [{
				"type": "ByteArray",
				"value": "dcb4dc1afa7b11d9e465a0ce2c887d97c9e7233b"
			}, {
				"type": "ByteArray",
				"value": ""
			}, {
				"type": "ByteArray",
				"value": "6bcc17c5628de5fc05a80cd87add35f0f3f1b0ab"
			}, {
				"type": "ByteArray",
				"value": "85fe115b"
			}, {
				"type": "ByteArray",
				"value": "36630a8759e5000f88500017adfa5c8fe2be32ff"
			}, {
				"type": "ByteArray",
				"value": "6a696e67687569"
			}, {
				"type": "ByteArray",
				"value": "9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08"
			}, {
				"type": "ByteArray",
				"value": ""
			}]
		}]
	}]
  } 


关键数据路径如下：

域名所有者：owner = result.stack[0]


对请求结果的解析如下：

                if (stack[ 0 ].type == "ByteArray")
                {
                    info.owner = (stack[ 0 ].value as string).hexToBytes();
                }
                if (stack[ 1 ].type == "ByteArray")
                {
                    info.register = (stack[ 1 ].value as string).hexToBytes();
                }
                if (stack[ 2 ].type == "ByteArray")
                {
                    info.resolver = (stack[ 2 ].value as string).hexToBytes();
                }
                if (stack[ 3 ].type == "Integer")
                {
                    info.ttl = new Neo.BigInteger(stack[ 3 ].value as string).toString();

                } if (stack[ 3 ].type = "ByteArray")
                {
                    let bt = (stack[ 3 ].value as string).hexToBytes();
                    info.ttl = Neo.BigInteger.fromUint8ArrayAutoSign(bt.clone()).toString();
                } if (stack[ 4 ].type = "ByteArray")
                {
                    let parentOwner = (stack[ 5 ].value as string).hexToBytes();
                } if (stack[ 5 ].type = "String")
                {
                    let domainstr = stack[ 5 ].value as string;
                } if (stack[ 6 ].type = "ByteArray")
                {
                    let parentHash = (stack[ 6 ].value as string).hexToBytes();
                } if (stack[ 7 ].type = "ByteArray")
                {
                    let bt = (stack[ 7 ].value as string).hexToBytes();
                    let root = Neo.BigInteger.fromUint8ArrayAutoSign(bt);
                }
                if (stack[ 7 ].type = "Integer")
                {
                    let a = new Neo.BigInteger(stack[ 7 ].value as string);
                }
            


其中address是域名中心的地址，domain就是用户域名的哈希结果。域名中心在解析到顶级域名为 *test* 时，会动态调用 *test* 注册器，请求响应域名的注册信息。

域名拍卖Id查询
----------------


域名注册
-----------

测试网中域名的注册过程相对简单，谁先注册域名谁就拥有这个域名的所有权。测试网注册域名首先需要获取 *test* 注册器的合约地址：

::


    var sb = new ThinNeo.ScriptBuilder();
    sb.EmitParamJson([ "(bytes)" + domain.toHexString() ]);//第二个参数是个数组
    sb.EmitPushString("getOwnerInfo");
    sb.EmitAppCall(address);

其中address是域名中心的地址，domain是 *test* 的哈希值。通过向域名管理中心发送 "getOwnerInfo" 指令并发送 *test* 的哈希值，就可以获取到 *test* 注册器的地址。

在获取到 *test* 注册器地址之后，就可以通过 *test* 注册器注册 *test* 域名:

::

    sb.EmitParamJson([ "(addr)" + address, "(bytes)" + nnshash.toHexString(), "(str)" + doamin ]);
    sb.EmitPushString("requestSubDomain");
    sb.EmitAppCall(address);

这里的address是前一步获取到的 *test* 注册器的地址，注册的命令为"requestSubDomain"。

注册域名需要三个参数:

- 第一个时域名本身
- 第二个是上一级级域名的哈希值
- 第三个是注册域名的用户的NEO账户地址

注意，在构造交易脚本是，参数倒序压栈。由于这个请求需要对用户的身份进行验证，所以本交易需要签名。


主网
===========

主网的接入比测试网复杂许多，因为主网除了会引入拍卖机制外还有为了实现代币循环系统而引入的经济模型。

域名查询
-----------

主网的域名查询和测试网一样，都是直接向域名管理中心发送查询命令，然后传入需要查询的域名的明文+上一级域名的哈希值，域名中心通过动态向上一级域名请求域名信息来获取域名的注册详情。

详情参考:ref:`domaincheck`。

域名开拍
-----------

如果用户查询的域名没有被注册或者上一个所有者没有及时续费，那么用户就可以申请开拍该域名，域名开拍合约脚本格式：

::
    
    appCall:DAPP_REG
    method:wantBuy
    params:[who,root_hash,sub_domain]

交易类型：InvocationTransaction

签名：用户签名

交易执行结果：

::

    {
	"jsonrpc": "2.0",
	"id": 1,
	"result": true
    }


加价
-----------

域名在开拍成功之后，域名将会进入一个拍卖周期。在有效的周期时间内，所有想要得到这个域名的人都可以对这个域名进行出价和加价。

为了防止有人恶意竞拍，有效拍卖时间分为一个固定的拍卖时间和一个不固定的随机时间，域名的拍卖将在随机时间内任何时刻结束。

用户要对拍卖中的域名出价，首先需要获取该域名的拍卖id。获取域名拍卖id的交易构造如下：

::

    appCall:DAPP_REG
    method:getSellingStateByFullhash
    params:[full_hash]

//    var id = info3.value.subItem[0].subItem[0].AsHash256();

完全哈希，详情参考:ref:`namehash`。

交易类型：InvocationTransaction
签名：无

交易执行结果：

::

    {
	"jsonrpc": "2.0",
	"id": 1,
	"result": [{
		"script": "20a9b961f8b37c39d969d764abff95435456ed0b5314d162edb85ba7c66e223f1951c11967657453656c6c696e675374617465427946756c6c6861736867f466384645d3e38445b8738bb7d9fa1a28665d50",
		"state": "HALT, BREAK",
		"gas_consumed": "0.843",
		"stack": [{
			"type": "Array",
			"value": [{
				"type": "ByteArray",
				"value": "8f0bfee402965aa7d718c1dc3108d9f20dd63295338938dd38ca802c0a9e23e2"
			}, {
				"type": "ByteArray",
				"value": "1ed2b38c11c70aa02adedf9fe807482472daef00689af3eeb6141346ec3f3c70"
			}, {
				"type": "ByteArray",
				"value": "6a696e67687569"
			}, {
				"type": "ByteArray",
				"value": ""
			}, {
				"type": "ByteArray",
				"value": "eb4817"
			}, {
				"type": "ByteArray",
				"value": ""
			}, {
				"type": "ByteArray",
				"value": ""
			}, {
				"type": "ByteArray",
				"value": ""
			}, {
				"type": "ByteArray",
				"value": ""
			}]
		}]
	  }]
    }

结果中，我们需要的竞拍id的解析路径为 bid_id=result['stack']['value'][0]['value']

解析到域名的拍卖地址后，通过这个地址，用户就可以参与域名的竞拍。竞拍交易脚本构造如下：

::

    appCall:DAPP_REG
    method:addPrice
    params:[
        who:竞拍人,
        bid_id：竞拍id,
        amount：出价
    ]

交易类型：InvocationTransaction
签名：用户签名

交易执行结果：

::

    {
	"jsonrpc": "2.0",
	"id": 1,
	"result": true
    }

在每次加价成功之后，竞拍合约都会重新判断最高出价者，如果多个人都加到最高价，那么先出最高价的为最高出价人。


结束竞拍
-----------

在经过固定竞拍期和随机竞拍期之后，域名拍卖就会结束。拍卖结束之后用户将不能再进行出价。

参与竞拍的竞拍人可以调用接口结束竞拍，如果是域名的拍得人，可以领取域名所有权，其余的拍卖参与人，可以取回竞拍出价的90%，剩余10%作为拍卖手续费。

结束竞拍接口如下:

::

    appCall:DAPP_REG
    method:endSelling
    params:[
        who:竞拍人,
        bid_id：竞拍id
    ]

交易类型：InvocationTransaction
签名：用户签名

交易执行结果:

::

    {
	"jsonrpc": "2.0",
	"id": 1,
	"result": true
    }




NNC
--------------

NNC是NNS系统内部为了实现SGAS循环而发布的UTXO资产，最小单位为1，不可再分割。NNC主要用在对用户注册域名而收取的手续费进行重新分配时，依据用户持有的NNC数量进行分配。


注册器充值
-----------

注册器余额查询
--------------


领取SGAS
------------



------------

~~~~~~~~~~~~~~~
