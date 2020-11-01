本文介绍Compound协议，以此打开Defi世界的大门

版本号：V2.1

# 名词解释
- liquidationIncentive

  一个乘数：代表用户调用清算收到的超出百分比值，例如 1.05可获得5％的奖励。 此折扣适用于被扣押的资产（即用作抵押的资产）。

- collateralFactor

  一个乘数，代表您基于抵押物可以借入的额度，0.9表示允许借入抵押品价值的90％。一定是在0和1之间。

- closeFactor

  大于0.05且小于或等于0.9的数字，当清算账户借款时，closeFactor乘以给定资产的borrowCurrent，以计算最大repayAmount。

- maxAssets

  一个帐户可以参与的最大资产数量（借入或用作抵押）。 没有借款，帐户也可以铸币、赎回或转让。

- reserveFactor

  计入准备金的累计利息，在[0，1]之间，并可能低于0.10。

- borrowCurrent

  用户借入的给定资产，包括截至当前块的累计利息。 borrowCurrent=用户存储的本金乘以市场的当前利息指数，除以用户的存储利息指数。

- sumCollateral

  用户提供的资产的抵押物价值，包括累计利息，用以太计息。 这是用户所有资产的总和：代币余额乘以该代币的（存储的）基础汇率，乘以以eth为单位的资产价值，乘以资产的collateralFactor。

  >注意：我们在这里使用存储的汇率，而不是计算每个抵押资产的新汇率。 增量应该很小并且仅用于帐户流动性检查。

- sumBorrow

  用户借入资产的价值，包括累计利息，用Ether作为单位。 就是说，每个用户资产的borrowCurrent之和。

- accountLiquidity

  以Ether表示，用户帐户的sumCollateral减去sumBorrow（sumCollateral≥sumBorrow）。对于不健康账户，这个数值可能低于零。

- maxCloseValue

  用户在给定资产中的借贷余额乘以closeFactor; 清算人可以偿还多少价值。

- seizeTokens

  从被清算用户转移到清算人的cTokens数量。 这是seizeAmount乘以清算激励，乘以给定资产对的价格比率，除以抵押资产的汇率。

- totalBorrowBalance

  截至当前块，货币市场上所有帐户的总借款余额（含累计利息）。注意：totalBorrowBalance将严格大于v1中的totalBorrows，因此，如果使用相同利率模型，利率计算的结果不同。

- assets<sub>account</sub>

  帐户参与的一组市场，最大的是maxAssets

- blocks
  
  计算单利时，blocks指的是自上次利息指数计算以来已过去的区块数。该最新块存储为   interestBlockNumber<sub>asset</sub> ,并且随着利息索引存储而存储，blocks等于当前块的数量减去interestBlockNumber<sub>asset</sub> 。

- rate

  计算单利时，利率是指市场当前的利率。 这是之前blocks个块“生效”的利率。

- Exchange Rate Stored

  cToken到基础资产的最近存储的汇率。 这个不包含市场自上次累计利息以来的借入利息。

- Exchange Rate Current 
  cToken和基础资产之间的当前汇率（包括所有已调整的借款利息）。


# Exceptional States

我们假设在任何错误情况下：

a）协议正常退出，通过事件正常退出，如果尚未发生副作用，通过事件描述错误

b）交易失败完全。

除以下内容外，所有异常都需要注意。许多功能分为两个命令：累计利息和执行新动作。目标是分离应该发生的两个离散事件。首先，每次我们积累市场的利息，我们帮助平衡市场余额（并将单利转化为复利）。其次，截至该区块，市场利息已完全累积的情况下，新功能才是确的。随着利息的更新，这些新鲜的功能有不同的关注点（通常与累计利息无关）。因此，我们将这些功能构建为两个动作的总和，以简化理解和建模这两个单独的行为。实际上，这意味着即使某个方法正常失败，交易可能仍会对该市场累计利息（这是一件好事！）。

# Interest Rate Model Contract

对于每种资产，都有一个利息指数。 我们有效地跟踪了任意账户随着时间的推移本金的增长。 我们使用该帐户的利息与初始本金的比率计算该时间间隔内任何给定帐户的利息增长。利率模型合约规定了任意时刻的简单利率（即累计每笔交易就变成复利）。我们强迫这个利率模型是对市场上资产的现金、借款和储备的纯函数。更多信息，请参见利息指数计算附录。

**borrowRate**(address **cToken**, amount **cash**, amount **borrows**, amount **reserves**) **returns** uint

- 返回当前市场利率
- 注意：cToken是Compound cToken合约，不是基础资产地址。

# Price Oracle Contract

Compound协议使用来自智能合约的价格，即价格预言。Comptroller和Liquidate Borrow函数在此预言中引用价格。 对于不同的复合市场可能存在多个预言。

**getUnderlyingPrice**(CToken **cToken**) **returns** uint

- 返回标的资产的价格（以尾数表示）

# cToken Contract

cToken充当ERC-20接口，将成为用户与Compound协议交互的主要位置。当用户铸造，赎回，借入，偿还，清算借款或转让cToken，她将操作cToken合约。用户唯一需要做的是在另一份合约上执行进入，退出资产操作。（请参见下面的Comptroller contract）。

每个cToken都引用一个基础token。 不过，这通常是基本的ERC-20合约，它可能是Ether本身，也可能是复杂资产。 cToken是该基础货币资产余额的最终持有人，每次收取或发出资产的请求都源自cToken合约。最初，cEth（复合以太币令牌）将是唯一的资产，与Ether交互，而不是ERC-20资产。

# Note about cToken Money Markets

货币市场是第一版Compound Protocol的核心内容。 货币市场中曾经存在的功能现在存在于cToken中，而旧市场struct扩展到每个cToken市场。 与政策和流动性有关的函数推迟到Comptroller合约。

