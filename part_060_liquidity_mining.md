# Part 060: Liquidity Mining - Gauge System and Ve-Tokens

## สารบัญ (Table of Contents)
1. บทนำ Liquidity Mining
2. Gauge System (Curve-style)
3. Vote-Escrow Token (veCRV style)
4. Gauge Weight Voting
5. Emission Schedule
6. Bribe System
7. Tests

---

## 1. บทนำ Liquidity Mining

Curve Finance สร้าง Gauge + ve-token system:
- **Gauge**: track liquidity ใน pool แต่ละ pool
- **Vote-Escrow (ve)**: lock tokens รับ voting power
- **Gauge Voting**: เลือกว่า pool ไหนได้ emissions เท่าไหร่
- **Bribes**: โปรเจกต์จ่ายเงินให้ veToken holders vote ให้ gauge ตัวเอง

**Flywheel Effect:**
```
Protocol Revenue -> Bribes -> ve Holders vote -> More Emissions -> More LP -> More Revenue
```

---

## 2. VeToken (Vote-Escrow Token)

```vyper
# @version 0.4.0
# contracts/VotingEscrow.vy
# Vote-Escrow Token (veCRV style)

from vyper.interfaces import ERC20

# Events
event Deposit:
    provider: indexed(address)
    value: uint256
    locktime: uint256
    depositType: uint8
    ts: uint256

event Withdraw:
    provider: indexed(address)
    value: uint256
    ts: uint256

event Supply:
    prevSupply: uint256
    supply: uint256

# Struct for locked balance
struct LockedBalance:
    amount: int128
    end: uint256

# Events constants
CREATE_LOCK_TYPE: constant(uint8) = 1
INCREASE_LOCK_AMOUNT: constant(uint8) = 2
INCREASE_UNLOCK_TIME: constant(uint8) = 3

WEEK: constant(uint256) = 7 * 24 * 3600
MAXTIME: constant(uint256) = 4 * 365 * 24 * 3600  # 4 years
MULTIPLIER: constant(uint256) = 10**18

# State
token: public(address)       # locked token (CRV)
name: public(String[64])
symbol: public(String[32])
version: public(String[32])
decimals: public(uint8)

supply: public(uint256)      # total locked supply
locked: public(HashMap[address, LockedBalance])

epoch: public(uint256)

# Point history for global and per-address bias/slope
struct Point:
    bias: int128     # voting power ปัจจุบัน
    slope: int128    # อัตรา decay ต่อ second
    ts: uint256      # timestamp
    blk: uint256     # block number

pointHistory: HashMap[uint256, Point]  # epoch -> Point
userPointHistory: HashMap[address, HashMap[uint256, Point]]
userPointEpoch: HashMap[address, uint256]
slopeChanges: HashMap[uint256, int128]  # timestamp -> slope change

controller: public(address)
transfersEnabled: public(bool)

@deploy
def __init__(
    _token: address,
    _name: String[64],
    _symbol: String[32],
    _version: String[32]
):
    self.token = _token
    self.name = _name
    self.symbol = _symbol
    self.version = _version
    self.decimals = 18
    self.controller = msg.sender
    
    self.pointHistory[0] = Point({
        bias: 0,
        slope: 0,
        ts: block.timestamp,
        blk: block.number
    })

# ===== Internal: Linear Interpolation =====

@internal
def _checkpoint(addr: address, old_locked: LockedBalance, new_locked: LockedBalance):
    """
    Record global and per-user slopes and biases
    เรียกเมื่อ create/increase/withdraw lock
    """
    u_old: Point = Point({bias: 0, slope: 0, ts: 0, blk: 0})
    u_new: Point = Point({bias: 0, slope: 0, ts: 0, blk: 0})
    
    old_dslope: int128 = 0
    new_dslope: int128 = 0
    _epoch: uint256 = self.epoch
    
    if addr != empty(address):
        # Calculate slopes and biases
        # slope = amount / MAXTIME
        # bias = slope * (end - now)
        
        if old_locked.end > block.timestamp and old_locked.amount > 0:
            u_old.slope = old_locked.amount / convert(MAXTIME, int128)
            u_old.bias = u_old.slope * convert(old_locked.end - block.timestamp, int128)
        
        if new_locked.end > block.timestamp and new_locked.amount > 0:
            u_new.slope = new_locked.amount / convert(MAXTIME, int128)
            u_new.bias = u_new.slope * convert(new_locked.end - block.timestamp, int128)
        
        old_dslope = self.slopeChanges[old_locked.end]
        
        if new_locked.end != 0:
            if new_locked.end == old_locked.end:
                new_dslope = old_dslope
            else:
                new_dslope = self.slopeChanges[new_locked.end]
    
    last_point: Point = self.pointHistory[_epoch]
    
    if last_point.ts == 0:
        last_point = Point({bias: 0, slope: 0, ts: block.timestamp, blk: block.number})
    
    last_checkpoint: uint256 = last_point.ts
    initial_last_point: Point = last_point
    block_slope: uint256 = 0
    
    if block.timestamp > last_point.ts:
        block_slope = MULTIPLIER * (block.number - last_point.blk) / (block.timestamp - last_point.ts)
    
    t_i: uint256 = last_checkpoint / WEEK * WEEK
    
    for i: uint256 in range(255):
        t_i += WEEK
        d_slope: int128 = 0
        
        if t_i > block.timestamp:
            t_i = block.timestamp
        else:
            d_slope = self.slopeChanges[t_i]
        
        last_point.bias -= last_point.slope * convert(t_i - last_checkpoint, int128)
        last_point.slope += d_slope
        
        if last_point.bias < 0:
            last_point.bias = 0
        if last_point.slope < 0:
            last_point.slope = 0
        
        last_checkpoint = t_i
        last_point.ts = t_i
        last_point.blk = initial_last_point.blk + block_slope * (t_i - initial_last_point.ts) / MULTIPLIER
        
        _epoch += 1
        
        if t_i == block.timestamp:
            last_point.blk = block.number
            break
        else:
            self.pointHistory[_epoch] = last_point
    
    self.epoch = _epoch
    
    if addr != empty(address):
        last_point.slope += u_new.slope - u_old.slope
        last_point.bias += u_new.bias - u_old.bias
        
        if last_point.slope < 0:
            last_point.slope = 0
        if last_point.bias < 0:
            last_point.bias = 0
    
    self.pointHistory[_epoch] = last_point
    
    if addr != empty(address):
        if old_locked.end > block.timestamp:
            old_dslope += u_old.slope
            if new_locked.end == old_locked.end:
                old_dslope -= u_new.slope
            self.slopeChanges[old_locked.end] = old_dslope
        
        if new_locked.end > block.timestamp:
            if new_locked.end > old_locked.end:
                new_dslope -= u_new.slope
                self.slopeChanges[new_locked.end] = new_dslope
        
        user_epoch: uint256 = self.userPointEpoch[addr] + 1
        self.userPointEpoch[addr] = user_epoch
        u_new.ts = block.timestamp
        u_new.blk = block.number
        self.userPointHistory[addr][user_epoch] = u_new

# ===== Create/Modify Locks =====

@internal
def _depositFor(
    _addr: address,
    _value: uint256,
    unlock_time: uint256,
    locked_balance: LockedBalance,
    deposit_type: uint8
):
    """Internal deposit function"""
    _locked: LockedBalance = locked_balance
    supply_before: uint256 = self.supply
    
    self.supply = supply_before + _value
    old_locked: LockedBalance = _locked
    
    _locked.amount += convert(_value, int128)
    if unlock_time != 0:
        _locked.end = unlock_time
    
    self.locked[_addr] = _locked
    
    self._checkpoint(_addr, old_locked, _locked)
    
    if _value != 0:
        assert ERC20(self.token).transferFrom(_addr, self, _value), "Transfer failed"
    
    log Deposit(_addr, _value, _locked.end, deposit_type, block.timestamp)
    log Supply(supply_before, supply_before + _value)

@external
@nonreentrant("lock")
def createLock(_value: uint256, _unlock_time: uint256):
    """
    สร้าง lock ใหม่
    
    Parameters:
        _value: จำนวนที่ lock
        _unlock_time: เวลา unlock (rounded to week)
    """
    unlock_time: uint256 = _unlock_time / WEEK * WEEK
    _locked: LockedBalance = self.locked[msg.sender]
    
    assert _value > 0, "Need non-zero value"
    assert _locked.amount == 0, "Withdraw old tokens first"
    assert unlock_time > block.timestamp, "Can only lock future"
    assert unlock_time <= block.timestamp + MAXTIME, "Voting lock too long"
    
    self._depositFor(msg.sender, _value, unlock_time, _locked, CREATE_LOCK_TYPE)

@external
@nonreentrant("lock")
def increaseLockAmount(_value: uint256):
    """เพิ่มจำนวน locked tokens"""
    _locked: LockedBalance = self.locked[msg.sender]
    
    assert _value > 0, "Need non-zero value"
    assert _locked.amount > 0, "No existing lock found"
    assert _locked.end > block.timestamp, "Cannot add to expired lock"
    
    self._depositFor(msg.sender, _value, 0, _locked, INCREASE_LOCK_AMOUNT)

@external
@nonreentrant("lock")
def increaseUnlockTime(_unlock_time: uint256):
    """ขยาย unlock time"""
    _locked: LockedBalance = self.locked[msg.sender]
    unlock_time: uint256 = _unlock_time / WEEK * WEEK
    
    assert _locked.end > block.timestamp, "Lock expired"
    assert _locked.amount > 0, "Nothing is locked"
    assert unlock_time > _locked.end, "Can only increase lock duration"
    assert unlock_time <= block.timestamp + MAXTIME, "Voting lock too long"
    
    self._depositFor(msg.sender, 0, unlock_time, _locked, INCREASE_UNLOCK_TIME)

@external
@nonreentrant("lock")
def withdraw():
    """ถอน locked tokens หลัง lock หมด"""
    _locked: LockedBalance = self.locked[msg.sender]
    
    assert block.timestamp >= _locked.end, "The lock didn't expire"
    
    value: uint256 = convert(_locked.amount, uint256)
    
    old_locked: LockedBalance = _locked
    _locked.end = 0
    _locked.amount = 0
    self.locked[msg.sender] = _locked
    supply_before: uint256 = self.supply
    self.supply = supply_before - value
    
    self._checkpoint(msg.sender, old_locked, _locked)
    
    assert ERC20(self.token).transfer(msg.sender, value), "Transfer failed"
    
    log Withdraw(msg.sender, value, block.timestamp)
    log Supply(supply_before, supply_before - value)

# ===== Voting Power =====

@internal
@view
def _findUserEpoch(
    _addr: address,
    _block: uint256,
    max_epoch: uint256
) -> uint256:
    """Binary search สำหรับหา user epoch ที่ใกล้กับ block"""
    _min: uint256 = 0
    _max: uint256 = max_epoch
    
    for _: uint256 in range(128):
        if _min >= _max:
            break
        _mid: uint256 = (_min + _max + 1) / 2
        if self.userPointHistory[_addr][_mid].blk <= _block:
            _min = _mid
        else:
            _max = _mid - 1
    
    return _min

@external
@view
def balanceOf(addr: address, t: uint256 = block.timestamp) -> uint256:
    """
    Voting power ของ address ณ timestamp t
    Decays linearly ตามเวลา
    """
    _epoch: uint256 = self.userPointEpoch[addr]
    
    if _epoch == 0:
        return 0
    
    last_point: Point = self.userPointHistory[addr][_epoch]
    last_point.bias -= last_point.slope * convert(t - last_point.ts, int128)
    
    if last_point.bias < 0:
        last_point.bias = 0
    
    return convert(last_point.bias, uint256)

@external
@view
def balanceOfAt(addr: address, _block: uint256) -> uint256:
    """Voting power ณ block ที่ระบุ"""
    assert _block <= block.number, "Future block"
    
    _min: uint256 = 0
    _max: uint256 = self.userPointEpoch[addr]
    
    for _: uint256 in range(128):
        if _min >= _max:
            break
        _mid: uint256 = (_min + _max + 1) / 2
        if self.userPointHistory[addr][_mid].blk <= _block:
            _min = _mid
        else:
            _max = _mid - 1
    
    upoint: Point = self.userPointHistory[addr][_min]
    
    max_epoch: uint256 = self.epoch
    _epoch: uint256 = self._findUserEpoch(addr, _block, max_epoch)
    
    point_0: Point = self.pointHistory[_epoch]
    d_block: uint256 = 0
    d_t: uint256 = 0
    
    if _epoch < max_epoch:
        point_1: Point = self.pointHistory[_epoch + 1]
        d_block = point_1.blk - point_0.blk
        d_t = point_1.ts - point_0.ts
    else:
        d_block = block.number - point_0.blk
        d_t = block.timestamp - point_0.ts
    
    block_time: uint256 = point_0.ts
    if d_block != 0:
        block_time += d_t * (_block - point_0.blk) / d_block
    
    upoint.bias -= upoint.slope * convert(block_time - upoint.ts, int128)
    
    if upoint.bias >= 0:
        return convert(upoint.bias, uint256)
    return 0

@external
@view
def totalSupply(t: uint256 = block.timestamp) -> uint256:
    """Total voting power ณ timestamp t"""
    _epoch: uint256 = self.epoch
    last_point: Point = self.pointHistory[_epoch]
    
    return self._supplyAt(last_point, t)

@internal
@view
def _supplyAt(point: Point, t: uint256) -> uint256:
    last_point: Point = point
    t_i: uint256 = last_point.ts / WEEK * WEEK
    
    for i: uint256 in range(255):
        t_i += WEEK
        d_slope: int128 = 0
        
        if t_i > t:
            t_i = t
        else:
            d_slope = self.slopeChanges[t_i]
        
        last_point.bias -= last_point.slope * convert(t_i - last_point.ts, int128)
        
        if t_i == t:
            break
        
        last_point.slope += d_slope
        last_point.ts = t_i
    
    if last_point.bias < 0:
        last_point.bias = 0
    
    return convert(last_point.bias, uint256)
```

