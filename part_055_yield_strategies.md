# Part 055: Yield Strategies - Vault System

## สารบัญ (Table of Contents)
1. บทนำ Yield Strategies
2. Base Vault (ERC4626)
3. Strategy Interface
4. Compound Strategy
5. Aave Strategy
6. Strategy Harvesting
7. Multi-Strategy Vault
8. Tests

---

## 1. บทนำ Yield Strategies

Yield strategy เป็น pattern ที่:
- **Vault** เก็บ assets ของ users
- **Strategy** นำ assets ไปฝากใน protocol ต่างๆ เพื่อ yield
- **Harvesting** เก็บ rewards และ compound กลับ

**ERC-4626: Tokenized Vault Standard**
- Standardizes vault interface
- shares = receipt tokens สำหรับ depositors
- เปลี่ยน shares ↔ assets ผ่าน exchange rate

---

## 2. Base Vault (ERC-4626)

```vyper
# @version 0.4.0
# contracts/BaseVault.vy
# ERC-4626 Tokenized Vault Standard

from vyper.interfaces import ERC20

# Events
event Deposit:
    sender: indexed(address)
    owner: indexed(address)
    assets: uint256
    shares: uint256

event Withdraw:
    sender: indexed(address)
    receiver: indexed(address)
    owner: indexed(address)
    assets: uint256
    shares: uint256

event Transfer:
    sender: indexed(address)
    receiver: indexed(address)
    value: uint256

event Approval:
    owner: indexed(address)
    spender: indexed(address)
    value: uint256

event StrategyAdded:
    strategy: indexed(address)
    debtRatio: uint256

event StrategyRevoked:
    strategy: indexed(address)

event Harvested:
    strategy: indexed(address)
    gain: uint256
    loss: uint256
    debtPayment: uint256
    totalDebt: uint256

# ERC20 State
name: public(String[64])
symbol: public(String[32])
decimals: public(uint8)
totalSupply: public(uint256)
balanceOf: public(HashMap[address, uint256])
allowance: public(HashMap[address, HashMap[address, uint256]])

# Vault State
asset: public(address)      # underlying token
totalAssets: public(uint256)  # total assets managed

# Strategy State
struct StrategyParams:
    activation: uint256    # block ที่ activate
    debtRatio: uint256     # % debt ที่ strategy จัดการ (10000 = 100%)
    minDebtPerHarvest: uint256
    maxDebtPerHarvest: uint256
    totalDebt: uint256     # assets ที่ strategy มีอยู่
    totalGain: uint256
    totalLoss: uint256
    lastReport: uint256    # block ล่าสุดที่ harvest

strategies: public(HashMap[address, StrategyParams])
withdrawalQueue: DynArray[address, 20]  # ลำดับการถอนจาก strategies

debtRatioSum: uint256  # ผลรวมของ debtRatio ทุก strategy
totalDebt: uint256     # total ที่ strategies มีอยู่

governance: public(address)
management: public(address)
guardian: public(address)
treasury: public(address)

# Fees
managementFee: public(uint256)   # per year (200 = 2%)
performanceFee: public(uint256)  # per gain (2000 = 20%)
DEGRADATION_COEFFICIENT: constant(uint256) = 10**18
MAX_BPS: constant(uint256) = 10000

# Emergency
emergencyShutdown: public(bool)

@deploy
def __init__(
    _asset: address,
    _name: String[64],
    _symbol: String[32],
    _governance: address,
    _treasury: address
):
    self.asset = _asset
    self.name = _name
    self.symbol = _symbol
    self.decimals = 18
    self.governance = _governance
    self.treasury = _treasury
    self.management = _governance
    self.guardian = _governance
    
    self.managementFee = 200    # 2% annual
    self.performanceFee = 2000  # 20% of gains

# ===== ERC-4626 Core =====

@external
@view
def convertToShares(assets: uint256) -> uint256:
    """
    แปลง assets เป็น shares
    shares = assets * totalSupply / totalAssets
    """
    _totalSupply: uint256 = self.totalSupply
    _totalAssets: uint256 = self._totalAssets()
    
    if _totalSupply == 0 or _totalAssets == 0:
        return assets  # 1:1 สำหรับ first deposit
    
    return assets * _totalSupply / _totalAssets

@external
@view
def convertToAssets(shares: uint256) -> uint256:
    """
    แปลง shares เป็น assets
    assets = shares * totalAssets / totalSupply
    """
    _totalSupply: uint256 = self.totalSupply
    
    if _totalSupply == 0:
        return shares
    
    return shares * self._totalAssets() / _totalSupply

@internal
@view
def _totalAssets() -> uint256:
    """Total assets ทั้งหมด (vault + strategies)"""
    return ERC20(self.asset).balanceOf(self) + self.totalDebt

@external
def deposit(assets: uint256, receiver: address) -> uint256:
    """
    ฝาก assets เพื่อรับ shares
    
    Parameters:
        assets: จำนวน assets ที่ต้องการฝาก
        receiver: ผู้รับ shares
    
    Returns:
        shares: จำนวน shares ที่ได้รับ
    """
    assert not self.emergencyShutdown, "Emergency shutdown"
    assert assets > 0, "Zero assets"
    assert receiver != empty(address), "Zero receiver"
    
    shares: uint256 = self.convertToShares(assets)
    
    # โอน assets เข้า vault
    assert ERC20(self.asset).transferFrom(msg.sender, self, assets), "Transfer failed"
    
    # Mint shares
    self.totalSupply += shares
    self.balanceOf[receiver] += shares
    
    log Transfer(empty(address), receiver, shares)
    log Deposit(msg.sender, receiver, assets, shares)
    
    return shares

@external
def mint(shares: uint256, receiver: address) -> uint256:
    """Mint shares โดยระบุ shares amount"""
    assert not self.emergencyShutdown, "Emergency shutdown"
    
    assets: uint256 = self.convertToAssets(shares)
    
    assert ERC20(self.asset).transferFrom(msg.sender, self, assets), "Transfer failed"
    
    self.totalSupply += shares
    self.balanceOf[receiver] += shares
    
    log Transfer(empty(address), receiver, shares)
    log Deposit(msg.sender, receiver, assets, shares)
    
    return assets

@external
def withdraw(
    assets: uint256,
    receiver: address,
    owner: address
) -> uint256:
    """
    ถอน assets โดยระบุ assets amount
    
    Returns:
        shares: จำนวน shares ที่ burn
    """
    shares: uint256 = self.convertToShares(assets)
    
    if msg.sender != owner:
        allowed: uint256 = self.allowance[owner][msg.sender]
        assert allowed >= shares, "Insufficient allowance"
        self.allowance[owner][msg.sender] = allowed - shares
    
    assert self.balanceOf[owner] >= shares, "Insufficient shares"
    
    # ถอน assets จาก vault (หรือ strategies ถ้าจำเป็น)
    actualAssets: uint256 = self._withdraw(assets, receiver)
    
    # Burn shares
    self.balanceOf[owner] -= shares
    self.totalSupply -= shares
    
    log Transfer(owner, empty(address), shares)
    log Withdraw(msg.sender, receiver, owner, actualAssets, shares)
    
    return shares

@external
def redeem(
    shares: uint256,
    receiver: address,
    owner: address
) -> uint256:
    """Redeem shares เพื่อรับ assets"""
    if msg.sender != owner:
        allowed: uint256 = self.allowance[owner][msg.sender]
        assert allowed >= shares, "Insufficient allowance"
        self.allowance[owner][msg.sender] = allowed - shares
    
    assert self.balanceOf[owner] >= shares, "Insufficient shares"
    
    assets: uint256 = self.convertToAssets(shares)
    actualAssets: uint256 = self._withdraw(assets, receiver)
    
    self.balanceOf[owner] -= shares
    self.totalSupply -= shares
    
    log Transfer(owner, empty(address), shares)
    log Withdraw(msg.sender, receiver, owner, actualAssets, shares)
    
    return actualAssets

@internal
def _withdraw(assets: uint256, receiver: address) -> uint256:
    """
    ถอน assets จาก vault หรือ strategies
    ดึงจาก strategies ตาม withdrawalQueue ถ้าใน vault ไม่พอ
    """
    vaultBalance: uint256 = ERC20(self.asset).balanceOf(self)
    
    if vaultBalance >= assets:
        # มีพอใน vault
        assert ERC20(self.asset).transfer(receiver, assets), "Transfer failed"
        return assets
    
    # ต้องดึงจาก strategies
    remaining: uint256 = assets - vaultBalance
    
    for strategy: address in self.withdrawalQueue:
        if remaining == 0:
            break
        
        strategyBalance: uint256 = self.strategies[strategy].totalDebt
        if strategyBalance == 0:
            continue
        
        withdrawAmount: uint256 = remaining
        if withdrawAmount > strategyBalance:
            withdrawAmount = strategyBalance
        
        # เรียก strategy ให้ withdraw
        # IStrategy(strategy).withdraw(withdrawAmount)
        
        self.strategies[strategy].totalDebt -= withdrawAmount
        self.totalDebt -= withdrawAmount
        remaining -= withdrawAmount
    
    totalWithdrawn: uint256 = assets - remaining
    assert ERC20(self.asset).transfer(receiver, totalWithdrawn), "Transfer failed"
    
    return totalWithdrawn

# ===== Strategy Management =====

@external
def addStrategy(
    strategy: address,
    debtRatio: uint256,
    minDebtPerHarvest: uint256,
    maxDebtPerHarvest: uint256
):
    """เพิ่ม strategy ใหม่"""
    assert msg.sender == self.governance, "Not governance"
    assert strategy != empty(address), "Zero strategy"
    assert self.strategies[strategy].activation == 0, "Strategy exists"
    assert self.debtRatioSum + debtRatio <= MAX_BPS, "Debt ratio too high"
    
    self.strategies[strategy] = StrategyParams({
        activation: block.number,
        debtRatio: debtRatio,
        minDebtPerHarvest: minDebtPerHarvest,
        maxDebtPerHarvest: maxDebtPerHarvest,
        totalDebt: 0,
        totalGain: 0,
        totalLoss: 0,
        lastReport: block.number
    })
    
    self.debtRatioSum += debtRatio
    self.withdrawalQueue.append(strategy)
    
    log StrategyAdded(strategy, debtRatio)

@external
def revokeStrategy(strategy: address):
    """ยกเลิก strategy"""
    assert msg.sender == self.governance or msg.sender == self.management, "Not authorized"
    
    self.debtRatioSum -= self.strategies[strategy].debtRatio
    self.strategies[strategy].debtRatio = 0
    
    log StrategyRevoked(strategy)

@external
def report(
    gain: uint256,
    loss: uint256,
    debtPayment: uint256
) -> uint256:
    """
    Strategy รายงานผล
    เรียกโดย strategy หลัง harvest
    
    Parameters:
        gain: กำไรจาก harvest
        loss: ขาดทุนจาก strategy
        debtPayment: จำนวนที่คืนให้ vault
    
    Returns:
        debt: debt ที่ strategy ควรมี
    """
    strategy: address = msg.sender
    
    assert self.strategies[strategy].activation != 0, "Not a strategy"
    
    # Performance fee
    if gain > 0:
        feeAmount: uint256 = gain * self.performanceFee / MAX_BPS
        # Mint shares สำหรับ treasury
        treasuryShares: uint256 = feeAmount * self.totalSupply / self._totalAssets()
        self.balanceOf[self.treasury] += treasuryShares
        self.totalSupply += treasuryShares
        
        self.strategies[strategy].totalGain += gain
    
    if loss > 0:
        self.strategies[strategy].totalLoss += loss
        if loss > self.strategies[strategy].totalDebt:
            loss = self.strategies[strategy].totalDebt
        self.strategies[strategy].totalDebt -= loss
        self.totalDebt -= loss
    
    # คำนวณ debt ที่ควรเป็น
    totalFreeAssets: uint256 = ERC20(self.asset).balanceOf(self) + debtPayment
    debtTarget: uint256 = self._totalAssets() * self.strategies[strategy].debtRatio / MAX_BPS
    
    debt: uint256 = 0
    if totalFreeAssets + self.strategies[strategy].totalDebt > debtTarget:
        # ต้องคืน debt
        debt = 0
    else:
        # ส่ง assets เพิ่ม
        debt = debtTarget - self.strategies[strategy].totalDebt
        if debt > totalFreeAssets:
            debt = totalFreeAssets
        
        if debt > 0:
            assert ERC20(self.asset).transfer(strategy, debt), "Transfer failed"
            self.strategies[strategy].totalDebt += debt
            self.totalDebt += debt
    
    self.strategies[strategy].lastReport = block.number
    
    log Harvested(strategy, gain, loss, debtPayment, self.totalDebt)
    
    return debt

# ===== ERC20 =====

@external
def transfer(to: address, amount: uint256) -> bool:
    assert self.balanceOf[msg.sender] >= amount, "Insufficient"
    self.balanceOf[msg.sender] -= amount
    self.balanceOf[to] += amount
    log Transfer(msg.sender, to, amount)
    return True

@external
def transferFrom(from_: address, to: address, amount: uint256) -> bool:
    if self.allowance[from_][msg.sender] != max_value(uint256):
        assert self.allowance[from_][msg.sender] >= amount, "Insufficient allowance"
        self.allowance[from_][msg.sender] -= amount
    assert self.balanceOf[from_] >= amount, "Insufficient"
    self.balanceOf[from_] -= amount
    self.balanceOf[to] += amount
    log Transfer(from_, to, amount)
    return True

@external
def approve(spender: address, amount: uint256) -> bool:
    self.allowance[msg.sender][spender] = amount
    log Approval(msg.sender, spender, amount)
    return True

# ===== Admin =====

@external
def setEmergencyShutdown(active: bool):
    assert msg.sender == self.governance or msg.sender == self.guardian, "Not authorized"
    self.emergencyShutdown = active

@external
def setManagementFee(fee: uint256):
    assert msg.sender == self.governance, "Not governance"
    assert fee <= 500, "Too high"  # Max 5%
    self.managementFee = fee

@external
def setPerformanceFee(fee: uint256):
    assert msg.sender == self.governance, "Not governance"
    assert fee <= 5000, "Too high"  # Max 50%
    self.performanceFee = fee
```

