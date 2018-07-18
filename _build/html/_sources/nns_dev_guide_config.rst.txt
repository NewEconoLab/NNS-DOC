接口标准
===========

变量类型
------------

1. 域名拍卖id	hex256
#. 用户地址	    hex160
#. 域名哈希	    hex256
#. 合约地址	    hex160
#. 用户资产额度 BigIngeger

变量/参数定义
-------------

常量
~~~~~~~~

1. SGAS 合约：     DAPP_SGAS     => hex160
#. NNC合约：       DAPP_NNC      => hex160
#. 竞拍注册器：     DAPP_REG     =>  hex160
#. 先到先得注册器： DAPP_REG_FIFO => hex160


转账
~~~~~~~

1. 转账源地址：    who     => hex160
#. 转账目的地址：  target  => hex160
#. 转账金额：      amount  => BigIngeger

域名拍卖
~~~~~~~~~~

1. 拍卖id ：bid_id =>hex256
#. 域名哈希：domain_hash => hex256
#. 根域名哈希：root_hash =>hex256
#. 子域名：sub_domain =>string
#. 域名完全哈希：full_hash=>hex256

数据结构
------------

NNS合约的对外统一接口有两个参数，一个是用于接收具体命令的，另一个是接收Object类型的参数列表的，所有合约的接口定义都如下：

::

    public static object Main(string method, object[] args)

因此调用NNS合约的数据结构可描述为：

::

    {
        appCall：合约地址
        method：合约方法
        params：参数列表
    }


