# Part 004: ชนิดข้อมูลพื้นฐาน (Primitive Types)

## สารบัญ
1. [Integer Types](#integers)
2. [Boolean](#boolean)
3. [Address](#address)
4. [Bytes Types](#bytes)
5. [String](#string)
6. [Decimal](#decimal)
7. [Type Conversion](#conversion)
8. [Constants และ Immutables](#constants)
9. [Type Safety](#safety)
10. [ตัวอย่างการใช้งาน](#examples)

---

## 1. Integer Types {#integers}

### Unsigned Integers (uint)

```python
# @version 0.4.0

# uint8: 0 ถึง 255 (2^8 - 1)
small_number: uint8

# uint16: 0 ถึง 65,535 (2^16 - 1)
port_number: uint16

# uint32: 0 ถึง 4,294,967,295 (2^32 - 1)
unix_timestamp: uint32

# uint64: 0 ถึง 18,446,744,073,709,551,615 (2^64 - 1)
large_value: uint64

# uint128: ใช้บ่อยใน DeFi
token_amount_small: uint128

# uint256: 0 ถึง 2^256 - 1 (ประมาณ 1.157 × 10^77)
# ค่า Default สำหรับ Integer ใน Vyper
token_amount: uint256
```

### Signed Integers (int)

```python
# @version 0.4.0

# int8: -128 ถึง 127
tiny_int: int8

# int16: -32,768 ถึง 32,767  
small_int: int16

# int128: ใช้สำหรับ Price Difference, PnL
price_change: int128

# int256: -2^255 ถึง 2^255 - 1
big_signed: int256
```

### Integer Operations

```python
# @version 0.4.0

a: uint256
b: uint256

@external
def integer_operations():
    # Arithmetic
    sum_val: uint256 = 10 + 5        # 15
    diff: uint256 = 10 - 5           # 5
    product: uint256 = 10 * 5        # 50
    quotient: uint256 = 10 / 5       # 2 (integer division)
    remainder: uint256 = 10 % 3      # 1
    power: uint256 = 2 ** 10         # 1024
    
    # Bitwise Operations
    and_val: uint256 = 0b1010 & 0b1100   # 0b1000 = 8
    or_val: uint256 = 0b1010 | 0b1100    # 0b1110 = 14
    xor_val: uint256 = 0b1010 ^ 0b1100   # 0b0110 = 6
    not_val: uint256 = ~uint256(0)        # 2^256 - 1
    shift_left: uint256 = 1 << 10        # 1024
    shift_right: uint256 = 1024 >> 1     # 512
    
    # Comparison
    is_equal: bool = 10 == 10     # True
    not_equal: bool = 10 != 5     # True
    greater: bool = 10 > 5        # True
    less: bool = 5 < 10           # True
    gte: bool = 10 >= 10          # True
    lte: bool = 10 <= 10          # True
```

### Overflow Protection

```python
# @version 0.4.0

# Vyper ป้องกัน Overflow โดย Default!
@external
def safe_math():
    # ถ้า overflow → Revert automatically
    max_uint: uint256 = max_value(uint256)
    # max_uint + 1  ← จะ Revert!
    
    # ถ้า underflow → Revert automatically
    min_uint: uint256 = 0
    # min_uint - 1  ← จะ Revert!

@pure
@external
def safe_add(a: uint256, b: uint256) -> uint256:
    # ถ้า a + b > max uint256 → Revert
    return a + b  # ปลอดภัยโดย Default

@pure
@external  
def safe_multiply(a: uint256, b: uint256) -> uint256:
    # ถ้า a * b > max uint256 → Revert
    return a * b  # ปลอดภัยโดย Default
```

### Integer Literals

```python
# @version 0.4.0

@external
def literal_examples():
    # Decimal
    dec: uint256 = 1000000
    dec_readable: uint256 = 1_000_000  # ใช้ _ แยกหลัก
    
    # Hexadecimal
    hex_val: uint256 = 0xFF           # 255
    hex_big: uint256 = 0xDEADBEEF     # 3735928559
    
    # Binary
    bin_val: uint256 = 0b11111111     # 255
    
    # Octal
    oct_val: uint256 = 0o377          # 255
    
    # Scientific (ใน Comments เท่านั้น)
    # 1e18 = 10**18 (ใช้สำหรับ Wei/ETH)
    one_ether: uint256 = 10 ** 18
    one_ether_readable: uint256 = 1_000_000_000_000_000_000
```

---

## 2. Boolean {#boolean}

```python
# @version 0.4.0

is_active: bool
is_paused: bool

@deploy
def __init__():
    self.is_active = True
    self.is_paused = False

@external
def boolean_operations():
    a: bool = True
    b: bool = False
    
    # Logical Operators
    and_result: bool = a and b   # False
    or_result: bool = a or b     # True
    not_result: bool = not a     # False
    
    # Short-circuit evaluation
    # ถ้า a เป็น False, b ไม่ถูก Evaluate
    result: bool = False and some_expensive_check()
    
    # ถ้า a เป็น True, b ไม่ถูก Evaluate
    result2: bool = True or some_expensive_check()

@internal
def some_expensive_check() -> bool:
    return True

@view
@external
def get_status() -> (bool, bool):
    return self.is_active, not self.is_paused
```

---

## 3. Address {#address}

```python
# @version 0.4.0

owner: address
treasury: address
zero_addr: address

@deploy
def __init__():
    self.owner = msg.sender
    self.treasury = 0x742d35Cc6634C0532925a3b8D4C9C3E3F2e1d70f
    self.zero_addr = empty(address)  # 0x000...000

@external
def address_operations():
    # Properties ของ Address
    addr: address = msg.sender
    
    # ดู ETH Balance
    bal: uint256 = addr.balance
    
    # ตรวจสอบว่าเป็น Zero Address
    is_zero: bool = addr == empty(address)
    
    # ตรวจสอบว่าเป็น Contract
    # (code_size > 0 = Contract)
    code_size: uint256 = addr.code_size  # หรือ addr.codesize
    is_contract: bool = addr.is_contract

@payable
@external
def send_ether_to_address(recipient: address, amount: uint256):
    """ส่ง ETH ไปยัง Address"""
    assert amount <= self.balance, "Insufficient balance"
    
    # วิธีที่ 1: send() - Simple, revert on failure
    send(recipient, amount)
    
    # วิธีที่ 2: raw_call() - More control
    # success: bool = raw_call(recipient, b"", value=amount)

@view
@external
def check_address(addr: address) -> (bool, uint256, bool):
    """
    Returns: (is_zero, balance, is_contract)
    """
    return (
        addr == empty(address),
        addr.balance,
        addr.is_contract
    )
```

### Address Type Safety

```python
# @version 0.4.0

# ✅ ถูกต้อง: Explicit address
valid_addr: address = 0x742d35Cc6634C0532925a3b8D4C9C3E3F2e1d70f

# ❌ ผิด: ต้องใช้ checksum address (EIP-55)
# invalid: address = 0x742d35cc6634c0532925a3b8d4c9c3e3f2e1d70f

# Checksum: Vyper บังคับใช้ EIP-55 Checksum
# Tools: web3.py: web3.to_checksum_address(addr)
# Online: https://ethsum.netlify.app/
```

---

## 4. Bytes Types {#bytes}

### Fixed-Size Bytes

```python
# @version 0.4.0

# bytes1 ถึง bytes32: Fixed size
tx_hash: bytes32         # Transaction Hash
keccak_result: bytes32   # Keccak256 Hash
single_byte: bytes1
sixteen_bytes: bytes16

@external
def bytes_operations():
    # bytes32 literal
    hash_val: bytes32 = 0xabcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890
    
    # Empty bytes32
    empty_hash: bytes32 = empty(bytes32)
    
    # Convert to uint256
    as_uint: uint256 = convert(hash_val, uint256)
    
    # Bitwise operations บน bytes32
    masked: bytes32 = hash_val & 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF000000000000000000000000
```

### Dynamic Bytes

```python
# @version 0.4.0

# Bytes[N]: Dynamic bytes ความยาวสูงสุด N
data: Bytes[1024]         # สูงสุด 1024 bytes
signature: Bytes[65]      # ECDSA Signature
calldata_cache: Bytes[10000]

@external
def bytes_dynamic():
    # สร้าง Bytes
    b: Bytes[10] = b"\x01\x02\x03"
    
    # ความยาว
    length: uint256 = len(b)  # 3
    
    # Slicing
    first_byte: Bytes[1] = slice(b, 0, 1)
    
    # Concat
    combined: Bytes[20] = concat(b, b"\x04\x05")
    
    # Convert Bytes ↔ String (ระวัง encoding)
    as_string: String[10] = convert(b, String[10])

@external
def hash_data(data: Bytes[1000]) -> bytes32:
    """Hash arbitrary bytes data"""
    return keccak256(data)

@external  
def verify_hash(data: Bytes[100], expected_hash: bytes32) -> bool:
    """ตรวจสอบ Hash"""
    return keccak256(data) == expected_hash
```

---

## 5. String {#string}

```python
# @version 0.4.0

# String[N]: String ความยาวสูงสุด N characters
name: String[50]
symbol: String[10]
description: String[500]
uri: String[2048]

@deploy
def __init__():
    self.name = "MyToken"
    self.symbol = "MTK"

@view
@external
def string_operations() -> String[100]:
    s1: String[20] = "Hello"
    s2: String[20] = " World"
    
    # Concatenation
    combined: String[40] = concat(s1, s2)  # "Hello World"
    
    # Length (จำนวน bytes, ไม่ใช่ characters)
    length: uint256 = len(s1)  # 5
    
    return combined

@view
@external
def build_uri(token_id: uint256) -> String[100]:
    """สร้าง Token URI"""
    # Convert uint256 to String - ต้องใช้ Helper
    # Vyper ไม่มี built-in uint to string
    return concat(
        "https://api.example.com/token/",
        uint_to_string(token_id)
    )

@pure
@internal
def uint_to_string(value: uint256) -> String[78]:
    """Convert uint256 to decimal string"""
    if value == 0:
        return "0"
    
    # Buffer สำหรับ digits
    result: String[78] = ""
    temp: uint256 = value
    
    # สร้าง digit array
    for _ in range(78):
        if temp == 0:
            break
        digit: uint256 = temp % 10
        temp = temp / 10
        # ต่อ digit เข้าไปข้างหน้า
        # Note: นี่เป็นแค่ pseudo-code แสดงแนวคิด
        # การทำจริงต้องใช้ bytes manipulation
    
    return result

@pure
@external
def compare_strings(s1: String[100], s2: String[100]) -> bool:
    """เปรียบเทียบ Strings"""
    # Vyper ไม่มี == สำหรับ String โดยตรง
    # ต้องเปรียบเทียบผ่าน keccak256
    return keccak256(convert(s1, Bytes[100])) == keccak256(convert(s2, Bytes[100]))
```

---

## 6. Decimal {#decimal}

```python
# @version 0.4.0
# หมายเหตุ: Decimal type อาจถูกลบออกใน Vyper versions ใหม่
# แนะนำใช้ Fixed-point arithmetic แทน

# Decimal: จุดทศนิยม 10 หลัก
# ช่วง: -170141183460469231731.6877907923932 ถึง 170141183460469231731.6877907923932

price: decimal
rate: decimal

@deploy
def __init__():
    self.price = 1.5
    self.rate = 0.025  # 2.5%

@view
@external
def decimal_math() -> decimal:
    a: decimal = 10.5
    b: decimal = 3.2
    
    add_result: decimal = a + b     # 13.7
    sub_result: decimal = a - b     # 7.3
    mul_result: decimal = a * b     # 33.6
    div_result: decimal = a / b     # 3.28125
    
    return mul_result

# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# Alternative: Fixed-Point Arithmetic (แนะนำ)
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

PRECISION: constant(uint256) = 10**18  # 18 decimals

@pure
@internal
def fixed_multiply(a: uint256, b: uint256) -> uint256:
    """คูณ fixed-point numbers"""
    return (a * b) / PRECISION

@pure
@internal
def fixed_divide(a: uint256, b: uint256) -> uint256:
    """หาร fixed-point numbers"""
    return (a * PRECISION) / b

@pure
@external
def calculate_percentage(amount: uint256, percentage: uint256) -> uint256:
    """
    คำนวณ Percentage
    percentage: 1e18 = 100%, 5e16 = 5%, 1e16 = 1%
    """
    return fixed_multiply(amount, percentage)
```

---

## 7. Type Conversion {#conversion}

```python
# @version 0.4.0

@external
def type_conversions():
    # uint ↔ int
    unsigned: uint256 = 100
    signed: int256 = convert(unsigned, int256)   # 100
    back: uint256 = convert(signed, uint256)      # 100 (ถ้า >= 0)
    
    # uint ↔ bool
    num: uint256 = 1
    as_bool: bool = convert(num, bool)            # True
    zero: uint256 = 0
    as_false: bool = convert(zero, bool)          # False
    
    # address ↔ uint160
    addr: address = msg.sender
    as_uint: uint160 = convert(addr, uint160)
    back_addr: address = convert(as_uint, address)
    
    # bytes ↔ uint
    b: bytes32 = 0x0000000000000000000000000000000000000000000000000000000000000001
    as_num: uint256 = convert(b, uint256)         # 1
    
    # uint ↔ bytes32
    val: uint256 = 255
    as_bytes: bytes32 = convert(val, bytes32)
    
    # String ↔ Bytes
    s: String[10] = "Hello"
    as_bytes2: Bytes[10] = convert(s, Bytes[10])
    
    # Downcast (ระวัง!)
    big: uint256 = 1000
    small: uint8 = convert(big, uint8)  # ถ้า > 255 → Revert!

@pure
@external
def safe_downcast(value: uint256) -> uint8:
    """Downcast ปลอดภัย"""
    assert value <= 255, "Value too large for uint8"
    return convert(value, uint8)

@pure
@external
def uint_to_int(value: uint256) -> int256:
    """Convert uint256 to int256"""
    # ต้องแน่ใจว่า value ไม่เกิน int256 max
    assert value <= convert(max_value(int256), uint256), "Overflow"
    return convert(value, int256)
```

---

## 8. Constants และ Immutables {#constants}

```python
# @version 0.4.0

# ════════════════════════
# CONSTANTS
# ════════════════════════
# กำหนดที่ compile-time, ไม่ใช้ Storage
MAX_SUPPLY: constant(uint256) = 21_000_000 * 10**18
DECIMALS: constant(uint8) = 18
SYMBOL: constant(String[5]) = "BTC"
WETH_ADDRESS: constant(address) = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2
ZERO_HASH: constant(bytes32) = empty(bytes32)

# Mathematical constants
PRECISION: constant(uint256) = 10**18
BASIS_POINTS: constant(uint256) = 10000  # 100%

# Time constants
SECONDS_PER_MINUTE: constant(uint256) = 60
SECONDS_PER_HOUR: constant(uint256) = 3600
SECONDS_PER_DAY: constant(uint256) = 86400
SECONDS_PER_WEEK: constant(uint256) = 604800
SECONDS_PER_YEAR: constant(uint256) = 31536000

# ════════════════════════
# IMMUTABLES
# ════════════════════════
# กำหนดใน __init__ ครั้งเดียว, เก็บใน Bytecode (ไม่ใช้ Storage)
owner: immutable(address)
deploy_time: immutable(uint256)
initial_price: immutable(uint256)
token_name: immutable(String[30])

@deploy
def __init__(price: uint256, name: String[30]):
    owner = msg.sender              # ตั้งค่า immutable
    deploy_time = block.timestamp
    initial_price = price
    token_name = name

@view
@external
def get_constants() -> (uint256, uint8, String[5]):
    return MAX_SUPPLY, DECIMALS, SYMBOL

@view
@external
def get_immutables() -> (address, uint256, uint256, String[30]):
    return owner, deploy_time, initial_price, token_name

# ════════════════════════
# การใช้งาน Constants ใน Functions
# ════════════════════════

@pure
@external
def calculate_fee(amount: uint256, fee_bps: uint256) -> uint256:
    """คำนวณ Fee ใน Basis Points"""
    # fee_bps: 100 = 1%, 10 = 0.1%, 1 = 0.01%
    return (amount * fee_bps) / BASIS_POINTS

@pure  
@external
def eth_to_wei(eth_amount: uint256) -> uint256:
    """แปลง ETH เป็น Wei"""
    return eth_amount * PRECISION

@pure
@external
def weeks_to_seconds(weeks: uint256) -> uint256:
    """แปลง Weeks เป็น Seconds"""
    return weeks * SECONDS_PER_WEEK
```

---

## 9. Type Safety {#safety}

```python
# @version 0.4.0

# Vyper มี Strong Static Typing

@external
def type_safety_demo():
    # ✅ ถูก: ประเภทตรงกัน
    a: uint256 = 100
    b: uint256 = 200
    c: uint256 = a + b
    
    # ❌ ผิด: ไม่สามารถบวก uint กับ int
    # x: uint256 = 100
    # y: int256 = -50
    # z: uint256 = x + y  ← Compile Error!
    
    # ❌ ผิด: ไม่สามารถ assign ผิดประเภท
    # num: uint256 = "hello"  ← Compile Error!
    
    # ✅ ถูก: ต้อง explicit convert
    x: uint256 = 100
    y: int256 = convert(x, int256)
    
    # Type checking ช่วยจับ Bug:
    # ถ้าไม่มี type safety อาจทำให้:
    # - Integer Overflow/Underflow
    # - Wrong address comparison
    # - Incorrect fee calculation

@pure
@external
def strictly_typed(
    amount: uint256,
    fee_rate: uint256,
    max_fee: uint256
) -> uint256:
    """Function ที่มี Type Safety เข้มงวด"""
    # fee_rate: 0-10000 (basis points)
    assert fee_rate <= 10000, "Fee rate too high"
    
    fee: uint256 = (amount * fee_rate) / 10000
    
    # Clamp to max_fee
    if fee > max_fee:
        return max_fee
    
    return fee
```

---

## 10. ตัวอย่างการใช้งาน {#examples}

### Contract: TypesDemo

```python
# @version 0.4.0
# SPDX-License-Identifier: MIT
# ไฟล์: contracts/TypesDemo.vy

"""
ตัวอย่างการใช้งาน Types ทุกประเภท
"""

# ════════════════════════
# CONSTANTS
# ════════════════════════
MAX_NAME_LENGTH: constant(uint256) = 50
GENESIS_TIME: constant(uint256) = 1609459200  # Jan 1, 2021

# ════════════════════════
# IMMUTABLES
# ════════════════════════
contract_owner: immutable(address)
contract_id: immutable(bytes32)

# ════════════════════════
# STATE VARIABLES
# ════════════════════════

# Integers
total_count: uint256
balance: uint128
small_value: uint8
temperature: int8        # อาจเป็น negative

# Boolean
is_active: bool
is_verified: bool

# Address
admin: address
fee_recipient: address

# Bytes
data_hash: bytes32
last_signature: Bytes[65]

# String
protocol_name: String[50]
version: String[10]

# Mapping (จะอธิบายเพิ่มใน Part 010)
balances: HashMap[address, uint256]
approved: HashMap[address, HashMap[address, bool]]

# ════════════════════════
# EVENTS
# ════════════════════════
event DataStored:
    key: indexed(bytes32)
    value: uint256
    timestamp: uint256

# ════════════════════════
# CONSTRUCTOR
# ════════════════════════
@deploy
def __init__(name: String[50], id_bytes: bytes32):
    contract_owner = msg.sender
    contract_id = id_bytes
    
    self.protocol_name = name
    self.version = "1.0.0"
    self.is_active = True
    self.admin = msg.sender
    self.fee_recipient = msg.sender
    self.total_count = 0

# ════════════════════════
# GETTER FUNCTIONS
# ════════════════════════

@view
@external
def get_all_state() -> (
    uint256,    # total_count
    bool,       # is_active
    address,    # admin
    String[50], # protocol_name
    String[10]  # version
):
    return (
        self.total_count,
        self.is_active,
        self.admin,
        self.protocol_name,
        self.version
    )

@view
@external
def get_contract_info() -> (address, bytes32):
    return contract_owner, contract_id

# ════════════════════════
# UTILITY FUNCTIONS  
# ════════════════════════

@pure
@external
def pack_data(
    a: uint128,
    b: uint128
) -> uint256:
    """Pack two uint128 into one uint256"""
    return convert(a, uint256) | (convert(b, uint256) << 128)

@pure
@external
def unpack_data(packed: uint256) -> (uint128, uint128):
    """Unpack uint256 into two uint128"""
    low: uint128 = convert(packed & convert(max_value(uint128), uint256), uint128)
    high: uint128 = convert(packed >> 128, uint128)
    return low, high

@pure
@external
def to_bool(value: uint256) -> bool:
    """Convert uint256 to bool"""
    return value != 0

@pure
@external
def count_bits(value: uint256) -> uint256:
    """Count set bits (popcount) ใน uint256"""
    count: uint256 = 0
    n: uint256 = value
    for _ in range(256):
        if n == 0:
            break
        count += n & 1
        n = n >> 1
    return count

@view
@external
def is_power_of_two(value: uint256) -> bool:
    """ตรวจสอบว่าเป็น Power of 2"""
    if value == 0:
        return False
    return (value & (value - 1)) == 0

@pure
@external
def min_uint(a: uint256, b: uint256) -> uint256:
    """ค่าต่ำกว่าของสอง uint256"""
    if a < b:
        return a
    return b

@pure
@external
def max_uint(a: uint256, b: uint256) -> uint256:
    """ค่าสูงกว่าของสอง uint256"""
    if a > b:
        return a
    return b

@pure
@external
def clamp(
    value: uint256,
    min_val: uint256,
    max_val: uint256
) -> uint256:
    """จำกัดค่าอยู่ระหว่าง min และ max"""
    assert min_val <= max_val, "min > max"
    
    if value < min_val:
        return min_val
    if value > max_val:
        return max_val
    return value

@pure
@external
def abs_diff(a: uint256, b: uint256) -> uint256:
    """Absolute difference ระหว่าง a และ b"""
    if a >= b:
        return a - b
    return b - a

# ════════════════════════
# HASH FUNCTIONS
# ════════════════════════

@pure
@external
def hash_address(addr: address) -> bytes32:
    return keccak256(convert(addr, bytes32))

@pure
@external
def hash_two_values(a: uint256, b: uint256) -> bytes32:
    return keccak256(
        concat(
            convert(a, bytes32),
            convert(b, bytes32)
        )
    )

@pure
@external
def compute_pair_id(token_a: address, token_b: address) -> bytes32:
    """สร้าง Unique ID จากคู่ Address (order-independent)"""
    # Sort addresses เพื่อให้ได้ผลเหมือนกันไม่ว่า order ใด
    if convert(token_a, uint160) < convert(token_b, uint160):
        return keccak256(
            concat(convert(token_a, bytes32), convert(token_b, bytes32))
        )
    else:
        return keccak256(
            concat(convert(token_b, bytes32), convert(token_a, bytes32))
        )
```

### Test TypesDemo

```python
# tests/test_types_demo.py
import boa
import pytest

@pytest.fixture
def demo():
    contract_id = b'\x01' * 32  # bytes32 = 0x0101...01
    return boa.load(
        "contracts/TypesDemo.vy",
        "MyProtocol",
        contract_id
    )

class TestPackUnpack:
    def test_pack_unpack_roundtrip(self, demo):
        a: int = 12345
        b: int = 67890
        
        packed = demo.pack_data(a, b)
        low, high = demo.unpack_data(packed)
        
        assert low == a
        assert high == b
    
    def test_pack_max_values(self, demo):
        max_128 = 2**128 - 1
        packed = demo.pack_data(max_128, max_128)
        low, high = demo.unpack_data(packed)
        
        assert low == max_128
        assert high == max_128

class TestMathHelpers:
    def test_min(self, demo):
        assert demo.min_uint(5, 10) == 5
        assert demo.min_uint(10, 5) == 5
        assert demo.min_uint(5, 5) == 5
    
    def test_max(self, demo):
        assert demo.max_uint(5, 10) == 10
        assert demo.max_uint(10, 5) == 10
    
    def test_clamp(self, demo):
        assert demo.clamp(5, 0, 10) == 5
        assert demo.clamp(15, 0, 10) == 10
        assert demo.clamp(-5, 0, 10) == 0  # underflow protection
    
    def test_abs_diff(self, demo):
        assert demo.abs_diff(10, 5) == 5
        assert demo.abs_diff(5, 10) == 5
        assert demo.abs_diff(5, 5) == 0
    
    def test_is_power_of_two(self, demo):
        assert demo.is_power_of_two(1) == True
        assert demo.is_power_of_two(2) == True
        assert demo.is_power_of_two(4) == True
        assert demo.is_power_of_two(1024) == True
        assert demo.is_power_of_two(0) == False
        assert demo.is_power_of_two(3) == False
        assert demo.is_power_of_two(5) == False

class TestHashing:
    def test_hash_consistency(self, demo):
        addr = "0x742d35Cc6634C0532925a3b8D4C9C3E3F2e1d70f"
        hash1 = demo.hash_address(addr)
        hash2 = demo.hash_address(addr)
        assert hash1 == hash2
    
    def test_pair_id_order_independent(self, demo):
        addr_a = "0x742d35Cc6634C0532925a3b8D4C9C3E3F2e1d70f"
        addr_b = "0x1f9840a85d5aF5bf1D1762F925BDADdC4201F984"
        
        id1 = demo.compute_pair_id(addr_a, addr_b)
        id2 = demo.compute_pair_id(addr_b, addr_a)
        
        assert id1 == id2  # Order ไม่สำคัญ
```

---

## สรุป Part 004

ในส่วนนี้คุณได้เรียนรู้:
- ✅ Integer Types: uint8/16/32/64/128/256 และ int8/128/256
- ✅ Boolean: True/False และ Logical Operators
- ✅ Address: Properties, ETH Balance, is_contract
- ✅ Bytes Types: bytes1-32 (fixed) และ Bytes[N] (dynamic)
- ✅ String[N]: String handling ใน Vyper
- ✅ Decimal: Fixed-point arithmetic
- ✅ Type Conversion: convert() function
- ✅ Constants และ Immutables
- ✅ Type Safety ของ Vyper

## แบบฝึกหัด

1. **สร้าง** Contract ที่เก็บข้อมูลส่วนตัว (ชื่อ, อายุ, address) ด้วย Types ที่เหมาะสม
2. **เขียน** Functions แปลง Wei ↔ ETH (1 ETH = 10^18 Wei)
3. **สร้าง** Bit manipulation Library (pack, unpack, count bits)
4. **ทดสอบ** Overflow Protection ของ Vyper
5. **เปรียบเทียบ** Gas Cost ระหว่างการใช้ uint8 vs uint256

---

**ก่อนหน้า: [Part 003 - Smart Contract แรก](part_003_first_contract.md)**  
**ต่อไป: [Part 005 - ตัวแปรและ State Variables](part_005_variables.md)**
