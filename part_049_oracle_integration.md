# Part 049: Oracle Integration in Vyper

## สารบัญ
1. [Overview](#overview)
2. [Chainlink Price Feeds](#chainlink)
3. [TWAP Oracles](#twap)
4. [Oracle Security](#security)
5. [OracleConsumer Contract](#oracle-consumer)
6. [Testing with Mock Oracles](#testing)

---

## 1. Overview {#overview}

Oracle คือ bridge ระหว่าง blockchain กับข้อมูลจากโลกภายนอก เนื่องจาก Smart Contract ไม่สามารถเข้าถึงข้อมูลจากภายนอก blockchain ได้โดยตรง

### ประเภทของ Oracles

- **Price Oracles**: ราคา ETH/USD, BTC/USD, Token prices
- **Random Number Oracles**: VRF (Verifiable Random Function)
- **Event Oracles**: ผลการแข่งขัน, สภาพอากาศ
- **Cross-chain Oracles**: ข้อมูลจาก chain อื่น

### Oracle Providers

- **Chainlink**: Oracle network ที่ใหญ่และน่าเชื่อถือที่สุด
- **Band Protocol**: Multi-chain oracle
- **Pyth Network**: Low-latency oracle สำหรับ DeFi
- **UMA**: Optimistic Oracle
- **Tellor**: Decentralized oracle

### Oracle Attack Vectors

1. **Price Manipulation**: Flash loan attacks on DEX-based oracles
2. **Stale Data**: ข้อมูลที่ไม่ได้ update นาน
3. **Circuit Breakers**: ราคาเปลี่ยนแปลงกะทันหัน
4. **Front-running**: ใช้ข้อมูลที่จะ update

---

## 2. Chainlink Price Feeds {#chainlink}

### Chainlink Price Feed Interface

```vyper
# @version 0.4.0
# interfaces/AggregatorV3.vy
# @title Chainlink AggregatorV3 Interface

interface AggregatorV3Interface:
    def decimals() -> uint8: view
    def description() -> String[100]: view
    def version() -> uint256: view
    def getRoundData(
        _roundId: uint80
    ) -> (uint80, int256, uint256, uint256, uint80): view
    def latestRoundData() -> (uint80, int256, uint256, uint256, uint80): view
```

### Basic Price Feed Usage

```vyper
# @version 0.4.0
# @title PriceFeedReader
# @notice อ่านราคาจาก Chainlink Price Feed

interface AggregatorV3Interface:
    def latestRoundData() -> (uint80, int256, uint256, uint256, uint80): view
    def decimals() -> uint8: view
    def description() -> String[100]: view

# Chainlink Price Feed addresses (Mainnet)
# ETH/USD: 0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419
# BTC/USD: 0xF4030086522a5bEEa4988F8cA5B36dbC97BeE88c
# LINK/USD: 0x2c1d072e956AFFC0D435Cb7AC308d97936Ed4a3f

ETH_USD_FEED: immutable(address)
BTC_USD_FEED: immutable(address)

@deploy
def __init__(eth_usd: address, btc_usd: address):
    ETH_USD_FEED = eth_usd
    BTC_USD_FEED = btc_usd

@external
@view
def get_eth_price() -> int256:
    """Get ETH price in USD (8 decimals)"""
    round_id: uint80 = 0
    price: int256 = 0
    started_at: uint256 = 0
    updated_at: uint256 = 0
    answered_in_round: uint80 = 0
    
    (round_id, price, started_at, updated_at, answered_in_round) = \
        AggregatorV3Interface(ETH_USD_FEED).latestRoundData()
    
    assert price > 0, "Invalid price"
    return price

@external
@view
def get_eth_price_with_decimals() -> (int256, uint8):
    """Get ETH price with decimal information"""
    round_id: uint80 = 0
    price: int256 = 0
    started_at: uint256 = 0
    updated_at: uint256 = 0
    answered_in_round: uint80 = 0
    
    (round_id, price, started_at, updated_at, answered_in_round) = \
        AggregatorV3Interface(ETH_USD_FEED).latestRoundData()
    
    decimals: uint8 = AggregatorV3Interface(ETH_USD_FEED).decimals()
    
    return price, decimals

@external
@view
def convert_eth_to_usd(eth_amount: uint256) -> uint256:
    """แปลง ETH amount เป็น USD value"""
    round_id: uint80 = 0
    price: int256 = 0
    started_at: uint256 = 0
    updated_at: uint256 = 0
    answered_in_round: uint80 = 0
    
    (round_id, price, started_at, updated_at, answered_in_round) = \
        AggregatorV3Interface(ETH_USD_FEED).latestRoundData()
    
    assert price > 0, "Invalid price"
    
    # price มี 8 decimals, eth_amount มี 18 decimals
    # Result จะมี 18 decimals
    usd_value: uint256 = eth_amount * convert(price, uint256) // 10**8
    return usd_value
```

---

## 3. TWAP Oracles {#twap}

Time-Weighted Average Price (TWAP) ใช้ราคาเฉลี่ยตลอดช่วงเวลา ป้องกัน price manipulation

```vyper
# @version 0.4.0
# @title TWAPOracle
# @notice TWAP implementation สำหรับ on-chain price averaging

struct PriceObservation:
    timestamp: uint256
    price: uint256
    cumulative_price: uint256

MAX_OBSERVATIONS: constant(uint256) = 100
MIN_TWAP_PERIOD: constant(uint256) = 300  # 5 minutes minimum

observations: DynArray[PriceObservation, 100]
observation_count: uint256
latest_price: public(uint256)
owner: public(address)
price_source: public(address)  # Contract ที่ให้ราคา

@deploy
def __init__(price_source: address, initial_price: uint256):
    self.owner = msg.sender
    self.price_source = price_source
    self.latest_price = initial_price
    
    # Initial observation
    self.observations.append(PriceObservation({
        timestamp: block.timestamp,
        price: initial_price,
        cumulative_price: initial_price * block.timestamp
    }))
    self.observation_count = 1

@external
def update_price(new_price: uint256):
    """อัพเดทราคาและบันทึก observation"""
    assert msg.sender == self.owner or msg.sender == self.price_source, "Unauthorized"
    assert new_price > 0, "Zero price"
    
    # คำนวณ cumulative price
    last_obs: PriceObservation = self.observations[len(self.observations) - 1]
    
    # ถ้า observations เต็ม ลบอันแรกออก
    if len(self.observations) >= MAX_OBSERVATIONS:
        # Remove first element by shifting (ไม่มี pop front ใน Vyper)
        # ใน production อาจใช้ circular buffer แทน
        pass
    
    new_cumulative: uint256 = last_obs.cumulative_price + \
        last_obs.price * (block.timestamp - last_obs.timestamp)
    
    self.observations.append(PriceObservation({
        timestamp: block.timestamp,
        price: new_price,
        cumulative_price: new_cumulative
    }))
    
    self.latest_price = new_price

@external
@view
def get_twap(period: uint256) -> uint256:
    """
    คำนวณ TWAP สำหรับ period ที่กำหนด (ใน seconds)
    """
    assert period >= MIN_TWAP_PERIOD, "Period too short"
    assert len(self.observations) >= 2, "Insufficient observations"
    
    target_time: uint256 = block.timestamp - period
    latest_obs: PriceObservation = self.observations[len(self.observations) - 1]
    
    # หา observation ที่ใกล้เคียง target_time ที่สุด
    old_cumulative: uint256 = 0
    old_timestamp: uint256 = 0
    
    for i: uint256 in range(100):
        if i >= len(self.observations):
            break
        
        obs: PriceObservation = self.observations[i]
        
        if obs.timestamp <= target_time:
            old_cumulative = obs.cumulative_price
            old_timestamp = obs.timestamp
    
    if old_timestamp == 0:
        # ไม่มี observation เก่าพอ
        return self.latest_price
    
    # คำนวณ cumulative price ของ latest
    current_cumulative: uint256 = latest_obs.cumulative_price + \
        latest_obs.price * (block.timestamp - latest_obs.timestamp)
    
    # TWAP = (current_cumulative - old_cumulative) / time_elapsed
    time_elapsed: uint256 = block.timestamp - old_timestamp
    if time_elapsed == 0:
        return self.latest_price
    
    return (current_cumulative - old_cumulative) // time_elapsed

@external
@view
def get_observations() -> DynArray[PriceObservation, 100]:
    return self.observations
```

---

## 4. Oracle Security {#security}

### Staleness Check

```vyper
# @version 0.4.0
# @title SecureOracle
# @notice Oracle ที่มี security checks

interface AggregatorV3Interface:
    def latestRoundData() -> (uint80, int256, uint256, uint256, uint80): view
    def decimals() -> uint8: view

# Security parameters
MAX_STALENESS: constant(uint256) = 3600     # 1 hour max staleness
MAX_PRICE_CHANGE: constant(uint256) = 1000  # 10% max price change in 1 tx
PRICE_DENOMINATOR: constant(uint256) = 10000

price_feed: public(address)
last_valid_price: public(uint256)
last_update_time: public(uint256)
circuit_breaker_active: public(bool)
owner: public(address)

@deploy
def __init__(feed: address):
    self.price_feed = feed
    self.owner = msg.sender
    self.circuit_breaker_active = False

@internal
@view
def _get_raw_price() -> (uint256, uint256):
    """Get raw price from Chainlink"""
    round_id: uint80 = 0
    price: int256 = 0
    started_at: uint256 = 0
    updated_at: uint256 = 0
    answered_in_round: uint80 = 0
    
    (round_id, price, started_at, updated_at, answered_in_round) = \
        AggregatorV3Interface(self.price_feed).latestRoundData()
    
    return convert(price, uint256), updated_at

@external
@view
def get_safe_price() -> uint256:
    """
    Get price with full security checks:
    1. Staleness check
    2. Circuit breaker check
    3. Price sanity check
    """
    assert not self.circuit_breaker_active, "Circuit breaker active"
    
    price: uint256 = 0
    updated_at: uint256 = 0
    (price, updated_at) = self._get_raw_price()
    
    # Staleness check
    assert block.timestamp - updated_at <= MAX_STALENESS, "Price is stale"
    
    # Price sanity check
    assert price > 0, "Invalid price: zero"
    assert price < 2**128, "Invalid price: overflow"
    
    return price

@external
@view
def get_price_with_validation() -> (uint256, bool, uint256):
    """
    Returns (price, is_valid, age_in_seconds)
    """
    price: uint256 = 0
    updated_at: uint256 = 0
    (price, updated_at) = self._get_raw_price()
    
    age: uint256 = block.timestamp - updated_at
    is_valid: bool = (
        price > 0 and
        age <= MAX_STALENESS and
        not self.circuit_breaker_active
    )
    
    return price, is_valid, age

@external
def trigger_circuit_breaker():
    """Manual circuit breaker"""
    assert msg.sender == self.owner, "Not owner"
    self.circuit_breaker_active = True

@external
def reset_circuit_breaker():
    assert msg.sender == self.owner, "Not owner"
    self.circuit_breaker_active = False
```

---

## 5. OracleConsumer Contract {#oracle-consumer}

```vyper
# @version 0.4.0
# @title OracleConsumer
# @notice DeFi contract ที่ใช้ oracle สำหรับ pricing

interface AggregatorV3Interface:
    def latestRoundData() -> (uint80, int256, uint256, uint256, uint80): view
    def decimals() -> uint8: view

interface IERC20:
    def transfer(to: address, amount: uint256) -> bool: nonpayable
    def transferFrom(frm: address, to: address, amount: uint256) -> bool: nonpayable
    def balanceOf(owner: address) -> uint256: view

# ===== Events =====
event Deposited:
    user: indexed(address)
    eth_amount: uint256
    usd_value: uint256

event Borrowed:
    user: indexed(address)
    stablecoin_amount: uint256
    collateral_eth: uint256

event Liquidated:
    user: indexed(address)
    liquidator: indexed(address)
    collateral_seized: uint256
    debt_covered: uint256

event PriceUpdated:
    new_price: uint256
    timestamp: uint256

# ===== Structs =====
struct UserPosition:
    collateral_eth: uint256      # ETH ที่ deposit (in wei)
    borrowed_usd: uint256        # USD ที่ borrow (in 18 decimals)
    last_update: uint256

# ===== Constants =====
COLLATERAL_RATIO: constant(uint256) = 15000  # 150% collateral ratio
LIQUIDATION_THRESHOLD: constant(uint256) = 12500  # 125% liquidation
LIQUIDATION_BONUS: constant(uint256) = 500  # 5% bonus for liquidators
BASIS_POINTS: constant(uint256) = 10000
MAX_STALENESS: constant(uint256) = 3600  # 1 hour
PRICE_PRECISION: constant(uint256) = 10**8  # Chainlink uses 8 decimals

# ===== State =====
owner: public(address)
eth_usd_feed: public(address)
stablecoin: public(address)

positions: public(HashMap[address, UserPosition])
total_collateral: public(uint256)
total_borrowed: public(uint256)

circuit_breaker: public(bool)
last_valid_price: public(uint256)
last_price_update: public(uint256)

@deploy
def __init__(
    eth_usd_feed: address,
    stablecoin: address
):
    assert eth_usd_feed != empty(address), "Zero oracle"
    assert stablecoin != empty(address), "Zero token"
    
    self.owner = msg.sender
    self.eth_usd_feed = eth_usd_feed
    self.stablecoin = stablecoin
    self.circuit_breaker = False

# ===== Internal Oracle Functions =====

@internal
@view
def _get_eth_price() -> uint256:
    """Get ETH/USD price with full security checks"""
    assert not self.circuit_breaker, "Circuit breaker active"
    
    round_id: uint80 = 0
    price: int256 = 0
    started_at: uint256 = 0
    updated_at: uint256 = 0
    answered_in_round: uint80 = 0
    
    (round_id, price, started_at, updated_at, answered_in_round) = \
        AggregatorV3Interface(self.eth_usd_feed).latestRoundData()
    
    # Security checks
    assert price > 0, "Invalid price"
    assert updated_at > 0, "Round not complete"
    assert block.timestamp - updated_at <= MAX_STALENESS, "Stale price"
    assert round_id >= answered_in_round, "Stale round"
    
    return convert(price, uint256)

@internal
@view
def _eth_to_usd(eth_amount: uint256) -> uint256:
    """แปลง ETH เป็น USD value"""
    price: uint256 = self._get_eth_price()
    # eth_amount: 18 decimals, price: 8 decimals
    # result: 18 decimals (USD)
    return eth_amount * price // PRICE_PRECISION

@internal
@view
def _get_collateral_ratio(user: address) -> uint256:
    """Get collateral ratio ของ user (in basis points)"""
    position: UserPosition = self.positions[user]
    
    if position.borrowed_usd == 0:
        return max_value(uint256)  # ไม่มีหนี้ = infinite collateral ratio
    
    collateral_usd: uint256 = self._eth_to_usd(position.collateral_eth)
    
    # ratio = collateral_usd / borrowed_usd * 10000
    return collateral_usd * BASIS_POINTS // position.borrowed_usd

# ===== User Functions =====

@external
@payable
def deposit():
    """Deposit ETH เป็น collateral"""
    assert msg.value > 0, "Must deposit ETH"
    
    eth_price: uint256 = self._get_eth_price()
    usd_value: uint256 = msg.value * eth_price // PRICE_PRECISION
    
    self.positions[msg.sender].collateral_eth += msg.value
    self.positions[msg.sender].last_update = block.timestamp
    self.total_collateral += msg.value
    
    log Deposited(msg.sender, msg.value, usd_value)

@external
def borrow(usd_amount: uint256):
    """Borrow stablecoins โดยใช้ ETH เป็น collateral"""
    assert usd_amount > 0, "Amount must be positive"
    
    position: UserPosition = self.positions[msg.sender]
    collateral_usd: uint256 = self._eth_to_usd(position.collateral_eth)
    
    new_borrowed: uint256 = position.borrowed_usd + usd_amount
    
    # Check collateral ratio >= 150%
    required_collateral: uint256 = new_borrowed * COLLATERAL_RATIO // BASIS_POINTS
    assert collateral_usd >= required_collateral, "Insufficient collateral"
    
    self.positions[msg.sender].borrowed_usd = new_borrowed
    self.positions[msg.sender].last_update = block.timestamp
    self.total_borrowed += usd_amount
    
    # Mint/transfer stablecoins to user
    assert IERC20(self.stablecoin).transfer(msg.sender, usd_amount), "Transfer failed"
    
    log Borrowed(msg.sender, usd_amount, position.collateral_eth)

@external
def repay(usd_amount: uint256):
    """Repay borrowed stablecoins"""
    position: UserPosition = self.positions[msg.sender]
    assert position.borrowed_usd >= usd_amount, "Repay exceeds debt"
    
    # Transfer stablecoins from user
    assert IERC20(self.stablecoin).transferFrom(
        msg.sender, self, usd_amount
    ), "Transfer failed"
    
    self.positions[msg.sender].borrowed_usd -= usd_amount
    self.positions[msg.sender].last_update = block.timestamp
    self.total_borrowed -= usd_amount

@external
def withdraw(eth_amount: uint256):
    """Withdraw ETH collateral"""
    position: UserPosition = self.positions[msg.sender]
    assert position.collateral_eth >= eth_amount, "Insufficient collateral"
    
    new_collateral: uint256 = position.collateral_eth - eth_amount
    
    # Check ว่ายังมี collateral พอหลัง withdraw
    if position.borrowed_usd > 0:
        new_collateral_usd: uint256 = self._eth_to_usd(new_collateral)
        required: uint256 = position.borrowed_usd * COLLATERAL_RATIO // BASIS_POINTS
        assert new_collateral_usd >= required, "Would undercollateralize"
    
    self.positions[msg.sender].collateral_eth = new_collateral
    self.positions[msg.sender].last_update = block.timestamp
    self.total_collateral -= eth_amount
    
    send(msg.sender, eth_amount)

@external
def liquidate(user: address):
    """
    Liquidate undercollateralized position
    Liquidator repays debt and receives collateral + bonus
    """
    assert user != msg.sender, "Cannot liquidate self"
    
    position: UserPosition = self.positions[user]
    assert position.borrowed_usd > 0, "No debt"
    
    # Check ว่า position undercollateralized
    collateral_ratio: uint256 = self._get_collateral_ratio(user)
    assert collateral_ratio < LIQUIDATION_THRESHOLD, "Not liquidatable"
    
    # คำนวณ ETH ที่จะ seize
    eth_price: uint256 = self._get_eth_price()
    
    # ETH value of debt
    debt_eth: uint256 = position.borrowed_usd * PRICE_PRECISION // eth_price
    
    # Bonus for liquidator
    bonus_eth: uint256 = debt_eth * LIQUIDATION_BONUS // BASIS_POINTS
    total_eth_seized: uint256 = debt_eth + bonus_eth
    
    # ไม่ seize เกิน collateral ที่มี
    if total_eth_seized > position.collateral_eth:
        total_eth_seized = position.collateral_eth
    
    # Transfer debt from liquidator to contract
    assert IERC20(self.stablecoin).transferFrom(
        msg.sender, self, position.borrowed_usd
    ), "Transfer failed"
    
    # Update position
    self.positions[user].collateral_eth -= total_eth_seized
    self.positions[user].borrowed_usd = 0
    self.total_collateral -= total_eth_seized
    self.total_borrowed -= position.borrowed_usd
    
    # Send seized ETH to liquidator
    send(msg.sender, total_eth_seized)
    
    log Liquidated(user, msg.sender, total_eth_seized, position.borrowed_usd)

# ===== View Functions =====

@external
@view
def get_eth_price() -> uint256:
    return self._get_eth_price()

@external
@view
def get_position(user: address) -> (uint256, uint256, uint256):
    """Returns (collateral_eth, borrowed_usd, collateral_ratio_bps)"""
    position: UserPosition = self.positions[user]
    ratio: uint256 = self._get_collateral_ratio(user)
    return position.collateral_eth, position.borrowed_usd, ratio

@external
@view
def is_liquidatable(user: address) -> bool:
    ratio: uint256 = self._get_collateral_ratio(user)
    return ratio < LIQUIDATION_THRESHOLD

@external
@view
def max_borrow(user: address) -> uint256:
    """Maximum amount ที่ user สามารถ borrow ได้"""
    position: UserPosition = self.positions[user]
    collateral_usd: uint256 = self._eth_to_usd(position.collateral_eth)
    
    max_total: uint256 = collateral_usd * BASIS_POINTS // COLLATERAL_RATIO
    
    if max_total <= position.borrowed_usd:
        return 0
    
    return max_total - position.borrowed_usd

# ===== Admin =====

@external
def set_circuit_breaker(active: bool):
    assert msg.sender == self.owner, "Not owner"
    self.circuit_breaker = active

@external
def recover_tokens(token: address, amount: uint256):
    assert msg.sender == self.owner, "Not owner"
    IERC20(token).transfer(self.owner, amount)
```

---

## 6. Testing with Mock Oracles {#testing}

### Mock Chainlink Aggregator

```vyper
# @version 0.4.0
# @title MockAggregatorV3
# @notice Mock Chainlink aggregator สำหรับ testing

event AnswerUpdated:
    current: indexed(int256)
    round_id: indexed(uint256)
    updated_at: uint256

answer: public(int256)
round_id: public(uint80)
updated_at: public(uint256)
decimals_val: public(uint8)
description_val: public(String[100])

@deploy
def __init__(initial_price: int256, _decimals: uint8):
    self.answer = initial_price
    self.decimals_val = _decimals
    self.round_id = 1
    self.updated_at = block.timestamp
    self.description_val = "Mock Price Feed"

@external
@view
def decimals() -> uint8:
    return self.decimals_val

@external
@view
def description() -> String[100]:
    return self.description_val

@external
@view
def version() -> uint256:
    return 1

@external
@view
def latestRoundData() -> (uint80, int256, uint256, uint256, uint80):
    return (
        self.round_id,           # roundId
        self.answer,             # answer
        self.updated_at,         # startedAt
        self.updated_at,         # updatedAt
        self.round_id            # answeredInRound
    )

@external
@view
def getRoundData(
    _roundId: uint80
) -> (uint80, int256, uint256, uint256, uint80):
    return (
        _roundId,
        self.answer,
        self.updated_at,
        self.updated_at,
        _roundId
    )

@external
def updateAnswer(new_answer: int256):
    """Update mock price"""
    self.answer = new_answer
    self.round_id += 1
    self.updated_at = block.timestamp
    log AnswerUpdated(new_answer, convert(self.round_id, uint256), block.timestamp)

@external
def setUpdatedAt(timestamp: uint256):
    """Simulate stale price"""
    self.updated_at = timestamp
```

### Test Suite

```python
# tests/test_oracle_consumer.py
import pytest
import boa

# Mock token fixture
@pytest.fixture
def deployer():
    return boa.env.generate_address()

@pytest.fixture
def alice():
    addr = boa.env.generate_address()
    boa.env.set_balance(addr, 100 * 10**18)
    return addr

@pytest.fixture
def liquidator():
    addr = boa.env.generate_address()
    boa.env.set_balance(addr, 10 * 10**18)
    return addr

@pytest.fixture
def stablecoin(deployer):
    with boa.env.prank(deployer):
        token = boa.load(
            "contracts/Token.vy",
            "USD Stablecoin",
            "USDC",
            10**12 * 10**18
        )
        # Mint to deployer for initial funding
        token.mint(deployer, 10**9 * 10**18)
    return token

@pytest.fixture
def mock_oracle(deployer):
    """Mock oracle với ETH price = $2000"""
    with boa.env.prank(deployer):
        return boa.load(
            "contracts/MockAggregatorV3.vy",
            2000 * 10**8,  # $2000 with 8 decimals
            8              # 8 decimals
        )

@pytest.fixture
def oracle_consumer(deployer, mock_oracle, stablecoin):
    with boa.env.prank(deployer):
        consumer = boa.load(
            "contracts/OracleConsumer.vy",
            mock_oracle.address,
            stablecoin.address
        )
        # Fund consumer with stablecoins
        stablecoin.transfer(consumer.address, 10**6 * 10**18)
    return consumer


class TestOracleConsumer:
    def test_get_eth_price(self, oracle_consumer, mock_oracle):
        """ราคา ETH ต้องตรงกับที่ mock oracle ให้"""
        price = oracle_consumer.get_eth_price()
        assert price == 2000 * 10**8  # $2000 with 8 decimals
    
    def test_deposit_eth(self, oracle_consumer, alice):
        """Deposit ETH เป็น collateral"""
        deposit_amount = 1 * 10**18  # 1 ETH
        
        with boa.env.prank(alice):
            oracle_consumer.deposit(value=deposit_amount)
        
        position = oracle_consumer.positions(alice)
        assert position[0] == deposit_amount  # collateral_eth
        assert position[1] == 0               # borrowed_usd
    
    def test_borrow_stablecoin(self, oracle_consumer, alice, stablecoin):
        """Borrow stablecoins ด้วย collateral"""
        with boa.env.prank(alice):
            oracle_consumer.deposit(value=1 * 10**18)
        
        # Max borrow = 1 ETH * $2000 / 1.5 = $1333.33
        max_borrow = oracle_consumer.max_borrow(alice)
        borrow_amount = max_borrow * 90 // 100  # Borrow 90% of max
        
        alice_balance_before = stablecoin.balanceOf(alice)
        
        with boa.env.prank(alice):
            oracle_consumer.borrow(borrow_amount)
        
        assert stablecoin.balanceOf(alice) == alice_balance_before + borrow_amount
    
    def test_collateral_ratio_enforcement(self, oracle_consumer, alice):
        """ไม่สามารถ borrow เกิน 150% collateral ratio"""
        with boa.env.prank(alice):
            oracle_consumer.deposit(value=1 * 10**18)
        
        # 1 ETH = $2000
        # 150% CR => max borrow = $2000 / 1.5 = $1333.33
        # ลองยืม $1400 (เกิน max)
        too_much = 1400 * 10**18
        
        with pytest.raises(Exception, match="Insufficient collateral"):
            with boa.env.prank(alice):
                oracle_consumer.borrow(too_much)
    
    def test_liquidation_when_undercollateralized(
        self, oracle_consumer, alice, liquidator, mock_oracle, stablecoin, deployer
    ):
        """Liquidation เมื่อราคา ETH ลดลง"""
        # Alice deposits 1 ETH @ $2000
        with boa.env.prank(alice):
            oracle_consumer.deposit(value=1 * 10**18)
        
        # Alice borrows $1200 (CR = 166%)
        with boa.env.prank(alice):
            oracle_consumer.borrow(1200 * 10**18)
        
        # ETH price drops to $1400
        with boa.env.prank(deployer):
            mock_oracle.updateAnswer(1400 * 10**8)
        
        # Check ว่า undercollateralized
        assert oracle_consumer.is_liquidatable(alice) == True
        
        # Liquidator liquidates
        with boa.env.prank(deployer):
            stablecoin.mint(liquidator, 2000 * 10**18)
        
        with boa.env.prank(liquidator):
            stablecoin.approve(oracle_consumer.address, 2000 * 10**18)
            oracle_consumer.liquidate(alice)
        
        # Alice's debt cleared
        position = oracle_consumer.positions(alice)
        assert position[1] == 0  # borrowed_usd = 0
    
    def test_stale_price_reverts(self, oracle_consumer, mock_oracle, deployer, alice):
        """Stale price ควร revert"""
        with boa.env.prank(alice):
            oracle_consumer.deposit(value=1 * 10**18)
        
        # Simulate stale price (2 hours ago)
        with boa.env.prank(deployer):
            mock_oracle.setUpdatedAt(
                boa.env.evm.time - 7200  # 2 hours ago
            )
        
        with pytest.raises(Exception, match="Stale price"):
            oracle_consumer.get_eth_price()
    
    def test_circuit_breaker(self, oracle_consumer, deployer, alice):
        """Circuit breaker ป้องกันการใช้งาน"""
        with boa.env.prank(deployer):
            oracle_consumer.set_circuit_breaker(True)
        
        with pytest.raises(Exception, match="Circuit breaker active"):
            with boa.env.prank(alice):
                oracle_consumer.deposit(value=1 * 10**18)
    
    def test_repay_debt(self, oracle_consumer, alice, stablecoin, deployer):
        """Repay debt"""
        with boa.env.prank(alice):
            oracle_consumer.deposit(value=2 * 10**18)
        
        borrow_amount = 1000 * 10**18
        with boa.env.prank(alice):
            oracle_consumer.borrow(borrow_amount)
        
        # Repay half
        with boa.env.prank(alice):
            stablecoin.approve(oracle_consumer.address, borrow_amount // 2)
            oracle_consumer.repay(borrow_amount // 2)
        
        position = oracle_consumer.positions(alice)
        assert position[1] == borrow_amount // 2
    
    def test_withdraw_collateral(self, oracle_consumer, alice, mock_oracle):
        """Withdraw collateral"""
        with boa.env.prank(alice):
            oracle_consumer.deposit(value=2 * 10**18)
        
        eth_before = boa.env.get_balance(alice)
        
        with boa.env.prank(alice):
            oracle_consumer.withdraw(1 * 10**18)
        
        eth_after = boa.env.get_balance(alice)
        assert eth_after > eth_before
        
        position = oracle_consumer.positions(alice)
        assert position[0] == 1 * 10**18  # collateral_eth
    
    def test_price_update_triggers_event(self, mock_oracle, deployer):
        """Price update ควร emit event"""
        with boa.env.prank(deployer):
            mock_oracle.updateAnswer(2500 * 10**8)
        
        assert mock_oracle.answer() == 2500 * 10**8


class TestOracleSecurity:
    def test_negative_price_rejected(self, oracle_consumer, mock_oracle, deployer, alice):
        """Negative price ควร revert"""
        with boa.env.prank(deployer):
            mock_oracle.updateAnswer(-1)  # Negative price
        
        with pytest.raises(Exception, match="Invalid price"):
            oracle_consumer.get_eth_price()
    
    def test_zero_price_rejected(self, oracle_consumer, mock_oracle, deployer):
        """Zero price ควร revert"""
        with boa.env.prank(deployer):
            mock_oracle.updateAnswer(0)
        
        with pytest.raises(Exception, match="Invalid price"):
            oracle_consumer.get_eth_price()


if __name__ == "__main__":
    pytest.main([__file__, "-v"])
```

---

## สรุป

### Oracle Security Checklist

```
✅ Staleness Checks:
[ ] ตรวจสอบ updatedAt timestamp
[ ] Set MAX_STALENESS ที่เหมาะสม (30 min - 1 hour)
[ ] Revert ถ้า price เก่าเกินไป

✅ Price Validity:
[ ] ตรวจสอบ price > 0
[ ] ตรวจสอบ answeredInRound >= roundId
[ ] ตรวจสอบ startedAt > 0

✅ Circuit Breakers:
[ ] Manual circuit breaker สำหรับ emergency
[ ] Automatic circuit breaker เมื่อราคาเปลี่ยนมาก

✅ Multiple Oracles:
[ ] ใช้ oracle หลายตัวและ average
[ ] Cross-check between Chainlink และ TWAP

✅ TWAP Integration:
[ ] ใช้ TWAP สำหรับ operations ที่ sensitive
[ ] Minimum TWAP period (อย่างน้อย 30 นาที)
```
