# SatoshiOptions docs

## 合约

* `contracts/Route.sol`: 路由合约
* `contracts/SatoshiOptions_Charm.sol`: 期权合约，管理 nft
* `contracts/public/BinaryOptions.sol`: 指数期权算法
* `contracts/public/LinearOption.sol`: 指数线性算法

## 配置和地址

```js
// 网络
kovan.infura.io
// 测试用的代币
WETH  0x791229928Be5F33194E779787b63555D32F8AA9E
// 资产
BTC  0x1d8E11b10e35AC2E355b39D8e0798D25Cd621837
// 系统代币
Charm  0x38c2A5a38365b50383f51cfD500a04501e4a6109

// 配置合约
Config  0x1E9a5746a4ba4355bf89CAF331Ae51ce00fF9eb8

// 开仓策略合约
// 二元
BinaryOptions  0x0Ee4454F6FD8c64e5633f34bF844105450257009
// 线性
LinearOptions  0xEd95A8a9421AbBC5474a922bF6736C37292aF68C

// 期权合约
SatoshiOpstion_Charm  0x67cABa39b375aAb0615442a8D799bC1aeB59e082

// 路由合约 
router to  0x34E248C842354669E38532Ec5d9c906c5018B54E

// 签名地址
// 私钥： 0x1b502936fcfa1381d1bc454dac74f1a2d2c7e4ed7634fe1acc57b0fa32c5f26e 【勿在生产环境中使用】
const SIGNER_ADDRESS = "0x9548B3682cD65D3265C92d5111a9782c86Ca886d";
```

## 接口

数据类型转换 `getInt128`

```js
// uint256 -> int128
const BigNumber = require('bignumber.js');
function getInt128(num) {
    let _num = (new BigNumber(num).multipliedBy(new BigNumber(2).pow(64))).toString(10);
    _num = _num.split('.')[0];
    return _num
}
```

价格签名 `getPriceData`

```js
const web3 = new Web3();
const abi = require('ethereumjs-abi');
const PRIVATE_KEY = "0x1b502936fcfa1381d1bc454dac74f1a2d2c7e4ed7634fe1acc57b0fa32c5f26e";  
let nonce = new BigNumber(0);
const SIGNER_ADDRESS = web3.eth.accounts.privateKeyToAccount(PRIVATE_KEY).address; //2109

// nonce 获取
const nonce = SatoshiOptions_Charm.methods.seenNonces(sender)

// tokenAddress: string, tradePrice: uint128, nonce: number
function getPriceData(tokenAddress, tradePrice, nonce) {
    const deadline = (new Date() / 1000 + 60).toFixed(0)
    const parameterTypes = ["address", "uint128", "uint256", "uint256", "address"];
    const parameterValues = [tokenAddress, tradePrice.toString(), nonce, deadline,SIGNER_ADDRESS];
    const hash = "0x" + abi.soliditySHA3(parameterTypes, parameterValues).toString("hex");
    const signature_ = web3.eth.accounts.sign(hash, PRIVATE_KEY);

    return {
        tokenAddress,
        tradePrice: tradePrice.toString(),
        nonce,
        deadline,
        signature: signature_.signature
    };
}
```

开仓 `buyOptions(bool direction,uint128 _delta,uint128 _bk,uint128 _cppcNum,address _strategy,IIssuerForSatoshiOptions.SignedPriceInput) -> (uint256 pid, uint256 mintBalance)`

```js
const web3 = new Web3();

// Route.sol
const options = Router()

// 根据参数获得开仓数量
const signature = getPriceData(tokenAddress, tradePrice, nonce)
const methods = await options.methods.buyOptions(
    true, // true 开多，false 看空
    getInt128(delta), // delta
    getInt128(2), // 杠杆
    getInt128(1.3213), // 金额 
    strategyAddress, // 策略 合约地址 ：contracts/public/BinaryOptions.sol | contracts/public/LinearOption.sol
    [
        tokenAddress, // 标的币种
        tradePrice, // 交易价格
        nonce, // 签名 有效 nonce
        signature // 签名
    ]
)

// 列子
const {
    tokenAddress,
    tradePrice,
    nonce,
    deadline,
    signature
} = getPriceData(BTC.address, getInt128(1123.31), 0 )
tx = await router.buyOptions(
    false, // true 开多，false 看空
    getInt128(6), // delta
    getInt128(1.05), // 杠杆
    getInt128(1e18), // 金额
    LinearOptions.address, // 策略 合约地址 ：contracts/public/BinaryOptions.sol | contracts/public/LinearOption.sol
    [
        tokenAddress, // 标的币种
        tradePrice, // 交易价格
        nonce, // 签名 有效 nonce
        deadline,
        signature // 签名
    ]
)


const calls = await methods.call()
// 开仓数量 wei
calls.mintBalance
// 仓位 id
calls.pid

// 开仓上链

// 1. 授权
tx = await BTC.approve(router.address, amount)
await tx.wait()
// 2. 交易
const tx = await methods.send()
```

获取仓位数量 `balanceOf(address, uint256) -> string`
```js
// SatoshiOpstion_Charm.sol
const options = SatoshiOpstion_Charm()

// 获得开仓数量
let balanceNFT = await SatoshiOpstion_Charm.methods.balanceOf(ownerAddress, pid).call()
```

获取仓位详情 `getNftInfoFor(uint256 _pid) -> NftData`
```js
// SatoshiOpstion_Charm.sol
const options = SatoshiOpstion_Charm()

// 获得开仓详情
const NftData = await SatoshiOpstion_Charm.methods.getNftInfoFor(pid).call()

// NftData.delta : delta
// NftData.createTime : 开仓时间
// NftData.openPrice : 开仓价格
// NftData.direction : 方向
// NftData.bk : 杠杆
// NftData.K : 目标价
// NftData.tradeToken : 交易 token
// NftData.strategy : 期权策略合约地址
```



平仓 `sellOptions(uint256 pid, uint128 amount, IIssuerForSatoshiOptions.SignedPriceInput) -> string`

```js
const web3 = new Web3();

// Route.sol
const options = Router()

// 根据参数获得开仓数量
const signature = getPriceData(tokenAddress, tradePrice, nonce)
const methods = await options.methods.sellOptions(
    pid, // 仓位id: number
    amount, // 平仓数量: string
    [
        tokenAddress, // 标的币种
        tradePrice, // 交易价格
        nonce, // 签名 有效 nonce
        signature // 签名
    ]
)


// 平仓后可以获得的代币奖励
const tokens = await methods.call()

// 平仓上链
const tx = await methods.send()
```
