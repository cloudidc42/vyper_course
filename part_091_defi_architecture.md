# Part 091: Real-world DeFi Protocol Architecture

## สารบัญ
1. [Protocol Architecture Overview](#overview)
2. [Core Components](#core)
3. [Modular Design](#modular)
4. [Upgrade Patterns](#upgrades)
5. [Integration Patterns](#integration)
6. [Data Flow Design](#dataflow)
7. [Testing Architecture](#testing-arch)
8. [Real Protocol: Curve-Like AMM](#curve-like)
9. [Protocol Governance Architecture](#governance)
10. [Production Deployment Architecture](#production)

---

## 1. Protocol Architecture Overview {#overview}

### สถาปัตยกรรม DeFi Protocol ระดับ Production

```
┌─────────────────────────────────────────────────────────┐
│                     FRONTEND LAYER                       │
│              React / Next.js / Web3 Integration          │
└─────────────────────────┬───────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────┐
│                    MIDDLEWARE LAYER                       │
│        Subgraph (The Graph) / Indexers / APIs            │
└─────────────────────────┬───────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────┐
│                  SMART CONTRACT LAYER                     │
│                                                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐ │
│  │   Router    │  │  Registry   │  │   Governance    │ │
│  └──────┬──────┘  └──────┬──────┘  └────────┬────────┘ │
│         │                │                   │          │
│  ┌──────▼──────────────────────────────────▼────────┐  │
│  │              CORE PROTOCOL CONTRACTS              │  │
│  │  Pool A │ Pool B │ Vault │ Oracle │ Factory       │  │
│  └──────────────────────────────────────────────────┘  │
│                          │                              │
│  ┌──────────────────────▼──────────────────────────┐   │
│  │           INFRASTRUCTURE CONTRACTS               │   │
│  │  Access Control │ Pause Guardian │ Timelock      │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────┐
│               EXTERNAL DEPENDENCIES                      │
│     Chainlink Oracle │ Uniswap │ Aave │ Compound         │
└─────────────────────────────────────────────────────────┘
```

---

## 2. Core Components {#core}

### 2.1 Registry Contract

```python
# @version 0.4.0
# SPDX-License-Identifier: MIT
# ไฟล์: contracts/core/Registry.vy

"""
Protocol Registry
- Central source of truth สำหรับ Contract Addresses
- Governance controls updates
- Version tracking
"""

# ════════════════════════
# EVENTS
# ════════════════════════
event ContractRegistered:
    name: indexed(String[32])
    addr: indexed(address)
    version: uint256

event ContractUpdated:
    name: indexed(String[32])
    old_addr: indexed(address)
    new_addr: indexed(address)

# ════════════════════════
# STRUCTS
# ════════════════════════
struct ContractInfo:
    addr: address
    version: uint256
    registered_at: uint256
    is_active: bool

# ════════════════════════
# STATE
# ════════════════════════

# Key → Contract Info
contracts: HashMap[String[32], ContractInfo]
contract_names: DynArray[String[32], 100]

# Address → Name lookup
address_to_name: HashMap[address, String[32]]

# Governance
owner: address
governance: address

# History
update_history: HashMap[String[32], DynArray[address, 10]]

# ════════════════════════
# CONSTRUCTOR
# ════════════════════════

@deploy
def __init__(governance_addr: address):
    self.owner = msg.sender
    self.governance = governance_addr

# ════════════════════════
# REGISTRATION
# ════════════════════════

@external
def register(name: String[32], addr: address):
    """Register a new contract"""
    assert msg.sender == self.governance, "Not governance"
    assert addr != empty(address), "Zero address"
    assert self.contracts[name].addr == empty(address), "Already registered"
    
    self.contracts[name] = ContractInfo({
        addr: addr,
        version: 1,
        registered_at: block.timestamp,
        is_active: True
    })
    
    self.contract_names.append(name)
    self.address_to_name[addr] = name
    
    log ContractRegistered(name, addr, 1)

@external
def update(name: String[32], new_addr: address):
    """Update contract address"""
    assert msg.sender == self.governance, "Not governance"
    assert new_addr != empty(address), "Zero address"
    
    old_info: ContractInfo = self.contracts[name]
    assert old_info.addr != empty(address), "Not registered"
    
    old_addr: address = old_info.addr
    new_version: uint256 = old_info.version + 1
    
    self.contracts[name] = ContractInfo({
        addr: new_addr,
        version: new_version,
        registered_at: block.timestamp,
        is_active: True
    })
    
    # Update reverse lookup
    self.address_to_name[old_addr] = ""
    self.address_to_name[new_addr] = name
    
    # Store history
    history: DynArray[address, 10] = self.update_history[name]
    if len(history) < 10:
        self.update_history[name].append(old_addr)
    
    log ContractUpdated(name, old_addr, new_addr)

# ════════════════════════
# VIEW FUNCTIONS
# ════════════════════════

@view
@external
def get_address(name: String[32]) -> address:
    info: ContractInfo = self.contracts[name]
    assert info.is_active, "Contract inactive"
    return info.addr

@view
@external
def get_info(name: String[32]) -> ContractInfo:
    return self.contracts[name]

@view
@external
def get_all_contracts() -> DynArray[String[32], 100]:
    return self.contract_names

@view
@external
def is_registered(addr: address) -> bool:
    name: String[32] = self.address_to_name[addr]
    if len(name) == 0:
        return False
    return self.contracts[name].addr == addr

@view
@external
def get_name(addr: address) -> String[32]:
    return self.address_to_name[addr]
```

### 2.2 Factory Contract

```python
# @version 0.4.0
# SPDX-License-Identifier: MIT
# ไฟล์: contracts/core/PoolFactory.vy

"""
Pool Factory
สร้าง Pool Contracts แบบ Minimal Clone (EIP-1167)
"""

interface IPool:
    def initialize(
        token_a: address,
        token_b: address,
        fee: uint256,
        admin: address
    ): nonpayable

# ════════════════════════
# EVENTS
# ════════════════════════

event PoolCreated:
    token_a: indexed(address)
    token_b: indexed(address)
    fee: indexed(uint256)
    pool: address
    pool_count: uint256

# ════════════════════════
# STATE
# ════════════════════════

pool_implementation: address
registry: address
owner: address

# Tracking
all_pools: DynArray[address, 10000]
pool_by_tokens: HashMap[address, HashMap[address, HashMap[uint256, address]]]

# Fee tiers
valid_fees: DynArray[uint256, 5]

# ════════════════════════
# CONSTRUCTOR
# ════════════════════════

@deploy
def __init__(impl: address, registry_addr: address):
    self.pool_implementation = impl
    self.registry = registry_addr
    self.owner = msg.sender
    
    # Valid fee tiers: 0.01%, 0.05%, 0.3%, 1%
    self.valid_fees.append(10)    # 0.01%
    self.valid_fees.append(50)    # 0.05%
    self.valid_fees.append(300)   # 0.3%
    self.valid_fees.append(1000)  # 1%
    self.valid_fees.append(3000)  # 3%

# ════════════════════════
# FACTORY FUNCTIONS
# ════════════════════════

@external
def create_pool(
    token_a: address,
    token_b: address,
    fee: uint256
) -> address:
    """
    Create a new liquidity pool
    """
    assert token_a != token_b, "Same tokens"
    assert token_a != empty(address), "Zero token_a"
    assert token_b != empty(address), "Zero token_b"
    assert self._is_valid_fee(fee), "Invalid fee"
    
    # Sort tokens for canonical pair
    token0: address = token_a
    token1: address = token_b
    
    if convert(token_a, uint160) > convert(token_b, uint160):
        token0 = token_b
        token1 = token_a
    
    # Check if pool already exists
    assert (
        self.pool_by_tokens[token0][token1][fee] == empty(address)
    ), "Pool exists"
    
    # Deploy minimal clone
    pool: address = self._deploy_clone(self.pool_implementation)
    
    # Initialize pool
    IPool(pool).initialize(token0, token1, fee, self.owner)
    
    # Register pool
    self.all_pools.append(pool)
    self.pool_by_tokens[token0][token1][fee] = pool
    
    log PoolCreated(token0, token1, fee, pool, len(self.all_pools))
    
    return pool

@internal
def _deploy_clone(implementation: address) -> address:
    """Deploy EIP-1167 Minimal Proxy Clone"""
    # EIP-1167 bytecode prefix: 0x3d602d80600a3d3981f3363d3d373d3d3d363d73
    # implementation address (20 bytes)
    # suffix: 0x5af43d82803e903d91602b57fd5bf3
    
    impl_bytes: bytes20 = convert(implementation, bytes20)
    
    # Create minimal proxy bytecode
    # This is simplified - real implementation uses raw bytecode
    
    # Return address (simplified for demonstration)
    return implementation  # Placeholder

@view
@internal
def _is_valid_fee(fee: uint256) -> bool:
    for valid_fee: uint256 in self.valid_fees:
        if fee == valid_fee:
            return True
    return False

# ════════════════════════
# VIEW FUNCTIONS
# ════════════════════════

@view
@external
def get_pool(
    token_a: address,
    token_b: address,
    fee: uint256
) -> address:
    """Get pool address for token pair"""
    token0: address = token_a
    token1: address = token_b
    
    if convert(token_a, uint160) > convert(token_b, uint160):
        token0 = token_b
        token1 = token_a
    
    return self.pool_by_tokens[token0][token1][fee]

@view
@external
def get_all_pools() -> DynArray[address, 10000]:
    return self.all_pools

@view
@external
def pool_count() -> uint256:
    return len(self.all_pools)
```

### 2.3 Router Contract

```python
# @version 0.4.0
# SPDX-License-Identifier: MIT
# ไฟล์: contracts/core/Router.vy

"""
Swap Router
- จัดการ Multi-hop Swaps
- Slippage Protection
- Deadline Check
"""

interface IPool:
    def swap(
        amount0_out: uint256,
        amount1_out: uint256,
        to: address
    ): nonpayable
    
    def get_reserves() -> (uint256, uint256): view
    def token0() -> address: view
    def token1() -> address: view

interface IERC20:
    def transferFrom(sender: address, receiver: address, amount: uint256) -> bool: nonpayable
    def transfer(receiver: address, amount: uint256) -> bool: nonpayable
    def balanceOf(account: address) -> uint256: view
    def approve(spender: address, amount: uint256) -> bool: nonpayable

interface IFactory:
    def get_pool(token_a: address, token_b: address, fee: uint256) -> address: view

# ════════════════════════
# STRUCTS
# ════════════════════════

struct SwapParams:
    token_in: address
    token_out: address
    fee: uint256
    recipient: address
    deadline: uint256
    amount_in: uint256
    amount_out_minimum: uint256

struct ExactInputParams:
    path: DynArray[address, 10]   # [tokenA, tokenB, tokenC, ...]
    fees: DynArray[uint256, 9]    # fee for each hop
    recipient: address
    deadline: uint256
    amount_in: uint256
    amount_out_minimum: uint256

# ════════════════════════
# STATE
# ════════════════════════

factory: address
WETH: address

# ════════════════════════
# CONSTRUCTOR
# ════════════════════════

@deploy
def __init__(factory_addr: address, weth_addr: address):
    self.factory = factory_addr
    self.WETH = weth_addr

# ════════════════════════
# SINGLE HOP SWAP
# ════════════════════════

@external
def exact_input_single(params: SwapParams) -> uint256:
    """
    Single-hop exact input swap
    """
    assert block.timestamp <= params.deadline, "Expired"
    assert params.amount_in > 0, "Zero input"
    
    # Get pool
    pool: address = IFactory(self.factory).get_pool(
        params.token_in,
        params.token_out,
        params.fee
    )
    assert pool != empty(address), "Pool not found"
    
    # Transfer tokens to pool
    IERC20(params.token_in).transferFrom(
        msg.sender,
        pool,
        params.amount_in
    )
    
    # Calculate output
    amount_out: uint256 = self._get_amount_out(
        params.amount_in,
        pool,
        params.token_in
    )
    
    assert amount_out >= params.amount_out_minimum, "Slippage protection"
    
    # Determine swap direction
    token0: address = IPool(pool).token0()
    amount0_out: uint256 = 0
    amount1_out: uint256 = 0
    
    if params.token_in == token0:
        amount1_out = amount_out
    else:
        amount0_out = amount_out
    
    # Execute swap
    IPool(pool).swap(amount0_out, amount1_out, params.recipient)
    
    return amount_out

@internal
def _get_amount_out(
    amount_in: uint256,
    pool: address,
    token_in: address
) -> uint256:
    """คำนวณ Output Amount"""
    r0: uint256 = 0
    r1: uint256 = 0
    (r0, r1) = IPool(pool).get_reserves()
    
    token0: address = IPool(pool).token0()
    
    reserve_in: uint256 = 0
    reserve_out: uint256 = 0
    
    if token_in == token0:
        reserve_in = r0
        reserve_out = r1
    else:
        reserve_in = r1
        reserve_out = r0
    
    # Apply 0.3% fee
    amount_in_with_fee: uint256 = amount_in * 9970
    numerator: uint256 = amount_in_with_fee * reserve_out
    denominator: uint256 = reserve_in * 10000 + amount_in_with_fee
    
    return numerator / denominator

# ════════════════════════
# MULTI-HOP SWAP
# ════════════════════════

@external
def exact_input(params: ExactInputParams) -> uint256:
    """
    Multi-hop exact input swap
    path: [tokenA, tokenB, tokenC]
    fees: [fee_AB, fee_BC]
    """
    assert block.timestamp <= params.deadline, "Expired"
    assert len(params.path) >= 2, "Invalid path"
    assert len(params.fees) == len(params.path) - 1, "Fee mismatch"
    
    current_amount: uint256 = params.amount_in
    current_token: address = params.path[0]
    
    # Transfer initial tokens from user
    IERC20(current_token).transferFrom(
        msg.sender,
        self,
        params.amount_in
    )
    
    # Swap through each hop
    for i: uint256 in range(9):
        if i >= len(params.fees):
            break
        
        next_token: address = params.path[i + 1]
        fee: uint256 = params.fees[i]
        
        # Get pool
        pool: address = IFactory(self.factory).get_pool(
            current_token,
            next_token,
            fee
        )
        assert pool != empty(address), "Pool not found in path"
        
        # Transfer to pool
        IERC20(current_token).approve(pool, current_amount)
        IERC20(current_token).transfer(pool, current_amount)
        
        # Calculate next amount
        next_amount: uint256 = self._get_amount_out(
            current_amount,
            pool,
            current_token
        )
        
        # Determine output direction
        token0: address = IPool(pool).token0()
        amount0_out: uint256 = 0
        amount1_out: uint256 = 0
        
        if current_token == token0:
            amount1_out = next_amount
        else:
            amount0_out = next_amount
        
        # Last hop → send to recipient
        # Otherwise → send to this contract
        is_last: bool = (i == len(params.fees) - 1)
        swap_recipient: address = params.recipient if is_last else self
        
        IPool(pool).swap(amount0_out, amount1_out, swap_recipient)
        
        current_amount = next_amount
        current_token = next_token
    
    assert current_amount >= params.amount_out_minimum, "Slippage protection"
    
    return current_amount

# ════════════════════════
# VIEW FUNCTIONS
# ════════════════════════

@view
@external
def quote_exact_input(
    path: DynArray[address, 10],
    fees: DynArray[uint256, 9],
    amount_in: uint256
) -> uint256:
    """Quote สำหรับ Multi-hop Swap"""
    assert len(path) >= 2, "Invalid path"
    
    current_amount: uint256 = amount_in
    
    for i: uint256 in range(9):
        if i >= len(fees):
            break
        
        pool: address = IFactory(self.factory).get_pool(
            path[i],
            path[i + 1],
            fees[i]
        )
        
        if pool == empty(address):
            return 0
        
        current_amount = self._get_amount_out(
            current_amount,
            pool,
            path[i]
        )
    
    return current_amount
```

---

## 3. Modular Design {#modular}

### 3.1 Vyper Modules (0.4.0+)

```python
# @version 0.4.0
# SPDX-License-Identifier: MIT

# ════════════════════════
# MODULE: Ownable
# ════════════════════════
# ไฟล์: modules/Ownable.vy

# Vyper 0.4.0 มี Module system
# (เป็น Feature ใหม่, syntax อาจแตกต่างเล็กน้อย)

owner: address
pending_owner: address

event OwnershipTransferred:
    previous_owner: indexed(address)
    new_owner: indexed(address)

@deploy
def __init__():
    self.owner = msg.sender

@internal
def _check_owner():
    assert msg.sender == self.owner, "Ownable: caller is not the owner"

@external
def transfer_ownership(new_owner: address):
    self._check_owner()
    assert new_owner != empty(address), "Ownable: new owner is the zero address"
    self.pending_owner = new_owner

@external
def accept_ownership():
    assert msg.sender == self.pending_owner, "Ownable: caller is not the pending owner"
    old_owner: address = self.owner
    self.owner = self.pending_owner
    self.pending_owner = empty(address)
    log OwnershipTransferred(old_owner, self.owner)

@external
def renounce_ownership():
    self._check_owner()
    log OwnershipTransferred(self.owner, empty(address))
    self.owner = empty(address)
```

```python
# @version 0.4.0
# SPDX-License-Identifier: MIT
# ไฟล์: contracts/MyProtocol.vy

# Import modules (Vyper 0.4.0)
# from . import Ownable

"""
Protocol Contract ที่ใช้ Modular Design
"""

# Module State Variables
protocol_fee: uint256
treasury: address
pools: DynArray[address, 1000]

# ════════════════════════
# PROTOCOL LOGIC
# ════════════════════════

@external
def set_protocol_fee(fee: uint256):
    """เฉพาะ Owner เท่านั้น"""
    # self._check_owner()  # จาก Module
    assert fee <= 1000, "Fee too high"
    self.protocol_fee = fee

@external
def set_treasury(new_treasury: address):
    # self._check_owner()
    assert new_treasury != empty(address), "Zero address"
    self.treasury = new_treasury

@external
def add_pool(pool: address):
    # self._check_owner()
    assert pool != empty(address), "Zero address"
    self.pools.append(pool)

@view
@external
def get_pools() -> DynArray[address, 1000]:
    return self.pools
```

---

## 4. Real Protocol: Curve-Like Stable AMM {#curve-like}

```python
# @version 0.4.0
# SPDX-License-Identifier: MIT
# ไฟล์: contracts/StableSwap.vy

"""
StableSwap AMM (Curve-like)
Optimized for stablecoins and similar-priced assets

Key Differences from Uniswap:
- Uses amplified invariant: A*n*sum + D = A*D*n + D^(n+1)/(n^n * prod)
- Much lower slippage for same-priced assets
- Higher capital efficiency for stable pairs
"""

from vyper.interfaces import ERC20

# ════════════════════════
# CONSTANTS  
# ════════════════════════

N_COINS: constant(uint256) = 2           # จำนวน Coins ใน Pool
A_PRECISION: constant(uint256) = 100     # Amplification precision
PRECISION: constant(uint256) = 10**18
FEE_DENOMINATOR: constant(uint256) = 10**10

# ════════════════════════
# STATE
# ════════════════════════

coins: address[N_COINS]          # Pool tokens
balances: uint256[N_COINS]       # Virtual balances

initial_A: uint256               # Starting amplification parameter
future_A: uint256                # Target amplification
initial_A_time: uint256          # Start time of A change
future_A_time: uint256           # End time of A change

fee: uint256                     # Swap fee (in FEE_DENOMINATOR units)
admin_fee: uint256               # Admin fee portion

lp_token: address                # LP token address
owner: address

# ════════════════════════
# EVENTS
# ════════════════════════

event TokenExchange:
    buyer: indexed(address)
    sold_id: int128
    tokens_sold: uint256
    bought_id: int128
    tokens_bought: uint256

event AddLiquidity:
    provider: indexed(address)
    token_amounts: uint256[N_COINS]
    fees: uint256[N_COINS]
    invariant: uint256
    token_supply: uint256

event RemoveLiquidity:
    provider: indexed(address)
    token_amounts: uint256[N_COINS]
    fees: uint256[N_COINS]
    token_supply: uint256

event RampA:
    old_A: uint256
    new_A: uint256
    initial_time: uint256
    future_time: uint256

# ════════════════════════
# CONSTRUCTOR
# ════════════════════════

@deploy
def __init__(
    coin0: address,
    coin1: address,
    amplification: uint256,
    swap_fee: uint256,
    admin_fee_rate: uint256,
    lp_token_addr: address
):
    self.coins[0] = coin0
    self.coins[1] = coin1
    
    # Set initial amplification parameter
    # A = 100 means 100x more efficient than Uniswap for stable pairs
    self.initial_A = amplification * A_PRECISION
    self.future_A = amplification * A_PRECISION
    
    self.fee = swap_fee        # e.g., 4000000 = 0.04%
    self.admin_fee = admin_fee_rate
    self.lp_token = lp_token_addr
    self.owner = msg.sender

# ════════════════════════
# INVARIANT CALCULATION
# ════════════════════════

@view
@internal
def _A() -> uint256:
    """Get current amplification parameter (handles ramp)"""
    t1: uint256 = self.future_A_time
    A1: uint256 = self.future_A
    
    if block.timestamp < t1:
        A0: uint256 = self.initial_A
        t0: uint256 = self.initial_A_time
        
        # Linear interpolation
        # A = A0 + (A1 - A0) * (t - t0) / (t1 - t0)
        if A1 > A0:
            return A0 + (A1 - A0) * (block.timestamp - t0) / (t1 - t0)
        else:
            return A0 - (A0 - A1) * (block.timestamp - t0) / (t1 - t0)
    
    return A1

@view
@internal
def _get_D(xp: uint256[N_COINS], amp: uint256) -> uint256:
    """
    Calculate D invariant using Newton's method
    
    StableSwap invariant:
    A*n^n*sum(x_i) + D = A*D*n^n + D^(n+1) / (n^n * prod(x_i))
    
    Where:
    - A = amplification parameter
    - n = number of coins
    - x_i = balances
    - D = invariant
    """
    
    S: uint256 = xp[0] + xp[1]  # Sum of all balances
    
    if S == 0:
        return 0
    
    Dprev: uint256 = 0
    D: uint256 = S
    Ann: uint256 = amp * N_COINS  # A*n
    
    for _i: uint256 in range(255):
        D_P: uint256 = D
        
        # Calculate D_P = D^(n+1) / (n^n * prod(x_i))
        for x: uint256 in xp:
            D_P = D_P * D / (x * N_COINS)
        
        Dprev = D
        
        # Newton's iteration
        # D = (Ann*S + D_P*N_COINS) * D / ((Ann-1)*D + (N_COINS+1)*D_P)
        D = (
            (Ann * S / A_PRECISION + D_P * N_COINS) * D /
            ((Ann - A_PRECISION) * D / A_PRECISION + (N_COINS + 1) * D_P)
        )
        
        # Convergence check
        if D > Dprev:
            if D - Dprev <= 1:
                break
        else:
            if Dprev - D <= 1:
                break
    
    return D

@view
@internal
def _get_y(
    i: int128,
    j: int128,
    x: uint256,
    xp_: uint256[N_COINS]
) -> uint256:
    """
    Calculate y: balance of coin j when coin i has balance x
    
    Done by solving the invariant for y using Newton's method
    """
    amp: uint256 = self._A()
    D: uint256 = self._get_D(xp_, amp)
    
    Ann: uint256 = amp * N_COINS
    c: uint256 = D
    S_: uint256 = 0
    
    _x: uint256 = 0
    y_prev: uint256 = 0
    
    for _i: int128 in range(N_COINS):
        if _i == i:
            _x = x
        elif _i != j:
            _x = xp_[convert(_i, uint256)]
        else:
            continue
        
        S_ += _x
        c = c * D / (_x * N_COINS)
    
    c = c * D * A_PRECISION / (Ann * N_COINS)
    b: uint256 = S_ + D * A_PRECISION / Ann
    
    y: uint256 = D
    
    for _i: uint256 in range(255):
        y_prev = y
        y = (y * y + c) / (2 * y + b - D)
        
        if y > y_prev:
            if y - y_prev <= 1:
                break
        else:
            if y_prev - y <= 1:
                break
    
    return y

# ════════════════════════
# SWAP
# ════════════════════════

@external
def exchange(
    i: int128,
    j: int128,
    dx: uint256,
    min_dy: uint256
) -> uint256:
    """
    Exchange coin i for coin j
    i: index of input coin
    j: index of output coin
    dx: amount of input coin
    min_dy: minimum output (slippage protection)
    """
    assert i != j, "Same coin"
    assert i >= 0 and i < convert(N_COINS, int128), "Invalid i"
    assert j >= 0 and j < convert(N_COINS, int128), "Invalid j"
    
    rates: uint256[N_COINS] = [PRECISION, PRECISION]
    
    old_balances: uint256[N_COINS] = self.balances
    xp: uint256[N_COINS] = self._xp_mem(old_balances, rates)
    
    # Current balance of coin i adjusted for precision
    x: uint256 = xp[convert(i, uint256)] + dx * rates[convert(i, uint256)] / PRECISION
    
    # Calculate new balance of coin j
    y: uint256 = self._get_y(i, j, x, xp)
    
    # Amount out = old balance - new balance
    dy: uint256 = xp[convert(j, uint256)] - y - 1
    
    # Apply fee
    dy_fee: uint256 = dy * self.fee / FEE_DENOMINATOR
    dy = (dy - dy_fee) * PRECISION / rates[convert(j, uint256)]
    
    assert dy >= min_dy, "Slippage protection"
    
    # Admin fee portion
    dy_admin_fee: uint256 = dy_fee * self.admin_fee / FEE_DENOMINATOR
    dy_admin_fee = dy_admin_fee * PRECISION / rates[convert(j, uint256)]
    
    # Update balances
    self.balances[convert(i, uint256)] = old_balances[convert(i, uint256)] + dx
    self.balances[convert(j, uint256)] = (
        old_balances[convert(j, uint256)] - dy - dy_admin_fee
    )
    
    # Transfer tokens
    ERC20(self.coins[convert(i, uint256)]).transferFrom(msg.sender, self, dx)
    ERC20(self.coins[convert(j, uint256)]).transfer(msg.sender, dy)
    
    log TokenExchange(msg.sender, i, dx, j, dy)
    
    return dy

@view
@internal
def _xp_mem(
    _balances: uint256[N_COINS],
    _rates: uint256[N_COINS]
) -> uint256[N_COINS]:
    """แปลง balances ด้วย precision rates"""
    result: uint256[N_COINS] = _balances
    for i: uint256 in range(N_COINS):
        result[i] = _balances[i] * _rates[i] / PRECISION
    return result

# ════════════════════════
# VIEW FUNCTIONS
# ════════════════════════

@view
@external
def get_dy(i: int128, j: int128, dx: uint256) -> uint256:
    """Quote amount out for exchange"""
    rates: uint256[N_COINS] = [PRECISION, PRECISION]
    xp: uint256[N_COINS] = self._xp_mem(self.balances, rates)
    
    x: uint256 = xp[convert(i, uint256)] + dx * rates[convert(i, uint256)] / PRECISION
    y: uint256 = self._get_y(i, j, x, xp)
    
    dy: uint256 = (xp[convert(j, uint256)] - y - 1) * PRECISION / rates[convert(j, uint256)]
    fee_amount: uint256 = dy * self.fee / FEE_DENOMINATOR
    
    return dy - fee_amount

@view
@external
def get_virtual_price() -> uint256:
    """
    Virtual price: D / total_supply
    วัดว่า LP Token มีมูลค่าเพิ่มขึ้นแค่ไหน
    """
    D: uint256 = self._get_D(
        self._xp_mem(self.balances, [PRECISION, PRECISION]),
        self._A()
    )
    
    lp_supply: uint256 = ERC20(self.lp_token).totalSupply()
    
    if lp_supply == 0:
        return PRECISION
    
    return D * PRECISION / lp_supply

@view
@external
def A() -> uint256:
    return self._A() / A_PRECISION

@view
@external
def get_balances() -> uint256[N_COINS]:
    return self.balances

# ════════════════════════
# ADMIN FUNCTIONS
# ════════════════════════

RAMP_A_DELAY: constant(uint256) = 86400 * 1  # 1 day minimum

@external
def ramp_A(future_A: uint256, future_time: uint256):
    """
    Gradually change amplification parameter
    ทำให้ราคา adjust ได้โดยไม่เกิด Arbitrage ขนาดใหญ่
    """
    assert msg.sender == self.owner, "Not owner"
    assert block.timestamp >= self.initial_A_time + RAMP_A_DELAY, "Too soon"
    assert future_time >= block.timestamp + RAMP_A_DELAY, "Insufficient time"
    
    initial_A: uint256 = self._A()
    future_A_precise: uint256 = future_A * A_PRECISION
    
    assert future_A > 0 and future_A < 10**6, "Invalid A"
    
    self.initial_A = initial_A
    self.future_A = future_A_precise
    self.initial_A_time = block.timestamp
    self.future_A_time = future_time
    
    log RampA(initial_A, future_A_precise, block.timestamp, future_time)
```

---

## 5. Protocol Governance Architecture {#governance}

```python
# @version 0.4.0
# SPDX-License-Identifier: MIT
# ไฟล์: contracts/governance/Governor.vy

"""
On-chain Governance Contract
- Token-weighted voting
- Timelock execution
- Multi-stage proposal lifecycle
"""

interface IVotingToken:
    def getVotes(account: address) -> uint256: view
    def getPastVotes(account: address, block_number: uint256) -> uint256: view
    def getPastTotalSupply(block_number: uint256) -> uint256: view

# ════════════════════════
# ENUMS (as constants)
# ════════════════════════
PROPOSAL_PENDING: constant(uint8) = 0
PROPOSAL_ACTIVE: constant(uint8) = 1
PROPOSAL_CANCELED: constant(uint8) = 2
PROPOSAL_DEFEATED: constant(uint8) = 3
PROPOSAL_SUCCEEDED: constant(uint8) = 4
PROPOSAL_QUEUED: constant(uint8) = 5
PROPOSAL_EXPIRED: constant(uint8) = 6
PROPOSAL_EXECUTED: constant(uint8) = 7

# ════════════════════════
# STRUCTS
# ════════════════════════

struct ProposalCore:
    proposer: address
    vote_start: uint256      # Block number when voting starts
    vote_end: uint256        # Block number when voting ends
    executed: bool
    canceled: bool

struct ProposalVote:
    against_votes: uint256
    for_votes: uint256
    abstain_votes: uint256

struct ProposalAction:
    target: address
    value: uint256
    calldata: Bytes[4096]

# ════════════════════════
# STATE
# ════════════════════════

token: address               # Governance token
timelock: address            # Timelock controller

proposal_count: uint256
proposals: HashMap[uint256, ProposalCore]
proposal_votes: HashMap[uint256, ProposalVote]
has_voted: HashMap[uint256, HashMap[address, bool]]
vote_support: HashMap[uint256, HashMap[address, uint8]]

# Actions per proposal
proposal_actions: HashMap[uint256, DynArray[ProposalAction, 10]]

# Parameters
voting_delay: uint256        # Blocks until voting starts
voting_period: uint256       # Blocks for voting
proposal_threshold: uint256  # Tokens needed to propose
quorum_numerator: uint256    # % of total supply needed

# ════════════════════════
# EVENTS
# ════════════════════════

event ProposalCreated:
    proposal_id: indexed(uint256)
    proposer: indexed(address)
    vote_start: uint256
    vote_end: uint256
    description: String[1000]

event VoteCast:
    voter: indexed(address)
    proposal_id: indexed(uint256)
    support: uint8
    weight: uint256
    reason: String[200]

event ProposalExecuted:
    proposal_id: indexed(uint256)

event ProposalCanceled:
    proposal_id: indexed(uint256)

# ════════════════════════
# CONSTRUCTOR
# ════════════════════════

@deploy
def __init__(
    token_addr: address,
    timelock_addr: address,
    initial_voting_delay: uint256,
    initial_voting_period: uint256,
    initial_proposal_threshold: uint256,
    initial_quorum_numerator: uint256
):
    self.token = token_addr
    self.timelock = timelock_addr
    self.voting_delay = initial_voting_delay
    self.voting_period = initial_voting_period
    self.proposal_threshold = initial_proposal_threshold
    self.quorum_numerator = initial_quorum_numerator

# ════════════════════════
# PROPOSAL LIFECYCLE
# ════════════════════════

@external
def propose(
    targets: DynArray[address, 10],
    values: DynArray[uint256, 10],
    calldatas: DynArray[Bytes[4096], 10],
    description: String[1000]
) -> uint256:
    """
    Create new governance proposal
    """
    assert len(targets) > 0, "No actions"
    assert len(targets) == len(values), "Mismatch"
    assert len(targets) == len(calldatas), "Mismatch"
    
    # Check proposer has enough tokens
    proposer_votes: uint256 = IVotingToken(self.token).getVotes(msg.sender)
    assert proposer_votes >= self.proposal_threshold, "Insufficient tokens"
    
    # Calculate voting window
    start: uint256 = block.number + self.voting_delay
    end: uint256 = start + self.voting_period
    
    # Create proposal
    self.proposal_count += 1
    proposal_id: uint256 = self.proposal_count
    
    self.proposals[proposal_id] = ProposalCore({
        proposer: msg.sender,
        vote_start: start,
        vote_end: end,
        executed: False,
        canceled: False
    })
    
    # Store actions
    for i: uint256 in range(10):
        if i >= len(targets):
            break
        self.proposal_actions[proposal_id].append(ProposalAction({
            target: targets[i],
            value: values[i],
            calldata: calldatas[i]
        }))
    
    log ProposalCreated(proposal_id, msg.sender, start, end, description)
    
    return proposal_id

@external
def cast_vote(
    proposal_id: uint256,
    support: uint8,  # 0=Against, 1=For, 2=Abstain
    reason: String[200]
) -> uint256:
    """
    Cast vote on proposal
    """
    assert not self.has_voted[proposal_id][msg.sender], "Already voted"
    
    proposal: ProposalCore = self.proposals[proposal_id]
    assert self._state(proposal_id) == PROPOSAL_ACTIVE, "Proposal not active"
    
    # Get voting weight at snapshot
    weight: uint256 = IVotingToken(self.token).getPastVotes(
        msg.sender,
        proposal.vote_start
    )
    
    assert weight > 0, "No voting power"
    
    # Record vote
    self.has_voted[proposal_id][msg.sender] = True
    self.vote_support[proposal_id][msg.sender] = support
    
    if support == 0:
        self.proposal_votes[proposal_id].against_votes += weight
    elif support == 1:
        self.proposal_votes[proposal_id].for_votes += weight
    elif support == 2:
        self.proposal_votes[proposal_id].abstain_votes += weight
    else:
        raise "Invalid support value"
    
    log VoteCast(msg.sender, proposal_id, support, weight, reason)
    
    return weight

@external
def execute(proposal_id: uint256):
    """
    Execute successful proposal
    """
    assert self._state(proposal_id) == PROPOSAL_SUCCEEDED, "Proposal not succeeded"
    
    self.proposals[proposal_id].executed = True
    
    # Execute each action through timelock
    for action: ProposalAction in self.proposal_actions[proposal_id]:
        # In production, actions go through timelock first
        # For simplicity, direct execution here
        result: Bytes[32] = b""
        success: bool = False
        
        if action.value > 0:
            success, result = raw_call(
                action.target,
                action.calldata,
                max_outsize=32,
                value=action.value,
                revert_on_failure=False
            )
        else:
            success, result = raw_call(
                action.target,
                action.calldata,
                max_outsize=32,
                revert_on_failure=False
            )
        
        assert success, "Action execution failed"
    
    log ProposalExecuted(proposal_id)

# ════════════════════════
# VIEW FUNCTIONS
# ════════════════════════

@view
@internal
def _state(proposal_id: uint256) -> uint8:
    proposal: ProposalCore = self.proposals[proposal_id]
    
    if proposal.canceled:
        return PROPOSAL_CANCELED
    
    if proposal.executed:
        return PROPOSAL_EXECUTED
    
    if block.number <= proposal.vote_start:
        return PROPOSAL_PENDING
    
    if block.number <= proposal.vote_end:
        return PROPOSAL_ACTIVE
    
    votes: ProposalVote = self.proposal_votes[proposal_id]
    
    # Check quorum
    total_supply: uint256 = IVotingToken(self.token).getPastTotalSupply(
        proposal.vote_start
    )
    quorum: uint256 = (total_supply * self.quorum_numerator) / 100
    
    if votes.for_votes + votes.abstain_votes < quorum:
        return PROPOSAL_DEFEATED
    
    if votes.for_votes <= votes.against_votes:
        return PROPOSAL_DEFEATED
    
    return PROPOSAL_SUCCEEDED

@view
@external
def state(proposal_id: uint256) -> uint8:
    return self._state(proposal_id)

@view
@external
def get_votes(proposal_id: uint256) -> ProposalVote:
    return self.proposal_votes[proposal_id]

@view
@external
def has_voted_on(proposal_id: uint256, account: address) -> bool:
    return self.has_voted[proposal_id][account]
```

---

## 6. Production Deployment Architecture {#production}

```python
# deploy/deploy_production.py
"""
Production Deployment Script
ใช้สำหรับ Deploy Protocol ใหม่บน Mainnet
"""

import json
import time
from web3 import Web3
from eth_account import Account

def deploy_protocol():
    """Deploy complete protocol"""
    
    print("🚀 Starting Protocol Deployment")
    print("=" * 50)
    
    # 1. Deploy Token
    print("\n📋 Step 1: Deploying Governance Token...")
    # governance_token = deploy_contract("GovernanceToken", [name, symbol, initial_supply])
    
    # 2. Deploy Timelock
    print("📋 Step 2: Deploying Timelock...")
    TIMELOCK_DELAY = 2 * 24 * 3600  # 2 days
    # timelock = deploy_contract("TimelockController", [TIMELOCK_DELAY, [governance], [governance]])
    
    # 3. Deploy Governor
    print("📋 Step 3: Deploying Governor...")
    VOTING_DELAY = 1       # 1 block
    VOTING_PERIOD = 50400  # ~1 week
    PROPOSAL_THRESHOLD = 1_000_000 * 10**18  # 1M tokens
    QUORUM = 4            # 4% quorum
    # governor = deploy_contract("Governor", [token, timelock, VOTING_DELAY, VOTING_PERIOD, PROPOSAL_THRESHOLD, QUORUM])
    
    # 4. Deploy Registry
    print("📋 Step 4: Deploying Registry...")
    # registry = deploy_contract("Registry", [timelock])
    
    # 5. Deploy Factory
    print("📋 Step 5: Deploying Pool Factory...")
    # factory = deploy_contract("PoolFactory", [pool_implementation, registry])
    
    # 6. Deploy Router
    print("📋 Step 6: Deploying Router...")
    # router = deploy_contract("Router", [factory, WETH_ADDRESS])
    
    # 7. Setup Roles
    print("\n🔑 Setting up roles...")
    # timelock.grantRole(PROPOSER_ROLE, governor)
    # timelock.grantRole(EXECUTOR_ROLE, governor)
    # timelock.revokeRole(ADMIN_ROLE, deployer)  # Renounce admin!
    
    # 8. Register contracts in Registry
    print("\n📝 Registering contracts...")
    # registry.register("factory", factory)
    # registry.register("router", router)
    # registry.register("governor", governor)
    
    print("\n✅ Deployment Complete!")
    print("=" * 50)
    
    # Save deployment info
    deployment_info = {
        "network": "mainnet",
        "timestamp": int(time.time()),
        "contracts": {
            # "governance_token": governance_token.address,
            # "timelock": timelock.address,
            # "governor": governor.address,
            # "registry": registry.address,
            # "factory": factory.address,
            # "router": router.address,
        }
    }
    
    with open("deployment.json", "w") as f:
        json.dump(deployment_info, f, indent=2)
    
    print("\n📄 Deployment info saved to deployment.json")
    
    return deployment_info


def verify_deployment(deployment_info: dict):
    """Verify all contracts are deployed correctly"""
    
    print("\n🔍 Verifying Deployment...")
    
    # Check each contract
    for name, address in deployment_info["contracts"].items():
        print(f"  Checking {name}: {address}")
        # code_size = w3.eth.get_code(address)
        # assert len(code_size) > 0, f"{name} not deployed!"
        print(f"  ✅ {name} verified")
    
    print("\n✅ All contracts verified!")


if __name__ == "__main__":
    info = deploy_protocol()
    verify_deployment(info)
```

### Production Checklist

```markdown
## Pre-Deployment Checklist

### Code Quality
- [ ] All tests passing (100% coverage)
- [ ] Fuzzing tests run
- [ ] Invariant tests pass
- [ ] Static analysis (Slither) clean
- [ ] Manual code review complete

### Security
- [ ] External security audit completed
- [ ] Audit findings addressed
- [ ] Emergency pause mechanism tested
- [ ] Access control verified
- [ ] Oracle manipulation tested

### Deployment
- [ ] Deploy script tested on testnet
- [ ] Gas estimates verified
- [ ] Contract verification on Etherscan planned
- [ ] Multi-sig for admin operations
- [ ] Timelock delay set appropriately (min 24-48h)

### Documentation
- [ ] README updated
- [ ] Contract documentation complete
- [ ] Deployment guide written
- [ ] Admin guide written

### Monitoring
- [ ] Monitoring setup (Tenderly, Openzeppelin Defender)
- [ ] Alert rules configured
- [ ] Incident response plan ready

### Legal
- [ ] Terms of Service ready (if applicable)
- [ ] Regulatory compliance checked
- [ ] Bug bounty program set up
```

---

## สรุป Part 091

ในส่วนนี้คุณได้เรียนรู้:
- ✅ Real-world Protocol Architecture
- ✅ Registry, Factory, Router Pattern
- ✅ Modular Design ใน Vyper 0.4.0
- ✅ Curve-like StableSwap AMM
- ✅ Governance Architecture (Governor Contract)
- ✅ Production Deployment Strategy
- ✅ Pre-deployment Checklist

## แบบฝึกหัด

1. **Deploy** Protocol บน Sepolia Testnet ครบทุก Component
2. **สร้าง** Subgraph สำหรับ Index Events
3. **เขียน** Governance Proposal ครั้งแรก
4. **ทดสอบ** Emergency Pause ในทุก Scenarios
5. **Verify** Contracts บน Etherscan

---

**ก่อนหน้า: [Part 090 - Whitepaper Writing](part_090_whitepaper.md)**  
**ต่อไป: [Part 092 - Production Deployment Checklist](part_092_deployment_checklist.md)**
