# Part 054: Lending Protocol - Complete Implementation

## สารบัญ (Table of Contents)
1. บทนำ Lending Protocol
2. Interest Rate Model
3. Collateral Management
4. Supply / Withdraw
5. Borrow / Repay
6. Health Factor
7. Liquidation Mechanism
8. Price Oracle Integration
9. Risk Parameters
10. Tests

---

## 1. บทนำ Lending Protocol

Lending Protocol ช่วยให้ผู้ใช้สามารถ:
- **Supply**: ฝาก assets เพื่อรับ interest
- **Borrow**: ยืม assets โดยใช้ collateral
- **Repay**: คืน assets ที่ยืม
- **Liquidate**: ชำระ position ที่ under-collateralized

**Compound V2 Architecture:**
- cTokens = receipt tokens สำหรับ suppliers
- Exchange Rate = ราคาเปลี่ยนระหว่าง cToken กับ underlying
- Collateral Factor = อัตราส่วน max borrow ต่อ collateral

---

## 2. Interest Rate Model

```vyper
# @version 0.4.0
# contracts/InterestRateModel.vy
# Jump Rate Interest Rate Model

# Events
event NewInterestRateModel:
    baseRatePerYear: uint256
    multiplierPerYear: uint256
    jumpMultiplierPerYear: uint256
    kink: uint256

# Constants
BLOCKS_PER_YEAR: constant(uint256) = 2102400  # ~15 sec blocks
SCALE: constant(uint256) = 10**18

# Parameters
baseRatePerBlock: public(uint256)
multiplierPerBlock: public(uint256)
jumpMultiplierPerBlock: public(uint256)
kink: public(uint256)  # Utilization rate ที่ jump rate เริ่ม (scaled by 1e18)

owner: public(address)

@deploy
def __init__(
    baseRatePerYear: uint256,
    multiplierPerYear: uint256,
    jumpMultiplierPerYear: uint256,
    _kink: uint256
):
    """
    Parameters:
        baseRatePerYear: อัตราดอกเบี้ยพื้นฐานต่อปี (scaled 1e18) เช่น 2% = 0.02e18
        multiplierPerYear: rate เพิ่มต่อ % utilization
        jumpMultiplierPerYear: rate เพิ่มหลัง kink
        _kink: utilization % ที่ jump (0.8e18 = 80%)
    """
    self.owner = msg.sender
    self.baseRatePerBlock = baseRatePerYear / BLOCKS_PER_YEAR
    self.multiplierPerBlock = multiplierPerYear / BLOCKS_PER_YEAR
    self.jumpMultiplierPerBlock = jumpMultiplierPerYear / BLOCKS_PER_YEAR
    self.kink = _kink
    
    log NewInterestRateModel(baseRatePerYear, multiplierPerYear, jumpMultiplierPerYear, _kink)

@external
@view
def utilizationRate(cash: uint256, borrows: uint256, reserves: uint256) -> uint256:
    """
    คำนวณ utilization rate
    
    Utilization = borrows / (cash + borrows - reserves)
    
    Returns:
        utilization rate (scaled 1e18, 0 = 0%, 1e18 = 100%)
    """
    if borrows == 0:
        return 0
    
    total: uint256 = cash + borrows
    if total <= reserves:
        return 0
    
    return borrows * SCALE / (total - reserves)

@external
@view
def getBorrowRate(cash: uint256, borrows: uint256, reserves: uint256) -> uint256:
    """
    คำนวณ borrow rate ต่อ block
    
    ถ้า utilization <= kink:
        rate = base + multiplier * utilization
    ถ้า utilization > kink:
        rate = base + multiplier * kink + jumpMultiplier * (utilization - kink)
    
    Returns:
        borrow rate ต่อ block (scaled 1e18)
    """
    util: uint256 = self.utilizationRate(cash, borrows, reserves)
    
    if util <= self.kink:
        return self.baseRatePerBlock + self.multiplierPerBlock * util / SCALE
    else:
        normalRate: uint256 = self.baseRatePerBlock + self.multiplierPerBlock * self.kink / SCALE
        excessUtil: uint256 = util - self.kink
        return normalRate + self.jumpMultiplierPerBlock * excessUtil / SCALE

@external
@view
def getSupplyRate(
    cash: uint256,
    borrows: uint256,
    reserves: uint256,
    reserveFactor: uint256
) -> uint256:
    """
    คำนวณ supply rate ต่อ block
    
    supply rate = borrow rate * utilization * (1 - reserveFactor)
    
    Returns:
        supply rate ต่อ block (scaled 1e18)
    """
    oneMinusReserveFactor: uint256 = SCALE - reserveFactor
    borrowRate: uint256 = self.getBorrowRate(cash, borrows, reserves)
    rateToPool: uint256 = borrowRate * oneMinusReserveFactor / SCALE
    util: uint256 = self.utilizationRate(cash, borrows, reserves)
    
    return rateToPool * util / SCALE
```

---

## 3. cToken (Lending Market)

