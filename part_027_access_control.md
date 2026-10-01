# Part 027: Access Control

## สารบัญ
1. [บทนำ Role-Based Access Control](#บทนำ)
2. [Admin Roles](#admin-roles)
3. [Grant/Revoke Roles](#grantrevoke-roles)
4. [Multiple Admin Support](#multiple-admin-support)
5. [ตัวอย่าง: Multi-Role Contract](#ตัวอย่าง-multi-role-contract)
6. [Test Code](#test-code)

---

## บทนำ

**Role-Based Access Control (RBAC)** เป็นระบบการควบคุมการเข้าถึงที่ยืดหยุ่นกว่า Ownable Pattern โดยแทนที่จะมีแค่ "owner" คนเดียว เราสามารถกำหนด "roles" หลายบทบาทได้

### เปรียบเทียบกับ Ownable

| คุณสมบัติ | Ownable | Access Control |
|----------|---------|----------------|
| ผู้มีสิทธิ์ | 1 คน | หลายคน |
| บทบาท | 1 บทบาท | หลายบทบาท |
| ความยืดหยุ่น | ต่ำ | สูง |
| ความซับซ้อน | ต่ำ | ปานกลาง |

### Use Cases
- DeFi Protocol ที่มีหลาย admin
- DAO ที่มีคณะกรรมการ
- Game ที่มีทั้ง game master และ moderator
- Enterprise contract ที่มีหลายระดับสิทธิ์

---

## Admin Roles

### นิยาม Role ด้วย Constants

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0

# Role constants - ใช้ bytes32 เพื่อเซฟ gas
DEFAULT_ADMIN_ROLE: constant(bytes32) = 0x0000000000000000000000000000000000000000000000000000000000000000
MINTER_ROLE: constant(bytes32) = keccak256("MINTER_ROLE")
PAUSER_ROLE: constant(bytes32) = keccak256("PAUSER_ROLE")
UPGRADER_ROLE: constant(bytes32) = keccak256("UPGRADER_ROLE")
```

### โครงสร้างข้อมูล Role

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0

# ข้อมูลของแต่ละ role
struct RoleData:
    members: HashMap[address, bool]   # สมาชิกใน role
    admin_role: bytes32               # role ที่เป็น admin ของ role นี้

# mapping จาก role -> RoleData
roles: HashMap[bytes32, RoleData]
```

---

## Grant/Revoke Roles

### ระบบ Grant และ Revoke

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title AccessControl Basic

DEFAULT_ADMIN_ROLE: constant(bytes32) = empty(bytes32)
MINTER_ROLE: constant(bytes32) = keccak256("MINTER_ROLE")
PAUSER_ROLE: constant(bytes32) = keccak256("PAUSER_ROLE")

event RoleGranted:
    role: indexed(bytes32)
    account: indexed(address)
    sender: indexed(address)

event RoleRevoked:
    role: indexed(bytes32)
    account: indexed(address)
    sender: indexed(address)

event RoleAdminChanged:
    role: indexed(bytes32)
    previous_admin_role: indexed(bytes32)
    new_admin_role: indexed(bytes32)

# role -> account -> bool
role_members: HashMap[bytes32, HashMap[address, bool]]
# role -> admin_role
role_admin: HashMap[bytes32, bytes32]

@deploy
def __init__():
    # ให้ deployer เป็น DEFAULT_ADMIN
    self._grant_role(DEFAULT_ADMIN_ROLE, msg.sender)

@internal
def _grant_role(role: bytes32, account: address):
    if not self.role_members[role][account]:
        self.role_members[role][account] = True
        log RoleGranted(role, account, msg.sender)

@internal
def _revoke_role(role: bytes32, account: address):
    if self.role_members[role][account]:
        self.role_members[role][account] = False
        log RoleRevoked(role, account, msg.sender)

@internal
def _check_role(role: bytes32, account: address):
    assert self.role_members[role][account], "AccessControl: account missing role"

@internal
def _set_role_admin(role: bytes32, admin_role: bytes32):
    previous_admin: bytes32 = self.role_admin[role]
    self.role_admin[role] = admin_role
    log RoleAdminChanged(role, previous_admin, admin_role)

@external
def grant_role(role: bytes32, account: address):
    """
    @notice มอบ role ให้ account
    @dev เฉพาะ admin ของ role นั้นเท่านั้น
    """
    admin_role: bytes32 = self.role_admin[role]
    self._check_role(admin_role, msg.sender)
    self._grant_role(role, account)

@external
def revoke_role(role: bytes32, account: address):
    """
    @notice เพิกถอน role จาก account
    @dev เฉพาะ admin ของ role นั้นเท่านั้น
    """
    admin_role: bytes32 = self.role_admin[role]
    self._check_role(admin_role, msg.sender)
    self._revoke_role(role, account)

@external
def renounce_role(role: bytes32, account: address):
    """
    @notice สละ role ของตัวเอง
    @dev ทุกคนสามารถสละ role ของตัวเองได้
    """
    assert account == msg.sender, "AccessControl: can only renounce for self"
    self._revoke_role(role, account)

@view
@external
def has_role(role: bytes32, account: address) -> bool:
    """ตรวจสอบว่า account มี role หรือไม่"""
    return self.role_members[role][account]

@view
@external
def get_role_admin(role: bytes32) -> bytes32:
    """ดึง admin role ของ role นั้น"""
    return self.role_admin[role]
```

---

## Multiple Admin Support

### รองรับ Admin หลายระดับ

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Multi-Level Access Control

DEFAULT_ADMIN_ROLE: constant(bytes32) = empty(bytes32)
SUPER_ADMIN_ROLE: constant(bytes32) = keccak256("SUPER_ADMIN_ROLE")
ADMIN_ROLE: constant(bytes32) = keccak256("ADMIN_ROLE")
OPERATOR_ROLE: constant(bytes32) = keccak256("OPERATOR_ROLE")
USER_ROLE: constant(bytes32) = keccak256("USER_ROLE")

event RoleGranted:
    role: indexed(bytes32)
    account: indexed(address)
    sender: indexed(address)

event RoleRevoked:
    role: indexed(bytes32)
    account: indexed(address)
    sender: indexed(address)

role_members: HashMap[bytes32, HashMap[address, bool]]
role_admin: HashMap[bytes32, bytes32]

@deploy
def __init__():
    # ตั้งค่า role hierarchy
    # SUPER_ADMIN เป็น admin ของ ADMIN
    # ADMIN เป็น admin ของ OPERATOR
    # OPERATOR เป็น admin ของ USER
    self.role_admin[SUPER_ADMIN_ROLE] = DEFAULT_ADMIN_ROLE
    self.role_admin[ADMIN_ROLE] = SUPER_ADMIN_ROLE
    self.role_admin[OPERATOR_ROLE] = ADMIN_ROLE
    self.role_admin[USER_ROLE] = OPERATOR_ROLE
    
    # Deployer ได้รับ DEFAULT_ADMIN_ROLE และ SUPER_ADMIN_ROLE
    self.role_members[DEFAULT_ADMIN_ROLE][msg.sender] = True
    self.role_members[SUPER_ADMIN_ROLE][msg.sender] = True

@internal
def _check_role(role: bytes32, account: address):
    assert self.role_members[role][account], \
        "AccessControl: account is missing required role"

@internal
def _grant_role(role: bytes32, account: address):
    if not self.role_members[role][account]:
        self.role_members[role][account] = True
        log RoleGranted(role, account, msg.sender)

@internal
def _revoke_role(role: bytes32, account: address):
    if self.role_members[role][account]:
        self.role_members[role][account] = False
        log RoleRevoked(role, account, msg.sender)

@external
def grant_role(role: bytes32, account: address):
    admin_role: bytes32 = self.role_admin[role]
    self._check_role(admin_role, msg.sender)
    self._grant_role(role, account)

@external
def revoke_role(role: bytes32, account: address):
    admin_role: bytes32 = self.role_admin[role]
    self._check_role(admin_role, msg.sender)
    self._revoke_role(role, account)

@external
def renounce_role(role: bytes32):
    """สละ role ของตัวเอง"""
    self._revoke_role(role, msg.sender)

@view
@external
def has_role(role: bytes32, account: address) -> bool:
    return self.role_members[role][account]

@view
@external
def get_role_admin(role: bytes32) -> bytes32:
    return self.role_admin[role]

@view
@external
def is_super_admin(account: address) -> bool:
    return self.role_members[SUPER_ADMIN_ROLE][account]

@view
@external
def is_admin(account: address) -> bool:
    return self.role_members[ADMIN_ROLE][account] or \
           self.role_members[SUPER_ADMIN_ROLE][account]

@view
@external
def is_operator(account: address) -> bool:
    return self.role_members[OPERATOR_ROLE][account] or \
           self.role_members[ADMIN_ROLE][account] or \
           self.role_members[SUPER_ADMIN_ROLE][account]
```

---

## ตัวอย่าง: Multi-Role Contract

### DeFi Protocol พร้อม Role-Based Access Control

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title DeFi Token with Access Control
# @notice Token ที่มีระบบ role-based access control ครบถ้วน

# ==================== Role Constants ====================

DEFAULT_ADMIN_ROLE: constant(bytes32) = empty(bytes32)
MINTER_ROLE: constant(bytes32) = keccak256("MINTER_ROLE")
BURNER_ROLE: constant(bytes32) = keccak256("BURNER_ROLE")
PAUSER_ROLE: constant(bytes32) = keccak256("PAUSER_ROLE")
BLACKLIST_ROLE: constant(bytes32) = keccak256("BLACKLIST_ROLE")
UPGRADER_ROLE: constant(bytes32) = keccak256("UPGRADER_ROLE")

# ==================== Events ====================

event RoleGranted:
    role: indexed(bytes32)
    account: indexed(address)
    sender: indexed(address)

event RoleRevoked:
    role: indexed(bytes32)
    account: indexed(address)
    sender: indexed(address)

event Transfer:
    sender: indexed(address)
    receiver: indexed(address)
    amount: uint256

event Mint:
    to: indexed(address)
    amount: uint256
    minter: indexed(address)

event Burn:
    from_: indexed(address)
    amount: uint256
    burner: indexed(address)

event Paused:
    account: indexed(address)

event Unpaused:
    account: indexed(address)

event Blacklisted:
    account: indexed(address)
    status: bool

event FeeUpdated:
    old_fee: uint256
    new_fee: uint256

# ==================== State Variables ====================

# Access Control
role_members: HashMap[bytes32, HashMap[address, bool]]
role_admin: HashMap[bytes32, bytes32]

# Token
name: public(String[64])
symbol: public(String[8])
decimals: public(uint8)
total_supply: public(uint256)
max_supply: public(uint256)

balances: HashMap[address, uint256]
allowances: HashMap[address, HashMap[address, uint256]]

# Features
is_paused: public(bool)
is_blacklisted: public(HashMap[address, bool])
transfer_fee_bps: public(uint256)  # basis points (100 = 1%)
fee_collector: public(address)

# ==================== Constructor ====================

@deploy
def __init__(
    _name: String[64],
    _symbol: String[8],
    _decimals: uint8,
    _max_supply: uint256,
    _fee_collector: address
):
    """
    @notice Deploy Token พร้อมระบบ Access Control
    """
    self.name = _name
    self.symbol = _symbol
    self.decimals = _decimals
    self.max_supply = _max_supply
    self.fee_collector = _fee_collector
    self.transfer_fee_bps = 0  # ไม่มี fee เริ่มต้น
    
    # Setup role hierarchy
    self.role_admin[MINTER_ROLE] = DEFAULT_ADMIN_ROLE
    self.role_admin[BURNER_ROLE] = DEFAULT_ADMIN_ROLE
    self.role_admin[PAUSER_ROLE] = DEFAULT_ADMIN_ROLE
    self.role_admin[BLACKLIST_ROLE] = DEFAULT_ADMIN_ROLE
    self.role_admin[UPGRADER_ROLE] = DEFAULT_ADMIN_ROLE
    
    # ให้ deployer เป็น admin ทุก role
    self.role_members[DEFAULT_ADMIN_ROLE][msg.sender] = True
    self.role_members[MINTER_ROLE][msg.sender] = True
    self.role_members[PAUSER_ROLE][msg.sender] = True
    self.role_members[BLACKLIST_ROLE][msg.sender] = True

# ==================== Access Control Internal ====================

@internal
def _check_role(role: bytes32):
    assert self.role_members[role][msg.sender], \
        "AccessControl: missing role"

@internal
def _grant_role(role: bytes32, account: address):
    if not self.role_members[role][account]:
        self.role_members[role][account] = True
        log RoleGranted(role, account, msg.sender)

@internal
def _revoke_role(role: bytes32, account: address):
    if self.role_members[role][account]:
        self.role_members[role][account] = False
        log RoleRevoked(role, account, msg.sender)

# ==================== Access Control External ====================

@external
def grant_role(role: bytes32, account: address):
    """
    @notice มอบ role ให้ account (เฉพาะ admin)
    """
    admin_role: bytes32 = self.role_admin[role]
    assert self.role_members[admin_role][msg.sender], \
        "AccessControl: caller is not role admin"
    self._grant_role(role, account)

@external
def revoke_role(role: bytes32, account: address):
    """
    @notice เพิกถอน role (เฉพาะ admin)
    """
    admin_role: bytes32 = self.role_admin[role]
    assert self.role_members[admin_role][msg.sender], \
        "AccessControl: caller is not role admin"
    self._revoke_role(role, account)

@external
def renounce_role(role: bytes32):
    """
    @notice สละ role ของตัวเอง
    """
    self._revoke_role(role, msg.sender)

# ==================== Token Functions ====================

@internal
def _before_transfer(from_: address, to: address, amount: uint256):
    """
    @dev ตรวจสอบก่อน transfer ทุกครั้ง
    """
    assert not self.is_paused, "Token is paused"
    assert not self.is_blacklisted[from_], "Sender is blacklisted"
    assert not self.is_blacklisted[to], "Receiver is blacklisted"

@internal
def _calculate_fee(amount: uint256) -> uint256:
    """
    @dev คำนวณ fee จาก amount
    """
    if self.transfer_fee_bps == 0:
        return 0
    return amount * self.transfer_fee_bps / 10000

@external
def transfer(to: address, amount: uint256) -> bool:
    """
    @notice โอน token พร้อมหัก fee (ถ้ามี)
    """
    self._before_transfer(msg.sender, to, amount)
    
    fee: uint256 = self._calculate_fee(amount)
    net_amount: uint256 = amount - fee
    
    assert self.balances[msg.sender] >= amount, "Insufficient balance"
    
    self.balances[msg.sender] -= amount
    self.balances[to] += net_amount
    
    if fee > 0:
        self.balances[self.fee_collector] += fee
    
    log Transfer(msg.sender, to, amount)
    return True

@external
def transfer_from(from_: address, to: address, amount: uint256) -> bool:
    """
    @notice โอน token ในนามของ from_
    """
    self._before_transfer(from_, to, amount)
    
    assert self.allowances[from_][msg.sender] >= amount, "Insufficient allowance"
    assert self.balances[from_] >= amount, "Insufficient balance"
    
    fee: uint256 = self._calculate_fee(amount)
    net_amount: uint256 = amount - fee
    
    self.allowances[from_][msg.sender] -= amount
    self.balances[from_] -= amount
    self.balances[to] += net_amount
    
    if fee > 0:
        self.balances[self.fee_collector] += fee
    
    log Transfer(from_, to, amount)
    return True

@external
def approve(spender: address, amount: uint256) -> bool:
    self.allowances[msg.sender][spender] = amount
    return True

# ==================== Minter Functions ====================

@external
def mint(to: address, amount: uint256):
    """
    @notice สร้าง token ใหม่ (เฉพาะ MINTER_ROLE)
    """
    self._check_role(MINTER_ROLE)
    assert to != empty(address), "Cannot mint to zero address"
    assert self.total_supply + amount <= self.max_supply, "Exceeds max supply"
    
    self.balances[to] += amount
    self.total_supply += amount
    
    log Mint(to, amount, msg.sender)
    log Transfer(empty(address), to, amount)

# ==================== Burner Functions ====================

@external
def burn(from_: address, amount: uint256):
    """
    @notice เผา token (เฉพาะ BURNER_ROLE หรือ owner เผาของตัวเอง)
    """
    if from_ != msg.sender:
        self._check_role(BURNER_ROLE)
    
    assert self.balances[from_] >= amount, "Insufficient balance"
    
    self.balances[from_] -= amount
    self.total_supply -= amount
    
    log Burn(from_, amount, msg.sender)
    log Transfer(from_, empty(address), amount)

# ==================== Pauser Functions ====================

@external
def pause():
    """
    @notice หยุด transfer ทั้งหมด (เฉพาะ PAUSER_ROLE)
    """
    self._check_role(PAUSER_ROLE)
    assert not self.is_paused, "Already paused"
    self.is_paused = True
    log Paused(msg.sender)

@external
def unpause():
    """
    @notice เปิดใช้งาน transfer อีกครั้ง (เฉพาะ PAUSER_ROLE)
    """
    self._check_role(PAUSER_ROLE)
    assert self.is_paused, "Not paused"
    self.is_paused = False
    log Unpaused(msg.sender)

# ==================== Blacklist Functions ====================

@external
def set_blacklist(account: address, status: bool):
    """
    @notice เพิ่ม/ลบ account จาก blacklist (เฉพาะ BLACKLIST_ROLE)
    """
    self._check_role(BLACKLIST_ROLE)
    self.is_blacklisted[account] = status
    log Blacklisted(account, status)

# ==================== Admin Functions ====================

@external
def set_transfer_fee(fee_bps: uint256):
    """
    @notice ตั้งค่า transfer fee (เฉพาะ DEFAULT_ADMIN)
    @param fee_bps Fee ในหน่วย basis points (100 = 1%, สูงสุด 1000 = 10%)
    """
    self._check_role(DEFAULT_ADMIN_ROLE)
    assert fee_bps <= 1000, "Fee too high (max 10%)"
    
    old_fee: uint256 = self.transfer_fee_bps
    self.transfer_fee_bps = fee_bps
    log FeeUpdated(old_fee, fee_bps)

@external
def set_fee_collector(new_collector: address):
    """
    @notice เปลี่ยน fee collector address (เฉพาะ DEFAULT_ADMIN)
    """
    self._check_role(DEFAULT_ADMIN_ROLE)
    assert new_collector != empty(address), "Invalid address"
    self.fee_collector = new_collector

# ==================== View Functions ====================

@view
@external
def balance_of(account: address) -> uint256:
    return self.balances[account]

@view
@external
def allowance(owner_addr: address, spender: address) -> uint256:
    return self.allowances[owner_addr][spender]

@view
@external
def has_role(role: bytes32, account: address) -> bool:
    return self.role_members[role][account]

@view
@external
def get_role_admin(role: bytes32) -> bytes32:
    return self.role_admin[role]

@view
@external
def is_minter(account: address) -> bool:
    return self.role_members[MINTER_ROLE][account]

@view
@external
def is_pauser(account: address) -> bool:
    return self.role_members[PAUSER_ROLE][account]

@view
@external
def is_admin(account: address) -> bool:
    return self.role_members[DEFAULT_ADMIN_ROLE][account]
```

---

## Test Code

```python
# tests/test_access_control.py
import pytest

DEFAULT_ADMIN_ROLE = b'\x00' * 32
MINTER_ROLE = b'\x97\x67' + b'\x00' * 30  # keccak256("MINTER_ROLE") (simplified)
PAUSER_ROLE = b'\x65\xd7' + b'\x00' * 30  # keccak256("PAUSER_ROLE") (simplified)

@pytest.fixture
def admin(accounts):
    return accounts[0]

@pytest.fixture
def minter(accounts):
    return accounts[1]

@pytest.fixture
def pauser(accounts):
    return accounts[2]

@pytest.fixture
def user(accounts):
    return accounts[3]

@pytest.fixture
def token(admin, project):
    return project.DeFiToken.deploy(
        "DeFi Token",
        "DFT",
        18,
        1_000_000 * 10**18,
        admin.address,  # fee collector
        sender=admin
    )

class TestAccessControl:
    
    def test_admin_has_default_role(self, token, admin):
        """Admin ควรมี DEFAULT_ADMIN_ROLE"""
        import eth_abi
        role = bytes(32)  # DEFAULT_ADMIN_ROLE = 0x00...00
        assert token.has_role(role, admin.address)
    
    def test_grant_minter_role(self, token, admin, minter):
        """Admin สามารถ grant MINTER_ROLE ได้"""
        from eth_utils import keccak
        minter_role = keccak(text="MINTER_ROLE")
        token.grant_role(minter_role, minter.address, sender=admin)
        assert token.has_role(minter_role, minter.address)
    
    def test_revoke_minter_role(self, token, admin, minter):
        """Admin สามารถ revoke MINTER_ROLE ได้"""
        from eth_utils import keccak
        minter_role = keccak(text="MINTER_ROLE")
        token.grant_role(minter_role, minter.address, sender=admin)
        token.revoke_role(minter_role, minter.address, sender=admin)
        assert not token.has_role(minter_role, minter.address)
    
    def test_non_admin_cannot_grant(self, token, user, minter):
        """User ธรรมดาไม่สามารถ grant role ได้"""
        from eth_utils import keccak
        minter_role = keccak(text="MINTER_ROLE")
        with pytest.raises(Exception):
            token.grant_role(minter_role, minter.address, sender=user)
    
    def test_minter_can_mint(self, token, admin, minter, user):
        """MINTER_ROLE สามารถ mint ได้"""
        from eth_utils import keccak
        minter_role = keccak(text="MINTER_ROLE")
        token.grant_role(minter_role, minter.address, sender=admin)
        
        amount = 1000 * 10**18
        token.mint(user.address, amount, sender=minter)
        assert token.balance_of(user.address) == amount
    
    def test_non_minter_cannot_mint(self, token, user):
        """Non-minter ไม่สามารถ mint ได้"""
        with pytest.raises(Exception):
            token.mint(user.address, 1000, sender=user)
    
    def test_pauser_can_pause(self, token, admin, pauser):
        """PAUSER_ROLE สามารถ pause ได้"""
        from eth_utils import keccak
        pauser_role = keccak(text="PAUSER_ROLE")
        token.grant_role(pauser_role, pauser.address, sender=admin)
        
        token.pause(sender=pauser)
        assert token.is_paused()
    
    def test_transfer_fails_when_paused(self, token, admin, minter, user, pauser):
        """Transfer ล้มเหลวเมื่อ pause"""
        from eth_utils import keccak
        
        minter_role = keccak(text="MINTER_ROLE")
        pauser_role = keccak(text="PAUSER_ROLE")
        
        token.grant_role(minter_role, admin.address, sender=admin)
        token.grant_role(pauser_role, admin.address, sender=admin)
        
        token.mint(admin.address, 1000 * 10**18, sender=admin)
        token.pause(sender=admin)
        
        with pytest.raises(Exception):
            token.transfer(user.address, 100 * 10**18, sender=admin)
    
    def test_blacklist_blocks_transfer(self, token, admin, user):
        """Blacklisted user ไม่สามารถ transfer ได้"""
        from eth_utils import keccak
        
        minter_role = keccak(text="MINTER_ROLE")
        blacklist_role = keccak(text="BLACKLIST_ROLE")
        
        token.grant_role(minter_role, admin.address, sender=admin)
        token.mint(admin.address, 1000 * 10**18, sender=admin)
        token.set_blacklist(admin.address, True, sender=admin)
        
        with pytest.raises(Exception):
            token.transfer(user.address, 100 * 10**18, sender=admin)
    
    def test_transfer_fee(self, token, admin, user):
        """ทดสอบการหัก transfer fee"""
        from eth_utils import keccak
        
        minter_role = keccak(text="MINTER_ROLE")
        token.grant_role(minter_role, admin.address, sender=admin)
        
        # ตั้ง fee 1% (100 basis points)
        token.set_transfer_fee(100, sender=admin)
        token.mint(admin.address, 1000 * 10**18, sender=admin)
        
        amount = 1000 * 10**18
        token.transfer(user.address, amount, sender=admin)
        
        fee = amount * 100 // 10000  # 1%
        expected_received = amount - fee
        
        assert token.balance_of(user.address) == expected_received
    
    def test_renounce_role(self, token, admin):
        """Admin สามารถสละ role ของตัวเองได้"""
        from eth_utils import keccak
        minter_role = keccak(text="MINTER_ROLE")
        
        # admin มี minter_role อยู่แล้ว
        token.renounce_role(minter_role, sender=admin)
        assert not token.has_role(minter_role, admin.address)
```

---

## สรุป

Access Control เป็นระบบที่ทรงพลังสำหรับ Contract ที่ซับซ้อน:

### Best Practices
1. **กำหนด Role ให้ชัดเจน** - ใช้ constants แทน magic strings
2. **หลีกเลี่ยง Role ที่ซ้อนกันมากเกินไป** - ทำให้ audit ยาก
3. **ทดสอบทุก Role** - ตรวจสอบว่า grant/revoke ทำงานถูกต้อง
4. **ระวัง DEFAULT_ADMIN_ROLE** - Role นี้ powerful มาก ควรใช้ multisig

---

[⬅️ Part 026: Ownable Pattern](part_026_ownable.md) | [Part 028: Pausable Pattern ➡️](part_028_pausable.md)
