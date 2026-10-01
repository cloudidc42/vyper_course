# Part 059: Advanced Staking with Multiple Reward Tokens

## สารบัญ (Table of Contents)
1. บทนำ Advanced Staking
2. Multi-Reward Staking
3. Boosted Rewards (Lock Multiplier)
4. Reward Snapshots
5. Staking Pool Factory
6. Tests

---

## 1. บทนำ Advanced Staking

Advanced staking system ประกอบด้วย:
- **Multiple reward tokens**: stake ครั้งเดียว รับหลาย rewards
- **Boosted rewards**: lock นานกว่า ได้ multiplier สูงกว่า
- **Snapshots**: calculate rewards ณ จุดใดก็ได้ในอดีต

**ความแตกต่างจาก Simple Staking:**
- Simple: reward token เดียว, ไม่มี multiplier
- Advanced: หลาย reward tokens, lock multiplier, snapshots

---

## 2. Multi-Reward Staking Contract

```vyper
# @version 0.4.0
# contracts/MultiRewardStaking.vy
# Staking พร้อม multiple reward tokens

from vyper.interfaces import ERC20

# Events
event Staked:
    user: indexed(address)
    amount: uint256
    lockDuration: uint256
    boostMultiplier: uint256

event Withdrawn:
    user: indexed(address)
    amount: uint256

event RewardTokenAdded:
    token: indexed(address)
    rewardRate: uint256

event RewardClaimed:
    user: indexed(address)
    token: indexed(address)
    amount: uint256

event RewardRateUpdated:
    token: indexed(address)
    newRate: uint256

# Structs
struct RewardToken:
    token: address
    rewardRate: uint256      # tokens per second (per total staked)
    rewardPerTokenStored: uint256
    lastUpdateTime: uint256
    periodFinish: uint256    # เมื่อไหร่ rewards หมด
    rewardDuration: uint256  # ระยะเวลา reward period

struct UserInfo:
    amount: uint256           # จำนวนที่ stake
    lockEnd: uint256          # timestamp ที่ lock หมด
    boostMultiplier: uint256  # 10000 = 1x, 15000 = 1.5x, 20000 = 2x
    boostedAmount: uint256    # amount * multiplier / 10000

struct UserRewardInfo:
    rewardPerTokenPaid: uint256
    rewards: uint256          # rewards ที่ยังไม่ claim

# Constants
MAX_LOCK_DURATION: constant(uint256) = 4 * 365 * 24 * 3600  # 4 years
MAX_BOOST: constant(uint256) = 40000  # 4x max
MIN_BOOST: constant(uint256) = 10000  # 1x min
BOOST_SCALE: constant(uint256) = 10000
MAX_REWARD_TOKENS: constant(uint256) = 10
SCALE: constant(uint256) = 10**18

# State
stakingToken: public(address)
governance: public(address)

totalSupply: public(uint256)          # ผลรวม staked amounts
totalBoostedSupply: public(uint256)   # ผลรวม boosted amounts

userInfo: HashMap[address, UserInfo]

# Reward tokens
rewardTokens: DynArray[address, 10]
rewardTokenInfo: HashMap[address, RewardToken]
userRewardInfo: HashMap[address, HashMap[address, UserRewardInfo]]  # user -> token -> info

# Lock duration -> boost multiplier
lockBoosts: HashMap[uint256, uint256]  # lock duration -> multiplier

@deploy
def __init__(_stakingToken: address, _governance: address):
    self.stakingToken = _stakingToken
    self.governance = _governance
    
    # ตั้งค่า boost multipliers
    # ไม่ lock: 1x
    self.lockBoosts[0] = 10000
    # 3 months: 1.25x
    self.lockBoosts[90 * 24 * 3600] = 12500
    # 6 months: 1.5x
    self.lockBoosts[180 * 24 * 3600] = 15000
    # 1 year: 2x
    self.lockBoosts[365 * 24 * 3600] = 20000
    # 2 years: 2.5x
    self.lockBoosts[2 * 365 * 24 * 3600] = 25000
    # 4 years: 4x
    self.lockBoosts[4 * 365 * 24 * 3600] = 40000

# ===== Reward Token Management =====

@external
def addRewardToken(
    token: address,
    rewardRate: uint256,
    rewardDuration: uint256
):
    """
    เพิ่ม reward token ใหม่
    
    Parameters:
        token: reward token address
        rewardRate: rewards ต่อ second ต่อ total staked
        rewardDuration: ระยะเวลา reward period
    """
    assert msg.sender == self.governance, "Not governance"
    assert len(self.rewardTokens) < MAX_REWARD_TOKENS, "Too many reward tokens"
    
    # ตรวจสอบว่ายังไม่มี
    for rt: address in self.rewardTokens:
        assert rt != token, "Token exists"
    
    self.rewardTokens.append(token)
    self.rewardTokenInfo[token] = RewardToken({
        token: token,
        rewardRate: rewardRate,
        rewardPerTokenStored: 0,
        lastUpdateTime: block.timestamp,
        periodFinish: block.timestamp + rewardDuration,
        rewardDuration: rewardDuration
    })
    
    log RewardTokenAdded(token, rewardRate)

@external
def notifyRewardAmount(token: address, amount: uint256):
    """
    เติม rewards ใหม่
    ต้องโอน tokens เข้าก่อน
    """
    assert msg.sender == self.governance, "Not governance"
    
    self._updateReward(empty(address))
    
    rtInfo: RewardToken = self.rewardTokenInfo[token]
    
    if block.timestamp >= rtInfo.periodFinish:
        rtInfo.rewardRate = amount / rtInfo.rewardDuration
    else:
        remaining: uint256 = rtInfo.periodFinish - block.timestamp
        leftover: uint256 = remaining * rtInfo.rewardRate
        rtInfo.rewardRate = (amount + leftover) / rtInfo.rewardDuration
    
    rtInfo.lastUpdateTime = block.timestamp
    rtInfo.periodFinish = block.timestamp + rtInfo.rewardDuration
    
    self.rewardTokenInfo[token] = rtInfo
    
    log RewardRateUpdated(token, rtInfo.rewardRate)

# ===== Reward Calculation =====

@internal
@view
def _rewardPerToken(token: address) -> uint256:
    """คำนวณ accumulated reward per boosted token"""
    rtInfo: RewardToken = self.rewardTokenInfo[token]
    
    if self.totalBoostedSupply == 0:
        return rtInfo.rewardPerTokenStored
    
    lastApplicableTime: uint256 = block.timestamp
    if rtInfo.periodFinish < lastApplicableTime:
        lastApplicableTime = rtInfo.periodFinish
    
    elapsed: uint256 = 0
    if lastApplicableTime > rtInfo.lastUpdateTime:
        elapsed = lastApplicableTime - rtInfo.lastUpdateTime
    
    return rtInfo.rewardPerTokenStored + (
        elapsed * rtInfo.rewardRate * SCALE / self.totalBoostedSupply
    )

@internal
@view
def _earned(user: address, token: address) -> uint256:
    """คำนวณ pending rewards"""
    userBoosted: uint256 = self.userInfo[user].boostedAmount
    userRewInfo: UserRewardInfo = self.userRewardInfo[user][token]
    
    return (
        userBoosted * (self._rewardPerToken(token) - userRewInfo.rewardPerTokenPaid) / SCALE +
        userRewInfo.rewards
    )

@internal
def _updateReward(user: address):
    """อัพเดท reward state สำหรับ user"""
    for token: address in self.rewardTokens:
        rtInfo: RewardToken = self.rewardTokenInfo[token]
        newRewardPerToken: uint256 = self._rewardPerToken(token)
        
        rtInfo.rewardPerTokenStored = newRewardPerToken
        
        lastApplicableTime: uint256 = block.timestamp
        if rtInfo.periodFinish < lastApplicableTime:
            lastApplicableTime = rtInfo.periodFinish
        rtInfo.lastUpdateTime = lastApplicableTime
        
        self.rewardTokenInfo[token] = rtInfo
        
        if user != empty(address):
            self.userRewardInfo[user][token].rewards = self._earned(user, token)
            self.userRewardInfo[user][token].rewardPerTokenPaid = newRewardPerToken

# ===== Stake =====

@external
def stake(amount: uint256, lockDuration: uint256):
    """
    Stake tokens พร้อม optional lock
    
    Parameters:
        amount: จำนวนที่ stake
        lockDuration: ระยะเวลา lock (0 = ไม่ lock)
    """
    assert amount > 0, "Zero amount"
    assert lockDuration <= MAX_LOCK_DURATION, "Lock too long"
    
    self._updateReward(msg.sender)
    
    # คำนวณ boost multiplier
    multiplier: uint256 = self._getBoostMultiplier(lockDuration)
    
    # อัพเดท user info
    userdata: UserInfo = self.userInfo[msg.sender]
    
    # ถ้า already staked, lock duration ต้องไม่สั้นกว่าเดิม
    if userdata.amount > 0:
        newLockEnd: uint256 = block.timestamp + lockDuration
        assert newLockEnd >= userdata.lockEnd or lockDuration == 0, "Cannot decrease lock"
    
    boostedAmount: uint256 = amount * multiplier / BOOST_SCALE
    
    userdata.amount += amount
    userdata.boostedAmount += boostedAmount
    if lockDuration > 0:
        userdata.lockEnd = block.timestamp + lockDuration
        userdata.boostMultiplier = multiplier
    
    self.userInfo[msg.sender] = userdata
    
    self.totalSupply += amount
    self.totalBoostedSupply += boostedAmount
    
    # โอน staking tokens เข้า
    assert ERC20(self.stakingToken).transferFrom(msg.sender, self, amount), "Transfer failed"
    
    log Staked(msg.sender, amount, lockDuration, multiplier)

@internal
@view
def _getBoostMultiplier(lockDuration: uint256) -> uint256:
    """คำนวณ boost multiplier จาก lock duration"""
    if lockDuration == 0:
        return MIN_BOOST
    
    # หา multiplier ที่เหมาะสม
    best: uint256 = MIN_BOOST
    
    # ตรวจ lock durations ที่ตั้งค่าไว้
    durations: DynArray[uint256, 10] = [
        0,
        90 * 24 * 3600,
        180 * 24 * 3600,
        365 * 24 * 3600,
        2 * 365 * 24 * 3600,
        4 * 365 * 24 * 3600
    ]
    
    for dur: uint256 in durations:
        if lockDuration >= dur and self.lockBoosts[dur] > best:
            best = self.lockBoosts[dur]
    
    return best

# ===== Withdraw =====

@external
def withdraw(amount: uint256):
    """
    ถอน staked tokens
    ต้อง lock หมดแล้ว
    """
    assert amount > 0, "Zero amount"
    
    userdata: UserInfo = self.userInfo[msg.sender]
    assert userdata.amount >= amount, "Insufficient"
    assert block.timestamp >= userdata.lockEnd, "Still locked"
    
    self._updateReward(msg.sender)
    
    # คำนวณ boosted amount ที่ถอน
    boostedWithdraw: uint256 = amount * userdata.boostedAmount / userdata.amount
    
    userdata.amount -= amount
    userdata.boostedAmount -= boostedWithdraw
    
    if userdata.amount == 0:
        userdata.lockEnd = 0
        userdata.boostMultiplier = 0
    
    self.userInfo[msg.sender] = userdata
    
    self.totalSupply -= amount
    self.totalBoostedSupply -= boostedWithdraw
    
    assert ERC20(self.stakingToken).transfer(msg.sender, amount), "Transfer failed"
    
    log Withdrawn(msg.sender, amount)

# ===== Claim Rewards =====

@external
def claimRewards() -> DynArray[uint256, 10]:
    """
    Claim ทุก reward tokens พร้อมกัน
    
    Returns:
        amounts: จำนวนแต่ละ reward token ที่ได้รับ
    """
    self._updateReward(msg.sender)
    
    amounts: DynArray[uint256, 10] = []
    
    for token: address in self.rewardTokens:
        reward: uint256 = self.userRewardInfo[msg.sender][token].rewards
        
        if reward > 0:
            self.userRewardInfo[msg.sender][token].rewards = 0
            assert ERC20(token).transfer(msg.sender, reward), "Transfer failed"
            log RewardClaimed(msg.sender, token, reward)
        
        amounts.append(reward)
    
    return amounts

@external
def claimRewardToken(token: address) -> uint256:
    """Claim reward token เดียว"""
    self._updateReward(msg.sender)
    
    reward: uint256 = self.userRewardInfo[msg.sender][token].rewards
    
    if reward > 0:
        self.userRewardInfo[msg.sender][token].rewards = 0
        assert ERC20(token).transfer(msg.sender, reward), "Transfer failed"
        log RewardClaimed(msg.sender, token, reward)
    
    return reward

# ===== View Functions =====

@external
@view
def earned(user: address, token: address) -> uint256:
    """Pending rewards สำหรับ user"""
    return self._earned(user, token)

@external
@view
def allEarned(user: address) -> DynArray[uint256, 10]:
    """Pending rewards ทุก token"""
    amounts: DynArray[uint256, 10] = []
    for token: address in self.rewardTokens:
        amounts.append(self._earned(user, token))
    return amounts

@external
@view
def getUserInfo(user: address) -> (uint256, uint256, uint256, uint256):
    """
    Returns: amount, lockEnd, boostMultiplier, boostedAmount
    """
    info: UserInfo = self.userInfo[user]
    return info.amount, info.lockEnd, info.boostMultiplier, info.boostedAmount

@external
@view
def getRewardTokens() -> DynArray[address, 10]:
    return self.rewardTokens

@external
@view
def rewardPerToken(token: address) -> uint256:
    return self._rewardPerToken(token)
```

