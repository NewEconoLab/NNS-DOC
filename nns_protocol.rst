*********
详细设计
*********

NNS协议规范
===========

通常我们在互联网上使用的url如下

::

    http://aaa.bbb.test/xxx 

其中：

 - http是协议(protocol)，NNS服务请求时会把域名和协议分开传递
 - aaa.bbb.test是域名，NNS服务请求时使用域名的hash
 - xxx是路径，路径不是在dns的层次处理，对于nns也一样，如果有路径，交由其他的方式处理

NNS的协议使用字符串定义

以下均为暂定

http协议
--------

**http** 协议指向一个string,表示一个互联网地址

addr协议
--------

**addr** 协议指向一个string，表示一个NEO address，形如 AdzQq1DmnHq86yyDUkU3jKdHwLUe2MLAVv

script协议
----------

**script** 协议指向一个byte[],表示一个NEO ScriptHash，形如0xf3b1c99910babe5c23d0b4fd0104ee84ffeec2a5

同一域名的不同协议做不同处理

**http://abc.test** 可以指向 http://www.163.com

**addr://abc.test** 可以指向 AdzQq1DmnHq86yyDUkU3jKdHwLUe2MLAVv

**script://abc.test** 可以指向 0xf3b1c99910babe5c23d0b4fd0104ee84ffeec2a5

.. _namehash:

NameHash算法详解
===============

NNS中存储的域名为32字节散列值，而不是域名原文的文本。这有几个设计原因：

 - 处理过程统一，允许任意长度的域名。 
 - 一定程度保留了域名的隐私 
 - 将域名转换为散列的算法称为NameHash

域名协议
--------

通常我们在互联网上使用的url如下

::

    http://aaa.bbb.test/xxx 

其中：

1. http是协议(protocol)，NNS服务请求时会把域名和协议分开传递

2. aaa.bbb.test是域名，NNS服务请求时使用域名的hash

3. xxx是路径，路径不是在dns的层次处理，对于nns也一样，如果有路径，交由其他的方式处理

NNS服务使用的不是域名，而是域名的名字数组，这样处理起来更加直接

域名 aaa.bb.test 转成字节数组就是["test","bb","aa"]

你可以这样调用解析

::

        NNS.ResolveFull("http",["test","bb","aa"]);

交由合约去计算出namehash ## NameHash算法
NameHash算法是将域名转成DomainArray以后，逐级连接计算hash的方法，代码如下:

::

        //域名转hash算法
        static byte[] nameHash(string domain)
        {
            return SmartContract.Sha256(domain.AsByteArray());
        }
        static byte[] nameHashSub(byte[] roothash, string subdomain)
        {
            var domain = SmartContract.Sha256(subdomain.AsByteArray()).Concat(roothash);
            return SmartContract.Sha256(domain);
        }
        static byte[] nameHashArray(string[] domainarray)
        {
            byte[] hash = nameHash(domainarray[0]);
            for (var i = 1; i < domainarray.Length; i++)
            {
                hash = nameHashSub(hash, domainarray[i]);
            }
            return hash;
        }

快速解析
-------

完整的解析传入整个DomainArray,由智能合约去逐个检查一层层解析。
计算NameHash的过程也可以挪到客户端算好，再传入智能合约。 调用方式如下

::

    //查询 http://aaa.bbb.test
    var hash = nameHashArray(["test","bbb"]);//可以客户端计算
    NNS.Resolve("http",hash,"aaa");//调用智能合约

或者

::

    //查询 http://bbb.test
    var hash = nameHashArray(["test","bbb"]);//可以客户端计算
    NNS.Resolve("http",hash,"");//调用智能合约

你也许会考虑查询 aaa.bbb.test 的过程为什么不是这样

::

    //查询 http://aaa.bbb.test
    var hash = nameHashArray(["test","bbb","aaa"]);//可以客户端计算
    NNS.Resolve("http",hash,"");//调用智能合约

我们要考虑aaa.bb.test
是否拥有一个独立的解析器，如果aaa.bb.test被卖给了别人，他指定了一个独立的解析器，这样是可以查询到的。
如果aaa.bb.test 并没有独立的解析器，他是有bb.test的解析器来解析。
那么这样就无法查询到

