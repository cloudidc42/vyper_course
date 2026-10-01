# Part 029: ReentrancyGuard

## สารบัญ
1. [Reentrancy Attack คืออะไร](#reentrancy-attack-คืออะไร)
2. [The DAO Hack](#the-dao-hack)
3. [@nonreentrant Decorator](#nonreentrant-decorator)
4. [Checks-Effects-Interactions Pattern](#checks-effects-interactions-pattern)
5. [Cross-Function Reentrancy](#cross-function-reentrancy)
6. [ตัวอย่าง: Secure Vault](#ตัวอย่าง-secure-vault)
7. [Test Code](#test-code)

---

## Reentrancy Attack คืออะไร

**Reentrancy Attack** เป็นการโจมตีที่ Contract ที่เป็นอันตรายเรียกกลับเข้า (re-enter) Contract เหยื่อก่อนที่จะสิ้นสุดการทำงาน ทำให้สามารถถอนเงินได้หลายครั้งจาก balance เดิม

### ลำดับการโจมตี

```
1. Attacker -> Victim.withdraw()
2. Victim ส่ง ETH ให้ Attacker
3. Attacker's receive() -> Victim.withdraw() [AGAIN!]
4. Victim ยังไม่ได้อัปเดต balance
5. ดำเนินต่อไปจนกว่า Victim จะหมด ETH
```

### Contract เสี่ยงโจมตี (ห้ามใช้!)

```vyper
# ❌ VULNERABLE CODE - อย่าใช้ใน production!
# @version 0.4.0

balances: HashMap[address, uint256]

@external
@payable
def deposit():
    self.balances[msg.sender] += msg.value

@external
def withdraw():
    amount: uint256 = self.balances[msg.sender]
    assert amount > 0, "Nothing to withdraw"
    
    # ❌ BUG: ส่ง ETH ก่อนอัปเดต balance!
    send(msg.sender, amount)  # Attacker สามารถ re-enter ได้ที่นี่
    
    # บรรทัดนี้ยังไม่ทำงานตอน re-enter
    self.balances[msg.sender] = 0
```

---

## The DAO Hack

### ประวัติศาสตร์ที่เปลี่ยนโลก Ethereum

ในปี 2016 Contract ชื่อ "The DAO" ซึ่งมีมูลค่ากว่า 150 ล้านดอลลาร์ถูก hack ด้วย Reentrancy Attack ส่งผลให้ Ethereum ต้องทำ Hard Fork เป็น Ethereum และ Ethereum Classic

### โค้ดแบบ Solidity ที่ถูกโจมตี (จำลอง)

```
// Solidity version of vulnerable DAO code (ตัวอย่างเพื่อการศึกษา)
function splitDAO(uint _proposalID, address _newCurator) noEther {
    // ...
    withdrawRewardFor(msg.sender); // ส่งก่อน
    totalSupply -= balances[msg.sender]; // ลด supply หลัง
    balances[msg.sender] = 0; // รีเซ็ต balance หลัง
    // Attacker สามารถ re-enter ได้ก่อนบรรทัดนี้!
}
```

### Attack Contract (ตัวอย่างเพื่อการศึกษา)

```vyper
# ❌ ตัวอย่าง Attack Contract - เพื่อการศึกษาเท่านั้น!
# @version 0.4.0

interface VulnerableVault:
    def deposit(): payable
    def withdraw(): nonpayable

target: address
owner: address
attack_count: uint256
MAX_ATTACKS: constant(uint256) = 10

@deploy
def __init__(_target: address):
    self.target = _target
    self.owner = msg.sender

@external
@payable
def attack():
    """เริ่ม attack"""
    assert msg.sender == self.owner
    self.attack_count = 0
    VulnerableVault(self.target).deposit(value=msg.value)
    VulnerableVault(self.target).withdraw()

@external
@payable
def __default__():
    """Re-enter เมื่อรับ ETH"""
    if self.attack_count < MAX_ATTACKS:
        self.attack_count += 1
        VulnerableVault(self.target).withdraw()  # Re-enter!

@external
def drain():
    """ดึงเงินที่ขโมยมา"""
    assert msg.sender == self.owner
    send(self.owner, self.balance)
```

---

## @nonreentrant Decorator

### วิธีป้องกัน Reentrancy ใน Vyper

Vyper มี built-in decorator `@nonreentrant` ที่ป้องกัน reentrancy โดยอัตโนมัติ

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Safe Vault with @nonreentrant

event Deposit:
    account: indexed(address)
    amount: uint256

event Withdrawal:
    account: indexed(address)
    amount: uint256

balances: public(HashMap[address, uint256])

@external
@payable
@nonreentrant
def deposit():
    """
    @notice ฝากเงิน
    @dev @nonreentrant ป้องกัน reentrancy attack
    """
    assert msg.value > 0, "Must send ETH"
    self.balances[msg.sender] += msg.value
    log Deposit(msg.sender, msg.value)

@external
@nonreentrant
def withdraw(amount: uint256):
    """
    @notice ถอนเงิน
    @dev @nonreentrant ป้องกัน reentrancy
    """
    assert self.balances[msg.sender] >= amount, "Insufficient balance"
    
    # @nonreentrant ป้องกัน re-entry อัตโนมัติ
    # แต่ยังควรใช้ CEI pattern
    self.balances[msg.sender] -= amount  # Effects ก่อน
    send(msg.sender, amount)            # Interactions หลัง
    
    log Withdrawal(msg.sender, amount)

@view
@external
def get_balance(account: address) -> uint256:
    return self.balances[account]
```

### วิธีทำงานของ @nonreentrant

```
1. ก่อนเรียก function: ตรวจสอบ lock
2. ถ้า lock = false: set lock = true, execute
3. ถ้า lock = true: revert "Reentrancy guard"
4. หลัง function เสร็จ: set lock = false
```

```vyper
# @version 0.4.0
# การทำงานภายในของ @nonreentrant (conceptual)

_reentrancy_lock: bool  # Vyper manage ให้อัตโนมัติ

@internal
def _nonreentrant_enter():
    assert not self._reentrancy_lock, "ReentrancyGuard: reentrant call"
    self._reentrancy_lock = True

@internal
def _nonreentrant_exit():
    self._reentrancy_lock = False
```

### Named Locks ใน Vyper 0.4.0

```vyper
# @version 0.4.0
# ใช้ named locks สำหรับ group ต่างๆ

@external
@nonreentrant
def function_group_a():
    """ทุก function ใน group เดียวกัน share lock เดียวกัน"""
    pass

@external
@nonreentrant
def function_group_a_alt():
    """จะ revert ถ้า function_group_a กำลังทำงาน"""
    pass
```

---

## Checks-Effects-Interactions Pattern

### CEI Pattern - วิธีป้องกัน Reentrancy ที่ดีที่สุด

ลำดับที่ถูกต้อง:
1. **Checks**: ตรวจสอบ conditions
2. **Effects**: อัปเดต state
3. **Interactions**: เรียก external contracts / ส่ง ETH

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title CEI Pattern Example

event Withdrawn:
    user: indexed(address)
    amount: uint256

balances: HashMap[address, uint256]

@external
@payable
def deposit():
    self.balances[msg.sender] += msg.value

@external
@nonreentrant
def withdraw(amount: uint256):
    # ======= CHECKS =======
    assert amount > 0, "Amount must be > 0"
    assert self.balances[msg.sender] >= amount, "Insufficient balance"
    
    # ======= EFFECTS =======
    # อัปเดต state ก่อนที่จะส่งเงิน (สำคัญมาก!)
    self.balances[msg.sender] -= amount
    
    # ======= INTERACTIONS =======
    # ส่งเงินหลังจากอัปเดต state แล้ว
    send(msg.sender, amount)
    
    log Withdrawn(msg.sender, amount)

# ❌ ผิด: Interactions ก่อน Effects
@external
def withdraw_wrong(amount: uint256):
    assert self.balances[msg.sender] >= amount
    
    # ❌ ส่ง ETH ก่อน (WRONG!)
    send(msg.sender, amount)
    
    # อัปเดต state หลัง (ถ้า re-enter จะเข้าถึง balance เดิม)
    self.balances[msg.sender] -= amount
```

### CEI กับ Token Transfers

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0

interface ERC20:
    def transfer(to: address, amount: uint256) -> bool: nonpayable
    def transferFrom(from_: address, to: address, amount: uint256) -> bool: nonpayable

token: address
user_balances: HashMap[address, uint256]

@external
@nonreentrant
def deposit_tokens(amount: uint256):
    # CHECKS
    assert amount > 0
    
    # EFFECTS (อัปเดตก่อน)
    self.user_balances[msg.sender] += amount
    
    # INTERACTIONS (ทำหลังสุด)
    ERC20(self.token).transferFrom(msg.sender, self, amount)

@external
@nonreentrant
def withdraw_tokens(amount: uint256):
    # CHECKS
    assert self.user_balances[msg.sender] >= amount
    
    # EFFECTS (ลด balance ก่อน)
    self.user_balances[msg.sender] -= amount
    
    # INTERACTIONS (โอนหลังสุด)
    ERC20(self.token).transfer(msg.sender, amount)
```

---

## Cross-Function Reentrancy

### ความเสี่ยงระหว่าง Functions

Cross-function reentrancy เกิดเมื่อ Function A เรียก external call แล้ว Attacker re-enter ผ่าน Function B แทน

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0

# ❌ VULNERABLE: Cross-function reentrancy
balances: HashMap[address, uint256]
lp_tokens: HashMap[address, uint256]

@external
def withdraw_eth():
    amount: uint256 = self.balances[msg.sender]
    # ❌ ส่ง ETH ก่อน
    send(msg.sender, amount)
    # State ยังไม่อัปเดต!
    self.balances[msg.sender] = 0

@external
def transfer_lp(to: address, amount: uint256):
    # ถ้า Attacker re-enter ผ่าน function นี้ตอนที่ withdraw_eth ทำงานอยู่
    # state ยังไม่ถูกอัปเดต ทำให้ transfer LP ได้โดยไม่ถูกต้อง
    assert self.lp_tokens[msg.sender] >= amount
    self.lp_tokens[msg.sender] -= amount
    self.lp_tokens[to] += amount
```

### วิธีป้องกัน Cross-Function Reentrancy

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# ✅ SAFE: ใช้ @nonreentrant ร่วมกับ CEI

balances: HashMap[address, uint256]
lp_tokens: HashMap[address, uint256]

@external
@nonreentrant
def withdraw_eth():
    amount: uint256 = self.balances[msg.sender]
    assert amount > 0
    
    # ✅ EFFECTS ก่อน
    self.balances[msg.sender] = 0
    
    # ✅ INTERACTIONS หลัง
    send(msg.sender, amount)

@external
@nonreentrant
def transfer_lp(to: address, amount: uint256):
    # @nonreentrant ป้องกัน re-entry จาก withdraw_eth
    assert self.lp_tokens[msg.sender] >= amount
    self.lp_tokens[msg.sender] -= amount
    self.lp_tokens[to] += amount
```

---

## ตัวอย่าง: Secure Vault

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Secure Vault
# @notice Vault ที่ป้องกัน Reentrancy อย่างสมบูรณ์

interface ERC20:
    def transfer(to: address, amount: uint256) -> bool: nonpayable
    def transferFrom(from_: address, to: address, amount: uint256) -> bool: nonpayable
    def balanceOf(account: address) -> uint256: view

# ==================== Events ====================

event Deposited:
    user: indexed(address)
    token: indexed(address)
    amount: uint256

event Withdrawn:
    user: indexed(address)
    token: indexed(address)
    amount: uint256

event ETHDeposited:
    user: indexed(address)
    amount: uint256

event ETHWithdrawn:
    user: indexed(address)
    amount: uint256

event EmergencyWithdraw:
    user: indexed(address)

# ==================== State Variables ====================

owner: public(address)
paused: public(bool)

# Token balances: user -> token -> amount
token_balances: public(HashMap[address, HashMap[address, uint256]])

# ETH balances
eth_balances: public(HashMap[address, uint256])

# Withdrawal limits per tx
max_withdrawal: public(uint256)

# ==================== Constructor ====================

@deploy
def __init__(_max_withdrawal: uint256):
    self.owner = msg.sender
    self.max_withdrawal = _max_withdrawal

# ==================== Internal ====================

@internal
def _only_owner():
    assert msg.sender == self.owner, "Not owner"

@internal
def _not_paused():
    assert not self.paused, "Vault is paused"

# ==================== ETH Functions ====================

@external
@payable
@nonreentrant
def deposit_eth():
    """
    @notice ฝาก ETH เข้า vault
    """
    self._not_paused()
    assert msg.value > 0, "Must send ETH"
    
    # EFFECTS
    self.eth_balances[msg.sender] += msg.value
    
    log ETHDeposited(msg.sender, msg.value)

@external
@nonreentrant
def withdraw_eth(amount: uint256):
    """
    @notice ถอน ETH จาก vault
    @dev ใช้ CEI pattern + @nonreentrant
    """
    self._not_paused()
    
    # CHECKS
    assert amount > 0, "Amount must be > 0"
    assert amount <= self.max_withdrawal, "Exceeds max withdrawal"
    assert self.eth_balances[msg.sender] >= amount, "Insufficient ETH balance"
    
    # EFFECTS (สำคัญ: ทำก่อน INTERACTIONS)
    self.eth_balances[msg.sender] -= amount
    
    # INTERACTIONS (ส่งหลังสุด)
    send(msg.sender, amount)
    
    log ETHWithdrawn(msg.sender, amount)

@external
@nonreentrant
def withdraw_all_eth():
    """
    @notice ถอน ETH ทั้งหมด
    """
    self._not_paused()
    
    # CHECKS
    amount: uint256 = self.eth_balances[msg.sender]
    assert amount > 0, "No ETH to withdraw"
    
    # EFFECTS
    self.eth_balances[msg.sender] = 0
    
    # INTERACTIONS
    send(msg.sender, amount)
    
    log ETHWithdrawn(msg.sender, amount)

# ==================== Token Functions ====================

@external
@nonreentrant
def deposit_token(token: address, amount: uint256):
    """
    @notice ฝาก ERC20 token เข้า vault
    """
    self._not_paused()
    
    # CHECKS
    assert token != empty(address), "Invalid token"
    assert amount > 0, "Amount must be > 0"
    
    # EFFECTS (อัปเดตก่อน)
    self.token_balances[msg.sender][token] += amount
    
    # INTERACTIONS (transferFrom หลังสุด)
    # Note: Token อาจมี callback ใน transferFrom
    success: bool = ERC20(token).transferFrom(msg.sender, self, amount)
    assert success, "Transfer failed"
    
    log Deposited(msg.sender, token, amount)

@external
@nonreentrant
def withdraw_token(token: address, amount: uint256):
    """
    @notice ถอน ERC20 token จาก vault
    """
    self._not_paused()
    
    # CHECKS
    assert token != empty(address), "Invalid token"
    assert amount > 0, "Amount must be > 0"
    assert amount <= self.max_withdrawal, "Exceeds max withdrawal"
    assert self.token_balances[msg.sender][token] >= amount, "Insufficient balance"
    
    # EFFECTS
    self.token_balances[msg.sender][token] -= amount
    
    # INTERACTIONS
    success: bool = ERC20(token).transfer(msg.sender, amount)
    assert success, "Transfer failed"
    
    log Withdrawn(msg.sender, token, amount)

@external
@nonreentrant
def withdraw_all_token(token: address):
    """
    @notice ถอน token ทั้งหมด
    """
    self._not_paused()
    
    # CHECKS
    amount: uint256 = self.token_balances[msg.sender][token]
    assert amount > 0, "No tokens to withdraw"
    
    # EFFECTS
    self.token_balances[msg.sender][token] = 0
    
    # INTERACTIONS
    success: bool = ERC20(token).transfer(msg.sender, amount)
    assert success, "Transfer failed"
    
    log Withdrawn(msg.sender, token, amount)

# ==================== Admin Functions ====================

@external
def pause():
    self._only_owner()
    self.paused = True

@external
def unpause():
    self._only_owner()
    self.paused = False

@external
def set_max_withdrawal(new_max: uint256):
    self._only_owner()
    self.max_withdrawal = new_max

# ==================== View Functions ====================

@view
@external
def get_eth_balance(user: address) -> uint256:
    return self.eth_balances[user]

@view
@external
def get_token_balance(user: address, token: address) -> uint256:
    return self.token_balances[user][token]

@view
@external
def get_vault_eth_balance() -> uint256:
    return self.balance
```

### Attack Test (เพื่อยืนยันว่าป้องกันได้)

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Reentrancy Attacker Contract (สำหรับ testing)

interface SecureVault:
    def deposit_eth(): payable
    def withdraw_eth(amount: uint256): nonpayable

vault: address
owner: address
attack_count: public(uint256)

@deploy
def __init__(_vault: address):
    self.vault = _vault
    self.owner = msg.sender

@external
@payable
def attack():
    """พยายาม attack"""
    assert msg.sender == self.owner
    self.attack_count = 0
    SecureVault(self.vault).deposit_eth(value=msg.value)
    SecureVault(self.vault).withdraw_eth(msg.value)

@external
@payable
def __default__():
    """พยายาม re-enter"""
    if self.attack_count < 3 and self.vault.balance > 0:
        self.attack_count += 1
        # จะ revert เพราะ @nonreentrant
        SecureVault(self.vault).withdraw_eth(1)
```

---

## Test Code

```python
# tests/test_reentrancy.py
import pytest

@pytest.fixture
def owner(accounts):
    return accounts[0]

@pytest.fixture
def attacker(accounts):
    return accounts[1]

@pytest.fixture
def user(accounts):
    return accounts[2]

@pytest.fixture
def vault(owner, project):
    return project.SecureVault.deploy(
        10 * 10**18,  # max 10 ETH per withdrawal
        sender=owner
    )

@pytest.fixture
def attack_contract(attacker, vault, project):
    return project.ReentrancyAttacker.deploy(
        vault.address,
        sender=attacker
    )

class TestReentrancyProtection:
    
    def test_deposit_eth(self, vault, user):
        """ทดสอบ deposit ETH"""
        amount = 1 * 10**18
        vault.deposit_eth(sender=user, value=amount)
        assert vault.get_eth_balance(user.address) == amount
    
    def test_withdraw_eth(self, vault, user):
        """ทดสอบ withdraw ETH"""
        amount = 1 * 10**18
        vault.deposit_eth(sender=user, value=amount)
        
        before = user.balance
        vault.withdraw_eth(amount, sender=user)
        after = user.balance
        
        assert after > before
        assert vault.get_eth_balance(user.address) == 0
    
    def test_reentrancy_attack_fails(self, vault, attack_contract, attacker):
        """ทดสอบว่า reentrancy attack ล้มเหลว"""
        # ฝาก ETH จากคนอื่นก่อน
        deposit_amount = 5 * 10**18
        
        # Attack ด้วย 1 ETH
        attack_amount = 1 * 10**18
        
        with pytest.raises(Exception) as exc_info:
            attack_contract.attack(
                sender=attacker,
                value=attack_amount
            )
        
        # Attack ควร revert
        assert "reentrant" in str(exc_info.value).lower() or \
               "reentrancy" in str(exc_info.value).lower() or \
               exc_info.value is not None
    
    def test_cannot_withdraw_more_than_max(self, vault, user):
        """ทดสอบว่าไม่สามารถถอนเกิน max"""
        amount = 100 * 10**18  # เกิน max ที่ตั้งไว้ (10 ETH)
        vault.deposit_eth(sender=user, value=amount)
        
        with pytest.raises(Exception):
            vault.withdraw_eth(amount, sender=user)
    
    def test_cannot_withdraw_more_than_balance(self, vault, user):
        """ทดสอบว่าไม่สามารถถอนเกิน balance"""
        amount = 1 * 10**18
        vault.deposit_eth(sender=user, value=amount)
        
        with pytest.raises(Exception):
            vault.withdraw_eth(amount + 1, sender=user)
    
    def test_multiple_users_isolated(self, vault, user, attacker):
        """ทดสอบว่า balance ของแต่ละ user แยกกัน"""
        vault.deposit_eth(sender=user, value=2 * 10**18)
        vault.deposit_eth(sender=attacker, value=3 * 10**18)
        
        assert vault.get_eth_balance(user.address) == 2 * 10**18
        assert vault.get_eth_balance(attacker.address) == 3 * 10**18
    
    def test_withdraw_all(self, vault, user):
        """ทดสอบ withdraw ทั้งหมด"""
        amount = 2 * 10**18
        vault.deposit_eth(sender=user, value=amount)
        
        vault.withdraw_all_eth(sender=user)
        
        assert vault.get_eth_balance(user.address) == 0

class TestCEIPattern:
    """ทดสอบว่า CEI pattern ทำงานถูกต้อง"""
    
    def test_state_updated_before_transfer(self, vault, user):
        """ตรวจสอบว่า state อัปเดตก่อน ETH ส่งออก"""
        amount = 1 * 10**18
        vault.deposit_eth(sender=user, value=amount)
        vault.withdraw_eth(amount, sender=user)
        
        # หลัง withdraw: balance ต้องเป็น 0
        assert vault.get_eth_balance(user.address) == 0
    
    def test_pause_prevents_withdraw(self, vault, owner, user):
        """Pause ป้องกัน withdraw"""
        vault.deposit_eth(sender=user, value=1 * 10**18)
        vault.pause(sender=owner)
        
        with pytest.raises(Exception):
            vault.withdraw_eth(1 * 10**18, sender=user)
```

---

## สรุป

การป้องกัน Reentrancy ใน Vyper:

| วิธี | ประสิทธิภาพ | เหมาะสำหรับ |
|-----|------------|------------|
| `@nonreentrant` | สูง | ทุกฟังก์ชันที่มี external calls |
| CEI Pattern | สูง | ต้องทำทุกครั้ง |
| ทั้งสอง | สูงสุด | Production contracts |

### กฎทอง
1. **ใช้ `@nonreentrant` เสมอ** เมื่อมี external calls
2. **CEI Pattern** ต้องทำก่อนเสมอ (ไม่ใช่แค่พึ่ง decorator)
3. **ระวัง Token callbacks** ERC777 มี hooks ที่อาจ re-enter
4. **Cross-function reentrancy** อย่าลืมป้องกัน function อื่นด้วย

---

[⬅️ Part 028: Pausable Pattern](part_028_pausable.md) | [Part 030: Timelock ➡️](part_030_timelock.md)