```vyper
# @version 0.4.0
# contracts/CToken.vy
# Compound-style cToken Market

from vyper.interfaces import ERC20

interface IInterestRateModel:
    def getBorrowRate(cash: uint256, borrows: uint256, reserves: uint256) -> uint256: view
    def getSupplyRate(cash: uint256, borrows: uint256, reserves: uint256, reserveFactor: uint256) -> uint256: view

interface IPriceOracle:
    def getUnderlyingPrice(cToken: address) -> uint256: view

interface IComptroller:
    def enterMarkets(cTokens: DynArray[address, 20]) -> DynArray[uint256, 20]: nonpayable
    def getAccountLiquidity(account: address) -> (uint256, uint256, uint256): view
    def borrowAllowed(cToken: address, borrower: address, borrowAmount: uint256) -> uint256: view
    def redeemAllowed(cToken: address, redeemer: address, redeemTokens: uint256) -> uint256: view
    def repayBorrowAllowed(cToken: address, payer: address, borrower: address, repayAmount: uint256) -> uint256: view
    def liquidateBorrowAllowed(cTokenBorrowed: address, cTokenCollateral: address, liquidator: address, borrower: address, repayAmount: uint256) -> uint256: view
    def liquidateCalculateSeizeTokens(cTokenBorrowed: address, cTokenCollateral: address, repayAmount: uint256) -> (uint256, uint256): view

# ERC20 Events
event Transfer:
    from_: indexed(address)
    to: indexed(address)
    amount: uint256

event Approval:
    owner: indexed(address)
    spender: indexed(address)
    amount: uint256

# Market Events
event Mint:
    minter: indexed(address)
    mintAmount: uint256
    mintTokens: uint256

event Redeem:
    redeemer: indexed(address)
    redeemAmount: uint256
    redeemTokens: uint256

event Borrow:
    borrower: indexed(address)
    borrowAmount: uint256
    accountBorrows: uint256
    totalBorrows: uint256

event RepayBorrow:
    payer: indexed(address)
    borrower: indexed(address)
    repayAmount: uint256
    accountBorrows: uint256
    totalBorrows: uint256

event LiquidateBorrow:
    liquidator: indexed(address)
    borrower: indexed(address)
    repayAmount: uint256
    cTokenCollateral: indexed(address)
    seizeTokens: uint256

event AccrueInterest:
    cashPrior: uint256
    interestAccumulated: uint256
    borrowIndex: uint256
    totalBorrows: uint256

event NewReserveFactor:
    oldReserveFactor: uint256
    newReserveFactor: uint256

# Struct สำหรับ borrow snapshot
struct BorrowSnapshot:
    principal: uint256    # จำนวนที่ยืมตอนยืม
    interestIndex: uint256  # borrow index ตอนนั้น

# ERC20 State
name: public(String[64])
symbol: public(String[32])
decimals: public(uint8)
totalSupply: public(uint256)
balanceOf: public(HashMap[address, uint256])
allowance: public(HashMap[address, HashMap[address, uint256]])

# Market State
underlying: public(address)
comptroller: public(address)
interestRateModel: public(address)
initialExchangeRateMantissa: public(uint256)
reserveFactorMantissa: public(uint256)  # สัดส่วนดอกเบี้ยที่เก็บเป็น reserve (0.1e18 = 10%)
admin: public(address)
pendingAdmin: public(address)

# Accrual State
accrualBlockNumber: public(uint256)
borrowIndex: public(uint256)    # Cumulative borrow index เริ่มต้น 1e18
totalBorrows: public(uint256)
totalReserves: public(uint256)

accountBorrows: HashMap[address, BorrowSnapshot]

SCALE: constant(uint256) = 10**18
MAX_RESERVE_FACTOR: constant(uint256) = 10**18  # 100%

@deploy
def __init__(
    _underlying: address,
    _comptroller: address,
    _interestRateModel: address,
    _initialExchangeRate: uint256,
    _name: String[64],
    _symbol: String[32],
    _decimals: uint8
):
    self.admin = msg.sender
    self.underlying = _underlying
    self.comptroller = _comptroller
    self.interestRateModel = _interestRateModel
    self.initialExchangeRateMantissa = _initialExchangeRate
    self.name = _name
    self.symbol = _symbol
    self.decimals = _decimals
    
    # เริ่มต้น borrow index ที่ 1
    self.borrowIndex = SCALE
    self.accrualBlockNumber = block.number

# ===== Interest Accrual =====

@internal
def _accrueInterest():
    """
    อัพเดทดอกเบี้ยตั้งแต่ block ล่าสุด
    เรียกก่อนทุก operation
    """
    currentBlock: uint256 = block.number
    accrualBlock: uint256 = self.accrualBlockNumber
    
    if accrualBlock == currentBlock:
        return
    
    cashPrior: uint256 = ERC20(self.underlying).balanceOf(self)
    borrowsPrior: uint256 = self.totalBorrows
    reservesPrior: uint256 = self.totalReserves
    borrowIndexPrior: uint256 = self.borrowIndex
    
    # คำนวณ borrow rate
    borrowRate: uint256 = IInterestRateModel(self.interestRateModel).getBorrowRate(
        cashPrior, borrowsPrior, reservesPrior
    )
    
    blockDelta: uint256 = currentBlock - accrualBlock
    
    # interestFactor = borrowRate * blockDelta
    interestFactor: uint256 = borrowRate * blockDelta
    
    # interestAccumulated = totalBorrows * interestFactor
    interestAccumulated: uint256 = interestFactor * borrowsPrior / SCALE
    
    # totalBorrows += interestAccumulated
    totalBorrowsNew: uint256 = borrowsPrior + interestAccumulated
    
    # totalReserves += interestAccumulated * reserveFactor
    totalReservesNew: uint256 = reservesPrior + interestAccumulated * self.reserveFactorMantissa / SCALE
    
    # borrowIndex *= (1 + interestFactor)
    borrowIndexNew: uint256 = borrowIndexPrior + borrowIndexPrior * interestFactor / SCALE
    
    self.accrualBlockNumber = currentBlock
    self.borrowIndex = borrowIndexNew
    self.totalBorrows = totalBorrowsNew
    self.totalReserves = totalReservesNew
    
    log AccrueInterest(cashPrior, interestAccumulated, borrowIndexNew, totalBorrowsNew)

# ===== Exchange Rate =====

@external
@view
def exchangeRateStored() -> uint256:
    """
    Exchange rate ล่าสุด (ไม่ accrueInterest ก่อน)
    
    exchangeRate = (cash + totalBorrows - totalReserves) / totalSupply
    """
    return self._exchangeRate()

@internal
@view
def _exchangeRate() -> uint256:
    _totalSupply: uint256 = self.totalSupply
    
    if _totalSupply == 0:
        return self.initialExchangeRateMantissa
    
    totalCash: uint256 = ERC20(self.underlying).balanceOf(self)
    cashPlusBorrowsMinusReserves: uint256 = totalCash + self.totalBorrows - self.totalReserves
    
    return cashPlusBorrowsMinusReserves * SCALE / _totalSupply

@external
def exchangeRateCurrent() -> uint256:
    """Exchange rate หลัง accrue interest"""
    self._accrueInterest()
    return self._exchangeRate()

# ===== Supply (Mint cTokens) =====

@external
def mint(mintAmount: uint256) -> uint256:
    """
    ฝาก underlying token เพื่อรับ cTokens
    
    Parameters:
        mintAmount: จำนวน underlying ที่ต้องการฝาก
    
    Returns:
        mintTokens: จำนวน cTokens ที่ได้รับ
    """
    self._accrueInterest()
    
    # โอน underlying เข้า
    assert ERC20(self.underlying).transferFrom(msg.sender, self, mintAmount), "Transfer failed"
    
    # คำนวณ cTokens
    exchangeRate: uint256 = self._exchangeRate()
    mintTokens: uint256 = mintAmount * SCALE / exchangeRate
    
    # Mint cTokens
    self.totalSupply += mintTokens
    self.balanceOf[msg.sender] += mintTokens
    
    log Transfer(empty(address), msg.sender, mintTokens)
    log Mint(msg.sender, mintAmount, mintTokens)
    
    return mintTokens

# ===== Redeem (Withdraw) =====

@external
def redeem(redeemTokens: uint256) -> uint256:
    """
    แลก cTokens เพื่อรับ underlying กลับ
    
    Parameters:
        redeemTokens: จำนวน cTokens ที่ต้องการแลก
    
    Returns:
        redeemAmount: จำนวน underlying ที่ได้คืน
    """
    self._accrueInterest()
    
    assert self.balanceOf[msg.sender] >= redeemTokens, "Insufficient balance"
    
    # ตรวจสอบว่าสามารถ redeem ได้
    allowed: uint256 = IComptroller(self.comptroller).redeemAllowed(
        self, msg.sender, redeemTokens
    )
    assert allowed == 0, "Redeem not allowed"
    
    # คำนวณ underlying amount
    exchangeRate: uint256 = self._exchangeRate()
    redeemAmount: uint256 = redeemTokens * exchangeRate / SCALE
    
    assert ERC20(self.underlying).balanceOf(self) >= redeemAmount, "Insufficient cash"
    
    # Burn cTokens
    self.balanceOf[msg.sender] -= redeemTokens
    self.totalSupply -= redeemTokens
    
    # โอน underlying ออก
    assert ERC20(self.underlying).transfer(msg.sender, redeemAmount), "Transfer failed"
    
    log Transfer(msg.sender, empty(address), redeemTokens)
    log Redeem(msg.sender, redeemAmount, redeemTokens)
    
    return redeemAmount

@external
def redeemUnderlying(redeemAmount: uint256) -> uint256:
    """Redeem โดยระบุ underlying amount แทน cToken amount"""
    self._accrueInterest()
    
    exchangeRate: uint256 = self._exchangeRate()
    redeemTokens: uint256 = redeemAmount * SCALE / exchangeRate
    
    assert self.balanceOf[msg.sender] >= redeemTokens, "Insufficient balance"
    
    allowed: uint256 = IComptroller(self.comptroller).redeemAllowed(
        self, msg.sender, redeemTokens
    )
    assert allowed == 0, "Redeem not allowed"
    
    assert ERC20(self.underlying).balanceOf(self) >= redeemAmount, "Insufficient cash"
    
    self.balanceOf[msg.sender] -= redeemTokens
    self.totalSupply -= redeemTokens
    
    assert ERC20(self.underlying).transfer(msg.sender, redeemAmount), "Transfer failed"
    
    log Transfer(msg.sender, empty(address), redeemTokens)
    log Redeem(msg.sender, redeemAmount, redeemTokens)
    
    return redeemTokens

# ===== Borrow =====

@external
def borrow(borrowAmount: uint256) -> uint256:
    """
    ยืม underlying tokens
    ต้องมี sufficient collateral
    
    Parameters:
        borrowAmount: จำนวนที่ต้องการยืม
    """
    self._accrueInterest()
    
    # ตรวจสอบจาก comptroller
    allowed: uint256 = IComptroller(self.comptroller).borrowAllowed(
        self, msg.sender, borrowAmount
    )
    assert allowed == 0, "Borrow not allowed"
    
    assert ERC20(self.underlying).balanceOf(self) >= borrowAmount, "Insufficient cash"
    
    # คำนวณ borrow balance ปัจจุบัน
    snapshot: BorrowSnapshot = self.accountBorrows[msg.sender]
    accountBorrowsPrior: uint256 = 0
    if snapshot.interestIndex != 0:
        accountBorrowsPrior = snapshot.principal * self.borrowIndex / snapshot.interestIndex
    
    # อัพเดท borrow state
    self.accountBorrows[msg.sender] = BorrowSnapshot({
        principal: accountBorrowsPrior + borrowAmount,
        interestIndex: self.borrowIndex
    })
    
    self.totalBorrows += borrowAmount
    
    # โอน tokens ออก
    assert ERC20(self.underlying).transfer(msg.sender, borrowAmount), "Transfer failed"
    
    log Borrow(
        msg.sender,
        borrowAmount,
        accountBorrowsPrior + borrowAmount,
        self.totalBorrows
    )
    
    return 0

# ===== Repay =====

@external
def repayBorrow(repayAmount: uint256) -> uint256:
    """
    คืน borrowed tokens
    
    Parameters:
        repayAmount: จำนวนที่ต้องการคืน (max_value(uint256) = คืนทั้งหมด)
    """
    self._accrueInterest()
    
    # คำนวณ borrow balance พร้อม interest
    snapshot: BorrowSnapshot = self.accountBorrows[msg.sender]
    accountBorrows: uint256 = 0
    if snapshot.interestIndex != 0:
        accountBorrows = snapshot.principal * self.borrowIndex / snapshot.interestIndex
    
    # คำนวณ actual repay amount
    actualRepayAmount: uint256 = repayAmount
    if repayAmount == max_value(uint256):
        actualRepayAmount = accountBorrows
    
    assert actualRepayAmount <= accountBorrows, "Repay too much"
    
    # โอน tokens เข้า
    assert ERC20(self.underlying).transferFrom(msg.sender, self, actualRepayAmount), "Transfer failed"
    
    # อัพเดท borrow state
    newAccountBorrows: uint256 = accountBorrows - actualRepayAmount
    if newAccountBorrows == 0:
        self.accountBorrows[msg.sender] = BorrowSnapshot({principal: 0, interestIndex: 0})
    else:
        self.accountBorrows[msg.sender] = BorrowSnapshot({
            principal: newAccountBorrows,
            interestIndex: self.borrowIndex
        })
    
    self.totalBorrows -= actualRepayAmount
    
    log RepayBorrow(msg.sender, msg.sender, actualRepayAmount, newAccountBorrows, self.totalBorrows)
    
    return 0

@external
def repayBorrowBehalf(borrower: address, repayAmount: uint256) -> uint256:
    """คืนแทนคนอื่น (สำหรับ liquidator)"""
    self._accrueInterest()
    
    snapshot: BorrowSnapshot = self.accountBorrows[borrower]
    accountBorrows: uint256 = 0
    if snapshot.interestIndex != 0:
        accountBorrows = snapshot.principal * self.borrowIndex / snapshot.interestIndex
    
    actualRepayAmount: uint256 = repayAmount
    if repayAmount == max_value(uint256):
        actualRepayAmount = accountBorrows
    
    assert actualRepayAmount <= accountBorrows, "Repay too much"
    
    assert ERC20(self.underlying).transferFrom(msg.sender, self, actualRepayAmount), "Transfer failed"
    
    newAccountBorrows: uint256 = accountBorrows - actualRepayAmount
    if newAccountBorrows == 0:
        self.accountBorrows[borrower] = BorrowSnapshot({principal: 0, interestIndex: 0})
    else:
        self.accountBorrows[borrower] = BorrowSnapshot({
            principal: newAccountBorrows,
            interestIndex: self.borrowIndex
        })
    
    self.totalBorrows -= actualRepayAmount
    
    log RepayBorrow(msg.sender, borrower, actualRepayAmount, newAccountBorrows, self.totalBorrows)
    
    return 0

# ===== Liquidation =====

@external
def liquidateBorrow(
    borrower: address,
    repayAmount: uint256,
    cTokenCollateral: address
) -> uint256:
    """
    Liquidate position ที่มี health factor ต่ำกว่า 1
    
    Parameters:
        borrower: address ของคนที่จะ liquidate
        repayAmount: จำนวน debt ที่ liquidator จะช่วยจ่าย
        cTokenCollateral: cToken ของ collateral ที่จะรับเป็นการตอบแทน
    """
    self._accrueInterest()
    
    # ตรวจสอบว่า liquidation อนุญาต
    allowed: uint256 = IComptroller(self.comptroller).liquidateBorrowAllowed(
        self, cTokenCollateral, msg.sender, borrower, repayAmount
    )
    assert allowed == 0, "Liquidation not allowed"
    
    # ตรวจสอบว่า borrower มีหนี้จริง
    snapshot: BorrowSnapshot = self.accountBorrows[borrower]
    assert snapshot.principal > 0, "No borrow"
    
    borrowBalance: uint256 = snapshot.principal * self.borrowIndex / snapshot.interestIndex
    
    # ไม่สามารถ liquidate มากกว่า close factor (เช่น 50% ของ borrow)
    assert repayAmount <= borrowBalance, "Repay too much"
    
    # คำนวณ collateral tokens ที่ liquidator จะได้
    err: uint256 = 0
    seizeTokens: uint256 = 0
    err, seizeTokens = IComptroller(self.comptroller).liquidateCalculateSeizeTokens(
        self, cTokenCollateral, repayAmount
    )
    assert err == 0, "Seize calculation failed"
    
    # Repay borrow
    assert ERC20(self.underlying).transferFrom(msg.sender, self, repayAmount), "Transfer failed"
    
    newBorrowBalance: uint256 = borrowBalance - repayAmount
    if newBorrowBalance == 0:
        self.accountBorrows[borrower] = BorrowSnapshot({principal: 0, interestIndex: 0})
    else:
        self.accountBorrows[borrower] = BorrowSnapshot({
            principal: newBorrowBalance,
            interestIndex: self.borrowIndex
        })
    
    self.totalBorrows -= repayAmount
    
    # โอน collateral ไปให้ liquidator
    # (seize จาก cTokenCollateral contract)
    # CToken(cTokenCollateral).seize(msg.sender, borrower, seizeTokens)
    
    log LiquidateBorrow(msg.sender, borrower, repayAmount, cTokenCollateral, seizeTokens)
    
    return 0

@external
def seize(liquidator: address, borrower: address, seizeTokens: uint256) -> uint256:
    """
    โอน cTokens จาก borrower ไปให้ liquidator
    เรียกโดย cToken อื่นเมื่อมีการ liquidate
    """
    # ตรวจสอบว่าเรียกโดย authorized cToken เท่านั้น
    # (ใน production ตรวจสอบผ่าน comptroller)
    
    # Protocol takes liquidation incentive bonus (8% เป็น reserve)
    protocolSeizeShare: uint256 = seizeTokens * 28 / 1000  # 2.8%
    liquidatorSeizeTokens: uint256 = seizeTokens - protocolSeizeShare
    
    self.totalReserves += protocolSeizeShare * self._exchangeRate() / SCALE
    
    self.balanceOf[borrower] -= seizeTokens
    self.balanceOf[liquidator] += liquidatorSeizeTokens
    
    log Transfer(borrower, liquidator, liquidatorSeizeTokens)
    log Transfer(borrower, empty(address), protocolSeizeShare)
    
    return 0

# ===== View Functions =====

@external
@view
def borrowBalanceStored(account: address) -> uint256:
    """
    borrow balance ณ ปัจจุบัน (ไม่ accrue)
    """
    snapshot: BorrowSnapshot = self.accountBorrows[account]
    if snapshot.principal == 0:
        return 0
    
    return snapshot.principal * self.borrowIndex / snapshot.interestIndex

@external
@view
def balanceOfUnderlying(account: address) -> uint256:
    """จำนวน underlying ที่ account มีใน cToken"""
    return self.balanceOf[account] * self._exchangeRate() / SCALE

@external
@view
def getCash() -> uint256:
    """Cash ที่มีใน market"""
    return ERC20(self.underlying).balanceOf(self)

# ===== Admin Functions =====

@external
def setReserveFactor(newReserveFactor: uint256):
    assert msg.sender == self.admin, "Not admin"
    assert newReserveFactor <= MAX_RESERVE_FACTOR, "Too high"
    
    oldReserveFactor: uint256 = self.reserveFactorMantissa
    self.reserveFactorMantissa = newReserveFactor
    
    log NewReserveFactor(oldReserveFactor, newReserveFactor)

# ===== ERC20 Functions =====

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
    assert self.balanceOf[from_] >= amount, "Insufficient balance"
    self.balanceOf[from_] -= amount
    self.balanceOf[to] += amount
    log Transfer(from_, to, amount)
    return True

@external
def approve(spender: address, amount: uint256) -> bool:
    self.allowance[msg.sender][spender] = amount
    log Approval(msg.sender, spender, amount)
    return True
```

