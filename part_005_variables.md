# Part 005: ตัวแปรและ State Variables

## สารบัญ
1. [State Variables](#state-variables)
2. [Local Variables](#local-variables)
3. [Global Variables (Built-in)](#global-variables)
4. [Storage Slots](#storage-slots)
5. [Variable Visibility](#visibility)
6. [Default Values](#defaults)
7. [Memory vs Storage](#memory-storage)
8. [ตัวอย่างการใช้งานจริง](#real-world)

---

## 1. State Variables {#state-variables}

State Variables เก็บอยู่ใน Blockchain Storage อย่างถาวร

### การประกาศและกำหนดค่า

```python
# @version 0.4.0

# ════════════════════════
# State Variable Declarations
# ════════════════════════

# พื้นฐาน
owner: address
total_supply: uint256
name: String[50]
paused: bool

# Collections
balances: HashMap[address, uint256]
allowances: HashMap[address, HashMap[address, uint256]]
tokens: uint256[100]     # Fixed-size array
items: DynArray[address, 1000]  # Dynamic array

# Structs
struct UserInfo:
    balance: uint256
    last_update: uint256
    is_active: bool
    referrer: address

users: HashMap[address, UserInfo]

# Nested mappings
votes: HashMap[uint256, HashMap[address, bool]]

# ════════════════════════
# Reading State Variables
# ════════════════════════

@view
@external
def get_owner() -> address:
    return self.owner  # ใช้ self. เสมอ

@view
@external
def get_balance(account: address) -> uint256:
    return self.balances[account]

@view  
@external
def get_user(account: address) -> UserInfo:
    return self.users[account]

# ════════════════════════
# Writing State Variables
# ════════════════════════

@external
def update_state():
    self.owner = msg.sender                    # Update value
    self.total_supply += 1000 * 10**18         # Increment
    self.balances[msg.sender] = 500            # Map update
    self.tokens[0] = 42                        # Array update
    
    # Struct update
    self.users[msg.sender] = UserInfo({
        balance: 1000,
        last_update: block.timestamp,
        is_active: True,
        referrer: empty(address)
    })
    
    # Update struct field เฉพาะ field
    self.users[msg.sender].balance += 500
    self.users[msg.sender].last_update = block.timestamp
```

### State Variable Patterns

```python
# @version 0.4.0

# ════════════════════════
# Pattern 1: Ownership
# ════════════════════════
owner: address
pending_owner: address  # For two-step ownership transfer

@external
def transfer_ownership(new_owner: address):
    assert msg.sender == self.owner, "Not owner"
    assert new_owner != empty(address), "Zero address"
    self.pending_owner = new_owner

@external
def accept_ownership():
    assert msg.sender == self.pending_owner, "Not pending owner"
    self.owner = self.pending_owner
    self.pending_owner = empty(address)

# ════════════════════════
# Pattern 2: Pausable
# ════════════════════════
paused: bool
pause_admin: address

@external
def pause():
    assert msg.sender == self.pause_admin, "Not admin"
    assert not self.paused, "Already paused"
    self.paused = True

@external
def unpause():
    assert msg.sender == self.pause_admin, "Not admin"
    assert self.paused, "Not paused"
    self.paused = False

@internal
def when_not_paused():
    assert not self.paused, "Contract is paused"

# ════════════════════════
# Pattern 3: Counters
# ════════════════════════
_token_id_counter: uint256
_nonce: HashMap[address, uint256]

@internal
def _next_token_id() -> uint256:
    self._token_id_counter += 1
    return self._token_id_counter

@internal
def _use_nonce(account: address) -> uint256:
    current: uint256 = self._nonce[account]
    self._nonce[account] += 1
    return current

# ════════════════════════
# Pattern 4: Timelocks
# ════════════════════════
TIMELOCK_DELAY: constant(uint256) = 48 * 3600  # 48 hours

locked_until: HashMap[bytes32, uint256]

@external
def queue_action(action_id: bytes32):
    assert msg.sender == self.owner, "Not owner"
    self.locked_until[action_id] = block.timestamp + TIMELOCK_DELAY

@external
def execute_action(action_id: bytes32):
    assert self.locked_until[action_id] > 0, "Not queued"
    assert block.timestamp >= self.locked_until[action_id], "Too early"
    
    # Execute action
    self.locked_until[action_id] = 0  # Clear
```

---

## 2. Local Variables {#local-variables}

Local Variables อยู่แค่ใน Function (ไม่ใช้ Gas สำหรับ Storage)

```python
# @version 0.4.0

total: uint256

@external
def local_variable_examples():
    # ════════════════════════
    # Declaration + Assignment
    # ════════════════════════
    x: uint256 = 100
    y: uint256 = 200
    z: uint256 = x + y  # 300
    
    # Re-assignment
    x = 500
    x += 100  # 600
    
    # ════════════════════════
    # Multiple Variables
    # ════════════════════════
    a: uint256 = 1
    b: uint256 = 2
    c: uint256 = 3
    
    # Swap (ไม่มี tuple swap ใน Vyper)
    temp: uint256 = a
    a = b
    b = temp
    
    # ════════════════════════
    # Struct as Local
    # ════════════════════════
    
    # Structs ต้องประกาศ Global level
    # แต่ใช้ได้ใน Function
    
    # ════════════════════════
    # Read State to Local (Gas Optimization)
    # ════════════════════════
    # ❌ ไม่ดี: อ่าน Storage หลายครั้ง
    # result1: uint256 = self.total + 1
    # result2: uint256 = self.total + 2
    # result3: uint256 = self.total + 3
    
    # ✅ ดี: อ่านครั้งเดียว
    cached_total: uint256 = self.total
    result1: uint256 = cached_total + 1
    result2: uint256 = cached_total + 2
    result3: uint256 = cached_total + 3

@view
@external
def complex_calculation(
    amount: uint256,
    rate: uint256,
    duration: uint256
) -> (uint256, uint256, uint256):
    """ตัวอย่าง Local Variables ใน Calculation"""
    
    # Intermediate calculations
    base_fee: uint256 = (amount * rate) / 10000
    time_factor: uint256 = duration / 86400  # Days
    time_bonus: uint256 = 0
    
    if time_factor >= 365:
        time_bonus = base_fee / 10  # 10% bonus
    elif time_factor >= 180:
        time_bonus = base_fee / 20  # 5% bonus
    elif time_factor >= 30:
        time_bonus = base_fee / 50  # 2% bonus
    
    total_fee: uint256 = base_fee + time_bonus
    net_amount: uint256 = amount - total_fee
    
    return base_fee, time_bonus, net_amount
```

---

## 3. Global Variables (Built-in) {#global-variables}

Vyper มี Built-in Variables ให้ใช้:

```python
# @version 0.4.0

@external
def builtin_variables_demo():
    
    # ════════════════════════
    # msg - Current Transaction
    # ════════════════════════
    sender: address = msg.sender    # Address ที่เรียก Function
    value: uint256 = msg.value      # ETH ที่ส่งมา (Wei)
    gas_left: uint256 = msg.gas     # Gas ที่เหลือ
    
    # ════════════════════════
    # block - Current Block
    # ════════════════════════
    block_num: uint256 = block.number       # Block Number ปัจจุบัน
    timestamp: uint256 = block.timestamp   # Unix Timestamp (seconds)
    block_hash: bytes32 = block.prevhash   # Hash ของ Block ก่อน
    coinbase: address = block.coinbase     # Validator Address
    difficulty: uint256 = block.difficulty # (PoW era, ใช้ไม่ได้ใน PoS)
    gas_limit: uint256 = block.gaslimit    # Gas Limit ของ Block
    base_fee: uint256 = block.basefee      # EIP-1559 Base Fee
    
    # ════════════════════════
    # tx - Transaction Info
    # ════════════════════════
    gas_price: uint256 = tx.gasprice       # Gas Price (Wei)
    origin: address = tx.origin            # Original Sender (ระวัง!)
    
    # ════════════════════════
    # chain_id
    # ════════════════════════
    chain: uint256 = chain.id              # Chain ID
    # 1 = Ethereum Mainnet
    # 5 = Goerli Testnet  
    # 11155111 = Sepolia Testnet
    # 137 = Polygon
    # 42161 = Arbitrum

@view
@external
def get_block_info() -> (uint256, uint256, bytes32):
    """ข้อมูล Block ปัจจุบัน"""
    return (
        block.number,
        block.timestamp,
        block.prevhash
    )

@view
@external
def is_mainnet() -> bool:
    """ตรวจสอบว่าอยู่บน Mainnet"""
    return chain.id == 1

@payable
@external
def check_msg():
    """ตรวจสอบ msg variables"""
    assert msg.value > 0, "Must send ETH"
    assert msg.sender != empty(address), "Invalid sender"
    
    # ⚠️ ระวัง: อย่าใช้ tx.origin สำหรับ Authentication!
    # tx.origin เป็น Original Caller ซึ่งอาจ Manipulate ได้
    # ใช้ msg.sender แทนเสมอ
```

### Security Note: msg.sender vs tx.origin

```python
# @version 0.4.0

# ════════════════════════
# ⚠️ SECURITY: tx.origin Vulnerability
# ════════════════════════

# สถานการณ์:
# User (tx.origin) → MaliciousContract → VulnerableContract

# ❌ อันตราย: ใช้ tx.origin
owner: address

@external
def vulnerable_function():
    # ถ้า User เรียก MaliciousContract
    # MaliciousContract เรียก Contract นี้
    # tx.origin = User (ผ่านได้!)
    # msg.sender = MaliciousContract
    assert tx.origin == self.owner, "Not owner"  # VULNERABLE!
    # Attacker สามารถ trick User ให้ทำ action นี้!

# ✅ ปลอดภัย: ใช้ msg.sender
@external
def safe_function():
    assert msg.sender == self.owner, "Not owner"  # Safe
    # msg.sender = Direct Caller เท่านั้น

# ════════════════════════
# Use Case ที่ OK สำหรับ tx.origin
# ════════════════════════
@view
@external
def was_called_directly() -> bool:
    """ตรวจสอบว่าถูกเรียกโดยตรงจาก EOA"""
    return msg.sender == tx.origin
```

---

## 4. Storage Slots {#storage-slots}

```python
# @version 0.4.0
# ────────────────────────────────────────
# Storage Layout ใน Vyper
# ────────────────────────────────────────
# แต่ละ State Variable ใช้ Storage Slots
# 1 Slot = 32 bytes (256 bits)
# Slot 0, 1, 2, ... (sequential)
# ────────────────────────────────────────

# Slot 0
var_a: uint256           # 1 slot (32 bytes)

# Slot 1  
var_b: uint256           # 1 slot

# Slot 2-3
struct MyStruct:
    field_a: uint256     # uses 1 slot
    field_b: uint256     # uses 1 slot

my_struct: MyStruct      # Slots 2-3

# Maps ใช้ keccak256 หา Slot
# keccak256(key . slot_number)
my_map: HashMap[address, uint256]  # Slot 4 (แต่ค่าอยู่ที่ keccak256(key.4))

# Arrays: contiguous storage
my_array: uint256[3]     # Slots 5, 6, 7

# ════════════════════════
# Layout Command
# ════════════════════════
# vyper -f layout contract.vy
# จะแสดง Storage Layout ของแต่ละตัวแปร

@external
def demonstrate_storage():
    # การอ่าน/เขียน Storage มีค่าใช้จ่าย Gas
    # SLOAD (read): 2100 gas (cold), 100 gas (warm)
    # SSTORE (write new): 20000 gas
    # SSTORE (update): 2900 gas
    
    # Cache Storage reads!
    cached: uint256 = self.var_a  # 1 SLOAD
    result: uint256 = cached * 2 + cached * 3  # No SLOAD
    self.var_b = result  # 1 SSTORE
```

### Packing Variables (Gas Optimization)

```python
# @version 0.4.0

# ════════════════════════
# ❌ ไม่ดี: ใช้หลาย Slots
# ════════════════════════
a1: uint256    # Slot 0 (32 bytes)
b1: uint8      # Slot 1 (32 bytes ทั้งๆ ที่ต้องการแค่ 1 byte!)
c1: uint8      # Slot 2 (32 bytes อีก!)

# ════════════════════════
# หมายเหตุ: Vyper ไม่มี Struct Packing แบบ Solidity
# แต่สามารถ Pack manually ได้
# ════════════════════════

# ✅ ทางเลือก: Manual Packing
packed_data: uint256  # เก็บหลายค่าใน 1 slot
# Bit layout: [8 bits: a][8 bits: b][240 bits: unused]

@external
def pack_two_uint8(a: uint8, b: uint8):
    """Pack สอง uint8 ลงใน uint256 single slot"""
    self.packed_data = (
        convert(a, uint256) |
        (convert(b, uint256) << 8)
    )

@view
@external
def unpack_first_uint8() -> uint8:
    """Unpack ค่าแรก"""
    return convert(self.packed_data & 0xFF, uint8)

@view
@external
def unpack_second_uint8() -> uint8:
    """Unpack ค่าที่สอง"""
    return convert((self.packed_data >> 8) & 0xFF, uint8)
```

---

## 5. Variable Visibility {#visibility}

```python
# @version 0.4.0

# ════════════════════════
# Vyper Variable Visibility
# ════════════════════════
# ใน Vyper ทุก State Variable เป็น Private โดย Default
# ต้องสร้าง Getter Function เองถ้าต้องการ Public access
# (ต่างจาก Solidity ที่มี public keyword)

owner: address            # Private: ไม่สร้าง auto-getter
balance: uint256          # Private: ต้องทำ getter เอง

# ════════════════════════
# สร้าง Getter Functions
# ════════════════════════

@view
@external
def get_owner() -> address:
    """Public getter for owner"""
    return self.owner

@view
@external
def get_balance(account: address) -> uint256:
    """Public getter for balance"""
    return self.balance

# ════════════════════════
# Internal Helper Functions
# ════════════════════════

@internal
def _validate_owner():
    """Internal: ใช้ได้แค่ใน Contract นี้"""
    assert msg.sender == self.owner, "Not owner"

@internal
def _update_balance(account: address, amount: uint256):
    """Internal: ใช้ร่วมกันหลาย Functions"""
    self.balance = amount

@external
def update_as_owner(new_balance: uint256):
    """Public: เรียก internal helper"""
    self._validate_owner()
    self._update_balance(msg.sender, new_balance)
```

---

## 6. Default Values {#defaults}

```python
# @version 0.4.0

# State Variables มี Default Values:
a: uint256           # Default: 0
b: int256            # Default: 0
c: bool              # Default: False
d: address           # Default: 0x0000...0000
e: bytes32           # Default: 0x0000...0000
f: String[50]        # Default: ""
g: Bytes[100]        # Default: b""

# Arrays: ทุก Element เป็น Default
h: uint256[5]        # Default: [0, 0, 0, 0, 0]
j: DynArray[uint256, 10]  # Default: [] (empty)

# Mappings: ทุก Key มี Default Value ของ Value Type
k: HashMap[address, uint256]  # k[any_address] = 0

@view
@external
def check_defaults() -> (uint256, bool, address, String[50]):
    return (
        self.a,   # 0
        self.c,   # False
        self.d,   # 0x000...
        self.f    # ""
    )

@view
@external
def is_initialized(account: address) -> bool:
    """ตรวจสอบว่า Account ถูก Initialize แล้วหรือยัง"""
    # HashMap Default คือ 0, ดังนั้นถ้า balance == 0 อาจยังไม่ได้ Initialize
    # ต้องมี explicit tracking
    return self.k[account] > 0

# ════════════════════════
# Pattern: Explicit Initialization Tracking
# ════════════════════════
is_registered: HashMap[address, bool]  # Default: False
user_data: HashMap[address, uint256]

@external
def register():
    assert not self.is_registered[msg.sender], "Already registered"
    self.is_registered[msg.sender] = True
    self.user_data[msg.sender] = block.timestamp

@view
@external
def is_user_registered(account: address) -> bool:
    return self.is_registered[account]
```

---

## 7. Memory vs Storage {#memory-storage}

```python
# @version 0.4.0

struct Position:
    token: address
    amount: uint256
    entry_price: uint256
    is_open: bool

# Storage: Persistent บน Blockchain
positions: HashMap[address, Position]
position_ids: DynArray[bytes32, 1000]

@external
def open_position(token: address, amount: uint256, price: uint256):
    """เปิด Position - Write to Storage"""
    
    # ════════════════════════
    # Writing to Storage
    # ════════════════════════
    
    # วิธีที่ 1: Direct assignment
    self.positions[msg.sender] = Position({
        token: token,
        amount: amount,
        entry_price: price,
        is_open: True
    })
    
    # วิธีที่ 2: Field-by-field (เหมาะเมื่อ update บางส่วน)
    self.positions[msg.sender].is_open = True
    self.positions[msg.sender].amount += amount

@view
@external
def get_position(account: address) -> Position:
    """อ่าน Position จาก Storage"""
    return self.positions[account]

@view
@external
def calculate_pnl(
    account: address,
    current_price: uint256
) -> int256:
    """
    คำนวณ PnL โดยใช้ Local Variable เพื่อ Cache
    
    Gas Optimization: อ่าน Storage ครั้งเดียว
    """
    # ════════════════════════
    # Cache Storage read
    # ════════════════════════
    pos: Position = self.positions[account]  # 1 SLOAD for struct
    
    if not pos.is_open:
        return 0
    
    # ใช้ pos (local) แทน self.positions[account] (storage)
    entry: uint256 = pos.entry_price   # จาก local cache
    amount: uint256 = pos.amount       # จาก local cache
    
    if current_price >= entry:
        profit: uint256 = (current_price - entry) * amount / entry
        return convert(profit, int256)
    else:
        loss: uint256 = (entry - current_price) * amount / entry
        return -convert(loss, int256)

@external
def batch_update_positions(
    accounts: DynArray[address, 100],
    new_prices: DynArray[uint256, 100]
):
    """
    Update หลาย Positions
    
    Pattern: Read → Modify in Memory → Write once
    """
    assert len(accounts) == len(new_prices), "Length mismatch"
    
    for i: uint256 in range(100):
        if i >= len(accounts):
            break
        
        account: address = accounts[i]
        
        # Read once (expensive)
        pos: Position = self.positions[account]
        
        if pos.is_open:
            # Modify "in memory" (local struct)
            pos.entry_price = new_prices[i]
            
            # Write once (expensive)
            self.positions[account] = pos
```

---

## 8. ตัวอย่างการใช้งานจริง {#real-world}

### Contract: AdvancedStorage

```python
# @version 0.4.0
# SPDX-License-Identifier: MIT

"""
AdvancedStorage - แสดง State Management Patterns ขั้นสูง
"""

# ════════════════════════
# STRUCTS
# ════════════════════════

struct Config:
    fee_rate: uint256        # Basis points (0-10000)
    min_deposit: uint256
    max_deposit: uint256
    is_active: bool
    treasury: address

struct UserAccount:
    balance: uint256
    locked_balance: uint256
    total_deposited: uint256
    total_withdrawn: uint256
    last_action_time: uint256
    nonce: uint256
    is_banned: bool
    tier: uint8              # 0=Basic, 1=Silver, 2=Gold, 3=Platinum

# ════════════════════════
# CONSTANTS
# ════════════════════════

# Tier thresholds
SILVER_THRESHOLD: constant(uint256) = 1000 * 10**18    # 1000 tokens
GOLD_THRESHOLD: constant(uint256) = 10000 * 10**18     # 10000 tokens
PLATINUM_THRESHOLD: constant(uint256) = 100000 * 10**18  # 100000 tokens

# Fee rates by tier (basis points)
BASIC_FEE: constant(uint256) = 100      # 1%
SILVER_FEE: constant(uint256) = 75      # 0.75%
GOLD_FEE: constant(uint256) = 50        # 0.5%
PLATINUM_FEE: constant(uint256) = 25    # 0.25%

# Timeouts
LOCK_DURATION: constant(uint256) = 7 * 24 * 3600  # 7 days

# ════════════════════════
# IMMUTABLES
# ════════════════════════

contract_owner: immutable(address)
deploy_time: immutable(uint256)

# ════════════════════════
# STATE VARIABLES
# ════════════════════════

# Configuration
config: Config

# User data
accounts: HashMap[address, UserAccount]

# Stats
total_users: uint256
total_volume: uint256
total_fees_collected: uint256

# Admin
is_admin: HashMap[address, bool]

# Whitelist
is_whitelisted: HashMap[address, bool]
whitelist_enabled: bool

# ════════════════════════
# EVENTS
# ════════════════════════

event Deposit:
    user: indexed(address)
    amount: uint256
    fee: uint256
    timestamp: uint256

event Withdrawal:
    user: indexed(address)
    amount: uint256
    fee: uint256
    timestamp: uint256

event TierUpgrade:
    user: indexed(address)
    old_tier: uint8
    new_tier: uint8

event ConfigUpdated:
    updated_by: indexed(address)
    parameter: String[50]

event UserBanned:
    user: indexed(address)
    banned_by: indexed(address)

# ════════════════════════
# CONSTRUCTOR
# ════════════════════════

@deploy
def __init__(
    fee_rate: uint256,
    min_deposit: uint256,
    max_deposit: uint256,
    treasury: address
):
    contract_owner = msg.sender
    deploy_time = block.timestamp
    
    self.config = Config({
        fee_rate: fee_rate,
        min_deposit: min_deposit,
        max_deposit: max_deposit,
        is_active: True,
        treasury: treasury
    })
    
    self.is_admin[msg.sender] = True
    self.whitelist_enabled = False

# ════════════════════════
# MODIFIERS (as internal functions)
# ════════════════════════

@internal
def _only_owner():
    assert msg.sender == contract_owner, "Only owner"

@internal
def _only_admin():
    assert self.is_admin[msg.sender], "Only admin"

@internal
def _when_active():
    assert self.config.is_active, "Contract paused"

@internal
def _not_banned(account: address):
    assert not self.accounts[account].is_banned, "User banned"

@internal
def _check_whitelist(account: address):
    if self.whitelist_enabled:
        assert self.is_whitelisted[account], "Not whitelisted"

# ════════════════════════
# PUBLIC FUNCTIONS
# ════════════════════════

@payable
@external
def deposit():
    """
    Deposit ETH
    """
    self._when_active()
    self._not_banned(msg.sender)
    self._check_whitelist(msg.sender)
    
    amount: uint256 = msg.value
    
    # Cache config
    cfg: Config = self.config
    
    assert amount >= cfg.min_deposit, "Below minimum"
    assert amount <= cfg.max_deposit, "Above maximum"
    
    # Get user account (cache)
    acc: UserAccount = self.accounts[msg.sender]
    
    # Calculate fee based on tier
    fee_rate: uint256 = self._get_fee_rate(acc.tier)
    fee: uint256 = (amount * fee_rate) / 10000
    net_amount: uint256 = amount - fee
    
    # Track new user
    if acc.total_deposited == 0:
        self.total_users += 1
    
    # Update user account
    acc.balance += net_amount
    acc.total_deposited += amount
    acc.last_action_time = block.timestamp
    acc.nonce += 1
    
    # Check tier upgrade
    old_tier: uint8 = acc.tier
    new_tier: uint8 = self._calculate_tier(acc.total_deposited)
    acc.tier = new_tier
    
    # Write back to storage
    self.accounts[msg.sender] = acc
    
    # Update global stats
    self.total_volume += amount
    self.total_fees_collected += fee
    
    # Send fee to treasury
    if fee > 0:
        send(cfg.treasury, fee)
    
    log Deposit(msg.sender, amount, fee, block.timestamp)
    
    if new_tier > old_tier:
        log TierUpgrade(msg.sender, old_tier, new_tier)

@external
def withdraw(amount: uint256):
    """
    Withdraw ETH
    """
    self._when_active()
    self._not_banned(msg.sender)
    
    # Cache user account
    acc: UserAccount = self.accounts[msg.sender]
    
    available: uint256 = acc.balance - acc.locked_balance
    assert available >= amount, "Insufficient balance"
    
    # Calculate fee
    fee_rate: uint256 = self._get_fee_rate(acc.tier)
    fee: uint256 = (amount * fee_rate) / 10000
    net_amount: uint256 = amount - fee
    
    # Update state BEFORE sending (ป้องกัน Reentrancy)
    acc.balance -= amount
    acc.total_withdrawn += amount
    acc.last_action_time = block.timestamp
    acc.nonce += 1
    
    self.accounts[msg.sender] = acc
    self.total_fees_collected += fee
    
    # Send fee
    cfg_treasury: address = self.config.treasury
    if fee > 0:
        send(cfg_treasury, fee)
    
    # Send to user
    send(msg.sender, net_amount)
    
    log Withdrawal(msg.sender, amount, fee, block.timestamp)

# ════════════════════════
# VIEW FUNCTIONS
# ════════════════════════

@view
@external
def get_account(user: address) -> UserAccount:
    return self.accounts[user]

@view
@external
def get_available_balance(user: address) -> uint256:
    acc: UserAccount = self.accounts[user]
    return acc.balance - acc.locked_balance

@view
@external
def get_stats() -> (uint256, uint256, uint256):
    """Total users, volume, fees"""
    return (
        self.total_users,
        self.total_volume,
        self.total_fees_collected
    )

@view
@external
def get_user_fee_rate(user: address) -> uint256:
    """Fee rate ของ user (basis points)"""
    return self._get_fee_rate(self.accounts[user].tier)

# ════════════════════════
# ADMIN FUNCTIONS
# ════════════════════════

@external
def update_fee_rate(new_rate: uint256):
    self._only_admin()
    assert new_rate <= 1000, "Fee too high (max 10%)"
    self.config.fee_rate = new_rate
    log ConfigUpdated(msg.sender, "fee_rate")

@external
def update_treasury(new_treasury: address):
    self._only_owner()
    assert new_treasury != empty(address), "Zero address"
    self.config.treasury = new_treasury
    log ConfigUpdated(msg.sender, "treasury")

@external
def set_admin(account: address, status: bool):
    self._only_owner()
    self.is_admin[account] = status

@external
def ban_user(user: address):
    self._only_admin()
    assert user != contract_owner, "Cannot ban owner"
    self.accounts[user].is_banned = True
    log UserBanned(user, msg.sender)

@external
def set_whitelist(account: address, status: bool):
    self._only_admin()
    self.is_whitelisted[account] = status

@external
def toggle_whitelist():
    self._only_admin()
    self.whitelist_enabled = not self.whitelist_enabled

@external
def pause():
    self._only_admin()
    self.config.is_active = False

@external
def unpause():
    self._only_admin()
    self.config.is_active = True

# ════════════════════════
# INTERNAL HELPERS
# ════════════════════════

@pure
@internal
def _get_fee_rate(tier: uint8) -> uint256:
    """Get fee rate based on tier"""
    if tier >= 3:
        return PLATINUM_FEE
    elif tier == 2:
        return GOLD_FEE
    elif tier == 1:
        return SILVER_FEE
    else:
        return BASIC_FEE

@pure
@internal
def _calculate_tier(total_deposited: uint256) -> uint8:
    """Calculate tier based on total deposits"""
    if total_deposited >= PLATINUM_THRESHOLD:
        return 3
    elif total_deposited >= GOLD_THRESHOLD:
        return 2
    elif total_deposited >= SILVER_THRESHOLD:
        return 1
    else:
        return 0
```

### Test AdvancedStorage

```python
# tests/test_advanced_storage.py
import boa
import pytest

MIN_DEPOSIT = 1 * 10**17   # 0.1 ETH
MAX_DEPOSIT = 10 * 10**18  # 10 ETH
FEE_RATE = 100             # 1%

@pytest.fixture
def treasury():
    return boa.env.generate_address("treasury")

@pytest.fixture
def contract(treasury):
    return boa.load(
        "contracts/AdvancedStorage.vy",
        FEE_RATE,
        MIN_DEPOSIT,
        MAX_DEPOSIT,
        treasury
    )

@pytest.fixture
def user():
    addr = boa.env.generate_address("user")
    boa.env.set_balance(addr, 100 * 10**18)  # 100 ETH
    return addr

@pytest.fixture
def user2():
    addr = boa.env.generate_address("user2")
    boa.env.set_balance(addr, 100 * 10**18)
    return addr

class TestDeposit:
    def test_basic_deposit(self, contract, user):
        deposit_amount = 1 * 10**18  # 1 ETH
        
        with boa.env.prank(user):
            contract.deposit(value=deposit_amount)
        
        acc = contract.get_account(user)
        fee = deposit_amount * FEE_RATE // 10000
        expected_balance = deposit_amount - fee
        
        assert acc.balance == expected_balance
        assert acc.total_deposited == deposit_amount
        assert acc.tier == 0  # Still basic

    def test_deposit_below_minimum_reverts(self, contract, user):
        with pytest.raises(Exception, match="Below minimum"):
            with boa.env.prank(user):
                contract.deposit(value=MIN_DEPOSIT - 1)

    def test_multiple_deposits_update_tier(self, contract, user):
        # Deposit enough for Silver tier (1000 ETH)
        # สำหรับ test ใช้ threshold ต่ำกว่า
        for _ in range(10):
            with boa.env.prank(user):
                contract.deposit(value=MAX_DEPOSIT)

class TestWithdraw:
    def test_basic_withdraw(self, contract, user):
        deposit = 5 * 10**18
        
        with boa.env.prank(user):
            contract.deposit(value=deposit)
        
        acc = contract.get_account(user)
        balance_before = acc.balance
        
        withdraw_amount = balance_before // 2
        
        with boa.env.prank(user):
            contract.withdraw(withdraw_amount)
        
        acc_after = contract.get_account(user)
        fee = withdraw_amount * FEE_RATE // 10000
        
        assert acc_after.balance == balance_before - withdraw_amount
        assert acc_after.total_withdrawn == withdraw_amount

    def test_withdraw_more_than_balance_reverts(self, contract, user):
        with boa.env.prank(user):
            contract.deposit(value=1 * 10**18)
        
        acc = contract.get_account(user)
        
        with pytest.raises(Exception, match="Insufficient balance"):
            with boa.env.prank(user):
                contract.withdraw(acc.balance + 1)

class TestStats:
    def test_track_total_users(self, contract, user, user2):
        with boa.env.prank(user):
            contract.deposit(value=1 * 10**18)
        
        total, _, _ = contract.get_stats()
        assert total == 1
        
        with boa.env.prank(user2):
            contract.deposit(value=1 * 10**18)
        
        total, _, _ = contract.get_stats()
        assert total == 2
```

---

## สรุป Part 005

ในส่วนนี้คุณได้เรียนรู้:
- ✅ State Variables: การประกาศ, อ่าน, เขียน
- ✅ Local Variables: การใช้ใน Functions
- ✅ Global Variables: msg, block, tx, chain
- ✅ Storage Slots: Layout ใน Blockchain
- ✅ Variable Visibility: Private by Default ใน Vyper
- ✅ Default Values ของแต่ละ Type
- ✅ Memory vs Storage patterns
- ✅ Design Patterns: Ownership, Pausable, Counters, Timelocks
- ✅ Contract ขั้นสูง: AdvancedStorage

## แบบฝึกหัด

1. **ปรับปรุง** AdvancedStorage ให้รองรับ ERC-20 Token deposit
2. **เพิ่ม** Referral System ให้ UserAccount struct
3. **สร้าง** Migration Function ที่ย้าย Data จาก UserAccount รูปแบบเก่าไปใหม่
4. **ทดสอบ** Gas Usage ของ Pattern ต่างๆ
5. **วิเคราะห์** Storage Layout ด้วย `vyper -f layout`

---

**ก่อนหน้า: [Part 004 - ชนิดข้อมูลพื้นฐาน](part_004_types.md)**  
**ต่อไป: [Part 006 - ฟังก์ชันและ Visibility](part_006_functions.md)**
