# Part 052: AMM Implementation - Uniswap V2 Style

## สารบัญ (Table of Contents)
1. บทนำ AMM (Automated Market Maker)
2. สูตรคณิตศาสตร์ x*y=k
3. LP Token Implementation
4. Add Liquidity
5. Remove Liquidity
6. Swap with Fee
7. Price Impact Calculation
8. Slippage Protection
9. Flash Swap
10. Price Oracle (TWAP)
11. Tests

---

## 1. บทนำ AMM (Automated Market Maker)

AMM คือโปรโตคอลที่ใช้สูตรคณิตศาสตร์ในการกำหนดราคา แทนการใช้ order book แบบดั้งเดิม
Uniswap V2 ใช้สูตร **x * y = k** ซึ่งหมายความว่า:
- x = จำนวน token A ใน pool
- y = จำนวน token B ใน pool  
- k = ค่าคงที่ (invariant)

เมื่อมีการ swap token A เป็น token B:
- x เพิ่มขึ้น (มีคนเอา token A มาใส่)
- y ลดลง (คนได้ token B ออกไป)
- k ยังคงเท่าเดิม (หรือเพิ่มขึ้นเล็กน้อยจาก fee)

```
# interfaces/IERC20.vy
from vyper.interfaces import ERC20

interface IERC20Extended:
    def name() -> String[64]: view
    def symbol() -> String[32]: view
    def decimals() -> uint8: view
    def totalSupply() -> uint256: view
    def balanceOf(account: address) -> uint256: view
    def transfer(to: address, amount: uint256) -> bool: nonpayable
    def transferFrom(from_: address, to: address, amount: uint256) -> bool: nonpayable
    def approve(spender: address, amount: uint256) -> bool: nonpayable
    def allowance(owner: address, spender: address) -> uint256: view
```

---

## 2. LP Token Contract

LP Token ใช้แทนส่วนแบ่งของ liquidity provider ใน pool

```vyper
# @version 0.4.0
# contracts/LPToken.vy
# LP Token สำหรับ AMM Pool

from vyper.interfaces import ERC20

implements: ERC20

# Events
event Transfer:
    sender: indexed(address)
    receiver: indexed(address)
    value: uint256

event Approval:
    owner: indexed(address)
    spender: indexed(address)
    value: uint256

event Mint:
    minter: indexed(address)
    amount: uint256

event Burn:
    burner: indexed(address)
    amount: uint256

# State Variables
name: public(String[64])
symbol: public(String[32])
decimals: public(uint8)
totalSupply: public(uint256)

balanceOf: public(HashMap[address, uint256])
allowance: public(HashMap[address, HashMap[address, uint256]])

minter: public(address)  # AMM contract ที่ mint/burn ได้

@deploy
def __init__(_name: String[64], _symbol: String[32]):
    self.name = _name
    self.symbol = _symbol
    self.decimals = 18
    self.minter = msg.sender

@internal
def _transfer(sender: address, receiver: address, amount: uint256):
    assert sender != empty(address), "Transfer from zero address"
    assert receiver != empty(address), "Transfer to zero address"
    assert self.balanceOf[sender] >= amount, "Insufficient balance"
    
    self.balanceOf[sender] -= amount
    self.balanceOf[receiver] += amount
    log Transfer(sender, receiver, amount)

@external
def transfer(to: address, amount: uint256) -> bool:
    self._transfer(msg.sender, to, amount)
    return True

@external
def transferFrom(from_: address, to: address, amount: uint256) -> bool:
    if self.allowance[from_][msg.sender] != max_value(uint256):
        assert self.allowance[from_][msg.sender] >= amount, "Insufficient allowance"
        self.allowance[from_][msg.sender] -= amount
    self._transfer(from_, to, amount)
    return True

@external
def approve(spender: address, amount: uint256) -> bool:
    assert spender != empty(address), "Approve to zero address"
    self.allowance[msg.sender][spender] = amount
    log Approval(msg.sender, spender, amount)
    return True

@external
def mint(to: address, amount: uint256):
    assert msg.sender == self.minter, "Only minter"
    assert to != empty(address), "Mint to zero address"
    
    self.totalSupply += amount
    self.balanceOf[to] += amount
    log Transfer(empty(address), to, amount)
    log Mint(to, amount)

@external
def burn(from_: address, amount: uint256):
    assert msg.sender == self.minter, "Only minter"
    assert self.balanceOf[from_] >= amount, "Insufficient balance"
    
    self.balanceOf[from_] -= amount
    self.totalSupply -= amount
    log Transfer(from_, empty(address), amount)
    log Burn(from_, amount)

@external
def set_minter(new_minter: address):
    assert msg.sender == self.minter, "Only minter"
    self.minter = new_minter
```

---

## 3. AMM Core Contract - Uniswap V2 Style

