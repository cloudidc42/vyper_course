# Part 064: MEV Protection - Sandwich Attacks, Slippage, and Private Mempools

## สารบัญ (Table of Contents)
1. บทนำ MEV
2. Commit-Reveal Scheme
3. TWAMM (Time-Weighted AMM)
4. Slippage Protection
5. Anti-Sandwich Measures
6. Tests

---

## 1. บทนำ MEV (Maximal Extractable Value)

**ประเภท MEV:**
- **Sandwich Attack**: Front-run + back-run user swap
- **Arbitrage**: ใช้ราคาต่างระหว่าง DEX
- **Liquidation**: แข่งกัน liquidate position

**วิธีป้องกัน:**
- Commit-Reveal: ซ่อน intent ก่อน execute
- TWAMM: แบ่ง order ขนาดใหญ่เป็น order เล็กๆ
- Private Mempool: ส่ง tx ตรงให้ validator (Flashbots)
- Slippage Limits: จำกัด max slippage

---

## 2. Commit-Reveal Swap

```vyper
# @version 0.4.0
# contracts/CommitRevealSwap.vy
# ซ่อน swap intent โดย commit hash ก่อน reveal

from vyper.interfaces import ERC20

interface IUniswapV2Router:
    def swapExactTokensForTokens(
        amountIn: uint256,
        amountOutMin: uint256,
        path: DynArray[address, 5],
        to: address,
        deadline: uint256
    ) -> DynArray[uint256, 5]: nonpayable

# Events
event Committed:
    commitId: indexed(bytes32)
    trader: indexed(address)
    revealDeadline: uint256

event Revealed:
    commitId: indexed(bytes32)
    trader: indexed(address)
    amountIn: uint256
    amountOutMin: uint256

event SwapExecuted:
    commitId: indexed(bytes32)
    amountIn: uint256
    amountOut: uint256

struct Commitment:
    trader: address
    commitHash: bytes32       # keccak256(tokenIn, tokenOut, amountIn, amountOutMin, nonce, secret)
    depositedAmount: uint256  # pre-deposit tokenIn
    tokenIn: address
    revealDeadline: uint256   # ต้อง reveal ก่อน deadline
    executed: bool
    cancelled: bool

# State
commitments: HashMap[bytes32, Commitment]
router: public(address)
commitDelay: public(uint256)   # minimum blocks ก่อน reveal
revealWindow: public(uint256)  # blocks ที่มีให้ reveal
governance: public(address)

@deploy
def __init__(_router: address, _governance: address):
    self.router = _router
    self.governance = _governance
    self.commitDelay = 2      # รอ 2 blocks หลัง commit
    self.revealWindow = 20    # reveal ภายใน 20 blocks

@external
def commit(
    commitHash: bytes32,
    tokenIn: address,
    amountIn: uint256
) -> bytes32:
    """
    Step 1: Commit to a swap โดยไม่เปิดเผย details
    
    commitHash = keccak256(abi.encode(tokenIn, tokenOut, amountIn, amountOutMin, nonce, secret))
    
    Pre-deposit token เพื่อป้องกัน front-running กับ approval
    
    Parameters:
        commitHash: hash ของ swap parameters
        tokenIn: token ที่จะขาย
        amountIn: จำนวนที่จะขาย
    
    Returns: commitId
    """
    assert amountIn > 0, "Zero amount"
    
    # ดึง token ก่อน reveal
    assert ERC20(tokenIn).transferFrom(msg.sender, self, amountIn), "Transfer failed"
    
    commitId: bytes32 = keccak256(abi.encode(msg.sender, commitHash, block.number))
    
    assert self.commitments[commitId].trader == empty(address), "Commit exists"
    
    revealDeadline: uint256 = block.number + self.commitDelay + self.revealWindow
    
    self.commitments[commitId] = Commitment({
        trader: msg.sender,
        commitHash: commitHash,
        depositedAmount: amountIn,
        tokenIn: tokenIn,
        revealDeadline: revealDeadline,
        executed: False,
        cancelled: False
    })
    
    log Committed(commitId, msg.sender, revealDeadline)
    
    return commitId

@external
def reveal(
    commitId: bytes32,
    tokenOut: address,
    amountOutMin: uint256,
    path: DynArray[address, 5],
    nonce: uint256,
    secret: bytes32
) -> uint256:
    """
    Step 2: Reveal swap details และ execute
    
    ต้อง reveal หลัง commitDelay blocks
    และก่อน revealDeadline
    
    Parameters:
        tokenOut: token ที่ต้องการรับ
        amountOutMin: minimum output
        path: swap path
        nonce: ตัวเลข random
        secret: secret ที่ใช้ใน commit hash
    
    Returns: amountOut
    """
    commitment: Commitment = self.commitments[commitId]
    
    assert commitment.trader == msg.sender, "Not trader"
    assert not commitment.executed, "Already executed"
    assert not commitment.cancelled, "Cancelled"
    assert block.number <= commitment.revealDeadline, "Reveal window expired"
    assert block.number >= commitment.revealDeadline - self.revealWindow, "Too early"
    
    # ตรวจ hash
    expectedHash: bytes32 = keccak256(
        abi.encode(
            commitment.tokenIn,
            tokenOut,
            commitment.depositedAmount,
            amountOutMin,
            nonce,
            secret
        )
    )
    assert expectedHash == commitment.commitHash, "Hash mismatch"
    
    self.commitments[commitId].executed = True
    
    # Execute swap
    assert ERC20(commitment.tokenIn).approve(self.router, commitment.depositedAmount), "Approve failed"
    
    amounts: DynArray[uint256, 5] = IUniswapV2Router(self.router).swapExactTokensForTokens(
        commitment.depositedAmount,
        amountOutMin,
        path,
        msg.sender,
        block.timestamp + 300
    )
    
    amountOut: uint256 = amounts[len(amounts) - 1]
    
    log Revealed(commitId, msg.sender, commitment.depositedAmount, amountOutMin)
    log SwapExecuted(commitId, commitment.depositedAmount, amountOut)
    
    return amountOut

@external
def cancelCommit(commitId: bytes32):
    """Cancel commit และรับ token คืน (ถ้า reveal window หมดแล้ว)"""
    commitment: Commitment = self.commitments[commitId]
    
    assert commitment.trader == msg.sender, "Not trader"
    assert not commitment.executed, "Already executed"
    assert not commitment.cancelled, "Already cancelled"
    assert block.number > commitment.revealDeadline, "Window not expired"
    
    self.commitments[commitId].cancelled = True
    
    assert ERC20(commitment.tokenIn).transfer(msg.sender, commitment.depositedAmount), "Refund failed"
```