---

## 4. Comptroller (Risk Management)

```vyper
# @version 0.4.0
# contracts/Comptroller.vy
# Risk Management - จัดการ collateral factors และ liquidation

interface ICToken:
    def underlying() -> address: view
    def borrowBalanceStored(account: address) -> uint256: view
    def balanceOf(account: address) -> uint256: view
    def exchangeRateStored() -> uint256: view

interface IPriceOracle:
    def getUnderlyingPrice(cToken: address) -> uint256: view

# Events
event MarketListed:
    cToken: indexed(address)

event MarketEntered:
    cToken: indexed(address)
    account: indexed(address)

event MarketExited:
    cToken: indexed(address)
    account: indexed(address)

event NewCollateralFactor:
    cToken: indexed(address)
    oldFactor: uint256
    newFactor: uint256

event NewLiquidationIncentive:
    oldIncentive: uint256
    newIncentive: uint256

event NewCloseFactor:
    oldFactor: uint256
    newFactor: uint256

# Struct
struct Market:
    isListed: bool
    collateralFactorMantissa: uint256  # 0 ถึง 0.9e18 (90%)
    isComped: bool  # รับ COMP rewards

# State
admin: public(address)
oracle: public(address)

markets: HashMap[address, Market]
accountAssets: HashMap[address, DynArray[address, 20]]  # assets ที่ account เข้า market

allMarkets: DynArray[address, 100]

closeFactorMantissa: public(uint256)       # % ของ debt ที่ liquidate ได้ต่อครั้ง (0.5e18 = 50%)
liquidationIncentiveMantissa: public(uint256)  # bonus สำหรับ liquidator (1.08e18 = 8%)
maxAssets: public(uint256)                 # จำนวน asset สูงสุดที่เข้า market ได้

SCALE: constant(uint256) = 10**18

@deploy
def __init__():
    self.admin = msg.sender
    self.closeFactorMantissa = 5 * 10**17   # 50%
    self.liquidationIncentiveMantissa = 108 * 10**16  # 108% (8% bonus)
    self.maxAssets = 20

@external
def supportMarket(cToken: address):
    """เพิ่ม market ใหม่"""
    assert msg.sender == self.admin, "Not admin"
    assert not self.markets[cToken].isListed, "Already listed"
    
    self.markets[cToken] = Market({
        isListed: True,
        collateralFactorMantissa: 0,
        isComped: False
    })
    
    self.allMarkets.append(cToken)
    
    log MarketListed(cToken)

@external
def setCollateralFactor(cToken: address, newCollateralFactor: uint256):
    """ตั้ง collateral factor สำหรับ market"""
    assert msg.sender == self.admin, "Not admin"
    assert self.markets[cToken].isListed, "Not listed"
    assert newCollateralFactor <= 9 * 10**17, "Too high"  # Max 90%
    
    oldFactor: uint256 = self.markets[cToken].collateralFactorMantissa
    self.markets[cToken].collateralFactorMantissa = newCollateralFactor
    
    log NewCollateralFactor(cToken, oldFactor, newCollateralFactor)

@external
def enterMarkets(cTokens: DynArray[address, 20]) -> DynArray[uint256, 20]:
    """เข้า markets เพื่อใช้เป็น collateral"""
    results: DynArray[uint256, 20] = []
    
    for cToken: address in cTokens:
        if self.markets[cToken].isListed:
            # เพิ่มใน account assets ถ้ายังไม่มี
            alreadyIn: bool = False
            for asset: address in self.accountAssets[msg.sender]:
                if asset == cToken:
                    alreadyIn = True
                    break
            
            if not alreadyIn:
                assert len(self.accountAssets[msg.sender]) < self.maxAssets, "Too many assets"
                self.accountAssets[msg.sender].append(cToken)
                log MarketEntered(cToken, msg.sender)
            
            results.append(0)  # Success
        else:
            results.append(9)  # Market not listed error
    
    return results

@external
def exitMarket(cToken: address) -> uint256:
    """ออกจาก market (ลบ collateral)"""
    assert self.markets[cToken].isListed, "Not listed"
    
    # ตรวจสอบว่าไม่มี borrow ที่ค้างอยู่
    borrowBalance: uint256 = ICToken(cToken).borrowBalanceStored(msg.sender)
    assert borrowBalance == 0, "Outstanding borrow"
    
    # ตรวจสอบ liquidity หลังออก
    # (simplified - production needs full check)
    
    # ลบออกจาก accountAssets
    newAssets: DynArray[address, 20] = []
    for asset: address in self.accountAssets[msg.sender]:
        if asset != cToken:
            newAssets.append(asset)
    self.accountAssets[msg.sender] = newAssets
    
    log MarketExited(cToken, msg.sender)
    
    return 0

# ===== Liquidity Calculation =====

@external
@view
def getAccountLiquidity(account: address) -> (uint256, uint256, uint256):
    """
    คำนวณ liquidity ของ account
    
    Returns:
        error: 0 = success
        liquidity: จำนวน collateral เกินกว่า borrow (ค่า > 0 = healthy)
        shortfall: จำนวนที่ขาด (ค่า > 0 = undercollateralized)
    """
    sumCollateral: uint256 = 0  # มูลค่า collateral รวม (USD)
    sumBorrow: uint256 = 0      # มูลค่า borrow รวม (USD)
    
    for cToken: address in self.accountAssets[account]:
        market: Market = self.markets[cToken]
        
        # ราคา underlying
        oraclePrice: uint256 = IPriceOracle(self.oracle).getUnderlyingPrice(cToken)
        
        # Collateral value
        cTokenBalance: uint256 = ICToken(cToken).balanceOf(account)
        exchangeRate: uint256 = ICToken(cToken).exchangeRateStored()
        underlyingBalance: uint256 = cTokenBalance * exchangeRate / SCALE
        
        collateralValue: uint256 = underlyingBalance * oraclePrice / SCALE
        collateralValueAdjusted: uint256 = collateralValue * market.collateralFactorMantissa / SCALE
        sumCollateral += collateralValueAdjusted
        
        # Borrow value
        borrowBalance: uint256 = ICToken(cToken).borrowBalanceStored(account)
        borrowValue: uint256 = borrowBalance * oraclePrice / SCALE
        sumBorrow += borrowValue
    
    if sumCollateral > sumBorrow:
        return 0, sumCollateral - sumBorrow, 0
    else:
        return 0, 0, sumBorrow - sumCollateral

@external
@view
def getHealthFactor(account: address) -> uint256:
    """
    คำนวณ Health Factor
    
    Health Factor = sum(collateral * collateral_factor) / sum(borrows)
    
    > 1e18 = healthy
    < 1e18 = can be liquidated
    = 0    = no borrows
    """
    err: uint256 = 0
    liquidity: uint256 = 0
    shortfall: uint256 = 0
    err, liquidity, shortfall = self.getAccountLiquidity(account)
    
    # คำนวณ health factor (simplified)
    sumCollateral: uint256 = 0
    sumBorrow: uint256 = 0
    
    for cToken: address in self.accountAssets[account]:
        market: Market = self.markets[cToken]
        oraclePrice: uint256 = IPriceOracle(self.oracle).getUnderlyingPrice(cToken)
        
        cTokenBalance: uint256 = ICToken(cToken).balanceOf(account)
        exchangeRate: uint256 = ICToken(cToken).exchangeRateStored()
        underlyingBalance: uint256 = cTokenBalance * exchangeRate / SCALE
        collateralValue: uint256 = underlyingBalance * oraclePrice / SCALE
        sumCollateral += collateralValue * market.collateralFactorMantissa / SCALE
        
        borrowBalance: uint256 = ICToken(cToken).borrowBalanceStored(account)
        sumBorrow += borrowBalance * oraclePrice / SCALE
    
    if sumBorrow == 0:
        return 0  # No borrows
    
    return sumCollateral * SCALE / sumBorrow

# ===== Borrow/Redeem Checks =====

@external
@view
def borrowAllowed(cToken: address, borrower: address, borrowAmount: uint256) -> uint256:
    """ตรวจสอบว่าสามารถยืมได้"""
    assert self.markets[cToken].isListed, "Market not listed"
    
    # ตรวจสอบว่า borrower เข้า market แล้ว
    inMarket: bool = False
    for asset: address in self.accountAssets[borrower]:
        if asset == cToken:
            inMarket = True
            break
    
    if not inMarket:
        return 3  # Not in market
    
    # ตรวจสอบ liquidity หลัง borrow
    # (simplified)
    return 0

@external
@view
def redeemAllowed(cToken: address, redeemer: address, redeemTokens: uint256) -> uint256:
    """ตรวจสอบว่าสามารถถอนได้"""
    if not self.markets[cToken].isListed:
        return 9
    return 0

@external
@view
def repayBorrowAllowed(
    cToken: address,
    payer: address,
    borrower: address,
    repayAmount: uint256
) -> uint256:
    """ตรวจสอบว่าสามารถ repay ได้"""
    if not self.markets[cToken].isListed:
        return 9
    return 0

@external
@view
def liquidateBorrowAllowed(
    cTokenBorrowed: address,
    cTokenCollateral: address,
    liquidator: address,
    borrower: address,
    repayAmount: uint256
) -> uint256:
    """ตรวจสอบว่าสามารถ liquidate ได้"""
    if not self.markets[cTokenBorrowed].isListed:
        return 9
    if not self.markets[cTokenCollateral].isListed:
        return 9
    
    # ตรวจสอบว่า borrower มี shortfall
    err: uint256 = 0
    liquidity: uint256 = 0
    shortfall: uint256 = 0
    err, liquidity, shortfall = self.getAccountLiquidity(borrower)
    
    if shortfall == 0:
        return 4  # No shortfall - cannot liquidate
    
    # ตรวจสอบ close factor
    borrowBalance: uint256 = ICToken(cTokenBorrowed).borrowBalanceStored(borrower)
    maxClose: uint256 = self.closeFactorMantissa * borrowBalance / SCALE
    
    if repayAmount > maxClose:
        return 17  # Too much to liquidate
    
    return 0

@external
@view
def liquidateCalculateSeizeTokens(
    cTokenBorrowed: address,
    cTokenCollateral: address,
    repayAmount: uint256
) -> (uint256, uint256):
    """
    คำนวณจำนวน collateral ที่ liquidator จะได้
    
    seizeTokens = repayAmount * price(borrowed) * liquidationIncentive / price(collateral) / exchangeRate
    """
    priceBorrowed: uint256 = IPriceOracle(self.oracle).getUnderlyingPrice(cTokenBorrowed)
    priceCollateral: uint256 = IPriceOracle(self.oracle).getUnderlyingPrice(cTokenCollateral)
    
    exchangeRate: uint256 = ICToken(cTokenCollateral).exchangeRateStored()
    
    # seizeTokens = repayAmount * priceBorrowed * liquidationIncentive / (priceCollateral * exchangeRate)
    numerator: uint256 = self.liquidationIncentiveMantissa * priceBorrowed
    denominator: uint256 = priceCollateral * exchangeRate / SCALE
    ratio: uint256 = numerator / denominator
    
    seizeTokens: uint256 = ratio * repayAmount / SCALE
    
    return 0, seizeTokens

@external
def setOracle(newOracle: address):
    assert msg.sender == self.admin, "Not admin"
    self.oracle = newOracle
```