而采用第一种查询方式，无论aaa.bb.test
是否拥有一个独立的解析器，都可以查询到。

顶级域名合约详解
===============

顶级域名合约的函数入口如下:

::

    public static object Main(string method, object[] args)

部署时采用 参数 0710，返回值 05 的配置

顶级域名的接口分为三部分

 - **通用接口** 不需要权限验证，所有人都可以调用
 - **所有者接口** 仅接受所有者签名，或者由所有者脚本调用有效
 - **注册器接口** 仅接受由注册器脚本调用有效

通用接口
---------

通用接口，不需要权限验证。 代码如下

::

    if (method == "rootName")
        return rootName();
    if (method == "rootNameHash")
        return rootNameHash();
    if (method == "getInfo")
        return getInfo((byte[])args[0]);
    if (method == "nameHash")
        return nameHash((string)args[0]);
    if (method == "nameHashSub")
        return nameHashSub((byte[])args[0], (string)args[1]);
    if (method == "nameHashArray")
        return nameHashArray((string[])args[0]);
    if (method == "resolve")
        return resolve((string)args[0], (byte[])args[1], (string)args[2]);
    if (method == "resolveFull")
        return resolveFull((string)args[0], (string[])args[1]);

根域名
~~~~~~

::

    rootName()

返回当前顶级域名合约对应的根域名 返回值为string

顶级域名哈希
~~~~~~~~~~~

::

    rootNameHash()

返回当前顶级域名合约对应的NameHash 返回值为byte[]

域名信息
~~~~~~~~

::
    
    getInfo(byte[] namehash)

返回一个域名的信息 返回值为一个如下数组

::

    [
        byte[] owner//所有者
        byte[] register//注册器
        byte[] resolver//解析器
        BigInteger ttl//到期时间
    ]

单级域名哈希
~~~~~~~~~~~

::

    nameHash(string domain)

将域名的一节转换为NameHash 比如

::

    nameHash("test") 
    nameHash("abc")

返回值为byte[]

子域名哈希
~~~~~~~~~

::

    nameHashSub(byte[] domainhash,string subdomain)

计算子域名的NameHash, 比如

::

    var hash = nameHash("test");
    var hashsub = nameHashSub(hash,"abc")//计算abc.test 的namehash

返回值为byte[]

域名哈希
~~~~~~~~

::

     nameHashArray(string[] nameArray)

计算NameArray的NameHash，aa.bb.cc.test,
对应的nameArray是["test","cc","bb","aa"]

::

    var hash = nameHashArray(["test","cc","bb","aa"]);

域名解析
~~~~~~~~

::

    resolve(string protocol,byte[] hash,string or int(0) subdomain)

- **protocol** 协议类型。比如http是将域名映射为一个网络地址，addr是将域名映射为一个NEO地址（这可能是最常用的映射）
- **hash** 要解析的域名Hash
- **subdomain** 要解析的子域名Name

应用代码如下

