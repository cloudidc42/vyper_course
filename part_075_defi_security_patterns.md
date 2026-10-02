# Part 075: DeFi Security Patterns (รูปแบบความปลอดภัย DeFi)

## สารบัญ
1. [บทนำ DeFi Security](#s1)
2. [Price Oracle Security (TWAP vs Spot)](#s2)
3. [Chainlink Circuit Breakers](#s3)
4. [Slippage Protection](#s4)
5. [Front-running Mitigation](#s5)
6. [Sandwich Attack Detection](#s6)
7. [Governance Security](#s7)
8. [SecureDeFiProtocol Contract](#s8)
9. [Security Testing with pytest](#s9)

---

## 1. บทนำ DeFi Security {#s1}

DeFi protocols มีความเสี่ยงเฉพาะตัวที่แตกต่างจาก smart contract ทั่วไป:

| ประเภทการโจมตี | คำอธิบาย | ตัวอย่างที่ถูกโจมตี |
|---|---|---|
| Price Oracle Manipulation | บิดเบือนราคาผ่าน flash loan | bZx (2020) |
| Sandwich Attack | Front-run + back-run trade ของ user | ทุก DEX |
| Flash Loan Governance | กู้ tokens แล้ว vote ทันที | Beanstalk (2022) |
| Reentrancy | เรียกซ้ำระหว่าง execution | The DAO (2016) |
| Liquidity Drain | ดึง liquidity ออกหมด | Multiple rugpulls |

### หลักการ Defense-in-Depth

```
Layer 1: Validation (input sanitization)
Layer 2: Rate limiting (prevent flash loan attacks)
Layer 3: Oracle security (TWAP, multiple sources)
Layer 4: Economic protection (slippage limits)
Layer 5: Governance delays (timelock)
Layer 6: Circuit breakers (pause on anomaly)
```

---

## 2. Price Oracle Security {#s2}

```python
# @version 0.4.0
# TWAPOracle.vy
# Time-Weighted Average Price oracle for manipulation resistance

interface IUniswapV2Pair:
    def getReserves() -> (uint112, uint112, uint32): view
    def price0CumulativeLast() -> uint256: view
    def price1CumulativeLast() -> uint256: view
    def token0() -> address: view
    def token1() -> address: view

# Fixed-point arithmetic using Q112
Q112: constant(uint256) = 2**112

struct Observation:
    timestamp: uint256
    price0_cumulative: uint256
    price1_cumulative: uint256

# ============================================================
# Storage
# ============================================================

owner: public(address)
pair: public(address)
token0: public(address)
token1: public(address)

# TWAP window in seconds (e.g. 1800 = 30 minutes)
period: public(uint256)

# Observations for TWAP calculation
observations: public(DynArray[Observation, 100])
observation_count: public(uint256)

# Cached prices (price * Q112)
price0_average: public(uint256)
price1_average: public(uint256)

last_update_time: public(uint256)

# ============================================================
# Events
# ============================================================

event PriceUpdated:
    price0: uint256
    price1: uint256
    timestamp: uint256

event ManipulationDetected:
    spot_price: uint256
    twap_price: uint256
    deviation: uint256

# ============================================================
# Constructor
# ============================================================

@deploy
def __init__(pair_address: address, twap_period: uint256):
    self.owner = msg.sender
    self.pair = pair_address
    self.period = twap_period
    
    self.token0 = IUniswapV2Pair(pair_address).token0()
    self.token1 = IUniswapV2Pair(pair_address).token1()
    
    # Initialize with current values
    reserve0: uint112 = 0
    reserve1: uint112 = 0
    block_timestamp_last: uint32 = 0
    reserve0, reserve1, block_timestamp_last = IUniswapV2Pair(pair_address).getReserves()
    
    initial_obs: Observation = Observation(
        timestamp=block.timestamp,
        price0_cumulative=IUniswapV2Pair(pair_address).price0CumulativeLast(),
        price1_cumulative=IUniswapV2Pair(pair_address).price1CumulativeLast()
    )
    self.observations.append(initial_obs)
    self.observation_count = 1

# ============================================================
# Price Calculation
# ============================================================

@view
@internal
def _get_current_cumulative_prices() -> (uint256, uint256, uint256):
    """Get current cumulative prices with extrapolation"""
    reserve0: uint112 = 0
    reserve1: uint112 = 0
    block_timestamp_last: uint32 = 0
    reserve0, reserve1, block_timestamp_last = IUniswapV2Pair(self.pair).getReserves()
    
    price0_cumulative: uint256 = IUniswapV2Pair(self.pair).price0CumulativeLast()
    price1_cumulative: uint256 = IUniswapV2Pair(self.pair).price1CumulativeLast()
    
    # If current timestamp is different from last update, extrapolate
    time_elapsed: uint256 = block.timestamp - convert(block_timestamp_last, uint256)
    if time_elapsed > 0 and reserve0 != 0 and reserve1 != 0:
        # price = reserve1/reserve0 * Q112
        price0_cumulative += (convert(reserve1, uint256) * Q112 / convert(reserve0, uint256)) * time_elapsed
        price1_cumulative += (convert(reserve0, uint256) * Q112 / convert(reserve1, uint256)) * time_elapsed
    
    return price0_cumulative, price1_cumulative, block.timestamp

@external
def update():
    """Update TWAP with latest observation"""
    price0_cumulative: uint256 = 0
    price1_cumulative: uint256 = 0
    current_time: uint256 = 0
    price0_cumulative, price1_cumulative, current_time = self._get_current_cumulative_prices()
    
    # Get oldest relevant observation
    oldest_obs: Observation = self.observations[0]
    
    time_elapsed: uint256 = current_time - oldest_obs.timestamp
    
    # Only update if period has passed
    assert time_elapsed >= self.period, "Period not elapsed"
    
    # Calculate TWAP
    self.price0_average = (price0_cumulative - oldest_obs.price0_cumulative) / time_elapsed
    self.price1_average = (price1_cumulative - oldest_obs.price1_cumulative) / time_elapsed
    
    # Store new observation
    new_obs: Observation = Observation(
        timestamp=current_time,
        price0_cumulative=price0_cumulative,
        price1_cumulative=price1_cumulative
    )
    
    if len(self.observations) >= 100:
        # Shift observations (remove oldest)
        for i: uint256 in range(1, 100):
            self.observations[i-1] = self.observations[i]
        self.observations[99] = new_obs
    else:
        self.observations.append(new_obs)
        self.observation_count += 1
    
    self.last_update_time = current_time
    
    log PriceUpdated(self.price0_average, self.price1_average, current_time)

@view
@external
def consult(token: address, amount_in: uint256) -> uint256:
    """
    Get TWAP price for amount_in of token
    Returns amount_out of the other token
    """
    if token == self.token0:
        return amount_in * self.price0_average / Q112
    else:
        assert token == self.token1, "Invalid token"
        return amount_in * self.price1_average / Q112

@view
@external
def is_price_manipulated(token: address, spot_amount_out: uint256, amount_in: uint256) -> bool:
    """
    Check if current spot price deviates significantly from TWAP
    Max 5% deviation allowed
    """
    twap_out: uint256 = 0
    if token == self.token0:
        twap_out = amount_in * self.price0_average / Q112
    else:
        twap_out = amount_in * self.price1_average / Q112
    
    if twap_out == 0:
        return False
    
    # Check for > 5% deviation
    if spot_amount_out > twap_out:
        deviation: uint256 = (spot_amount_out - twap_out) * 10000 / twap_out
        return deviation > 500  # > 5%
    else:
        deviation: uint256 = (twap_out - spot_amount_out) * 10000 / twap_out
        return deviation > 500  # > 5%
```

---

## 3. Chainlink Circuit Breakers {#s3}

```python
# @version 0.4.0
# ChainlinkOracleWithCircuitBreaker.vy
# Price oracle with Chainlink + circuit breakers to prevent stale/extreme prices

interface AggregatorV3Interface:
    def latestRoundData() -> (uint80, int256, uint256, uint256, uint80): view
    def decimals() -> uint8: view

# ============================================================
# Constants
# ============================================================

# Maximum allowed price deviation per update (10%)
MAX_PRICE_DEVIATION: constant(uint256) = 1000  # basis points

# Maximum staleness in seconds
MAX_STALENESS: constant(uint256) = 3600  # 1 hour

# Minimum price (prevent near-zero manipulation)
MIN_PRICE: constant(uint256) = 1  # $0.000001

# ============================================================
# Storage
# ============================================================

owner: public(address)
chainlink_feed: public(address)
feed_decimals: public(uint8)

# Circuit breaker state
circuit_breaker_open: public(bool)
paused_at: public(uint256)
pause_reason: public(String[100])

# Price bounds (set manually by admin)
min_price_bound: public(uint256)
max_price_bound: public(uint256)

# Last known good price
last_price: public(uint256)
last_price_timestamp: public(uint256)

# ============================================================
# Events
# ============================================================

event PriceReturned:
    price: uint256
    timestamp: uint256
    is_stale: bool

event CircuitBreakerTripped:
    reason: String[100]
    last_price: uint256
    new_price: uint256

event CircuitBreakerReset:
    by: indexed(address)

# ============================================================
# Constructor
# ============================================================

@deploy
def __init__(
    feed: address,
    min_bound: uint256,
    max_bound: uint256
):
    self.owner = msg.sender
    self.chainlink_feed = feed
    self.feed_decimals = AggregatorV3Interface(feed).decimals()
    self.min_price_bound = min_bound
    self.max_price_bound = max_bound

# ============================================================
# Safe Price Fetching
# ============================================================

@view
@external
def get_price() -> (uint256, bool):
    """
    Get latest price with safety checks
    Returns (price, is_stale)
    Reverts if circuit breaker is open
    """
    assert not self.circuit_breaker_open, "Circuit breaker open"
    
    round_id: uint80 = 0
    price: int256 = 0
    started_at: uint256 = 0
    updated_at: uint256 = 0
    answered_in_round: uint80 = 0
    round_id, price, started_at, updated_at, answered_in_round = AggregatorV3Interface(
        self.chainlink_feed
    ).latestRoundData()
    
    # Check for negative price
    assert price > 0, "Invalid negative price"
    
    uint_price: uint256 = convert(price, uint256)
    
    # Check price bounds
    assert uint_price >= self.min_price_bound, "Price below minimum bound"
    assert uint_price <= self.max_price_bound, "Price above maximum bound"
    
    # Check for stale data
    is_stale: bool = block.timestamp - updated_at > MAX_STALENESS
    
    # Check round completeness
    assert answered_in_round >= round_id, "Incomplete round"
    
    return uint_price, is_stale

@external
def get_price_safe() -> uint256:
    """
    Get price with full validation - reverts on any issue
    Use this for critical operations
    """
    assert not self.circuit_breaker_open, "Circuit breaker open"
    
    round_id: uint80 = 0
    price: int256 = 0
    started_at: uint256 = 0
    updated_at: uint256 = 0
    answered_in_round: uint80 = 0
    round_id, price, started_at, updated_at, answered_in_round = AggregatorV3Interface(
        self.chainlink_feed
    ).latestRoundData()
    
    assert price > 0, "Invalid price"
    
    uint_price: uint256 = convert(price, uint256)
    
    # Strict staleness check
    assert block.timestamp - updated_at <= MAX_STALENESS, "Price stale"
    
    # Bounds check
    assert uint_price >= self.min_price_bound, "Price too low"
    assert uint_price <= self.max_price_bound, "Price too high"
    
    # Deviation check from last known price
    if self.last_price > 0:
        old: uint256 = self.last_price
        new: uint256 = uint_price
        
        deviation: uint256 = 0
        if new > old:
            deviation = (new - old) * 10000 / old
        else:
            deviation = (old - new) * 10000 / old
        
        if deviation > MAX_PRICE_DEVIATION:
            self.circuit_breaker_open = True
            self.paused_at = block.timestamp
            self.pause_reason = "Extreme price deviation"
            log CircuitBreakerTripped("Extreme price deviation", old, new)
            raise "Circuit breaker: extreme deviation"
    
    # Update last known price
    # (In view function this won't work - this is for illustration)
    # In production, separate update function needed
    
    return uint_price

# ============================================================
# Circuit Breaker Management
# ============================================================

@external
def reset_circuit_breaker():
    """Reset circuit breaker after manual review"""
    assert msg.sender == self.owner, "Not owner"
    
    # Require at least 1 hour to pass
    assert block.timestamp >= self.paused_at + 3600, "Too soon to reset"
    
    self.circuit_breaker_open = False
    
    log CircuitBreakerReset(msg.sender)

@external
def update_bounds(min_price: uint256, max_price: uint256):
    """Update price bounds"""
    assert msg.sender == self.owner, "Not owner"
    assert min_price < max_price, "Invalid bounds"
    
    self.min_price_bound = min_price
    self.max_price_bound = max_price
```

---

## 4. Slippage Protection {#s4}

```python
# @version 0.4.0
# SlippageProtection.vy
# Patterns to protect users from excessive slippage

interface IAMM:
    def getAmountOut(
        amount_in: uint256,
        reserve_in: uint256,
        reserve_out: uint256
    ) -> uint256: view
    def getReserves() -> (uint256, uint256, uint256): view
    def swap(amount0_out: uint256, amount1_out: uint256, to: address, data: Bytes[65536]): nonpayable

# ============================================================
# Constants
# ============================================================

MAX_SLIPPAGE_BPS: constant(uint256) = 10000  # 100% = 10000 basis points
DEFAULT_MAX_SLIPPAGE: constant(uint256) = 50  # 0.5% default

# ============================================================
# Storage
# ============================================================

owner: public(address)
amm: public(address)

# Per-user slippage settings
user_max_slippage: public(HashMap[address, uint256])  # basis points

# Global circuit breaker
global_max_slippage: public(uint256)

# ============================================================
# Events
# ============================================================

event SwapExecuted:
    user: indexed(address)
    token_in: indexed(address)
    amount_in: uint256
    amount_out: uint256
    slippage_bps: uint256

event SlippageExceeded:
    user: indexed(address)
    expected: uint256
    actual: uint256

# ============================================================
# Constructor
# ============================================================

@deploy
def __init__(amm_address: address):
    self.owner = msg.sender
    self.amm = amm_address
    self.global_max_slippage = 300  # 3% global max

# ============================================================
# Slippage-Protected Swap
# ============================================================

@view
@external
def get_expected_output(
    reserve_in: uint256,
    reserve_out: uint256,
    amount_in: uint256,
    max_slippage_bps: uint256
) -> (uint256, uint256):
    """
    Calculate expected output and minimum acceptable output
    Returns (expected_out, min_out_with_slippage)
    """
    # Get ideal output
    expected_out: uint256 = IAMM(self.amm).getAmountOut(amount_in, reserve_in, reserve_out)
    
    # Apply slippage tolerance
    min_out: uint256 = expected_out * (10000 - max_slippage_bps) / 10000
    
    return expected_out, min_out

@external
def swap_with_slippage_protection(
    token_in: address,
    token_out: address,
    amount_in: uint256,
    min_amount_out: uint256,
    deadline: uint256
) -> uint256:
    """
    Execute swap with slippage and deadline protection
    """
    # Deadline check (prevents pending transactions from executing at bad times)
    assert block.timestamp <= deadline, "Transaction expired"
    
    # Check user's slippage settings
    user_slippage: uint256 = self.user_max_slippage[msg.sender]
    if user_slippage == 0:
        user_slippage = DEFAULT_MAX_SLIPPAGE
    
    # Get current reserves
    reserve0: uint256 = 0
    reserve1: uint256 = 0
    ts: uint256 = 0
    reserve0, reserve1, ts = IAMM(self.amm).getReserves()
    
    # Calculate expected output
    expected_out: uint256 = IAMM(self.amm).getAmountOut(amount_in, reserve0, reserve1)
    
    # Verify min_amount_out respects global max slippage
    global_min: uint256 = expected_out * (10000 - self.global_max_slippage) / 10000
    assert min_amount_out >= global_min, "Slippage exceeds global max"
    
    # Execute swap (simplified)
    # In production: transfer token_in, call swap, verify output
    
    # Verify actual output meets minimum
    actual_out: uint256 = 0  # would come from swap
    
    # This check happens after the swap
    if actual_out < min_amount_out:
        log SlippageExceeded(msg.sender, min_amount_out, actual_out)
        raise "Slippage too high"
    
    actual_slippage: uint256 = 0
    if expected_out > actual_out:
        actual_slippage = (expected_out - actual_out) * 10000 / expected_out
    
    log SwapExecuted(msg.sender, token_in, amount_in, actual_out, actual_slippage)
    return actual_out

@external
def set_max_slippage(slippage_bps: uint256):
    """Set personal max slippage tolerance"""
    assert slippage_bps <= self.global_max_slippage, "Exceeds global max"
    self.user_max_slippage[msg.sender] = slippage_bps
```

---

## 5. Front-running Mitigation {#s5}

```python
# @version 0.4.0
# FrontRunningProtection.vy
# Techniques to mitigate front-running attacks

# ============================================================
# Commit-Reveal Scheme
# ============================================================
# Pattern: User commits hash of (action, salt) in block N
#          User reveals action in block N+1 or later
#          This prevents front-runners from seeing the action

# ============================================================
# Storage
# ============================================================

owner: public(address)

# Commit-reveal for trade orders
struct TradeCommit:
    commit_hash: bytes32
    commit_block: uint256
    revealed: bool

commitments: public(HashMap[address, TradeCommit])

# Minimum blocks between commit and reveal
MIN_REVEAL_DELAY: constant(uint256) = 1
MAX_REVEAL_DELAY: constant(uint256) = 20

# Anti-sandwich: Per-block trade limits
block_trades: HashMap[uint256, uint256]  # block -> trade count
MAX_TRADES_PER_BLOCK: constant(uint256) = 5

# MEV protection: max price impact per block
block_price_impact: HashMap[uint256, uint256]  # block -> cumulative impact bps
MAX_BLOCK_PRICE_IMPACT: constant(uint256) = 200  # 2% max per block

# ============================================================
# Events
# ============================================================

event TradeCommitted:
    user: indexed(address)
    commit_hash: bytes32
    block_number: uint256

event TradeRevealed:
    user: indexed(address)
    trade_hash: bytes32

event FrontRunAttempted:
    suspicious_address: indexed(address)
    block_number: uint256

# ============================================================
# Constructor
# ============================================================

@deploy
def __init__():
    self.owner = msg.sender

# ============================================================
# Commit-Reveal Trade
# ============================================================

@external
def commit_trade(commit_hash: bytes32):
    """
    Step 1: Commit to a trade without revealing details
    @param commit_hash keccak256(abi.encodePacked(token_in, token_out, amount, min_out, salt))
    """
    self.commitments[msg.sender] = TradeCommit(
        commit_hash=commit_hash,
        commit_block=block.number,
        revealed=False
    )
    
    log TradeCommitted(msg.sender, commit_hash, block.number)

@external
def reveal_and_execute_trade(
    token_in: address,
    token_out: address,
    amount_in: uint256,
    min_amount_out: uint256,
    salt: bytes32
) -> uint256:
    """
    Step 2: Reveal trade parameters and execute
    Must match committed hash
    """
    commitment: TradeCommit = self.commitments[msg.sender]
    
    assert not commitment.revealed, "Already revealed"
    assert block.number >= commitment.commit_block + MIN_REVEAL_DELAY, "Too soon to reveal"
    assert block.number <= commitment.commit_block + MAX_REVEAL_DELAY, "Commitment expired"
    
    # Verify the hash matches
    expected_hash: bytes32 = keccak256(
        concat(
            convert(token_in, bytes20),
            convert(token_out, bytes20),
            convert(amount_in, bytes32),
            convert(min_amount_out, bytes32),
            salt
        )
    )
    
    assert commitment.commit_hash == expected_hash, "Hash mismatch"
    
    # Mark as revealed
    self.commitments[msg.sender].revealed = True
    
    log TradeRevealed(msg.sender, expected_hash)
    
    # Execute the trade (simplified - add actual AMM call here)
    return min_amount_out

# ============================================================
# Block-based Rate Limiting
# ============================================================

@internal
def _check_block_limits():
    """Limit trades per block to prevent MEV exploitation"""
    current_block: uint256 = block.number
    
    self.block_trades[current_block] += 1
    assert self.block_trades[current_block] <= MAX_TRADES_PER_BLOCK, "Too many trades this block"

# ============================================================
# Private Mempool Hint
# ============================================================

# Note: In production, use Flashbots or other private mempool
# to prevent front-running entirely. This is best practice
# alongside on-chain protections.

@view
@external
def generate_private_order_hash(
    token_in: address,
    token_out: address,
    amount: uint256,
    min_out: uint256,
    user_salt: bytes32
) -> bytes32:
    """Generate commitment hash for private order"""
    return keccak256(
        concat(
            convert(token_in, bytes20),
            convert(token_out, bytes20),
            convert(amount, bytes32),
            convert(min_out, bytes32),
            user_salt
        )
    )
```

---

## 6. Sandwich Attack Detection {#s6}

```python
# @version 0.4.0
# SandwichDetector.vy
# Detect and prevent sandwich attacks

# Sandwich attack pattern:
# 1. Attacker front-runs: buy before victim
# 2. Victim's trade executes (at worse price)
# 3. Attacker back-runs: sell after victim

interface IUniswapV2Pair:
    def getReserves() -> (uint256, uint256, uint256): view
    def price0CumulativeLast() -> uint256: view

# ============================================================
# Storage
# ============================================================

owner: public(address)
protected_pool: public(address)

# Track suspicious activity per address
struct AddressStats:
    buy_count: uint256
    sell_count: uint256
    last_buy_block: uint256
    last_sell_block: uint256
    flagged: bool

address_stats: public(HashMap[address, AddressStats])

# Block-level reserve tracking to detect manipulation
struct BlockReserves:
    reserve0_start: uint256
    reserve1_start: uint256
    reserve0_end: uint256
    reserve1_end: uint256
    trade_count: uint256

block_data: HashMap[uint256, BlockReserves]

# Thresholds
SUSPICIOUS_BLOCK_DELTA: constant(uint256) = 500  # 5% reserve change is suspicious
MAX_SAME_BLOCK_TRADES: constant(uint256) = 3

# ============================================================
# Events
# ============================================================

event SandwichDetected:
    block_number: uint256
    suspect: indexed(address)
    front_run_block: uint256
    back_run_block: uint256

event AddressFlagged:
    suspect: indexed(address)
    reason: String[50]

# ============================================================
# Constructor
# ============================================================

@deploy
def __init__(pool: address):
    self.owner = msg.sender
    self.protected_pool = pool

# ============================================================
# Sandwich Detection Logic
# ============================================================

@view
@external
def is_sandwich_attack(
    trader: address,
    is_buy: bool
) -> bool:
    """
    Detect if this trade is part of a sandwich attack pattern
    A sandwich attacker:
    1. Buys in same block as victim (front-run)
    2. Sells in the next block (back-run)
    """
    stats: AddressStats = self.address_stats[trader]
    
    if is_buy:
        # Suspicious if same address buys AND sells in adjacent blocks
        if stats.last_sell_block > 0:
            blocks_between: uint256 = block.number - stats.last_sell_block
            if blocks_between <= 2 and stats.sell_count > 0:
                return True
    else:
        # Sell after recent buy in same or adjacent block = sandwich
        if stats.last_buy_block > 0:
            blocks_between: uint256 = block.number - stats.last_buy_block
            if blocks_between <= 2 and stats.buy_count > 0:
                return True
    
    return False

@external
def record_trade(trader: address, is_buy: bool):
    """Record trade for pattern analysis"""
    stats: AddressStats = self.address_stats[trader]
    
    if is_buy:
        stats.buy_count += 1
        stats.last_buy_block = block.number
        
        # Flag if pattern detected
        if stats.last_sell_block > 0:
            if block.number - stats.last_sell_block <= 2:
                stats.flagged = True
                log AddressFlagged(trader, "Sandwich pattern: buy after recent sell")
    else:
        stats.sell_count += 1
        stats.last_sell_block = block.number
        
        if stats.last_buy_block > 0:
            if block.number - stats.last_buy_block <= 2:
                stats.flagged = True
                log AddressFlagged(trader, "Sandwich pattern: quick sell after buy")
    
    self.address_stats[trader] = stats

@view
@external
def check_reserve_manipulation() -> bool:
    """
    Check if current block has suspicious reserve changes
    Large sudden changes suggest price manipulation
    """
    current_data: BlockReserves = self.block_data[block.number]
    
    if current_data.reserve0_start == 0:
        return False
    
    # Calculate reserve change
    if current_data.reserve0_end > 0:
        delta0: uint256 = 0
        if current_data.reserve0_end > current_data.reserve0_start:
            delta0 = (current_data.reserve0_end - current_data.reserve0_start) * 10000 / current_data.reserve0_start
        else:
            delta0 = (current_data.reserve0_start - current_data.reserve0_end) * 10000 / current_data.reserve0_start
        
        if delta0 > SUSPICIOUS_BLOCK_DELTA:
            return True
    
    return False

@external
def block_flagged_address(suspect: address):
    """Block a flagged address from trading"""
    assert msg.sender == self.owner, "Not owner"
    self.address_stats[suspect].flagged = True

@view
@external
def is_flagged(trader: address) -> bool:
    """Check if address is flagged as suspicious"""
    return self.address_stats[trader].flagged
```

---

## 7. Governance Security {#s7}

```python
# @version 0.4.0
# SecureGovernance.vy
# Governance with flash loan protection and quorum manipulation prevention

# ============================================================
# Vulnerabilities Addressed:
# 1. Flash loan voting (borrow tokens, vote, return)
# 2. Quorum manipulation (buy tokens, reach quorum, sell)
# 3. Governance takeover via token accumulation
# 4. Proposal spam attacks
# ============================================================

from vyper.interfaces import ERC20

interface IVotingToken:
    def balanceOf(owner: address) -> uint256: view
    def getPastVotes(account: address, block_number: uint256) -> uint256: view  # Snapshot voting
    def totalSupply() -> uint256: view

# ============================================================
# Constants
# ============================================================

# Prevent flash loan voting: snapshot at proposal creation
# Users must have tokens BEFORE proposal is created

# Voting delay: blocks before voting starts after proposal
VOTING_DELAY: constant(uint256) = 1  # 1 block

# Voting period
VOTING_PERIOD: constant(uint256) = 40320  # ~1 week at 15s blocks

# Timelock delay (blocks before execution)
TIMELOCK_DELAY: constant(uint256) = 5760  # ~1 day

# Quorum: 4% of total supply
QUORUM_PERCENTAGE: constant(uint256) = 400  # basis points

# Proposal threshold: 1% to create proposals
PROPOSAL_THRESHOLD: constant(uint256) = 100  # basis points

# Max proposals per address to prevent spam
MAX_ACTIVE_PROPOSALS: constant(uint256) = 3

# ============================================================
# Structs
# ============================================================

struct Proposal:
    id: uint256
    proposer: address
    target: address
    calldata: Bytes[4096]
    snapshot_block: uint256  # Block for vote snapshot (prevents flash loan voting)
    vote_start: uint256
    vote_end: uint256
    execution_time: uint256
    for_votes: uint256
    against_votes: uint256
    abstain_votes: uint256
    executed: bool
    cancelled: bool

# ============================================================
# Storage
# ============================================================

owner: address
voting_token: public(address)

proposals: public(HashMap[uint256, Proposal])
proposal_count: public(uint256)

# Vote tracking (user -> proposal_id -> has_voted)
has_voted: public(HashMap[address, HashMap[uint256, bool]])
vote_weight: public(HashMap[address, HashMap[uint256, uint256]])

# Spam prevention
active_proposals_by_address: public(HashMap[address, uint256])

# ============================================================
# Events
# ============================================================

event ProposalCreated:
    id: indexed(uint256)
    proposer: indexed(address)
    snapshot_block: uint256
    vote_start: uint256
    vote_end: uint256

event Voted:
    voter: indexed(address)
    proposal_id: indexed(uint256)
    support: uint8
    weight: uint256

event ProposalExecuted:
    id: indexed(uint256)

event QuorumManipulationAttempt:
    address_suspect: indexed(address)
    proposal_id: indexed(uint256)

# ============================================================
# Constructor
# ============================================================

@deploy
def __init__(token: address):
    self.owner = msg.sender
    self.voting_token = token

# ============================================================
# Anti-Flash-Loan: Snapshot Voting
# ============================================================

@view
@internal
def _get_votes(account: address, block_number: uint256) -> uint256:
    """Get votes at snapshot block - prevents flash loan voting"""
    # Uses historical balance, not current balance
    # Must use ERC20Votes/Snapshot compatible token
    return IVotingToken(self.voting_token).getPastVotes(account, block_number)

@view
@internal
def _get_quorum(snapshot_block: uint256) -> uint256:
    """Calculate quorum based on total supply at snapshot"""
    # Get total supply at snapshot (can approximate with current for simplicity)
    total: uint256 = IVotingToken(self.voting_token).totalSupply()
    return total * QUORUM_PERCENTAGE / 10000

# ============================================================
# Proposal Management
# ============================================================

@external
def propose(
    target: address,
    calldata: Bytes[4096]
) -> uint256:
    """
    Create a governance proposal
    Requires PROPOSAL_THRESHOLD token balance at CURRENT block
    (not borrowed via flash loan - snapshot prevents that)
    """
    # Check proposer has enough tokens
    proposer_votes: uint256 = IVotingToken(self.voting_token).getPastVotes(
        msg.sender,
        block.number - 1  # Use previous block to prevent same-block manipulation
    )
    
    total_supply: uint256 = IVotingToken(self.voting_token).totalSupply()
    threshold: uint256 = total_supply * PROPOSAL_THRESHOLD / 10000
    
    assert proposer_votes >= threshold, "Below proposal threshold"
    
    # Prevent spam
    assert self.active_proposals_by_address[msg.sender] < MAX_ACTIVE_PROPOSALS, "Too many active proposals"
    
    # Create proposal
    proposal_id: uint256 = self.proposal_count + 1
    self.proposal_count = proposal_id
    
    # CRITICAL: Snapshot block is set to current block
    # Voting power is determined at THIS block, not when vote happens
    # This prevents: accumulate tokens -> wait for proposal -> vote -> sell
    snapshot_block: uint256 = block.number
    
    self.proposals[proposal_id] = Proposal(
        id=proposal_id,
        proposer=msg.sender,
        target=target,
        calldata=calldata,
        snapshot_block=snapshot_block,
        vote_start=block.number + VOTING_DELAY,
        vote_end=block.number + VOTING_DELAY + VOTING_PERIOD,
        execution_time=block.number + VOTING_DELAY + VOTING_PERIOD + TIMELOCK_DELAY,
        for_votes=0,
        against_votes=0,
        abstain_votes=0,
        executed=False,
        cancelled=False
    )
    
    self.active_proposals_by_address[msg.sender] += 1
    
    log ProposalCreated(
        proposal_id,
        msg.sender,
        snapshot_block,
        block.number + VOTING_DELAY,
        block.number + VOTING_DELAY + VOTING_PERIOD
    )
    
    return proposal_id

@external
def cast_vote(proposal_id: uint256, support: uint8):
    """
    Cast vote on proposal
    Vote weight is determined at snapshot block (fixed at proposal creation)
    support: 0=Against, 1=For, 2=Abstain
    """
    proposal: Proposal = self.proposals[proposal_id]
    
    assert not proposal.executed, "Already executed"
    assert not proposal.cancelled, "Cancelled"
    assert block.number >= proposal.vote_start, "Voting not started"
    assert block.number <= proposal.vote_end, "Voting ended"
    assert not self.has_voted[msg.sender][proposal_id], "Already voted"
    
    # Get voting weight at snapshot block (prevents flash loan voting)
    # Even if voter just acquired tokens, they won't count
    weight: uint256 = self._get_votes(msg.sender, proposal.snapshot_block)
    assert weight > 0, "No voting power"
    
    # Record vote
    self.has_voted[msg.sender][proposal_id] = True
    self.vote_weight[msg.sender][proposal_id] = weight
    
    if support == 0:
        self.proposals[proposal_id].against_votes += weight
    elif support == 1:
        self.proposals[proposal_id].for_votes += weight
    elif support == 2:
        self.proposals[proposal_id].abstain_votes += weight
    else:
        raise "Invalid support value"
    
    log Voted(msg.sender, proposal_id, support, weight)

@external
def execute(proposal_id: uint256):
    """Execute a passed proposal after timelock"""
    proposal: Proposal = self.proposals[proposal_id]
    
    assert not proposal.executed, "Already executed"
    assert not proposal.cancelled, "Cancelled"
    assert block.number > proposal.vote_end, "Voting still active"
    assert block.number >= proposal.execution_time, "Timelock not expired"
    
    # Check quorum
    quorum: uint256 = self._get_quorum(proposal.snapshot_block)
    total_votes: uint256 = proposal.for_votes + proposal.against_votes + proposal.abstain_votes
    
    assert total_votes >= quorum, "Quorum not reached"
    assert proposal.for_votes > proposal.against_votes, "Proposal defeated"
    
    self.proposals[proposal_id].executed = True
    self.active_proposals_by_address[proposal.proposer] -= 1
    
    # Execute the proposal
    raw_call(proposal.target, proposal.calldata)
    
    log ProposalExecuted(proposal_id)
```

---

## 8. SecureDeFiProtocol Contract {#s8}

```python
# @version 0.4.0
# SecureDeFiProtocol.vy
# Complete DeFi protocol with all security patterns integrated

from vyper.interfaces import ERC20

interface AggregatorV3Interface:
    def latestRoundData() -> (uint80, int256, uint256, uint256, uint80): view

interface IAMM:
    def getReserves() -> (uint256, uint256, uint256): view
    def getAmountOut(amountIn: uint256, reserveIn: uint256, reserveOut: uint256) -> uint256: view

# ============================================================
# Constants
# ============================================================

# Price oracle staleness threshold
MAX_ORACLE_STALENESS: constant(uint256) = 3600  # 1 hour

# Maximum price deviation from TWAP (basis points)
MAX_TWAP_DEVIATION: constant(uint256) = 500  # 5%

# Maximum single-transaction slippage
MAX_SLIPPAGE: constant(uint256) = 200  # 2%

# Minimum time between large operations (anti-flash-loan)
COOLDOWN_PERIOD: constant(uint256) = 1  # 1 block minimum

# Large operation threshold (requires extra checks)
LARGE_OP_THRESHOLD: constant(uint256) = 100_000 * 10**18

# ============================================================
# Storage
# ============================================================

owner: public(address)
paused: public(bool)

# Oracle
chainlink_oracle: public(address)

# TWAP
twap_price: public(uint256)
twap_timestamp: public(uint256)
twap_period: public(uint256)

# AMM for TWAP
reference_amm: public(address)
amm_price_cumulative: public(uint256)
amm_price_timestamp: public(uint256)

# Rate limiting
last_operation_block: public(HashMap[address, uint256])
operation_count_per_block: public(HashMap[uint256, uint256])

# User balances
balances: public(HashMap[address, uint256])
total_deposits: public(uint256)

# Circuit breaker
circuit_breaker_triggered: public(bool)
circuit_breaker_block: public(uint256)

# ============================================================
# Events
# ============================================================

event Deposit:
    user: indexed(address)
    amount: uint256
    shares: uint256

event Withdraw:
    user: indexed(address)
    amount: uint256

event CircuitBreakerTriggered:
    reason: String[100]
    block_number: uint256

event SecurityCheck:
    check_type: String[50]
    passed: bool

# ============================================================
# Constructor
# ============================================================

@deploy
def __init__(oracle: address, amm: address, twap_window: uint256):
    self.owner = msg.sender
    self.chainlink_oracle = oracle
    self.reference_amm = amm
    self.twap_period = twap_window
    self.twap_price = 0
    self.twap_timestamp = 0

# ============================================================
# Security Checks
# ============================================================

@view
@internal
def _get_chainlink_price() -> uint256:
    """Get Chainlink price with freshness validation"""
    round_id: uint80 = 0
    price: int256 = 0
    started_at: uint256 = 0
    updated_at: uint256 = 0
    answered_in_round: uint80 = 0
    round_id, price, started_at, updated_at, answered_in_round = AggregatorV3Interface(
        self.chainlink_oracle
    ).latestRoundData()
    
    # Validate freshness
    assert block.timestamp - updated_at <= MAX_ORACLE_STALENESS, "Oracle stale"
    
    # Validate round
    assert answered_in_round >= round_id, "Incomplete round"
    
    # Validate non-negative
    assert price > 0, "Invalid price"
    
    return convert(price, uint256)

@internal
def _check_price_not_manipulated(amount: uint256):
    """Compare Chainlink vs AMM price to detect manipulation"""
    if self.twap_price == 0:
        return  # Skip check if no TWAP history yet
    
    # Get current oracle price
    oracle_price: uint256 = self._get_chainlink_price()
    
    # Compare with TWAP
    deviation: uint256 = 0
    if oracle_price > self.twap_price:
        deviation = (oracle_price - self.twap_price) * 10000 / self.twap_price
    else:
        deviation = (self.twap_price - oracle_price) * 10000 / self.twap_price
    
    if deviation > MAX_TWAP_DEVIATION:
        self.circuit_breaker_triggered = True
        self.circuit_breaker_block = block.number
        log CircuitBreakerTriggered("Price deviation from TWAP", block.number)
        raise "Price manipulation detected"

@internal
def _check_rate_limit(user: address, amount: uint256):
    """Rate limiting to prevent flash loan exploitation"""
    # Block-level rate limiting
    current_block: uint256 = block.number
    self.operation_count_per_block[current_block] += 1
    assert self.operation_count_per_block[current_block] <= 10, "Block operation limit"
    
    # Large operation cooldown
    if amount >= LARGE_OP_THRESHOLD:
        assert current_block > self.last_operation_block[user] + COOLDOWN_PERIOD, "Cooldown active"
    
    self.last_operation_block[user] = current_block

@view
@internal
def _calculate_shares(amount: uint256) -> uint256:
    """Calculate deposit shares with manipulation-resistant pricing"""
    if self.total_deposits == 0:
        return amount
    
    # Get price from oracle (manipulation-resistant)
    oracle_price: uint256 = self._get_chainlink_price()
    
    # Shares proportional to deposited value vs total value
    # total_value = total_deposits * oracle_price
    # new_shares = amount * oracle_price / (total_deposits * oracle_price) * total_shares
    # Simplified: shares = amount / total_deposits * existing_shares
    return amount * (10**18) / self.total_deposits

# ============================================================
# Core Protocol Functions
# ============================================================

@external
def deposit(token: address, amount: uint256, min_shares: uint256) -> uint256:
    """
    Deposit with full security checks
    """
    # Basic validation
    assert not self.paused, "Protocol paused"
    assert not self.circuit_breaker_triggered, "Circuit breaker active"
    assert amount > 0, "Zero amount"
    
    # Security check: Rate limiting
    self._check_rate_limit(msg.sender, amount)
    
    # Security check: Price manipulation
    self._check_price_not_manipulated(amount)
    
    # Transfer tokens
    ERC20(token).transferFrom(msg.sender, self, amount)
    
    # Calculate shares with oracle price (not spot AMM price)
    shares: uint256 = self._calculate_shares(amount)
    
    # Slippage check on shares
    assert shares >= min_shares, "Insufficient shares received"
    
    # Update state
    self.balances[msg.sender] += shares
    self.total_deposits += amount
    
    log Deposit(msg.sender, amount, shares)
    return shares

@external
def withdraw(token: address, shares: uint256, min_amount: uint256) -> uint256:
    """
    Withdraw with security checks and slippage protection
    """
    assert not self.paused, "Protocol paused"
    assert not self.circuit_breaker_triggered, "Circuit breaker active"
    
    # Validate balance
    assert self.balances[msg.sender] >= shares, "Insufficient shares"
    
    # Rate limiting
    self._check_rate_limit(msg.sender, shares)
    
    # Calculate withdrawal amount
    amount: uint256 = shares * self.total_deposits / (10**18)
    
    # Slippage check
    assert amount >= min_amount, "Too much slippage"
    
    # Update state first (CEI pattern)
    self.balances[msg.sender] -= shares
    self.total_deposits -= amount
    
    # Transfer tokens
    ERC20(token).transfer(msg.sender, amount)
    
    log Withdraw(msg.sender, amount)
    return amount

# ============================================================
# TWAP Update
# ============================================================

@external
def update_twap():
    """Update TWAP price from AMM"""
    reserve0: uint256 = 0
    reserve1: uint256 = 0
    ts: uint256 = 0
    reserve0, reserve1, ts = IAMM(self.reference_amm).getReserves()
    
    if reserve0 > 0 and reserve1 > 0:
        # Simplified TWAP: just use current price
        # In production: use cumulative prices
        current_price: uint256 = reserve1 * 10**18 / reserve0
        
        if self.twap_price == 0:
            self.twap_price = current_price
        else:
            # Exponential moving average (simplified TWAP)
            self.twap_price = (self.twap_price * 9 + current_price) / 10
        
        self.twap_timestamp = block.timestamp

# ============================================================
# Emergency Functions
# ============================================================

@external
def pause(reason: String[100]):
    """Emergency pause"""
    assert msg.sender == self.owner, "Not owner"
    self.paused = True
    log CircuitBreakerTriggered(reason, block.number)

@external
def unpause():
    """Resume operations"""
    assert msg.sender == self.owner, "Not owner"
    self.paused = False

@external
def reset_circuit_breaker():
    """Reset circuit breaker after investigation"""
    assert msg.sender == self.owner, "Not owner"
    assert block.number >= self.circuit_breaker_block + 100, "Too soon"
    self.circuit_breaker_triggered = False
```

---

## 9. Security Testing with pytest {#s9}

```python
# tests/test_defi_security.py
import pytest
from ape import accounts, project, chain

@pytest.fixture
def deployer(accounts):
    return accounts[0]

@pytest.fixture
def attacker(accounts):
    return accounts[9]

@pytest.fixture
def mock_oracle(deployer, project):
    """Mock Chainlink oracle"""
    return deployer.deploy(project.MockChainlinkOracle, 2000 * 10**8)  # $2000

@pytest.fixture
def mock_amm(deployer, project):
    """Mock AMM with reserves"""
    amm = deployer.deploy(project.MockAMM)
    # Set reserves: 1 ETH = $2000 USDC
    amm.set_reserves(1000 * 10**18, 2_000_000 * 10**6, sender=deployer)
    return amm

@pytest.fixture
def secure_protocol(deployer, project, mock_oracle, mock_amm):
    return deployer.deploy(
        project.SecureDeFiProtocol,
        mock_oracle.address,
        mock_amm.address,
        1800  # 30 min TWAP
    )

# ============================================================
# Oracle Security Tests
# ============================================================

def test_oracle_staleness_check(secure_protocol, mock_oracle, deployer):
    """Protocol should reject stale oracle data"""
    # Advance time past staleness threshold
    chain.mine(1, timestamp=chain.pending_timestamp + 7200)  # 2 hours
    
    with pytest.raises(Exception, match="Oracle stale"):
        secure_protocol.deposit(
            mock_token.address,
            100 * 10**18,
            0,
            sender=deployer
        )

def test_twap_deviation_detection(secure_protocol, mock_oracle, mock_amm, deployer):
    """Detect large deviations between Chainlink and TWAP"""
    # First setup TWAP
    secure_protocol.update_twap(sender=deployer)
    
    # Now manipulate oracle price by 50%
    mock_oracle.set_price(3000 * 10**8, sender=deployer)  # $3000 vs $2000 TWAP
    
    with pytest.raises(Exception, match="Price manipulation detected"):
        secure_protocol.deposit(
            mock_token.address,
            1000 * 10**18,
            0,
            sender=deployer
        )

# ============================================================
# Flash Loan Protection Tests
# ============================================================

def test_governance_flash_loan_protection(project, deployer, accounts):
    """
    Voting power should be based on snapshot, not current balance
    An attacker cannot borrow tokens to vote
    """
    voter = accounts[1]
    
    # Voter has NO tokens at proposal creation
    gov_token = deployer.deploy(project.MockVotingToken)
    governance = deployer.deploy(project.SecureGovernance, gov_token.address)
    
    # Create proposal (deployer has tokens)
    gov_token.mint(deployer.address, 1_000_000 * 10**18, sender=deployer)
    proposal_id = governance.propose(
        deployer.address,
        b"",
        sender=deployer
    )
    
    # Voter acquires tokens AFTER proposal creation (simulated flash loan)
    chain.mine(1)  # Advance past VOTING_DELAY
    gov_token.mint(voter.address, 500_000 * 10**18, sender=deployer)
    
    # Voter tries to vote - should have 0 weight because snapshot was before
    with pytest.raises(Exception, match="No voting power"):
        governance.cast_vote(proposal_id, 1, sender=voter)

def test_quorum_manipulation_prevention(project, deployer, accounts):
    """Quorum should be based on snapshot supply, not current"""
    pass  # Detailed quorum test

# ============================================================
# Sandwich Attack Tests
# ============================================================

def test_sandwich_detection(project, deployer, accounts):
    """Detect sandwich attack pattern"""
    attacker = accounts[9]
    
    detector = deployer.deploy(project.SandwichDetector, deployer.address)
    
    # Simulate attacker front-run (buy)
    detector.record_trade(attacker.address, True, sender=deployer)  # buy
    
    # Mine a block (victim's trade)
    chain.mine(1)
    
    # Attacker back-run (sell in next block)
    is_sandwich = detector.is_sandwich_attack(attacker.address, False)
    assert is_sandwich == True, "Should detect sandwich pattern"
    
    # Record and check flagging
    detector.record_trade(attacker.address, False, sender=deployer)
    assert detector.is_flagged(attacker.address) == True

# ============================================================
# Rate Limiting Tests
# ============================================================

def test_rate_limiting_large_operations(secure_protocol, mock_token, deployer):
    """Large operations should enforce cooldown period"""
    large_amount = 200_000 * 10**18  # Above LARGE_OP_THRESHOLD
    mock_token.mint(deployer.address, large_amount * 2, sender=deployer)
    mock_token.approve(secure_protocol.address, large_amount * 2, sender=deployer)
    
    # First large operation succeeds
    secure_protocol.deposit(mock_token.address, large_amount, 0, sender=deployer)
    
    # Second large operation in same block should fail
    with pytest.raises(Exception, match="Cooldown active"):
        secure_protocol.deposit(mock_token.address, large_amount, 0, sender=deployer)
    
    # After mining blocks, it should succeed
    chain.mine(2)
    secure_protocol.deposit(mock_token.address, large_amount, 0, sender=deployer)

# ============================================================
# Circuit Breaker Tests
# ============================================================

def test_circuit_breaker_triggers_on_extreme_deviation(
    secure_protocol, mock_oracle, deployer, mock_token, accounts
):
    """Circuit breaker should trigger on extreme price change"""
    user = accounts[1]
    amount = 100 * 10**18
    mock_token.mint(user.address, amount, sender=deployer)
    mock_token.approve(secure_protocol.address, amount, sender=user)
    
    # Setup initial TWAP
    secure_protocol.update_twap(sender=deployer)
    
    # Manipulate price dramatically  
    mock_oracle.set_price(5000 * 10**8, sender=deployer)  # 2.5x jump
    
    # Should trigger circuit breaker
    with pytest.raises(Exception):
        secure_protocol.deposit(mock_token.address, amount, 0, sender=user)
    
    # Protocol should be in circuit breaker state
    # (if it didn't revert entirely, check state)

def test_slippage_protection():
    """
    All swaps should respect slippage limits
    and deadline parameters
    """
    pass  # Test with mock AMM and moving prices

# ============================================================
# Integration Security Test
# ============================================================

def test_full_attack_scenario(project, deployer, accounts, chain):
    """
    Simulate a complete attack scenario:
    1. Attacker tries flash loan governance attack
    2. Attacker tries price oracle manipulation
    3. Attacker tries sandwich attack
    All should be prevented
    """
    attacker = accounts[9]
    
    # Setup protocol
    # ... deploy and configure ...
    
    # Step 1: Flash loan governance attack
    # - Attacker borrows tokens
    # - Tries to vote
    # - Should fail: snapshot-based voting
    
    # Step 2: Price manipulation
    # - Attacker manipulates AMM price via large swap
    # - Tries to exploit the protocol at wrong price
    # - Should fail: TWAP deviation check
    
    # Step 3: Sandwich attack
    # - Attacker front-runs a large deposit
    # - Gets flagged and subsequent trades rejected
    
    pass  # Each sub-test above covers individual components
```

---

## สรุป

DeFi Security Patterns ที่สำคัญ:

1. **Price Oracle Security**
   - ใช้ TWAP แทน spot price สำหรับ operations สำคัญ
   - ตรวจสอบ staleness ของ Chainlink data
   - เปรียบเทียบ Chainlink กับ on-chain TWAP

2. **Anti-Flash-Loan**
   - Snapshot voting: วัด voting power ที่ block ของ proposal
   - Rate limiting: cooldown สำหรับ large operations
   - ไม่ใช้ current balance สำหรับ critical calculations

3. **Slippage Protection**
   - มี `min_amount_out` ทุก swap
   - มี `deadline` ป้องกัน pending transactions
   - Global max slippage limit

4. **Sandwich Attack Detection**
   - ติดตาม buy/sell pattern ต่อ address
   - Flag addresses ที่ทำ rapid buy-sell

5. **Governance Security**
   - Timelock ก่อน execute
   - Snapshot-based voting
   - Proposal threshold เพื่อป้องกัน spam
   - Quorum requirement

6. **Circuit Breakers**
   - หยุดโปรโตคอลเมื่อตรวจพบความผิดปกติ
   - Manual review ก่อน reset
   - Emergency pause function

---
[← Previous Part](part_074_flash_loan_strategies.md) | [→ Course Index](index.md)
