# Part 081: Storage Layout Optimization

## สารบัญ
1. Storage Layout ใน EVM
2. Slot Packing
3. Struct Layout
4. Mapping vs Array Gas Costs
5. Proxy Storage Collisions
6. EIP-1967 Storage Slots
7. การ Optimize Storage

---

## 1. Storage Layout ใน EVM

EVM ใช้ key-value storage ที่ mapping จาก 256-bit key ไปยัง 256-bit value แต่ละ key เรียกว่า "slot"

### หลักการจัดเรียง Storage

```
Contract Variables:
variable_0 → slot 0
variable_1 → slot 1
variable_2 → slot 2
...

Mapping: mapping[key] → keccak256(key, slot)
Array: arr[i] → keccak256(slot) + i
```

### Gas Costs

```
Cold SLOAD:  2100 gas
Warm SLOAD:  100 gas
SSTORE (new): 22100 gas (zero → non-zero)
SSTORE (update): 5000 gas (non-zero → non-zero)
SSTORE (clear): 5000 gas - 4800 refund (non-zero → zero)
```

---

## 2. Contract ที่แสดง Storage Layout

```vyper
# @version 0.4.0
# @title Storage Layout Demo
# @notice แสดงวิธีที่ Vyper จัดเรียง storage

# Slot 0: uint256
counter: public(uint256)

# Slot 1: address (20 bytes, แต่ใช้ slot เต็ม 32 bytes)
owner: public(address)

# Slot 2: bool (1 byte, แต่ใช้ slot เต็ม 32 bytes)
paused: public(bool)

# Slot 3: uint256
totalSupply: public(uint256)

# Mapping: balanceOf[addr] = keccak256(addr, 4)
balanceOf: public(HashMap[address, uint256])

# Mapping: allowance[owner][spender] = keccak256(spender, keccak256(owner, 5))
allowance: public(HashMap[address, HashMap[address, uint256]])

# Array: elements[i] = keccak256(6) + i
elements: public(DynArray[uint256, 100])

# Struct
struct UserData:
    balance: uint256
    lastUpdate: uint256
    isActive: bool

# userData[addr] = keccak256(addr, 7) สำหรับ struct ทั้งหมด
userData: public(HashMap[address, UserData])

@deploy
def __init__():
    self.owner = msg.sender

@view
@external
def getStorageLayout() -> String[256]:
    """
    อธิบาย storage layout
    """
    return "slot0: counter, slot1: owner, slot2: paused, slot3: totalSupply, slot4: balanceOf mapping, slot5: allowance mapping"
```

---

## 3. Slot Packing เพื่อลด Gas

ใน Solidity เราสามารถ pack variables เข้า slot เดียวกันได้ แต่ใน Vyper ไม่ได้ทำโดยอัตโนมัติ เราต้องใช้ Struct

