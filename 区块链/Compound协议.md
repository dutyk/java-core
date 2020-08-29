本文介绍Compound协议，以此打开Defi世界的大门

版本号：V2.1

# 常量
常量 | 描述
- | :- |
liquidationIncentive | 一个乘数，代表用户调用liquidate receives超出的百分比值，例如 1.05，有5％的红利。 此折扣适用于被扣押的资产(即用作抵押的资产)。 |
collateralFactor | 一个乘数，基于抵押品您可以借入的金额，例如.9允许借入抵押品价值的90％。 一定是在0和1之间。|
closeFactor | 当清算underwater账户的借款时，大于0.05且小于或等于0.9的数字，×给定资产的borrowCurrent，以计算最大repayAmount。|
maxAssets |一个帐户可以参与的最大资产数量(借入或用作抵押)。 这不会影响帐户在不借款时铸币、赎回或转账。|
reserveFactor | 计入准备金的应计利息部分，在[0，1]之间，并可能低于0.10。|

# Key Terms
值 | 描述
- | :- |
 borrowCurrent | 用户借入的给定资产，包括截至日期当前块的应计利息。 这是用户存储的本金×市场的当前利息指数，除以用户的存储利息指数。 |
 sumCollateral | 用户提供的资产的抵押物价值，包括应计利益(以太)。 这是用户所有资产的总和, 代币余额乘以该代币的(存储的)代币的基础汇率，乘以eth为单位的资产价值，乘以资产的collateralFactor。注意：我们在这里使用存储的汇率，而不是为每个抵押资产计算新汇率。 增量应该很小，这仅用于帐户流动性检查 |
 sumBorrow | 用户借入资产的价值，包括应计利息，以Ether为单位。 用户的每个资产的borroweCurrent之和。 |
 accountLiquidity | 用户帐户的sumCollateral值，以Ether表示，减去sumBorrow(对于sumCollateral≥sumBorrow)。 此数字对于不健康的账户，可能低于零。 |
 maxCloseValue | 用户在给定资产中的借贷余额乘以closeFactor; 清盘人可以偿还多少价值|
 seizeTokens |从清算用户转移到清盘人的cTokens数量。 这是seizeAmount乘以清算激励，乘以给定资产对的价格预言机的价格比率，除以抵押资产的汇率。 |
 totalBorrowBalance |当前块，货币市场，所有帐户的总借贷余额，含应计利息。注意：totalBorrowBalance将严格大于v1的totalBorrows，因此，如果使用相同的利率模型，利率计算的结果将有所不同。 |
 assets account | 帐户参与的一组市场，最大规模为maxAssets|
 blocks |计算简单利息时，区块指的是自上次利息指数计算以来已过去的区块数量。 最近的区块存储为interestBlockNumber(asset)，随着利息索引存储而存储，block=当前块的数量减去interestBlockNumber(asset)。 |
 rate | 计算简单利息时，利率是指市场当期利息利率。 对于blocks blocks这是之前“生效”的汇率。|
 Exchange Rate Stored |cToken到基础资产的最近存储汇率。 不包括市场上次利息应计以来的接入利息。 |
 Exchange Rate Current | 在cToken和基础资产之间的当前汇率(包括所有已调整的借入利息)。|

 # 特殊状态
我们假设在任何错误情况下，或者a)协议通过事件正常退出
如果尚未发生副作用，请描述错误，或者b)交易失败
完全。除以下内容外，请注意此规则的任何例外情况。
许多功能分为两个命令：产生兴趣和执行新动作。的
目标是分离应该发生的两个离散事件。首先，每次我们积累
市场的兴趣，我们帮助平衡余额(并将简单的兴趣转化为复利
兴趣)。其次，只有在市场感兴趣的情况下，新功能才是正确的
截至此区块已完全累积。随着兴趣的更新，这些新鲜的
功能有不同的关注点(通常与应计利息无关)。因此，我们
将这些功能构建为两个动作的总和，以简化理解和建模
这两个单独的行为。实际上，这意味着即使某个方法正常失败，
交易可能仍会对该市场产生兴趣(这是一件好事！)。
利率示范合约
对于每种资产，都有一个利息指数。我们有效地跟踪了本金的增长
随着时间的推移任意帐户。我们使用该帐户的利息与初始本金的比率
计算该时间间隔的子集内任何给定帐户的兴趣增长。的
利率模型合同规定了随时的简单利率(即
每笔交易的复利就变成复利)。我们强迫这个利率
该模型是对资产在市场上的现金，借款和储备的纯函数。
有关更多信息，请参见利息指数计算附录。
roweRate(地址cToken，现金金额，借款金额，储备金)返回uint
●返回当前市场利率
●注意：cToken是复合cToken合约，不是基础资产地址。
价格Oracle合同
复合协议使用来自智能合约的价格，即价格预言。的
主计长和清算借用函数在此预言中引用价格。多
对于不同的复合市场可能存在预言。
getUnderlyingPrice(CToken cToken)返回uint
●标的资产的返回价格(以尾数表示)

