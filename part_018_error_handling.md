# Part 018: Error Handling: assert และ raise

## สารบัญ
1. [assert Statement](#assert)
2. [raise Statement](#raise)
3. [UNREACHABLE](#unreachable)
4. [Error Patterns](#error-patterns)
5. [Custom Error Messages](#custom-errors)
6. [Gas Refund on Revert](#gas-refund)
7. [ตัวอย่าง: Validated Token Transfer](#validated-token)
8. [แบบฝึกหัด](#exercises)

---

## 1. assert Statement {#assert}

`assert` ตรวจสอบเงื่อนไขและ Revert ถ้าเงื่อนไขเป็น False

### Syntax พื้นฐาน

```vyper
# @version 0.4.0

@external
def assertExamples(value: uint256):
    # assert เงื่อนไขพื้นฐาน
    assert value > 0, "Value must be positive"
    
    # assert โดยไม่มี Message
    assert value < 1000
    
    # assert หลายเงื่อนไข
    assert value > 10, "Too small"
    assert value < 100, "Too large"
    assert value % 2 == 0, "Must be even"
```

### assert กับ Expression ซับซ้อน

```vyper
# @version 0.4.0

owner: address
paused: bool
balances: HashMap[address, uint256]

@deploy
def __init__():
    self.owner = msg.sender

@external
def transfer(to: address, amount: uint256):
    # ตรวจสอบหลายเงื่อนไข
    assert not self.paused, "Contract is paused"
    assert to != empty(address), "Invalid recipient"
    assert to != msg.sender, "Cannot transfer to self"
    assert amount > 0, "Zero amount"
    assert self.balances[msg.sender] >= amount, "Insufficient balance"
    
    self.balances[msg.sender] -= amount
    self.balances[to] += amount

@external
def onlyOwner():
    assert msg.sender == self.owner, "Caller is not the owner"
    # ...
```

### assert กับ Arithmetic

```vyper
# @version 0.4.0

@external
def safeMath(a: uint256, b: uint256) -> uint256:
    # Vyper 0.4.0 มี Built-in Overflow Protection
    # แต่ยังต้อง assert เงื่อนไข Logic
    
    # ป้องกัน Division by zero
    assert b != 0, "Division by zero"
    
    result: uint256 = a / b
    assert result > 0, "Result too small"
    
    return result

@external
def boundedAdd(a: uint256, b: uint256, maxValue: uint256) -> uint256:
    result: uint256 = a + b
    assert result <= maxValue, "Result exceeds maximum"
    return result
```

---

## 2. raise Statement {#raise}

`raise` ทำให้ Transaction Revert พร้อม Error Message

### Syntax ของ raise

```vyper
# @version 0.4.0

@external
def raiseExamples(choice: uint256):
    if choice == 1:
        raise "Error with message"
    elif choice == 2:
        raise  # Revert โดยไม่มี Message (Empty Revert)
    elif choice == 3:
        # raise กับ String แบบ Dynamic
        msg: String[100] = "Dynamic error"
        raise msg
```

### ความแตกต่าง assert vs raise

```vyper
# @version 0.4.0

@external
def compareAssertRaise(condition: bool, value: uint256):
    # assert: ตรวจสอบเงื่อนไข → raise ถ้า False
    # เหมาะกับ: Validation, Preconditions
    assert condition, "Condition failed"
    
    # raise: Revert ทันที (ไม่มีเงื่อนไข)
    # เหมาะกับ: Business Logic Errors, Unreachable Code
    if value == 0:
        raise "Value cannot be zero"
    
    if value > 1000000:
        raise "Value too large"
```

### raise ใน If-Else

```vyper
# @version 0.4.0

enum OrderStatus:
    PENDING
    CONFIRMED
    SHIPPED
    DELIVERED
    CANCELLED

orders: HashMap[uint256, OrderStatus]

@external
def cancelOrder(orderId: uint256):
    status: OrderStatus = self.orders[orderId]
    
    if status == OrderStatus.DELIVERED:
        raise "Cannot cancel: Order already delivered"
    elif status == OrderStatus.SHIPPED:
        raise "Cannot cancel: Order already shipped"
    elif status == OrderStatus.CANCELLED:
        raise "Order already cancelled"
    
    # ถ้าผ่านมาถึงนี่ได้ = PENDING หรือ CONFIRMED
    self.orders[orderId] = OrderStatus.CANCELLED
```

---

## 3. UNREACHABLE {#unreachable}

`UNREACHABLE` ใช้บอก Compiler ว่า Code ส่วนนั้นไม่ควรถูก Execute เพื่อ Optimize Gas

```vyper
# @version 0.4.0

@external
def unreachableExample(x: uint256) -> uint256:
    if x < 10:
        return x * 2
    elif x < 100:
        return x + 50
    elif x < 1000:
        return x - 10
    else:
        # ถ้า Logic ถูกต้อง ไม่ควรมาถึงนี่
        # แต่ Compiler ต้องการ return
        raise "UNREACHABLE"

@external
def processCategory(category: uint8) -> String[50]:
    # category อยู่ในช่วง 1-3 เท่านั้น (ถูก Validate ก่อนหน้า)
    if category == 1:
        return "Category A"
    elif category == 2:
        return "Category B"
    elif category == 3:
        return "Category C"
    else:
        raise "Invalid category"  # ควรไม่เกิด
```

---

## 4. Error Patterns {#error-patterns}

### 4.1 Guard Pattern

```vyper
# @version 0.4.0

owner: address
authorized: HashMap[address, bool]
paused: bool

@deploy
def __init__():
    self.owner = msg.sender
    self.authorized[msg.sender] = True

# ===== Guards (Internal Checks) =====

@internal
def _onlyOwner():
    assert msg.sender == self.owner, "Ownable: caller is not the owner"

@internal
def _onlyAuthorized():
    assert self.authorized[msg.sender], "Not authorized"

@internal
def _whenNotPaused():
    assert not self.paused, "Pausable: paused"

@internal
def _whenPaused():
    assert self.paused, "Pausable: not paused"

# ===== Admin Functions =====

@external
def pause():
    self._onlyOwner()
    self._whenNotPaused()
    self.paused = True

@external
def unpause():
    self._onlyOwner()
    self._whenPaused()
    self.paused = False

@external
def addAuthorized(addr: address):
    self._onlyOwner()
    assert addr != empty(address), "Invalid address"
    self.authorized[addr] = True

# ===== User Functions =====

@external
def doSomething():
    self._whenNotPaused()
    self._onlyAuthorized()
    # ทำงาน...
```

### 4.2 Validation Pattern

```vyper
# @version 0.4.0

@internal
def _validateAmount(amount: uint256):
    assert amount > 0, "Amount must be positive"
    assert amount <= 10**24, "Amount too large"  # Max 1M ETH

@internal
def _validateAddress(addr: address):
    assert addr != empty(address), "Zero address not allowed"
    assert addr != self, "Contract address not allowed"

@internal
def _validatePercentage(bps: uint256):
    assert bps <= 10000, "Percentage exceeds 100%"

@external
def validateAll(
    to: address,
    amount: uint256,
    feeBps: uint256
):
    self._validateAddress(to)
    self._validateAmount(amount)
    self._validatePercentage(feeBps)
    
    # ดำเนินการต่อ...
```

### 4.3 State Machine Pattern

```vyper
# @version 0.4.0

enum State:
    INACTIVE
    ACTIVE
    PENDING
    COMPLETED

currentState: State

@deploy
def __init__():
    self.currentState = State.INACTIVE

@internal
def _requireState(expected: State):
    if self.currentState != expected:
        raise "Invalid state for this operation"

@external
def activate():
    self._requireState(State.INACTIVE)
    self.currentState = State.ACTIVE

@external
def startProcess():
    self._requireState(State.ACTIVE)
    self.currentState = State.PENDING

@external
def complete():
    self._requireState(State.PENDING)
    self.currentState = State.COMPLETED
```

### 4.4 Access Control Pattern

```vyper
# @version 0.4.0

# Role-based Access Control
ADMIN_ROLE: constant(bytes32) = keccak256("ADMIN")
MINTER_ROLE: constant(bytes32) = keccak256("MINTER")
PAUSER_ROLE: constant(bytes32) = keccak256("PAUSER")

roles: HashMap[bytes32, HashMap[address, bool]]
owner: address

@deploy
def __init__():
    self.owner = msg.sender
    # Grant Admin role to deployer
    self.roles[ADMIN_ROLE][msg.sender] = True

@internal
def _checkRole(role: bytes32, account: address):
    assert self.roles[role][account], "AccessControl: missing role"

@external
def grantRole(role: bytes32, account: address):
    self._checkRole(ADMIN_ROLE, msg.sender)
    assert account != empty(address), "Invalid account"
    self.roles[role][account] = True

@external
def revokeRole(role: bytes32, account: address):
    self._checkRole(ADMIN_ROLE, msg.sender)
    self.roles[role][account] = False

@external
def hasRole(role: bytes32, account: address) -> bool:
    return self.roles[role][account]

@external
def mint(amount: uint256):
    self._checkRole(MINTER_ROLE, msg.sender)
    # mint...
```

---

## 5. Custom Error Messages {#custom-errors}

### 5.1 Error Message Best Practices

```vyper
# @version 0.4.0

@external
def goodErrorMessages(
    amount: uint256,
    recipient: address,
    deadline: uint256
):
    # ✅ ดี: ระบุสิ่งที่ผิดพลาดชัดเจน
    assert amount > 0, "Amount must be greater than zero"
    assert recipient != empty(address), "Recipient cannot be zero address"
    assert block.timestamp <= deadline, "Transaction deadline expired"
    
    # ✅ ดี: ใช้ชื่อ Contract/Module
    assert msg.sender == self.owner(), "Ownable: caller is not the owner"
    
    # ❌ ไม่ดี: ไม่ชัดเจน
    # assert amount > 0, "Error"
    # assert recipient != empty(address), "Bad"

@internal
@view
def owner() -> address:
    return msg.sender  # Simplified
```

### 5.2 Contextual Error Messages

```vyper
# @version 0.4.0

MAX_SUPPLY: constant(uint256) = 1000000
totalSupply: public(uint256)
balances: HashMap[address, uint256]

@external
def transfer(to: address, amount: uint256):
    # บอก Context ว่าผิดพลาดที่ไหน
    assert to != empty(address), "ERC20: transfer to zero address"
    assert amount > 0, "ERC20: transfer amount must be greater than zero"
    assert self.balances[msg.sender] >= amount, "ERC20: transfer amount exceeds balance"
    
    self.balances[msg.sender] -= amount
    self.balances[to] += amount

@external
def mint(to: address, amount: uint256):
    assert to != empty(address), "ERC20: mint to zero address"
    assert amount > 0, "ERC20: mint amount must be greater than zero"
    assert self.totalSupply + amount <= MAX_SUPPLY, "ERC20: cap exceeded"
    
    self.totalSupply += amount
    self.balances[to] += amount
```

---

## 6. Gas Refund on Revert {#gas-refund}

เมื่อ Transaction Revert Gas ที่ยังไม่ได้ใช้จะถูกคืน

```vyper
# @version 0.4.0

expensiveStorage: uint256[1000]

@external
def expensiveOperation(shouldRevert: bool):
    """
    @notice ถ้า Revert เกิดขึ้น Gas ที่ยังไม่ใช้จะถูกคืน
    @dev Gas ที่ใช้ไปแล้วก่อน Revert จะไม่ได้คืน
    """
    # ใช้ Gas ไปส่วนหนึ่ง
    for i: uint256 in range(100):
        self.expensiveStorage[i] = i
    
    # ถ้า Revert ที่นี่:
    # - Gas ที่ใช้ไปใน Loop ข้างบน = ไม่ได้คืน
    # - Gas ที่เหลือ = ได้คืน
    if shouldRevert:
        raise "Intentional revert"
    
    # ทำงานต่อ...
    for i: uint256 in range(100, 1000):
        self.expensiveStorage[i] = i * 2
```

### Fail-Fast Pattern - ใส่ Checks ไว้ต้น Function

```vyper
# @version 0.4.0

@external
def efficientFunction(
    to: address,
    amount: uint256,
    deadline: uint256
):
    """
    @notice ใส่ Checks ที่ไม่ใช้ Gas มากไว้ก่อนเสมอ
    @dev ทำให้ Fail เร็วขึ้น และคืน Gas มากขึ้น
    """
    # ✅ Cheap Checks ก่อน
    assert to != empty(address), "Zero address"
    assert amount > 0, "Zero amount"
    assert block.timestamp <= deadline, "Deadline expired"
    
    # 💰 Expensive Operations ทีหลัง
    # (Storage reads, external calls, loops)
    assert self.balances[msg.sender] >= amount, "Insufficient balance"
    
    self.balances[msg.sender] -= amount
    self.balances[to] += amount

balances: HashMap[address, uint256]
```

---

## 7. ตัวอย่าง: Validated Token Transfer {#validated-token}

```vyper
# @version 0.4.0
"""
@title Validated Token Transfer
@notice ERC-20-like token ที่มีการ Validation ครบถ้วน
@dev แสดงการใช้ assert และ raise ในทุกสถานการณ์
"""

# Events
event Transfer:
    sender: indexed(address)
    recipient: indexed(address)
    amount: uint256

event Approval:
    owner: indexed(address)
    spender: indexed(address)
    amount: uint256

event Mint:
    to: indexed(address)
    amount: uint256

event Burn:
    from_: indexed(address)
    amount: uint256

# State Variables
name: public(String[64])
symbol: public(String[32])
decimals: public(uint8)
totalSupply: public(uint256)

balances: HashMap[address, uint256]
allowances: HashMap[address, HashMap[address, uint256]]

owner: public(address)
minters: HashMap[address, bool]
paused: public(bool)
blacklisted: HashMap[address, bool]

# Constants
MAX_SUPPLY: constant(uint256) = 10**27  # 1 Billion tokens with 18 decimals
MAX_TRANSFER: constant(uint256) = 10**25  # Max 10M per transfer

@deploy
def __init__(
    _name: String[64],
    _symbol: String[32],
    initialSupply: uint256
):
    assert len(_name) > 0, "Token: name cannot be empty"
    assert len(_symbol) > 0, "Token: symbol cannot be empty"
    assert initialSupply <= MAX_SUPPLY, "Token: initial supply exceeds maximum"
    
    self.name = _name
    self.symbol = _symbol
    self.decimals = 18
    self.owner = msg.sender
    self.minters[msg.sender] = True
    
    if initialSupply > 0:
        self.totalSupply = initialSupply
        self.balances[msg.sender] = initialSupply
        log Mint(msg.sender, initialSupply)

# ==================== Internal Guards ====================

@internal
def _onlyOwner():
    assert msg.sender == self.owner, "Token: caller is not the owner"

@internal
def _onlyMinter():
    assert self.minters[msg.sender], "Token: caller is not a minter"

@internal
def _whenNotPaused():
    assert not self.paused, "Token: token transfer while paused"

@internal
def _validateAddress(addr: address, context: String[50]):
    if addr == empty(address):
        raise concat("Token: ", context, " is zero address")

@internal
def _notBlacklisted(addr: address):
    assert not self.blacklisted[addr], "Token: address is blacklisted"

# ==================== View Functions ====================

@external
@view
def balanceOf(account: address) -> uint256:
    assert account != empty(address), "Token: balance query for zero address"
    return self.balances[account]

@external
@view
def allowance(tokenOwner: address, spender: address) -> uint256:
    assert tokenOwner != empty(address), "Token: owner is zero address"
    assert spender != empty(address), "Token: spender is zero address"
    return self.allowances[tokenOwner][spender]

# ==================== Transfer Functions ====================

@external
def transfer(to: address, amount: uint256) -> bool:
    """
    @notice โอน Token ไปยัง address อื่น
    """
    # Guards
    self._whenNotPaused()
    self._notBlacklisted(msg.sender)
    self._notBlacklisted(to)
    
    # Validations
    assert to != empty(address), "Token: transfer to zero address"
    assert to != msg.sender, "Token: transfer to self"
    assert amount > 0, "Token: transfer amount is zero"
    assert amount <= MAX_TRANSFER, "Token: transfer amount exceeds limit"
    assert self.balances[msg.sender] >= amount, "Token: transfer amount exceeds balance"
    
    # State Changes
    self.balances[msg.sender] -= amount
    self.balances[to] += amount
    
    log Transfer(msg.sender, to, amount)
    return True

@external
def transferFrom(sender: address, to: address, amount: uint256) -> bool:
    """
    @notice โอน Token แทนผู้อื่น (ต้องมี allowance)
    """
    # Guards
    self._whenNotPaused()
    self._notBlacklisted(sender)
    self._notBlacklisted(to)
    self._notBlacklisted(msg.sender)
    
    # Validations
    assert sender != empty(address), "Token: transfer from zero address"
    assert to != empty(address), "Token: transfer to zero address"
    assert amount > 0, "Token: transfer amount is zero"
    assert amount <= MAX_TRANSFER, "Token: transfer amount exceeds limit"
    
    # Check balances and allowance
    assert self.balances[sender] >= amount, "Token: transfer amount exceeds balance"
    
    currentAllowance: uint256 = self.allowances[sender][msg.sender]
    if currentAllowance != max_value(uint256):  # Infinite approval
        assert currentAllowance >= amount, "Token: transfer amount exceeds allowance"
        self.allowances[sender][msg.sender] = currentAllowance - amount
    
    # State Changes
    self.balances[sender] -= amount
    self.balances[to] += amount
    
    log Transfer(sender, to, amount)
    return True

@external
def approve(spender: address, amount: uint256) -> bool:
    """
    @notice อนุญาตให้ spender ใช้ Token แทน
    """
    assert spender != empty(address), "Token: approve to zero address"
    assert spender != msg.sender, "Token: approve to self"
    
    self.allowances[msg.sender][spender] = amount
    log Approval(msg.sender, spender, amount)
    return True

@external
def increaseAllowance(spender: address, addedValue: uint256) -> bool:
    """
    @notice เพิ่ม Allowance เพื่อป้องกัน Approval Race Condition
    """
    assert spender != empty(address), "Token: approve to zero address"
    assert addedValue > 0, "Token: zero addition"
    
    newAllowance: uint256 = self.allowances[msg.sender][spender] + addedValue
    self.allowances[msg.sender][spender] = newAllowance
    
    log Approval(msg.sender, spender, newAllowance)
    return True

@external
def decreaseAllowance(spender: address, subtractedValue: uint256) -> bool:
    """
    @notice ลด Allowance
    """
    assert spender != empty(address), "Token: approve to zero address"
    
    currentAllowance: uint256 = self.allowances[msg.sender][spender]
    assert currentAllowance >= subtractedValue, "Token: decreased allowance below zero"
    
    self.allowances[msg.sender][spender] = currentAllowance - subtractedValue
    log Approval(msg.sender, spender, currentAllowance - subtractedValue)
    return True

# ==================== Admin Functions ====================

@external
def mint(to: address, amount: uint256):
    """
    @notice Mint Token ใหม่
    """
    self._onlyMinter()
    
    assert to != empty(address), "Token: mint to zero address"
    assert amount > 0, "Token: mint amount is zero"
    assert self.totalSupply + amount <= MAX_SUPPLY, "Token: supply cap exceeded"
    
    self.totalSupply += amount
    self.balances[to] += amount
    
    log Mint(to, amount)
    log Transfer(empty(address), to, amount)

@external
def burn(amount: uint256):
    """
    @notice Burn Token ของตัวเอง
    """
    assert amount > 0, "Token: burn amount is zero"
    assert self.balances[msg.sender] >= amount, "Token: burn amount exceeds balance"
    
    self.balances[msg.sender] -= amount
    self.totalSupply -= amount
    
    log Burn(msg.sender, amount)
    log Transfer(msg.sender, empty(address), amount)

@external
def pause():
    self._onlyOwner()
    assert not self.paused, "Token: already paused"
    self.paused = True

@external
def unpause():
    self._onlyOwner()
    assert self.paused, "Token: not paused"
    self.paused = False

@external
def blacklist(addr: address):
    self._onlyOwner()
    assert addr != empty(address), "Token: zero address"
    assert not self.blacklisted[addr], "Token: already blacklisted"
    self.blacklisted[addr] = True

@external
def removeFromBlacklist(addr: address):
    self._onlyOwner()
    assert self.blacklisted[addr], "Token: not blacklisted"
    self.blacklisted[addr] = False

@external
def addMinter(addr: address):
    self._onlyOwner()
    assert addr != empty(address), "Token: zero address"
    self.minters[addr] = True

@external
def removeMinter(addr: address):
    self._onlyOwner()
    assert addr != msg.sender, "Token: cannot remove yourself"
    self.minters[addr] = False
```

### Test Suite

```python
# tests/test_validated_token.py
import pytest
import boa

@pytest.fixture
def token():
    return boa.load(
        "contracts/ValidatedToken.vy",
        "Test Token",
        "TEST",
        10**24  # 1M tokens
    )

@pytest.fixture
def deployer():
    return boa.env.eoa

@pytest.fixture
def user1():
    addr = boa.env.generate_address()
    return addr

@pytest.fixture
def user2():
    addr = boa.env.generate_address()
    return addr

class TestDeploy:
    def test_initial_state(self, token, deployer):
        assert token.name() == "Test Token"
        assert token.symbol() == "TEST"
        assert token.decimals() == 18
        assert token.owner() == deployer
        assert token.totalSupply() == 10**24
        assert token.balanceOf(deployer) == 10**24

class TestTransfer:
    def test_basic_transfer(self, token, deployer, user1):
        amount = 10**20  # 100 tokens
        
        initial_deployer = token.balanceOf(deployer)
        initial_user1 = token.balanceOf(user1)
        
        token.transfer(user1, amount)
        
        assert token.balanceOf(deployer) == initial_deployer - amount
        assert token.balanceOf(user1) == initial_user1 + amount

    def test_transfer_zero_fails(self, token, user1):
        with pytest.raises(Exception, match="transfer amount is zero"):
            token.transfer(user1, 0)

    def test_transfer_to_zero_fails(self, token):
        with pytest.raises(Exception, match="transfer to zero address"):
            token.transfer("0x0000000000000000000000000000000000000000", 100)

    def test_transfer_insufficient_balance(self, token, user1, user2):
        with boa.env.prank(user1):
            with pytest.raises(Exception, match="exceeds balance"):
                token.transfer(user2, 1)

class TestApproval:
    def test_approve_and_transferFrom(self, token, deployer, user1, user2):
        amount = 10**20
        
        # Approve user1 to spend
        token.approve(user1, amount)
        assert token.allowance(deployer, user1) == amount
        
        # user1 transfers on behalf of deployer
        with boa.env.prank(user1):
            token.transferFrom(deployer, user2, amount)
        
        assert token.balanceOf(user2) == amount
        assert token.allowance(deployer, user1) == 0

    def test_transferFrom_exceeds_allowance(self, token, deployer, user1, user2):
        token.approve(user1, 100)
        
        with boa.env.prank(user1):
            with pytest.raises(Exception, match="exceeds allowance"):
                token.transferFrom(deployer, user2, 101)

class TestPause:
    def test_pause_stops_transfers(self, token, deployer, user1):
        token.pause()
        assert token.paused() == True
        
        with pytest.raises(Exception, match="paused"):
            token.transfer(user1, 100)

    def test_unpause_allows_transfers(self, token, user1):
        token.pause()
        token.unpause()
        
        token.transfer(user1, 100)  # Should succeed

class TestBlacklist:
    def test_blacklisted_cannot_transfer(self, token, user1):
        amount = 10**20
        token.transfer(user1, amount)
        token.blacklist(user1)
        
        with boa.env.prank(user1):
            with pytest.raises(Exception, match="blacklisted"):
                token.transfer(boa.env.generate_address(), 100)

class TestMintBurn:
    def test_mint(self, token, user1):
        initial_supply = token.totalSupply()
        amount = 10**20
        
        token.mint(user1, amount)
        
        assert token.totalSupply() == initial_supply + amount
        assert token.balanceOf(user1) == amount

    def test_burn(self, token, deployer):
        initial_balance = token.balanceOf(deployer)
        amount = 10**20
        
        token.burn(amount)
        
        assert token.balanceOf(deployer) == initial_balance - amount
        assert token.totalSupply() == 10**24 - amount

    def test_burn_insufficient_balance_fails(self, token, user1):
        with boa.env.prank(user1):
            with pytest.raises(Exception, match="exceeds balance"):
                token.burn(1)
```

---

## 8. แบบฝึกหัด {#exercises}

### แบบฝึกหัดที่ 1: Auction with Validation
สร้าง Auction Contract ที่มี Error Handling ครบถ้วน:
- Validate bid > current highest bid
- Validate auction is still active
- Validate bidder is not current highest bidder

### แบบฝึกหัดที่ 2: DAO Voting
สร้าง Voting Contract ที่:
- ป้องกัน Double Voting
- Validate Proposal State
- มี Quorum Requirements

### แบบฝึกหัดที่ 3: Lending Protocol
สร้าง Lending Contract ที่ Validate:
- Collateral Ratio
- Borrow Limit
- Liquidation Conditions

---

## สรุป

| Keyword | Use Case | Gas | Notes |
|---------|----------|-----|-------|
| `assert cond, "msg"` | Preconditions | ปกติ | Refund gas ที่เหลือ |
| `assert cond` | ไม่ต้อง message | ปกติ | |
| `raise "msg"` | Business errors | ปกติ | Refund gas ที่เหลือ |
| `raise` | Empty revert | น้อยสุด | |

**Revert คืน Gas ที่ยังไม่ใช้:**
```
Transaction Gas = Gas Used + Gas Refunded
                = Gas before revert + Unused gas
```

---

[← Part 017: Address Types](part_017_address.md) | [Part 019: Interface Basics →](part_019_interfaces.md)