```vyper
# @version 0.4.0
# contracts/UniswapV2Pair.vy
# AMM Pair Contract - Uniswap V2 Style
# ใช้สูตร x * y = k สำหรับการ swap

from vyper.interfaces import ERC20

interface ILPToken:
    def mint(to: address, amount: uint256): nonpayable
    def burn(from_: address, amount: uint256): nonpayable
    def totalSupply() -> uint256: view
    def balanceOf(account: address) -> uint256: view
    def transfer(to: address, amount: uint256) -> bool: nonpayable
    def transferFrom(from_: address, to: address, amount: uint256) -> bool: nonpayable

# Events
event LiquidityAdded:
    provider: indexed(address)
    amount0: uint256
    amount1: uint256
    shares: uint256

event LiquidityRemoved:
    provider: indexed(address)
    amount0: uint256
    amount1: uint256
    shares: uint256

event Swap:
    sender: indexed(address)
    recipient: indexed(address)
    amount0In: uint256
    amount1In: uint256
    amount0Out: uint256
    amount1Out: uint256

event Sync:
    reserve0: uint256
    reserve1: uint256

event FlashSwap:
    borrower: indexed(address)
    amount0: uint256
    amount1: uint256

# Constants
MINIMUM_LIQUIDITY: constant(uint256) = 1000  # ป้องกัน inflation attack
FEE_NUMERATOR: constant(uint256) = 997       # 0.3% fee (997/1000)
FEE_DENOMINATOR: constant(uint256) = 1000

# State Variables
token0: public(address)
token1: public(address)
lpToken: public(address)

reserve0: public(uint256)
reserve1: public(uint256)
blockTimestampLast: public(uint256)

# สำหรับ TWAP Oracle
price0CumulativeLast: public(uint256)
price1CumulativeLast: public(uint256)
kLast: public(uint256)  # reserve0 * reserve1 หลัง fee ล่าสุด

# Protocol fee (ไปให้ protocol treasury)
feeTo: public(address)
feeToSetter: public(address)

# Reentrancy guard
_locked: bool

@deploy
def __init__(
    _token0: address,
    _token1: address,
    _lpToken: address,
    _feeToSetter: address
):
    assert _token0 != empty(address), "Zero token0"
    assert _token1 != empty(address), "Zero token1"
    assert _token0 != _token1, "Identical tokens"
    
    self.token0 = _token0
    self.token1 = _token1
    self.lpToken = _lpToken
    self.feeToSetter = _feeToSetter
    self._locked = False

# ===== Math Helper Functions =====

@internal
@pure
def _sqrt(x: uint256) -> uint256:
    """คำนวณ square root โดยใช้ Babylonian method"""
    if x == 0:
        return 0
    
    z: uint256 = (x + 1) / 2
    y: uint256 = x
    
    for _: uint256 in range(256):
        if z >= y:
            break
        y = z
        z = (x / z + z) / 2
    
    return y

@internal
@pure
def _min(a: uint256, b: uint256) -> uint256:
    if a < b:
        return a
    return b

# ===== Reserve Management =====

@internal
def _update(
    balance0: uint256,
    balance1: uint256,
    _reserve0: uint256,
    _reserve1: uint256
):
    """อัพเดท reserves และ cumulative prices สำหรับ TWAP"""
    assert balance0 <= max_value(uint256), "Overflow"
    assert balance1 <= max_value(uint256), "Overflow"
    
    blockTimestamp: uint256 = block.timestamp
    timeElapsed: uint256 = blockTimestamp - self.blockTimestampLast
    
    if timeElapsed > 0 and _reserve0 != 0 and _reserve1 != 0:
        # UQ112x112 price accumulation
        # price0 = reserve1/reserve0, price1 = reserve0/reserve1
        self.price0CumulativeLast += (_reserve1 * 2**112 / _reserve0) * timeElapsed
        self.price1CumulativeLast += (_reserve0 * 2**112 / _reserve1) * timeElapsed
    
    self.reserve0 = balance0
    self.reserve1 = balance1
    self.blockTimestampLast = blockTimestamp
    
    log Sync(balance0, balance1)

@internal
def _mintFee(_reserve0: uint256, _reserve1: uint256) -> bool:
    """Mint protocol fee ถ้า feeTo ถูกตั้งค่า"""
    feeTo: address = self.feeTo
    feeOn: bool = feeTo != empty(address)
    _kLast: uint256 = self.kLast
    
    if feeOn:
        if _kLast != 0:
            rootK: uint256 = self._sqrt(_reserve0 * _reserve1)
            rootKLast: uint256 = self._sqrt(_kLast)
            if rootK > rootKLast:
                lp: ILPToken = ILPToken(self.lpToken)
                totalSupply: uint256 = lp.totalSupply()
                numerator: uint256 = totalSupply * (rootK - rootKLast)
                denominator: uint256 = rootK * 5 + rootKLast
                liquidity: uint256 = numerator / denominator
                if liquidity > 0:
                    lp.mint(feeTo, liquidity)
    elif _kLast != 0:
        self.kLast = 0
    
    return feeOn

# ===== Add Liquidity =====

@external
def addLiquidity(
    amount0Desired: uint256,
    amount1Desired: uint256,
    amount0Min: uint256,
    amount1Min: uint256,
    to: address,
    deadline: uint256
) -> (uint256, uint256, uint256):
    """
    เพิ่ม liquidity เข้า pool
    
    Returns:
        amount0: จำนวน token0 ที่ใช้จริง
        amount1: จำนวน token1 ที่ใช้จริง
        shares: จำนวน LP token ที่ได้รับ
    """
    assert not self._locked, "Reentrancy"
    self._locked = True
    assert block.timestamp <= deadline, "Expired"
    assert to != empty(address), "Zero address"
    
    _reserve0: uint256 = self.reserve0
    _reserve1: uint256 = self.reserve1
    
    amount0: uint256 = 0
    amount1: uint256 = 0
    
    lp: ILPToken = ILPToken(self.lpToken)
    totalSupply: uint256 = lp.totalSupply()
    
    if totalSupply == 0:
        # First liquidity provision - ใช้ทั้งหมด
        amount0 = amount0Desired
        amount1 = amount1Desired
    else:
        # คำนวณ optimal amounts โดยรักษา ratio
        amount1Optimal: uint256 = amount0Desired * _reserve1 / _reserve0
        if amount1Optimal <= amount1Desired:
            assert amount1Optimal >= amount1Min, "Slippage: token1"
            amount0 = amount0Desired
            amount1 = amount1Optimal
        else:
            amount0Optimal: uint256 = amount1Desired * _reserve0 / _reserve1
            assert amount0Optimal <= amount0Desired, "Invalid amounts"
            assert amount0Optimal >= amount0Min, "Slippage: token0"
            amount0 = amount0Optimal
            amount1 = amount1Desired
    
    # โอน tokens เข้า pool
    token0: ERC20 = ERC20(self.token0)
    token1: ERC20 = ERC20(self.token1)
    
    assert token0.transferFrom(msg.sender, self, amount0), "Transfer0 failed"
    assert token1.transferFrom(msg.sender, self, amount1), "Transfer1 failed"
    
    # Mint LP tokens
    shares: uint256 = 0
    feeOn: bool = self._mintFee(_reserve0, _reserve1)
    
    if totalSupply == 0:
        # Geometric mean ของ amounts - lock MINIMUM_LIQUIDITY ไว้ที่ address(0)
        shares = self._sqrt(amount0 * amount1) - MINIMUM_LIQUIDITY
        lp.mint(empty(address), MINIMUM_LIQUIDITY)  # Lock forever
    else:
        shares = self._min(
            amount0 * totalSupply / _reserve0,
            amount1 * totalSupply / _reserve1
        )
    
    assert shares > 0, "Insufficient liquidity minted"
    lp.mint(to, shares)
    
    # อัพเดท reserves
    balance0: uint256 = token0.balanceOf(self)
    balance1: uint256 = token1.balanceOf(self)
    self._update(balance0, balance1, _reserve0, _reserve1)
    
    if feeOn:
        self.kLast = self.reserve0 * self.reserve1
    
    log LiquidityAdded(to, amount0, amount1, shares)
    
    self._locked = False
    return amount0, amount1, shares

# ===== Remove Liquidity =====

@external
def removeLiquidity(
    shares: uint256,
    amount0Min: uint256,
    amount1Min: uint256,
    to: address,
    deadline: uint256
) -> (uint256, uint256):
    """
    ถอน liquidity ออกจาก pool
    
    Returns:
        amount0: จำนวน token0 ที่ได้คืน
        amount1: จำนวน token1 ที่ได้คืน
    """
    assert not self._locked, "Reentrancy"
    self._locked = True
    assert block.timestamp <= deadline, "Expired"
    assert shares > 0, "Zero shares"
    
    lp: ILPToken = ILPToken(self.lpToken)
    totalSupply: uint256 = lp.totalSupply()
    
    _reserve0: uint256 = self.reserve0
    _reserve1: uint256 = self.reserve1
    
    # คำนวณ amounts ตาม proportion
    amount0: uint256 = shares * _reserve0 / totalSupply
    amount1: uint256 = shares * _reserve1 / totalSupply
    
    assert amount0 >= amount0Min, "Slippage: token0"
    assert amount1 >= amount1Min, "Slippage: token1"
    
    # Burn LP tokens
    feeOn: bool = self._mintFee(_reserve0, _reserve1)
    lp.burn(msg.sender, shares)
    
    # โอน tokens ออก
    token0: ERC20 = ERC20(self.token0)
    token1: ERC20 = ERC20(self.token1)
    
    assert token0.transfer(to, amount0), "Transfer0 failed"
    assert token1.transfer(to, amount1), "Transfer1 failed"
    
    # อัพเดท reserves
    balance0: uint256 = token0.balanceOf(self)
    balance1: uint256 = token1.balanceOf(self)
    self._update(balance0, balance1, _reserve0, _reserve1)
    
    if feeOn:
        self.kLast = self.reserve0 * self.reserve1
    
    log LiquidityRemoved(to, amount0, amount1, shares)
    
    self._locked = False
    return amount0, amount1

# ===== Swap =====

@external
def swap(
    amount0Out: uint256,
    amount1Out: uint256,
    to: address,
    deadline: uint256
):
    """
    Swap tokens
    ผู้ใช้ต้องโอน tokens เข้า contract ก่อน แล้วค่อยเรียก swap
    
    Parameters:
        amount0Out: จำนวน token0 ที่ต้องการได้
        amount1Out: จำนวน token1 ที่ต้องการได้
        to: ที่อยู่ที่รับ tokens
    """
    assert not self._locked, "Reentrancy"
    self._locked = True
    assert block.timestamp <= deadline, "Expired"
    assert amount0Out > 0 or amount1Out > 0, "Insufficient output"
    
    _reserve0: uint256 = self.reserve0
    _reserve1: uint256 = self.reserve1
    
    assert amount0Out < _reserve0, "Insufficient liquidity"
    assert amount1Out < _reserve1, "Insufficient liquidity"
    
    token0: ERC20 = ERC20(self.token0)
    token1: ERC20 = ERC20(self.token1)
    
    # โอน tokens ออก
    if amount0Out > 0:
        assert token0.transfer(to, amount0Out), "Transfer failed"
    if amount1Out > 0:
        assert token1.transfer(to, amount1Out), "Transfer failed"
    
    # คำนวณ amounts ที่ได้รับ (หลังโอนออก)
    balance0: uint256 = token0.balanceOf(self)
    balance1: uint256 = token1.balanceOf(self)
    
    amount0In: uint256 = 0
    amount1In: uint256 = 0
    
    if balance0 > _reserve0 - amount0Out:
        amount0In = balance0 - (_reserve0 - amount0Out)
    if balance1 > _reserve1 - amount1Out:
        amount1In = balance1 - (_reserve1 - amount1Out)
    
    assert amount0In > 0 or amount1In > 0, "Insufficient input"
    
    # ตรวจสอบ invariant k หลัง fee
    # balance * 1000 - amountIn * 3 >= reserve * 1000
    balance0Adjusted: uint256 = balance0 * 1000 - amount0In * 3
    balance1Adjusted: uint256 = balance1 * 1000 - amount1In * 3
    
    assert balance0Adjusted * balance1Adjusted >= _reserve0 * _reserve1 * 1000000, "K invariant violated"
    
    self._update(balance0, balance1, _reserve0, _reserve1)
    
    log Swap(msg.sender, to, amount0In, amount1In, amount0Out, amount1Out)
    
    self._locked = False

# ===== Helper: Swap Exact Tokens For Tokens =====

@external
def swapExactToken0ForToken1(
    amountIn: uint256,
    amountOutMin: uint256,
    to: address,
    deadline: uint256
) -> uint256:
    """
    Swap token0 จำนวนที่กำหนด เพื่อ token1 ให้มากที่สุด
    """
    assert not self._locked, "Reentrancy"
    assert block.timestamp <= deadline, "Expired"
    
    _reserve0: uint256 = self.reserve0
    _reserve1: uint256 = self.reserve1
    
    # คำนวณ output amount
    amountOut: uint256 = self._getAmountOut(amountIn, _reserve0, _reserve1)
    assert amountOut >= amountOutMin, "Slippage exceeded"
    
    # โอน token0 เข้า
    token0: ERC20 = ERC20(self.token0)
    assert token0.transferFrom(msg.sender, self, amountIn), "Transfer failed"
    
    # Execute swap
    self._locked = True
    
    token1: ERC20 = ERC20(self.token1)
    assert token1.transfer(to, amountOut), "Transfer failed"
    
    balance0: uint256 = token0.balanceOf(self)
    balance1: uint256 = token1.balanceOf(self)
    self._update(balance0, balance1, _reserve0, _reserve1)
    
    log Swap(msg.sender, to, amountIn, 0, 0, amountOut)
    
    self._locked = False
    return amountOut

@external
def swapExactToken1ForToken0(
    amountIn: uint256,
    amountOutMin: uint256,
    to: address,
    deadline: uint256
) -> uint256:
    """
    Swap token1 จำนวนที่กำหนด เพื่อ token0 ให้มากที่สุด
    """
    assert not self._locked, "Reentrancy"
    assert block.timestamp <= deadline, "Expired"
    
    _reserve0: uint256 = self.reserve0
    _reserve1: uint256 = self.reserve1
    
    amountOut: uint256 = self._getAmountOut(amountIn, _reserve1, _reserve0)
    assert amountOut >= amountOutMin, "Slippage exceeded"
    
    token1: ERC20 = ERC20(self.token1)
    assert token1.transferFrom(msg.sender, self, amountIn), "Transfer failed"
    
    self._locked = True
    
    token0: ERC20 = ERC20(self.token0)
    assert token0.transfer(to, amountOut), "Transfer failed"
    
    balance0: uint256 = token0.balanceOf(self)
    balance1: uint256 = token1.balanceOf(self)
    self._update(balance0, balance1, _reserve0, _reserve1)
    
    log Swap(msg.sender, to, 0, amountIn, amountOut, 0)
    
    self._locked = False
    return amountOut

# ===== Price Calculation Functions =====

@internal
@pure
def _getAmountOut(
    amountIn: uint256,
    reserveIn: uint256,
    reserveOut: uint256
) -> uint256:
    """
    คำนวณ output amount จาก input amount
    สูตร: amountOut = (amountIn * 997 * reserveOut) / (reserveIn * 1000 + amountIn * 997)
    """
    assert amountIn > 0, "Insufficient input"
    assert reserveIn > 0 and reserveOut > 0, "Insufficient liquidity"
    
    amountInWithFee: uint256 = amountIn * FEE_NUMERATOR
    numerator: uint256 = amountInWithFee * reserveOut
    denominator: uint256 = reserveIn * FEE_DENOMINATOR + amountInWithFee
    
    return numerator / denominator

@internal
@pure
def _getAmountIn(
    amountOut: uint256,
    reserveIn: uint256,
    reserveOut: uint256
) -> uint256:
    """
    คำนวณ input amount ที่ต้องการจาก output amount
    สูตร: amountIn = (reserveIn * amountOut * 1000) / ((reserveOut - amountOut) * 997) + 1
    """
    assert amountOut > 0, "Insufficient output"
    assert reserveIn > 0 and reserveOut > 0, "Insufficient liquidity"
    assert amountOut < reserveOut, "Insufficient liquidity"
    
    numerator: uint256 = reserveIn * amountOut * FEE_DENOMINATOR
    denominator: uint256 = (reserveOut - amountOut) * FEE_NUMERATOR
    
    return numerator / denominator + 1

@external
@view
def getAmountOut(amountIn: uint256, tokenIn: address) -> uint256:
    """View function: คำนวณ output amount"""
    if tokenIn == self.token0:
        return self._getAmountOut(amountIn, self.reserve0, self.reserve1)
    else:
        return self._getAmountOut(amountIn, self.reserve1, self.reserve0)

@external
@view
def getAmountIn(amountOut: uint256, tokenOut: address) -> uint256:
    """View function: คำนวณ input amount"""
    if tokenOut == self.token1:
        return self._getAmountIn(amountOut, self.reserve0, self.reserve1)
    else:
        return self._getAmountIn(amountOut, self.reserve1, self.reserve0)

# ===== Price Impact Calculation =====

@external
@view
def getPriceImpact(amountIn: uint256, tokenIn: address) -> uint256:
    """
    คำนวณ price impact เป็น basis points (1 bp = 0.01%)
    
    Price impact = (midPrice - executionPrice) / midPrice * 10000
    """
    _reserve0: uint256 = self.reserve0
    _reserve1: uint256 = self.reserve1
    
    midPrice: uint256 = 0
    amountOut: uint256 = 0
    
    if tokenIn == self.token0:
        # ราคากลาง: reserve1/reserve0
        midPrice = _reserve1 * 10**18 / _reserve0
        amountOut = self._getAmountOut(amountIn, _reserve0, _reserve1)
        # execution price: amountOut/amountIn
        executionPrice: uint256 = amountOut * 10**18 / amountIn
        if midPrice > executionPrice:
            return (midPrice - executionPrice) * 10000 / midPrice
    else:
        midPrice = _reserve0 * 10**18 / _reserve1
        amountOut = self._getAmountOut(amountIn, _reserve1, _reserve0)
        executionPrice: uint256 = amountOut * 10**18 / amountIn
        if midPrice > executionPrice:
            return (midPrice - executionPrice) * 10000 / midPrice
    
    return 0

# ===== TWAP Oracle =====

@external
@view
def getSpotPrice(token: address) -> uint256:
    """
    ราคาปัจจุบัน (spot price) ไม่ควรใช้เป็น oracle เพราะ manipulable
    """
    if token == self.token0:
        return self.reserve1 * 10**18 / self.reserve0
    else:
        return self.reserve0 * 10**18 / self.reserve1

@external
@view  
def getReserves() -> (uint256, uint256, uint256):
    """Get current reserves"""
    return self.reserve0, self.reserve1, self.blockTimestampLast

# ===== Fee Management =====

@external
def setFeeTo(feeTo: address):
    assert msg.sender == self.feeToSetter, "Forbidden"
    self.feeTo = feeTo

@external
def setFeeToSetter(feeToSetter: address):
    assert msg.sender == self.feeToSetter, "Forbidden"
    self.feeToSetter = feeToSetter

# ===== Flash Swap =====

interface IFlashSwapCallback:
    def flashSwapCallback(
        amount0: uint256,
        amount1: uint256,
        data: Bytes[1024]
    ): nonpayable

@external
def flashSwap(
    amount0Out: uint256,
    amount1Out: uint256,
    to: address,
    data: Bytes[1024],
    deadline: uint256
):
    """
    Flash Swap - ยืม tokens โดยไม่ต้องมี collateral
    ต้องคืนภายใน transaction เดียวกัน พร้อม fee
    """
    assert not self._locked, "Reentrancy"
    self._locked = True
    assert block.timestamp <= deadline, "Expired"
    assert amount0Out > 0 or amount1Out > 0, "Insufficient output"
    
    _reserve0: uint256 = self.reserve0
    _reserve1: uint256 = self.reserve1
    
    assert amount0Out < _reserve0, "Insufficient liquidity"
    assert amount1Out < _reserve1, "Insufficient liquidity"
    
    token0: ERC20 = ERC20(self.token0)
    token1: ERC20 = ERC20(self.token1)
    
    # โอน tokens ไปให้ผู้ยืม
    if amount0Out > 0:
        assert token0.transfer(to, amount0Out), "Transfer failed"
    if amount1Out > 0:
        assert token1.transfer(to, amount1Out), "Transfer failed"
    
    # เรียก callback ของผู้ยืม
    if len(data) > 0:
        IFlashSwapCallback(to).flashSwapCallback(amount0Out, amount1Out, data)
    
    # ตรวจสอบว่าคืน tokens แล้ว
    balance0: uint256 = token0.balanceOf(self)
    balance1: uint256 = token1.balanceOf(self)
    
    amount0In: uint256 = 0
    amount1In: uint256 = 0
    
    if balance0 > _reserve0 - amount0Out:
        amount0In = balance0 - (_reserve0 - amount0Out)
    if balance1 > _reserve1 - amount1Out:
        amount1In = balance1 - (_reserve1 - amount1Out)
    
    assert amount0In > 0 or amount1In > 0, "Insufficient repayment"
    
    balance0Adjusted: uint256 = balance0 * 1000 - amount0In * 3
    balance1Adjusted: uint256 = balance1 * 1000 - amount1In * 3
    
    assert balance0Adjusted * balance1Adjusted >= _reserve0 * _reserve1 * 1000000, "K invariant violated"
    
    self._update(balance0, balance1, _reserve0, _reserve1)
    
    log FlashSwap(to, amount0Out, amount1Out)
    
    self._locked = False
```