---

## 3. Strategy Interface

```vyper
# @version 0.4.0
# contracts/BaseStrategy.vy
# Base class สำหรับ yield strategies

from vyper.interfaces import ERC20

interface IVault:
    def asset() -> address: view
    def report(gain: uint256, loss: uint256, debtPayment: uint256) -> uint256: nonpayable
    def strategies(strategy: address) -> (uint256, uint256, uint256, uint256, uint256, uint256, uint256, uint256): view

# Events
event Harvested:
    profit: uint256
    loss: uint256
    debtPayment: uint256
    debtOutstanding: uint256

event SetKeeper:
    keeper: indexed(address)

# State
vault: public(address)
want: public(address)     # underlying token
strategist: public(address)
keeper: public(address)
rewards: public(address)

# Booleans
isActive: public(bool)
emergencyExit: bool

@deploy
def __init__(
    _vault: address,
    _strategist: address,
    _rewards: address,
    _keeper: address
):
    self.vault = _vault
    self.want = IVault(_vault).asset()
    self.strategist = _strategist
    self.rewards = _rewards
    self.keeper = _keeper
    self.isActive = True

@internal
def _protectedTokens() -> DynArray[address, 10]:
    """Tokens ที่ห้าม sweep ออก"""
    tokens: DynArray[address, 10] = [self.want]
    return tokens

@external
@view
def estimatedTotalAssets() -> uint256:
    """
    ประมาณ assets ทั้งหมดที่ strategy มี
    รวม want balance + assets ที่ deploy ไปแล้ว
    """
    return ERC20(self.want).balanceOf(self)  # Simplified

@internal
def _prepareReturn(debtOutstanding: uint256) -> (uint256, uint256, uint256):
    """
    เตรียม assets สำหรับคืนให้ vault
    
    Returns:
        profit: กำไร
        loss: ขาดทุน
        debtPayment: จำนวนที่คืน
    """
    return 0, 0, 0  # Override in subclass

@internal
def _adjustPosition(debtOutstanding: uint256):
    """
    ปรับ position ตาม strategy
    Override in subclass
    """
    pass

@internal
def _liquidatePosition(amountNeeded: uint256) -> (uint256, uint256):
    """
    Liquidate position เพื่อถอน assets
    
    Returns:
        liquidatedAmount: จำนวนที่ถอนได้
        loss: ขาดทุนจากการถอน
    """
    wantBal: uint256 = ERC20(self.want).balanceOf(self)
    if wantBal >= amountNeeded:
        return amountNeeded, 0
    return wantBal, 0  # Simplified

@external
def harvest():
    """
    Harvest yield และรายงานให้ vault
    เรียกโดย keeper เป็นระยะ
    """
    assert msg.sender == self.keeper or msg.sender == self.strategist, "Not authorized"
    
    # ดึง debt outstanding จาก vault
    # _, _, _, _, totalDebt, _, _, _ = IVault(self.vault).strategies(self)
    debtOutstanding: uint256 = 0  # Simplified
    
    profit: uint256 = 0
    loss: uint256 = 0
    debtPayment: uint256 = 0
    
    if self.emergencyExit:
        # Emergency: liquidate ทั้งหมด
        total: uint256 = self.estimatedTotalAssets()
        liquidated: uint256 = 0
        lossAmount: uint256 = 0
        liquidated, lossAmount = self._liquidatePosition(total)
        
        debtPayment = liquidated
        loss = lossAmount
    else:
        profit, loss, debtPayment = self._prepareReturn(debtOutstanding)
    
    # Approve vault
    assert ERC20(self.want).approve(self.vault, debtPayment + profit), "Approve failed"
    
    # Report to vault
    debt: uint256 = IVault(self.vault).report(profit, loss, debtPayment)
    
    # ปรับ position
    if not self.emergencyExit:
        self._adjustPosition(debt)
    
    log Harvested(profit, loss, debtPayment, debt)

@external
def withdraw(amountNeeded: uint256) -> uint256:
    """
    Withdraw จาก strategy สำหรับ vault
    เรียกโดย vault เท่านั้น
    """
    assert msg.sender == self.vault, "Not vault"
    
    amountFreed: uint256 = 0
    loss: uint256 = 0
    amountFreed, loss = self._liquidatePosition(amountNeeded)
    
    assert ERC20(self.want).transfer(self.vault, amountFreed), "Transfer failed"
    
    return loss

@external
def setEmergencyExit():
    """เปิด emergency exit mode"""
    assert msg.sender == self.strategist, "Not strategist"
    self.emergencyExit = True

@external
def setKeeper(newKeeper: address):
    assert msg.sender == self.strategist, "Not strategist"
    self.keeper = newKeeper
    log SetKeeper(newKeeper)

@external
def sweep(token: address, amount: uint256):
    """ดึง tokens ที่ไม่ต้องการออก"""
    assert msg.sender == self.governance(), "Not governance"
    
    protected: DynArray[address, 10] = self._protectedTokens()
    for t: address in protected:
        assert token != t, "Protected token"
    
    assert ERC20(token).transfer(self.rewards, amount), "Transfer failed"

@internal
@view
def governance() -> address:
    return self.strategist  # Simplified
```

