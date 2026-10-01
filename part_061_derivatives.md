# Part 061: DeFi Derivatives - Options, Perpetuals, and Funding Rates

## สารบัญ (Table of Contents)
1. บทนำ DeFi Derivatives
2. Options Contract (European)
3. Perpetuals Basics
4. Funding Rate Mechanism
5. Settlement Mechanisms
6. Tests

---

## 1. บทนำ DeFi Derivatives

**ประเภท Derivatives:**
- **Options**: สิทธิ์ (ไม่ใช่ข้อบังคับ) ในการซื้อ/ขาย asset ในราคาที่กำหนด
- **Futures**: ข้อผูกพันในการซื้อ/ขาย asset ในอนาคต
- **Perpetuals**: Futures ที่ไม่มีวันหมดอายุ พร้อม funding rate

**Pricing Models:**
- Options: Black-Scholes (simplified on-chain)
- Perpetuals: Mark price + funding rate

---

## 2. Options Contract

```vyper
# @version 0.4.0
# contracts/OptionsVault.vy
# European Options - ใช้สิทธิ์ได้เฉพาะวัน expiry

from vyper.interfaces import ERC20

interface IPriceOracle:
    def getPrice(asset: address) -> uint256: view

# Events
event OptionCreated:
    optionId: indexed(uint256)
    writer: indexed(address)
    holder: indexed(address)
    isCall: bool
    strikePrice: uint256
    expiry: uint256
    premium: uint256

event OptionExercised:
    optionId: indexed(uint256)
    holder: indexed(address)
    pnl: uint256

event OptionExpired:
    optionId: indexed(uint256)
    writer: indexed(address)
    collateralReturned: uint256

# Struct
struct Option:
    writer: address           # ผู้ขาย option (ต้องวาง collateral)
    holder: address           # ผู้ซื้อ option (จ่าย premium)
    underlying: address       # asset ที่เป็น underlying
    collateralToken: address  # token ที่ใช้เป็น collateral (usually USDC)
    strikePrice: uint256      # ราคาใช้สิทธิ์
    expiry: uint256           # timestamp วันหมดอายุ
    size: uint256             # จำนวน underlying ที่ครอบคลุม
    collateralAmount: uint256 # collateral ที่ writer วาง
    premium: uint256          # premium ที่ holder จ่าย
    isCall: bool              # true = call, false = put
    exercised: bool
    settled: bool

# State
options: HashMap[uint256, Option]
nextOptionId: uint256
oracle: public(address)
governance: public(address)

# Fee
protocolFee: uint256  # basis points (50 = 0.5%)
feeTo: public(address)

SCALE: constant(uint256) = 10**18

@deploy
def __init__(_oracle: address, _governance: address, _feeTo: address):
    self.oracle = _oracle
    self.governance = _governance
    self.feeTo = _feeTo
    self.protocolFee = 50  # 0.5%
    self.nextOptionId = 1

# ===== Write Option =====

@external
def writeCallOption(
    holder: address,
    underlying: address,
    collateralToken: address,
    strikePrice: uint256,
    expiry: uint256,
    size: uint256,
    premium: uint256
) -> uint256:
    """
    ขาย Call Option
    
    Writer วาง underlying เป็น collateral (fully collateralized)
    ถ้า holder ใช้สิทธิ์: holder จ่าย strike price และได้ underlying
    
    Parameters:
        holder: ผู้ซื้อ option
        underlying: asset ที่เป็น underlying
        collateralToken: USDC หรือ stablecoin
        strikePrice: ราคาใช้สิทธิ์ (USD, scaled 1e18)
        expiry: timestamp วันหมดอายุ
        size: จำนวน underlying
        premium: premium ที่ holder ต้องจ่าย (collateralToken)
    """
    assert expiry > block.timestamp, "Expiry in past"
    assert size > 0, "Zero size"
    
    optionId: uint256 = self.nextOptionId
    self.nextOptionId += 1
    
    # Writer วาง underlying เป็น collateral
    assert ERC20(underlying).transferFrom(msg.sender, self, size), "Collateral failed"
    
    # Collect premium จาก holder
    if premium > 0:
        feeAmount: uint256 = premium * self.protocolFee / 10000
        assert ERC20(collateralToken).transferFrom(holder, self, premium), "Premium failed"
        assert ERC20(collateralToken).transfer(msg.sender, premium - feeAmount), "Premium transfer failed"
        if feeAmount > 0:
            assert ERC20(collateralToken).transfer(self.feeTo, feeAmount), "Fee failed"
    
    self.options[optionId] = Option({
        writer: msg.sender,
        holder: holder,
        underlying: underlying,
        collateralToken: collateralToken,
        strikePrice: strikePrice,
        expiry: expiry,
        size: size,
        collateralAmount: size,
        premium: premium,
        isCall: True,
        exercised: False,
        settled: False
    })
    
    log OptionCreated(optionId, msg.sender, holder, True, strikePrice, expiry, premium)
    
    return optionId

@external
def writePutOption(
    holder: address,
    underlying: address,
    collateralToken: address,
    strikePrice: uint256,
    expiry: uint256,
    size: uint256,
    premium: uint256
) -> uint256:
    """
    ขาย Put Option
    
    Writer วาง strike price * size ใน USDC เป็น collateral
    ถ้า holder ใช้สิทธิ์: holder ส่ง underlying และได้ USDC back
    """
    assert expiry > block.timestamp, "Expiry in past"
    assert size > 0, "Zero size"
    
    # Collateral = strike price * size (ใน USDC)
    collateralNeeded: uint256 = strikePrice * size / SCALE
    
    optionId: uint256 = self.nextOptionId
    self.nextOptionId += 1
    
    assert ERC20(collateralToken).transferFrom(msg.sender, self, collateralNeeded), "Collateral failed"
    
    if premium > 0:
        feeAmount: uint256 = premium * self.protocolFee / 10000
        assert ERC20(collateralToken).transferFrom(holder, self, premium), "Premium failed"
        assert ERC20(collateralToken).transfer(msg.sender, premium - feeAmount), "Premium transfer"
        if feeAmount > 0:
            assert ERC20(collateralToken).transfer(self.feeTo, feeAmount), "Fee failed"
    
    self.options[optionId] = Option({
        writer: msg.sender,
        holder: holder,
        underlying: underlying,
        collateralToken: collateralToken,
        strikePrice: strikePrice,
        expiry: expiry,
        size: size,
        collateralAmount: collateralNeeded,
        premium: premium,
        isCall: False,
        exercised: False,
        settled: False
    })
    
    log OptionCreated(optionId, msg.sender, holder, False, strikePrice, expiry, premium)
    
    return optionId

# ===== Exercise Option =====

@external
def exerciseOption(optionId: uint256):
    """
    ใช้สิทธิ์ option (European: ทำได้เฉพาะวัน expiry)
    
    Call: holder จ่าย strikePrice * size USDC -> ได้ underlying
    Put: holder ส่ง underlying -> ได้ strikePrice * size USDC
    """
    option: Option = self.options[optionId]
    
    assert msg.sender == option.holder, "Not holder"
    assert not option.exercised, "Already exercised"
    assert not option.settled, "Already settled"
    
    # European option: ใช้สิทธิ์ได้เฉพาะในช่วง expiry window
    assert block.timestamp >= option.expiry, "Too early"
    assert block.timestamp <= option.expiry + 3600, "Too late (1hr window)"
    
    # ดูราคา underlying
    currentPrice: uint256 = IPriceOracle(self.oracle).getPrice(option.underlying)
    
    pnl: uint256 = 0
    
    if option.isCall:
        # Call option: profitable ถ้าราคา > strike
        assert currentPrice > option.strikePrice, "OTM: not profitable"
        
        # Holder จ่าย strike * size
        paymentAmount: uint256 = option.strikePrice * option.size / SCALE
        assert ERC20(option.collateralToken).transferFrom(msg.sender, self, paymentAmount), "Payment failed"
        assert ERC20(option.collateralToken).transfer(option.writer, paymentAmount), "Payment to writer"
        
        # Holder รับ underlying
        assert ERC20(option.underlying).transfer(option.holder, option.size), "Delivery failed"
        
        pnl = (currentPrice - option.strikePrice) * option.size / SCALE
    else:
        # Put option: profitable ถ้าราคา < strike
        assert currentPrice < option.strikePrice, "OTM: not profitable"
        
        # Holder ส่ง underlying
        assert ERC20(option.underlying).transferFrom(msg.sender, self, option.size), "Asset delivery failed"
        assert ERC20(option.underlying).transfer(option.writer, option.size), "Asset to writer"
        
        # Holder รับ strike * size USDC
        payoutAmount: uint256 = option.strikePrice * option.size / SCALE
        assert ERC20(option.collateralToken).transfer(option.holder, payoutAmount), "Payout failed"
        
        pnl = (option.strikePrice - currentPrice) * option.size / SCALE
    
    self.options[optionId].exercised = True
    self.options[optionId].settled = True
    
    log OptionExercised(optionId, option.holder, pnl)

@external
def expireOption(optionId: uint256):
    """
    Settle option ที่ไม่ถูกใช้สิทธิ์ (คืน collateral ให้ writer)
    """
    option: Option = self.options[optionId]
    
    assert not option.settled, "Already settled"
    assert block.timestamp > option.expiry + 3600, "Not expired yet"
    
    self.options[optionId].settled = True
    
    if option.isCall:
        assert ERC20(option.underlying).transfer(option.writer, option.collateralAmount), "Return failed"
    else:
        assert ERC20(option.collateralToken).transfer(option.writer, option.collateralAmount), "Return failed"
    
    log OptionExpired(optionId, option.writer, option.collateralAmount)

# ===== Black-Scholes Approximation =====

@external
@pure
def approximatePremium(
    spotPrice: uint256,
    strikePrice: uint256,
    timeToExpiry: uint256,  # seconds
    impliedVolatility: uint256,  # scaled 1e18, e.g. 0.5e18 = 50% IV
    isCall: bool
) -> uint256:
    """
    ประมาณ premium โดย simplified Black-Scholes
    ใช้สำหรับ reference pricing เท่านั้น
    
    Simplified formula (ไม่ใช่ exact BS):
    Call: max(S - K, 0) + IV * S * sqrt(T) * 0.4
    Put: max(K - S, 0) + IV * S * sqrt(T) * 0.4
    
    Parameters:
        timeToExpiry: time ถึง expiry ใน seconds
        impliedVolatility: IV เช่น 0.8e18 = 80% annualized
    """
    # Intrinsic value
    intrinsicValue: uint256 = 0
    if isCall and spotPrice > strikePrice:
        intrinsicValue = spotPrice - strikePrice
    elif not isCall and strikePrice > spotPrice:
        intrinsicValue = strikePrice - spotPrice
    
    # Time value approximation
    # timeRatio = sqrt(T/YEAR) ≈ sqrt(timeToExpiry/31536000)
    YEAR: uint256 = 365 * 24 * 3600
    timeRatio: uint256 = 0
    
    if timeToExpiry > 0:
        # Simplified sqrt using integer arithmetic
        t: uint256 = timeToExpiry * SCALE / YEAR
        # sqrt(t) approximation
        timeRatio = t  # Simplified
    
    timeValue: uint256 = impliedVolatility * spotPrice / SCALE * timeRatio / SCALE * 40 / 100
    
    return intrinsicValue + timeValue

@external
@view
def getOption(optionId: uint256) -> Option:
    return self.options[optionId]
```

