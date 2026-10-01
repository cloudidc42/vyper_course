# Part 053: Uniswap V3 Concepts - Concentrated Liquidity

## สารบัญ (Table of Contents)
1. บทนำ Concentrated Liquidity
2. Tick System
3. Price Range และ Position
4. Liquidity Math
5. Simplified V3 Pool
6. Position Manager
7. Fee Collection per Position
8. Range Orders
9. Tests

---

## 1. บทนำ Concentrated Liquidity

Uniswap V3 ปฏิวัติวงการ AMM ด้วย **Concentrated Liquidity**:

**V2 vs V3:**
- V2: liquidity กระจายตลอดช่วง (0, ∞) - ไม่มีประสิทธิภาพ
- V3: liquidity อยู่ใน range ที่กำหนด [a, b] - มีประสิทธิภาพสูงกว่ามาก

**ข้อดีของ V3:**
- Capital efficiency สูงกว่า V2 ถึง 4000x
- LP ควบคุมช่วงราคาได้
- Fee tier หลายระดับ (0.05%, 0.3%, 1%)
- Range orders (limit orders แบบหนึ่ง)

**สูตรพื้นฐาน:**
```
L = sqrt(x * y)
p = y/x (ราคา)
sqrt_price = sqrt(p)
```

---

## 2. Tick System

Ticks คือจุดราคาที่แบ่งช่วงราคาออกเป็นส่วนเล็กๆ

```vyper
# @version 0.4.0
# contracts/TickMath.vy
# คำนวณ sqrt price จาก tick

# Constants
MIN_TICK: constant(int24) = -887272
MAX_TICK: constant(int24) = 887272
MIN_SQRT_RATIO: constant(uint160) = 4295128739
MAX_SQRT_RATIO: constant(uint160) = 1461446703485210103287273052203988822378723970342

@external
@pure
def tickToSqrtPriceX96(tick: int24) -> uint160:
    """
    แปลง tick เป็น sqrt(price) * 2^96
    
    ราคาแต่ละ tick = 1.0001^tick
    sqrt(ราคา) = 1.0001^(tick/2)
    
    Parameters:
        tick: ค่า tick (-887272 ถึง 887272)
    
    Returns:
        sqrt price แบบ Q64.96 fixed point
    """
    assert tick >= convert(MIN_TICK, int24), "Tick too low"
    assert tick <= convert(MAX_TICK, int24), "Tick too high"
    
    absTick: uint256 = 0
    if tick < 0:
        absTick = convert(-tick, uint256)
    else:
        absTick = convert(tick, uint256)
    
    # Simplified calculation (production code uses lookup table)
    # ratio = 2^128 / sqrt(1.0001)^absTick
    ratio: uint256 = 340282366920938463463374607431768211456  # 2^128
    
    # Apply tick bits (simplified)
    if absTick & 1 != 0:
        ratio = ratio * 340248342086729790484326174814286782778 / 2**128
    if absTick & 2 != 0:
        ratio = ratio * 340214320654664441115589010687249662991 / 2**128
    # ... more bits in production
    
    if tick > 0:
        ratio = max_value(uint256) / ratio
    
    # Convert to Q64.96
    sqrtPriceX96: uint256 = ratio >> 32
    
    return convert(sqrtPriceX96, uint160)

@external
@pure
def sqrtPriceX96ToTick(sqrtPriceX96: uint160) -> int24:
    """
    แปลง sqrt price เป็น tick ที่ใกล้ที่สุด
    """
    assert sqrtPriceX96 >= convert(MIN_SQRT_RATIO, uint160), "Price too low"
    assert sqrtPriceX96 <= convert(MAX_SQRT_RATIO, uint160), "Price too high"
    
    # log base 1.0001 calculation (simplified)
    price: uint256 = convert(sqrtPriceX96, uint256) ** 2 / 2**96
    
    # Approximate tick
    tick: int24 = 0  # Simplified
    
    return tick

@external
@pure
def getTickAtSqrtRatio(sqrtPriceX96: uint160) -> int24:
    """คำนวณ tick จาก sqrt price"""
    return self.sqrtPriceX96ToTick(sqrtPriceX96)
```

---

## 3. Simplified Uniswap V3 Pool

