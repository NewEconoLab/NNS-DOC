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

如果用户查询的域名没有被注册或者上一个所有者没有及时续费，那么用户就可以申请开拍该域名：

::
    
    var result = await nns_common.api_SendTransaction(prikey, reg_sc, "wantBuy",
                "(hex160)" + who.ToString(),
                "(hex256)" + roothash.ToString(),
                "(string)" + subname
                );

代码位置为NEL开发的neo-ThinSDK-cs中的smartContractDemo项目:

- **prikey** 用户私钥 byte[]
- **reg_sc** 域名注册器地址 Hash160
- **wantBuy** 申请开拍指令 string
- **who** 申请人地址 hex160
- **roothash** 注册器合约地址,获取方式见:ref hex256
- **subname** 申请人想申请的域名 string


加价
-----------

域名在开拍成功之后，域名将会进入一个拍卖周期。在有效的周期时间内，所有想要得到这个域名的人都可以对这个域名进行出价和加价。

为了防止有人恶意竞拍，有效拍卖时间分为一个固定的拍卖时间和一个不固定的随机时间，域名的拍卖将在随机时间内任何时刻结束。

用户要对拍卖中的域名出价，首先需要获取该域名的拍卖id。获取域名拍卖id的交易构造如下：

::

    //得到拍卖ID
    var info3 = await nns_common.api_InvokeScript(reg_sc, "getSellingStateByFullhash", "(hex256)" + fullhash.ToString());
    var id = info3.value.subItem[0].subItem[0].AsHash256();
    var who = this.scriptHash;

- **reg_sc** 域名注册器地址 byte[]
- **getSellingStateByFullhash** 获取拍卖id的命令 string
- **fullhash** 指定域名的完全哈希，详情参考:ref:`namehash`。hex256

这个接口可以获取到指定域名的拍卖地址，通过这个地址，用户就可以参与域名的竞拍。竞拍交易如下：

::

    var result = await nns_common.api_SendTransaction(this.prikey, reg_sc, "addPrice",
          "(hex160)" + who.ToString(),//参数1 who
          "(hex256)" + id.ToString(),//参数2 交易id
          "(int)1"+"00000000"//参数3，加价多少
          );

- **prikey** 用户私钥 byte[]
- **reg_sc** 注册器地址 Hash160
- **参数1** 用户地址 hex160
- **参数2** 交易id hex256
- **参数3** 加价金额 Biginteger

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

通过GAS兑换SGAS的原理是:

- 用户向SGAS合约地址转GAS
- 将交易id发送给SGAS

兑换GAS
-----------

注册器充值
-----------

注册器余额查询
--------------


领取SGAS
------------



------------

~~~~~~~~~~~~~~~