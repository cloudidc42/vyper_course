# Part 003: Smart Contract แรกของคุณ

## สารบัญ
1. [โครงสร้าง Vyper File](#structure)
2. [Version Pragma](#pragma)
3. [ตัวแปรพื้นฐาน](#variables)
4. [Functions](#functions)
5. [Decorators](#decorators)
6. [Constructor](#constructor)
7. [Contract แรก: SimpleStorage](#simple-storage)
8. [Contract: Greeting](#greeting)
9. [Contract: Counter](#counter)
10. [การ Compile และ Deploy](#compile-deploy)
11. [การทดสอบ](#testing)

---

## 1. โครงสร้าง Vyper File {#structure}

ทุก Vyper Contract มีโครงสร้างพื้นฐาน:

```python
# @version 0.4.0           ← Version Pragma (บังคับ)
# SPDX-License-Identifier: MIT  ← License (แนะนำ)

# ════════════════════════════
# IMPORTS (ถ้ามี)
# ════════════════════════════
from vyper.interfaces import ERC20

# ════════════════════════════
# INTERFACES (ถ้ามี)
# ════════════════════════════
interface IMyToken:
    def transfer(to: address, amount: uint256) -> bool: nonpayable

# ════════════════════════════
# EVENTS
# ════════════════════════════
event Transfer:
    sender: indexed(address)
    receiver: indexed(address)
    amount: uint256

# ════════════════════════════
# STATE VARIABLES
# ════════════════════════════
owner: address
balance: HashMap[address, uint256]
total_supply: uint256

# ════════════════════════════
# FUNCTIONS
# ════════════════════════════

@deploy
def __init__():
    """Constructor"""
    self.owner = msg.sender

@view
@external
def get_owner() -> address:
    """Getter function"""
    return self.owner

@external
def transfer(to: address, amount: uint256):
    """State-changing function"""
    assert self.balance[msg.sender] >= amount
    self.balance[msg.sender] -= amount
    self.balance[to] += amount
    log Transfer(msg.sender, to, amount)
```

---

## 2. Version Pragma {#pragma}

```python
# Version Pragma บอก Compiler ว่าต้องการ Vyper เวอร์ชันอะไร

# เวอร์ชันเดียว
# @version 0.4.0

# เวอร์ชัน range
# @version ^0.4.0   # 0.4.x ทุกเวอร์ชัน
# @version >=0.3.10  # 0.3.10 หรือสูงกว่า

# ทำไม Version Pragma ถึงสำคัญ:
# 1. ป้องกันการ Compile ด้วย Version ที่ไม่เข้ากัน
# 2. พฤติกรรมของ Contract ต้องแน่นอน
# 3. Security: แต่ละ Version อาจมี Bug Fixes
```

### ความแตกต่างของ Vyper Versions

```
0.2.x → 0.3.x: Breaking changes
0.3.x → 0.4.x: Breaking changes (เพิ่ม @deploy decorator)

สำคัญ:
- 0.4.0+: ใช้ @deploy แทน @external สำหรับ __init__
- 0.3.x: ใช้ @external def __init__() หรือไม่ต้อง decorator
```

---

## 3. ตัวแปรพื้นฐาน {#variables}

### State Variables

State Variables เก็บอยู่ใน Blockchain Storage (persistent):

```python
# @version 0.4.0

# ประกาศ State Variables
my_number: uint256          # Unsigned integer 256-bit
my_int: int256              # Signed integer 256-bit
my_bool: bool               # Boolean
my_address: address         # Ethereum Address
my_string: String[100]      # String ความยาวสูงสุด 100
my_bytes: Bytes[32]         # Raw bytes ความยาวสูงสุด 32
my_bytes32: bytes32         # Fixed 32 bytes
```

### Local Variables

Local Variables อยู่ใน Memory เท่านั้น (ไม่ persistent):

```python
@external
def example_function():
    # Local variables - อยู่ใน Memory
    local_num: uint256 = 100
    local_address: address = msg.sender
    local_bool: bool = True
    
    # ใช้ local variable
    result: uint256 = local_num * 2
```

### Constants

```python
# @version 0.4.0

# Constants - ค่าคงที่ ไม่เปลี่ยนแปลง
MAX_SUPPLY: constant(uint256) = 1_000_000 * 10**18
DECIMALS: constant(uint8) = 18
NAME: constant(String[20]) = "My Token"
ZERO_ADDRESS: constant(address) = 0x0000000000000000000000000000000000000000

# Immutables - ตั้งค่าได้ครั้งเดียวใน __init__
owner: immutable(address)
creation_time: immutable(uint256)

@deploy
def __init__():
    owner = msg.sender          # ตั้งค่า immutable
    creation_time = block.timestamp
```

---

## 4. Functions {#functions}

### ประเภทของ Functions

```python
# @version 0.4.0

# 1. View Function - อ่านข้อมูลเท่านั้น, ไม่ใช้ Gas (ถ้าเรียกจากนอก)
@view
@external
def read_data() -> uint256:
    return self.some_value

# 2. Pure Function - ไม่อ่านหรือเปลี่ยน State
@pure
@external
def calculate(a: uint256, b: uint256) -> uint256:
    return a + b

# 3. State-changing Function - เปลี่ยน State, ใช้ Gas
@external
def update_data(new_value: uint256):
    self.some_value = new_value

# 4. Payable Function - รับ ETH ได้
@payable
@external
def deposit():
    # msg.value คือ ETH ที่ส่งมา
    self.balances[msg.sender] += msg.value

# 5. Internal Function - เรียกได้แค่ใน Contract นี้
@internal
def _helper_function(value: uint256) -> uint256:
    return value * 2
```

### Function Parameters และ Return Values

```python
# @version 0.4.0

# Single return value
@view
@external
def get_sum(a: uint256, b: uint256) -> uint256:
    return a + b

# Multiple return values (Tuple)
@view
@external
def get_multiple() -> (uint256, bool, address):
    return 42, True, self.owner

# No return value
@external
def just_store(value: uint256):
    self.stored = value

# Default parameter values - ไม่มีใน Vyper!
# Vyper ไม่รองรับ default parameters เพื่อความชัดเจน
```

---

## 5. Decorators {#decorators}

Decorators ใน Vyper บอก Properties ของ Function:

```python
# @version 0.4.0

# ═══════════════════════════════════
# Visibility Decorators (ต้องมีสักอัน)
# ═══════════════════════════════════

# @external: เรียกจากภายนอกได้
@external
def public_function():
    pass

# @internal: เรียกได้แค่ภายใน Contract
@internal
def private_function():
    pass

# ═══════════════════════════════════
# State Mutability Decorators
# ═══════════════════════════════════

# @view: อ่าน State ได้, แต่ไม่เปลี่ยน
@view
@external
def read_only() -> uint256:
    return self.value

# @pure: ไม่อ่านหรือเปลี่ยน State
@pure
@external
def no_state(x: uint256) -> uint256:
    return x * 2

# @payable: รับ ETH ได้
@payable
@external
def receive_eth():
    pass

# ═══════════════════════════════════
# Special Decorators
# ═══════════════════════════════════

# @deploy: Constructor (Vyper 0.4.0+)
@deploy
def __init__():
    self.owner = msg.sender

# @nonreentrant: ป้องกัน Reentrancy Attack
@nonreentrant
@external
def safe_withdraw():
    amount: uint256 = self.balances[msg.sender]
    self.balances[msg.sender] = 0  # State update ก่อน
    send(msg.sender, amount)       # แล้วจึง Send ETH
```

### Decorator Combinations ที่ถูกต้อง

```python
# @version 0.4.0

# ✅ ถูกต้อง
@view
@external
def valid1() -> uint256: ...

@pure
@external  
def valid2(x: uint256) -> uint256: ...

@payable
@external
def valid3(): ...

@nonreentrant
@external
def valid4(): ...

@view
@internal
def valid5() -> uint256: ...

# ❌ ผิด
# @view @payable ไม่สามารถใช้ร่วมกัน (view ไม่รับ ETH)
# @pure @view ไม่สามารถใช้ร่วมกัน
# @external @internal ไม่สามารถใช้ร่วมกัน
```

---

## 6. Constructor {#constructor}

```python
# @version 0.4.0

owner: address
creation_block: uint256
initial_value: uint256

# Constructor รันครั้งเดียวตอน Deploy
@deploy
def __init__(initial_val: uint256):
    self.owner = msg.sender
    self.creation_block = block.number
    self.initial_value = initial_val

# Immutable Constructor
owner2: immutable(address)

@deploy
def __init__():
    owner2 = msg.sender  # กำหนด immutable ใน __init__
    # หลังจากนี้ owner2 ไม่สามารถเปลี่ยนได้

@view
@external
def get_owner() -> address:
    return owner2  # อ่าน immutable (ไม่ต้อง self.)
```

---

## 7. Contract แรก: SimpleStorage {#simple-storage}

```python
# @version 0.4.0
# SPDX-License-Identifier: MIT
# ไฟล์: contracts/SimpleStorage.vy

"""
SimpleStorage - Contract เก็บตัวเลขอย่างง่าย
สอนแนวคิด: State Variables, Functions, Access Control
"""

# ════════════════════════
# EVENTS
# ════════════════════════
event ValueChanged:
    old_value: uint256
    new_value: uint256
    changed_by: indexed(address)

# ════════════════════════
# STATE VARIABLES
# ════════════════════════
stored_value: uint256
owner: address

# ════════════════════════
# CONSTRUCTOR
# ════════════════════════
@deploy
def __init__(initial_value: uint256):
    self.owner = msg.sender
    self.stored_value = initial_value

# ════════════════════════
# PUBLIC FUNCTIONS
# ════════════════════════

@view
@external
def get() -> uint256:
    """
    อ่านค่าที่เก็บไว้
    Returns: ค่าปัจจุบัน
    """
    return self.stored_value

@external
def set(new_value: uint256):
    """
    ตั้งค่าใหม่ (ใครก็ได้)
    Args:
        new_value: ค่าใหม่ที่ต้องการเก็บ
    """
    old_value: uint256 = self.stored_value
    self.stored_value = new_value
    log ValueChanged(old_value, new_value, msg.sender)

@external
def increment():
    """เพิ่มค่าขึ้น 1"""
    self.stored_value += 1

@external
def decrement():
    """ลดค่าลง 1"""
    assert self.stored_value > 0, "Cannot decrement below zero"
    self.stored_value -= 1

@external
def reset():
    """รีเซ็ตค่าเป็น 0 (เฉพาะ owner)"""
    assert msg.sender == self.owner, "Only owner can reset"
    old_value: uint256 = self.stored_value
    self.stored_value = 0
    log ValueChanged(old_value, 0, msg.sender)

@view
@external
def get_owner() -> address:
    """ดูว่าใคร Deploy Contract นี้"""
    return self.owner
```

### ทดสอบ SimpleStorage

```python
# tests/test_simple_storage.py
import boa
import pytest
from boa.environment import Env

# ════════════════════════
# FIXTURES
# ════════════════════════

@pytest.fixture
def env():
    """สร้าง Environment ใหม่ทุก Test"""
    return Env()

@pytest.fixture
def deployer(env):
    """Account สำหรับ Deploy"""
    return env.generate_address("deployer")

@pytest.fixture
def user(env):
    """User Account"""
    return env.generate_address("user")

@pytest.fixture
def storage(deployer):
    """Deploy SimpleStorage Contract"""
    with boa.env.prank(deployer):
        contract = boa.load(
            "contracts/SimpleStorage.vy",
            100  # initial_value = 100
        )
    return contract

# ════════════════════════
# TESTS
# ════════════════════════

class TestDeployment:
    def test_initial_value(self, storage):
        assert storage.get() == 100
    
    def test_owner_is_deployer(self, storage, deployer):
        assert storage.get_owner() == deployer

class TestGetSet:
    def test_set_value(self, storage):
        storage.set(200)
        assert storage.get() == 200
    
    def test_set_zero(self, storage):
        storage.set(0)
        assert storage.get() == 0
    
    def test_set_max_uint256(self, storage):
        max_val = 2**256 - 1
        storage.set(max_val)
        assert storage.get() == max_val
    
    def test_anyone_can_set(self, storage, user):
        with boa.env.prank(user):
            storage.set(999)
        assert storage.get() == 999

class TestIncDecrement:
    def test_increment(self, storage):
        initial = storage.get()
        storage.increment()
        assert storage.get() == initial + 1
    
    def test_multiple_increments(self, storage):
        for i in range(5):
            storage.increment()
        assert storage.get() == 100 + 5
    
    def test_decrement(self, storage):
        initial = storage.get()
        storage.decrement()
        assert storage.get() == initial - 1
    
    def test_decrement_at_zero_reverts(self, storage):
        storage.set(0)
        with pytest.raises(Exception):
            storage.decrement()

class TestReset:
    def test_owner_can_reset(self, storage, deployer):
        with boa.env.prank(deployer):
            storage.reset()
        assert storage.get() == 0
    
    def test_non_owner_cannot_reset(self, storage, user):
        with pytest.raises(Exception, match="Only owner can reset"):
            with boa.env.prank(user):
                storage.reset()

class TestEvents:
    def test_value_changed_event(self, storage):
        storage.set(500)
        # ใน Titanoboa สามารถตรวจสอบ events ได้
        # logs = storage.get_logs()
        # assert logs[-1].event_type.name == "ValueChanged"
```

---

## 8. Contract: Greeting {#greeting}

```python
# @version 0.4.0
# SPDX-License-Identifier: MIT
# ไฟล์: contracts/Greeting.vy

"""
Greeting Contract
สอนแนวคิด: String handling, Ownership, Events
"""

# ════════════════════════
# CONSTANTS
# ════════════════════════
MAX_GREETING_LENGTH: constant(uint256) = 200

# ════════════════════════
# EVENTS  
# ════════════════════════
event GreetingUpdated:
    updater: indexed(address)
    old_greeting: String[200]
    new_greeting: String[200]

event OwnershipTransferred:
    old_owner: indexed(address)
    new_owner: indexed(address)

# ════════════════════════
# STATE VARIABLES
# ════════════════════════
greeting: String[200]
owner: address
update_count: uint256

# ════════════════════════
# CONSTRUCTOR
# ════════════════════════
@deploy
def __init__(initial_greeting: String[200]):
    assert len(initial_greeting) > 0, "Greeting cannot be empty"
    self.owner = msg.sender
    self.greeting = initial_greeting
    self.update_count = 0

# ════════════════════════
# FUNCTIONS
# ════════════════════════

@view
@external
def get_greeting() -> String[200]:
    """อ่าน Greeting ปัจจุบัน"""
    return self.greeting

@view
@external
def get_update_count() -> uint256:
    """จำนวนครั้งที่อัปเดต"""
    return self.update_count

@view
@external
def get_owner() -> address:
    """เจ้าของ Contract"""
    return self.owner

@external
def update_greeting(new_greeting: String[200]):
    """
    อัปเดต Greeting
    เฉพาะ owner เท่านั้น
    """
    assert msg.sender == self.owner, "Only owner"
    assert len(new_greeting) > 0, "Cannot be empty"
    
    old_greeting: String[200] = self.greeting
    self.greeting = new_greeting
    self.update_count += 1
    
    log GreetingUpdated(msg.sender, old_greeting, new_greeting)

@external
def transfer_ownership(new_owner: address):
    """โอน Ownership"""
    assert msg.sender == self.owner, "Only owner"
    assert new_owner != empty(address), "Invalid address"
    
    old_owner: address = self.owner
    self.owner = new_owner
    
    log OwnershipTransferred(old_owner, new_owner)

@view
@external
def greet(name: String[50]) -> String[260]:
    """
    สร้าง Greeting สำหรับชื่อที่กำหนด
    Args:
        name: ชื่อที่ต้องการ Greet
    Returns:
        Greeting string
    """
    # Vyper ไม่มี string concatenation โดยตรง
    # ต้องใช้ concat builtin
    return concat(self.greeting, ", ", name, "!")
```

---

## 9. Contract: Counter {#counter}

```python
# @version 0.4.0
# SPDX-License-Identifier: MIT
# ไฟล์: contracts/Counter.vy

"""
Counter Contract - Counter ที่ปลอดภัย
สอนแนวคิด: Overflow protection, Events, Access patterns
"""

# ════════════════════════
# CONSTANTS
# ════════════════════════
MAX_COUNT: constant(uint256) = 10**18  # ค่าสูงสุดที่ยอมรับ

# ════════════════════════
# EVENTS
# ════════════════════════
event Incremented:
    counter: uint256
    incrementer: indexed(address)

event Decremented:
    counter: uint256
    decrementer: indexed(address)

event Reset:
    resetter: indexed(address)

# ════════════════════════
# STATE VARIABLES
# ════════════════════════
count: uint256
owner: address

# ════════════════════════
# CONSTRUCTOR
# ════════════════════════
@deploy
def __init__():
    self.owner = msg.sender
    self.count = 0

# ════════════════════════
# PUBLIC FUNCTIONS
# ════════════════════════

@view
@external
def get_count() -> uint256:
    """ดูค่า Counter ปัจจุบัน"""
    return self.count

@external
def increment():
    """เพิ่ม Counter ขึ้น 1"""
    assert self.count < MAX_COUNT, "Counter at maximum"
    self.count += 1
    log Incremented(self.count, msg.sender)

@external
def increment_by(amount: uint256):
    """เพิ่ม Counter ขึ้น amount"""
    assert amount > 0, "Amount must be positive"
    assert self.count + amount <= MAX_COUNT, "Would exceed maximum"
    self.count += amount
    log Incremented(self.count, msg.sender)

@external
def decrement():
    """ลด Counter ลง 1"""
    assert self.count > 0, "Counter at minimum"
    self.count -= 1
    log Decremented(self.count, msg.sender)

@external
def decrement_by(amount: uint256):
    """ลด Counter ลง amount"""
    assert amount > 0, "Amount must be positive"
    assert self.count >= amount, "Would go below minimum"
    self.count -= amount
    log Decremented(self.count, msg.sender)

@external
def reset():
    """รีเซ็ต Counter (เฉพาะ owner)"""
    assert msg.sender == self.owner, "Only owner"
    self.count = 0
    log Reset(msg.sender)

@view
@external
def is_at_max() -> bool:
    """ตรวจสอบว่า Counter ถึงค่าสูงสุดหรือไม่"""
    return self.count >= MAX_COUNT

@view
@external
def is_at_min() -> bool:
    """ตรวจสอบว่า Counter เป็น 0 หรือไม่"""
    return self.count == 0

@view
@external
def distance_to_max() -> uint256:
    """ระยะห่างจาก MAX_COUNT"""
    return MAX_COUNT - self.count
```

### Test Counter

```python
# tests/test_counter.py
import boa
import pytest

@pytest.fixture
def counter():
    return boa.load("contracts/Counter.vy")

@pytest.fixture
def deployer():
    return boa.env.generate_address("deployer")

def test_initial_count(counter):
    assert counter.get_count() == 0

def test_is_at_min_initially(counter):
    assert counter.is_at_min() == True

def test_increment(counter):
    counter.increment()
    assert counter.get_count() == 1
    assert counter.is_at_min() == False

def test_increment_by(counter):
    counter.increment_by(10)
    assert counter.get_count() == 10

def test_decrement(counter):
    counter.increment_by(5)
    counter.decrement()
    assert counter.get_count() == 4

def test_decrement_by(counter):
    counter.increment_by(10)
    counter.decrement_by(3)
    assert counter.get_count() == 7

def test_cannot_decrement_below_zero(counter):
    with pytest.raises(Exception):
        counter.decrement()

def test_cannot_decrement_by_more_than_count(counter):
    counter.increment_by(5)
    with pytest.raises(Exception):
        counter.decrement_by(10)

def test_distance_to_max(counter):
    max_count = 10**18
    assert counter.distance_to_max() == max_count

def test_reset_by_owner(counter, deployer):
    with boa.env.prank(deployer):
        contract = boa.load("contracts/Counter.vy")
        contract.increment_by(100)
        assert contract.get_count() == 100
        contract.reset()
        assert contract.get_count() == 0
```

---

## 10. การ Compile และ Deploy {#compile-deploy}

### Compile ด้วย Vyper CLI

```bash
# Compile
vyper contracts/SimpleStorage.vy

# ดู ABI
vyper -f abi contracts/SimpleStorage.vy > artifacts/SimpleStorage_abi.json

# ดู Bytecode
vyper -f bytecode contracts/SimpleStorage.vy > artifacts/SimpleStorage_bytecode.txt

# Combined Output
vyper -f combined_json contracts/SimpleStorage.vy > artifacts/SimpleStorage.json
```

### Deploy ด้วย Web3.py

```python
# scripts/deploy_web3.py
import json
from web3 import Web3
from eth_account import Account
import subprocess

def compile_contract(filepath: str) -> dict:
    """Compile Vyper contract"""
    result = subprocess.run(
        ["vyper", "-f", "combined_json", filepath],
        capture_output=True,
        text=True
    )
    
    if result.returncode != 0:
        raise Exception(f"Compilation failed: {result.stderr}")
    
    return json.loads(result.stdout)

def deploy_contract(
    w3: Web3,
    compiled: dict,
    constructor_args: list,
    deployer_key: str
):
    """Deploy compiled contract"""
    contract_name = list(compiled.keys())[0]
    contract_data = compiled[contract_name]
    
    abi = contract_data["abi"]
    bytecode = contract_data["bytecode"]
    
    # Create contract
    Contract = w3.eth.contract(abi=abi, bytecode=bytecode)
    
    # Build transaction
    deployer = Account.from_key(deployer_key)
    nonce = w3.eth.get_transaction_count(deployer.address)
    
    tx = Contract.constructor(*constructor_args).build_transaction({
        "from": deployer.address,
        "nonce": nonce,
        "gas": 2000000,
        "gasPrice": w3.eth.gas_price,
    })
    
    # Sign and send
    signed_tx = w3.eth.account.sign_transaction(tx, deployer_key)
    tx_hash = w3.eth.send_raw_transaction(signed_tx.rawTransaction)
    
    # Wait for receipt
    receipt = w3.eth.wait_for_transaction_receipt(tx_hash)
    
    return w3.eth.contract(
        address=receipt.contractAddress,
        abi=abi
    )

def main():
    # Connect to local node (Hardhat/Anvil)
    w3 = Web3(Web3.HTTPProvider("http://localhost:8545"))
    
    print(f"Connected: {w3.is_connected()}")
    print(f"Chain ID: {w3.eth.chain_id}")
    
    # Compile
    compiled = compile_contract("contracts/SimpleStorage.vy")
    print("✅ Contract compiled")
    
    # Deploy (ใช้ Test Private Key - ห้ามใช้ใน Production!)
    TEST_PRIVATE_KEY = "0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80"
    
    contract = deploy_contract(
        w3,
        compiled,
        constructor_args=[100],  # initial_value = 100
        deployer_key=TEST_PRIVATE_KEY
    )
    
    print(f"✅ Contract deployed at: {contract.address}")
    
    # Test
    value = contract.functions.get().call()
    print(f"✅ Initial value: {value}")
    
    # Set new value
    deployer = Account.from_key(TEST_PRIVATE_KEY)
    tx = contract.functions.set(999).build_transaction({
        "from": deployer.address,
        "nonce": w3.eth.get_transaction_count(deployer.address),
        "gas": 100000,
        "gasPrice": w3.eth.gas_price,
    })
    
    signed = w3.eth.account.sign_transaction(tx, TEST_PRIVATE_KEY)
    tx_hash = w3.eth.send_raw_transaction(signed.rawTransaction)
    w3.eth.wait_for_transaction_receipt(tx_hash)
    
    new_value = contract.functions.get().call()
    print(f"✅ New value: {new_value}")

if __name__ == "__main__":
    main()
```

### Deploy ด้วย Titanoboa

```python
# scripts/deploy_titanoboa.py
import boa

def main():
    # สำหรับ Local Testing (ไม่ต้อง Node จริง)
    contract = boa.load("contracts/SimpleStorage.vy", 100)
    
    print(f"Deployed at: {contract.address}")
    print(f"Initial value: {contract.get()}")
    
    # Test
    contract.set(500)
    print(f"After set(500): {contract.get()}")
    
    contract.increment()
    print(f"After increment: {contract.get()}")
    
    # ดู Gas Usage
    with boa.env.anchor():
        result = contract.set(999)
        # gas = result.gas_used (ขึ้นอยู่กับ version)

if __name__ == "__main__":
    main()
```

---

## 11. การทดสอบ {#testing}

### รัน Tests

```bash
# รัน Tests ทั้งหมด
pytest tests/ -v

# รัน Test เฉพาะไฟล์
pytest tests/test_simple_storage.py -v

# รัน Test เฉพาะ Function
pytest tests/test_simple_storage.py::TestDeployment::test_initial_value -v

# ดู Coverage
pytest tests/ --cov=contracts --cov-report=html

# รัน Tests แบบ Parallel
pytest tests/ -n auto  # ต้องติดตั้ง pytest-xdist
```

### conftest.py

```python
# tests/conftest.py
import boa
import pytest

@pytest.fixture(scope="session")
def boa_env():
    """Session-scoped Boa environment"""
    return boa.env

@pytest.fixture(autouse=True)
def reset_env(boa_env):
    """Reset environment before each test"""
    with boa_env.anchor():
        yield

# Shared accounts
@pytest.fixture
def alice():
    return boa.env.generate_address("alice")

@pytest.fixture  
def bob():
    return boa.env.generate_address("bob")

@pytest.fixture
def carol():
    return boa.env.generate_address("carol")

# Fund accounts with ETH
@pytest.fixture
def funded_alice(alice):
    boa.env.set_balance(alice, 10 * 10**18)  # 10 ETH
    return alice
```

### Property-Based Testing

```python
# tests/test_counter_property.py
import boa
import pytest
from hypothesis import given, settings, assume
from hypothesis import strategies as st

@pytest.fixture
def counter():
    return boa.load("contracts/Counter.vy")

@given(amount=st.integers(min_value=1, max_value=10**15))
@settings(max_examples=50)
def test_increment_by_always_works(counter, amount):
    """การเพิ่มค่าต้องทำงานถูกต้องเสมอ"""
    initial = counter.get_count()
    counter.increment_by(amount)
    assert counter.get_count() == initial + amount

@given(
    add_amount=st.integers(min_value=1, max_value=10**10),
    sub_amount=st.integers(min_value=1, max_value=10**10)
)
def test_increment_then_decrement(counter, add_amount, sub_amount):
    """เพิ่มแล้วลดต้องได้ค่าที่ถูกต้อง"""
    assume(add_amount >= sub_amount)
    
    counter.increment_by(add_amount)
    counter.decrement_by(sub_amount)
    
    assert counter.get_count() == add_amount - sub_amount
```

---

## สรุป Part 003

ในส่วนนี้คุณได้เรียนรู้:
- ✅ โครงสร้างพื้นฐานของ Vyper File
- ✅ Version Pragma
- ✅ State Variables vs Local Variables
- ✅ Functions และประเภทต่างๆ
- ✅ Decorators: @external, @internal, @view, @pure, @payable, @deploy
- ✅ Constructor (@deploy def __init__)
- ✅ สร้าง Contracts 3 แบบ: SimpleStorage, Greeting, Counter
- ✅ Compile และ Deploy
- ✅ การทดสอบด้วย Titanoboa และ pytest

## แบบฝึกหัด

1. **แก้ไข** SimpleStorage ให้เก็บได้หลายค่าสำหรับแต่ละ Address
2. **เพิ่ม** Function ให้ Greeting Contract รองรับหลายภาษา
3. **สร้าง** MultiCounter Contract ที่มี Counter หลายตัว
4. **ทดสอบ** ทุก Contract ให้ Coverage 100%
5. **Deploy** บน Remix IDE และทดสอบ

---

**ก่อนหน้า: [Part 002 - ติดตั้ง Development Environment](part_002_setup.md)**  
**ต่อไป: [Part 004 - ชนิดข้อมูลพื้นฐาน](part_004_types.md)**
