# Part 076: Security Audit Techniques

## สารบัญ
1. [Security Mindset](#mindset)
2. [Common Vulnerabilities](#vulnerabilities)
3. [Attack Patterns](#attacks)
4. [Audit Methodology](#methodology)
5. [Static Analysis Tools](#tools)
6. [Security Checklist](#checklist)
7. [Real-world Exploit Examples](#exploits)
8. [Defensive Patterns](#defense)
9. [Audit Report Writing](#report)
10. [Practical Exercises](#exercises)

---

## 1. Security Mindset {#mindset}

### แนวคิดพื้นฐาน

```
Security Mindset:
"Assume everything is trying to attack your contract"

Questions to Ask:
1. ใครสามารถเรียก function นี้ได้?
2. ถ้า attacker เรียก function นี้ จะเกิดอะไรขึ้น?
3. Function นี้ทำอะไรกับ State?
4. มี Edge Case ที่ทำให้ Contract behave ผิดปกติไหม?
5. การเรียก External Contract มีความเสี่ยงอะไร?
```

### Security Properties

```python
# @version 0.4.0

"""
Security Properties ที่ Contract ควรมี:

1. Integrity: State ถูกต้องเสมอ
   - Balance ต้องไม่เกิน Total Supply
   - Sum of balances == total supply
   
2. Availability: Contract ทำงานได้เสมอ
   - ไม่ติด Deadlock
   - ไม่มี DoS vector
   
3. Confidentiality: ข้อมูล Private ไม่รั่วไหล
   - ระวัง: ทุกอย่างบน Blockchain เป็น Public!
   
4. Non-repudiation: Actions มี Proof
   - Events บันทึกการกระทำ
   
5. Authorization: ผู้มีสิทธิ์เท่านั้น
   - Proper access control
"""

# Invariants: สิ่งที่ต้องเป็นจริงเสมอ

total_supply: uint256
balances: HashMap[address, uint256]

# INVARIANT: sum(balances) == total_supply
# ถ้า invariant นี้ผิด → Bug หรือ Exploit!

@view
@external
def verify_invariant(accounts: DynArray[address, 100]) -> bool:
    """
    ตรวจสอบว่า Invariant ยังถูกต้อง
    (ใช้สำหรับ Testing เท่านั้น - ไม่ควรอยู่ใน Production)
    """
    sum_balances: uint256 = 0
    for account: address in accounts:
        sum_balances += self.balances[account]
    
    return sum_balances <= self.total_supply
```

---

## 2. Common Vulnerabilities {#vulnerabilities}

### 2.1 Reentrancy

```python
# @version 0.4.0

# ════════════════════════
# ❌ VULNERABLE Contract
# ════════════════════════
balances_unsafe: HashMap[address, uint256]

@external
def withdraw_unsafe():
    amount: uint256 = self.balances_unsafe[msg.sender]
    assert amount > 0, "No balance"
    
    # ❌ ส่ง ETH ก่อน Update State
    # Attacker สามารถ Re-enter ก่อน balance เป็น 0
    send(msg.sender, amount)
    
    # ❌ State Update หลัง ETH transfer
    self.balances_unsafe[msg.sender] = 0

# ════════════════════════
# ✅ SECURE Contract
# ════════════════════════
balances_safe: HashMap[address, uint256]

@nonreentrant
@external
def withdraw_safe():
    amount: uint256 = self.balances_safe[msg.sender]
    assert amount > 0, "No balance"
    
    # ✅ Update State ก่อน
    self.balances_safe[msg.sender] = 0
    
    # ✅ แล้วจึงส่ง ETH
    send(msg.sender, amount)

# ════════════════════════
# ATTACK Contract
# ════════════════════════

# (Pseudocode - this would be Solidity in practice)
# contract Attacker:
#     VulnerableContract victim
#     
#     function attack() external payable:
#         victim.deposit{value: 1 ether}()
#         victim.withdraw_unsafe()
#     
#     receive() external payable:
#         if address(victim).balance >= 1 ether:
#             victim.withdraw_unsafe()  # Re-enter!
```

### 2.2 Integer Overflow/Underflow

```python
# @version 0.4.0

# Vyper ป้องกัน Overflow/Underflow โดย Default
# แต่ต้องระวังใน Manual Arithmetic

# ❌ อาจมีปัญหา: หาร ก่อน คูณ
@pure
@external
def calculate_wrong(amount: uint256, rate: uint256) -> uint256:
    """
    ถ้า amount เล็กมากและ rate เล็กมาก
    amount / PRECISION * rate อาจ = 0
    """
    PRECISION: uint256 = 10**18
    return (amount / PRECISION) * rate  # ❌ Loss of precision

# ✅ ถูกต้อง: คูณก่อน หาร
@pure
@external
def calculate_correct(amount: uint256, rate: uint256) -> uint256:
    PRECISION: uint256 = 10**18
    return (amount * rate) / PRECISION  # ✅ Better precision

# ❌ อันตราย: ลบอาจ underflow
@external
def unsafe_subtract(a: uint256, b: uint256) -> uint256:
    return a - b  # Vyper จะ Revert ถ้า b > a, but check the logic

# ✅ ปลอดภัย: ตรวจสอบก่อน
@external
def safe_subtract(a: uint256, b: uint256) -> uint256:
    assert a >= b, "Underflow"
    return a - b
```

### 2.3 Front-Running

```python
# @version 0.4.0

"""
Front-Running: Attacker เห็น Transaction ใน Mempool
แล้วส่ง Transaction ด้วย Gas Price สูงกว่า
เพื่อให้ Transaction ตัวเองถูก Execute ก่อน

ตัวอย่าง DEX Front-Running:
1. User ส่ง Tx: ซื้อ ETH ราคา 1000 USDC
2. Attacker เห็นใน Mempool
3. Attacker ซื้อ ETH ก่อน (ราคาขึ้น)
4. User ซื้อ ETH ราคาแพงขึ้น (slippage)
5. Attacker ขาย ETH ได้กำไร (Sandwich Attack)
"""

# Mitigation 1: Slippage Protection
@external
def swap_with_slippage(
    amount_in: uint256,
    min_amount_out: uint256  # ✅ User กำหนด minimum
):
    amount_out: uint256 = self._calculate_output(amount_in)
    assert amount_out >= min_amount_out, "Slippage too high"
    
    self._execute_swap(amount_in, amount_out)

# Mitigation 2: Commit-Reveal Scheme
committed: HashMap[address, bytes32]
reveal_deadline: HashMap[address, uint256]

COMMIT_DURATION: constant(uint256) = 2  # blocks

@external
def commit(commitment: bytes32):
    """Step 1: Submit hidden order"""
    self.committed[msg.sender] = commitment
    self.reveal_deadline[msg.sender] = block.number + COMMIT_DURATION

@external
def reveal(action: uint256, salt: bytes32):
    """Step 2: Reveal the order"""
    assert block.number <= self.reveal_deadline[msg.sender], "Expired"
    
    # Verify commitment
    expected: bytes32 = keccak256(concat(
        convert(action, bytes32),
        salt,
        convert(msg.sender, bytes32)
    ))
    assert self.committed[msg.sender] == expected, "Invalid reveal"
    
    # Execute action
    self._execute_action(action)
    
    # Clear
    self.committed[msg.sender] = empty(bytes32)

@internal
def _execute_action(action: uint256):
    pass

@internal
def _calculate_output(amount: uint256) -> uint256:
    return amount  # Placeholder

@internal
def _execute_swap(amount_in: uint256, amount_out: uint256):
    pass
```

### 2.4 Oracle Manipulation

```python
# @version 0.4.0

interface IChainlinkOracle:
    def latestRoundData() -> (uint80, int256, uint256, uint256, uint80): view

# ════════════════════════
# ❌ Vulnerable: ใช้ Spot Price
# ════════════════════════
spot_oracle: address

@view
@internal
def _get_price_unsafe() -> uint256:
    """
    ❌ อันตราย: Flash loan อาจ manipulate spot price
    """
    # สำหรับ AMM spot price
    reserve0: uint256 = 1000 * 10**18
    reserve1: uint256 = 2000000 * 10**6  # USDC
    
    return reserve1 * 10**18 / reserve0

# ════════════════════════
# ✅ Secure: TWAP + Chainlink
# ════════════════════════
chainlink_feed: address
STALENESS_THRESHOLD: constant(uint256) = 3600  # 1 hour
MIN_PRICE: constant(uint256) = 100 * 10**6    # $100 min
MAX_PRICE: constant(uint256) = 1000000 * 10**6  # $1M max

@view
@internal
def _get_price_safe() -> uint256:
    """
    ✅ ปลอดภัย: Chainlink กับการตรวจสอบ
    """
    # Get Chainlink price
    round_id: uint80 = 0
    answer: int256 = 0
    started_at: uint256 = 0
    updated_at: uint256 = 0
    answered_in_round: uint80 = 0
    
    (round_id, answer, started_at, updated_at, answered_in_round) = (
        IChainlinkOracle(self.chainlink_feed).latestRoundData()
    )
    
    # Sanity checks
    assert answer > 0, "Negative price"
    assert updated_at > 0, "Round not complete"
    assert block.timestamp - updated_at <= STALENESS_THRESHOLD, "Stale price"
    assert round_id == answered_in_round, "Round incomplete"
    
    price: uint256 = convert(answer, uint256)
    
    # Bounds check
    assert price >= MIN_PRICE, "Price too low"
    assert price <= MAX_PRICE, "Price too high"
    
    return price
```

### 2.5 Access Control Issues

```python
# @version 0.4.0

# ════════════════════════
# ❌ Vulnerable: Missing Access Control
# ════════════════════════
dangerous_owner: address

@external
def set_owner_vulnerable(new_owner: address):
    """❌ ใครก็เรียกได้!"""
    self.dangerous_owner = new_owner  # CRITICAL VULNERABILITY!

# ════════════════════════
# ❌ Vulnerable: Wrong Caller Check
# ════════════════════════
@external
def use_tx_origin():
    """❌ tx.origin แทน msg.sender"""
    assert tx.origin == self.dangerous_owner, "Not owner"
    # Vulnerable to Phishing Attack

# ════════════════════════
# ✅ Secure: Proper Access Control
# ════════════════════════
owner: address
pending_owner: address

@external
def transfer_ownership(new_owner: address):
    """✅ Two-step ownership transfer"""
    assert msg.sender == self.owner, "Not owner"
    assert new_owner != empty(address), "Zero address"
    self.pending_owner = new_owner

@external
def accept_ownership():
    """✅ New owner must explicitly accept"""
    assert msg.sender == self.pending_owner, "Not pending owner"
    self.owner = self.pending_owner
    self.pending_owner = empty(address)

# ════════════════════════
# ❌ Vulnerable: Unprotected Initialization
# ════════════════════════
is_initialized: bool
vuln_owner: address

@external
def initialize_vulnerable(owner: address):
    """❌ ใครเรียกได้ก่อน จะเป็น owner"""
    self.vuln_owner = owner
    self.is_initialized = True

# ✅ Secure: Initialize only once or use constructor
@deploy
def __init__(initial_owner: address):
    """✅ Constructor ตั้งค่าได้ครั้งเดียว"""
    self.owner = initial_owner
```

### 2.6 Timestamp Manipulation

```python
# @version 0.4.0

"""
Validators สามารถ Manipulate block.timestamp ได้เล็กน้อย
(โดยทั่วไปไม่เกิน 15 วินาที)

ห้ามใช้ block.timestamp สำหรับ:
- Random number generation
- Lottery systems
- High-precision timing

ใช้ได้สำหรับ:
- Long timeouts (hours/days)
- Approximate timestamps
"""

# ❌ อันตราย: ใช้ timestamp สำหรับ Randomness
@view
@internal
def _pseudo_random() -> uint256:
    """❌ Manipulable!"""
    return convert(
        keccak256(
            concat(
                convert(block.timestamp, bytes32),
                convert(block.prevhash, bytes32)
            )
        ),
        uint256
    )

# ✅ OK: Long-term timelock
WEEK: constant(uint256) = 7 * 24 * 3600

locked_until: uint256

@external
def lock_funds():
    self.locked_until = block.timestamp + WEEK  # ✅ 7 days is fine

@external
def unlock():
    assert block.timestamp >= self.locked_until, "Still locked"
    # Proceed...
```

---

## 3. Attack Patterns {#attacks}

### 3.1 Sandwich Attack Simulation

```
Sandwich Attack:
1. User broadcasts: Buy 10 ETH @ max $2050
2. Bot spots it in mempool
3. Bot front-runs: Buy 5 ETH (ราคาขึ้นไป $2010)
4. User's tx executes: ซื้อ 10 ETH @ $2050 (แพงกว่า)
5. Bot back-runs: ขาย 5 ETH @ ราคาสูง = กำไร

Mitigation:
- ตั้ง slippage ต่ำ
- ใช้ Private Mempool (Flashbots)
- ใช้ DEX ที่มี MEV protection
```

### 3.2 Donation Attack

```python
# @version 0.4.0

"""
Donation Attack:
แทนที่จะ Deposit ปกติ ใช้ transferFrom หรือ selfdestruct
เพื่อ Inflate balance โดยไม่ผ่าน Deposit logic

ตัวอย่าง:
ถ้า share price = totalAssets / totalShares
Attacker Donate ก่อนใคร Deposit → Inflate share price
User deposit 1 wei → ได้ 0 shares (round down)
Attacker withdraw ทั้งหมด → กำไร
"""

# ❌ Vulnerable: คำนวณ shares จาก balance โดยตรง
@view
@internal
def _preview_deposit_unsafe(assets: uint256) -> uint256:
    supply: uint256 = self.total_shares
    
    if supply == 0:
        return assets  # First deposit
    
    # ❌ อาจถูก manipulate ผ่าน Donation
    return (assets * supply) / self.token.balanceOf(self)

# ✅ Secure: Virtual shares (EIP-4626 inspired)
VIRTUAL_SHARES: constant(uint256) = 1000  # Buffer
VIRTUAL_ASSETS: constant(uint256) = 1000

@view
@internal
def _preview_deposit_safe(assets: uint256) -> uint256:
    supply: uint256 = self.total_shares + VIRTUAL_SHARES
    total_assets: uint256 = self.total_assets + VIRTUAL_ASSETS
    
    # Virtual assets/shares ทำให้ Donation ไม่คุ้ม
    return (assets * supply) / total_assets

total_shares: uint256
total_assets: uint256

interface ERC20Token:
    def balanceOf(account: address) -> uint256: view
    def transferFrom(sender: address, receiver: address, amount: uint256) -> bool: nonpayable

token: address
```

### 3.3 Price Oracle Manipulation

```python
# @version 0.4.0

"""
Oracle Manipulation Attack:
1. ยืม Flash Loan จำนวนมาก
2. Swap ใน Pool เพื่อ Manipulate Spot Price
3. เรียก Protocol ที่ใช้ Spot Price
4. คืน Flash Loan

Protection: TWAP (Time-Weighted Average Price)
"""

# TWAP Implementation
observations: DynArray[uint256, 10800]  # Store 3 hours (assuming 1 obs/second)
last_observation_time: uint256

TWAP_WINDOW: constant(uint256) = 3600  # 1 hour

@external
def record_price(price: uint256):
    """Record price observation"""
    current_time: uint256 = block.timestamp
    
    # Add observation
    if len(self.observations) < 10800:
        self.observations.append(price)
    else:
        # Rotate (simplified)
        pass
    
    self.last_observation_time = current_time

@view
@external
def get_twap() -> uint256:
    """Get Time-Weighted Average Price"""
    count: uint256 = len(self.observations)
    
    if count == 0:
        return 0
    
    # Use last N observations
    window: uint256 = min(count, 3600)
    
    sum_price: uint256 = 0
    for i: uint256 in range(3600):
        if i >= window:
            break
        # Get observation from end
        idx: uint256 = count - 1 - i
        sum_price += self.observations[idx]
    
    return sum_price / window
```

---

## 4. Audit Methodology {#methodology}

### Systematic Audit Process

```
Phase 1: Scoping (1-2 days)
├── Read Documentation
├── Understand System Architecture  
├── Identify Critical Components
└── Prioritize Areas to Review

Phase 2: Code Review (3-7 days)
├── Manual Line-by-Line Review
├── Logic and Math Verification
├── Access Control Review
├── External Call Analysis
└── State Machine Analysis

Phase 3: Testing (2-3 days)
├── Unit Tests
├── Integration Tests
├── Fuzzing
├── Invariant Testing
└── PoC Exploit Writing

Phase 4: Reporting (1-2 days)
├── Document Findings
├── Severity Rating
├── Recommendations
└── Final Report
```

### Severity Classification

```
Critical (9-10):
- Loss of user funds
- Unauthorized access to privileged functions
- Complete protocol compromise
- Example: Reentrancy leading to fund drain

High (7-8):
- Partial fund loss
- Denial of service
- Example: Oracle manipulation causing bad debt

Medium (4-6):
- Incorrect behavior
- Limited financial impact
- Example: Fee calculation error

Low (1-3):
- Best practice violations
- Minor inefficiencies
- Example: Missing events

Informational (0):
- Gas optimizations
- Code style
- Documentation issues
```

---

## 5. Static Analysis Tools {#tools}

### Slither

```bash
# ติดตั้ง
pip install slither-analyzer

# รัน Analysis
slither contracts/MyContract.vy

# ดู Detectors ที่มี
slither --list-detectors

# รัน Detector เฉพาะ
slither contracts/MyContract.vy --detect reentrancy-eth

# Export Report
slither contracts/MyContract.vy --json report.json
```

### Mythril

```bash
# ติดตั้ง
pip install mythril

# Analyze Contract
myth analyze contracts/MyContract.vy

# ดู Analysis เฉพาะ
myth analyze contracts/MyContract.vy -t 10  # timeout 10s
```

### Manual Checklist

```python
"""
MANUAL AUDIT CHECKLIST:

Access Control:
□ Owner functions protected?
□ Role-based access correct?
□ No use of tx.origin for auth?
□ Ownership transfer two-step?
□ Privileged roles minimized?

Reentrancy:
□ State updated before external calls?
□ @nonreentrant used where needed?
□ No recursive calls possible?

Integer Arithmetic:
□ No unchecked math operations?
□ Division before multiplication avoided?
□ Precision loss acceptable?
□ Overflow/underflow possible?

External Calls:
□ Return values checked?
□ Failed calls handled?
□ Trusted/untrusted distinction?
□ Interfaces match expected contracts?

Oracle:
□ Price manipulation resistant?
□ TWAP used for DEX prices?
□ Staleness checks in place?
□ Circuit breakers for extreme prices?

Logic:
□ Correct order of operations?
□ Edge cases handled?
□ Rounding behavior acceptable?
□ State invariants maintained?

Events:
□ Important state changes logged?
□ Correct indexed parameters?
□ All fields included?

Gas:
□ No unbounded loops?
□ DoS through gas griefing possible?
□ Storage reads optimized?
"""
```

---

## 6. Security Checklist {#checklist}

```python
# @version 0.4.0
# SPDX-License-Identifier: MIT

"""
Secure Contract Template
ทุก Function ผ่าน Security Checklist
"""

# ════════════════════════
# SECURITY FEATURES
# ════════════════════════

# 1. Ownership
owner: address
pending_owner: address

# 2. Role-based Access
admin_role: HashMap[address, bool]
operator_role: HashMap[address, bool]

# 3. Circuit Breaker
paused: bool
emergency_shutdown: bool

# 4. Rate Limiting
action_count: HashMap[address, uint256]
action_window_start: HashMap[address, uint256]
MAX_ACTIONS_PER_WINDOW: constant(uint256) = 10
WINDOW_DURATION: constant(uint256) = 3600  # 1 hour

# 5. Withdrawal Limits
daily_withdrawal: HashMap[address, uint256]
last_withdrawal_day: HashMap[address, uint256]
MAX_DAILY_WITHDRAWAL: constant(uint256) = 10 * 10**18  # 10 ETH

# 6. Input Validation
MIN_AMOUNT: constant(uint256) = 1000  # Dust prevention
MAX_AMOUNT: constant(uint256) = 1000 * 10**18

# ════════════════════════
# INTERNAL CHECKS
# ════════════════════════

@internal
def _only_owner():
    assert msg.sender == self.owner, "Ownable: not owner"

@internal
def _only_admin():
    assert self.admin_role[msg.sender], "AccessControl: not admin"

@internal
def _when_not_paused():
    assert not self.paused, "Pausable: paused"
    assert not self.emergency_shutdown, "Emergency: shutdown"

@internal
def _rate_limit(user: address):
    current_time: uint256 = block.timestamp
    
    if current_time - self.action_window_start[user] > WINDOW_DURATION:
        # Reset window
        self.action_window_start[user] = current_time
        self.action_count[user] = 0
    
    self.action_count[user] += 1
    assert self.action_count[user] <= MAX_ACTIONS_PER_WINDOW, "Rate limited"

@internal
def _check_withdrawal_limit(user: address, amount: uint256):
    current_day: uint256 = block.timestamp / 86400
    
    if self.last_withdrawal_day[user] < current_day:
        self.daily_withdrawal[user] = 0
        self.last_withdrawal_day[user] = current_day
    
    assert (
        self.daily_withdrawal[user] + amount <= MAX_DAILY_WITHDRAWAL
    ), "Daily limit exceeded"
    
    self.daily_withdrawal[user] += amount

@internal
def _validate_amount(amount: uint256):
    assert amount >= MIN_AMOUNT, "Amount too small"
    assert amount <= MAX_AMOUNT, "Amount too large"

@internal
def _validate_address(addr: address):
    assert addr != empty(address), "Zero address"
    assert addr != self, "Cannot be self"

# ════════════════════════
# SECURE EXTERNAL FUNCTIONS
# ════════════════════════

@nonreentrant
@external
def secure_withdraw(amount: uint256):
    """
    Secure withdrawal with all checks
    """
    self._when_not_paused()
    self._validate_amount(amount)
    self._rate_limit(msg.sender)
    self._check_withdrawal_limit(msg.sender, amount)
    
    # Check balance BEFORE transfer
    balance: uint256 = self._get_balance(msg.sender)
    assert balance >= amount, "Insufficient balance"
    
    # Update state BEFORE external call (CEI pattern)
    self._deduct_balance(msg.sender, amount)
    
    # Transfer LAST
    send(msg.sender, amount)

@internal
def _get_balance(account: address) -> uint256:
    return 0  # Placeholder

@internal
def _deduct_balance(account: address, amount: uint256):
    pass  # Placeholder
```

---

## 7. Real-world Exploit Examples {#exploits}

### 7.1 The DAO Hack (2016) - Reentrancy

```
Amount Lost: 3.6M ETH (~$60M at the time)

Vulnerability:
1. DAO Contract sends ETH to malicious contract
2. Malicious contract's fallback calls back into DAO
3. DAO checks balance AFTER transfer
4. State not updated yet → Same withdrawal again
5. Repeat until DAO drained

Impact: Ethereum Classic fork
Lesson: Always update state BEFORE external calls
```

### 7.2 Compound Oracle Manipulation (2020)

```
Amount Lost: ~$90M in liquidations

Attack:
1. Low liquidity COMP/ETH on Coinbase Pro
2. Flash loan to manipulate COMP price
3. Compound used Coinbase as oracle
4. Borrowed assets worth more than collateral
5. Mass liquidations

Lesson: Use TWAP, not spot price
```

### 7.3 Ronin Bridge Hack (2022)

```
Amount Lost: $625M

Vulnerability:
- 9 validator keys needed
- Attacker compromised 5 keys
- Had temporary control of 1 Axie DAO key
- Total: 6/9 → Majority!

Lesson:
- Decentralize validators
- Multi-signature with hardware keys
- Monitor large withdrawals
```

### 7.4 Euler Finance Hack (2023)

```
Amount Lost: $197M

Vulnerability:
- donate() function increased dToken balance
- Without proper accounting
- Led to bad debt in the protocol

Lesson:
- Every state change must be accounted for
- Beware donation attacks
- Test invariants thoroughly
```

---

## 8. Defensive Patterns {#defense}

### 8.1 Checks-Effects-Interactions

```python
# @version 0.4.0

balances: HashMap[address, uint256]

@nonreentrant
@external
def withdraw(amount: uint256):
    # ════════════════════
    # 1. CHECKS
    # ════════════════════
    assert amount > 0, "Zero amount"
    assert self.balances[msg.sender] >= amount, "Insufficient balance"
    # No external calls here!
    
    # ════════════════════
    # 2. EFFECTS (State Changes)
    # ════════════════════
    self.balances[msg.sender] -= amount
    # All state changes done
    
    # ════════════════════
    # 3. INTERACTIONS (External Calls)
    # ════════════════════
    # State is already updated, reentrancy safe
    send(msg.sender, amount)
```

### 8.2 Pull Payment Pattern

```python
# @version 0.4.0

"""
Push Payment: ส่ง ETH ทันที → อาจ fail หรือ reentrancy
Pull Payment: Record ว่าต้องจ่าย → User Claim เอง
"""

# ════════════════════════
# ❌ Push Payment (Dangerous)
# ════════════════════════
@external
def distribute_to_all(recipients: DynArray[address, 100], amounts: DynArray[uint256, 100]):
    """❌ ถ้า 1 Transfer fail → ทั้งหมด fail"""
    for i: uint256 in range(100):
        if i >= len(recipients):
            break
        send(recipients[i], amounts[i])  # ❌ Can fail

# ════════════════════════
# ✅ Pull Payment (Safe)
# ════════════════════════
pending_withdrawals: HashMap[address, uint256]

@external
def distribute_pull(recipients: DynArray[address, 100], amounts: DynArray[uint256, 100]):
    """✅ Record ไว้ก่อน User Claim เอง"""
    for i: uint256 in range(100):
        if i >= len(recipients):
            break
        self.pending_withdrawals[recipients[i]] += amounts[i]

@nonreentrant
@external
def claim():
    """User pulls their payment"""
    amount: uint256 = self.pending_withdrawals[msg.sender]
    assert amount > 0, "Nothing to claim"
    
    self.pending_withdrawals[msg.sender] = 0
    send(msg.sender, amount)
```

### 8.3 Emergency Mechanisms

```python
# @version 0.4.0

"""
Emergency Mechanisms:
1. Pause: หยุดการทำงานชั่วคราว
2. Emergency Withdraw: User ดึงเงินออกได้เสมอ
3. Rescue: Admin ช่วย rescue funds
"""

owner: address
paused: bool
emergency_mode: bool

user_deposits: HashMap[address, uint256]
total_locked: uint256

@external
def pause():
    assert msg.sender == self.owner, "Not owner"
    self.paused = True

@external
def enable_emergency_mode():
    """
    Enable mode ที่ให้ User withdraw โดยไม่ต้องรอ
    (ในกรณีที่มีบั๊กหรือ exploit)
    """
    assert msg.sender == self.owner, "Not owner"
    self.emergency_mode = True

@nonreentrant
@external
def emergency_withdraw():
    """
    Emergency Withdrawal: ใช้ได้ตลอดเวลา
    ไม่สนว่า contract paused หรือไม่
    """
    assert self.emergency_mode, "Not in emergency mode"
    
    amount: uint256 = self.user_deposits[msg.sender]
    assert amount > 0, "No deposits"
    
    self.user_deposits[msg.sender] = 0
    self.total_locked -= amount
    
    send(msg.sender, amount)

@external
def rescue_token(token: address, to: address, amount: uint256):
    """Rescue stuck tokens (non-protocol tokens only)"""
    assert msg.sender == self.owner, "Not owner"
    
    # ห้าม rescue protocol tokens!
    assert token != self, "Cannot rescue protocol token"
    
    # Transfer
    # ERC20(token).transfer(to, amount)
```

---

## 9. Audit Report Writing {#report}

### Report Structure

```markdown
# Security Audit Report: [Protocol Name]
**Audited by:** [Your Name/Organization]
**Date:** [Date]
**Version:** [Contract Version]
**Scope:** [Files Audited]

## Executive Summary
[2-3 paragraphs สรุปสิ่งที่ตรวจสอบ, ผลลัพธ์, และ Risk Level]

## Scope
- files/contracts/*.vy
- Commit hash: 0xabc...
- Audit period: [dates]

## Risk Classification
| Severity | Count |
|----------|-------|
| Critical | 0     |
| High     | 1     |
| Medium   | 2     |
| Low      | 3     |
| Info     | 5     |

## Findings

### [H-01] Reentrancy in withdraw()
**Severity:** High
**Location:** contracts/Vault.vy:45

**Description:**
The `withdraw()` function updates the user's balance AFTER sending ETH...

**Impact:**
An attacker can drain the contract by re-entering withdraw()...

**Proof of Concept:**
```python
# Attack contract pseudocode
...
```

**Recommendation:**
Apply Checks-Effects-Interactions pattern...

**Fix:**
```python
# Updated code
...
```

**Status:** Fixed in commit 0xdef...

---

## Appendix
- Testing methodology
- Tools used
- References
```

---

## 10. แบบฝึกหัด {#exercises}

```python
# @version 0.4.0
# SPDX-License-Identifier: MIT

"""
EXERCISE: Find the Vulnerabilities!
Contract นี้มี Vulnerability อยู่กี่จุด?
"""

owner: address
balances: HashMap[address, uint256]
approved: HashMap[address, HashMap[address, bool]]
total: uint256

@deploy
def __init__():
    self.owner = msg.sender

# BUG 1: -------------------------
@external
def initialize(new_owner: address):
    self.owner = new_owner  # 🐛 ใครเรียกได้!

# BUG 2: -------------------------
@payable
@external
def deposit():
    self.balances[msg.sender] += msg.value
    self.total = self.total + msg.value

# BUG 3: -------------------------
@external
def withdraw(amount: uint256):
    # 🐛 Missing check: amount > 0
    # 🐛 Missing: balance check before subtract
    send(msg.sender, amount)  # 🐛 ส่งก่อน Update!
    self.balances[msg.sender] -= amount
    self.total -= amount

# BUG 4: -------------------------
@view
@external
def get_random_winner() -> address:
    # 🐛 Not random!
    seed: uint256 = convert(block.prevhash, uint256) + block.timestamp
    # ...
    return msg.sender

# BUG 5: -------------------------
@external
def set_approved(operator: address):
    # 🐛 Anyone can approve anyone
    self.approved[msg.sender][operator] = True

# SOLUTIONS:
# Bug 1: require(msg.sender == owner) หรือ ลบ initialize()
# Bug 2: ดี แต่ควร emit event
# Bug 3: ต้องมี balance check, update before send, @nonreentrant
# Bug 4: ไม่มี True Randomness บน Blockchain (ใช้ Chainlink VRF)
# Bug 5: ตั้งใจให้ user approve operator? แต่ควร verify
```

---

## สรุป Part 076

ในส่วนนี้คุณได้เรียนรู้:
- ✅ Security Mindset และ Properties
- ✅ Vulnerabilities: Reentrancy, Overflow, Front-running, Oracle, Access Control
- ✅ Attack Patterns: Sandwich, Donation, Price Manipulation
- ✅ Audit Methodology และ Process
- ✅ Static Analysis Tools: Slither, Mythril
- ✅ Real-world Exploits: DAO, Compound, Euler
- ✅ Defensive Patterns: CEI, Pull Payment, Emergency Mechanisms
- ✅ Audit Report Writing

## แบบฝึกหัด

1. **Audit** Contract ใน Exercise section หา Bugs ทั้งหมด
2. **Fix** ทุก Bug ที่พบ
3. **เขียน** Mini Audit Report สำหรับ Contract ที่ Fix แล้ว
4. **รัน** Slither บน Contracts จาก Parts ก่อนหน้า
5. **สร้าง** PoC สำหรับ Reentrancy Attack

---

**ก่อนหน้า: [Part 075 - Account Abstraction](part_075_account_abstraction.md)**  
**ต่อไป: [Part 077 - Formal Verification](part_077_formal_verification.md)**