# Market Functions

borrowRatePerBlock()

- 返回此cToken的当前每个区块借贷利率尾数

supplyRatePerBlock()

- 计算供应利息：

$$underlying = totalSupply × exchangeRate$$
$$borrowsPer = totalBorrows ÷ underlying$$
$$supplyRate = borrowRate × (1 − reserveFactor) × borrowsPer$$

- 返回此cToken当前每个区块的供应利率尾数

# Accrue Interest()

我们会在每个操作中累计利息并更新借入指数。 这增加了复利，逼近真实值，无论其余操作成功与否。

- $totalCash = invoke:getCashPrior()$

  注意：可能会进行外部调用
- 我们获得了利率（自上次更新以来一直有效）：

$$borrowRate = call: interestModel.borrowRate(this, totalCash, totalBorrows, totalReserves)$$
$$simpleInterestFactor = Δblocks × borrowRate$$
- 更新borrowIndex
$$borrowIndexNew = borrowIndex × (1 + simpleInterestFactor)$$
- 计算累计利息
$$interestAccumulated = totalBorrows × simpleInterestFactor$$
- 更新borrows和reserves
$$totalBorrowsNew = totalBorrows + interestAccumulated$$
$$totalReservesNew = totalReserves + interestAccumulated × reserveFactor$$
- 把更新保存到区块链

  设置：accrualBlockNumber = getBlockNumber()

  设置：borrowIndex = borrowIndexNew

  设置：totalBorrows = totalBorrowsNew

  设置：totalReserves = totalReservesNew
- 发送AccrueInterest事件


[CErc20] **Mint**( uint **mintAmount**)

- 校验 调用 AccrueInterest() = 0
- 返回调用MintFresh(msg.sender, mintAmount)的结果

[CEther] **Mint()** **payable**
- 校验 调用 AccrueInterest() = 0
- 返回调用MintFresh(msg.sender, mintAmount)

[Internal] **MintFresh**( address **minter**, uint **mintAmount**)

用户从自己的地址向市场提供资产，作为交换，获得cToken余额。

- 如果调用$comptroller.mintAllowed(this, minter, mintAmount) \neq 0$, 则失败
- 校验市场的block number 等于当前的block number
- 如果调用checkTransferIn(minter, mintAmount) 失败，则失败
- 获取当前的汇率，计算铸造的cTokens数量：

$$exchangeRate = 调用:Exchange Rate Stored()$$
$$mintTokens = mintAmount ÷ exchangeRate$$

  >注意：除法将舍去为零，因此有可能铸造0个代币

- 我们计算出新的cToken总供应量和minter令牌余额：
$talSupplyNew = totalSupply + mintTokens$，如果溢出，则失败
$accountTokensNew_{minter} = accountTokens_{minter} + mintTokens$, 如果溢出，则失败

- 我们已经完成了计算。（如果任何计算由于错误而失败，我们将已返回失败代码）。现在我们可以开始产生影响了。

- 调用doTransferIn，对于minter和mintAmount

  注意：cToken必须处理ERC-20和底层ETH之间的变化。

  成功完成后，cToken会持有额外的mintAmount

  如果doTransferIn失败，尽管我们检查了前提条件，我们将回滚，因为我们不确定是否发生副作用

- 把之前计算的值保存起来：
$$设置：totalSupply = cTokenSupplyNew$$
$$设置：accountTokens_{minter} = accountTokensNew_{minter}$$

- 发送Mint事件，包括minter , mintAmount , mintTokens
- 发送this到minter的Transfer事件
- 调用comptroller.mintVerify(this, minter, mintAmount, mintTokens)

**Redeem**( uint **redeemTokens**)

- 校验调用AccrueInterest() = 0
- 调用RedeemFresh(msg.sender, redeemTokens, 0)，返回

**RedeemUnderlying**(uint **redeemAmount**)

- 校验调用AccrueInterest() = 0
- 调用RedeemFresh(msg.sender, 0, redeemAmount)，返回

[ Internal ] **RedeemFresh**(address **redeemer**, uint **redeemTokensIn**, uint **redeemAmountIn**)

用户放弃cToken并将协议中的基础ERC-20资产接收到她自己的钱包。

- exchangeRate = 调用 ExchangeRateStored()
- 如果redeemTokensIn > 0 :
  
  获取汇率，计算需要赎回的数量：

  $$redeemTokens = redeemTokensIn$$
  $$redeemAmount = redeemTokensIn × exchangeRate$$

- 否则，
  获取汇率，计算需要赎回的数量：
  $$redeemTokens = redeemAmountIn ÷ exchangeRate$$
  $$redeemAmount = redeemAmountIn$$
- 如果调用

   $comptroller.redeemAllowed(this, redeemer, redeemTokens) \neq 0$，则失败
- 校验市场的block number 等于当前的block number
- 计算新的total supply和redeemer token 余额：

  $totalSupplyNew = totalSupply − redeemTokens$, 如果溢出，则失败
  $accountTokensNew_{redeemer} = accountTokens_{redeemer} − redeemTokens$，如果溢出，则失败
- 如果协议没有足够的cash，正常失败
- 我们已经完成了计算。（如果任何计算由于错误而失败，我们将已返回失败代码）。现在我们可以开始产生影响了。
- 调用doTransferOut，对于redeemer和redeemAmount

  注意：cToken必须处理ERC-20和底层ETH之间的变化。

  成功完成后，cToken减少redeemAmount

  如果doTransferOut失败，尽管我们检查了前提条件，我们将回滚，因为我们不确定是否发生副作用

- 把计算的结果存储起来
$$设置： totalSupply = totalSupplyNew$$
$$设置：accountTokens_{redeemer} = accountTokensNew_{redeemer}$$
- 发送Redeem事件，包括redeemer , redeemAmount , redeemTokens
- 发送redeemer到this的Transfer事件
- 调用comptroller.redeemVerify(this, redeemer, redeemAmount, redeemTokens)

