# 迅雷链BaaS平台 SDK说明文档

迅雷链提供 Go-SDK 以方便开发者与区块链系统（BaaS接入层）进行通信和交互。

## 1 SDK功能

1. 封装区块链接口，实现账户管理、普通交易和合约交易等功能；
2. 集成多个账户的账户信息，作为钱包账户服务中心；
3. 与Baas接入层交互。

简言之：SDK接收账户请求并封装，随后将其转发至BaaS接入层。

## 2 SDK目录结构

```sh
.
├── sdk.go              // SDK接口的结构体实例
├── sdkapi.go           // SDK实例的所有接口实现
├── args.go             // SDK使用的消息结构
├── account.go          // SDK账户管理
├── config.go           // SDK包所需的所有配置信息
├── log.go              // SDK包日志接口
├── dnscache.go         // BaaS接入层的DNS解析缓存
├── client.go           // 封装与BaaS接入层交互的客户端
├── httpcli.go          // 封装简易HTTP请求方法
├── util.go             // 通用函数
└── errors.go           // 错误码
```

## 3 SDK使用

开发者在使用SDK时，需遵循以下步骤：

### 3.1 设置SDK全局配置。

在SDK中，其需要的所有配置信息如下：
```go
// AuthInfo 负责通过认证与BaaS交互
type AuthInfo struct {
	ChainID string `json:"chainid"`          // 链ID
	ID      string `json:"id"`               // BaaS为开发者分配的ID
	Key     string `json:"key"`              // BaaS为开发者分配的Key
}

// Config SDK配置信息
type Config struct {
	Keystore               string            // Keystore目录 保存用户账户秘钥
	UnlockAccounts         map[string]string
	RPCProtocal            string            // BaaS接入层 协议
	XHost                  string            // BaaS接入层 Host
	Namespace              string            // 区块链名称空间 tcapi
	ChainID                int64             // 链ID
	GetGasPrice            bool              // 是否从BaaS获取GasPrice
	AuthInfo               AuthInfo
}
```
开发者需构造SDK包内的Config类型，填充其信息并将构造的Config作为入参构造SDK。
需要注意的是，`UnlockAccounts` 和 `AuthInfo` 需开发者自行解析。
构造过程如本目录下的Server示例所示：
```go
  // 1. 构造基础配置
  sdkConf := &sdk.Config{
    Keystore:               conf.Keystore,
    UnlockAccounts:         make(map[string]string),
    RPCProtocal:            conf.RPCProtocal,
    XHost:                  conf.XHost,
    Namespace:              conf.Namespace,
    ChainID:                conf.ChainID,
    GetGasPrice:            conf.GetGasPrice,
  }
  // 2. 解析得到 AuthInfo
  authInfoJSON, err := ioutil.ReadFile(authInfoFilePath)
  err = json.Unmarshal(authInfoJSON, &sdkConf.AuthInfo)
  // 3. 解析得到 UnlockAccounts
  passwdsJSON, err := ioutil.ReadFile(passwdFile)
  err = json.Unmarshal(passwdsJSON, &sdkConf.UnlockAccounts)
  // 4. 返回构造的sdkConf
  return sdkConf
```

### 3.2 获取SDK实例

SDK实现了所有与BaaS交互的必要接口。

需要注意的是，获取SDK实例时需传入 `Config` 和满足 `sdk.Logger` 接口的log实现，该log将在SDK内部提供日志功能。

开发者可以通过调用以下接口`获取`和`释放`SDK资源：
```go
func NewSDK(cfg *Config, log Logger) (*SDKImpl, error)
```

随后即可通过调用该实例的方法进行指向BaaS的接口调用：
```go
import(
  "github.com/XunleiBlockchain/baas-sdk-go"
)

func foobar() {
  // ------- INIT -------
  // 1. get sdk Config
  sdkConf := &sdk.Config{
    ...
  }
  // 2. init log
  logger := log.NewLogger()
  // 3. init sdk
  mySDK := sdk.NewSDK(sdkConf, logger)

  // ------- USE -------
  // encode params
  resp, xerr := mySDK.GetBalance(params...)
  if xerr != nil || xerr.Code != 0 {
    // handle error
  }
  // parse resp
}
```
## 4 SDK接口

