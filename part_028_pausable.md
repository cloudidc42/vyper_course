# Part 028: Pausable Pattern

## สารบัญ
1. [บทนำ Pausable Pattern](#บทนำ)
2. [Pause/Unpause Mechanism](#pauseunpause-mechanism)
3. [Circuit Breaker](#circuit-breaker)
4. [Emergency Stop](#emergency-stop)
5. [Partial Pause](#partial-pause)
6. [ตัวอย่าง: Pausable DEX](#ตัวอย่าง-pausable-dex)
7. [Test Code](#test-code)

---

## บทนำ

**Pausable Pattern** เป็น Design Pattern ที่ให้สามารถหยุด (pause) การทำงานของ Contract ได้ชั่วคราวในกรณีฉุกเฉิน เช่น พบ bug, ถูก hack, หรือต้องการอัปเดต

### เมื่อไหร่ควรใช้ Pausable?
- DeFi Protocol ที่มีมูลค่าสูง
- Contract ที่ยังอยู่ในช่วง Beta
- ระบบที่ต้องการ emergency stop
- Protocol ที่มีการอัปเดตบ่อย

### Trade-offs
- **ข้อดี**: ป้องกันความเสียหายในกรณีฉุกเฉิน
- **ข้อเสีย**: ทำให้ Contract มี centralized control บางส่วน

---

## Pause/Unpause Mechanism

### โครงสร้างพื้นฐาน

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Basic Pausable

event Paused:
    account: indexed(address)

event Unpaused:
    account: indexed(address)

owner: public(address)
paused: public(bool)

@deploy
def __init__():
    self.owner = msg.sender
    self.paused = False

@internal
def _require_not_paused():
    """ตรวจสอบว่า contract ไม่ได้ pause อยู่"""
    assert not self.paused, "Pausable: paused"

@internal
def _require_paused():
    """ตรวจสอบว่า contract pause อยู่"""
    assert self.paused, "Pausable: not paused"

@external
def pause():
    """หยุดการทำงาน (เฉพาะ owner)"""
    assert msg.sender == self.owner, "Not owner"
    self._require_not_paused()
    self.paused = True
    log Paused(msg.sender)

@external
def unpause():
    """เริ่มการทำงานใหม่ (เฉพาะ owner)"""
    assert msg.sender == self.owner, "Not owner"
    self._require_paused()
    self.paused = False
    log Unpaused(msg.sender)

@external
def sensitive_action():
    """ฟังก์ชันที่ต้องการ not-paused state"""
    self._require_not_paused()
    # logic สำคัญ
    pass
```

---

## Circuit Breaker

### Auto-Pause เมื่อเกิดเหตุการณ์ผิดปกติ

Circuit Breaker Pattern จะ pause contract โดยอัตโนมัติเมื่อตรวจพบเหตุการณ์ผิดปกติ

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Circuit Breaker

event CircuitBreakerTripped:
    reason: String[100]
    block_number: uint256

event Paused:
    account: indexed(address)

event Unpaused:
    account: indexed(address)

owner: public(address)
paused: public(bool)

# Circuit breaker parameters
daily_withdrawal_limit: public(uint256)   # จำกัดการถอนต่อวัน
daily_withdrawn: public(uint256)          # ถอนไปแล้ววันนี้
last_withdrawal_day: public(uint256)      # วันสุดท้ายที่ถอน
total_balance: public(uint256)            # ยอดคงเหลือทั้งหมด
min_balance_threshold: public(uint256)    # ยอดขั้นต่ำก่อน auto-pause

@deploy
def __init__(_daily_limit: uint256, _min_threshold: uint256):
    self.owner = msg.sender
    self.daily_withdrawal_limit = _daily_limit
    self.min_balance_threshold = _min_threshold

@internal
def _check_and_update_daily_limit(amount: uint256):
    """
    ตรวจสอบ daily withdrawal limit และอัปเดตตัวนับ
    """
    current_day: uint256 = block.timestamp / 86400  # วินาทีต่อวัน
    
    if current_day > self.last_withdrawal_day:
        # รีเซ็ต daily counter เมื่อขึ้นวันใหม่
        self.daily_withdrawn = 0
        self.last_withdrawal_day = current_day
    
    new_total: uint256 = self.daily_withdrawn + amount
    
    if new_total > self.daily_withdrawal_limit:
        # Circuit breaker: ถอนเกินลิมิต -> auto pause
        self.paused = True
        log CircuitBreakerTripped(
            "Daily withdrawal limit exceeded",
            block.number
        )
        assert False, "Circuit breaker: daily limit exceeded"
    
    self.daily_withdrawn = new_total

@internal
def _check_balance_threshold():
    """
    ตรวจสอบ balance threshold
    """
    if self.total_balance < self.min_balance_threshold:
        self.paused = True
        log CircuitBreakerTripped(
            "Balance below minimum threshold",
            block.number
        )
        assert False, "Circuit breaker: balance too low"

@external
@payable
def deposit():
    """ฝากเงิน"""
    assert not self.paused, "Contract is paused"
    self.total_balance += msg.value

@external
def withdraw(amount: uint256):
    """ถอนเงิน พร้อม circuit breaker checks"""
    assert not self.paused, "Contract is paused"
    assert self.total_balance >= amount, "Insufficient balance"
    
    # ตรวจสอบ circuit breakers
    self._check_and_update_daily_limit(amount)
    
    self.total_balance -= amount
    
    # ตรวจสอบ balance threshold หลังถอน
    self._check_balance_threshold()
    
    send(msg.sender, amount)

@external
def pause():
    """Manual pause"""
    assert msg.sender == self.owner, "Not owner"
    self.paused = True
    log Paused(msg.sender)

@external
def unpause():
    """Manual unpause (เฉพาะ owner)"""
    assert msg.sender == self.owner, "Not owner"
    self.paused = False
    log Unpaused(msg.sender)

@external
def update_daily_limit(new_limit: uint256):
    """อัปเดต daily withdrawal limit"""
    assert msg.sender == self.owner, "Not owner"
    self.daily_withdrawal_limit = new_limit

@external
def update_min_threshold(new_threshold: uint256):
    """อัปเดต minimum balance threshold"""
    assert msg.sender == self.owner, "Not owner"
    self.min_balance_threshold = new_threshold
```

---

## Emergency Stop

### ระบบ Emergency Stop หลายระดับ

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Emergency Stop System

event EmergencyStop:
    level: uint256
    caller: indexed(address)
    reason: String[200]

event EmergencyResume:
    caller: indexed(address)

# Emergency levels
LEVEL_NORMAL: constant(uint256) = 0
LEVEL_CAUTION: constant(uint256) = 1   # จำกัดบางฟังก์ชัน
LEVEL_WARNING: constant(uint256) = 2   # จำกัดส่วนใหญ่
LEVEL_EMERGENCY: constant(uint256) = 3 # หยุดทุกอย่าง

owner: public(address)
emergency_level: public(uint256)

# Roles
guardians: public(HashMap[address, bool])  # คนที่ trigger emergency ได้
council: public(HashMap[address, bool])    # คนที่ resume ได้
council_count: public(uint256)
resume_approvals: public(HashMap[address, bool])
resume_approval_count: public(uint256)
required_approvals: public(uint256)

@deploy
def __init__(_required_approvals: uint256):
    self.owner = msg.sender
    self.emergency_level = LEVEL_NORMAL
    self.required_approvals = _required_approvals
    self.guardians[msg.sender] = True
    self.council[msg.sender] = True
    self.council_count = 1

@internal
def _check_not_emergency():
    assert self.emergency_level == LEVEL_NORMAL, "Emergency stop active"

@internal
def _check_caution_ok():
    assert self.emergency_level < LEVEL_WARNING, "Too high emergency level"

@external
def trigger_emergency(level: uint256, reason: String[200]):
    """
    เรียกใช้ emergency stop
    เฉพาะ guardian เท่านั้น
    """
    assert self.guardians[msg.sender], "Not a guardian"
    assert level > 0 and level <= LEVEL_EMERGENCY, "Invalid level"
    assert level > self.emergency_level, "Can only increase level"
    
    self.emergency_level = level
    log EmergencyStop(level, msg.sender, reason)

@external
def approve_resume():
    """
    Council member vote เพื่อ resume
    """
    assert self.council[msg.sender], "Not a council member"
    assert self.emergency_level > LEVEL_NORMAL, "Not in emergency"
    assert not self.resume_approvals[msg.sender], "Already approved"
    
    self.resume_approvals[msg.sender] = True
    self.resume_approval_count += 1
    
    if self.resume_approval_count >= self.required_approvals:
        # Auto-resume เมื่อได้รับ approval เพียงพอ
        self.emergency_level = LEVEL_NORMAL
        self.resume_approval_count = 0
        log EmergencyResume(msg.sender)

@external
def add_guardian(account: address):
    """เพิ่ม guardian (owner เท่านั้น)"""
    assert msg.sender == self.owner, "Not owner"
    self.guardians[account] = True

@external
def add_council(account: address):
    """เพิ่ม council member (owner เท่านั้น)"""
    assert msg.sender == self.owner, "Not owner"
    self.council[account] = True
    self.council_count += 1

# ฟังก์ชันตัวอย่างที่มีการตรวจสอบ emergency level
@external
def normal_only_function():
    """เฉพาะ LEVEL_NORMAL"""
    self._check_not_emergency()
    pass

@external
def caution_ok_function():
    """ทำงานได้จนถึง LEVEL_CAUTION"""
    self._check_caution_ok()
    pass
```

---

## Partial Pause

### Pause เฉพาะบางฟังก์ชัน

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Partial Pause System

event FunctionPaused:
    function_id: indexed(bytes32)
    caller: indexed(address)

event FunctionUnpaused:
    function_id: indexed(bytes32)
    caller: indexed(address)

# Function IDs
DEPOSIT_FUNCTION: constant(bytes32) = keccak256("deposit")
WITHDRAW_FUNCTION: constant(bytes32) = keccak256("withdraw")
SWAP_FUNCTION: constant(bytes32) = keccak256("swap")
ADD_LIQUIDITY_FUNCTION: constant(bytes32) = keccak256("addLiquidity")
REMOVE_LIQUIDITY_FUNCTION: constant(bytes32) = keccak256("removeLiquidity")

owner: public(address)
function_paused: public(HashMap[bytes32, bool])

@deploy
def __init__():
    self.owner = msg.sender

@internal
def _require_function_active(function_id: bytes32):
    assert not self.function_paused[function_id], "Function is paused"

@external
def pause_function(function_id: bytes32):
    """Pause function เฉพาะ"""
    assert msg.sender == self.owner, "Not owner"
    self.function_paused[function_id] = True
    log FunctionPaused(function_id, msg.sender)

@external
def unpause_function(function_id: bytes32):
    """Unpause function เฉพาะ"""
    assert msg.sender == self.owner, "Not owner"
    self.function_paused[function_id] = False
    log FunctionUnpaused(function_id, msg.sender)

@external
def pause_all():
    """Pause ทุกฟังก์ชัน"""
    assert msg.sender == self.owner, "Not owner"
    self.function_paused[DEPOSIT_FUNCTION] = True
    self.function_paused[WITHDRAW_FUNCTION] = True
    self.function_paused[SWAP_FUNCTION] = True
    self.function_paused[ADD_LIQUIDITY_FUNCTION] = True
    self.function_paused[REMOVE_LIQUIDITY_FUNCTION] = True

@external
def deposit():
    """ฝากเงิน"""
    self._require_function_active(DEPOSIT_FUNCTION)
    # logic

@external
def withdraw():
    """ถอนเงิน"""
    self._require_function_active(WITHDRAW_FUNCTION)
    # logic

@external
def swap():
    """Swap tokens"""
    self._require_function_active(SWAP_FUNCTION)
    # logic
```

---

## ตัวอย่าง: Pausable DEX

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Pausable DEX (Decentralized Exchange)
# @notice DEX พร้อม Pausable และ Circuit Breaker

interface ERC20:
    def transfer(to: address, amount: uint256) -> bool: nonpayable
    def transferFrom(from_: address, to: address, amount: uint256) -> bool: nonpayable
    def balanceOf(account: address) -> uint256: view
    def approve(spender: address, amount: uint256) -> bool: nonpayable

# ==================== Events ====================

event Paused:
    account: indexed(address)

event Unpaused:
    account: indexed(address)

event FunctionPaused:
    function_id: indexed(bytes32)

event FunctionUnpaused:
    function_id: indexed(bytes32)

event LiquidityAdded:
    provider: indexed(address)
    token_a_amount: uint256
    token_b_amount: uint256
    lp_tokens: uint256

event LiquidityRemoved:
    provider: indexed(address)
    token_a_amount: uint256
    token_b_amount: uint256
    lp_tokens: uint256

event Swap:
    trader: indexed(address)
    token_in: indexed(address)
    token_out: indexed(address)
    amount_in: uint256
    amount_out: uint256

event EmergencyWithdraw:
    account: indexed(address)
    amount: uint256

# ==================== Function IDs ====================

SWAP_FN: constant(bytes32) = keccak256("swap")
ADD_LIQ_FN: constant(bytes32) = keccak256("addLiquidity")
REMOVE_LIQ_FN: constant(bytes32) = keccak256("removeLiquidity")

# ==================== State Variables ====================

owner: public(address)
paused: public(bool)
function_paused: public(HashMap[bytes32, bool])

token_a: public(address)
token_b: public(address)

reserve_a: public(uint256)
reserve_b: public(uint256)
total_lp_supply: public(uint256)
lp_balances: public(HashMap[address, uint256])

fee_bps: public(uint256)    # Trading fee (30 = 0.3%)
protocol_fee_bps: public(uint256)  # Protocol fee (5 = 0.05%)

# Emergency withdrawal
emergency_mode: public(bool)

# ==================== Constructor ====================

@deploy
def __init__(
    _token_a: address,
    _token_b: address,
    _fee_bps: uint256
):
    self.owner = msg.sender
    self.token_a = _token_a
    self.token_b = _token_b
    self.fee_bps = _fee_bps
    self.protocol_fee_bps = 5

# ==================== Modifiers (Internal) ====================

@internal
def _only_owner():
    assert msg.sender == self.owner, "Not owner"

@internal
def _not_paused():
    assert not self.paused, "DEX is paused"

@internal
def _function_active(fn_id: bytes32):
    assert not self.function_paused[fn_id], "Function is paused"

# ==================== Pause Functions ====================

@external
def pause():
    """Pause ทั้ง DEX"""
    self._only_owner()
    assert not self.paused, "Already paused"
    self.paused = True
    log Paused(msg.sender)

@external
def unpause():
    """Unpause DEX"""
    self._only_owner()
    assert self.paused, "Not paused"
    self.paused = False
    log Unpaused(msg.sender)

@external
def pause_function(fn_id: bytes32):
    """Pause ฟังก์ชันเฉพาะ"""
    self._only_owner()
    self.function_paused[fn_id] = True
    log FunctionPaused(fn_id)

@external
def unpause_function(fn_id: bytes32):
    """Unpause ฟังก์ชันเฉพาะ"""
    self._only_owner()
    self.function_paused[fn_id] = False
    log FunctionUnpaused(fn_id)

@external
def enable_emergency_mode():
    """เปิด emergency mode (ให้ users ถอนได้โดยตรง)"""
    self._only_owner()
    self.emergency_mode = True
    self.paused = True
    log Paused(msg.sender)

# ==================== DEX Functions ====================

@external
def add_liquidity(
    amount_a: uint256,
    amount_b: uint256,
    min_lp_tokens: uint256
) -> uint256:
    """
    @notice เพิ่ม liquidity ใน pool
    @return จำนวน LP tokens ที่ได้รับ
    """
    self._not_paused()
    self._function_active(ADD_LIQ_FN)
    
    assert amount_a > 0 and amount_b > 0, "Zero amounts"
    
    lp_tokens: uint256 = 0
    
    if self.total_lp_supply == 0:
        # ครั้งแรก: geometric mean
        lp_tokens = isqrt(amount_a * amount_b)
    else:
        # คำนวณสัดส่วน
        lp_from_a: uint256 = amount_a * self.total_lp_supply / self.reserve_a
        lp_from_b: uint256 = amount_b * self.total_lp_supply / self.reserve_b
        lp_tokens = min(lp_from_a, lp_from_b)
    
    assert lp_tokens >= min_lp_tokens, "Insufficient LP tokens"
    
    # โอน tokens เข้า contract
    ERC20(self.token_a).transferFrom(msg.sender, self, amount_a)
    ERC20(self.token_b).transferFrom(msg.sender, self, amount_b)
    
    # อัปเดต reserves และ LP
    self.reserve_a += amount_a
    self.reserve_b += amount_b
    self.total_lp_supply += lp_tokens
    self.lp_balances[msg.sender] += lp_tokens
    
    log LiquidityAdded(msg.sender, amount_a, amount_b, lp_tokens)
    return lp_tokens

@external
def remove_liquidity(
    lp_tokens: uint256,
    min_amount_a: uint256,
    min_amount_b: uint256
) -> (uint256, uint256):
    """
    @notice ถอน liquidity จาก pool
    @return (amount_a, amount_b) ที่ได้รับ
    """
    self._not_paused()
    self._function_active(REMOVE_LIQ_FN)
    
    assert lp_tokens > 0, "Zero LP tokens"
    assert self.lp_balances[msg.sender] >= lp_tokens, "Insufficient LP"
    
    amount_a: uint256 = lp_tokens * self.reserve_a / self.total_lp_supply
    amount_b: uint256 = lp_tokens * self.reserve_b / self.total_lp_supply
    
    assert amount_a >= min_amount_a, "Slippage: token A"
    assert amount_b >= min_amount_b, "Slippage: token B"
    
    self.lp_balances[msg.sender] -= lp_tokens
    self.total_lp_supply -= lp_tokens
    self.reserve_a -= amount_a
    self.reserve_b -= amount_b
    
    ERC20(self.token_a).transfer(msg.sender, amount_a)
    ERC20(self.token_b).transfer(msg.sender, amount_b)
    
    log LiquidityRemoved(msg.sender, amount_a, amount_b, lp_tokens)
    return amount_a, amount_b

@external
def swap_a_to_b(
    amount_in: uint256,
    min_amount_out: uint256
) -> uint256:
    """
    @notice Swap token A -> token B
    @return จำนวน token B ที่ได้รับ
    """
    self._not_paused()
    self._function_active(SWAP_FN)
    
    assert amount_in > 0, "Zero input"
    
    # คำนวณ output ด้วย AMM formula: x * y = k
    fee: uint256 = amount_in * self.fee_bps / 10000
    amount_in_with_fee: uint256 = amount_in - fee
    
    # dy = y * dx / (x + dx)
    amount_out: uint256 = self.reserve_b * amount_in_with_fee / (self.reserve_a + amount_in_with_fee)
    
    assert amount_out >= min_amount_out, "Slippage too high"
    assert amount_out < self.reserve_b, "Insufficient liquidity"
    
    ERC20(self.token_a).transferFrom(msg.sender, self, amount_in)
    ERC20(self.token_b).transfer(msg.sender, amount_out)
    
    self.reserve_a += amount_in
    self.reserve_b -= amount_out
    
    log Swap(msg.sender, self.token_a, self.token_b, amount_in, amount_out)
    return amount_out

@external
def swap_b_to_a(
    amount_in: uint256,
    min_amount_out: uint256
) -> uint256:
    """
    @notice Swap token B -> token A
    """
    self._not_paused()
    self._function_active(SWAP_FN)
    
    assert amount_in > 0, "Zero input"
    
    fee: uint256 = amount_in * self.fee_bps / 10000
    amount_in_with_fee: uint256 = amount_in - fee
    
    amount_out: uint256 = self.reserve_a * amount_in_with_fee / (self.reserve_b + amount_in_with_fee)
    
    assert amount_out >= min_amount_out, "Slippage too high"
    assert amount_out < self.reserve_a, "Insufficient liquidity"
    
    ERC20(self.token_b).transferFrom(msg.sender, self, amount_in)
    ERC20(self.token_a).transfer(msg.sender, amount_out)
    
    self.reserve_b += amount_in
    self.reserve_a -= amount_out
    
    log Swap(msg.sender, self.token_b, self.token_a, amount_in, amount_out)
    return amount_out

@external
def emergency_withdraw():
    """
    @notice ถอน LP tokens ในกรณีฉุกเฉิน
    @dev ทำงานเฉพาะเมื่อ emergency_mode = True
    """
    assert self.emergency_mode, "Not in emergency mode"
    
    lp_tokens: uint256 = self.lp_balances[msg.sender]
    assert lp_tokens > 0, "No LP tokens"
    
    amount_a: uint256 = lp_tokens * self.reserve_a / self.total_lp_supply
    amount_b: uint256 = lp_tokens * self.reserve_b / self.total_lp_supply
    
    self.lp_balances[msg.sender] = 0
    self.total_lp_supply -= lp_tokens
    self.reserve_a -= amount_a
    self.reserve_b -= amount_b
    
    ERC20(self.token_a).transfer(msg.sender, amount_a)
    ERC20(self.token_b).transfer(msg.sender, amount_b)
    
    log EmergencyWithdraw(msg.sender, lp_tokens)

# ==================== View Functions ====================

@view
@external
def get_price_a_to_b() -> uint256:
    """ราคา token A ใน token B"""
    if self.reserve_a == 0:
        return 0
    return self.reserve_b * 10**18 / self.reserve_a

@view
@external
def get_price_b_to_a() -> uint256:
    """ราคา token B ใน token A"""
    if self.reserve_b == 0:
        return 0
    return self.reserve_a * 10**18 / self.reserve_b

@view
@external
def get_amount_out_a_to_b(amount_in: uint256) -> uint256:
    """คำนวณ output สำหรับ A->B swap"""
    fee: uint256 = amount_in * self.fee_bps / 10000
    amount_in_net: uint256 = amount_in - fee
    return self.reserve_b * amount_in_net / (self.reserve_a + amount_in_net)

@view
@external
def get_lp_balance(account: address) -> uint256:
    return self.lp_balances[account]

@view
@external
def is_function_active(fn_id: bytes32) -> bool:
    return not self.function_paused[fn_id] and not self.paused
```

---

## Test Code

```python
# tests/test_pausable_dex.py
import pytest
from eth_utils import keccak

SWAP_FN = keccak(text="swap")
ADD_LIQ_FN = keccak(text="addLiquidity")
REMOVE_LIQ_FN = keccak(text="removeLiquidity")

@pytest.fixture
def owner(accounts):
    return accounts[0]

@pytest.fixture
def user(accounts):
    return accounts[1]

@pytest.fixture
def token_a(owner, project):
    return project.MockERC20.deploy("Token A", "TKA", 18, sender=owner)

@pytest.fixture
def token_b(owner, project):
    return project.MockERC20.deploy("Token B", "TKB", 18, sender=owner)

@pytest.fixture
def dex(owner, token_a, token_b, project):
    return project.PausableDEX.deploy(
        token_a.address,
        token_b.address,
        30,  # 0.3% fee
        sender=owner
    )

class TestPausable:
    
    def test_initial_state(self, dex):
        """ตรวจสอบ state เริ่มต้น"""
        assert not dex.paused()
        assert not dex.emergency_mode()
    
    def test_pause(self, dex, owner):
        """ทดสอบ pause"""
        dex.pause(sender=owner)
        assert dex.paused()
    
    def test_unpause(self, dex, owner):
        """ทดสอบ unpause"""
        dex.pause(sender=owner)
        dex.unpause(sender=owner)
        assert not dex.paused()
    
    def test_non_owner_cannot_pause(self, dex, user):
        """Non-owner ไม่สามารถ pause ได้"""
        with pytest.raises(Exception):
            dex.pause(sender=user)
    
    def test_swap_fails_when_paused(self, dex, owner, user, token_a, token_b):
        """Swap ล้มเหลวเมื่อ pause"""
        # Setup: mint tokens and add liquidity
        amount = 1000 * 10**18
        token_a.mint(owner.address, amount * 2, sender=owner)
        token_b.mint(owner.address, amount * 2, sender=owner)
        token_a.approve(dex.address, amount, sender=owner)
        token_b.approve(dex.address, amount, sender=owner)
        dex.add_liquidity(amount, amount, 0, sender=owner)
        
        # Pause
        dex.pause(sender=owner)
        
        # Try swap
        token_a.mint(user.address, 100 * 10**18, sender=owner)
        token_a.approve(dex.address, 100 * 10**18, sender=user)
        
        with pytest.raises(Exception):
            dex.swap_a_to_b(100 * 10**18, 0, sender=user)
    
    def test_partial_pause_swap_only(self, dex, owner, user, token_a, token_b):
        """ทดสอบ pause เฉพาะ swap function"""
        # Pause swap only
        dex.pause_function(SWAP_FN, sender=owner)
        assert not dex.is_function_active(SWAP_FN)
        assert dex.is_function_active(ADD_LIQ_FN)
    
    def test_emergency_mode(self, dex, owner, user, token_a, token_b):
        """ทดสอบ emergency mode"""
        # Setup
        amount = 1000 * 10**18
        token_a.mint(owner.address, amount, sender=owner)
        token_b.mint(owner.address, amount, sender=owner)
        token_a.approve(dex.address, amount, sender=owner)
        token_b.approve(dex.address, amount, sender=owner)
        dex.add_liquidity(amount, amount, 0, sender=owner)
        
        # Enable emergency
        dex.enable_emergency_mode(sender=owner)
        assert dex.emergency_mode()
        assert dex.paused()
        
        # User can emergency withdraw
        dex.emergency_withdraw(sender=owner)
        assert dex.get_lp_balance(owner.address) == 0

class TestDEXFunctionality:
    
    def test_add_liquidity(self, dex, owner, token_a, token_b):
        """ทดสอบเพิ่ม liquidity"""
        amount = 1000 * 10**18
        token_a.mint(owner.address, amount, sender=owner)
        token_b.mint(owner.address, amount, sender=owner)
        token_a.approve(dex.address, amount, sender=owner)
        token_b.approve(dex.address, amount, sender=owner)
        
        lp = dex.add_liquidity(amount, amount, 0, sender=owner)
        assert dex.get_lp_balance(owner.address) > 0
    
    def test_swap(self, dex, owner, user, token_a, token_b):
        """ทดสอบ swap"""
        amount = 10000 * 10**18
        token_a.mint(owner.address, amount, sender=owner)
        token_b.mint(owner.address, amount, sender=owner)
        token_a.approve(dex.address, amount, sender=owner)
        token_b.approve(dex.address, amount, sender=owner)
        dex.add_liquidity(amount, amount, 0, sender=owner)
        
        swap_amount = 100 * 10**18
        token_a.mint(user.address, swap_amount, sender=owner)
        token_a.approve(dex.address, swap_amount, sender=user)
        
        before_b = token_b.balanceOf(user.address)
        dex.swap_a_to_b(swap_amount, 0, sender=user)
        after_b = token_b.balanceOf(user.address)
        
        assert after_b > before_b
```

---

## สรุป

Pausable Pattern มีความสำคัญในโลก DeFi:

| Feature | Description |
|---------|-------------|
| Global Pause | หยุดทุกอย่างทันที |
| Partial Pause | เลือก pause บางฟังก์ชัน |
| Circuit Breaker | Auto-pause เมื่อผิดปกติ |
| Emergency Mode | ให้ users ถอนได้แม้ pause |

### Best Practices
- มี timelock ก่อน pause/unpause ใน production
- ใช้ multisig สำหรับ pause functionality
- ทดสอบ emergency scenarios เสมอ
- แจ้ง users ทันทีเมื่อ pause

---

[⬅️ Part 027: Access Control](part_027_access_control.md) | [Part 029: ReentrancyGuard ➡️](part_029_reentrancy.md)
