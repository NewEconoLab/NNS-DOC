**************
NNS开发者手册
**************

现阶段NNS在测试网已经部署完成，主网部署也即将进行，开发者可以根据本指导手册完成在自己开发的NEO钱包中对NNS的接入。

在NNS的架构中，我们采用了一个域名管理中心合约来对不同的顶级域名进行管理，测试网顶级域名管理中心合约地址：

::

    0x954f285a93eed7b4aed9396a7806a5812f1a5950

测试网
===========

在测试网中，域名的注册使用先到先得的方式进行，没有拍卖的过程。在用户输入域名之后，调用接口查询域名是否已注册，如果已注册则显示注册信息，如果没有注册，则允许用户进行注册。

由于测试网中注册域名没有拍卖过程，所以测试网的域名注册不需要SGAS以及拍卖等复杂操作。

.. _domaincheck:

域名查询解析
-------------

我们在测试网中部署的顶级域名为 **test** 顶级域名，在用户注册域名时，需要获取顶级域名的哈希值，然后拼接上用户输入的域名，详情参考:ref:`domainhash`。
获取域名注册信息的脚本构造如下：

::

    var sb = new ThinNeo.ScriptBuilder();
    sb.EmitParamJson([ "(bytes)" + domain.toHexString() ]);//第二个参数是个数组
    sb.EmitPushString("getOwnerInfo");
    sb.EmitAppCall(address);

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

    var result = await nns_common.api_SendTransaction(prikey, reg_sc, "endSelling",
        "(hex160)" + who.ToString(),//参数1 who
        "(hex256)" + id.ToString()//参数2 交易id
        );

- **prikey** 用户私钥 byte[]
- **reg_sc** 注册器地址 Hash160
- **参数1** 用户地址 hex160
- **参数2** 交易id hex256

SGAS
-------------

SGAS是NNS系统内发布的NEP5资产，与GAS进行1:1绑定，用户可以通过SGAS合约用GAS换取SGAS用于域名拍卖，同时也可以通过SGAS合约将持有的SGAS兑换成等量GAS。
同时由于SGAS是NEP5资产，所以支持NEP5标准的所有接口。

NNC
--------------

NNC是NNS系统内部为了实现SGAS循环而发布的UTXO资产，最小单位为1，不可再分割。NNC主要用在对用户注册域名而收取的手续费进行重新分配时，依据用户持有的NNC数量进行分配。

兑换SGAS
-----------

通过GAS兑换SGAS的步骤是:

- 用户通过转账将自己持有的GAS转到SGAS合约账户
- 将交易id传递给SGAS，并调用sgas合约的mintTokens方法。

交易类型：InvocationTransaction
转账对象：DAPP_SGAS合约
签名：用户签名

交易脚本结构：

::

    appCall:DAPP_SGAS
    method:mintTokens
    params:[]

交易构造示例代码：

::

    Transaction tran = null;
    {
        byte[] script = null;
        using(var sb = new ScriptBuilder())
        {
            var array = new MyJson.JsonNode_Array();
            sb.EmitParamJson(array);//参数倒序入
            sb.EmitParamString("mintTokens");//参数倒序入
            sb.EmitAppCall(DAPP_SGAS);//nep5脚本
            script = sb.ToArray();
        }
        var target = ThinNeo.Helper.GetAddressFromScriptHash(DAPP_SGAS);
        subPrintLine("contract address=" + target);//往合约地址转账

        //生成交易
        tran = Helper.makeTran(dir[Config.id_GAS], target, new Hash256(Config.id_GAS), amount);
        tran.type = TransactionType.InvocationTransaction;
        var idata = new InvokeTransData();
        tran.extdata = idata;
        idata.script = script;

        // sign and broadcast
        var signdata = ThinNeo.Helper.Sign(tran.GetMessage(), prikey);
        tran.AddWitness(signdata, pubkey, address);
        var trandata = tran.GetRawData();
        var strtrandata = ThinNeo.Helper.Bytes2HexString(trandata);
        byte[] postdata;
        var url = Helper.MakeRpcUrlPost(Config.api, "sendrawtransaction", out postdata, new MyJson.JsonNode_ValueString(strtrandata));
        var result = await Helper.HttpPost(url, postdata);
        var json = MyJson.Parse(result).AsDict();
        if (json.ContainsKey("result")) {
            var resultv = json["result"].AsList()[0].AsDict();
            var txid = resultv["txid"].AsString();
            subPrintLine("txid=" + txid);
        }
    }

执行结果

::

    {
	"jsonrpc": "2.0",
	"id": 1,
	"result": [{
		"sendrawtransactionresult": true,
		"txid": "0x8d2e558b848cfad1049943e5799b7d34fd85090a5b301ae2a1a12de76455cb0e"
	  }]
    }


兑换GAS
-----------