SDK提供以下接口：
```go
type SDK interface {
  // 新建用户账户
  NewAccount(params interface{}) (interface{}, *Error)
  // 获取SDK管理的所有用户账户
  Accounts(params interface{}) (interface{}, *Error)
  // 获取某账户地址的余额
  GetBalance(params interface{}) (interface{}, *Error)
  // 获取某账户地址的Nonce
  GetTransactionCount(params interface{}) (interface{}, *Error)
  // 获取指定交易的信息
  GetTransactionByHash(params interface{}) (interface{}, *Error)
  // 获取指定交易收据
  GetTransactionReceipt(params interface{}) (interface{}, *Error)
  // 发送交易
  SendTransaction(params interface{}) (interface{}, *Error)
  // 发送合约交易
  SendContractTransaction(params interface{}) (interface{}, *Error)
  // 执行消息调用（无需创建交易）
  Call(params interface{}) (interface{}, *Error)
}
```

## 5 接口详细说明

SDK接口的输入和输出参数均以JSON格式编码，该格式定义于 `args.go`。如有需要，开发者可以在源码中看到更底层的内容。

接口说明如下：

### 5.1 accounts

功能描述：
获取钱包管理的全部账户地址

参数：
none

返回结果：
账户地址数组

示例：
```json
//request
[
 {}
]
//result
{
 "0x33d4fcb75ce608920c7e5755304c282141dfc4dc", "0x7a4877494b59c0bd929747800ab86a8b89380ac5", "0x36419474a02a507e236dc473648197f07ab4722e", "0x7fc423bd7ed1f5d17a92bdb8c39ed620f48f7559", "0x8f470d7f2b2db7b83accd008ddabc5423c06044b", "0x622bc0938fae8b028fcf124f9ba8580719009fdc"
}

```

### 5.2 newAccount

功能描述：
创建新的账户

参数：
账户密码

返回结果：
新账户地址

示例：
```json
//request
[
 {
  "123456"
 }
]

//result
{
 "0x84d8698746dbe68c97965c48c7b56979c577df11"
}

```

### 5.3 getBalance

功能描述：
查询账户余额

参数：
账户地址

返回结果：
账户地址余额

示例：
```json
//request
[
 {
  "0x33d4fcb75ce608920c7e5755304c282141dfc4dc"
 }
]
//result
{
 99030093892100000000170
}

```

### 5.4 getTransactionCount

功能描述：
查询账户nonce值

参数：
账户地址

返回结果：
账户地址当前nonce值

示例：
```json
//request
[
 {
  "0x622bc0938fae8b028fcf124f9ba8580719009fdc"
 }
]

//result
{
 26
}

```

### 5.5 sendTransaction

功能描述：
普通转账交易

参数：
- Object： 交易对象
  - from: 转出账户地址
  - to: 转入账户地址
  - value: 转账金额 (推荐使用十进制)
  - nonce: (可选) from地址nonce值
- from账户密码(可选)

示例：
```json
[
 {
  "from": "0x622bc0938fae8b028fcf124f9ba8580719009fdc",
  "to": "0x33d4fcb75ce608920c7e5755304c282141dfc4dc",
  "value": "0x10", // 16 wei
 }，"12345678"
]
```

说明：**如果启动服务时，已经解锁了from账户，可以不用再传密码参数**

返回结果：
交易hash

示例：
```json
//request
[
 {
  "from": "0x622bc0938fae8b028fcf124f9ba8580719009fdc",
  "to": "0x33d4fcb75ce608920c7e5755304c282141dfc4dc",
  "value":"1200"
 },
 "12345678"
]
//result
{
 "0x517490b857200702453f32ed0574487b44587958ff39b26554df4f4991cae18c"
}

```

### 5.6 sendContractTransaction
功能描述：