---

## 4. Router Contract

Router ทำหน้าที่เป็นตัวกลางระหว่าง user กับ AMM pairs

```vyper
# @version 0.4.0
# contracts/Router.vy
# Router สำหรับ AMM - จัดการ multi-hop swaps และ liquidity

from vyper.interfaces import ERC20

interface IPair:
    def token0() -> address: view
    def token1() -> address: view
    def getReserves() -> (uint256, uint256, uint256): view
    def swap(amount0Out: uint256, amount1Out: uint256, to: address, deadline: uint256): nonpayable
    def addLiquidity(amount0Desired: uint256, amount1Desired: uint256, amount0Min: uint256, amount1Min: uint256, to: address, deadline: uint256) -> (uint256, uint256, uint256): nonpayable
    def removeLiquidity(shares: uint256, amount0Min: uint256, amount1Min: uint256, to: address, deadline: uint256) -> (uint256, uint256): nonpayable

interface IFactory:
    def getPair(tokenA: address, tokenB: address) -> address: view
    def createPair(tokenA: address, tokenB: address) -> address: nonpayable

# Events
event SwapExecuted:
    sender: indexed(address)
    amountIn: uint256
    amountOut: uint256
    path: DynArray[address, 10]

# State
factory: public(address)
WETH: public(address)

@deploy
def __init__(_factory: address, _weth: address):
    self.factory = _factory
    self.WETH = _weth

@internal
@view
def _getAmountOut(
    amountIn: uint256,
    reserveIn: uint256,
    reserveOut: uint256
) -> uint256:
    amountInWithFee: uint256 = amountIn * 997
    numerator: uint256 = amountInWithFee * reserveOut
    denominator: uint256 = reserveIn * 1000 + amountInWithFee
    return numerator / denominator

@external
@view
def getAmountsOut(amountIn: uint256, path: DynArray[address, 10]) -> DynArray[uint256, 10]:
    """
    คำนวณ output amounts สำหรับ multi-hop swap
    """
    assert len(path) >= 2, "Invalid path"
    amounts: DynArray[uint256, 10] = []
    amounts.append(amountIn)
    
    for i: uint256 in range(9):
        if i >= len(path) - 1:
            break
        pair: address = IFactory(self.factory).getPair(path[i], path[i+1])
        assert pair != empty(address), "Pair not found"
        reserve0: uint256 = 0
        reserve1: uint256 = 0
        timestamp: uint256 = 0
        reserve0, reserve1, timestamp = IPair(pair).getReserves()
        token0: address = IPair(pair).token0()
        reserveIn: uint256 = 0
        reserveOut: uint256 = 0
        if path[i] == token0:
            reserveIn = reserve0
            reserveOut = reserve1
        else:
            reserveIn = reserve1
            reserveOut = reserve0
        amounts.append(self._getAmountOut(amounts[i], reserveIn, reserveOut))
    
    return amounts

@external
def swapExactTokensForTokens(
    amountIn: uint256,
    amountOutMin: uint256,
    path: DynArray[address, 10],
    to: address,
    deadline: uint256
) -> DynArray[uint256, 10]:
    """
    Swap exact input tokens ผ่าน path ที่กำหนด
    """
    assert block.timestamp <= deadline, "Expired"
    
    amounts: DynArray[uint256, 10] = self.getAmountsOut(amountIn, path)
    assert amounts[len(amounts) - 1] >= amountOutMin, "Slippage exceeded"
    
    # โอน input token เข้า first pair
    firstPair: address = IFactory(self.factory).getPair(path[0], path[1])
    assert ERC20(path[0]).transferFrom(msg.sender, firstPair, amounts[0]), "Transfer failed"
    
    # Execute swaps ผ่าน pairs
    for i: uint256 in range(9):
        if i >= len(path) - 1:
            break
        
        input_token: address = path[i]
        output_token: address = path[i+1]
        pair: address = IFactory(self.factory).getPair(input_token, output_token)
        
        # กำหนด to address (next pair หรือ final recipient)
        _to: address = to
        if i < len(path) - 2:
            _to = IFactory(self.factory).getPair(path[i+1], path[i+2])
        
        token0: address = IPair(pair).token0()
        amount0Out: uint256 = 0
        amount1Out: uint256 = 0
        
        if output_token == token0:
            amount0Out = amounts[i+1]
        else:
            amount1Out = amounts[i+1]
        
        IPair(pair).swap(amount0Out, amount1Out, _to, deadline)
    
    log SwapExecuted(msg.sender, amountIn, amounts[len(amounts) - 1], path)
    
    return amounts
```