---

## 3. Lock-Up Boosted Rewards (Vote-Escrow Style)

```vyper
# @version 0.4.0
# contracts/VeBoostStaking.vy
# Vote-Escrow style boosted staking

from vyper.interfaces import ERC20

# Events
event LockCreated:
    user: indexed(address)
    amount: uint256
    unlockTime: uint256
    votingPower: uint256

event LockExtended:
    user: indexed(address)
    newUnlockTime: uint256
    newVotingPower: uint256

event LockIncreased:
    user: indexed(address)
    additionalAmount: uint256
    newVotingPower: uint256

event Unlocked:
    user: indexed(address)
    amount: uint256

# Structs
struct Lock:
    amount: uint256
    end: uint256      # unlock timestamp (rounded to week)

# State
lockToken: public(address)         # token ที่ lock
rewardToken: public(address)       # reward token
governance: public(address)

locks: HashMap[address, Lock]
totalLocked: public(uint256)

# Reward state
rewardRate: public(uint256)
rewardPerTokenStored: public(uint256)
lastUpdateTime: public(uint256)
periodFinish: public(uint256)

userRewardPerTokenPaid: HashMap[address, uint256]
rewards: HashMap[address, uint256]

MAX_LOCK_TIME: constant(uint256) = 4 * 365 * 24 * 3600  # 4 years
WEEK: constant(uint256) = 7 * 24 * 3600
SCALE: constant(uint256) = 10**18

@deploy
def __init__(
    _lockToken: address,
    _rewardToken: address,
    _governance: address
):
    self.lockToken = _lockToken
    self.rewardToken = _rewardToken
    self.governance = _governance
    self.lastUpdateTime = block.timestamp

# ===== Voting Power Calculation =====

@external
@view
def votingPower(user: address) -> uint256:
    """
    Voting power = amount * (remaining_lock_time / MAX_LOCK_TIME)
    
    ยิ่ง lock นานยิ่งมี voting power สูง
    Decay ลงตามเวลาที่ผ่านไป
    """
    return self._votingPower(user, block.timestamp)

@internal
@view
def _votingPower(user: address, timestamp: uint256) -> uint256:
    lock: Lock = self.locks[user]
    
    if lock.end <= timestamp:
        return 0
    
    remaining: uint256 = lock.end - timestamp
    return lock.amount * remaining / MAX_LOCK_TIME

@external
@view
def totalVotingPower() -> uint256:
    """Voting power รวมทั้งหมด (approximate)"""
    # ใน production ต้องใช้ global slope/bias calculation
    return self.totalLocked  # Simplified

# ===== Lock Functions =====

@external
def createLock(amount: uint256, unlockTime: uint256):
    """
    สร้าง lock ใหม่
    
    Parameters:
        amount: จำนวนที่ lock
        unlockTime: เวลา unlock (ต้องเป็นทวีคูณของ week)
    """
    assert self.locks[msg.sender].amount == 0, "Lock exists"
    assert amount > 0, "Zero amount"
    
    # Round to week
    roundedUnlock: uint256 = unlockTime / WEEK * WEEK
    assert roundedUnlock > block.timestamp, "Unlock in past"
    assert roundedUnlock <= block.timestamp + MAX_LOCK_TIME, "Lock too long"
    
    self._updateReward(msg.sender)
    
    assert ERC20(self.lockToken).transferFrom(msg.sender, self, amount), "Transfer failed"
    
    self.locks[msg.sender] = Lock({
        amount: amount,
        end: roundedUnlock
    })
    
    self.totalLocked += amount
    
    vp: uint256 = self._votingPower(msg.sender, block.timestamp)
    log LockCreated(msg.sender, amount, roundedUnlock, vp)

@external
def increaseLockAmount(additionalAmount: uint256):
    """เพิ่ม จำนวนที่ lock"""
    assert self.locks[msg.sender].amount > 0, "No lock"
    assert self.locks[msg.sender].end > block.timestamp, "Lock expired"
    assert additionalAmount > 0, "Zero amount"
    
    self._updateReward(msg.sender)
    
    assert ERC20(self.lockToken).transferFrom(msg.sender, self, additionalAmount), "Transfer failed"
    
    self.locks[msg.sender].amount += additionalAmount
    self.totalLocked += additionalAmount
    
    vp: uint256 = self._votingPower(msg.sender, block.timestamp)
    log LockIncreased(msg.sender, additionalAmount, vp)

@external
def extendLock(newUnlockTime: uint256):
    """ขยาย lock duration"""
    lock: Lock = self.locks[msg.sender]
    assert lock.amount > 0, "No lock"
    
    roundedUnlock: uint256 = newUnlockTime / WEEK * WEEK
    assert roundedUnlock > lock.end, "Must extend"
    assert roundedUnlock <= block.timestamp + MAX_LOCK_TIME, "Too long"
    
    self._updateReward(msg.sender)
    
    self.locks[msg.sender].end = roundedUnlock
    
    vp: uint256 = self._votingPower(msg.sender, block.timestamp)
    log LockExtended(msg.sender, roundedUnlock, vp)

@external
def withdraw():
    """ถอน locked tokens (หลัง lock หมดแล้ว)"""
    lock: Lock = self.locks[msg.sender]
    assert lock.amount > 0, "No lock"
    assert block.timestamp >= lock.end, "Still locked"
    
    self._updateReward(msg.sender)
    
    amount: uint256 = lock.amount
    self.locks[msg.sender] = Lock({amount: 0, end: 0})
    self.totalLocked -= amount
    
    assert ERC20(self.lockToken).transfer(msg.sender, amount), "Transfer failed"
    
    log Unlocked(msg.sender, amount)

# ===== Rewards =====

@internal
@view
def _rewardPerToken() -> uint256:
    if self.totalLocked == 0:
        return self.rewardPerTokenStored
    
    lastTime: uint256 = block.timestamp
    if self.periodFinish < lastTime:
        lastTime = self.periodFinish
    
    elapsed: uint256 = 0
    if lastTime > self.lastUpdateTime:
        elapsed = lastTime - self.lastUpdateTime
    
    return self.rewardPerTokenStored + elapsed * self.rewardRate * SCALE / self.totalLocked

@internal
@view
def _earned(user: address) -> uint256:
    return (
        self.locks[user].amount * (self._rewardPerToken() - self.userRewardPerTokenPaid[user]) / SCALE +
        self.rewards[user]
    )

@internal
def _updateReward(user: address):
    rpt: uint256 = self._rewardPerToken()
    self.rewardPerTokenStored = rpt
    
    lastTime: uint256 = block.timestamp
    if self.periodFinish < lastTime:
        lastTime = self.periodFinish
    self.lastUpdateTime = lastTime
    
    if user != empty(address):
        self.rewards[user] = self._earned(user)
        self.userRewardPerTokenPaid[user] = rpt

@external
def claimReward() -> uint256:
    """Claim pending rewards"""
    self._updateReward(msg.sender)
    
    reward: uint256 = self.rewards[msg.sender]
    
    if reward > 0:
        self.rewards[msg.sender] = 0
        assert ERC20(self.rewardToken).transfer(msg.sender, reward), "Transfer failed"
    
    return reward

@external
def notifyRewardAmount(amount: uint256, duration: uint256):
    """เติม rewards"""
    assert msg.sender == self.governance, "Not governance"
    
    self._updateReward(empty(address))
    
    if block.timestamp >= self.periodFinish:
        self.rewardRate = amount / duration
    else:
        remaining: uint256 = self.periodFinish - block.timestamp
        leftover: uint256 = remaining * self.rewardRate
        self.rewardRate = (amount + leftover) / duration
    
    self.lastUpdateTime = block.timestamp
    self.periodFinish = block.timestamp + duration

@external
@view
def earned(user: address) -> uint256:
    return self._earned(user)

@external
@view
def getLock(user: address) -> (uint256, uint256):
    """Returns: amount, end"""
    lock: Lock = self.locks[user]
    return lock.amount, lock.end
```

