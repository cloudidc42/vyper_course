# Part 035: Staking Contract

## สารบัญ
1. [บทนำ Staking](#บทนำ)
2. [Basic Staking](#basic-staking)
3. [Reward Calculation](#reward-calculation)
4. [APY/APR](#apyapr)
5. [Lock Periods](#lock-periods)
6. [ตัวอย่าง: Token Staking Pool](#ตัวอย่าง-token-staking-pool)
7. [Test Code](#test-code)

---

## บทนำ

**Staking** คือการล็อค token เพื่อรับ reward เป็นกลไกหลักใน DeFi ที่ใช้สร้าง yield

### ประเภท Staking
- **Simple Staking**: stake token รับ reward เป็น token เดิม
- **LP Staking**: stake LP token รับ token อื่น
- **NFT Staking**: stake NFT รับ token
- **Dual Reward**: รับ reward 2 ประเภท

---

## Basic Staking

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Basic Staking

interface ERC20:
    def transfer(to: address, amount: uint256) -> bool: nonpayable
    def transferFrom(from_: address, to: address, amount: uint256) -> bool: nonpayable
    def balanceOf(account: address) -> uint256: view

event Staked:
    user: indexed(address)
    amount: uint256

event Withdrawn:
    user: indexed(address)
    amount: uint256

event RewardPaid:
    user: indexed(address)
    reward: uint256

staking_token: public(address)
reward_token: public(address)

total_supply: public(uint256)
balances: public(HashMap[address, uint256])

reward_per_second: public(uint256)        # reward ต่อวินาที
last_update_time: public(uint256)
reward_per_token_stored: public(uint256)  # accumulated reward ต่อ token

user_reward_per_token_paid: public(HashMap[address, uint256])
rewards: public(HashMap[address, uint256])

@deploy
def __init__(
    _staking_token: address,
    _reward_token: address,
    _reward_per_second: uint256
):
    self.staking_token = _staking_token
    self.reward_token = _reward_token
    self.reward_per_second = _reward_per_second
    self.last_update_time = block.timestamp

@internal
def _reward_per_token() -> uint256:
    if self.total_supply == 0:
        return self.reward_per_token_stored
    
    time_delta: uint256 = block.timestamp - self.last_update_time
    return self.reward_per_token_stored + (
        time_delta * self.reward_per_second * 10**18 / self.total_supply
    )

@internal
def _earned(account: address) -> uint256:
    return (
        self.balances[account] * 
        (self._reward_per_token() - self.user_reward_per_token_paid[account]) / 
        10**18
    ) + self.rewards[account]

@internal
def _update_reward(account: address):
    self.reward_per_token_stored = self._reward_per_token()
    self.last_update_time = block.timestamp
    
    if account != empty(address):
        self.rewards[account] = self._earned(account)
        self.user_reward_per_token_paid[account] = self.reward_per_token_stored

@external
def stake(amount: uint256):
    assert amount > 0, "Cannot stake 0"
    
    self._update_reward(msg.sender)
    
    self.total_supply += amount
    self.balances[msg.sender] += amount
    
    ERC20(self.staking_token).transferFrom(msg.sender, self, amount)
    
    log Staked(msg.sender, amount)

@external
def withdraw(amount: uint256):
    assert amount > 0, "Cannot withdraw 0"
    assert self.balances[msg.sender] >= amount
    
    self._update_reward(msg.sender)
    
    self.total_supply -= amount
    self.balances[msg.sender] -= amount
    
    ERC20(self.staking_token).transfer(msg.sender, amount)
    
    log Withdrawn(msg.sender, amount)

@external
def get_reward():
    self._update_reward(msg.sender)
    
    reward: uint256 = self.rewards[msg.sender]
    if reward > 0:
        self.rewards[msg.sender] = 0
        ERC20(self.reward_token).transfer(msg.sender, reward)
        log RewardPaid(msg.sender, reward)

@view
@external
def earned(account: address) -> uint256:
    return self._earned(account)
```

---

## Reward Calculation

### วิธีคำนวณ Reward

```
reward_per_token = accumulated reward ต่อ 1 token stake

สูตร:
reward_per_token += (time_elapsed * reward_rate) / total_staked

earned = balance * (reward_per_token - user_paid_per_token) + pending_reward
```

### ตัวอย่างการคำนวณ

```
สมมติ:
- Reward rate = 100 tokens/วัน
- Alice stake 1000 tokens
- Bob stake 3000 tokens
- Total staked = 4000 tokens

หลังจาก 1 วัน:
- Total reward = 100 tokens
- Alice share = 1000/4000 = 25% -> ได้ 25 tokens
- Bob share = 3000/4000 = 75% -> ได้ 75 tokens
```

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Reward Calculator Demo

# ตัวอย่างการคำนวณ reward แบบ step-by-step

reward_per_second: public(uint256)  # tokens per second (x 10^18 precision)
total_staked: public(uint256)

# ต่อ user
staked_amount: public(HashMap[address, uint256])
reward_debt: public(HashMap[address, uint256])  # reward ที่หักออกแล้ว

accumulated_reward_per_token: public(uint256)  # cumulative
last_update: public(uint256)

@deploy
def __init__(_reward_per_second: uint256):
    self.reward_per_second = _reward_per_second
    self.last_update = block.timestamp

@internal
def _update():
    """อัปเดต accumulated reward"""
    if self.total_staked == 0:
        self.last_update = block.timestamp
        return
    
    elapsed: uint256 = block.timestamp - self.last_update
    new_reward: uint256 = elapsed * self.reward_per_second * 10**18 / self.total_staked
    
    self.accumulated_reward_per_token += new_reward
    self.last_update = block.timestamp

@internal
def _pending_reward(user: address) -> uint256:
    """คำนวณ pending reward ของ user"""
    return self.staked_amount[user] * self.accumulated_reward_per_token / 10**18 - \
           self.reward_debt[user]

@external
def deposit(amount: uint256):
    self._update()
    
    # Harvest reward ก่อน stake เพิ่ม
    pending: uint256 = self._pending_reward(msg.sender)
    # ... pay pending reward ...
    
    self.staked_amount[msg.sender] += amount
    self.total_staked += amount
    
    # อัปเดต debt
    self.reward_debt[msg.sender] = self.staked_amount[msg.sender] * \
                                   self.accumulated_reward_per_token / 10**18

@view
@external
def pending_reward(user: address) -> uint256:
    # คำนวณแบบ view (ไม่อัปเดต state)
    if self.total_staked == 0:
        return 0
    
    elapsed: uint256 = block.timestamp - self.last_update
    accumulated: uint256 = self.accumulated_reward_per_token + \
                           elapsed * self.reward_per_second * 10**18 / self.total_staked
    
    return self.staked_amount[user] * accumulated / 10**18 - \
           self.reward_debt[user]
```

---

## APY/APR

### คำนวณ APY และ APR

```
APR = (reward_per_year / total_staked) * 100

APY = ((1 + APR/n)^n - 1) * 100
      โดย n = จำนวน compound ต่อปี
```

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title APR Calculator

# APR calculation (ทำบน frontend แต่ contract ให้ข้อมูล)

SECONDS_PER_YEAR: constant(uint256) = 31536000

reward_rate: public(uint256)    # tokens per second
total_staked: public(uint256)
reward_token_price: public(uint256)   # price in terms of staking token
staking_token_price: public(uint256)  # ราคาเปรียบเทียบกัน

@view
@external
def get_apr_bps() -> uint256:
    """
    @notice คำนวณ APR ในหน่วย basis points
    @dev 10000 bps = 100% APR
    @return APR ใน basis points
    """
    if self.total_staked == 0:
        return 0
    
    # reward per year
    yearly_rewards: uint256 = self.reward_rate * SECONDS_PER_YEAR
    
    # APR = yearly_rewards / total_staked * 10000 (bps)
    return yearly_rewards * 10000 / self.total_staked

@view
@external
def get_daily_reward(amount: uint256) -> uint256:
    """
    @notice คำนวณ reward ต่อวันสำหรับ amount ที่ stake
    """
    if self.total_staked == 0:
        return 0
    
    daily_total: uint256 = self.reward_rate * 86400
    return daily_total * amount / self.total_staked
```

---

## Lock Periods

### Staking พร้อม Lock Period

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Staking with Lock Periods

interface ERC20:
    def transfer(to: address, amount: uint256) -> bool: nonpayable
    def transferFrom(from_: address, to: address, amount: uint256) -> bool: nonpayable

event Staked:
    user: indexed(address)
    amount: uint256
    lock_until: uint256
    reward_multiplier: uint256

event Withdrawn:
    user: indexed(address)
    amount: uint256
    early_exit_penalty: uint256

struct StakePosition:
    amount: uint256
    lock_until: uint256
    reward_multiplier: uint256  # บัวส์ reward (100 = 1x, 150 = 1.5x)
    stake_time: uint256

# Lock tiers
LOCK_30_DAYS: constant(uint256) = 30 * 86400
LOCK_90_DAYS: constant(uint256) = 90 * 86400
LOCK_180_DAYS: constant(uint256) = 180 * 86400
LOCK_365_DAYS: constant(uint256) = 365 * 86400

MULTIPLIER_NO_LOCK: constant(uint256) = 100    # 1x
MULTIPLIER_30_DAYS: constant(uint256) = 125    # 1.25x
MULTIPLIER_90_DAYS: constant(uint256) = 150    # 1.5x
MULTIPLIER_180_DAYS: constant(uint256) = 175   # 1.75x
MULTIPLIER_365_DAYS: constant(uint256) = 200   # 2x

EARLY_EXIT_PENALTY_BPS: constant(uint256) = 1000  # 10% penalty

staking_token: public(address)
positions: public(HashMap[address, StakePosition])

@deploy
def __init__(_token: address):
    self.staking_token = _token

@internal
def _get_multiplier(lock_days: uint256) -> uint256:
    if lock_days == 0:
        return MULTIPLIER_NO_LOCK
    elif lock_days <= 30:
        return MULTIPLIER_30_DAYS
    elif lock_days <= 90:
        return MULTIPLIER_90_DAYS
    elif lock_days <= 180:
        return MULTIPLIER_180_DAYS
    else:
        return MULTIPLIER_365_DAYS

@external
def stake(amount: uint256, lock_days: uint256):
    """
    @notice Stake tokens พร้อมเลือก lock period
    @param lock_days จำนวนวันที่ lock (0 = flexible)
    """
    assert amount > 0
    assert lock_days <= 365, "Max lock 365 days"
    
    multiplier: uint256 = self._get_multiplier(lock_days)
    lock_until: uint256 = block.timestamp + lock_days * 86400
    
    current: StakePosition = self.positions[msg.sender]
    
    if current.amount > 0:
        # เพิ่มใน position เดิม (ถ้า lock ยาวกว่าหรือเท่ากัน)
        assert lock_until >= current.lock_until, "Cannot reduce lock period"
        self.positions[msg.sender].amount += amount
        if lock_until > current.lock_until:
            self.positions[msg.sender].lock_until = lock_until
            self.positions[msg.sender].reward_multiplier = multiplier
    else:
        self.positions[msg.sender] = StakePosition({
            amount: amount,
            lock_until: lock_until,
            reward_multiplier: multiplier,
            stake_time: block.timestamp
        })
    
    ERC20(self.staking_token).transferFrom(msg.sender, self, amount)
    
    log Staked(msg.sender, amount, lock_until, multiplier)

@external
def withdraw(amount: uint256):
    """
    @notice ถอน tokens
    @dev ถ้าก่อน lock_until จะมี penalty
    """
    position: StakePosition = self.positions[msg.sender]
    assert position.amount >= amount, "Insufficient staked"
    
    penalty: uint256 = 0
    
    if block.timestamp < position.lock_until:
        # Early exit - คิด penalty
        penalty = amount * EARLY_EXIT_PENALTY_BPS / 10000
    
    net_amount: uint256 = amount - penalty
    
    self.positions[msg.sender].amount -= amount
    
    ERC20(self.staking_token).transfer(msg.sender, net_amount)
    
    # Penalty ถูกเผาหรือไปที่ treasury
    if penalty > 0:
        ERC20(self.staking_token).transfer(empty(address), penalty)  # burn
    
    log Withdrawn(msg.sender, amount, penalty)

@view
@external
def get_effective_stake(account: address) -> uint256:
    """ดู effective stake (คำนึง multiplier)"""
    pos: StakePosition = self.positions[account]
    return pos.amount * pos.reward_multiplier / 100

@view
@external
def is_locked(account: address) -> bool:
    return block.timestamp < self.positions[account].lock_until

@view
@external
def time_until_unlock(account: address) -> uint256:
    lock_until: uint256 = self.positions[account].lock_until
    if block.timestamp >= lock_until:
        return 0
    return lock_until - block.timestamp
```

---

## ตัวอย่าง: Token Staking Pool

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Token Staking Pool
# @notice Staking Pool สมบูรณ์พร้อมทุก feature

interface ERC20:
    def transfer(to: address, amount: uint256) -> bool: nonpayable
    def transferFrom(from_: address, to: address, amount: uint256) -> bool: nonpayable
    def balanceOf(account: address) -> uint256: view

# ==================== Events ====================

event Staked:
    user: indexed(address)
    amount: uint256
    total: uint256

event Withdrawn:
    user: indexed(address)
    amount: uint256
    remaining: uint256

event RewardClaimed:
    user: indexed(address)
    amount: uint256

event RewardAdded:
    amount: uint256
    duration: uint256
    rate: uint256

event EmergencyWithdraw:
    user: indexed(address)
    amount: uint256

# ==================== State Variables ====================

owner: public(address)

staking_token: public(address)
reward_token: public(address)

total_supply: public(uint256)
balances: public(HashMap[address, uint256])

# Reward distribution
reward_rate: public(uint256)              # token per second
period_finish: public(uint256)            # when reward period ends
last_update_time: public(uint256)
reward_per_token_stored: public(uint256)

user_reward_per_token_paid: public(HashMap[address, uint256])
rewards: public(HashMap[address, uint256])

# Lock system
lock_duration: public(uint256)
stake_time: public(HashMap[address, uint256])

# Pool settings
is_paused: public(bool)
emergency_mode: public(bool)
reward_duration: public(uint256)

# ==================== Constructor ====================

@deploy
def __init__(
    _staking_token: address,
    _reward_token: address,
    _lock_duration: uint256,
    _reward_duration: uint256
):
    self.owner = msg.sender
    self.staking_token = _staking_token
    self.reward_token = _reward_token
    self.lock_duration = _lock_duration
    self.reward_duration = _reward_duration
    self.last_update_time = block.timestamp

# ==================== Reward Logic ====================

@internal
def _last_time_reward_applicable() -> uint256:
    if block.timestamp < self.period_finish:
        return block.timestamp
    return self.period_finish

@internal
def _reward_per_token() -> uint256:
    if self.total_supply == 0:
        return self.reward_per_token_stored
    
    return self.reward_per_token_stored + (
        (_last_time_reward_applicable(self) - self.last_update_time) *
        self.reward_rate * 10**18 / self.total_supply
    )

@internal
def _earned(account: address) -> uint256:
    return (
        self.balances[account] *
        (self._reward_per_token() - self.user_reward_per_token_paid[account]) /
        10**18
    ) + self.rewards[account]

@internal
def _update_reward(account: address):
    self.reward_per_token_stored = self._reward_per_token()
    self.last_update_time = self._last_time_reward_applicable()
    
    if account != empty(address):
        self.rewards[account] = self._earned(account)
        self.user_reward_per_token_paid[account] = self.reward_per_token_stored

# ==================== Staking Functions ====================

@external
@nonreentrant
def stake(amount: uint256):
    """
    @notice Stake tokens
    """
    assert not self.is_paused, "Pool is paused"
    assert amount > 0, "Cannot stake 0"
    
    self._update_reward(msg.sender)
    
    # Record stake time สำหรับ lock
    if self.balances[msg.sender] == 0:
        self.stake_time[msg.sender] = block.timestamp
    
    self.total_supply += amount
    self.balances[msg.sender] += amount
    
    ERC20(self.staking_token).transferFrom(msg.sender, self, amount)
    
    log Staked(msg.sender, amount, self.balances[msg.sender])

@external
@nonreentrant
def withdraw(amount: uint256):
    """
    @notice ถอน staked tokens
    """
    assert not self.is_paused, "Pool is paused"
    assert amount > 0, "Cannot withdraw 0"
    assert self.balances[msg.sender] >= amount, "Insufficient balance"
    
    # ตรวจสอบ lock period
    if self.lock_duration > 0:
        assert block.timestamp >= self.stake_time[msg.sender] + self.lock_duration, \
            "Still locked"
    
    self._update_reward(msg.sender)
    
    self.total_supply -= amount
    self.balances[msg.sender] -= amount
    
    ERC20(self.staking_token).transfer(msg.sender, amount)
    
    log Withdrawn(msg.sender, amount, self.balances[msg.sender])

@external
@nonreentrant
def claim_reward():
    """
    @notice Claim reward tokens
    """
    assert not self.is_paused, "Pool is paused"
    
    self._update_reward(msg.sender)
    
    reward: uint256 = self.rewards[msg.sender]
    
    if reward > 0:
        self.rewards[msg.sender] = 0
        ERC20(self.reward_token).transfer(msg.sender, reward)
        log RewardClaimed(msg.sender, reward)

@external
@nonreentrant
def exit():
    """
    @notice ถอน + claim ในคำสั่งเดียว
    """
    self.withdraw(self.balances[msg.sender])
    self.claim_reward()

# ==================== Admin Functions ====================

@external
def notify_reward_amount(amount: uint256):
    """
    @notice เพิ่ม reward ใน pool
    @dev Admin โอน reward tokens ก่อนแล้ว call ฟังก์ชันนี้
    """
    assert msg.sender == self.owner, "Not owner"
    
    self._update_reward(empty(address))
    
    if block.timestamp >= self.period_finish:
        # period ใหม่
        self.reward_rate = amount / self.reward_duration
    else:
        # ต่อ period เดิม
        remaining: uint256 = self.period_finish - block.timestamp
        leftover: uint256 = remaining * self.reward_rate
        self.reward_rate = (amount + leftover) / self.reward_duration
    
    self.last_update_time = block.timestamp
    self.period_finish = block.timestamp + self.reward_duration
    
    log RewardAdded(amount, self.reward_duration, self.reward_rate)

@external
def set_paused(state: bool):
    assert msg.sender == self.owner
    self.is_paused = state

@external
def set_lock_duration(new_duration: uint256):
    assert msg.sender == self.owner
    self.lock_duration = new_duration

@external
def enable_emergency_mode():
    """Emergency mode ให้ user ถอน token โดยไม่ต้องรอ lock"""
    assert msg.sender == self.owner
    self.emergency_mode = True

@external
@nonreentrant
def emergency_withdraw():
    """ถอน token ในกรณีฉุกเฉิน (สูญเสีย reward)"""
    assert self.emergency_mode, "Not in emergency"
    
    amount: uint256 = self.balances[msg.sender]
    assert amount > 0, "Nothing to withdraw"
    
    self.balances[msg.sender] = 0
    self.total_supply -= amount
    self.rewards[msg.sender] = 0
    
    ERC20(self.staking_token).transfer(msg.sender, amount)
    
    log EmergencyWithdraw(msg.sender, amount)

# ==================== View Functions ====================

@view
@external
def earned(account: address) -> uint256:
    return self._earned(account)

@view
@external
def reward_per_token() -> uint256:
    return self._reward_per_token()

@view
@external
def get_reward_for_duration() -> uint256:
    return self.reward_rate * self.reward_duration

@view
@external
def is_locked(account: address) -> bool:
    if self.lock_duration == 0:
        return False
    return block.timestamp < self.stake_time[account] + self.lock_duration

@view
@external
def time_until_unlock(account: address) -> uint256:
    if self.lock_duration == 0:
        return 0
    unlock_time: uint256 = self.stake_time[account] + self.lock_duration
    if block.timestamp >= unlock_time:
        return 0
    return unlock_time - block.timestamp

@view
@external
def get_apr_bps() -> uint256:
    """APR ใน basis points (10000 = 100%)"""
    if self.total_supply == 0 or self.reward_rate == 0:
        return 0
    
    yearly_rewards: uint256 = self.reward_rate * 31536000
    return yearly_rewards * 10000 / self.total_supply

@view
@external
def get_staker_info(account: address) -> (uint256, uint256, uint256, uint256):
    """Return (staked, earned, stake_time, unlock_time)"""
    staked: uint256 = self.balances[account]
    earned_amount: uint256 = self._earned(account)
    stake_t: uint256 = self.stake_time[account]
    unlock_t: uint256 = stake_t + self.lock_duration
    
    return staked, earned_amount, stake_t, unlock_t
```

---

## Test Code

```python
# tests/test_staking.py
import pytest

ONE_DAY = 86400
ONE_WEEK = 7 * ONE_DAY

@pytest.fixture
def owner(accounts):
    return accounts[0]

@pytest.fixture
def alice(accounts):
    return accounts[1]

@pytest.fixture
def bob(accounts):
    return accounts[2]

@pytest.fixture
def staking_token(owner, project):
    token = project.MockERC20.deploy("Stake", "STK", 18, sender=owner)
    return token

@pytest.fixture
def reward_token(owner, project):
    token = project.MockERC20.deploy("Reward", "RWD", 18, sender=owner)
    return token

@pytest.fixture
def pool(owner, staking_token, reward_token, project):
    return project.StakingPool.deploy(
        staking_token.address,
        reward_token.address,
        ONE_WEEK,    # 1 week lock
        30 * ONE_DAY, # 30 day reward period
        sender=owner
    )

class TestBasicStaking:
    
    def test_stake_tokens(self, pool, alice, staking_token, owner):
        """ทดสอบ stake"""
        amount = 1000 * 10**18
        staking_token.mint(alice.address, amount, sender=owner)
        staking_token.approve(pool.address, amount, sender=alice)
        
        pool.stake(amount, sender=alice)
        
        assert pool.balances(alice.address) == amount
        assert pool.total_supply() == amount
    
    def test_cannot_stake_zero(self, pool, alice):
        """ทดสอบว่า stake 0 ไม่ได้"""
        with pytest.raises(Exception):
            pool.stake(0, sender=alice)
    
    def test_cannot_withdraw_before_lock(self, pool, alice, staking_token, owner):
        """ทดสอบว่าถอนก่อน lock period ไม่ได้"""
        amount = 1000 * 10**18
        staking_token.mint(alice.address, amount, sender=owner)
        staking_token.approve(pool.address, amount, sender=alice)
        pool.stake(amount, sender=alice)
        
        with pytest.raises(Exception):
            pool.withdraw(amount, sender=alice)
    
    def test_withdraw_after_lock(self, pool, alice, staking_token, owner, chain):
        """ทดสอบถอนหลัง lock period"""
        amount = 1000 * 10**18
        staking_token.mint(alice.address, amount, sender=owner)
        staking_token.approve(pool.address, amount, sender=alice)
        pool.stake(amount, sender=alice)
        
        chain.mine(deltatime=ONE_WEEK + 1)
        
        before = staking_token.balanceOf(alice.address)
        pool.withdraw(amount, sender=alice)
        after = staking_token.balanceOf(alice.address)
        
        assert after - before == amount
        assert pool.balances(alice.address) == 0

class TestRewardCalculation:
    
    def test_earn_rewards_over_time(self, pool, alice, staking_token, reward_token, owner, chain):
        """ทดสอบว่าได้รับ reward ตามเวลา"""
        stake_amount = 1000 * 10**18
        reward_amount = 30000 * 10**18  # 30k reward for 30 days
        
        # Setup
        staking_token.mint(alice.address, stake_amount, sender=owner)
        staking_token.approve(pool.address, stake_amount, sender=alice)
        
        reward_token.mint(pool.address, reward_amount, sender=owner)
        pool.notify_reward_amount(reward_amount, sender=owner)
        
        pool.stake(stake_amount, sender=alice)
        
        # เดิน time 1 วัน
        chain.mine(deltatime=ONE_DAY)
        
        # ควรได้รับ reward ประมาณ 1000 tokens (1/30 ของ 30000)
        earned = pool.earned(alice.address)
        assert earned > 0
    
    def test_two_stakers_split_reward(
        self, pool, alice, bob, staking_token, reward_token, owner, chain
    ):
        """ทดสอบ 2 คน split reward"""
        amount = 1000 * 10**18
        reward_amount = 30000 * 10**18
        
        staking_token.mint(alice.address, amount, sender=owner)
        staking_token.mint(bob.address, amount, sender=owner)
        staking_token.approve(pool.address, amount, sender=alice)
        staking_token.approve(pool.address, amount, sender=bob)
        
        reward_token.mint(pool.address, reward_amount, sender=owner)
        pool.notify_reward_amount(reward_amount, sender=owner)
        
        # Both stake same amount
        pool.stake(amount, sender=alice)
        pool.stake(amount, sender=bob)
        
        chain.mine(deltatime=ONE_DAY)
        
        alice_earned = pool.earned(alice.address)
        bob_earned = pool.earned(bob.address)
        
        # Should be roughly equal (within 1% due to timing)
        assert abs(int(alice_earned) - int(bob_earned)) < alice_earned // 100
    
    def test_claim_reward(self, pool, alice, staking_token, reward_token, owner, chain):
        """ทดสอบ claim reward"""
        amount = 1000 * 10**18
        reward_amount = 30000 * 10**18
        
        staking_token.mint(alice.address, amount, sender=owner)
        staking_token.approve(pool.address, amount, sender=alice)
        
        reward_token.mint(pool.address, reward_amount, sender=owner)
        pool.notify_reward_amount(reward_amount, sender=owner)
        
        pool.stake(amount, sender=alice)
        
        chain.mine(deltatime=ONE_DAY)
        
        before = reward_token.balanceOf(alice.address)
        pool.claim_reward(sender=alice)
        after = reward_token.balanceOf(alice.address)
        
        assert after > before

class TestEmergency:
    
    def test_emergency_withdraw(self, pool, alice, staking_token, owner):
        """ทดสอบ emergency withdraw"""
        amount = 1000 * 10**18
        staking_token.mint(alice.address, amount, sender=owner)
        staking_token.approve(pool.address, amount, sender=alice)
        pool.stake(amount, sender=alice)
        
        # Enable emergency
        pool.enable_emergency_mode(sender=owner)
        
        before = staking_token.balanceOf(alice.address)
        pool.emergency_withdraw(sender=alice)
        after = staking_token.balanceOf(alice.address)
        
        assert after - before == amount
        assert pool.balances(alice.address) == 0
```

---

## สรุป

Staking Contract เป็น Pattern สำคัญใน DeFi:

| Feature | Description |
|---------|-------------|
| Basic Staking | Stake และรับ reward |
| Lock Period | เพิ่ม multiplier สำหรับ lock |
| Reward Rate | คำนวณแบบ per-second |
| APR/APY | ดึงข้อมูลสำหรับ UI |
| Emergency | ป้องกันกรณีฉุกเฉิน |

---

[⬅️ Part 034: Auction Contract](part_034_auction.md) | [Part 036: Vesting Contract ➡️](part_036_vesting.md)