---

## 3. TWAMM (Time-Weighted AMM)

```vyper
# @version 0.4.0
# contracts/TWAMM.vy
# Long-term orders แบ่งเป็น virtual sub-orders ตามเวลา

from vyper.interfaces import ERC20

interface IUniswapV2Pair:
    def getReserves() -> (uint112, uint112, uint32): view
    def swap(amount0Out: uint256, amount1Out: uint256, to: address, data: Bytes[1024]): nonpayable

# Events
event LongOrderCreated:
    orderId: indexed(uint256)
    trader: indexed(address)
    sellingToken: address
    buyingToken: address
    amountPerInterval: uint256
    numberOfIntervals: uint256

event OrderExecuted:
    orderId: indexed(uint256)
    intervalsExecuted: uint256
    amountOut: uint256

event OrderCancelled:
    orderId: indexed(uint256)
    refundAmount: uint256

struct LongTermOrder:
    trader: address
    sellingToken: address
    buyingToken: address
    amountPerInterval: uint256  # จำนวนขายต่อ interval
    totalIntervals: uint256
    executedIntervals: uint256
    lastExecutionBlock: uint256
    creationBlock: uint256
    cancelled: bool

struct TWAMMState:
    lastVirtualOrderBlock: uint256
    # Virtual order rates
    token0SellRate: uint256
    token1SellRate: uint256

# State
orders: HashMap[uint256, LongTermOrder]
nextOrderId: uint256
pair: public(address)      # Uniswap V2 pair
token0: public(address)
token1: public(address)

twammState: TWAMMState
orderInterval: uint256     # blocks ระหว่าง execution

# User balances ที่รอ claim
pendingBalances: HashMap[address, HashMap[address, uint256]]

governance: public(address)

SCALE: constant(uint256) = 10**18

@deploy
def __init__(_pair: address, _token0: address, _token1: address, _interval: uint256, _governance: address):
    self.pair = _pair
    self.token0 = _token0
    self.token1 = _token1
    self.orderInterval = _interval
    self.governance = _governance
    self.twammState = TWAMMState({
        lastVirtualOrderBlock: block.number,
        token0SellRate: 0,
        token1SellRate: 0
    })

@external
def createLongTermOrder(
    sellingToken: address,
    buyingToken: address,
    totalAmount: uint256,
    numberOfIntervals: uint256
) -> uint256:
    """
    สร้าง long-term order ที่จะ execute ทีละ interval
    
    ตัวอย่าง: ขาย 10,000 USDC ใน 100 intervals
    -> ขาย 100 USDC ต่อ interval
    
    ป้องกัน price impact จาก large single trade
    
    Parameters:
        totalAmount: ยอดรวมที่ต้องการขาย
        numberOfIntervals: แบ่งเป็นกี่ interval
    
    Returns: orderId
    """
    assert sellingToken == self.token0 or sellingToken == self.token1, "Invalid token"
    assert numberOfIntervals > 0, "Zero intervals"
    assert totalAmount > 0, "Zero amount"
    
    amountPerInterval: uint256 = totalAmount / numberOfIntervals
    assert amountPerInterval > 0, "Amount too small"
    
    # Collect total amount
    assert ERC20(sellingToken).transferFrom(msg.sender, self, totalAmount), "Transfer failed"
    
    orderId: uint256 = self.nextOrderId
    self.nextOrderId += 1
    
    self.orders[orderId] = LongTermOrder({
        trader: msg.sender,
        sellingToken: sellingToken,
        buyingToken: buyingToken,
        amountPerInterval: amountPerInterval,
        totalIntervals: numberOfIntervals,
        executedIntervals: 0,
        lastExecutionBlock: block.number,
        creationBlock: block.number,
        cancelled: False
    })
    
    # อัพเดท sell rate
    if sellingToken == self.token0:
        self.twammState.token0SellRate += amountPerInterval
    else:
        self.twammState.token1SellRate += amountPerInterval
    
    log LongTermOrderCreated(
        orderId, msg.sender, sellingToken, buyingToken,
        amountPerInterval, numberOfIntervals
    )
    
    return orderId

@external
def executeVirtualOrders(maxOrders: uint256):
    """
    Execute pending virtual orders ตาม time-weighted schedule
    
    ใครก็สามารถ call ได้ (incentivized)
    เป็น key mechanism ที่ทำให้ TWAMM ทำงาน
    
    Parameters:
        maxOrders: จำนวน orders สูงสุดที่ process
    """
    blocksElapsed: uint256 = block.number - self.twammState.lastVirtualOrderBlock
    if blocksElapsed < self.orderInterval:
        return
    
    intervalsToExecute: uint256 = blocksElapsed / self.orderInterval
    
    ordersProcessed: uint256 = 0
    
    for orderId: uint256 in range(1000):
        if orderId >= self.nextOrderId or ordersProcessed >= maxOrders:
            break
        
        order: LongTermOrder = self.orders[orderId]
        if order.cancelled or order.executedIntervals >= order.totalIntervals:
            continue
        
        # คำนวณ intervals ที่ต้อง execute
        orderIntervalsElapsed: uint256 = (block.number - order.lastExecutionBlock) / self.orderInterval
        intervalsForOrder: uint256 = min(
            orderIntervalsElapsed,
            order.totalIntervals - order.executedIntervals
        )
        
        if intervalsForOrder == 0:
            continue
        
        # Execute swap สำหรับ intervals นี้
        swapAmount: uint256 = order.amountPerInterval * intervalsForOrder
        
        # Swap on Uniswap
        amountOut: uint256 = self._executeSwap(
            order.sellingToken,
            order.buyingToken,
            swapAmount
        )
        
        # เพิ่ม output เข้า pending balance
        self.pendingBalances[order.trader][order.buyingToken] += amountOut
        
        self.orders[orderId].executedIntervals += intervalsForOrder
        self.orders[orderId].lastExecutionBlock = block.number
        
        ordersProcessed += 1
        
        log OrderExecuted(orderId, intervalsForOrder, amountOut)
    
    self.twammState.lastVirtualOrderBlock = block.number

@internal
def _executeSwap(tokenIn: address, tokenOut: address, amountIn: uint256) -> uint256:
    """Execute swap บน Uniswap V2"""
    assert ERC20(tokenIn).approve(self.pair, amountIn), "Approve failed"
    
    reserves0: uint112 = 0
    reserves1: uint112 = 0
    _: uint32 = 0
    reserves0, reserves1, _ = IUniswapV2Pair(self.pair).getReserves()
    
    amountOut: uint256 = 0
    amount0Out: uint256 = 0
    amount1Out: uint256 = 0
    
    if tokenIn == self.token0:
        amountOut = self._getAmountOut(amountIn, convert(reserves0, uint256), convert(reserves1, uint256))
        amount1Out = amountOut
    else:
        amountOut = self._getAmountOut(amountIn, convert(reserves1, uint256), convert(reserves0, uint256))
        amount0Out = amountOut
    
    assert ERC20(tokenIn).transfer(self.pair, amountIn), "Token transfer failed"
    IUniswapV2Pair(self.pair).swap(amount0Out, amount1Out, self, b"")
    
    return amountOut

@internal
@pure
def _getAmountOut(amountIn: uint256, reserveIn: uint256, reserveOut: uint256) -> uint256:
    amountInWithFee: uint256 = amountIn * 997
    numerator: uint256 = amountInWithFee * reserveOut
    denominator: uint256 = reserveIn * 1000 + amountInWithFee
    return numerator / denominator

@external
def withdrawProceeds(token: address) -> uint256:
    """Withdraw accumulated swap proceeds"""
    amount: uint256 = self.pendingBalances[msg.sender][token]
    assert amount > 0, "Nothing to withdraw"
    
    self.pendingBalances[msg.sender][token] = 0
    
    assert ERC20(token).transfer(msg.sender, amount), "Transfer failed"
    
    return amount

@external
def cancelOrder(orderId: uint256) -> uint256:
    """Cancel order และรับ unexecuted tokens คืน"""
    order: LongTermOrder = self.orders[orderId]
    
    assert order.trader == msg.sender, "Not trader"
    assert not order.cancelled, "Already cancelled"
    
    self.orders[orderId].cancelled = True
    
    # คืน unexecuted portion
    remainingIntervals: uint256 = order.totalIntervals - order.executedIntervals
    refundAmount: uint256 = order.amountPerInterval * remainingIntervals
    
    if refundAmount > 0:
        assert ERC20(order.sellingToken).transfer(msg.sender, refundAmount), "Refund failed"
    
    # ลด sell rate
    if order.sellingToken == self.token0:
        if self.twammState.token0SellRate >= order.amountPerInterval:
            self.twammState.token0SellRate -= order.amountPerInterval
    else:
        if self.twammState.token1SellRate >= order.amountPerInterval:
            self.twammState.token1SellRate -= order.amountPerInterval
    
    log OrderCancelled(orderId, refundAmount)
    
    return refundAmount
```