---

## 4. Compound Strategy

```vyper
# @version 0.4.0
# contracts/CompoundStrategy.vy
# Yield strategy ที่ฝากใน Compound

from vyper.interfaces import ERC20

interface ICToken:
    def mint(mintAmount: uint256) -> uint256: nonpayable
    def redeem(redeemTokens: uint256) -> uint256: nonpayable
    def redeemUnderlying(redeemAmount: uint256) -> uint256: nonpayable
    def balanceOf(account: address) -> uint256: view
    def exchangeRateCurrent() -> uint256: nonpayable
    def exchangeRateStored() -> uint256: view
    def balanceOfUnderlying(account: address) -> uint256: view

interface IComptroller:
    def claimComp(holder: address): nonpayable
    def compAccrued(holder: address) -> uint256: view

interface IUniswapRouter:
    def swapExactTokensForTokens(
        amountIn: uint256,
        amountOutMin: uint256,
        path: DynArray[address, 5],
        to: address,
        deadline: uint256
    ) -> DynArray[uint256, 5]: nonpayable

interface IVault:
    def asset() -> address: view
    def report(gain: uint256, loss: uint256, debtPayment: uint256) -> uint256: nonpayable

# Events
event Harvested:
    profit: uint256
    loss: uint256
    compClaimed: uint256
    compSold: uint256

# State
vault: public(address)
want: public(address)
cToken: public(address)
comp: public(address)
comptroller: public(address)
uniswapRouter: public(address)
weth: public(address)

strategist: public(address)
keeper: public(address)

minCompToSell: uint256  # ขาย COMP ขั้นต่ำ (ป้องกัน gas waste)

@deploy
def __init__(
    _vault: address,
    _cToken: address,
    _comp: address,
    _comptroller: address,
    _uniswapRouter: address,
    _weth: address,
    _strategist: address
):
    self.vault = _vault
    self.want = IVault(_vault).asset()
    self.cToken = _cToken
    self.comp = _comp
    self.comptroller = _comptroller
    self.uniswapRouter = _uniswapRouter
    self.weth = _weth
    self.strategist = _strategist
    self.keeper = _strategist
    self.minCompToSell = 10**17  # 0.1 COMP

@internal
@view
def _valueInWant(compAmount: uint256) -> uint256:
    """ประมาณมูลค่า COMP ใน want token"""
    # ใน production: ใช้ oracle หรือ AMM price
    return compAmount  # Placeholder

@internal
def _claimAndSellComp() -> uint256:
    """Claim COMP rewards และขายเป็น want token"""
    compAccrued: uint256 = IComptroller(self.comptroller).compAccrued(self)
    
    if compAccrued < self.minCompToSell:
        return 0
    
    # Claim COMP
    IComptroller(self.comptroller).claimComp(self)
    
    compBalance: uint256 = ERC20(self.comp).balanceOf(self)
    
    if compBalance == 0:
        return 0
    
    # Sell COMP -> want ผ่าน Uniswap
    # path: COMP -> WETH -> want
    ERC20(self.comp).approve(self.uniswapRouter, compBalance)
    
    path: DynArray[address, 5] = [self.comp, self.weth, self.want]
    amounts: DynArray[uint256, 5] = IUniswapRouter(self.uniswapRouter).swapExactTokensForTokens(
        compBalance,
        0,  # ไม่มี slippage protection (simplified)
        path,
        self,
        block.timestamp + 300
    )
    
    return amounts[len(amounts) - 1]

@internal
@view
def _getUnderlyingBalance() -> uint256:
    """ดู balance ของ want ที่ฝากใน cToken"""
    cTokenBalance: uint256 = ICToken(self.cToken).balanceOf(self)
    exchangeRate: uint256 = ICToken(self.cToken).exchangeRateStored()
    return cTokenBalance * exchangeRate / 10**18

@external
@view
def estimatedTotalAssets() -> uint256:
    """Total assets = want balance + underlying ใน cToken"""
    return ERC20(self.want).balanceOf(self) + self._getUnderlyingBalance()

@internal
def _depositWant(amount: uint256):
    """ฝาก want เข้า Compound"""
    if amount == 0:
        return
    
    ERC20(self.want).approve(self.cToken, amount)
    ICToken(self.cToken).mint(amount)

@internal
def _withdrawWant(amount: uint256) -> uint256:
    """ถอน want จาก Compound"""
    if amount == 0:
        return 0
    
    underlyingBalance: uint256 = self._getUnderlyingBalance()
    withdrawAmount: uint256 = amount
    if withdrawAmount > underlyingBalance:
        withdrawAmount = underlyingBalance
    
    ICToken(self.cToken).redeemUnderlying(withdrawAmount)
    return withdrawAmount

@external
def harvest():
    """Harvest COMP rewards และ report ให้ vault"""
    assert msg.sender == self.keeper or msg.sender == self.strategist, "Not authorized"
    
    # Claim และขาย COMP
    wantFromComp: uint256 = self._claimAndSellComp()
    
    # คำนวณ profit/loss
    totalAssets: uint256 = self.estimatedTotalAssets()
    debt: uint256 = 0  # จาก vault.strategies
    
    profit: uint256 = 0
    loss: uint256 = 0
    
    if totalAssets > debt:
        profit = totalAssets - debt
    elif debt > totalAssets:
        loss = debt - totalAssets
    
    debtPayment: uint256 = 0
    
    # Report ให้ vault
    ERC20(self.want).approve(self.vault, profit + debtPayment)
    newDebt: uint256 = IVault(self.vault).report(profit, loss, debtPayment)
    
    # ฝาก/ถอนตาม debt target
    wantBalance: uint256 = ERC20(self.want).balanceOf(self)
    
    if wantBalance > 0:
        self._depositWant(wantBalance)
    
    log Harvested(profit, loss, 0, wantFromComp)

@external
def withdraw(amountNeeded: uint256) -> uint256:
    """Withdraw สำหรับ vault"""
    assert msg.sender == self.vault, "Not vault"
    
    wantBalance: uint256 = ERC20(self.want).balanceOf(self)
    
    if wantBalance < amountNeeded:
        self._withdrawWant(amountNeeded - wantBalance)
    
    wantBalance = ERC20(self.want).balanceOf(self)
    
    if wantBalance < amountNeeded:
        ERC20(self.want).transfer(self.vault, wantBalance)
        return amountNeeded - wantBalance  # loss
    
    ERC20(self.want).transfer(self.vault, amountNeeded)
    return 0
```