```vyper
# @version 0.4.0
# contracts/ConcentratedLiquidityPool.vy
# Simplified Concentrated Liquidity Pool

from vyper.interfaces import ERC20

# Structs
struct Tick:
    liquidityGross: uint128    # รวม liquidity ที่มี reference ถึง tick นี้
    liquidityNet: int128       # liquidity เพิ่ม/ลดเมื่อผ่าน tick
    feeGrowthOutside0X128: uint256  # fee growth สำหรับ token0 นอก range
    feeGrowthOutside1X128: uint256  # fee growth สำหรับ token1 นอก range
    initialized: bool

struct Position:
    liquidity: uint128         # จำนวน liquidity
    feeGrowthInside0LastX128: uint256  # fee growth ที่ position เก็บล่าสุด
    feeGrowthInside1LastX128: uint256
    tokensOwed0: uint128       # fee ที่รอ collect
    tokensOwed1: uint128

struct Slot0:
    sqrtPriceX96: uint160     # sqrt(price) * 2^96
    tick: int24                # tick ปัจจุบัน
    observationIndex: uint16   # index ของ observation ล่าสุด
    feeProtocol: uint8         # protocol fee percentage

# Events
event Initialize:
    sqrtPriceX96: uint160
    tick: int24

event Mint:
    sender: indexed(address)
    owner: indexed(address)
    tickLower: int24
    tickUpper: int24
    amount: uint128
    amount0: uint256
    amount1: uint256

event Burn:
    owner: indexed(address)
    tickLower: int24
    tickUpper: int24
    amount: uint128
    amount0: uint256
    amount1: uint256

event Swap:
    sender: indexed(address)
    recipient: indexed(address)
    amount0: int256
    amount1: int256
    sqrtPriceX96: uint160
    liquidity: uint128
    tick: int24

event Collect:
    owner: indexed(address)
    recipient: address
    tickLower: int24
    tickUpper: int24
    amount0: uint128
    amount1: uint128

# State Variables
token0: public(address)
token1: public(address)
fee: public(uint24)           # Fee tier (500, 3000, 10000)
tickSpacing: public(int24)    # ระยะห่างระหว่าง ticks

maxLiquidityPerTick: public(uint128)

slot0: public(Slot0)
feeGrowthGlobal0X128: public(uint256)
feeGrowthGlobal1X128: public(uint256)
protocolFees0: public(uint128)
protocolFees1: public(uint128)
liquidity: public(uint128)    # liquidity รวมที่ active อยู่ตอนนี้

# Mappings
ticks: HashMap[int24, Tick]
tickBitmap: HashMap[int16, uint256]  # Bitmap สำหรับหา initialized ticks
positions: HashMap[bytes32, Position]

factory: public(address)
_locked: bool

@deploy
def __init__(
    _token0: address,
    _token1: address,
    _fee: uint24,
    _tickSpacing: int24
):
    self.token0 = _token0
    self.token1 = _token1
    self.fee = _fee
    self.tickSpacing = _tickSpacing
    self.factory = msg.sender

@external
def initialize(sqrtPriceX96: uint160):
    """
    Initialize pool ด้วย initial price
    ต้องเรียกก่อน mint ครั้งแรก
    """
    assert self.slot0.sqrtPriceX96 == 0, "Already initialized"
    
    tick: int24 = 0  # Calculate from sqrtPriceX96 in production
    
    self.slot0 = Slot0({
        sqrtPriceX96: sqrtPriceX96,
        tick: tick,
        observationIndex: 0,
        feeProtocol: 0
    })
    
    log Initialize(sqrtPriceX96, tick)

# ===== Liquidity Math =====

@internal
@pure
def _getLiquidityForAmounts(
    sqrtRatioX96: uint160,
    sqrtRatioAX96: uint160,
    sqrtRatioBX96: uint160,
    amount0: uint256,
    amount1: uint256
) -> uint128:
    """
    คำนวณ liquidity จาก amounts
    
    ถ้าราคาปัจจุบันอยู่ต่ำกว่า range: liquidity = amount0 * (sqrtA * sqrtB) / (sqrtB - sqrtA)
    ถ้าราคาปัจจุบันอยู่ใน range: liquidity = min(L0, L1)
    ถ้าราคาปัจจุบันอยู่สูงกว่า range: liquidity = amount1 / (sqrtB - sqrtA)
    """
    _sqrtRatioAX96: uint160 = sqrtRatioAX96
    _sqrtRatioBX96: uint160 = sqrtRatioBX96
    
    if _sqrtRatioAX96 > _sqrtRatioBX96:
        _sqrtRatioAX96 = sqrtRatioBX96
        _sqrtRatioBX96 = sqrtRatioAX96
    
    liquidity: uint128 = 0
    
    if sqrtRatioX96 <= _sqrtRatioAX96:
        # ราคาอยู่ต่ำกว่า range - ใช้แค่ token0
        # L = amount0 * (sqrtA * sqrtB) / (sqrtB - sqrtA) / 2^96
        numerator: uint256 = amount0 * convert(_sqrtRatioAX96, uint256) * convert(_sqrtRatioBX96, uint256) / 2**96
        denominator: uint256 = convert(_sqrtRatioBX96 - _sqrtRatioAX96, uint256)
        liquidity = convert(numerator / denominator, uint128)
    elif sqrtRatioX96 < _sqrtRatioBX96:
        # ราคาอยู่ใน range - ใช้ทั้งสอง tokens
        liquidity0: uint128 = 0
        liquidity1: uint128 = 0
        
        # Simplified calculation
        liquidity0 = convert(amount0 * convert(_sqrtRatioAX96, uint256) / 2**96, uint128)
        liquidity1 = convert(amount1 * 2**96 / convert(_sqrtRatioBX96 - sqrtRatioX96, uint256), uint128)
        
        if liquidity0 < liquidity1:
            liquidity = liquidity0
        else:
            liquidity = liquidity1
    else:
        # ราคาอยู่สูงกว่า range - ใช้แค่ token1
        # L = amount1 / (sqrtB - sqrtA) * 2^96
        liquidity = convert(amount1 * 2**96 / convert(_sqrtRatioBX96 - _sqrtRatioAX96, uint256), uint128)
    
    return liquidity

@internal
@pure
def _getAmountsForLiquidity(
    sqrtRatioX96: uint160,
    sqrtRatioAX96: uint160,
    sqrtRatioBX96: uint160,
    liquidity: uint128
) -> (uint256, uint256):
    """
    คำนวณ amounts จาก liquidity
    ใช้เมื่อต้องการรู้ว่า deposit/withdraw ต้องใช้ tokens เท่าไหร่
    """
    _sqrtRatioAX96: uint160 = sqrtRatioAX96
    _sqrtRatioBX96: uint160 = sqrtRatioBX96
    
    if _sqrtRatioAX96 > _sqrtRatioBX96:
        _sqrtRatioAX96 = sqrtRatioBX96
        _sqrtRatioBX96 = sqrtRatioAX96
    
    amount0: uint256 = 0
    amount1: uint256 = 0
    
    if sqrtRatioX96 <= _sqrtRatioAX96:
        # ต้องการแค่ token0
        amount0 = convert(liquidity, uint256) * 2**96 * convert(_sqrtRatioBX96 - _sqrtRatioAX96, uint256) / (convert(_sqrtRatioAX96, uint256) * convert(_sqrtRatioBX96, uint256))
    elif sqrtRatioX96 < _sqrtRatioBX96:
        # ต้องการทั้งสอง
        amount0 = convert(liquidity, uint256) * 2**96 * convert(_sqrtRatioBX96 - sqrtRatioX96, uint256) / (convert(sqrtRatioX96, uint256) * convert(_sqrtRatioBX96, uint256))
        amount1 = convert(liquidity, uint256) * convert(sqrtRatioX96 - _sqrtRatioAX96, uint256) / 2**96
    else:
        # ต้องการแค่ token1
        amount1 = convert(liquidity, uint256) * convert(_sqrtRatioBX96 - _sqrtRatioAX96, uint256) / 2**96
    
    return amount0, amount1

# ===== Position Management =====

@internal
def _getPositionKey(owner: address, tickLower: int24, tickUpper: int24) -> bytes32:
    """สร้าง key สำหรับ position"""
    return keccak256(concat(
        convert(owner, bytes20),
        convert(convert(tickLower, int256), bytes32),
        convert(convert(tickUpper, int256), bytes32)
    ))

@internal
def _updatePosition(
    owner: address,
    tickLower: int24,
    tickUpper: int24,
    liquidityDelta: int128,
    tick: int24
) -> Position:
    """อัพเดท position สำหรับ owner"""
    positionKey: bytes32 = self._getPositionKey(owner, tickLower, tickUpper)
    position: Position = self.positions[positionKey]
    
    feeGrowthInside0X128: uint256 = 0
    feeGrowthInside1X128: uint256 = 0
    
    # Calculate fee growth inside range
    # (Simplified - production needs proper tick fee accounting)
    feeGrowthInside0X128 = self.feeGrowthGlobal0X128
    feeGrowthInside1X128 = self.feeGrowthGlobal1X128
    
    # Accumulate fees owed
    if position.liquidity > 0:
        tokensOwed0: uint256 = convert(position.liquidity, uint256) * (feeGrowthInside0X128 - position.feeGrowthInside0LastX128) / 2**128
        tokensOwed1: uint256 = convert(position.liquidity, uint256) * (feeGrowthInside1X128 - position.feeGrowthInside1LastX128) / 2**128
        position.tokensOwed0 += convert(tokensOwed0, uint128)
        position.tokensOwed1 += convert(tokensOwed1, uint128)
    
    position.feeGrowthInside0LastX128 = feeGrowthInside0X128
    position.feeGrowthInside1LastX128 = feeGrowthInside1X128
    
    if liquidityDelta != 0:
        if liquidityDelta > 0:
            position.liquidity += convert(liquidityDelta, uint128)
        else:
            position.liquidity -= convert(-liquidityDelta, uint128)
    
    self.positions[positionKey] = position
    return position

# ===== Mint (Add Liquidity) =====

@external
def mint(
    recipient: address,
    tickLower: int24,
    tickUpper: int24,
    amount: uint128,
    data: Bytes[256]
) -> (uint256, uint256):
    """
    เพิ่ม liquidity ใน range [tickLower, tickUpper]
    
    Parameters:
        recipient: เจ้าของ position
        tickLower: tick ล่างของ range
        tickUpper: tick บนของ range
        amount: จำนวน liquidity ที่ต้องการเพิ่ม
    
    Returns:
        amount0: token0 ที่ต้องการ
        amount1: token1 ที่ต้องการ
    """
    assert not self._locked, "Reentrancy"
    self._locked = True
    
    assert amount > 0, "Zero liquidity"
    assert tickLower < tickUpper, "Invalid tick range"
    assert tickLower >= convert(MIN_TICK, int24), "Tick too low"
    assert tickUpper <= convert(MAX_TICK, int24), "Tick too high"
    assert tickLower % self.tickSpacing == 0, "Invalid tickLower"
    assert tickUpper % self.tickSpacing == 0, "Invalid tickUpper"
    
    _slot0: Slot0 = self.slot0
    assert _slot0.sqrtPriceX96 != 0, "Not initialized"
    
    # คำนวณ amounts ที่ต้องการ
    sqrtRatioAX96: uint160 = 0  # sqrtPriceAtTick(tickLower) - simplified
    sqrtRatioBX96: uint160 = 0  # sqrtPriceAtTick(tickUpper) - simplified
    
    # Simplified: ใช้ค่าประมาณ
    sqrtRatioAX96 = convert(convert(tickLower + 887272, uint256) * 2**96 / 1774544, uint160)
    sqrtRatioBX96 = convert(convert(tickUpper + 887272, uint256) * 2**96 / 1774544, uint160)
    
    amount0: uint256 = 0
    amount1: uint256 = 0
    amount0, amount1 = self._getAmountsForLiquidity(
        _slot0.sqrtPriceX96,
        sqrtRatioAX96,
        sqrtRatioBX96,
        amount
    )
    
    # อัพเดท ticks
    # (Simplified - production needs full tick management)
    
    # อัพเดท global liquidity ถ้า position อยู่ใน range ปัจจุบัน
    if _slot0.tick >= tickLower and _slot0.tick < tickUpper:
        self.liquidity += amount
    
    # อัพเดท position
    self._updatePosition(recipient, tickLower, tickUpper, convert(amount, int128), _slot0.tick)
    
    # โอน tokens
    token0: ERC20 = ERC20(self.token0)
    token1: ERC20 = ERC20(self.token1)
    
    if amount0 > 0:
        assert token0.transferFrom(msg.sender, self, amount0), "Transfer0 failed"
    if amount1 > 0:
        assert token1.transferFrom(msg.sender, self, amount1), "Transfer1 failed"
    
    log Mint(msg.sender, recipient, tickLower, tickUpper, amount, amount0, amount1)
    
    self._locked = False
    return amount0, amount1

# ===== Burn (Remove Liquidity) =====

@external
def burn(
    tickLower: int24,
    tickUpper: int24,
    amount: uint128
) -> (uint256, uint256):
    """
    ลด liquidity ใน range
    
    Returns:
        amount0: token0 ที่จะได้รับ
        amount1: token1 ที่จะได้รับ
    """
    assert not self._locked, "Reentrancy"
    self._locked = True
    
    _slot0: Slot0 = self.slot0
    
    sqrtRatioAX96: uint160 = convert(convert(tickLower + 887272, uint256) * 2**96 / 1774544, uint160)
    sqrtRatioBX96: uint160 = convert(convert(tickUpper + 887272, uint256) * 2**96 / 1774544, uint160)
    
    amount0: uint256 = 0
    amount1: uint256 = 0
    amount0, amount1 = self._getAmountsForLiquidity(
        _slot0.sqrtPriceX96,
        sqrtRatioAX96,
        sqrtRatioBX96,
        amount
    )
    
    # อัพเดท global liquidity
    if _slot0.tick >= tickLower and _slot0.tick < tickUpper:
        self.liquidity -= amount
    
    # อัพเดท position (เพิ่ม tokens owed แทนการโอนทันที)
    positionKey: bytes32 = self._getPositionKey(msg.sender, tickLower, tickUpper)
    position: Position = self.positions[positionKey]
    position.liquidity -= amount
    position.tokensOwed0 += convert(amount0, uint128)
    position.tokensOwed1 += convert(amount1, uint128)
    self.positions[positionKey] = position
    
    log Burn(msg.sender, tickLower, tickUpper, amount, amount0, amount1)
    
    self._locked = False
    return amount0, amount1

# ===== Collect Fees =====

@external
def collect(
    recipient: address,
    tickLower: int24,
    tickUpper: int24,
    amount0Requested: uint128,
    amount1Requested: uint128
) -> (uint128, uint128):
    """
    Collect fees ที่สะสมไว้
    
    Parameters:
        amount0Requested: จำนวน token0 ที่ต้องการ collect (max_value = collect ทั้งหมด)
        amount1Requested: จำนวน token1 ที่ต้องการ collect
    """
    assert not self._locked, "Reentrancy"
    self._locked = True
    
    positionKey: bytes32 = self._getPositionKey(msg.sender, tickLower, tickUpper)
    position: Position = self.positions[positionKey]
    
    amount0: uint128 = amount0Requested
    if amount0 > position.tokensOwed0:
        amount0 = position.tokensOwed0
    
    amount1: uint128 = amount1Requested
    if amount1 > position.tokensOwed1:
        amount1 = position.tokensOwed1
    
    if amount0 > 0:
        position.tokensOwed0 -= amount0
        assert ERC20(self.token0).transfer(recipient, convert(amount0, uint256)), "Transfer0 failed"
    if amount1 > 0:
        position.tokensOwed1 -= amount1
        assert ERC20(self.token1).transfer(recipient, convert(amount1, uint256)), "Transfer1 failed"
    
    self.positions[positionKey] = position
    
    log Collect(msg.sender, recipient, tickLower, tickUpper, amount0, amount1)
    
    self._locked = False
    return amount0, amount1

# ===== Swap =====

@external
def swap(
    recipient: address,
    zeroForOne: bool,
    amountSpecified: int256,
    sqrtPriceLimitX96: uint160,
    data: Bytes[256]
) -> (int256, int256):
    """
    Swap tokens ใน pool
    
    Parameters:
        zeroForOne: ถ้า true = swap token0 -> token1, false = swap token1 -> token0
        amountSpecified: > 0 = exact input, < 0 = exact output
        sqrtPriceLimitX96: ราคา limit สำหรับ swap (ป้องกัน slippage)
    
    Returns:
        amount0: token0 delta (+ = pool ได้รับ, - = pool ให้ออก)
        amount1: token1 delta
    """
    assert not self._locked, "Reentrancy"
    self._locked = True
    
    _slot0: Slot0 = self.slot0
    assert _slot0.sqrtPriceX96 != 0, "Not initialized"
    
    if zeroForOne:
        assert sqrtPriceLimitX96 < _slot0.sqrtPriceX96, "Price limit too high"
        assert sqrtPriceLimitX96 > convert(MIN_SQRT_RATIO, uint160), "Price limit too low"
    else:
        assert sqrtPriceLimitX96 > _slot0.sqrtPriceX96, "Price limit too low"
        assert sqrtPriceLimitX96 < convert(MAX_SQRT_RATIO, uint160), "Price limit too high"
    
    # Simplified swap logic (production needs step-by-step tick crossing)
    amount0: int256 = 0
    amount1: int256 = 0
    
    _liquidity: uint128 = self.liquidity
    
    if amountSpecified > 0:
        # Exact input
        amountIn: uint256 = convert(amountSpecified, uint256)
        feeAmount: uint256 = amountIn * convert(self.fee, uint256) / 1000000
        amountInAfterFee: uint256 = amountIn - feeAmount
        
        # Simplified price calculation
        if zeroForOne:
            # token0 in, token1 out
            amount0 = convert(amountIn, int256)
            # amountOut = liquidity * (1/sqrtA - 1/sqrtB)
            # Simplified: use current price
            currentPrice: uint256 = convert(_slot0.sqrtPriceX96, uint256) ** 2 / 2**96
            amountOut: uint256 = amountInAfterFee * currentPrice / 10**18
            amount1 = -convert(amountOut, int256)
        else:
            amount1 = convert(amountIn, int256)
            currentPrice: uint256 = convert(_slot0.sqrtPriceX96, uint256) ** 2 / 2**96
            amountOut: uint256 = amountInAfterFee * 10**18 / currentPrice
            amount0 = -convert(amountOut, int256)
        
        # Update fee growth
        if zeroForOne:
            self.feeGrowthGlobal0X128 += feeAmount * 2**128 / convert(_liquidity, uint256)
        else:
            self.feeGrowthGlobal1X128 += feeAmount * 2**128 / convert(_liquidity, uint256)
    
    # Transfer tokens
    token0: ERC20 = ERC20(self.token0)
    token1: ERC20 = ERC20(self.token1)
    
    if amount0 < 0:
        assert token0.transfer(recipient, convert(-amount0, uint256)), "Transfer0 failed"
    if amount1 < 0:
        assert token1.transfer(recipient, convert(-amount1, uint256)), "Transfer1 failed"
    if amount0 > 0:
        assert token0.transferFrom(msg.sender, self, convert(amount0, uint256)), "TransferFrom0 failed"
    if amount1 > 0:
        assert token1.transferFrom(msg.sender, self, convert(amount1, uint256)), "TransferFrom1 failed"
    
    log Swap(msg.sender, recipient, amount0, amount1, _slot0.sqrtPriceX96, _liquidity, _slot0.tick)
    
    self._locked = False
    return amount0, amount1

# ===== View Functions =====

@external
@view
def getPosition(
    owner: address,
    tickLower: int24,
    tickUpper: int24
) -> (uint128, uint256, uint256, uint128, uint128):
    """
    ดู position ของ owner
    
    Returns:
        liquidity, feeGrowthInside0LastX128, feeGrowthInside1LastX128, tokensOwed0, tokensOwed1
    """
    positionKey: bytes32 = self._getPositionKey(owner, tickLower, tickUpper)
    pos: Position = self.positions[positionKey]
    return (
        pos.liquidity,
        pos.feeGrowthInside0LastX128,
        pos.feeGrowthInside1LastX128,
        pos.tokensOwed0,
        pos.tokensOwed1
    )

@external
@view
def getCurrentPrice() -> (uint160, int24):
    """ราคาและ tick ปัจจุบัน"""
    return self.slot0.sqrtPriceX96, self.slot0.tick

@external
@view
def getActiveLiquidity() -> uint128:
    """Liquidity ที่ active อยู่ตอนนี้"""
    return self.liquidity
```

