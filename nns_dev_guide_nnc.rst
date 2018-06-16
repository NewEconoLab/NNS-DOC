NNC 开发者手册
=================

获取nnc发行总量
-----------------
本接口用于获取NNC的总发行量

合约脚本格式：

::

    appCall:  DAPP_NNC
    method:   totalSupply
    params:   []
    交易类型：InvocationTransaction
    签名：    无

交易执行结果：

::

    {
    "jsonrpc": "2.0",
    "id": 1,
    "result": true
    }

结果解析路径：

例：【totalSupply =result['stack']['value'][0]['value']】

获取NNC合约名
------------------
合约脚本格式：

::

    appCall:  DAPP_NNC
    method:   name
    params:   []
    交易类型：InvocationTransaction
    签名：    无

交易执行结果：

::

    {
    "jsonrpc": "2.0",
    "id": 1,
    "result": true
    }

结果解析路径：

例：【name =result['stack']['value'][0]['value']】

获取NNC的符号
---------------------
NEP5标准方法，用于获取token的符号。

合约脚本格式：

::

    appCall: DAPP_NNC
    method:	 symbol
    params:	 []
    交易类型：InvocationTransaction
    签名：    无

交易执行结果：

::

   {
    "jsonrpc": "2.0",
    "id": 1,
    "result": true
    }

结果解析路径：

例：【symbol =result['stack']['value'][0]['value']】

获取NNC精度
---------------------
NEP5标准方法，用于获取token的精度。

合约脚本格式：

::

    appCall: DAPP_NNC
    method:	 decimals
    params:	 []
    交易类型：InvocationTransaction
    签名：    无

交易执行结果：

::

   {
    "jsonrpc": "2.0",
    "id": 1,
    "result": true
   }

结果解析路径：

例：【decimals =result['stack']['value'][0]['value']】


初始化nnc 
-----------
在NNC合约部署后，由管理员进行初始化操作，本接口只能管理员调用。

合约脚本格式：

::

    appCall:   DAPP_NNC
    method:    symbol
    params:    []
    交易类型： InvocationTransaction
    签名：     管理员签名

交易执行结果：

::

   {
    "jsonrpc": "2.0",
    "id": 1,
    "result": true
    }


获取账户余额
---------------
合约脚本格式：

::

    appCall: DAPP_NNC
    method:  balanceOf
    params:	 [who]
    交易类型：InvocationTransaction
    签名：	 无

交易执行结果：

::

   {
    "jsonrpc": "2.0",
    "id": 1,
    "result": true
    }

结果解析路径：

例：【balanceOf =result['stack']['value'][0]['value']】


普通账户转账
-----------------
NEP5标准方法，用于账户间NNC的转账

合约脚本格式：

::

    appCall: DAPP_NNC
    method:	 transfer
    params:	 [from,to,value]
    交易类型：InvocationTransaction
    签名：    需要from的签名

交易执行结果：

    {
       "jsonrpc":"2.0",
       "id":1,
       "result":[
        {
              "sendrawtransactionresult":true,
              "txid":"0x3902ed858b4ae73619ec57fbf70730aa6129ba6f6ca910fb1c3f08d552f03ec9"
        }]
    }

合约调用NNC转账
------------------------
本方法是为了提供NNS转入SGAS手续费的时候使用，仅仅能从别的合约动态调用，无法通过交易触发。

本接口首先通过SGAS合约将注册器合约收取的手续费转入到NNC合约地址，然后调用NNC，标记转入的SGAS数量。

合约脚本格式：

::

    appCall: DAPP_SGAS
    method:	 transfer_app
    params:	 [who,target,value]
    appCall: DAPP_NCC
    method:  useGas
    params:	 [txid]
    交易类型：InvocationTransaction
    签名：	 无

交易执行结果：

::

   {
    "jsonrpc": "2.0",
    "id": 1,
    "result": true
    }


领取分红
-------------
NNS系统的域名注册手续费收益会完全再分配给NNC的持有者，用户可以通过调用本方法领取属于自己的SGAS分红。

合约脚本格式：

::

    appCall: DAPP_REG
    method:	 claim
    params:	 [who]
    交易类型：InvocationTransaction
    签名：	  用户

交易执行结果：

::

   {
	"jsonrpc": "2.0",
	"id": 1,
	"result": [{
		"sendrawtransactionresult": true,
		"txid": "0x376df8dd0c03ef5184f8ea28da98fe9da81f0c1e2aa6e86f8e86c14f83a3e3d1"
	}]
   }


查询可以领取的分红数
----------------------
本方法用于查询用户可以提取的分红数量。

合约脚本格式：

::
    appCall:	DAPP_REG
    method:	canClaimCount
    params:	[who]
    交易类型：	InvocationTransaction
    签名：	无

交易执行结果：

::

   {
	"jsonrpc": "2.0",
	"id": 1,
	"result": [{
		"script": "1436630a8759e5000f88500017adfa5c8fe2be32ff51c10d63616e436c61696d436f756e74677b1363536065a16bb208e9c9df9344d5fd0cfad8",
		"state": "HALT, BREAK",
		"gas_consumed": "0.476",
		"stack": [{
			"type": "Integer",
			"value": "200000001"
		}]
	}]
    }

结果解析路径

例：【claimCount=result['stack'][0]['value']['value']】

查询系统费收入
-------------------------
通过调用这个接口可以获取到当前NNS系统已收取且未被cliam的sgas手续费总量。

合约脚本格式：

::

    appCall:  DAPP_REG
    method:	  getTotalMoney
    params:	  []
    交易类型： InvocationTransaction
    签名：	  无

交易执行结果：

::

   {
	"jsonrpc": "2.0",
	"id": 1,
	"result": [{
		"script": "010051c10d676574546f74616c4d6f6e6579677b1363536065a16bb208e9c9df9344d5fd0cfad8",
		"state": "HALT, BREAK",
		"gas_consumed": "0.722",
		"stack": [{
			"type": "ByteArray",
			"value": "0484d717"
		}]
	}]
    }

结果解析路径

例：【totalMoney=result['stack'][0]['value']】 转Biginteger