---

## 5. Aave Strategy

```vyper
# @version 0.4.0
# contracts/AaveStrategy.vy
# Yield strategy ที่ฝากใน Aave V3

from vyper.interfaces import ERC20

interface IAaveLendingPool:
    def supply(asset: address, amount: uint256, onBehalfOf: address, referralCode: uint16): nonpayable
    def withdraw(asset: address, amount: uint256, to: address) -> uint256: nonpayable
    def getUserAccountData(user: address) -> (uint256, uint256, uint256, uint256, uint256, uint256): view

interface IAToken:
    def balanceOf(account: address) -> uint256: view
    def scaledBalanceOf(account: address) -> uint256: view

interface IAaveRewards:
    def claimRewards(assets: DynArray[address, 5], amount: uint256, to: address, rewardToken: address) -> uint256: nonpayable
    def getUserRewards(assets: DynArray[address, 5], user: address, reward: address) -> uint256: view

interface IVault:
    def asset() -> address: view
    def report(gain: uint256, loss: uint256, debtPayment: uint256) -> uint256: nonpayable

# Events
event Harvested:
    profit: uint256
    loss: uint256
    rewardsClaimed: uint256

# State
vault: public(address)
want: public(address)
aToken: public(address)        # aToken ของ want ใน Aave
lendingPool: public(address)
rewards: public(address)       # Aave rewards contract
rewardToken: public(address)   # AAVE token

strategist: public(address)
keeper: public(address)

@deploy
def __init__(
    _vault: address,
    _aToken: address,
    _lendingPool: address,
    _rewards: address,
    _rewardToken: address,
    _strategist: address
):
    self.vault = _vault
    self.want = IVault(_vault).asset()
    self.aToken = _aToken
    self.lendingPool = _lendingPool
    self.rewards = _rewards
    self.rewardToken = _rewardToken
    self.strategist = _strategist
    self.keeper = _strategist

@external
@view
def estimatedTotalAssets() -> uint256:
    """aToken balance = underlying balance (1:1 ใน Aave)"""
    return IAToken(self.aToken).balanceOf(self) + ERC20(self.want).balanceOf(self)

@internal
def _depositWant(amount: uint256):
    """ฝาก want เข้า Aave"""
    ERC20(self.want).approve(self.lendingPool, amount)
    IAaveLendingPool(self.lendingPool).supply(self.want, amount, self, 0)

@internal
def _withdrawWant(amount: uint256) -> uint256:
    """ถอน want จาก Aave"""
    return IAaveLendingPool(self.lendingPool).withdraw(self.want, amount, self)

@internal
def _claimRewards() -> uint256:
    """Claim AAVE rewards"""
    assets: DynArray[address, 5] = [self.aToken]
    pendingRewards: uint256 = IAaveRewards(self.rewards).getUserRewards(
        assets,
        self,
        self.rewardToken
    )
    
    if pendingRewards == 0:
        return 0
    
    claimed: uint256 = IAaveRewards(self.rewards).claimRewards(
        assets,
        max_value(uint256),
        self,
        self.rewardToken
    )
    
    return claimed

@external
def harvest():
    """Harvest AAVE rewards และ report"""
    assert msg.sender == self.keeper or msg.sender == self.strategist, "Not authorized"
    
    rewardsClaimed: uint256 = self._claimRewards()
    
    totalAssets: uint256 = self.estimatedTotalAssets()
    debt: uint256 = 0  # จาก vault
    
    profit: uint256 = 0
    loss: uint256 = 0
    
    if totalAssets > debt:
        profit = totalAssets - debt
    else:
        loss = debt - totalAssets
    
    ERC20(self.want).approve(self.vault, profit)
    newDebt: uint256 = IVault(self.vault).report(profit, loss, 0)
    
    wantBal: uint256 = ERC20(self.want).balanceOf(self)
    if wantBal > 0:
        self._depositWant(wantBal)
    
    log Harvested(profit, loss, rewardsClaimed)

@external
def withdraw(amountNeeded: uint256) -> uint256:
    assert msg.sender == self.vault, "Not vault"
    
    wantBal: uint256 = ERC20(self.want).balanceOf(self)
    
    if wantBal < amountNeeded:
        self._withdrawWant(amountNeeded - wantBal)
    
    wantBal = ERC20(self.want).balanceOf(self)
    
    if wantBal < amountNeeded:
        ERC20(self.want).transfer(self.vault, wantBal)
        return amountNeeded - wantBal
    
    ERC20(self.want).transfer(self.vault, amountNeeded)
    return 0
```