---

## 4. Position Manager NFT

```vyper
# @version 0.4.0
# contracts/NonfungiblePositionManager.vy
# NFT Position Manager - แทน positions ด้วย NFT

# Struct สำหรับเก็บข้อมูล position
struct PositionInfo:
    nonce: uint96
    operator: address
    token0: address
    token1: address
    fee: uint24
    tickLower: int24
    tickUpper: int24
    liquidity: uint128
    feeGrowthInside0LastX128: uint256
    feeGrowthInside1LastX128: uint256
    tokensOwed0: uint128
    tokensOwed1: uint128

# Events
event IncreaseLiquidity:
    tokenId: indexed(uint256)
    liquidity: uint128
    amount0: uint256
    amount1: uint256

event DecreaseLiquidity:
    tokenId: indexed(uint256)
    liquidity: uint128
    amount0: uint256
    amount1: uint256

event Collect:
    tokenId: indexed(uint256)
    recipient: address
    amount0: uint128
    amount1: uint128

event Transfer:
    from_: indexed(address)
    to: indexed(address)
    tokenId: indexed(uint256)

# ERC721 State
name: public(String[64])
symbol: public(String[32])
balanceOf: public(HashMap[address, uint256])
ownerOf: public(HashMap[uint256, address])
getApproved: public(HashMap[uint256, address])
isApprovedForAll: public(HashMap[address, HashMap[address, bool]])

# Position State
positions: HashMap[uint256, PositionInfo]
nextTokenId: uint256
factory: public(address)

@deploy
def __init__(_factory: address):
    self.name = "Uniswap V3 Positions NFT-V1"
    self.symbol = "UNI-V3-POS"
    self.factory = _factory
    self.nextTokenId = 1

@internal
def _mint(to: address, tokenId: uint256):
    assert to != empty(address), "Mint to zero"
    assert self.ownerOf[tokenId] == empty(address), "Token exists"
    self.balanceOf[to] += 1
    self.ownerOf[tokenId] = to
    log Transfer(empty(address), to, tokenId)

@internal
def _burn(tokenId: uint256):
    owner: address = self.ownerOf[tokenId]
    assert owner != empty(address), "Token not found"
    self.balanceOf[owner] -= 1
    self.ownerOf[tokenId] = empty(address)
    log Transfer(owner, empty(address), tokenId)

@external
def mint(
    token0: address,
    token1: address,
    fee: uint24,
    tickLower: int24,
    tickUpper: int24,
    amount0Desired: uint256,
    amount1Desired: uint256,
    amount0Min: uint256,
    amount1Min: uint256,
    recipient: address,
    deadline: uint256
) -> (uint256, uint128, uint256, uint256):
    """
    สร้าง position ใหม่และ mint NFT
    
    Returns:
        tokenId: ID ของ NFT
        liquidity: liquidity ที่ได้
        amount0: token0 ที่ใช้
        amount1: token1 ที่ใช้
    """
    assert block.timestamp <= deadline, "Expired"
    
    tokenId: uint256 = self.nextTokenId
    self.nextTokenId += 1
    
    # สร้าง position (simplified)
    liquidity: uint128 = 0  # Calculate from amounts
    amount0: uint256 = 0
    amount1: uint256 = 0
    
    self.positions[tokenId] = PositionInfo({
        nonce: 0,
        operator: empty(address),
        token0: token0,
        token1: token1,
        fee: fee,
        tickLower: tickLower,
        tickUpper: tickUpper,
        liquidity: liquidity,
        feeGrowthInside0LastX128: 0,
        feeGrowthInside1LastX128: 0,
        tokensOwed0: 0,
        tokensOwed1: 0
    })
    
    self._mint(recipient, tokenId)
    
    log IncreaseLiquidity(tokenId, liquidity, amount0, amount1)
    
    return tokenId, liquidity, amount0, amount1

@external
def increaseLiquidity(
    tokenId: uint256,
    amount0Desired: uint256,
    amount1Desired: uint256,
    amount0Min: uint256,
    amount1Min: uint256,
    deadline: uint256
) -> (uint128, uint256, uint256):
    """เพิ่ม liquidity ใน position ที่มีอยู่"""
    assert block.timestamp <= deadline, "Expired"
    assert self.ownerOf[tokenId] == msg.sender, "Not owner"
    
    position: PositionInfo = self.positions[tokenId]
    
    # Calculate and add liquidity (simplified)
    liquidity: uint128 = 0
    amount0: uint256 = 0
    amount1: uint256 = 0
    
    position.liquidity += liquidity
    self.positions[tokenId] = position
    
    log IncreaseLiquidity(tokenId, liquidity, amount0, amount1)
    
    return liquidity, amount0, amount1

@external
def decreaseLiquidity(
    tokenId: uint256,
    liquidity: uint128,
    amount0Min: uint256,
    amount1Min: uint256,
    deadline: uint256
) -> (uint256, uint256):
    """ลด liquidity ใน position"""
    assert block.timestamp <= deadline, "Expired"
    assert self.ownerOf[tokenId] == msg.sender, "Not owner"
    
    position: PositionInfo = self.positions[tokenId]
    assert position.liquidity >= liquidity, "Insufficient liquidity"
    
    amount0: uint256 = 0
    amount1: uint256 = 0
    
    position.liquidity -= liquidity
    position.tokensOwed0 += convert(amount0, uint128)
    position.tokensOwed1 += convert(amount1, uint128)
    self.positions[tokenId] = position
    
    log DecreaseLiquidity(tokenId, liquidity, amount0, amount1)
    
    return amount0, amount1

@external
def collect(
    tokenId: uint256,
    recipient: address,
    amount0Max: uint128,
    amount1Max: uint128
) -> (uint128, uint128):
    """Collect fees จาก position"""
    assert self.ownerOf[tokenId] == msg.sender, "Not owner"
    
    position: PositionInfo = self.positions[tokenId]
    
    amount0: uint128 = amount0Max
    if amount0 > position.tokensOwed0:
        amount0 = position.tokensOwed0
    
    amount1: uint128 = amount1Max
    if amount1 > position.tokensOwed1:
        amount1 = position.tokensOwed1
    
    position.tokensOwed0 -= amount0
    position.tokensOwed1 -= amount1
    self.positions[tokenId] = position
    
    if amount0 > 0:
        assert ERC20(position.token0).transfer(recipient, convert(amount0, uint256)), "Transfer0 failed"
    if amount1 > 0:
        assert ERC20(position.token1).transfer(recipient, convert(amount1, uint256)), "Transfer1 failed"
    
    log Collect(tokenId, recipient, amount0, amount1)
    
    return amount0, amount1

@external
def transferFrom(from_: address, to: address, tokenId: uint256):
    """โอน NFT"""
    assert self.ownerOf[tokenId] == from_, "Not owner"
    assert to != empty(address), "Zero address"
    assert (
        msg.sender == from_ or
        self.isApprovedForAll[from_][msg.sender] or
        self.getApproved[tokenId] == msg.sender
    ), "Not authorized"
    
    self.balanceOf[from_] -= 1
    self.balanceOf[to] += 1
    self.ownerOf[tokenId] = to
    self.getApproved[tokenId] = empty(address)
    
    log Transfer(from_, to, tokenId)

@external
def approve(spender: address, tokenId: uint256):
    owner: address = self.ownerOf[tokenId]
    assert msg.sender == owner or self.isApprovedForAll[owner][msg.sender], "Not authorized"
    self.getApproved[tokenId] = spender

@external
def setApprovalForAll(operator: address, approved: bool):
    self.isApprovedForAll[msg.sender][operator] = approved

@external
@view
def getPositionInfo(tokenId: uint256) -> PositionInfo:
    return self.positions[tokenId]
```