---

## 3. Perpetuals Contract

```vyper
# @version 0.4.0
# contracts/PerpetualMarket.vy
# Perpetual Futures Market

from vyper.interfaces import ERC20

interface IPriceOracle:
    def getPrice(asset: address) -> uint256: view
    def getMarkPrice(market: address) -> uint256: view

# Events
event PositionOpened:
    trader: indexed(address)
    isLong: bool
    size: uint256
    entryPrice: uint256
    collateral: uint256
    leverage: uint256

event PositionClosed:
    trader: indexed(address)
    pnl: int256
    fundingFee: int256

event FundingRateUpdated:
    fundingRate: int256
    timestamp: uint256

event Liquidated:
    trader: indexed(address)
    liquidator: indexed(address)
    pnl: int256

# Structs
struct Position:
    size: uint256        # position size ใน underlying
    collateral: uint256  # USDC collateral
    entryPrice: uint256  # ราคาเปิด position
    isLong: bool         # true = long, false = short
    lastFundingIndex: uint256  # funding index ณ ตอนเปิด

# State
oracle: public(address)
collateralToken: public(address)  # USDC
underlying: public(address)

positions: HashMap[address, Position]

# Pool state
longOpenInterest: public(uint256)   # total long position size
shortOpenInterest: public(uint256)  # total short position size
insuranceFund: public(uint256)      # ชดเชยกรณี bankrupt

# Funding rate
fundingRate: public(int256)         # current hourly funding rate (scaled 1e18)
fundingIndex: public(uint256)       # cumulative funding
lastFundingTime: public(uint256)

# Parameters
maxLeverage: public(uint256)        # max leverage (20 = 20x)
liquidationThreshold: public(uint256)  # collateral ratio ที่ liquidate (5% = 500)
maintenanceMargin: public(uint256)  # minimum margin ratio (10% = 1000)

governance: public(address)

# Fees
openFeeBps: uint256     # fee เมื่อเปิด position (10 = 0.1%)
closeFeeBps: uint256    # fee เมื่อปิด position

SCALE: constant(uint256) = 10**18
FUNDING_PERIOD: constant(uint256) = 3600  # 1 hour
MAX_FUNDING_RATE: constant(int256) = 375 * 10**15  # 0.0375% per hour max = ~33% APY

@deploy
def __init__(
    _oracle: address,
    _collateralToken: address,
    _underlying: address,
    _governance: address
):
    self.oracle = _oracle
    self.collateralToken = _collateralToken
    self.underlying = _underlying
    self.governance = _governance
    self.maxLeverage = 20
    self.liquidationThreshold = 500  # 5%
    self.maintenanceMargin = 1000    # 10%
    self.openFeeBps = 10
    self.closeFeeBps = 10
    self.lastFundingTime = block.timestamp

# ===== Funding Rate =====

@internal
def _updateFundingRate():
    """
    อัพเดท funding rate ทุก hour
    
    Funding Rate = (Mark Price - Index Price) / Index Price * k
    
    ถ้า long > short: longs จ่าย funding ให้ shorts
    ถ้า short > long: shorts จ่าย funding ให้ longs
    """
    if block.timestamp < self.lastFundingTime + FUNDING_PERIOD:
        return
    
    elapsed: uint256 = block.timestamp - self.lastFundingTime
    
    markPrice: uint256 = IPriceOracle(self.oracle).getMarkPrice(self)
    indexPrice: uint256 = IPriceOracle(self.oracle).getPrice(self.underlying)
    
    if indexPrice == 0:
        return
    
    # funding rate = (mark - index) / index * premiumFraction
    priceDiff: int256 = convert(markPrice, int256) - convert(indexPrice, int256)
    newFundingRate: int256 = priceDiff * 3 / convert(indexPrice, int256)  # k = 3
    
    # Clamp to max
    if newFundingRate > MAX_FUNDING_RATE:
        newFundingRate = MAX_FUNDING_RATE
    elif newFundingRate < -MAX_FUNDING_RATE:
        newFundingRate = -MAX_FUNDING_RATE
    
    self.fundingRate = newFundingRate
    
    # อัพเดท cumulative funding index
    intervals: uint256 = elapsed / FUNDING_PERIOD
    self.fundingIndex += convert(newFundingRate * convert(intervals, int256), uint256)
    self.lastFundingTime = block.timestamp
    
    log FundingRateUpdated(newFundingRate, block.timestamp)

@external
def updateFundingRate():
    """Public function อัพเดท funding rate"""
    self._updateFundingRate()

# ===== Open Position =====

@external
def openPosition(
    isLong: bool,
    collateralAmount: uint256,
    leverage: uint256
) -> uint256:
    """
    เปิด perpetual position
    
    Parameters:
        isLong: true = long, false = short
        collateralAmount: USDC collateral
        leverage: 1-20x
    
    Returns:
        size: position size ใน underlying (USD value)
    """
    assert positions[msg.sender].size == 0, "Position exists"
    assert leverage >= 1 and leverage <= self.maxLeverage, "Invalid leverage"
    assert collateralAmount > 0, "Zero collateral"
    
    self._updateFundingRate()
    
    currentPrice: uint256 = IPriceOracle(self.oracle).getPrice(self.underlying)
    assert currentPrice > 0, "No price"
    
    # ขนาด position ใน USD
    positionSize: uint256 = collateralAmount * leverage
    
    # Collect collateral + fee
    feeAmount: uint256 = positionSize * self.openFeeBps / 10000
    
    assert ERC20(self.collateralToken).transferFrom(
        msg.sender, self, collateralAmount + feeAmount
    ), "Transfer failed"
    
    # เพิ่ม fee เข้า insurance fund
    self.insuranceFund += feeAmount
    
    # บันทึก position
    self.positions[msg.sender] = Position({
        size: positionSize,
        collateral: collateralAmount,
        entryPrice: currentPrice,
        isLong: isLong,
        lastFundingIndex: self.fundingIndex
    })
    
    if isLong:
        self.longOpenInterest += positionSize
    else:
        self.shortOpenInterest += positionSize
    
    log PositionOpened(msg.sender, isLong, positionSize, currentPrice, collateralAmount, leverage)
    
    return positionSize

# ===== Close Position =====

@external
def closePosition() -> int256:
    """
    ปิด position และรับ PnL
    
    Returns:
        pnl: กำไร/ขาดทุน (positive = กำไร)
    """
    pos: Position = self.positions[msg.sender]
    assert pos.size > 0, "No position"
    
    self._updateFundingRate()
    
    currentPrice: uint256 = IPriceOracle(self.oracle).getPrice(self.underlying)
    
    # คำนวณ PnL
    pnl: int256 = self._calculatePnL(pos, currentPrice)
    
    # คำนวณ funding fee
    fundingFee: int256 = self._calculateFundingFee(pos)
    
    netPnL: int256 = pnl - fundingFee
    
    # คำนวณ closing fee
    closeFee: uint256 = pos.size * self.closeFeeBps / 10000
    
    # คำนวณ payout
    payout: int256 = convert(pos.collateral, int256) + netPnL - convert(closeFee, int256)
    
    if isLong:
        self.longOpenInterest -= pos.size
    else:
        self.shortOpenInterest -= pos.size
    
    self.positions[msg.sender] = Position({size: 0, collateral: 0, entryPrice: 0, isLong: False, lastFundingIndex: 0})
    
    if payout > 0:
        assert ERC20(self.collateralToken).transfer(msg.sender, convert(payout, uint256)), "Payout failed"
    
    self.insuranceFund += closeFee
    
    log PositionClosed(msg.sender, pnl, fundingFee)
    
    return netPnL

@internal
@view
def _calculatePnL(pos: Position, currentPrice: uint256) -> int256:
    """
    คำนวณ PnL
    
    Long: (currentPrice - entryPrice) * size / entryPrice
    Short: (entryPrice - currentPrice) * size / entryPrice
    """
    if pos.isLong:
        if currentPrice >= pos.entryPrice:
            return convert((currentPrice - pos.entryPrice) * pos.size / pos.entryPrice, int256)
        else:
            return -convert((pos.entryPrice - currentPrice) * pos.size / pos.entryPrice, int256)
    else:
        if pos.entryPrice >= currentPrice:
            return convert((pos.entryPrice - currentPrice) * pos.size / pos.entryPrice, int256)
        else:
            return -convert((currentPrice - pos.entryPrice) * pos.size / pos.entryPrice, int256)

@internal
@view
def _calculateFundingFee(pos: Position) -> int256:
    """
    คำนวณ accumulated funding fee
    
    fundingPayment = position.size * (currentFundingIndex - entryFundingIndex)
    
    ถ้า long: จ่าย positive funding rate (ขาดทุน)
    ถ้า short: รับ positive funding rate (กำไร)
    """
    indexDiff: int256 = convert(self.fundingIndex, int256) - convert(pos.lastFundingIndex, int256)
    fundingPayment: int256 = indexDiff * convert(pos.size, int256) / convert(SCALE, int256)
    
    if pos.isLong:
        return fundingPayment
    else:
        return -fundingPayment

# ===== Liquidation =====

@external
def liquidate(trader: address):
    """
    Liquidate under-collateralized position
    
    Liquidation threshold: collateral ratio < maintenanceMargin
    """
    pos: Position = self.positions[trader]
    assert pos.size > 0, "No position"
    
    self._updateFundingRate()
    
    currentPrice: uint256 = IPriceOracle(self.oracle).getPrice(self.underlying)
    
    # คำนวณ collateral ratio
    pnl: int256 = self._calculatePnL(pos, currentPrice)
    fundingFee: int256 = self._calculateFundingFee(pos)
    
    remainingCollateral: int256 = convert(pos.collateral, int256) + pnl - fundingFee
    
    # Collateral ratio = remainingCollateral / size * 10000
    collateralRatioBps: uint256 = 0
    if remainingCollateral > 0:
        collateralRatioBps = convert(remainingCollateral, uint256) * 10000 / pos.size
    
    assert collateralRatioBps <= self.maintenanceMargin, "Position healthy"
    
    # Liquidator reward
    liquidatorReward: uint256 = pos.size * self.liquidationThreshold / 10000 / 2  # half of threshold
    
    if pos.isLong:
        self.longOpenInterest -= pos.size
    else:
        self.shortOpenInterest -= pos.size
    
    self.positions[trader] = Position({size: 0, collateral: 0, entryPrice: 0, isLong: False, lastFundingIndex: 0})
    
    # จ่าย liquidator reward
    if remainingCollateral > 0:
        remaining: uint256 = convert(remainingCollateral, uint256)
        reward: uint256 = liquidatorReward if liquidatorReward <= remaining else remaining
        assert ERC20(self.collateralToken).transfer(msg.sender, reward), "Reward failed"
        
        # ส่วนที่เหลือเข้า insurance fund
        self.insuranceFund += remaining - reward
    
    log Liquidated(trader, msg.sender, pnl)

# ===== View Functions =====

@external
@view
def getPositionPnL(trader: address) -> (int256, int256, int256):
    """
    ดู PnL ของ position
    Returns: pnl, fundingFee, netPnL
    """
    pos: Position = self.positions[trader]
    if pos.size == 0:
        return 0, 0, 0
    
    currentPrice: uint256 = IPriceOracle(self.oracle).getPrice(self.underlying)
    pnl: int256 = self._calculatePnL(pos, currentPrice)
    fundingFee: int256 = self._calculateFundingFee(pos)
    
    return pnl, fundingFee, pnl - fundingFee

@external
@view
def getMarginRatio(trader: address) -> uint256:
    """
    Margin ratio ของ position
    Returns: ratio ใน basis points (1000 = 10%)
    """
    pos: Position = self.positions[trader]
    if pos.size == 0:
        return max_value(uint256)
    
    currentPrice: uint256 = IPriceOracle(self.oracle).getPrice(self.underlying)
    pnl: int256 = self._calculatePnL(pos, currentPrice)
    fundingFee: int256 = self._calculateFundingFee(pos)
    
    remainingCollateral: int256 = convert(pos.collateral, int256) + pnl - fundingFee
    
    if remainingCollateral <= 0:
        return 0
    
    return convert(remainingCollateral, uint256) * 10000 / pos.size
```