---

## 4. Reward Snapshots

```vyper
# @version 0.4.0
# contracts/SnapshotStaking.vy
# Staking พร้อม snapshot สำหรับ governance

from vyper.interfaces import ERC20

# Struct
struct Snapshot:
    fromBlock: uint256
    value: uint256

# Events
event StakeSnapshot:
    user: indexed(address)
    value: uint256
    id: uint256

# State
stakingToken: public(address)
governance: public(address)

totalStakedHistory: DynArray[Snapshot, 10000]
userStakeHistory: HashMap[address, DynArray[Snapshot, 1000]]

_totalStaked: uint256
userStaked: HashMap[address, uint256]

@deploy
def __init__(_stakingToken: address, _governance: address):
    self.stakingToken = _stakingToken
    self.governance = _governance

@internal
def _writeSnapshot(
    history: DynArray[Snapshot, 10000],
    value: uint256
) -> DynArray[Snapshot, 10000]:
    nSnapshots: uint256 = len(history)
    
    newHistory: DynArray[Snapshot, 10000] = history
    
    if nSnapshots > 0 and history[nSnapshots - 1].fromBlock == block.number:
        newHistory[nSnapshots - 1].value = value
    else:
        newHistory.append(Snapshot({fromBlock: block.number, value: value}))
    
    return newHistory

@internal
def _writeUserSnapshot(
    user: address,
    history: DynArray[Snapshot, 1000],
    value: uint256
) -> DynArray[Snapshot, 1000]:
    nSnapshots: uint256 = len(history)
    
    newHistory: DynArray[Snapshot, 1000] = history
    
    if nSnapshots > 0 and history[nSnapshots - 1].fromBlock == block.number:
        newHistory[nSnapshots - 1].value = value
    else:
        newHistory.append(Snapshot({fromBlock: block.number, value: value}))
        log StakeSnapshot(user, value, len(newHistory))
    
    return newHistory

@external
def stake(amount: uint256):
    assert amount > 0, "Zero"
    
    assert ERC20(self.stakingToken).transferFrom(msg.sender, self, amount), "Transfer failed"
    
    self.userStaked[msg.sender] += amount
    self._totalStaked += amount
    
    self.userStakeHistory[msg.sender] = self._writeUserSnapshot(
        msg.sender,
        self.userStakeHistory[msg.sender],
        self.userStaked[msg.sender]
    )
    
    self.totalStakedHistory = self._writeSnapshot(
        self.totalStakedHistory,
        self._totalStaked
    )

@external
def withdraw(amount: uint256):
    assert self.userStaked[msg.sender] >= amount, "Insufficient"
    
    self.userStaked[msg.sender] -= amount
    self._totalStaked -= amount
    
    self.userStakeHistory[msg.sender] = self._writeUserSnapshot(
        msg.sender,
        self.userStakeHistory[msg.sender],
        self.userStaked[msg.sender]
    )
    
    self.totalStakedHistory = self._writeSnapshot(
        self.totalStakedHistory,
        self._totalStaked
    )
    
    assert ERC20(self.stakingToken).transfer(msg.sender, amount), "Transfer failed"

@external
@view
def getPriorStake(account: address, blockNumber: uint256) -> uint256:
    """ดู stake amount ณ block ที่ระบุ"""
    history: DynArray[Snapshot, 1000] = self.userStakeHistory[account]
    
    if len(history) == 0:
        return 0
    
    if history[len(history) - 1].fromBlock <= blockNumber:
        return history[len(history) - 1].value
    
    if history[0].fromBlock > blockNumber:
        return 0
    
    # Binary search
    low: uint256 = 0
    high: uint256 = len(history) - 1
    
    for _: uint256 in range(20):
        if low >= high:
            break
        mid: uint256 = (low + high + 1) / 2
        if history[mid].fromBlock <= blockNumber:
            low = mid
        else:
            high = mid - 1
    
    return history[low].value

@external
@view
def getPriorTotalStaked(blockNumber: uint256) -> uint256:
    """ดู total staked ณ block ที่ระบุ"""
    history: DynArray[Snapshot, 10000] = self.totalStakedHistory
    
    if len(history) == 0:
        return 0
    
    if history[len(history) - 1].fromBlock <= blockNumber:
        return history[len(history) - 1].value
    
    if history[0].fromBlock > blockNumber:
        return 0
    
    low: uint256 = 0
    high: uint256 = len(history) - 1
    
    for _: uint256 in range(20):
        if low >= high:
            break
        mid: uint256 = (low + high + 1) / 2
        if history[mid].fromBlock <= blockNumber:
            low = mid
        else:
            high = mid - 1
    
    return history[low].value
```