**Borrow**(uint **borrowAmount**)

- 校验AccrueInterest() = 0
- 调用BorrowAssetFresh(msg.sender, borrowAmount)，返回

[Internal] **BorrowFresh**(address **borrower**, uint **borrowAmount**)

用户从协议借款
- 如果调用$comptroller.borrowAllowed(this, borrower, borrowAmount) \neq 0$,则失败
- 校验市场的block number 等于当前的block number
- 计算新的borrower和总借款余额：
$$accountBorrows = 调用： BorrowBalanceStored(borrower)$$
$ccountBorrowsNew = accountBorrows + borrowAmount$,如果溢出，则失败

$totalBorrowsNew  = totalBorrows + borrowAmount$,如果溢出，则失败
- 如果协议没有足够的cash，正常失败
- 我们已经完成了计算。（如果任何计算由于错误而失败，我们将已返回失败代码）。现在我们可以开始产生影响了。
- 调用doTransferOut，对于borrower和borrowAmount

  注意：cToken必须处理ERC-20和底层ETH之间的变化。

  成功完成后，cToken减少borrowAmount

  如果doTransferOut失败，尽管我们检查了前提条件，我们将回滚，因为我们不确定是否发生副作用

- 把计算结果存储起来：
  $$设置：accountBorrows_{borrower} = \left \{  accountBorrowsNew, borrowIndex\right \}$$
  $$设置：totalBorrows = totalBorrowsNew$$

- 发送Borrow事件，包含borrower , borrowAmount , accountBorrowsNew ,
totalBorrowsNew
- 调用comptroller.borrowVerify(this, borrower, borrowAmount)

[CErc20]**RepayBorrow**(uint **repayAmount**)
- 校验AccrueInterest() = 0
- 调用RepayBorrowFresh(msg.sender, msg.sender, repayAmount)，返回

[CEther] **RepayBorrow**() payable

- 校验AccrueInterest() = 0
- 调用RepayBorrowFresh(msg.sender, msg.sender, msg.value)，返回

[CErc20] **RepayBorrowBehalf**(address **borrower**, uint **repayAmount**)

代表另一个用户偿还借款。 消息发送者仍然是付款人，但是您可以
指定其他帐户来付款。

- 校验AccrueInterest() = 0
- 调用RepayBorrowFresh(msg.sender, borrower, repayAmount)，返回

[CEther] **RepayBorrowBehalf**(address **borrower**) **payable**

代表另一个用户偿还借款。 消息发送者仍然是付款人，但是您可以
指定其他帐户来付款。
- 校验AccrueInterest() = 0
- 调用RepayBorrowFresh(msg.sender, borrower, repayAmount)，返回

[Internal] **RepayBorrowFresh**(address **payer**, address **borrower**, uint **repayAmount**)

借款由付款人（可能与借款人相同）偿还。

- 如果调用$comptroller.repayBorrowAllowed(this, payer, borrower, repayAmount) \neq 0$,则失败
- 校验市场的block number 等于当前的block number
- 获取borrower的借款，和累加的利息
$$accountBorrows = 调用： BorrowBalanceStored(borrower)$$
- 如果repayAmount = −1

  repayAmount = accountBorrows

- 如果checkTransferIn(underlying, payer, repayAmount) 失败，则失败
- 计算新的borrower和总的borrower余额：
$accountBorrowsNew = accountBorrows − repayAmount$,如果repayAmount > accountBorrows，则失败
$totalBorrowsNew = totalBorrows − repayAmount$,如果repayAmount > totalBorrows，则失败
- 我们已经完成了计算。（如果任何计算由于错误而失败，我们将已返回失败代码）。现在我们可以开始产生影响了。
- 调用doTransferIn，为了payer 和 repayAmount

  注意：cToken必须处理ERC-20和底层ETH之间的变化。

  成功完成后，cToken增加repayAmount

  如果doTransferIn失败，尽管我们检查了前提条件，我们将回滚，因为我们不确定是否发生副作用
- 把计算结果存储起来：
$$设置：accountBorrows_{borrower} = \left \{  accountBorrowsNew, borrowIndex\right \}$$
$$设置：totalBorrows = totalBorrowsNew$$
- 发送RepayBorrow事件，包含payer , borrower , repayAmount , accountBorrowsNew ,
totalBorrowsNew
- 调用comptroller.repayBorrowVerify(this, payer, borrower, repayAmount)

[CErc20] **LiquidateBorrow**(address **borrower**, CToken **cTokenCollateral**, uint **repayAmount**)
- 校验调用AccrueInterest() = 0
- 校验调用cTokenCollateral.AccrueInterest() = 0
- 调用LiquidateBorrowFresh(msg.sender, borrower, repayAmount, cTokenCollateral)，返回

[CEther] **LiquidateBorrow**(address **borrower**, CToken **cTokenCollateral**) **payable**
- 校验调用AccrueInterest() = 0
- 校验调用cTokenCollateral.AccrueInterest() = 0
- 调用LiquidateBorrowFresh(msg.sender, borrower, msg.value, cTokenCollateral)，返回

[Internal] **LiquidateBorrowFresh**(address **liquidator**, address **borrower**, uint **repayAmount**, CToken **cTokenCollateral**)

清算人代表借款人偿还该市场中的一定数量的基础资产，并在抵押市场上获取适当数量的代币

- 如果调用$comptroller.liquidateBorrowAllowed(this, ...arguments) \neq 0$，则失败
- 校验市场的block number 等于当前的block number
- 校验cTokenCollateral市场的block number等于当前的block number

如果$调用：cTokenCollateral.accrualBlockNumber() \neq block.number$，则失败

