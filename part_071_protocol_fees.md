# Part 071: Protocol Fee Mechanisms (กลไกค่าธรรมเนียม Protocol)

## สารบัญ
1. [บทนำ Protocol Fees](#s1)
2. [Fee-on-Transfer Tokens](#s2)
3. [Protocol Fee Switch](#s3)
4. [Fee Distribution](#s4)
5. [Dynamic Fee Curves](#s5)
6. [ProtocolFeeManager Contract](#s6)
7. [FeeDistributor Contract](#s7)
8. [Security Considerations](#s8)

---

## 1. บทนำ Protocol Fees {#s1}

**Protocol fees** คือกลไกที่ protocols ใช้เพื่อสร้างรายได้และ sustain ตัวเอง
ค่าธรรมเนียมสามารถแจกจ่ายไปยัง:
- **Stakers**: ผู้ stake governance tokens
- **Treasury**: กองทุนพัฒนา protocol
- **Liquidity Providers**: ผู้ให้ liquidity
- **Buyback**: ซื้อคืน token เพื่อ burn

### ประเภทของ Fee Structures

| ประเภท | คำอธิบาย | ตัวอย่าง |
|---|---|---|
| Fixed Fee | อัตราคงที่ | 0.3% ทุก swap |
| Tiered Fee | ตามปริมาณ/stake | 0.1-0.3% ตาม tier |
| Dynamic Fee | ตาม volatility | สูงขึ้นในช่วง volatile |
| Fee-on-Transfer | ตัด % ทุก transfer | Deflationary tokens |

---

## 2. Fee-on-Transfer Tokens {#s2}

```python
# @version 0.4.0
# FeeOnTransferToken.vy
# ERC-20 token that takes a fee on every transfer
# Common in deflationary token designs

from vyper.interfaces import ERC20

# ============================================================
# Constants
# ============================================================

MAX_FEE: constant(uint256) = 1000  # 10% maximum fee (basis points)
FEE_DENOMINATOR: constant(uint256) = 10000  # Basis points

# ============================================================
# Storage
# ============================================================

name: public(String[64])
symbol: public(String[32])
decimals: public(uint8)
total_supply: public(uint256)

balances: HashMap[address, uint256]
allowances: HashMap[address, HashMap[address, uint256]]

owner: public(address)

# Fee configuration
transfer_fee: public(uint256)      # Fee in basis points
burn_portion: public(uint256)       # % of fee that gets burned
treasury_portion: public(uint256)   # % of fee to treasury
staker_portion: public(uint256)     # % of fee to stakers

treasury: public(address)
staking_contract: public(address)

# Fee exemptions
fee_exempt: public(HashMap[address, bool])

# Collected fees
total_fees_collected: public(uint256)
total_burned: public(uint256)

# ============================================================
# Events
# ============================================================

event Transfer:
    sender: indexed(address)
    receiver: indexed(address)
    value: uint256

event Approval:
    owner: indexed(address)
    spender: indexed(address)
    value: uint256

event FeeCollected:
    from_: indexed(address)
    fee_amount: uint256
    burned: uint256
    to_treasury: uint256
    to_stakers: uint256

event FeeUpdated:
    old_fee: uint256
    new_fee: uint256

# ============================================================
# Constructor
# ============================================================

@deploy
def __init__(
    token_name: String[64],
    token_symbol: String[32],
    initial_supply: uint256,
    fee_bps: uint256,
    treasury_address: address
):
    assert fee_bps <= MAX_FEE, "Fee too high"
    assert treasury_address != empty(address), "Invalid treasury"
    
    self.name = token_name
    self.symbol = token_symbol
    self.decimals = 18
    self.total_supply = initial_supply
    self.balances[msg.sender] = initial_supply
    self.owner = msg.sender
    
    # Fee configuration: 30% burn, 40% treasury, 30% stakers
    self.transfer_fee = fee_bps
    self.burn_portion = 3000      # 30%
    self.treasury_portion = 4000  # 40%
    self.staker_portion = 3000    # 30%
    
    self.treasury = treasury_address
    
    # Owner is exempt from fees
    self.fee_exempt[msg.sender] = True
    
    log Transfer(empty(address), msg.sender, initial_supply)

# ============================================================
# Fee Calculation
# ============================================================

@view
@internal
def _calculate_fee(amount: uint256) -> uint256:
    """Calculate fee amount from transfer amount"""
    return amount * self.transfer_fee / FEE_DENOMINATOR

@view
@internal
def _split_fee(fee: uint256) -> (uint256, uint256, uint256):
    """
    Split fee into burn, treasury, and staker portions
    Returns (burn, treasury, stakers)
    """
    burn_amount: uint256 = fee * self.burn_portion / FEE_DENOMINATOR
    treasury_amount: uint256 = fee * self.treasury_portion / FEE_DENOMINATOR
    staker_amount: uint256 = fee - burn_amount - treasury_amount
    
    return burn_amount, treasury_amount, staker_amount

# ============================================================
# Transfer Logic with Fee
# ============================================================

@internal
def _transfer(from_: address, to: address, amount: uint256):
    """
    Internal transfer with fee-on-transfer logic
    """
    assert from_ != empty(address), "Transfer from zero"
    assert to != empty(address), "Transfer to zero"
    assert self.balances[from_] >= amount, "Insufficient balance"
    
    # Check if fee exempt
    if self.fee_exempt[from_] or self.fee_exempt[to]:
        # No fee for exempt addresses
        self.balances[from_] -= amount
        self.balances[to] += amount
        log Transfer(from_, to, amount)
        return
    
    # Calculate fee
    fee: uint256 = self._calculate_fee(amount)
    transfer_amount: uint256 = amount - fee
    
    # Split the fee
    burn_amount: uint256 = 0
    treasury_amount: uint256 = 0
    staker_amount: uint256 = 0
    burn_amount, treasury_amount, staker_amount = self._split_fee(fee)
    
    # Apply transfer
    self.balances[from_] -= amount
    self.balances[to] += transfer_amount
    
    # Burn portion
    if burn_amount > 0:
        self.total_supply -= burn_amount
        self.total_burned += burn_amount
        log Transfer(from_, empty(address), burn_amount)
    
    # Treasury portion
    if treasury_amount > 0:
        self.balances[self.treasury] += treasury_amount
        log Transfer(from_, self.treasury, treasury_amount)
    
    # Staker portion
    if staker_amount > 0 and self.staking_contract != empty(address):
        self.balances[self.staking_contract] += staker_amount
        log Transfer(from_, self.staking_contract, staker_amount)
    elif staker_amount > 0:
        # If no staking contract, send to treasury
        self.balances[self.treasury] += staker_amount
    
    self.total_fees_collected += fee
    
    log Transfer(from_, to, transfer_amount)
    log FeeCollected(from_, fee, burn_amount, treasury_amount, staker_amount)

@external
def transfer(to: address, amount: uint256) -> bool:
    self._transfer(msg.sender, to, amount)
    return True

@external
def transferFrom(from_: address, to: address, amount: uint256) -> bool:
    assert self.allowances[from_][msg.sender] >= amount, "Insufficient allowance"
    self.allowances[from_][msg.sender] -= amount
    self._transfer(from_, to, amount)
    return True

@view
@external
def balanceOf(account: address) -> uint256:
    return self.balances[account]

@external
def approve(spender: address, amount: uint256) -> bool:
    self.allowances[msg.sender][spender] = amount
    log Approval(msg.sender, spender, amount)
    return True

@view
@external
def allowance(owner: address, spender: address) -> uint256:
    return self.allowances[owner][spender]

# ============================================================
# Admin Functions
# ============================================================

@external
def set_transfer_fee(new_fee: uint256):
    assert msg.sender == self.owner, "Not owner"
    assert new_fee <= MAX_FEE, "Fee too high"
    
    old_fee: uint256 = self.transfer_fee
    self.transfer_fee = new_fee
    log FeeUpdated(old_fee, new_fee)

@external
def set_fee_exempt(account: address, exempt: bool):
    assert msg.sender == self.owner, "Not owner"
    self.fee_exempt[account] = exempt

@external
def set_staking_contract(staking: address):
    assert msg.sender == self.owner, "Not owner"
    self.staking_contract = staking

@view
@external
def get_transfer_amount(gross_amount: uint256) -> (uint256, uint256):
    """Calculate how much recipient gets vs fee"""
    fee: uint256 = self._calculate_fee(gross_amount)
    return gross_amount - fee, fee
```

---

## 3. Protocol Fee Switch {#s3}

Uniswap-style fee switch ที่ governance ควบคุม

```python
# @version 0.4.0
# ProtocolFeeSwitch.vy
# Governance-controlled protocol fee switch
# Based on Uniswap V2/V3 fee switch pattern

# ============================================================
# Storage
# ============================================================

owner: public(address)
pending_owner: public(address)

# Fee switch state
fee_on: public(bool)
protocol_fee_numerator: public(uint256)    # Fee numerator
protocol_fee_denominator: public(uint256)  # Fee denominator

# Fee recipient
fee_to: public(address)
fee_to_setter: public(address)

# Accumulated fees per token
accumulated_fees: public(HashMap[address, uint256])

# Minimum accrual before distributing
fee_distribution_threshold: public(uint256)

# ============================================================
# Events
# ============================================================

event FeeToChanged:
    old_fee_to: indexed(address)
    new_fee_to: indexed(address)

event FeeSwitchToggled:
    new_state: bool
    fee_numerator: uint256
    fee_denominator: uint256

event FeesDistributed:
    token: indexed(address)
    amount: uint256
    recipient: indexed(address)

event OwnershipTransferStarted:
    previous_owner: indexed(address)
    new_owner: indexed(address)

# ============================================================
# Constructor
# ============================================================

@deploy
def __init__(initial_fee_to_setter: address):
    self.owner = msg.sender
    self.fee_to_setter = initial_fee_to_setter
    
    # Start with fee off
    self.fee_on = False
    self.protocol_fee_numerator = 1
    self.protocol_fee_denominator = 6  # 1/6 of LP fees (Uniswap V2 pattern)
    
    self.fee_distribution_threshold = 10**18  # 1 token minimum

# ============================================================
# Fee Switch Functions
# ============================================================

@external
def turn_fee_on(fee_to_address: address):
    """
    Enable protocol fee collection
    Only callable by fee_to_setter (typically governance)
    """
    assert msg.sender == self.fee_to_setter, "Not fee setter"
    assert fee_to_address != empty(address), "Invalid fee recipient"
    
    self.fee_on = True
    self.fee_to = fee_to_address
    
    log FeeSwitchToggled(True, self.protocol_fee_numerator, self.protocol_fee_denominator)

@external
def turn_fee_off():
    """Disable protocol fee collection"""
    assert msg.sender == self.fee_to_setter, "Not fee setter"
    
    self.fee_on = False
    self.fee_to = empty(address)
    
    log FeeSwitchToggled(False, 0, 0)

@external
def set_fee_rate(numerator: uint256, denominator: uint256):
    """
    Update the fee rate
    Common patterns:
    - 1/6 of LP fees (like Uniswap V2)
    - 1/4 of LP fees
    - Custom percentage
    """
    assert msg.sender == self.fee_to_setter, "Not fee setter"
    assert denominator > 0, "Invalid denominator"
    assert numerator <= denominator, "Fee > 100%"
    assert numerator * 10 <= denominator, "Fee > 10%"  # Max 10%
    
    self.protocol_fee_numerator = numerator
    self.protocol_fee_denominator = denominator

@external
def change_fee_to_setter(new_setter: address):
    """Transfer fee setter role (governance can change this)"""
    assert msg.sender == self.owner, "Not owner"
    self.fee_to_setter = new_setter

# ============================================================
# Fee Calculation
# ============================================================

@view
@external
def calculate_protocol_fee(lp_fee_amount: uint256) -> uint256:
    """
    Calculate protocol's share of LP fees
    If fee is 1/6, and LP fee is 0.3%, protocol gets 0.05%
    """
    if not self.fee_on:
        return 0
    
    return lp_fee_amount * self.protocol_fee_numerator / self.protocol_fee_denominator

@internal
def _accrue_protocol_fee(token: address, lp_fee: uint256):
    """Accrue protocol fees for a token"""
    if not self.fee_on or self.fee_to == empty(address):
        return
    
    protocol_fee: uint256 = lp_fee * self.protocol_fee_numerator / self.protocol_fee_denominator
    self.accumulated_fees[token] += protocol_fee

@external
def distribute_fees(token: address):
    """
    Distribute accumulated fees to fee_to address
    Anyone can call to trigger distribution
    """
    amount: uint256 = self.accumulated_fees[token]
    assert amount >= self.fee_distribution_threshold, "Below threshold"
    assert self.fee_to != empty(address), "No fee recipient"
    
    self.accumulated_fees[token] = 0
    
    from vyper.interfaces import ERC20
    ERC20(token).transfer(self.fee_to, amount)
    
    log FeesDistributed(token, amount, self.fee_to)
```

---

## 4. Fee Distribution {#s4}

```python
# @version 0.4.0
# FeeDistributor.vy
# Distribute protocol fees to multiple beneficiaries
# Weighted distribution among stakers, treasury, and dev fund

from vyper.interfaces import ERC20

# ============================================================
# Structs
# ============================================================

struct Beneficiary:
    recipient: address
    weight: uint256      # Weight in basis points (total = 10000)
    name: String[30]
    total_received: uint256

# ============================================================
# Storage
# ============================================================

owner: public(address)

# Beneficiaries
beneficiaries: public(DynArray[Beneficiary, 10])
total_weight: public(uint256)

# Supported fee tokens
supported_tokens: public(DynArray[address, 20])
is_supported_token: public(HashMap[address, bool])

# Accumulated undistributed fees per token
pending_fees: public(HashMap[address, uint256])

# Total distributed
total_distributed: public(HashMap[address, uint256])

# Distribution epoch tracking
last_distribution: public(uint256)
distribution_frequency: public(uint256)  # Minimum time between distributions

# ============================================================
# Events
# ============================================================

event FeeAccrued:
    token: indexed(address)
    amount: uint256

event FeesDistributed:
    token: indexed(address)
    total_amount: uint256
    epoch: uint256

event BeneficiaryAdded:
    recipient: indexed(address)
    weight: uint256

event BeneficiaryUpdated:
    recipient: indexed(address)
    old_weight: uint256
    new_weight: uint256

# ============================================================
# Constructor
# ============================================================

@deploy
def __init__():
    self.owner = msg.sender
    self.distribution_frequency = 86400  # Minimum 1 day between distributions

@external
def add_beneficiary(recipient: address, weight: uint256, name: String[30]):
    """
    Add a fee beneficiary
    @param recipient Address to receive fees
    @param weight Relative weight (will be normalized to 10000 total)
    """
    assert msg.sender == self.owner, "Not owner"
    assert recipient != empty(address), "Invalid recipient"
    assert weight > 0, "Weight must be positive"
    assert len(self.beneficiaries) < 10, "Max 10 beneficiaries"
    
    # Check recipient not already added
    for b: Beneficiary in self.beneficiaries:
        assert b.recipient != recipient, "Already added"
    
    self.beneficiaries.append(Beneficiary(
        recipient=recipient,
        weight=weight,
        name=name,
        total_received=0
    ))
    self.total_weight += weight
    
    log BeneficiaryAdded(recipient, weight)

@external
def update_weights(new_weights: DynArray[uint256, 10]):
    """Update beneficiary weights"""
    assert msg.sender == self.owner, "Not owner"
    assert len(new_weights) == len(self.beneficiaries), "Length mismatch"
    
    total: uint256 = 0
    for w: uint256 in new_weights:
        total += w
    
    self.total_weight = total
    
    for i: uint256 in range(10):
        if i >= len(self.beneficiaries):
            break
        self.beneficiaries[i].weight = new_weights[i]

@external
def add_supported_token(token: address):
    """Add a token that can be distributed"""
    assert msg.sender == self.owner, "Not owner"
    assert not self.is_supported_token[token], "Already supported"
    
    self.supported_tokens.append(token)
    self.is_supported_token[token] = True

@external
def receive_fees(token: address, amount: uint256):
    """
    Receive fees from protocol contracts
    Protocol sends fees here for distribution
    """
    assert self.is_supported_token[token], "Token not supported"
    assert amount > 0, "Zero amount"
    
    # Transfer tokens from caller
    ERC20(token).transferFrom(msg.sender, self, amount)
    
    self.pending_fees[token] += amount
    log FeeAccrued(token, amount)

@external
def distribute_token(token: address):
    """
    Distribute accumulated fees for a specific token
    """
    assert self.is_supported_token[token], "Token not supported"
    assert block.timestamp >= self.last_distribution + self.distribution_frequency, \
        "Too soon"
    
    amount: uint256 = self.pending_fees[token]
    assert amount > 0, "No fees to distribute"
    
    if self.total_weight == 0:
        return
    
    # Reset pending
    self.pending_fees[token] = 0
    self.last_distribution = block.timestamp
    
    # Distribute to each beneficiary
    distributed: uint256 = 0
    for i: uint256 in range(10):
        if i >= len(self.beneficiaries):
            break
        
        b: Beneficiary = self.beneficiaries[i]
        beneficiary_amount: uint256 = amount * b.weight / self.total_weight
        
        if beneficiary_amount > 0 and i < len(self.beneficiaries) - 1:
            ERC20(token).transfer(b.recipient, beneficiary_amount)
            self.beneficiaries[i].total_received += beneficiary_amount
            distributed += beneficiary_amount
    
    # Last beneficiary gets remainder to prevent dust
    last_idx: uint256 = len(self.beneficiaries) - 1
    remainder: uint256 = amount - distributed
    if remainder > 0:
        last_b: Beneficiary = self.beneficiaries[last_idx]
        ERC20(token).transfer(last_b.recipient, remainder)
        self.beneficiaries[last_idx].total_received += remainder
    
    self.total_distributed[token] += amount
    log FeesDistributed(token, amount, block.timestamp)

@external
def distribute_all():
    """Distribute fees for all supported tokens"""
    assert block.timestamp >= self.last_distribution + self.distribution_frequency, \
        "Too soon"
    
    for token: address in self.supported_tokens:
        if self.pending_fees[token] > 0:
            self._distribute_single(token)

@internal
def _distribute_single(token: address):
    """Internal distribution for one token"""
    amount: uint256 = self.pending_fees[token]
    if amount == 0 or self.total_weight == 0:
        return
    
    self.pending_fees[token] = 0
    distributed: uint256 = 0
    
    for i: uint256 in range(10):
        if i >= len(self.beneficiaries):
            break
        
        b: Beneficiary = self.beneficiaries[i]
        b_amount: uint256 = amount * b.weight / self.total_weight
        
        if b_amount > 0 and i < len(self.beneficiaries) - 1:
            ERC20(token).transfer(b.recipient, b_amount)
            self.beneficiaries[i].total_received += b_amount
            distributed += b_amount
    
    if len(self.beneficiaries) > 0:
        last_idx: uint256 = len(self.beneficiaries) - 1
        remainder: uint256 = amount - distributed
        if remainder > 0:
            ERC20(token).transfer(self.beneficiaries[last_idx].recipient, remainder)
            self.beneficiaries[last_idx].total_received += remainder
    
    self.total_distributed[token] += amount

@view
@external
def get_pending_fees() -> DynArray[uint256, 20]:
    """Get pending fees for all supported tokens"""
    result: DynArray[uint256, 20] = []
    for token: address in self.supported_tokens:
        result.append(self.pending_fees[token])
    return result
```

---

## 5. Dynamic Fee Curves {#s5}

```python
# @version 0.4.0
# DynamicFeeCalculator.vy
# Dynamic fee calculation based on market conditions
# Higher fees during high volatility, lower during stable periods

# ============================================================
# Interface for price oracle
# ============================================================

interface IChainlinkOracle:
    def latestRoundData() -> (uint80, int256, uint256, uint256, uint80): view

# ============================================================
# Constants
# ============================================================

# Fee bounds
MIN_FEE: constant(uint256) = 5     # 0.05% minimum fee
MAX_FEE: constant(uint256) = 300   # 3% maximum fee
BASE_FEE: constant(uint256) = 30   # 0.3% base fee
FEE_DENOM: constant(uint256) = 10000

# Volatility thresholds (percentage change in basis points)
LOW_VOL_THRESHOLD: constant(uint256) = 50    # 0.5% price change
HIGH_VOL_THRESHOLD: constant(uint256) = 500  # 5% price change

# ============================================================
# Storage
# ============================================================

owner: public(address)
oracle: public(address)

# Historical prices for volatility calculation
price_history: public(DynArray[uint256, 24])  # Last 24 price observations
last_price_update: public(uint256)
price_update_interval: public(uint256)

# Current fee
current_fee: public(uint256)
fee_last_updated: public(uint256)

# Volume tracking for fee adjustment
hourly_volume: public(uint256)
volume_window_start: public(uint256)

# ============================================================
# Events
# ============================================================

event FeeUpdated:
    old_fee: uint256
    new_fee: uint256
    volatility: uint256

event PriceRecorded:
    price: uint256
    timestamp: uint256

# ============================================================
# Constructor
# ============================================================

@deploy
def __init__(price_oracle: address):
    self.owner = msg.sender
    self.oracle = price_oracle
    self.current_fee = BASE_FEE
    self.price_update_interval = 3600  # 1 hour

# ============================================================
# Price and Volatility
# ============================================================

@internal
def _get_current_price() -> uint256:
    """Get current price from oracle"""
    round_id: uint80 = 0
    answer: int256 = 0
    started_at: uint256 = 0
    updated_at: uint256 = 0
    answered_in_round: uint80 = 0
    
    round_id, answer, started_at, updated_at, answered_in_round = \
        IChainlinkOracle(self.oracle).latestRoundData()
    
    assert answer > 0, "Invalid price"
    assert block.timestamp - updated_at < 3600, "Stale price"
    
    return convert(answer, uint256)

@external
def update_price():
    """Record current price for volatility calculation"""
    assert block.timestamp >= self.last_price_update + self.price_update_interval, \
        "Too soon"
    
    price: uint256 = self._get_current_price()
    
    # Add to history (rolling window of 24)
    if len(self.price_history) >= 24:
        # Remove oldest (shift array - simplified)
        new_history: DynArray[uint256, 24] = []
        for i: uint256 in range(24):
            if i > 0 and i < len(self.price_history):
                new_history.append(self.price_history[i])
        new_history.append(price)
        self.price_history = new_history
    else:
        self.price_history.append(price)
    
    self.last_price_update = block.timestamp
    log PriceRecorded(price, block.timestamp)

@view
@internal
def _calculate_volatility() -> uint256:
    """
    Calculate price volatility as max % change in history
    Returns volatility in basis points
    """
    history: DynArray[uint256, 24] = self.price_history
    
    if len(history) < 2:
        return 0
    
    max_change: uint256 = 0
    
    for i: uint256 in range(23):
        if i + 1 >= len(history):
            break
        
        p1: uint256 = history[i]
        p2: uint256 = history[i + 1]
        
        if p1 == 0:
            continue
        
        # Calculate absolute percentage change
        change: uint256 = 0
        if p2 > p1:
            change = (p2 - p1) * FEE_DENOM / p1
        else:
            change = (p1 - p2) * FEE_DENOM / p1
        
        if change > max_change:
            max_change = change
    
    return max_change

# ============================================================
# Dynamic Fee Calculation
# ============================================================

@view
@external
def calculate_current_fee() -> uint256:
    """
    Calculate fee based on current market conditions
    Higher volatility = higher fee
    """
    volatility: uint256 = self._calculate_volatility()
    return self._fee_curve(volatility)

@view
@internal
def _fee_curve(volatility: uint256) -> uint256:
    """
    Fee curve function:
    - Low volatility: BASE_FEE or lower
    - High volatility: up to MAX_FEE
    
    Uses a piecewise linear curve:
    - 0-LOW_VOL: MIN_FEE to BASE_FEE
    - LOW_VOL-HIGH_VOL: BASE_FEE to MAX_FEE
    - >HIGH_VOL: MAX_FEE
    """
    if volatility <= LOW_VOL_THRESHOLD:
        # Linear from MIN_FEE to BASE_FEE
        if LOW_VOL_THRESHOLD == 0:
            return BASE_FEE
        return MIN_FEE + (BASE_FEE - MIN_FEE) * volatility / LOW_VOL_THRESHOLD
    
    elif volatility <= HIGH_VOL_THRESHOLD:
        # Linear from BASE_FEE to MAX_FEE
        range_vol: uint256 = HIGH_VOL_THRESHOLD - LOW_VOL_THRESHOLD
        excess_vol: uint256 = volatility - LOW_VOL_THRESHOLD
        return BASE_FEE + (MAX_FEE - BASE_FEE) * excess_vol / range_vol
    
    else:
        return MAX_FEE

@external
def update_fee():
    """
    Update the stored fee based on current volatility
    Can be called by anyone to keep fee fresh
    """
    volatility: uint256 = self._calculate_volatility()
    new_fee: uint256 = self._fee_curve(volatility)
    
    old_fee: uint256 = self.current_fee
    self.current_fee = new_fee
    self.fee_last_updated = block.timestamp
    
    log FeeUpdated(old_fee, new_fee, volatility)

@view
@external
def get_fee_for_amount(amount: uint256) -> uint256:
    """Get fee amount for a specific trade size"""
    return amount * self.current_fee / FEE_DENOM
```

---

## 6. ProtocolFeeManager Contract {#s6}

```python
# @version 0.4.0
# ProtocolFeeManager.vy
# Central fee management for entire protocol
# Controls fee collection, rates, and distribution

from vyper.interfaces import ERC20

interface IFeeDistributor:
    def receive_fees(token: address, amount: uint256): nonpayable

interface IPair:
    def collect_fees() -> (uint256, uint256): nonpayable

# ============================================================
# Structs
# ============================================================

struct PoolFeeConfig:
    base_fee: uint256      # Base fee in bps
    protocol_share: uint256  # Protocol's share in bps (of base_fee)
    lp_share: uint256        # LP's share in bps (of base_fee)
    dynamic_enabled: bool    # Use dynamic fees
    last_fee: uint256        # Last computed fee
    last_update: uint256

# ============================================================
# Storage
# ============================================================

owner: public(address)
governance: public(address)

# Fee distributor contract
fee_distributor: public(address)

# Per-pool fee configuration
pool_fees: public(HashMap[address, PoolFeeConfig])

# Global defaults
default_base_fee: public(uint256)
default_protocol_share: public(uint256)

# Fee collection state
last_collection_time: public(HashMap[address, uint256])
collection_cooldown: public(uint256)

# Emergency pause
paused: public(bool)

# Total stats
total_protocol_fees_collected: public(HashMap[address, uint256])

# ============================================================
# Events
# ============================================================

event PoolFeeConfigured:
    pool: indexed(address)
    base_fee: uint256
    protocol_share: uint256

event FeesCollectedFromPool:
    pool: indexed(address)
    token0_amount: uint256
    token1_amount: uint256

event GlobalConfigUpdated:
    default_fee: uint256
    protocol_share: uint256

# ============================================================
# Constructor
# ============================================================

@deploy
def __init__(distributor: address):
    self.owner = msg.sender
    self.governance = msg.sender
    self.fee_distributor = distributor
    
    # Default: 0.3% fee, 1/6 to protocol (0.05%)
    self.default_base_fee = 30       # 0.3%
    self.default_protocol_share = 1667  # 1/6 ≈ 16.67%
    
    self.collection_cooldown = 3600  # 1 hour minimum

# ============================================================
# Pool Fee Management
# ============================================================

@external
def configure_pool_fee(
    pool: address,
    base_fee: uint256,
    protocol_share: uint256,
    use_dynamic: bool
):
    """Configure fee for a specific pool"""
    assert msg.sender == self.governance, "Not governance"
    assert base_fee <= 1000, "Fee too high (max 10%)"
    assert protocol_share <= 5000, "Protocol share too high (max 50%)"
    
    lp_share: uint256 = 10000 - protocol_share
    
    self.pool_fees[pool] = PoolFeeConfig(
        base_fee=base_fee,
        protocol_share=protocol_share,
        lp_share=lp_share,
        dynamic_enabled=use_dynamic,
        last_fee=base_fee,
        last_update=block.timestamp
    )
    
    log PoolFeeConfigured(pool, base_fee, protocol_share)

@view
@external
def get_pool_fee(pool: address) -> (uint256, uint256, uint256):
    """
    Get fee configuration for a pool
    Returns (total_fee, lp_fee, protocol_fee) in bps
    """
    config: PoolFeeConfig = self.pool_fees[pool]
    
    total_fee: uint256 = config.base_fee
    if total_fee == 0:
        total_fee = self.default_base_fee
    
    protocol_share: uint256 = config.protocol_share
    if protocol_share == 0:
        protocol_share = self.default_protocol_share
    
    protocol_fee: uint256 = total_fee * protocol_share / 10000
    lp_fee: uint256 = total_fee - protocol_fee
    
    return total_fee, lp_fee, protocol_fee

@external
def collect_and_distribute(pools: DynArray[address, 20], tokens: DynArray[address, 20]):
    """
    Collect fees from pools and send to distributor
    Can be called by anyone (incentivized in some designs)
    """
    assert not self.paused, "Paused"
    
    for pool: address in pools:
        if block.timestamp < self.last_collection_time[pool] + self.collection_cooldown:
            continue
        
        # Collect from pool
        amount0: uint256 = 0
        amount1: uint256 = 0
        amount0, amount1 = IPair(pool).collect_fees()
        
        self.last_collection_time[pool] = block.timestamp
        log FeesCollectedFromPool(pool, amount0, amount1)
    
    # Send collected tokens to distributor
    for token: address in tokens:
        balance: uint256 = ERC20(token).balanceOf(self)
        if balance > 0:
            ERC20(token).approve(self.fee_distributor, balance)
            IFeeDistributor(self.fee_distributor).receive_fees(token, balance)
            self.total_protocol_fees_collected[token] += balance

@external
def update_global_config(new_default_fee: uint256, new_protocol_share: uint256):
    """Update global fee defaults"""
    assert msg.sender == self.governance, "Not governance"
    assert new_default_fee <= 500, "Max 5%"
    assert new_protocol_share <= 5000, "Max 50% of fees"
    
    self.default_base_fee = new_default_fee
    self.default_protocol_share = new_protocol_share
    
    log GlobalConfigUpdated(new_default_fee, new_protocol_share)

@external
def emergency_pause():
    """Emergency pause fee collection"""
    assert msg.sender == self.owner, "Not owner"
    self.paused = True

@external
def unpause():
    """Unpause fee collection"""
    assert msg.sender == self.owner, "Not owner"
    self.paused = False
```

---

## 7. FeeDistributor Contract {#s7}

```python
# @version 0.4.0
# FeeDistributorAdvanced.vy
# Advanced fee distribution with staking rewards

from vyper.interfaces import ERC20

interface IVotingEscrow:
    def balanceOf(account: address) -> uint256: view
    def totalSupply() -> uint256: view

# ============================================================
# Storage
# ============================================================

owner: public(address)
governance_token: public(address)
voting_escrow: public(address)  # veToken for boost

# Fee tokens
fee_tokens: public(DynArray[address, 10])
is_fee_token: public(HashMap[address, bool])

# Distribution epochs (weekly)
EPOCH_DURATION: constant(uint256) = 86400 * 7  # 1 week
current_epoch: public(uint256)
epoch_start: public(HashMap[uint256, uint256])

# Fees per epoch
epoch_fees: public(HashMap[uint256, HashMap[address, uint256]])

# User claim tracking
last_claimed_epoch: public(HashMap[address, uint256])

# Emergency
emergency_return: public(address)

# ============================================================
# Events
# ============================================================

event FeeAdded:
    epoch: uint256
    token: indexed(address)
    amount: uint256

event FeesClaimed:
    user: indexed(address)
    epoch: uint256
    token: indexed(address)
    amount: uint256

event EpochAdvanced:
    new_epoch: uint256

# ============================================================
# Constructor
# ============================================================

@deploy
def __init__(
    gov_token: address,
    ve_token: address,
    emergency_addr: address
):
    self.owner = msg.sender
    self.governance_token = gov_token
    self.voting_escrow = ve_token
    self.emergency_return = emergency_addr
    
    self.current_epoch = 0
    self.epoch_start[0] = block.timestamp

# ============================================================
# Epoch Management
# ============================================================

@external
def advance_epoch():
    """Start a new distribution epoch"""
    current_epoch_start: uint256 = self.epoch_start[self.current_epoch]
    assert block.timestamp >= current_epoch_start + EPOCH_DURATION, "Epoch not complete"
    
    self.current_epoch += 1
    self.epoch_start[self.current_epoch] = block.timestamp
    
    log EpochAdvanced(self.current_epoch)

# ============================================================
# Fee Addition
# ============================================================

@external
def add_fees(token: address, amount: uint256):
    """Add fees to current epoch"""
    assert self.is_fee_token[token], "Not fee token"
    assert amount > 0, "Zero amount"
    
    ERC20(token).transferFrom(msg.sender, self, amount)
    
    epoch: uint256 = self.current_epoch
    self.epoch_fees[epoch][token] += amount
    
    log FeeAdded(epoch, token, amount)

# ============================================================
# Fee Claiming
# ============================================================

@view
@external
def claimable_fees(user: address, token: address) -> uint256:
    """Calculate claimable fees for a user"""
    total: uint256 = 0
    
    last_epoch: uint256 = self.last_claimed_epoch[user]
    current: uint256 = self.current_epoch
    
    if last_epoch >= current:
        return 0
    
    # Sum fees from unclaimed epochs
    for epoch: uint256 in range(52):  # Max 52 epochs (1 year)
        check_epoch: uint256 = last_epoch + epoch
        if check_epoch >= current:
            break
        
        epoch_amount: uint256 = self.epoch_fees[check_epoch][token]
        if epoch_amount == 0:
            continue
        
        # User's share based on ve balance
        ve_supply: uint256 = IVotingEscrow(self.voting_escrow).totalSupply()
        if ve_supply == 0:
            continue
        
        user_balance: uint256 = IVotingEscrow(self.voting_escrow).balanceOf(user)
        user_share: uint256 = epoch_amount * user_balance / ve_supply
        total += user_share
    
    return total

@external
def claim_fees(tokens: DynArray[address, 10]):
    """Claim accumulated fees for specified tokens"""
    user: address = msg.sender
    current: uint256 = self.current_epoch
    last_epoch: uint256 = self.last_claimed_epoch[user]
    
    assert last_epoch < current, "Already claimed"
    
    # Process each epoch
    for epoch: uint256 in range(52):
        check_epoch: uint256 = last_epoch + epoch
        if check_epoch >= current:
            break
        
        ve_supply: uint256 = IVotingEscrow(self.voting_escrow).totalSupply()
        if ve_supply == 0:
            continue
        
        user_balance: uint256 = IVotingEscrow(self.voting_escrow).balanceOf(user)
        if user_balance == 0:
            continue
        
        for token: address in tokens:
            epoch_amount: uint256 = self.epoch_fees[check_epoch][token]
            if epoch_amount == 0:
                continue
            
            user_amount: uint256 = epoch_amount * user_balance / ve_supply
            
            if user_amount > 0:
                ERC20(token).transfer(user, user_amount)
                log FeesClaimed(user, check_epoch, token, user_amount)
    
    self.last_claimed_epoch[user] = current - 1

@external
def add_fee_token(token: address):
    """Add a supported fee token"""
    assert msg.sender == self.owner, "Not owner"
    assert not self.is_fee_token[token], "Already added"
    
    self.fee_tokens.append(token)
    self.is_fee_token[token] = True
```

---

## 8. Security Considerations {#s8}

```python
# @version 0.4.0
# FeeSecurityPatterns.vy
# Security patterns for fee contracts

from vyper.interfaces import ERC20

# ============================================================
# Fee Calculation Precision
# ============================================================

# SAFE: Use multiplication before division to maintain precision
@view
@internal
def _safe_fee_calc(amount: uint256, fee_bps: uint256) -> uint256:
    # GOOD: multiply first, then divide
    return amount * fee_bps / 10000

# UNSAFE: This can lose precision with small amounts
# @view
# @internal
# def _unsafe_fee_calc(amount: uint256, fee_bps: uint256) -> uint256:
#     fee_rate: uint256 = fee_bps / 10000  # This is 0 for fee_bps < 10000!
#     return amount * fee_rate

# ============================================================
# Reentrancy in Fee Distribution
# ============================================================

distributing: bool

@internal
def _safe_distribute(token: address, to: address, amount: uint256):
    """Protected distribution"""
    assert not self.distributing, "Reentrancy detected"
    self.distributing = True
    
    ERC20(token).transfer(to, amount)
    
    self.distributing = False

# ============================================================
# Fee Cap Enforcement
# ============================================================

MAX_PROTOCOL_FEE: constant(uint256) = 300  # 3% absolute max

@internal
def _validated_fee(fee_bps: uint256) -> uint256:
    """Ensure fee never exceeds maximum"""
    if fee_bps > MAX_PROTOCOL_FEE:
        return MAX_PROTOCOL_FEE
    return fee_bps

# ============================================================
# Front-running Protection for Fee Changes
# ============================================================

# Timelocked fee changes
struct PendingFeeChange:
    new_fee: uint256
    effective_time: uint256

FEE_CHANGE_DELAY: constant(uint256) = 86400 * 2  # 2 days notice

pending_fee_change: public(PendingFeeChange)

@external
def propose_fee_change(new_fee: uint256):
    """Propose a fee change with delay (front-run protection)"""
    # Users can see fee changes coming and plan accordingly
    self.pending_fee_change = PendingFeeChange(
        new_fee=new_fee,
        effective_time=block.timestamp + FEE_CHANGE_DELAY
    )

@external
def apply_fee_change():
    """Apply a pending fee change after delay"""
    assert block.timestamp >= self.pending_fee_change.effective_time, \
        "Change not ready"
    # Apply the fee change
    # ...
```

### สรุปหลักการ Fee Design

| หลักการ | คำอธิบาย |
|---|---|
| Predictability | ผู้ใช้ควรรู้ค่าธรรมเนียมล่วงหน้า |
| Fairness | ไม่คิดเกินไปจาก small users |
| Sustainability | สร้างรายได้พอสำหรับการพัฒนา |
| Transparency | Fee logic ควร open-source |
| Governance | ผู้ถือ token ควบคุม fee changes |

---

## สรุป

Protocol fee mechanisms เป็นส่วนสำคัญของ DeFi protocol:
- **Fee-on-Transfer**: Token ที่ตัด fee ทุก transfer
- **Fee Switch**: Governance-controlled fee enable/disable
- **Fee Distribution**: แจก fees ไปยังผู้มีส่วนได้ส่วนเสีย
- **Dynamic Fees**: ปรับ fee ตาม volatility
- **Security**: ระวัง precision loss, reentrancy, front-running

---
[← Previous Part](part_070_cross_chain_messaging.md) | [→ Next Part](part_072_token_distribution.md)