---

## 5. Tests

```python
# tests/test_staking.py
import pytest
from brownie import MultiRewardStaking, VeBoostStaking, MockERC20, accounts, chain

@pytest.fixture
def setup():
    owner = accounts[0]
    alice = accounts[1]
    bob = accounts[2]
    
    staking_token = MockERC20.deploy("Staking Token", "STK", 18, {"from": owner})
    reward1 = MockERC20.deploy("Reward1", "RWD1", 18, {"from": owner})
    reward2 = MockERC20.deploy("Reward2", "RWD2", 18, {"from": owner})
    
    staking = MultiRewardStaking.deploy(
        staking_token.address, owner.address, {"from": owner}
    )
    
    # Add reward tokens
    staking.addRewardToken(reward1.address, 10**15, 30 * 24 * 3600, {"from": owner})
    staking.addRewardToken(reward2.address, 5 * 10**14, 30 * 24 * 3600, {"from": owner})
    
    # Fund rewards
    reward1.mint(owner, 10**22, {"from": owner})
    reward2.mint(owner, 10**22, {"from": owner})
    reward1.transfer(staking.address, 10**22, {"from": owner})
    reward2.transfer(staking.address, 10**22, {"from": owner})
    
    staking.notifyRewardAmount(reward1.address, 10**22, {"from": owner})
    staking.notifyRewardAmount(reward2.address, 10**22, {"from": owner})
    
    # Mint staking tokens
    staking_token.mint(alice, 10**22, {"from": owner})
    staking_token.mint(bob, 10**22, {"from": owner})
    
    return owner, alice, bob, staking_token, reward1, reward2, staking

def test_stake_and_earn(setup):
    owner, alice, bob, staking_token, reward1, reward2, staking = setup
    
    staking_token.approve(staking.address, 10**21, {"from": alice})
    staking.stake(10**21, 0, {"from": alice})  # No lock
    
    chain.sleep(7 * 24 * 3600)
    chain.mine(1)
    
    earned1 = staking.earned(alice.address, reward1.address)
    earned2 = staking.earned(alice.address, reward2.address)
    
    print(f"Reward1 earned: {earned1 / 10**18:.4f}")
    print(f"Reward2 earned: {earned2 / 10**18:.4f}")
    
    assert earned1 > 0
    assert earned2 > 0

def test_boost_multiplier(setup):
    owner, alice, bob, staking_token, reward1, reward2, staking = setup
    
    amount = 10**21
    staking_token.approve(staking.address, amount * 2, {"from": alice})
    staking_token.approve(staking.address, amount * 2, {"from": bob})
    
    # Alice stakes without lock
    staking.stake(amount, 0, {"from": alice})
    
    # Bob stakes with 1 year lock (2x boost)
    staking.stake(amount, 365 * 24 * 3600, {"from": bob})
    
    chain.sleep(7 * 24 * 3600)
    chain.mine(1)
    
    alice_earned = staking.earned(alice.address, reward1.address)
    bob_earned = staking.earned(bob.address, reward1.address)
    
    print(f"Alice earned (no lock): {alice_earned / 10**18:.4f}")
    print(f"Bob earned (1yr lock, 2x): {bob_earned / 10**18:.4f}")
    
    # Bob should earn ~2x more than Alice
    assert bob_earned > alice_earned * 18 // 10  # ~1.8x minimum

def test_claim_rewards(setup):
    owner, alice, bob, staking_token, reward1, reward2, staking = setup
    
    staking_token.approve(staking.address, 10**21, {"from": alice})
    staking.stake(10**21, 0, {"from": alice})
    
    chain.sleep(30 * 24 * 3600)
    chain.mine(1)
    
    before1 = reward1.balanceOf(alice.address)
    before2 = reward2.balanceOf(alice.address)
    
    staking.claimRewards({"from": alice})
    
    after1 = reward1.balanceOf(alice.address)
    after2 = reward2.balanceOf(alice.address)
    
    print(f"Claimed reward1: {(after1 - before1) / 10**18:.4f}")
    print(f"Claimed reward2: {(after2 - before2) / 10**18:.4f}")
    
    assert after1 > before1
    assert after2 > before2

def test_ve_staking(setup):
    owner, alice, bob, staking_token, reward1, reward2, staking_multi = setup
    
    ve_staking = VeBoostStaking.deploy(
        staking_token.address,
        reward1.address,
        owner.address,
        {"from": owner}
    )
    
    reward1.mint(owner, 10**22, {"from": owner})
    reward1.transfer(ve_staking.address, 10**22, {"from": owner})
    ve_staking.notifyRewardAmount(10**22, 30 * 24 * 3600, {"from": owner})
    
    amount = 10**21
    staking_token.approve(ve_staking.address, amount, {"from": alice})
    
    unlock_time = chain.time() + 365 * 24 * 3600
    ve_staking.createLock(amount, unlock_time, {"from": alice})
    
    vp = ve_staking.votingPower(alice.address)
    print(f"Alice voting power: {vp / 10**18:.4f}")
    assert vp > 0
    
    chain.sleep(30 * 24 * 3600)
    chain.mine(1)
    
    earned = ve_staking.earned(alice.address)
    print(f"Alice earned: {earned / 10**18:.4f}")
    assert earned > 0
```

---

## 6. สรุป

### Key Concepts:

**1. Boosted Rewards Formula**
```
effective_reward = base_reward * boost_multiplier
boost_multiplier = f(lock_duration)
```

**2. Multi-Token Rewards**
- แต่ละ token มี rewardPerToken แยกกัน
- Update ทุก token เมื่อ stake/withdraw/claim

**3. Vote-Escrow (ve) Model**
- Voting power = amount × (remaining_time / max_time)
- Decay ตามเวลา
- ส่งเสริมการ long-term commitment

**4. Snapshot Mechanism**
- บันทึก stake ณ แต่ละ block
- ใช้สำหรับ governance voting
- Binary search ประสิทธิภาพสูง