- 如果liquidator = borrower，则失败
- 如果repayAmount = 0，则失败
- 如果repayAmount =− 1，则失败
- 计算获取的collateral tokens数量
$$seizeTokens = 调用：comptroller.liquidateCalculateSeizeTokens(this,cTokenCollateral, repayAmount)$$

- 如果seizeTokens > cTokenCollateral.balanceOf(borrower)，则失败

- 如果调用$RepayBorrowFresh(liquidator, borrower, repayAmount) \neq 0$，则失败
- 如果调用$cTokenCollateral.seize(liquidator, borrower, seizeTokens) \neq 0$，则回滚
- 发送LiquidateBorrow事件，包含liquidator, borrower , repayAmount ,
cTokenCollateral , seizeTokens
- 调用comptroller.liquidateBorrowVerify(this, ...arguments, ...state)

**seize**(address **liquidator**, address **borrower**, uint **seizeTokens**) **returns** uint

- 如果调用$comptroller.seizeAllowed(this, msg.sender, liquidator, borrower, seizeTokens) \neq 0$，则失败

  注意：抵押合同必须使用msg.sender作为borrowed CToken的地址，这一点很重要。它用Comptroller校验。 如果有参数被使用，那么任何人都可以欺骗该调用。

- 如果borrower = liquidator，则失败
- 计算新的borrower和liquidator token balances：

$borrowerTokensNew = accountTokens[borrower] − seizeTokens$，如果溢出则失败
$liquidatorTokensNew = accountTokens[liquidator] + seizeTokens$，如果溢出，则失败

- 把计算结果存储起来：
$$ccountTokens[borrower] = borrowerTokensNew$$
$$accountTokens[liquidator] = liquidatorTokensNew$$
- 发送Transfer事件
- 调用comptroller.seizeVerify(this, msg.sender, liquidator, borrower, seizeTokens)

# Administrative Functions

**constructor**(EIP20 **underlying**, address **interestRateModel**, address **comptroller**, scaled **initialExchangeRate**)

- 将msg.sender设置为admin
- 将underlying设置为underlying
- 将initialExchangeRate设置为initialExchangeRate
- 将marketBlockNumber设置为block number
- 将market borrow index设置为1e18
- 将reserve factor设置为0
- 调用_setMarketInterestRateModelFresh(interestRateModel)
- 调用_setMarketComptroller(comptroller)

**_setReserveFactor**(scaled **newReserveFactor**)
- 校验调用AccrueInterest() = 0
- 调用_setReserveFactorFresh(newReserveFactor)，返回

[Internal] **_setReserveFactorFresh**(scaled **newReserveFactor**)
- 校验caller 是 admin
- 校验市场的block number等于当前block number
- 校验newReserveFactor ≤ maxReserveFactor
- 将reserveFactor保存为newReserveFactor
- 发送NewReserveFactor(oldReserveFactor, newReserveFactor)事件

**_reduceReserves**(uint **amount**)
- 校验调用AccrueInterest() = 0
- 调用_reduceReservesFresh(amount)，返回

[Internal] **_reduceReservesFresh**(uint **reduceAmount**)
- 校验caller是admin
- 校验市场的block number等于当前block number
- 校验$amount ≤ reserves_{n}$
- 如果协议没有足够的基础资金，正常失败
- 保存$rserves_{n+1} = reserves_{n} − reduceAmount$
- 调用doTransferOut(underlying, reduceAmount, admin)
  
  如果命令失败，则回滚

- 发送$NewReserves(admin, reduceAmount, reserves_{n+1} )$事件

**_setPendingAdmin**(address **newPendingAdmin**)
- 校验caller是admin
- 保存pendingAdmin为newPendingAdmin
- 发送NewPendingAdmin(oldPendingAdmin, newPendingAdmin)事件

**_acceptAdmin()**
- 校验caller是pendingAdmin，pendingAdmin ≠ address(0)
- 保存admin为pendingAdmin
- 复位pendingAdmin
- 发送NewAdmin(oldAdmin, newAdmin)事件
- 发送NewPendingAdmin(oldPendingAdmin, 0)事件

**_setInterestRateModel**(address **newInterestRateModel**)
- 校验调用AccrueInterest() = 0
- 调用_setFreshInterestModelFresh(newInterestRateModel)，返回

[Internal] **_setInterestRateModelFresh**(address **newInterestRateModel**)
- 校验caller是admin
- 校验市场的block number等于当前block number
- 跟踪市场当前的interest rate model
- 确保调用newInterestRateModel.isInterestRateModel()，返回true
- 设置利率模型为newInterestRateModel
- 发送NewInterestRateModel ( oldInterestRateModel , newInterestRateModel )事件

**_setComptroller**(address **newComptroller**)
- 校验caller是admin
- 跟踪市场资产当前的comptroller是oldComptroller
- 确保调用comptroller.isComptroller()，返回true
- 设置comptroller为newComptroller
- 发送NewComptroller(oldComptroller , newComptroller )事件


# Token Functions

**name()** **returns** string
- 返回name

**symbol()** **returns** string
- 返回symbol

**decimals()** **returns** uint
- 返回decimals

[External] **getCash()** **returns** uint
- 调用getCashPrior()，返回

**transfer**(address **to**, uint **amount**) **returns** bool
- 调用transferTokens(msg.sender, msg.sender, to, amount)
  
  如果失败，则回滚
  发送Transfer(msg.sender, to, tokens)事件

[Internal] **transferTokens**(address spender, address src, address dst, uint amount) returns uint

  从source转移amount的cTokens到dest
- 如果调用comptroller.redeemAllowed(this, src, amount)返回false，则失败
- 失败，除非spender是source，或者cTokenAllowed[src][spender] ≥ amount
- 如果accountTokens[src] < amount，则失败
- accountTokens[src] −= amount
  
  由于上面有校验，所以不可能溢出