```vyper
# @version 0.4.0
# @title Slot Packing Demo
# @notice แสดงการ pack ข้อมูลเข้า slot เดียว

# ===== ไม่ Optimized: 3 slots =====
balance_inefficient: uint256     # slot 0: 32 bytes (ใช้แค่ 8 bytes)
decimals_inefficient: uint8      # slot 1: 32 bytes (ใช้แค่ 1 byte!)
lastUpdate_inefficient: uint256  # slot 2: 32 bytes

# ===== Optimized: ใช้ Struct สำหรับ packing =====
# แต่ Vyper pack struct fields เต็ม slot แต่ละตัว
# ดังนั้น optimization จริงๆ คือลดจำนวน storage reads

struct PackedUserInfo:
    # Fields ถูกเข้าถึงพร้อมกัน 1 SLOAD แทน 3 SLOAD
    balance: uint256
    lastUpdate: uint256
    isActive: bool

packedUsers: HashMap[address, PackedUserInfo]

# ===== Manual Bit Packing สำหรับ Maximum Optimization =====
# เก็บข้อมูลหลายค่าใน uint256 เดียว

# Layout: | balance(128 bits) | timestamp(64 bits) | flags(64 bits) |
packedData: HashMap[address, uint256]

BALANCE_MASK: constant(uint256) = (1 << 128) - 1
TIMESTAMP_MASK: constant(uint256) = ((1 << 64) - 1) << 128
FLAGS_MASK: constant(uint256) = ((1 << 64) - 1) << 192

@deploy
def __init__():
    pass

@internal
@pure
def _packUserData(balance: uint128, timestamp: uint64, flags: uint64) -> uint256:
    """
    Pack 3 values เข้า 1 uint256
    """
    return (
        convert(balance, uint256) |
        (convert(timestamp, uint256) << 128) |
        (convert(flags, uint256) << 192)
    )

@internal
@pure
def _unpackBalance(packed: uint256) -> uint128:
    return convert(packed & BALANCE_MASK, uint128)

@internal
@pure
def _unpackTimestamp(packed: uint256) -> uint64:
    return convert((packed >> 128) & ((1 << 64) - 1), uint64)

@internal
@pure
def _unpackFlags(packed: uint256) -> uint64:
    return convert(packed >> 192, uint64)

@external
def setUserData(
    user: address,
    balance: uint128,
    timestamp: uint64,
    flags: uint64
):
    """
    บันทึกข้อมูล user ใน 1 storage slot (1 SSTORE)
    """
    self.packedData[user] = self._packUserData(balance, timestamp, flags)

@view
@external
def getUserBalance(user: address) -> uint128:
    """
    อ่าน balance จาก packed data (1 SLOAD)
    """
    return self._unpackBalance(self.packedData[user])

@view
@external
def getUserTimestamp(user: address) -> uint64:
    """
    อ่าน timestamp จาก packed data (1 SLOAD)
    """
    return self._unpackTimestamp(self.packedData[user])

@external
def updateBalance(user: address, newBalance: uint128):
    """
    อัพเดต balance โดยไม่เปลี่ยน fields อื่น
    Read-Modify-Write: 1 SLOAD + 1 SSTORE
    """
    packed: uint256 = self.packedData[user]
    timestamp: uint64 = self._unpackTimestamp(packed)
    flags: uint64 = self._unpackFlags(packed)
    
    self.packedData[user] = self._packUserData(newBalance, timestamp, flags)
```

---

## 4. Struct Layout Optimization

```vyper
# @version 0.4.0
# @title Struct Layout Optimization
# @notice เปรียบเทียบ struct layouts ต่างๆ

# ===== Version 1: Unoptimized =====
struct UserV1:
    # 3 separate storage reads จำเป็นเพื่อ read ทั้ง struct
    name: String[32]        # 1 slot (หรือมากกว่า)
    balance: uint256        # 1 slot
    lastLogin: uint256      # 1 slot
    level: uint8            # 1 slot (ใช้แค่ 1 byte!)
    isActive: bool          # 1 slot (ใช้แค่ 1 byte!)

# ===== Version 2: Better =====
struct UserV2:
    # Vyper เข้าถึง struct ทั้ง block ในการ SLOAD เดียว
    # เมื่อใช้ HashMap ทุก field อยู่ในตำแหน่งต่างกัน
    balance: uint256        # base slot + 0
    lastLogin: uint256      # base slot + 1
    isActive: bool          # base slot + 2
    level: uint8            # base slot + 3

# ===== Version 3: Maximum Packed =====
# ใช้ uint256 เดียวสำหรับข้อมูลที่เกี่ยวข้องกัน
struct UserV3:
    # layout ใน 1 slot: | balance(192) | lastLogin(40) | level(16) | isActive(8) |
    packedInfo: uint256
    name: String[32]  # ยังคงต้องแยก slot

usersV1: HashMap[address, UserV1]
usersV2: HashMap[address, UserV2]
usersV3: HashMap[address, UserV3]

@deploy
def __init__():
    pass

# Gas comparison:
# V1 write: ~5 SSTOREs
# V2 write: ~5 SSTOREs (Vyper doesn't auto-pack)
# V3 write: ~2 SSTOREs (packedInfo + name)

@external
def writeUserV1(user: address, balance: uint256, isActive: bool):
    # หลาย SSTORE operations
    self.usersV1[user].balance = balance        # SSTORE
    self.usersV1[user].isActive = isActive      # SSTORE
    self.usersV1[user].lastLogin = block.timestamp  # SSTORE

@external
def writeUserV3Packed(
    user: address,
    balance: uint192,
    level: uint16,
    isActive: bool
):
    # 1 SSTORE สำหรับ packed data
    packed: uint256 = (
        convert(balance, uint256) |
        (convert(level, uint256) << 192) |
        (convert(isActive, uint256) << 208)
    )
    self.usersV3[user].packedInfo = packed  # 1 SSTORE
```