---

## 5. Price Oracle

```vyper
# @version 0.4.0
# contracts/SimplePriceOracle.vy
# Simple Price Oracle สำหรับ Lending Protocol

# Events
event PricePosted:
    asset: indexed(address)
    previousPrice: uint256
    newPrice: uint256

prices: HashMap[address, uint256]
admin: public(address)
# Mapping from cToken to underlying
underlyingTokens: HashMap[address, address]

@deploy
def __init__():
    self.admin = msg.sender

@external
def setUnderlyingPrice(cToken: address, underlyingPriceMantissa: uint256):
    """ตั้งราคา underlying token (สำหรับ testing)"""
    assert msg.sender == self.admin, "Not admin"
    
    oldPrice: uint256 = self.prices[cToken]
    self.prices[cToken] = underlyingPriceMantissa
    
    log PricePosted(cToken, oldPrice, underlyingPriceMantissa)

@external
@view
def getUnderlyingPrice(cToken: address) -> uint256:
    """
    ดึงราคา underlying ของ cToken (USD, scaled 1e18)
    
    Production: ใช้ Chainlink หรือ TWAP oracle
    """
    price: uint256 = self.prices[cToken]
    assert price != 0, "No price"
    return price

@external
def setPrice(asset: address, price: uint256):
    assert msg.sender == self.admin, "Not admin"
    self.prices[asset] = price
```