- accountTokens[dst] += amount
  
  如果溢出，则失败
- !(spender = src或者cTokenAllowed[src][spender] = maxUint)

  cTokenAllowed[src][spender] −= amount

totalSupply() returns uint
- 返回totalSupply

allowance(address owner, address spender) returns uint
- 返回cTokenAllowed[owner][spender]

balanceOf(address account) returns uint256
- 返回accountTokens[account]

balanceOfUnderlying(address account) returns uint
- 返回accountTokens[account] × invoke ExchangeRateCurrent()

approve(address spender, uint256 amount) returns bool
- 重写cTokenAllowed[msg.sender][spender] = amount
- 发送Approval(msg.sender, spender, amount)事件

# Exchange Rates

ExchangeRateCurrent() returns uint
- 调用AccrueInterest()
- 调用ExchangeRateStored()，返回

ExchangeRateStored()
- 注意：不能保证market是最新的
- 如果，没有铸币：
  
  exchangeRate = initial exchange rate
- 否则，

  totalCash = 调用 getCash()，注意：可能要外部调用
  $$exchangeRate = \frac{totalCash +totalBorrows−totalReserves}{totalSupply}$$

- 返回exchangeRate

# Borrow Balances

TotalBorrowsCurrent(address account) returns uint
- 调用AccrueInterest()
- 返回totalBorrows

BorrowBalanceCurrent(address account) returns uint
- 调用AccrueInterest()
- 调用BorrowBalanceStored()

BorrowBalanceStored(address borrower)
- 注意：不能保证market是最新的
- 从cToken获取storage
$$borrowBalance_{borrower} = accountBorrows[borrower]$$
$$borrowIndex_{borrower} = accountBorrowIndex[borrower]$$
- 如果$borrowBalance_{borrower} = 0$,$borrowIndex_{borrower}$也可能是0，与其除数是0导致失败，直接返回0
- $$recentBorrowBalance_{borrower}=\frac{borrowBalance_{borrower}×borrowIndex_{stored}}{borrowIndex_{borrower}}$$
- 返回$recentBorrowBalance_{borrower}$

# Safe Token
[Internal] checkTransferIn(address account, uint underylingAmount) returns uint

- 调用EIP20(underlying).allowance(account, address(this)))，如果结果小于underlyingAmount，则失败
- 调用EIP20(underlying).balanceOf(account))，如果结果小于underlyingAmount，则失败
- 返回ok


[Internal] doTransferIn(address account, uint underlyingAmount)

- 如果msg.value>0，则回滚，没有有效的用户，发送value
- 调用EIP20(underlying).transferFrom(account, address(this), underlyingAmount)

  如果不是true，就回滚

  注意：要处理非标准ERC-20 tokens

[Internal] getCashPrior() returns uint
- 调用EIP20(underlying).balanceOf(address(this)))，返回

[Internal] doTransferOut(address account, uint underlyingAmount)
- 调用EIP20(underlying).transfer(account, underlyingAmount)

  如果不是true，就回滚

  注意：要处理非标准ERC-20 tokens

# Safe Token (cETH)

为了实现cETH，我们添加了一个fallback函数，该函数调用mint（msg.value）。
另外，我们重写安全令牌方法如下：

[Internal] checkTransferIn(address account, uint underylingAmount) returns uint
- 如果msg.sender ≠ account，则失败
- 如果msg.value ≠ underlyingAmount，则失败
- 返回ok

[Internal] doTransferIn(address account, uint underlyingAmount)

- 如果msg.value ≠ underlyingAmount，则失败

  我们只是进行完整性检查，应该已经调用了checkTransferIn

[Internal] getCashPrior() returns uint
- 返回this.balance − msg.value，确保避免溢出

[Internal] doTransferOut(address account, uint underlyingAmount)
- 调用account.transfer(underlyingAmount)，确保transfer需要的最小的gas

# Comptroller Contract

Comptroller实现Liquidity Checker API规范。 其中最重要的liquidityChecker.redeemAllowed(), liquidityChecker.borrowAllowed(), liquidityChecker.liquidateBorrowAllowed().

Comptroller还实现了防御挂钩机制，以防止意外发生的、未来的漏洞。 这些*Verify函数当前是禁止操作的，但只能采取最后手段，潜在地还原任何可能违反协议预期行为的协议事务，因此使用户资产面临风险。注意：为了无缝升级Comptroller而不更改cToken市场引用的Comptroller地址，有时我们使用一种称为委托的调用。

# User Market Functions

终端用户将直接调用这两个功能（进入市场和退出市场）。这仅是对希望借款的用户的要求。也就是说，代币持有者不借款就不需要（也不应该）调用这些函数。

EnterMarkets(address[] cTokens) returns bool[]

发件人包括在计算帐户流动性时应使用的资产地址列表。 在借用资产之前，必须以这种方式添加一个或多个提供的资产，以提供抵押。 必须以这种方式添加要借入的任何资产，然后才允许借入。 返回值是将传入的资产映射到用户是否最终在那个市场。
- 对于每个cToken，作为一个参数

  - 我们检查用户是否已经在cToken中，如果已经存在，则为true，否则，继续
  - 我们检查用户是否已达到maxAssets，如果达到，则为false，否则，继续
  - 我们检查cToken是否列出，如果没有列出，则为false，否则继续进行。
  - 校验$oraclePrice_{asset} \neq 0$
  - 将asset添加到$assets_{user}$,发布到用户的资产列表，设置$memberships_{cToken,user}$为true，为true
  - 继续进行下一个资产，请注意，如果上一项添加到列表中，列表大小增加，并且必须在存储时比较maxAssets
  - 发送MarketEntered(cToken, msg.sender )事件
- 返回关于用户当前是否在传递的cToken中