---

## 3. Gauge Controller

```vyper
# @version 0.4.0
# contracts/GaugeController.vy
# จัดการ gauge weights สำหรับ emission distribution

interface IVotingEscrow:
    def balanceOf(addr: address, t: uint256) -> uint256: view
    def totalSupply(t: uint256) -> uint256: view
    def locked(addr: address) -> (int128, uint256): view

# Events
event AddType:
    name: String[64]
    typeId: int128

event NewTypeWeight:
    typeId: int128
    time: uint256
    weight: uint256
    totalWeight: uint256

event NewGauge:
    addr: indexed(address)
    gaugeType: int128
    weight: uint256

event VoteForGauge:
    time: uint256
    user: indexed(address)
    gaugeAddr: indexed(address)
    weight: uint256

# Struct
struct Point:
    bias: uint256
    slope: uint256

struct VotedSlope:
    slope: uint256
    power: uint256    # voting power used (0 - 10000)
    end: uint256

# Constants
MULTIPLIER: constant(uint256) = 10**18
WEIGHT_VOTE_DELAY: constant(uint256) = 10 * 86400  # 10 days
MAX_LOCK_DURATION: constant(uint256) = 4 * 365 * 86400

# State
admin: public(address)
futureAdmin: public(address)
token: public(address)     # governance token (CRV)
votingEscrow: public(address)

# Gauge data
nGauges: public(int128)
gauges: public(HashMap[int128, address])
gaugeTypes: public(HashMap[address, int128])

# Type data
nGaugeTypes: public(int128)
gaugeSumWeightsPerType: HashMap[int128, HashMap[uint256, Point]]
gaugeTypeWeights: HashMap[int128, HashMap[uint256, uint256]]
gaugeLastScheduled: HashMap[int128, uint256]

# Global data
points_total: HashMap[uint256, uint256]
time_total: public(uint256)

# Per gauge point
points_weight: HashMap[address, HashMap[uint256, Point]]
time_weight: HashMap[address, uint256]
changes_weight: HashMap[address, HashMap[uint256, uint256]]

# Voting power per user per gauge
vote_user_slopes: HashMap[address, HashMap[address, VotedSlope]]
vote_user_power: public(HashMap[address, uint256])  # total voting power used
last_user_vote: HashMap[address, HashMap[address, uint256]]

WEEK: constant(uint256) = 7 * 86400

@deploy
def __init__(_token: address, _votingEscrow: address):
    self.admin = msg.sender
    self.token = _token
    self.votingEscrow = _votingEscrow
    self.time_total = block.timestamp / WEEK * WEEK

@external
def addType(_name: String[64], weight: uint256 = 0):
    """เพิ่ม gauge type ใหม่"""
    assert msg.sender == self.admin, "Not admin"
    typeId: int128 = self.nGaugeTypes
    self.nGaugeTypes = typeId + 1
    
    if weight != 0:
        _total_weight: uint256 = self._get_total_weight()
        _type_weight: uint256 = weight
        
        self.gaugeTypeWeights[typeId][block.timestamp / WEEK * WEEK] = weight
        _total_weight += _type_weight
        
        log NewTypeWeight(typeId, block.timestamp, weight, _total_weight)
    
    log AddType(_name, typeId)

@external
def addGauge(addr: address, gaugeType: int128, weight: uint256 = 0):
    """เพิ่ม gauge ใหม่"""
    assert msg.sender == self.admin, "Not admin"
    assert gaugeType >= 0 and gaugeType < self.nGaugeTypes, "Invalid type"
    assert self.gaugeTypes[addr] == 0, "Gauge exists"
    
    n: int128 = self.nGauges
    self.nGauges = n + 1
    self.gauges[n] = addr
    self.gaugeTypes[addr] = gaugeType + 1  # +1 เพื่อ distinguish จาก 0
    
    if weight > 0:
        # Set initial weight
        next_time: uint256 = (block.timestamp + WEEK) / WEEK * WEEK
        self.points_weight[addr][next_time].bias = weight
        self.time_weight[addr] = next_time
    
    log NewGauge(addr, gaugeType, weight)

@external
def voteForGaugeWeights(gaugeAddr: address, userWeight: uint256):
    """
    Vote สำหรับ gauge weight
    
    Parameters:
        gaugeAddr: gauge ที่ต้องการ vote
        userWeight: percentage ของ voting power ที่ใช้ (0 - 10000)
    """
    escrow: IVotingEscrow = IVotingEscrow(self.votingEscrow)
    
    slope: uint256 = 0
    lockedEnd: uint256 = 0
    lockedAmount: int128 = 0
    lockedAmount, lockedEnd = escrow.locked(msg.sender)
    
    nextTime: uint256 = (block.timestamp + WEEK) / WEEK * WEEK
    
    assert lockedEnd > nextTime, "Your token lock expires too soon"
    assert userWeight >= 0 and userWeight <= 10000, "You used all your voting power"
    assert block.timestamp >= self.last_user_vote[msg.sender][gaugeAddr] + WEIGHT_VOTE_DELAY, "Cannot vote so often"
    
    gaugeType: int128 = self.gaugeTypes[gaugeAddr] - 1
    assert gaugeType >= 0, "Gauge not added"
    
    # คำนวณ new slope
    userBalance: uint256 = escrow.balanceOf(msg.sender, block.timestamp)
    oldWeight: VotedSlope = self.vote_user_slopes[msg.sender][gaugeAddr]
    oldBias: uint256 = oldWeight.slope * (oldWeight.end - nextTime)
    
    newSlope: uint256 = userBalance * userWeight / 10000 / (lockedEnd - nextTime)
    newBias: uint256 = newSlope * (lockedEnd - nextTime)
    
    oldSumBias: uint256 = self.gaugeSumWeightsPerType[gaugeType][nextTime].bias
    
    # อัพเดท gauge weight
    # (Simplified - production needs full slope/bias tracking)
    
    # อัพเดท user's used voting power
    powerUsed: uint256 = self.vote_user_power[msg.sender]
    powerUsed = powerUsed + userWeight - oldWeight.power
    self.vote_user_power[msg.sender] = powerUsed
    assert powerUsed <= 10000, "Used too much power"
    
    self.vote_user_slopes[msg.sender][gaugeAddr] = VotedSlope({
        slope: newSlope,
        power: userWeight,
        end: lockedEnd
    })
    
    self.last_user_vote[msg.sender][gaugeAddr] = block.timestamp
    
    log VoteForGauge(block.timestamp, msg.sender, gaugeAddr, userWeight)

@external
@view
def gaugeRelativeWeight(addr: address, time: uint256 = block.timestamp) -> uint256:
    """
    Relative weight ของ gauge ณ timestamp
    Returns: weight * 1e18 / total_weight
    """
    return self._gaugeRelativeWeight(addr, time / WEEK * WEEK)

@internal
@view
def _gaugeRelativeWeight(addr: address, time: uint256) -> uint256:
    # Simplified calculation
    # Production: ใช้ full point history
    return MULTIPLIER / convert(self.nGauges, uint256)  # Equal weight for simplicity

@internal
@view
def _get_total_weight() -> uint256:
    total: uint256 = 0
    for i: int128 in range(100):
        if i >= self.nGaugeTypes:
            break
        total += self.gaugeTypeWeights[i][block.timestamp / WEEK * WEEK]
    return total

@external
@view
def getGaugeWeight(addr: address) -> uint256:
    """Weight ของ gauge ปัจจุบัน"""
    return self.points_weight[addr][self.time_weight[addr]].bias
```

