# Part 015: Wei และ Ether Units

## สารบัญ
1. [Wei, Gwei, Ether](#units)
2. [msg.value](#msg-value)
3. [send() vs raw_call()](#send-vs-rawcall)
4. [Receive ETH Patterns](#receive-patterns)
5. [Payable Functions](#payable-functions)
6. [ETH Transfer Security](#security)
7. [ตัวอย่าง: Simple ETH Wallet Contract](#example)

---

## 1. Wei, Gwei, Ether {#units}

ETH มีหน่วยย่อยหลายระดับ ใน Vyper ใช้ constants สำหรับ conversion

```python
# @version 0.4.0

# ════════════════════════════════════════
# ETH Units
# ════════════════════════════════════════

# 1 Ether = 10^18 Wei
# 1 Gwei  = 10^9  Wei

# Vyper unit literals (0.4.0):
# ใช้ as_wei_value() หรือ constants

# Constants
WEI_PER_GWEI: constant(uint256) = 10**9
WEI_PER_ETH: constant(uint256) = 10**18
GWEI_PER_ETH: constant(uint256) = 10**9

# ETH amounts ในหน่วย Wei
ONE_ETH: constant(uint256) = 10**18         # 1 ETH
HALF_ETH: constant(uint256) = 5 * 10**17   # 0.5 ETH
TENTH_ETH: constant(uint256) = 10**17      # 0.1 ETH
MILLI_ETH: constant(uint256) = 10**15      # 0.001 ETH (1 milliether)
GWEI_100: constant(uint256) = 100 * 10**9  # 100 Gwei

@pure
@external
def to_wei(eth: uint256) -> uint256:
    """Convert ETH to Wei"""
    return eth * WEI_PER_ETH

@pure
@external
def to_gwei(eth: uint256) -> uint256:
    """Convert ETH to Gwei"""
    return eth * GWEI_PER_ETH

@pure
@external
def from_wei(wei: uint256) -> uint256:
    """Convert Wei to ETH (integer, truncates)"""
    return wei / WEI_PER_ETH

@pure
@external
def from_gwei(gwei: uint256) -> uint256:
    """Convert Gwei to Wei"""
    return gwei * WEI_PER_GWEI
```

### Unit Conversion ในทางปฏิบัติ

```python
# @version 0.4.0

# ════════════════════════════════════════
# Practical Unit Usage
# ════════════════════════════════════════

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Price Calculations
# ━━━━━━━━━━━━━━━━━━━━━━━━━

@pure
@external
def calculate_eth_value(
    token_amount: uint256,
    price_per_token_wei: uint256  # Price in Wei
) -> uint256:
    """Calculate ETH value of token amount"""
    return (token_amount * price_per_token_wei) / 10**18

@pure
@external
def calculate_gas_cost(
    gas_used: uint256,
    gas_price_gwei: uint256  # Gas price in Gwei
) -> uint256:
    """Calculate transaction cost in Wei"""
    return gas_used * gas_price_gwei * 10**9

@pure
@external
def format_as_gwei(wei_amount: uint256) -> (uint256, uint256):
    """Split wei amount into gwei and remainder"""
    gwei: uint256 = wei_amount / 10**9
    remainder: uint256 = wei_amount % 10**9
    return gwei, remainder

@pure
@external
def format_as_eth_gwei(wei_amount: uint256) -> (uint256, uint256, uint256):
    """Split wei into ETH, Gwei remainder, Wei remainder"""
    eth: uint256 = wei_amount / 10**18
    gwei_rem: uint256 = (wei_amount % 10**18) / 10**9
    wei_rem: uint256 = wei_amount % 10**9
    return eth, gwei_rem, wei_rem
```

### as_wei_value (Vyper Built-in)

```python
# @version 0.4.0

# ════════════════════════════════════════
# as_wei_value() Built-in
# ════════════════════════════════════════

# ใช้สำหรับ convert หน่วยใน code
# as_wei_value(amount, unit) -> uint256

@pure
@external
def demo_as_wei() -> (uint256, uint256, uint256):
    one_eth: uint256 = as_wei_value(1, "ether")    # 10^18
    one_gwei: uint256 = as_wei_value(1, "gwei")    # 10^9
    one_wei: uint256 = as_wei_value(1, "wei")      # 1

    return one_eth, one_gwei, one_wei

# Supported units in as_wei_value:
# "wei"     = 1
# "gwei"    = 10^9
# "ether"   = 10^18

MIN_DEPOSIT: constant(uint256) = as_wei_value(1, "gwei")     # 1 Gwei
MAX_DEPOSIT: constant(uint256) = as_wei_value(100, "ether")  # 100 ETH
FEE: constant(uint256) = as_wei_value(1, "gwei")             # 1 Gwei flat fee
```

---

## 2. msg.value {#msg-value}

`msg.value` คือจำนวน Wei ที่ส่งมาพร้อมกับ transaction

```python
# @version 0.4.0

# ════════════════════════════════════════
# msg.value
# ════════════════════════════════════════

# msg.value = amount of Wei sent with the transaction
# ใช้ได้ใน @payable functions เท่านั้น
# ใน non-payable functions: msg.value = 0 เสมอ

@payable
@external
def accept_eth():
    # msg.value is the Wei sent
    received: uint256 = msg.value
    assert received > 0, "No ETH sent"

@payable
@external
def accept_minimum(minimum: uint256):
    assert msg.value >= minimum, "Insufficient ETH"

@payable
@external
def accept_exact(expected: uint256):
    assert msg.value == expected, "Wrong amount"
    # Exact amount required

# msg.value ไม่สามารถอ่านได้ใน @view/@pure
# @view
# @external
# def bad_view() -> uint256:
#     return msg.value  # Error: msg.value = 0 in view
```

### msg.value Pattern ต่างๆ

```python
# @version 0.4.0

owner: address
prices: HashMap[uint256, uint256]  # item_id -> price in Wei
inventory: HashMap[uint256, uint256]  # item_id -> quantity

event Purchase:
    buyer: indexed(address)
    item_id: indexed(uint256)
    quantity: uint256
    total_paid: uint256
    change: uint256

@deploy
def __init__():
    self.owner = msg.sender
    self.prices[1] = 10**16   # 0.01 ETH
    self.prices[2] = 5 * 10**16  # 0.05 ETH
    self.inventory[1] = 100
    self.inventory[2] = 50

@payable
@external
def buy_item(item_id: uint256, quantity: uint256):
    price: uint256 = self.prices[item_id]
    assert price > 0, "Item not found"
    assert self.inventory[item_id] >= quantity, "Out of stock"
    assert quantity > 0, "Zero quantity"

    total_price: uint256 = price * quantity
    assert msg.value >= total_price, "Insufficient payment"

    # Update inventory
    self.inventory[item_id] -= quantity

    # Return change
    change: uint256 = msg.value - total_price
    if change > 0:
        send(msg.sender, change)

    log Purchase(msg.sender, item_id, quantity, total_price, change)

@view
@external
def get_total_cost(item_id: uint256, quantity: uint256) -> uint256:
    return self.prices[item_id] * quantity
```

---

## 3. send() vs raw_call() {#send-vs-rawcall}

### send()

```python
# @version 0.4.0

# ════════════════════════════════════════
# send() - Simple ETH Transfer
# ════════════════════════════════════════

# send(to, amount):
# - ส่ง Wei ให้ address
# - Gas limit: 2300 (สำหรับ EIP-1884+: ขึ้นอยู่กับ version)
# - Revert ถ้า transfer fail
# - ใช้สำหรับ simple ETH transfer

balances: HashMap[address, uint256]

@deploy
def __init__():
    pass

@payable
@external
def deposit():
    self.balances[msg.sender] += msg.value

@nonreentrant
@external
def withdraw(amount: uint256):
    assert self.balances[msg.sender] >= amount
    self.balances[msg.sender] -= amount
    send(msg.sender, amount)  # Simple, reverts on failure

@external
def send_to_multiple(recipients: DynArray[address, 10], amount: uint256):
    total: uint256 = amount * convert(len(recipients), uint256)
    assert self.balance >= total, "Insufficient contract balance"
    for recipient: address in recipients:
        send(recipient, amount)
```

### raw_call()

```python
# @version 0.4.0

# ════════════════════════════════════════
# raw_call() - Low-Level ETH Transfer
# ════════════════════════════════════════

# raw_call(to, data, value=amount, max_outsize=N, revert_on_failure=bool)
# - Low-level call
# - ปรับ gas limit ได้
# - สามารถ handle failure โดยไม่ revert

@nonreentrant
@external
def safe_transfer(to: address, amount: uint256) -> bool:
    """Transfer ETH, return success/failure without reverting"""
    success: bool = raw_call(
        to,
        b"",
        value=amount,
        revert_on_failure=False  # ไม่ revert ถ้า fail
    )
    return success

@nonreentrant
@external
def call_contract(
    target: address,
    data: Bytes[256],
    eth_amount: uint256
) -> Bytes[256]:
    """Call contract with data and ETH"""
    response: Bytes[256] = raw_call(
        target,
        data,
        value=eth_amount,
        max_outsize=256
    )
    return response

# static call (เหมือน @view)
@view
@external
def static_read(target: address, data: Bytes[64]) -> Bytes[32]:
    response: Bytes[32] = raw_call(
        target,
        data,
        max_outsize=32,
        is_static_call=True  # Cannot modify state
    )
    return response
```

### send() vs raw_call() Comparison

```python
# @version 0.4.0

# ════════════════════════════════════════
# send() vs raw_call() Comparison
# ════════════════════════════════════════

# send():
# ✅ Simple, readable
# ✅ Reverts on failure (safe default)
# ❌ Cannot handle failure gracefully
# ❌ Less control over gas

# raw_call():
# ✅ Can handle failures
# ✅ Can pass data to called contract
# ✅ Can do static calls
# ❌ More complex
# ❌ Need to handle return values

balances: HashMap[address, uint256]
failed_transfers: HashMap[address, uint256]  # Fallback for failed sends

@deploy
def __init__():
    pass

@payable
@external
def deposit():
    self.balances[msg.sender] += msg.value

@nonreentrant
@external
def withdraw(amount: uint256):
    assert self.balances[msg.sender] >= amount
    self.balances[msg.sender] -= amount

    # Try raw_call first, fallback to record
    success: bool = raw_call(
        msg.sender,
        b"",
        value=amount,
        revert_on_failure=False
    )

    if not success:
        # Record for later retry
        self.failed_transfers[msg.sender] += amount
        self.balances[msg.sender] += amount  # Restore

@nonreentrant
@external
def retry_failed_transfer():
    amount: uint256 = self.failed_transfers[msg.sender]
    assert amount > 0, "No failed transfer"
    self.failed_transfers[msg.sender] = 0
    send(msg.sender, amount)  # Will revert if still fails
```

---

## 4. Receive ETH Patterns {#receive-patterns}

```python
# @version 0.4.0

# ════════════════════════════════════════
# Receive ETH Patterns
# ════════════════════════════════════════

# Pattern 1: @default (fallback)
@payable
@external
def __default__():
    pass  # Accept any ETH sent without function call

# Pattern 2: Specific deposit function
@payable
@external
def deposit():
    pass  # More explicit and documentable

# Pattern 3: Reject ETH
@external
def __default__():
    pass  # No @payable = reject ETH

# Pattern 4: Accept and track
owner: address
received: HashMap[address, uint256]
total_received: uint256

event ETHReceived:
    sender: indexed(address)
    amount: uint256

@deploy
def __init__():
    self.owner = msg.sender

@payable
@external
def __default__():
    assert msg.value > 0, "Zero ETH"
    self.received[msg.sender] += msg.value
    self.total_received += msg.value
    log ETHReceived(msg.sender, msg.value)
```

### Accepting ETH from Contracts

```python
# @version 0.4.0

# ════════════════════════════════════════
# Receiving ETH from Smart Contracts
# ════════════════════════════════════════

# Vyper contracts ต้องมี @payable + @default
# หรือ @payable function เพื่อรับ ETH จาก contracts อื่น

# ❌ Contract นี้จะ reject ETH จาก send/transfer
# (ไม่มี @default หรือ @payable function)

owner: address
total: uint256

@deploy
def __init__():
    self.owner = msg.sender

# ✅ เพิ่ม @default เพื่อรับ ETH จาก contract อื่น
@payable
@external
def __default__():
    self.total += msg.value

# สามารถรับ ETH จาก:
# - EOA direct transfer
# - selfdestruct() (ไม่สามารถป้องกันได้)
# - block reward (สำหรับ coinbase)
# - send()/transfer() จาก contract อื่น
```

---

## 5. Payable Functions {#payable-functions}

```python
# @version 0.4.0

# ════════════════════════════════════════
# Payable Function Patterns
# ════════════════════════════════════════

struct Purchase:
    buyer: address
    item_id: uint256
    quantity: uint256
    amount_paid: uint256
    timestamp: uint256

owner: address
item_prices: HashMap[uint256, uint256]
purchases: DynArray[Purchase, 10000]
revenue: uint256

event Purchased:
    buyer: indexed(address)
    item_id: indexed(uint256)
    quantity: uint256
    amount_paid: uint256

@deploy
def __init__():
    self.owner = msg.sender

@external
def set_price(item_id: uint256, price: uint256):
    assert msg.sender == self.owner
    self.item_prices[item_id] = price

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Pattern: Exact Payment Required
# ━━━━━━━━━━━━━━━━━━━━━━━━━
@payable
@external
def buy_exact(item_id: uint256, quantity: uint256):
    price: uint256 = self.item_prices[item_id]
    total: uint256 = price * quantity
    assert msg.value == total, "Exact payment required"

    self.purchases.append(Purchase({
        buyer: msg.sender,
        item_id: item_id,
        quantity: quantity,
        amount_paid: msg.value,
        timestamp: block.timestamp
    }))
    self.revenue += msg.value
    log Purchased(msg.sender, item_id, quantity, msg.value)

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Pattern: Overpayment with Refund
# ━━━━━━━━━━━━━━━━━━━━━━━━━
@nonreentrant
@payable
@external
def buy_with_refund(item_id: uint256, quantity: uint256):
    price: uint256 = self.item_prices[item_id]
    total: uint256 = price * quantity
    assert msg.value >= total, "Insufficient payment"

    # Process purchase
    self.purchases.append(Purchase({
        buyer: msg.sender,
        item_id: item_id,
        quantity: quantity,
        amount_paid: total,
        timestamp: block.timestamp
    }))
    self.revenue += total

    # Refund excess
    if msg.value > total:
        send(msg.sender, msg.value - total)

    log Purchased(msg.sender, item_id, quantity, total)

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Pattern: Dynamic Pricing
# ━━━━━━━━━━━━━━━━━━━━━━━━━
bonding_curve_supply: uint256

@view
@internal
def _current_price() -> uint256:
    # Linear bonding curve: price increases with supply
    base: uint256 = 10**15  # 0.001 ETH base
    increment: uint256 = self.bonding_curve_supply * 10**13
    return base + increment

@nonreentrant
@payable
@external
def buy_bonding_curve():
    current_price: uint256 = self._current_price()
    assert msg.value >= current_price, "Price too low"

    self.bonding_curve_supply += 1
    self.revenue += current_price

    # Refund excess
    if msg.value > current_price:
        send(msg.sender, msg.value - current_price)

@view
@external
def get_current_price() -> uint256:
    return self._current_price()
```

---

## 6. ETH Transfer Security {#security}

```python
# @version 0.4.0

# ════════════════════════════════════════
# ETH Transfer Security Best Practices
# ════════════════════════════════════════

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# 1. Always use CEI Pattern
# ━━━━━━━━━━━━━━━━━━━━━━━━━
balances: HashMap[address, uint256]

@deploy
def __init__():
    pass

@payable
@external
def deposit():
    self.balances[msg.sender] += msg.value

# ✅ CEI: Checks → Effects → Interactions
@nonreentrant
@external
def withdraw_safe(amount: uint256):
    # CHECKS
    assert amount > 0, "Zero amount"
    assert self.balances[msg.sender] >= amount, "Insufficient"

    # EFFECTS (update state before sending)
    self.balances[msg.sender] -= amount

    # INTERACTIONS (send last)
    send(msg.sender, amount)

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# 2. Handle Failed Transfers
# ━━━━━━━━━━━━━━━━━━━━━━━━━
pending: HashMap[address, uint256]

@nonreentrant
@external
def send_with_fallback(to: address, amount: uint256):
    """Graceful failure handling"""
    success: bool = raw_call(to, b"", value=amount, revert_on_failure=False)
    if not success:
        # Store for later manual withdrawal
        self.pending[to] += amount

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# 3. Avoid Sending ETH in Loops
# ━━━━━━━━━━━━━━━━━━━━━━━━━

# ❌ Dangerous: If one send fails, all fail
# @external
# def bad_distribute(recipients: DynArray[address, 10], amount: uint256):
#     for r in recipients:
#         send(r, amount)  # If any fails, all revert!

# ✅ Better: Pull pattern or handle failures
claimable: HashMap[address, uint256]

@external
def distribute_safe(recipients: DynArray[address, 10], amount: uint256):
    """Push to pending, let users pull"""
    for r: address in recipients:
        self.claimable[r] += amount

@nonreentrant
@external
def claim():
    amount: uint256 = self.claimable[msg.sender]
    assert amount > 0, "Nothing to claim"
    self.claimable[msg.sender] = 0
    send(msg.sender, amount)

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# 4. Gas Considerations for Calls
# ━━━━━━━━━━━━━━━━━━━━━━━━━

# send() ใน Vyper ส่ง ETH พร้อม stipend gas
# ถ้า recipient เป็น contract จะต้องมี receive function ที่ใช้ gas น้อย

@nonreentrant
@external
def smart_send(to: address, amount: uint256):
    """Send with enough gas for receive()"""
    raw_call(
        to,
        b"",
        value=amount,
        # gas=50000,  # Can specify gas if needed
        revert_on_failure=True
    )

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# 5. Validate Amount
# ━━━━━━━━━━━━━━━━━━━━━━━━━

MIN_AMOUNT: constant(uint256) = 10**14  # 0.0001 ETH
MAX_AMOUNT: constant(uint256) = 10**21  # 1000 ETH

@payable
@external
def validated_deposit():
    assert msg.value >= MIN_AMOUNT, "Below minimum"
    assert msg.value <= MAX_AMOUNT, "Above maximum"
    self.balances[msg.sender] += msg.value
```

### Anti-Patterns to Avoid

```python
# @version 0.4.0

# ════════════════════════════════════════
# ETH Anti-Patterns
# ════════════════════════════════════════

# ❌ 1. Sending to zero address
# send(empty(address), amount)  # Will revert in Vyper but still check

# ❌ 2. Not checking balance before send
# send(recipient, 999 * 10**18)  # May revert if insufficient balance

# ❌ 3. Integer overflow in amount calculation
# amount: uint256 = price * quantity  # Could overflow for large values

# ❌ 4. Sending ETH then updating state
# send(msg.sender, amount)  # Interaction before effects
# self.balances[msg.sender] -= amount  # Too late!

# ✅ Correct patterns:
owner: address
balances: HashMap[address, uint256]

@deploy
def __init__():
    self.owner = msg.sender

@nonreentrant
@external
def correct_withdraw(amount: uint256):
    # Check
    assert amount > 0 and amount <= self.balance, "Invalid amount"
    assert self.balances[msg.sender] >= amount, "Insufficient"
    # Effects first
    self.balances[msg.sender] -= amount
    # Interaction last
    send(msg.sender, amount)
```

---

## 7. ตัวอย่าง: Simple ETH Wallet Contract {#example}

Wallet ที่จัดการ ETH ส่วนตัว พร้อม Security ครบถ้วน

```python
# @version 0.4.0

# ════════════════════════════════════════════════════════════
# SimpleETHWallet Contract
# Personal ETH Wallet ที่ปลอดภัยและใช้งานได้จริง
# ════════════════════════════════════════════════════════════

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Constants & Immutables
# ━━━━━━━━━━━━━━━━━━━━━━━━━
MIN_DEPOSIT: constant(uint256) = as_wei_value(1, "gwei")     # 1 Gwei min
MAX_SINGLE_WITHDRAWAL: constant(uint256) = as_wei_value(10, "ether")  # 10 ETH max per tx
DAILY_LIMIT_DEFAULT: constant(uint256) = as_wei_value(1, "ether")   # 1 ETH default daily

OWNER: immutable(address)
CREATED_AT: immutable(uint256)

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Structs
# ━━━━━━━━━━━━━━━━━━━━━━━━━

struct Transaction:
    to: address
    amount: uint256
    timestamp: uint256
    note: String[100]
    tx_type: uint8  # 0=deposit, 1=withdrawal, 2=transfer

struct DailyUsage:
    amount: uint256
    day: uint256

struct WalletConfig:
    daily_limit: uint256
    require_note: bool
    min_deposit: uint256
    max_withdrawal: uint256

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# State Variables
# ━━━━━━━━━━━━━━━━━━━━━━━━━

config: WalletConfig
daily_spent: DailyUsage
transaction_history: DynArray[Transaction, 1000]
is_locked: bool
lock_until: uint256

# Authorized operators
operators: HashMap[address, bool]
operator_limits: HashMap[address, uint256]  # per-tx limit

# Statistics
total_deposited: uint256
total_withdrawn: uint256

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Events
# ━━━━━━━━━━━━━━━━━━━━━━━━━

event Deposited:
    from_addr: indexed(address)
    amount: uint256
    balance: uint256
    note: String[100]

event Withdrawn:
    to: indexed(address)
    amount: uint256
    balance: uint256
    note: String[100]

event Transferred:
    to: indexed(address)
    amount: uint256
    note: String[100]

event Locked:
    until: uint256

event Unlocked:
    by: indexed(address)

event OperatorAdded:
    operator: indexed(address)
    limit: uint256

event OperatorRemoved:
    operator: indexed(address)

event ConfigUpdated:
    daily_limit: uint256

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Constructor
# ━━━━━━━━━━━━━━━━━━━━━━━━━

@payable
@deploy
def __init__(daily_limit: uint256):
    OWNER = msg.sender
    CREATED_AT = block.timestamp

    # Default config
    self.config = WalletConfig({
        daily_limit: daily_limit if daily_limit > 0 else DAILY_LIMIT_DEFAULT,
        require_note: False,
        min_deposit: MIN_DEPOSIT,
        max_withdrawal: MAX_SINGLE_WITHDRAWAL
    })

    # Record initial deposit if any
    if msg.value > 0:
        self.total_deposited += msg.value
        self.transaction_history.append(Transaction({
            to: OWNER,
            amount: msg.value,
            timestamp: block.timestamp,
            note: "Initial deposit",
            tx_type: 0
        }))
        log Deposited(OWNER, msg.value, self.balance, "Initial deposit")

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Internal Helpers
# ━━━━━━━━━━━━━━━━━━━━━━━━━

@internal
def _require_owner():
    assert msg.sender == OWNER, "Not owner"

@internal
def _require_authorized(max_amount: uint256):
    if msg.sender != OWNER:
        assert self.operators[msg.sender], "Not authorized"
        limit: uint256 = self.operator_limits[msg.sender]
        assert max_amount <= limit, "Exceeds operator limit"

@internal
def _check_not_locked():
    assert not self.is_locked, "Wallet locked"
    if self.lock_until > 0:
        assert block.timestamp >= self.lock_until, "Timelock active"

@view
@internal
def _today() -> uint256:
    return block.timestamp / 86400

@internal
def _check_daily_limit(amount: uint256):
    today: uint256 = self._today()
    if self.daily_spent.day != today:
        # New day, reset
        self.daily_spent.amount = 0
        self.daily_spent.day = today
    assert self.daily_spent.amount + amount <= self.config.daily_limit, "Daily limit exceeded"
    self.daily_spent.amount += amount

@internal
def _record_tx(to: address, amount: uint256, note: String[100], tx_type: uint8):
    if convert(len(self.transaction_history), uint256) < 1000:
        self.transaction_history.append(Transaction({
            to: to,
            amount: amount,
            timestamp: block.timestamp,
            note: note,
            tx_type: tx_type
        }))

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Deposit Functions
# ━━━━━━━━━━━━━━━━━━━━━━━━━

@nonreentrant
@payable
@external
def deposit(note: String[100]):
    """Deposit ETH into wallet"""
    assert msg.value >= self.config.min_deposit, "Below minimum"
    if self.config.require_note:
        assert len(note) > 0, "Note required"

    self.total_deposited += msg.value
    self._record_tx(msg.sender, msg.value, note, 0)

    log Deposited(msg.sender, msg.value, self.balance, note)

@payable
@external
def __default__():
    """Accept plain ETH transfers"""
    if msg.value >= self.config.min_deposit:
        self.total_deposited += msg.value
        self._record_tx(msg.sender, msg.value, "Direct transfer", 0)
        log Deposited(msg.sender, msg.value, self.balance, "Direct transfer")

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Withdrawal Functions
# ━━━━━━━━━━━━━━━━━━━━━━━━━

@nonreentrant
@external
def withdraw(amount: uint256, note: String[100]):
    """Withdraw ETH from wallet"""
    self._check_not_locked()
    self._require_authorized(amount)

    # Validate
    assert amount > 0, "Zero amount"
    assert amount <= self.config.max_withdrawal, "Exceeds max per tx"
    assert amount <= self.balance, "Insufficient balance"
    self._check_daily_limit(amount)

    if self.config.require_note:
        assert len(note) > 0, "Note required"

    # Record
    self.total_withdrawn += amount
    self._record_tx(msg.sender, amount, note, 1)
    remaining: uint256 = self.balance - amount  # Calculate before send

    # Transfer
    send(OWNER, amount)

    log Withdrawn(OWNER, amount, remaining, note)

@nonreentrant
@external
def withdraw_all(note: String[100]):
    """Withdraw entire balance"""
    self._require_owner()
    self._check_not_locked()

    amount: uint256 = self.balance
    assert amount > 0, "Empty wallet"

    self.total_withdrawn += amount
    self._record_tx(OWNER, amount, note, 1)

    send(OWNER, amount)
    log Withdrawn(OWNER, amount, 0, note)

@nonreentrant
@external
def send_to(to: address, amount: uint256, note: String[100]):
    """Send ETH to another address"""
    self._require_owner()
    self._check_not_locked()

    assert to != empty(address), "Invalid recipient"
    assert to != self, "Cannot send to self"
    assert amount > 0, "Zero amount"
    assert amount <= self.balance, "Insufficient"
    self._check_daily_limit(amount)

    self._record_tx(to, amount, note, 2)

    # Use raw_call to handle contracts
    success: bool = raw_call(to, b"", value=amount, revert_on_failure=False)
    if not success:
        # Don't revert, but record failure
        self._record_tx(to, amount, "FAILED", 2)
        raise "Transfer to recipient failed"

    log Transferred(to, amount, note)

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Security Functions
# ━━━━━━━━━━━━━━━━━━━━━━━━━

@external
def lock(duration: uint256):
    """Lock wallet for duration seconds"""
    self._require_owner()
    self.is_locked = True
    if duration > 0:
        self.lock_until = block.timestamp + duration
    log Locked(self.lock_until)

@external
def unlock():
    """Unlock wallet"""
    self._require_owner()
    assert self.is_locked, "Not locked"
    if self.lock_until > 0:
        assert block.timestamp >= self.lock_until, "Timelock not expired"
    self.is_locked = False
    self.lock_until = 0
    log Unlocked(msg.sender)

@external
def add_operator(operator: address, limit: uint256):
    """Add operator with per-tx ETH limit"""
    self._require_owner()
    assert operator != empty(address), "Invalid"
    assert operator != OWNER, "Owner is already authorized"
    assert limit > 0, "Zero limit"
    self.operators[operator] = True
    self.operator_limits[operator] = limit
    log OperatorAdded(operator, limit)

@external
def remove_operator(operator: address):
    """Remove operator"""
    self._require_owner()
    self.operators[operator] = False
    self.operator_limits[operator] = 0
    log OperatorRemoved(operator)

@external
def update_config(
    daily_limit: uint256,
    require_note: bool,
    min_dep: uint256,
    max_withdrawal: uint256
):
    """Update wallet configuration"""
    self._require_owner()
    assert daily_limit > 0, "Zero daily limit"
    assert max_withdrawal > 0 and max_withdrawal <= MAX_SINGLE_WITHDRAWAL
    self.config.daily_limit = daily_limit
    self.config.require_note = require_note
    self.config.min_deposit = min_dep
    self.config.max_withdrawal = max_withdrawal
    log ConfigUpdated(daily_limit)

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# View Functions
# ━━━━━━━━━━━━━━━━━━━━━━━━━

@view
@external
def get_balance() -> uint256:
    return self.balance

@view
@external
def get_owner() -> address:
    return OWNER

@view
@external
def get_config() -> WalletConfig:
    return self.config

@view
@external
def get_daily_remaining() -> uint256:
    today: uint256 = self._today()
    if self.daily_spent.day != today:
        return self.config.daily_limit
    spent: uint256 = self.daily_spent.amount
    if spent >= self.config.daily_limit:
        return 0
    return self.config.daily_limit - spent

@view
@external
def is_locked_status() -> (bool, uint256):
    return self.is_locked, self.lock_until

@view
@external
def get_stats() -> (uint256, uint256, uint256):
    return self.total_deposited, self.total_withdrawn, self.balance

@view
@external
def get_transaction_count() -> uint256:
    return convert(len(self.transaction_history), uint256)

@view
@external
def get_recent_transactions(count: uint256) -> DynArray[Transaction, 20]:
    assert count <= 20, "Max 20"
    result: DynArray[Transaction, 20] = []
    n: uint256 = convert(len(self.transaction_history), uint256)
    if n == 0:
        return result
    start: uint256 = 0
    if n > count:
        start = n - count
    for i: uint256 in range(start, start + 20, bound=20):
        if i >= n:
            break
        result.append(self.transaction_history[i])
    return result

@view
@external
def is_operator(addr: address) -> bool:
    return self.operators[addr]

@view
@external
def operator_limit(addr: address) -> uint256:
    return self.operator_limits[addr]

@view
@external
def get_created_at() -> uint256:
    return CREATED_AT
```

### Test Code

```python
# tests/test_eth_wallet.py
import pytest
from brownie import SimpleETHWallet, accounts, Wei, reverts, chain

DAILY_LIMIT = Wei("1 ether")

@pytest.fixture
def wallet(accounts):
    return SimpleETHWallet.deploy(
        DAILY_LIMIT,
        {'from': accounts[0], 'value': Wei("5 ether")}
    )

class TestDeposit:
    def test_initial_deposit(self, wallet, accounts):
        balance = wallet.get_balance()
        assert balance == Wei("5 ether")

    def test_deposit(self, wallet, accounts):
        wallet.deposit("Test deposit", {'from': accounts[1], 'value': Wei("1 ether")})
        assert wallet.get_balance() == Wei("6 ether")

    def test_below_min_reverts(self, wallet, accounts):
        with reverts("Below minimum"):
            wallet.deposit("test", {'from': accounts[1], 'value': 1})

class TestWithdrawal:
    def test_withdraw(self, wallet, accounts):
        initial = accounts[0].balance()
        wallet.withdraw(Wei("0.5 ether"), "Test", {'from': accounts[0]})
        # Approximately: balance increased by 0.5 ETH (minus gas)
        assert accounts[0].balance() > initial

    def test_daily_limit_enforced(self, wallet, accounts):
        wallet.withdraw(DAILY_LIMIT, "Max", {'from': accounts[0]})
        with reverts("Daily limit exceeded"):
            wallet.withdraw(1, "Over limit", {'from': accounts[0]})

    def test_daily_resets(self, wallet, accounts):
        wallet.withdraw(DAILY_LIMIT, "Today", {'from': accounts[0]})
        chain.sleep(86400 + 1)  # Skip to next day
        chain.mine()
        # Should succeed on next day
        wallet.withdraw(DAILY_LIMIT, "Next day", {'from': accounts[0]})

class TestSecurity:
    def test_lock_prevents_withdrawal(self, wallet, accounts):
        wallet.lock(3600, {'from': accounts[0]})  # Lock 1 hour
        with reverts("Timelock active"):
            wallet.withdraw(Wei("0.1 ether"), "Locked", {'from': accounts[0]})

    def test_unlock_after_duration(self, wallet, accounts):
        wallet.lock(100, {'from': accounts[0]})
        chain.sleep(101)
        chain.mine()
        wallet.unlock({'from': accounts[0]})
        wallet.withdraw(Wei("0.1 ether"), "Unlocked", {'from': accounts[0]})

class TestOperator:
    def test_operator_can_withdraw(self, wallet, accounts):
        limit = Wei("0.1 ether")
        wallet.add_operator(accounts[1], limit, {'from': accounts[0]})
        wallet.withdraw(limit, "Operator withdraw", {'from': accounts[1]})

    def test_operator_cannot_exceed_limit(self, wallet, accounts):
        limit = Wei("0.1 ether")
        wallet.add_operator(accounts[1], limit, {'from': accounts[0]})
        with reverts("Exceeds operator limit"):
            wallet.withdraw(limit + 1, "Too much", {'from': accounts[1]})

class TestStats:
    def test_stats_tracking(self, wallet, accounts):
        wallet.deposit("add", {'from': accounts[1], 'value': Wei("1 ether")})
        wallet.withdraw(Wei("0.5 ether"), "remove", {'from': accounts[0]})
        deposited, withdrawn, balance = wallet.get_stats()
        assert deposited == Wei("6 ether")  # 5 initial + 1
        assert withdrawn == Wei("0.5 ether")
```

---

## สรุป

- ✅ **Wei = smallest unit**: 1 ETH = 10^18 Wei
- ✅ **as_wei_value()**: แปลงหน่วยใน code
- ✅ **msg.value**: ETH ที่ส่งมาพร้อม transaction (Wei)
- ✅ **send()**: ส่ง ETH แบบง่าย, revert ถ้า fail
- ✅ **raw_call()**: Low-level, handle failure ได้
- ✅ **CEI + @nonreentrant**: Double protection สำหรับ ETH transfers
- ✅ **Pull Payment**: ให้ user ดึงเงินเอง แทน contract ส่งให้

## แบบฝึกหัด

1. **สร้าง** Multi-sig Wallet ที่ต้องการ approval 2/3 ก่อน withdraw
2. **เพิ่ม** Gas Price Oracle เพื่อ estimate fee ก่อน transaction
3. **สร้าง** Recurring Payment Contract (weekly/monthly)
4. **ทดสอบ** Reentrancy attack บน Wallet ที่ไม่ได้ป้องกัน
5. **สร้าง** Emergency Recovery System พร้อม Timelock

---

**ก่อนหน้า: [Part 014 - Modifiers และ @view/@pure](part_014_decorators.md)**  
**ต่อไป: [Part 016 - Interfaces](part_016_interfaces.md)** *(เร็วๆ นี้)*