---

## 5. Mapping vs Array Gas Costs

```vyper
# @version 0.4.0
# @title Mapping vs Array Comparison
# @notice เปรียบเทียบ gas costs ของ mapping และ array

# ===== Approach 1: DynArray =====
# Random access: O(1) แต่ต้องรู้ index
# Iteration: O(n) - iterate ทุก elements
# Insert: O(1) ที่ท้าย
# Delete: O(n) ถ้าต้องการรักษา order

participantsList: DynArray[address, 1000]
participantIndex: HashMap[address, uint256]  # address -> index in array
isParticipant: HashMap[address, bool]

# ===== Approach 2: EnumerableSet pattern =====
# ใช้ทั้ง mapping และ array ร่วมกัน

struct EnumerableAddressSet:
    # Array สำหรับ iteration
    values: DynArray[address, 10000]
    # Mapping สำหรับ O(1) membership check
    indexes: HashMap[address, uint256]  # 1-indexed

holders: EnumerableAddressSet

@deploy
def __init__():
    pass

@internal
def _addToSet(value: address) -> bool:
    """เพิ่ม address เข้า set (no duplicates)"""
    if self.holders.indexes[value] != 0:
        return False  # Already exists
    
    self.holders.values.append(value)
    self.holders.indexes[value] = len(self.holders.values)  # 1-indexed
    
    return True

@internal
def _removeFromSet(value: address) -> bool:
    """ลบ address จาก set"""
    valueIndex: uint256 = self.holders.indexes[value]
    if valueIndex == 0:
        return False  # Doesn't exist
    
    # Swap with last element
    lastIndex: uint256 = len(self.holders.values)
    
    if valueIndex != lastIndex:
        lastValue: address = self.holders.values[lastIndex - 1]
        self.holders.values[valueIndex - 1] = lastValue
        self.holders.indexes[lastValue] = valueIndex
    
    # Remove last element
    self.holders.values.pop()
    self.holders.indexes[value] = 0
    
    return True

@internal
@view
def _setContains(value: address) -> bool:
    return self.holders.indexes[value] != 0

@external
def addHolder(holder: address):
    self._addToSet(holder)

@external
def removeHolder(holder: address):
    self._removeFromSet(holder)

@view
@external
def isHolder(addr: address) -> bool:
    return self._setContains(addr)

@view
@external
def getHolders() -> DynArray[address, 10000]:
    return self.holders.values

@view
@external
def holderCount() -> uint256:
    return len(self.holders.values)
```

---

## 6. Proxy Storage Collision Problem

```vyper
# @version 0.4.0
# @title Proxy Storage Collision Demo
# @notice แสดงปัญหา storage collision และวิธีแก้

# ===== ปัญหา: Storage Collision =====
# 
# Proxy Contract:          Implementation Contract:
# slot 0: _admin           slot 0: owner
# slot 1: _implementation  slot 1: balance
# slot 2: ...              slot 2: ...
#
# เมื่อ delegatecall: implementation เขียน "owner" ไปที่ slot 0
# แต่ proxy เก็บ "_admin" ไว้ที่ slot 0 → COLLISION!

# ===== วิธีแก้: EIP-1967 =====
# ใช้ random slots จาก keccak256 ที่ไม่น่าจะชนกับ linear layout

# Implementation slot: keccak256("eip1967.proxy.implementation") - 1
IMPLEMENTATION_SLOT: constant(bytes32) = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc

# Admin slot: keccak256("eip1967.proxy.admin") - 1  
ADMIN_SLOT: constant(bytes32) = 0xb53127684a568b3173ae13b9f8a6016e243e63b6e8ee1178d6a717850b5d6103

# Beacon slot: keccak256("eip1967.proxy.beacon") - 1
BEACON_SLOT: constant(bytes32) = 0xa3f0ad74e5423aebfd80d3ef4346578335a9a72aeaee59ff6cb3582b35133d50

@deploy
def __init__():
    pass

# หมายเหตุ: Vyper ไม่รองรับ arbitrary storage slot access โดยตรง
# ต้องใช้ inline assembly ผ่าน raw_call หรือ special built-ins
# ดังนั้น proxy pattern มักเขียนใน Solidity แต่ implementation ใน Vyper
```