---

## 4. Slippage Protection Router

```vyper
# @version 0.4.0
# contracts/SlippageProtectedRouter.vy
# Router ที่มี slippage protection และ MEV resistance

from vyper.interfaces import ERC20

interface IUniswapV2Router:
    def swapExactTokensForTokens(
        amountIn: uint256,
        amountOutMin: uint256,
        path: DynArray[address, 5],
        to: address,
        deadline: uint256
    ) -> DynArray[uint256, 5]: nonpayable
    def getAmountsOut(
        amountIn: uint256,
        path: DynArray[address, 5]
    ) -> DynArray[uint256, 5]: view

# Events
event SwapProtected:
    trader: indexed(address)
    amountIn: uint256
    amountOut: uint256
    slippage: uint256

event SandwichDetected:
    trader: indexed(address)
    expectedOut: uint256
    actualOut: uint256

# State
router: public(address)
governance: public(address)

# Anti-sandwich: track price per block
blockPriceSnapshot: HashMap[uint256, HashMap[bytes32, uint256]]  # block -> pairKey -> price

# Max slippage per address (custom limits)
userMaxSlippage: HashMap[address, uint256]  # basis points, 0 = use default
defaultMaxSlippage: public(uint256)          # default max slippage (100 = 1%)

# Blacklisted pairs (too volatile)
blacklistedPairs: HashMap[bytes32, bool]

SCALE: constant(uint256) = 10**18

@deploy
def __init__(_router: address, _governance: address):
    self.router = _router
    self.governance = _governance
    self.defaultMaxSlippage = 100  # 1% default

@external
def setUserMaxSlippage(maxSlippage: uint256):
    """ตั้ง max slippage สำหรับตัวเอง (0 = use default)"""
    assert maxSlippage <= 500, "Too high (max 5%)"
    self.userMaxSlippage[msg.sender] = maxSlippage

@external
def swapWithProtection(
    amountIn: uint256,
    path: DynArray[address, 5],
    to: address,
    deadline: uint256
) -> uint256:
    """
    Swap พร้อม MEV protection
    
    1. ตรวจ max slippage
    2. บันทึก pre-swap price
    3. Execute swap
    4. ตรวจ post-swap price (sandwich detection)
    
    Returns: amountOut
    """
    assert len(path) >= 2, "Invalid path"
    assert deadline >= block.timestamp, "Expired"
    
    # ดึง expected output
    expectedAmounts: DynArray[uint256, 5] = IUniswapV2Router(self.router).getAmountsOut(amountIn, path)
    expectedOut: uint256 = expectedAmounts[len(expectedAmounts) - 1]
    
    # คำนวณ min output ตาม slippage limit
    userSlippage: uint256 = self.userMaxSlippage[msg.sender]
    if userSlippage == 0:
        userSlippage = self.defaultMaxSlippage
    
    amountOutMin: uint256 = expectedOut * (10000 - userSlippage) / 10000
    
    # ดึง token จาก user
    assert ERC20(path[0]).transferFrom(msg.sender, self, amountIn), "Transfer failed"
    assert ERC20(path[0]).approve(self.router, amountIn), "Approve failed"
    
    # Snapshot price ก่อน swap
    pairKey: bytes32 = keccak256(abi.encode(path[0], path[len(path) - 1]))
    self.blockPriceSnapshot[block.number][pairKey] = expectedOut * SCALE / amountIn
    
    # Execute swap
    amounts: DynArray[uint256, 5] = IUniswapV2Router(self.router).swapExactTokensForTokens(
        amountIn,
        amountOutMin,
        path,
        to,
        deadline
    )
    
    actualOut: uint256 = amounts[len(amounts) - 1]
    
    # ตรวจ sandwich: ถ้า actual << expected อาจโดน sandwich
    if actualOut < expectedOut * 98 / 100:
        log SandwichDetected(msg.sender, expectedOut, actualOut)
    
    # คำนวณ actual slippage
    slippage: uint256 = 0
    if actualOut < expectedOut:
        slippage = (expectedOut - actualOut) * 10000 / expectedOut
    
    log SwapProtected(msg.sender, amountIn, actualOut, slippage)
    
    return actualOut

@external
@view
def getExpectedOutput(
    amountIn: uint256,
    path: DynArray[address, 5]
) -> (uint256, uint256):
    """
    ดู expected output และ min output ตาม slippage limit
    
    Returns: (expectedOut, minOut)
    """
    amounts: DynArray[uint256, 5] = IUniswapV2Router(self.router).getAmountsOut(amountIn, path)
    expectedOut: uint256 = amounts[len(amounts) - 1]
    
    userSlippage: uint256 = self.userMaxSlippage[msg.sender]
    if userSlippage == 0:
        userSlippage = self.defaultMaxSlippage
    
    minOut: uint256 = expectedOut * (10000 - userSlippage) / 10000
    
    return expectedOut, minOut
```