---

## 4. Liquidity Gauge

```vyper
# @version 0.4.0
# contracts/LiquidityGauge.vy
# Gauge สำหรับ track liquidity และ distribute emissions

from vyper.interfaces import ERC20

interface IController:
    def period() -> int128: view
    def periodWrite() -> int128: nonpayable
    def periodTimestamp(p: int128) -> uint256: view
    def gaugeRelativeWeight(addr: address, time: uint256) -> uint256: view
    def votingEscrow() -> address: view
    def checkpoint(): nonpayable
    def checkpointGauge(addr: address): nonpayable

interface IMinter:
    def token() -> address: view
    def controller() -> address: view
    def minted(user: address, gauge: address) -> uint256: view
    def mint(gaugeAddr: address): nonpayable

interface IVotingEscrow:
    def balanceOf(addr: address, t: uint256) -> uint256: view
    def totalSupply(t: uint256) -> uint256: view

# Events
event Deposit:
    provider: indexed(address)
    value: uint256

event Withdraw:
    provider: indexed(address)
    value: uint256

event UpdateLiquidityLimit:
    user: indexed(address)
    originalBalance: uint256
    originalSupply: uint256
    workingBalance: uint256
    workingSupply: uint256

# Constants
TOKENLESS_PRODUCTION: constant(uint256) = 40
BOOST_WARMUP: constant(uint256) = 2 * 7 * 86400
WEEK: constant(uint256) = 7 * 86400

# State
minter: public(address)
crvToken: public(address)
controller: public(address)
votingEscrow: public(address)
lpToken: public(address)

balanceOf: public(HashMap[address, uint256])
totalSupply: public(uint256)
futureEpochTime: public(uint256)

workingBalances: public(HashMap[address, uint256])
workingSupply: public(uint256)

# Integration point for rewards
period: public(int128)
periodTimestamp: public(HashMap[int128, uint256])
integrateInvSupply: public(HashMap[int128, uint256])
integrateFraction: public(HashMap[address, uint256])
integrateInvSupplyOf: public(HashMap[address, uint256])
integrateCheckpointOf: public(HashMap[address, uint256])
inflationRate: public(uint256)

@deploy
def __init__(
    _lpToken: address,
    _minter: address,
    _admin: address
):
    self.lpToken = _lpToken
    self.minter = _minter
    self.crvToken = IMinter(_minter).token()
    self.controller = IMinter(_minter).controller()
    self.votingEscrow = IController(self.controller).votingEscrow()
    
    self.periodTimestamp[0] = block.timestamp

@internal
def _updateLiquidityLimit(addr: address, l: uint256, L: uint256):
    """
    คำนวณ working balance โดยใช้ boost จาก veToken
    
    Working balance = min(
        l * (1 - TOKENLESS_PRODUCTION/100) + 
        L * veBalance/veTotalSupply * TOKENLESS_PRODUCTION/100,
        l
    )
    
    ถ้ามี veToken: ได้ boost สูงสุด 2.5x
    ถ้าไม่มี: ได้แค่ 40% ของ proportional share
    """
    voting_balance: uint256 = IVotingEscrow(self.votingEscrow).balanceOf(addr, block.timestamp)
    voting_total: uint256 = IVotingEscrow(self.votingEscrow).totalSupply(block.timestamp)
    
    lim: uint256 = l * TOKENLESS_PRODUCTION / 100
    
    if voting_total > 0:
        lim += L * voting_balance / voting_total * (100 - TOKENLESS_PRODUCTION) / 100
    
    if lim > l:
        lim = l
    
    oldBal: uint256 = self.workingBalances[addr]
    self.workingBalances[addr] = lim
    self.workingSupply = self.workingSupply + lim - oldBal
    
    log UpdateLiquidityLimit(addr, l, L, lim, self.workingSupply)

@internal
def _checkpoint(addr: address):
    """Internal checkpoint สำหรับ reward accrual"""
    controller: IController = IController(self.controller)
    _period: int128 = self.period
    _periodTime: uint256 = self.periodTimestamp[_period]
    
    _integratInvSupply: uint256 = self.integrateInvSupply[_period]
    rate: uint256 = self.inflationRate
    
    prevFutureEpoch: uint256 = self.futureEpochTime
    
    newRate: uint256 = rate
    _workingSupply: uint256 = self.workingSupply
    
    controller.checkpointGauge(self)
    
    _prevWeekTime: uint256 = _periodTime / WEEK * WEEK
    _weekTime: uint256 = (block.timestamp + WEEK) / WEEK * WEEK
    
    if _weekTime != _prevWeekTime:
        pass  # Cross week boundary handling
    
    if _workingSupply > 0:
        _integratInvSupply += rate * (_weekTime - _prevWeekTime) * controller.gaugeRelativeWeight(self, _prevWeekTime) / _workingSupply
    
    _period += 1
    self.period = _period
    self.periodTimestamp[_period] = block.timestamp
    self.integrateInvSupply[_period] = _integratInvSupply
    
    self.integrateFraction[addr] += self.workingBalances[addr] * (
        _integratInvSupply - self.integrateInvSupplyOf[addr]
    ) / 10**18
    self.integrateInvSupplyOf[addr] = _integratInvSupply
    self.integrateCheckpointOf[addr] = block.timestamp

@external
def userCheckpoint(addr: address) -> bool:
    """Checkpoint สำหรับ user"""
    assert msg.sender == addr or msg.sender == self.minter, "Not authorized"
    self._checkpoint(addr)
    self._updateLiquidityLimit(addr, self.balanceOf[addr], self.totalSupply)
    return True

@external
def deposit(value: uint256, addr: address = msg.sender):
    """
    Deposit LP tokens เข้า gauge
    """
    assert value > 0, "Zero value"
    
    self._checkpoint(addr)
    
    self.balanceOf[addr] += value
    self.totalSupply += value
    
    self._updateLiquidityLimit(addr, self.balanceOf[addr], self.totalSupply)
    
    assert ERC20(self.lpToken).transferFrom(msg.sender, self, value), "Transfer failed"
    
    log Deposit(addr, value)

@external
def withdraw(value: uint256):
    """
    ถอน LP tokens จาก gauge
    """
    assert value > 0, "Zero value"
    
    self._checkpoint(msg.sender)
    
    self.balanceOf[msg.sender] -= value
    self.totalSupply -= value
    
    self._updateLiquidityLimit(msg.sender, self.balanceOf[msg.sender], self.totalSupply)
    
    assert ERC20(self.lpToken).transfer(msg.sender, value), "Transfer failed"
    
    log Withdraw(msg.sender, value)

@external
def claimableTokens(addr: address) -> uint256:
    """จำนวน CRV ที่ claim ได้"""
    self._checkpoint(addr)
    return self.integrateFraction[addr] - IMinter(self.minter).minted(addr, self)
```