---

## 5. Range Orders

Range orders คือการใส่ single-sided liquidity ใน range ที่เฉพาะเจาะจง เพื่อทำ limit order แบบ AMM

```vyper
# @version 0.4.0
# contracts/RangeOrderManager.vy
# จัดการ Range Orders

interface IPool:
    def mint(recipient: address, tickLower: int24, tickUpper: int24, amount: uint128, data: Bytes[256]) -> (uint256, uint256): nonpayable
    def burn(tickLower: int24, tickUpper: int24, amount: uint128) -> (uint256, uint256): nonpayable
    def collect(recipient: address, tickLower: int24, tickUpper: int24, amount0Requested: uint128, amount1Requested: uint128) -> (uint128, uint128): nonpayable
    def slot0() -> (uint160, int24, uint16, uint8): view

struct RangeOrder:
    owner: address
    pool: address
    tickLower: int24
    tickUpper: int24
    liquidity: uint128
    tokenIn: address
    amountIn: uint256
    filled: bool

# Events
event RangeOrderCreated:
    orderId: indexed(uint256)
    owner: indexed(address)
    pool: address
    tickLower: int24
    tickUpper: int24

event RangeOrderFilled:
    orderId: indexed(uint256)
    owner: indexed(address)
    amountOut: uint256

event RangeOrderCanceled:
    orderId: indexed(uint256)
    owner: indexed(address)

orders: HashMap[uint256, RangeOrder]
nextOrderId: uint256

@deploy
def __init__():
    self.nextOrderId = 1

@external
def createRangeOrder(
    pool: address,
    tickLower: int24,
    tickUpper: int24,
    liquidity: uint128,
    tokenIn: address,
    amountIn: uint256
) -> uint256:
    """
    สร้าง range order (limit order)
    
    ถ้าราคาขึ้นเหนือ range: แปลง token0 -> token1 ทั้งหมด
    ถ้าราคาลงต่ำกว่า range: แปลง token1 -> token0 ทั้งหมด
    """
    orderId: uint256 = self.nextOrderId
    self.nextOrderId += 1
    
    # โอน tokenIn เข้า
    assert ERC20(tokenIn).transferFrom(msg.sender, self, amountIn), "Transfer failed"
    
    # Approve pool
    assert ERC20(tokenIn).approve(pool, amountIn), "Approve failed"
    
    # เพิ่ม single-sided liquidity
    IPool(pool).mint(self, tickLower, tickUpper, liquidity, b"")
    
    self.orders[orderId] = RangeOrder({
        owner: msg.sender,
        pool: pool,
        tickLower: tickLower,
        tickUpper: tickUpper,
        liquidity: liquidity,
        tokenIn: tokenIn,
        amountIn: amountIn,
        filled: False
    })
    
    log RangeOrderCreated(orderId, msg.sender, pool, tickLower, tickUpper)
    
    return orderId

@external
def checkAndFillOrder(orderId: uint256):
    """
    ตรวจสอบว่า range order ถูก fill แล้วหรือยัง
    ถ้าราคาผ่าน range ไปแล้ว ให้ withdraw
    """
    order: RangeOrder = self.orders[orderId]
    assert not order.filled, "Already filled"
    
    pool: IPool = IPool(order.pool)
    sqrtPriceX96: uint160 = 0
    currentTick: int24 = 0
    sqrtPriceX96, currentTick, _, _ = pool.slot0()
    
    # ตรวจสอบว่าราคาผ่าน range หรือยัง
    if currentTick >= order.tickUpper or currentTick < order.tickLower:
        # Range ถูก fill แล้ว - withdraw
        pool.burn(order.tickLower, order.tickUpper, order.liquidity)
        
        amount0: uint128 = 0
        amount1: uint128 = 0
        amount0, amount1 = pool.collect(
            order.owner,
            order.tickLower,
            order.tickUpper,
            max_value(uint128),
            max_value(uint128)
        )
        
        self.orders[orderId].filled = True
        
        amountOut: uint256 = convert(amount0, uint256) + convert(amount1, uint256)
        log RangeOrderFilled(orderId, order.owner, amountOut)

@external
def cancelRangeOrder(orderId: uint256):
    """ยกเลิก range order และถอน liquidity"""
    order: RangeOrder = self.orders[orderId]
    assert msg.sender == order.owner, "Not owner"
    assert not order.filled, "Already filled"
    
    pool: IPool = IPool(order.pool)
    pool.burn(order.tickLower, order.tickUpper, order.liquidity)
    pool.collect(
        order.owner,
        order.tickLower,
        order.tickUpper,
        max_value(uint128),
        max_value(uint128)
    )
    
    self.orders[orderId].filled = True
    
    log RangeOrderCanceled(orderId, order.owner)
```