---

## 5. Tests

```python
# tests/test_mev_protection.py
import pytest
import secrets
from brownie import CommitRevealSwap, TWAMM, SlippageProtectedRouter
from brownie import MockERC20, MockUniswapV2Router, MockUniswapV2Pair, accounts, chain, web3
from eth_abi import encode

SCALE = 10**18

@pytest.fixture
def setup():
    owner = accounts[0]
    alice = accounts[1]
    attacker = accounts[2]
    
    usdc = MockERC20.deploy("USDC", "USDC", 6, {"from": owner})
    weth = MockERC20.deploy("WETH", "WETH", 18, {"from": owner})
    
    router = MockUniswapV2Router.deploy({"from": owner})
    pair = MockUniswapV2Pair.deploy(usdc.address, weth.address, {"from": owner})
    
    commit_swap = CommitRevealSwap.deploy(router.address, owner.address, {"from": owner})
    twamm = TWAMM.deploy(pair.address, usdc.address, weth.address, 10, owner.address, {"from": owner})
    protected = SlippageProtectedRouter.deploy(router.address, owner.address, {"from": owner})
    
    usdc.mint(alice, 10**10, {"from": owner})
    weth.mint(alice, 10**21, {"from": owner})
    usdc.mint(attacker, 10**10, {"from": owner})
    
    return owner, alice, attacker, usdc, weth, router, pair, commit_swap, twamm, protected

def test_commit_reveal_flow(setup):
    owner, alice, attacker, usdc, weth, router, pair, commit_swap, twamm, protected = setup
    
    amount_in = 1000 * 10**6  # $1000 USDC
    amount_out_min = 0.4 * SCALE  # min 0.4 ETH
    
    # สร้าง secret
    nonce = 12345
    secret = secrets.token_bytes(32)
    
    # สร้าง commit hash
    commit_hash = web3.keccak(
        encode(
            ["address", "address", "uint256", "uint256", "uint256", "bytes32"],
            [usdc.address, weth.address, amount_in, amount_out_min, nonce, secret]
        )
    )
    
    # Commit
    usdc.approve(commit_swap.address, amount_in, {"from": alice})
    commit_id = commit_swap.commit(commit_hash, usdc.address, amount_in, {"from": alice}).return_value
    
    print(f"Committed: {commit_id.hex()}")
    
    # รอ delay
    chain.mine(3)
    
    # Reveal and execute
    path = [usdc.address, weth.address]
    tx = commit_swap.reveal(
        commit_id,
        weth.address,
        amount_out_min,
        path,
        nonce,
        secret,
        {"from": alice}
    )
    
    print(f"Swap executed!")

def test_twamm_long_order(setup):
    owner, alice, attacker, usdc, weth, router, pair, commit_swap, twamm, protected = setup
    
    total_amount = 10000 * 10**6  # $10,000 USDC
    intervals = 100  # แบ่ง 100 intervals
    
    usdc.approve(twamm.address, total_amount, {"from": alice})
    
    order_id = twamm.createLongTermOrder(
        usdc.address,
        weth.address,
        total_amount,
        intervals,
        {"from": alice}
    ).return_value
    
    print(f"TWAMM order created: {order_id}")
    print(f"Amount per interval: {total_amount // intervals / 10**6:.2f} USDC")
    
    # รอ intervals
    chain.mine(10 * 10)  # 10 intervals
    
    # Execute
    twamm.executeVirtualOrders(50, {"from": owner})
    
    # ดู proceeds
    proceeds = twamm.pendingBalances(alice.address, weth.address)
    print(f"WETH proceeds: {proceeds / SCALE:.4f} ETH")

def test_slippage_protection(setup):
    owner, alice, attacker, usdc, weth, router, pair, commit_swap, twamm, protected = setup
    
    # ตั้ง slippage limit 0.5%
    protected.setUserMaxSlippage(50, {"from": alice})
    
    amount_in = 1000 * 10**6
    path = [usdc.address, weth.address]
    
    usdc.approve(protected.address, amount_in, {"from": alice})
    
    expected, min_out = protected.getExpectedOutput(amount_in, path)
    print(f"Expected: {expected / SCALE:.4f} ETH")
    print(f"Min out (0.5% slippage): {min_out / SCALE:.4f} ETH")
    
    # Swap with protection
    protected.swapWithProtection(
        amount_in,
        path,
        alice.address,
        chain.time() + 300,
        {"from": alice}
    )
```

---

## 6. สรุป

### MEV Protection Strategies:

**1. Commit-Reveal**
- ซ่อน swap parameters ก่อน
- Attacker ไม่รู้ว่าจะ swap อะไร จึงป้องกัน front-running ได้
- Trade-off: ต้อง 2 transactions

**2. TWAMM**
- แบ่ง large order เป็น small orders ตามเวลา
- Price impact น้อยลงมาก
- Arbitrageurs ช่วย maintain price ให้ถูก

**3. Private Mempool (Flashbots)**
- ส่ง tx ตรงให้ validator ไม่ผ่าน public mempool
- Attacker ไม่เห็น tx จึง front-run ไม่ได้
- ต้องใช้ Flashbots Protect RPC

**4. Slippage Limits**
- ป้องกัน worst-case execution
- แต่ไม่ได้ป้องกัน sandwich (แค่จำกัด damage)
- ต้อง balance ระหว่าง protection และ reverts

**5. Sandwich Attack Math**
```
Attacker buys X tokens (front-run)
Victim buys tokens at higher price
Attacker sells X tokens (back-run) at profit

Profit = (price_after_victim - price_before_victim) * X
        - gas_costs
```