---

## 5. Factory Contract

```vyper
# @version 0.4.0
# contracts/Factory.vy
# Factory สำหรับสร้าง AMM Pairs

# Events
event PairCreated:
    token0: indexed(address)
    token1: indexed(address)
    pair: address
    pairCount: uint256

# State
getPair: public(HashMap[address, HashMap[address, address]])
allPairs: public(DynArray[address, 10000])
feeTo: public(address)
feeToSetter: public(address)

pairTemplate: address  # Template สำหรับ create pair

@deploy
def __init__(_feeToSetter: address):
    self.feeToSetter = _feeToSetter

@external
@view
def allPairsLength() -> uint256:
    return len(self.allPairs)

@external
def createPair(tokenA: address, tokenB: address) -> address:
    """
    สร้าง pair ใหม่ระหว่าง tokenA และ tokenB
    """
    assert tokenA != tokenB, "Identical tokens"
    assert tokenA != empty(address) and tokenB != empty(address), "Zero address"
    assert self.getPair[tokenA][tokenB] == empty(address), "Pair exists"
    
    # Sort tokens
    token0: address = tokenA
    token1: address = tokenB
    if convert(tokenA, uint256) > convert(tokenB, uint256):
        token0 = tokenB
        token1 = tokenA
    
    # In production: deploy new pair contract
    # For simplicity, using a placeholder
    # pair: address = create_from_blueprint(self.pairTemplate, token0, token1, self.feeTo, self.feeToSetter)
    
    # Store pair
    # self.getPair[token0][token1] = pair
    # self.getPair[token1][token0] = pair
    # self.allPairs.append(pair)
    
    # log PairCreated(token0, token1, pair, len(self.allPairs))
    
    # return pair
    return empty(address)  # Placeholder

@external
def setFeeTo(feeTo: address):
    assert msg.sender == self.feeToSetter, "Forbidden"
    self.feeTo = feeTo

@external
def setFeeToSetter(feeToSetter: address):
    assert msg.sender == self.feeToSetter, "Forbidden"
    self.feeToSetter = feeToSetter
```