---

## 7. Implementation Contract สำหรับ Proxy

```vyper
# @version 0.4.0
# @title Token Implementation (สำหรับ Proxy)
# @notice Implementation ที่ proxy จะ delegatecall หา
# @dev Storage layout ต้องไม่ชนกับ proxy

# ===== Storage Layout =====
# slot 0: _initialized (bool) - guard สำหรับ initialize
# slot 1: owner (address)
# slot 2: name (String[64])
# ...
# (proxy ใช้ EIP-1967 slots ที่ > 2^255 ดังนั้นไม่ชน)

_initialized: bool
_initializing: bool

owner: public(address)
name: public(String[64])
symbol: public(String[32])
decimals: public(uint8)
totalSupply: public(uint256)
balanceOf: public(HashMap[address, uint256])
allowance: public(HashMap[address, HashMap[address, uint256]])

event Transfer:
    sender: indexed(address)
    receiver: indexed(address)
    value: uint256

event Initialized:
    version: uint8

@deploy
def __init__():
    # Lock implementation contract
    self._initialized = True

@external
def initialize(
    _owner: address,
    _name: String[64],
    _symbol: String[32],
    _initialSupply: uint256
):
    """
    แทน constructor สำหรับ upgradeable contracts
    เรียกได้ครั้งเดียวเท่านั้น
    """
    assert not self._initialized, "Already initialized"
    assert not self._initializing, "Initializing"
    
    self._initializing = True
    
    self.owner = _owner
    self.name = _name
    self.symbol = _symbol
    self.decimals = 18
    self.totalSupply = _initialSupply
    self.balanceOf[_owner] = _initialSupply
    
    self._initialized = True
    self._initializing = False
    
    log Transfer(empty(address), _owner, _initialSupply)
    log Initialized(1)

@external
def transfer(to: address, amount: uint256) -> bool:
    assert self.balanceOf[msg.sender] >= amount, "Insufficient balance"
    
    self.balanceOf[msg.sender] -= amount
    self.balanceOf[to] += amount
    
    log Transfer(msg.sender, to, amount)
    return True
```

---

## 8. Storage Optimization Patterns

