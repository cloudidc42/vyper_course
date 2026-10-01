# Part 010: HashMap (Mapping)

## สารบัญ
1. [HashMap พื้นฐาน](#basics)
2. [Nested Mappings](#nested)
3. [Mapping Patterns](#patterns)
4. [ไม่สามารถ Iterate Mappings](#no-iterate)
5. [Pattern: Enumerable Mapping](#enumerable)
6. [ตัวอย่าง: Token Allowance Contract](#example)

---

## 1. HashMap พื้นฐาน {#basics}

HashMap เป็นโครงสร้างข้อมูลแบบ Key-Value Store บน Blockchain

```python
# @version 0.4.0

# ════════════════════════════════════════
# HashMap[KeyType, ValueType]
# ════════════════════════════════════════

# Declaration
balances: HashMap[address, uint256]
names: HashMap[address, String[50]]
is_active: HashMap[address, bool]
scores: HashMap[bytes32, uint256]
owner_of: HashMap[uint256, address]  # token_id -> owner

@deploy
def __init__():
    # Initial values
    self.balances[msg.sender] = 1000000
    self.names[msg.sender] = "Owner"
    self.is_active[msg.sender] = True

# Basic operations
@external
def set_balance(account: address, amount: uint256):
    self.balances[account] = amount

@view
@external
def get_balance(account: address) -> uint256:
    return self.balances[account]  # Returns 0 if key not found

@external
def increment_balance(account: address, amount: uint256):
    self.balances[account] += amount

@external
def decrement_balance(account: address, amount: uint256):
    assert self.balances[account] >= amount, "Insufficient"
    self.balances[account] -= amount

@external
def delete_entry(account: address):
    self.balances[account] = 0  # Reset to default (can't truly delete)
    self.names[account] = ""
    self.is_active[account] = False
```

### HashMap Default Values

```python
# @version 0.4.0

# HashMap คืนค่า default ถ้า key ไม่มีอยู่
# uint256 -> 0
# bool -> False
# address -> 0x0000...0000
# String -> ""

uint_map: HashMap[address, uint256]
bool_map: HashMap[address, bool]
addr_map: HashMap[address, address]
str_map: HashMap[address, String[50]]

@view
@external
def check_defaults(account: address) -> (uint256, bool, address, String[50]):
    return (
        self.uint_map[account],   # 0
        self.bool_map[account],   # False
        self.addr_map[account],   # 0x0000...0
        self.str_map[account]     # ""
    )

# Pattern: ใช้ค่า default เป็นเงื่อนไข
@view
@external
def is_registered(user: address) -> bool:
    # ถ้า timestamp เป็น 0 = ยังไม่ได้ register
    return self.uint_map[user] > 0

# Pattern: ตรวจสอบ vs สร้างใหม่
registered_at: HashMap[address, uint256]

@external
def register_once():
    assert self.registered_at[msg.sender] == 0, "Already registered"
    self.registered_at[msg.sender] = block.timestamp
```

### HashMap กับ Struct Values

```python
# @version 0.4.0

struct UserInfo:
    name: String[50]
    email_hash: bytes32
    balance: uint256
    tier: uint8
    created_at: uint256
    last_login: uint256

users: HashMap[address, UserInfo]

@deploy
def __init__():
    pass

@external
def create_user(name: String[50], email_hash: bytes32):
    assert self.users[msg.sender].created_at == 0, "Already exists"
    assert len(name) > 0, "Name required"
    self.users[msg.sender] = UserInfo({
        name: name,
        email_hash: email_hash,
        balance: 0,
        tier: 0,
        created_at: block.timestamp,
        last_login: block.timestamp
    })

@external
def update_login():
    assert self.users[msg.sender].created_at > 0, "Not registered"
    self.users[msg.sender].last_login = block.timestamp

@external
def add_balance(amount: uint256):
    assert self.users[msg.sender].created_at > 0, "Not registered"
    self.users[msg.sender].balance += amount

@view
@external
def get_user(user: address) -> UserInfo:
    return self.users[user]

@view
@external
def get_user_name(user: address) -> String[50]:
    return self.users[user].name

# Update specific field
@external
def update_tier(user: address, tier: uint8):
    assert self.users[user].created_at > 0, "User not found"
    self.users[user].tier = tier
```

---

## 2. Nested Mappings {#nested}

```python
# @version 0.4.0

# ════════════════════════════════════════
# Nested HashMap[K1, HashMap[K2, V]]
# ════════════════════════════════════════

# ERC20-style allowances: owner -> spender -> amount
allowances: HashMap[address, HashMap[address, uint256]]

# Permission system: user -> permission -> enabled
permissions: HashMap[address, HashMap[bytes32, bool]]

# 2D score table: game_id -> player -> score
game_scores: HashMap[uint256, HashMap[address, uint256]]

# 3-level nesting: season -> game -> player -> score
season_scores: HashMap[uint256, HashMap[uint256, HashMap[address, uint256]]]

@external
def set_allowance(spender: address, amount: uint256):
    self.allowances[msg.sender][spender] = amount

@view
@external
def get_allowance(owner: address, spender: address) -> uint256:
    return self.allowances[owner][spender]

@external
def use_allowance(owner: address, amount: uint256):
    assert self.allowances[owner][msg.sender] >= amount, "Insufficient allowance"
    self.allowances[owner][msg.sender] -= amount

@external
def grant_permission(user: address, perm: bytes32):
    self.permissions[msg.sender][perm] = True

@view
@external
def has_permission(user: address, granter: address, perm: bytes32) -> bool:
    return self.permissions[granter][perm]

@external
def submit_score(game_id: uint256, score: uint256):
    current: uint256 = self.game_scores[game_id][msg.sender]
    if score > current:
        self.game_scores[game_id][msg.sender] = score

@view
@external
def get_score(game_id: uint256, player: address) -> uint256:
    return self.game_scores[game_id][player]
```

### Nested Mapping Patterns

```python
# @version 0.4.0

# ════════════════════════════════════════
# Advanced Nested Mapping Patterns
# ════════════════════════════════════════

# Relationship mapping: A <-> B (bidirectional)
follows: HashMap[address, HashMap[address, bool]]  # follower -> following -> bool
follower_count: HashMap[address, uint256]

@external
def follow(target: address):
    assert target != msg.sender, "Cannot follow yourself"
    assert target != empty(address), "Invalid address"
    if not self.follows[msg.sender][target]:
        self.follows[msg.sender][target] = True
        self.follower_count[target] += 1

@external
def unfollow(target: address):
    if self.follows[msg.sender][target]:
        self.follows[msg.sender][target] = False
        self.follower_count[target] -= 1

@view
@external
def is_following(follower: address, target: address) -> bool:
    return self.follows[follower][target]

# Vote tracking: proposal -> voter -> has_voted
voted: HashMap[uint256, HashMap[address, bool]]
vote_count: HashMap[uint256, uint256]

@external
def vote(proposal_id: uint256):
    assert not self.voted[proposal_id][msg.sender], "Already voted"
    self.voted[proposal_id][msg.sender] = True
    self.vote_count[proposal_id] += 1

@view
@external
def has_voted(proposal_id: uint256, voter: address) -> bool:
    return self.voted[proposal_id][voter]

@view
@external
def get_votes(proposal_id: uint256) -> uint256:
    return self.vote_count[proposal_id]
```

---

## 3. Mapping Patterns {#patterns}

### Pattern: Ownership

```python
# @version 0.4.0

owner_of: HashMap[uint256, address]    # token_id -> owner
owned_count: HashMap[address, uint256]  # owner -> count

@external
def mint(to: address, token_id: uint256):
    assert self.owner_of[token_id] == empty(address), "Already minted"
    assert to != empty(address), "Invalid address"
    self.owner_of[token_id] = to
    self.owned_count[to] += 1

@external
def transfer(to: address, token_id: uint256):
    assert self.owner_of[token_id] == msg.sender, "Not owner"
    assert to != empty(address), "Invalid address"
    self.owned_count[msg.sender] -= 1
    self.owner_of[token_id] = to
    self.owned_count[to] += 1

@view
@external
def owner(token_id: uint256) -> address:
    return self.owner_of[token_id]

@view
@external
def balance_of(account: address) -> uint256:
    return self.owned_count[account]
```

### Pattern: Whitelist/Blacklist

```python
# @version 0.4.0

owner: address
whitelist: HashMap[address, bool]
blacklist: HashMap[address, bool]
whitelist_expiry: HashMap[address, uint256]  # 0 = never expires

@deploy
def __init__():
    self.owner = msg.sender

@internal
def _only_owner():
    assert msg.sender == self.owner, "Not owner"

@external
def add_to_whitelist(addr: address, expiry: uint256):
    self._only_owner()
    self.whitelist[addr] = True
    self.whitelist_expiry[addr] = expiry

@external
def remove_from_whitelist(addr: address):
    self._only_owner()
    self.whitelist[addr] = False

@external
def ban(addr: address):
    self._only_owner()
    self.blacklist[addr] = True
    self.whitelist[addr] = False  # Remove from whitelist too

@view
@external
def is_allowed(addr: address) -> bool:
    if self.blacklist[addr]:
        return False
    if not self.whitelist[addr]:
        return False
    expiry: uint256 = self.whitelist_expiry[addr]
    if expiry > 0 and block.timestamp > expiry:
        return False
    return True
```

### Pattern: Counter Mapping

```python
# @version 0.4.0

# Counters for various metrics
action_count: HashMap[address, uint256]
daily_actions: HashMap[uint256, HashMap[address, uint256]]  # day -> user -> count
MAX_DAILY: constant(uint256) = 10

@view
@internal
def _today() -> uint256:
    return block.timestamp / 86400  # Day number

@external
def perform_action():
    today: uint256 = self._today()
    assert self.daily_actions[today][msg.sender] < MAX_DAILY, "Daily limit reached"
    self.daily_actions[today][msg.sender] += 1
    self.action_count[msg.sender] += 1

@view
@external
def get_daily_usage(user: address) -> uint256:
    return self.daily_actions[self._today()][user]

@view
@external
def get_total_actions(user: address) -> uint256:
    return self.action_count[user]
```

### Pattern: Deposit/Withdrawal Tracking

```python
# @version 0.4.0

deposits: HashMap[address, uint256]
total_deposits: uint256
lock_until: HashMap[address, uint256]

event Deposited:
    user: indexed(address)
    amount: uint256

event Withdrawn:
    user: indexed(address)
    amount: uint256

@deploy
def __init__():
    pass

@payable
@external
def deposit(lock_duration: uint256):
    assert msg.value > 0, "Zero deposit"
    self.deposits[msg.sender] += msg.value
    self.total_deposits += msg.value
    if lock_duration > 0:
        new_lock: uint256 = block.timestamp + lock_duration
        if new_lock > self.lock_until[msg.sender]:
            self.lock_until[msg.sender] = new_lock
    log Deposited(msg.sender, msg.value)

@external
def withdraw(amount: uint256):
    assert self.deposits[msg.sender] >= amount, "Insufficient"
    assert block.timestamp >= self.lock_until[msg.sender], "Funds locked"
    self.deposits[msg.sender] -= amount
    self.total_deposits -= amount
    send(msg.sender, amount)
    log Withdrawn(msg.sender, amount)

@view
@external
def get_deposit(user: address) -> uint256:
    return self.deposits[user]

@view
@external
def is_locked(user: address) -> bool:
    return block.timestamp < self.lock_until[user]
```

---

## 4. ไม่สามารถ Iterate Mappings {#no-iterate}

Mapping ใน Vyper ไม่สามารถ iterate ได้เพราะไม่รู้ว่ามี key อะไรบ้าง

```python
# @version 0.4.0

balances: HashMap[address, uint256]

# ❌ ไม่สามารถทำแบบนี้:
# for key in self.balances:  # Error! Cannot iterate mapping
#     pass

# ❌ ไม่สามารถนับจำนวน keys:
# count = len(self.balances)  # Error!

# ✅ แก้ปัญหาด้วย Parallel Array
users: DynArray[address, 1000]
user_balances: HashMap[address, uint256]
user_exists: HashMap[address, bool]

@deploy
def __init__():
    pass

@external
def add_user(user: address):
    if not self.user_exists[user]:
        self.users.append(user)
        self.user_exists[user] = True

@external
def set_balance(user: address, amount: uint256):
    assert self.user_exists[user], "User not found"
    self.user_balances[user] = amount

# ✅ สามารถ iterate users array แล้ว lookup mapping
@view
@external
def total_balance() -> uint256:
    total: uint256 = 0
    for user: address in self.users:
        total += self.user_balances[user]
    return total

@view
@external
def get_all_balances() -> DynArray[uint256, 1000]:
    result: DynArray[uint256, 1000] = []
    for user: address in self.users:
        result.append(self.user_balances[user])
    return result

@view
@external
def user_count() -> uint256:
    return convert(len(self.users), uint256)
```

---

## 5. Pattern: Enumerable Mapping {#enumerable}

Pattern สำหรับทำให้ Mapping สามารถ iterate ได้

```python
# @version 0.4.0

# ════════════════════════════════════════
# Enumerable Mapping Pattern
# HashMap ที่ iterate ได้
# ════════════════════════════════════════

MAX_SIZE: constant(uint256) = 500

struct EnumMap:
    # The actual mapping (for O(1) lookup)
    values: HashMap[address, uint256]
    # Ordered list of keys (for iteration)
    keys: DynArray[address, 500]
    # Index tracking (for O(1) removal)
    key_index: HashMap[address, uint256]  # key -> index+1 (0 means not present)

# Storage
balances_map: HashMap[address, uint256]
balances_keys: DynArray[address, 500]
balances_index: HashMap[address, uint256]  # index + 1 (0 = not exists)

@deploy
def __init__():
    pass

@internal
def _set(key: address, value: uint256):
    if self.balances_index[key] == 0:
        # New key
        assert convert(len(self.balances_keys), uint256) < MAX_SIZE, "Map full"
        self.balances_keys.append(key)
        self.balances_index[key] = convert(len(self.balances_keys), uint256)  # index + 1
    self.balances_map[key] = value

@internal
def _remove(key: address):
    idx: uint256 = self.balances_index[key]
    if idx == 0:
        return  # Key not found
    # Swap with last
    last_key: address = self.balances_keys[convert(len(self.balances_keys), uint256) - 1]
    self.balances_keys[idx - 1] = last_key
    self.balances_index[last_key] = idx
    # Pop last
    self.balances_keys.pop()
    self.balances_index[key] = 0
    self.balances_map[key] = 0

@external
def set_balance(account: address, amount: uint256):
    assert account != empty(address), "Invalid address"
    if amount == 0:
        self._remove(account)
    else:
        self._set(account, amount)

@external
def add_balance(account: address, amount: uint256):
    new_val: uint256 = self.balances_map[account] + amount
    self._set(account, new_val)

@view
@external
def get_balance(account: address) -> uint256:
    return self.balances_map[account]

@view
@external
def contains(account: address) -> bool:
    return self.balances_index[account] > 0

@view
@external
def size() -> uint256:
    return convert(len(self.balances_keys), uint256)

# Iteration is now possible!
@view
@external
def total_balance() -> uint256:
    total: uint256 = 0
    for key: address in self.balances_keys:
        total += self.balances_map[key]
    return total

@view
@external
def get_all_accounts() -> DynArray[address, 500]:
    return self.balances_keys

@view
@external
def get_all_balances() -> DynArray[uint256, 500]:
    result: DynArray[uint256, 500] = []
    for key: address in self.balances_keys:
        result.append(self.balances_map[key])
    return result

@view
@external
def get_page(offset: uint256, limit: uint256) -> DynArray[address, 50]:
    assert limit <= 50, "Max 50 per page"
    n: uint256 = convert(len(self.balances_keys), uint256)
    result: DynArray[address, 50] = []
    end: uint256 = offset + limit
    if end > n:
        end = n
    for i: uint256 in range(offset, offset + 50, bound=50):
        if i >= end:
            break
        result.append(self.balances_keys[i])
    return result
```

---

## 6. ตัวอย่าง: Token Allowance Contract {#example}

Contract ที่จัดการ Token Allowances ครบ ERC-20 standard

```python
# @version 0.4.0

# ════════════════════════════════════════════════════════════
# ERC20Token Contract with Full Allowance System
# ════════════════════════════════════════════════════════════

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Interface
# ━━━━━━━━━━━━━━━━━━━━━━━━━
interface IERC20:
    def transfer(to: address, amount: uint256) -> bool: nonpayable
    def transferFrom(sender: address, recipient: address, amount: uint256) -> bool: nonpayable
    def approve(spender: address, amount: uint256) -> bool: nonpayable
    def allowance(owner: address, spender: address) -> uint256: view
    def balanceOf(account: address) -> uint256: view
    def totalSupply() -> uint256: view

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Constants
# ━━━━━━━━━━━━━━━━━━━━━━━━━
NAME: constant(String[20]) = "AllowanceToken"
SYMBOL: constant(String[6]) = "ALLOW"
DECIMALS: constant(uint8) = 18
MAX_SUPPLY: constant(uint256) = 1_000_000_000 * 10**18  # 1 billion

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# State Variables
# ━━━━━━━━━━━━━━━━━━━━━━━━━
owner: address
total_supply: uint256

# Core mappings
balances: HashMap[address, uint256]
allowances: HashMap[address, HashMap[address, uint256]]

# Operator approvals (all tokens)
operators: HashMap[address, HashMap[address, bool]]

# Nonces for permit (EIP-2612)
nonces: HashMap[address, uint256]

# Blacklist
is_blocked: HashMap[address, bool]

# Spending limits per day
daily_limit: HashMap[address, uint256]
daily_spent: HashMap[address, HashMap[uint256, uint256]]  # user -> day -> spent

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Events
# ━━━━━━━━━━━━━━━━━━━━━━━━━
event Transfer:
    sender: indexed(address)
    recipient: indexed(address)
    amount: uint256

event Approval:
    owner: indexed(address)
    spender: indexed(address)
    amount: uint256

event OperatorSet:
    owner: indexed(address)
    operator: indexed(address)
    approved: bool

event DailyLimitSet:
    user: indexed(address)
    limit: uint256

event Blocked:
    account: indexed(address)

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Constructor
# ━━━━━━━━━━━━━━━━━━━━━━━━━
@deploy
def __init__(initial_supply: uint256):
    assert initial_supply <= MAX_SUPPLY, "Exceeds max supply"
    self.owner = msg.sender
    self.total_supply = initial_supply
    self.balances[msg.sender] = initial_supply
    log Transfer(empty(address), msg.sender, initial_supply)

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Internal Functions
# ━━━━━━━━━━━━━━━━━━━━━━━━━

@view
@internal
def _today() -> uint256:
    return block.timestamp / 86400

@internal
def _check_and_update_daily_limit(user: address, amount: uint256):
    limit: uint256 = self.daily_limit[user]
    if limit == 0:
        return  # No limit set
    today: uint256 = self._today()
    spent: uint256 = self.daily_spent[user][today]
    assert spent + amount <= limit, "Daily limit exceeded"
    self.daily_spent[user][today] += amount

@internal
def _transfer(sender: address, recipient: address, amount: uint256):
    assert sender != empty(address), "Transfer from zero"
    assert recipient != empty(address), "Transfer to zero"
    assert not self.is_blocked[sender], "Sender blocked"
    assert not self.is_blocked[recipient], "Recipient blocked"
    assert self.balances[sender] >= amount, "Insufficient balance"
    assert amount > 0, "Zero transfer"

    self.balances[sender] -= amount
    self.balances[recipient] += amount
    log Transfer(sender, recipient, amount)

@internal
def _approve(owner_addr: address, spender: address, amount: uint256):
    assert owner_addr != empty(address), "Approve from zero"
    assert spender != empty(address), "Approve to zero"
    self.allowances[owner_addr][spender] = amount
    log Approval(owner_addr, spender, amount)

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# External ERC20 Functions
# ━━━━━━━━━━━━━━━━━━━━━━━━━

@external
def transfer(to: address, amount: uint256) -> bool:
    self._check_and_update_daily_limit(msg.sender, amount)
    self._transfer(msg.sender, to, amount)
    return True

@external
def transferFrom(sender: address, recipient: address, amount: uint256) -> bool:
    # Check operator approval (unlimited)
    if not self.operators[sender][msg.sender]:
        # Check specific allowance
        current_allowance: uint256 = self.allowances[sender][msg.sender]
        assert current_allowance >= amount, "Insufficient allowance"
        # Infinite allowance: max_value(uint256)
        if current_allowance != max_value(uint256):
            self.allowances[sender][msg.sender] -= amount
            log Approval(sender, msg.sender, self.allowances[sender][msg.sender])

    self._check_and_update_daily_limit(sender, amount)
    self._transfer(sender, recipient, amount)
    return True

@external
def approve(spender: address, amount: uint256) -> bool:
    self._approve(msg.sender, spender, amount)
    return True

@external
def increase_allowance(spender: address, added_value: uint256) -> bool:
    """Increase allowance by delta (safer than approve)"""
    new_allowance: uint256 = self.allowances[msg.sender][spender] + added_value
    self._approve(msg.sender, spender, new_allowance)
    return True

@external
def decrease_allowance(spender: address, subtracted_value: uint256) -> bool:
    """Decrease allowance by delta"""
    current: uint256 = self.allowances[msg.sender][spender]
    assert current >= subtracted_value, "Decreased below zero"
    self._approve(msg.sender, spender, current - subtracted_value)
    return True

@external
def set_operator(operator: address, approved: bool):
    """Set operator approval for all tokens"""
    assert operator != msg.sender, "Cannot be own operator"
    assert operator != empty(address), "Invalid operator"
    self.operators[msg.sender][operator] = approved
    log OperatorSet(msg.sender, operator, approved)

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Extended Allowance Features
# ━━━━━━━━━━━━━━━━━━━━━━━━━

@external
def set_daily_limit(limit: uint256):
    """Set daily spending limit for yourself"""
    self.daily_limit[msg.sender] = limit
    log DailyLimitSet(msg.sender, limit)

@view
@external
def get_daily_remaining(user: address) -> uint256:
    """Get remaining daily allowance"""
    limit: uint256 = self.daily_limit[user]
    if limit == 0:
        return max_value(uint256)  # No limit
    spent: uint256 = self.daily_spent[user][self._today()]
    if spent >= limit:
        return 0
    return limit - spent

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Admin Functions
# ━━━━━━━━━━━━━━━━━━━━━━━━━

@external
def mint(to: address, amount: uint256):
    assert msg.sender == self.owner, "Not owner"
    assert to != empty(address), "Invalid address"
    assert self.total_supply + amount <= MAX_SUPPLY, "Exceeds max"
    self.total_supply += amount
    self.balances[to] += amount
    log Transfer(empty(address), to, amount)

@external
def burn(amount: uint256):
    assert self.balances[msg.sender] >= amount, "Insufficient"
    self.balances[msg.sender] -= amount
    self.total_supply -= amount
    log Transfer(msg.sender, empty(address), amount)

@external
def block_account(account: address):
    assert msg.sender == self.owner, "Not owner"
    self.is_blocked[account] = True
    log Blocked(account)

@external
def unblock_account(account: address):
    assert msg.sender == self.owner, "Not owner"
    self.is_blocked[account] = False

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# View Functions
# ━━━━━━━━━━━━━━━━━━━━━━━━━

@view
@external
def name() -> String[20]:
    return NAME

@view
@external
def symbol() -> String[6]:
    return SYMBOL

@view
@external
def decimals() -> uint8:
    return DECIMALS

@view
@external
def totalSupply() -> uint256:
    return self.total_supply

@view
@external
def balanceOf(account: address) -> uint256:
    return self.balances[account]

@view
@external
def allowance(owner_addr: address, spender: address) -> uint256:
    return self.allowances[owner_addr][spender]

@view
@external
def is_operator(owner_addr: address, operator: address) -> bool:
    return self.operators[owner_addr][operator]

@view
@external
def is_account_blocked(account: address) -> bool:
    return self.is_blocked[account]

@view
@external
def get_nonce(account: address) -> uint256:
    return self.nonces[account]
```

### Test Code

```python
# tests/test_token_allowance.py
import pytest
from brownie import ERC20Token, accounts, reverts

INITIAL_SUPPLY = 1_000_000 * 10**18

@pytest.fixture
def token(accounts):
    return ERC20Token.deploy(INITIAL_SUPPLY, {'from': accounts[0]})

class TestBasicTransfer:
    def test_initial_balance(self, token, accounts):
        assert token.balanceOf(accounts[0]) == INITIAL_SUPPLY

    def test_transfer(self, token, accounts):
        amount = 1000 * 10**18
        token.transfer(accounts[1], amount, {'from': accounts[0]})
        assert token.balanceOf(accounts[1]) == amount
        assert token.balanceOf(accounts[0]) == INITIAL_SUPPLY - amount

    def test_transfer_zero_reverts(self, token, accounts):
        with reverts("Zero transfer"):
            token.transfer(accounts[1], 0, {'from': accounts[0]})

class TestAllowance:
    def test_approve_and_transfer_from(self, token, accounts):
        amount = 500 * 10**18
        token.approve(accounts[1], amount, {'from': accounts[0]})
        assert token.allowance(accounts[0], accounts[1]) == amount

        token.transferFrom(accounts[0], accounts[2], amount, {'from': accounts[1]})
        assert token.balanceOf(accounts[2]) == amount
        assert token.allowance(accounts[0], accounts[1]) == 0

    def test_infinite_allowance(self, token, accounts):
        MAX = 2**256 - 1
        token.approve(accounts[1], MAX, {'from': accounts[0]})
        amount = 100 * 10**18
        token.transferFrom(accounts[0], accounts[2], amount, {'from': accounts[1]})
        # Infinite allowance should not decrease
        assert token.allowance(accounts[0], accounts[1]) == MAX

    def test_increase_decrease_allowance(self, token, accounts):
        token.approve(accounts[1], 1000, {'from': accounts[0]})
        token.increase_allowance(accounts[1], 500, {'from': accounts[0]})
        assert token.allowance(accounts[0], accounts[1]) == 1500
        token.decrease_allowance(accounts[1], 300, {'from': accounts[0]})
        assert token.allowance(accounts[0], accounts[1]) == 1200

    def test_operator_approval(self, token, accounts):
        token.set_operator(accounts[1], True, {'from': accounts[0]})
        assert token.is_operator(accounts[0], accounts[1]) == True
        amount = 100 * 10**18
        token.transferFrom(accounts[0], accounts[2], amount, {'from': accounts[1]})
        assert token.balanceOf(accounts[2]) == amount

class TestDailyLimit:
    def test_daily_limit(self, token, accounts):
        limit = 1000 * 10**18
        token.set_daily_limit(limit, {'from': accounts[0]})
        token.transfer(accounts[1], limit, {'from': accounts[0]})
        remaining = token.get_daily_remaining(accounts[0])
        assert remaining == 0

    def test_daily_limit_exceeded(self, token, accounts):
        limit = 100 * 10**18
        token.set_daily_limit(limit, {'from': accounts[0]})
        with reverts("Daily limit exceeded"):
            token.transfer(accounts[1], limit + 1, {'from': accounts[0]})

class TestBlocking:
    def test_block_account(self, token, accounts):
        token.block_account(accounts[1], {'from': accounts[0]})
        with reverts("Sender blocked"):
            token.transfer(accounts[2], 100, {'from': accounts[1]})
```

---

## สรุป

- ✅ **HashMap[K, V]**: Key-Value store บน Blockchain
- ✅ **Default values**: คืน 0/false/empty ถ้าไม่มี key
- ✅ **Nested HashMap**: จัดการ 2D/3D relationships
- ✅ **ไม่ iterate ได้**: ต้องเก็บ key list แยกต่างหาก
- ✅ **Enumerable Pattern**: HashMap + DynArray keys + index mapping
- ✅ **Allowance Pattern**: ERC-20 approve/transferFrom

## แบบฝึกหัด

1. **สร้าง** Multi-token Wallet ที่ track หลาย ERC-20 tokens
2. **เพิ่ม** Time-locked Allowances (expire หลังจากวันที่กำหนด)
3. **สร้าง** Subscription System ด้วย nested mappings
4. **ทดสอบ** Reentrancy ใน transferFrom
5. **วิเคราะห์** Gas cost ของ SLOAD/SSTORE สำหรับ nested mapping

---

**ก่อนหน้า: [Part 009 - Arrays และ DynArray](part_009_arrays.md)**  
**ต่อไป: [Part 011 - Structs](part_011_structs.md)**
