

CGAS合约手册
====================

基于NEO进行dapp开发的过程中，基于UTXO的GAS并不方便作为dapp的代币使用，
于是为了便于在DAPP中直接接入GAS，
NEL开发了基于NEP5的CGAS合约，用于与GAS进行一比一兑换，
搭建了应用合约与UTXO之间的桥梁。

尽管CGAS开发的初衷是为了NNS域名竞拍系统的功能实现，
但是在设计的时候便将CGAS定义为一种通用的NEP5合约，
因此任何需要在DAPP中使用GAS作为燃料的合约都可以直接使用CGAS。

除了标准的NEP5合约接口和功能之外，CGAS合约增加了CGAS和GAS互相兑换的功能。

.. note::
   CGAS原名SGAS，现由NEO官方发布及维护管理。

兑换CGAS
-----------

通过GAS兑换CGAS的步骤是:

- 用户通过转账将自己持有的GAS转到CGAS合约账户
- 将交易id传递给CGAS，并调用CGAS合约的mintTokens方法。


交易脚本结构：

::

    appCall: DAPP_CGAS
    method:  mintTokens
    params:  []
    交易类型：InvocationTransaction
    签名：    用户签名

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
            sb.EmitAppCall(DAPP_CGAS);//nep5脚本
            script = sb.ToArray();
        }
        var target = ThinNeo.Helper.GetAddressFromScriptHash(DAPP_CGAS);
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

兑换GAS的原理是用户发送兑换请求，然后从CGAS合约账户中转出指定额度的GAS到用户账户中，同时在用户账户中销毁指定额度的CGAS。

出于安全考虑，NEO合约账户并不能主动发起utxo交易，因此用户通过CGAS兑换GAS的过程相较于GAS兑换CGAS要复杂一些。

为了实现用户兑换GAS时的自动触发，用户首先需要发送交易在CGAS合约账户中生成一笔指定额度的output。

然后调用CGAS合约的refund方法，在refund方法中，会将这笔output标记为只有该用户可以领取。

用户在成功在CGAS合约账户创建output并标记之后，就可以从CGAS账户中转出这笔GAS了。

因此兑换GAS需要分为两步：

- 第一步：拆分output并标记。
- 第二步：领取GAS。

拆分output是通过CGAS合约账户自己向自己账户转账的形式构造出一个指定金额的output，然后通过调用CGAS合约的refund方法将这个output标记为只能转账给该用户。

.. warning:: 但是由于根据用户地址获取到的output很可能存在上一个共识周期里被别的用户标记过的，因此需要拆分output之前先向CGAS合约查询一下该output是否已经被标记过

脚本构造如下：

::

    appCall:DAPP_CGAS
    method:refund
    params:[who]
    交易类型：InvocationTransaction
    签名：CGAS合约签名，用户签名

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
            var shash = Config.dapp_CGAS;
            sb.EmitAppCall(shash);//nep5脚本
            script = sb.ToArray();
        }

        //CGAS 自己给自己转账   用来生成一个utxo  合约会把这个utxo标记给发起的地址使用
        tran = Helper.makeTran(newlist, CGAS_address, new Hash256(Config.id_GAS), amount);
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

.. warning:: 这里一定要获取合约脚本，并添加到签名列表。如果不添加这个合约脚本作为签名，交易将无法执行

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

在上一步中，已经成功在CGAS合约账户中创建了指定金额的GAS的output，此时，只要构造交易，从合约账户中转出这笔output就可以了。

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


充值注册器
-------------


从注册器退款
--------------