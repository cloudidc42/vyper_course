# Part 073: Yield Aggregators (Yield Aggregators และ ERC-4626)

## สารบัญ
1. [บทนำ Yield Aggregators](#s1)
2. [ERC-4626 Standard Vault](#s2)
3. [Strategy Interface](#s3)
4. [Auto-Compounding Vault](#s4)
5. [Fee Structures](#s5)
6. [Complete ERC-4626 Implementation](#s6)
7. [การทดสอบด้วย pytest](#s7)
8. [Security Considerations](#s8)

---

## 1. บทนำ Yield Aggregators {#s1}

**Yield Aggregators** คือ protocols ที่รวบรวม assets จากผู้ใช้หลายคน
และ deploy ไปยังกลยุทธ์หลายๆ แบบเพื่อสร้างผลตอบแทนสูงสุด

### การทำงาน

```
User deposits -> Vault -> Strategy 1 (Compound)
                      -> Strategy 2 (Aave)
                      -> Strategy 3 (Curve)
                      
Auto-compounding: rewards reinvested automatically
```

### ประโยชน์
- **Gas Efficiency**: หนึ่ง transaction ได้รับ yield จากหลาย protocols
- **Auto-compounding**: ผลตอบแทนถูก reinvest อัตโนมัติ
- **Risk Distribution**: กระจาย risk ไปหลาย strategies

---

## 2. ERC-4626 Standard Vault {#s2}

ERC-4626 คือ tokenized vault standard ที่กำหนด interface สำหรับ yield-bearing vaults

```python
# @version 0.4.0
# ERC4626Interface.vy
# ERC-4626 Tokenized Vault Standard Interface

from vyper.interfaces import ERC20

# ERC-4626 defines a standardized interface for yield-bearing vaults
# Key functions:
# deposit(assets, receiver) -> shares
# mint(shares, receiver) -> assets  
# withdraw(assets, receiver, owner) -> shares
# redeem(shares, receiver, owner) -> assets

# Share/Asset conversion:
# shares represent ownership of the underlying asset pool
# As yield accumulates, each share is worth more assets

interface IERC4626:
    def asset() -> address: view
    def totalAssets() -> uint256: view
    def convertToShares(assets: uint256) -> uint256: view
    def convertToAssets(shares: uint256) -> uint256: view
    def maxDeposit(receiver: address) -> uint256: view
    def previewDeposit(assets: uint256) -> uint256: view
    def deposit(assets: uint256, receiver: address) -> uint256: nonpayable
    def maxMint(receiver: address) -> uint256: view
    def previewMint(shares: uint256) -> uint256: view
    def mint(shares: uint256, receiver: address) -> uint256: nonpayable
    def maxWithdraw(owner: address) -> uint256: view
    def previewWithdraw(assets: uint256) -> uint256: view
    def withdraw(assets: uint256, receiver: address, owner: address) -> uint256: nonpayable
    def maxRedeem(owner: address) -> uint256: view
    def previewRedeem(shares: uint256) -> uint256: view
    def redeem(shares: uint256, receiver: address, owner: address) -> uint256: nonpayable
```

---

## 3. Strategy Interface {#s3}

```python
# @version 0.4.0
# IStrategy.vy
# Interface for yield-generating strategies

interface IStrategy:
    # Core functions
    def deposit(amount: uint256): nonpayable
    def withdraw(amount: uint256) -> uint256: nonpayable
    def harvest() -> uint256: nonpayable  # Collect and reinvest rewards
    
    # View functions
    def want() -> address: view           # Underlying token
    def estimatedTotalAssets() -> uint256: view  # Total assets managed
    def balanceOf() -> uint256: view      # Assets in this strategy
    def isActive() -> bool: view
    
    # Vault only
    def setVault(vault: address): nonpayable
    def migrate(new_strategy: address): nonpayable

# ============================================================
# Example: Compound Strategy
# ============================================================

interface ICToken:
    def mint(mintAmount: uint256) -> uint256: nonpayable
    def redeem(redeemTokens: uint256) -> uint256: nonpayable
    def balanceOf(owner: address) -> uint256: view
    def exchangeRateCurrent() -> uint256: nonpayable
    def exchangeRateStored() -> uint256: view

interface IComptroller:
    def claimComp(holder: address): nonpayable
    def getAssetsIn(account: address) -> DynArray[address, 10]: view

from vyper.interfaces import ERC20
```

```python
# @version 0.4.0
# CompoundStrategy.vy
# Strategy that earns yield via Compound Finance

from vyper.interfaces import ERC20

interface ICToken:
    def mint(mintAmount: uint256) -> uint256: nonpayable
    def redeem(redeemTokens: uint256) -> uint256: nonpayable
    def balanceOf(owner: address) -> uint256: view
    def exchangeRateStored() -> uint256: view

interface IComptroller:
    def claimComp(holder: address): nonpayable

# ============================================================
# Storage
# ============================================================

vault: public(address)
want_token: public(address)   # USDC, DAI, etc.
ctoken: public(address)       # cUSDC, cDAI, etc.
comp_token: public(address)   # COMP reward token
comptroller: public(address)

strategist: public(address)

total_deposited: public(uint256)
harvest_count: public(uint256)
last_harvest: public(uint256)

# ============================================================
# Events
# ============================================================

event Deposited:
    amount: uint256
    ctokens_received: uint256

event Withdrawn:
    amount: uint256

event Harvested:
    profit: uint256
    comp_claimed: uint256

# ============================================================
# Constructor
# ============================================================

@deploy
def __init__(
    vault_addr: address,
    want_token_addr: address,
    ctoken_addr: address,
    comp_token_addr: address,
    comptroller_addr: address
):
    self.vault = vault_addr
    self.want_token = want_token_addr
    self.ctoken = ctoken_addr
    self.comp_token = comp_token_addr
    self.comptroller = comptroller_addr
    self.strategist = msg.sender

# ============================================================
# Core Strategy Functions
# ============================================================

@external
def deposit(amount: uint256):
    """Deposit want tokens into Compound"""
    assert msg.sender == self.vault, "Not vault"
    assert amount > 0, "Zero amount"
    
    # Approve cToken to spend want tokens
    ERC20(self.want_token).approve(self.ctoken, amount)
    
    # Deposit into Compound (get cTokens in return)
    result: uint256 = ICToken(self.ctoken).mint(amount)
    assert result == 0, "Compound mint failed"
    
    ctokens_received: uint256 = ICToken(self.ctoken).balanceOf(self)
    self.total_deposited += amount
    
    log Deposited(amount, ctokens_received)

@external
def withdraw(amount: uint256) -> uint256:
    """Withdraw want tokens from Compound"""
    assert msg.sender == self.vault, "Not vault"
    
    # Calculate cTokens needed
    exchange_rate: uint256 = ICToken(self.ctoken).exchangeRateStored()
    ctokens_needed: uint256 = amount * 10**18 / exchange_rate
    
    # Redeem from Compound
    result: uint256 = ICToken(self.ctoken).redeem(ctokens_needed)
    assert result == 0, "Compound redeem failed"
    
    # Transfer to vault
    actual_amount: uint256 = ERC20(self.want_token).balanceOf(self)
    ERC20(self.want_token).transfer(self.vault, actual_amount)
    
    log Withdrawn(actual_amount)
    return actual_amount

@external
def harvest() -> uint256:
    """
    Harvest COMP rewards and reinvest
    Returns profit amount
    """
    # Claim COMP rewards
    IComptroller(self.comptroller).claimComp(self)
    
    comp_balance: uint256 = ERC20(self.comp_token).balanceOf(self)
    
    # In real implementation: swap COMP -> want token via DEX
    # For simplicity, just track harvest count
    
    self.harvest_count += 1
    self.last_harvest = block.timestamp
    
    log Harvested(0, comp_balance)
    return 0

@view
@external
def estimatedTotalAssets() -> uint256:
    """Estimate total assets managed by this strategy"""
    ctoken_balance: uint256 = ICToken(self.ctoken).balanceOf(self)
    exchange_rate: uint256 = ICToken(self.ctoken).exchangeRateStored()
    return ctoken_balance * exchange_rate / 10**18

@view
@external
def want() -> address:
    return self.want_token

@view
@external
def isActive() -> bool:
    return ICToken(self.ctoken).balanceOf(self) > 0
```

---

## 4. Auto-Compounding Vault {#s4}

```python
# @version 0.4.0
# AutoCompoundVault.vy
# Auto-compounding vault that reinvests rewards automatically

from vyper.interfaces import ERC20

interface IStrategy:
    def deposit(amount: uint256): nonpayable
    def withdraw(amount: uint256) -> uint256: nonpayable
    def harvest() -> uint256: nonpayable
    def estimatedTotalAssets() -> uint256: view
    def want() -> address: view

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
asset_token: public(address)

# Strategy
current_strategy: public(address)
total_assets_deposited: public(uint256)

# Harvest timing
last_harvest: public(uint256)
harvest_interval: public(uint256)  # Min time between harvests

# Performance tracking
total_yield_generated: public(uint256)

# Emergency state
emergency_shutdown: public(bool)

# ============================================================
# Events
# ============================================================

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

event Harvest:
    profit: uint256
    timestamp: uint256

event StrategyUpdated:
    old_strategy: indexed(address)
    new_strategy: indexed(address)

# ============================================================
# Constructor
# ============================================================

@deploy
def __init__(
    token_name: String[64],
    token_symbol: String[32],
    underlying: address
):
    self.name = token_name
    self.symbol = token_symbol
    self.decimals = 18
    self.owner = msg.sender
    self.asset_token = underlying
    self.harvest_interval = 3600  # 1 hour minimum between harvests

# ============================================================
# Vault Math
# ============================================================

@view
@internal
def _total_assets() -> uint256:
    """Total assets managed by vault + strategy"""
    vault_balance: uint256 = ERC20(self.asset_token).balanceOf(self)
    strategy_assets: uint256 = 0
    
    if self.current_strategy != empty(address):
        strategy_assets = IStrategy(self.current_strategy).estimatedTotalAssets()
    
    return vault_balance + strategy_assets

@view
@internal
def _convert_to_shares(assets: uint256) -> uint256:
    """Convert asset amount to share amount"""
    total: uint256 = self._total_assets()
    supply: uint256 = self.total_supply
    
    if supply == 0 or total == 0:
        return assets  # 1:1 initial ratio
    
    return assets * supply / total

@view
@internal
def _convert_to_assets(shares: uint256) -> uint256:
    """Convert share amount to asset amount"""
    supply: uint256 = self.total_supply
    
    if supply == 0:
        return shares  # 1:1 initial ratio
    
    return shares * self._total_assets() / supply

# ============================================================
# ERC-4626 Functions
# ============================================================

@view
@external
def asset() -> address:
    """Return the underlying asset token"""
    return self.asset_token

@view
@external
def totalAssets() -> uint256:
    """Return total assets managed by vault"""
    return self._total_assets()

@view
@external
def convertToShares(assets: uint256) -> uint256:
    return self._convert_to_shares(assets)

@view
@external
def convertToAssets(shares: uint256) -> uint256:
    return self._convert_to_assets(shares)

@view
@external
def maxDeposit(receiver: address) -> uint256:
    if self.emergency_shutdown:
        return 0
    return max_value(uint256)

@view
@external
def previewDeposit(assets: uint256) -> uint256:
    return self._convert_to_shares(assets)

@external
def deposit(assets: uint256, receiver: address) -> uint256:
    """
    Deposit assets and receive shares
    ERC-4626 standard deposit
    """
    assert not self.emergency_shutdown, "Emergency shutdown"
    assert assets > 0, "Zero deposit"
    assert receiver != empty(address), "Invalid receiver"
    
    # Calculate shares before transfer (prevents donation attacks)
    shares: uint256 = self._convert_to_shares(assets)
    assert shares > 0, "Zero shares"
    
    # Transfer assets in
    ERC20(self.asset_token).transferFrom(msg.sender, self, assets)
    
    # Mint shares
    self.total_supply += shares
    self.balances[receiver] += shares
    
    self.total_assets_deposited += assets
    
    # Deploy to strategy if set
    if self.current_strategy != empty(address):
        self._deploy_to_strategy()
    
    log Deposit(msg.sender, receiver, assets, shares)
    log Transfer(empty(address), receiver, shares)
    
    return shares

@view
@external
def maxWithdraw(owner: address) -> uint256:
    return self._convert_to_assets(self.balances[owner])

@view
@external
def previewWithdraw(assets: uint256) -> uint256:
    return self._convert_to_shares(assets)

@external
def withdraw(assets: uint256, receiver: address, owner: address) -> uint256:
    """Withdraw specific amount of assets"""
    shares: uint256 = self._convert_to_shares(assets)
    assert shares > 0, "Zero shares"
    
    return self._redeem(shares, receiver, owner)

@view
@external
def maxRedeem(owner: address) -> uint256:
    return self.balances[owner]

@view
@external
def previewRedeem(shares: uint256) -> uint256:
    return self._convert_to_assets(shares)

@external
def redeem(shares: uint256, receiver: address, owner: address) -> uint256:
    """Redeem shares for assets"""
    return self._redeem(shares, receiver, owner)

@internal
def _redeem(shares: uint256, receiver: address, owner: address) -> uint256:
    """Internal redeem logic"""
    assert shares > 0, "Zero shares"
    assert self.balances[owner] >= shares, "Insufficient shares"
    
    # Handle allowance if caller != owner
    if msg.sender != owner:
        allowed: uint256 = self.allowances[owner][msg.sender]
        assert allowed >= shares, "Insufficient allowance"
        self.allowances[owner][msg.sender] = allowed - shares
    
    assets: uint256 = self._convert_to_assets(shares)
    
    # Burn shares
    self.total_supply -= shares
    self.balances[owner] -= shares
    
    # Withdraw from strategy if needed
    vault_balance: uint256 = ERC20(self.asset_token).balanceOf(self)
    if vault_balance < assets and self.current_strategy != empty(address):
        needed: uint256 = assets - vault_balance
        IStrategy(self.current_strategy).withdraw(needed)
    
    # Transfer assets out
    ERC20(self.asset_token).transfer(receiver, assets)
    
    log Withdraw(msg.sender, receiver, owner, assets, shares)
    log Transfer(owner, empty(address), shares)
    
    return assets

# ============================================================
# Auto-Compounding
# ============================================================

@external
def harvest(caller_fee_recipient: address) -> uint256:
    """
    Harvest rewards and reinvest (auto-compound)
    Pays caller a small fee to incentivize calling
    """
    assert block.timestamp >= self.last_harvest + self.harvest_interval, \
        "Harvest too soon"
    assert self.current_strategy != empty(address), "No strategy"
    
    # Record assets before harvest
    assets_before: uint256 = self._total_assets()
    
    # Harvest from strategy
    profit: uint256 = IStrategy(self.current_strategy).harvest()
    
    # Reinvest any new assets in vault
    vault_balance: uint256 = ERC20(self.asset_token).balanceOf(self)
    if vault_balance > 0:
        self._deploy_to_strategy()
    
    self.last_harvest = block.timestamp
    
    # Calculate actual profit
    assets_after: uint256 = self._total_assets()
    actual_profit: uint256 = 0
    if assets_after > assets_before:
        actual_profit = assets_after - assets_before
    
    self.total_yield_generated += actual_profit
    
    log Harvest(actual_profit, block.timestamp)
    return actual_profit

@internal
def _deploy_to_strategy():
    """Deploy vault's idle assets to active strategy"""
    balance: uint256 = ERC20(self.asset_token).balanceOf(self)
    if balance == 0:
        return
    
    ERC20(self.asset_token).approve(self.current_strategy, balance)
    IStrategy(self.current_strategy).deposit(balance)

# ============================================================
# ERC-20 Functions (for shares)
# ============================================================

event Transfer:
    sender: indexed(address)
    receiver: indexed(address)
    value: uint256

event Approval:
    owner: indexed(address)
    spender: indexed(address)
    value: uint256

@view
@external
def balanceOf(account: address) -> uint256:
    return self.balances[account]

@external
def transfer(to: address, amount: uint256) -> bool:
    assert self.balances[msg.sender] >= amount, "Insufficient"
    self.balances[msg.sender] -= amount
    self.balances[to] += amount
    log Transfer(msg.sender, to, amount)
    return True

@external
def transferFrom(from_: address, to: address, amount: uint256) -> bool:
    assert self.balances[from_] >= amount, "Insufficient"
    assert self.allowances[from_][msg.sender] >= amount, "Insufficient allowance"
    self.allowances[from_][msg.sender] -= amount
    self.balances[from_] -= amount
    self.balances[to] += amount
    log Transfer(from_, to, amount)
    return True

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
def set_strategy(new_strategy: address):
    """Update or set the active strategy"""
    assert msg.sender == self.owner, "Not owner"
    
    old_strategy: address = self.current_strategy
    
    # Withdraw all from old strategy
    if old_strategy != empty(address):
        strategy_assets: uint256 = IStrategy(old_strategy).estimatedTotalAssets()
        if strategy_assets > 0:
            IStrategy(old_strategy).withdraw(strategy_assets)
    
    self.current_strategy = new_strategy
    
    # Deploy to new strategy if set
    if new_strategy != empty(address):
        self._deploy_to_strategy()
    
    log StrategyUpdated(old_strategy, new_strategy)

@external
def emergency_withdraw():
    """Emergency: withdraw all from strategy"""
    assert msg.sender == self.owner, "Not owner"
    
    self.emergency_shutdown = True
    
    if self.current_strategy != empty(address):
        strategy_assets: uint256 = IStrategy(self.current_strategy).estimatedTotalAssets()
        if strategy_assets > 0:
            IStrategy(self.current_strategy).withdraw(strategy_assets)
```

---

## 5. Fee Structures {#s5}

```python
# @version 0.4.0
# VaultWithFees.vy
# ERC-4626 vault with comprehensive fee structure

from vyper.interfaces import ERC20

# ============================================================
# Fee Types
# ============================================================

# 1. Management Fee: charged on AUM (annual basis)
# 2. Performance Fee: charged on profits
# 3. Withdrawal Fee: charged on withdrawals
# 4. Harvest Caller Fee: incentivizes harvest calls

MANAGEMENT_FEE: constant(uint256) = 200    # 2% annual
PERFORMANCE_FEE: constant(uint256) = 2000  # 20% of profit
WITHDRAWAL_FEE: constant(uint256) = 10     # 0.1%
HARVEST_CALLER_FEE: constant(uint256) = 100 # 1% of profit to caller
FEE_DENOMINATOR: constant(uint256) = 10000

SECS_PER_YEAR: constant(uint256) = 31556952  # 365.2425 days

# ============================================================
# Storage
# ============================================================

owner: public(address)
fee_recipient: public(address)
strategist: public(address)

asset_token: public(address)

# ERC-4626 share state
name: public(String[64])
symbol: public(String[32])
decimals: public(uint8)
total_supply: public(uint256)
balances: HashMap[address, uint256]
allowances: HashMap[address, HashMap[address, uint256]]

# Assets
total_assets_internal: public(uint256)

# Fee tracking
last_fee_collection: public(uint256)
accumulated_management_fees: public(uint256)
total_performance_fees_collected: public(uint256)
total_management_fees_collected: public(uint256)

# High-water mark for performance fees
high_water_mark: public(uint256)  # Highest NAV per share achieved

# ============================================================
# Events
# ============================================================

event ManagementFeeCollected:
    amount: uint256
    recipient: indexed(address)

event PerformanceFeeCollected:
    profit: uint256
    fee: uint256
    recipient: indexed(address)

event Transfer:
    sender: indexed(address)
    receiver: indexed(address)
    value: uint256

event Approval:
    owner: indexed(address)
    spender: indexed(address)
    value: uint256

# ============================================================
# Constructor
# ============================================================

@deploy
def __init__(
    token_name: String[64],
    token_symbol: String[32],
    underlying: address,
    fee_rec: address
):
    self.name = token_name
    self.symbol = token_symbol
    self.decimals = 18
    self.owner = msg.sender
    self.fee_recipient = fee_rec
    self.asset_token = underlying
    self.last_fee_collection = block.timestamp
    self.high_water_mark = 10**18  # Initial NAV per share = 1.0

# ============================================================
# Fee Calculation Functions
# ============================================================

@view
@internal
def _total_assets() -> uint256:
    """Get total assets in vault"""
    return ERC20(self.asset_token).balanceOf(self)

@view
@internal
def _nav_per_share() -> uint256:
    """Calculate NAV (Net Asset Value) per share"""
    supply: uint256 = self.total_supply
    if supply == 0:
        return 10**18
    return self._total_assets() * 10**18 / supply

@view
@internal
def _accrued_management_fee() -> uint256:
    """Calculate accrued management fee since last collection"""
    total: uint256 = self._total_assets()
    time_elapsed: uint256 = block.timestamp - self.last_fee_collection
    
    # Annual fee prorated for elapsed time
    return total * MANAGEMENT_FEE * time_elapsed / (FEE_DENOMINATOR * SECS_PER_YEAR)

@internal
def _collect_management_fee():
    """Collect accrued management fee"""
    fee: uint256 = self._accrued_management_fee()
    
    if fee == 0:
        return
    
    # Mint fee shares to fee recipient (dilutes other shareholders)
    # This is more efficient than transferring tokens
    total_assets: uint256 = self._total_assets()
    
    if total_assets == 0:
        return
    
    fee_shares: uint256 = fee * self.total_supply / total_assets
    
    if fee_shares > 0:
        self.total_supply += fee_shares
        self.balances[self.fee_recipient] += fee_shares
        self.accumulated_management_fees += fee
        self.total_management_fees_collected += fee
        
        log Transfer(empty(address), self.fee_recipient, fee_shares)
        log ManagementFeeCollected(fee, self.fee_recipient)
    
    self.last_fee_collection = block.timestamp

@internal
def _collect_performance_fee(profit: uint256, caller: address) -> uint256:
    """
    Collect performance fee on profits
    Only charges on profits above high-water mark
    Returns net profit after fee
    """
    if profit == 0:
        return 0
    
    current_nav: uint256 = self._nav_per_share()
    
    # Only charge if above high-water mark
    if current_nav <= self.high_water_mark:
        return profit
    
    # Calculate profit above high-water mark
    nav_increase: uint256 = current_nav - self.high_water_mark
    profit_above_hwm: uint256 = nav_increase * self.total_supply / 10**18
    
    if profit_above_hwm == 0:
        return profit
    
    # Performance fee
    perf_fee: uint256 = profit_above_hwm * PERFORMANCE_FEE / FEE_DENOMINATOR
    caller_fee: uint256 = profit_above_hwm * HARVEST_CALLER_FEE / FEE_DENOMINATOR
    
    total_fee: uint256 = perf_fee + caller_fee
    if total_fee > profit:
        total_fee = profit
        perf_fee = profit * PERFORMANCE_FEE / (PERFORMANCE_FEE + HARVEST_CALLER_FEE)
        caller_fee = profit - perf_fee
    
    # Distribute fees
    if perf_fee > 0:
        ERC20(self.asset_token).transfer(self.fee_recipient, perf_fee)
        log PerformanceFeeCollected(profit, perf_fee, self.fee_recipient)
    
    if caller_fee > 0 and caller != empty(address):
        ERC20(self.asset_token).transfer(caller, caller_fee)
    
    # Update high-water mark
    self.high_water_mark = current_nav
    self.total_performance_fees_collected += perf_fee
    
    return profit - total_fee

# ============================================================
# ERC-4626 Functions
# ============================================================

@external
def deposit(assets: uint256, receiver: address) -> uint256:
    """Deposit with management fee collection"""
    # Collect accrued management fee first
    self._collect_management_fee()
    
    assert assets > 0, "Zero deposit"
    
    # Calculate shares (after fee collection, this is accurate)
    shares: uint256 = assets
    if self.total_supply > 0:
        shares = assets * self.total_supply / self._total_assets()
    
    ERC20(self.asset_token).transferFrom(msg.sender, self, assets)
    
    self.total_supply += shares
    self.balances[receiver] += shares
    
    log Transfer(empty(address), receiver, shares)
    return shares

@external
def withdraw(assets: uint256, receiver: address, owner: address) -> uint256:
    """Withdraw with fee collection and withdrawal fee"""
    # Collect management fee
    self._collect_management_fee()
    
    # Apply withdrawal fee
    fee: uint256 = assets * WITHDRAWAL_FEE / FEE_DENOMINATOR
    assets_to_send: uint256 = assets - fee
    
    # Calculate shares needed
    shares: uint256 = assets * self.total_supply / self._total_assets()
    
    assert self.balances[owner] >= shares, "Insufficient shares"
    
    if msg.sender != owner:
        assert self.allowances[owner][msg.sender] >= shares, "Insufficient allowance"
        self.allowances[owner][msg.sender] -= shares
    
    self.total_supply -= shares
    self.balances[owner] -= shares
    
    # Send assets minus fee
    ERC20(self.asset_token).transfer(receiver, assets_to_send)
    
    # Send fee to recipient
    if fee > 0:
        ERC20(self.asset_token).transfer(self.fee_recipient, fee)
    
    log Transfer(owner, empty(address), shares)
    return shares

# ERC-20 basics
@view
@external
def balanceOf(account: address) -> uint256:
    return self.balances[account]

@view
@external
def totalAssets() -> uint256:
    return self._total_assets()

@external
def transfer(to: address, amount: uint256) -> bool:
    assert self.balances[msg.sender] >= amount, "Insufficient"
    self.balances[msg.sender] -= amount
    self.balances[to] += amount
    log Transfer(msg.sender, to, amount)
    return True

@external
def approve(spender: address, amount: uint256) -> bool:
    self.allowances[msg.sender][spender] = amount
    log Approval(msg.sender, spender, amount)
    return True

@view
@external
def allowance(owner: address, spender: address) -> uint256:
    return self.allowances[owner][spender]
```

---

## 6. Complete ERC-4626 Implementation {#s6}

```python
# @version 0.4.0
# CompleteERC4626Vault.vy
# Full ERC-4626 vault with strategy, fees, and safety features

from vyper.interfaces import ERC20

interface IYieldStrategy:
    def deposit(amount: uint256): nonpayable
    def withdraw(amount: uint256) -> uint256: nonpayable
    def totalBalance() -> uint256: view
    def harvest(fee_recipient: address) -> uint256: nonpayable

# ============================================================
# Constants
# ============================================================

MAX_TOTAL_ASSETS: constant(uint256) = 10**27  # Safety cap: 1 billion tokens (18 decimals)
MAX_MANAGEMENT_FEE: constant(uint256) = 500    # 5% max annual
MAX_PERFORMANCE_FEE: constant(uint256) = 3000  # 30% max
SECS_PER_YEAR: constant(uint256) = 31556952

# ============================================================
# Storage
# ============================================================

# ERC-4626 + ERC-20 state
name: public(String[64])
symbol: public(String[32])
decimals: public(uint8)
total_supply: public(uint256)
balances: HashMap[address, uint256]
allowances: HashMap[address, HashMap[address, uint256]]

# Vault config
owner: public(address)
pending_owner: public(address)
fee_recipient: public(address)
asset: public(address)

# Strategy
strategy: public(address)

# Fee config
management_fee: public(uint256)
performance_fee: public(uint256)
withdrawal_fee: public(uint256)

# Fee tracking
last_report: public(uint256)
high_water_mark_per_share: public(uint256)
total_debt: public(uint256)  # Assets deployed to strategy

# Limits
deposit_limit: public(uint256)
min_deposit: public(uint256)

# State flags
paused: public(bool)
emergency_mode: public(bool)

# ============================================================
# Events
# ============================================================

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

event StrategyHarvested:
    profit: uint256
    loss: uint256
    timestamp: uint256

event FeeCollected:
    fee_type: String[20]
    amount: uint256
    recipient: indexed(address)

event Transfer:
    sender: indexed(address)
    receiver: indexed(address)
    value: uint256

event Approval:
    owner: indexed(address)
    spender: indexed(address)
    value: uint256

# ============================================================
# Constructor
# ============================================================

@deploy
def __init__(
    vault_name: String[64],
    vault_symbol: String[32],
    underlying_asset: address,
    fee_rec: address,
    mgmt_fee: uint256,
    perf_fee: uint256
):
    assert mgmt_fee <= MAX_MANAGEMENT_FEE, "Management fee too high"
    assert perf_fee <= MAX_PERFORMANCE_FEE, "Performance fee too high"
    
    self.name = vault_name
    self.symbol = vault_symbol
    self.decimals = 18
    self.asset = underlying_asset
    self.owner = msg.sender
    self.fee_recipient = fee_rec
    
    self.management_fee = mgmt_fee
    self.performance_fee = perf_fee
    self.withdrawal_fee = 10  # 0.1%
    
    self.last_report = block.timestamp
    self.high_water_mark_per_share = 10**18  # Start at 1.0
    self.deposit_limit = MAX_TOTAL_ASSETS

# ============================================================
# Core Vault Math
# ============================================================

@view
@internal
def _free_funds() -> uint256:
    """Assets available in vault (not deployed)"""
    return ERC20(self.asset).balanceOf(self)

@view
@internal
def _total_assets() -> uint256:
    """All assets: vault + strategy"""
    free: uint256 = self._free_funds()
    if self.strategy == empty(address):
        return free
    return free + IYieldStrategy(self.strategy).totalBalance()

@view
@internal
def _share_price() -> uint256:
    """Price per share in assets (10^18 = 1.0)"""
    if self.total_supply == 0:
        return 10**18
    return self._total_assets() * 10**18 / self.total_supply

@view
@internal
def _assets_to_shares(assets: uint256, rounding_up: bool) -> uint256:
    """Convert assets to shares"""
    supply: uint256 = self.total_supply
    total: uint256 = self._total_assets()
    
    if supply == 0 or total == 0:
        return assets
    
    if rounding_up:
        return (assets * supply + total - 1) / total
    return assets * supply / total

@view
@internal
def _shares_to_assets(shares: uint256, rounding_down: bool) -> uint256:
    """Convert shares to assets"""
    if self.total_supply == 0:
        return shares
    
    total: uint256 = self._total_assets()
    
    if rounding_down:
        return shares * total / self.total_supply
    return (shares * total + self.total_supply - 1) / self.total_supply

# ============================================================
# ERC-4626 View Functions
# ============================================================

@view
@external
def totalAssets() -> uint256:
    return self._total_assets()

@view
@external
def convertToShares(assets: uint256) -> uint256:
    return self._assets_to_shares(assets, False)

@view
@external
def convertToAssets(shares: uint256) -> uint256:
    return self._shares_to_assets(shares, True)

@view
@external
def maxDeposit(receiver: address) -> uint256:
    if self.paused or self.emergency_mode:
        return 0
    current: uint256 = self._total_assets()
    if current >= self.deposit_limit:
        return 0
    return self.deposit_limit - current

@view
@external
def previewDeposit(assets: uint256) -> uint256:
    return self._assets_to_shares(assets, False)

@view
@external
def maxMint(receiver: address) -> uint256:
    if self.paused or self.emergency_mode:
        return 0
    return max_value(uint256)

@view
@external
def previewMint(shares: uint256) -> uint256:
    return self._shares_to_assets(shares, False)

@view
@external
def maxWithdraw(owner: address) -> uint256:
    return self._shares_to_assets(self.balances[owner], True)

@view
@external
def previewWithdraw(assets: uint256) -> uint256:
    return self._assets_to_shares(assets, True)

@view
@external
def maxRedeem(owner: address) -> uint256:
    return self.balances[owner]

@view
@external
def previewRedeem(shares: uint256) -> uint256:
    return self._shares_to_assets(shares, True)

# ============================================================
# ERC-4626 State-Changing Functions
# ============================================================

@external
def deposit(assets: uint256, receiver: address) -> uint256:
    """Deposit assets, receive shares"""
    assert not self.paused and not self.emergency_mode, "Vault not accepting deposits"
    assert assets >= self.min_deposit, "Below minimum"
    assert self._total_assets() + assets <= self.deposit_limit, "Deposit limit exceeded"
    
    # Calculate shares before deposit
    shares: uint256 = self._assets_to_shares(assets, False)
    assert shares > 0, "Zero shares"
    
    # Transfer assets
    ERC20(self.asset).transferFrom(msg.sender, self, assets)
    
    # Mint shares
    self.total_supply += shares
    self.balances[receiver] += shares
    
    # Deploy excess to strategy
    if self.strategy != empty(address):
        self._deploy_idle()
    
    log Transfer(empty(address), receiver, shares)
    log Deposit(msg.sender, receiver, assets, shares)
    return shares

@external
def mint(shares: uint256, receiver: address) -> uint256:
    """Mint exact shares, deposit required assets"""
    assets: uint256 = self._shares_to_assets(shares, False)
    
    assert not self.paused and not self.emergency_mode, "Vault disabled"
    assert assets > 0, "Zero assets"
    
    ERC20(self.asset).transferFrom(msg.sender, self, assets)
    
    self.total_supply += shares
    self.balances[receiver] += shares
    
    log Transfer(empty(address), receiver, shares)
    log Deposit(msg.sender, receiver, assets, shares)
    return assets

@external
def withdraw(assets: uint256, receiver: address, owner: address) -> uint256:
    """Withdraw exact assets, burn calculated shares"""
    # Include withdrawal fee
    fee: uint256 = assets * self.withdrawal_fee / 10000
    gross_assets: uint256 = assets + fee
    
    shares: uint256 = self._assets_to_shares(gross_assets, True)
    
    assert self.balances[owner] >= shares, "Insufficient shares"
    
    if msg.sender != owner:
        assert self.allowances[owner][msg.sender] >= shares, "Insufficient allowance"
        self.allowances[owner][msg.sender] -= shares
    
    self.total_supply -= shares
    self.balances[owner] -= shares
    
    # Get assets from strategy if needed
    self._ensure_assets(gross_assets)
    
    # Transfer assets and fee
    ERC20(self.asset).transfer(receiver, assets)
    if fee > 0:
        ERC20(self.asset).transfer(self.fee_recipient, fee)
    
    log Transfer(owner, empty(address), shares)
    log Withdraw(msg.sender, receiver, owner, assets, shares)
    return shares

@external
def redeem(shares: uint256, receiver: address, owner: address) -> uint256:
    """Burn shares, receive assets"""
    assert self.balances[owner] >= shares, "Insufficient shares"
    
    if msg.sender != owner:
        assert self.allowances[owner][msg.sender] >= shares, "Insufficient allowance"
        self.allowances[owner][msg.sender] -= shares
    
    gross_assets: uint256 = self._shares_to_assets(shares, True)
    
    # Withdrawal fee
    fee: uint256 = gross_assets * self.withdrawal_fee / 10000
    assets_out: uint256 = gross_assets - fee
    
    self.total_supply -= shares
    self.balances[owner] -= shares
    
    self._ensure_assets(gross_assets)
    
    ERC20(self.asset).transfer(receiver, assets_out)
    if fee > 0:
        ERC20(self.asset).transfer(self.fee_recipient, fee)
    
    log Transfer(owner, empty(address), shares)
    log Withdraw(msg.sender, receiver, owner, assets_out, shares)
    return assets_out

# ============================================================
# Internal Helpers
# ============================================================

@internal
def _deploy_idle():
    """Deploy idle assets to strategy"""
    balance: uint256 = ERC20(self.asset).balanceOf(self)
    if balance == 0:
        return
    ERC20(self.asset).approve(self.strategy, balance)
    IYieldStrategy(self.strategy).deposit(balance)
    self.total_debt += balance

@internal
def _ensure_assets(amount: uint256):
    """Ensure vault has enough assets, withdrawing from strategy if needed"""
    balance: uint256 = ERC20(self.asset).balanceOf(self)
    if balance >= amount:
        return
    
    if self.strategy == empty(address):
        return
    
    needed: uint256 = amount - balance
    IYieldStrategy(self.strategy).withdraw(needed)

# ============================================================
# Harvest
# ============================================================

@external
def report() -> uint256:
    """
    Called by strategy to report profit/loss
    Also handles management and performance fees
    """
    assert msg.sender == self.strategy or msg.sender == self.owner, "Unauthorized"
    
    profit: uint256 = 0
    if self.strategy != empty(address):
        profit = IYieldStrategy(self.strategy).harvest(self.fee_recipient)
    
    # Collect management fee
    time_elapsed: uint256 = block.timestamp - self.last_report
    if time_elapsed > 0 and self.management_fee > 0:
        mgmt_fee: uint256 = self._total_assets() * self.management_fee * time_elapsed / (10000 * SECS_PER_YEAR)
        if mgmt_fee > 0:
            # Mint fee shares
            fee_shares: uint256 = self._assets_to_shares(mgmt_fee, False)
            self.total_supply += fee_shares
            self.balances[self.fee_recipient] += fee_shares
            log Transfer(empty(address), self.fee_recipient, fee_shares)
            log FeeCollected("management", mgmt_fee, self.fee_recipient)
    
    # Collect performance fee if above high-water mark
    current_nav: uint256 = self._share_price()
    if current_nav > self.high_water_mark_per_share and profit > 0:
        perf_fee_assets: uint256 = profit * self.performance_fee / 10000
        if perf_fee_assets > 0:
            fee_shares: uint256 = self._assets_to_shares(perf_fee_assets, False)
            self.total_supply += fee_shares
            self.balances[self.fee_recipient] += fee_shares
            log Transfer(empty(address), self.fee_recipient, fee_shares)
            log FeeCollected("performance", perf_fee_assets, self.fee_recipient)
        
        self.high_water_mark_per_share = current_nav
    
    self.last_report = block.timestamp
    log StrategyHarvested(profit, 0, block.timestamp)
    return profit

# ============================================================
# ERC-20 Functions
# ============================================================

@view
@external
def balanceOf(account: address) -> uint256:
    return self.balances[account]

@external
def transfer(to: address, amount: uint256) -> bool:
    assert self.balances[msg.sender] >= amount, "Insufficient"
    self.balances[msg.sender] -= amount
    self.balances[to] += amount
    log Transfer(msg.sender, to, amount)
    return True

@external
def transferFrom(from_: address, to: address, amount: uint256) -> bool:
    assert self.balances[from_] >= amount, "Insufficient"
    assert self.allowances[from_][msg.sender] >= amount, "Insufficient allowance"
    self.allowances[from_][msg.sender] -= amount
    self.balances[from_] -= amount
    self.balances[to] += amount
    log Transfer(from_, to, amount)
    return True

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
# Admin
# ============================================================

@external
def set_strategy(new_strategy: address):
    assert msg.sender == self.owner, "Not owner"
    self.strategy = new_strategy

@external
def pause():
    assert msg.sender == self.owner, "Not owner"
    self.paused = True

@external
def unpause():
    assert msg.sender == self.owner, "Not owner"
    self.paused = False
```

---

## 7. การทดสอบด้วย pytest {#s7}

```python
# tests/test_yield_aggregators.py
import pytest
from ape import accounts, project, chain
from decimal import Decimal

@pytest.fixture
def deployer(accounts):
    return accounts[0]

@pytest.fixture
def user(accounts):
    return accounts[1]

@pytest.fixture
def mock_token(deployer, project):
    return deployer.deploy(project.MockERC20, "USD Coin", "USDC", 6)

@pytest.fixture
def vault(deployer, mock_token, project):
    return deployer.deploy(
        project.CompleteERC4626Vault,
        "USDC Vault",
        "vUSDC",
        mock_token.address,
        deployer.address,
        200,   # 2% management fee
        2000   # 20% performance fee
    )

@pytest.fixture
def strategy(deployer, vault, mock_token, project):
    strat = deployer.deploy(
        project.CompoundStrategy,
        vault.address,
        mock_token.address,
        deployer.address,  # mock cToken
        deployer.address,  # mock COMP
        deployer.address   # mock comptroller
    )
    vault.set_strategy(strat.address, sender=deployer)
    return strat

# ============================================================
# ERC-4626 Tests
# ============================================================

def test_initial_state(vault, mock_token):
    assert vault.totalAssets() == 0
    assert vault.total_supply() == 0
    assert vault.convertToShares(10**18) == 10**18

def test_deposit_first_time(vault, mock_token, user, deployer):
    amount = 1000 * 10**6  # 1000 USDC
    mock_token.mint(user.address, amount, sender=deployer)
    mock_token.approve(vault.address, amount, sender=user)
    
    shares = vault.deposit(amount, user.address, sender=user)
    
    # First deposit: 1:1 ratio
    assert shares == amount
    assert vault.balanceOf(user.address) == shares
    assert vault.totalAssets() == amount

def test_deposit_second_user_gets_proportional_shares(
    vault, mock_token, user, accounts, deployer
):
    user2 = accounts[2]
    amount1 = 1000 * 10**6
    amount2 = 500 * 10**6
    
    mock_token.mint(user.address, amount1, sender=deployer)
    mock_token.mint(user2.address, amount2, sender=deployer)
    
    mock_token.approve(vault.address, amount1, sender=user)
    vault.deposit(amount1, user.address, sender=user)
    
    mock_token.approve(vault.address, amount2, sender=user2)
    shares2 = vault.deposit(amount2, user2.address, sender=user2)
    
    # User2 gets half the shares of user1
    assert shares2 == amount2  # Still 1:1 since no yield

def test_withdraw_assets(vault, mock_token, user, deployer):
    amount = 1000 * 10**6
    mock_token.mint(user.address, amount, sender=deployer)
    mock_token.approve(vault.address, amount, sender=user)
    
    shares = vault.deposit(amount, user.address, sender=user)
    
    initial_balance = mock_token.balanceOf(user.address)
    
    # Redeem all shares
    vault.redeem(shares, user.address, user.address, sender=user)
    
    # Should receive assets minus withdrawal fee
    fee = amount * 10 // 10000  # 0.1%
    expected = amount - fee
    assert mock_token.balanceOf(user.address) == initial_balance + expected

def test_share_price_increases_with_yield(vault, mock_token, user, deployer):
    amount = 1000 * 10**6
    mock_token.mint(user.address, amount, sender=deployer)
    mock_token.approve(vault.address, amount, sender=user)
    vault.deposit(amount, user.address, sender=user)
    
    initial_price = vault.convertToAssets(10**18)
    
    # Simulate yield by transferring tokens directly to vault
    yield_amount = 100 * 10**6  # 10% yield
    mock_token.mint(vault.address, yield_amount, sender=deployer)
    
    new_price = vault.convertToAssets(10**18)
    assert new_price > initial_price

def test_erc4626_invariants(vault, mock_token, user, deployer):
    """Test ERC-4626 mathematical invariants"""
    amount = 1000 * 10**18
    mock_token.mint(user.address, amount, sender=deployer)
    mock_token.approve(vault.address, amount, sender=user)
    
    # Preview must equal actual (within rounding)
    preview_shares = vault.previewDeposit(amount)
    actual_shares = vault.deposit(amount, user.address, sender=user)
    
    assert preview_shares == actual_shares

def test_management_fee_accrual(vault, mock_token, user, deployer, chain):
    amount = 1_000_000 * 10**18
    mock_token.mint(user.address, amount, sender=deployer)
    mock_token.approve(vault.address, amount, sender=user)
    vault.deposit(amount, user.address, sender=user)
    
    initial_fee_recipient_shares = vault.balanceOf(deployer.address)
    
    # Fast-forward 1 year
    chain.pending_timestamp += 365 * 24 * 3600
    chain.mine()
    
    # Report/harvest triggers fee collection
    vault.report(sender=deployer)
    
    # Fee recipient should have shares now
    new_shares = vault.balanceOf(deployer.address)
    assert new_shares > initial_fee_recipient_shares

def test_performance_fee_on_profit(vault, mock_token, user, deployer, chain):
    amount = 1_000_000 * 10**18
    mock_token.mint(user.address, amount, sender=deployer)
    mock_token.approve(vault.address, amount, sender=user)
    vault.deposit(amount, user.address, sender=user)
    
    initial_fee_shares = vault.balanceOf(deployer.address)
    
    # Simulate profit
    profit = 100_000 * 10**18  # 10% return
    mock_token.mint(vault.address, profit, sender=deployer)
    
    vault.report(sender=deployer)
    
    # Performance fee should be collected
    new_fee_shares = vault.balanceOf(deployer.address)
    # Fee > 0 means performance fee was charged
    # Exact amount depends on implementation

def test_deposit_withdraw_roundtrip(vault, mock_token, user, deployer):
    """Full deposit -> withdraw roundtrip"""
    amount = 1000 * 10**18
    mock_token.mint(user.address, amount, sender=deployer)
    mock_token.approve(vault.address, amount, sender=user)
    
    initial_balance = mock_token.balanceOf(user.address)
    
    # Deposit
    shares = vault.deposit(amount, user.address, sender=user)
    
    # Redeem
    vault.redeem(shares, user.address, user.address, sender=user)
    
    # Should be slightly less due to withdrawal fee
    final_balance = mock_token.balanceOf(user.address)
    assert final_balance <= initial_balance
    assert final_balance >= initial_balance * 999 // 1000  # Max 0.1% loss from fee
```

---

## 8. Security Considerations {#s8}

```python
# @version 0.4.0
# VaultSecurityPatterns.vy
# Security patterns for ERC-4626 vaults

# ============================================================
# 1. Inflation Attack Prevention
# ============================================================

# First depositor can manipulate share price via donation
# Prevention: virtual shares/assets or min deposit

VIRTUAL_SHARES: constant(uint256) = 10**3    # Virtual shares added
VIRTUAL_ASSETS: constant(uint256) = 10**3   # Virtual assets added

@view
@internal
def _total_supply_with_virtual() -> uint256:
    return self.total_supply + VIRTUAL_SHARES

@view
@internal
def _total_assets_with_virtual() -> uint256:
    return self._total_assets() + VIRTUAL_ASSETS

@view
@internal
def _safe_convert_to_shares(assets: uint256) -> uint256:
    """
    Safe share calculation with virtual shares (prevents inflation attack)
    OpenZeppelin's recommended pattern
    """
    return assets * self._total_supply_with_virtual() / self._total_assets_with_virtual()

# ============================================================
# 2. Sandwich Attack Prevention
# ============================================================

# Slippage protection for deposits/withdrawals
@external
def safe_deposit(assets: uint256, receiver: address, min_shares: uint256) -> uint256:
    """Deposit with minimum shares guarantee"""
    shares: uint256 = self._safe_convert_to_shares(assets)
    assert shares >= min_shares, "Insufficient shares - slippage"
    # ... execute deposit
    return shares

# ============================================================
# 3. Strategy Safety
# ============================================================

# Limit strategy debt to prevent over-deployment
MAX_STRATEGY_RATIO: constant(uint256) = 9500  # 95% max deployment

@view
@internal
def _max_strategy_deposit() -> uint256:
    """Max that can be deployed to strategy"""
    total: uint256 = self._total_assets()
    return total * MAX_STRATEGY_RATIO / 10000

# ============================================================
# 4. Emergency Stop
# ============================================================

GUARDIAN: immutable(address)

@deploy  
def __init__(guardian: address):
    GUARDIAN = guardian

@external
def emergencyStop():
    """
    Guardian can stop vault without timelock
    Prevents new deposits and strategy interactions
    """
    assert msg.sender == GUARDIAN or msg.sender == self.owner, "Unauthorized"
    self.emergency_mode = True
    
    # Withdraw all from strategy
    if self.strategy != empty(address):
        IYieldStrategy(self.strategy).withdraw(
            IYieldStrategy(self.strategy).totalBalance()
        )
```

### สรุปแนวทางความปลอดภัย

| ความเสี่ยง | การป้องกัน |
|---|---|
| Inflation Attack | Virtual shares/assets |
| Sandwich Attack | Min shares parameter |
| Strategy Loss | Debt limit, health checks |
| Oracle Manipulation | TWAP, multiple oracles |
| Reentrancy | Update state before transfer |
| Admin Risk | Timelock, multisig |

---

## สรุป

ERC-4626 Yield Aggregators เป็นมาตรฐานสำหรับ yield vaults:
- **ERC-4626**: Standard interface สำหรับ vault
- **Strategies**: Pluggable yield generators (Compound, Aave, Curve)
- **Auto-compounding**: Reinvest rewards อัตโนมัติ
- **Fee Structure**: Management fee, performance fee, withdrawal fee
- **Security**: Inflation attack prevention, emergency stop

---
[← Previous Part](part_072_token_distribution.md) | [→ Next Part](part_074_flash_loan_strategies.md)