代币合约
cToken充当ERC-20接口，将成为用户与之交互的主要位置
复合协议。当用户铸造，赎回，借入，偿还借入，清算
借用或转让cToken，她将对cToken合同本身这样做。唯一的行动
用户在另一份合同上执行的操作是通过主计长进出资产
(请参见下面的主计长合同)。
每个cToken都引用一个基础。不过，这通常是基本的ERC-20合同
它可能是醚本身，也可能是复杂资产。 cToken是该底层证券的最终持有人
资产余额以及每次收取或发出资产的请求都源自cToken合同。
最初，cEth(复合以太币令牌)将是唯一的资产，而是与Ether交互
ERC-20资产。
关于cToken货币市场的注意事项
货币市场是第一版《复合议定书》的核心内容。的
货币市场中曾经存在的功能现在存在于cToken中，而旧市场
struct扩展到每个cToken市场。与政策和流动性有关的职能是
推迟到主计长合同。
市场职能
loanRatePerBlock()
●返回此cToken的当前每块借贷利率尾数
supplyRatePerBlock()
●我们计算供应率：
○下方的otalSupply u = t×exchangeRate
○借款人=借款总额÷基础
○供应率=借贷率×(1-储备因子)×借贷每
●返回此cToken的当前每块供应利率尾数
应计利息()
我们会在每个操作中产生利息并更新借入指数。这增加了
复利，逼近真实值，无论是否进行其余操作
成功与否。

t =我getCashPrior()
○注意：可能会拨打外部电话
●我们获得了利率(自上次更新以来一直有效)：
○借款利率=调用interestModel.borrowRate(this，totalCash，totalBorrows，totalReserves)
○simpleInterestFactor =Δ块×借位率
●我们更新roweIndex：
○借入索引新=借入索引×(1 + simpleInterestFactor)
●我们计算应计利息：
○利息累计=总借贷×simpleInterestFactor
●我们会更新借款和准备金：
○totalBorrowsNew =总借贷+利息累计
○总储备新=总储备+利息累计×储备因子
●我们将更新存储回区块链：
○设置accrualBlockNumber = getBlockNumber()
○设置借款人索引=借款人索引新
○设置totalBorrows = totalBorrowsNew
○设置totalReserves = totalReservesNew
●发出AccrueInterest事件
[CErc20]薄荷(uint mintAmount)
●检查调用Accrue Interest()= 0
●返回调用Mint Fresh(msg.sender，mintAmount)
[CEther]需支付Mint()
●检查调用Accrue Interest()= 0
●返回调用Mint Fresh(msg.sender，msg.value)
[内部]薄荷新鲜(地址minter，uint mintAmount)
用户从自己的地址向市场提供资产，并获得cToken余额
作为交换。
●如果调用comptroller.mintAllowed(this，minter，mintAmount)= / 0，则失败
●验证市场的区块编号等于当前的区块编号
●如果调用checkTransferIn(minter，mintAmount)失败，则失败
●我们获取当前汇率并计算要铸造的cToken数量：
○exchangeRate =调用汇率Stored()
○mintTokens = mintAmount÷exchangeRate

注意：除法将舍去为零，因此有可能
铸造0个代币
●我们计算出新的cToken总供应量和minter令牌余额：
○otalSupplyNew totalSupply t = + mintTokens
■溢出失败
○accountTokensNewminter = accountTokensminter + mintTokens
■溢出失败
●我们已经完成了计算。 (如果任何计算由于错误而失败，我们将
已返回并带有失败代码)。现在我们可以开始效果了。
●我们为薄荷糖和mintAmount调用doTransferIn
○注意：cToken必须处理ERC-20和底层ETH之间的差异。
○成功完成后，cToken会持有额外的薄荷钱
○如果doTransferIn失败，尽管我们检查了前提条件，我们将还原
因为我们不确定副作用是否发生
●我们将先前计算的值写入存储中：
○设置totalSupply = cTokenSupplyNew
○设置accountTokensminter = accountTokensNewminter
●使用minter，mintAmount，mintTokens发出Mint事件
●将转移事件从此发送到铸币商
●调用comptroller.mintV erify(this，minter，miningAmount，mininTokens)
兑换(uint redeemTokens)
●检查调用Accrue Interest()= 0
●返回调用Redeem Fresh(msg.sender，redeemTokens，0)
赎回底层证券(uint redeemAmount)
●检查调用Accrue Interest()= 0
●返回调用即可赎回Fresh(msg.sender，0，redeemAmount)
[内部]兑换新鲜(地址兑换者，uint redeemTokensIn，uint redeemAmountIn)
用户放弃cToken并将协议中的基础ERC-20资产接收到
她自己的钱包。
●exchangeRate =调用汇率Stored()
●如果redeemTokensIn> 0：
○我们获得当前汇率并计算要兑换的金额：
■redeemTokens = redeemTokensIn
■redeemAmount = redeemTokensIn×exchangeRate
●其他：