兑换GAS的原理是用户发送兑换请求，然后从SGAS合约账户中转出指定额度的GAS到用户账户中，同时在用户账户中销毁指定额度的SGAS。

出于安全考虑，NEO合约账户并不能主动发起utxo交易，因此用户通过SGAS兑换GAS的过程相较于GAS兑换SGAS要复杂一些。

为了实现用户兑换GAS时的自动触发，用户首先需要发送交易在SGAS合约账户中生成一笔指定额度的output。

然后调用SGAS合约的refund方法，在refund方法中，会将这笔output标记为只有该用户可以领取。

用户在成功在SGAS合约账户创建output并标记之后，用户就可以从SGAS账户中转出这笔GAS了。

因此兑换GAS需要分为两步：
- 第一步：拆分output并标记。
- 第二步：领取GAS。

拆分output是通过SGAS合约账户自己向自己账户转账的形式构造出一个指定金额的output，然后通过调用SGAS合约的refund方法将这个output标记为只能转账给该用户。

交易类型：InvocationTransaction
签名：SGAS合约签名，用户签名

脚本构造如下：

::

    appCall:DAPP_SGAS
    method:refund
    params:[who]

交易构造示例代码：

::

    Transaction tran = null;
    {
        byte[] script = null;
        using (var sb = new ScriptBuilder())
        {
            var array = new MyJson.JsonNode_Array();
            array.AddArrayValue("(bytes)" + ThinNeo.Helper.Bytes2HexString(scriptHash));
            sb.EmitParamJson(array);//参数倒序入
            sb.EmitParamJson(new MyJson.JsonNode_ValueString("(str)refund"));//参数倒序入
            var shash = Config.dapp_sgas;
            sb.EmitAppCall(shash);//nep5脚本
            script = sb.ToArray();
        }

        //sgas 自己给自己转账   用来生成一个utxo  合约会把这个utxo标记给发起的地址使用
        tran = Helper.makeTran(newlist, sgas_address, new Hash256(Config.id_GAS), amount);
        tran.type = TransactionType.InvocationTransaction;
        var idata = new InvokeTransData();
        tran.extdata = idata;
        idata.script = script;

        //附加鉴证
        tran.attributes = new ThinNeo.Attribute[1];
        tran.attributes[0] = new ThinNeo.Attribute();
        tran.attributes[0].usage = TransactionAttributeUsage.Script;
        tran.attributes[0].data = scriptHash;
    }

    // 智能合约签名
    // ...
    // 提款人签名
    // ...

执行结果：

::

   {
	"jsonrpc": "2.0",
	"id": 1,
	"result": [{
		"sendrawtransactionresult": true,
		"txid": "0x58a5d2b134fbe4662ae964fb53d5d66a0ff7c1aa7d588b8d406494d8e3c455c5"
	  }]
    }

交易在新一轮的共识中，如果交易验证成功，就可以进行GAS兑换的第二步了。

在上一步中，已经成功在SGAS合约账户中创建了指定金额的GAS的output，此时，只要构造交易，从合约账户中转出这笔output就可以了。

交易需要的output的构造如下

::

    Utxo utxo = new Utxo(address, txid, Config.id_GAS, amount, 0);

交易类型：ContactTransaction
签名：合约签名

执行结果：

::
    {
	"jsonrpc": "2.0",
	"id": 1,
	"result": [{
		"sendrawtransactionresult": true,
		"txid": "0x132bd0164411fd6ef1ed2223dce40ca07f00f91a79af070b2336aa80c49252e8"
	  }]
    }


注册器充值
-----------

注册器余额查询
--------------


领取SGAS
------------



------------

~~~~~~~~~~~~~~~

接口标准
===========

变量类型
-------

域名拍卖id	hex256
用户地址	hex160
域名哈希	hex256
合约地址	hex160
用户资产    BigIngeger

变量/参数定义
-----------

常量
~~~~~~~~

SGAS合约：DAPP_SGAS=>hex160
NNC合约： DAPP_NNC =>hex160
注册器：  DAPP_REG =>hex160


转账
~~~~~~~

转账源地址：who     =>hex160
转账目的地址：target =>hex160
转账金额：amount =>BigIngeger

域名拍卖
~~~~~~~~~~

拍卖id ：bid_id =>hex256
域名哈希：domain_hash => hex256
根域名哈希：root_hash =>hex256
子域名：sub_domain =>string
域名完全哈希：full_hash=>hex256

数据结构
------------

NNS合约的对外统一接口有两个参数，一个是用于接收具体命令的，另一个是接收Object类型的参数列表的，所有合约的接口定义都如下：

::

    public static object Main(string method, object[] args)

因此调用NNS合约的数据结构可描述为：

{
    appCall：合约地址
    method：合约方法
    params：参数列表
}


封装
-------


+----------+------------+
|          |            |
|          |            |
+----------+------------+
|
|
+----------+-----------