---

## 4. Tests

```python
# tests/test_derivatives.py
import pytest
from brownie import OptionsVault, PerpetualMarket, MockERC20, MockOracle, accounts, chain

SCALE = 10**18

@pytest.fixture
def setup():
    owner = accounts[0]
    alice = accounts[1]
    bob = accounts[2]
    
    usdc = MockERC20.deploy("USDC", "USDC", 6, {"from": owner})
    weth = MockERC20.deploy("WETH", "WETH", 18, {"from": owner})
    
    oracle = MockOracle.deploy({"from": owner})
    oracle.setPrice(weth.address, 2000 * SCALE, {"from": owner})
    
    options = OptionsVault.deploy(oracle.address, owner.address, owner.address, {"from": owner})
    perps = PerpetualMarket.deploy(oracle.address, usdc.address, weth.address, owner.address, {"from": owner})
    
    # Mint tokens
    usdc.mint(alice, 10**10, {"from": owner})
    usdc.mint(bob, 10**10, {"from": owner})
    weth.mint(alice, 10**21, {"from": owner})
    weth.mint(bob, 10**21, {"from": owner})
    
    return owner, alice, bob, usdc, weth, oracle, options, perps

def test_call_option(setup):
    owner, alice, bob, usdc, weth, oracle, options, perps = setup
    
    strike = 2200 * SCALE  # Strike at $2200
    expiry = chain.time() + 7 * 24 * 3600  # 7 days
    size = 10**18  # 1 ETH
    premium = 50 * 10**6  # $50 USDC
    
    # Bob (writer) writes call option
    weth.approve(options.address, size, {"from": bob})
    usdc.approve(options.address, premium, {"from": alice})  # Alice pays premium
    
    option_id = options.writeCallOption(
        alice.address,  # holder
        weth.address,   # underlying
        usdc.address,   # collateral token
        strike,
        expiry,
        size,
        premium,
        {"from": bob}
    ).return_value
    
    print(f"Call option created with ID: {option_id}")
    
    # Price goes up to $2500
    oracle.setPrice(weth.address, 2500 * SCALE, {"from": owner})
    
    # Fast forward to expiry
    chain.sleep(7 * 24 * 3600)
    chain.mine(1)
    
    # Alice exercises
    payment = 2200 * 10**6  # $2200 USDC
    usdc.approve(options.address, payment, {"from": alice})
    
    alice_weth_before = weth.balanceOf(alice.address)
    options.exerciseOption(option_id, {"from": alice})
    alice_weth_after = weth.balanceOf(alice.address)
    
    assert alice_weth_after - alice_weth_before == size
    print(f"Alice exercised and received {(alice_weth_after - alice_weth_before) / 10**18:.4f} ETH")

def test_perpetual_long(setup):
    owner, alice, bob, usdc, weth, oracle, options, perps = setup
    
    # Alice opens long
    collateral = 1000 * 10**6  # $1000 USDC
    leverage = 5  # 5x
    
    usdc.approve(perps.address, collateral * 2, {"from": alice})
    
    perps.openPosition(True, collateral, leverage, {"from": alice})
    
    # Price goes up 10%
    oracle.setPrice(weth.address, 2200 * SCALE, {"from": owner})
    
    pnl, funding, net_pnl = perps.getPositionPnL(alice.address)
    
    print(f"PnL: ${pnl / 10**6:.2f}")
    print(f"Funding fee: ${funding / 10**6:.2f}")
    print(f"Net PnL: ${net_pnl / 10**6:.2f}")
    
    # Close position
    before = usdc.balanceOf(alice.address)
    perps.closePosition({"from": alice})
    after = usdc.balanceOf(alice.address)
    
    print(f"Alice received: ${(after - before) / 10**6:.2f}")

def test_funding_rate(setup):
    owner, alice, bob, usdc, weth, oracle, options, perps = setup
    
    initial_funding = perps.fundingRate()
    print(f"Initial funding rate: {initial_funding}")
    
    # Set mark price higher than index (longs dominant)
    # In practice, mark price comes from oracle
    
    chain.sleep(3600)
    chain.mine(1)
    
    perps.updateFundingRate({"from": owner})
    
    new_funding = perps.fundingRate()
    print(f"Updated funding rate: {new_funding}")

def test_liquidation(setup):
    owner, alice, bob, usdc, weth, oracle, options, perps = setup
    
    # Alice opens long with high leverage
    collateral = 100 * 10**6  # $100 USDC
    leverage = 10  # 10x = $1000 position
    
    usdc.approve(perps.address, collateral * 2, {"from": alice})
    perps.openPosition(True, collateral, leverage, {"from": alice})
    
    # Price drops 15% (big loss for 10x long)
    oracle.setPrice(weth.address, 1700 * SCALE, {"from": owner})
    
    margin = perps.getMarginRatio(alice.address)
    print(f"Margin ratio after price drop: {margin / 100:.2f}%")
    
    # Bob liquidates
    if margin <= perps.maintenanceMargin():
        bob_before = usdc.balanceOf(bob.address)
        perps.liquidate(alice.address, {"from": bob})
        bob_after = usdc.balanceOf(bob.address)
        print(f"Bob received liquidation reward: ${(bob_after - bob_before) / 10**6:.2f}")
```

---

## 5. สรุป

### Derivatives Concepts:

**1. Options Pricing**
- Intrinsic value + Time value
- Higher IV = Higher premium
- Decay toward expiry (theta)

**2. Perpetuals vs Futures**
- Futures: หมดอายุ, ราคา converge กับ spot
- Perpetuals: ไม่หมดอายุ, ใช้ funding rate แทน

**3. Funding Rate Mechanism**
- ป้องกัน mark price diverge จาก index
- Positive rate: longs จ่าย shorts
- Negative rate: shorts จ่าย longs

**4. Risk Management**
- Leverage = amplifies both gains AND losses
- Maintenance margin = last line before liquidation
- Insurance fund = cover bankrupt positions