---

## 6. ตัวอย่างการใช้งาน (Advanced AMM Features)

### Impermanent Loss Calculator

```vyper
# @version 0.4.0
# contracts/ILCalculator.vy
# คำนวณ Impermanent Loss

@external
@pure
def calculateImpermanentLoss(
    priceRatioChange: uint256  # เปลี่ยนแปลงราคา เป็น % * 100 (เช่น 200 = ราคาเพิ่ม 2x)
) -> uint256:
    """
    คำนวณ impermanent loss เป็น basis points
    
    IL = 2 * sqrt(priceRatio) / (1 + priceRatio) - 1
    
    Parameters:
        priceRatioChange: ราคาเปลี่ยนเป็น ratio * 10000 (10000 = ไม่เปลี่ยน, 20000 = เพิ่ม 2x)
    
    Returns:
        Impermanent loss เป็น basis points
    """
    # Simplified approximation
    # For exact calculation need fixed-point math library
    
    if priceRatioChange == 10000:
        return 0  # ไม่มี IL เมื่อราคาไม่เปลี่ยน
    
    # IL approximation using integer arithmetic
    # IL ≈ (sqrt(k) - 1)^2 / (sqrt(k) + 1)^2 * 10000
    # where k = new_price/old_price
    
    return 0  # Simplified placeholder
```

