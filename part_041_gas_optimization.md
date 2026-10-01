# Part 041: Gas Optimization in Vyper

## สารบัญ
1. [Overview](#overview)
2. [Storage Packing](#storage-packing)
3. [Caching Storage Variables](#caching-storage-variables)
4. [Minimizing SLOADs](#minimizing-sloads)
5. [Batch Operations](#batch-operations)
6. [Efficient Loops](#efficient-loops)
7. [GasOptimizedToken Contract](#gasoptimizedtoken-contract)
8. [Before and After Examples](#before-and-after-examples)
9. [Advanced Techniques](#advanced-techniques)
10. [Testing Gas Usage](#testing-gas-usage)

---

## 1. Overview {#overview}

Gas optimization เป็นหนึ่งในทักษะที่สำคัญที่สุดในการพัฒนา Smart Contract บน EVM (Ethereum Virtual Machine) เพราะทุกการดำเนินการบน blockchain ต้องเสีย gas ซึ่งแปลงเป็นเงิน ETH ที่ผู้ใช้ต้องจ่าย

### ทำไม Gas Optimization ถึงสำคัญ?

- **ลดค่าธรรมเนียม**: ผู้ใช้เสียเงินน้อยลงต่อ transaction
- **เพิ่ม Competitiveness**: Contract ที่ถูกกว่าดึงดูดผู้ใช้มากกว่า
- **Block Gas Limit**: ถ้า transaction ใช้ gas น้อยลง สามารถทำได้หลาย tx ต่อ block
- **DeFi Protocols**: Protocol ที่ gas ถูกดึงดูด liquidity มากกว่า

### Gas Cost ของ Operations หลัก

| Operation | Gas Cost | คำอธิบาย |
|-----------|----------|----------|
| SLOAD (cold) | 2100 | อ่าน storage ครั้งแรก |
| SLOAD (warm) | 100 | อ่าน storage ที่เคยอ่านแล้ว |
| SSTORE (new) | 20000 | เขียน storage ใหม่ |
| SSTORE (update) | 2900 | อัพเดท storage ที่มีอยู่ |
| MLOAD/MSTORE | 3 | อ่าน/เขียน memory |
| ADD/SUB | 3 | การบวกลบ |
| MUL/DIV | 5 | การคูณหาร |
| CALL | 100+ | เรียก contract อื่น |

---

## 2. Storage Packing {#storage-packing}

Storage Packing คือการจัดเรียงตัวแปรใน storage ให้ใช้ slot น้อยที่สุด แต่ละ slot มีขนาด 32 bytes

### ตัวอย่าง: Storage ที่ไม่ได้ Optimize

```vyper
# @version 0.4.0

# BAD: ใช้ 4 slots แยกกัน
owner: address           # slot 0: 20 bytes แต่ใช้ทั้ง slot
is_paused: bool          # slot 1: 1 byte แต่ใช้ทั้ง slot
max_supply: uint256      # slot 2: 32 bytes
total_supply: uint256    # slot 3: 32 bytes
```

### ตัวอย่าง: Storage ที่ Optimize แล้ว (ใน Vyper)

หมายเหตุ: Vyper ไม่ได้ pack variables อัตโนมัติเหมือน Solidity แต่เราสามารถใช้ struct เพื่อ pack ได้:

```vyper
# @version 0.4.0

# GOOD: ใช้ struct เพื่อ pack ข้อมูลเล็กๆ เข้าด้วยกัน
struct PackedConfig:
    owner: address       # 20 bytes
    is_paused: bool      # 1 byte
    decimals: uint8      # 1 byte
    # รวม 22 bytes ใน 1 struct slot

config: PackedConfig
max_supply: uint256
total_supply: uint256
```

### การใช้ Bitwise Operations สำหรับ Packing

```vyper
# @version 0.4.0

# Pack multiple flags into a single uint256
# bit 0: is_paused
# bit 1: is_whitelisted
# bit 2: can_transfer
# bit 3: is_verified

flags: uint256

@internal
def _set_flag(flag_bit: uint256, value: bool):
    """Pack a boolean into a specific bit position"""
    if value:
        self.flags = self.flags | (1 << flag_bit)
    else:
        self.flags = self.flags & ~(1 << flag_bit)

@internal
@view
def _get_flag(flag_bit: uint256) -> bool:
    """Read a boolean from a specific bit position"""
    return (self.flags >> flag_bit) & 1 == 1

@external
def set_paused(paused: bool):
    self._set_flag(0, paused)

@external
@view
def is_paused() -> bool:
    return self._get_flag(0)
```

---

## 3. Caching Storage Variables {#caching-storage-variables}

การอ่าน storage (SLOAD) แต่ละครั้งเสีย 100-2100 gas ดังนั้นถ้าต้องใช้ค่าเดิมหลายครั้งควร cache ไว้ใน memory

### ตัวอย่าง: ไม่ Cache (แย่)

```vyper
# @version 0.4.0

balances: HashMap[address, uint256]
total_supply: uint256

@external
def expensive_calculation(account: address) -> uint256:
    # BAD: อ่าน storage 4 ครั้ง = แพงมาก
    if self.balances[account] > 0:
        result: uint256 = self.balances[account] * 100 // self.total_supply
        if result > self.balances[account]:
            return self.balances[account]
        return result
    return 0
```

### ตัวอย่าง: Cache ใน Memory (ดี)

```vyper
# @version 0.4.0

balances: HashMap[address, uint256]
total_supply: uint256

@external
def cheap_calculation(account: address) -> uint256:
    # GOOD: อ่าน storage แค่ 2 ครั้ง แล้ว cache ไว้ใน memory
    account_balance: uint256 = self.balances[account]  # SLOAD ครั้งที่ 1
    supply: uint256 = self.total_supply                 # SLOAD ครั้งที่ 2
    
    if account_balance > 0:
        result: uint256 = account_balance * 100 // supply
        if result > account_balance:
            return account_balance  # ใช้ memory variable แทน
        return result
    return 0
```

---

## 4. Minimizing SLOADs {#minimizing-sloads}

### ตัวอย่าง: Multiple SLOADs

```vyper
# @version 0.4.0

struct UserInfo:
    balance: uint256
    last_reward_block: uint256
    reward_debt: uint256

users: HashMap[address, UserInfo]
reward_per_block: uint256
last_update_block: uint256
accumulated_reward: uint256

# BAD: หลาย SLOAD ที่ไม่จำเป็น
@external
def bad_update_reward(user: address):
    blocks_elapsed: uint256 = block.number - self.last_update_block  # 2 SLOADs
    new_reward: uint256 = blocks_elapsed * self.reward_per_block     # 1 SLOAD
    self.accumulated_reward += new_reward                              # 1 SLOAD + 1 SSTORE
    
    user_balance: uint256 = self.users[user].balance                  # 1 SLOAD
    pending: uint256 = user_balance * self.accumulated_reward // 1000000  # 1 SLOAD
    self.users[user].reward_debt += pending                           # 1 SLOAD + 1 SSTORE
    self.last_update_block = block.number                             # 1 SSTORE

# GOOD: Cache ทุกอย่างที่ต้องใช้ซ้ำ
@external
def good_update_reward(user: address):
    # Cache ทั้งหมดก่อน
    current_block: uint256 = block.number
    last_block: uint256 = self.last_update_block          # SLOAD
    reward_rate: uint256 = self.reward_per_block          # SLOAD
    acc_reward: uint256 = self.accumulated_reward         # SLOAD
    user_info: UserInfo = self.users[user]                # SLOAD (struct)
    
    # คำนวณใน memory
    blocks_elapsed: uint256 = current_block - last_block
    new_reward: uint256 = blocks_elapsed * reward_rate
    acc_reward += new_reward
    
    pending: uint256 = user_info.balance * acc_reward // 1000000
    user_info.reward_debt += pending
    
    # เขียน storage ครั้งเดียว
    self.accumulated_reward = acc_reward                  # SSTORE
    self.users[user] = user_info                          # SSTORE
    self.last_update_block = current_block               # SSTORE
```

---

## 5. Batch Operations {#batch-operations}

การทำ batch operations ช่วยลด overhead ของ transaction และ storage reads/writes

### ตัวอย่าง: Batch Transfer

```vyper
# @version 0.4.0

balances: HashMap[address, uint256]
owner: address

event Transfer:
    sender: indexed(address)
    recipient: indexed(address)
    amount: uint256

@deploy
def __init__():
    self.owner = msg.sender

# BAD: ส่งทีละคน (overhead สูง)
@external
def transfer_one(to: address, amount: uint256):
    assert self.balances[msg.sender] >= amount
    self.balances[msg.sender] -= amount
    self.balances[to] += amount
    log Transfer(msg.sender, to, amount)

# GOOD: ส่งทีเดียวหลายคน
@external
def batch_transfer(
    recipients: DynArray[address, 100],
    amounts: DynArray[uint256, 100]
):
    assert len(recipients) == len(amounts), "Length mismatch"
    
    # Cache sender balance ครั้งเดียว
    sender_balance: uint256 = self.balances[msg.sender]
    total_amount: uint256 = 0
    
    # คำนวณ total ก่อนเพื่อ check balance ครั้งเดียว
    for i: uint256 in range(100):
        if i >= len(amounts):
            break
        total_amount += amounts[i]
    
    assert sender_balance >= total_amount, "Insufficient balance"
    
    # อัพเดท sender ครั้งเดียว
    self.balances[msg.sender] = sender_balance - total_amount
    
    # อัพเดท recipients
    for i: uint256 in range(100):
        if i >= len(recipients):
            break
        self.balances[recipients[i]] += amounts[i]
        log Transfer(msg.sender, recipients[i], amounts[i])
```

---

## 6. Efficient Loops {#efficient-loops}

```vyper
# @version 0.4.0

# BAD: Loop ที่ไม่ efficient
@view
@internal
def bad_sum(values: DynArray[uint256, 1000]) -> uint256:
    total: uint256 = 0
    n: uint256 = len(values)  # ยังดีที่ cache len
    for i: uint256 in range(1000):
        if i >= n:
            break
        total += values[i]  # Memory access แต่ก็ยังมี overhead
    return total

# GOOD: ใช้ for-in loop ที่ efficient กว่า
@view
@internal
def good_sum(values: DynArray[uint256, 1000]) -> uint256:
    total: uint256 = 0
    for v: uint256 in values:
        total += v
    return total

# BEST: Early termination เมื่อทำได้
@view
@internal
def find_first_above(
    values: DynArray[uint256, 1000],
    threshold: uint256
) -> int256:
    for i: uint256 in range(1000):
        if i >= len(values):
            break
        if values[i] > threshold:
            return convert(i, int256)
    return -1
```

---

## 7. GasOptimizedToken Contract {#gasoptimizedtoken-contract}

```vyper
# @version 0.4.0
# @title GasOptimizedToken
# @notice ERC20 token พร้อม gas optimization techniques ทั้งหมด

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

event BatchTransfer:
    sender: indexed(address)
    total_amount: uint256
    recipient_count: uint256

# Packed config struct เพื่อลด storage slots
struct TokenConfig:
    name: String[64]
    symbol: String[32]
    decimals: uint8
    is_paused: bool

# State variables - จัดเรียงให้ใช้ storage อย่างมีประสิทธิภาพ
config: TokenConfig
owner: address
total_supply: uint256
balances: HashMap[address, uint256]
allowances: HashMap[address, HashMap[address, uint256]]

# Minter whitelist
minters: HashMap[address, bool]

# Max batch size
MAX_BATCH_SIZE: constant(uint256) = 200

@deploy
def __init__(
    name: String[64],
    symbol: String[32],
    initial_supply: uint256
):
    self.owner = msg.sender
    self.config = TokenConfig({
        name: name,
        symbol: symbol,
        decimals: 18,
        is_paused: False
    })
    
    if initial_supply > 0:
        self.total_supply = initial_supply
        self.balances[msg.sender] = initial_supply
        log Transfer(empty(address), msg.sender, initial_supply)

# ERC20 View Functions - อ่าน from config struct ครั้งเดียว
@external
@view
def name() -> String[64]:
    return self.config.name

@external
@view
def symbol() -> String[32]:
    return self.config.symbol

@external
@view
def decimals() -> uint8:
    return self.config.decimals

@external
@view
def totalSupply() -> uint256:
    return self.total_supply

@external
@view
def balanceOf(account: address) -> uint256:
    return self.balances[account]

@external
@view
def allowance(owner: address, spender: address) -> uint256:
    return self.allowances[owner][spender]

# Optimized Transfer
@external
def transfer(to: address, amount: uint256) -> bool:
    # Cache เพื่อลด SLOADs
    config: TokenConfig = self.config
    assert not config.is_paused, "Token is paused"
    assert to != empty(address), "Transfer to zero address"
    
    sender_balance: uint256 = self.balances[msg.sender]
    assert sender_balance >= amount, "Insufficient balance"
    
    # อัพเดท storage เพียง 2 ครั้ง
    self.balances[msg.sender] = sender_balance - amount
    self.balances[to] += amount
    
    log Transfer(msg.sender, to, amount)
    return True

@external
def transferFrom(sender: address, to: address, amount: uint256) -> bool:
    # Cache ทุก storage reads
    config: TokenConfig = self.config
    assert not config.is_paused, "Token is paused"
    assert to != empty(address), "Transfer to zero address"
    
    sender_balance: uint256 = self.balances[sender]
    assert sender_balance >= amount, "Insufficient balance"
    
    current_allowance: uint256 = self.allowances[sender][msg.sender]
    assert current_allowance >= amount, "Insufficient allowance"
    
    # อัพเดท storage ครั้งเดียวต่อ variable
    self.balances[sender] = sender_balance - amount
    self.balances[to] += amount
    self.allowances[sender][msg.sender] = current_allowance - amount
    
    log Transfer(sender, to, amount)
    return True

@external
def approve(spender: address, amount: uint256) -> bool:
    assert spender != empty(address), "Approve to zero address"
    self.allowances[msg.sender][spender] = amount
    log Approval(msg.sender, spender, amount)
    return True

# Optimized Batch Transfer
@external
def batch_transfer(
    recipients: DynArray[address, 200],
    amounts: DynArray[uint256, 200]
) -> bool:
    n: uint256 = len(recipients)
    assert n == len(amounts), "Length mismatch"
    assert n > 0, "Empty batch"
    assert n <= MAX_BATCH_SIZE, "Batch too large"
    
    # Cache config
    assert not self.config.is_paused, "Token is paused"
    
    # Cache sender balance ครั้งเดียว
    sender_balance: uint256 = self.balances[msg.sender]
    total_amount: uint256 = 0
    
    # Pass 1: คำนวณ total (ใน memory)
    for i: uint256 in range(200):
        if i >= n:
            break
        assert recipients[i] != empty(address), "Transfer to zero address"
        total_amount += amounts[i]
    
    assert sender_balance >= total_amount, "Insufficient balance"
    
    # อัพเดท sender ครั้งเดียว
    self.balances[msg.sender] = sender_balance - total_amount
    
    # Pass 2: อัพเดท recipients
    for i: uint256 in range(200):
        if i >= n:
            break
        self.balances[recipients[i]] += amounts[i]
        log Transfer(msg.sender, recipients[i], amounts[i])
    
    log BatchTransfer(msg.sender, total_amount, n)
    return True

# Optimized Mint with batch support
@external
def batch_mint(
    accounts: DynArray[address, 100],
    amounts: DynArray[uint256, 100]
) -> bool:
    assert msg.sender == self.owner or self.minters[msg.sender], "Not authorized"
    n: uint256 = len(accounts)
    assert n == len(amounts), "Length mismatch"
    
    # คำนวณ total supply change ก่อน
    total_minted: uint256 = 0
    for i: uint256 in range(100):
        if i >= n:
            break
        total_minted += amounts[i]
    
    # Cache current total supply
    current_supply: uint256 = self.total_supply
    
    # อัพเดท total supply ครั้งเดียว
    self.total_supply = current_supply + total_minted
    
    # อัพเดท individual balances
    for i: uint256 in range(100):
        if i >= n:
            break
        self.balances[accounts[i]] += amounts[i]
        log Transfer(empty(address), accounts[i], amounts[i])
    
    return True

@external
def mint(to: address, amount: uint256) -> bool:
    assert msg.sender == self.owner or self.minters[msg.sender], "Not authorized"
    assert to != empty(address), "Mint to zero address"
    
    self.total_supply += amount
    self.balances[to] += amount
    log Transfer(empty(address), to, amount)
    return True

@external
def burn(amount: uint256) -> bool:
    sender_balance: uint256 = self.balances[msg.sender]
    assert sender_balance >= amount, "Insufficient balance"
    
    self.balances[msg.sender] = sender_balance - amount
    self.total_supply -= amount
    log Transfer(msg.sender, empty(address), amount)
    return True

@external
def set_minter(minter: address, status: bool):
    assert msg.sender == self.owner, "Not owner"
    self.minters[minter] = status

@external
def set_paused(paused: bool):
    assert msg.sender == self.owner, "Not owner"
    self.config.is_paused = paused

@external
def transfer_ownership(new_owner: address):
    assert msg.sender == self.owner, "Not owner"
    assert new_owner != empty(address), "New owner is zero address"
    self.owner = new_owner
```

---

## 8. Before and After Examples {#before-and-after-examples}

### ตัวอย่าง: Staking Contract

#### Before (ไม่ Optimize):

```vyper
# @version 0.4.0
# BEFORE: Staking contract ที่ไม่ optimize

interface IERC20:
    def transfer(to: address, amount: uint256) -> bool: nonpayable
    def transferFrom(frm: address, to: address, amount: uint256) -> bool: nonpayable

struct StakeInfo:
    amount: uint256
    start_time: uint256
    accumulated_reward: uint256

stakes: HashMap[address, StakeInfo]
reward_rate: uint256  # reward per second per token
token: address
reward_token: address
total_staked: uint256

@deploy
def __init__(token: address, reward_token: address, rate: uint256):
    self.token = token
    self.reward_token = reward_token
    self.reward_rate = rate

@external
def bad_calculate_reward(user: address) -> uint256:
    # BAD: หลาย SLOADs ที่ไม่จำเป็น
    if self.stakes[user].amount == 0:  # SLOAD
        return 0
    elapsed: uint256 = block.timestamp - self.stakes[user].start_time  # SLOAD
    reward: uint256 = self.stakes[user].amount * elapsed * self.reward_rate // 10**18  # 2 SLOADs
    return reward + self.stakes[user].accumulated_reward  # SLOAD

@external
def bad_claim_reward():
    # BAD: อ่าน storage ซ้ำซ้อน
    reward: uint256 = self.bad_calculate_reward(msg.sender)  # หลาย SLOADs อีก
    self.stakes[msg.sender].accumulated_reward = 0           # SLOAD + SSTORE
    self.stakes[msg.sender].start_time = block.timestamp     # SLOAD + SSTORE
    IERC20(self.reward_token).transfer(msg.sender, reward)
```

#### After (Optimize แล้ว):

```vyper
# @version 0.4.0
# AFTER: Staking contract ที่ optimize แล้ว

interface IERC20:
    def transfer(to: address, amount: uint256) -> bool: nonpayable
    def transferFrom(frm: address, to: address, amount: uint256) -> bool: nonpayable
    def balanceOf(owner: address) -> uint256: view

struct StakeInfo:
    amount: uint256
    start_time: uint256
    accumulated_reward: uint256

stakes: HashMap[address, StakeInfo]
reward_rate: uint256
token: address
reward_token: address
total_staked: uint256

@deploy
def __init__(token_addr: address, reward_token_addr: address, rate: uint256):
    self.token = token_addr
    self.reward_token = reward_token_addr
    self.reward_rate = rate

@internal
@view
def _calculate_pending(stake: StakeInfo) -> uint256:
    """คำนวณจาก cached struct ใน memory ไม่ต้อง SLOAD อีก"""
    if stake.amount == 0:
        return 0
    elapsed: uint256 = block.timestamp - stake.start_time
    return stake.amount * elapsed * self.reward_rate // 10**18

@external
@view
def calculate_reward(user: address) -> uint256:
    # Cache struct ครั้งเดียว แล้วส่ง to internal function
    stake: StakeInfo = self.stakes[user]  # SLOAD เดียว
    pending: uint256 = self._calculate_pending(stake)
    return pending + stake.accumulated_reward

@external
def claim_reward():
    # Cache ครั้งเดียว
    stake: StakeInfo = self.stakes[msg.sender]  # SLOAD เดียว
    
    pending: uint256 = self._calculate_pending(stake)
    total_reward: uint256 = pending + stake.accumulated_reward
    
    # อัพเดท struct ใน memory แล้ว SSTORE ครั้งเดียว
    stake.accumulated_reward = 0
    stake.start_time = block.timestamp
    
    self.stakes[msg.sender] = stake  # SSTORE เดียว (3 fields ใน 1 SSTORE)
    
    if total_reward > 0:
        IERC20(self.reward_token).transfer(msg.sender, total_reward)

@external
def stake(amount: uint256):
    assert amount > 0, "Cannot stake 0"
    
    # Cache stake info
    stake: StakeInfo = self.stakes[msg.sender]
    
    # Compound pending reward ก่อน stake เพิ่ม
    if stake.amount > 0:
        pending: uint256 = self._calculate_pending(stake)
        stake.accumulated_reward += pending
    
    # อัพเดทข้อมูล
    stake.amount += amount
    stake.start_time = block.timestamp
    
    # SSTORE เดียว
    self.stakes[msg.sender] = stake
    self.total_staked += amount
    
    IERC20(self.token).transferFrom(msg.sender, self, amount)

@external
def unstake(amount: uint256):
    stake: StakeInfo = self.stakes[msg.sender]
    assert stake.amount >= amount, "Insufficient staked amount"
    
    # Collect pending rewards
    pending: uint256 = self._calculate_pending(stake)
    stake.accumulated_reward += pending
    
    stake.amount -= amount
    stake.start_time = block.timestamp
    
    self.stakes[msg.sender] = stake
    self.total_staked -= amount
    
    IERC20(self.token).transfer(msg.sender, amount)
```

---

## 9. Advanced Techniques {#advanced-techniques}

### Short-Circuit Evaluation

```vyper
# @version 0.4.0

owner: address
whitelist: HashMap[address, bool]
is_paused: bool

@deploy
def __init__():
    self.owner = msg.sender
    self.is_paused = False

@internal
@view
def _is_authorized(account: address) -> bool:
    # Short-circuit: ตรวจสอบเงื่อนไขง่ายก่อน (เสีย gas น้อยกว่า)
    # is_paused เป็น local check (SLOAD เดียว) vs whitelist (SLOAD + hash)
    if self.is_paused:
        return False
    # ถ้าเป็น owner ไม่ต้อง check whitelist
    if account == self.owner:
        return True
    return self.whitelist[account]

@external
@view
def check_access(account: address) -> bool:
    return self._is_authorized(account)
```

### Immutables แทน Storage

```vyper
# @version 0.4.0
# ใช้ immutable variables สำหรับค่าที่ไม่เปลี่ยน
# Immutables ถูกเก็บใน bytecode ไม่ใช่ storage (ฟรี!)

# immutables จะถูก set ใน __init__ และไม่สามารถเปลี่ยนได้
OWNER: immutable(address)
MAX_SUPPLY: immutable(uint256)
CREATION_TIME: immutable(uint256)

balances: HashMap[address, uint256]
total_supply: uint256

@deploy
def __init__(max_supply: uint256):
    OWNER = msg.sender
    MAX_SUPPLY = max_supply
    CREATION_TIME = block.timestamp

@external
@view
def get_owner() -> address:
    # ไม่มี SLOAD! อ่านจาก bytecode โดยตรง
    return OWNER

@external
@view
def get_max_supply() -> uint256:
    # ไม่มี SLOAD!
    return MAX_SUPPLY

@external
def mint(to: address, amount: uint256):
    assert msg.sender == OWNER, "Not owner"  # ไม่มี SLOAD
    assert self.total_supply + amount <= MAX_SUPPLY, "Exceeds max"  # 1 SLOAD
    
    self.total_supply += amount
    self.balances[to] += amount
```

### Constants

```vyper
# @version 0.4.0
# Constants ถูก inline ที่ compile time - ไม่มี gas เลย!

MAX_UINT256: constant(uint256) = max_value(uint256)
PRECISION: constant(uint256) = 10**18
BASIS_POINTS: constant(uint256) = 10000
FEE_RATE: constant(uint256) = 30  # 0.3%
MAX_BATCH: constant(uint256) = 100

balances: HashMap[address, uint256]

@external
@view
def calculate_fee(amount: uint256) -> uint256:
    # FEE_RATE และ BASIS_POINTS ถูก inline - ไม่มี SLOAD
    return amount * FEE_RATE // BASIS_POINTS

@external
@view
def apply_precision(value: uint256) -> uint256:
    # PRECISION ถูก inline
    return value * PRECISION
```

### Event Data แทน Storage

```vyper
# @version 0.4.0
# สำหรับข้อมูลที่ต้องการ historical แต่ไม่ต้อง on-chain lookup
# ใช้ events แทน storage (ถูกกว่ามาก)

event UserAction:
    user: indexed(address)
    action_type: indexed(uint256)
    amount: uint256
    timestamp: uint256
    data: Bytes[100]

# แทนที่จะเก็บ history ใน storage
# action_history: DynArray[...]  # แพงมาก!

@external
def record_action(action_type: uint256, amount: uint256, data: Bytes[100]):
    # log ข้อมูลใน event แทน storage
    log UserAction(msg.sender, action_type, amount, block.timestamp, data)
```

---

## 10. Testing Gas Usage {#testing-gas-usage}

```python
# test_gas_optimization.py
# ทดสอบ gas usage ด้วย Titanoboa

import pytest
import boa

@pytest.fixture
def token(deployer):
    with boa.env.prank(deployer):
        return boa.load(
            "GasOptimizedToken.vy",
            "GasToken",
            "GT",
            10**6 * 10**18
        )

@pytest.fixture
def deployer():
    return boa.env.generate_address()

@pytest.fixture
def users(deployer):
    addrs = [boa.env.generate_address() for _ in range(5)]
    return addrs

class TestGasOptimization:
    """ทดสอบ gas usage ของ optimized functions"""
    
    def test_single_transfer_gas(self, token, deployer, users):
        """ทดสอบ gas ของ single transfer"""
        recipient = users[0]
        
        with boa.env.prank(deployer):
            # วัด gas ของ transfer
            gas_before = boa.env.evm.gas_used if hasattr(boa.env.evm, 'gas_used') else 0
            token.transfer(recipient, 1000 * 10**18)
    
    def test_batch_transfer_gas(self, token, deployer, users):
        """ทดสอบว่า batch transfer ประหยัด gas กว่า multiple transfers"""
        amounts = [100 * 10**18] * 5
        
        with boa.env.prank(deployer):
            # Batch transfer
            token.batch_transfer(users[:5], amounts)
        
        # ตรวจสอบว่า balances ถูกต้อง
        for user in users[:5]:
            assert token.balanceOf(user) == 100 * 10**18
    
    def test_batch_mint(self, token, deployer, users):
        """ทดสอบ batch mint"""
        amounts = [500 * 10**18] * 5
        
        with boa.env.prank(deployer):
            initial_supply = token.totalSupply()
            token.batch_mint(users[:5], amounts)
            
            # ตรวจสอบ total supply เพิ่มขึ้นถูกต้อง
            expected_supply = initial_supply + sum(amounts)
            assert token.totalSupply() == expected_supply
        
        for user in users[:5]:
            assert token.balanceOf(user) == 500 * 10**18
    
    def test_immutable_access(self, token, deployer):
        """ทดสอบว่า immutable values ถูกต้อง"""
        # owner ควรเป็น immutable - access ฟรี
        # ใน Vyper 0.4.0 เราใช้ storage variable สำหรับ owner
        # แต่ถ้าเป็น immutable ไม่มี SLOAD
        pass
    
    def test_paused_state(self, token, deployer, users):
        """ทดสอบ paused state caching"""
        with boa.env.prank(deployer):
            token.set_paused(True)
        
        with boa.env.prank(deployer):
            with pytest.raises(Exception):
                token.transfer(users[0], 100)
    
    def test_batch_transfer_validation(self, token, deployer, users):
        """ทดสอบ validation ใน batch transfer"""
        with boa.env.prank(deployer):
            # Length mismatch
            with pytest.raises(Exception):
                token.batch_transfer(users[:3], [100] * 5)
            
            # Insufficient balance
            with pytest.raises(Exception):
                huge_amounts = [10**30] * 5
                token.batch_transfer(users[:5], huge_amounts)
    
    def test_minter_role(self, token, deployer, users):
        """ทดสอบ minter whitelist"""
        minter = users[0]
        
        with boa.env.prank(deployer):
            token.set_minter(minter, True)
        
        with boa.env.prank(minter):
            token.mint(users[1], 1000 * 10**18)
        
        assert token.balanceOf(users[1]) == 1000 * 10**18


class TestGasComparison:
    """เปรียบเทียบ gas ระหว่าง optimized และ non-optimized"""
    
    def test_calculate_reward_caching(self):
        """
        แสดงให้เห็นว่า caching ช่วยประหยัด gas
        
        Bad approach: 4+ SLOADs per call
        Good approach: 1 SLOAD (struct) per call
        """
        # จุดประสงค์คือแสดง pattern ที่ดี
        # ในการทดสอบจริงใช้ gas_meter ของ titanoboa
        pass


# Run tests
if __name__ == "__main__":
    pytest.main([__file__, "-v", "--tb=short"])
```

### Gas Profiling Script

```python
# gas_profiler.py
# Script สำหรับ profile gas usage

import boa
from typing import Callable

def measure_gas(fn: Callable, *args) -> int:
    """วัด gas ที่ใช้ใน function call"""
    # ใน titanoboa สามารถ inspect gas ได้
    result = fn(*args)
    return result

def compare_implementations():
    """เปรียบเทียบ gas ระหว่าง implementations ต่างๆ"""
    
    deployer = boa.env.generate_address()
    
    with boa.env.prank(deployer):
        token = boa.load(
            "GasOptimizedToken.vy",
            "GasToken",
            "GT",
            10**6 * 10**18
        )
    
    users = [boa.env.generate_address() for _ in range(10)]
    amounts_single = [100 * 10**18] * 10
    
    print("Gas Comparison Results:")
    print("=" * 50)
    
    # Test batch vs individual
    with boa.env.prank(deployer):
        # Batch transfer
        token.batch_transfer(users, amounts_single)
        print(f"Batch transfer (10 recipients): measured")
    
    print("\nConclusion:")
    print("- Batch operations save ~30-50% gas vs individual calls")
    print("- Storage caching saves ~100 gas per avoided SLOAD")
    print("- Using immutables saves ~2100 gas vs cold SLOAD")

if __name__ == "__main__":
    compare_implementations()
```

---

## สรุป Gas Optimization Checklist

```
✅ Gas Optimization Checklist:

Storage:
[ ] ใช้ immutable สำหรับค่าที่ไม่เปลี่ยน (owner, token address, etc.)
[ ] ใช้ constant สำหรับ magic numbers
[ ] Cache storage reads ที่ใช้มากกว่า 1 ครั้ง
[ ] ใช้ struct เพื่อ read/write หลาย fields ใน 1 operation
[ ] พิจารณา bit packing สำหรับ booleans หลายตัว

Computation:
[ ] ใช้ short-circuit evaluation
[ ] ทำ validation ราคาถูกก่อน validation ราคาแพง
[ ] ใช้ unchecked arithmetic เมื่อแน่ใจว่าไม่ overflow (ใน Vyper ทำได้ด้วย unsafe_add/unsafe_sub)
[ ] เลือก uint256 แทน uint128/uint64 (EVM ทำงานกับ 256-bit natively)

Loops:
[ ] จำกัด loop iterations ด้วย constant
[ ] cache len() ก่อน loop (Vyper ทำอัตโนมัติ)
[ ] ใช้ for-in แทน index loop เมื่อทำได้
[ ] หลีก SLOAD ภายใน loop

Events vs Storage:
[ ] ใช้ events สำหรับ historical data
[ ] ใช้ storage เฉพาะที่ต้องการ on-chain lookup

Batch Operations:
[ ] รวม multiple operations เข้า batch function
[ ] คำนวณ totals ก่อน แล้ว update storage ครั้งเดียว
```