合约执行交易，支持传1个参数，2个参数和3个参数:
- 1个参数则只穿交易对象
- 2个参数则传交易对象和扩展对象
- 3个参数则传交易对象、账户密码、扩展对象

参数：
- Object： 交易对象
  - from: 转出账户地址
  - to: 转入账户地址
  - gas：(可选，默认90000)手续费
  - gasPrice：(可选，默认1e11)
  - value: 转账金额
  - data：执行合约code
  - nonce: (可选) from地址nonce值
- from账户密码(可选)
- Object:  扩展对象（可选）
  - callback:  回调地址
  - prepay_id: 预交易id
  - service_id: 第三方服务号
  - sign：交易签名(签名方式：**MD5(SHA256(callback=XXX&prepay_id=XXX&service_id=XXX&to=XXX&value=XXX&key=XXX)), key为第三方服务号密匙**)
  - tx_type：交易类型描述
  - title：交易主题
  - desc：主题描述

示例：
```json
[
 {
  "from": "0x622bc0938fae8b028fcf124f9ba8580719009fdc",
  "to": "0x7f7f7dbf351d4272eb282f16091c96b4819007f5",
  "data": "0x49f3870b0000000000000000000000000000000000000000000000000000000000000001"
 }, "12345678", {
  "callback": "http://www.baidu.com",
  "prepay_id": "201805171922030000010176643187014087",
  "service_id": "0",
  "sign": "80fa49b1c7e5ec06ab595850dc8e8f87",
  "tx_type": "contract",
  "title": "test",
  "desc": "this is a test by skl"
 }
]
```

说明：**如果启动服务时，已经解锁了from账户，可以不用再传密码参数**

返回结果：
交易hash

示例：
```json
//request
[
 {
  "from": "0x622bc0938fae8b028fcf124f9ba8580719009fdc",
  "to": "0x7f7f7dbf351d4272eb282f16091c96b4819007f5",
  "data": "0x49f3870b0000000000000000000000000000000000000000000000000000000000000001"
 }, "12345678", {
  "callback": "http://www.baidu.com",
  "prepay_id": "201805171922030000010176643187014087",
  "service_id": "0",
  "sign": "80fa49b1c7e5ec06ab595850dc8e8f87",
  "tx_type": "contract",
  "title": "test",
  "desc": "this is a test by skl"
 }
]
//result
{
 "0x517490b857200702453f32ed0574487b44587958ff39b26554df4f4991cae18c"
}

```

## 6 错误码说明

| 错误码 | 错误信息                             | 说明                                      |
| ------ | ------------------------------------ | ----------------------------------------- |
| 0      | success                              | 请求成功                                  |
| -1000  | invalid method                       | 接口请求方法错误                          |
| -1001  | params err                           | 参数错误， 一般由Params参数个数不一致导致 |
| -1002  | find account err                     | 账号查找错误                              |
| -1003  | rpc getBalance err                   | 查看余额 rpc调用失败                      |
| -1004  | ks.NewAccount err                    | 新建账号错误                              |
| -1005  | rpc getNonce err                     | 获取Nonce值 rpc调用失败                   |
| -1006  | SDK SignTx err                    | 交易签名错误                              |
| -1007  | bal EncodeToBytes err                | RLP编码错误                               |
| -1008  | rpc sendTransaction err              | 发送交易 rpc调用失败                      |
| -1009  | SDK SignTxWithPassphrase err      | 钱包签名交易错误                          |
| -1012  | ContractExtension err                | 合约交易参数解析错误                      |
| -1013  | rpc sendContractTransaction err      | 发送合约交易 rpc调用失败                  |
| -1018  | json unmarshal err                   | json解析错误                              |
| -1019  | rpc getTransactionByHash err         | 查询交易 rpc调用失败                      |
| -1020  | rpc getTransactionReceipt err        | 查询收据 rpc调用失败                      |
| -1021  | SendTxArgs err                       | 发送交易参数解析错误 或 获取Nonce值错误   |

注：其他错误码由BaaS透传返回
