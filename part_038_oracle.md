# Part 038: Oracle Integration

## สารบัญ
1. [Oracle Concept](#oracle-concept)
2. [Chainlink Price Feed](#chainlink-price-feed)
3. [TWAP Oracle](#twap-oracle)
4. [Oracle Manipulation Risks](#oracle-manipulation-risks)
5. [ตัวอย่าง: Price-based Contract](#ตัวอย่าง-price-based-contract)
6. [Test Code](#test-code)

---

## Oracle Concept

### Oracle คืออะไร?

**Oracle** คือสะพานเชื่อมระหว่างโลก On-chain และ Off-chain เพราะ Smart Contract ไม่สามารถเข้าถึงข้อมูลภายนอก Blockchain ได้โดยตรง

### ประเภท Oracle
- **Price Feeds**: ราคา ETH/USD, BTC/USD ฯลฯ
- **Random Number**: Chainlink VRF
- **Weather Data**: สำหรับ insurance
- **Sports Results**: สำหรับ prediction markets

### Oracle Providers ที่นิยม
- **Chainlink**: มาตรฐานใน DeFi
- **Pyth Network**: High-frequency, cross-chain
- **Band Protocol**: ใช้ใน Cosmos
- **TWAP (On-chain)**: ดึงจาก DEX (Uniswap, etc.)

---

## Chainlink Price Feed

### Interface สำหรับ Chainlink

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Chainlink Price Feed Integration

# Chainlink AggregatorV3Interface
interface AggregatorV3Interface:
    def decimals() -> uint8: view
    def description() -> String[100]: view
    def version() -> uint256: view
    def getRoundData(roundId: uint80) -> (uint80, int256, uint256, uint256, uint80): view
    def latestRoundData() -> (uint80, int256, uint256, uint256, uint80): view

# Price Feed addresses (Ethereum Mainnet)
ETH_USD_FEED: constant(address) = 0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419
BTC_USD_FEED: constant(address) = 0xF4030086522a5bEEa4988F8cA5B36dbC97BeE88b

owner: public(address)
price_feed: public(address)

# Safety parameters
MAX_PRICE_AGE: constant(uint256) = 3600  # 1 ชั่วโมง
MIN_PRICE: constant(int256) = 1          # ราคาต้องมากกว่า 0

@deploy
def __init__(_feed: address):
    self.owner = msg.sender
    self.price_feed = _feed

@internal
def _get_price() -> int256:
    """
    @dev ดึงราคาล่าสุดจาก Chainlink
    @return price ในหน่วยที่ feed กำหนด (ปกติ 8 decimals)
    """
    feed: AggregatorV3Interface = AggregatorV3Interface(self.price_feed)
    
    round_id: uint80 = 0
    price: int256 = 0
    started_at: uint256 = 0
    updated_at: uint256 = 0
    answered_in_round: uint80 = 0
    
    (round_id, price, started_at, updated_at, answered_in_round) = feed.latestRoundData()
    
    # Validate price
    assert price > MIN_PRICE, "Invalid price: too low"
    assert updated_at > 0, "Invalid: no update time"
    assert block.timestamp - updated_at <= MAX_PRICE_AGE, "Price too stale"
    assert answered_in_round >= round_id, "Stale round"
    
    return price

@view
@external
def get_latest_price() -> int256:
    """ดึงราคาล่าสุด"""
    return self._get_price()

@view
@external
def get_price_usd(amount_eth: uint256) -> uint256:
    """
    @notice แปลง ETH amount เป็น USD
    @param amount_eth จำนวน ETH ใน wei
    @return มูลค่าใน USD (18 decimals)
    """
    price: int256 = self._get_price()  # 8 decimals
    # price * amount_eth / 10^8 = USD value
    # แต่ amount_eth ใน wei (10^18) ดังนั้น
    # USD = price * amount_eth / 10^26
    return convert(price, uint256) * amount_eth / (10**8)

@view
@external
def get_eth_for_usd(usd_amount: uint256) -> uint256:
    """
    @notice แปลง USD amount เป็น ETH
    @param usd_amount จำนวน USD (8 decimals)
    @return จำนวน ETH ใน wei
    """
    price: int256 = self._get_price()  # USD per ETH, 8 decimals
    # ETH = usd_amount / price * 10^18
    return usd_amount * (10**18) / convert(price, uint256)
```

### Multi-Asset Price Aggregator

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Multi-Asset Price Aggregator

interface AggregatorV3Interface:
    def decimals() -> uint8: view
    def latestRoundData() -> (uint80, int256, uint256, uint256, uint80): view

MAX_FEEDS: constant(uint256) = 20
MAX_STALENESS: constant(uint256) = 3600  # 1 hour

struct PriceFeed:
    feed_address: address
    decimals: uint8
    description: String[50]
    active: bool

owner: public(address)
feeds: public(HashMap[String[20], PriceFeed])  # symbol -> feed
feed_symbols: DynArray[String[20], MAX_FEEDS]

@deploy
def __init__():
    self.owner = msg.sender

@external
def add_feed(
    symbol: String[20],
    feed_address: address,
    description: String[50]
):
    """เพิ่ม price feed"""
    assert msg.sender == self.owner
    assert feed_address != empty(address)
    
    decimals: uint8 = AggregatorV3Interface(feed_address).decimals()
    
    if self.feeds[symbol].feed_address == empty(address):
        self.feed_symbols.append(symbol)
    
    self.feeds[symbol] = PriceFeed({
        feed_address: feed_address,
        decimals: decimals,
        description: description,
        active: True
    })

@view
@external
def get_price(symbol: String[20]) -> (int256, uint256, uint8):
    """
    ดึงราคาสำหรับ symbol
    @return (price, updated_at, decimals)
    """
    feed_info: PriceFeed = self.feeds[symbol]
    assert feed_info.active, "Feed not active"
    
    feed: AggregatorV3Interface = AggregatorV3Interface(feed_info.feed_address)
    
    round_id: uint80 = 0
    price: int256 = 0
    started_at: uint256 = 0
    updated_at: uint256 = 0
    answered_in_round: uint80 = 0
    
    (round_id, price, started_at, updated_at, answered_in_round) = feed.latestRoundData()
    
    assert price > 0, "Invalid price"
    assert block.timestamp - updated_at <= MAX_STALENESS, "Price stale"
    
    return price, updated_at, feed_info.decimals

@view
@external
def get_price_normalized(symbol: String[20]) -> uint256:
    """
    ดึงราคาและ normalize เป็น 18 decimals
    """
    price: int256 = 0
    updated_at: uint256 = 0
    decimals: uint8 = 0
    
    (price, updated_at, decimals) = self.get_price(symbol)
    
    # Normalize to 18 decimals
    if decimals < 18:
        return convert(price, uint256) * (10 ** convert(18 - decimals, uint256))
    elif decimals > 18:
        return convert(price, uint256) / (10 ** convert(decimals - 18, uint256))
    else:
        return convert(price, uint256)
```

---

## TWAP Oracle

### Time-Weighted Average Price

TWAP คำนวณราคาเฉลี่ยตามเวลา ทำให้ทนต่อการ manipulate ได้ดีกว่า spot price

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title TWAP Oracle
# @notice Time-Weighted Average Price Oracle แบบ simple

struct PricePoint:
    price: uint256
    timestamp: uint256

MAX_OBSERVATIONS: constant(uint256) = 100

owner: public(address)
observations: public(HashMap[uint256, PricePoint])
observation_count: public(uint256)
observation_index: public(uint256)  # circular buffer index

TWAP_PERIOD: constant(uint256) = 1800  # 30 นาที

@deploy
def __init__():
    self.owner = msg.sender

@external
def record_price(price: uint256):
    """
    @notice บันทึกราคา (เรียกจาก authorized source)
    """
    assert msg.sender == self.owner, "Not owner"
    assert price > 0, "Invalid price"
    
    idx: uint256 = self.observation_index
    
    self.observations[idx] = PricePoint({
        price: price,
        timestamp: block.timestamp
    })
    
    self.observation_index = (idx + 1) % MAX_OBSERVATIONS
    
    if self.observation_count < MAX_OBSERVATIONS:
        self.observation_count += 1

@view
@external
def get_twap() -> uint256:
    """
    @notice คำนวณ TWAP สำหรับช่วงเวลาที่กำหนด
    @return ราคาเฉลี่ย
    """
    count: uint256 = self.observation_count
    assert count > 0, "No observations"
    
    cutoff: uint256 = block.timestamp - TWAP_PERIOD
    
    price_sum: uint256 = 0
    weight_sum: uint256 = 0
    prev_time: uint256 = block.timestamp
    
    # ค้นหาจาก latest ย้อนหลัง
    for i: uint256 in range(MAX_OBSERVATIONS):
        if i >= count:
            break
        
        # Index ใน circular buffer
        idx: uint256 = 0
        if self.observation_index >= i + 1:
            idx = self.observation_index - i - 1
        else:
            idx = MAX_OBSERVATIONS + self.observation_index - i - 1
        
        obs: PricePoint = self.observations[idx]
        
        if obs.timestamp < cutoff:
            # ถ้าอยู่นอก period ให้หยุด
            # ใช้ partial weight
            remaining: uint256 = obs.timestamp - cutoff + (prev_time - obs.timestamp)
            if remaining > 0:
                price_sum += obs.price * remaining
                weight_sum += remaining
            break
        
        # น้ำหนักของ observation นี้คือเวลาตั้งแต่ observation นี้จนถึงครั้งก่อน
        time_weight: uint256 = prev_time - obs.timestamp
        price_sum += obs.price * time_weight
        weight_sum += time_weight
        prev_time = obs.timestamp
    
    if weight_sum == 0:
        return self.observations[(self.observation_index - 1) % MAX_OBSERVATIONS].price
    
    return price_sum / weight_sum
```

### Uniswap V3-style TWAP

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Uniswap-style TWAP Reader

interface IUniswapV3Pool:
    def observe(
        secondsAgos: DynArray[uint32, 2]
    ) -> (DynArray[int56, 2], DynArray[uint160, 2]): view
    def token0() -> address: view
    def token1() -> address: view

pool: public(address)
twap_interval: public(uint32)

@deploy
def __init__(_pool: address, _interval: uint32):
    self.pool = _pool
    self.twap_interval = _interval

@view
@external
def get_twap_tick() -> int24:
    """
    @notice ดึง TWAP tick จาก Uniswap V3
    @return arithmetic mean tick
    """
    seconds_agos: DynArray[uint32, 2] = [self.twap_interval, 0]
    
    tick_cumulatives: DynArray[int56, 2] = []
    liquidity_cumulatives: DynArray[uint160, 2] = []
    
    (tick_cumulatives, liquidity_cumulatives) = IUniswapV3Pool(self.pool).observe(seconds_agos)
    
    tick_delta: int56 = tick_cumulatives[1] - tick_cumulatives[0]
    mean_tick: int24 = convert(tick_delta / convert(self.twap_interval, int56), int24)
    
    return mean_tick
```

---

## Oracle Manipulation Risks

### ความเสี่ยงและการป้องกัน

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Oracle with Safety Checks

interface AggregatorV3Interface:
    def latestRoundData() -> (uint80, int256, uint256, uint256, uint80): view

event PriceOutOfBounds:
    price: int256
    min_price: int256
    max_price: int256

event OracleStale:
    last_update: uint256
    current_time: uint256

# Safety parameters
MIN_PRICE: constant(int256) = 100_000_000    # $1.00 (8 decimals)
MAX_PRICE: constant(int256) = 100_000_000_000_000  # $1,000,000
MAX_DEVIATION_BPS: constant(uint256) = 1000  # 10% max deviation
MAX_AGE: constant(uint256) = 3600            # 1 hour

owner: public(address)
primary_feed: public(address)
fallback_feed: public(address)
last_valid_price: public(int256)
circuit_breaker_active: public(bool)

@deploy
def __init__(_primary: address, _fallback: address):
    self.owner = msg.sender
    self.primary_feed = _primary
    self.fallback_feed = _fallback

@internal
def _get_validated_price(feed: address) -> int256:
    """
    ดึงราคาพร้อม validation ครบถ้วน
    """
    round_id: uint80 = 0
    price: int256 = 0
    started_at: uint256 = 0
    updated_at: uint256 = 0
    answered_in_round: uint80 = 0
    
    (round_id, price, started_at, updated_at, answered_in_round) = \
        AggregatorV3Interface(feed).latestRoundData()
    
    # 1. ตรวจสอบ price ไม่เป็น 0 หรือลบ
    assert price > 0, "Price must be positive"
    
    # 2. ตรวจสอบ price อยู่ใน range
    if price < MIN_PRICE or price > MAX_PRICE:
        log PriceOutOfBounds(price, MIN_PRICE, MAX_PRICE)
        assert False, "Price out of bounds"
    
    # 3. ตรวจสอบ freshness
    if block.timestamp - updated_at > MAX_AGE:
        log OracleStale(updated_at, block.timestamp)
        assert False, "Price too stale"
    
    # 4. ตรวจสอบ round completeness
    assert answered_in_round >= round_id, "Incomplete round"
    
    return price

@internal
def _check_deviation(new_price: int256, old_price: int256) -> bool:
    """ตรวจสอบว่าราคาไม่กระโดดมากเกินไป"""
    if old_price == 0:
        return True
    
    diff: int256 = new_price - old_price
    if diff < 0:
        diff = -diff
    
    deviation_bps: uint256 = convert(diff, uint256) * 10000 / convert(old_price, uint256)
    return deviation_bps <= MAX_DEVIATION_BPS

@external
def get_safe_price() -> int256:
    """
    @notice ดึงราคาด้วย circuit breaker
    """
    if self.circuit_breaker_active:
        assert False, "Circuit breaker active"
    
    # ลอง primary feed
    price: int256 = 0
    try_primary: bool = True
    
    if try_primary:
        price = self._get_validated_price(self.primary_feed)
    
    # ตรวจสอบ deviation จากราคาครั้งก่อน
    if not self._check_deviation(price, self.last_valid_price):
        # Activate circuit breaker
        self.circuit_breaker_active = True
        assert False, "Price deviation too large"
    
    self.last_valid_price = price
    return price

@external
def reset_circuit_breaker():
    """Reset circuit breaker (owner only)"""
    assert msg.sender == self.owner
    self.circuit_breaker_active = False

@external
def use_fallback():
    """ใช้ fallback feed"""
    assert msg.sender == self.owner
    # Swap feeds
    temp: address = self.primary_feed
    self.primary_feed = self.fallback_feed
    self.fallback_feed = temp
```

---

## ตัวอย่าง: Price-based Contract

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Price-based Lending Contract
# @notice Contract ที่ใช้ราคาจาก Oracle สำหรับ lending

interface AggregatorV3Interface:
    def latestRoundData() -> (uint80, int256, uint256, uint256, uint80): view
    def decimals() -> uint8: view

interface ERC20:
    def transfer(to: address, amount: uint256) -> bool: nonpayable
    def transferFrom(from_: address, to: address, amount: uint256) -> bool: nonpayable
    def balanceOf(account: address) -> uint256: view

# ==================== Constants ====================

MAX_LTV_BPS: constant(uint256) = 7500       # 75% LTV
LIQUIDATION_LTV_BPS: constant(uint256) = 8500  # 85% LTV (liquidation threshold)
LIQUIDATION_BONUS_BPS: constant(uint256) = 500  # 5% bonus สำหรับ liquidator
BORROW_RATE_PER_YEAR_BPS: constant(uint256) = 500  # 5% APR
ORACLE_MAX_AGE: constant(uint256) = 3600    # 1 ชั่วโมง
PRICE_DECIMALS: constant(uint256) = 8       # Chainlink ใช้ 8 decimals

# ==================== Events ====================

event Deposited:
    user: indexed(address)
    amount: uint256

event Borrowed:
    user: indexed(address)
    amount: uint256

event Repaid:
    user: indexed(address)
    amount: uint256

event Liquidated:
    user: indexed(address)
    liquidator: indexed(address)
    collateral_seized: uint256
    debt_repaid: uint256

# ==================== Structs ====================

struct Position:
    collateral: uint256     # ETH collateral ใน wei
    debt: uint256           # Stablecoin debt
    last_update: uint256    # ล่าสุดที่ update interest

# ==================== State Variables ====================

owner: public(address)
eth_usd_feed: public(address)
stable_token: public(address)

positions: public(HashMap[address, Position])
total_collateral: public(uint256)
total_debt: public(uint256)

is_paused: public(bool)

# ==================== Constructor ====================

@deploy
def __init__(
    _feed: address,
    _stable_token: address
):
    self.owner = msg.sender
    self.eth_usd_feed = _feed
    self.stable_token = _stable_token

# ==================== Oracle Functions ====================

@internal
def _get_eth_price() -> uint256:
    """ดึงราคา ETH/USD จาก Chainlink"""
    feed: AggregatorV3Interface = AggregatorV3Interface(self.eth_usd_feed)
    
    round_id: uint80 = 0
    price: int256 = 0
    started_at: uint256 = 0
    updated_at: uint256 = 0
    answered_in_round: uint80 = 0
    
    (round_id, price, started_at, updated_at, answered_in_round) = feed.latestRoundData()
    
    assert price > 0, "Invalid ETH price"
    assert block.timestamp - updated_at <= ORACLE_MAX_AGE, "ETH price stale"
    assert answered_in_round >= round_id, "Incomplete round"
    
    return convert(price, uint256)  # 8 decimals

@internal
def _eth_to_usd(eth_amount: uint256) -> uint256:
    """แปลง ETH เป็น USD value"""
    price: uint256 = self._get_eth_price()
    # price = USD per ETH (8 decimals)
    # eth_amount = wei (18 decimals)
    # result = USD (8 decimals)
    return price * eth_amount / (10**18)

@internal
def _calculate_health_factor(position: Position) -> uint256:
    """
    คำนวณ Health Factor
    HF > 10000 = safe
    HF < 10000 = at risk of liquidation
    """
    if position.debt == 0:
        return max_value(uint256)
    
    collateral_value_usd: uint256 = self._eth_to_usd(position.collateral)
    
    # Health factor = (collateral * liquidation_threshold) / debt
    # คูณ 10000 เพื่อ precision
    liquidation_value: uint256 = collateral_value_usd * LIQUIDATION_LTV_BPS / 10000
    
    return liquidation_value * 10000 / position.debt

# ==================== Core Functions ====================

@external
@payable
@nonreentrant
def deposit():
    """
    @notice ฝาก ETH เป็น collateral
    """
    assert not self.is_paused, "Paused"
    assert msg.value > 0, "Must deposit ETH"
    
    self._accrue_interest(msg.sender)
    
    self.positions[msg.sender].collateral += msg.value
    self.total_collateral += msg.value
    
    log Deposited(msg.sender, msg.value)

@external
@nonreentrant
def borrow(amount: uint256):
    """
    @notice กู้ stablecoin โดยใช้ ETH เป็น collateral
    """
    assert not self.is_paused, "Paused"
    assert amount > 0, "Must borrow > 0"
    
    self._accrue_interest(msg.sender)
    
    position: Position = self.positions[msg.sender]
    new_debt: uint256 = position.debt + amount
    
    # ตรวจสอบ LTV
    collateral_value: uint256 = self._eth_to_usd(position.collateral)
    max_borrow: uint256 = collateral_value * MAX_LTV_BPS / 10000
    
    assert new_debt <= max_borrow, "Exceeds max LTV"
    
    self.positions[msg.sender].debt = new_debt
    self.total_debt += amount
    
    # ส่ง stablecoin
    ERC20(self.stable_token).transfer(msg.sender, amount)
    
    log Borrowed(msg.sender, amount)

@external
@nonreentrant
def repay(amount: uint256):
    """
    @notice ชำระหนี้
    """
    self._accrue_interest(msg.sender)
    
    position: Position = self.positions[msg.sender]
    repay_amount: uint256 = min(amount, position.debt)
    
    assert repay_amount > 0, "No debt"
    
    # รับ stablecoin
    ERC20(self.stable_token).transferFrom(msg.sender, self, repay_amount)
    
    self.positions[msg.sender].debt -= repay_amount
    self.total_debt -= repay_amount
    
    log Repaid(msg.sender, repay_amount)

@external
@nonreentrant
def withdraw(amount: uint256):
    """
    @notice ถอน ETH collateral
    """
    self._accrue_interest(msg.sender)
    
    position: Position = self.positions[msg.sender]
    assert position.collateral >= amount, "Insufficient collateral"
    
    # ตรวจสอบว่า HF ยังปลอดภัยหลังถอน
    new_collateral: uint256 = position.collateral - amount
    
    if position.debt > 0:
        new_collateral_usd: uint256 = self._eth_to_usd(new_collateral)
        min_collateral: uint256 = position.debt * 10000 / MAX_LTV_BPS
        assert new_collateral_usd >= min_collateral, "Would violate LTV"
    
    self.positions[msg.sender].collateral = new_collateral
    self.total_collateral -= amount
    
    send(msg.sender, amount)

@external
@nonreentrant
def liquidate(borrower: address):
    """
    @notice Liquidate position ที่ HF < 1
    """
    assert not self.is_paused, "Paused"
    
    self._accrue_interest(borrower)
    
    position: Position = self.positions[borrower]
    assert position.debt > 0, "No debt"
    
    # ตรวจสอบว่า HF ต่ำกว่า 1
    hf: uint256 = self._calculate_health_factor(position)
    assert hf < 10000, "Not liquidatable"
    
    # คำนวณ collateral ที่จะยึด
    eth_price: uint256 = self._get_eth_price()
    
    # ยึด collateral เป็นมูลค่าเท่ากับ debt + bonus
    debt_in_eth: uint256 = position.debt * 10**18 / eth_price
    collateral_to_seize: uint256 = debt_in_eth * (10000 + LIQUIDATION_BONUS_BPS) / 10000
    
    # ไม่ยึดเกิน collateral ที่มี
    collateral_to_seize = min(collateral_to_seize, position.collateral)
    
    # รับ stablecoin จาก liquidator
    ERC20(self.stable_token).transferFrom(msg.sender, self, position.debt)
    
    # ล้าง position
    self.total_debt -= position.debt
    self.total_collateral -= collateral_to_seize
    
    self.positions[borrower].debt = 0
    self.positions[borrower].collateral -= collateral_to_seize
    
    # ส่ง collateral ให้ liquidator
    send(msg.sender, collateral_to_seize)
    
    log Liquidated(borrower, msg.sender, collateral_to_seize, position.debt)

# ==================== Interest ====================

@internal
def _accrue_interest(user: address):
    """คำนวณและอัปเดต interest"""
    position: Position = self.positions[user]
    
    if position.debt == 0 or position.last_update == 0:
        self.positions[user].last_update = block.timestamp
        return
    
    elapsed: uint256 = block.timestamp - position.last_update
    interest: uint256 = position.debt * BORROW_RATE_PER_YEAR_BPS * elapsed / (10000 * 31536000)
    
    self.positions[user].debt += interest
    self.positions[user].last_update = block.timestamp

# ==================== View Functions ====================

@view
@external
def get_health_factor(user: address) -> uint256:
    return self._calculate_health_factor(self.positions[user])

@view
@external
def get_position(user: address) -> Position:
    return self.positions[user]

@view
@external
def get_eth_price_usd() -> uint256:
    return self._get_eth_price()

@view
@external
def get_max_borrow(user: address) -> uint256:
    position: Position = self.positions[user]
    collateral_value: uint256 = self._eth_to_usd(position.collateral)
    max_borrow: uint256 = collateral_value * MAX_LTV_BPS / 10000
    
    if position.debt >= max_borrow:
        return 0
    
    return max_borrow - position.debt

@view
@external
def is_liquidatable(user: address) -> bool:
    return self._calculate_health_factor(self.positions[user]) < 10000

@external
def set_paused(state: bool):
    assert msg.sender == self.owner
    self.is_paused = state
```

---

## Test Code

```python
# tests/test_oracle.py
import pytest

@pytest.fixture
def owner(accounts):
    return accounts[0]

@pytest.fixture
def user(accounts):
    return accounts[1]

@pytest.fixture
def mock_feed(owner, project):
    """Mock Chainlink Feed"""
    return project.MockAggregatorV3.deploy(
        8,  # decimals
        2000_00000000,  # $2000 ETH price (8 decimals)
        sender=owner
    )

@pytest.fixture
def stable_token(owner, project):
    t = project.MockERC20.deploy("USD Stable", "USDS", 8, sender=owner)
    t.mint(owner.address, 1_000_000 * 10**8, sender=owner)
    return t

@pytest.fixture
def lending(owner, mock_feed, stable_token, project):
    contract = project.OracleLending.deploy(
        mock_feed.address,
        stable_token.address,
        sender=owner
    )
    # ส่ง stablecoin เข้า contract
    stable_token.transfer(contract.address, 1_000_000 * 10**8, sender=owner)
    return contract

class TestOracleIntegration:
    
    def test_get_eth_price(self, lending, mock_feed):
        """ทดสอบดึงราคา ETH"""
        price = lending.get_eth_price_usd()
        assert price == 2000_00000000  # $2000
    
    def test_deposit_and_borrow(self, lending, user):
        """ทดสอบ deposit ETH และ borrow"""
        # ฝาก 1 ETH
        deposit_amount = 1 * 10**18
        lending.deposit(sender=user, value=deposit_amount)
        
        # max borrow = 1 ETH * $2000 * 75% = $1500
        max_borrow = lending.get_max_borrow(user.address)
        expected_max = 2000_00000000 * 75 // 100  # $1500 ใน 8 decimals
        
        # Borrow $1000
        borrow_amount = 1000_00000000
        lending.borrow(borrow_amount, sender=user)
        
        position = lending.get_position(user.address)
        assert position.debt == borrow_amount
    
    def test_health_factor_calculation(self, lending, user):
        """ทดสอบคำนวณ Health Factor"""
        lending.deposit(sender=user, value=1 * 10**18)
        lending.borrow(1000_00000000, sender=user)
        
        hf = lending.get_health_factor(user.address)
        # HF = (2000 * 0.85) / 1000 * 10000 = 17000
        assert hf > 10000  # safe
    
    def test_liquidation_when_price_drops(
        self, lending, user, mock_feed, accounts, owner
    ):
        """ทดสอบ liquidation เมื่อราคา ETH ตก"""
        liquidator = accounts[2]
        
        lending.deposit(sender=user, value=1 * 10**18)
        lending.borrow(1400_00000000, sender=user)  # Borrow $1400
        
        # ราคา ETH ตก 30% เหลือ $1400
        mock_feed.updatePrice(1400_00000000, sender=owner)
        
        # ตอนนี้ HF ต่ำกว่า 1
        assert lending.is_liquidatable(user.address)
        
        # Liquidator ทำ liquidation

class TestOracleSafety:
    
    def test_stale_price_rejected(self, lending, mock_feed, owner, chain):
        """ทดสอบว่าราคาเก่าถูก reject"""
        # เดิน time ข้ามไป 2 ชั่วโมง
        chain.mine(deltatime=7200)
        
        with pytest.raises(Exception) as e:
            lending.get_eth_price_usd()
        
        assert "stale" in str(e.value).lower()
    
    def test_zero_price_rejected(self, lending, mock_feed, owner):
        """ทดสอบว่าราคา 0 ถูก reject"""
        mock_feed.updatePrice(0, sender=owner)
        
        with pytest.raises(Exception):
            lending.get_eth_price_usd()
```

---

## สรุป

Oracle เป็น critical component ใน DeFi:

| Oracle Type | ข้อดี | ข้อเสีย |
|------------|-------|---------|
| Chainlink | Decentralized, reliable | ต้องจ่าย LINK |
| TWAP | On-chain, manipulation-resistant | Delayed |
| Custom | Flexible | Centralized risk |

### Security Checklist
- [ ] ตรวจสอบ price staleness
- [ ] ตรวจสอบ price bounds
- [ ] ใช้ TWAP แทน spot price
- [ ] มี fallback oracle
- [ ] Circuit breaker สำหรับ extreme movement

---

[⬅️ Part 037: Merkle Tree Proofs](part_037_merkle.md) | [Part 039: Cross-Contract Calls ➡️](part_039_cross_contract.md)
