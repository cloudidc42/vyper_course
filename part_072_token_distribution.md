# Part 072: Token Distribution Mechanisms (กลไกการแจก Token)

## สารบัญ
1. [บทนำ Token Distribution](#s1)
2. [Fair Launch Patterns](#s2)
3. [Dutch Auction Token Sale](#s3)
4. [Bonding Curve Token](#s4)
5. [Liquidity Bootstrapping Pool (LBP)](#s5)
6. [IDO Contract](#s6)
7. [Lockdrop Mechanism](#s7)
8. [Security Considerations](#s8)

---

## 1. บทนำ Token Distribution {#s1}

**Token distribution** คือกระบวนการกระจาย tokens ไปยังผู้ใช้งาน
วิธีการกระจายส่งผลต่อ:
- **Decentralization**: ความกระจายตัวของ token holders
- **Price Discovery**: การกำหนดมูลค่าที่ยุติธรรม
- **Community Building**: สร้างชุมชนที่แข็งแกร่ง
- **Fairness**: ทุกคนมีโอกาสเท่ากัน

### ประเภทของ Token Distribution

| วิธี | ลักษณะ | ข้อดี/ข้อเสีย |
|---|---|---|
| Fair Launch | ไม่มี pre-sale | +Fair, -ทุนน้อย |
| Dutch Auction | ราคาลดลงเรื่อยๆ | +Price discovery, -Complex |
| IDO | Initial DEX Offering | +Liquidity, -Sniping |
| LBP | Weighted pool | +Anti-bot, -Complex |
| Bonding Curve | Algorithmic price | +Continuous, -Manipulation |
| Lockdrop | Lock existing assets | +Fair distribution |

---

## 2. Fair Launch Patterns {#s2}

```python
# @version 0.4.0
# FairLaunchToken.vy
# Token with fair launch mechanism
# No VC allocation, no pre-mine, equal opportunity

from vyper.interfaces import ERC20

# ============================================================
# Constants
# ============================================================

TOTAL_SUPPLY: constant(uint256) = 100_000_000 * 10**18  # 100M tokens
MINING_SUPPLY: constant(uint256) = 60_000_000 * 10**18  # 60% to miners
LP_SUPPLY: constant(uint256) = 30_000_000 * 10**18      # 30% to LP
ECOSYSTEM_SUPPLY: constant(uint256) = 10_000_000 * 10**18  # 10% ecosystem

# Mining parameters
BLOCKS_PER_DAY: constant(uint256) = 7200
INITIAL_EMISSION: constant(uint256) = 100 * 10**18  # 100 tokens per block
HALVING_PERIOD: constant(uint256) = BLOCKS_PER_DAY * 365  # 1 year

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

# Mining state
start_block: public(uint256)
total_mined: public(uint256)
miner_balances: public(HashMap[address, uint256])  # Accumulated rewards
miner_deposits: public(HashMap[address, uint256])  # Staked LP tokens

# Pool state
total_staked: public(uint256)
rewards_per_token_stored: public(uint256)
user_reward_per_token_paid: public(HashMap[address, uint256])

# Liquidity provision
lp_token: public(address)
lp_pool: public(address)

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

event Staked:
    user: indexed(address)
    amount: uint256

event Withdrawn:
    user: indexed(address)
    amount: uint256

event RewardPaid:
    user: indexed(address)
    reward: uint256

event Launch:
    block_number: uint256
    initial_emission: uint256

# ============================================================
# Constructor
# ============================================================

@deploy
def __init__(token_name: String[64], token_symbol: String[32]):
    self.name = token_name
    self.symbol = token_symbol
    self.decimals = 18
    self.owner = msg.sender
    
    # No pre-mine - all starts from launch
    self.total_supply = 0
    self.start_block = block.number
    
    log Launch(block.number, INITIAL_EMISSION)

# ============================================================
# Mining Reward Calculation
# ============================================================

@view
@internal
def _emission_at_block(block_num: uint256) -> uint256:
    """
    Calculate emission rate at a given block
    Halves every HALVING_PERIOD blocks
    """
    if block_num < self.start_block:
        return 0
    
    blocks_elapsed: uint256 = block_num - self.start_block
    halvings: uint256 = blocks_elapsed / HALVING_PERIOD
    
    # Halve emission for each halving period
    if halvings >= 10:
        return 0  # After 10 halvings, emission is negligible
    
    emission: uint256 = INITIAL_EMISSION
    for i: uint256 in range(10):
        if i >= halvings:
            break
        emission = emission / 2
    
    return emission

@view
@external
def current_emission() -> uint256:
    """Get current emission rate per block"""
    return self._emission_at_block(block.number)

@view
@internal
def _rewards_per_token() -> uint256:
    """Calculate accumulated rewards per staked token"""
    if self.total_staked == 0:
        return self.rewards_per_token_stored
    
    blocks_since: uint256 = block.number - self.start_block
    emission: uint256 = self._emission_at_block(block.number)
    
    new_rewards: uint256 = emission * blocks_since
    
    return self.rewards_per_token_stored + new_rewards * 10**18 / self.total_staked

@view
@internal
def _earned(account: address) -> uint256:
    """Calculate earned rewards for an account"""
    return (
        self.miner_deposits[account] * 
        (self._rewards_per_token() - self.user_reward_per_token_paid[account]) / 
        10**18 + 
        self.miner_balances[account]
    )

# ============================================================
# Staking Functions
# ============================================================

@external
def stake(amount: uint256):
    """Stake LP tokens to earn mining rewards"""
    assert amount > 0, "Cannot stake 0"
    assert self.lp_token != empty(address), "LP token not set"
    
    # Update rewards first
    self._update_reward(msg.sender)
    
    # Transfer LP tokens
    ERC20(self.lp_token).transferFrom(msg.sender, self, amount)
    
    self.total_staked += amount
    self.miner_deposits[msg.sender] += amount
    
    log Staked(msg.sender, amount)

@external
def withdraw(amount: uint256):
    """Withdraw staked LP tokens"""
    assert amount > 0, "Cannot withdraw 0"
    assert self.miner_deposits[msg.sender] >= amount, "Insufficient stake"
    
    self._update_reward(msg.sender)
    
    self.total_staked -= amount
    self.miner_deposits[msg.sender] -= amount
    
    ERC20(self.lp_token).transfer(msg.sender, amount)
    
    log Withdrawn(msg.sender, amount)

@external
def claim_reward():
    """Claim accumulated mining rewards"""
    self._update_reward(msg.sender)
    
    reward: uint256 = self.miner_balances[msg.sender]
    assert reward > 0, "No rewards"
    
    self.miner_balances[msg.sender] = 0
    
    # Mint reward tokens
    assert self.total_mined + reward <= MINING_SUPPLY, "Mining supply exhausted"
    self.total_mined += reward
    self.total_supply += reward
    self.balances[msg.sender] += reward
    
    log Transfer(empty(address), msg.sender, reward)
    log RewardPaid(msg.sender, reward)

@internal
def _update_reward(account: address):
    """Update reward tracking state"""
    rpt: uint256 = self._rewards_per_token()
    self.rewards_per_token_stored = rpt
    
    if account != empty(address):
        self.miner_balances[account] = self._earned(account)
        self.user_reward_per_token_paid[account] = rpt

# ============================================================
# ERC-20 Functions
# ============================================================

@view
@external
def balanceOf(account: address) -> uint256:
    return self.balances[account]

@external
def transfer(to: address, amount: uint256) -> bool:
    assert to != empty(address), "Transfer to zero"
    assert self.balances[msg.sender] >= amount, "Insufficient balance"
    self.balances[msg.sender] -= amount
    self.balances[to] += amount
    log Transfer(msg.sender, to, amount)
    return True

@external
def transferFrom(from_: address, to: address, amount: uint256) -> bool:
    assert self.balances[from_] >= amount, "Insufficient balance"
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

# Admin
@external
def set_lp_token(token: address):
    assert msg.sender == self.owner, "Not owner"
    self.lp_token = token
```

---

## 3. Dutch Auction Token Sale {#s3}

```python
# @version 0.4.0
# DutchAuction.vy
# Dutch auction token sale - price decreases over time
# Fair price discovery mechanism

from vyper.interfaces import ERC20

# ============================================================
# Structs
# ============================================================

struct AuctionConfig:
    token: address          # Token being sold
    start_price: uint256    # Starting price per token (in ETH)
    end_price: uint256      # Floor price (minimum)
    start_time: uint256     # Auction start timestamp
    end_time: uint256       # Auction end timestamp
    total_tokens: uint256   # Tokens available for sale
    tokens_sold: uint256    # Tokens already committed

struct UserBid:
    committed_eth: uint256
    tokens_purchased: uint256
    claimed: bool

# ============================================================
# Storage
# ============================================================

owner: public(address)
auction: public(AuctionConfig)

# Bids per user
bids: public(HashMap[address, UserBid])
total_eth_committed: public(uint256)

# State
finalized: bool
clearing_price: public(uint256)

# Unsold tokens recipient
unsold_recipient: public(address)

# ============================================================
# Events
# ============================================================

event AuctionCreated:
    token: indexed(address)
    start_price: uint256
    end_price: uint256
    total_tokens: uint256

event BidPlaced:
    bidder: indexed(address)
    eth_amount: uint256
    current_price: uint256

event AuctionFinalized:
    clearing_price: uint256
    total_raised: uint256
    tokens_sold: uint256

event TokensClaimed:
    bidder: indexed(address)
    token_amount: uint256
    eth_refund: uint256

# ============================================================
# Constructor
# ============================================================

@deploy
def __init__(
    token: address,
    start_price: uint256,
    end_price: uint256,
    duration: uint256,
    total_tokens: uint256
):
    assert start_price > end_price, "Start must be > end price"
    assert end_price > 0, "End price must be > 0"
    assert total_tokens > 0, "Must sell some tokens"
    
    self.owner = msg.sender
    self.unsold_recipient = msg.sender
    
    self.auction = AuctionConfig(
        token=token,
        start_price=start_price,
        end_price=end_price,
        start_time=block.timestamp,
        end_time=block.timestamp + duration,
        total_tokens=total_tokens,
        tokens_sold=0
    )
    
    log AuctionCreated(token, start_price, end_price, total_tokens)

# ============================================================
# Price Calculation
# ============================================================

@view
@external
def current_price() -> uint256:
    """
    Calculate current auction price
    Linearly decreases from start_price to end_price over duration
    """
    a: AuctionConfig = self.auction
    
    if block.timestamp <= a.start_time:
        return a.start_price
    
    if block.timestamp >= a.end_time:
        return a.end_price
    
    # Linear interpolation
    elapsed: uint256 = block.timestamp - a.start_time
    duration: uint256 = a.end_time - a.start_time
    price_range: uint256 = a.start_price - a.end_price
    
    price_decrease: uint256 = price_range * elapsed / duration
    return a.start_price - price_decrease

@view
@external
def tokens_for_eth(eth_amount: uint256) -> uint256:
    """Calculate how many tokens you get for given ETH"""
    price: uint256 = self.current_price()
    if price == 0:
        return 0
    return eth_amount * 10**18 / price

# ============================================================
# Auction Functions
# ============================================================

@external
@payable
def bid():
    """
    Place a bid in the Dutch auction
    Commits ETH at current price - gets refund if final price is lower
    """
    a: AuctionConfig = self.auction
    
    assert block.timestamp >= a.start_time, "Auction not started"
    assert block.timestamp < a.end_time, "Auction ended"
    assert not self.finalized, "Auction finalized"
    assert msg.value > 0, "Must send ETH"
    
    current_price: uint256 = self.current_price()
    tokens_wanted: uint256 = msg.value * 10**18 / current_price
    
    # Check token availability
    remaining: uint256 = a.total_tokens - a.tokens_sold
    
    if tokens_wanted > remaining:
        # Cap to remaining tokens, refund excess ETH
        tokens_wanted = remaining
    
    # Update bid
    old_bid: UserBid = self.bids[msg.sender]
    self.bids[msg.sender] = UserBid(
        committed_eth=old_bid.committed_eth + msg.value,
        tokens_purchased=old_bid.tokens_purchased + tokens_wanted,
        claimed=False
    )
    
    self.auction.tokens_sold += tokens_wanted
    self.total_eth_committed += msg.value
    
    log BidPlaced(msg.sender, msg.value, current_price)
    
    # Auto-finalize if all tokens sold
    if self.auction.tokens_sold >= a.total_tokens:
        self._finalize()

@internal
def _finalize():
    """Finalize the auction"""
    if self.finalized:
        return
    
    self.finalized = True
    
    # Clearing price is the current price at finalization
    self.clearing_price = self.current_price()
    
    log AuctionFinalized(
        self.clearing_price,
        self.total_eth_committed,
        self.auction.tokens_sold
    )

@external
def finalize():
    """Manually finalize auction after end time"""
    assert block.timestamp >= self.auction.end_time, "Auction not ended"
    assert not self.finalized, "Already finalized"
    self._finalize()

@external
def claim():
    """
    Claim tokens and ETH refund after finalization
    Gets tokens at clearing price, refunded for any overpayment
    """
    assert self.finalized, "Not finalized"
    
    bid: UserBid = self.bids[msg.sender]
    assert bid.committed_eth > 0, "No bid"
    assert not bid.claimed, "Already claimed"
    
    clearing_price: uint256 = self.clearing_price
    
    # Calculate tokens at clearing price
    tokens_at_clearing: uint256 = bid.committed_eth * 10**18 / clearing_price
    
    # Cap to actual committed tokens (in case of rounding)
    if tokens_at_clearing > bid.tokens_purchased:
        tokens_at_clearing = bid.tokens_purchased
    
    # Calculate ETH refund
    eth_spent: uint256 = tokens_at_clearing * clearing_price / 10**18
    eth_refund: uint256 = 0
    if bid.committed_eth > eth_spent:
        eth_refund = bid.committed_eth - eth_spent
    
    # Mark as claimed
    self.bids[msg.sender] = UserBid(
        committed_eth=bid.committed_eth,
        tokens_purchased=bid.tokens_purchased,
        claimed=True
    )
    
    # Transfer tokens
    ERC20(self.auction.token).transfer(msg.sender, tokens_at_clearing)
    
    # Refund excess ETH
    if eth_refund > 0:
        send(msg.sender, eth_refund)
    
    log TokensClaimed(msg.sender, tokens_at_clearing, eth_refund)

@external
def withdraw_proceeds():
    """Owner withdraws ETH proceeds after auction"""
    assert msg.sender == self.owner, "Not owner"
    assert self.finalized, "Not finalized"
    
    send(msg.sender, self.balance)
```

---

## 4. Bonding Curve Token {#s4}

```python
# @version 0.4.0
# BondingCurveToken.vy
# Token with automatic bonding curve pricing
# Price increases as supply increases

# ============================================================
# Bonding Curve Parameters
# ============================================================

# Linear bonding curve: price = base_price + slope * supply
# Or Bancor-style: uses reserve ratio

# ============================================================
# Storage
# ============================================================

name: public(String[64])
symbol: public(String[32])
decimals: public(uint8)
total_supply: public(uint256)

balances: HashMap[address, uint256]

owner: public(address)

# Bonding curve parameters
base_price: public(uint256)   # Starting price in ETH (wei per token)
slope: public(uint256)        # Price increase per token sold (wei)
reserve: public(uint256)      # ETH held as reserve

# Fees
buy_fee: public(uint256)   # 0.5% = 50 bps
sell_fee: public(uint256)  # 1% = 100 bps
fee_recipient: public(address)

# ============================================================
# Events
# ============================================================

event TokensBought:
    buyer: indexed(address)
    eth_in: uint256
    tokens_out: uint256
    new_price: uint256

event TokensSold:
    seller: indexed(address)
    tokens_in: uint256
    eth_out: uint256
    new_price: uint256

event Transfer:
    sender: indexed(address)
    receiver: indexed(address)
    value: uint256

# ============================================================
# Constructor
# ============================================================

@deploy
def __init__(
    token_name: String[64],
    token_symbol: String[32],
    initial_price: uint256,
    price_slope: uint256
):
    self.name = token_name
    self.symbol = token_symbol
    self.decimals = 18
    self.owner = msg.sender
    self.fee_recipient = msg.sender
    
    # Bonding curve: price increases as more tokens are minted
    self.base_price = initial_price  # Starting price in wei per token
    self.slope = price_slope         # Wei increase per token
    
    self.buy_fee = 50    # 0.5%
    self.sell_fee = 100  # 1%

# ============================================================
# Price Calculation
# ============================================================

@view
@external
def spot_price() -> uint256:
    """Current buy price per token"""
    return self.base_price + self.slope * self.total_supply / 10**18

@view
@external
def buy_price_for(token_amount: uint256) -> uint256:
    """Calculate ETH needed to buy a specific amount of tokens"""
    current_supply: uint256 = self.total_supply
    
    # Integrate price curve: integral from supply to supply+amount
    # For linear: price(x) = base + slope * x
    # Integral = base * amount + slope * (supply + supply+amount) * amount / 2
    
    avg_price: uint256 = (
        (self.base_price + self.slope * current_supply / 10**18) +
        (self.base_price + self.slope * (current_supply + token_amount) / 10**18)
    ) / 2
    
    eth_required: uint256 = avg_price * token_amount / 10**18
    fee: uint256 = eth_required * self.buy_fee / 10000
    
    return eth_required + fee

@view
@external
def sell_price_for(token_amount: uint256) -> uint256:
    """Calculate ETH received for selling tokens"""
    current_supply: uint256 = self.total_supply
    
    if token_amount > current_supply:
        return 0
    
    # Average price between current and new supply
    new_supply: uint256 = current_supply - token_amount
    
    avg_price: uint256 = (
        (self.base_price + self.slope * current_supply / 10**18) +
        (self.base_price + self.slope * new_supply / 10**18)
    ) / 2
    
    eth_return: uint256 = avg_price * token_amount / 10**18
    fee: uint256 = eth_return * self.sell_fee / 10000
    
    return eth_return - fee

# ============================================================
# Buy/Sell Functions
# ============================================================

@external
@payable
def buy(min_tokens_out: uint256):
    """
    Buy tokens by sending ETH
    Price increases after each purchase
    """
    assert msg.value > 0, "Must send ETH"
    
    # Calculate fee
    fee: uint256 = msg.value * self.buy_fee / 10000
    eth_for_tokens: uint256 = msg.value - fee
    
    # Calculate tokens to mint
    current_price: uint256 = self.base_price + self.slope * self.total_supply / 10**18
    tokens_to_mint: uint256 = eth_for_tokens * 10**18 / current_price
    
    # Slippage protection
    assert tokens_to_mint >= min_tokens_out, "Insufficient output"
    
    # Mint tokens
    self.total_supply += tokens_to_mint
    self.balances[msg.sender] += tokens_to_mint
    self.reserve += eth_for_tokens
    
    # Transfer fee
    if fee > 0:
        send(self.fee_recipient, fee)
    
    new_price: uint256 = self.base_price + self.slope * self.total_supply / 10**18
    
    log Transfer(empty(address), msg.sender, tokens_to_mint)
    log TokensBought(msg.sender, msg.value, tokens_to_mint, new_price)

@external
def sell(token_amount: uint256, min_eth_out: uint256):
    """
    Sell tokens back to the bonding curve
    Price decreases after each sale
    """
    assert token_amount > 0, "Must sell some tokens"
    assert self.balances[msg.sender] >= token_amount, "Insufficient balance"
    
    # Calculate ETH return
    current_price: uint256 = self.base_price + self.slope * self.total_supply / 10**18
    eth_return: uint256 = current_price * token_amount / 10**18
    
    # Apply sell fee
    fee: uint256 = eth_return * self.sell_fee / 10000
    eth_after_fee: uint256 = eth_return - fee
    
    assert eth_after_fee >= min_eth_out, "Insufficient output"
    assert self.reserve >= eth_after_fee, "Insufficient reserve"
    
    # Burn tokens
    self.total_supply -= token_amount
    self.balances[msg.sender] -= token_amount
    self.reserve -= eth_after_fee
    
    # Send ETH and fee
    send(msg.sender, eth_after_fee)
    if fee > 0:
        send(self.fee_recipient, fee)
    
    new_price: uint256 = self.base_price + self.slope * self.total_supply / 10**18
    
    log Transfer(msg.sender, empty(address), token_amount)
    log TokensSold(msg.sender, token_amount, eth_after_fee, new_price)

# ERC-20 basics
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
```

---

## 5. Liquidity Bootstrapping Pool (LBP) {#s5}

```python
# @version 0.4.0
# LiquidityBootstrappingPool.vy
# Simplified LBP for fair token distribution
# Weights shift over time from high token/low collateral to balanced

from vyper.interfaces import ERC20

# ============================================================
# Storage
# ============================================================

owner: public(address)

# Pool tokens
project_token: public(address)    # Token being launched
collateral_token: public(address)  # USDC, ETH, etc.

# Pool state
project_balance: public(uint256)
collateral_balance: public(uint256)

# Weight parameters (in basis points, total = 10000)
initial_project_weight: public(uint256)  # e.g., 9600 = 96%
final_project_weight: public(uint256)    # e.g., 5000 = 50%
current_project_weight: public(uint256)

# Timing
start_time: public(uint256)
end_time: public(uint256)

# Fee
swap_fee: public(uint256)

# Status
paused: public(bool)
initialized: bool

# ============================================================
# Events
# ============================================================

event PoolInitialized:
    project_token: indexed(address)
    collateral_token: indexed(address)
    initial_weight: uint256

event Swap:
    buyer: indexed(address)
    collateral_in: uint256
    tokens_out: uint256
    current_weight: uint256

event WeightUpdated:
    new_weight: uint256
    timestamp: uint256

# ============================================================
# Constructor
# ============================================================

@deploy
def __init__(
    project_token_addr: address,
    collateral_token_addr: address,
    init_project_weight: uint256,
    final_project_weight_val: uint256,
    duration: uint256,
    fee_bps: uint256
):
    assert init_project_weight > final_project_weight_val, "Initial > final"
    assert init_project_weight <= 9800, "Max 98% initial weight"
    assert final_project_weight_val >= 2000, "Min 20% final weight"
    assert fee_bps <= 300, "Max 3% fee"
    
    self.owner = msg.sender
    self.project_token = project_token_addr
    self.collateral_token = collateral_token_addr
    
    self.initial_project_weight = init_project_weight
    self.final_project_weight = final_project_weight_val
    self.current_project_weight = init_project_weight
    
    self.start_time = block.timestamp
    self.end_time = block.timestamp + duration
    
    self.swap_fee = fee_bps

# ============================================================
# Weight Calculation
# ============================================================

@view
@external
def get_current_weight() -> uint256:
    """
    Get current project token weight
    Weight decreases linearly from initial to final over duration
    This creates downward price pressure over time - discourages bots
    """
    if block.timestamp <= self.start_time:
        return self.initial_project_weight
    
    if block.timestamp >= self.end_time:
        return self.final_project_weight
    
    elapsed: uint256 = block.timestamp - self.start_time
    duration: uint256 = self.end_time - self.start_time
    
    weight_range: uint256 = self.initial_project_weight - self.final_project_weight
    weight_decrease: uint256 = weight_range * elapsed / duration
    
    return self.initial_project_weight - weight_decrease

# ============================================================
# Spot Price
# ============================================================

@view
@external
def spot_price() -> uint256:
    """
    Calculate spot price of project token in collateral
    Based on current weights and balances
    Price = (collateral_balance / collateral_weight) / (project_balance / project_weight)
    """
    project_weight: uint256 = self.get_current_weight()
    collateral_weight: uint256 = 10000 - project_weight
    
    if self.project_balance == 0 or project_weight == 0:
        return 0
    
    # Balancer-style price: (B_c / W_c) / (B_t / W_t)
    # = (B_c * W_t) / (B_t * W_c)
    return (
        self.collateral_balance * project_weight * 10**18 /
        (self.project_balance * collateral_weight)
    )

@view
@external
def calculate_out_given_in(collateral_in: uint256) -> uint256:
    """
    Calculate project tokens out for collateral in
    Simplified constant weight AMM formula
    """
    project_weight: uint256 = self.get_current_weight()
    collateral_weight: uint256 = 10000 - project_weight
    
    # Fee deduction
    collateral_in_fee: uint256 = collateral_in * (10000 - self.swap_fee) / 10000
    
    # Simplified: use spot price for approximation
    spot: uint256 = self.spot_price()
    if spot == 0:
        return 0
    
    return collateral_in_fee * 10**18 / spot

# ============================================================
# Swap Function
# ============================================================

@external
def buy_tokens(collateral_in: uint256, min_tokens_out: uint256):
    """
    Buy project tokens with collateral
    LBP discourages front-running by constantly decreasing price
    """
    assert not self.paused, "Paused"
    assert block.timestamp >= self.start_time, "Not started"
    assert collateral_in > 0, "Zero input"
    
    # Transfer collateral in
    ERC20(self.collateral_token).transferFrom(msg.sender, self, collateral_in)
    
    # Calculate tokens out
    tokens_out: uint256 = self.calculate_out_given_in(collateral_in)
    assert tokens_out >= min_tokens_out, "Insufficient output"
    assert tokens_out <= self.project_balance, "Insufficient liquidity"
    
    # Update balances
    fee_amount: uint256 = collateral_in * self.swap_fee / 10000
    self.collateral_balance += collateral_in - fee_amount
    self.project_balance -= tokens_out
    
    # Transfer fee to owner
    if fee_amount > 0:
        ERC20(self.collateral_token).transfer(self.owner, fee_amount)
    
    # Transfer tokens to buyer
    ERC20(self.project_token).transfer(msg.sender, tokens_out)
    
    current_weight: uint256 = self.get_current_weight()
    log Swap(msg.sender, collateral_in, tokens_out, current_weight)

@external
def add_liquidity(project_amount: uint256, collateral_amount: uint256):
    """Initialize or add liquidity to the pool"""
    assert msg.sender == self.owner, "Not owner"
    
    ERC20(self.project_token).transferFrom(msg.sender, self, project_amount)
    ERC20(self.collateral_token).transferFrom(msg.sender, self, collateral_amount)
    
    self.project_balance += project_amount
    self.collateral_balance += collateral_amount
    
    if not self.initialized:
        self.initialized = True
        log PoolInitialized(self.project_token, self.collateral_token, self.initial_project_weight)

@external
def withdraw_liquidity():
    """Owner withdraws remaining liquidity after LBP ends"""
    assert msg.sender == self.owner, "Not owner"
    assert block.timestamp >= self.end_time, "LBP not ended"
    
    # Transfer remaining tokens
    if self.project_balance > 0:
        ERC20(self.project_token).transfer(msg.sender, self.project_balance)
        self.project_balance = 0
    
    if self.collateral_balance > 0:
        ERC20(self.collateral_token).transfer(msg.sender, self.collateral_balance)
        self.collateral_balance = 0
```

---

## 6. IDO Contract {#s6}

```python
# @version 0.4.0
# IDOContract.vy
# Initial DEX Offering - Complete Implementation
# Includes whitelist, vesting, refund mechanism

from vyper.interfaces import ERC20

# ============================================================
# Structs
# ============================================================

struct Sale:
    token: address
    token_price: uint256        # Price in payment token (6 decimals for USDC)
    total_tokens: uint256       # Total tokens for sale
    tokens_sold: uint256        # Tokens committed
    start_time: uint256
    end_time: uint256
    min_buy: uint256            # Minimum purchase per user
    max_buy: uint256            # Maximum purchase per user
    hard_cap: uint256           # Maximum to raise
    soft_cap: uint256           # Minimum to raise for success
    total_raised: uint256       # Total payment tokens raised

struct UserAllocation:
    committed: uint256          # Payment tokens committed
    tokens_bought: uint256      # Project tokens allocated
    claimed_tokens: uint256     # Tokens already claimed
    refunded: bool

struct VestingSchedule:
    cliff: uint256              # Cliff period in seconds
    duration: uint256           # Total vesting duration
    initial_unlock: uint256     # Percentage unlocked at TGE (basis points)

# ============================================================
# Storage
# ============================================================

owner: public(address)
sale: public(Sale)
vesting: public(VestingSchedule)

# Whitelisting
whitelist_enabled: public(bool)
whitelisted: public(HashMap[address, bool])
whitelist_cap: public(HashMap[address, uint256])  # Custom cap per user

# User allocations
allocations: public(HashMap[address, UserAllocation])
participant_count: public(uint256)

# Payment token
payment_token: public(address)  # USDC etc.

# Post-sale
finalized: public(bool)
sale_success: public(bool)
token_claim_start: public(uint256)  # When users can start claiming

# ============================================================
# Events
# ============================================================

event SaleCreated:
    token: indexed(address)
    start_time: uint256
    end_time: uint256
    total_tokens: uint256

event Committed:
    user: indexed(address)
    payment_amount: uint256
    tokens_allocated: uint256

event SaleFinalized:
    success: bool
    total_raised: uint256

event TokensClaimed:
    user: indexed(address)
    amount: uint256

event Refunded:
    user: indexed(address)
    amount: uint256

# ============================================================
# Constructor
# ============================================================

@deploy
def __init__(
    project_token: address,
    payment_token_addr: address,
    token_price: uint256,
    total_tokens: uint256,
    start_time: uint256,
    duration: uint256,
    min_purchase: uint256,
    max_purchase: uint256,
    hard_cap: uint256,
    soft_cap: uint256,
    cliff_seconds: uint256,
    vesting_seconds: uint256,
    initial_unlock_bps: uint256
):
    assert project_token != empty(address), "Invalid token"
    assert token_price > 0, "Invalid price"
    assert total_tokens > 0, "Invalid amount"
    assert start_time >= block.timestamp, "Start in past"
    assert hard_cap >= soft_cap, "Hard cap < soft cap"
    assert initial_unlock_bps <= 10000, "Invalid unlock %"
    
    self.owner = msg.sender
    self.payment_token = payment_token_addr
    
    self.sale = Sale(
        token=project_token,
        token_price=token_price,
        total_tokens=total_tokens,
        tokens_sold=0,
        start_time=start_time,
        end_time=start_time + duration,
        min_buy=min_purchase,
        max_buy=max_purchase,
        hard_cap=hard_cap,
        soft_cap=soft_cap,
        total_raised=0
    )
    
    self.vesting = VestingSchedule(
        cliff=cliff_seconds,
        duration=vesting_seconds,
        initial_unlock=initial_unlock_bps
    )
    
    log SaleCreated(project_token, start_time, start_time + duration, total_tokens)

# ============================================================
# Participation Functions
# ============================================================

@external
def commit(payment_amount: uint256):
    """
    Commit payment tokens to participate in IDO
    Tokens are allocated at this stage
    """
    s: Sale = self.sale
    
    # Check sale is active
    assert block.timestamp >= s.start_time, "Sale not started"
    assert block.timestamp < s.end_time, "Sale ended"
    assert not self.finalized, "Already finalized"
    
    # Check whitelist
    if self.whitelist_enabled:
        assert self.whitelisted[msg.sender], "Not whitelisted"
    
    # Check limits
    assert payment_amount >= s.min_buy, "Below minimum"
    
    user_alloc: UserAllocation = self.allocations[msg.sender]
    total_user_payment: uint256 = user_alloc.committed + payment_amount
    
    # Check per-user cap
    user_cap: uint256 = self.whitelist_cap[msg.sender]
    if user_cap == 0:
        user_cap = s.max_buy
    assert total_user_payment <= user_cap, "Exceeds user cap"
    
    # Check hard cap
    assert s.total_raised + payment_amount <= s.hard_cap, "Exceeds hard cap"
    
    # Calculate token allocation
    tokens_allocated: uint256 = payment_amount * 10**18 / s.token_price
    
    # Check token availability
    remaining_tokens: uint256 = s.total_tokens - s.tokens_sold
    assert tokens_allocated <= remaining_tokens, "Insufficient tokens"
    
    # Transfer payment
    ERC20(self.payment_token).transferFrom(msg.sender, self, payment_amount)
    
    # Update state
    if user_alloc.committed == 0:
        self.participant_count += 1
    
    self.allocations[msg.sender] = UserAllocation(
        committed=user_alloc.committed + payment_amount,
        tokens_bought=user_alloc.tokens_bought + tokens_allocated,
        claimed_tokens=0,
        refunded=False
    )
    
    self.sale.total_raised += payment_amount
    self.sale.tokens_sold += tokens_allocated
    
    log Committed(msg.sender, payment_amount, tokens_allocated)

# ============================================================
# Finalization
# ============================================================

@external
def finalize():
    """
    Finalize the sale after it ends
    Determines success/failure based on soft cap
    """
    assert msg.sender == self.owner, "Not owner"
    assert block.timestamp >= self.sale.end_time, "Sale not ended"
    assert not self.finalized, "Already finalized"
    
    self.finalized = True
    
    is_success: bool = self.sale.total_raised >= self.sale.soft_cap
    self.sale_success = is_success
    
    if is_success:
        self.token_claim_start = block.timestamp + self.vesting.cliff
        # Transfer raised funds to owner
        raised: uint256 = self.sale.total_raised
        ERC20(self.payment_token).transfer(self.owner, raised)
    
    log SaleFinalized(is_success, self.sale.total_raised)

# ============================================================
# Claiming
# ============================================================

@view
@external
def claimable_tokens(user: address) -> uint256:
    """Calculate currently claimable tokens"""
    if not self.finalized or not self.sale_success:
        return 0
    
    alloc: UserAllocation = self.allocations[user]
    if alloc.tokens_bought == 0:
        return 0
    
    total_vested: uint256 = self._vested_amount(alloc.tokens_bought)
    return total_vested - alloc.claimed_tokens

@view
@internal
def _vested_amount(total: uint256) -> uint256:
    """Calculate vested tokens based on schedule"""
    if block.timestamp < self.token_claim_start:
        return 0
    
    v: VestingSchedule = self.vesting
    
    # Initial TGE unlock
    tge_amount: uint256 = total * v.initial_unlock / 10000
    
    # After cliff, linear vesting
    time_since_start: uint256 = block.timestamp - self.token_claim_start
    
    if time_since_start >= v.duration:
        return total  # Fully vested
    
    # Linear portion
    linear_total: uint256 = total - tge_amount
    vested_linear: uint256 = linear_total * time_since_start / v.duration
    
    return tge_amount + vested_linear

@external
def claim():
    """Claim vested tokens"""
    assert self.finalized, "Not finalized"
    assert self.sale_success, "Sale failed - use refund"
    assert block.timestamp >= self.token_claim_start, "Cliff not passed"
    
    user: address = msg.sender
    alloc: UserAllocation = self.allocations[user]
    assert alloc.tokens_bought > 0, "Nothing to claim"
    
    claimable: uint256 = self._vested_amount(alloc.tokens_bought) - alloc.claimed_tokens
    assert claimable > 0, "Nothing claimable now"
    
    self.allocations[user].claimed_tokens += claimable
    
    ERC20(self.sale.token).transfer(user, claimable)
    
    log TokensClaimed(user, claimable)

@external
def refund():
    """Refund if sale failed to reach soft cap"""
    assert self.finalized, "Not finalized"
    assert not self.sale_success, "Sale successful - use claim"
    
    user: address = msg.sender
    alloc: UserAllocation = self.allocations[user]
    assert alloc.committed > 0, "Nothing to refund"
    assert not alloc.refunded, "Already refunded"
    
    amount: uint256 = alloc.committed
    self.allocations[user].refunded = True
    
    ERC20(self.payment_token).transfer(user, amount)
    
    log Refunded(user, amount)

# ============================================================
# Admin Functions
# ============================================================

@external
def add_to_whitelist(users: DynArray[address, 100], custom_caps: DynArray[uint256, 100]):
    assert msg.sender == self.owner, "Not owner"
    assert len(users) == len(custom_caps), "Length mismatch"
    
    for i: uint256 in range(100):
        if i >= len(users):
            break
        self.whitelisted[users[i]] = True
        if custom_caps[i] > 0:
            self.whitelist_cap[users[i]] = custom_caps[i]

@external
def enable_whitelist(enabled: bool):
    assert msg.sender == self.owner, "Not owner"
    self.whitelist_enabled = enabled
```

---

## 7. Lockdrop Mechanism {#s7}

```python
# @version 0.4.0
# Lockdrop.vy
# Token distribution via asset locking
# Users lock existing assets to earn new tokens

from vyper.interfaces import ERC20

# ============================================================
# Structs
# ============================================================

struct LockPosition:
    asset: address
    amount: uint256
    lock_duration: uint256  # Seconds
    lock_start: uint256
    points_earned: uint256  # Allocation points
    claimed: bool

# ============================================================
# Storage
# ============================================================

owner: public(address)
reward_token: public(address)

# Lock positions
next_lock_id: public(uint256)
lock_positions: public(HashMap[uint256, LockPosition])
user_lock_ids: public(HashMap[address, DynArray[uint256, 20]])

# Supported assets and their multipliers
supported_assets: public(HashMap[address, bool])
asset_multipliers: public(HashMap[address, uint256])  # Multiplier in bps

# Total points for distribution
total_points: public(uint256)
total_reward_tokens: public(uint256)

# Timing
start_time: public(uint256)
end_time: public(uint256)
claim_start: public(uint256)

# Duration multipliers
# Lock longer = more points multiplier
duration_thresholds: public(DynArray[uint256, 5])  # Seconds
duration_multipliers: public(DynArray[uint256, 5])  # Basis points

# ============================================================
# Events
# ============================================================

event Locked:
    user: indexed(address)
    lock_id: uint256
    asset: indexed(address)
    amount: uint256
    duration: uint256
    points: uint256

event Unlocked:
    user: indexed(address)
    lock_id: uint256
    asset: indexed(address)
    amount: uint256

event RewardClaimed:
    user: indexed(address)
    lock_id: uint256
    reward_amount: uint256

# ============================================================
# Constructor
# ============================================================

@deploy
def __init__(
    reward_token_addr: address,
    total_rewards: uint256,
    lock_start: uint256,
    lock_end: uint256,
    claim_start_time: uint256
):
    self.owner = msg.sender
    self.reward_token = reward_token_addr
    self.total_reward_tokens = total_rewards
    self.start_time = lock_start
    self.end_time = lock_end
    self.claim_start = claim_start_time
    
    # Default duration multipliers
    # 1 week = 1x, 1 month = 2x, 3 months = 4x, 6 months = 8x, 1 year = 16x
    self.duration_thresholds = [
        604800,      # 1 week
        2592000,     # 30 days
        7776000,     # 90 days
        15552000,    # 180 days
        31536000     # 365 days
    ]
    self.duration_multipliers = [
        10000,   # 1x
        20000,   # 2x
        40000,   # 4x
        80000,   # 8x
        160000   # 16x
    ]

# ============================================================
# Lock Functions
# ============================================================

@view
@internal
def _duration_multiplier(duration: uint256) -> uint256:
    """Get multiplier for a given lock duration"""
    result: uint256 = 10000  # Default 1x
    
    for i: uint256 in range(5):
        if i >= len(self.duration_thresholds):
            break
        if duration >= self.duration_thresholds[i]:
            result = self.duration_multipliers[i]
    
    return result

@view
@external
def calculate_points(
    asset: address,
    amount: uint256,
    duration: uint256
) -> uint256:
    """Preview points for a potential lock"""
    asset_mult: uint256 = self.asset_multipliers[asset]
    if asset_mult == 0:
        asset_mult = 10000  # 1x default
    
    duration_mult: uint256 = self._duration_multiplier(duration)
    
    # Points = amount * asset_multiplier * duration_multiplier / 10000^2
    return amount * asset_mult / 10000 * duration_mult / 10000

@external
def lock(asset: address, amount: uint256, duration: uint256) -> uint256:
    """
    Lock an asset to earn reward tokens
    @param asset The token to lock
    @param amount Amount to lock
    @param duration Lock duration in seconds
    @return lock_id
    """
    assert self.supported_assets[asset], "Asset not supported"
    assert block.timestamp >= self.start_time, "Lockdrop not started"
    assert block.timestamp < self.end_time, "Lockdrop ended"
    assert amount > 0, "Zero amount"
    assert duration >= self.duration_thresholds[0], "Duration too short"
    
    # Transfer asset
    ERC20(asset).transferFrom(msg.sender, self, amount)
    
    # Calculate points
    points: uint256 = self.calculate_points(asset, amount, duration)
    
    # Create position
    lock_id: uint256 = self.next_lock_id
    self.next_lock_id += 1
    
    self.lock_positions[lock_id] = LockPosition(
        asset=asset,
        amount=amount,
        lock_duration=duration,
        lock_start=block.timestamp,
        points_earned=points,
        claimed=False
    )
    
    self.user_lock_ids[msg.sender].append(lock_id)
    self.total_points += points
    
    log Locked(msg.sender, lock_id, asset, amount, duration, points)
    
    return lock_id

@external
def unlock(lock_id: uint256):
    """Unlock assets after lock duration"""
    pos: LockPosition = self.lock_positions[lock_id]
    
    assert pos.amount > 0, "Invalid lock"
    assert block.timestamp >= pos.lock_start + pos.lock_duration, "Still locked"
    
    # Return assets
    amount: uint256 = pos.amount
    asset: address = pos.asset
    
    self.lock_positions[lock_id].amount = 0
    
    ERC20(asset).transfer(msg.sender, amount)
    
    log Unlocked(msg.sender, lock_id, asset, amount)

@external
def claim_reward(lock_id: uint256):
    """Claim reward tokens based on lock points"""
    assert block.timestamp >= self.claim_start, "Claiming not started"
    
    pos: LockPosition = self.lock_positions[lock_id]
    assert pos.points_earned > 0, "No points"
    assert not pos.claimed, "Already claimed"
    
    self.lock_positions[lock_id].claimed = True
    
    # Calculate reward based on share of total points
    reward: uint256 = 0
    if self.total_points > 0:
        reward = self.total_reward_tokens * pos.points_earned / self.total_points
    
    assert reward > 0, "No reward"
    
    ERC20(self.reward_token).transfer(msg.sender, reward)
    
    log RewardClaimed(msg.sender, lock_id, reward)

@external
def add_supported_asset(asset: address, multiplier: uint256):
    assert msg.sender == self.owner, "Not owner"
    self.supported_assets[asset] = True
    self.asset_multipliers[asset] = multiplier
```

---

## 8. Security Considerations {#s8}

```python
# @version 0.4.0
# TokenDistributionSecurity.vy
# Security patterns for token distribution

# ============================================================
# 1. Front-Running Protection in IDO
# ============================================================

# Commit-Reveal Scheme for Fair Launch
struct Commitment:
    hash: bytes32
    block_number: uint256
    revealed: bool

commitments: HashMap[address, Commitment]
REVEAL_DELAY: constant(uint256) = 2  # Must wait 2 blocks before reveal

@external
def commit_to_buy(secret: bytes32):
    """First: commit a hash of your purchase amount"""
    # Hash = keccak256(amount || secret)
    self.commitments[msg.sender] = Commitment(
        hash=secret,
        block_number=block.number,
        revealed=False
    )

@external
def reveal_and_buy(amount: uint256, secret: bytes32):
    """Then reveal the actual amount"""
    commit: Commitment = self.commitments[msg.sender]
    
    # Must wait before revealing
    assert block.number >= commit.block_number + REVEAL_DELAY, "Wait to reveal"
    assert not commit.revealed, "Already revealed"
    
    # Verify commitment matches
    expected_hash: bytes32 = keccak256(
        concat(convert(amount, bytes32), secret)
    )
    assert commit.hash == expected_hash, "Invalid reveal"
    
    commit.revealed = True
    # Execute the purchase...

# ============================================================
# 2. Sybil Attack Protection
# ============================================================

# Rate limiting per address
user_last_action: HashMap[address, uint256]
ACTION_COOLDOWN: constant(uint256) = 60  # 1 minute

@internal
def _check_sybil():
    assert block.timestamp >= self.user_last_action[msg.sender] + ACTION_COOLDOWN, \
        "Action cooldown"
    self.user_last_action[msg.sender] = block.timestamp

# ============================================================
# 3. Dutch Auction Safety
# ============================================================

# Minimum price floor to prevent too-low clearing
ABSOLUTE_MIN_PRICE: constant(uint256) = 10**15  # 0.001 ETH

@internal
def _safe_dutch_price(calculated_price: uint256) -> uint256:
    if calculated_price < ABSOLUTE_MIN_PRICE:
        return ABSOLUTE_MIN_PRICE
    return calculated_price

# ============================================================
# 4. IDO Soft Cap Refund Security
# ============================================================

# Funds must be locked until finalization
# Only refundable after confirmed failure
# Prevents malicious owner from keeping funds AND calling failure
```

### การทดสอบ pytest

```python
# tests/test_token_distribution.py
import pytest
from ape import accounts, project, chain
import time

@pytest.fixture
def deployer(accounts):
    return accounts[0]

@pytest.fixture
def dutch_auction(deployer, project):
    token = deployer.deploy(project.MockToken, 1_000_000 * 10**18)
    auction = deployer.deploy(
        project.DutchAuction,
        token.address,
        10**18,       # 1 ETH start price
        10**16,       # 0.01 ETH end price
        86400,        # 1 day duration
        10_000 * 10**18  # 10k tokens
    )
    token.transfer(auction.address, 10_000 * 10**18, sender=deployer)
    return auction

def test_dutch_price_decreases(dutch_auction, chain):
    start_price = dutch_auction.current_price()
    chain.pending_timestamp += 43200  # 12 hours
    chain.mine()
    mid_price = dutch_auction.current_price()
    assert mid_price < start_price

def test_dutch_bid_success(dutch_auction, accounts, deployer):
    user = accounts[1]
    price = dutch_auction.current_price()
    
    dutch_auction.bid(sender=user, value=price)
    
    bid = dutch_auction.bids(user.address)
    assert bid[0] > 0  # committed_eth

def test_lbp_weight_decreases(project, deployer, accounts):
    # LBP starts at 96% project weight and decreases
    payment_token = deployer.deploy(project.MockToken, 10**24)
    project_token = deployer.deploy(project.MockToken, 10**24)
    
    lbp = deployer.deploy(
        project.LiquidityBootstrappingPool,
        project_token.address,
        payment_token.address,
        9600,  # 96% initial
        5000,  # 50% final
        86400 * 7,  # 7 days
        30     # 0.3% fee
    )
    
    initial_weight = lbp.get_current_weight()
    assert initial_weight == 9600

@pytest.fixture
def ido(deployer, project):
    payment = deployer.deploy(project.MockToken, 10**24)
    sale_token = deployer.deploy(project.MockToken, 10**24)
    
    start = chain.pending_timestamp + 100
    
    ido_contract = deployer.deploy(
        project.IDOContract,
        sale_token.address,
        payment.address,
        10**6,              # 1 USDC per token
        100_000 * 10**18,   # 100k tokens
        start,
        86400,              # 1 day
        100 * 10**6,        # Min 100 USDC
        10_000 * 10**6,     # Max 10k USDC
        1_000_000 * 10**6,  # 1M hard cap
        100_000 * 10**6,    # 100k soft cap
        0,                  # No cliff
        86400 * 30,         # 30 day vesting
        10000               # 100% TGE
    )
    
    sale_token.transfer(ido_contract.address, 100_000 * 10**18, sender=deployer)
    
    return ido_contract, payment, sale_token

def test_ido_commit(ido, accounts, chain):
    ido_contract, payment, _ = ido
    user = accounts[1]
    
    # Approve payment
    payment.transfer(user.address, 1_000 * 10**6, sender=accounts[0])
    payment.approve(ido_contract.address, 1_000 * 10**6, sender=user)
    
    # Wait for start
    chain.pending_timestamp += 200
    chain.mine()
    
    ido_contract.commit(1_000 * 10**6, sender=user)
    alloc = ido_contract.allocations(user.address)
    assert alloc[0] == 1_000 * 10**6  # committed amount
```

---

## สรุป

Token distribution mechanisms หลากหลายสำหรับ DeFi:
- **Fair Launch**: Mining-based, ไม่มี pre-mine
- **Dutch Auction**: Price discovery ที่ยุติธรรม
- **Bonding Curve**: Algorithmic continuous pricing
- **LBP**: Anti-bot distribution ด้วย shifting weights
- **IDO**: Complete sale with vesting and refunds
- **Lockdrop**: Reward users who lock existing assets

---
[← Previous Part](part_071_protocol_fees.md) | [→ Next Part](part_073_yield_aggregators.md)