ExitMarket(address cToken) returns bool

发件人提供了不再希望包含在帐户流动性计算中的资产。由于必须包括所有借入资产，因此该功能的目的是从用户的抵押品清单中删除资产。 返回值是一个布尔值，指示调用后用户是否不在市场中。

- 获取sender的cToken的tokensHeld和amountOwed
- 如果用户有借款余额，则失败。amountOwed ≠ 0
- 如果用户不允许赎回所有的tokens，则失败
  - 例如，调用$redeemAllowed(cToken, msg.sender, tokensHeld) \neq 0$
- 如果用户已经不在市场中，则失败
- 设置cToken account membership为false
- 从账户的资产列表删除cToken
- 发送MarketExited (cToken, msg.sender )事件
- 返回true，用户不在这个市场中

# Policy Hook Functions

这些功能是验证是否允许用户执行给定操作的核心。 这应该是基于多种因素。一个或多个因素将由以下功能校验。

1.cToken必须是已知的“受支持”资产。 这适用于所有功能，并且必须总是被检查。
2.功能完成后，用户必须保持足够的流动性。 例如，如果用户有太多未偿还的借贷，则无法赎回代币。
3.用户必须声明她打算借入的所有资产（并用作抵押）。
4.用户必须通过所有的KYC校验和其他政策规则。
5.为了流动性，caller必须是资产本身（借入的cToken合约）

mintAllowed(CToken cToken, address minter, uint mintAmount) returns uint

- 如果cToken不在列表，则失败
- 可能包含PolicyHook-type 校验
- 返回ok

mintVerify(CToken cToken, address minter, uint mintAmount, uint mintTokens) returns uint

- 不做任何事情，将来可能回滚，作为防御钩子

redeemAllowed(CToken cToken, address redeemer, uint redeemTokens) returns uint

- 如果cToken不在列表，则失败
- 可能包含PolicyHook-type 校验
- 返回ok，如果redeemer没有资产关系
- (error, liquidity, shortfall) = 调用getHypotheticalAccountLiquidityInternal(redeemer, cToken, redeemTokens, 0)
- 如果$error \neq 0$ 或者shortfall > 0，返回失败
- 其他情况，返回ok

redeemVerify(CToken cToken, address redeemer, uint redeemAmount, uint redeemTokens) returns uint

- 不做任何事情，将来可能回滚，作为防御钩子

borrowAllowed(CToken cToken, address borrower, uint borrowAmount) returns uint
- 如果cToken不在列表，则失败
- 可能包含PolicyHook-type 校验
- 返回ok，如果borrower没有资产关系
- 如果$oraclePrice_{asset} = 0$，则失败
- (error, liquidity, shortfall) = 调用getHypotheticalAccountLiquidityInternal(borrower, cToken, 0, borrowAmount)
- 如果$error \neq 0$ 或者shortfall > 0，返回失败
- 其他情况，返回ok

borrowVerify(CToken cToken, address borrower, uint borrowAmount)
- 不做任何事情，将来可能回滚，作为防御钩子

repayBorrowAllowed(CToken cToken, address payer, address borrower, uint repayAmount) returns uint
- 如果cToken不在列表，则失败
- 可能包含PolicyHook-type 校验
- 否则，返回ok

repayBorrowVerify(CToken cToken, address payer, address borrower, uint repayAmount)
- 不做任何事情，将来可能回滚，作为防御钩子

liquidateBorrowAllowed(CToken cTokenBorrowed, CToken cTokenCollateral, address
borrower, address liquidator, uint repayAmount) returns uint
- 如果cTokenBorrowed或者cTokenCollateral不在列表，则失败
- 可能包含PolicyHook-type 校验
- (error, liquidity, shortfall) = 调用getAccountLiquidityInternal(borrower)
- 如果$error \neq 0$ 或者shortfall > 0，返回失败
  - borrower必须有shortfall，以便于可以清算
- $borrowBalance_{account} = 调用：cTokenBorrowed.BorrowBalanceStored()$
  - 在调用之前，要累加利息，以使这个值保持最新
- 计算maxCloseValue，这个借款人可以关闭的总额度：
  - $maxCloseValue = borrowBalance_{account}⋅closeFactor$
- 如果repayAmount > maxCloseValue，则失败
- 其他情况，返回ok

liquidateBorrowVerify(CToken cTokenBorrowed, CToken cTokenCollateral, address
borrower, address liquidator, uint repayAmount, uint seizeTokens)
- 不做任何事情，将来可能回滚，作为防御钩子

seizeAllowed(CToken cTokenCollateral, CToken cTokenBorrowed, address borrower,
address liquidator, uint seizeTokens) returns uint
- 如果cTokenCollateral或者cTokenBorrowed不在列表，则失败
- 可能包含PolicyHook-type 校验
- 调用$cTokenCollateral.comptroller() \neq call cTokenBorrowed.comptroller()$，则失败
- 其他情况，返回ok

seizeVerify(CToken cTokenCollateral, CToken cTokenBorrowed, address borrower, address liquidator, uint seizeTokens)
- 不做任何事情，将来可能回滚，作为防御钩子

transferAllowed(CToken cToken, address src, address dst, uint transferTokens) returns uint
- 返回redeemAllowed(cToken, src, transferTokens)

transferVerify(CToken cToken, address src, address dst, uint transferTokens)
- 不做任何事情，将来可能回滚，作为防御钩子

# Liquidity / Liquidation Calculations

getAssetsIn(address account) returns address[]
- 返回所在的资产列表

checkMembership(address account, CToken cToken) returns bool
- 如果用户在这个资产，返回true

getAccountLiquidity(address account) returns int
- 调用getHypotheticalAccountLiquidity(account, CToken(0), 0, 0)，返回

getHypotheticalAccountLiquidity(address account, CToken cToken, uint redeemTokens, uint borrowAmount) returns (uint, uint)

