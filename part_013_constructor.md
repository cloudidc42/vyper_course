# Part 013: Constructor และ Initialization

## สารบัญ
1. [@deploy def __init__](#init)
2. [Constructor Parameters](#parameters)
3. [Immutables](#immutables)
4. [Initialization Patterns](#init-patterns)
5. [Factory Pattern Basics](#factory)
6. [ตัวอย่าง: Configurable Token Contract](#example)

---

## 1. @deploy def __init__ {#init}

Constructor ใน Vyper 0.4.0 ใช้ `@deploy` decorator

```python
# @version 0.4.0

# ════════════════════════════════════════
# @deploy def __init__
# ════════════════════════════════════════

# Constructor พื้นฐาน
owner: address

@deploy
def __init__():
    self.owner = msg.sender

# Constructor ใน Vyper 0.4.0 ต้องใช้ @deploy
# ใน Vyper 0.3.x ใช้แค่ def __init__() โดยไม่มี decorator
```

### Constructor ทำงานอย่างไร

```python
# @version 0.4.0

# ════════════════════════════════════════
# Constructor Execution
# ════════════════════════════════════════

# Constructor ทำงานครั้งเดียวตอน deploy
# ไม่สามารถเรียกซ้ำได้หลัง deploy
# msg.sender ใน __init__ = deployer

name: String[50]
symbol: String[10]
owner: address
created_at: uint256
total_supply: uint256

event ContractCreated:
    owner: indexed(address)
    name: String[50]
    created_at: uint256

@deploy
def __init__(
    name: String[50],
    symbol: String[10],
    initial_supply: uint256
):
    # Set up all state in constructor
    self.name = name
    self.symbol = symbol
    self.owner = msg.sender
    self.created_at = block.timestamp
    self.total_supply = initial_supply

    # Can emit events in constructor
    log ContractCreated(msg.sender, name, block.timestamp)

@view
@external
def get_creation_info() -> (String[50], address, uint256):
    return self.name, self.owner, self.created_at
```

### Constructor กับ ETH

```python
# @version 0.4.0

# Constructor รับ ETH ได้ (ถ้าใส่ @payable)
owner: address
initial_balance: uint256

@payable
@deploy
def __init__():
    self.owner = msg.sender
    self.initial_balance = msg.value
    # ETH ส่งมาตอน deploy เก็บอยู่ใน contract

@view
@external
def get_initial_balance() -> uint256:
    return self.initial_balance
```

---

## 2. Constructor Parameters {#parameters}

```python
# @version 0.4.0

# ════════════════════════════════════════
# Constructor Parameters
# ════════════════════════════════════════

struct TokenConfig:
    name: String[50]
    symbol: String[10]
    decimals: uint8
    initial_supply: uint256
    owner: address

# Primitive parameters
owner: address
name: String[50]
max_supply: uint256
fee_bps: uint256
is_mintable: bool

@deploy
def __init__(
    name: String[50],
    max_supply: uint256,
    fee_bps: uint256,
    is_mintable: bool
):
    assert len(name) > 0, "Name required"
    assert max_supply > 0, "Zero supply"
    assert fee_bps <= 10000, "Fee too high"

    self.owner = msg.sender
    self.name = name
    self.max_supply = max_supply
    self.fee_bps = fee_bps
    self.is_mintable = is_mintable
```

### Parameters หลากหลาย Types

```python
# @version 0.4.0

# ════════════════════════════════════════
# Various Parameter Types in Constructor
# ════════════════════════════════════════

struct VestingSchedule:
    cliff: uint256
    duration: uint256
    amount: uint256

owner: address
treasury: address
token_address: address
start_time: uint256
vesting: VestingSchedule
beneficiaries: DynArray[address, 100]
allocations: DynArray[uint256, 100]
metadata_hash: bytes32

@deploy
def __init__(
    treasury: address,
    token: address,
    cliff_duration: uint256,
    vesting_duration: uint256,
    total_amount: uint256,
    initial_beneficiaries: DynArray[address, 100],
    initial_allocations: DynArray[uint256, 100],
    metadata: bytes32
):
    assert treasury != empty(address), "Invalid treasury"
    assert token != empty(address), "Invalid token"
    assert cliff_duration < vesting_duration, "Cliff >= duration"
    assert len(initial_beneficiaries) == len(initial_allocations), "Length mismatch"

    self.owner = msg.sender
    self.treasury = treasury
    self.token_address = token
    self.start_time = block.timestamp
    self.vesting = VestingSchedule({
        cliff: cliff_duration,
        duration: vesting_duration,
        amount: total_amount
    })
    self.beneficiaries = initial_beneficiaries
    self.allocations = initial_allocations
    self.metadata_hash = metadata

    # Validate total allocations
    total: uint256 = 0
    for a: uint256 in initial_allocations:
        total += a
    assert total == total_amount, "Allocation mismatch"
```

### Constructor Validation

```python
# @version 0.4.0

MINIMUM_PERIOD: constant(uint256) = 7 * 24 * 3600  # 1 week
MAXIMUM_FEE: constant(uint256) = 3000  # 30%
MAX_ADMIN_COUNT: constant(uint256) = 10

owner: address
admins: DynArray[address, 10]
fee_bps: uint256
lock_period: uint256

@deploy
def __init__(
    admins: DynArray[address, 10],
    fee_bps: uint256,
    lock_period: uint256
):
    # Validate inputs
    assert len(admins) > 0, "Need at least one admin"
    assert len(admins) <= MAX_ADMIN_COUNT, "Too many admins"
    assert fee_bps <= MAXIMUM_FEE, "Fee exceeds maximum"
    assert lock_period >= MINIMUM_PERIOD, "Lock period too short"

    # Check for duplicates and zero addresses
    for i: uint256 in range(10, bound=10):
        if i >= convert(len(admins), uint256):
            break
        admin: address = admins[i]
        assert admin != empty(address), "Zero address admin"
        # Check no duplicates
        for j: uint256 in range(10, bound=10):
            if j >= i:
                break
            assert admins[j] != admin, "Duplicate admin"

    self.owner = msg.sender
    self.admins = admins
    self.fee_bps = fee_bps
    self.lock_period = lock_period
```

---

## 3. Immutables {#immutables}

Immutables ถูก set ครั้งเดียวใน constructor แล้วไม่เปลี่ยน เก็บใน bytecode (ถูกกว่า storage)

```python
# @version 0.4.0

# ════════════════════════════════════════
# Immutables
# ════════════════════════════════════════

# Immutables ประกาศด้วย immutable()
OWNER: immutable(address)
TOKEN: immutable(address)
CREATION_TIME: immutable(uint256)
MAX_SUPPLY: immutable(uint256)
NAME: immutable(String[50])
CHAIN_ID: immutable(uint256)

@deploy
def __init__(
    token: address,
    max_supply: uint256,
    name: String[50]
):
    # Immutables ต้อง set ใน __init__
    OWNER = msg.sender
    TOKEN = token
    CREATION_TIME = block.timestamp
    MAX_SUPPLY = max_supply
    NAME = name
    CHAIN_ID = chain.id  # block.chainid ใน Vyper

# ใช้ immutables ใน functions (ไม่มี self.)
@view
@external
def get_owner() -> address:
    return OWNER

@view
@external
def get_max_supply() -> uint256:
    return MAX_SUPPLY

@view
@external
def get_name() -> String[50]:
    return NAME

@view
@external
def get_creation_time() -> uint256:
    return CREATION_TIME
```

### Immutable vs Constant vs Storage

```python
# @version 0.4.0

# ════════════════════════════════════════
# Immutable vs Constant vs Storage
# ════════════════════════════════════════

# CONSTANT: รู้ค่า ณ compile time, เก็บใน bytecode
DECIMALS: constant(uint8) = 18
MAX_INT: constant(uint256) = max_value(uint256)
ZERO_BYTES: constant(bytes32) = empty(bytes32)

# IMMUTABLE: รู้ค่าตอน deploy, เก็บใน bytecode
OWNER: immutable(address)
TOKEN_ADDRESS: immutable(address)
DEPLOY_BLOCK: immutable(uint256)

# STORAGE: เปลี่ยนได้, เก็บใน storage slots (แพงกว่า)
total_supply: uint256
balances: HashMap[address, uint256]

@deploy
def __init__(token: address):
    OWNER = msg.sender
    TOKEN_ADDRESS = token
    DEPLOY_BLOCK = block.number

# Gas comparison (approximate reads):
# CONSTANT: 0 gas (embedded in bytecode)
# IMMUTABLE: ~3 gas (in bytecode)
# STORAGE: ~800 gas (SLOAD)

@view
@external
def get_constant() -> uint8:
    return DECIMALS  # 0 gas

@view
@external
def get_immutable() -> address:
    return OWNER  # ~3 gas

@view
@external
def get_storage() -> uint256:
    return self.total_supply  # ~800 gas
```

### Immutables ที่ซับซ้อน

```python
# @version 0.4.0

# Immutables สำหรับ DeFi
WETH: immutable(address)
USDC: immutable(address)
UNISWAP_ROUTER: immutable(address)
FEE_TIER: immutable(uint24)
TICK_SPACING: immutable(int24)

# Computed immutables
PAIR_HASH: immutable(bytes32)

@deploy
def __init__(
    weth: address,
    usdc: address,
    router: address,
    fee: uint24,
    tick: int24
):
    WETH = weth
    USDC = usdc
    UNISWAP_ROUTER = router
    FEE_TIER = fee
    TICK_SPACING = tick
    # Compute pair hash at deploy time
    PAIR_HASH = keccak256(
        concat(
            convert(convert(weth, uint256), bytes32),
            convert(convert(usdc, uint256), bytes32)
        )
    )

@view
@external
def get_pair_hash() -> bytes32:
    return PAIR_HASH

@view
@external
def get_pool_config() -> (address, address, uint24):
    return WETH, USDC, FEE_TIER
```

---

## 4. Initialization Patterns {#init-patterns}

### Pattern: Two-Phase Initialization

```python
# @version 0.4.0

# ════════════════════════════════════════
# Two-Phase Initialization
# Deploy -> Initialize
# ════════════════════════════════════════

owner: address
initialized: bool
config_address: address
treasury: address
fee_bps: uint256

@deploy
def __init__():
    # Phase 1: Minimal initialization
    self.owner = msg.sender
    self.initialized = False

@external
def initialize(
    config: address,
    treasury: address,
    fee_bps: uint256
):
    # Phase 2: Full initialization (can only call once)
    assert msg.sender == self.owner, "Not owner"
    assert not self.initialized, "Already initialized"
    assert config != empty(address), "Invalid config"
    assert treasury != empty(address), "Invalid treasury"
    assert fee_bps <= 1000, "Fee too high"

    self.config_address = config
    self.treasury = treasury
    self.fee_bps = fee_bps
    self.initialized = True

@internal
def _require_initialized():
    assert self.initialized, "Not initialized"

@external
def do_something():
    self._require_initialized()
    # ... rest of logic
```

### Pattern: Upgradeable Proxy

```python
# @version 0.4.0

# ════════════════════════════════════════
# Proxy-Compatible Initialization
# (สำหรับ Upgradeable Contracts)
# ════════════════════════════════════════

owner: address
initialized: bool
version: uint256

# Slot สำหรับ admin (เพื่อหลีกเลี่ยง collision)
# ใช้ keccak256-derived slot

@deploy
def __init__():
    # สำหรับ logic contract: initialize minimal
    self.owner = msg.sender

@external
def initialize_proxy(new_owner: address):
    """ใช้สำหรับ proxy initialization (เรียกครั้งเดียว)"""
    assert not self.initialized, "Already initialized"
    assert new_owner != empty(address), "Invalid owner"
    self.owner = new_owner
    self.initialized = True
    self.version = 1
```

### Pattern: Registry-Based Initialization

```python
# @version 0.4.0

interface IRegistry:
    def get_address(key: bytes32) -> address: view
    def is_registered(addr: address) -> bool: view

REGISTRY: immutable(address)
owner: address
token: address
pool: address

@deploy
def __init__(registry: address):
    assert registry != empty(address), "Invalid registry"
    REGISTRY = registry
    self.owner = msg.sender
    # Lookup addresses from registry at deploy time
    self.token = IRegistry(registry).get_address(keccak256("TOKEN"))
    self.pool = IRegistry(registry).get_address(keccak256("POOL"))

@view
@external
def get_addresses() -> (address, address):
    return self.token, self.pool
```

### Pattern: Clone-Friendly Initialization

```python
# @version 0.4.0

# ════════════════════════════════════════
# Minimal Clone Pattern (EIP-1167 compatible)
# ════════════════════════════════════════

owner: address
name: String[50]
decimals: uint8
total_supply: uint256
balances: HashMap[address, uint256]
is_initialized: bool

@deploy
def __init__():
    # Logic contract: just mark owner
    self.owner = msg.sender

@external
def init(
    owner_addr: address,
    name: String[50],
    decimals: uint8,
    initial_supply: uint256
):
    """Called by factory after cloning"""
    assert not self.is_initialized, "Already init"
    assert owner_addr != empty(address), "Invalid"

    self.owner = owner_addr
    self.name = name
    self.decimals = decimals
    self.total_supply = initial_supply
    self.balances[owner_addr] = initial_supply
    self.is_initialized = True
```

---

## 5. Factory Pattern Basics {#factory}

```python
# @version 0.4.0

# ════════════════════════════════════════
# Factory Pattern
# Contract ที่สร้าง Contract อื่น
# ════════════════════════════════════════

# Child contract (ที่ Factory สร้าง)
# ไฟล์: SimpleVault.vy
# @version 0.4.0
# owner: address
# @deploy
# def __init__(vault_owner: address):
#     self.owner = vault_owner

# Factory Contract
VAULT_BYTECODE: immutable(Bytes[10000])

struct VaultInfo:
    vault_address: address
    owner: address
    created_at: uint256
    vault_id: uint256

owner: address
vaults: DynArray[VaultInfo, 1000]
vault_by_owner: HashMap[address, DynArray[address, 100]]
vault_count: uint256

event VaultCreated:
    vault_address: indexed(address)
    vault_owner: indexed(address)
    vault_id: indexed(uint256)

@deploy
def __init__():
    self.owner = msg.sender
    # In practice, bytecode would be set here
    VAULT_BYTECODE = b""  # Placeholder

@external
def create_vault() -> address:
    """Create a new vault for the caller"""
    # In Vyper, use create_minimal_proxy_to or raw_call with CREATE2
    # This is a simplified version

    vault_id: uint256 = self.vault_count
    self.vault_count += 1

    # Record vault info
    # (actual address would come from create_minimal_proxy_to)
    vault_addr: address = empty(address)  # Placeholder

    self.vaults.append(VaultInfo({
        vault_address: vault_addr,
        owner: msg.sender,
        created_at: block.timestamp,
        vault_id: vault_id
    }))
    self.vault_by_owner[msg.sender].append(vault_addr)

    log VaultCreated(vault_addr, msg.sender, vault_id)
    return vault_addr

@view
@external
def get_vaults_by_owner(owner_addr: address) -> DynArray[address, 100]:
    return self.vault_by_owner[owner_addr]

@view
@external
def get_vault_count() -> uint256:
    return self.vault_count
```

### Factory with create_minimal_proxy_to

```python
# @version 0.4.0

# ════════════════════════════════════════
# Minimal Proxy Factory (EIP-1167)
# ════════════════════════════════════════

interface IMinimalVault:
    def init(owner_addr: address, name: String[50]) -> bool: nonpayable
    def owner() -> address: view

IMPLEMENTATION: immutable(address)

struct DeployedVault:
    addr: address
    name: String[50]
    owner_addr: address
    deployed_at: uint256

deployed_vaults: DynArray[DeployedVault, 10000]
vault_index: HashMap[address, uint256]  # vault addr -> index+1

event VaultDeployed:
    vault: indexed(address)
    owner: indexed(address)
    name: String[50]

@deploy
def __init__(implementation: address):
    assert implementation != empty(address), "Invalid implementation"
    IMPLEMENTATION = implementation

@external
def deploy_vault(name: String[50]) -> address:
    """Deploy a minimal proxy vault"""
    assert len(name) > 0, "Name required"

    # Create minimal proxy
    new_vault: address = create_minimal_proxy_to(IMPLEMENTATION)

    # Initialize
    IMinimalVault(new_vault).init(msg.sender, name)

    # Record
    self.vault_index[new_vault] = convert(len(self.deployed_vaults), uint256) + 1
    self.deployed_vaults.append(DeployedVault({
        addr: new_vault,
        name: name,
        owner_addr: msg.sender,
        deployed_at: block.timestamp
    }))

    log VaultDeployed(new_vault, msg.sender, name)
    return new_vault

@view
@external
def get_vault_info(vault: address) -> DeployedVault:
    idx: uint256 = self.vault_index[vault]
    assert idx > 0, "Vault not found"
    return self.deployed_vaults[idx - 1]

@view
@external
def vault_count() -> uint256:
    return convert(len(self.deployed_vaults), uint256)
```

---

## 6. ตัวอย่าง: Configurable Token Contract {#example}

Token ที่ configure ได้หลายอย่างตอน deploy

```python
# @version 0.4.0

# ════════════════════════════════════════════════════════════
# ConfigurableToken Contract
# ERC-20 Token ที่ configure behavior ตอน deploy ได้
# ════════════════════════════════════════════════════════════

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Immutables (set at deploy, never change)
# ━━━━━━━━━━━━━━━━━━━━━━━━━
NAME: immutable(String[50])
SYMBOL: immutable(String[10])
DECIMALS: immutable(uint8)
MAX_SUPPLY: immutable(uint256)
DEPLOY_TIME: immutable(uint256)
DEPLOY_BLOCK: immutable(uint256)

# Feature flags (set at deploy)
IS_MINTABLE: immutable(bool)
IS_BURNABLE: immutable(bool)
IS_PAUSABLE: immutable(bool)
HAS_TRANSFER_FEE: immutable(bool)

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Configurable Constants (set at deploy)
# ━━━━━━━━━━━━━━━━━━━━━━━━━
TRANSFER_FEE_BPS: immutable(uint256)  # 0 if no fee
FEE_RECIPIENT: immutable(address)

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# State Variables
# ━━━━━━━━━━━━━━━━━━━━━━━━━
owner: address
total_supply: uint256
balances: HashMap[address, uint256]
allowances: HashMap[address, HashMap[address, uint256]]
paused: bool

# Minter role (if mintable)
minters: HashMap[address, bool]

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Events
# ━━━━━━━━━━━━━━━━━━━━━━━━━
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
    from_addr: indexed(address)
    amount: uint256

event Paused:
    by: indexed(address)

event Unpaused:
    by: indexed(address)

event MinterAdded:
    minter: indexed(address)

event FeeTaken:
    from_addr: indexed(address)
    fee_amount: uint256

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Constructor
# ━━━━━━━━━━━━━━━━━━━━━━━━━

@deploy
def __init__(
    name: String[50],
    symbol: String[10],
    decimals: uint8,
    initial_supply: uint256,
    max_supply: uint256,
    is_mintable: bool,
    is_burnable: bool,
    is_pausable: bool,
    has_transfer_fee: bool,
    transfer_fee_bps: uint256,
    fee_recipient: address
):
    # Input validation
    assert len(name) > 0, "Name required"
    assert len(symbol) > 0, "Symbol required"
    assert decimals <= 18, "Max 18 decimals"
    assert initial_supply <= max_supply, "Initial > max supply"

    # Fee validation
    if has_transfer_fee:
        assert transfer_fee_bps > 0, "Fee bps required"
        assert transfer_fee_bps <= 1000, "Max 10% fee"
        assert fee_recipient != empty(address), "Fee recipient required"

    # Set immutables
    NAME = name
    SYMBOL = symbol
    DECIMALS = decimals
    MAX_SUPPLY = max_supply
    DEPLOY_TIME = block.timestamp
    DEPLOY_BLOCK = block.number
    IS_MINTABLE = is_mintable
    IS_BURNABLE = is_burnable
    IS_PAUSABLE = is_pausable
    HAS_TRANSFER_FEE = has_transfer_fee
    TRANSFER_FEE_BPS = transfer_fee_bps
    FEE_RECIPIENT = fee_recipient

    # Set state
    self.owner = msg.sender
    self.total_supply = initial_supply
    self.balances[msg.sender] = initial_supply

    if is_mintable:
        self.minters[msg.sender] = True

    log Transfer(empty(address), msg.sender, initial_supply)

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Internal Helpers
# ━━━━━━━━━━━━━━━━━━━━━━━━━

@internal
def _require_owner():
    assert msg.sender == self.owner, "Not owner"

@internal
def _require_not_paused():
    if IS_PAUSABLE:
        assert not self.paused, "Paused"

@view
@internal
def _calculate_fee(amount: uint256) -> uint256:
    if not HAS_TRANSFER_FEE:
        return 0
    return (amount * TRANSFER_FEE_BPS) / 10000

@internal
def _transfer(from_addr: address, to: address, amount: uint256):
    self._require_not_paused()
    assert from_addr != empty(address), "From zero"
    assert to != empty(address), "To zero"
    assert amount > 0, "Zero amount"

    fee: uint256 = self._calculate_fee(amount)
    net_amount: uint256 = amount - fee

    assert self.balances[from_addr] >= amount, "Insufficient balance"

    self.balances[from_addr] -= amount
    self.balances[to] += net_amount

    if fee > 0:
        self.balances[FEE_RECIPIENT] += fee
        log FeeTaken(from_addr, fee)
        log Transfer(from_addr, FEE_RECIPIENT, fee)

    log Transfer(from_addr, to, net_amount)

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# External Functions
# ━━━━━━━━━━━━━━━━━━━━━━━━━

@external
def transfer(to: address, amount: uint256) -> bool:
    self._transfer(msg.sender, to, amount)
    return True

@external
def transfer_from(sender: address, recipient: address, amount: uint256) -> bool:
    allowed: uint256 = self.allowances[sender][msg.sender]
    assert allowed >= amount, "Insufficient allowance"
    if allowed != max_value(uint256):
        self.allowances[sender][msg.sender] -= amount
    self._transfer(sender, recipient, amount)
    return True

@external
def approve(spender: address, amount: uint256) -> bool:
    assert spender != empty(address), "Zero spender"
    self.allowances[msg.sender][spender] = amount
    log Approval(msg.sender, spender, amount)
    return True

@external
def mint(to: address, amount: uint256):
    assert IS_MINTABLE, "Not mintable"
    assert self.minters[msg.sender], "Not minter"
    assert to != empty(address), "Invalid address"
    assert self.total_supply + amount <= MAX_SUPPLY, "Exceeds max"

    self.total_supply += amount
    self.balances[to] += amount
    log Transfer(empty(address), to, amount)
    log Mint(to, amount)

@external
def burn(amount: uint256):
    assert IS_BURNABLE, "Not burnable"
    assert self.balances[msg.sender] >= amount, "Insufficient"

    self.balances[msg.sender] -= amount
    self.total_supply -= amount
    log Transfer(msg.sender, empty(address), amount)
    log Burn(msg.sender, amount)

@external
def pause():
    assert IS_PAUSABLE, "Not pausable"
    self._require_owner()
    assert not self.paused, "Already paused"
    self.paused = True
    log Paused(msg.sender)

@external
def unpause():
    assert IS_PAUSABLE, "Not pausable"
    self._require_owner()
    assert self.paused, "Not paused"
    self.paused = False
    log Unpaused(msg.sender)

@external
def add_minter(new_minter: address):
    assert IS_MINTABLE, "Not mintable"
    self._require_owner()
    assert new_minter != empty(address), "Invalid"
    self.minters[new_minter] = True
    log MinterAdded(new_minter)

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# View Functions
# ━━━━━━━━━━━━━━━━━━━━━━━━━

@view
@external
def name() -> String[50]:
    return NAME

@view
@external
def symbol() -> String[10]:
    return SYMBOL

@view
@external
def decimals() -> uint8:
    return DECIMALS

@view
@external
def total_supply_view() -> uint256:
    return self.total_supply

@view
@external
def max_supply() -> uint256:
    return MAX_SUPPLY

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
def is_mintable() -> bool:
    return IS_MINTABLE

@view
@external
def is_burnable() -> bool:
    return IS_BURNABLE

@view
@external
def is_pausable() -> bool:
    return IS_PAUSABLE

@view
@external
def has_transfer_fee() -> bool:
    return HAS_TRANSFER_FEE

@view
@external
def transfer_fee_bps() -> uint256:
    return TRANSFER_FEE_BPS

@view
@external
def fee_recipient() -> address:
    return FEE_RECIPIENT

@view
@external
def get_token_config() -> (String[50], String[10], uint8, uint256, bool, bool, bool):
    return (NAME, SYMBOL, DECIMALS, MAX_SUPPLY, IS_MINTABLE, IS_BURNABLE, IS_PAUSABLE)

@view
@external
def is_paused() -> bool:
    if IS_PAUSABLE:
        return self.paused
    return False

@view
@external
def is_minter(account: address) -> bool:
    return self.minters[account]

@view
@external
def deploy_time() -> uint256:
    return DEPLOY_TIME

@view
@external
def deploy_block() -> uint256:
    return DEPLOY_BLOCK
```

### Test Code

```python
# tests/test_configurable_token.py
import pytest
from brownie import ConfigurableToken, accounts, reverts

def deploy_token(
    name="TestToken",
    symbol="TST",
    decimals=18,
    initial=1_000_000 * 10**18,
    max_sup=10_000_000 * 10**18,
    mintable=True,
    burnable=True,
    pausable=True,
    has_fee=False,
    fee_bps=0,
    fee_recipient=None
):
    if fee_recipient is None:
        fee_recipient = accounts[9]
    return ConfigurableToken.deploy(
        name, symbol, decimals, initial, max_sup,
        mintable, burnable, pausable, has_fee, fee_bps, fee_recipient,
        {'from': accounts[0]}
    )

class TestImmutables:
    def test_name_immutable(self):
        token = deploy_token(name="Immutable")
        assert token.name() == "Immutable"

    def test_symbol_immutable(self):
        token = deploy_token(symbol="IMM")
        assert token.symbol() == "IMM"

    def test_max_supply_immutable(self):
        max_sup = 5_000_000 * 10**18
        token = deploy_token(max_sup=max_sup)
        assert token.max_supply() == max_sup

    def test_feature_flags(self):
        token = deploy_token(mintable=True, burnable=False, pausable=True)
        assert token.is_mintable() == True
        assert token.is_burnable() == False
        assert token.is_pausable() == True

class TestMint:
    def test_mint(self):
        token = deploy_token()
        initial = token.balance_of(accounts[0])
        token.mint(accounts[1], 1000, {'from': accounts[0]})
        assert token.balance_of(accounts[1]) == 1000

    def test_mint_exceeds_max(self):
        token = deploy_token(max_sup=1_000_000 * 10**18)
        with reverts("Exceeds max"):
            token.mint(accounts[1], 1, {'from': accounts[0]})

    def test_non_mintable_reverts(self):
        token = deploy_token(mintable=False)
        with reverts("Not mintable"):
            token.mint(accounts[1], 1000, {'from': accounts[0]})

class TestFee:
    def test_transfer_fee(self):
        token = deploy_token(has_fee=True, fee_bps=300)  # 3% fee
        token.transfer(accounts[1], 10000, {'from': accounts[0]})
        assert token.balance_of(accounts[1]) == 9700
        assert token.balance_of(accounts[9]) == 300  # fee_recipient

class TestPause:
    def test_pause_blocks_transfer(self):
        token = deploy_token()
        token.pause({'from': accounts[0]})
        with reverts("Paused"):
            token.transfer(accounts[1], 100, {'from': accounts[0]})

    def test_unpause_allows_transfer(self):
        token = deploy_token()
        token.pause({'from': accounts[0]})
        token.unpause({'from': accounts[0]})
        token.transfer(accounts[1], 100, {'from': accounts[0]})
        assert token.balance_of(accounts[1]) == 100

class TestConstructorValidation:
    def test_initial_gt_max_reverts(self):
        with reverts("Initial > max supply"):
            ConfigurableToken.deploy(
                "T", "T", 18,
                10**20,  # initial > max
                10**19,  # max
                False, False, False, False, 0, accounts[9],
                {'from': accounts[0]}
            )
```

---

## สรุป

- ✅ **@deploy def __init__**: Constructor ใน Vyper 0.4.0
- ✅ **Constructor parameters**: รับ primitives, structs, arrays ได้
- ✅ **Immutables**: Set ครั้งเดียวใน constructor, ถูกกว่า storage
- ✅ **Constant vs Immutable**: Constant รู้ตอน compile, Immutable รู้ตอน deploy
- ✅ **Two-phase init**: Deploy ก่อน แล้ว initialize ทีหลัง
- ✅ **Factory pattern**: Contract สร้าง Contract อื่น

## แบบฝึกหัด

1. **สร้าง** DAO Token ที่มี governance features configure ได้ตอน deploy
2. **เพิ่ม** Whitelist-only mode ใน ConfigurableToken
3. **สร้าง** Token Factory ที่ deploy ConfigurableToken หลายตัว
4. **ทดสอบ** Gas savings ของ immutable vs storage variable
5. **สร้าง** Vesting Contract ที่มี cliff/duration/amount เป็น immutables

---

**ก่อนหน้า: [Part 012 - Events และ Logging](part_012_events.md)**  
**ต่อไป: [Part 014 - Modifiers และ @view/@pure](part_014_decorators.md)**