```vyper
# @version 0.4.0
# @title Storage Optimization Patterns
# @notice รวม patterns สำหรับ optimize storage usage

# ===== Pattern 1: Lazy Initialization =====
# ไม่เขียน default values (ค่า default = 0 อยู่แล้ว)

userCreated: HashMap[address, bool]

@external
def createUser():
    # ไม่ต้องทำ: self.userCreated[msg.sender] = False
    # เพียงแค่ใช้ค่า default (false)
    
    # ทำเมื่อจำเป็น:
    self.userCreated[msg.sender] = True  # 1 SSTORE

# ===== Pattern 2: Dirty Slots =====
# ลดการ write ด้วยการรวม writes เข้าด้วยกัน

pendingBalance: HashMap[address, uint256]
pendingCount: HashMap[address, uint256]

@external
def addPending(user: address, amount: uint256):
    # ไม่ดี: 2 separate writes
    # self.pendingBalance[user] += amount  # SSTORE
    # self.pendingCount[user] += 1          # SSTORE
    
    # ดีกว่า: อ่านครั้งเดียว update พร้อมกัน
    # แต่ใน Vyper ต้องทำแยก slots
    self.pendingBalance[user] += amount
    self.pendingCount[user] += 1

# ===== Pattern 3: Nonce Pattern =====
# ใช้ nonce แทน boolean mapping สำหรับ "used" tracking

nonces: HashMap[bytes32, uint256]

@external
def executeOrder(orderId: bytes32, amount: uint256):
    assert self.nonces[orderId] == 0, "Order already executed"
    
    self.nonces[orderId] = 1  # mark as used
    
    # execute order...

# ===== Pattern 4: Checkpoint Pattern =====
# ใช้ checkpoints แทน full history

struct Checkpoint:
    fromBlock: uint32
    value: uint256

checkpoints: HashMap[address, DynArray[Checkpoint, 1000]]
numCheckpoints: HashMap[address, uint256]

@internal
def _writeCheckpoint(delegatee: address, newValue: uint256):
    nCheckpoints: uint256 = self.numCheckpoints[delegatee]
    
    if nCheckpoints > 0:
        lastCheckpoint: Checkpoint = self.checkpoints[delegatee][nCheckpoints - 1]
        if lastCheckpoint.fromBlock == convert(block.number, uint32):
            # Update ใน block เดียวกัน - ไม่เพิ่ม checkpoint ใหม่
            self.checkpoints[delegatee][nCheckpoints - 1].value = newValue
            return
    
    self.checkpoints[delegatee].append(Checkpoint({
        fromBlock: convert(block.number, uint32),
        value: newValue
    }))
    self.numCheckpoints[delegatee] += 1

@view
@internal
def _getPriorValue(account: address, blockNumber: uint256) -> uint256:
    """Binary search หา value ที่ block number"""
    if self.numCheckpoints[account] == 0:
        return 0
    
    # Most recent: O(1)
    lastCheckpoint: Checkpoint = self.checkpoints[account][self.numCheckpoints[account] - 1]
    if convert(lastCheckpoint.fromBlock, uint256) <= blockNumber:
        return lastCheckpoint.value
    
    # First checkpoint: O(1)
    firstCheckpoint: Checkpoint = self.checkpoints[account][0]
    if convert(firstCheckpoint.fromBlock, uint256) > blockNumber:
        return 0
    
    # Binary search: O(log n)
    lower: uint256 = 0
    upper: uint256 = self.numCheckpoints[account] - 1
    
    for _: uint256 in range(1000):  # max iterations
        if lower >= upper:
            break
        
        center: uint256 = upper - (upper - lower) / 2
        cp: Checkpoint = self.checkpoints[account][center]
        
        if convert(cp.fromBlock, uint256) == blockNumber:
            return cp.value
        elif convert(cp.fromBlock, uint256) < blockNumber:
            lower = center
        else:
            upper = center - 1
    
    return self.checkpoints[account][lower].value

@deploy
def __init__():
    pass
```

---

## 9. Storage Layout Verification

```python
# scripts/verify_storage_layout.py
# ตรวจสอบ storage layout ของ contracts

import json
from web3 import Web3

def compute_slot(variable_index: int) -> int:
    """คำนวณ slot สำหรับ state variable ที่ index"""
    return variable_index

def compute_mapping_slot(key: str, mapping_slot: int) -> str:
    """คำนวณ slot สำหรับ mapping key"""
    # slot = keccak256(key . mapping_slot)
    key_padded = key.rjust(64, '0')
    slot_padded = hex(mapping_slot)[2:].rjust(64, '0')
    
    combined = bytes.fromhex(key_padded + slot_padded)
    return Web3.keccak(combined).hex()

def compute_array_slot(base_slot: int, index: int) -> str:
    """คำนวณ slot สำหรับ dynamic array element"""
    # First slot = keccak256(base_slot)
    # element[i] = keccak256(base_slot) + i
    base = bytes.fromhex(hex(base_slot)[2:].rjust(64, '0'))
    first_slot = int(Web3.keccak(base).hex(), 16)
    return hex(first_slot + index)

def read_storage_slot(w3, contract_address: str, slot: str) -> str:
    """อ่านค่าจาก storage slot โดยตรง"""
    return w3.eth.get_storage_at(
        contract_address,
        int(slot, 16)
    ).hex()

# ตัวอย่างการใช้งาน
print("Storage Layout Analysis:")
print(f"Slot 0 (counter): {compute_slot(0)}")
print(f"Slot 1 (owner): {compute_slot(1)}")

user_addr = "0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266"
slot_4 = 4  # balanceOf mapping
balance_slot = compute_mapping_slot(
    user_addr[2:].lower().rjust(64, '0'),  # address padded to 32 bytes
    slot_4
)
print(f"balanceOf[{user_addr}] at slot: {balance_slot}")

# อ่านจาก live node
# w3 = Web3(Web3.HTTPProvider("http://localhost:8545"))
# value = read_storage_slot(w3, contract_address, balance_slot)
# print(f"Value: {value}")
```

