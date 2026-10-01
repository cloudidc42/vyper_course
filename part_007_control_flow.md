# Part 007: Control Flow: If/Elif/Else

## สารบัญ
1. [If/Elif/Else Syntax](#if-elif-else)
2. [Comparison Operators](#comparison)
3. [Logical Operators: and, or, not](#logical)
4. [Ternary-Style Patterns](#ternary)
5. [Short-Circuit Evaluation](#short-circuit)
6. [assert vs raise](#assert-raise)
7. [ตัวอย่าง: Tiered Pricing Contract](#example)

---

## 1. If/Elif/Else Syntax {#if-elif-else}

Vyper ใช้ Syntax เหมือน Python สำหรับ Control Flow

### พื้นฐาน If/Elif/Else

```python
# @version 0.4.0

# ════════════════════════════════════════
# Basic If/Elif/Else
# ════════════════════════════════════════

@pure
@external
def classify_number(n: int256) -> String[10]:
    if n > 0:
        return "positive"
    elif n < 0:
        return "negative"
    else:
        return "zero"

@pure
@external
def classify_amount(amount: uint256) -> String[20]:
    if amount == 0:
        return "empty"
    elif amount < 100:
        return "small"
    elif amount < 1000:
        return "medium"
    elif amount < 10000:
        return "large"
    else:
        return "very_large"
```

### Nested If Statements

```python
# @version 0.4.0

owner: address
balances: HashMap[address, uint256]
whitelist: HashMap[address, bool]
blacklist: HashMap[address, bool]
paused: bool
min_amount: uint256
max_amount: uint256

@deploy
def __init__():
    self.owner = msg.sender
    self.min_amount = 100
    self.max_amount = 1000000

@external
def transfer(to: address, amount: uint256) -> bool:
    # Nested conditionals
    if self.paused:
        return False
    else:
        if self.blacklist[msg.sender]:
            return False
        else:
            if amount < self.min_amount or amount > self.max_amount:
                return False
            else:
                if self.balances[msg.sender] < amount:
                    return False
                else:
                    self.balances[msg.sender] -= amount
                    self.balances[to] += amount
                    return True

# ✅ ดีกว่า: Early Return Pattern
@external
def transfer_clean(to: address, amount: uint256) -> bool:
    if self.paused:
        return False
    if self.blacklist[msg.sender]:
        return False
    if amount < self.min_amount or amount > self.max_amount:
        return False
    if self.balances[msg.sender] < amount:
        return False

    self.balances[msg.sender] -= amount
    self.balances[to] += amount
    return True
```

### If ใน Storage Operations

```python
# @version 0.4.0

struct UserTier:
    tier: uint8    # 0=Free, 1=Basic, 2=Premium, 3=VIP
    discount: uint256  # basis points

users: HashMap[address, UserTier]

@deploy
def __init__():
    pass

@external
def set_user_tier(user: address, tier: uint8):
    discount: uint256 = 0
    if tier == 0:
        discount = 0      # Free: no discount
    elif tier == 1:
        discount = 500    # Basic: 5%
    elif tier == 2:
        discount = 1500   # Premium: 15%
    elif tier == 3:
        discount = 3000   # VIP: 30%
    else:
        assert False, "Invalid tier"

    self.users[user] = UserTier({tier: tier, discount: discount})

@view
@external
def calculate_price(user: address, base_price: uint256) -> uint256:
    tier_info: UserTier = self.users[user]
    if tier_info.discount == 0:
        return base_price
    discounted: uint256 = (base_price * tier_info.discount) / 10000
    return base_price - discounted
```

---

## 2. Comparison Operators {#comparison}

```python
# @version 0.4.0

# ════════════════════════════════════════
# Comparison Operators
# ════════════════════════════════════════

# == : เท่ากัน
# != : ไม่เท่ากัน
# >  : มากกว่า
# >= : มากกว่าหรือเท่ากัน
# <  : น้อยกว่า
# <= : น้อยกว่าหรือเท่ากัน

@pure
@external
def demo_comparisons(a: uint256, b: uint256) -> bool:
    equal: bool = (a == b)
    not_equal: bool = (a != b)
    greater: bool = (a > b)
    greater_eq: bool = (a >= b)
    less: bool = (a < b)
    less_eq: bool = (a <= b)
    return equal

# Address comparison
@view
@external
def is_owner(addr: address) -> bool:
    owner: address = 0x0000000000000000000000000000000000000000
    return addr != empty(address) and addr == owner

# String comparison (ใน Vyper ไม่ได้โดยตรง ต้องใช้ keccak256)
@pure
@external
def strings_equal(a: String[50], b: String[50]) -> bool:
    return keccak256(a) == keccak256(b)

# Bytes comparison
@pure
@external
def bytes_equal(a: bytes32, b: bytes32) -> bool:
    return a == b

# Struct comparison (field by field)
struct Point:
    x: uint256
    y: uint256

@pure
@external
def points_equal(p1: Point, p2: Point) -> bool:
    return p1.x == p2.x and p1.y == p2.y
```

### Comparison กับ Special Values

```python
# @version 0.4.0

owner: address
token: address

@deploy
def __init__():
    self.owner = msg.sender

@view
@external
def check_address(addr: address) -> String[30]:
    if addr == empty(address):       # เช็ค zero address
        return "zero_address"
    elif addr == self:               # เช็คว่าเป็น contract นี้เอง
        return "this_contract"
    elif addr == self.owner:         # เช็คว่าเป็น owner
        return "owner"
    else:
        return "other"

@pure
@external
def check_max_uint(value: uint256) -> bool:
    MAX: constant(uint256) = max_value(uint256)
    return value == MAX

@pure
@external
def is_in_range(value: uint256, min_v: uint256, max_v: uint256) -> bool:
    return value >= min_v and value <= max_v
```

---

## 3. Logical Operators: and, or, not {#logical}

```python
# @version 0.4.0

# ════════════════════════════════════════
# Logical Operators
# ════════════════════════════════════════

owner: address
admins: HashMap[address, bool]
paused: bool
maintenance: bool

@deploy
def __init__():
    self.owner = msg.sender

# AND: ทั้งสองต้องเป็น True
@view
@external
def is_active_owner() -> bool:
    return msg.sender == self.owner and not self.paused

# OR: อย่างน้อยหนึ่งเป็น True
@view
@external
def is_authorized() -> bool:
    return msg.sender == self.owner or self.admins[msg.sender]

# NOT: กลับค่า bool
@view
@external
def is_operational() -> bool:
    return not self.paused and not self.maintenance

# Complex combinations
@view
@external
def can_execute(user: address, amount: uint256) -> bool:
    is_auth: bool = user == self.owner or self.admins[user]
    is_valid: bool = amount > 0 and amount <= 10000
    is_running: bool = not self.paused and not self.maintenance
    return is_auth and is_valid and is_running

# Multiple conditions
@pure
@external
def validate_address_and_amount(addr: address, amount: uint256) -> String[50]:
    if addr == empty(address) or amount == 0:
        return "invalid_inputs"
    elif amount < 100 or amount > 1000000:
        return "amount_out_of_range"
    elif not (addr != empty(address) and amount > 0):
        return "unreachable"  # This won't happen
    else:
        return "valid"
```

### De Morgan's Laws (กฎ De Morgan)

```python
# @version 0.4.0

# not (A and B) == not A or not B
# not (A or B)  == not A and not B

@pure
@external
def demo_demorgan(a: bool, b: bool) -> (bool, bool, bool, bool):
    # These pairs should always be equal
    pair1_left: bool = not (a and b)
    pair1_right: bool = not a or not b

    pair2_left: bool = not (a or b)
    pair2_right: bool = not a and not b

    return pair1_left, pair1_right, pair2_left, pair2_right
```

---

## 4. Ternary-Style Patterns {#ternary}

Vyper ไม่มี Ternary Operator (`x if cond else y`) ต้องใช้ If/Else แทน

```python
# @version 0.4.0

# ════════════════════════════════════════
# Ternary-Style Patterns
# ════════════════════════════════════════

# Python: result = a if condition else b
# Vyper: ต้องใช้ if/else block

@pure
@external
def get_larger(a: uint256, b: uint256) -> uint256:
    if a >= b:
        return a
    return b

@pure
@external
def get_sign(n: int256) -> int256:
    if n > 0:
        return 1
    elif n < 0:
        return -1
    return 0

@pure
@external
def apply_cap(value: uint256, cap: uint256) -> uint256:
    if value > cap:
        return cap
    return value

# Pattern: Conditional Assignment
@view
@external
def get_effective_fee(base_fee: uint256, is_premium: bool) -> uint256:
    discount: uint256 = 0
    if is_premium:
        discount = base_fee / 2
    return base_fee - discount

# Pattern: Multi-branch assignment
@pure
@external
def categorize_and_process(amount: uint256) -> uint256:
    multiplier: uint256 = 1
    if amount < 1000:
        multiplier = 1
    elif amount < 10000:
        multiplier = 2
    elif amount < 100000:
        multiplier = 3
    else:
        multiplier = 5
    return amount * multiplier

# Pattern: Safe operation with fallback
@view
@external
def safe_divide(a: uint256, b: uint256) -> uint256:
    if b == 0:
        return 0  # fallback value
    return a / b
```

---

## 5. Short-Circuit Evaluation {#short-circuit}

Vyper ใช้ Short-Circuit Evaluation เหมือน Python

- `A and B`: ถ้า A เป็น False จะไม่ประเมิน B
- `A or B`: ถ้า A เป็น True จะไม่ประเมิน B

```python
# @version 0.4.0

# ════════════════════════════════════════
# Short-Circuit Evaluation
# ════════════════════════════════════════

balances: HashMap[address, uint256]
allowances: HashMap[address, HashMap[address, uint256]]

@view
@external
def can_transfer_from(
    spender: address,
    owner_addr: address,
    amount: uint256
) -> bool:
    # Short-circuit: ถ้า amount == 0 จะไม่เช็ค balance
    return amount > 0 and self.balances[owner_addr] >= amount

@view
@external
def has_access(user: address, token_id: uint256) -> bool:
    owner_addr: address = empty(address)  # placeholder
    # Short-circuit: ถ้าเป็น owner ไม่ต้องเช็ค allowance
    return user == owner_addr or self.allowances[owner_addr][user] > 0

# Pattern: Guard with short-circuit
@external
def process_if_valid(amount: uint256, recipient: address) -> bool:
    # เช็ค recipient ก่อน เพราะถูกกว่า
    if recipient == empty(address) or amount == 0:
        return False

    # เช็ค balance หลัง (เข้าถึง storage = แพงกว่า)
    if self.balances[msg.sender] < amount:
        return False

    self.balances[msg.sender] -= amount
    self.balances[recipient] += amount
    return True
```

---

## 6. assert vs raise {#assert-raise}

### assert

ตรวจสอบเงื่อนไข ถ้าเป็น False จะ revert พร้อม error message

```python
# @version 0.4.0

# ════════════════════════════════════════
# assert Statement
# ════════════════════════════════════════

owner: address
paused: bool
balances: HashMap[address, uint256]
MAX_SUPPLY: constant(uint256) = 10**27

@deploy
def __init__():
    self.owner = msg.sender

@internal
def _only_owner():
    assert msg.sender == self.owner, "Not owner"

@internal
def _not_paused():
    assert not self.paused, "Contract paused"

@internal
def _valid_address(addr: address):
    assert addr != empty(address), "Zero address"

@internal
def _sufficient_balance(addr: address, amount: uint256):
    assert self.balances[addr] >= amount, "Insufficient balance"

@external
def withdraw(amount: uint256):
    self._only_owner()
    self._not_paused()
    assert amount > 0, "Amount must be positive"
    assert amount <= self.balance, "Amount exceeds contract balance"
    send(msg.sender, amount)

@external
def mint(to: address, amount: uint256):
    self._only_owner()
    self._valid_address(to)
    assert amount > 0, "Zero mint"
    assert self.balances[to] + amount <= MAX_SUPPLY, "Exceeds max supply"
    self.balances[to] += amount

# assert ใน loop
@external
def batch_check(amounts: DynArray[uint256, 10]) -> uint256:
    total: uint256 = 0
    for amount: uint256 in amounts:
        assert amount > 0, "Zero amount in batch"
        assert amount <= 10000, "Amount too large"
        total += amount
    return total
```

### raise

ใช้สำหรับ Custom Errors (Vyper 0.4.0)

```python
# @version 0.4.0

# ════════════════════════════════════════
# raise Statement
# ════════════════════════════════════════

# Custom error types
struct InsufficientBalance:
    have: uint256
    need: uint256

owner: address
balances: HashMap[address, uint256]

@deploy
def __init__():
    self.owner = msg.sender

@external
def withdraw(amount: uint256):
    if msg.sender != self.owner:
        raise "Unauthorized: caller is not owner"

    if amount == 0:
        raise "InvalidAmount: cannot withdraw zero"

    if self.balance < amount:
        raise "InsufficientFunds: not enough ETH in contract"

    send(msg.sender, amount)

# raise vs assert comparison
@pure
@external
def validate_input_assert(value: uint256) -> uint256:
    assert value > 0, "Value must be positive"  # Revert with message
    assert value <= 10000, "Value too large"
    return value * 2

@pure
@external
def validate_input_raise(value: uint256) -> uint256:
    if value == 0:
        raise "ValueError: value must be positive"
    if value > 10000:
        raise "ValueError: value too large"
    return value * 2
```

### เมื่อใช้ assert vs raise

```python
# @version 0.4.0

# ════════════════════════════════════════
# assert vs raise Use Cases
# ════════════════════════════════════════

owner: address
balances: HashMap[address, uint256]
nonces: HashMap[address, uint256]

@deploy
def __init__():
    self.owner = msg.sender

# ✅ ใช้ assert: สำหรับ preconditions ที่ชัดเจน
@external
def transfer(to: address, amount: uint256):
    assert to != empty(address), "Transfer to zero address"
    assert amount > 0, "Zero transfer"
    assert self.balances[msg.sender] >= amount, "Insufficient balance"
    self.balances[msg.sender] -= amount
    self.balances[to] += amount

# ✅ ใช้ raise: สำหรับ complex conditional errors
@external
def complex_operation(user: address, value: uint256, nonce: uint256):
    if user == empty(address):
        raise "InvalidUser: zero address"

    if nonce != self.nonces[user]:
        raise "InvalidNonce: nonce mismatch"

    if value == 0 or value > 10**20:
        raise "InvalidValue: out of acceptable range"

    self.nonces[user] += 1
    self.balances[user] += value
```

---

## 7. ตัวอย่าง: Tiered Pricing Contract {#example}

Contract จัดการระบบราคาแบบ Tier ที่ซับซ้อน

```python
# @version 0.4.0

# ════════════════════════════════════════════════════════════
# TieredPricing Contract
# ระบบราคาแบบ Tier สำหรับ Marketplace
# ════════════════════════════════════════════════════════════

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Enums (using constants)
# ━━━━━━━━━━━━━━━━━━━━━━━━━
TIER_FREE: constant(uint8) = 0
TIER_BASIC: constant(uint8) = 1
TIER_PRO: constant(uint8) = 2
TIER_ENTERPRISE: constant(uint8) = 3

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Constants
# ━━━━━━━━━━━━━━━━━━━━━━━━━
BASE_PRICE: constant(uint256) = 100 * 10**18      # 100 tokens
BULK_THRESHOLD: constant(uint256) = 10            # Orders >= 10 = bulk
LARGE_THRESHOLD: constant(uint256) = 100          # Orders >= 100 = large
SEASONAL_DISCOUNT: constant(uint256) = 500        # 5% seasonal discount
MAX_DISCOUNT_BPS: constant(uint256) = 5000        # Max 50% discount

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Structs
# ━━━━━━━━━━━━━━━━━━━━━━━━━
struct UserProfile:
    tier: uint8
    total_spent: uint256
    order_count: uint256
    is_partner: bool
    joined_at: uint256

struct PriceCalculation:
    base_price: uint256
    tier_discount: uint256
    volume_discount: uint256
    seasonal_discount: uint256
    partner_discount: uint256
    final_price: uint256
    total_discount_bps: uint256

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# State Variables
# ━━━━━━━━━━━━━━━━━━━━━━━━━
owner: address
users: HashMap[address, UserProfile]
seasonal_sale_active: bool
price_per_unit: uint256
revenue: uint256

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Events
# ━━━━━━━━━━━━━━━━━━━━━━━━━
event PurchaseMade:
    buyer: indexed(address)
    quantity: uint256
    final_price: uint256
    discount_bps: uint256

event TierUpgraded:
    user: indexed(address)
    old_tier: uint8
    new_tier: uint8

event SeasonalSaleToggled:
    active: bool

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Constructor
# ━━━━━━━━━━━━━━━━━━━━━━━━━
@deploy
def __init__(initial_price: uint256):
    self.owner = msg.sender
    self.price_per_unit = initial_price

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Internal Helper Functions
# ━━━━━━━━━━━━━━━━━━━━━━━━━

@pure
@internal
def _get_tier_discount(tier: uint8) -> uint256:
    """Get discount in basis points based on tier"""
    if tier == TIER_FREE:
        return 0
    elif tier == TIER_BASIC:
        return 500    # 5%
    elif tier == TIER_PRO:
        return 1500   # 15%
    elif tier == TIER_ENTERPRISE:
        return 3000   # 30%
    return 0

@pure
@internal
def _get_volume_discount(quantity: uint256) -> uint256:
    """Get volume discount in basis points"""
    if quantity >= LARGE_THRESHOLD:
        return 1000   # 10% for large orders
    elif quantity >= BULK_THRESHOLD:
        return 500    # 5% for bulk orders
    return 0

@pure
@internal
def _apply_discount(price: uint256, discount_bps: uint256) -> uint256:
    """Apply discount in basis points to price"""
    if discount_bps == 0:
        return price
    discount_amount: uint256 = (price * discount_bps) / 10000
    return price - discount_amount

@pure
@internal
def _cap_discount(total_bps: uint256) -> uint256:
    """Cap total discount at MAX_DISCOUNT_BPS"""
    if total_bps > MAX_DISCOUNT_BPS:
        return MAX_DISCOUNT_BPS
    return total_bps

@internal
def _update_user_tier(user: address):
    """Auto-upgrade tier based on spending"""
    total_spent: uint256 = self.users[user].total_spent
    current_tier: uint8 = self.users[user].tier

    new_tier: uint8 = current_tier

    if total_spent >= 10000 * 10**18:
        new_tier = TIER_ENTERPRISE
    elif total_spent >= 1000 * 10**18:
        new_tier = TIER_PRO
    elif total_spent >= 100 * 10**18:
        new_tier = TIER_BASIC

    if new_tier != current_tier:
        self.users[user].tier = new_tier
        log TierUpgraded(user, current_tier, new_tier)

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# External View Functions
# ━━━━━━━━━━━━━━━━━━━━━━━━━

@view
@external
def calculate_price(
    buyer: address,
    quantity: uint256
) -> PriceCalculation:
    """Calculate full price breakdown for a purchase"""
    assert quantity > 0, "Quantity must be positive"

    profile: UserProfile = self.users[buyer]
    base_total: uint256 = self.price_per_unit * quantity

    # Calculate individual discounts
    tier_disc: uint256 = self._get_tier_discount(profile.tier)
    volume_disc: uint256 = self._get_volume_discount(quantity)
    seasonal_disc: uint256 = 0
    partner_disc: uint256 = 0

    if self.seasonal_sale_active:
        seasonal_disc = SEASONAL_DISCOUNT

    if profile.is_partner:
        partner_disc = 1000  # 10% partner discount

    # Sum and cap discounts
    total_disc: uint256 = self._cap_discount(
        tier_disc + volume_disc + seasonal_disc + partner_disc
    )
    final: uint256 = self._apply_discount(base_total, total_disc)

    return PriceCalculation({
        base_price: base_total,
        tier_discount: tier_disc,
        volume_discount: volume_disc,
        seasonal_discount: seasonal_disc,
        partner_discount: partner_disc,
        final_price: final,
        total_discount_bps: total_disc
    })

@view
@external
def get_user_tier_name(user: address) -> String[15]:
    tier: uint8 = self.users[user].tier
    if tier == TIER_FREE:
        return "Free"
    elif tier == TIER_BASIC:
        return "Basic"
    elif tier == TIER_PRO:
        return "Pro"
    elif tier == TIER_ENTERPRISE:
        return "Enterprise"
    return "Unknown"

@view
@external
def can_use_feature(user: address, feature_id: uint8) -> bool:
    """Check if user's tier allows access to a feature"""
    tier: uint8 = self.users[user].tier

    if feature_id == 0:  # Basic feature
        return True  # All tiers

    elif feature_id == 1:  # Advanced feature
        return tier >= TIER_BASIC

    elif feature_id == 2:  # Pro feature
        return tier >= TIER_PRO

    elif feature_id == 3:  # Enterprise feature
        return tier == TIER_ENTERPRISE

    return False

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# External Write Functions
# ━━━━━━━━━━━━━━━━━━━━━━━━━

@payable
@external
def purchase(quantity: uint256):
    """Make a purchase with tiered pricing"""
    assert quantity > 0, "Quantity must be positive"
    assert quantity <= 1000, "Max 1000 per transaction"

    # Register user if new
    if self.users[msg.sender].joined_at == 0:
        self.users[msg.sender] = UserProfile({
            tier: TIER_FREE,
            total_spent: 0,
            order_count: 0,
            is_partner: False,
            joined_at: block.timestamp
        })

    # Get price calculation
    calc: PriceCalculation = self.calculate_price(msg.sender, quantity)

    # Verify payment
    assert msg.value >= calc.final_price, "Insufficient payment"

    # Update user stats
    self.users[msg.sender].total_spent += calc.final_price
    self.users[msg.sender].order_count += quantity

    # Auto-upgrade tier
    self._update_user_tier(msg.sender)

    # Refund excess payment
    if msg.value > calc.final_price:
        send(msg.sender, msg.value - calc.final_price)

    # Track revenue
    self.revenue += calc.final_price

    log PurchaseMade(msg.sender, quantity, calc.final_price, calc.total_discount_bps)

@external
def set_partner(user: address, status: bool):
    """Set partner status (owner only)"""
    assert msg.sender == self.owner, "Not owner"
    assert user != empty(address), "Invalid address"
    self.users[user].is_partner = status

@external
def toggle_seasonal_sale():
    """Toggle seasonal discount (owner only)"""
    assert msg.sender == self.owner, "Not owner"
    self.seasonal_sale_active = not self.seasonal_sale_active
    log SeasonalSaleToggled(self.seasonal_sale_active)

@external
def set_price(new_price: uint256):
    """Update base price (owner only)"""
    assert msg.sender == self.owner, "Not owner"
    assert new_price > 0, "Price must be positive"
    self.price_per_unit = new_price

@external
def withdraw_revenue():
    """Withdraw collected revenue (owner only)"""
    assert msg.sender == self.owner, "Not owner"
    amount: uint256 = self.balance
    assert amount > 0, "Nothing to withdraw"
    self.revenue = 0
    send(self.owner, amount)

@view
@external
def get_user_profile(user: address) -> UserProfile:
    return self.users[user]
```

### Test Code

```python
# tests/test_tiered_pricing.py
import pytest
from brownie import TieredPricing, accounts, Wei

PRICE = Wei("1 ether")  # 1 ETH per unit

@pytest.fixture
def pricing(accounts):
    return TieredPricing.deploy(PRICE, {'from': accounts[0]})

class TestTierDiscounts:
    def test_free_tier_no_discount(self, pricing, accounts):
        calc = pricing.calculate_price(accounts[1], 1)
        assert calc[0] == PRICE  # base = price
        assert calc[6] == 0  # no discount

    def test_bulk_discount(self, pricing, accounts):
        calc = pricing.calculate_price(accounts[1], 10)  # >= 10 = bulk
        assert calc[2] == 500  # 5% volume discount

    def test_large_order_discount(self, pricing, accounts):
        calc = pricing.calculate_price(accounts[1], 100)
        assert calc[2] == 1000  # 10% volume discount

    def test_seasonal_discount(self, pricing, accounts):
        pricing.toggle_seasonal_sale({'from': accounts[0]})
        calc = pricing.calculate_price(accounts[1], 1)
        assert calc[3] == 500  # 5% seasonal

    def test_partner_discount(self, pricing, accounts):
        pricing.set_partner(accounts[1], True, {'from': accounts[0]})
        calc = pricing.calculate_price(accounts[1], 1)
        assert calc[4] == 1000  # 10% partner

    def test_max_discount_cap(self, pricing, accounts):
        # Enable all discounts
        pricing.toggle_seasonal_sale({'from': accounts[0]})
        pricing.set_partner(accounts[1], True, {'from': accounts[0]})
        # Large order (10%), partner (10%), seasonal (5%) = 25% < 50% cap
        calc = pricing.calculate_price(accounts[1], 100)
        assert calc[6] <= 5000  # Max 50%

class TestAutoTierUpgrade:
    def test_upgrade_to_basic(self, pricing, accounts):
        # Buy enough to reach Basic tier (100 token minimum)
        pricing.purchase(1, {'from': accounts[1], 'value': PRICE})
        # Would need multiple purchases to reach 100 ETH threshold

    def test_tier_name(self, pricing, accounts):
        assert pricing.get_user_tier_name(accounts[1]) == "Free"

class TestFeatureAccess:
    def test_basic_feature_all_tiers(self, pricing, accounts):
        assert pricing.can_use_feature(accounts[1], 0) == True

    def test_pro_feature_blocked_for_free(self, pricing, accounts):
        assert pricing.can_use_feature(accounts[1], 2) == False
```

---

## สรุป

- ✅ **if/elif/else**: Control flow พื้นฐาน เหมือน Python
- ✅ **Comparison operators**: ==, !=, >, >=, <, <=
- ✅ **Logical operators**: and, or, not
- ✅ **ไม่มี Ternary Operator**: ใช้ if/else block แทน
- ✅ **Short-circuit evaluation**: ตรวจสอบ cheap condition ก่อน
- ✅ **assert**: สำหรับ preconditions, revert with message
- ✅ **raise**: สำหรับ conditional errors ที่ซับซ้อน

## แบบฝึกหัด

1. **สร้าง** Grading System ที่แปลงคะแนน 0-100 เป็นเกรด A-F
2. **ปรับปรุง** TieredPricing ให้มี Time-based tiers (ราคาเพิ่มขึ้นตามเวลา)
3. **สร้าง** Multi-sig Transaction Validator ที่ต้องได้รับ approval อย่างน้อย 2/3
4. **ทดสอบ** Edge cases ของ assert (empty address, overflow, underflow)
5. **วิเคราะห์** Gas cost ของ if/elif chain vs nested if

---

**ก่อนหน้า: [Part 006 - ฟังก์ชันและ Visibility](part_006_functions.md)**  
**ต่อไป: [Part 008 - Loops: For Loop](part_008_loops.md)**