---

## 7. Tests

```python
# tests/test_amm.py
import pytest
from brownie import (
    LPToken,
    UniswapV2Pair,
    MockERC20,
    accounts,
    chain
)

@pytest.fixture
def setup():
    owner = accounts[0]
    alice = accounts[1]
    bob = accounts[2]
    
    # Deploy mock tokens
    token0 = MockERC20.deploy("Token A", "TKNA", 18, {"from": owner})
    token1 = MockERC20.deploy("Token B", "TKNB", 18, {"from": owner})
    
    # Sort tokens
    if token0.address > token1.address:
        token0, token1 = token1, token0
    
    # Deploy LP token
    lp_token = LPToken.deploy("LP Token", "LP", {"from": owner})
    
    # Deploy pair
    pair = UniswapV2Pair.deploy(
        token0.address,
        token1.address,
        lp_token.address,
        owner.address,
        {"from": owner}
    )
    
    # Set minter
    lp_token.set_minter(pair.address, {"from": owner})
    
    # Mint tokens
    token0.mint(alice, 10**24, {"from": owner})
    token1.mint(alice, 10**24, {"from": owner})
    token0.mint(bob, 10**24, {"from": owner})
    token1.mint(bob, 10**24, {"from": owner})
    
    return owner, alice, bob, token0, token1, lp_token, pair

def test_add_initial_liquidity(setup):
    owner, alice, bob, token0, token1, lp_token, pair = setup
    
    amount0 = 10**21  # 1000 tokens
    amount1 = 10**21  # 1000 tokens
    deadline = chain.time() + 3600
    
    # Approve
    token0.approve(pair.address, amount0, {"from": alice})
    token1.approve(pair.address, amount1, {"from": alice})
    
    # Add liquidity
    tx = pair.addLiquidity(
        amount0,
        amount1,
        0,  # no slippage protection for first add
        0,
        alice.address,
        deadline,
        {"from": alice}
    )
    
    actual0, actual1, shares = tx.return_value
    
    assert actual0 == amount0
    assert actual1 == amount1
    assert shares > 0
    
    # Check LP token balance
    lp_balance = lp_token.balanceOf(alice.address)
    assert lp_balance == shares

def test_swap(setup):
    owner, alice, bob, token0, token1, lp_token, pair = setup
    
    # First add liquidity
    amount0 = 10**21
    amount1 = 10**21
    deadline = chain.time() + 3600
    
    token0.approve(pair.address, amount0, {"from": alice})
    token1.approve(pair.address, amount1, {"from": alice})
    pair.addLiquidity(amount0, amount1, 0, 0, alice.address, deadline, {"from": alice})
    
    # Now swap: bob swaps token0 for token1
    swap_amount = 10**19  # 10 tokens
    
    # Calculate expected output
    expected_out = pair.getAmountOut(swap_amount, token0.address)
    
    # Approve and swap
    token0.approve(pair.address, swap_amount, {"from": bob})
    
    bob_token1_before = token1.balanceOf(bob.address)
    
    pair.swapExactToken0ForToken1(
        swap_amount,
        expected_out * 99 // 100,  # 1% slippage
        bob.address,
        deadline,
        {"from": bob}
    )
    
    bob_token1_after = token1.balanceOf(bob.address)
    
    assert bob_token1_after - bob_token1_before >= expected_out * 99 // 100

def test_remove_liquidity(setup):
    owner, alice, bob, token0, token1, lp_token, pair = setup
    
    amount0 = 10**21
    amount1 = 10**21
    deadline = chain.time() + 3600
    
    token0.approve(pair.address, amount0, {"from": alice})
    token1.approve(pair.address, amount1, {"from": alice})
    _, _, shares = pair.addLiquidity(
        amount0, amount1, 0, 0, alice.address, deadline, {"from": alice}
    ).return_value
    
    # Remove half of liquidity
    remove_shares = shares // 2
    lp_token.approve(pair.address, remove_shares, {"from": alice})
    
    alice_t0_before = token0.balanceOf(alice.address)
    alice_t1_before = token1.balanceOf(alice.address)
    
    pair.removeLiquidity(remove_shares, 0, 0, alice.address, deadline, {"from": alice})
    
    alice_t0_after = token0.balanceOf(alice.address)
    alice_t1_after = token1.balanceOf(alice.address)
    
    assert alice_t0_after > alice_t0_before
    assert alice_t1_after > alice_t1_before

def test_price_impact(setup):
    owner, alice, bob, token0, token1, lp_token, pair = setup
    
    # Add liquidity
    amount0 = 10**21
    amount1 = 10**21
    deadline = chain.time() + 3600
    
    token0.approve(pair.address, amount0, {"from": alice})
    token1.approve(pair.address, amount1, {"from": alice})
    pair.addLiquidity(amount0, amount1, 0, 0, alice.address, deadline, {"from": alice})
    
    # Small swap - low price impact
    small_swap = 10**18  # 1 token (0.1% of pool)
    small_impact = pair.getPriceImpact(small_swap, token0.address)
    
    # Large swap - high price impact
    large_swap = 10**20  # 100 tokens (10% of pool)
    large_impact = pair.getPriceImpact(large_swap, token0.address)
    
    assert large_impact > small_impact
    print(f"Small swap impact: {small_impact} bps ({small_impact/100}%)")
    print(f"Large swap impact: {large_impact} bps ({large_impact/100}%)")

def test_k_invariant(setup):
    """ตรวจสอบว่า k invariant ถูกรักษาไว้"""
    owner, alice, bob, token0, token1, lp_token, pair = setup
    
    amount0 = 10**21
    amount1 = 2 * 10**21  # ratio 1:2
    deadline = chain.time() + 3600
    
    token0.approve(pair.address, amount0, {"from": alice})
    token1.approve(pair.address, amount1, {"from": alice})
    pair.addLiquidity(amount0, amount1, 0, 0, alice.address, deadline, {"from": alice})
    
    r0_before, r1_before, _ = pair.getReserves()
    k_before = r0_before * r1_before
    
    # Swap
    swap_amount = 10**19
    token0.approve(pair.address, swap_amount, {"from": bob})
    pair.swapExactToken0ForToken1(swap_amount, 0, bob.address, deadline, {"from": bob})
    
    r0_after, r1_after, _ = pair.getReserves()
    k_after = r0_after * r1_after
    
    # k ต้องเพิ่มขึ้นหรือเท่าเดิม (เพิ่มขึ้นเพราะ fee)
    assert k_after >= k_before, "K invariant violated"
    print(f"K before: {k_before}")
    print(f"K after:  {k_after}")
    print(f"K increase: {(k_after - k_before) * 100 / k_before:.4f}%")
```

