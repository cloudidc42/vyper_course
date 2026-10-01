# Part 014: Modifiers และ @view/@pure

## สารบัญ
1. [All Decorators ใน Vyper](#all-decorators)
2. [Composing Decorators](#composing)
3. [@nonreentrant ในเชิงลึก](#nonreentrant)
4. [Reentrancy Attack อธิบาย](#reentrancy-attack)
5. [Protection Patterns](#protection-patterns)
6. [ตัวอย่าง: Vault Contract with Reentrancy Protection](#example)

---

## 1. All Decorators ใน Vyper {#all-decorators}

Vyper ใช้ Decorators แทน Modifiers ของ Solidity

```python
# @version 0.4.0

# ════════════════════════════════════════
# All Available Decorators
# ════════════════════════════════════════

# 1. @external  - callable from outside
# 2. @internal  - callable only inside
# 3. @view      - reads state, no writes
# 4. @pure      - no state access at all
# 5. @payable   - accepts ETH
# 6. @nonreentrant - prevents reentrant calls
# 7. @deploy    - marks constructor (__init__)

owner: address

@deploy
def __init__():
    self.owner = msg.sender

# @external
@external
def external_func():
    pass

# @internal
@internal
def internal_func():
    pass

# @view + @external
@view
@external
def view_func() -> address:
    return self.owner

# @pure + @external
@pure
@external
def pure_func(a: uint256, b: uint256) -> uint256:
    return a + b

# @payable + @external
@payable
@external
def payable_func():
    pass  # accepts ETH

# @nonreentrant + @external
@nonreentrant
@external
def nonreentrant_func():
    pass
```

### @external ในเชิงลึก

```python
# @version 0.4.0

# ════════════════════════════════════════
# @external Details
# ════════════════════════════════════════

# @external functions:
# - มี function selector (4 bytes of keccak256)
# - เรียกได้จาก: EOA, other contracts, frontend
# - ไม่เรียกได้จาก: ภายใน contract เดียวกัน
# - สร้าง ABI entry

@external
def standard_external(x: uint256) -> uint256:
    return x * 2

# ❌ ไม่สามารถเรียก @external จาก @internal
# @internal
# def try_call_external():
#     self.standard_external(5)  # Error!

# ✅ ต้องมี @internal version แยก
@internal
def _double(x: uint256) -> uint256:
    return x * 2

@external
def external_wrapper(x: uint256) -> uint256:
    return self._double(x)
```

### @internal ในเชิงลึก

```python
# @version 0.4.0

# ════════════════════════════════════════
# @internal Details
# ════════════════════════════════════════

# @internal functions:
# - ไม่มี function selector
# - เรียกด้วย self.function_name()
# - ไม่สร้าง ABI entry
# - ถูกกว่า @external (ไม่มี dispatch overhead)
# - Inlined โดย compiler ในบางกรณี

owner: address
balances: HashMap[address, uint256]

@internal
def _only_owner():
    assert msg.sender == self.owner, "Not owner"

@internal
def _check_balance(addr: address, amount: uint256):
    assert self.balances[addr] >= amount, "Insufficient"

@internal
def _update_balance(from_addr: address, to: address, amount: uint256):
    self._check_balance(from_addr, amount)
    self.balances[from_addr] -= amount
    self.balances[to] += amount

@deploy
def __init__():
    self.owner = msg.sender

@external
def transfer(to: address, amount: uint256):
    self._update_balance(msg.sender, to, amount)

@external
def admin_transfer(from_addr: address, to: address, amount: uint256):
    self._only_owner()
    self._update_balance(from_addr, to, amount)
```

### @view ในเชิงลึก

```python
# @version 0.4.0

# ════════════════════════════════════════
# @view Details
# ════════════════════════════════════════

# @view functions:
# - อ่าน state ได้ (self.*)
# - อ่าน global vars ได้ (msg, block, tx, chain)
# - ไม่เขียน state (SSTORE ไม่ได้)
# - ไม่ส่ง ETH ไม่ได้
# - เรียก @view หรือ @pure function อื่นได้
# - ฟรี Gas เมื่อเรียก off-chain (eth_call)
# - เสีย Gas เมื่อ on-chain contract เรียก

total_supply: uint256
balances: HashMap[address, uint256]
prices: HashMap[uint256, uint256]

@deploy
def __init__():
    self.total_supply = 1000000

@view
@external
def balance_of(account: address) -> uint256:
    return self.balances[account]

@view
@external
def total_supply_view() -> uint256:
    return self.total_supply

# @view + @internal
@view
@internal
def _calculate_value(account: address, price_id: uint256) -> uint256:
    balance: uint256 = self.balances[account]
    price: uint256 = self.prices[price_id]
    return balance * price

@view
@external
def get_portfolio_value(account: address, price_id: uint256) -> uint256:
    return self._calculate_value(account, price_id)

# ❌ @view ไม่สามารถ:
# @view
# @external
# def bad_view():
#     self.total_supply = 0  # Error: cannot write
#     send(msg.sender, 100)  # Error: cannot send ETH
```

### @pure ในเชิงลึก

```python
# @version 0.4.0

# ════════════════════════════════════════
# @pure Details
# ════════════════════════════════════════

# @pure functions:
# - ไม่อ่านและไม่เขียน state (self.* ไม่ได้)
# - ไม่อ่าน global vars (msg.sender, block.timestamp ไม่ได้)
# - เป็น pure computation เท่านั้น
# - เหมาะสำหรับ math utilities, encoders, decoders

@pure
@external
def add(a: uint256, b: uint256) -> uint256:
    return a + b

@pure
@external
def percentage(amount: uint256, bps: uint256) -> uint256:
    return (amount * bps) / 10000

@pure
@external
def encode_pair(a: uint128, b: uint128) -> uint256:
    return convert(a, uint256) * 2**128 + convert(b, uint256)

@pure
@external
def is_valid_bps(bps: uint256) -> bool:
    return bps <= 10000

# ❌ @pure ไม่สามารถ:
# @pure
# @external
# def bad_pure() -> address:
#     return msg.sender   # Error: no global access
#     return self.owner   # Error: no state access
```

### @payable ในเชิงลึก

```python
# @version 0.4.0

# ════════════════════════════════════════
# @payable Details
# ════════════════════════════════════════

# @payable functions:
# - รับ ETH ได้ (msg.value > 0)
# - ถ้าไม่มี @payable แต่ส่ง ETH มา -> revert
# - สามารถ combine กับ @external หรือ @nonreentrant

deposits: HashMap[address, uint256]

@deploy
def __init__():
    pass

# @payable function
@payable
@external
def deposit():
    assert msg.value > 0, "Must send ETH"
    self.deposits[msg.sender] += msg.value

# @payable + @nonreentrant
@nonreentrant
@payable
@external
def safe_deposit():
    assert msg.value > 0, "Must send ETH"
    self.deposits[msg.sender] += msg.value

# ไม่มี @payable = revert ถ้าส่ง ETH มา
@external
def no_eth(x: uint256) -> uint256:
    # msg.value == 0 เสมอที่นี่ (enforced by EVM)
    return x

# Constructor สามารถรับ ETH ได้ด้วย
initial_eth: uint256

@payable
@deploy
def __init__with_eth():
    self.initial_eth = msg.value
```

---

## 2. Composing Decorators {#composing}

```python
# @version 0.4.0

# ════════════════════════════════════════
# Valid Decorator Combinations
# ════════════════════════════════════════

owner: address

@deploy
def __init__():
    self.owner = msg.sender

# ✅ Valid combinations:

# 1. @external alone
@external
def ext_only():
    pass

# 2. @view + @external
@view
@external
def view_ext() -> address:
    return self.owner

# 3. @pure + @external
@pure
@external
def pure_ext(x: uint256) -> uint256:
    return x

# 4. @payable + @external
@payable
@external
def payable_ext():
    pass

# 5. @nonreentrant + @external
@nonreentrant
@external
def nonreentrant_ext():
    pass

# 6. @nonreentrant + @payable + @external
@nonreentrant
@payable
@external
def full_combo():
    pass

# 7. @view + @internal
@view
@internal
def view_int() -> address:
    return self.owner

# 8. @pure + @internal
@pure
@internal
def pure_int(x: uint256) -> uint256:
    return x * 2

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# ❌ Invalid combinations:
# ━━━━━━━━━━━━━━━━━━━━━━━━━
# @view + @payable  -> Cannot: view can't receive ETH
# @pure + @payable  -> Cannot: pure can't access msg.value
# @nonreentrant + @internal -> Not supported
# @payable + @internal -> Not supported
```

---

## 3. @nonreentrant ในเชิงลึก {#nonreentrant}

```python
# @version 0.4.0

# ════════════════════════════════════════
# @nonreentrant Deep Dive
# ════════════════════════════════════════

# @nonreentrant ทำงานอย่างไร:
# 1. ตรวจสอบ lock flag ก่อนเข้าฟังก์ชัน
# 2. ตั้ง lock flag เป็น True
# 3. Execute ฟังก์ชัน
# 4. Reset lock flag กลับเป็น False
# 5. ถ้ามีการเรียกซ้ำ: lock = True -> revert

# Vyper ใช้ transient storage (EIP-1153) หรือ regular storage
# สำหรับ lock flag โดยอัตโนมัติ

owner: address
balances: HashMap[address, uint256]
locked: bool  # ไม่ต้องประกาศเอง - Vyper จัดการให้

@deploy
def __init__():
    self.owner = msg.sender

# @nonreentrant ป้องกันการเรียกซ้ำ
@nonreentrant
@external
def withdraw(amount: uint256):
    assert self.balances[msg.sender] >= amount
    # Update state FIRST (ยังไงก็ปลอดภัยกว่าด้วย @nonreentrant)
    self.balances[msg.sender] -= amount
    # ถ้า attacker เรียก withdraw อีกครั้งใน receive(),
    # จะ revert เพราะ lock ยังติดอยู่
    send(msg.sender, amount)

@nonreentrant
@payable
@external
def deposit():
    self.balances[msg.sender] += msg.value
```

### Named Locks

```python
# @version 0.4.0

# ════════════════════════════════════════
# Named Nonreentrant Locks
# ════════════════════════════════════════

# @nonreentrant ใช้ lock name เดียวกัน = share lock
# ป้องกัน cross-function reentrancy

owner: address
token_balances: HashMap[address, uint256]
eth_balances: HashMap[address, uint256]

@deploy
def __init__():
    self.owner = msg.sender

# ✅ ทั้งสองฟังก์ชันใช้ lock เดียวกัน
# ถ้า withdraw_eth ถูกเรียก ขณะที่ withdraw_token กำลังทำงาน -> revert
@nonreentrant
@external
def withdraw_eth(amount: uint256):
    assert self.eth_balances[msg.sender] >= amount
    self.eth_balances[msg.sender] -= amount
    send(msg.sender, amount)

@nonreentrant
@external
def withdraw_token(amount: uint256):
    assert self.token_balances[msg.sender] >= amount
    self.token_balances[msg.sender] -= amount
    # ... transfer tokens
```

---

## 4. Reentrancy Attack อธิบาย {#reentrancy-attack}

```python
# @version 0.4.0

# ════════════════════════════════════════
# Reentrancy Attack Explained
# ════════════════════════════════════════

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# VULNERABLE CONTRACT (ห้ามใช้)
# ━━━━━━━━━━━━━━━━━━━━━━━━━

balances: HashMap[address, uint256]

@payable
@external
def deposit():
    self.balances[msg.sender] += msg.value

# ❌ VULNERABLE: Checks-Interactions-Effects (ผิดลำดับ)
@external
def vulnerable_withdraw():
    amount: uint256 = self.balances[msg.sender]
    assert amount > 0, "Nothing"

    # 1. Interaction (ส่ง ETH ก่อน) <- อันตราย!
    raw_call(msg.sender, b"", value=amount)

    # 2. Effects (update state หลัง) <- สาย!
    # ถ้า msg.sender เป็น attacker contract:
    # - receive() ถูกเรียกตอน raw_call
    # - attacker เรียก vulnerable_withdraw อีกครั้ง
    # - self.balances[attacker] ยังเป็นค่าเดิม (ยังไม่ได้อัปเดต!)
    # - attacker ได้เงินซ้ำ!
    self.balances[msg.sender] = 0  # Too late!
```

### Attack Contract (ตัวอย่างเพื่อเข้าใจ)

```solidity
// Solidity Attacker Contract (educational only)
// contract Attacker {
//     VulnerableVault public vault;
//
//     constructor(address _vault) {
//         vault = VulnerableVault(_vault);
//     }
//
//     function attack() external payable {
//         vault.deposit{value: 1 ether}();
//         vault.vulnerable_withdraw();
//     }
//
//     receive() external payable {
//         if (address(vault).balance >= 1 ether) {
//             vault.vulnerable_withdraw(); // Reenter!
//         }
//     }
// }
```

### Attack Pattern ทำงานอย่างไร

```
Attack Sequence:
1. Attacker deposits 1 ETH
2. Attacker calls vulnerable_withdraw()
3. Contract: checks balance[attacker] = 1 ETH ✓
4. Contract: sends 1 ETH to attacker (raw_call)
5. Attacker's receive() triggers!
6. Attacker calls vulnerable_withdraw() AGAIN
7. Contract: checks balance[attacker] = 1 ETH ✓ (ยังไม่อัปเดต!)
8. Contract: sends 1 ETH to attacker AGAIN
9. Repeat until vault is empty
10. Contract finally sets balance[attacker] = 0 (too late)

Result: Attacker drained entire vault!
```

---

## 5. Protection Patterns {#protection-patterns}

### Pattern 1: Checks-Effects-Interactions (CEI)

```python
# @version 0.4.0

# ════════════════════════════════════════
# Checks-Effects-Interactions Pattern
# ════════════════════════════════════════

balances: HashMap[address, uint256]

@deploy
def __init__():
    pass

@payable
@external
def deposit():
    self.balances[msg.sender] += msg.value

# ✅ CEI Pattern: ถูกลำดับ
@external
def safe_withdraw_cei():
    amount: uint256 = self.balances[msg.sender]

    # 1. CHECKS
    assert amount > 0, "Nothing to withdraw"

    # 2. EFFECTS (update state FIRST)
    self.balances[msg.sender] = 0

    # 3. INTERACTIONS (external call LAST)
    send(msg.sender, amount)
    # ถ้า attacker reenter ตอนนี้:
    # - balance[attacker] = 0 แล้ว
    # - assert amount > 0 จะ fail!
```

### Pattern 2: @nonreentrant Lock

```python
# @version 0.4.0

# ════════════════════════════════════════
# @nonreentrant Lock Pattern
# ════════════════════════════════════════

balances: HashMap[address, uint256]

@deploy
def __init__():
    pass

@payable
@external
def deposit():
    self.balances[msg.sender] += msg.value

# ✅ @nonreentrant prevents reentry regardless of order
@nonreentrant
@external
def locked_withdraw():
    amount: uint256 = self.balances[msg.sender]
    assert amount > 0, "Nothing"

    # Lock is set, any reentry attempt will revert
    raw_call(msg.sender, b"", value=amount)

    # This runs after the call returns (no reentry possible)
    self.balances[msg.sender] = 0

# ✅ Best: Both CEI + @nonreentrant
@nonreentrant
@external
def doubly_safe_withdraw():
    amount: uint256 = self.balances[msg.sender]
    assert amount > 0, "Nothing"
    # CEI: effects first
    self.balances[msg.sender] = 0
    # Then interaction
    send(msg.sender, amount)
```

### Pattern 3: Pull Payment

```python
# @version 0.4.0

# ════════════════════════════════════════
# Pull Payment Pattern
# ════════════════════════════════════════
# แทนที่จะ push ETH ไปหา recipient
# ให้ recipient pull ETH เองแทน

pending_withdrawals: HashMap[address, uint256]

@deploy
def __init__():
    pass

# Contract accumulates pending payments
@internal
def _schedule_payment(to: address, amount: uint256):
    self.pending_withdrawals[to] += amount

# Users pull their own payments
@nonreentrant
@external
def withdraw():
    amount: uint256 = self.pending_withdrawals[msg.sender]
    assert amount > 0, "Nothing to withdraw"

    # CEI pattern
    self.pending_withdrawals[msg.sender] = 0
    send(msg.sender, amount)

@view
@external
def pending_for(account: address) -> uint256:
    return self.pending_withdrawals[account]
```

### Pattern 4: Mutex per User

```python
# @version 0.4.0

# ════════════════════════════════════════
# Per-User Mutex Pattern
# ════════════════════════════════════════

balances: HashMap[address, uint256]
locked: HashMap[address, bool]

@deploy
def __init__():
    pass

@payable
@external
def deposit():
    self.balances[msg.sender] += msg.value

@external
def withdraw_with_mutex(amount: uint256):
    assert not self.locked[msg.sender], "Locked"
    assert self.balances[msg.sender] >= amount, "Insufficient"

    # Lock per user
    self.locked[msg.sender] = True

    # CEI
    self.balances[msg.sender] -= amount
    send(msg.sender, amount)

    # Unlock
    self.locked[msg.sender] = False
    # Note: @nonreentrant is still better for gas efficiency
```

---

## 6. ตัวอย่าง: Vault Contract with Reentrancy Protection {#example}

Vault ที่มี Reentrancy Protection ครบถ้วน

```python
# @version 0.4.0

# ════════════════════════════════════════════════════════════
# SecureVault Contract
# Vault ที่ปลอดภัยจาก Reentrancy Attack
# ════════════════════════════════════════════════════════════

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Interfaces
# ━━━━━━━━━━━━━━━━━━━━━━━━━
interface IERC20:
    def transfer(to: address, amount: uint256) -> bool: nonpayable
    def transferFrom(sender: address, to: address, amount: uint256) -> bool: nonpayable
    def balanceOf(account: address) -> uint256: view
    def approve(spender: address, amount: uint256) -> bool: nonpayable

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Constants & Immutables
# ━━━━━━━━━━━━━━━━━━━━━━━━━
MIN_DEPOSIT: constant(uint256) = 10**15         # 0.001 ETH
MAX_DEPOSIT: constant(uint256) = 100 * 10**18   # 100 ETH
WITHDRAWAL_FEE_BPS: constant(uint256) = 50       # 0.5%

OWNER: immutable(address)
CREATION_TIME: immutable(uint256)

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Structs
# ━━━━━━━━━━━━━━━━━━━━━━━━━
struct VaultPosition:
    eth_balance: uint256
    token_balance: HashMap[address, uint256]  # token -> amount
    total_deposited: uint256
    deposit_count: uint256
    last_activity: uint256

struct VaultStats:
    total_eth: uint256
    total_users: uint256
    total_deposits: uint256
    total_withdrawals: uint256

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# State Variables
# ━━━━━━━━━━━━━━━━━━━━━━━━━
positions: HashMap[address, VaultPosition]
user_token_deposits: HashMap[address, HashMap[address, uint256]]  # user -> token -> amount

stats: VaultStats
fee_collected: uint256
paused: bool

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Events
# ━━━━━━━━━━━━━━━━━━━━━━━━━
event ETHDeposited:
    user: indexed(address)
    amount: uint256
    new_balance: uint256

event ETHWithdrawn:
    user: indexed(address)
    amount: uint256
    fee: uint256
    remaining: uint256

event TokenDeposited:
    user: indexed(address)
    token: indexed(address)
    amount: uint256

event TokenWithdrawn:
    user: indexed(address)
    token: indexed(address)
    amount: uint256
    fee: uint256

event EmergencyWithdrawal:
    user: indexed(address)
    eth_amount: uint256

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Constructor
# ━━━━━━━━━━━━━━━━━━━━━━━━━
@deploy
def __init__():
    OWNER = msg.sender
    CREATION_TIME = block.timestamp
    self.paused = False

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Internal Helpers
# ━━━━━━━━━━━━━━━━━━━━━━━━━

@view
@internal
def _only_owner():
    assert msg.sender == OWNER, "Not owner"

@internal
def _not_paused():
    assert not self.paused, "Vault paused"

@view
@internal
def _calculate_fee(amount: uint256) -> uint256:
    return (amount * WITHDRAWAL_FEE_BPS) / 10000

@internal
def _register_activity(user: address):
    self.positions[user].last_activity = block.timestamp

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# ETH Deposit/Withdrawal
# ━━━━━━━━━━━━━━━━━━━━━━━━━

@nonreentrant
@payable
@external
def deposit_eth():
    """Deposit ETH into vault"""
    self._not_paused()
    assert msg.value >= MIN_DEPOSIT, "Below minimum"
    assert msg.value <= MAX_DEPOSIT, "Above maximum"

    # Track if new user
    if self.positions[msg.sender].total_deposited == 0:
        self.stats.total_users += 1

    # Update position (EFFECTS before any potential calls)
    self.positions[msg.sender].eth_balance += msg.value
    self.positions[msg.sender].total_deposited += msg.value
    self.positions[msg.sender].deposit_count += 1
    self._register_activity(msg.sender)

    # Update stats
    self.stats.total_eth += msg.value
    self.stats.total_deposits += 1

    log ETHDeposited(msg.sender, msg.value, self.positions[msg.sender].eth_balance)

@nonreentrant
@external
def withdraw_eth(amount: uint256):
    """Withdraw ETH from vault - CEI + nonreentrant"""
    self._not_paused()

    # === CHECKS ===
    assert amount > 0, "Zero amount"
    assert self.positions[msg.sender].eth_balance >= amount, "Insufficient balance"

    # === EFFECTS === (Before any external calls!)
    fee: uint256 = self._calculate_fee(amount)
    net_amount: uint256 = amount - fee

    self.positions[msg.sender].eth_balance -= amount
    self.stats.total_eth -= amount
    self.fee_collected += fee
    self.stats.total_withdrawals += 1
    self._register_activity(msg.sender)

    remaining: uint256 = self.positions[msg.sender].eth_balance

    # === INTERACTIONS === (External calls LAST)
    send(msg.sender, net_amount)
    if fee > 0:
        send(OWNER, fee)

    log ETHWithdrawn(msg.sender, amount, fee, remaining)

@nonreentrant
@external
def withdraw_all_eth():
    """Withdraw entire ETH balance"""
    amount: uint256 = self.positions[msg.sender].eth_balance
    assert amount > 0, "Nothing to withdraw"

    # CEI Pattern
    fee: uint256 = self._calculate_fee(amount)
    net: uint256 = amount - fee

    # Effects
    self.positions[msg.sender].eth_balance = 0
    self.stats.total_eth -= amount
    self.fee_collected += fee
    self.stats.total_withdrawals += 1
    self._register_activity(msg.sender)

    # Interactions
    send(msg.sender, net)
    if fee > 0:
        send(OWNER, fee)

    log ETHWithdrawn(msg.sender, amount, fee, 0)

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Token Deposit/Withdrawal
# ━━━━━━━━━━━━━━━━━━━━━━━━━

@nonreentrant
@external
def deposit_token(token: address, amount: uint256):
    """Deposit ERC20 token into vault"""
    self._not_paused()
    assert token != empty(address), "Invalid token"
    assert amount > 0, "Zero amount"

    # Transfer token from user to vault
    # Note: user must approve first
    token_contract: IERC20 = IERC20(token)

    # CHECKS
    # (allowance check happens in transferFrom)

    # EFFECTS (update state before external call)
    self.user_token_deposits[msg.sender][token] += amount
    self._register_activity(msg.sender)

    # INTERACTIONS
    success: bool = token_contract.transferFrom(msg.sender, self, amount)
    assert success, "Transfer failed"

    log TokenDeposited(msg.sender, token, amount)

@nonreentrant
@external
def withdraw_token(token: address, amount: uint256):
    """Withdraw ERC20 token from vault"""
    self._not_paused()
    assert token != empty(address), "Invalid token"
    assert amount > 0, "Zero amount"

    # CHECKS
    deposited: uint256 = self.user_token_deposits[msg.sender][token]
    assert deposited >= amount, "Insufficient token balance"

    # EFFECTS
    fee: uint256 = self._calculate_fee(amount)
    net: uint256 = amount - fee

    self.user_token_deposits[msg.sender][token] -= amount
    self._register_activity(msg.sender)

    # INTERACTIONS
    token_contract: IERC20 = IERC20(token)
    success: bool = token_contract.transfer(msg.sender, net)
    assert success, "Transfer to user failed"

    if fee > 0:
        success = token_contract.transfer(OWNER, fee)
        assert success, "Fee transfer failed"

    log TokenWithdrawn(msg.sender, token, amount, fee)

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Emergency Functions
# ━━━━━━━━━━━━━━━━━━━━━━━━━

@nonreentrant
@external
def emergency_withdraw():
    """Emergency withdrawal - skip fee, works when paused"""
    amount: uint256 = self.positions[msg.sender].eth_balance
    assert amount > 0, "Nothing"

    # Effects first
    self.positions[msg.sender].eth_balance = 0
    self.stats.total_eth -= amount

    # Interaction
    send(msg.sender, amount)
    log EmergencyWithdrawal(msg.sender, amount)

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Admin Functions
# ━━━━━━━━━━━━━━━━━━━━━━━━━

@external
def pause():
    self._only_owner()
    assert not self.paused, "Already paused"
    self.paused = True

@external
def unpause():
    self._only_owner()
    assert self.paused, "Not paused"
    self.paused = False

@nonreentrant
@external
def collect_fees():
    """Owner collects accumulated fees"""
    self._only_owner()
    amount: uint256 = self.fee_collected
    assert amount > 0, "No fees"
    self.fee_collected = 0
    send(OWNER, amount)

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# View Functions
# ━━━━━━━━━━━━━━━━━━━━━━━━━

@view
@external
def get_eth_balance(user: address) -> uint256:
    return self.positions[user].eth_balance

@view
@external
def get_token_balance(user: address, token: address) -> uint256:
    return self.user_token_deposits[user][token]

@view
@external
def get_stats() -> VaultStats:
    return self.stats

@view
@external
def get_owner() -> address:
    return OWNER

@view
@external
def is_paused() -> bool:
    return self.paused

@view
@external
def fee_collected_amount() -> uint256:
    return self.fee_collected

@view
@external
def calculate_withdrawal(amount: uint256) -> (uint256, uint256):
    """Returns (net_amount, fee)"""
    fee: uint256 = self._calculate_fee(amount)
    return amount - fee, fee

@view
@external
def get_position_info(user: address) -> (uint256, uint256, uint256, uint256):
    """Returns (eth_balance, total_deposited, deposit_count, last_activity)"""
    p: VaultPosition = self.positions[user]
    return p.eth_balance, p.total_deposited, p.deposit_count, p.last_activity
```

### Test Code

```python
# tests/test_secure_vault.py
import pytest
from brownie import SecureVault, accounts, Wei, reverts, chain

@pytest.fixture
def vault(accounts):
    return SecureVault.deploy({'from': accounts[0]})

class TestETHDeposit:
    def test_deposit(self, vault, accounts):
        amount = Wei("1 ether")
        vault.deposit_eth({'from': accounts[1], 'value': amount})
        assert vault.get_eth_balance(accounts[1]) == amount

    def test_below_minimum_reverts(self, vault, accounts):
        with reverts("Below minimum"):
            vault.deposit_eth({'from': accounts[1], 'value': Wei("0.0001 ether")})

    def test_above_maximum_reverts(self, vault, accounts):
        with reverts("Above maximum"):
            vault.deposit_eth({'from': accounts[1], 'value': Wei("101 ether")})

class TestETHWithdrawal:
    def test_withdraw(self, vault, accounts):
        vault.deposit_eth({'from': accounts[1], 'value': Wei("1 ether")})
        initial = accounts[1].balance()
        vault.withdraw_eth(Wei("0.5 ether"), {'from': accounts[1]})
        # After 0.5% fee: receive 0.5 * (1 - 0.005) = 0.4975 ETH
        assert vault.get_eth_balance(accounts[1]) == Wei("0.5 ether")

    def test_fee_taken(self, vault, accounts):
        vault.deposit_eth({'from': accounts[1], 'value': Wei("1 ether")})
        _, fee = vault.calculate_withdrawal(Wei("1 ether"))
        assert fee == Wei("1 ether") * 50 // 10000

    def test_insufficient_balance_reverts(self, vault, accounts):
        with reverts("Insufficient balance"):
            vault.withdraw_eth(Wei("1 ether"), {'from': accounts[1]})

class TestReentrancy:
    def test_reentrancy_protected(self, vault, accounts):
        # nonreentrant decorator prevents reentrancy
        # Testing indirectly: deposit and immediate withdraw should work
        vault.deposit_eth({'from': accounts[1], 'value': Wei("1 ether")})
        vault.withdraw_all_eth({'from': accounts[1]})
        assert vault.get_eth_balance(accounts[1]) == 0

class TestPause:
    def test_pause_blocks_deposit(self, vault, accounts):
        vault.pause({'from': accounts[0]})
        with reverts("Vault paused"):
            vault.deposit_eth({'from': accounts[1], 'value': Wei("1 ether")})

    def test_emergency_withdraw_works_when_paused(self, vault, accounts):
        vault.deposit_eth({'from': accounts[1], 'value': Wei("1 ether")})
        vault.pause({'from': accounts[0]})
        # Emergency withdraw bypasses pause
        vault.emergency_withdraw({'from': accounts[1]})
        assert vault.get_eth_balance(accounts[1]) == 0
```

---

## สรุป

- ✅ **@external**: เรียกได้จากภายนอก, มี ABI entry
- ✅ **@internal**: เรียกได้เฉพาะภายใน, ถูกกว่า
- ✅ **@view**: อ่าน state ไม่เขียน, ฟรี off-chain
- ✅ **@pure**: ไม่แตะ state เลย, pure computation
- ✅ **@payable**: รับ ETH ได้
- ✅ **@nonreentrant**: ป้องกัน reentrancy attack
- ✅ **CEI Pattern**: Checks → Effects → Interactions
- ✅ **Double protection**: @nonreentrant + CEI

## แบบฝึกหัด

1. **เขียน** Attack Contract และทดสอบ attack กับ VulnerableVault
2. **ปรับปรุง** SecureVault ให้รองรับ multi-token strategy
3. **สร้าง** Flash Loan Contract ที่ใช้ @nonreentrant ป้องกัน
4. **ทดสอบ** Gas ของ @nonreentrant vs manual lock
5. **วิเคราะห์** DAO Treasury ที่ปลอดภัยจาก governance attacks

---

**ก่อนหน้า: [Part 013 - Constructor และ Initialization](part_013_constructor.md)**  
**ต่อไป: [Part 015 - Wei และ Ether Units](part_015_units.md)**
