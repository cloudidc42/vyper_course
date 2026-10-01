# Part 008: Loops: For Loop

## สารบัญ
1. [For Loop พื้นฐาน](#for-basics)
2. [for i in range()](#for-range)
3. [for item in array](#for-array)
4. [Bounded Loops](#bounded)
5. [ไม่มี break ใน Vyper](#no-break)
6. [Loop Patterns](#patterns)
7. [Gas Considerations](#gas)
8. [ตัวอย่าง: Batch Transfer Contract](#example)

---

## 1. For Loop พื้นฐาน {#for-basics}

Vyper มีเฉพาะ `for` loop ไม่มี `while` loop เพื่อความปลอดภัยด้านก๊าซ

```python
# @version 0.4.0

# ════════════════════════════════════════
# Basic For Loop Syntax
# ════════════════════════════════════════

@pure
@external
def sum_range(n: uint256) -> uint256:
    total: uint256 = 0
    for i: uint256 in range(n, bound=100):
        total += i
    return total

@pure
@external
def sum_fixed() -> uint256:
    total: uint256 = 0
    for i: uint256 in range(10):  # 0 to 9
        total += i
    return total  # = 45
```

### For Loop vs Python

```python
# @version 0.4.0

# Python: for i in range(10): ...
# Vyper: for i: uint256 in range(10): ...

# ต้องระบุ type ของ loop variable เสมอ
# ต้องมี bound=N เมื่อ range argument ไม่ใช่ literal

@pure
@external
def demo_types() -> uint256:
    # uint256 loop variable
    sum1: uint256 = 0
    for i: uint256 in range(5):
        sum1 += i

    # int128 loop variable
    sum2: int128 = 0
    for j: int128 in range(5):
        sum2 += j

    return sum1
```

---

## 2. for i in range() {#for-range}

```python
# @version 0.4.0

# ════════════════════════════════════════
# range() Variations
# ════════════════════════════════════════

@pure
@external
def range_basic() -> uint256:
    # range(N): 0 to N-1
    total: uint256 = 0
    for i: uint256 in range(5):     # i = 0, 1, 2, 3, 4
        total += i
    return total  # 0+1+2+3+4 = 10

@pure
@external
def range_start_stop(start: uint256, stop: uint256) -> uint256:
    # range(start, stop): start to stop-1
    total: uint256 = 0
    for i: uint256 in range(start, stop, bound=100):
        total += i
    return total

@pure
@external
def range_fixed_start() -> uint256:
    # range(5, 10): 5, 6, 7, 8, 9
    total: uint256 = 0
    for i: uint256 in range(5, 10):
        total += i
    return total  # 5+6+7+8+9 = 35

@pure
@external
def count_down() -> int128:
    # ใช้ int128 สำหรับ countdown
    result: int128 = 0
    for i: int128 in range(10, 0, bound=10):
        result = i  # จะได้ 10, 9, 8, ..., 1
    return result  # return สุดท้าย = 1
```

### range() กับ Dynamic Values

```python
# @version 0.4.0

MAX_ITERATIONS: constant(uint256) = 100

@pure
@external
def sum_to_n(n: uint256) -> uint256:
    assert n <= MAX_ITERATIONS, "n too large"
    total: uint256 = 0
    # bound= กำหนด upper bound สำหรับ compiler
    for i: uint256 in range(n, bound=MAX_ITERATIONS):
        total += i
    return total

@pure
@external
def sum_range_dynamic(start: uint256, count: uint256) -> uint256:
    assert count <= 50, "Too many iterations"
    total: uint256 = 0
    for i: uint256 in range(start, start + count, bound=50):
        total += i
    return total

@pure
@external
def factorial(n: uint256) -> uint256:
    assert n <= 20, "n too large (overflow risk)"
    result: uint256 = 1
    for i: uint256 in range(1, n + 1, bound=21):
        result *= i
    return result

@pure
@external
def fibonacci(n: uint256) -> uint256:
    assert n <= 50, "n too large"
    if n == 0:
        return 0
    if n == 1:
        return 1
    a: uint256 = 0
    b: uint256 = 1
    for i: uint256 in range(2, n + 1, bound=51):
        c: uint256 = a + b
        a = b
        b = c
    return b
```

---

## 3. for item in array {#for-array}

```python
# @version 0.4.0

# ════════════════════════════════════════
# Iterating Arrays
# ════════════════════════════════════════

@pure
@external
def sum_array(values: uint256[5]) -> uint256:
    total: uint256 = 0
    for v: uint256 in values:
        total += v
    return total

@pure
@external
def sum_dynamic(values: DynArray[uint256, 50]) -> uint256:
    total: uint256 = 0
    for v: uint256 in values:
        total += v
    return total

@pure
@external
def find_max(values: DynArray[uint256, 100]) -> uint256:
    assert len(values) > 0, "Empty array"
    max_val: uint256 = values[0]
    for v: uint256 in values:
        if v > max_val:
            max_val = v
    return max_val

@pure
@external
def find_min(values: DynArray[uint256, 100]) -> uint256:
    assert len(values) > 0, "Empty array"
    min_val: uint256 = values[0]
    for v: uint256 in values:
        if v < min_val:
            min_val = v
    return min_val

@pure
@external
def count_above_threshold(
    values: DynArray[uint256, 100],
    threshold: uint256
) -> uint256:
    count: uint256 = 0
    for v: uint256 in values:
        if v > threshold:
            count += 1
    return count

@pure
@external
def all_positive(values: DynArray[uint256, 50]) -> bool:
    for v: uint256 in values:
        if v == 0:
            return False
    return True
```

### Iterating Struct Arrays

```python
# @version 0.4.0

struct Contribution:
    contributor: address
    amount: uint256
    timestamp: uint256

contributions: DynArray[Contribution, 200]

@deploy
def __init__():
    pass

@payable
@external
def contribute():
    assert msg.value > 0, "Must send ETH"
    self.contributions.append(Contribution({
        contributor: msg.sender,
        amount: msg.value,
        timestamp: block.timestamp
    }))

@view
@external
def total_contributions() -> uint256:
    total: uint256 = 0
    for c: Contribution in self.contributions:
        total += c.amount
    return total

@view
@external
def get_contributor_total(contributor: address) -> uint256:
    total: uint256 = 0
    for c: Contribution in self.contributions:
        if c.contributor == contributor:
            total += c.amount
    return total

@view
@external
def get_largest_contributor() -> address:
    if len(self.contributions) == 0:
        return empty(address)
    best_addr: address = self.contributions[0].contributor
    best_amount: uint256 = 0
    for c: Contribution in self.contributions:
        if c.amount > best_amount:
            best_amount = c.amount
            best_addr = c.contributor
    return best_addr
```

---

## 4. Bounded Loops {#bounded}

Vyper ต้องรู้ upper bound ของ loop ณ Compile time เพื่อความปลอดภัย

```python
# @version 0.4.0

# ════════════════════════════════════════
# Bounded Loops - Required in Vyper
# ════════════════════════════════════════

# bound= parameter บอก compiler ว่า loop จะวนสูงสุดกี่รอบ
# ใช้กับ range(dynamic_value)

MAX_USERS: constant(uint256) = 500
MAX_TOKENS: constant(uint256) = 1000
MAX_ROUNDS: constant(uint256) = 52  # 52 weeks

users: DynArray[address, 500]
balances: HashMap[address, uint256]
reward_per_round: uint256

@deploy
def __init__(reward: uint256):
    self.reward_per_round = reward

@external
def distribute_rewards(rounds: uint256):
    assert rounds <= MAX_ROUNDS, "Too many rounds"
    total_users: uint256 = convert(len(self.users), uint256)

    for r: uint256 in range(rounds, bound=MAX_ROUNDS):
        for i: uint256 in range(total_users, bound=MAX_USERS):
            user: address = self.users[i]
            self.balances[user] += self.reward_per_round

# ✅ Compile-time known bound (no bound= needed)
@pure
@external
def sum_first_ten() -> uint256:
    total: uint256 = 0
    for i: uint256 in range(10):  # 10 is literal = OK
        total += i
    return total

# ✅ Dynamic bound with bound= parameter
@pure
@external
def sum_up_to(n: uint256) -> uint256:
    assert n <= 100, "Too large"
    total: uint256 = 0
    for i: uint256 in range(n, bound=100):  # bound=100 tells compiler
        total += i
    return total

# ❌ ไม่ compile ได้:
# @pure
# @external
# def bad_loop(n: uint256) -> uint256:
#     for i: uint256 in range(n):  # Error! n ไม่รู้ bound
#         pass
#     return 0
```

---

## 5. ไม่มี break ใน Vyper {#no-break}

Vyper ไม่มี `break`, `continue`, หรือ `while` loop เพื่อป้องกัน Infinite Loop

```python
# @version 0.4.0

# ════════════════════════════════════════
# No break/continue - Workarounds
# ════════════════════════════════════════

# ❌ ไม่มี break ใน Vyper:
# for i: uint256 in range(10):
#     if i == 5:
#         break  # Error!

# ✅ Workaround 1: Flag variable
@pure
@external
def find_index(values: DynArray[uint256, 50], target: uint256) -> int256:
    found_at: int256 = -1
    for i: uint256 in range(50, bound=50):
        if i >= len(values):
            break  # Vyper 0.4.0 supports break!
        if values[i] == target and found_at == -1:
            found_at = convert(i, int256)
    return found_at

# ✅ Vyper 0.4.0 มี break ใน for loop!
@pure
@external
def find_first(values: DynArray[uint256, 100], target: uint256) -> int256:
    for i: uint256 in range(100, bound=100):
        if i >= len(values):
            break
        if values[i] == target:
            return convert(i, int256)
    return -1

# ✅ Early return pattern
@pure
@external
def contains(values: DynArray[address, 100], target: address) -> bool:
    for addr: address in values:
        if addr == target:
            return True  # ออกจาก function ทันที
    return False

# ✅ Skip pattern (simulate continue)
@pure
@external
def sum_even(values: DynArray[uint256, 50]) -> uint256:
    total: uint256 = 0
    for v: uint256 in values:
        if v % 2 != 0:
            continue  # Vyper 0.4.0 supports continue!
        total += v
    return total

# ✅ Flag pattern (pre-0.4.0 compatible)
@pure
@external
def find_first_above(values: DynArray[uint256, 100], threshold: uint256) -> uint256:
    result: uint256 = 0
    found: bool = False
    for v: uint256 in values:
        if not found and v > threshold:
            result = v
            found = True
    return result
```

---

## 6. Loop Patterns {#patterns}

### Pattern: Accumulator

```python
# @version 0.4.0

struct Vote:
    voter: address
    candidate: uint256
    weight: uint256

votes: DynArray[Vote, 1000]

@view
@external
def tally_votes(num_candidates: uint256) -> DynArray[uint256, 10]:
    assert num_candidates <= 10, "Too many candidates"
    tallies: DynArray[uint256, 10] = []

    # Initialize tallies
    for i: uint256 in range(10, bound=10):
        if i >= num_candidates:
            break
        tallies.append(0)

    # Count votes
    for vote: Vote in self.votes:
        if vote.candidate < num_candidates:
            tallies[vote.candidate] += vote.weight

    return tallies
```

### Pattern: Filter

```python
# @version 0.4.0

active_users: DynArray[address, 500]
user_balance: HashMap[address, uint256]

@view
@external
def get_wealthy_users(min_balance: uint256) -> DynArray[address, 100]:
    result: DynArray[address, 100] = []
    for user: address in self.active_users:
        if self.user_balance[user] >= min_balance:
            if len(result) < 100:
                result.append(user)
    return result
```

### Pattern: Transform

```python
# @version 0.4.0

@pure
@external
def double_all(values: DynArray[uint256, 50]) -> DynArray[uint256, 50]:
    result: DynArray[uint256, 50] = []
    for v: uint256 in values:
        result.append(v * 2)
    return result

@pure
@external
def apply_discount(
    prices: DynArray[uint256, 20],
    discount_bps: uint256
) -> DynArray[uint256, 20]:
    result: DynArray[uint256, 20] = []
    for price: uint256 in prices:
        discounted: uint256 = price - (price * discount_bps / 10000)
        result.append(discounted)
    return result
```

### Pattern: Reduce

```python
# @version 0.4.0

@pure
@external
def product(values: DynArray[uint256, 20]) -> uint256:
    result: uint256 = 1
    for v: uint256 in values:
        result *= v
    return result

@pure
@external
def geometric_mean_approx(values: DynArray[uint256, 10]) -> uint256:
    assert len(values) > 0, "Empty"
    product: uint256 = 1
    for v: uint256 in values:
        product *= v
    # Integer approximation of nth root
    n: uint256 = convert(len(values), uint256)
    # Simple: return product / n (not true geometric mean)
    return product / n
```

### Pattern: Sliding Window

```python
# @version 0.4.0

price_history: DynArray[uint256, 200]

@view
@external
def moving_average(window: uint256) -> uint256:
    assert window > 0 and window <= 50, "Invalid window"
    n: uint256 = convert(len(self.price_history), uint256)
    if n < window:
        return 0
    total: uint256 = 0
    start: uint256 = n - window
    for i: uint256 in range(start, start + window, bound=50):
        total += self.price_history[i]
    return total / window
```

---

## 7. Gas Considerations {#gas}

```python
# @version 0.4.0

# ════════════════════════════════════════
# Gas-Efficient Loop Patterns
# ════════════════════════════════════════

# ━━━━━━━━━━━━━━━━━━━━━━━━
# ❌ Gas-Expensive: Storage access in every iteration
# ━━━━━━━━━━━━━━━━━━━━━━━━
users: DynArray[address, 100]
balances: HashMap[address, uint256]

@view
@external
def sum_balances_expensive() -> uint256:
    total: uint256 = 0
    for user: address in self.users:
        total += self.balances[user]  # Storage read every iteration!
    return total

# ━━━━━━━━━━━━━━━━━━━━━━━━
# ✅ Gas-Efficient: Cache storage in memory
# ━━━━━━━━━━━━━━━━━━━━━━━━
@view
@external
def sum_balances_efficient() -> uint256:
    # Cache users array in memory first
    cached_users: DynArray[address, 100] = self.users
    total: uint256 = 0
    for user: address in cached_users:
        total += self.balances[user]
    return total

# ━━━━━━━━━━━━━━━━━━━━━━━━
# ❌ Many iterations = High gas
# ━━━━━━━━━━━━━━━━━━━━━━━━
@external
def update_all_expensive():
    # 500 iterations * 20k gas each = 10M gas!
    for user: address in self.users:
        self.balances[user] = self.balances[user] * 2

# ━━━━━━━━━━━━━━━━━━━━━━━━
# ✅ Batch processing with pagination
# ━━━━━━━━━━━━━━━━━━━━━━━━
last_processed: uint256

@external
def update_batch(batch_size: uint256):
    assert batch_size <= 20, "Max 20 per batch"
    start: uint256 = self.last_processed
    n: uint256 = convert(len(self.users), uint256)
    end: uint256 = start + batch_size
    if end > n:
        end = n
    for i: uint256 in range(start, start + 20, bound=20):
        if i >= end:
            break
        user: address = self.users[i]
        self.balances[user] = self.balances[user] * 2
    self.last_processed = end
    if end >= n:
        self.last_processed = 0  # Reset for next round

# ━━━━━━━━━━━━━━━━━━━━━━━━
# Gas estimation guidelines
# ━━━━━━━━━━━━━━━━━━━━━━━━
# SLOAD (storage read): ~800 gas
# SSTORE (storage write): ~20,000 gas (cold), ~2,900 gas (warm)
# Simple addition: ~3 gas
# Loop overhead: ~10 gas/iteration
# Max gas per tx: 30,000,000 gas

# Safe limits:
# - Pure computation: up to 10,000 iterations
# - Storage reads: up to 1,000 iterations
# - Storage writes: up to 100 iterations
```

---

## 8. ตัวอย่าง: Batch Transfer Contract {#example}

Contract ที่ทำ Bulk ETH/Token Transfers อย่างมีประสิทธิภาพ

```python
# @version 0.4.0

# ════════════════════════════════════════════════════════════
# BatchTransfer Contract
# ส่ง ETH หรือ Token ให้หลายคนในธุรกรรมเดียว
# ════════════════════════════════════════════════════════════

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Constants
# ━━━━━━━━━━━━━━━━━━━━━━━━━
MAX_RECIPIENTS: constant(uint256) = 200
MAX_BATCH_SIZE: constant(uint256) = 50  # Per transaction

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Interfaces
# ━━━━━━━━━━━━━━━━━━━━━━━━━
interface IERC20:
    def transfer(to: address, amount: uint256) -> bool: nonpayable
    def transferFrom(sender: address, to: address, amount: uint256) -> bool: nonpayable
    def balanceOf(account: address) -> uint256: view

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Structs
# ━━━━━━━━━━━━━━━━━━━━━━━━━
struct Recipient:
    addr: address
    amount: uint256

struct BatchResult:
    total_sent: uint256
    success_count: uint256
    fail_count: uint256

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# State Variables
# ━━━━━━━━━━━━━━━━━━━━━━━━━
owner: address
fee_bps: uint256           # Fee in basis points
fee_collector: address
total_eth_sent: uint256
total_transfers: uint256

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Events
# ━━━━━━━━━━━━━━━━━━━━━━━━━
event BatchETHSent:
    sender: indexed(address)
    recipient_count: uint256
    total_amount: uint256

event BatchTokenSent:
    sender: indexed(address)
    token: indexed(address)
    recipient_count: uint256
    total_amount: uint256

event FeeCollected:
    payer: indexed(address)
    fee_amount: uint256

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Constructor
# ━━━━━━━━━━━━━━━━━━━━━━━━━
@deploy
def __init__(fee_bps: uint256, fee_collector: address):
    self.owner = msg.sender
    self.fee_bps = fee_bps            # e.g., 50 = 0.5%
    self.fee_collector = fee_collector

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Internal Helpers
# ━━━━━━━━━━━━━━━━━━━━━━━━━

@pure
@internal
def _calculate_fee(amount: uint256) -> uint256:
    return (amount * self.fee_bps) / 10000

@pure
@internal
def _sum_amounts(recipients: DynArray[Recipient, 200]) -> uint256:
    total: uint256 = 0
    for r: Recipient in recipients:
        total += r.amount
    return total

@internal
def _validate_recipients(recipients: DynArray[Recipient, 200]):
    assert len(recipients) > 0, "No recipients"
    assert len(recipients) <= MAX_BATCH_SIZE, "Too many recipients"
    for r: Recipient in recipients:
        assert r.addr != empty(address), "Invalid recipient address"
        assert r.amount > 0, "Zero amount"

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# External Functions
# ━━━━━━━━━━━━━━━━━━━━━━━━━

@payable
@external
def batch_send_eth(recipients: DynArray[Recipient, 200]) -> BatchResult:
    """
    ส่ง ETH ให้หลายคนพร้อมกัน
    msg.value ต้องครอบคลุม total amount + fee
    """
    self._validate_recipients(recipients)

    total_amount: uint256 = self._sum_amounts(recipients)
    fee: uint256 = self._calculate_fee(total_amount)
    required: uint256 = total_amount + fee

    assert msg.value >= required, "Insufficient ETH sent"

    # Collect fee
    if fee > 0:
        send(self.fee_collector, fee)
        log FeeCollected(msg.sender, fee)

    # Send to each recipient
    success_count: uint256 = 0
    fail_count: uint256 = 0

    for r: Recipient in recipients:
        # send() reverts on failure, so we use raw_call for resilience
        result: bool = raw_call(
            r.addr,
            b"",
            value=r.amount,
            revert_on_failure=False
        )
        if result:
            success_count += 1
        else:
            fail_count += 1
            # Return failed amount to sender
            send(msg.sender, r.amount)

    # Refund excess
    excess: uint256 = msg.value - required
    if excess > 0:
        send(msg.sender, excess)

    # Update stats
    self.total_eth_sent += total_amount
    self.total_transfers += success_count

    log BatchETHSent(msg.sender, success_count, total_amount)

    return BatchResult({
        total_sent: total_amount,
        success_count: success_count,
        fail_count: fail_count
    })

@external
def batch_send_token(
    token: address,
    recipients: DynArray[Recipient, 200]
) -> BatchResult:
    """
    ส่ง ERC20 Token ให้หลายคนพร้อมกัน
    ต้อง approve contract ก่อน
    """
    self._validate_recipients(recipients)

    total_amount: uint256 = self._sum_amounts(recipients)
    fee: uint256 = self._calculate_fee(total_amount)
    total_needed: uint256 = total_amount + fee

    # Transfer tokens to this contract first
    token_contract: IERC20 = IERC20(token)
    success: bool = token_contract.transferFrom(
        msg.sender, self, total_needed
    )
    assert success, "Token transfer to contract failed"

    # Collect fee in tokens
    if fee > 0:
        token_contract.transfer(self.fee_collector, fee)

    # Send to each recipient
    success_count: uint256 = 0
    fail_count: uint256 = 0
    failed_amounts: uint256 = 0

    for r: Recipient in recipients:
        result: bool = token_contract.transfer(r.addr, r.amount)
        if result:
            success_count += 1
        else:
            fail_count += 1
            failed_amounts += r.amount

    # Return failed amounts to sender
    if failed_amounts > 0:
        token_contract.transfer(msg.sender, failed_amounts)

    log BatchTokenSent(msg.sender, token, success_count, total_amount)

    return BatchResult({
        total_sent: total_amount - failed_amounts,
        success_count: success_count,
        fail_count: fail_count
    })

@external
def batch_send_equal_eth(
    recipients: DynArray[address, 200],
    amount_each: uint256
) -> uint256:
    """
    ส่ง ETH เท่ากันทุกคน
    """
    assert len(recipients) > 0, "No recipients"
    assert len(recipients) <= MAX_BATCH_SIZE, "Too many"
    assert amount_each > 0, "Zero amount"

    n: uint256 = convert(len(recipients), uint256)
    total: uint256 = amount_each * n
    fee: uint256 = self._calculate_fee(total)

    assert msg.value >= total + fee, "Insufficient ETH"

    if fee > 0:
        send(self.fee_collector, fee)

    sent_count: uint256 = 0
    for addr: address in recipients:
        if addr != empty(address):
            send(addr, amount_each)
            sent_count += 1

    # Refund excess
    excess: uint256 = msg.value - (total + fee)
    if excess > 0:
        send(msg.sender, excess)

    self.total_eth_sent += total
    self.total_transfers += sent_count
    log BatchETHSent(msg.sender, sent_count, total)
    return sent_count

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Admin Functions
# ━━━━━━━━━━━━━━━━━━━━━━━━━

@external
def set_fee(new_fee_bps: uint256):
    assert msg.sender == self.owner, "Not owner"
    assert new_fee_bps <= 500, "Max 5% fee"
    self.fee_bps = new_fee_bps

@external
def set_fee_collector(new_collector: address):
    assert msg.sender == self.owner, "Not owner"
    assert new_collector != empty(address), "Invalid address"
    self.fee_collector = new_collector

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# View Functions
# ━━━━━━━━━━━━━━━━━━━━━━━━━

@view
@external
def calculate_required_eth(
    recipients: DynArray[Recipient, 200]
) -> (uint256, uint256):
    """Returns (total_amount, fee)"""
    total: uint256 = self._sum_amounts(recipients)
    fee: uint256 = self._calculate_fee(total)
    return total, fee

@view
@external
def get_stats() -> (uint256, uint256):
    """Returns (total_eth_sent, total_transfers)"""
    return self.total_eth_sent, self.total_transfers
```

### Test Code

```python
# tests/test_batch_transfer.py
import pytest
from brownie import BatchTransfer, accounts, Wei, reverts

@pytest.fixture
def batch(accounts):
    return BatchTransfer.deploy(
        50,              # 0.5% fee
        accounts[9],     # fee collector
        {'from': accounts[0]}
    )

class TestBatchETH:
    def test_send_to_multiple(self, batch, accounts):
        recipients = [
            (accounts[1], Wei("0.1 ether")),
            (accounts[2], Wei("0.2 ether")),
            (accounts[3], Wei("0.3 ether")),
        ]
        structs = [{"addr": r[0], "amount": r[1]} for r in recipients]
        total = sum(r[1] for r in recipients)
        fee = total * 50 // 10000

        initial_balances = [accounts[i].balance() for i in range(1, 4)]

        tx = batch.batch_send_eth(
            structs,
            {'from': accounts[4], 'value': total + fee}
        )

        # Check recipients received amounts
        for i, r in enumerate(recipients):
            assert accounts[i + 1].balance() == initial_balances[i] + r[1]

    def test_equal_distribution(self, batch, accounts):
        addrs = [accounts[i] for i in range(1, 6)]
        amount_each = Wei("0.1 ether")
        n = len(addrs)
        total = amount_each * n
        fee = total * 50 // 10000

        initial = [a.balance() for a in addrs]
        batch.batch_send_equal_eth(
            addrs,
            amount_each,
            {'from': accounts[0], 'value': total + fee + Wei("0.01 ether")}
        )

        for i, addr in enumerate(addrs):
            assert addr.balance() == initial[i] + amount_each

    def test_insufficient_eth_reverts(self, batch, accounts):
        structs = [{"addr": accounts[1], "amount": Wei("1 ether")}]
        with reverts("Insufficient ETH sent"):
            batch.batch_send_eth(
                structs,
                {'from': accounts[0], 'value': Wei("0.5 ether")}
            )

    def test_empty_recipients_reverts(self, batch, accounts):
        with reverts("No recipients"):
            batch.batch_send_eth([], {'from': accounts[0], 'value': Wei("1 ether")})

class TestStats:
    def test_stats_updated(self, batch, accounts):
        structs = [{"addr": accounts[1], "amount": Wei("0.1 ether")}]
        batch.batch_send_eth(structs, {'from': accounts[0], 'value': Wei("0.2 ether")})
        eth_sent, transfers = batch.get_stats()
        assert eth_sent == Wei("0.1 ether")
        assert transfers == 1
```

---

## สรุป

- ✅ **for i in range(N)**: Loop ด้วยตัวเลข (N ต้องรู้ล่วงหน้าหรือมี bound=)
- ✅ **for item in array**: Loop ผ่าน Fixed/Dynamic Array
- ✅ **bound=N**: บอก Compiler ว่า Loop วนสูงสุดกี่รอบ
- ✅ **break, continue**: มีใน Vyper 0.4.0
- ✅ **Early return**: แทน break ใน version เก่า
- ✅ **Flag pattern**: แทน continue ใน version เก่า
- ✅ **Gas optimization**: Cache storage, ใช้ pagination สำหรับ large datasets

## แบบฝึกหัด

1. **สร้าง** Voting Contract ที่นับคะแนนเสียงด้วย loop
2. **เพิ่ม** Pagination ให้ BatchTransfer เพื่อรองรับ 1000+ recipients
3. **สร้าง** Statistics Contract: mean, median, std deviation
4. **ทดสอบ** Gas usage ของ loop size 10, 50, 100, 200
5. **สร้าง** Airdrop Contract ที่แจก Token จาก Merkle Proof

---

**ก่อนหน้า: [Part 007 - Control Flow: If/Elif/Else](part_007_control_flow.md)**  
**ต่อไป: [Part 009 - Arrays และ DynArray](part_009_arrays.md)**