::

    var hash = nameHashArray(["test","cc","bb","aa"]);//客户端计算好
    resolve("http",hash,0)//合约解析 http://aa.bb.cc.test

    or

    var hash = nameHashArray(["test","cc","bb");//客户端计算好
    resolve("http",hash,“aa")//合约解析 http://aa.bb.cc.test

返回类型为byte[]，具体byte[]如何解读，由不同的协议定义，addr协议，byte[]就存的字符串。
协议约定另外撰文。

除了二级域名的解析，必须使用resolve("http",hash,0)的方式，其余的解析建议都使用resolve("http",hash,"aa")的方式。

域名完整解析
~~~~~~~~~~~~

::

    resolveFull(string protocol,string[] nameArray)

解析域名，完整模式

- **protocol** 协议类型
- **nameArray** 要解析的域名数组

这种解析方式唯一的不同就是会逐级验证一下所有权是否和登记的一致，一般用resolve即可

返回类型同resolve

所有者接口
---------

所有者接口全部为 owner\_SetXXX(byte[] srcowner,byte[] nnshash,byte[]
xxx)的形式。 xxx 均是scripthash。

返回值均为 一个byte array [0] 表示失败 [1] 表示成功

所有者接口均接受账户地址直接签名调用和智能合约所有者调用。

如果所有者是智能合约，那么所有者应该自己判断权限，不满足条件，不要发起对顶级域名合约的appcall

域名转让
~~~~~~~~

::

    owner\_SetOwner(byte[] srcowner,byte[] nnshash,byte[] newowner)

转让域名所有权，域名的所有者可以是一个账户地址，也可以是一个智能合约。

- **srcowner** 仅在 所有者是账户地址时用来验证签名，他是地址的scripthash
- **nnshash** 是要操作的域名namehash
- **newowner** 是新的所有者的地址的scripthash

域名注册
~~~~~~~~

::

    owner\_SetRegister(byte[] srcowner,byte[] nnshash,byte[] newregister)

设置域名注册器合约（域名注册器为一个智能合约）

域名注册器参数形式必须也是0710，返回05

必须实现如下接口：

::

        public static object Main(string method, object[] args)
        {
            if (method == "getSubOwner")
                return getSubOwner((byte[])args[0], (string)args[1]);
            ...

        getSubOwner(byte[] nnshash,string subdomain)

任何人可调用注册器的接口检查子域的所有者。

对于域名注册器的其他接口形式不做规范，官方提供的注册器会另外撰文说明。

用户自己实现的域名注册器，仅需实现getSubOwner接口

域名解析
~~~~~~~~

::

    owner\_SetResolve(byte[] srcowner,byte[] nnshash,byte[] newresolver)

设置域名解析器合约（域名解析器为一个智能合约）

域名解析器参数形式必须也是0710，返回05

必须实现如下接口

::

        public static byte[] Main(string method, object[] args)
        {
            if (method == "resolve")
                return resolve((string)args[0], (byte[])args[1]);
            ...
        
        resolve(string protocol,byte[] nnshash)

任何人可调用解析器接口进行解析

对于域名解析器的其它接口形式不做规范，官方提供的解析器会另外撰文说明。

用户自己实现的域名解析器，仅需实现resolve 接口

注册器接口
----------

注册器接口由注册器智能合约进行调用，只有一个

::

    register\_SetSubdomainOwner(byte[] nnshash,string subdomain,byte[] newowner,BigInteger ttl)

注册一个子域名

 - **nnshash** 是要操作的域名namehash
 - **subdomain** 是要操作的子域名
 - **newowner** 是新的所有者的地址的scripthash
 - **ttl** 是域名过期时间（区块高度）

成功返回 [1] ,失败返回 [0]

所有者合约详解
=============

所有者合约工作方式
-----------------

所有者合约采用Appcall的形式调用顶级域名合约的owner_SetXXX 接口
::

        [Appcall("dffbdd534a41dd4c56ba5ccba9dfaaf4f84e1362")]
        static extern object rootCall(string method, object[] arr);

顶级域名合约会检查调用栈，取出调用它的合约和顶级域名合约管理的所有者进行比对。

所以只有所有者合约可以实现管理。

所有者合约的意义
===============

用户可以用所有者合约实现复杂的合约所有权模式

比如：

- 双人共有，双人签名操作。
- 多人共有，投票操作。

注册器合约详解
==============
                                         
注册器合约采用Appcall的形式调用顶级域名合约的register\_SetSubdomainOwner接口

::

        [Appcall("dffbdd534a41dd4c56ba5ccba9dfaaf4f84e1362")]
        static extern object rootCall(string method, object[] arr);

顶级域名合约会检查调用栈，取出调用它的合约和顶级域名合约管理的注册器进行比对。

所以只有指定的注册器合约可以实现管理。

合约入口
--------

注册器参数形式必须也是0710，返回05

::

        public static object Main(string method, object[] args)
        {
            if (method == "getSubOwner")
                return getSubOwner((byte[])args[0], (string)args[1]);
            if (method == "requestSubDomain")
                return requestSubDomain((byte[])args[0], (byte[])args[1], (string)args[2]);
            ...

查询拥有者
---------

::

    getSubOwner(byte[] nnshash,string subdomain)

此接口为注册器规范要求，必须实现，完整解析域名时会调用此接口验证权利

nnshash 为域名hash

subdomain 为子域名

返回 byte[] 所有者地址，或者空

FIFS域名申请
-----------

::

        requestSubDomain(byte[] who,byte[] nnshash,string subdomain)

此接口为演示的先到先得注册器使用，用户调用注册器的这个接口申请域名

 - **who** 谁在申请
 - **nnshash** 申请哪个域名
 - **subdomain** 申请的子域名


解析器合约详解
==============

1.解析器自己保存解析信息

2.顶级域名合约会以nep4的方式调用解析器的解析接口获取解析信息。

3.解析器设置解析数据时采用Appcall的形式调用顶级域名合约的getInfo接口来验证域名所有权

::

        [Appcall("dffbdd534a41dd4c56ba5ccba9dfaaf4f84e1362")]
        static extern object rootCall(string method, object[] arr);

任何合约都可以通过 appcall 顶级域名合约getInfo接口的方式来验证域名所有权

合约入口
--------

注册器参数形式必须也是0710，返回05

::

        public static byte[] Main(string method, object[] args)
        {
            if (method == "resolve")
                return resolve((string)args[0], (byte[])args[1]);
            if (method == "setResolveData")
                return setResolveData((byte[])args[0], (byte[])args[1], (string)args[2], (string)args[3], (byte[])args[4]);
            ...

解析域名
-------

::

    resolve(string protocol,byte[] nnshash)

此接口为解析器规范要求，必须实现，完整解析域名时会调用此接口最终解析

- **protocol** 协议类型
- **nnshash** 要解析的域名

返回byte[] 解析数据

设置解析参数
-----------

::

    setResolveData(byte[] owner,byte[] nnshash,string or int[0] subdomain,string protocol,byte[] data)

此接口为演示的标准解析器所有，所有者（目前还只支持账户地址所有者）可以调用此接口配置解析数据

- **owner** 所有者
- **nnshash** 设置哪个域名
- **subdomain** 设置的子域名（可以传0，如果设置的就是域名解析，非子域名传0）
- **protocol** 协议字符串
- **data** 解析数据

返回[1]表示成功 或者[0]表示失败


投标竞拍域名注册方式详解
======================

竞标服务
--------

竞标服务以确定谁有权注册某一个二级域名为服务目标。服务进程分为四个阶段：开标、投标、揭标、中标。

开标
-----

任意未被注册或已过期且不违反域定义的域名都可被任意标准地址（账户）申请开标。一旦开标立即进入投标阶段。

投标
----

投标由开标启动，为期72小时，在这段时间内任意标准地址（账户）可以提交一个加密的报价，并支付NNC保证金。
投标人通过发送NNC同时附加其自定义的一组8位任意字符的sha256散列值作为报价用以隐藏真实报价，以防止没有必要的恶性竞争。
投标人不足1人，竞标自动结束，域名立即进入可开标状态。

揭标
-----

投标过程结束后进入48小时的揭标过程。在此期间，投标人需要提交报价的明文和加密字符串明文，用以验证揭标人的真正出价。
揭标后，保证金会扣除部分系统费用后返还。如果投标人没有揭标，视为放弃竞标。揭标人不足1人，竞标自动结束，域名立即进入可开标状态。

中标
-----

揭标结束后，中标人需要通过一笔交易正式获得域名的所有权。
关于投标竞拍模式的域名分配规则会在以后版本进一步细化。

交易服务
--------

交易服务支持域名登记员发布域名所有权转让邀约，支持固定价格转让和荷兰式减价拍卖方式转让。 

无锁定可循环分配代币NNC技术实现
=============================

NNS的经济系统需要一种资产,故我们设计了一种资产。NNS的经济系统要求资产总量不变,且拍卖所得和租金消耗均视为销毁。
故我们设计的资产具有消耗功能,消耗资产会重新分配。

因为销毁和重分配会循环进行,因此我们称为可循环分配代币.无锁定指的是分配过程不会造成用户资产锁定,下文详述。

重新分配机制
-------------

我们为代币使用了销毁接口,主要的销毁方式为：

1. 支付租金,系统销毁

2. 二级域名拍卖所得,系统销毁

3. 任何人想要销毁他自己的部分代币,系统销毁

一旦销毁,就计入销毁池,销毁池的资产会再进入奖池，用户可从奖池中领走资产。

无锁定的领奖机制	
---------------

通常考虑这样的系统，一般如拍卖将系统分为开标、投标、揭标、中标四个阶段，用户资金在投标阶段必须支付入系统，意味锁定，中标后被消耗，或者流标退还，意味解锁。

而NNC代币，并不划分为：参加领奖 等待开奖，领奖几个阶段，不涉及用户资产的锁定。
 
NNC使用奖池队列的机制，如图，只保持固定数量的奖池（比如五个）

产生超过五个奖池时，最早的奖池被销毁。

配合奖池机制再将用户资产分为两个部分：固定资产和零钱，其中固定资产的持有时间只能增加，而只有固定资产持有时间小于奖池领奖时间才能领奖的模式。
领奖后固定资产持有时间增加，犹如币天被消耗掉，用这种方式防止重复领取。

奖池细节
--------

代币会维持几个奖池,每个奖池产生时把销毁池里的资产全部移入奖池，如果超过最大奖池数，把最早的奖池也销毁，把销毁奖池里剩余的资产也计入新奖池。

奖池数量为确定值，比如每4096块产生一个奖池，最多维持五个奖池，产生第六个奖池时，销毁第一个奖池，并把他的资产也全部放入最新的奖池。
（以上数值几个奖池，多久产生一个奖池均为暂定）
每个奖池会对应一个块，这个块就是领奖时间，只有持币时间早于领奖时间，才可以领奖。

固定资产和零钱的细节
-------------------

固定资产和零钱。其中固定资产记录一个持有时间。

固定资产和零钱只影响领奖数额，对其余功能没有影响。

固定资产+零钱=用户的资产余额。

固定资产没有小数部分，小数部分均计入零钱。下文提到视为固定资产均表示将整数部分视为固定资产，小数部分计入零钱。不再重复说明，统称“视为固定资产”。

用户转账，优先使用零钱，不足部分使用固定资产。

用户转账转出方：固定资产只能减少。

用户转账转入方：固定资产不变，转入金额全部计入零钱。

只有两种方法使固定资产增加：

1. 创建账户（向一个没有nnc代币的地址转入视为创建账户）

转入的资产视为固定资产，持有时间为转入块。

2. 领奖

领奖后将个人资产包括领奖所得，视为固定资产，持有时间为领奖块。
然后用户领奖时只能逐个奖池领取自己固定时间<奖池的部分领取比率算法为 当前奖池数额/（总发行量-当前奖池数额）。
用户在每个奖池里的可领数额为：自己的固定资产*当前奖池领取比率。
	
以数字来说明，现有奖池3个，分别是 4096，8000，10000块产生的。用户的固定资产100，持有时间7000.则他不能领第一个奖池，
只能领第二个和第三个奖池的资金。当前块10500，一旦用户领奖，他的资产持有时间就变成10500，哪个奖池他都不能领了。

比如有一个奖池，内有资金五千三十万，他的领取比率就是
50300000 /（1000000000-503000000）=0.123587223，用户有100固定资产，则能在此奖池内领取12.3587223个NNC。

NNC接口（仅述相比NEP5多出来的接口）
---------------------------------

NNC首先符合NEP5规范，NEP规范接口不再赘述

balanceOfDetail(byte[] address)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

返回用户资产细节，有多少固定资产，有多少零钱，总额，固定资产持有块，无需签名，任何人可查。

返回结构体：

::

    {
        现金数额
        固定资产数额
        固定资产持有块
        余额（固定资产+现金）
    }

use(byte[] address,BigInteger value) 
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

消耗一个账户的若干资产，需账户签名

消耗资产即进入奖池

getBonus(byte[] address)
~~~~~~~~~~~~~~~~~~~~~~~~~~

指定账户领奖，需账户签名。

领奖后会将该账户总资产视为固定资产并变动该账户的固定资产持有块。

checkBonus()
~~~~~~~~~~~~~~
检查当前奖池，无需签名

返回Array<BonusInfo>:

::

    BonusInfo
    {
        StartBlock;//领奖块
        BonusCount;//该奖池总量
        BonusValue;//该奖池剩余
        LastIndex;//上一个bonus的id
    }

newBonus ()
~~~~~~~~~~~

产生新奖池，任何人均可调用，但是奖池产生要符合奖池间隔，所以重复调用并无作用。此接口可视为督促产生新奖池。