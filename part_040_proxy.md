# Part 040: Proxy Patterns

## สารบัญ
1. [Proxy Concept](#proxy-concept)
2. [Minimal Proxy (EIP-1167)](#minimal-proxy-eip-1167)
3. [UUPS Proxy ใน Vyper](#uups-proxy-ใน-vyper)
4. [Storage Concerns](#storage-concerns)
5. [ตัวอย่าง: Clone Factory](#ตัวอย่าง-clone-factory)
6. [Test Code](#test-code)

---

## Proxy Concept

### Proxy Pattern คืออะไร?

**Proxy Pattern** คือการแยก Logic (Implementation) ออกจาก State (Storage)

```
User --> [Proxy Contract] --> [Implementation Contract]
              |                         |
         (State/Storage)          (Business Logic)
```

### ประโยชน์
1. **Upgradeable**: เปลี่ยน logic โดยไม่เสีย state
2. **Clone Factory**: Deploy contract ถูกลงมาก (EIP-1167)
3. **Gas Savings**: หลาย instances ใช้ code เดียวกัน

### ข้อจำกัดใน Vyper
Vyper ออกแบบมาเพื่อ **security** และ **simplicity** จึงไม่รองรับ:
- Assembly โดยตรง (มีแค่ `raw_call` ที่รองรับ delegatecall)
- Dynamic dispatch ที่ซับซ้อน

แต่ยังสามารถทำได้ผ่าน:
- `raw_call` กับ `is_delegate_call=True`
- EIP-1167 Minimal Proxy
- Factory Pattern

---

## Minimal Proxy (EIP-1167)

### EIP-1167 คืออะไร?

**EIP-1167** คือ standard สำหรับ deploy "clone" ของ contract ด้วย bytecode ขนาดเล็ก (45 bytes) ที่ delegate call ไปยัง implementation

### Bytecode ของ Minimal Proxy

```
3d602d80600a3d3981f3363d3d373d3d3d363d73
<implementation_address_20_bytes>
5af43d82803e903d91602b57fd5bf3
```

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Minimal Proxy Factory (EIP-1167)

event CloneDeployed:
    implementation: indexed(address)
    clone: indexed(address)
    salt: bytes32

owner: public(address)
implementation: public(address)
clones: public(DynArray[address, 1000])

@deploy
def __init__(_implementation: address):
    self.owner = msg.sender
    self.implementation = _implementation

@internal
def _create_clone(implementation_addr: address, salt: bytes32) -> address:
    """
    @notice Deploy EIP-1167 Minimal Proxy
    @dev Bytecode: 0x3d602d80600a3d3981f3363d3d373d3d3d363d73{impl}5af43d82803e903d91602b57fd5bf3
    """
    # EIP-1167 minimal proxy bytecode
    # Part 1: 0x3d602d80600a3d3981f3363d3d373d3d3d363d73
    # Part 2: <implementation address 20 bytes>
    # Part 3: 0x5af43d82803e903d91602b57fd5bf3
    
    bytecode: Bytes[55] = concat(
        0x3d602d80600a3d3981f3363d3d373d3d3d363d73,
        convert(implementation_addr, bytes20),
        0x5af43d82803e903d91602b57fd5bf3
    )
    
    clone_address: address = create_from_blueprint(
        implementation_addr,
        salt=salt,
        code_offset=0
    )
    
    return clone_address

@external
def deploy_clone(salt: bytes32) -> address:
    """Deploy clone ด้วย CREATE2"""
    assert msg.sender == self.owner, "Not owner"
    
    clone: address = self._create_clone(self.implementation, salt)
    self.clones.append(clone)
    
    log CloneDeployed(self.implementation, clone, salt)
    
    return clone

@view
@external
def predict_clone_address(salt: bytes32) -> address:
    """คำนวณ address ของ clone ล่วงหน้า"""
    # EIP-1167 bytecode hash สำหรับ CREATE2
    bytecode_hash: bytes32 = keccak256(concat(
        0x3d602d80600a3d3981f3363d3d373d3d3d363d73,
        convert(self.implementation, bytes20),
        0x5af43d82803e903d91602b57fd5bf3
    ))
    
    return convert(
        keccak256(
            concat(
                b"\xff",
                convert(self, bytes20),
                salt,
                bytecode_hash
            )
        ),
        address
    )

@view
@external
def get_all_clones() -> DynArray[address, 1000]:
    return self.clones
```

### ตัวอย่างใช้งาน: Token Clone Factory

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Simple Token Implementation (for cloning)
# @notice Contract นี้จะถูก clone

event Transfer:
    sender: indexed(address)
    receiver: indexed(address)
    value: uint256

event Initialized:
    owner: address
    name: String[64]
    symbol: String[32]

# State variables ที่ clone แต่ละตัวมีของตัวเอง
name: public(String[64])
symbol: public(String[32])
decimals: public(uint8)
total_supply: public(uint256)
balances: public(HashMap[address, uint256])
owner: public(address)
initialized: public(bool)

@deploy
def __init__():
    # Constructor ว่างเปล่า (implementation ไม่ initialize จริง)
    pass

@external
def initialize(
    _name: String[64],
    _symbol: String[32],
    _decimals: uint8,
    _initial_supply: uint256,
    _owner: address
):
    """
    @notice Initialize clone instance
    @dev เรียกแทน constructor สำหรับ clones
    """
    assert not self.initialized, "Already initialized"
    
    self.name = _name
    self.symbol = _symbol
    self.decimals = _decimals
    self.owner = _owner
    self.initialized = True
    
    if _initial_supply > 0:
        self.total_supply = _initial_supply
        self.balances[_owner] = _initial_supply
        log Transfer(empty(address), _owner, _initial_supply)
    
    log Initialized(_owner, _name, _symbol)

@external
def transfer(to: address, amount: uint256) -> bool:
    assert self.initialized, "Not initialized"
    assert self.balances[msg.sender] >= amount, "Insufficient balance"
    
    self.balances[msg.sender] -= amount
    self.balances[to] += amount
    
    log Transfer(msg.sender, to, amount)
    return True

@external
def mint(to: address, amount: uint256):
    assert msg.sender == self.owner, "Not owner"
    
    self.total_supply += amount
    self.balances[to] += amount
    log Transfer(empty(address), to, amount)
```

---

## UUPS Proxy ใน Vyper

### UUPS (Universal Upgradeable Proxy Standard)

ใน Vyper สามารถทำ Upgradeable Proxy โดยใช้ `raw_call` กับ `is_delegate_call=True`

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Simple Upgradeable Proxy
# @notice Proxy ที่ใช้ delegatecall

event Upgraded:
    old_implementation: indexed(address)
    new_implementation: indexed(address)

# Storage slot สำหรับ implementation address
# EIP-1967: keccak256("eip1967.proxy.implementation") - 1
IMPLEMENTATION_SLOT: constant(bytes32) = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc

admin: public(address)
implementation: public(address)

@deploy
def __init__(_implementation: address, _admin: address):
    assert _implementation != empty(address), "Invalid implementation"
    self.implementation = _implementation
    self.admin = _admin

@external
def upgrade(new_implementation: address):
    """อัปเกรด implementation"""
    assert msg.sender == self.admin, "Not admin"
    assert new_implementation != empty(address), "Invalid address"
    
    old: address = self.implementation
    self.implementation = new_implementation
    
    log Upgraded(old, new_implementation)

@external
@payable
def __default__():
    """
    Fallback: delegate call ไปยัง implementation
    """
    impl: address = self.implementation
    assert impl != empty(address), "No implementation"
    
    # Delegate call ไปยัง implementation
    response: Bytes[32768] = raw_call(
        impl,
        msg.data,
        max_outsize=32768,
        is_delegate_call=True,
        value=msg.value
    )
    
    return response
```

### Implementation Contract ที่ Upgradeable

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title V1 Implementation

# Storage layout ต้องตรงกับ V2!
owner: public(address)
value: public(uint256)
version: public(String[10])

@deploy
def __init__():
    pass

@external
def initialize(_owner: address):
    assert self.owner == empty(address), "Already initialized"
    self.owner = _owner
    self.version = "v1"

@external
def set_value(new_value: uint256):
    assert msg.sender == self.owner, "Not owner"
    self.value = new_value

@view
@external
def get_info() -> (address, uint256, String[10]):
    return self.owner, self.value, self.version
```

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title V2 Implementation (Upgraded)
# @notice เพิ่มฟีเจอร์ใหม่ แต่ storage layout เหมือนกัน

# Storage layout เหมือน V1 ทุกอย่าง!
owner: public(address)
value: public(uint256)
version: public(String[10])

# Storage ใหม่ต้องอยู่ท้ายสุด
multiplier: public(uint256)

@deploy
def __init__():
    pass

@external
def initialize(_owner: address):
    assert self.owner == empty(address), "Already initialized"
    self.owner = _owner
    self.version = "v2"
    self.multiplier = 1

@external
def set_value(new_value: uint256):
    assert msg.sender == self.owner, "Not owner"
    self.value = new_value * self.multiplier  # ฟีเจอร์ใหม่!

@external
def set_multiplier(new_multiplier: uint256):
    assert msg.sender == self.owner, "Not owner"
    assert new_multiplier > 0, "Must > 0"
    self.multiplier = new_multiplier

@view
@external
def get_info() -> (address, uint256, String[10]):
    return self.owner, self.value, self.version
```

---

## Storage Concerns

### ⚠️ สำคัญมาก: Storage Collision

เมื่อใช้ delegatecall storage ของ proxy และ implementation ต้องไม่ชนกัน

```
❌ Storage Collision (อย่าทำ)

Proxy Storage:           Implementation Storage:
slot 0: admin            slot 0: owner      <-- ชนกัน!
slot 1: implementation   slot 1: value      <-- ชนกัน!
```

```
✅ Safe Storage (ทำแบบนี้)

Proxy (EIP-1967):        Implementation Storage:
slot 0x360894...: impl   slot 0: owner      <-- ไม่ชน
slot 0xb53127...: admin  slot 1: value      <-- ไม่ชน
```

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Storage Layout Demo

# ❌ แบบไม่ปลอดภัย - admin ชนกับ owner ของ implementation
struct UnsafeProxy:
    admin: address        # slot 0 - ชน!
    implementation: address  # slot 1 - ชน!

# ✅ แบบปลอดภัย - ใช้ EIP-1967 slots
# Implementation address stored at: 
# keccak256('eip1967.proxy.implementation') - 1
# = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc

# Admin address stored at:
# keccak256('eip1967.proxy.admin') - 1  
# = 0xb53127684a568b3173ae13b9f8a6016e243e63b6e8ee1178d6a717850b5d6103
```

---

## ตัวอย่าง: Clone Factory

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Token Clone Factory
# @notice Factory สำหรับ deploy ERC20 token clones

interface ITokenImpl:
    def initialize(
        name: String[64],
        symbol: String[32],
        decimals: uint8,
        initial_supply: uint256,
        owner: address
    ): nonpayable
    def balanceOf(account: address) -> uint256: view
    def transfer(to: address, amount: uint256) -> bool: nonpayable

event TokenCreated:
    creator: indexed(address)
    token: indexed(address)
    name: String[64]
    symbol: String[32]
    initial_supply: uint256

struct TokenInfo:
    token_address: address
    name: String[64]
    symbol: String[32]
    creator: address
    created_at: uint256

MAX_TOKENS: constant(uint256) = 10000
CREATION_FEE: constant(uint256) = 1000000000000000  # 0.001 ETH

owner: public(address)
implementation: public(address)
fee_recipient: public(address)

tokens: public(DynArray[TokenInfo, MAX_TOKENS])
tokens_by_creator: public(HashMap[address, DynArray[address, 100]])
token_count: public(uint256)

@deploy
def __init__(_implementation: address, _fee_recipient: address):
    self.owner = msg.sender
    self.implementation = _implementation
    self.fee_recipient = _fee_recipient

@external
@payable
def create_token(
    name: String[64],
    symbol: String[32],
    decimals: uint8,
    initial_supply: uint256
) -> address:
    """
    @notice สร้าง token ใหม่โดย clone จาก implementation
    """
    assert msg.value >= CREATION_FEE, "Insufficient fee"
    
    # สร้าง salt จาก creator + name + timestamp
    salt: bytes32 = keccak256(
        concat(
            convert(msg.sender, bytes32),
            convert(block.timestamp, bytes32),
            convert(self.token_count, bytes32)
        )
    )
    
    # Deploy clone โดยใช้ create_from_blueprint
    new_token: address = create_from_blueprint(
        self.implementation,
        salt=salt
    )
    
    # Initialize token
    ITokenImpl(new_token).initialize(
        name,
        symbol,
        decimals,
        initial_supply,
        msg.sender
    )
    
    # บันทึกข้อมูล
    token_info: TokenInfo = TokenInfo({
        token_address: new_token,
        name: name,
        symbol: symbol,
        creator: msg.sender,
        created_at: block.timestamp
    })
    
    self.tokens.append(token_info)
    self.tokens_by_creator[msg.sender].append(new_token)
    self.token_count += 1
    
    # ส่ง fee ให้ recipient
    send(self.fee_recipient, msg.value)
    
    log TokenCreated(msg.sender, new_token, name, symbol, initial_supply)
    
    return new_token

@view
@external
def get_tokens_by_creator(creator: address) -> DynArray[address, 100]:
    """ดูรายการ tokens ที่ creator สร้าง"""
    return self.tokens_by_creator[creator]

@view
@external
def get_all_tokens() -> DynArray[TokenInfo, MAX_TOKENS]:
    """ดูรายการ tokens ทั้งหมด"""
    return self.tokens

@view
@external
def get_token_info(index: uint256) -> TokenInfo:
    assert index < self.token_count, "Invalid index"
    return self.tokens[index]

@external
def update_implementation(new_impl: address):
    """อัปเดต implementation (ไม่กระทบ clones ที่มีอยู่)"""
    assert msg.sender == self.owner, "Not owner"
    assert new_impl != empty(address), "Invalid address"
    self.implementation = new_impl

@external
def withdraw_fees():
    """เบิก fees"""
    assert msg.sender == self.owner, "Not owner"
    send(self.fee_recipient, self.balance)
```

### Complete Working Contract: Multi-Vault Factory

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Vault Implementation (for cloning)
# @notice ERC4626-inspired Vault ที่จะถูก clone

interface IERC20:
    def balanceOf(account: address) -> uint256: view
    def transfer(to: address, amount: uint256) -> bool: nonpayable
    def transferFrom(from_: address, to: address, amount: uint256) -> bool: nonpayable
    def approve(spender: address, amount: uint256) -> bool: nonpayable

event Deposit:
    caller: indexed(address)
    owner: indexed(address)
    assets: uint256
    shares: uint256

event Withdraw:
    caller: indexed(address)
    receiver: indexed(address)
    owner: indexed(address)
    assets: uint256
    shares: uint256

event VaultInitialized:
    asset: indexed(address)
    vault_name: String[64]

# State variables
asset: public(address)
vault_name: public(String[64])
vault_symbol: public(String[32])
total_assets_stored: public(uint256)
total_shares: public(uint256)
shares: public(HashMap[address, uint256])
owner: public(address)
initialized: public(bool)

@deploy
def __init__():
    pass

@external
def initialize(
    _asset: address,
    _name: String[64],
    _symbol: String[32],
    _owner: address
):
    assert not self.initialized, "Already initialized"
    assert _asset != empty(address), "Invalid asset"
    
    self.asset = _asset
    self.vault_name = _name
    self.vault_symbol = _symbol
    self.owner = _owner
    self.initialized = True
    
    log VaultInitialized(_asset, _name)

@internal
def _convert_to_shares(assets: uint256) -> uint256:
    if self.total_shares == 0 or self.total_assets_stored == 0:
        return assets
    return assets * self.total_shares / self.total_assets_stored

@internal
def _convert_to_assets(share_amount: uint256) -> uint256:
    if self.total_shares == 0:
        return share_amount
    return share_amount * self.total_assets_stored / self.total_shares

@external
@nonreentrant
def deposit(assets: uint256, receiver: address) -> uint256:
    assert self.initialized, "Not initialized"
    assert assets > 0, "Zero assets"
    
    share_amount: uint256 = self._convert_to_shares(assets)
    
    # CEI pattern
    self.total_assets_stored += assets
    self.total_shares += share_amount
    self.shares[receiver] += share_amount
    
    IERC20(self.asset).transferFrom(msg.sender, self, assets)
    
    log Deposit(msg.sender, receiver, assets, share_amount)
    
    return share_amount

@external
@nonreentrant
def withdraw(assets: uint256, receiver: address, owner_addr: address) -> uint256:
    assert self.initialized, "Not initialized"
    assert assets > 0, "Zero assets"
    
    share_amount: uint256 = self._convert_to_shares(assets)
    assert self.shares[owner_addr] >= share_amount, "Insufficient shares"
    
    self.shares[owner_addr] -= share_amount
    self.total_shares -= share_amount
    self.total_assets_stored -= assets
    
    IERC20(self.asset).transfer(receiver, assets)
    
    log Withdraw(msg.sender, receiver, owner_addr, assets, share_amount)
    
    return share_amount

@view
@external
def preview_deposit(assets: uint256) -> uint256:
    return self._convert_to_shares(assets)

@view
@external
def preview_withdraw(assets: uint256) -> uint256:
    return self._convert_to_shares(assets)

@view
@external
def max_withdraw(owner_addr: address) -> uint256:
    return self._convert_to_assets(self.shares[owner_addr])

@view
@external
def balance_of(account: address) -> uint256:
    return self.shares[account]
```

---

## Test Code

```python
# tests/test_proxy.py
import pytest
from ape import accounts, project

@pytest.fixture
def owner(accounts):
    return accounts[0]

@pytest.fixture
def user(accounts):
    return accounts[1]

@pytest.fixture
def token_impl(owner, project):
    """Deploy implementation contract"""
    return project.SimpleTokenImpl.deploy(sender=owner)

@pytest.fixture
def factory(owner, token_impl, project):
    return project.TokenCloneFactory.deploy(
        token_impl.address,
        owner.address,
        sender=owner
    )

@pytest.fixture
def mock_erc20(owner, project):
    t = project.MockERC20.deploy("Test Token", "TEST", 18, sender=owner)
    return t

@pytest.fixture
def vault_impl(owner, project):
    return project.VaultImpl.deploy(sender=owner)

class TestCloneFactory:
    
    def test_create_token(self, factory, user):
        """ทดสอบสร้าง token clone"""
        creation_fee = factory.CREATION_FEE()
        
        tx = factory.create_token(
            "My Token",
            "MTK",
            18,
            1_000_000 * 10**18,
            sender=user,
            value=creation_fee
        )
        
        # ตรวจสอบ event
        event = tx.events.filter(factory.TokenCreated)[0]
        assert event.creator == user.address
        assert event.name == "My Token"
        
        # ตรวจสอบ token count
        assert factory.token_count() == 1
    
    def test_clone_is_independent(self, factory, user, accounts):
        """ทดสอบว่า clones แต่ละตัวมี state ของตัวเอง"""
        creation_fee = factory.CREATION_FEE()
        user2 = accounts[2]
        
        tx1 = factory.create_token(
            "Token One", "ONE", 18, 1000 * 10**18,
            sender=user, value=creation_fee
        )
        
        tx2 = factory.create_token(
            "Token Two", "TWO", 18, 2000 * 10**18,
            sender=user2, value=creation_fee
        )
        
        token1 = tx1.events.filter(factory.TokenCreated)[0].token
        token2 = tx2.events.filter(factory.TokenCreated)[0].token
        
        # Clones ต้องเป็น address คนละตัว
        assert token1 != token2
        
        # แต่ละ clone มี state ของตัวเอง
        t1 = project.SimpleTokenImpl.at(token1)
        t2 = project.SimpleTokenImpl.at(token2)
        
        assert t1.name() == "Token One"
        assert t2.name() == "Token Two"
        assert t1.balanceOf(user.address) == 1000 * 10**18
        assert t2.balanceOf(user2.address) == 2000 * 10**18
    
    def test_insufficient_fee_rejected(self, factory, user):
        """ทดสอบ reject เมื่อ fee ไม่พอ"""
        with pytest.raises(Exception):
            factory.create_token(
                "Token", "TKN", 18, 1000,
                sender=user,
                value=0  # ไม่ส่ง fee
            )
    
    def test_get_tokens_by_creator(self, factory, user):
        """ทดสอบดูรายการ tokens ของ creator"""
        creation_fee = factory.CREATION_FEE()
        
        for i in range(3):
            factory.create_token(
                f"Token {i}", f"TKN{i}", 18, 1000 * 10**18,
                sender=user, value=creation_fee
            )
        
        tokens = factory.get_tokens_by_creator(user.address)
        assert len(tokens) == 3

class TestVaultClone:
    
    def test_vault_deposit_withdraw(
        self, vault_impl, mock_erc20, owner, user, project
    ):
        """ทดสอบ vault deposit/withdraw"""
        # Deploy vault ผ่าน factory
        vault_factory = project.VaultCloneFactory.deploy(
            vault_impl.address,
            owner.address,
            sender=owner
        )
        
        tx = vault_factory.create_vault(
            mock_erc20.address,
            "My Vault",
            "mvTKN",
            sender=owner,
            value=vault_factory.CREATION_FEE()
        )
        
        vault_addr = tx.events.filter(vault_factory.VaultCreated)[0].vault
        vault = project.VaultImpl.at(vault_addr)
        
        # Mint tokens ให้ user
        mock_erc20.mint(user.address, 100 * 10**18, sender=owner)
        mock_erc20.approve(vault_addr, 100 * 10**18, sender=user)
        
        # Deposit
        vault.deposit(50 * 10**18, user.address, sender=user)
        
        assert vault.balance_of(user.address) == 50 * 10**18
        
        # Withdraw
        vault.withdraw(50 * 10**18, user.address, user.address, sender=user)
        
        assert vault.balance_of(user.address) == 0
        assert mock_erc20.balanceOf(user.address) == 100 * 10**18

class TestUpgradeableProxy:
    
    def test_upgrade_implementation(self, owner, project):
        """ทดสอบ upgrade implementation"""
        v1 = project.ImplV1.deploy(sender=owner)
        proxy = project.SimpleProxy.deploy(v1.address, owner.address, sender=owner)
        
        # ใช้งาน v1
        impl_via_proxy = project.ImplV1.at(proxy.address)
        impl_via_proxy.initialize(owner.address, sender=owner)
        impl_via_proxy.set_value(42, sender=owner)
        assert impl_via_proxy.value() == 42
        
        # Upgrade เป็น v2
        v2 = project.ImplV2.deploy(sender=owner)
        proxy.upgrade(v2.address, sender=owner)
        
        # State ยังอยู่
        impl_v2_via_proxy = project.ImplV2.at(proxy.address)
        assert impl_v2_via_proxy.value() == 42  # state preserved
        assert impl_v2_via_proxy.version() == "v2"
    
    def test_only_admin_can_upgrade(self, owner, user, project):
        """ทดสอบว่าแค่ admin upgrade ได้"""
        v1 = project.ImplV1.deploy(sender=owner)
        proxy = project.SimpleProxy.deploy(v1.address, owner.address, sender=owner)
        v2 = project.ImplV2.deploy(sender=owner)
        
        with pytest.raises(Exception):
            proxy.upgrade(v2.address, sender=user)
```

---

## สรุป

Proxy Patterns ใน Vyper:

| Pattern | Use Case | Gas Cost |
|---------|----------|----------|
| EIP-1167 Clone | หลาย instances ของ contract เดียวกัน | ต่ำมาก (~45 bytes) |
| Upgradeable Proxy | ต้องการ upgrade logic | ปานกลาง |
| Factory + Clone | Launchpad, DEX, Vault factory | ต่ำ |

### Key Takeaways
1. **EIP-1167** ดีที่สุดสำหรับ clone factory - ถูกมาก
2. **Storage Layout** ต้องคงเดิมเมื่อ upgrade
3. **initialize()** แทน constructor สำหรับ clones
4. **Delegatecall** ใช้ storage ของ proxy แต่ code ของ implementation

### Security Checklist
- [ ] ตรวจสอบ storage layout ไม่ชนกัน
- [ ] มี `initialized` guard ใน initialize()
- [ ] Protect upgrade function ด้วย access control
- [ ] Test ทุก state transition หลัง upgrade
- [ ] ระวัง selfdestruct ใน implementation

---

[⬅️ Part 039: Cross-Contract Calls](part_039_cross_contract.md) | [Part 001: Introduction ➡️](part_001_intro.md)

---

## จบ Series: Security Patterns & DeFi (Part 026-040)

ยินดีด้วย! คุณได้เรียนรู้ Security Patterns และ DeFi Patterns ครบทั้ง 15 บทแล้ว:

| Part | หัวข้อ |
|------|--------|
| 026 | Ownable Pattern |
| 027 | Access Control (RBAC) |
| 028 | Pausable Pattern |
| 029 | ReentrancyGuard |
| 030 | Timelock |
| 031 | Multisig Wallet |
| 032 | Escrow |
| 033 | Voting & Governance |
| 034 | Auction Patterns |
| 035 | Staking |
| 036 | Token Vesting |
| 037 | Merkle Tree Proofs |
| 038 | Oracle Integration |
| 039 | Cross-Contract Calls |
| 040 | Proxy Patterns |