---

## 5. Bribe System

```vyper
# @version 0.4.0
# contracts/BribeVault.vy
# Bribe system สำหรับ veToken holders

from vyper.interfaces import ERC20

interface IVotingEscrow:
    def balanceOf(addr: address, t: uint256) -> uint256: view
    def totalSupply(t: uint256) -> uint256: view

# Events
event BribeDeposited:
    gauge: indexed(address)
    token: indexed(address)
    briber: indexed(address)
    amount: uint256
    epoch: uint256

event BribeClaimed:
    gauge: indexed(address)
    token: indexed(address)
    claimant: indexed(address)
    amount: uint256
    epoch: uint256

# Struct
struct Bribe:
    token: address
    amount: uint256
    epoch: uint256      # week timestamp
    totalVotes: uint256  # total ve votes for gauge in this epoch
    claimedAmount: uint256

# State
votingEscrow: public(address)
gaugeController: public(address)

bribes: HashMap[address, HashMap[address, HashMap[uint256, Bribe]]]  # gauge -> token -> epoch -> bribe
hasClaimed: HashMap[address, HashMap[address, HashMap[uint256, bool]]]  # voter -> gauge -> epoch -> claimed
nextProposalId: uint256

WEEK: constant(uint256) = 7 * 24 * 3600

@deploy
def __init__(_votingEscrow: address, _gaugeController: address):
    self.votingEscrow = _votingEscrow
    self.gaugeController = _gaugeController

@external
def depositBribe(
    gauge: address,
    token: address,
    amount: uint256
):
    """
    วาง bribe สำหรับ gauge ใน epoch ปัจจุบัน
    Bribers จ่ายเงินให้ veToken holders ที่ vote ให้ gauge ตัวเอง
    
    Parameters:
        gauge: gauge ที่ต้องการให้มี votes เพิ่ม
        token: reward token ที่จะให้
        amount: จำนวน reward
    """
    assert amount > 0, "Zero amount"
    
    epoch: uint256 = block.timestamp / WEEK * WEEK
    
    assert ERC20(token).transferFrom(msg.sender, self, amount), "Transfer failed"
    
    existing: Bribe = self.bribes[gauge][token][epoch]
    
    if existing.token == empty(address):
        self.bribes[gauge][token][epoch] = Bribe({
            token: token,
            amount: amount,
            epoch: epoch,
            totalVotes: IVotingEscrow(self.votingEscrow).totalSupply(epoch),
            claimedAmount: 0
        })
    else:
        self.bribes[gauge][token][epoch].amount += amount
    
    log BribeDeposited(gauge, token, msg.sender, amount, epoch)

@external
def claimBribe(gauge: address, token: address, epoch: uint256) -> uint256:
    """
    Claim bribe rewards
    ต้องเป็น veToken holder ที่ vote ให้ gauge นั้นใน epoch นั้น
    
    Claimable = bribe_amount * user_vote_weight / total_votes
    """
    assert not self.hasClaimed[msg.sender][gauge][epoch], "Already claimed"
    
    bribe: Bribe = self.bribes[gauge][token][epoch]
    assert bribe.token != empty(address), "No bribe"
    assert block.timestamp >= epoch + WEEK, "Epoch not ended"
    
    # ดู vote weight ของ user ณ epoch นั้น
    userVotingPower: uint256 = IVotingEscrow(self.votingEscrow).balanceOf(
        msg.sender,
        epoch
    )
    
    if userVotingPower == 0 or bribe.totalVotes == 0:
        return 0
    
    # คำนวณ proportional reward
    # ใน production: ใช้ actual gauge vote weight ของ user
    # ตอนนี้ simplified โดยใช้ veBalance / veTotalSupply
    claimAmount: uint256 = bribe.amount * userVotingPower / bribe.totalVotes
    
    self.hasClaimed[msg.sender][gauge][epoch] = True
    self.bribes[gauge][token][epoch].claimedAmount += claimAmount
    
    assert ERC20(token).transfer(msg.sender, claimAmount), "Transfer failed"
    
    log BribeClaimed(gauge, token, msg.sender, claimAmount, epoch)
    
    return claimAmount

@external
def claimMultipleBribes(
    gauges: DynArray[address, 10],
    tokens: DynArray[address, 10],
    epochs: DynArray[uint256, 10]
) -> DynArray[uint256, 10]:
    """Claim bribes หลายอัน"""
    amounts: DynArray[uint256, 10] = []
    
    for i: uint256 in range(10):
        if i >= len(gauges):
            break
        
        amount: uint256 = 0
        
        if not self.hasClaimed[msg.sender][gauges[i]][epochs[i]]:
            bribe: Bribe = self.bribes[gauges[i]][tokens[i]][epochs[i]]
            
            if bribe.token != empty(address) and block.timestamp >= epochs[i] + WEEK:
                userVP: uint256 = IVotingEscrow(self.votingEscrow).balanceOf(
                    msg.sender, epochs[i]
                )
                
                if userVP > 0 and bribe.totalVotes > 0:
                    amount = bribe.amount * userVP / bribe.totalVotes
                    self.hasClaimed[msg.sender][gauges[i]][epochs[i]] = True
                    self.bribes[gauges[i]][tokens[i]][epochs[i]].claimedAmount += amount
                    
                    assert ERC20(tokens[i]).transfer(msg.sender, amount), "Transfer failed"
        
        amounts.append(amount)
    
    return amounts

@external
@view
def getBribe(gauge: address, token: address, epoch: uint256) -> Bribe:
    return self.bribes[gauge][token][epoch]

@external
@view
def getClaimable(user: address, gauge: address, token: address, epoch: uint256) -> uint256:
    """จำนวน bribe ที่ user claim ได้"""
    if self.hasClaimed[user][gauge][epoch]:
        return 0
    
    bribe: Bribe = self.bribes[gauge][token][epoch]
    if bribe.token == empty(address) or bribe.totalVotes == 0:
        return 0
    
    userVP: uint256 = IVotingEscrow(self.votingEscrow).balanceOf(user, epoch)
    return bribe.amount * userVP / bribe.totalVotes
```

