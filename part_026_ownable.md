# Part 026: Ownable Pattern

## สารบัญ
1. [บทนำ Ownable Pattern](#บทนำ)
2. [Owner Variable](#owner-variable)
3. [Ownership Transfer](#ownership-transfer)
4. [Renounce Ownership](#renounce-ownership)
5. [Two-Step Transfer](#two-step-transfer)
6. [Full Ownable Module](#full-ownable-module)
7. [ตัวอย่างการใช้งาน](#ตัวอย่างการใช้งาน)
8. [Test Code](#test-code)

---

## บทนำ

**Ownable Pattern** เป็น Design Pattern พื้นฐานที่สุดใน Smart Contract โดยมีแนวคิดว่า Contract ควรมีเจ้าของ (Owner) ที่มีสิทธิ์พิเศษในการจัดการ Contract นั้น

### ทำไมต้องใช้ Ownable?
- ควบคุมการเข้าถึงฟังก์ชันสำคัญ
- สามารถอัปเดตค่า configuration ต่างๆ ได้
- ป้องกันการเรียกใช้งานโดยไม่ได้รับอนุญาต
- รองรับการโอนความเป็นเจ้าของในอนาคต

---

## Owner Variable

### โครงสร้างพื้นฐาน

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0

# ตัวแปร owner เก็บที่อยู่ของเจ้าของ Contract
owner: public(address)

@deploy
def __init__():
    # ตั้งค่า owner เป็น msg.sender (ผู้ deploy)
    self.owner = msg.sender
```

### การตรวจสอบ Owner

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0

owner: public(address)

event OwnershipTransferred:
    previous_owner: indexed(address)
    new_owner: indexed(address)

@deploy
def __init__():
    self.owner = msg.sender
    log OwnershipTransferred(empty(address), msg.sender)

@internal
def _check_owner():
    """
    ฟังก์ชัน internal สำหรับตรวจสอบว่า msg.sender เป็น owner หรือไม่
    """
    assert msg.sender == self.owner, "Ownable: caller is not the owner"

@external
def sensitive_function():
    """
    ฟังก์ชันที่ต้องการสิทธิ์ owner เท่านั้น
    """
    self._check_owner()
    # logic สำคัญอยู่ที่นี่
    pass
```

---

## Ownership Transfer

### การโอนความเป็นเจ้าของแบบพื้นฐาน

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0

owner: public(address)

event OwnershipTransferred:
    previous_owner: indexed(address)
    new_owner: indexed(address)

@deploy
def __init__():
    self.owner = msg.sender
    log OwnershipTransferred(empty(address), msg.sender)

@internal
def _check_owner():
    assert msg.sender == self.owner, "Ownable: caller is not the owner"

@internal
def _transfer_ownership(new_owner: address):
    """
    ฟังก์ชัน internal สำหรับโอน ownership
    """
    old_owner: address = self.owner
    self.owner = new_owner
    log OwnershipTransferred(old_owner, new_owner)

@external
def transfer_ownership(new_owner: address):
    """
    โอน ownership ไปยัง address ใหม่
    เฉพาะ owner เท่านั้นที่เรียกได้
    """
    self._check_owner()
    assert new_owner != empty(address), "Ownable: new owner is the zero address"
    self._transfer_ownership(new_owner)
```

---

## Renounce Ownership

### การสละสิทธิ์ความเป็นเจ้าของ

เมื่อ renounce ownership แล้ว ไม่มีใครเป็นเจ้าของ Contract อีกต่อไป ฟังก์ชันที่ต้องการสิทธิ์ owner จะไม่สามารถเรียกได้อีก

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0

owner: public(address)

event OwnershipTransferred:
    previous_owner: indexed(address)
    new_owner: indexed(address)

@deploy
def __init__():
    self.owner = msg.sender
    log OwnershipTransferred(empty(address), msg.sender)

@internal
def _check_owner():
    assert msg.sender == self.owner, "Ownable: caller is not the owner"

@internal
def _transfer_ownership(new_owner: address):
    old_owner: address = self.owner
    self.owner = new_owner
    log OwnershipTransferred(old_owner, new_owner)

@external
def transfer_ownership(new_owner: address):
    self._check_owner()
    assert new_owner != empty(address), "Ownable: new owner is the zero address"
    self._transfer_ownership(new_owner)

@external
def renounce_ownership():
    """
    สละสิทธิ์ความเป็นเจ้าของ Contract
    หลังจากนี้ไม่มี owner อีกต่อไป (owner = address(0))
    คำเตือน: การกระทำนี้ไม่สามารถย้อนกลับได้!
    """
    self._check_owner()
    self._transfer_ownership(empty(address))
```

---

## Two-Step Transfer

### การโอน Ownership แบบ 2 ขั้นตอน (ปลอดภัยกว่า)

ปัญหาของการโอนแบบ 1 ขั้นตอนคือ ถ้าพิมพ์ address ผิด ownership จะสูญหายไปตลอดกาล Two-step transfer แก้ปัญหานี้โดยให้ผู้รับต้อง "ยืนยัน" ก่อน

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0

owner: public(address)
pending_owner: public(address)

event OwnershipTransferred:
    previous_owner: indexed(address)
    new_owner: indexed(address)

event OwnershipTransferStarted:
    previous_owner: indexed(address)
    new_owner: indexed(address)

@deploy
def __init__():
    self.owner = msg.sender
    log OwnershipTransferred(empty(address), msg.sender)

@internal
def _check_owner():
    assert msg.sender == self.owner, "Ownable2Step: caller is not the owner"

@external
def transfer_ownership(new_owner: address):
    """
    เริ่มกระบวนการโอน ownership (ขั้นตอนที่ 1)
    owner กำหนด pending_owner แต่ยังไม่โอนจริง
    """
    self._check_owner()
    self.pending_owner = new_owner
    log OwnershipTransferStarted(self.owner, new_owner)

@external
def accept_ownership():
    """
    ยืนยันการรับ ownership (ขั้นตอนที่ 2)
    เฉพาะ pending_owner เท่านั้นที่เรียกได้
    """
    assert msg.sender == self.pending_owner, "Ownable2Step: caller is not the new owner"
    
    old_owner: address = self.owner
    self.owner = msg.sender
    self.pending_owner = empty(address)
    
    log OwnershipTransferred(old_owner, msg.sender)

@external
def renounce_ownership():
    """
    สละสิทธิ์ความเป็นเจ้าของ
    ยังคงใช้แบบ 1 ขั้นตอน เพราะ owner ต้องการยืนยันการกระทำนี้เอง
    """
    self._check_owner()
    old_owner: address = self.owner
    self.owner = empty(address)
    self.pending_owner = empty(address)
    log OwnershipTransferred(old_owner, empty(address))
```

---

## Full Ownable Module

### Ownable Contract สมบูรณ์พร้อมฟีเจอร์ครบถ้วน

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Ownable
# @notice Contract ที่มีระบบ Ownership ครบถ้วน
# @dev ใช้ Two-Step Transfer เพื่อความปลอดภัย

# ==================== Events ====================

event OwnershipTransferStarted:
    previous_owner: indexed(address)
    new_owner: indexed(address)

event OwnershipTransferred:
    previous_owner: indexed(address)
    new_owner: indexed(address)

event OwnershipRenounced:
    previous_owner: indexed(address)

# ==================== State Variables ====================

owner: public(address)
pending_owner: public(address)

# ==================== Constructor ====================

@deploy
def __init__():
    """
    @notice ตั้งค่า owner เป็น deployer
    """
    initial_owner: address = msg.sender
    self.owner = initial_owner
    log OwnershipTransferred(empty(address), initial_owner)

# ==================== Internal Functions ====================

@internal
def _check_owner():
    """
    @dev ตรวจสอบว่า caller เป็น owner
    """
    assert msg.sender == self.owner, "Ownable: caller is not the owner"

@internal  
def _check_owner_or_pending():
    """
    @dev ตรวจสอบว่า caller เป็น owner หรือ pending_owner
    """
    assert (msg.sender == self.owner or msg.sender == self.pending_owner), \
        "Ownable: caller is not owner or pending owner"

# ==================== Owner Functions ====================

@external
def transfer_ownership(new_owner: address):
    """
    @notice เริ่มกระบวนการโอน ownership
    @param new_owner Address ของเจ้าของใหม่
    @dev กำหนด pending_owner แต่ยังไม่โอนจริง
         new_owner ต้อง accept_ownership() เพื่อรับ ownership
    """
    self._check_owner()
    assert new_owner != empty(address), "Ownable: new owner cannot be zero address"
    assert new_owner != self.owner, "Ownable: new owner is already the owner"
    
    self.pending_owner = new_owner
    log OwnershipTransferStarted(self.owner, new_owner)

@external
def accept_ownership():
    """
    @notice ยืนยันการรับ ownership
    @dev เฉพาะ pending_owner เท่านั้น
    """
    assert msg.sender == self.pending_owner, \
        "Ownable: caller is not the pending owner"
    
    old_owner: address = self.owner
    self.owner = msg.sender
    self.pending_owner = empty(address)
    
    log OwnershipTransferred(old_owner, msg.sender)

@external
def cancel_transfer_ownership():
    """
    @notice ยกเลิกการโอน ownership ที่รอดำเนินการอยู่
    @dev เฉพาะ owner เท่านั้น
    """
    self._check_owner()
    assert self.pending_owner != empty(address), "Ownable: no pending transfer"
    
    self.pending_owner = empty(address)

@external
def renounce_ownership():
    """
    @notice สละสิทธิ์ความเป็นเจ้าของ Contract
    @dev คำเตือน: ไม่สามารถย้อนกลับได้!
         ฟังก์ชันทั้งหมดที่ต้องการ owner จะไม่สามารถเรียกได้อีก
    """
    self._check_owner()
    
    old_owner: address = self.owner
    self.owner = empty(address)
    self.pending_owner = empty(address)
    
    log OwnershipRenounced(old_owner)
    log OwnershipTransferred(old_owner, empty(address))

# ==================== View Functions ====================

@view
@external
def is_owner(account: address) -> bool:
    """
    @notice ตรวจสอบว่า address เป็น owner หรือไม่
    @param account Address ที่ต้องการตรวจสอบ
    @return True ถ้าเป็น owner
    """
    return account == self.owner

@view
@external
def has_pending_transfer() -> bool:
    """
    @notice ตรวจสอบว่ามี pending transfer อยู่หรือไม่
    @return True ถ้ามี pending transfer
    """
    return self.pending_owner != empty(address)
```

---

## ตัวอย่างการใช้งาน

### Contract ที่ใช้ Ownable Pattern

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Token with Ownable
# @notice ตัวอย่าง Token ที่ใช้ Ownable Pattern

# ==================== Events ====================

event OwnershipTransferred:
    previous_owner: indexed(address)
    new_owner: indexed(address)

event Transfer:
    sender: indexed(address)
    receiver: indexed(address)
    amount: uint256

event Mint:
    to: indexed(address)
    amount: uint256

event Burn:
    from_: indexed(address)
    amount: uint256

event MaxSupplyUpdated:
    old_max: uint256
    new_max: uint256

# ==================== State Variables ====================

owner: public(address)
pending_owner: public(address)

name: public(String[64])
symbol: public(String[8])
decimals: public(uint8)
total_supply: public(uint256)
max_supply: public(uint256)

balances: HashMap[address, uint256]
allowances: HashMap[address, HashMap[address, uint256]]

# ==================== Constructor ====================

@deploy
def __init__(
    _name: String[64],
    _symbol: String[8],
    _decimals: uint8,
    _max_supply: uint256
):
    self.owner = msg.sender
    self.name = _name
    self.symbol = _symbol
    self.decimals = _decimals
    self.max_supply = _max_supply
    
    log OwnershipTransferred(empty(address), msg.sender)

# ==================== Internal ====================

@internal
def _check_owner():
    assert msg.sender == self.owner, "Ownable: not owner"

# ==================== Ownable Functions ====================

@external
def transfer_ownership(new_owner: address):
    self._check_owner()
    assert new_owner != empty(address), "Invalid address"
    self.pending_owner = new_owner

@external
def accept_ownership():
    assert msg.sender == self.pending_owner, "Not pending owner"
    old_owner: address = self.owner
    self.owner = msg.sender
    self.pending_owner = empty(address)
    log OwnershipTransferred(old_owner, msg.sender)

# ==================== Token Functions ====================

@external
def transfer(to: address, amount: uint256) -> bool:
    assert self.balances[msg.sender] >= amount, "Insufficient balance"
    self.balances[msg.sender] -= amount
    self.balances[to] += amount
    log Transfer(msg.sender, to, amount)
    return True

@external
def approve(spender: address, amount: uint256) -> bool:
    self.allowances[msg.sender][spender] = amount
    return True

@external
def transfer_from(from_: address, to: address, amount: uint256) -> bool:
    assert self.allowances[from_][msg.sender] >= amount, "Insufficient allowance"
    assert self.balances[from_] >= amount, "Insufficient balance"
    
    self.allowances[from_][msg.sender] -= amount
    self.balances[from_] -= amount
    self.balances[to] += amount
    
    log Transfer(from_, to, amount)
    return True

@external
def mint(to: address, amount: uint256):
    """
    @notice สร้าง Token ใหม่ (เฉพาะ owner)
    """
    self._check_owner()
    assert to != empty(address), "Cannot mint to zero address"
    assert self.total_supply + amount <= self.max_supply, "Exceeds max supply"
    
    self.balances[to] += amount
    self.total_supply += amount
    
    log Mint(to, amount)
    log Transfer(empty(address), to, amount)

@external
def burn(amount: uint256):
    """
    @notice เผา Token ของตัวเอง
    """
    assert self.balances[msg.sender] >= amount, "Insufficient balance"
    
    self.balances[msg.sender] -= amount
    self.total_supply -= amount
    
    log Burn(msg.sender, amount)
    log Transfer(msg.sender, empty(address), amount)

@external
def set_max_supply(new_max: uint256):
    """
    @notice อัปเดต max supply (เฉพาะ owner)
    """
    self._check_owner()
    assert new_max >= self.total_supply, "Cannot set below current supply"
    
    old_max: uint256 = self.max_supply
    self.max_supply = new_max
    
    log MaxSupplyUpdated(old_max, new_max)

@view
@external
def balance_of(account: address) -> uint256:
    return self.balances[account]

@view
@external
def allowance(owner_addr: address, spender: address) -> uint256:
    return self.allowances[owner_addr][spender]
```

---

## Test Code

```python
# tests/test_ownable.py
import pytest
from eth_account import Account

@pytest.fixture
def owner(accounts):
    return accounts[0]

@pytest.fixture
def new_owner(accounts):
    return accounts[1]

@pytest.fixture
def other(accounts):
    return accounts[2]

@pytest.fixture
def ownable_token(owner, project):
    return project.OwnableToken.deploy(
        "My Token",
        "MTK",
        18,
        1_000_000 * 10**18,
        sender=owner
    )

class TestOwnable:
    
    def test_initial_owner(self, ownable_token, owner):
        """ทดสอบว่า owner ถูกตั้งค่าถูกต้องตอน deploy"""
        assert ownable_token.owner() == owner.address
    
    def test_transfer_ownership_step1(self, ownable_token, owner, new_owner):
        """ทดสอบการเริ่มโอน ownership"""
        ownable_token.transfer_ownership(new_owner.address, sender=owner)
        assert ownable_token.pending_owner() == new_owner.address
        assert ownable_token.owner() == owner.address  # ยังไม่โอน
    
    def test_accept_ownership(self, ownable_token, owner, new_owner):
        """ทดสอบการรับ ownership"""
        ownable_token.transfer_ownership(new_owner.address, sender=owner)
        ownable_token.accept_ownership(sender=new_owner)
        
        assert ownable_token.owner() == new_owner.address
        assert ownable_token.pending_owner() == "0x0000000000000000000000000000000000000000"
    
    def test_cannot_accept_if_not_pending(self, ownable_token, owner, other):
        """ทดสอบว่าคนที่ไม่ใช่ pending_owner ไม่สามารถ accept ได้"""
        ownable_token.transfer_ownership("0x1234567890123456789012345678901234567890", sender=owner)
        
        with pytest.raises(Exception):
            ownable_token.accept_ownership(sender=other)
    
    def test_only_owner_can_transfer(self, ownable_token, other, new_owner):
        """ทดสอบว่าเฉพาะ owner เท่านั้นที่โอนได้"""
        with pytest.raises(Exception):
            ownable_token.transfer_ownership(new_owner.address, sender=other)
    
    def test_only_owner_can_mint(self, ownable_token, owner, other):
        """ทดสอบว่าเฉพาะ owner เท่านั้น mint ได้"""
        with pytest.raises(Exception):
            ownable_token.mint(other.address, 1000, sender=other)
    
    def test_mint_increases_supply(self, ownable_token, owner, other):
        """ทดสอบว่า mint เพิ่ม supply ถูกต้อง"""
        amount = 1000 * 10**18
        ownable_token.mint(other.address, amount, sender=owner)
        
        assert ownable_token.balance_of(other.address) == amount
        assert ownable_token.total_supply() == amount
    
    def test_renounce_ownership(self, ownable_token, owner):
        """ทดสอบการ renounce ownership"""
        # ต้องเพิ่มฟังก์ชัน renounce_ownership ใน contract ก่อน
        pass
    
    def test_ownership_transferred_event(self, ownable_token, owner, new_owner):
        """ทดสอบว่า event ถูก emit ถูกต้อง"""
        tx = ownable_token.transfer_ownership(new_owner.address, sender=owner)
        ownable_token.accept_ownership(sender=new_owner)
        # ตรวจสอบ event logs

class TestOwnableToken:
    
    def test_mint_and_transfer(self, ownable_token, owner, other):
        """ทดสอบ mint และ transfer"""
        amount = 500 * 10**18
        ownable_token.mint(owner.address, amount, sender=owner)
        ownable_token.transfer(other.address, 100 * 10**18, sender=owner)
        
        assert ownable_token.balance_of(owner.address) == 400 * 10**18
        assert ownable_token.balance_of(other.address) == 100 * 10**18
    
    def test_cannot_exceed_max_supply(self, ownable_token, owner):
        """ทดสอบว่าไม่สามารถ mint เกิน max supply"""
        max_supply = ownable_token.max_supply()
        
        with pytest.raises(Exception):
            ownable_token.mint(owner.address, max_supply + 1, sender=owner)
    
    def test_burn_reduces_supply(self, ownable_token, owner):
        """ทดสอบว่า burn ลด supply ถูกต้อง"""
        amount = 1000 * 10**18
        ownable_token.mint(owner.address, amount, sender=owner)
        ownable_token.burn(500 * 10**18, sender=owner)
        
        assert ownable_token.total_supply() == 500 * 10**18
```

---

## สรุป

Ownable Pattern เป็นพื้นฐานสำคัญที่ควรเข้าใจก่อน:

| Pattern | ความปลอดภัย | ความซับซ้อน | ใช้เมื่อ |
|---------|-------------|-------------|---------|
| Simple Ownable | ปานกลาง | ต่ำ | Contract ง่ายๆ |
| Two-Step Transfer | สูง | ปานกลาง | Production contracts |
| Renounce Ownership | N/A (permanent) | ต่ำ | Decentralize การควบคุม |

### ข้อควรระวัง
- ระวังการ renounce ownership เพราะไม่สามารถย้อนกลับได้
- ใช้ Two-Step Transfer ในโค้ด Production เสมอ
- ตรวจสอบ address ให้ถี่ถ้วนก่อนโอน

---

[⬅️ Part 025: Events and Logging](part_025_events.md) | [Part 027: Access Control ➡️](part_027_access_control.md)