---

## 10. Gas Cost Analysis

```vyper
# @version 0.4.0
# @title Gas Cost Analysis
# @notice เปรียบเทียบ gas costs ของ patterns ต่างๆ

# Expensive: 3 separate storage reads
name_expensive: String[64]
balance_expensive: uint256
active_expensive: bool

# Cheaper: Struct (ยังคงแยก slots แต่ logic cleaner)
struct Info:
    balance: uint256
    lastUpdate: uint256
    active: bool

userInfo: HashMap[address, Info]

# Cheapest for related data: Pack into single uint256
# | balance (128 bits) | lastUpdate (64 bits) | active (1 bit) | flags (63 bits) |
packedInfo: HashMap[address, uint256]

@deploy
def __init__():
    pass

@external
def updateExpensive(balance: uint256, active: bool):
    """
    2 SSTOREs (balance + active)
    Gas: ~10000 (warm) or ~44200 (cold)
    """
    self.balance_expensive = balance
    self.active_expensive = active

@external
def updateStruct(balance: uint256, active: bool):
    """
    2 SSTOREs (balance + active ยังคงแยก slots)
    Gas: ~10000 (warm) or ~44200 (cold)
    """
    self.userInfo[msg.sender].balance = balance
    self.userInfo[msg.sender].active = active

@external
def updatePacked(balance: uint128, active: bool):
    """
    1 SSTORE (ทั้ง balance และ active ใน slot เดียว!)
    Gas: ~5000 (warm) or ~22100 (cold) - ประหยัดครึ่งหนึ่ง!
    """
    packed: uint256 = convert(balance, uint256)
    if active:
        packed |= (1 << 128)
    
    self.packedInfo[msg.sender] = packed  # 1 SSTORE

@view
@external
def readPacked() -> (uint128, bool):
    """
    1 SLOAD สำหรับทั้ง 2 values
    Gas: ~100 (warm) or ~2100 (cold)
    """
    packed: uint256 = self.packedInfo[msg.sender]
    balance: uint128 = convert(packed & ((1 << 128) - 1), uint128)
    active: bool = (packed >> 128) & 1 == 1
    return balance, active

# ===== Immutable Variables (ไม่ใช้ Storage) =====
# immutable ใน Vyper เก็บใน bytecode ไม่ใช่ storage
# ประหยัดทั้ง deploy gas และ read gas

WETH: immutable(address)
MAX_SUPPLY: immutable(uint256)

@deploy
def __init__(_weth: address, _maxSupply: uint256):
    WETH = _weth
    MAX_SUPPLY = _maxSupply

@view
@external
def getWETH() -> address:
    # ไม่ต้องทำ SLOAD! อ่านจาก bytecode โดยตรง
    return WETH
```

---

## 11. Storage Patterns สำหรับ Gas Refunds

