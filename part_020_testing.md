# Part 020: การทดสอบเบื้องต้นด้วย Pytest

## สารบัญ
1. [Titanoboa Setup](#titanoboa-setup)
2. [Pytest Basics](#pytest-basics)
3. [Testing State Changes](#state-changes)
4. [Testing Events](#testing-events)
5. [Testing Reverts](#testing-reverts)
6. [Coverage](#coverage)
7. [ตัวอย่าง: Complete Test Suite](#complete-test-suite)
8. [แบบฝึกหัด](#exercises)

---

## 1. Titanoboa Setup {#titanoboa-setup}

Titanoboa คือ Testing Framework สำหรับ Vyper ที่รวดเร็วและใช้งานง่าย

### การติดตั้ง

```bash
# สร้าง Virtual Environment
python -m venv venv
source venv/bin/activate  # Linux/Mac
# หรือ venv\Scripts\activate  # Windows

# ติดตั้ง Dependencies
pip install titanoboa pytest pytest-cov

# หรือใช้ requirements.txt
pip install -r requirements.txt
```

### requirements.txt

```
titanoboa>=0.2.0
pytest>=7.0.0
pytest-cov>=4.0.0
hypothesis>=6.0.0
```

### โครงสร้าง Project

```
my_project/
├── contracts/
│   ├── MyToken.vy
│   ├── MyNFT.vy
│   └── interfaces/
│       └── IERC20.vy
├── tests/
│   ├── conftest.py       # Shared fixtures
│   ├── test_token.py
│   ├── test_nft.py
│   └── test_integration.py
├── scripts/
│   └── deploy.py
├── pytest.ini
└── requirements.txt
```

### pytest.ini

```ini
[pytest]
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*
addopts = -v --tb=short
```

### conftest.py

```python
# tests/conftest.py
import pytest
import boa

@pytest.fixture(scope="session")
def accounts():
    """สร้าง Test Accounts"""
    return [boa.env.generate_address() for _ in range(10)]

@pytest.fixture(scope="session")
def deployer():
    """Account สำหรับ Deploy"""
    return boa.env.eoa

@pytest.fixture(scope="function")
def token(deployer):
    """Deploy Token ใหม่สำหรับแต่ละ Test"""
    return boa.load("contracts/MyToken.vy", "Test", "TEST", 10**24)

@pytest.fixture(scope="session")
def token_session():
    """Deploy Token ครั้งเดียวสำหรับทุก Test ใน Session"""
    return boa.load("contracts/MyToken.vy", "Test", "TEST", 10**24)
```

---

## 2. Pytest Basics {#pytest-basics}

### การเขียน Test Function

```python
# tests/test_basic.py
import boa
import pytest

def test_simple():
    """Test ง่ายที่สุด"""
    contract = boa.loads("""
# @version 0.4.0

value: public(uint256)

@deploy
def __init__(v: uint256):
    self.value = v

@external
def setValue(v: uint256):
    self.value = v
    """, 42)
    
    assert contract.value() == 42
    
    contract.setValue(100)
    assert contract.value() == 100

def test_with_fixture(token):
    """Test ที่ใช้ Fixture"""
    deployer = boa.env.eoa
    
    # ตรวจสอบ Initial State
    assert token.name() == "Test"
    assert token.symbol() == "TEST"
    assert token.totalSupply() == 10**24
    assert token.balanceOf(deployer) == 10**24
```

### Fixture Scopes

```python
# tests/conftest.py
import pytest
import boa

# function: Deploy ใหม่ทุก Test (ค่า Default)
@pytest.fixture(scope="function")
def fresh_token():
    return boa.load("contracts/Token.vy", "Token", "TKN", 10**24)

# class: Deploy ใหม่ทุก Class
@pytest.fixture(scope="class")
def class_token():
    return boa.load("contracts/Token.vy", "Token", "TKN", 10**24)

# module: Deploy ใหม่ทุก Module (ไฟล์)
@pytest.fixture(scope="module")
def module_token():
    return boa.load("contracts/Token.vy", "Token", "TKN", 10**24)

# session: Deploy ครั้งเดียวทั้ง Session
@pytest.fixture(scope="session")
def session_token():
    return boa.load("contracts/Token.vy", "Token", "TKN", 10**24)
```

### Parametrize Tests

```python
# tests/test_parametrize.py
import pytest
import boa

@pytest.fixture
def token():
    return boa.load("contracts/Token.vy", "Token", "TKN", 10**24)

@pytest.mark.parametrize("amount", [
    1,           # Min amount
    10**18,      # 1 token
    10**20,      # 100 tokens
    10**23,      # Max normal amount
])
def test_transfer_amounts(token, amount):
    """ทดสอบการ Transfer กับ Amount ต่างๆ"""
    user = boa.env.generate_address()
    token.transfer(user, amount)
    assert token.balanceOf(user) == amount

@pytest.mark.parametrize("name,symbol,supply", [
    ("Token A", "TKNA", 10**24),
    ("Token B", "TKNB", 10**18),
    ("Very Long Token Name", "VLT", 1),
])
def test_deploy_variations(name, symbol, supply):
    """ทดสอบ Deploy กับ Parameters ต่างๆ"""
    token = boa.load("contracts/Token.vy", name, symbol, supply)
    
    assert token.name() == name
    assert token.symbol() == symbol
    assert token.totalSupply() == supply
```

---

## 3. Testing State Changes {#state-changes}

### ทดสอบ Storage Changes

```python
# tests/test_state_changes.py
import boa
import pytest

@pytest.fixture
def vault():
    return boa.load("contracts/Vault.vy")

def test_deposit_updates_balance(vault):
    """ทดสอบว่า Deposit อัปเดต Balance"""
    user = boa.env.eoa
    amount = 10**18  # 1 ETH
    
    # บันทึก State ก่อน
    balance_before = vault.balanceOf(user)
    
    # ทำการ Deposit
    vault.deposit(value=amount)
    
    # ตรวจสอบ State หลัง
    balance_after = vault.balanceOf(user)
    assert balance_after == balance_before + amount

def test_multiple_deposits(vault):
    """ทดสอบ Deposit หลายครั้ง"""
    user = boa.env.eoa
    deposits = [10**17, 5 * 10**17, 10**18]
    
    for amount in deposits:
        vault.deposit(value=amount)
    
    expected_total = sum(deposits)
    assert vault.balanceOf(user) == expected_total
    assert vault.totalDeposited() == expected_total
```

### ทดสอบด้วย Time Travel

```python
# tests/test_time.py
import boa
import pytest

@pytest.fixture
def time_lock():
    """Deploy TimeLockedVault"""
    return boa.load("contracts/TimeLockedVault.vy", 3600)  # 1 hour

def test_unlock_after_time(time_lock):
    """ทดสอบว่า Unlock ได้หลังครบเวลา"""
    time_lock.deposit(value=10**18)
    
    # ยังไม่ครบเวลา
    assert time_lock.isUnlocked() == False
    
    # เร่งเวลา 1 ชั่วโมง + 1 วินาที
    boa.env.time_travel(seconds=3601)
    
    assert time_lock.isUnlocked() == True

def test_time_remaining_decreases(time_lock):
    """ทดสอบว่า Time Remaining ลดลงตามเวลา"""
    initial_remaining = time_lock.timeRemaining()
    
    boa.env.time_travel(seconds=1800)  # 30 นาที
    
    new_remaining = time_lock.timeRemaining()
    assert new_remaining < initial_remaining
    assert initial_remaining - new_remaining >= 1800
```

### ทดสอบด้วย Block Manipulation

```python
# tests/test_blocks.py
import boa

def test_block_based_timing():
    """ทดสอบ Logic ที่อิง Block Number"""
    contract = boa.loads("""
# @version 0.4.0

startBlock: public(uint256)
DURATION: constant(uint256) = 100  # 100 blocks

@deploy
def __init__():
    self.startBlock = block.number

@external
@view
def isActive() -> bool:
    return block.number <= self.startBlock + DURATION
    """)
    
    assert contract.isActive() == True
    
    # เร่งไป 50 Blocks
    boa.env.mine(50)
    assert contract.isActive() == True
    
    # เร่งอีก 60 Blocks (รวม 110)
    boa.env.mine(60)
    assert contract.isActive() == False
```

### ทดสอบด้วย Impersonation (Prank)

```python
# tests/test_prank.py
import boa
import pytest

@pytest.fixture
def token():
    return boa.load("contracts/Token.vy", "Token", "TKN", 10**24)

def test_transfer_as_different_user(token):
    """ทดสอบ Transfer ในฐานะ User อื่น"""
    deployer = boa.env.eoa
    user1 = boa.env.generate_address()
    user2 = boa.env.generate_address()
    
    # Deployer ส่งให้ User1
    token.transfer(user1, 10**20)
    
    # User1 ส่งให้ User2 (ใช้ prank)
    with boa.env.prank(user1):
        token.transfer(user2, 5 * 10**19)
    
    assert token.balanceOf(user2) == 5 * 10**19

def test_owner_only_function(token):
    """ทดสอบว่าเฉพาะ Owner ทำได้"""
    attacker = boa.env.generate_address()
    
    with boa.env.prank(attacker):
        with pytest.raises(Exception):
            token.mint(attacker, 10**20)
```

---

## 4. Testing Events {#testing-events}

### วิธีดู Events ใน Titanoboa

```python
# tests/test_events.py
import boa
import pytest

@pytest.fixture
def token():
    return boa.load("contracts/Token.vy", "Token", "TKN", 10**24)

def test_transfer_emits_event(token):
    """ทดสอบว่า Transfer Emit Event"""
    user = boa.env.generate_address()
    amount = 10**18
    
    token.transfer(user, amount)
    
    # ดู Logs ของ Transaction ล่าสุด
    logs = token.get_logs()
    assert len(logs) > 0
    
    # ตรวจสอบ Event แรก
    transfer_log = logs[0]
    # ขึ้นอยู่กับ API ของ Titanoboa

def test_events_with_receipt(token):
    """ทดสอบ Events ผ่าน Receipt"""
    user = boa.env.generate_address()
    amount = 10**18
    
    # เก็บ Return Value (Receipt)
    with token.as_receiver():
        token.transfer(user, amount)
    
    # ตรวจสอบ Events
    events = token.get_logs()
    print("Events:", events)
```

### Contract สำหรับทดสอบ Events

```vyper
# contracts/EventContract.vy
# @version 0.4.0

event ValueSet:
    setter: indexed(address)
    oldValue: uint256
    newValue: uint256

event Transfer:
    from_: indexed(address)
    to: indexed(address)
    amount: uint256

value: public(uint256)
balances: HashMap[address, uint256]

@deploy
def __init__():
    self.value = 0

@external
def setValue(newVal: uint256):
    oldVal: uint256 = self.value
    self.value = newVal
    log ValueSet(msg.sender, oldVal, newVal)

@external
@payable
def deposit():
    self.balances[msg.sender] += msg.value
    log Transfer(empty(address), msg.sender, msg.value)
```

```python
# tests/test_event_contract.py
import boa
import pytest

@pytest.fixture
def contract():
    return boa.load("contracts/EventContract.vy")

def test_value_set_event(contract):
    """ทดสอบ ValueSet Event"""
    contract.setValue(42)
    
    # ดู Events
    logs = contract.get_logs()
    assert len(logs) >= 1

def test_multiple_events(contract):
    """ทดสอบ Events หลายครั้ง"""
    values = [10, 20, 30, 40, 50]
    
    for v in values:
        contract.setValue(v)
    
    # ตรวจสอบค่าสุดท้าย
    assert contract.value() == 50
```

---

## 5. Testing Reverts {#testing-reverts}

### ทดสอบว่า Function Revert

```python
# tests/test_reverts.py
import boa
import pytest

@pytest.fixture
def token():
    return boa.load("contracts/Token.vy", "Token", "TKN", 10**24)

def test_revert_on_insufficient_balance(token):
    """ทดสอบว่า Revert เมื่อ Balance ไม่พอ"""
    user = boa.env.generate_address()
    
    with boa.env.prank(user):
        with pytest.raises(Exception):
            token.transfer(boa.env.eoa, 1)

def test_revert_with_message(token):
    """ทดสอบ Revert พร้อม Error Message"""
    user = boa.env.generate_address()
    
    with boa.env.prank(user):
        with pytest.raises(Exception, match="Insufficient balance"):
            token.transfer(boa.env.eoa, 1)

def test_revert_zero_transfer(token):
    """ทดสอบ Revert เมื่อ Transfer 0"""
    user = boa.env.generate_address()
    
    with pytest.raises(Exception, match="Zero amount"):
        token.transfer(user, 0)

def test_no_revert_valid_transfer(token):
    """ทดสอบว่าไม่ Revert เมื่อ Transfer ถูกต้อง"""
    user = boa.env.generate_address()
    amount = 10**18
    
    # ไม่ควร Raise Exception
    token.transfer(user, amount)
    assert token.balanceOf(user) == amount

def test_revert_restores_state(token):
    """ทดสอบว่า Revert คืน State กลับ"""
    deployer = boa.env.eoa
    user = boa.env.generate_address()
    
    initial_balance = token.balanceOf(deployer)
    
    # ลอง Transfer เงินเกิน Balance
    with pytest.raises(Exception):
        token.transfer(user, initial_balance + 1)
    
    # State ต้องไม่เปลี่ยน
    assert token.balanceOf(deployer) == initial_balance
    assert token.balanceOf(user) == 0
```

### Testing Complex Revert Scenarios

```python
# tests/test_complex_reverts.py
import boa
import pytest

@pytest.fixture
def vault():
    return boa.load("contracts/TimeLockedVault.vy", 3600)

def test_cannot_withdraw_before_unlock(vault):
    """ทดสอบว่าถอนก่อนครบเวลาไม่ได้"""
    vault.deposit(value=10**18)
    
    with pytest.raises(Exception, match="Still locked"):
        vault.withdraw()

def test_cannot_deposit_zero(vault):
    """ทดสอบว่าฝาก 0 ไม่ได้"""
    with pytest.raises(Exception, match="Must send ETH"):
        vault.deposit(value=0)

def test_can_withdraw_after_time(vault):
    """ทดสอบว่าถอนได้หลังครบเวลา"""
    vault.deposit(value=10**18)
    boa.env.time_travel(seconds=3601)
    
    # ไม่ควร Raise Exception
    vault.withdraw()
```

---

## 6. Coverage {#coverage}

### การวัด Code Coverage

```bash
# รัน Tests พร้อม Coverage
pytest --cov=contracts --cov-report=html --cov-report=term

# ดู Coverage Report
# ใน Terminal
# และ htmlcov/index.html ในรูปแบบ HTML
```

### pytest.ini สำหรับ Coverage

```ini
[pytest]
testpaths = tests
addopts = --cov=contracts --cov-report=term-missing --cov-fail-under=80
```

### ตัวอย่าง Coverage Report

```
Name                        Stmts   Miss  Cover
-----------------------------------------------
contracts/Token.vy             87     12    86%
contracts/Vault.vy             45      5    89%
contracts/NFT.vy              123     34    72%
-----------------------------------------------
TOTAL                         255     51    80%
```

---

## 7. ตัวอย่าง: Complete Test Suite {#complete-test-suite}

Contract ที่จะทดสอบ:

```vyper
# contracts/SimpleBank.vy
# @version 0.4.0
"""
@title Simple Bank
@notice ธนาคารอย่างง่ายสำหรับทดสอบ
"""

event Deposited:
    user: indexed(address)
    amount: uint256

event Withdrawn:
    user: indexed(address)
    amount: uint256

event OwnerChanged:
    oldOwner: indexed(address)
    newOwner: indexed(address)

owner: public(address)
balances: public(HashMap[address, uint256])
totalDeposited: public(uint256)
paused: public(bool)

MIN_DEPOSIT: constant(uint256) = 10**15  # 0.001 ETH
MAX_DEPOSIT: constant(uint256) = 100 * 10**18  # 100 ETH

@deploy
def __init__():
    self.owner = msg.sender
    self.paused = False

@external
@payable
def deposit():
    assert not self.paused, "Bank: paused"
    assert msg.value >= MIN_DEPOSIT, "Bank: below minimum deposit"
    assert msg.value <= MAX_DEPOSIT, "Bank: above maximum deposit"
    
    self.balances[msg.sender] += msg.value
    self.totalDeposited += msg.value
    
    log Deposited(msg.sender, msg.value)

@external
def withdraw(amount: uint256):
    assert not self.paused, "Bank: paused"
    assert amount > 0, "Bank: zero amount"
    assert self.balances[msg.sender] >= amount, "Bank: insufficient balance"
    
    self.balances[msg.sender] -= amount
    self.totalDeposited -= amount
    
    log Withdrawn(msg.sender, amount)
    send(msg.sender, amount)

@external
def withdrawAll():
    amount: uint256 = self.balances[msg.sender]
    assert amount > 0, "Bank: no balance"
    
    self.balances[msg.sender] = 0
    self.totalDeposited -= amount
    
    log Withdrawn(msg.sender, amount)
    send(msg.sender, amount)

@external
def pause():
    assert msg.sender == self.owner, "Bank: not owner"
    assert not self.paused, "Bank: already paused"
    self.paused = True

@external
def unpause():
    assert msg.sender == self.owner, "Bank: not owner"
    assert self.paused, "Bank: not paused"
    self.paused = False

@external
def transferOwnership(newOwner: address):
    assert msg.sender == self.owner, "Bank: not owner"
    assert newOwner != empty(address), "Bank: zero address"
    
    oldOwner: address = self.owner
    self.owner = newOwner
    
    log OwnerChanged(oldOwner, newOwner)

@external
@view
def getBalance(user: address) -> uint256:
    return self.balances[user]
```

### Complete Test Suite

```python
# tests/test_simple_bank.py
"""
Complete Test Suite สำหรับ SimpleBank Contract
"""
import pytest
import boa
from eth_utils import to_wei

# ==================== Fixtures ====================

@pytest.fixture
def deployer():
    """Deployer address"""
    return boa.env.eoa

@pytest.fixture
def user1():
    """User 1 address"""
    addr = boa.env.generate_address()
    boa.env.set_balance(addr, to_wei(10, 'ether'))
    return addr

@pytest.fixture
def user2():
    """User 2 address"""
    addr = boa.env.generate_address()
    boa.env.set_balance(addr, to_wei(10, 'ether'))
    return addr

@pytest.fixture
def attacker():
    """Attacker address"""
    addr = boa.env.generate_address()
    boa.env.set_balance(addr, to_wei(10, 'ether'))
    return addr

@pytest.fixture
def bank():
    """Deploy fresh SimpleBank"""
    return boa.load("contracts/SimpleBank.vy")

@pytest.fixture
def bank_with_deposits(bank, user1, user2):
    """Bank ที่มี Deposits แล้ว"""
    with boa.env.prank(user1):
        bank.deposit(value=to_wei(1, 'ether'))
    with boa.env.prank(user2):
        bank.deposit(value=to_wei(2, 'ether'))
    return bank

# ==================== Deployment Tests ====================

class TestDeployment:
    def test_initial_owner(self, bank, deployer):
        """ทดสอบว่า Owner ถูก Set ตอน Deploy"""
        assert bank.owner() == deployer

    def test_initial_state(self, bank):
        """ทดสอบ State เริ่มต้น"""
        assert bank.totalDeposited() == 0
        assert bank.paused() == False

    def test_initial_balances_zero(self, bank, user1, user2):
        """ทดสอบว่า Balance เริ่มต้นเป็น 0"""
        assert bank.balances(user1) == 0
        assert bank.balances(user2) == 0

# ==================== Deposit Tests ====================

class TestDeposit:
    def test_basic_deposit(self, bank, user1):
        """ทดสอบการฝากขั้นพื้นฐาน"""
        amount = to_wei(1, 'ether')
        
        with boa.env.prank(user1):
            bank.deposit(value=amount)
        
        assert bank.balances(user1) == amount
        assert bank.totalDeposited() == amount

    def test_deposit_below_minimum_fails(self, bank, user1):
        """ทดสอบว่าฝากต่ำกว่า Minimum ไม่ได้"""
        with boa.env.prank(user1):
            with pytest.raises(Exception, match="below minimum deposit"):
                bank.deposit(value=10**14)  # น้อยกว่า 0.001 ETH

    def test_deposit_above_maximum_fails(self, bank, user1):
        """ทดสอบว่าฝากเกิน Maximum ไม่ได้"""
        with boa.env.prank(user1):
            with pytest.raises(Exception, match="above maximum deposit"):
                bank.deposit(value=to_wei(101, 'ether'))

    def test_multiple_deposits(self, bank, user1):
        """ทดสอบการฝากหลายครั้ง"""
        amounts = [to_wei(0.5, 'ether'), to_wei(1, 'ether'), to_wei(1.5, 'ether')]
        
        for amount in amounts:
            with boa.env.prank(user1):
                bank.deposit(value=amount)
        
        assert bank.balances(user1) == sum(amounts)

    def test_deposit_when_paused_fails(self, bank, user1):
        """ทดสอบว่าฝากไม่ได้เมื่อ Paused"""
        bank.pause()
        
        with boa.env.prank(user1):
            with pytest.raises(Exception, match="paused"):
                bank.deposit(value=to_wei(1, 'ether'))

    def test_deposit_emits_event(self, bank, user1):
        """ทดสอบว่า Deposit Emit Event"""
        amount = to_wei(1, 'ether')
        
        with boa.env.prank(user1):
            bank.deposit(value=amount)
        
        logs = bank.get_logs()
        assert len(logs) > 0

# ==================== Withdrawal Tests ====================

class TestWithdrawal:
    def test_basic_withdrawal(self, bank, user1):
        """ทดสอบการถอนขั้นพื้นฐาน"""
        deposit_amount = to_wei(2, 'ether')
        withdraw_amount = to_wei(1, 'ether')
        
        with boa.env.prank(user1):
            bank.deposit(value=deposit_amount)
            
            initial_balance = boa.env.get_balance(user1)
            bank.withdraw(withdraw_amount)
            final_balance = boa.env.get_balance(user1)
        
        assert bank.balances(user1) == deposit_amount - withdraw_amount
        assert final_balance > initial_balance  # ได้รับ ETH กลับ

    def test_withdraw_more_than_balance_fails(self, bank, user1):
        """ทดสอบว่าถอนเกิน Balance ไม่ได้"""
        deposit_amount = to_wei(1, 'ether')
        
        with boa.env.prank(user1):
            bank.deposit(value=deposit_amount)
            
            with pytest.raises(Exception, match="insufficient balance"):
                bank.withdraw(deposit_amount + 1)

    def test_withdraw_zero_fails(self, bank, user1):
        """ทดสอบว่าถอน 0 ไม่ได้"""
        with boa.env.prank(user1):
            bank.deposit(value=to_wei(1, 'ether'))
            
            with pytest.raises(Exception, match="zero amount"):
                bank.withdraw(0)

    def test_withdraw_all(self, bank, user1):
        """ทดสอบการถอนทั้งหมด"""
        deposit_amount = to_wei(1, 'ether')
        
        with boa.env.prank(user1):
            bank.deposit(value=deposit_amount)
            bank.withdrawAll()
        
        assert bank.balances(user1) == 0

    def test_withdraw_with_no_balance_fails(self, bank, user1):
        """ทดสอบว่าถอนโดยไม่มีเงินฝากไม่ได้"""
        with boa.env.prank(user1):
            with pytest.raises(Exception, match="no balance"):
                bank.withdrawAll()

    def test_multiple_users_independent(self, bank_with_deposits, user1, user2):
        """ทดสอบว่า Balance ของแต่ละ User เป็น Independent"""
        assert bank_with_deposits.balances(user1) == to_wei(1, 'ether')
        assert bank_with_deposits.balances(user2) == to_wei(2, 'ether')
        
        # User1 ถอน ไม่ควรกระทบ User2
        with boa.env.prank(user1):
            bank_with_deposits.withdrawAll()
        
        assert bank_with_deposits.balances(user1) == 0
        assert bank_with_deposits.balances(user2) == to_wei(2, 'ether')

# ==================== Pause Tests ====================

class TestPause:
    def test_owner_can_pause(self, bank, deployer):
        """ทดสอบว่า Owner Pause ได้"""
        assert bank.paused() == False
        bank.pause()
        assert bank.paused() == True

    def test_non_owner_cannot_pause(self, bank, attacker):
        """ทดสอบว่า Non-owner Pause ไม่ได้"""
        with boa.env.prank(attacker):
            with pytest.raises(Exception, match="not owner"):
                bank.pause()

    def test_cannot_pause_twice(self, bank):
        """ทดสอบว่า Pause ซ้ำไม่ได้"""
        bank.pause()
        with pytest.raises(Exception, match="already paused"):
            bank.pause()

    def test_owner_can_unpause(self, bank):
        """ทดสอบว่า Owner Unpause ได้"""
        bank.pause()
        bank.unpause()
        assert bank.paused() == False

    def test_operations_blocked_when_paused(self, bank, user1, user2):
        """ทดสอบว่า Operations ถูก Block เมื่อ Paused"""
        # Deposit ก่อน
        with boa.env.prank(user1):
            bank.deposit(value=to_wei(1, 'ether'))
        
        # Pause
        bank.pause()
        
        # ทดสอบว่า Deposit ไม่ได้
        with boa.env.prank(user2):
            with pytest.raises(Exception, match="paused"):
                bank.deposit(value=to_wei(1, 'ether'))
        
        # ทดสอบว่า Withdraw ไม่ได้
        with boa.env.prank(user1):
            with pytest.raises(Exception, match="paused"):
                bank.withdrawAll()

# ==================== Ownership Tests ====================

class TestOwnership:
    def test_transfer_ownership(self, bank, deployer, user1):
        """ทดสอบการโอน Ownership"""
        bank.transferOwnership(user1)
        assert bank.owner() == user1

    def test_non_owner_cannot_transfer(self, bank, attacker):
        """ทดสอบว่า Non-owner โอน Ownership ไม่ได้"""
        with boa.env.prank(attacker):
            with pytest.raises(Exception, match="not owner"):
                bank.transferOwnership(attacker)

    def test_transfer_to_zero_fails(self, bank):
        """ทดสอบว่าโอนไปยัง Zero Address ไม่ได้"""
        with pytest.raises(Exception, match="zero address"):
            bank.transferOwnership("0x0000000000000000000000000000000000000000")

    def test_ownership_event(self, bank, deployer, user1):
        """ทดสอบว่า OwnerChanged Event ถูก Emit"""
        bank.transferOwnership(user1)
        
        logs = bank.get_logs()
        assert len(logs) > 0

# ==================== Integration Tests ====================

class TestIntegration:
    def test_full_lifecycle(self, bank, user1, user2):
        """ทดสอบ Life Cycle ทั้งหมด"""
        # User1 Deposit
        with boa.env.prank(user1):
            bank.deposit(value=to_wei(5, 'ether'))
        
        # User2 Deposit
        with boa.env.prank(user2):
            bank.deposit(value=to_wei(3, 'ether'))
        
        assert bank.totalDeposited() == to_wei(8, 'ether')
        
        # User1 Withdraw บางส่วน
        with boa.env.prank(user1):
            bank.withdraw(to_wei(2, 'ether'))
        
        assert bank.balances(user1) == to_wei(3, 'ether')
        assert bank.totalDeposited() == to_wei(6, 'ether')
        
        # User2 Withdraw ทั้งหมด
        with boa.env.prank(user2):
            bank.withdrawAll()
        
        assert bank.balances(user2) == 0
        assert bank.totalDeposited() == to_wei(3, 'ether')

    def test_revert_does_not_change_state(self, bank, user1):
        """ทดสอบว่า Revert ไม่เปลี่ยน State"""
        with boa.env.prank(user1):
            bank.deposit(value=to_wei(1, 'ether'))
        
        initial_balance = bank.balances(user1)
        initial_total = bank.totalDeposited()
        
        # พยายามถอนเงินเกิน
        with boa.env.prank(user1):
            with pytest.raises(Exception):
                bank.withdraw(initial_balance + to_wei(1, 'ether'))
        
        # State ต้องไม่เปลี่ยน
        assert bank.balances(user1) == initial_balance
        assert bank.totalDeposited() == initial_total
```

---

## 8. แบบฝึกหัด {#exercises}

### แบบฝึกหัดที่ 1: Test Coverage 100%
เขียน Test Suite ให้ครอบคลุม Contract ที่สร้างใน Part 016-019 ทุกบรรทัด

### แบบฝึกหัดที่ 2: Property-Based Testing
ใช้ Hypothesis Library เพื่อสร้าง Fuzz Tests

### แบบฝึกหัดที่ 3: Integration Testing
เขียน Test ที่ทดสอบการทำงานร่วมกันของ Contract หลายตัว

---

## สรุป

| Concept | Command/Code | หมายเหตุ |
|---------|-------------|----------|
| Deploy Contract | `boa.load("file.vy", args...)` | |
| Deploy from String | `boa.loads(source, args...)` | |
| Check State | `contract.variable()` | |
| Call Function | `contract.function(args)` | |
| Send ETH | `contract.function(value=amount)` | |
| Impersonate | `with boa.env.prank(addr):` | |
| Time Travel | `boa.env.time_travel(seconds=n)` | |
| Mine Blocks | `boa.env.mine(n)` | |
| Test Revert | `pytest.raises(Exception, match="...")` | |
| Get Logs | `contract.get_logs()` | |
| Run Coverage | `pytest --cov=contracts` | |

---

[← Part 019: Interface Basics](part_019_interfaces.md) | [Part 021: ERC-20 Standard →](part_021_erc20_standard.md)
