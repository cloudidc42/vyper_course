# Part 074: Flash Loan Strategies (กลยุทธ์ Flash Loan)

## สารบัญ
1. [บทนำ Flash Loans](#s1)
2. [Flash Loan Receiver Interface](#s2)
3. [Flash Loan Provider](#s3)
4. [Arbitrage Between AMMs](#s4)
5. [Collateral Swaps](#s5)
6. [Leveraged Positions](#s6)
7. [FlashLoanArbitrage Contract](#s7)
8. [Security Considerations](#s8)

---

## 1. บทนำ Flash Loans {#s1}

**Flash Loans** คือการกู้ยืม tokens ในปริมาณใดก็ได้ โดยต้องคืนภายใน transaction เดียวกัน
ไม่ต้องมี collateral แต่ถ้าไม่คืน transaction จะ revert ทั้งหมด

### Use Cases หลัก

| Use Case | คำอธิบาย |
|---|---|
| Arbitrage | ซื้อถูกขายแพงใน DEXs ต่างกัน |
| Collateral Swap | เปลี่ยน collateral โดยไม่ต้องปิด position |
| Self-liquidation | ชำระหนี้ตัวเองก่อน liquidation penalty |
| Leveraged Positions | สร้าง leverage ใน single tx |

### กระบวนการ Flash Loan

```
1. Call flashLoan(amount)
2. Provider transfers tokens to receiver
3. Receiver executes strategy
4. Receiver approves amount + fee
5. Provider pulls back amount + fee
6. If step 5 fails -> entire tx reverts
```

---

## 2. Flash Loan Receiver Interface {#s2}

```python
# @version 0.4.0
# IFlashLoanReceiver.vy
# Standard Flash Loan Receiver Interface (EIP-3156)

# EIP-3156 Flash Loan interfaces

interface IERC3156FlashLender:
    def maxFlashLoan(token: address) -> uint256: view
    def flashFee(token: address, amount: uint256) -> uint256: view
    def flashLoan(
        receiver: address,
        token: address,
        amount: uint256,
        data: Bytes[65536]
    ) -> bool: nonpayable

interface IERC3156FlashBorrower:
    def onFlashLoan(
        initiator: address,
        token: address,
        amount: uint256,
        fee: uint256,
        data: Bytes[65536]
    ) -> bytes32: nonpayable

# EIP-3156 callback return value
FLASH_LOAN_CALLBACK_SUCCESS: constant(bytes32) = keccak256("ERC3156FlashBorrower.onFlashLoan")
```

---

## 3. Flash Loan Provider {#s3}

```python
# @version 0.4.0
# FlashLoanProvider.vy
# EIP-3156 compliant Flash Loan Provider

from vyper.interfaces import ERC20

interface IERC3156FlashBorrower:
    def onFlashLoan(
        initiator: address,
        token: address,
        amount: uint256,
        fee: uint256,
        data: Bytes[65536]
    ) -> bytes32: nonpayable

# EIP-3156 success callback value
CALLBACK_SUCCESS: constant(bytes32) = keccak256("ERC3156FlashBorrower.onFlashLoan")

# ============================================================
# Storage
# ============================================================

owner: public(address)

# Fee configuration (in basis points)
flash_fee_rate: public(uint256)  # e.g., 9 = 0.09%

# Supported tokens
supported_tokens: public(HashMap[address, bool])

# Statistics
total_flash_loans: public(uint256)
total_fees_earned: public(HashMap[address, uint256])

# Reentrancy guard
locked: bool

# ============================================================
# Events
# ============================================================

event FlashLoan:
    receiver: indexed(address)
    token: indexed(address)
    amount: uint256
    fee: uint256

event FeeCollected:
    token: indexed(address)
    amount: uint256

event TokenAdded:
    token: indexed(address)

# ============================================================
# Constructor
# ============================================================

@deploy
def __init__(fee_rate: uint256):
    assert fee_rate <= 1000, "Fee too high (max 10%)"
    self.owner = msg.sender
    self.flash_fee_rate = fee_rate

# ============================================================
# EIP-3156 Flash Lender Interface
# ============================================================

@view
@external
def maxFlashLoan(token: address) -> uint256:
    """Maximum available for flash loan"""
    if not self.supported_tokens[token]:
        return 0
    return ERC20(token).balanceOf(self)

@view
@external
def flashFee(token: address, amount: uint256) -> uint256:
    """Calculate fee for flash loan"""
    assert self.supported_tokens[token], "Token not supported"
    return amount * self.flash_fee_rate / 10000

@external
def flashLoan(
    receiver: address,
    token: address,
    amount: uint256,
    data: Bytes[65536]
) -> bool:
    """
    Execute a flash loan
    @param receiver Contract to receive the loan
    @param token Token to borrow
    @param amount Amount to borrow
    @param data Arbitrary data passed to receiver
    """
    # Reentrancy protection
    assert not self.locked, "Reentrancy"
    self.locked = True
    
    assert self.supported_tokens[token], "Token not supported"
    assert amount > 0, "Zero amount"
    
    # Calculate fee
    fee: uint256 = amount * self.flash_fee_rate / 10000
    
    # Check available liquidity
    balance_before: uint256 = ERC20(token).balanceOf(self)
    assert balance_before >= amount, "Insufficient liquidity"
    
    # Transfer tokens to receiver
    ERC20(token).transfer(receiver, amount)
    
    # Call receiver's callback
    callback_return: bytes32 = IERC3156FlashBorrower(receiver).onFlashLoan(
        msg.sender,
        token,
        amount,
        fee,
        data
    )
    
    # Verify callback was successful
    assert callback_return == CALLBACK_SUCCESS, "Flash loan callback failed"
    
    # Verify repayment
    balance_after: uint256 = ERC20(token).balanceOf(self)
    assert balance_after >= balance_before + fee, "Flash loan not repaid"
    
    # Update stats
    self.total_flash_loans += 1
    self.total_fees_earned[token] += fee
    
    self.locked = False
    
    log FlashLoan(receiver, token, amount, fee)
    return True

# ============================================================
# Admin Functions
# ============================================================

@external
def add_token(token: address):
    """Add a supported flash loan token"""
    assert msg.sender == self.owner, "Not owner"
    self.supported_tokens[token] = True
    log TokenAdded(token)

@external
def withdraw_fees(token: address, to: address):
    """Withdraw collected fees"""
    assert msg.sender == self.owner, "Not owner"
    
    # Only withdraw fees, keep liquidity
    # In practice, you'd track fees separately
    # This is simplified
    balance: uint256 = ERC20(token).balanceOf(self)
    fees: uint256 = self.total_fees_earned[token]
    
    if fees > 0 and fees <= balance:
        self.total_fees_earned[token] = 0
        ERC20(token).transfer(to, fees)
        log FeeCollected(token, fees)
```

---

## 4. Arbitrage Between AMMs {#s4}

```python
# @version 0.4.0
# FlashArbitrageBasic.vy
# Basic flash loan arbitrage between two AMMs

from vyper.interfaces import ERC20

interface IERC3156FlashLender:
    def flashLoan(
        receiver: address,
        token: address,
        amount: uint256,
        data: Bytes[65536]
    ) -> bool: nonpayable
    def flashFee(token: address, amount: uint256) -> uint256: view

interface IAMM:
    def getAmountOut(amount_in: uint256, reserve_in: uint256, reserve_out: uint256) -> uint256: view
    def swap(amount0_out: uint256, amount1_out: uint256, to: address, data: Bytes[65536]): nonpayable
    def getReserves() -> (uint256, uint256, uint256): view  # reserve0, reserve1, timestamp

CALLBACK_SUCCESS: constant(bytes32) = keccak256("ERC3156FlashBorrower.onFlashLoan")

# ============================================================
# Storage
# ============================================================

owner: public(address)
flash_lender: public(address)

# Last arbitrage result
last_profit: public(uint256)
total_profit: public(uint256)
total_arbitrages: public(uint256)

# ============================================================
# Events
# ============================================================

event ArbitrageExecuted:
    token_in: indexed(address)
    amount_in: uint256
    profit: uint256
    dex_buy: indexed(address)
    dex_sell: indexed(address)

event ArbitrageFailed:
    reason: String[100]

# ============================================================
# Constructor
# ============================================================

@deploy
def __init__(lender: address):
    self.owner = msg.sender
    self.flash_lender = lender

# ============================================================
# Arbitrage Check
# ============================================================

@view
@external
def check_arbitrage(
    dex_a: address,
    dex_b: address,
    token_a: address,
    token_b: address,
    amount: uint256
) -> (int256, uint256, uint256):
    """
    Check if arbitrage is profitable
    @return (profit, amount_out_a, amount_out_b)
    Positive profit means buy on dex_a, sell on dex_b is profitable
    """
    # Get reserves from DEX A
    reserve_a0: uint256 = 0
    reserve_a1: uint256 = 0
    ts: uint256 = 0
    reserve_a0, reserve_a1, ts = IAMM(dex_a).getReserves()
    
    # Get reserves from DEX B
    reserve_b0: uint256 = 0
    reserve_b1: uint256 = 0
    ts2: uint256 = 0
    reserve_b0, reserve_b1, ts2 = IAMM(dex_b).getReserves()
    
    # Simulate buying token_b with token_a on DEX A
    amount_out_a: uint256 = IAMM(dex_a).getAmountOut(amount, reserve_a0, reserve_a1)
    
    # Simulate selling token_b for token_a on DEX B
    amount_out_b: uint256 = IAMM(dex_b).getAmountOut(amount_out_a, reserve_b1, reserve_b0)
    
    # Calculate flash loan fee
    flash_fee: uint256 = IERC3156FlashLender(self.flash_lender).flashFee(token_a, amount)
    
    # Profit = amount received - original amount - flash fee
    total_cost: uint256 = amount + flash_fee
    
    if amount_out_b > total_cost:
        profit: uint256 = amount_out_b - total_cost
        return convert(profit, int256), amount_out_a, amount_out_b
    else:
        loss: uint256 = total_cost - amount_out_b
        return -convert(loss, int256), amount_out_a, amount_out_b

# ============================================================
# Execute Arbitrage
# ============================================================

@external
def execute_arbitrage(
    token_a: address,
    token_b: address,
    amount: uint256,
    dex_buy: address,   # Where to buy token_b
    dex_sell: address,  # Where to sell token_b
    min_profit: uint256
) -> bool:
    """
    Execute arbitrage using flash loan
    @param token_a The token to borrow (and repay)
    @param token_b The token to trade via
    @param amount Amount to borrow
    @param dex_buy DEX where token_b is cheap
    @param dex_sell DEX where token_b is expensive
    @param min_profit Minimum profit to proceed
    """
    assert msg.sender == self.owner, "Not owner"
    
    # Encode trade params in calldata
    data: Bytes[160] = concat(
        convert(token_b, bytes20),
        convert(dex_buy, bytes20),
        convert(dex_sell, bytes20),
        convert(min_profit, bytes32),
        convert(amount, bytes32)
    )
    
    # Execute flash loan
    IERC3156FlashLender(self.flash_lender).flashLoan(
        self,
        token_a,
        amount,
        data
    )
    
    return True

@external
def onFlashLoan(
    initiator: address,
    token: address,
    amount: uint256,
    fee: uint256,
    data: Bytes[65536]
) -> bytes32:
    """
    Flash loan callback - execute the arbitrage
    Called by flash loan provider
    """
    assert msg.sender == self.flash_lender, "Not flash lender"
    assert initiator == self, "Not self"
    
    # Decode parameters
    token_b: address = convert(convert(slice(data, 0, 20), bytes20), address)
    dex_buy: address = convert(convert(slice(data, 20, 20), bytes20), address)
    dex_sell: address = convert(convert(slice(data, 40, 20), bytes20), address)
    min_profit: uint256 = convert(slice(data, 60, 32), uint256)
    
    # Step 1: Buy token_b on dex_buy using borrowed token
    ERC20(token).approve(dex_buy, amount)
    
    # Get expected output
    reserve_buy0: uint256 = 0
    reserve_buy1: uint256 = 0
    ts: uint256 = 0
    reserve_buy0, reserve_buy1, ts = IAMM(dex_buy).getReserves()
    
    amount_out: uint256 = IAMM(dex_buy).getAmountOut(amount, reserve_buy0, reserve_buy1)
    
    # Execute swap on dex_buy (get token_b)
    IAMM(dex_buy).swap(0, amount_out, self, b"")
    
    # Step 2: Sell token_b on dex_sell for token_a
    ERC20(token_b).approve(dex_sell, amount_out)
    
    reserve_sell0: uint256 = 0
    reserve_sell1: uint256 = 0
    ts2: uint256 = 0
    reserve_sell0, reserve_sell1, ts2 = IAMM(dex_sell).getReserves()
    
    amount_back: uint256 = IAMM(dex_sell).getAmountOut(amount_out, reserve_sell1, reserve_sell0)
    
    # Execute swap on dex_sell (get token_a back)
    IAMM(dex_sell).swap(amount_back, 0, self, b"")
    
    # Step 3: Calculate and verify profit
    total_repay: uint256 = amount + fee
    final_balance: uint256 = ERC20(token).balanceOf(self)
    
    assert final_balance >= total_repay + min_profit, "Insufficient profit"
    
    profit: uint256 = final_balance - total_repay
    
    # Step 4: Approve repayment
    ERC20(token).approve(self.flash_lender, total_repay)
    
    # Update stats
    self.last_profit = profit
    self.total_profit += profit
    self.total_arbitrages += 1
    
    log ArbitrageExecuted(token, amount, profit, dex_buy, dex_sell)
    
    return CALLBACK_SUCCESS
```

---

## 5. Collateral Swaps {#s5}

```python
# @version 0.4.0
# CollateralSwap.vy
# Swap collateral in a lending protocol using flash loans
# Example: Change collateral from ETH to WBTC without closing position

from vyper.interfaces import ERC20

interface IERC3156FlashLender:
    def flashLoan(
        receiver: address,
        token: address,
        amount: uint256,
        data: Bytes[65536]
    ) -> bool: nonpayable
    def flashFee(token: address, amount: uint256) -> uint256: view

interface ILendingProtocol:
    def supply(token: address, amount: uint256): nonpayable
    def withdraw(token: address, amount: uint256): nonpayable
    def borrow(token: address, amount: uint256): nonpayable
    def repay(token: address, amount: uint256): nonpayable
    def getDebtAmount(user: address, token: address) -> uint256: view
    def getCollateralAmount(user: address, token: address) -> uint256: view

interface IDEX:
    def swap(
        token_in: address,
        token_out: address,
        amount_in: uint256,
        min_out: uint256,
        recipient: address
    ) -> uint256: nonpayable

CALLBACK_SUCCESS: constant(bytes32) = keccak256("ERC3156FlashBorrower.onFlashLoan")

flash_lender: public(address)
lending_protocol: public(address)
dex: public(address)
owner: public(address)

event CollateralSwapped:
    old_collateral: indexed(address)
    new_collateral: indexed(address)
    amount: uint256

@deploy
def __init__(lender: address, lending: address, swap_dex: address):
    self.flash_lender = lender
    self.lending_protocol = lending
    self.dex = swap_dex
    self.owner = msg.sender

@external
def swap_collateral(
    old_collateral: address,
    new_collateral: address,
    collateral_amount: uint256,
    min_new_collateral: uint256
):
    """
    Swap collateral type without closing position
    
    Flow:
    1. Flash loan new_collateral
    2. Supply new_collateral as new collateral
    3. Withdraw old_collateral
    4. Swap old_collateral -> new_collateral on DEX
    5. Repay flash loan
    """
    assert msg.sender == self.owner, "Not owner"
    
    # Estimate how much new_collateral we need
    # (simplified - in production, account for slippage)
    fee: uint256 = IERC3156FlashLender(self.flash_lender).flashFee(
        new_collateral,
        min_new_collateral
    )
    
    data: Bytes[128] = concat(
        convert(old_collateral, bytes20),
        convert(new_collateral, bytes20),
        convert(collateral_amount, bytes32),
        convert(min_new_collateral, bytes32)
    )
    
    IERC3156FlashLender(self.flash_lender).flashLoan(
        self,
        new_collateral,
        min_new_collateral,
        data
    )

@external
def onFlashLoan(
    initiator: address,
    token: address,   # new_collateral
    amount: uint256,
    fee: uint256,
    data: Bytes[65536]
) -> bytes32:
    """Execute the collateral swap"""
    assert msg.sender == self.flash_lender, "Not lender"
    
    # Decode params
    old_collateral: address = convert(convert(slice(data, 0, 20), bytes20), address)
    new_collateral: address = convert(convert(slice(data, 20, 20), bytes20), address)
    collateral_amount: uint256 = convert(slice(data, 40, 32), uint256)
    min_new_collateral: uint256 = convert(slice(data, 72, 32), uint256)
    
    # 1. Supply new collateral to lending protocol
    ERC20(new_collateral).approve(self.lending_protocol, amount)
    ILendingProtocol(self.lending_protocol).supply(new_collateral, amount)
    
    # 2. Withdraw old collateral
    ILendingProtocol(self.lending_protocol).withdraw(old_collateral, collateral_amount)
    
    # 3. Swap old -> new collateral on DEX
    ERC20(old_collateral).approve(self.dex, collateral_amount)
    
    received: uint256 = IDEX(self.dex).swap(
        old_collateral,
        new_collateral,
        collateral_amount,
        amount + fee,  # Must receive at least enough to repay flash loan
        self
    )
    
    assert received >= amount + fee, "Insufficient swap output"
    
    # 4. Approve flash loan repayment
    ERC20(new_collateral).approve(self.flash_lender, amount + fee)
    
    log CollateralSwapped(old_collateral, new_collateral, collateral_amount)
    
    return CALLBACK_SUCCESS
```

---

## 6. Leveraged Positions {#s6}

```python
# @version 0.4.0
# FlashLoanLeverage.vy
# Create leveraged positions using flash loans
# Example: 3x leveraged ETH position

from vyper.interfaces import ERC20

interface IERC3156FlashLender:
    def flashLoan(
        receiver: address,
        token: address,
        amount: uint256,
        data: Bytes[65536]
    ) -> bool: nonpayable
    def flashFee(token: address, amount: uint256) -> uint256: view

interface ILendingProtocol:
    def supply(token: address, amount: uint256): nonpayable
    def borrow(token: address, amount: uint256): nonpayable
    def repay(token: address, amount: uint256): nonpayable
    def getCollateralFactor(token: address) -> uint256: view

interface IDEX:
    def swap(
        token_in: address,
        token_out: address,
        amount_in: uint256,
        min_out: uint256,
        recipient: address
    ) -> uint256: nonpayable

CALLBACK_SUCCESS: constant(bytes32) = keccak256("ERC3156FlashBorrower.onFlashLoan")

flash_lender: public(address)
lending_protocol: public(address)
dex: public(address)
owner: public(address)

# Track user positions
struct LeveragedPosition:
    collateral_token: address
    debt_token: address
    collateral_amount: uint256
    debt_amount: uint256
    leverage_ratio: uint256  # e.g., 3 = 3x leverage

user_positions: public(HashMap[address, LeveragedPosition])

event LeverageCreated:
    user: indexed(address)
    collateral: indexed(address)
    initial_amount: uint256
    leverage: uint256
    final_collateral: uint256

@deploy
def __init__(lender: address, lending: address, swap_dex: address):
    self.flash_lender = lender
    self.lending_protocol = lending
    self.dex = swap_dex
    self.owner = msg.sender

@view
@external
def estimate_leverage(
    collateral_token: address,
    debt_token: address,
    amount: uint256,
    leverage: uint256  # e.g., 300 = 3x
) -> (uint256, uint256):
    """
    Estimate flash loan amount for desired leverage
    Returns (flash_amount_needed, total_collateral)
    """
    # 3x leverage means: supply 3x amount, borrow 2x
    borrow_amount: uint256 = amount * (leverage - 100) / 100
    total_collateral: uint256 = amount + borrow_amount
    
    return borrow_amount, total_collateral

@external
def create_leveraged_position(
    collateral_token: address,
    debt_token: address,
    initial_amount: uint256,
    leverage: uint256,  # e.g., 300 = 3x
    max_slippage: uint256  # basis points
):
    """
    Create a leveraged long position
    
    Flow for 3x leverage with 1000 USDC:
    1. Flash loan 2000 USDC
    2. Convert 3000 USDC -> 3000 USDC worth of ETH on DEX
    3. Supply 3000 USDC worth of ETH as collateral
    4. Borrow 2000 USDC from lending protocol
    5. Repay flash loan with 2000 USDC
    """
    # Transfer initial collateral from user
    ERC20(collateral_token).transferFrom(msg.sender, self, initial_amount)
    
    # Calculate flash amount
    borrow_amount: uint256 = initial_amount * (leverage - 100) / 100
    
    data: Bytes[192] = concat(
        convert(collateral_token, bytes20),
        convert(debt_token, bytes20),
        convert(initial_amount, bytes32),
        convert(leverage, bytes32),
        convert(max_slippage, bytes32),
        convert(msg.sender, bytes20)
    )
    
    IERC3156FlashLender(self.flash_lender).flashLoan(
        self,
        debt_token,
        borrow_amount,
        data
    )
    
    log LeverageCreated(msg.sender, collateral_token, initial_amount, leverage, 0)

@external
def onFlashLoan(
    initiator: address,
    token: address,
    amount: uint256,
    fee: uint256,
    data: Bytes[65536]
) -> bytes32:
    """Execute leverage creation"""
    assert msg.sender == self.flash_lender, "Not lender"
    
    # Decode params
    collateral_token: address = convert(convert(slice(data, 0, 20), bytes20), address)
    debt_token: address = convert(convert(slice(data, 20, 20), bytes20), address)
    initial_amount: uint256 = convert(slice(data, 40, 32), uint256)
    leverage: uint256 = convert(slice(data, 72, 32), uint256)
    max_slippage: uint256 = convert(slice(data, 104, 32), uint256)
    user: address = convert(convert(slice(data, 136, 20), bytes20), address)
    
    # Total debt_token available (initial + flash loan)
    total_debt_token: uint256 = initial_amount + amount
    
    # Swap all debt_token -> collateral_token
    min_out: uint256 = total_debt_token * (10000 - max_slippage) / 10000
    ERC20(debt_token).approve(self.dex, total_debt_token)
    
    collateral_received: uint256 = IDEX(self.dex).swap(
        debt_token,
        collateral_token,
        total_debt_token,
        min_out,
        self
    )
    
    # Supply collateral to lending protocol
    ERC20(collateral_token).approve(self.lending_protocol, collateral_received)
    ILendingProtocol(self.lending_protocol).supply(collateral_token, collateral_received)
    
    # Borrow debt_token to repay flash loan
    repay_amount: uint256 = amount + fee
    ILendingProtocol(self.lending_protocol).borrow(debt_token, repay_amount)
    
    # Approve repayment
    ERC20(debt_token).approve(self.flash_lender, repay_amount)
    
    # Store position
    self.user_positions[user] = LeveragedPosition(
        collateral_token=collateral_token,
        debt_token=debt_token,
        collateral_amount=collateral_received,
        debt_amount=repay_amount,
        leverage_ratio=leverage
    )
    
    return CALLBACK_SUCCESS
```

---

## 7. FlashLoanArbitrage Contract {#s7}

```python
# @version 0.4.0
# FlashLoanArbitrage.vy
# Complete Flash Loan Arbitrage Contract
# Finds and executes profitable trades between two AMMs

from vyper.interfaces import ERC20

interface IERC3156FlashLender:
    def flashLoan(
        receiver: address,
        token: address,
        amount: uint256,
        data: Bytes[65536]
    ) -> bool: nonpayable
    def flashFee(token: address, amount: uint256) -> uint256: view
    def maxFlashLoan(token: address) -> uint256: view

interface IUniswapV2Pair:
    def getReserves() -> (uint112, uint112, uint32): view
    def swap(amount0Out: uint256, amount1Out: uint256, to: address, data: Bytes[65536]): nonpayable
    def token0() -> address: view
    def token1() -> address: view

interface IUniswapV2Router:
    def getAmountOut(amountIn: uint256, reserveIn: uint256, reserveOut: uint256) -> uint256: view
    def swapExactTokensForTokens(
        amountIn: uint256,
        amountOutMin: uint256,
        path: DynArray[address, 5],
        to: address,
        deadline: uint256
    ) -> DynArray[uint256, 5]: nonpayable

CALLBACK_SUCCESS: constant(bytes32) = keccak256("ERC3156FlashBorrower.onFlashLoan")
UNISWAP_FEE: constant(uint256) = 997  # 0.3% fee, so 99.7% of input reaches swap

# ============================================================
# Storage
# ============================================================

owner: public(address)
flash_lender: public(address)

# DEX configurations
dex_a_router: public(address)
dex_b_router: public(address)

# Profit tracking
total_profit: public(uint256)
total_flash_loans_taken: public(uint256)
profits_per_token: public(HashMap[address, uint256])

# Slippage protection
max_slippage: public(uint256)  # basis points

# Minimum profit threshold
min_profit_threshold: public(uint256)

# ============================================================
# Events
# ============================================================

event ArbitrageOpportunity:
    token_borrow: indexed(address)
    token_trade: indexed(address)
    amount: uint256
    estimated_profit: uint256

event ArbitrageSuccess:
    token: indexed(address)
    amount_borrowed: uint256
    profit: uint256
    net_profit_after_fee: uint256

event ArbitrageFailed:
    token: indexed(address)
    amount: uint256
    reason: String[50]

event ProfitWithdrawn:
    token: indexed(address)
    amount: uint256

# ============================================================
# Constructor
# ============================================================

@deploy
def __init__(
    lender: address,
    router_a: address,
    router_b: address
):
    self.owner = msg.sender
    self.flash_lender = lender
    self.dex_a_router = router_a
    self.dex_b_router = router_b
    self.max_slippage = 100      # 1% max slippage
    self.min_profit_threshold = 10**15  # 0.001 ETH minimum profit

# ============================================================
# Profit Calculation
# ============================================================

@view
@internal
def _get_amount_out_uni(
    router: address,
    amount_in: uint256,
    reserve_in: uint256,
    reserve_out: uint256
) -> uint256:
    """Uniswap V2 amount out calculation"""
    if reserve_in == 0 or reserve_out == 0:
        return 0
    
    amount_in_with_fee: uint256 = amount_in * UNISWAP_FEE
    numerator: uint256 = amount_in_with_fee * reserve_out
    denominator: uint256 = reserve_in * 1000 + amount_in_with_fee
    
    if denominator == 0:
        return 0
    
    return numerator / denominator

@view
@external
def find_best_arbitrage(
    pair_a: address,
    pair_b: address,
    amount: uint256
) -> (int256, address, address):
    """
    Find the best arbitrage direction between two pairs
    Returns (profit_or_loss, buy_dex, sell_dex)
    """
    # Get pair A reserves
    reserve_a0: uint112 = 0
    reserve_a1: uint112 = 0
    timestamp_a: uint32 = 0
    reserve_a0, reserve_a1, timestamp_a = IUniswapV2Pair(pair_a).getReserves()
    
    # Get pair B reserves
    reserve_b0: uint112 = 0
    reserve_b1: uint112 = 0
    timestamp_b: uint32 = 0
    reserve_b0, reserve_b1, timestamp_b = IUniswapV2Pair(pair_b).getReserves()
    
    # Get tokens
    token0_a: address = IUniswapV2Pair(pair_a).token0()
    token1_a: address = IUniswapV2Pair(pair_a).token1()
    
    # Calculate: buy on A, sell on B
    out_from_a: uint256 = self._get_amount_out_uni(
        self.dex_a_router,
        amount,
        convert(reserve_a0, uint256),
        convert(reserve_a1, uint256)
    )
    
    out_from_b_after_a: uint256 = self._get_amount_out_uni(
        self.dex_b_router,
        out_from_a,
        convert(reserve_b1, uint256),
        convert(reserve_b0, uint256)
    )
    
    # Flash loan fee
    flash_fee: uint256 = IERC3156FlashLender(self.flash_lender).flashFee(token0_a, amount)
    total_cost: uint256 = amount + flash_fee
    
    if out_from_b_after_a > total_cost:
        return convert(out_from_b_after_a - total_cost, int256), pair_a, pair_b
    
    # Calculate: buy on B, sell on A
    out_from_b: uint256 = self._get_amount_out_uni(
        self.dex_b_router,
        amount,
        convert(reserve_b0, uint256),
        convert(reserve_b1, uint256)
    )
    
    out_from_a_after_b: uint256 = self._get_amount_out_uni(
        self.dex_a_router,
        out_from_b,
        convert(reserve_a1, uint256),
        convert(reserve_a0, uint256)
    )
    
    if out_from_a_after_b > total_cost:
        return convert(out_from_a_after_b - total_cost, int256), pair_b, pair_a
    
    # No profitable direction
    max_loss: uint256 = total_cost
    if out_from_b_after_a > out_from_a_after_b:
        max_loss = total_cost - out_from_b_after_a
    else:
        max_loss = total_cost - out_from_a_after_b
    
    return -convert(max_loss, int256), pair_a, pair_b

# ============================================================
# Execute Arbitrage
# ============================================================

@external
def execute_arbitrage(
    token_borrow: address,
    token_trade: address,
    pair_buy: address,
    pair_sell: address,
    amount: uint256,
    min_profit: uint256
) -> bool:
    """
    Execute arbitrage between two pairs using flash loan
    @param token_borrow Token to flash loan
    @param token_trade Token to trade through
    @param pair_buy Pair where we buy token_trade
    @param pair_sell Pair where we sell token_trade
    @param amount Amount to borrow
    @param min_profit Minimum acceptable profit
    """
    assert msg.sender == self.owner, "Not owner"
    assert min_profit >= self.min_profit_threshold, "Below minimum profit"
    
    # Check flash loan availability
    max_loan: uint256 = IERC3156FlashLender(self.flash_lender).maxFlashLoan(token_borrow)
    assert amount <= max_loan, "Exceeds max flash loan"
    
    # Encode arbitrage params
    data: Bytes[256] = concat(
        convert(token_trade, bytes20),
        convert(pair_buy, bytes20),
        convert(pair_sell, bytes20),
        convert(min_profit, bytes32)
    )
    
    log ArbitrageOpportunity(token_borrow, token_trade, amount, min_profit)
    
    # Execute flash loan
    IERC3156FlashLender(self.flash_lender).flashLoan(
        self,
        token_borrow,
        amount,
        data
    )
    
    return True

@external
def onFlashLoan(
    initiator: address,
    token: address,  # token_borrow
    amount: uint256,
    fee: uint256,
    data: Bytes[65536]
) -> bytes32:
    """
    Execute the arbitrage trade after receiving flash loan
    """
    assert msg.sender == self.flash_lender, "Not lender"
    assert initiator == self, "Not self"
    
    # Decode params
    token_trade: address = convert(convert(slice(data, 0, 20), bytes20), address)
    pair_buy: address = convert(convert(slice(data, 20, 20), bytes20), address)
    pair_sell: address = convert(convert(slice(data, 40, 20), bytes20), address)
    min_profit: uint256 = convert(slice(data, 60, 32), uint256)
    
    token_borrow: address = token
    
    # ============================================================
    # Step 1: Buy token_trade on pair_buy using token_borrow
    # ============================================================
    
    # Get reserves for buy pair
    reserve_buy0: uint112 = 0
    reserve_buy1: uint112 = 0
    ts1: uint32 = 0
    reserve_buy0, reserve_buy1, ts1 = IUniswapV2Pair(pair_buy).getReserves()
    
    # Determine token order
    token0_buy: address = IUniswapV2Pair(pair_buy).token0()
    
    amount_out_buy: uint256 = 0
    if token0_buy == token_borrow:
        # token_borrow is token0, we get token1 (token_trade)
        amount_out_buy = self._get_amount_out_uni(
            self.dex_a_router,
            amount,
            convert(reserve_buy0, uint256),
            convert(reserve_buy1, uint256)
        )
        # Transfer token_borrow to pair and swap
        ERC20(token_borrow).transfer(pair_buy, amount)
        IUniswapV2Pair(pair_buy).swap(0, amount_out_buy, self, b"")
    else:
        # token_borrow is token1, we get token0 (token_trade)
        amount_out_buy = self._get_amount_out_uni(
            self.dex_a_router,
            amount,
            convert(reserve_buy1, uint256),
            convert(reserve_buy0, uint256)
        )
        ERC20(token_borrow).transfer(pair_buy, amount)
        IUniswapV2Pair(pair_buy).swap(amount_out_buy, 0, self, b"")
    
    assert amount_out_buy > 0, "Buy swap failed"
    
    # ============================================================
    # Step 2: Sell token_trade on pair_sell for token_borrow
    # ============================================================
    
    reserve_sell0: uint112 = 0
    reserve_sell1: uint112 = 0
    ts2: uint32 = 0
    reserve_sell0, reserve_sell1, ts2 = IUniswapV2Pair(pair_sell).getReserves()
    
    token0_sell: address = IUniswapV2Pair(pair_sell).token0()
    
    amount_out_sell: uint256 = 0
    if token0_sell == token_trade:
        # token_trade is token0, we get token1 (token_borrow)
        amount_out_sell = self._get_amount_out_uni(
            self.dex_b_router,
            amount_out_buy,
            convert(reserve_sell0, uint256),
            convert(reserve_sell1, uint256)
        )
        ERC20(token_trade).transfer(pair_sell, amount_out_buy)
        IUniswapV2Pair(pair_sell).swap(0, amount_out_sell, self, b"")
    else:
        # token_trade is token1, we get token0 (token_borrow)
        amount_out_sell = self._get_amount_out_uni(
            self.dex_b_router,
            amount_out_buy,
            convert(reserve_sell1, uint256),
            convert(reserve_sell0, uint256)
        )
        ERC20(token_trade).transfer(pair_sell, amount_out_buy)
        IUniswapV2Pair(pair_sell).swap(amount_out_sell, 0, self, b"")
    
    # ============================================================
    # Step 3: Verify profit and repay
    # ============================================================
    
    total_repay: uint256 = amount + fee
    final_balance: uint256 = ERC20(token_borrow).balanceOf(self)
    
    assert final_balance >= total_repay, "Cannot repay flash loan"
    
    profit: uint256 = 0
    if final_balance > total_repay:
        profit = final_balance - total_repay
    
    assert profit >= min_profit, "Profit below minimum"
    
    # Approve repayment
    ERC20(token_borrow).approve(self.flash_lender, total_repay)
    
    # Update stats
    self.total_profit += profit
    self.total_flash_loans_taken += 1
    self.profits_per_token[token_borrow] += profit
    
    log ArbitrageSuccess(token_borrow, amount, profit, profit)
    
    return CALLBACK_SUCCESS

# ============================================================
# Admin Functions
# ============================================================

@external
def withdraw_profit(token: address, amount: uint256):
    """Withdraw accumulated profit"""
    assert msg.sender == self.owner, "Not owner"
    
    balance: uint256 = ERC20(token).balanceOf(self)
    assert balance >= amount, "Insufficient balance"
    
    ERC20(token).transfer(msg.sender, amount)
    log ProfitWithdrawn(token, amount)

@external
def withdraw_all(tokens: DynArray[address, 10]):
    """Withdraw all profits for multiple tokens"""
    assert msg.sender == self.owner, "Not owner"
    
    for token: address in tokens:
        balance: uint256 = ERC20(token).balanceOf(self)
        if balance > 0:
            ERC20(token).transfer(msg.sender, balance)
            log ProfitWithdrawn(token, balance)

@external
def update_settings(min_profit: uint256, slippage: uint256):
    """Update arbitrage settings"""
    assert msg.sender == self.owner, "Not owner"
    assert slippage <= 500, "Max 5% slippage"
    
    self.min_profit_threshold = min_profit
    self.max_slippage = slippage
```

---

## 8. Security Considerations {#s8}

```python
# @version 0.4.0
# FlashLoanSecurity.vy
# Security patterns specific to flash loan contracts

# ============================================================
# 1. Callback Authentication
# ============================================================

# ALWAYS verify msg.sender is the flash lender
# and initiator is your own contract
@external
def onFlashLoan(
    initiator: address,
    token: address,
    amount: uint256,
    fee: uint256,
    data: Bytes[65536]
) -> bytes32:
    # CRITICAL: Both checks are needed
    assert msg.sender == self.flash_lender, "Not the flash lender"
    assert initiator == self, "Unexpected initiator"
    
    # Process...
    return keccak256("ERC3156FlashBorrower.onFlashLoan")

# ============================================================
# 2. Price Manipulation Prevention
# ============================================================

# Flash loans can be used to manipulate prices in AMMs
# Never use spot price for important calculations
# Use TWAP (Time Weighted Average Price) instead

# ============================================================
# 3. Reentrancy via Flash Loans
# ============================================================

# Flash loan callbacks can trigger reentrancy
# ALWAYS use reentrancy guards in contracts that receive flash loans

locked: bool

@internal
def _non_reentrant():
    assert not self.locked, "Reentrancy guard"
    self.locked = True

@internal
def _end_non_reentrant():
    self.locked = False

# ============================================================
# 4. Sandwich Attacks
# ============================================================

# Arbitrage bots can front-run your flash loan arbitrage
# Use private mempools or accept smaller profit margins

# ============================================================
# 5. Governance Attack via Flash Loans
# ============================================================

# Snapshot governance votes at proposal creation time
# NOT at voting time - prevents flash loan voting attacks

# Bad: uses current balance at vote time
# Good: uses balance at snapshot block
```

### การทดสอบ pytest

```python
# tests/test_flash_loans.py
import pytest
from ape import accounts, project, chain

@pytest.fixture
def deployer(accounts):
    return accounts[0]

@pytest.fixture
def mock_token(deployer, project):
    return deployer.deploy(project.MockERC20, "Test Token", "TEST", 18)

@pytest.fixture
def flash_provider(deployer, project):
    provider = deployer.deploy(project.FlashLoanProvider, 9)  # 0.09% fee
    return provider

@pytest.fixture
def funded_provider(flash_provider, mock_token, deployer):
    amount = 1_000_000 * 10**18
    mock_token.mint(flash_provider.address, amount, sender=deployer)
    flash_provider.add_token(mock_token.address, sender=deployer)
    return flash_provider

def test_flash_loan_basic(funded_provider, mock_token, deployer, project):
    receiver = deployer.deploy(project.SimpleFlashReceiver, funded_provider.address)
    
    amount = 10_000 * 10**18
    fee = funded_provider.flashFee(mock_token.address, amount)
    
    # Fund receiver with fee amount
    mock_token.mint(receiver.address, fee * 2, sender=deployer)
    
    # Execute flash loan
    funded_provider.flashLoan(
        receiver.address,
        mock_token.address,
        amount,
        b"",
        sender=deployer
    )
    
    assert funded_provider.total_flash_loans() == 1

def test_flash_loan_arbitrage_profitable(project, deployer, accounts):
    # Setup mock AMMs with price discrepancy
    # DEX A: 1 ETH = 1000 USDC
    # DEX B: 1 ETH = 1010 USDC
    # Profit: 10 USDC per ETH (minus fees)
    
    # This test validates the profit calculation logic
    pass  # Would require mock AMM setup

def test_flash_loan_fails_without_repayment(funded_provider, mock_token, deployer, project):
    """Flash loan must revert if not repaid"""
    # Deploy a receiver that doesn't repay
    bad_receiver = deployer.deploy(project.BadFlashReceiver, funded_provider.address)
    
    amount = 100 * 10**18
    
    with pytest.raises(Exception):
        funded_provider.flashLoan(
            bad_receiver.address,
            mock_token.address,
            amount,
            b"",
            sender=deployer
        )

def test_flash_loan_fee_calculation(funded_provider, mock_token):
    amount = 1_000 * 10**18
    fee = funded_provider.flashFee(mock_token.address, amount)
    
    # 0.09% of 1000 = 0.9 tokens
    expected_fee = amount * 9 // 10000
    assert fee == expected_fee
```

---

## สรุป

Flash Loan strategies เป็นเครื่องมือทรงพลังใน DeFi:
- **Arbitrage**: ทำกำไรจากราคาต่างกันใน DEXs
- **Collateral Swap**: เปลี่ยน collateral โดยไม่ปิด position
- **Leverage Creation**: สร้าง leverage ใน single transaction
- **EIP-3156**: มาตรฐาน Flash Loan interface
- **Security**: Verify callback, reentrancy guard, TWAP prices

---
[← Previous Part](part_073_yield_aggregators.md) | [→ Next Part](part_075_defi_security_patterns.md)