---

## 6. Tests

```python
# tests/test_lending.py
import pytest
from brownie import (
    CToken,
    Comptroller,
    InterestRateModel,
    SimplePriceOracle,
    MockERC20,
    accounts,
    chain
)

SCALE = 10**18

@pytest.fixture
def setup():
    owner = accounts[0]
    alice = accounts[1]
    bob = accounts[2]
    
    # Deploy tokens
    usdc = MockERC20.deploy("USDC", "USDC", 6, {"from": owner})
    weth = MockERC20.deploy("Wrapped ETH", "WETH", 18, {"from": owner})
    
    # Deploy oracle
    oracle = SimplePriceOracle.deploy({"from": owner})
    
    # Set prices (USDC = $1, ETH = $2000)
    oracle.setUnderlyingPrice("0x0000...", 1 * SCALE, {"from": owner})  # placeholder
    
    # Deploy interest rate model
    # 2% base, 20% multiplier, 100% jump, 80% kink
    irm = InterestRateModel.deploy(
        int(0.02 * SCALE),  # 2% base
        int(0.20 * SCALE),  # 20% multiplier
        int(1.0 * SCALE),   # 100% jump
        int(0.80 * SCALE),  # 80% kink
        {"from": owner}
    )
    
    # Deploy comptroller
    comptroller = Comptroller.deploy({"from": owner})
    comptroller.setOracle(oracle.address, {"from": owner})
    
    # Deploy cTokens
    cUSDC = CToken.deploy(
        usdc.address,
        comptroller.address,
        irm.address,
        int(0.02 * SCALE),  # initial exchange rate
        "Compound USDC",
        "cUSDC",
        8,
        {"from": owner}
    )
    
    cWETH = CToken.deploy(
        weth.address,
        comptroller.address,
        irm.address,
        int(0.02 * SCALE),
        "Compound WETH",
        "cWETH",
        8,
        {"from": owner}
    )
    
    # Setup markets
    comptroller.supportMarket(cUSDC.address, {"from": owner})
    comptroller.supportMarket(cWETH.address, {"from": owner})
    comptroller.setCollateralFactor(cWETH.address, int(0.75 * SCALE), {"from": owner})
    
    # Set prices
    oracle.setUnderlyingPrice(cUSDC.address, 1 * SCALE, {"from": owner})
    oracle.setUnderlyingPrice(cWETH.address, 2000 * SCALE, {"from": owner})
    
    # Mint tokens
    usdc.mint(alice, 10**9, {"from": owner})  # 1000 USDC
    weth.mint(alice, 10**21, {"from": owner})  # 1000 ETH
    usdc.mint(bob, 10**10, {"from": owner})   # 10000 USDC
    
    return owner, alice, bob, usdc, weth, cUSDC, cWETH, comptroller, oracle

def test_supply(setup):
    owner, alice, bob, usdc, weth, cUSDC, cWETH, comptroller, oracle = setup
    
    # Alice supplies USDC
    supply_amount = 10**8  # 100 USDC (6 decimals)
    usdc.approve(cUSDC.address, supply_amount, {"from": alice})
    
    cTokens = cUSDC.mint(supply_amount, {"from": alice})
    
    assert cUSDC.balanceOf(alice.address) > 0
    print(f"Alice received {cUSDC.balanceOf(alice.address)} cUSDC")

def test_borrow(setup):
    owner, alice, bob, usdc, weth, cUSDC, cWETH, comptroller, oracle = setup
    
    # Bob supplies USDC (for borrower to borrow from)
    usdc.approve(cUSDC.address, 10**9, {"from": bob})
    cUSDC.mint(10**9, {"from": bob})
    
    # Alice supplies ETH as collateral
    weth_supply = 10**18  # 1 ETH
    weth.approve(cWETH.address, weth_supply, {"from": alice})
    cWETH.mint(weth_supply, {"from": alice})
    
    # Alice enters ETH market
    comptroller.enterMarkets([cWETH.address], {"from": alice})
    
    # Alice borrows USDC (max 75% of $2000 = $1500)
    borrow_amount = 10**8  # 100 USDC
    
    cUSDC.borrow(borrow_amount, {"from": alice})
    
    assert usdc.balanceOf(alice.address) >= borrow_amount
    
    borrow_balance = cUSDC.borrowBalanceStored(alice.address)
    print(f"Alice's borrow balance: {borrow_balance}")

def test_health_factor(setup):
    owner, alice, bob, usdc, weth, cUSDC, cWETH, comptroller, oracle = setup
    
    # Setup: Bob supplies USDC, Alice supplies ETH and borrows
    usdc.approve(cUSDC.address, 10**9, {"from": bob})
    cUSDC.mint(10**9, {"from": bob})
    
    weth.approve(cWETH.address, 10**18, {"from": alice})
    cWETH.mint(10**18, {"from": alice})
    comptroller.enterMarkets([cWETH.address], {"from": alice})
    
    cUSDC.borrow(10**8, {"from": alice})  # 100 USDC
    
    # Check health factor
    health = comptroller.getHealthFactor(alice.address)
    print(f"Health factor: {health / SCALE:.2f}")
    assert health > SCALE  # Healthy position
    
    # Simulate price drop (ETH drops 80%)
    oracle.setUnderlyingPrice(cWETH.address, 400 * SCALE, {"from": owner})
    
    health_after = comptroller.getHealthFactor(alice.address)
    print(f"Health factor after price drop: {health_after / SCALE:.2f}")
    # May be < 1 now (liquidatable)

def test_interest_accrual(setup):
    owner, alice, bob, usdc, weth, cUSDC, cWETH, comptroller, oracle = setup
    
    # Supply
    usdc.approve(cUSDC.address, 10**9, {"from": bob})
    cUSDC.mint(10**9, {"from": bob})
    
    weth.approve(cWETH.address, 10**18, {"from": alice})
    cWETH.mint(10**18, {"from": alice})
    comptroller.enterMarkets([cWETH.address], {"from": alice})
    
    cUSDC.borrow(10**8, {"from": alice})
    
    initial_balance = cUSDC.borrowBalanceStored(alice.address)
    
    # Advance blocks
    chain.mine(1000)
    
    # Accrue interest
    cUSDC.exchangeRateCurrent({"from": owner})
    
    new_balance = cUSDC.borrowBalanceStored(alice.address)
    
    assert new_balance > initial_balance
    print(f"Interest accrued: {new_balance - initial_balance}")

def test_interest_rate_model(setup):
    owner, alice, bob, usdc, weth, cUSDC, cWETH, comptroller, oracle = setup
    
    irm = InterestRateModel.at(cUSDC.interestRateModel())
    
    # 0% utilization
    rate0 = irm.getBorrowRate(10**9, 0, 0)
    print(f"Borrow rate at 0% utilization: {rate0 / SCALE * 100:.4f}% per block")
    
    # 80% utilization (at kink)
    cash = 10**9
    borrows = int(4 * 10**9)  # 80% utilization
    rate80 = irm.getBorrowRate(cash, borrows, 0)
    print(f"Borrow rate at 80% utilization: {rate80 / SCALE * 100:.4f}% per block")
    
    # 90% utilization (above kink)
    borrows90 = int(9 * 10**9)
    rate90 = irm.getBorrowRate(cash, borrows90, 0)
    print(f"Borrow rate at 90% utilization: {rate90 / SCALE * 100:.4f}% per block")
    
    # Rate jump dramatically above kink
    assert rate90 > rate80 * 2
```

---

## 7. สรุป

### Key Concepts:

**1. Exchange Rate**
- Exchange rate เพิ่มขึ้นเมื่อดอกเบี้ยสะสม
- cToken holders ได้ประโยชน์โดยอัตโนมัติ

**2. Interest Rate Jump Model**
- 0-80% utilization: rate เพิ่มเรื่อยๆ
- >80% utilization: rate เพิ่มแบบ exponential
- บังคับให้ borrowers คืนเงินเมื่อ pool ใกล้หมด

**3. Collateral Factor**
- ETH: 75% (ยืมได้ 75% ของมูลค่า ETH)
- USDC: 85% (stable = ยืมได้มากกว่า)
- ยิ่ง volatile ยิ่งมี collateral factor ต่ำ

**4. Health Factor Formula**
```
Health Factor = Σ(collateral_value × collateral_factor) / Σ(borrow_value)

> 1.0 = Safe
< 1.0 = Liquidatable
```

**5. Liquidation Bonus**
- Liquidator ได้ collateral มูลค่า 108% ของ debt ที่จ่าย
- ส่วนเกิน 2.8% ไปเป็น protocol reserve
- สร้าง incentive ให้ liquidators ทำงาน
