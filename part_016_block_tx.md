# Part 016: Block และ Transaction Variables

## สารบัญ
1. [Block Variables](#block-variables)
2. [Transaction Variables](#transaction-variables)
3. [Chain Variables](#chain-variables)
4. [Time-based Logic](#time-based-logic)
5. [ตัวอย่าง: Time-locked Vault Contract](#time-locked-vault)
6. [แบบฝึกหัด](#exercises)

---

## 1. Block Variables {#block-variables}

Vyper มี Built-in Variables สำหรับเข้าถึงข้อมูลของ Block ปัจจุบันในระหว่างการ Execute Transaction

### 1.1 block.number

`block.number` คือหมายเลข Block ปัจจุบัน (นับจาก 0 จาก Genesis Block)

```vyper
# @version 0.4.0

blockNumber: public(uint256)

@deploy
def __init__():
    self.blockNumber = block.number

@external
def getCurrentBlock() -> uint256:
    return block.number

@external
def getBlocksSince(pastBlock: uint256) -> uint256:
    assert block.number >= pastBlock, "Invalid past block"
    return block.number - pastBlock
```

**ข้อควรรู้:**
- Ethereum สร้าง Block ใหม่ทุก ~12 วินาที (หลัง The Merge)
- `block.number` เพิ่มขึ้นทีละ 1 ทุก Block
- ใช้เพื่อ Timing แทน Timestamp ได้ในบางกรณี

### 1.2 block.timestamp

`block.timestamp` คือ Unix Timestamp (วินาทีนับจาก 1 Jan 1970) ของ Block ปัจจุบัน

```vyper
# @version 0.4.0

deployTime: public(uint256)
SECONDS_PER_DAY: constant(uint256) = 86400

@deploy
def __init__():
    self.deployTime = block.timestamp

@external
@view
def getDaysSinceDeploy() -> uint256:
    elapsed: uint256 = block.timestamp - self.deployTime
    return elapsed / SECONDS_PER_DAY

@external
@view
def isExpired(expiryTime: uint256) -> bool:
    return block.timestamp >= expiryTime
```

**ข้อควรรู้:**
- Validator สามารถ Manipulate Timestamp ได้เล็กน้อย (~15 วินาที)
- อย่าใช้เพื่อ Random Number Generation
- เหมาะสำหรับ Time-lock ที่ช่วงเวลายาว (ชั่วโมง/วัน)

### 1.3 block.prevhash

`block.prevhash` คือ Hash ของ Block ก่อนหน้า (bytes32)

```vyper
# @version 0.4.0

@external
@view
def getPrevBlockHash() -> bytes32:
    return block.prevhash

@external
@view
def isBlockHashValid(expectedHash: bytes32) -> bool:
    # ใช้เพื่อ Verify chain integrity
    return block.prevhash == expectedHash
```

**คำเตือน:** อย่าใช้ `block.prevhash` เพื่อสร้าง Randomness เพราะ Validator รู้ค่านี้ล่วงหน้า

### 1.4 block.coinbase

`block.coinbase` คือ Address ของ Validator ที่สร้าง Block นี้ (ผู้รับ Block Reward)

```vyper
# @version 0.4.0

validatorRewards: HashMap[address, uint256]

@external
@payable
def rewardValidator():
    # ส่ง ETH ให้ Validator ที่ Include Transaction นี้
    validator: address = block.coinbase
    self.validatorRewards[validator] += msg.value

@external
@view
def getCurrentValidator() -> address:
    return block.coinbase
```

### 1.5 block.gaslimit

`block.gaslimit` คือ Gas Limit สูงสุดของ Block ปัจจุบัน

```vyper
# @version 0.4.0

@external
@view
def getBlockGasLimit() -> uint256:
    return block.gaslimit

@external
@view
def isGasLimitSufficient(requiredGas: uint256) -> bool:
    return block.gaslimit >= requiredGas
```

### 1.6 block.basefee

`block.basefee` คือ Base Fee ต่อ Gas Unit ของ Block นี้ (EIP-1559)

```vyper
# @version 0.4.0

@external
@view
def getBaseFee() -> uint256:
    return block.basefee

@external
@view
def estimateTxCost(gasUnits: uint256) -> uint256:
    # ประมาณค่าใช้จ่ายขั้นต่ำ (ไม่รวม Priority Fee)
    return block.basefee * gasUnits
```

**EIP-1559 Model:**
- `Base Fee`: ถูก Burn (ไม่ให้ Validator)
- `Priority Fee (Tip)`: ให้ Validator
- `Max Fee`: ราคาสูงสุดที่ผู้ส่งยอมจ่าย

---

## 2. Transaction Variables {#transaction-variables}

### 2.1 msg.sender

`msg.sender` คือ Address ที่เรียก Function นี้ (อาจเป็น EOA หรือ Contract)

```vyper
# @version 0.4.0

owner: public(address)
balances: public(HashMap[address, uint256])

@deploy
def __init__():
    self.owner = msg.sender

@external
@payable
def deposit():
    self.balances[msg.sender] += msg.value
    
@external
def withdraw(amount: uint256):
    assert self.balances[msg.sender] >= amount, "Insufficient balance"
    self.balances[msg.sender] -= amount
    send(msg.sender, amount)

@external
def onlyOwnerFunction():
    assert msg.sender == self.owner, "Not owner"
    # ทำงานที่ Owner เท่านั้นทำได้
```

**msg.sender ใน Context ต่างๆ:**
```
EOA ------call-----> ContractA.foo()
                     msg.sender = EOA
                     
EOA ------call-----> ContractA.foo()
                          |
                          call----> ContractB.bar()
                                    msg.sender = ContractA (!)
```

### 2.2 msg.value

`msg.value` คือจำนวน Wei ที่ส่งมาพร้อมกับ Transaction

```vyper
# @version 0.4.0

MIN_DEPOSIT: constant(uint256) = 1000000000000000  # 0.001 ETH in Wei
MAX_DEPOSIT: constant(uint256) = 10000000000000000000  # 10 ETH in Wei

totalDeposited: public(uint256)

@external
@payable
def deposit():
    assert msg.value >= MIN_DEPOSIT, "Deposit too small"
    assert msg.value <= MAX_DEPOSIT, "Deposit too large"
    self.totalDeposited += msg.value
    
@external
@view
def getContractBalance() -> uint256:
    return self.balance

@external
@payable
def buyTokens() -> uint256:
    # 1 ETH = 1000 Token
    PRICE_PER_TOKEN: uint256 = 1000000000000000  # 0.001 ETH per token
    tokens: uint256 = msg.value / PRICE_PER_TOKEN
    assert tokens > 0, "Not enough ETH sent"
    return tokens
```

### 2.3 msg.gas

`msg.gas` คือ Gas ที่เหลืออยู่ใน Transaction ปัจจุบัน

```vyper
# @version 0.4.0

MIN_GAS_REQUIRED: constant(uint256) = 50000

@external
def expensiveOperation():
    # ตรวจสอบว่ามี Gas เพียงพอก่อน Execute
    assert msg.gas >= MIN_GAS_REQUIRED, "Insufficient gas"
    
    # ทำงานที่ต้องใช้ Gas มาก
    for i: uint256 in range(100):
        pass  # แต่ละ iteration ใช้ Gas

@external
@view
def getRemainingGas() -> uint256:
    return msg.gas
```

### 2.4 tx.gasprice

`tx.gasprice` คือ Gas Price ที่ผู้ส่ง Transaction ตั้งไว้ (Wei per Gas)

```vyper
# @version 0.4.0

MAX_GAS_PRICE: constant(uint256) = 100000000000  # 100 Gwei

@external
def protectedOperation():
    # ป้องกัน Front-running ด้วยการจำกัด Gas Price
    assert tx.gasprice <= MAX_GAS_PRICE, "Gas price too high"
    # ทำงาน...

@external
@view
def getCurrentGasPrice() -> uint256:
    return tx.gasprice

@external
@view  
def estimateTotalGasCost(gasUnits: uint256) -> uint256:
    return tx.gasprice * gasUnits
```

### 2.5 tx.origin

`tx.origin` คือ Address ของ EOA ที่เริ่ม Transaction (ต่างจาก msg.sender)

```vyper
# @version 0.4.0

@external
def example():
    # tx.origin = EOA ที่เริ่ม Transaction เสมอ
    # msg.sender = ผู้เรียก Function โดยตรง (อาจเป็น Contract)
    
    txStarter: address = tx.origin
    directCaller: address = msg.sender
    
    # ทั้งสองอาจเท่ากัน (ถ้า EOA เรียก Contract โดยตรง)
    # หรือต่างกัน (ถ้า Contract เรียก Contract)
    
@external
@view
def isSenderOrigin() -> bool:
    # True ถ้า msg.sender คือ EOA (ไม่ผ่าน Contract)
    return msg.sender == tx.origin
```

**คำเตือน:** อย่าใช้ `tx.origin` เพื่อ Authentication เพราะเสี่ยงต่อ Phishing Attack!

```vyper
# ❌ ไม่ปลอดภัย - ใช้ tx.origin เพื่อ Authentication
@external
def unsafeFunction():
    assert tx.origin == self.owner, "Not owner"  # อันตราย!

# ✅ ปลอดภัย - ใช้ msg.sender
@external  
def safeFunction():
    assert msg.sender == self.owner, "Not owner"  # ถูกต้อง
```

---

## 3. Chain Variables {#chain-variables}

### 3.1 chain.id

`chain.id` คือ ID ของ Blockchain ที่ Contract Deploy อยู่

```vyper
# @version 0.4.0

# Chain IDs ที่สำคัญ
# 1 = Ethereum Mainnet
# 11155111 = Sepolia Testnet
# 137 = Polygon Mainnet
# 56 = BNB Smart Chain

EXPECTED_CHAIN_ID: immutable(uint256)

@deploy
def __init__(chainId: uint256):
    EXPECTED_CHAIN_ID = chainId

@external
@view
def getChainId() -> uint256:
    return chain.id

@external
@view  
def isCorrectChain() -> bool:
    return chain.id == EXPECTED_CHAIN_ID
```

**ประโยชน์ของ chain.id:**
- ป้องกัน Replay Attack ข้ามเครือข่าย
- ใช้ใน EIP-712 (Structured Data Signing)
- ตรวจสอบว่า Contract ทำงานบน Chain ที่ถูกต้อง

---

## 4. Time-based Logic {#time-based-logic}

### 4.1 Time Constants

```vyper
# @version 0.4.0

# Time Constants (ใน Vyper ใช้ uint256 วินาที)
MINUTE: constant(uint256) = 60
HOUR: constant(uint256) = 3600
DAY: constant(uint256) = 86400
WEEK: constant(uint256) = 604800
MONTH: constant(uint256) = 2592000   # 30 days
YEAR: constant(uint256) = 31536000   # 365 days
```

### 4.2 Deadline Pattern

```vyper
# @version 0.4.0

DURATION: constant(uint256) = 86400 * 7  # 7 วัน

deadline: public(uint256)
isActive: public(bool)

@deploy
def __init__():
    self.deadline = block.timestamp + DURATION
    self.isActive = True

@external
def doAction():
    assert block.timestamp <= self.deadline, "Deadline passed"
    assert self.isActive, "Not active"
    # ทำงาน...

@external
def checkAndDeactivate():
    if block.timestamp > self.deadline:
        self.isActive = False
```

### 4.3 Cooldown Pattern

```vyper
# @version 0.4.0

COOLDOWN_PERIOD: constant(uint256) = 3600  # 1 ชั่วโมง

lastAction: HashMap[address, uint256]

@external
def actionWithCooldown():
    # ตรวจสอบ Cooldown
    lastTime: uint256 = self.lastAction[msg.sender]
    if lastTime != 0:
        assert block.timestamp >= lastTime + COOLDOWN_PERIOD, "Cooldown active"
    
    # บันทึกเวลาล่าสุด
    self.lastAction[msg.sender] = block.timestamp
    
    # ทำงาน...

@external
@view
def getCooldownRemaining(user: address) -> uint256:
    lastTime: uint256 = self.lastAction[user]
    if lastTime == 0:
        return 0
    
    cooldownEnd: uint256 = lastTime + COOLDOWN_PERIOD
    if block.timestamp >= cooldownEnd:
        return 0
    
    return cooldownEnd - block.timestamp
```

### 4.4 Vesting Schedule

```vyper
# @version 0.4.0

struct VestingSchedule:
    totalAmount: uint256
    startTime: uint256
    duration: uint256
    claimed: uint256

vestings: HashMap[address, VestingSchedule]
token: address

@deploy
def __init__(tokenAddress: address):
    self.token = tokenAddress

@external
def createVesting(beneficiary: address, amount: uint256, duration: uint256):
    assert beneficiary != empty(address), "Invalid beneficiary"
    assert amount > 0, "Amount must be positive"
    assert duration > 0, "Duration must be positive"
    
    self.vestings[beneficiary] = VestingSchedule({
        totalAmount: amount,
        startTime: block.timestamp,
        duration: duration,
        claimed: 0
    })

@external
@view
def vestedAmount(beneficiary: address) -> uint256:
    schedule: VestingSchedule = self.vestings[beneficiary]
    if schedule.totalAmount == 0:
        return 0
    
    elapsed: uint256 = block.timestamp - schedule.startTime
    if elapsed >= schedule.duration:
        return schedule.totalAmount
    
    return schedule.totalAmount * elapsed / schedule.duration

@external
@view
def claimableAmount(beneficiary: address) -> uint256:
    vested: uint256 = self.vestedAmount(beneficiary)
    claimed: uint256 = self.vestings[beneficiary].claimed
    if vested <= claimed:
        return 0
    return vested - claimed
```

---

## 5. ตัวอย่าง: Time-locked Vault Contract {#time-locked-vault}

Contract นี้เป็น Vault ที่ล็อค ETH ไว้จนกว่าจะถึงเวลาที่กำหนด

```vyper
# @version 0.4.0
"""
@title Time-locked Vault
@notice เก็บ ETH และล็อคจนกว่าจะถึงเวลาที่กำหนด
@dev รองรับการฝาก ETH หลายครั้ง แต่ถอนได้เมื่อล็อคไทม์ผ่านเท่านั้น
"""

# Events
event Deposited:
    depositor: indexed(address)
    amount: uint256
    unlockTime: uint256

event Withdrawn:
    recipient: indexed(address)
    amount: uint256

event LockExtended:
    by: indexed(address)
    newUnlockTime: uint256

# State Variables
owner: public(address)
unlockTime: public(uint256)
totalDeposited: public(uint256)
deposits: public(HashMap[address, uint256])

MIN_LOCK_DURATION: constant(uint256) = 3600        # 1 ชั่วโมง
MAX_LOCK_DURATION: constant(uint256) = 31536000 * 5 # 5 ปี
MAX_EXTENSION: constant(uint256) = 31536000         # 1 ปีต่อครั้ง

@deploy
def __init__(lockDuration: uint256):
    """
    @param lockDuration ระยะเวลาล็อคเป็นวินาที
    """
    assert lockDuration >= MIN_LOCK_DURATION, "Lock too short"
    assert lockDuration <= MAX_LOCK_DURATION, "Lock too long"
    
    self.owner = msg.sender
    self.unlockTime = block.timestamp + lockDuration

@external
@payable
def deposit():
    """
    @notice ฝาก ETH เข้า Vault
    """
    assert msg.value > 0, "Must send ETH"
    
    self.deposits[msg.sender] += msg.value
    self.totalDeposited += msg.value
    
    log Deposited(msg.sender, msg.value, self.unlockTime)

@external
def withdraw():
    """
    @notice ถอน ETH ออกจาก Vault (ต้องรอจนครบเวลา)
    """
    assert msg.sender == self.owner, "Not owner"
    assert block.timestamp >= self.unlockTime, "Still locked"
    
    amount: uint256 = self.balance
    assert amount > 0, "Nothing to withdraw"
    
    log Withdrawn(msg.sender, amount)
    send(msg.sender, amount)

@external
def withdrawPartial(amount: uint256):
    """
    @notice ถอน ETH บางส่วน (เฉพาะหลังล็อคหมด)
    """
    assert msg.sender == self.owner, "Not owner"
    assert block.timestamp >= self.unlockTime, "Still locked"
    assert amount > 0, "Amount must be positive"
    assert self.balance >= amount, "Insufficient balance"
    
    log Withdrawn(msg.sender, amount)
    send(msg.sender, amount)

@external
def extendLock(additionalSeconds: uint256):
    """
    @notice ขยายระยะเวลาล็อค
    """
    assert msg.sender == self.owner, "Not owner"
    assert additionalSeconds > 0, "Must extend by positive amount"
    assert additionalSeconds <= MAX_EXTENSION, "Extension too long"
    
    newUnlockTime: uint256 = self.unlockTime + additionalSeconds
    self.unlockTime = newUnlockTime
    
    log LockExtended(msg.sender, newUnlockTime)

@external
@view
def timeRemaining() -> uint256:
    """
    @notice เวลาที่เหลือก่อน Vault ปลดล็อค (วินาที)
    """
    if block.timestamp >= self.unlockTime:
        return 0
    return self.unlockTime - block.timestamp

@external
@view
def isUnlocked() -> bool:
    """
    @notice ตรวจสอบว่า Vault ปลดล็อคแล้วหรือยัง
    """
    return block.timestamp >= self.unlockTime

@external
@view
def getVaultInfo() -> (address, uint256, uint256, uint256, bool):
    """
    @notice ดึงข้อมูลของ Vault ทั้งหมด
    @return owner, unlockTime, balance, timeRemaining, isUnlocked
    """
    remaining: uint256 = 0
    if block.timestamp < self.unlockTime:
        remaining = self.unlockTime - block.timestamp
    
    return (
        self.owner,
        self.unlockTime,
        self.balance,
        remaining,
        block.timestamp >= self.unlockTime
    )

@external
@view
def getBlockInfo() -> (uint256, uint256, uint256):
    """
    @notice ดึงข้อมูล Block ปัจจุบัน
    @return blockNumber, timestamp, basefee
    """
    return (block.number, block.timestamp, block.basefee)
```

### Test Suite สำหรับ Time-locked Vault

```python
# tests/test_time_locked_vault.py
import pytest
import boa
from boa.test import strategy

@pytest.fixture
def vault():
    """Deploy vault with 1 hour lock"""
    return boa.load(
        "contracts/TimeLockedVault.vy",
        3600  # 1 hour
    )

@pytest.fixture
def funded_vault(vault):
    """Vault ที่มี ETH อยู่แล้ว"""
    vault.deposit(value=10**18)  # 1 ETH
    return vault

def test_deploy(vault):
    """ทดสอบ Deploy"""
    import boa
    deployer = boa.env.eoa
    
    assert vault.owner() == deployer
    assert vault.totalDeposited() == 0
    assert vault.isUnlocked() == False

def test_deposit(vault):
    """ทดสอบการฝาก ETH"""
    amount = 10**18  # 1 ETH
    vault.deposit(value=amount)
    
    deployer = boa.env.eoa
    assert vault.deposits(deployer) == amount
    assert vault.totalDeposited() == amount

def test_deposit_zero_fails(vault):
    """ทดสอบว่าฝาก 0 ETH ไม่ได้"""
    with pytest.raises(Exception, match="Must send ETH"):
        vault.deposit(value=0)

def test_withdraw_before_unlock_fails(funded_vault):
    """ทดสอบว่าถอนก่อนเวลาไม่ได้"""
    with pytest.raises(Exception, match="Still locked"):
        funded_vault.withdraw()

def test_withdraw_after_unlock(funded_vault):
    """ทดสอบถอนหลังครบเวลา"""
    import boa
    
    # เร่งเวลา
    boa.env.time_travel(seconds=3601)
    
    assert funded_vault.isUnlocked() == True
    
    initial_balance = boa.env.get_balance(boa.env.eoa)
    funded_vault.withdraw()
    
    # Balance ควรเพิ่มขึ้น
    assert boa.env.get_balance(boa.env.eoa) > initial_balance

def test_time_remaining(funded_vault):
    """ทดสอบ timeRemaining"""
    remaining = funded_vault.timeRemaining()
    assert remaining > 0
    assert remaining <= 3600
    
    # เร่งเวลาผ่าน
    boa.env.time_travel(seconds=3601)
    assert funded_vault.timeRemaining() == 0

def test_extend_lock(funded_vault):
    """ทดสอบการขยายเวลาล็อค"""
    old_unlock_time = funded_vault.unlockTime()
    
    funded_vault.extendLock(1800)  # ขยายอีก 30 นาที
    
    assert funded_vault.unlockTime() == old_unlock_time + 1800

def test_only_owner_can_withdraw(funded_vault):
    """ทดสอบว่าเฉพาะ Owner ถอนได้"""
    boa.env.time_travel(seconds=3601)
    
    with boa.env.prank(boa.env.generate_address()):
        with pytest.raises(Exception, match="Not owner"):
            funded_vault.withdraw()

def test_events(vault):
    """ทดสอบ Events"""
    amount = 10**18
    
    with vault.prank():
        receipt = vault.deposit(value=amount)
    
    # ตรวจสอบ Event
    events = vault.get_logs()
    assert len(events) > 0

def test_get_block_info(vault):
    """ทดสอบ getBlockInfo"""
    block_num, timestamp, basefee = vault.getBlockInfo()
    
    assert block_num > 0
    assert timestamp > 0
    assert basefee >= 0
```

---

## 6. แบบฝึกหัด {#exercises}

### แบบฝึกหัดที่ 1: Auction Contract
สร้าง Contract สำหรับ Auction ที่:
- กำหนดเวลาเริ่มและสิ้นสุด Auction
- ผู้เสนอราคาสูงสุดชนะ
- ผู้แพ้ได้รับเงินคืนหลัง Auction สิ้นสุด

### แบบฝึกหัดที่ 2: Recurring Payment
สร้าง Contract สำหรับ Subscription ที่:
- ผู้ใช้จ่ายเงินทุกเดือน
- ตรวจสอบ Cooldown ระหว่างการจ่าย
- หยุด Service ถ้าไม่จ่ายในเวลา

### แบบฝึกหัดที่ 3: Multi-sig Time-lock
สร้าง Contract ที่ต้องการหลาย Signature และมี Time-lock

---

## สรุป

| Variable | Type | Description |
|----------|------|-------------|
| `block.number` | uint256 | หมายเลข Block ปัจจุบัน |
| `block.timestamp` | uint256 | Unix Timestamp (วินาที) |
| `block.prevhash` | bytes32 | Hash ของ Block ก่อนหน้า |
| `block.coinbase` | address | Address ของ Validator |
| `block.gaslimit` | uint256 | Gas Limit ของ Block |
| `block.basefee` | uint256 | Base Fee (Wei/Gas) |
| `msg.sender` | address | ผู้เรียก Function โดยตรง |
| `msg.value` | uint256 | ETH ที่ส่งมา (Wei) |
| `msg.gas` | uint256 | Gas ที่เหลืออยู่ |
| `tx.gasprice` | uint256 | Gas Price ของ Transaction |
| `tx.origin` | address | EOA ที่เริ่ม Transaction |
| `chain.id` | uint256 | Chain ID ของ Network |

---

[← Part 015: Events และ Logging](part_015_events.md) | [Part 017: Address Types →](part_017_address.md)