---

## 6. Tests

```python
# tests/test_concentrated_liquidity.py
import pytest
from brownie import ConcentratedLiquidityPool, MockERC20, accounts, chain

Q96 = 2**96

def tick_to_sqrt_price(tick):
    """คำนวณ sqrt price จาก tick"""
    return int(1.0001 ** (tick / 2) * Q96)

@pytest.fixture
def setup():
    owner = accounts[0]
    alice = accounts[1]
    bob = accounts[2]
    
    token0 = MockERC20.deploy("Token A", "TKNA", 18, {"from": owner})
    token1 = MockERC20.deploy("Token B", "TKNB", 18, {"from": owner})
    
    if token0.address > token1.address:
        token0, token1 = token1, token0
    
    # Deploy pool with 0.3% fee, tickSpacing 60
    pool = ConcentratedLiquidityPool.deploy(
        token0.address,
        token1.address,
        3000,  # 0.3% fee
        60,    # tick spacing
        {"from": owner}
    )
    
    # Initialize at price 1:1
    sqrt_price = tick_to_sqrt_price(0)
    pool.initialize(sqrt_price, {"from": owner})
    
    # Mint tokens
    for user in [alice, bob]:
        token0.mint(user, 10**24, {"from": owner})
        token1.mint(user, 10**24, {"from": owner})
    
    return owner, alice, bob, token0, token1, pool

def test_initialize(setup):
    _, _, _, _, _, pool = setup
    
    sqrt_price, tick = pool.getCurrentPrice()
    assert sqrt_price > 0
    print(f"Initial sqrt price: {sqrt_price}")
    print(f"Initial tick: {tick}")

def test_mint_position(setup):
    _, alice, _, token0, token1, pool = setup
    
    tick_lower = -600
    tick_upper = 600
    liquidity = 10**15
    
    token0.approve(pool.address, 10**24, {"from": alice})
    token1.approve(pool.address, 10**24, {"from": alice})
    
    amount0, amount1 = pool.mint(
        alice.address,
        tick_lower,
        tick_upper,
        liquidity,
        b"",
        {"from": alice}
    )
    
    assert pool.getActiveLiquidity() == liquidity
    
    pos = pool.getPosition(alice.address, tick_lower, tick_upper)
    assert pos[0] == liquidity  # liquidity
    
    print(f"Minted position: liquidity={liquidity}")
    print(f"Amount0 deposited: {amount0}")
    print(f"Amount1 deposited: {amount1}")

def test_swap_within_range(setup):
    _, alice, bob, token0, token1, pool = setup
    
    # Add liquidity
    token0.approve(pool.address, 10**24, {"from": alice})
    token1.approve(pool.address, 10**24, {"from": alice})
    
    pool.mint(alice.address, -6000, 6000, 10**15, b"", {"from": alice})
    
    # Bob swaps
    token0.approve(pool.address, 10**20, {"from": bob})
    
    bob_t1_before = token1.balanceOf(bob.address)
    
    # Swap token0 for token1
    sqrt_limit = int(tick_to_sqrt_price(-100))  # Slightly lower price
    
    pool.swap(
        bob.address,
        True,  # zeroForOne
        10**18,  # amountSpecified
        sqrt_limit,
        b"",
        {"from": bob}
    )
    
    bob_t1_after = token1.balanceOf(bob.address)
    print(f"Bob received token1: {bob_t1_after - bob_t1_before}")

def test_collect_fees(setup):
    _, alice, bob, token0, token1, pool = setup
    
    # Add liquidity
    token0.approve(pool.address, 10**24, {"from": alice})
    token1.approve(pool.address, 10**24, {"from": alice})
    pool.mint(alice.address, -6000, 6000, 10**15, b"", {"from": alice})
    
    # Multiple swaps to generate fees
    token0.approve(pool.address, 10**22, {"from": bob})
    for _ in range(5):
        pool.swap(bob.address, True, 10**18, 1, b"", {"from": bob})
    
    # Collect fees
    before0 = token0.balanceOf(alice.address)
    before1 = token1.balanceOf(alice.address)
    
    pool.collect(
        alice.address,
        -6000,
        6000,
        max(2**128 - 1, 0),
        max(2**128 - 1, 0),
        {"from": alice}
    )
    
    after0 = token0.balanceOf(alice.address)
    after1 = token1.balanceOf(alice.address)
    
    fees0 = after0 - before0
    fees1 = after1 - before1
    
    print(f"Fees earned: {fees0} token0, {fees1} token1")
```

---

## 7. สรุปความแตกต่าง V2 vs V3

| Feature | Uniswap V2 | Uniswap V3 |
|---------|-----------|-----------|
| Liquidity | Spread 0 to ∞ | Concentrated in range |
| Capital Efficiency | ~25-100% | Up to 4000x better |
| Fee Tiers | Fixed 0.3% | 0.05%, 0.3%, 1% |
| LP Positions | Fungible ERC20 | Non-fungible NFT |
| Impermanent Loss | Lower risk | Higher risk in narrow range |
| Complexity | Simple | Complex |
| Gas Costs | Lower | Higher |

### เมื่อไหรควรใช้ V3 Range:
- **Wide range (-50% to +100%)**: เหมาะกับ volatile assets
- **Narrow range (±5%)**: เหมาะกับ stablecoin pairs (USDC/USDT)
- **Single-sided**: ทำ limit orders

### Risk Management:
- Concentrated positions มี impermanent loss สูงกว่า
- ราคาออกนอก range = position ไม่ earn fees
- ต้องจัดการ rebalancing อย่างสม่ำเสมอ