```vyper
# @version 0.4.0
# @title Storage Refund Patterns
# @notice ใช้ gas refunds ให้เต็มประสิทธิภาพ

# ===== SSTORE Refund Rules (EIP-3529) =====
# - Clearing slot (non-zero → zero): refund 4800 gas
# - Max refund: 20% ของ gas used in transaction

# Pattern: Transient storage simulation
# เก็บ data ชั่วคราว แล้ว clear ท้าย transaction

tempData: HashMap[address, uint256]
tempActive: bool

@external
def executeWithTempData(user: address, data: uint256):
    """
    ใช้ temporary storage แล้ว clear เพื่อรับ refund
    """
    # Set temporary data
    self.tempData[user] = data
    self.tempActive = True
    
    # Use data...
    result: uint256 = self.tempData[user] * 2
    
    # Clear for refund (4800 gas refund per slot cleared)
    self.tempData[user] = 0
    self.tempActive = False
    
    # Net cost: SSTORE(non-zero) + SSTORE(zero) - refund
    # = 22100 + 5000 - 4800 = 22300 per slot
    # vs keeping: 22100 + 5000 = 27100

# EIP-1153: Transient Storage (future optimization)
# TSTORE / TLOAD: temporary storage ที่ clear ท้าย transaction automatically
# ยังไม่ available ใน Vyper 0.4.0 แต่กำลังจะมาใน future versions
```

---

## 12. Storage Layout สำหรับ Diamond Pattern

```vyper
# @version 0.4.0
# @title Diamond Storage
# @notice Storage layout สำหรับ EIP-2535 Diamond Pattern

# Diamond Pattern ใช้ namespaced storage เพื่อหลีกเลี่ยง collision
# แต่ละ facet ใช้ storage ที่แตกต่างกัน

# ===== Namespace Storage Pattern =====
# base slot = keccak256("diamond.storage.ERC20") 
# ใช้ค่า constant ที่คำนวณไว้ล่วงหน้า

# keccak256("diamond.storage.ERC20") =
ERC20_STORAGE_SLOT: constant(bytes32) = 0x52c63247e1f47db19d5ce0460030c497f067ca4cebf71ba98eeadabe20bace00

# keccak256("diamond.storage.Access") =
ACCESS_STORAGE_SLOT: constant(bytes32) = 0x6e85bea13e6eb6a5d2516bb1acece88b8756b6a4a6c2ad0c8c97d9de88a0dd02

# ใน implementation facet จะอ้างอิง slot เหล่านี้ผ่าน delegatecall

@deploy
def __init__():
    pass
```

---

## 13. สรุป Storage Optimization

### Best Practices

```
1. Pack related variables ด้วย Manual Bit Packing
   - ประหยัดได้ถึง 50% สำหรับ variables ขนาดเล็ก
   
2. ใช้ Immutable Variables สำหรับค่าคงที่
   - ไม่ใช้ storage เลย!
   
3. ใช้ Events แทน Storage สำหรับ Historical Data
   - Events: ~8 gas/byte
   - Storage: ~22100 gas/32 bytes (new)
   
4. Lazy Initialization
   - อย่า initialize variables ที่มีค่า default = 0
   
5. Clear Unused Storage เพื่อ Gas Refund
   - แต่ระวัง: EIP-3529 จำกัด refund ที่ 20%
   
6. ใช้ Struct สำหรับ Grouped Data
   - ทำให้ code clean แม้ gas savings จำกัด
   
7. EnumerableSet Pattern สำหรับ Iterable Mappings
   - O(1) membership check + O(n) iteration
   
8. Checkpoint Pattern สำหรับ Historical Balances
   - O(log n) lookup แทน O(n)
```

### Storage Slot Cheat Sheet

```
Regular variable[i]  → slot i
Mapping[key]         → keccak256(key, slot)
Array[i]             → keccak256(slot) + i
Struct.field         → slot + field_offset
Nested mapping       → keccak256(key2, keccak256(key1, slot))
EIP-1967 impl        → 0x360894...bbc
EIP-1967 admin       → 0xb53127...103
```

---

## แบบฝึกหัด

1. วิเคราะห์ storage layout ของ Uniswap V2 Pair contract
2. Implement manual bit packing สำหรับ user data ที่มี 4 fields
3. เปรียบเทียบ gas cost ของ mapping vs array สำหรับ 100 operations
4. ออกแบบ storage layout สำหรับ lending protocol ที่ minimize gas
5. Implement EnumerableSet ที่รองรับ uint256 แทน address

---

*จบ Part 081: Storage Layout Optimization*