---

## 6. Tests

```python
# tests/test_liquidity_mining.py
import pytest
from brownie import VotingEscrow, BribeVault, MockERC20, accounts, chain

WEEK = 7 * 24 * 3600
MAXTIME = 4 * 365 * 24 * 3600

@pytest.fixture
def setup():
    owner = accounts[0]
    alice = accounts[1]
    bob = accounts[2]
    
    crv = MockERC20.deploy("CRV Token", "CRV", 18, {"from": owner})
    
    ve_crv = VotingEscrow.deploy(
        crv.address, "Vote-Escrowed CRV", "veCRV", "1.0",
        {"from": owner}
    )
    
    # Mint CRV
    crv.mint(alice, 10**22, {"from": owner})
    crv.mint(bob, 5 * 10**21, {"from": owner})
    
    return owner, alice, bob, crv, ve_crv

def test_create_lock(setup):
    owner, alice, bob, crv, ve_crv = setup
    
    amount = 10**21  # 1000 CRV
    unlock_time = chain.time() + 365 * 24 * 3600
    
    crv.approve(ve_crv.address, amount, {"from": alice})
    ve_crv.createLock(amount, unlock_time, {"from": alice})
    
    vp = ve_crv.balanceOf(alice.address)
    print(f"Alice voting power: {vp / 10**18:.4f} veCRV")
    
    assert vp > 0
    assert vp < amount  # voting power < locked amount (decay)

def test_voting_power_decay(setup):
    owner, alice, bob, crv, ve_crv = setup
    
    amount = 10**21
    unlock_time = chain.time() + 365 * 24 * 3600
    
    crv.approve(ve_crv.address, amount, {"from": alice})
    ve_crv.createLock(amount, unlock_time, {"from": alice})
    
    vp_now = ve_crv.balanceOf(alice.address)
    
    # Skip 6 months
    chain.sleep(180 * 24 * 3600)
    chain.mine(1)
    
    vp_later = ve_crv.balanceOf(alice.address)
    
    print(f"VP at creation: {vp_now / 10**18:.4f}")
    print(f"VP after 6 months: {vp_later / 10**18:.4f}")
    
    assert vp_later < vp_now  # Voting power decays

def test_bribe_system(setup):
    owner, alice, bob, crv, ve_crv = setup
    
    # Alice locks CRV
    amount = 10**21
    unlock_time = chain.time() + 2 * 365 * 24 * 3600
    crv.approve(ve_crv.address, amount, {"from": alice})
    ve_crv.createLock(amount, unlock_time, {"from": alice})
    
    # Deploy bribe vault (simplified - without full gauge controller)
    # In production would need real gauge controller
    
    # Setup bribe token
    usdc = MockERC20.deploy("USDC", "USDC", 6, {"from": owner})
    usdc.mint(bob, 10**9, {"from": owner})
    
    print("Bribe system test setup complete")
    print(f"Alice veCRV balance: {ve_crv.balanceOf(alice.address) / 10**18:.4f}")
```

---

## 7. สรุป

### ve-Token + Gauge Economics:

**1. วงจร ve-Token**
```
Lock CRV -> Get veCRV -> Vote for gauges -> Pools get emissions -> Protocol earns fees -> Bribes
```

**2. Boost Mechanism**
- Minimum boost: 40% ของ proportional rewards
- Maximum boost: 250% (2.5x) ด้วย max veCRV
- Incentivize long-term locking

**3. Bribe Market**
- ทำให้ protocol จ่ายตรงให้ voters
- Voters maximize revenue โดย vote ให้ pool ที่ bribe ดีที่สุด
- สร้าง competition ระหว่าง protocols

**4. Emission Schedule**
- Decreasing over time (inflation control)
- Distributed ตาม gauge weights
- Weekly epochs ป้องกัน manipulation
