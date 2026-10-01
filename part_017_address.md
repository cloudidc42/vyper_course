# Part 017: Address Types และ Payable

## สารบัญ
1. [Address Type Basics](#address-basics)
2. [Address Properties](#address-properties)
3. [Sending ETH](#sending-eth)
4. [raw_call สำหรับ ETH](#raw-call)
5. [ตัวอย่าง: MultiSend Contract](#multisend)
6. [Best Practices](#best-practices)
7. [แบบฝึกหัด](#exercises)

---

## 1. Address Type Basics {#address-basics}

ใน Vyper, `address` เป็น Type พื้นฐานสำหรับเก็บ Ethereum Address (20 bytes)

### การประกาศตัวแปร Address

```vyper
# @version 0.4.0

# ประกาศตัวแปร address
owner: public(address)
treasury: address
zeroAddress: address  # Default = empty(address) = 0x0000...0000

# Constant address
BURN_ADDRESS: constant(address) = 0x000000000000000000000000000000000000dEaD
WETH: constant(address) = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2

@deploy
def __init__():
    self.owner = msg.sender
    self.treasury = msg.sender

@external
@view
def isValidAddress(addr: address) -> bool:
    # ตรวจสอบว่าไม่ใช่ Zero Address
    return addr != empty(address)
```

### Address Literals

```vyper
# @version 0.4.0

@external
@view
def getAddressInfo() -> (address, address, bool):
    # Address literal ต้องเป็น Checksummed Address
    validAddr: address = 0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045
    zeroAddr: address = empty(address)
    
    isZero: bool = validAddr == empty(address)
    
    return validAddr, zeroAddr, isZero
```

### การเปรียบเทียบ Address

```vyper
# @version 0.4.0

whitelist: HashMap[address, bool]

@deploy
def __init__():
    self.whitelist[msg.sender] = True

@external
def addToWhitelist(addr: address):
    assert msg.sender == self.owner(), "Not owner"
    assert addr != empty(address), "Invalid address"
    assert not self.whitelist[addr], "Already whitelisted"
    self.whitelist[addr] = True

@internal
@view
def owner() -> address:
    return msg.sender  # Simplified

@external
@view
def isWhitelisted(addr: address) -> bool:
    return self.whitelist[addr]

@external
@view
def isSelf(addr: address) -> bool:
    # self คือ address ของ Contract นี้
    return addr == self
```

---

## 2. Address Properties {#address-properties}

### 2.1 .balance

ดู ETH Balance (Wei) ของ Address ใดๆ

```vyper
# @version 0.4.0

@external
@view
def getBalance(addr: address) -> uint256:
    """
    @notice ดู ETH Balance ของ address
    @param addr address ที่ต้องการดู balance
    @return balance ใน Wei
    """
    return addr.balance

@external
@view
def getContractBalance() -> uint256:
    """ดู Balance ของ Contract นี้"""
    return self.balance

@external
@view
def getOwnerBalance() -> uint256:
    return self.owner.balance

@external
@view
def isWealthy(addr: address, threshold: uint256) -> bool:
    """
    @notice ตรวจสอบว่า Address มี ETH มากกว่า Threshold
    """
    return addr.balance >= threshold

owner: address

@deploy
def __init__():
    self.owner = msg.sender
```

### 2.2 .code_size (is_contract check)

ตรวจสอบว่า Address เป็น Smart Contract หรือ EOA

```vyper
# @version 0.4.0

@external
@view
def isContract(addr: address) -> bool:
    """
    @notice ตรวจสอบว่า address เป็น Smart Contract
    @dev Contract มี code_size > 0, EOA มี code_size = 0
    """
    return addr.code_size > 0

@external
@view  
def isEOA(addr: address) -> bool:
    """
    @notice ตรวจสอบว่า address เป็น EOA (Externally Owned Account)
    """
    return addr.code_size == 0

@external
@view
def getCodeSize(addr: address) -> uint256:
    """ดูขนาด Bytecode ของ Contract"""
    return addr.code_size
```

**ข้อควรระวัง:**
```
ระหว่าง Constructor ของ Contract กำลังทำงาน:
  - addr.code_size จะเป็น 0
  - ดังนั้น isContract() จะคืนค่า false แม้เป็น Contract
  → ไม่ควรใช้เพื่อ Security Check ที่สำคัญ
```

### 2.3 การใช้ Address ร่วมกับ Interface

```vyper
# @version 0.4.0

interface IERC20:
    def balanceOf(account: address) -> uint256: view
    def transfer(to: address, amount: uint256) -> bool: nonpayable

@external
@view
def getTokenBalance(token: address, account: address) -> uint256:
    """
    @notice ดู Token Balance ของ account
    """
    assert token.code_size > 0, "Not a contract"
    return IERC20(token).balanceOf(account)
```

---

## 3. Sending ETH {#sending-eth}

### 3.1 send() - วิธีมาตรฐาน

`send()` ส่ง ETH ไปยัง Address โดยตรง

```vyper
# @version 0.4.0

balances: public(HashMap[address, uint256])

event Withdrawal:
    recipient: indexed(address)
    amount: uint256

@external
@payable
def deposit():
    """ฝาก ETH เข้า Contract"""
    assert msg.value > 0, "Must send ETH"
    self.balances[msg.sender] += msg.value

@external
def withdraw(amount: uint256):
    """
    @notice ถอน ETH ออกจาก Contract
    @dev ใช้ Pattern: Check-Effects-Interactions
    """
    # Check
    assert amount > 0, "Amount must be positive"
    assert self.balances[msg.sender] >= amount, "Insufficient balance"
    
    # Effects (อัปเดต State ก่อนส่ง ETH)
    self.balances[msg.sender] -= amount
    
    # Interactions (ส่ง ETH หลังสุด)
    log Withdrawal(msg.sender, amount)
    send(msg.sender, amount)

@external
def withdrawAll():
    """ถอน ETH ทั้งหมด"""
    amount: uint256 = self.balances[msg.sender]
    assert amount > 0, "Nothing to withdraw"
    
    self.balances[msg.sender] = 0
    
    log Withdrawal(msg.sender, amount)
    send(msg.sender, amount)
```

**Gas Stipend ของ send():**
- ส่ง Gas Stipend 2300 Gas ไปด้วย (สำหรับ Fallback Function)
- ถ้า Recipient เป็น Contract ที่ต้องการ Gas มากกว่า 2300 จะ Revert

### 3.2 ส่ง ETH ไปยัง Payable Contract

```vyper
# @version 0.4.0

interface IReceiver:
    def receivePayment(data: String[100]): payable

@external
@payable
def sendToContract(target: address, data: String[100]):
    """ส่ง ETH พร้อม Data ไปยัง Contract"""
    assert msg.value > 0, "Must send ETH"
    assert target.code_size > 0, "Not a contract"
    
    # เรียก Payable Function ของ Contract อื่น
    IReceiver(target).receivePayment(data, value=msg.value)
```

---

## 4. raw_call สำหรับ ETH {#raw-call}

`raw_call()` ใช้สำหรับ Low-level Call และสามารถส่ง ETH ได้

### 4.1 Syntax ของ raw_call

```vyper
# @version 0.4.0

@external
@payable
def sendETH(recipient: address):
    """
    @notice ส่ง ETH ด้วย raw_call
    @dev ใช้เมื่อต้องการ Gas มากกว่า 2300
    """
    assert msg.value > 0, "Must send ETH"
    assert recipient != empty(address), "Invalid recipient"
    
    # raw_call ส่ง ETH โดยไม่เรียก Function
    raw_call(
        recipient,
        b"",          # Empty calldata
        value=msg.value,
        gas=2300      # หรือ msg.gas สำหรับ Gas ทั้งหมด
    )

@external
@payable
def sendETHWithData(recipient: address, data: Bytes[256]):
    """ส่ง ETH พร้อม Data"""
    success: bool = raw_call(
        recipient,
        data,
        value=msg.value,
        revert_on_failure=False
    )
    assert success, "Transfer failed"
```

### 4.2 การใช้ raw_call อย่างปลอดภัย

```vyper
# @version 0.4.0

@external
@payable
def safeTransfer(recipient: address, amount: uint256):
    """
    @notice ส่ง ETH อย่างปลอดภัย
    @dev ตรวจสอบ Success และ Revert ถ้าล้มเหลว
    """
    assert self.balance >= amount, "Insufficient balance"
    assert recipient != empty(address), "Invalid recipient"
    
    # ใช้ raw_call พร้อม gas สูงกว่า 2300
    success: bool = raw_call(
        recipient,
        b"",
        value=amount,
        gas=50000,
        revert_on_failure=False
    )
    
    if not success:
        raise "ETH transfer failed"
```

### 4.3 Comparison: send vs raw_call

```
send(addr, amount)
  ✓ ง่ายและอ่านง่าย  
  ✓ Gas stipend: 2300 gas
  ✗ จะ Revert ถ้า Recipient ต้องการ Gas มากกว่า 2300
  
raw_call(addr, b"", value=amount)
  ✓ ควบคุม Gas ได้
  ✓ ใช้กับ Smart Wallet ได้
  ✗ ซับซ้อนกว่า
  ✗ ต้องตรวจสอบ Return Value
```

---

## 5. ตัวอย่าง: MultiSend Contract {#multisend}

Contract ที่ส่ง ETH ไปยังหลาย Address พร้อมกัน

```vyper
# @version 0.4.0
"""
@title MultiSend Contract
@notice ส่ง ETH ไปยังหลาย Address ในธุรกรรมเดียว
@dev สนับสนุนทั้งการส่งจำนวนเท่ากันและจำนวนต่างกัน
"""

# Events
event BatchSent:
    sender: indexed(address)
    totalAmount: uint256
    recipientCount: uint256

event IndividualSent:
    recipient: indexed(address)
    amount: uint256

event RefundSent:
    recipient: indexed(address)
    amount: uint256

# Constants
MAX_RECIPIENTS: constant(uint256) = 200

# State
owner: public(address)
feePercent: public(uint256)  # Fee เป็น basis points (1% = 100)
feesCollected: public(uint256)

MAX_FEE: constant(uint256) = 500  # 5% maximum fee

@deploy
def __init__(feePercent: uint256):
    """
    @param feePercent ค่าธรรมเนียมเป็น basis points
    """
    assert feePercent <= MAX_FEE, "Fee too high"
    self.owner = msg.sender
    self.feePercent = feePercent

@external
@payable
def sendEqual(recipients: DynArray[address, MAX_RECIPIENTS]):
    """
    @notice ส่ง ETH จำนวนเท่ากันให้ทุกคน
    @param recipients รายชื่อผู้รับ
    """
    count: uint256 = len(recipients)
    assert count > 0, "No recipients"
    assert msg.value > 0, "Must send ETH"
    
    # คำนวณ Fee
    fee: uint256 = msg.value * self.feePercent / 10000
    netAmount: uint256 = msg.value - fee
    
    # จำนวนต่อคน
    amountPerRecipient: uint256 = netAmount / count
    assert amountPerRecipient > 0, "Amount too small"
    
    # ส่ง ETH
    totalSent: uint256 = 0
    for recipient: address in recipients:
        assert recipient != empty(address), "Invalid recipient"
        send(recipient, amountPerRecipient)
        log IndividualSent(recipient, amountPerRecipient)
        totalSent += amountPerRecipient
    
    # เก็บ Fee
    self.feesCollected += fee
    
    # คืนเงินส่วนที่เหลือ (จาก Integer Division)
    refund: uint256 = msg.value - totalSent - fee
    if refund > 0:
        send(msg.sender, refund)
        log RefundSent(msg.sender, refund)
    
    log BatchSent(msg.sender, totalSent, count)

@external
@payable
def sendDifferent(
    recipients: DynArray[address, MAX_RECIPIENTS],
    amounts: DynArray[uint256, MAX_RECIPIENTS]
):
    """
    @notice ส่ง ETH จำนวนต่างกันให้แต่ละคน
    @param recipients รายชื่อผู้รับ
    @param amounts จำนวน ETH ของแต่ละคน (Wei)
    """
    count: uint256 = len(recipients)
    assert count > 0, "No recipients"
    assert count == len(amounts), "Length mismatch"
    
    # คำนวณ Total Amount ที่ต้องส่ง
    totalRequired: uint256 = 0
    for amount: uint256 in amounts:
        totalRequired += amount
    
    # คำนวณ Fee
    fee: uint256 = totalRequired * self.feePercent / 10000
    
    assert msg.value >= totalRequired + fee, "Insufficient ETH sent"
    
    # ส่ง ETH แต่ละคน
    for i: uint256 in range(MAX_RECIPIENTS):
        if i >= count:
            break
        
        recipient: address = recipients[i]
        amount: uint256 = amounts[i]
        
        assert recipient != empty(address), "Invalid recipient"
        assert amount > 0, "Amount must be positive"
        
        send(recipient, amount)
        log IndividualSent(recipient, amount)
    
    # เก็บ Fee
    self.feesCollected += fee
    
    # คืนเงินส่วนเกิน
    refund: uint256 = msg.value - totalRequired - fee
    if refund > 0:
        send(msg.sender, refund)
        log RefundSent(msg.sender, refund)
    
    log BatchSent(msg.sender, totalRequired, count)

@external
def collectFees():
    """
    @notice Owner เก็บ Fee ที่สะสม
    """
    assert msg.sender == self.owner, "Not owner"
    
    amount: uint256 = self.feesCollected
    assert amount > 0, "No fees to collect"
    
    self.feesCollected = 0
    send(self.owner, amount)

@external
def updateFee(newFeePercent: uint256):
    """
    @notice อัปเดต Fee Percentage
    """
    assert msg.sender == self.owner, "Not owner"
    assert newFeePercent <= MAX_FEE, "Fee too high"
    
    self.feePercent = newFeePercent

@external
def transferOwnership(newOwner: address):
    """
    @notice โอน Ownership
    """
    assert msg.sender == self.owner, "Not owner"
    assert newOwner != empty(address), "Invalid address"
    
    self.owner = newOwner

@external
@view
def calculateFee(amount: uint256) -> uint256:
    """
    @notice คำนวณ Fee สำหรับ Amount ที่กำหนด
    """
    return amount * self.feePercent / 10000

@external
@view
def estimateCostEqual(
    recipientCount: uint256,
    totalAmount: uint256
) -> (uint256, uint256, uint256):
    """
    @notice ประมาณค่าใช้จ่ายสำหรับ sendEqual
    @return fee, amountPerRecipient, totalNeeded
    """
    fee: uint256 = totalAmount * self.feePercent / 10000
    netAmount: uint256 = totalAmount - fee
    amountPerRecipient: uint256 = netAmount / recipientCount
    
    return fee, amountPerRecipient, totalAmount

@external
@view
def getContractInfo() -> (address, uint256, uint256):
    """
    @return owner, feePercent, feesCollected
    """
    return self.owner, self.feePercent, self.feesCollected
```

### Test Suite สำหรับ MultiSend

```python
# tests/test_multisend.py
import pytest
import boa

@pytest.fixture
def owner():
    return boa.env.eoa

@pytest.fixture
def multisend(owner):
    """Deploy MultiSend with 1% fee"""
    return boa.load("contracts/MultiSend.vy", 100)  # 1% = 100 bps

@pytest.fixture
def recipients():
    """สร้าง Address ผู้รับ 5 คน"""
    return [boa.env.generate_address() for _ in range(5)]

def test_deploy(multisend, owner):
    """ทดสอบ Deploy"""
    assert multisend.owner() == owner
    assert multisend.feePercent() == 100
    assert multisend.feesCollected() == 0

def test_send_equal(multisend, recipients):
    """ทดสอบส่ง ETH เท่ากัน"""
    amount = 5 * 10**18  # 5 ETH total (1 ETH each)
    
    initial_balances = [boa.env.get_balance(r) for r in recipients]
    
    multisend.sendEqual(recipients, value=amount)
    
    # คำนวณจำนวนที่แต่ละคนได้
    fee = amount * 100 // 10000  # 1%
    net = amount - fee
    per_recipient = net // len(recipients)
    
    for i, recipient in enumerate(recipients):
        new_balance = boa.env.get_balance(recipient)
        assert new_balance == initial_balances[i] + per_recipient

def test_send_different(multisend, recipients):
    """ทดสอบส่ง ETH ต่างกัน"""
    amounts = [10**18 * (i+1) for i in range(5)]  # 1,2,3,4,5 ETH
    total = sum(amounts)
    fee = total * 100 // 10000
    
    initial_balances = [boa.env.get_balance(r) for r in recipients]
    
    multisend.sendDifferent(recipients, amounts, value=total + fee + 10**15)
    
    for i, (recipient, amount) in enumerate(zip(recipients, amounts)):
        new_balance = boa.env.get_balance(recipient)
        assert new_balance == initial_balances[i] + amount

def test_send_no_recipients_fails(multisend):
    """ทดสอบว่าส่งโดยไม่มีผู้รับไม่ได้"""
    with pytest.raises(Exception):
        multisend.sendEqual([], value=10**18)

def test_collect_fees(multisend, recipients):
    """ทดสอบการเก็บ Fee"""
    amount = 5 * 10**18
    multisend.sendEqual(recipients, value=amount)
    
    expected_fee = amount * 100 // 10000
    assert multisend.feesCollected() == expected_fee
    
    initial_balance = boa.env.get_balance(boa.env.eoa)
    multisend.collectFees()
    
    assert multisend.feesCollected() == 0
    assert boa.env.get_balance(boa.env.eoa) > initial_balance

def test_only_owner_collects_fees(multisend, recipients):
    """ทดสอบว่าเฉพาะ Owner เก็บ Fee ได้"""
    multisend.sendEqual(recipients, value=5 * 10**18)
    
    attacker = boa.env.generate_address()
    with boa.env.prank(attacker):
        with pytest.raises(Exception, match="Not owner"):
            multisend.collectFees()

def test_update_fee(multisend):
    """ทดสอบอัปเดต Fee"""
    multisend.updateFee(200)  # 2%
    assert multisend.feePercent() == 200

def test_fee_too_high_fails(multisend):
    """ทดสอบว่า Fee เกิน 5% ไม่ได้"""
    with pytest.raises(Exception, match="Fee too high"):
        multisend.updateFee(501)

def test_length_mismatch_fails(multisend, recipients):
    """ทดสอบว่า recipients และ amounts ต้องมีขนาดเท่ากัน"""
    amounts = [10**18] * 3  # น้อยกว่า recipients
    total = sum(amounts)
    
    with pytest.raises(Exception, match="Length mismatch"):
        multisend.sendDifferent(recipients, amounts, value=total)

def test_refund_excess(multisend, recipients):
    """ทดสอบการคืนเงินส่วนเกิน"""
    initial_balance = boa.env.get_balance(boa.env.eoa)
    
    # ส่งเงินเกินกว่าที่ต้องการ
    excess = 5 * 10**17  # 0.5 ETH เกิน
    amounts = [10**18] * 5
    total = sum(amounts)
    fee = total * 100 // 10000
    
    multisend.sendDifferent(recipients, amounts, value=total + fee + excess)
    
    # Balance ควรได้คืนเงินส่วนเกิน
    final_balance = boa.env.get_balance(boa.env.eoa)
    # final_balance ≈ initial_balance - total - fee
```

---

## 6. Best Practices {#best-practices}

### 6.1 Checks-Effects-Interactions Pattern

```vyper
# @version 0.4.0

balances: HashMap[address, uint256]

@external
def withdraw(amount: uint256):
    # 1. Checks - ตรวจสอบเงื่อนไขทั้งหมดก่อน
    assert amount > 0, "Zero amount"
    assert self.balances[msg.sender] >= amount, "Insufficient"
    assert self.balance >= amount, "Contract has no funds"
    
    # 2. Effects - อัปเดต State ก่อนส่ง ETH
    self.balances[msg.sender] -= amount
    
    # 3. Interactions - ส่ง ETH หลังสุด
    send(msg.sender, amount)
```

### 6.2 ป้องกัน Reentrancy

```vyper
# @version 0.4.0

locked: bool
balances: HashMap[address, uint256]

@external
def secureWithdraw(amount: uint256):
    # Mutex Lock เพื่อป้องกัน Reentrancy
    assert not self.locked, "Reentrant call"
    self.locked = True
    
    assert self.balances[msg.sender] >= amount, "Insufficient"
    self.balances[msg.sender] -= amount
    send(msg.sender, amount)
    
    self.locked = False
```

### 6.3 ตรวจสอบ Address ก่อนใช้งาน

```vyper
# @version 0.4.0

@internal
def validateAddress(addr: address):
    assert addr != empty(address), "Zero address"
    # สามารถเพิ่ม Checks เพิ่มเติมได้

@external
def sendToAddress(recipient: address, amount: uint256):
    self.validateAddress(recipient)
    assert amount > 0, "Zero amount"
    assert self.balance >= amount, "Insufficient"
    send(recipient, amount)
```

---

## 7. แบบฝึกหัด {#exercises}

### แบบฝึกหัดที่ 1: Airdrop Contract
สร้าง Contract ที่ Airdrop Token/ETH ให้กับ Whitelist โดย:
- Owner สามารถเพิ่ม/ลบ Whitelist
- แต่ละ Address รับได้ครั้งเดียว
- ต้องมี reentrancy protection

### แบบฝึกหัดที่ 2: Payment Splitter
สร้าง Contract ที่แบ่งรายได้ตามสัดส่วน:
- กำหนด Payees และ Shares ตอน Deploy
- ใครก็ได้ส่ง ETH เข้า Contract
- แต่ละคน Withdraw ได้ตามสัดส่วน

### แบบฝึกหัดที่ 3: Escrow
สร้าง Escrow Contract ที่:
- Buyer ฝาก ETH
- Arbiter ตัดสินใจปล่อยเงิน
- ถ้า Dispute ให้ Arbiter ยุติ

---

## สรุป

| Feature | Syntax | หมายเหตุ |
|---------|--------|----------|
| ดู Balance | `addr.balance` | ใน Wei |
| ตรวจสอบ Contract | `addr.code_size > 0` | อาจผิดพลาดใน Constructor |
| ส่ง ETH (ง่าย) | `send(addr, amount)` | Gas stipend 2300 |
| ส่ง ETH (ยืดหยุ่น) | `raw_call(addr, b"", value=amount)` | ควบคุม Gas ได้ |
| Address เปล่า | `empty(address)` | = 0x000...000 |
| Contract Address | `self` | Address ของ Contract นี้ |

---

[← Part 016: Block และ Transaction Variables](part_016_block_tx.md) | [Part 018: Error Handling →](part_018_error_handling.md)