---

## 8. สรุปและ Best Practices

### ประเด็นสำคัญที่ต้องรู้:

**1. Minimum Liquidity Lock**
- Lock 1000 wei LP tokens ที่ address(0) ป้องกัน inflation attack
- ไม่สามารถถอนออกได้ตลอดไป

**2. ค่า Fee**
- 0.3% ต่อ swap (997/1000)
- 1/6 ของ fee ไปที่ protocol (ถ้าเปิดใช้)

**3. TWAP Oracle**
- ใช้ cumulative prices แทน spot price
- ป้องกัน price manipulation

**4. Slippage Protection**
- ตั้ง `amountOutMin` ที่เหมาะสม
- ใช้ `deadline` ป้องกัน stale transactions

**5. Reentrancy Guard**
- ใช้ `_locked` boolean ป้องกัน reentrancy
- Vyper มี built-in protection แต่ก็ดีที่ทำ explicit

**6. Price Impact vs Slippage**
- Price impact = ผลกระทบต่อราคาจากขนาด trade
- Slippage = ความแตกต่างระหว่างราคาที่คาดหวังกับที่ได้จริง
- ทั้งสองมีผลต่อ profitability ของการ trade

```
# การคำนวณ optimal trade size
# สำหรับ trade ที่มี price impact น้อยกว่า 1%:
# trade_size < 0.01 * sqrt(reserve0 * reserve1)
# ≈ 1% ของ pool size
```