- $assets_{account}$是用户进入的资产列表
- 计算用户的sumCollateral和sumBorrowPlusEffects
  - 使用stored data为每一个抵押cToken计算exchangeRateStored，没有累加利息
- 初始化sumCollatera = 0 , sumBorrowPlusEffects = 0
- 对于每个$asset \in assets_{account} :$
  - $cTokenBalance_{account} = 调用: cToken.balanceOf(account)$
  - $borrowBalance_{account} = 调用: cToken.BorrowBalanceStored(account)$
  - $exchangeRate = 调用:  cToken.Exchange Rate Stored()$
  - $collateralFactor = markets[asset].collateralFactor$
  - $oraclePrice = 调用:  oracle.getUnderlyingPrice(asset)$,如果oraclePrice = 0，则失败
- tokensToDollars = collateralFactor · exchangeRate · oraclePrice
- $sumCollateral += tokensToDollars · cTokenBalance_{account}$
- $sumBorrowPlusEffects += oraclePrice · borrowBalance_{account}$
- 如果asset = cToken（在市场中查找）
  - 可能受影响的赎回
    - sumBorrowPlusEffects += tokensToDollars · redeemTokens
  - 可能受影响的借款
    - sumBorrowPlusEffects += oraclePrice · borrowAmount
- 如果sumCollateral > sumBorrowPlusEffects
  - liquidity = sumCollateral − sumBorrowPlusEffects
  - shortfall = 0
- 否则
  - liquidity = 0
  - shortfall = sumBorrowPlusEffects − sumCollateral
- 返回两个无符号数：liquidity和shortfall


liquidateCalculateSeizeTokens(CToken cTokenBorrowed, CToken cTokenCollateral, uint
repayAmount) returns uint

- 读取borrowed和collateral markets预言价格
  - $price_{borrowed} = call: oracle.getUnderlyingPrice(cTokenBorrowed)$
  - $price_{collateral} = call: oracle.getUnderlyingPrice(cTokenCollateral)$
  - 如果$price_{borrowed} = 0 或 price_{collateral} = 0$，则失败
- 获取exchange rate，计算获取的抵押tokens数量
  - $exchangeRate_{collateral} = call: cTokenCollateral.ExchangeRateStored()$
  - $seizeAmount = repayAmount × liquidationIncentive_{stored} × \frac{price_{collateral}}{price_{borrowed}}$
  - $seizeTokens = seizeAmount ÷ exchangeRate_{collateral}$
- 返回seizeTokens

# Comptroller Admin Functions

constructor()
- 设置caller为admin

_setPendingAdmin( address newPendingAdmin) returns uint
- 校验caller是admin
- 用新的newPendingAdmin替换PendingAdmin保存
- 发送NewPendingAdmin (oldPendingAdmin, newPendingAdmin)事件

_acceptAdmin() returns uint
- 校验caller是pendingAdmin，pendingAdmin ≠ address(0)
- 保存admin为pendingAdmin
- 复位pendingAdmin
- 发送NewAdmin (oldAdmin, newAdmin)事件
- 发送NewPendingAdmin (oldPendingAdmin, 0)事件

_setPriceOracle( address newOracle)
- 校验caller是admin，或者currentImplementation，origin是admin
- 确保调用newOracle.isPriceOracle()，返回true
- 设置comptroller's oracle为新的oracle
- 发送NewPriceOracle ( oldOracle , newOracle )事件

_setLiquidationIncentive( scaled newIncentive)
- 校验caller是admin，或者currentImplementation，origin是admin
- 校验de-scaled 1 ≤ newLiquidationDiscount ≤ 1.5
- 设置liquidation incentive为新的newIncentive
- 发送NewLiquidationIncentive (oldIncentive, newIncentive)事件

_setCollateralFactor( CToken cToken, scaled newFactor) returns uint
- 校验caller是admin，或者currentImplementation，origin是admin
- 校验market是在列表的
- 校验newFactor ≤ 0.9
- 如果newFactor > 0，oracle.getUnderlyingPrice(cToken)≠0，则失败
- 设置市场的collateral factor为newFactor
- 发送NewCollateralFactor ( cToken , oldFactor , newFactor)事件

_setCloseFactor( scaled newCloseFactor) returns uint
- 校验caller是admin，或者currentImplementation，origin是admin
- 校验0.05 < newCloseFactor ≤ 0.9
- 设置close factor为newCloseFactor
- 发送NewCloseFactor (old CloseFactor , newCloseFactor)事件

_setMaxAssets( uint newMaxAssets) returns uint
- 校验caller是admin，或者currentImplementation，origin是admin
- 设置maxAssets为newMaxAssets
- 发送NewMaxAssets (oldMaxAssets, newMaxAssets)事件

_supportMarket( CToken cToken) returns uint
- 校验caller是admin
- 校验asset不在列表
- 确保调用cToken.isCToken()返回true
- 设置market is listed为true
- 将cToken添加到市场列表
- 发送MarketListed (cToken)事件

# Implementation Upgrade Functions
comptroller设计为可升级的代理，其灵感来自于zeppelinOS描述的模式。 简而言之，这个模式是：
- 部署新的实现
- 调用comptroller._setPendingImplementation(newImplementation)
- 调用newImplementation.becomeBrains(comptroller, ...)

_setPendingImplementation( address newPendingImplementation) returns uint
- 校验caller是admin
- 设置pendingComptrollerImplementation的新值newPendingImplementation
- 发送NewPendingImplementation ( oldPendingAdminImplementation ,
newPendingImplementation )事件

_acceptImplementation() returns uint
- 校验caller是pendingComptrollerImplementation，pendingComptrollerImplementation ≠ address(0)
- 保存comptrollerImplementation的新值pendingComptrollerImplementation
- 复位pendingComptrollerImplementation
- 发送NewImplementation (oldImplementation, newImplementation)事件
- 发送NewPendingImplementation (oldPendingImplementation, 0)事件

