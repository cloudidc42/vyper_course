# Part 051: DeFi Fundamentals

## สารบัญ
1. [DeFi คืออะไร?](#defi-intro)
2. [Key DeFi Concepts](#key-concepts)
3. [Liquidity Pools](#liquidity-pools)
4. [AMM Theory](#amm-theory)
5. [Impermanent Loss](#impermanent-loss)
6. [Yield Farming](#yield-farming)
7. [Flash Loans](#flash-loans)
8. [DeFi Risks](#risks)
9. [Smart Contract สำหรับ DeFi](#contracts)
10. [Example: Liquidity Pool](#example)

---

## 1. DeFi คืออะไร? {#defi-intro}

DeFi (Decentralized Finance) คือระบบการเงินที่ทำงานบน Blockchain โดยไม่ต้องผ่านตัวกลาง

### Traditional Finance vs DeFi

```
Traditional Finance:          DeFi:
━━━━━━━━━━━━━━━━━━━          ━━━━━━━━━━━━
ธนาคาร (Custodial)           Self-custody
KYC/AML required             Permissionless
Business hours               24/7/365
Country restrictions         Global access
High fees                    Lower fees
Slow settlement              Instant settlement
Opaque processes             Transparent code
Single points of failure     Decentralized
Censorship possible          Censorship resistant
```

### DeFi Stack

```
Application Layer (UI/UX)
    ├── Aggregators (1inch, Paraswap)
    ├── Yield Optimizers (Yearn, Convex)
    └── Portfolio trackers (Zapper, Debank)
        
Protocol Layer (Smart Contracts)
    ├── DEX (Uniswap, Curve, Balancer)
    ├── Lending (Aave, Compound)
    ├── Derivatives (dYdX, GMX)
    ├── Stablecoins (MakerDAO, Frax)
    └── Insurance (Nexus Mutual)
        
Infrastructure Layer
    ├── Oracles (Chainlink, Pyth)
    ├── Bridges (Stargate, Hop)
    └── Layer 2 (Arbitrum, Optimism)
        
Base Layer (Blockchain)
    └── Ethereum (EVM)
```

---

## 2. Key DeFi Concepts {#key-concepts}

### TVL (Total Value Locked)

```python
# TVL = มูลค่าทั้งหมดที่ Lock อยู่ใน Protocol
# วัดใน USD

# ตัวอย่าง TVL Contract
# @version 0.4.0

total_eth_locked: uint256   # In Wei
total_token_locked: HashMap[address, uint256]

@view
@external
def get_tvl_eth() -> uint256:
    return self.total_eth_locked

@view
@external
def get_token_tvl(token: address) -> uint256:
    return self.total_token_locked[token]
```

### APY vs APR

```
APR (Annual Percentage Rate):
= (Yield / Principal) × 100
= Simple interest rate

APY (Annual Percentage Yield):
= (1 + APR/n)^n - 1
= Compounded interest rate

ตัวอย่าง:
APR = 10%
n = 365 (daily compounding)
APY = (1 + 0.10/365)^365 - 1 ≈ 10.52%
```

```python
# @version 0.4.0

PRECISION: constant(uint256) = 10**18
SECONDS_PER_YEAR: constant(uint256) = 365 * 24 * 3600

@pure
@external
def apr_to_apy(
    apr: uint256,  # Annual rate * PRECISION (e.g., 10% = 0.1 * 10^18)
    compounds_per_year: uint256
) -> uint256:
    """
    แปลง APR เป็น APY
    apr: rate ต่อปี (scaled by PRECISION)
    compounds_per_year: จำนวนครั้ง compound ต่อปี
    """
    # APY = (1 + APR/n)^n - 1
    rate_per_period: uint256 = apr / compounds_per_year
    
    # Simple approximation สำหรับ n ใหญ่
    # APY ≈ APR + APR²/2 + ...
    return apr + (apr * apr) / (2 * PRECISION)

@pure
@external
def calculate_apy_rewards(
    principal: uint256,
    apy: uint256,          # Annual yield (scaled by PRECISION)
    duration_seconds: uint256
) -> uint256:
    """คำนวณ Rewards จาก APY"""
    # Rewards = principal * apy * duration / year
    return (principal * apy * duration_seconds) / (PRECISION * SECONDS_PER_YEAR)
```

### Price Impact

```python
# @version 0.4.0

# Price Impact = ผลกระทบต่อราคาจากขนาด Trade

@pure
@external
def calculate_price_impact(
    reserve_in: uint256,    # Reserve ของ input token
    amount_in: uint256      # จำนวน input token
) -> uint256:
    """
    คำนวณ Price Impact เป็น basis points
    สูตร AMM: price_impact = amount_in / (reserve_in + amount_in)
    """
    return (amount_in * 10000) / (reserve_in + amount_in)
```

---

## 3. Liquidity Pools {#liquidity-pools}

### แนวคิด Liquidity Pool

```
Traditional Order Book:          Liquidity Pool (AMM):
━━━━━━━━━━━━━━━━━━━━━━━          ━━━━━━━━━━━━━━━━━━━━━
Buyer ↔ Seller match             Trader ↔ Pool

Buy Order: 10 ETH @ $2000        Reserve ETH: 1000
Sell Order: 5 ETH @ $1999        Reserve USDC: 2,000,000
                                 Price: 2000 USDC/ETH (auto)

ต้องหา Counterparty               Pool เป็น Counterparty เสมอ
ราคาตาม Order Book               ราคาตาม Formula (x*y=k)
```

### LP Token (Liquidity Provider Token)

```python
# @version 0.4.0

"""
LP Token แทน Share ของ Liquidity Pool
เมื่อ Add Liquidity → Mint LP Token
เมื่อ Remove Liquidity → Burn LP Token

LP Token Value = Pool TVL / Total LP Supply
"""

# State
lp_token_name: String[50]
lp_total_supply: uint256
lp_balances: HashMap[address, uint256]

reserve_a: uint256
reserve_b: uint256

@internal
def _mint_lp(to: address, amount: uint256):
    """Mint LP Tokens"""
    self.lp_balances[to] += amount
    self.lp_total_supply += amount

@internal
def _burn_lp(from_: address, amount: uint256):
    """Burn LP Tokens"""
    assert self.lp_balances[from_] >= amount, "Insufficient LP"
    self.lp_balances[from_] -= amount
    self.lp_total_supply -= amount

@view
@external
def get_lp_value_per_token() -> (uint256, uint256):
    """
    มูลค่าต่อ LP Token
    Returns: (amount_a, amount_b) per LP token
    """
    if self.lp_total_supply == 0:
        return 0, 0
    
    amount_a: uint256 = (self.reserve_a * 10**18) / self.lp_total_supply
    amount_b: uint256 = (self.reserve_b * 10**18) / self.lp_total_supply
    
    return amount_a, amount_b
```

---

## 4. AMM Theory {#amm-theory}

### Constant Product Formula: x * y = k

```python
# @version 0.4.0

"""
Uniswap V2 AMM Formula:
x * y = k (constant)

x = reserve ของ Token A
y = reserve ของ Token B  
k = constant (product)

เมื่อ Trade:
- ส่ง dx ของ Token A
- รับ dy ของ Token B
- (x + dx) * (y - dy) = k
- dy = y - k/(x + dx)
- dy = y * dx / (x + dx)
"""

@pure
@external
def get_amount_out(
    amount_in: uint256,
    reserve_in: uint256,
    reserve_out: uint256
) -> uint256:
    """
    คำนวณจำนวน Output จาก Input (ไม่รวม Fee)
    Formula: y * dx / (x + dx)
    """
    assert amount_in > 0, "Insufficient input"
    assert reserve_in > 0 and reserve_out > 0, "Insufficient liquidity"
    
    numerator: uint256 = amount_in * reserve_out
    denominator: uint256 = reserve_in + amount_in
    
    return numerator / denominator

@pure
@external
def get_amount_out_with_fee(
    amount_in: uint256,
    reserve_in: uint256,
    reserve_out: uint256,
    fee_bps: uint256  # fee in basis points (30 = 0.3%)
) -> uint256:
    """
    คำนวณจำนวน Output หลังหัก Fee
    Uniswap V2 ใช้ fee = 0.3% (30 basis points)
    """
    assert amount_in > 0, "Insufficient input"
    assert reserve_in > 0 and reserve_out > 0, "Insufficient liquidity"
    
    # หัก fee จาก input
    fee_factor: uint256 = 10000 - fee_bps  # e.g., 9970 for 0.3% fee
    amount_in_with_fee: uint256 = amount_in * fee_factor
    
    numerator: uint256 = amount_in_with_fee * reserve_out
    denominator: uint256 = (reserve_in * 10000) + amount_in_with_fee
    
    return numerator / denominator

@pure
@external
def get_amount_in(
    amount_out: uint256,
    reserve_in: uint256,
    reserve_out: uint256
) -> uint256:
    """
    คำนวณจำนวน Input ที่ต้องการเพื่อได้ Output ที่ต้องการ
    Formula: x * dy / (y - dy) + 1
    """
    assert amount_out > 0, "Insufficient output"
    assert reserve_in > 0 and reserve_out > 0, "Insufficient liquidity"
    assert amount_out < reserve_out, "Insufficient liquidity"
    
    numerator: uint256 = reserve_in * amount_out
    denominator: uint256 = reserve_out - amount_out
    
    return (numerator / denominator) + 1  # +1 for rounding

@pure
@external
def quote(
    amount_a: uint256,
    reserve_a: uint256,
    reserve_b: uint256
) -> uint256:
    """
    คำนวณ Amount B ที่เท่ากันตาม Ratio ของ Pool
    ใช้สำหรับ Add Liquidity
    """
    assert amount_a > 0, "Insufficient amount"
    assert reserve_a > 0 and reserve_b > 0, "Insufficient reserves"
    
    return (amount_a * reserve_b) / reserve_a
```

### Spot Price

```python
# @version 0.4.0

@pure
@external
def spot_price(
    reserve_a: uint256,
    reserve_b: uint256,
    precision: uint256  # 10**18 สำหรับ 18 decimals
) -> uint256:
    """
    Spot Price ของ Token A ใน Token B
    price = reserve_b / reserve_a
    """
    return (reserve_b * precision) / reserve_a

@pure
@external
def price_after_trade(
    reserve_in: uint256,
    reserve_out: uint256,
    amount_in: uint256,
    precision: uint256
) -> uint256:
    """
    ราคาหลัง Trade
    new_price = (reserve_out - amount_out) / (reserve_in + amount_in)
    """
    # คำนวณ amount_out ก่อน
    numerator: uint256 = amount_in * reserve_out
    denominator: uint256 = reserve_in + amount_in
    amount_out: uint256 = numerator / denominator
    
    new_reserve_in: uint256 = reserve_in + amount_in
    new_reserve_out: uint256 = reserve_out - amount_out
    
    return (new_reserve_out * precision) / new_reserve_in
```

---

## 5. Impermanent Loss {#impermanent-loss}

### แนวคิด Impermanent Loss

```
Impermanent Loss (IL) เกิดเมื่อราคาของ Tokens เปลี่ยน
เมื่อเทียบกับการ Hold (HODL) เฉยๆ

สูตร:
IL = 2*sqrt(r) / (1 + r) - 1

ที่ r = ราคาเปลี่ยนแปลงเทียบกับตอน Deposit

ตัวอย่าง:
ราคา ETH ขึ้น 2x:
r = 2
IL = 2*sqrt(2)/(1+2) - 1
   = 2*1.414/3 - 1
   = 0.943 - 1
   = -5.72%

นั่นคือ LP ได้ผลตอบแทนน้อยกว่า HODL 5.72%
```

```python
# @version 0.4.0

# คำนวณ Impermanent Loss (approximate using integer math)
# PRECISION = 10**18

PRECISION: constant(uint256) = 10**18

@pure
@external
def calculate_il_percentage(
    price_ratio_x1000: uint256  # ราคาเปลี่ยน * 1000 (e.g., 2000 = 2x)
) -> uint256:
    """
    คำนวณ Impermanent Loss เป็น Percentage * 1000
    price_ratio_x1000: new_price/old_price * 1000
    
    Returns: IL percentage * 1000 (e.g., 57 = 5.7% IL)
    """
    r: uint256 = price_ratio_x1000
    
    # sqrt approximation: Babylonian method
    # IL = 2*sqrt(r)/((1+r)) - 1
    # Simplified: IL ≈ (sqrt(r) - 1)^2 / (2*sqrt(r)) * something
    
    # สำหรับ Production ใช้ library ที่มี sqrt
    # นี่เป็นแค่ approximation
    
    if r == 1000:  # No price change
        return 0
    
    # คร่าวๆ: IL เพิ่มตาม quadratic เมื่อ r ห่างจาก 1
    if r > 1000:
        deviation: uint256 = r - 1000
    else:
        deviation: uint256 = 1000 - r
    
    # rough approximation: IL ≈ deviation^2 / (2 * r * 1000000)
    return (deviation * deviation) / (2 * r * 1000)

@view
@external
def lp_value_vs_hodl(
    initial_eth: uint256,      # ETH ที่ deposit ไว้ (wei)
    initial_token: uint256,    # Token ที่ deposit ไว้
    current_eth_price: uint256, # ETH ราคาปัจจุบัน (USDC per ETH, scaled)
    initial_eth_price: uint256  # ETH ราคาตอน deposit
) -> (uint256, uint256):
    """
    เปรียบเทียบมูลค่า LP กับ HODL
    Returns: (lp_value, hodl_value) ใน USDC (scaled)
    """
    price_ratio: uint256 = (current_eth_price * PRECISION) / initial_eth_price
    
    # ใน AMM, เมื่อราคาเปลี่ยน reserves rebalance
    # ETH ใน pool ∝ 1/sqrt(price_ratio)
    # Token ใน pool ∝ sqrt(price_ratio)
    
    # Approximate: LP value = 2 * sqrt(price_ratio) * initial_value
    # (Simplified version, real version needs sqrt)
    
    initial_total_value: uint256 = (
        (initial_eth * initial_eth_price) / PRECISION + 
        initial_token
    )
    
    hodl_value: uint256 = (
        (initial_eth * current_eth_price) / PRECISION + 
        initial_token
    )
    
    # LP value (simplified - assumes constant k)
    lp_value: uint256 = initial_total_value  # Very simplified
    
    return lp_value, hodl_value
```

---

## 6. Yield Farming {#yield-farming}

### Reward Distribution Mechanism

```python
# @version 0.4.0
# ไฟล์: contracts/YieldFarm.vy
# SPDX-License-Identifier: MIT

"""
Simple Yield Farming Contract
- Users stake LP tokens
- Earn reward tokens over time
- Reward per second = totalReward / duration
"""

from vyper.interfaces import ERC20

# ════════════════════════
# CONSTANTS
# ════════════════════════
PRECISION: constant(uint256) = 10**18

# ════════════════════════
# STATE VARIABLES
# ════════════════════════

# Tokens
lp_token: address     # Token ที่ Stake
reward_token: address # Token ที่ได้รับเป็น Reward

# Reward tracking
reward_rate: uint256        # Reward per second (scaled by PRECISION)
last_update_time: uint256   # ครั้งล่าสุดที่อัปเดต rewards
reward_per_token_stored: uint256  # Accumulated reward per LP token

# User tracking
user_reward_per_token_paid: HashMap[address, uint256]
rewards: HashMap[address, uint256]
balances: HashMap[address, uint256]

# Totals
total_supply: uint256
reward_end_time: uint256  # Rewards end at this time

owner: address

# ════════════════════════
# EVENTS
# ════════════════════════
event Staked:
    user: indexed(address)
    amount: uint256

event Withdrawn:
    user: indexed(address)
    amount: uint256

event RewardPaid:
    user: indexed(address)
    reward: uint256

event RewardAdded:
    reward_amount: uint256
    duration: uint256

# ════════════════════════
# CONSTRUCTOR
# ════════════════════════

@deploy
def __init__(lp_token_addr: address, reward_token_addr: address):
    self.lp_token = lp_token_addr
    self.reward_token = reward_token_addr
    self.owner = msg.sender

# ════════════════════════
# REWARD CALCULATION
# ════════════════════════

@view
@internal
def _last_time_reward_applicable() -> uint256:
    """เวลาล่าสุดที่ Reward ยังใช้ได้"""
    if block.timestamp < self.reward_end_time:
        return block.timestamp
    return self.reward_end_time

@view
@internal
def _reward_per_token() -> uint256:
    """Accumulated reward per LP token"""
    if self.total_supply == 0:
        return self.reward_per_token_stored
    
    time_elapsed: uint256 = (
        self._last_time_reward_applicable() - self.last_update_time
    )
    
    return (
        self.reward_per_token_stored + 
        (time_elapsed * self.reward_rate * PRECISION / self.total_supply)
    )

@view
@internal
def _earned(account: address) -> uint256:
    """คำนวณ Reward ที่ User ได้รับ"""
    return (
        self.balances[account] * 
        (_reward_per_token() - self.user_reward_per_token_paid[account]) / 
        PRECISION + 
        self.rewards[account]
    )

@internal
def _update_reward(account: address):
    """Update Reward State"""
    self.reward_per_token_stored = self._reward_per_token()
    self.last_update_time = self._last_time_reward_applicable()
    
    if account != empty(address):
        self.rewards[account] = self._earned(account)
        self.user_reward_per_token_paid[account] = self.reward_per_token_stored

# ════════════════════════
# PUBLIC FUNCTIONS
# ════════════════════════

@external
def stake(amount: uint256):
    """Stake LP Tokens"""
    assert amount > 0, "Cannot stake 0"
    
    self._update_reward(msg.sender)
    
    self.total_supply += amount
    self.balances[msg.sender] += amount
    
    # Transfer LP tokens from user
    assert ERC20(self.lp_token).transferFrom(
        msg.sender, self, amount
    ), "Transfer failed"
    
    log Staked(msg.sender, amount)

@external
def withdraw(amount: uint256):
    """Withdraw staked LP Tokens"""
    assert amount > 0, "Cannot withdraw 0"
    assert self.balances[msg.sender] >= amount, "Insufficient balance"
    
    self._update_reward(msg.sender)
    
    self.total_supply -= amount
    self.balances[msg.sender] -= amount
    
    # Transfer LP tokens back to user
    assert ERC20(self.lp_token).transfer(
        msg.sender, amount
    ), "Transfer failed"
    
    log Withdrawn(msg.sender, amount)

@external
def claim_reward():
    """Claim earned rewards"""
    self._update_reward(msg.sender)
    
    reward: uint256 = self.rewards[msg.sender]
    
    if reward > 0:
        self.rewards[msg.sender] = 0
        
        assert ERC20(self.reward_token).transfer(
            msg.sender, reward
        ), "Reward transfer failed"
        
        log RewardPaid(msg.sender, reward)

@external
def exit():
    """Withdraw all staked tokens and claim rewards"""
    self.withdraw(self.balances[msg.sender])
    self.claim_reward()

# ════════════════════════
# VIEW FUNCTIONS
# ════════════════════════

@view
@external
def earned(account: address) -> uint256:
    """ดู Reward ที่ยังไม่ Claim"""
    return self._earned(account)

@view
@external
def get_stake(account: address) -> uint256:
    """ดูจำนวน LP Tokens ที่ Staked"""
    return self.balances[account]

@view
@external
def get_reward_rate() -> uint256:
    """Reward Rate ต่อวินาที"""
    return self.reward_rate

# ════════════════════════
# ADMIN FUNCTIONS
# ════════════════════════

@external
def notify_reward_amount(reward: uint256, duration: uint256):
    """
    กำหนด Reward Distribution
    เรียกหลังจากโอน reward_token เข้า contract
    """
    assert msg.sender == self.owner, "Only owner"
    
    self._update_reward(empty(address))
    
    if block.timestamp >= self.reward_end_time:
        # Reward period ended, start new one
        self.reward_rate = reward / duration
    else:
        # Add to existing period
        remaining: uint256 = self.reward_end_time - block.timestamp
        leftover: uint256 = remaining * self.reward_rate
        self.reward_rate = (reward + leftover) / duration
    
    self.last_update_time = block.timestamp
    self.reward_end_time = block.timestamp + duration
    
    log RewardAdded(reward, duration)
```

---

## 7. Flash Loans {#flash-loans}

### Flash Loan Concept

```
Flash Loan ยืม ETH/Token จำนวนมาก
→ ใช้ในการทำ Arbitrage/Liquidation
→ คืนใน Transaction เดียวกัน
→ ถ้าคืนไม่ได้ → Transaction Revert

Use Cases:
1. Arbitrage: ซื้อถูกที่ A, ขายแพงที่ B
2. Liquidation: Liquidate Position ที่ Under-collateralized
3. Collateral Swap: เปลี่ยน Collateral โดยไม่ต้องมีทุน
4. Self-liquidation: ปิด Position ตัวเอง
```

```python
# @version 0.4.0
# ไฟล์: contracts/FlashLoanProvider.vy

interface IFlashLoanReceiver:
    def on_flash_loan(
        initiator: address,
        token: address,
        amount: uint256,
        fee: uint256,
        data: Bytes[1024]
    ) -> bytes32: nonpayable

from vyper.interfaces import ERC20

# ════════════════════════
# CONSTANTS
# ════════════════════════
FLASH_LOAN_FEE: constant(uint256) = 9  # 0.09% fee (basis points / 100)
CALLBACK_SUCCESS: constant(bytes32) = 0x439148f0bbc682ca079e46d6e2c2f0c1e3b820f1a291b069d8882abf8cf18dd9

owner: address
supported_tokens: HashMap[address, bool]

event FlashLoan:
    receiver: indexed(address)
    token: indexed(address)
    amount: uint256
    fee: uint256

@deploy
def __init__():
    self.owner = msg.sender

@external
def flash_loan(
    receiver: address,
    token: address,
    amount: uint256,
    data: Bytes[1024]
) -> bool:
    """
    Execute a Flash Loan
    """
    assert self.supported_tokens[token], "Token not supported"
    assert amount > 0, "Amount must be > 0"
    
    token_contract: ERC20 = ERC20(token)
    balance_before: uint256 = token_contract.balanceOf(self)
    
    assert balance_before >= amount, "Insufficient liquidity"
    
    # Calculate fee
    fee: uint256 = (amount * FLASH_LOAN_FEE) / 10000
    
    # Transfer tokens to receiver
    assert token_contract.transfer(receiver, amount), "Transfer failed"
    
    # Call receiver callback
    result: bytes32 = IFlashLoanReceiver(receiver).on_flash_loan(
        msg.sender,
        token,
        amount,
        fee,
        data
    )
    
    assert result == CALLBACK_SUCCESS, "Flash loan callback failed"
    
    # Verify repayment
    balance_after: uint256 = token_contract.balanceOf(self)
    assert balance_after >= balance_before + fee, "Flash loan not repaid"
    
    log FlashLoan(receiver, token, amount, fee)
    
    return True

@external
def add_token(token: address):
    assert msg.sender == self.owner, "Only owner"
    self.supported_tokens[token] = True

@external
def remove_token(token: address):
    assert msg.sender == self.owner, "Only owner"
    self.supported_tokens[token] = False
```

### Flash Loan Receiver Example

```python
# @version 0.4.0
# ไฟล์: contracts/FlashLoanArbitrage.vy

interface IFlashLoanProvider:
    def flash_loan(
        receiver: address,
        token: address,
        amount: uint256,
        data: Bytes[1024]
    ) -> bool: nonpayable

interface IDEX:
    def swap(
        token_in: address,
        token_out: address,
        amount_in: uint256
    ) -> uint256: nonpayable

from vyper.interfaces import ERC20

CALLBACK_SUCCESS: constant(bytes32) = 0x439148f0bbc682ca079e46d6e2c2f0c1e3b820f1a291b069d8882abf8cf18dd9

flash_loan_provider: address
owner: address
profit: uint256

@deploy
def __init__(provider: address):
    self.flash_loan_provider = provider
    self.owner = msg.sender

@external
def execute_arbitrage(
    token: address,
    amount: uint256,
    dex_a: address,
    dex_b: address,
    token_intermediate: address
):
    """
    เริ่ม Arbitrage ด้วย Flash Loan
    1. ยืม token จาก provider
    2. ขายที่ DEX A ได้ token_intermediate
    3. ซื้อ token กลับที่ DEX B
    4. คืน Flash Loan + ได้กำไร
    """
    assert msg.sender == self.owner, "Only owner"
    
    # Encode arbitrage params
    data: Bytes[1024] = concat(
        convert(dex_a, bytes32),
        convert(dex_b, bytes32),
        convert(token_intermediate, bytes32)
    )
    
    IFlashLoanProvider(self.flash_loan_provider).flash_loan(
        self,
        token,
        amount,
        data
    )

@external
def on_flash_loan(
    initiator: address,
    token: address,
    amount: uint256,
    fee: uint256,
    data: Bytes[1024]
) -> bytes32:
    """
    Flash Loan Callback
    Execute arbitrage logic here
    """
    assert msg.sender == self.flash_loan_provider, "Not provider"
    
    # Decode params
    dex_a: address = convert(slice(data, 0, 32), address)
    dex_b: address = convert(slice(data, 32, 64), address)
    token_intermediate: address = convert(slice(data, 64, 96), address)
    
    # Approve DEX A to spend token
    ERC20(token).approve(dex_a, amount)
    
    # Swap at DEX A: token → token_intermediate
    intermediate_amount: uint256 = IDEX(dex_a).swap(
        token,
        token_intermediate,
        amount
    )
    
    # Approve DEX B
    ERC20(token_intermediate).approve(dex_b, intermediate_amount)
    
    # Swap at DEX B: token_intermediate → token
    final_amount: uint256 = IDEX(dex_b).swap(
        token_intermediate,
        token,
        intermediate_amount
    )
    
    # Calculate profit
    total_repayment: uint256 = amount + fee
    assert final_amount >= total_repayment, "No arbitrage opportunity"
    
    self.profit += final_amount - total_repayment
    
    # Approve repayment
    ERC20(token).approve(self.flash_loan_provider, total_repayment)
    
    return CALLBACK_SUCCESS

@external
def withdraw_profit(token: address):
    """Withdraw profits"""
    assert msg.sender == self.owner, "Only owner"
    balance: uint256 = ERC20(token).balanceOf(self)
    if balance > 0:
        ERC20(token).transfer(self.owner, balance)
    self.profit = 0
```

---

## 8. DeFi Risks {#risks}

### ประเภทของความเสี่ยง

```python
# @version 0.4.0

"""
DeFi Risk Categories:

1. Smart Contract Risk
   - Bugs in code
   - Logic errors
   - Access control issues
   - Example: The DAO hack (Reentrancy)

2. Oracle Risk
   - Price manipulation
   - Oracle failure
   - Flash loan oracle attacks
   - Example: Mango Markets ($114M)

3. Liquidity Risk
   - Low liquidity pools
   - Slippage too high
   - Cannot exit position
   
4. Governance Risk
   - Malicious proposal
   - 51% attack on governance
   - Admin key compromise
   
5. Economic Risk
   - Tokenomics failure
   - Death spiral
   - Bank run
   - Example: Terra/Luna collapse
   
6. Systemic Risk
   - Protocol dependencies
   - Composability attacks
   - Black swan events
"""

# ════════════════════════
# RISK MITIGATION PATTERNS
# ════════════════════════

# 1. Circuit Breaker
max_tvl: uint256
current_tvl: uint256

@internal
def _check_circuit_breaker(amount: uint256):
    """ป้องกัน TVL เกิน limit"""
    assert self.current_tvl + amount <= self.max_tvl, "TVL limit reached"

# 2. Slippage Protection
@pure
@external
def check_slippage(
    expected: uint256,
    actual: uint256,
    max_slippage_bps: uint256  # e.g., 50 = 0.5%
) -> bool:
    """ตรวจสอบว่า Slippage ไม่เกิน limit"""
    if actual >= expected:
        return True
    
    slippage: uint256 = ((expected - actual) * 10000) / expected
    return slippage <= max_slippage_bps

# 3. Price Staleness Check
last_price_update: uint256
PRICE_STALENESS_THRESHOLD: constant(uint256) = 3600  # 1 hour

@internal
def _check_price_freshness():
    """ตรวจสอบว่าราคาไม่ outdated"""
    assert (
        block.timestamp - self.last_price_update <= PRICE_STALENESS_THRESHOLD
    ), "Price too stale"

# 4. Emergency Pause
paused: bool
pause_admin: address

@external
def emergency_pause():
    """Pause ฉุกเฉิน"""
    assert msg.sender == self.pause_admin, "Not authorized"
    self.paused = True

# 5. Rate Limiting
last_action: HashMap[address, uint256]
COOLDOWN: constant(uint256) = 60  # 1 minute

@internal
def _rate_limit(user: address):
    """ป้องกัน Rapid actions"""
    assert (
        block.timestamp >= self.last_action[user] + COOLDOWN
    ), "Cooldown not expired"
    self.last_action[user] = block.timestamp
```

---

## 9. Smart Contract สำหรับ DeFi {#contracts}

### Pattern: Upgradeable Protocol Config

```python
# @version 0.4.0
# ไฟล์: contracts/ProtocolConfig.vy

"""
Protocol Configuration Contract
แยก Configuration จาก Logic เพื่อให้ Update ได้
"""

struct FeeConfig:
    deposit_fee: uint256      # basis points
    withdrawal_fee: uint256
    performance_fee: uint256
    protocol_fee: uint256

struct LimitConfig:
    min_deposit: uint256
    max_deposit: uint256
    max_tvl: uint256
    max_slippage: uint256    # basis points

struct TimelockConfig:
    deposit_delay: uint256
    withdrawal_delay: uint256
    governance_delay: uint256

# State
fee_config: FeeConfig
limit_config: LimitConfig
timelock_config: TimelockConfig

owner: address
governance: address
fee_treasury: address

# Pending changes (timelock)
pending_fee_config: FeeConfig
pending_config_time: uint256
CHANGE_DELAY: constant(uint256) = 48 * 3600

event ConfigChangeProposed:
    proposed_by: indexed(address)
    effective_time: uint256

event ConfigChanged:
    changed_by: indexed(address)

@deploy
def __init__(
    treasury: address,
    initial_deposit_fee: uint256,
    initial_max_tvl: uint256
):
    self.owner = msg.sender
    self.governance = msg.sender
    self.fee_treasury = treasury
    
    self.fee_config = FeeConfig({
        deposit_fee: initial_deposit_fee,
        withdrawal_fee: initial_deposit_fee,
        performance_fee: 2000,  # 20%
        protocol_fee: 500       # 5%
    })
    
    self.limit_config = LimitConfig({
        min_deposit: 1 * 10**16,    # 0.01 ETH
        max_deposit: 100 * 10**18,  # 100 ETH
        max_tvl: initial_max_tvl,
        max_slippage: 100           # 1%
    })
    
    self.timelock_config = TimelockConfig({
        deposit_delay: 0,
        withdrawal_delay: 0,
        governance_delay: 48 * 3600
    })

@view
@external
def get_fee_config() -> FeeConfig:
    return self.fee_config

@view
@external
def get_limit_config() -> LimitConfig:
    return self.limit_config

@external
def propose_fee_change(new_config: FeeConfig):
    """Propose fee config change with timelock"""
    assert msg.sender == self.governance, "Not governance"
    assert new_config.deposit_fee <= 1000, "Fee too high"
    assert new_config.withdrawal_fee <= 1000, "Fee too high"
    assert new_config.performance_fee <= 3000, "Fee too high"
    assert new_config.protocol_fee <= 1000, "Fee too high"
    
    self.pending_fee_config = new_config
    self.pending_config_time = block.timestamp + CHANGE_DELAY
    
    log ConfigChangeProposed(msg.sender, self.pending_config_time)

@external
def execute_fee_change():
    """Execute fee change after timelock"""
    assert self.pending_config_time > 0, "No pending change"
    assert block.timestamp >= self.pending_config_time, "Timelock active"
    
    self.fee_config = self.pending_fee_config
    self.pending_config_time = 0
    
    log ConfigChanged(msg.sender)
```

---

## 10. ตัวอย่าง: Liquidity Pool {#example}

```python
# @version 0.4.0
# SPDX-License-Identifier: MIT
# ไฟล์: contracts/SimplePair.vy

"""
Simple AMM Liquidity Pool (Uniswap V2-like)
Features:
- Add/Remove Liquidity
- Swap Token A ↔ Token B
- LP Token minting/burning
- 0.3% Swap Fee
"""

from vyper.interfaces import ERC20

# ════════════════════════
# CONSTANTS
# ════════════════════════
MINIMUM_LIQUIDITY: constant(uint256) = 1000  # Prevent division by zero
PRECISION: constant(uint256) = 10**18
FEE_BPS: constant(uint256) = 30  # 0.3%

# ════════════════════════
# EVENTS
# ════════════════════════
event Mint:
    sender: indexed(address)
    amount0: uint256
    amount1: uint256

event Burn:
    sender: indexed(address)
    amount0: uint256
    amount1: uint256
    to: indexed(address)

event Swap:
    sender: indexed(address)
    amount0_in: uint256
    amount1_in: uint256
    amount0_out: uint256
    amount1_out: uint256
    to: indexed(address)

event Sync:
    reserve0: uint256
    reserve1: uint256

event Transfer:
    sender: indexed(address)
    receiver: indexed(address)
    value: uint256

# ════════════════════════
# STATE VARIABLES
# ════════════════════════

# LP Token
name: String[32]
symbol: String[8]
decimals: uint8
total_supply: uint256
balances: HashMap[address, uint256]
allowances: HashMap[address, HashMap[address, uint256]]

# Pool
token0: address
token1: address
reserve0: uint256
reserve1: uint256
k_last: uint256  # reserve0 * reserve1, ล่าสุดหลัง fee

factory: address
initialized: bool

# ════════════════════════
# CONSTRUCTOR
# ════════════════════════

@deploy
def __init__():
    self.factory = msg.sender
    self.decimals = 18

@external
def initialize(token0_addr: address, token1_addr: address):
    """Initialize pool (called by factory)"""
    assert msg.sender == self.factory, "Not factory"
    assert not self.initialized, "Already initialized"
    
    self.token0 = token0_addr
    self.token1 = token1_addr
    self.initialized = True
    
    # Set LP token name
    self.name = "Simple LP Token"
    self.symbol = "SLP"

# ════════════════════════
# LP TOKEN FUNCTIONS
# ════════════════════════

@internal
def _mint_lp(to: address, amount: uint256):
    self.total_supply += amount
    self.balances[to] += amount
    log Transfer(empty(address), to, amount)

@internal
def _burn_lp(from_: address, amount: uint256):
    self.balances[from_] -= amount
    self.total_supply -= amount
    log Transfer(from_, empty(address), amount)

@external
def transfer(to: address, amount: uint256) -> bool:
    self.balances[msg.sender] -= amount
    self.balances[to] += amount
    log Transfer(msg.sender, to, amount)
    return True

@view
@external
def balanceOf(account: address) -> uint256:
    return self.balances[account]

# ════════════════════════
# INTERNAL FUNCTIONS
# ════════════════════════

@view
@internal
def _get_reserves() -> (uint256, uint256):
    return self.reserve0, self.reserve1

@internal
def _update(balance0: uint256, balance1: uint256):
    self.reserve0 = balance0
    self.reserve1 = balance1
    log Sync(balance0, balance1)

@pure
@internal
def _sqrt(y: uint256) -> uint256:
    """Integer square root (Babylonian method)"""
    if y == 0:
        return 0
    
    z: uint256 = y
    x: uint256 = y / 2 + 1
    
    for _ in range(256):
        if x >= z:
            break
        z = x
        x = (y / x + x) / 2
    
    return z

@pure
@internal
def _min(x: uint256, y: uint256) -> uint256:
    if x < y:
        return x
    return y

# ════════════════════════
# LIQUIDITY FUNCTIONS
# ════════════════════════

@external
def add_liquidity(
    token0_amount: uint256,
    token1_amount: uint256,
    min_lp_amount: uint256
) -> uint256:
    """
    Add Liquidity to Pool
    Returns: LP tokens minted
    """
    assert self.initialized, "Not initialized"
    assert token0_amount > 0 and token1_amount > 0, "Insufficient amounts"
    
    r0: uint256 = self.reserve0
    r1: uint256 = self.reserve1
    
    # Transfer tokens in
    ERC20(self.token0).transferFrom(msg.sender, self, token0_amount)
    ERC20(self.token1).transferFrom(msg.sender, self, token1_amount)
    
    lp_amount: uint256 = 0
    
    if self.total_supply == 0:
        # First liquidity: geometric mean
        lp_amount = self._sqrt(token0_amount * token1_amount) - MINIMUM_LIQUIDITY
        
        # Mint minimum liquidity to zero address (permanently locked)
        self._mint_lp(empty(address), MINIMUM_LIQUIDITY)
    else:
        # Subsequent liquidity: proportional to reserves
        lp_from_0: uint256 = (token0_amount * self.total_supply) / r0
        lp_from_1: uint256 = (token1_amount * self.total_supply) / r1
        lp_amount = self._min(lp_from_0, lp_from_1)
    
    assert lp_amount >= min_lp_amount, "Slippage protection"
    assert lp_amount > 0, "Insufficient liquidity minted"
    
    self._mint_lp(msg.sender, lp_amount)
    
    # Update reserves
    new_balance0: uint256 = ERC20(self.token0).balanceOf(self)
    new_balance1: uint256 = ERC20(self.token1).balanceOf(self)
    self._update(new_balance0, new_balance1)
    
    log Mint(msg.sender, token0_amount, token1_amount)
    
    return lp_amount

@external
def remove_liquidity(
    lp_amount: uint256,
    min_amount0: uint256,
    min_amount1: uint256
) -> (uint256, uint256):
    """
    Remove Liquidity
    Returns: (amount0, amount1) returned to user
    """
    assert lp_amount > 0, "Insufficient LP amount"
    assert self.balances[msg.sender] >= lp_amount, "Insufficient LP balance"
    
    total_lp: uint256 = self.total_supply
    r0: uint256 = self.reserve0
    r1: uint256 = self.reserve1
    
    # Calculate token amounts proportional to LP share
    amount0: uint256 = (lp_amount * r0) / total_lp
    amount1: uint256 = (lp_amount * r1) / total_lp
    
    assert amount0 >= min_amount0, "Insufficient amount0"
    assert amount1 >= min_amount1, "Insufficient amount1"
    
    # Burn LP tokens
    self._burn_lp(msg.sender, lp_amount)
    
    # Transfer tokens to user
    ERC20(self.token0).transfer(msg.sender, amount0)
    ERC20(self.token1).transfer(msg.sender, amount1)
    
    # Update reserves
    new_balance0: uint256 = ERC20(self.token0).balanceOf(self)
    new_balance1: uint256 = ERC20(self.token1).balanceOf(self)
    self._update(new_balance0, new_balance1)
    
    log Burn(msg.sender, amount0, amount1, msg.sender)
    
    return amount0, amount1

# ════════════════════════
# SWAP FUNCTION
# ════════════════════════

@external
def swap(
    amount0_out: uint256,
    amount1_out: uint256,
    to: address
):
    """
    Execute Swap
    ต้องส่ง tokens ก่อน แล้วจึงเรียก swap
    (Flash Swap style)
    """
    assert amount0_out > 0 or amount1_out > 0, "Insufficient output amount"
    
    r0: uint256 = self.reserve0
    r1: uint256 = self.reserve1
    
    assert amount0_out < r0, "Insufficient liquidity"
    assert amount1_out < r1, "Insufficient liquidity"
    
    # Transfer output tokens
    if amount0_out > 0:
        ERC20(self.token0).transfer(to, amount0_out)
    if amount1_out > 0:
        ERC20(self.token1).transfer(to, amount1_out)
    
    # Check new balances
    balance0: uint256 = ERC20(self.token0).balanceOf(self)
    balance1: uint256 = ERC20(self.token1).balanceOf(self)
    
    # Calculate input amounts
    amount0_in: uint256 = 0
    if balance0 > r0 - amount0_out:
        amount0_in = balance0 - (r0 - amount0_out)
    
    amount1_in: uint256 = 0
    if balance1 > r1 - amount1_out:
        amount1_in = balance1 - (r1 - amount1_out)
    
    assert amount0_in > 0 or amount1_in > 0, "Insufficient input amount"
    
    # Verify k invariant with fee
    # k must not decrease (adjusting for 0.3% fee)
    balance0_adjusted: uint256 = balance0 * 10000 - amount0_in * FEE_BPS
    balance1_adjusted: uint256 = balance1 * 10000 - amount1_in * FEE_BPS
    
    assert (
        balance0_adjusted * balance1_adjusted >= 
        r0 * r1 * 10000 * 10000
    ), "K invariant violated"
    
    self._update(balance0, balance1)
    
    log Swap(msg.sender, amount0_in, amount1_in, amount0_out, amount1_out, to)

# ════════════════════════
# VIEW FUNCTIONS
# ════════════════════════

@view
@external
def get_reserves() -> (uint256, uint256):
    return self._get_reserves()

@view
@external
def get_amount_out(
    amount_in: uint256,
    is_token0_in: bool
) -> uint256:
    """คำนวณ Amount ที่ได้รับจาก Swap"""
    r0: uint256 = self.reserve0
    r1: uint256 = self.reserve1
    
    if is_token0_in:
        reserve_in: uint256 = r0
        reserve_out: uint256 = r1
    else:
        reserve_in: uint256 = r1
        reserve_out: uint256 = r0
    
    # Apply fee
    amount_in_with_fee: uint256 = amount_in * (10000 - FEE_BPS)
    numerator: uint256 = amount_in_with_fee * reserve_out
    denominator: uint256 = reserve_in * 10000 + amount_in_with_fee
    
    return numerator / denominator

@view
@external
def get_price0() -> uint256:
    """ราคา Token0 ใน Token1 (scaled by PRECISION)"""
    if self.reserve0 == 0:
        return 0
    return (self.reserve1 * PRECISION) / self.reserve0

@view
@external
def get_price1() -> uint256:
    """ราคา Token1 ใน Token0 (scaled by PRECISION)"""
    if self.reserve1 == 0:
        return 0
    return (self.reserve0 * PRECISION) / self.reserve1
```

### Tests

```python
# tests/test_simple_pair.py
import boa
import pytest

@pytest.fixture
def token0():
    # Deploy mock ERC20
    return boa.load("contracts/MockERC20.vy", "Token0", "TK0", 18)

@pytest.fixture
def token1():
    return boa.load("contracts/MockERC20.vy", "Token1", "TK1", 18)

@pytest.fixture
def pair(token0, token1):
    contract = boa.load("contracts/SimplePair.vy")
    contract.initialize(token0.address, token1.address)
    return contract

@pytest.fixture
def lp_provider():
    addr = boa.env.generate_address("lp")
    return addr

def test_add_liquidity(pair, token0, token1, lp_provider):
    amount = 1000 * 10**18
    
    # Mint tokens to lp_provider
    token0.mint(lp_provider, amount)
    token1.mint(lp_provider, amount)
    
    with boa.env.prank(lp_provider):
        token0.approve(pair.address, amount)
        token1.approve(pair.address, amount)
        
        lp_minted = pair.add_liquidity(amount, amount, 0)
    
    assert lp_minted > 0
    r0, r1 = pair.get_reserves()
    assert r0 == amount
    assert r1 == amount
```

---

## สรุป Part 051

ในส่วนนี้คุณได้เรียนรู้:
- ✅ DeFi Fundamentals และ Stack
- ✅ TVL, APY, APR, Price Impact
- ✅ Liquidity Pool และ LP Tokens
- ✅ AMM Theory: x*y=k formula
- ✅ Impermanent Loss
- ✅ Yield Farming Contract
- ✅ Flash Loans
- ✅ DeFi Risk Management
- ✅ Simple AMM Pair Contract

## แบบฝึกหัด

1. **ปรับปรุง** SimplePair ให้มี Protocol Fee (0.05%)
2. **เพิ่ม** TWAP Oracle เข้าไปใน Pair Contract
3. **สร้าง** Router Contract ที่ช่วย Multi-hop Swap
4. **คำนวณ** IL สำหรับ ETH ที่ราคาขึ้น 3x
5. **ทดสอบ** Flash Loan arbitrage scenario

---

**ก่อนหน้า: [Part 050 - Etherscan Verification](part_050_etherscan.md)**  
**ต่อไป: [Part 052 - AMM Implementation](part_052_amm.md)**