---

## 6. Tests

```python
# tests/test_vault.py
import pytest
from brownie import BaseVault, MockERC20, accounts, chain

SCALE = 10**18

@pytest.fixture
def setup():
    owner = accounts[0]
    alice = accounts[1]
    bob = accounts[2]
    
    want = MockERC20.deploy("USDC", "USDC", 6, {"from": owner})
    
    vault = BaseVault.deploy(
        want.address,
        "yvUSDC",
        "yvUSDC",
        owner.address,
        owner.address,
        {"from": owner}
    )
    
    # Mint tokens
    want.mint(alice, 10**9, {"from": owner})
    want.mint(bob, 10**9, {"from": owner})
    
    return owner, alice, bob, want, vault

def test_deposit(setup):
    owner, alice, bob, want, vault = setup
    
    deposit_amount = 10**8  # 100 USDC
    want.approve(vault.address, deposit_amount, {"from": alice})
    
    shares = vault.deposit(deposit_amount, alice.address, {"from": alice})
    
    assert vault.balanceOf(alice.address) == shares
    assert vault.totalSupply() == shares
    print(f"Deposited {deposit_amount}, received {shares} shares")

def test_withdraw(setup):
    owner, alice, bob, want, vault = setup
    
    deposit_amount = 10**8
    want.approve(vault.address, deposit_amount, {"from": alice})
    shares = vault.deposit(deposit_amount, alice.address, {"from": alice})
    
    # Withdraw half
    withdraw_shares = shares // 2
    
    before_balance = want.balanceOf(alice.address)
    vault.redeem(withdraw_shares, alice.address, alice.address, {"from": alice})
    after_balance = want.balanceOf(alice.address)
    
    assert after_balance > before_balance
    print(f"Redeemed {withdraw_shares} shares for {after_balance - before_balance} USDC")

def test_exchange_rate(setup):
    owner, alice, bob, want, vault = setup
    
    # Alice deposits
    want.approve(vault.address, 10**8, {"from": alice})
    vault.deposit(10**8, alice.address, {"from": alice})
    
    # Bob deposits
    want.approve(vault.address, 10**8, {"from": bob})
    vault.deposit(10**8, bob.address, {"from": bob})
    
    # Both should have same shares ratio
    alice_shares = vault.balanceOf(alice.address)
    bob_shares = vault.balanceOf(bob.address)
    
    alice_assets = vault.convertToAssets(alice_shares)
    bob_assets = vault.convertToAssets(bob_shares)
    
    print(f"Alice: {alice_shares} shares = {alice_assets} USDC")
    print(f"Bob: {bob_shares} shares = {bob_assets} USDC")
    
    # Should be equal (same deposit)
    assert abs(alice_assets - bob_assets) <= 1  # Allow 1 wei rounding

def test_multiple_deposits(setup):
    owner, alice, bob, want, vault = setup
    
    # Initial deposit
    want.approve(vault.address, 10**8, {"from": alice})
    shares1 = vault.deposit(10**8, alice.address, {"from": alice})
    
    exchange_rate_before = vault.convertToAssets(10**18)
    
    # Simulate yield (send tokens directly to vault)
    want.mint(vault.address, 10**7, {"from": owner})  # 10% yield
    
    exchange_rate_after = vault.convertToAssets(10**18)
    
    print(f"Exchange rate before: {exchange_rate_before}")
    print(f"Exchange rate after: {exchange_rate_after}")
    assert exchange_rate_after > exchange_rate_before
    
    # Bob deposits AFTER yield
    want.approve(vault.address, 10**8, {"from": bob})
    shares2 = vault.deposit(10**8, bob.address, {"from": bob})
    
    # Bob should get fewer shares (exchange rate increased)
    assert shares2 < shares1
    print(f"Alice shares: {shares1}, Bob shares: {shares2}")
```

---

## 7. สรุป

### ERC-4626 Key Concepts:

**1. Exchange Rate**
- เพิ่มขึ้นเมื่อ vault ทำ yield
- Depositors ที่มาก่อนได้ประโยชน์เมื่อ rate สูง

**2. Strategy Lifecycle:**
```
addStrategy() -> deposit() -> harvest() (ทุก N blocks) -> withdraw()
```

**3. Debt Ratio**
- แต่ละ strategy มี % debt ที่รับผิดชอบ
- vault จัดสรร assets ตาม ratio
- รวมกันต้องไม่เกิน 100%

**4. Emergency Shutdown**
- ปิด deposit ทั้งหมด
- strategies liquidate positions
- users สามารถ withdraw ได้เท่านั้น

**5. Fee Structure**
- Management fee: คิดต่อปี จาก AUM
- Performance fee: คิดจาก gains เท่านั้น
- Protocol-friendly: users จ่ายตาม performance