>注意：becomeBrains被implementation调用，其他方法被active comptroller调用

_becomeBrains( address unitroller, address oracle, uint closeFactor, uint maxAssets) returns uint
- 校验caller是admin of unitroller
- 确保调用unitroller._acceptImplementation()，返回true
- 确保调用unitroller._setMarketPriceOracle(oracle) = 0
- 确保调用unitroller._setCloseFactor(closeFactor) = 0
- 确保调用unitroller._setMaxAssets(maxAssets) = 0
- 确保调用unitroller._setLiquidationIncentive(liquidationIncentiveMinMantissa) = 0

# Maximillion Contract
对于cErc20合约，我们支持使用特殊的“ -1”金额完全偿还借款。由于CToken合约被批准转让其所需的东西，它可以直接在repayBorrow函数确定借入金额、转移确定的金额，累计利息。

对于cEther，事情的运作方式略有不同。 由于“ -1”实际上是UINT_MAX，因此实际上
任何人都不可能将这笔款项转入还款功能。 为了彻底还清cEther的借款，我们部署了单独的合约来处理收款的详细信息，收集以太币足以偿还借贷加上最近的利息，然后退还多付的金额。

RepayBehalf( address borrower) payable
- 保存收到的Ether数量
  - received = msg.value
- 读取当前的借款金额和累计利息
  - borrows = invoke cEther.borrowBalanceCurrent(borrower)
- 如果received > borrows，偿还借款金额，退还
  - 调用cEther.repayBorrowBehalf.value(borrows)(borrower)
  - 返还received − borrows
- 否则，偿还提供的Ether额度
  - 调用cEther.repayBorrowBehalf.value(received)(borrower)

# Appendix
## Market States

给定的资产可能处于四个状态之一，这会影响哪些功能可用以及上面的资产使用方式。

- 不支持，资产不属于“listedAssets”集合的一部分。 在以下情况下不使用“supportMarket”的资产，计算sumCollateral和所有操作都应该失败。

- 在列表，资产是“listedAssets”集合的一部分。 在计算sumCollateral时不使用它，只有对该资产的“供应”，“提取”和“repayBorrow”操作应能正常运行。 该资产必须具有与其关联的利率模型。

- 借款，资产是“listedAssets”集合的一部分。 计算sumCollateral时不使用它，该资产上的所有操作应正常运行。 资产必须具有与其相关的非零价格和利率模型。市场非零价格允许从那个市场借钱。

- 抵押，资产是“listedAssets”集合的一部分。计算sumCollateral时使用它，该资产上的所有操作应正常运行。资产必须具有非零价格，非零抵押因子和利率模型
与之相关。 具有非零抵押因子的借贷市场是可借入且可用作抵押。

关于成员关系，listed⊆borrow⊆collateral（listed⋂unsupported）=⊘

关于可用功能，listed⊇borrow⊇collateral and unsupported=⊘

## Interest Index Calculation

利率指数（interest index）跟踪自协议部署以来，1美元（或恒定初始金额）债务的利息。固定利率的借款资产利息随着时间的推移。

每当利率发生变化时，指数（index）就会将简单的利率公式，应用于快照自那时以来的先前利率的影响：

$$index_{0} = c, c > 0$$
$$index_{i+△blocks} = index_{i} ⋅ (1 + △blocks × ratei)$$

每当索引更新时，我们也会将总借入（和备付金）金额更新为自上一次更新依赖，对该资产的所有当前借入（保留）单位的影响：

$$borrows_{i+△blocks} = borrows_{i} ⋅ (1 + △blocks × rate_{i})$$


总借入金额用于管理市场的分类帐，并经常更新为保持准确的最新信息。 另一方面，个人借贷只是在采取与该特定借用相关的操作时更新。

创建借款时，我们将其与本金和利息指数一起存储。 每当采取影响借款本金的行动时，我们首先根据上次事件以来累积的利息计算新的本金。为了确定累积了多少利息，我们采用当前指数值,并将其与存储在借款余额。 自本金最后一次记录以来，这些值的比率将产生$1 的债务变化。然后我们为其计算一个新的本金以与新的指数。

$borrowIndex_{i+b}/borrowIndex_{i}$比率，是这个阶段的单利，应用于整个期间，以产生本金利息：

$$effectiveRate = borrowIndex_{i+△blocks} / borrowIndex_{i}$$
$$principal_{i+△blocks} = principal_{i} ⋅ (1 + △blocks ⋅ effectiveRate) + Δ$$
$$borrowIndex_{i+△blocks} = borrowIndex_{i}$$
## Mint/Redeem exchange rate remains the same when minting and redeeming coins

设置
$$A = assets = cash + borrows$$，协议持有
$$L = liabilities = tokens$$,铸币，没有赎回
$$dA = assets变化 = cash supplied|withdrawn or borrows sent|repaid$$
$$dL = liabilities变化 = tokens to mint|redeem$$
$$R = 汇率 = A / L$$
$$A^{′} = A + dA$$
$$L^{′} = L + dL$$
$$R^{′} = A′ / L′$$

证明R′ = R，dA / dL = R = A / L
$$dL = dA / R = L * dA / A$$
$$R^{′} = A^{′} / L^{′}$$
$$= (A + dA) / (L + dL)$$
$$= (A + dA) / (L + L * dA / A)$$
$$= (A + dA) / (L * (1 + dA / A))$$
$$= (A + dA) / (L * (A / A + dA / A))$$
$$= (A + dA) / (L * (A + dA) / A)$$
$$= 1 / L / A$$
$$= A / L$$
$$= R$$


# 问题

## Comp合约


