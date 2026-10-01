# Part 006: ฟังก์ชันและ Visibility

## สารบัญ
1. [Function Basics](#function-basics)
2. [Visibility Decorators: @external, @internal](#visibility)
3. [State Decorators: @view, @pure, @payable](#state-decorators)
4. [@nonreentrant Decorator](#nonreentrant)
5. [Function Parameters และ Return Values](#params-returns)
6. [Internal Helper Pattern](#helper-pattern)
7. [Fallback Function: @default](#fallback)
8. [Function Selectors](#selectors)
9. [ตัวอย่าง: Math Library Contract](#example)

---

## 1. Function Basics {#function-basics}

ฟังก์ชันใน Vyper ประกาศด้วย `def` เหมือน Python แต่ต้องมี Decorator กำกับเสมอ

### โครงสร้างพื้นฐาน

```python
# @version 0.4.0

# ════════════════════════════════════════
# Function Structure
# ════════════════════════════════════════

@external
def simple_function():
    pass

@external
def function_with_params(x: uint256, y: address) -> bool:
    return x > 0

@external
def multiple_returns(amount: uint256) -> (uint256, bool):
    return amount * 2, True
```

### กฎการตั้งชื่อ

```python
# @version 0.4.0

# ✅ ชื่อที่ถูกต้อง
@external
def transfer_tokens(recipient: address, amount: uint256):
    pass

@external
def get_balance_of(account: address) -> uint256:
    return 0

@internal
def _calculate_fee(amount: uint256) -> uint256:
    return amount / 100  # 1% fee

# ❌ ห้ามใช้ตัวเลขนำหน้า, ตัวอักษรพิเศษ
# def 123function(): ...  (Error!)
# def my-function(): ...  (Error!)
```

---

## 2. Visibility Decorators: @external, @internal {#visibility}

### @external

ฟังก์ชันที่เรียกได้จากภายนอก Contract (จาก EOA, Contract อื่น, หรือ Frontend)

```python
# @version 0.4.0

owner: address
balance: HashMap[address, uint256]

@deploy
def __init__():
    self.owner = msg.sender

# เรียกได้จากภายนอก
@external
def deposit():
    self.balance[msg.sender] += msg.value

@external
def withdraw(amount: uint256):
    assert self.balance[msg.sender] >= amount, "Insufficient balance"
    self.balance[msg.sender] -= amount
    send(msg.sender, amount)

@view
@external
def get_balance(account: address) -> uint256:
    return self.balance[account]
```

### @internal

ฟังก์ชันที่เรียกได้เฉพาะภายใน Contract เดียวกัน ไม่สามารถเรียกจากภายนอกได้

```python
# @version 0.4.0

total_supply: uint256
balances: HashMap[address, uint256]

@deploy
def __init__(initial_supply: uint256):
    self.total_supply = initial_supply
    self.balances[msg.sender] = initial_supply

# ฟังก์ชัน Internal - ใช้ภายในเท่านั้น
@internal
def _mint(to: address, amount: uint256):
    self.total_supply += amount
    self.balances[to] += amount

@internal
def _burn(from_addr: address, amount: uint256):
    assert self.balances[from_addr] >= amount, "Insufficient balance"
    self.total_supply -= amount
    self.balances[from_addr] -= amount

@internal
def _transfer(sender: address, recipient: address, amount: uint256):
    assert self.balances[sender] >= amount, "Insufficient balance"
    assert recipient != empty(address), "Transfer to zero address"
    self.balances[sender] -= amount
    self.balances[recipient] += amount

# External Functions ที่เรียก Internal
@external
def transfer(to: address, amount: uint256) -> bool:
    self._transfer(msg.sender, to, amount)
    return True

@external
def mint(to: address, amount: uint256):
    # ตรวจสอบ permission ที่นี่ก่อนเรียก internal
    self._mint(to, amount)
```

### ข้อแตกต่างที่สำคัญ

```python
# @version 0.4.0

counter: uint256

@internal
def _increment(amount: uint256) -> uint256:
    # Internal: เข้าถึง self.storage ได้
    self.counter += amount
    return self.counter

@view
@internal
def _double(x: uint256) -> uint256:
    # Internal view: อ่าน state ได้ แต่ไม่เปลี่ยน
    return x * 2

@pure
@internal
def _add(a: uint256, b: uint256) -> uint256:
    # Internal pure: ไม่เข้าถึง state เลย
    return a + b

@external
def process(value: uint256) -> uint256:
    doubled: uint256 = self._double(value)
    summed: uint256 = self._add(doubled, 10)
    result: uint256 = self._increment(summed)
    return result
```

---

## 3. State Decorators: @view, @pure, @payable {#state-decorators}

### @view

อ่าน State ได้แต่ไม่เปลี่ยนแปลง State ไม่เสีย Gas เมื่อเรียกจากภายนอก (off-chain)

```python
# @version 0.4.0

name: String[50]
symbol: String[10]
total_supply: uint256
balances: HashMap[address, uint256]
DECIMALS: constant(uint8) = 18

@deploy
def __init__():
    self.name = "MyToken"
    self.symbol = "MTK"
    self.total_supply = 1_000_000 * 10**18
    self.balances[msg.sender] = self.total_supply

# @view: อ่านได้ ไม่เขียน
@view
@external
def get_name() -> String[50]:
    return self.name

@view
@external
def get_symbol() -> String[10]:
    return self.symbol

@view
@external
def get_total_supply() -> uint256:
    return self.total_supply

@view
@external
def balance_of(account: address) -> uint256:
    return self.balances[account]

@view
@external
def get_info() -> (String[50], String[10], uint256, uint8):
    # Return หลายค่าใน @view
    return self.name, self.symbol, self.total_supply, DECIMALS

# ❌ นี้จะ Error - @view ไม่ให้เขียน storage
# @view
# @external
# def bad_view():
#     self.total_supply = 0  # Error!
```

### @pure

ไม่อ่านและไม่เขียน State เลย เป็น Pure Computation

```python
# @version 0.4.0

# @pure: ไม่เข้าถึง state
@pure
@external
def calculate_percentage(amount: uint256, percent: uint256) -> uint256:
    return (amount * percent) / 100

@pure
@external
def calculate_compound_interest(
    principal: uint256,
    rate: uint256,  # rate in basis points (1% = 100)
    periods: uint256
) -> uint256:
    # คำนวณดอกเบี้ยทบต้นแบบง่าย (integer approximation)
    result: uint256 = principal
    for i: uint256 in range(periods, bound=365):
        result = result + (result * rate) / 10000
    return result

@pure
@external
def is_valid_amount(amount: uint256) -> bool:
    return amount > 0 and amount <= 10**27  # Max reasonable amount

@pure
@external
def pack_data(a: uint128, b: uint128) -> uint256:
    # Pack สอง uint128 เป็น uint256
    return convert(a, uint256) * (2**128) + convert(b, uint256)

@pure
@external
def unpack_data(packed: uint256) -> (uint128, uint128):
    a: uint128 = convert(packed / (2**128), uint128)
    b: uint128 = convert(packed % (2**128), uint128)
    return a, b

# ❌ Error - @pure ไม่ให้อ่าน state
# @pure
# @external
# def bad_pure() -> uint256:
#     return self.total_supply  # Error!
```

### @payable

รับ ETH ได้ ถ้าไม่ใส่ @payable แล้วมี msg.value > 0 จะ revert อัตโนมัติ

```python
# @version 0.4.0

owner: address
deposits: HashMap[address, uint256]
total_eth: uint256

event Deposit:
    sender: indexed(address)
    amount: uint256

event Withdrawal:
    recipient: indexed(address)
    amount: uint256

@deploy
def __init__():
    self.owner = msg.sender

# @payable: รับ ETH ได้
@payable
@external
def deposit():
    assert msg.value > 0, "Must send ETH"
    self.deposits[msg.sender] += msg.value
    self.total_eth += msg.value
    log Deposit(msg.sender, msg.value)

@payable
@external
def deposit_for(beneficiary: address):
    assert msg.value > 0, "Must send ETH"
    assert beneficiary != empty(address), "Invalid address"
    self.deposits[beneficiary] += msg.value
    self.total_eth += msg.value
    log Deposit(beneficiary, msg.value)

@external
def withdraw(amount: uint256):
    assert self.deposits[msg.sender] >= amount, "Insufficient balance"
    self.deposits[msg.sender] -= amount
    self.total_eth -= amount
    send(msg.sender, amount)
    log Withdrawal(msg.sender, amount)

@view
@external
def get_my_deposit() -> uint256:
    return self.deposits[msg.sender]

@view
@external
def get_contract_balance() -> uint256:
    return self.balance  # self.balance = ETH balance ของ contract

# ❌ ฟังก์ชันที่ไม่มี @payable จะ revert ถ้าส่ง ETH มาด้วย
@external
def no_eth_allowed(x: uint256) -> uint256:
    return x * 2  # revert ถ้า msg.value > 0
```

---

## 4. @nonreentrant Decorator {#nonreentrant}

ป้องกัน Reentrancy Attack โดยล็อคฟังก์ชันไม่ให้ถูกเรียกซ้ำในระหว่างที่ยังทำงานอยู่

```python
# @version 0.4.0

# ════════════════════════════════════════
# Reentrancy Attack Example & Protection
# ════════════════════════════════════════

owner: address
balances: HashMap[address, uint256]

event Withdraw:
    user: indexed(address)
    amount: uint256

@deploy
def __init__():
    self.owner = msg.sender

@payable
@external
def deposit():
    self.balances[msg.sender] += msg.value

# ❌ ไม่ปลอดภัย - ถูก Reentrancy Attack ได้
@external
def unsafe_withdraw(amount: uint256):
    assert self.balances[msg.sender] >= amount
    # ส่ง ETH ก่อน - Attacker contract จะเรียก withdraw อีกครั้ง!
    raw_call(msg.sender, b"", value=amount)
    self.balances[msg.sender] -= amount  # อาจทำงานหลังถูก reenter

# ✅ ปลอดภัย - @nonreentrant ป้องกัน Reentrancy
@nonreentrant
@external
def safe_withdraw(amount: uint256):
    assert self.balances[msg.sender] >= amount, "Insufficient balance"
    # Checks-Effects-Interactions Pattern
    # 1. Effects ก่อน
    self.balances[msg.sender] -= amount
    # 2. แล้วค่อย Interactions
    send(msg.sender, amount)
    log Withdraw(msg.sender, amount)

# @nonreentrant ป้องกันการเรียกซ้ำได้แม้จาก function อื่น
@nonreentrant
@payable
@external
def deposit_and_withdraw(withdraw_amount: uint256):
    # Deposit ก่อน
    self.balances[msg.sender] += msg.value
    # แล้ว Withdraw
    if withdraw_amount > 0:
        assert self.balances[msg.sender] >= withdraw_amount
        self.balances[msg.sender] -= withdraw_amount
        send(msg.sender, withdraw_amount)
```

### ทำความเข้าใจ Reentrancy

```python
# @version 0.4.0

# ════════════════════════════════════════
# Complex Reentrancy Protection
# ════════════════════════════════════════

owner: address
balances: HashMap[address, uint256]
total_deposited: uint256

@deploy
def __init__():
    self.owner = msg.sender

@nonreentrant
@payable
@external
def deposit():
    assert msg.value >= 1000000000000000, "Minimum 0.001 ETH"
    self.balances[msg.sender] += msg.value
    self.total_deposited += msg.value

@nonreentrant
@external
def withdraw_all():
    amount: uint256 = self.balances[msg.sender]
    assert amount > 0, "Nothing to withdraw"
    # Update state FIRST (Checks-Effects-Interactions)
    self.balances[msg.sender] = 0
    self.total_deposited -= amount
    # Then interact
    send(msg.sender, amount)

@nonreentrant
@external
def withdraw_partial(amount: uint256):
    assert self.balances[msg.sender] >= amount, "Insufficient"
    # Effects first
    self.balances[msg.sender] -= amount
    self.total_deposited -= amount
    # Interaction last
    send(msg.sender, amount)

@view
@external
def get_balance(user: address) -> uint256:
    return self.balances[user]

@view
@external
def get_contract_balance() -> uint256:
    return self.balance
```

---

## 5. Function Parameters และ Return Values {#params-returns}

### Parameter Types

```python
# @version 0.4.0

# ════════════════════════════════════════
# All Parameter Types
# ════════════════════════════════════════

struct Order:
    buyer: address
    amount: uint256
    price: uint256

# Primitive Types
@external
def func_primitives(
    a: uint256,
    b: int256,
    c: bool,
    d: address,
    e: bytes32,
    f: uint8
) -> bool:
    return a > 0

# String และ Bytes
@external
def func_strings(
    name: String[100],
    data: Bytes[256],
    hash: bytes32
) -> String[100]:
    return name

# Arrays
@external
def func_arrays(
    fixed: uint256[5],
    dynamic: DynArray[address, 100]
) -> uint256:
    return fixed[0]

# Struct
@external
def func_struct(order: Order) -> uint256:
    return order.amount * order.price

# ไม่มี Default Parameters ใน Vyper
# def func(x: uint256 = 0): ... ❌ Error!
# แก้ด้วยการสร้าง overloaded functions:
@external
def process_with_fee(amount: uint256, fee: uint256) -> uint256:
    return amount - fee

@external
def process_default_fee(amount: uint256) -> uint256:
    # เรียก version ที่มี fee พร้อม default value
    return self.process_with_fee(amount, amount / 100)  # 1% default fee
```

### Return Values

```python
# @version 0.4.0

struct TokenInfo:
    name: String[50]
    symbol: String[10]
    supply: uint256

name: String[50]
symbol: String[10]
total_supply: uint256

@deploy
def __init__():
    self.name = "TestToken"
    self.symbol = "TST"
    self.total_supply = 1000000

# Single return
@view
@external
def get_supply() -> uint256:
    return self.total_supply

# Tuple return
@view
@external
def get_token_basics() -> (String[50], String[10]):
    return self.name, self.symbol

# Multiple values
@view
@external
def get_full_info() -> (String[50], String[10], uint256):
    return self.name, self.symbol, self.total_supply

# Struct return
@view
@external
def get_token_info() -> TokenInfo:
    return TokenInfo({
        name: self.name,
        symbol: self.symbol,
        supply: self.total_supply
    })

# Array return
@view
@external
def get_top_holders(limit: uint256) -> DynArray[address, 10]:
    result: DynArray[address, 10] = []
    # ตัวอย่าง - return empty array
    return result

# Boolean return (common pattern)
@external
def try_transfer(to: address, amount: uint256) -> bool:
    if amount == 0:
        return False
    if to == empty(address):
        return False
    # ทำ transfer
    return True
```

---

## 6. Internal Helper Pattern {#helper-pattern}

Pattern ที่ใช้บ่อยในการจัดระเบียบ Code

```python
# @version 0.4.0

# ════════════════════════════════════════
# Internal Helper Pattern
# ════════════════════════════════════════

owner: address
balances: HashMap[address, uint256]
allowances: HashMap[address, HashMap[address, uint256]]
total_supply: uint256
paused: bool

event Transfer:
    sender: indexed(address)
    recipient: indexed(address)
    amount: uint256

event Approval:
    owner: indexed(address)
    spender: indexed(address)
    amount: uint256

@deploy
def __init__(initial_supply: uint256):
    self.owner = msg.sender
    self.total_supply = initial_supply
    self.balances[msg.sender] = initial_supply

# ════════════════
# Internal Helpers (ขึ้นต้นด้วย _)
# ════════════════

@internal
def _require_owner():
    assert msg.sender == self.owner, "Not owner"

@internal
def _require_not_paused():
    assert not self.paused, "Contract is paused"

@internal
def _require_valid_address(addr: address):
    assert addr != empty(address), "Invalid address"

@internal
def _require_sufficient_balance(addr: address, amount: uint256):
    assert self.balances[addr] >= amount, "Insufficient balance"

@internal
def _transfer_tokens(from_addr: address, to: address, amount: uint256):
    self._require_valid_address(to)
    self._require_sufficient_balance(from_addr, amount)
    self.balances[from_addr] -= amount
    self.balances[to] += amount
    log Transfer(from_addr, to, amount)

@internal
def _approve_tokens(owner_addr: address, spender: address, amount: uint256):
    self._require_valid_address(spender)
    self.allowances[owner_addr][spender] = amount
    log Approval(owner_addr, spender, amount)

# ════════════════
# External Functions ใช้ Internal Helpers
# ════════════════

@external
def transfer(to: address, amount: uint256) -> bool:
    self._require_not_paused()
    self._transfer_tokens(msg.sender, to, amount)
    return True

@external
def transfer_from(sender: address, recipient: address, amount: uint256) -> bool:
    self._require_not_paused()
    allowed: uint256 = self.allowances[sender][msg.sender]
    assert allowed >= amount, "Insufficient allowance"
    self.allowances[sender][msg.sender] -= amount
    self._transfer_tokens(sender, recipient, amount)
    return True

@external
def approve(spender: address, amount: uint256) -> bool:
    self._approve_tokens(msg.sender, spender, amount)
    return True

@external
def pause():
    self._require_owner()
    self.paused = True

@external
def unpause():
    self._require_owner()
    self.paused = False

@external
def mint(to: address, amount: uint256):
    self._require_owner()
    self._require_not_paused()
    self._require_valid_address(to)
    self.total_supply += amount
    self.balances[to] += amount
    log Transfer(empty(address), to, amount)
```

---

## 7. Fallback Function: @default {#fallback}

ฟังก์ชันที่ถูกเรียกเมื่อมีการส่ง ETH มาโดยไม่ระบุฟังก์ชัน หรือเรียก function selector ที่ไม่มี

```python
# @version 0.4.0

# ════════════════════════════════════════
# @default (Fallback) Function
# ════════════════════════════════════════

owner: address
received_eth: uint256

event Received:
    sender: indexed(address)
    amount: uint256

@deploy
def __init__():
    self.owner = msg.sender

# @default: รับ ETH ที่ส่งมาโดยไม่ระบุ function
@payable
@external
def __default__():
    self.received_eth += msg.value
    log Received(msg.sender, msg.value)

@view
@external
def get_received() -> uint256:
    return self.received_eth

@external
def withdraw():
    assert msg.sender == self.owner, "Not owner"
    amount: uint256 = self.balance
    self.received_eth = 0
    send(self.owner, amount)
```

### @default แบบไม่รับ ETH

```python
# @version 0.4.0

# @default ที่ reject ETH
@external
def __default__():
    # ไม่มี @payable = revert อัตโนมัติถ้าส่ง ETH มา
    # ใช้สำหรับ Log การเรียก function ที่ไม่มี
    pass
```

---

## 8. Function Selectors {#selectors}

Function Selector คือ 4 bytes แรกของ Keccak256 hash ของ function signature

```python
# @version 0.4.0

# ════════════════════════════════════════
# Function Selectors
# ════════════════════════════════════════

# transfer(address,uint256) -> selector = bytes4(keccak256("transfer(address,uint256)"))
# = 0xa9059cbb

TRANSFER_SELECTOR: constant(bytes4) = 0xa9059cbb
APPROVE_SELECTOR: constant(bytes4) = 0x095ea7b3
BALANCE_OF_SELECTOR: constant(bytes4) = 0x70a08231

@view
@external
def call_balance_of(token: address, account: address) -> uint256:
    # เรียก ERC20 balanceOf ด้วย raw_call
    response: Bytes[32] = raw_call(
        token,
        concat(
            BALANCE_OF_SELECTOR,
            convert(convert(account, uint256), bytes32)
        ),
        max_outsize=32,
        is_static_call=True
    )
    return convert(response, uint256)

@external
def call_transfer(token: address, to: address, amount: uint256) -> bool:
    # เรียก ERC20 transfer ด้วย raw_call
    response: Bytes[32] = raw_call(
        token,
        concat(
            TRANSFER_SELECTOR,
            convert(convert(to, uint256), bytes32),
            convert(amount, bytes32)
        ),
        max_outsize=32
    )
    return convert(response, bool)
```

---

## 9. ตัวอย่าง: Math Library Contract {#example}

Contract ที่รวม Mathematical Functions ครบถ้วน พร้อม Internal/External Visibility

```python
# @version 0.4.0

# ════════════════════════════════════════════════════════════
# MathLibrary Contract
# รวม Mathematical utility functions สำหรับ DeFi applications
# ════════════════════════════════════════════════════════════

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Constants
# ━━━━━━━━━━━━━━━━━━━━━━━━━
PRECISION: constant(uint256) = 10**18  # 18 decimal precision
MAX_ITERATIONS: constant(uint256) = 256
WAD: constant(uint256) = 10**18  # 1.0 in WAD math
RAY: constant(uint256) = 10**27  # 1.0 in RAY math

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Events
# ━━━━━━━━━━━━━━━━━━━━━━━━━
event Calculated:
    operation: String[20]
    result: uint256

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Internal Pure Functions
# ━━━━━━━━━━━━━━━━━━━━━━━━━

@pure
@internal
def _min(a: uint256, b: uint256) -> uint256:
    if a < b:
        return a
    return b

@pure
@internal
def _max(a: uint256, b: uint256) -> uint256:
    if a > b:
        return a
    return b

@pure
@internal
def _abs_diff(a: uint256, b: uint256) -> uint256:
    if a >= b:
        return a - b
    return b - a

@pure
@internal
def _clamp(value: uint256, min_val: uint256, max_val: uint256) -> uint256:
    if value < min_val:
        return min_val
    if value > max_val:
        return max_val
    return value

@pure
@internal
def _safe_add(a: uint256, b: uint256) -> uint256:
    result: uint256 = a + b
    assert result >= a, "Addition overflow"
    return result

@pure
@internal
def _safe_mul(a: uint256, b: uint256) -> uint256:
    if a == 0:
        return 0
    result: uint256 = a * b
    assert result / a == b, "Multiplication overflow"
    return result

@pure
@internal
def _wad_mul(a: uint256, b: uint256) -> uint256:
    # WAD multiplication: (a * b) / 10^18
    return (a * b) / WAD

@pure
@internal
def _wad_div(a: uint256, b: uint256) -> uint256:
    # WAD division: (a * 10^18) / b
    assert b > 0, "Division by zero"
    return (a * WAD) / b

@pure
@internal
def _ray_mul(a: uint256, b: uint256) -> uint256:
    # RAY multiplication: (a * b) / 10^27
    return (a * b) / RAY

@pure
@internal
def _percentage(amount: uint256, bps: uint256) -> uint256:
    # Calculate percentage in basis points (1% = 100 bps)
    return (amount * bps) / 10000

@pure
@internal
def _sqrt(x: uint256) -> uint256:
    # Integer square root using Babylonian method
    if x == 0:
        return 0
    z: uint256 = (x + 1) / 2
    y: uint256 = x
    for _: uint256 in range(256, bound=MAX_ITERATIONS):
        if z >= y:
            break
        y = z
        z = (x / z + z) / 2
    return y

@pure
@internal
def _power(base: uint256, exp: uint256) -> uint256:
    # Fast exponentiation
    result: uint256 = 1
    b: uint256 = base
    e: uint256 = exp
    for _: uint256 in range(256, bound=MAX_ITERATIONS):
        if e == 0:
            break
        if e % 2 == 1:
            result = result * b
        b = b * b
        e = e / 2
    return result

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# External View Functions
# ━━━━━━━━━━━━━━━━━━━━━━━━━

@pure
@external
def min(a: uint256, b: uint256) -> uint256:
    """Return the minimum of two values"""
    return self._min(a, b)

@pure
@external
def max(a: uint256, b: uint256) -> uint256:
    """Return the maximum of two values"""
    return self._max(a, b)

@pure
@external
def clamp(value: uint256, min_val: uint256, max_val: uint256) -> uint256:
    """Clamp value between min and max"""
    return self._clamp(value, min_val, max_val)

@pure
@external
def sqrt(x: uint256) -> uint256:
    """Integer square root"""
    return self._sqrt(x)

@pure
@external
def power(base: uint256, exp: uint256) -> uint256:
    """Integer power: base^exp"""
    return self._power(base, exp)

@pure
@external
def wad_mul(a: uint256, b: uint256) -> uint256:
    """WAD multiplication (18 decimal precision)"""
    return self._wad_mul(a, b)

@pure
@external
def wad_div(a: uint256, b: uint256) -> uint256:
    """WAD division (18 decimal precision)"""
    return self._wad_div(a, b)

@pure
@external
def percentage(amount: uint256, bps: uint256) -> uint256:
    """Calculate percentage in basis points"""
    return self._percentage(amount, bps)

@pure
@external
def calculate_fee(amount: uint256, fee_bps: uint256) -> (uint256, uint256):
    """
    Calculate fee and net amount
    Returns: (fee_amount, net_amount)
    """
    fee: uint256 = self._percentage(amount, fee_bps)
    return fee, amount - fee

@pure
@external
def calculate_slippage(
    expected: uint256,
    actual: uint256,
    max_slippage_bps: uint256
) -> bool:
    """Check if actual amount is within acceptable slippage"""
    if actual >= expected:
        return True
    diff: uint256 = expected - actual
    slippage_bps: uint256 = (diff * 10000) / expected
    return slippage_bps <= max_slippage_bps

@pure
@external
def compound_interest(
    principal: uint256,
    rate_bps: uint256,  # Annual rate in basis points
    periods: uint256    # Number of compounding periods
) -> uint256:
    """
    Calculate compound interest
    Formula: P * (1 + r)^n (integer approximation)
    """
    result: uint256 = principal
    for _: uint256 in range(365, bound=365):
        if _ >= periods:
            break
        interest: uint256 = self._percentage(result, rate_bps)
        result = result + interest
    return result

@pure
@external
def interpolate(
    start: uint256,
    end_val: uint256,
    current: uint256,
    total: uint256
) -> uint256:
    """
    Linear interpolation
    Returns value at 'current' position between start and end
    """
    assert total > 0, "Total must be > 0"
    if current >= total:
        return end_val
    if end_val >= start:
        return start + ((end_val - start) * current) / total
    return start - ((start - end_val) * current) / total

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Stateful Functions (with events)
# ━━━━━━━━━━━━━━━━━━━━━━━━━

@external
def calculate_and_log(a: uint256, b: uint256) -> uint256:
    """Calculate square root of sum and log result"""
    sum_val: uint256 = self._safe_add(a, b)
    result: uint256 = self._sqrt(sum_val)
    log Calculated("sqrt_sum", result)
    return result
```

### Test Code

```python
# tests/test_math_library.py
import pytest
from brownie import MathLibrary, accounts

@pytest.fixture
def math_lib():
    return MathLibrary.deploy({'from': accounts[0]})

class TestBasicOperations:
    def test_min(self, math_lib):
        assert math_lib.min(5, 3) == 3
        assert math_lib.min(0, 100) == 0
        assert math_lib.min(7, 7) == 7

    def test_max(self, math_lib):
        assert math_lib.max(5, 3) == 5
        assert math_lib.max(0, 100) == 100

    def test_clamp(self, math_lib):
        assert math_lib.clamp(5, 1, 10) == 5
        assert math_lib.clamp(0, 1, 10) == 1
        assert math_lib.clamp(15, 1, 10) == 10

    def test_sqrt(self, math_lib):
        assert math_lib.sqrt(0) == 0
        assert math_lib.sqrt(1) == 1
        assert math_lib.sqrt(4) == 2
        assert math_lib.sqrt(9) == 3
        assert math_lib.sqrt(100) == 10
        assert math_lib.sqrt(2) == 1  # Floor sqrt

    def test_power(self, math_lib):
        assert math_lib.power(2, 0) == 1
        assert math_lib.power(2, 10) == 1024
        assert math_lib.power(3, 3) == 27

class TestDeFiMath:
    def test_wad_mul(self, math_lib):
        WAD = 10**18
        # 2.0 * 3.0 = 6.0
        assert math_lib.wad_mul(2 * WAD, 3 * WAD) == 6 * WAD

    def test_percentage(self, math_lib):
        # 10% of 1000 = 100
        assert math_lib.percentage(1000, 1000) == 100
        # 0.5% of 10000 = 50
        assert math_lib.percentage(10000, 50) == 50

    def test_calculate_fee(self, math_lib):
        fee, net = math_lib.calculate_fee(10000, 300)  # 3% fee
        assert fee == 300
        assert net == 9700

    def test_slippage_check(self, math_lib):
        # 1% slippage, 1% max -> OK
        assert math_lib.calculate_slippage(1000, 990, 100) == True
        # 2% slippage, 1% max -> Fail
        assert math_lib.calculate_slippage(1000, 980, 100) == False

    def test_compound_interest(self, math_lib):
        # 1000 principal, 10% rate (1000 bps), 1 period = 1100
        result = math_lib.compound_interest(1000, 1000, 1)
        assert result == 1100
```

---

## สรุป

- ✅ **@external**: เรียกได้จากภายนอก Contract
- ✅ **@internal**: เรียกได้เฉพาะภายใน Contract
- ✅ **@view**: อ่าน State ได้ ไม่เขียน
- ✅ **@pure**: ไม่อ่านและไม่เขียน State
- ✅ **@payable**: รับ ETH ได้
- ✅ **@nonreentrant**: ป้องกัน Reentrancy Attack
- ✅ **@default**: Fallback function สำหรับรับ ETH ที่ไม่ระบุ function
- ✅ **Internal Helper Pattern**: จัดระเบียบ Code ด้วย `_` prefix

## แบบฝึกหัด

1. **สร้าง** Access Control Contract ที่มี roles: owner, admin, user โดยใช้ internal helpers
2. **เพิ่ม** batch operations ใน MathLibrary ให้คำนวณหลายค่าพร้อมกัน
3. **ทดสอบ** reentrancy attack กับ contract ที่ไม่ได้ป้องกัน
4. **สร้าง** Function ที่รับ DynArray[uint256, 100] แล้ว return ค่า min, max, sum
5. **วิเคราะห์** Gas ของ @view vs @pure เมื่อเรียกจาก on-chain vs off-chain

---

**ก่อนหน้า: [Part 005 - ตัวแปรและ State Variables](part_005_variables.md)**  
**ต่อไป: [Part 007 - Control Flow: If/Elif/Else](part_007_control_flow.md)**
