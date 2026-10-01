# Part 085: Protocol Economics และ Tokenomics

## สารบัญ
1. Tokenomics Fundamentals
2. Token Distribution Models
3. Inflation/Deflation Mechanisms
4. Fee Structures
5. Value Accrual
6. veToken Model
7. Implementation ตัวอย่าง

---

## 1. Tokenomics Fundamentals

Tokenomics คือการออกแบบระบบเศรษฐกิจของ token ซึ่งกำหนดว่า token มีคุณค่าอย่างไรและจูงใจให้ users ทำ behaviors ที่ต้องการ

### คำถามหลักของ Tokenomics

```
Supply:
- Total supply เท่าไหร่?
- มีการ inflate/deflate?
- Distribution schedule คืออะไร?

Demand:
- ทำไมต้องถือ token?
- มี utility อะไร?
- Governance rights?

Value Accrual:
- Protocol revenue ไปไหน?
- Token stakers ได้อะไร?
- Buyback and burn?
```

---

## 2. Governance Token ที่สมบูรณ์

```vyper
# @version 0.4.0
# @title Protocol Governance Token
# @notice Token ที่มี governance, staking, และ fee distribution

from vyper.interfaces import ERC20

event Transfer:
    sender: indexed(address)
    receiver: indexed(address)
    value: uint256

event Approval:
    owner: indexed(address)
    spender: indexed(address)
    value: uint256

event DelegateChanged:
    delegator: indexed(address)
    fromDelegate: indexed(address)
    toDelegate: indexed(address)

event DelegateVotesChanged:
    delegate: indexed(address)
    previousBalance: uint256
    newBalance: uint256

event TokensBurned:
    burner: indexed(address)
    amount: uint256
    totalBurned: uint256

event InflationMinted:
    recipient: indexed(address)
    amount: uint256
    timestamp: uint256

# Structs
struct Checkpoint:
    fromBlock: uint32
    votes: uint256

# Token basics
name: public(String[64])
symbol: public(String[32])
decimals: public(uint8)
totalSupply: public(uint256)
balanceOf: public(HashMap[address, uint256])
allowance: public(HashMap[address, HashMap[address, uint256]])

# Governance (delegation)
delegates: public(HashMap[address, address])
checkpoints: public(HashMap[address, DynArray[Checkpoint, 1000]])
numCheckpoints: public(HashMap[address, uint256])

# Tokenomics
totalBurned: public(uint256)
maxSupply: public(uint256)
inflationRate: public(uint256)  # basis points per year
lastInflationTime: public(uint256)

owner: public(address)
minter: public(address)
treasury: public(address)

INFLATION_PRECISION: constant(uint256) = 10000
SECONDS_PER_YEAR: constant(uint256) = 365 * 24 * 3600

@deploy
def __init__(
    _name: String[64],
    _symbol: String[32],
    _initialSupply: uint256,
    _maxSupply: uint256,
    _inflationRate: uint256,
    _treasury: address
):
    assert _maxSupply >= _initialSupply, "Max supply too low"
    assert _inflationRate <= 1000, "Inflation too high"  # max 10% per year
    
    self.name = _name
    self.symbol = _symbol
    self.decimals = 18
    self.owner = msg.sender
    self.minter = msg.sender
    self.treasury = _treasury
    self.maxSupply = _maxSupply
    self.inflationRate = _inflationRate
    self.lastInflationTime = block.timestamp
    
    self.totalSupply = _initialSupply
    self.balanceOf[msg.sender] = _initialSupply
    
    log Transfer(empty(address), msg.sender, _initialSupply)

# ===== ERC-20 =====

@external
def transfer(to: address, amount: uint256) -> bool:
    self._transfer(msg.sender, to, amount)
    return True

@external
def approve(spender: address, amount: uint256) -> bool:
    self.allowance[msg.sender][spender] = amount
    log Approval(msg.sender, spender, amount)
    return True

@external
def transferFrom(from_addr: address, to: address, amount: uint256) -> bool:
    if self.allowance[from_addr][msg.sender] != max_value(uint256):
        assert self.allowance[from_addr][msg.sender] >= amount, "Insufficient allowance"
        self.allowance[from_addr][msg.sender] -= amount
    
    self._transfer(from_addr, to, amount)
    return True

@internal
def _transfer(from_addr: address, to: address, amount: uint256):
    assert from_addr != empty(address), "Transfer from zero"
    assert to != empty(address), "Transfer to zero"
    assert self.balanceOf[from_addr] >= amount, "Insufficient balance"
    
    self.balanceOf[from_addr] -= amount
    self.balanceOf[to] += amount
    
    # Update voting power
    self._moveDelegates(
        self.delegates[from_addr],
        self.delegates[to],
        amount
    )
    
    log Transfer(from_addr, to, amount)

# ===== BURNING =====

@external
def burn(amount: uint256):
    """เผา tokens เพื่อลด supply"""
    assert self.balanceOf[msg.sender] >= amount, "Insufficient balance"
    
    self.balanceOf[msg.sender] -= amount
    self.totalSupply -= amount
    self.totalBurned += amount
    
    # Update voting power
    self._moveDelegates(
        self.delegates[msg.sender],
        empty(address),
        amount
    )
    
    log TokensBurned(msg.sender, amount, self.totalBurned)
    log Transfer(msg.sender, empty(address), amount)

# ===== INFLATION =====

@external
def mintInflation():
    """
    Mint tokens ตาม inflation rate
    เรียกได้ทุก 30 วัน
    """
    assert msg.sender == self.minter, "Not minter"
    assert block.timestamp >= self.lastInflationTime + 30 * 24 * 3600, "Too soon"
    
    timePassed: uint256 = block.timestamp - self.lastInflationTime
    inflationAmount: uint256 = (
        self.totalSupply * 
        self.inflationRate * 
        timePassed / 
        (INFLATION_PRECISION * SECONDS_PER_YEAR)
    )
    
    assert self.totalSupply + inflationAmount <= self.maxSupply, "Exceeds max supply"
    
    self.totalSupply += inflationAmount
    self.balanceOf[self.treasury] += inflationAmount
    self.lastInflationTime = block.timestamp
    
    log InflationMinted(self.treasury, inflationAmount, block.timestamp)
    log Transfer(empty(address), self.treasury, inflationAmount)

# ===== GOVERNANCE (DELEGATION) =====

@external
def delegate(delegatee: address):
    """มอบสิทธิ์ vote ให้ address อื่น"""
    self._delegate(msg.sender, delegatee)

@internal
def _delegate(delegator: address, delegatee: address):
    current: address = self.delegates[delegator]
    delegatorBalance: uint256 = self.balanceOf[delegator]
    
    self.delegates[delegator] = delegatee
    
    log DelegateChanged(delegator, current, delegatee)
    
    self._moveDelegates(current, delegatee, delegatorBalance)

@internal
def _moveDelegates(srcRep: address, dstRep: address, amount: uint256):
    if srcRep != dstRep and amount > 0:
        if srcRep != empty(address):
            srcRepNum: uint256 = self.numCheckpoints[srcRep]
            srcRepOld: uint256 = self.checkpoints[srcRep][srcRepNum - 1].votes if srcRepNum > 0 else 0
            srcRepNew: uint256 = srcRepOld - amount
            self._writeCheckpoint(srcRep, srcRepNum, srcRepOld, srcRepNew)
        
        if dstRep != empty(address):
            dstRepNum: uint256 = self.numCheckpoints[dstRep]
            dstRepOld: uint256 = self.checkpoints[dstRep][dstRepNum - 1].votes if dstRepNum > 0 else 0
            dstRepNew: uint256 = dstRepOld + amount
            self._writeCheckpoint(dstRep, dstRepNum, dstRepOld, dstRepNew)

@internal
def _writeCheckpoint(
    delegatee: address,
    nCheckpoints: uint256,
    oldVotes: uint256,
    newVotes: uint256
):
    blockNumber: uint32 = convert(block.number, uint32)
    
    if nCheckpoints > 0 and self.checkpoints[delegatee][nCheckpoints - 1].fromBlock == blockNumber:
        self.checkpoints[delegatee][nCheckpoints - 1].votes = newVotes
    else:
        self.checkpoints[delegatee].append(Checkpoint({
            fromBlock: blockNumber,
            votes: newVotes
        }))
        self.numCheckpoints[delegatee] += 1
    
    log DelegateVotesChanged(delegatee, oldVotes, newVotes)

@view
@external
def getCurrentVotes(account: address) -> uint256:
    nCheckpoints: uint256 = self.numCheckpoints[account]
    return self.checkpoints[account][nCheckpoints - 1].votes if nCheckpoints > 0 else 0

@view
@external
def getPriorVotes(account: address, blockNumber: uint256) -> uint256:
    assert blockNumber < block.number, "Not yet determined"
    
    nCheckpoints: uint256 = self.numCheckpoints[account]
    if nCheckpoints == 0:
        return 0
    
    if self.checkpoints[account][nCheckpoints - 1].fromBlock <= convert(blockNumber, uint32):
        return self.checkpoints[account][nCheckpoints - 1].votes
    
    if self.checkpoints[account][0].fromBlock > convert(blockNumber, uint32):
        return 0
    
    lower: uint256 = 0
    upper: uint256 = nCheckpoints - 1
    
    for _: uint256 in range(1000):
        if lower >= upper:
            break
        center: uint256 = upper - (upper - lower) / 2
        cp: Checkpoint = self.checkpoints[account][center]
        if cp.fromBlock == convert(blockNumber, uint32):
            return cp.votes
        elif cp.fromBlock < convert(blockNumber, uint32):
            lower = center
        else:
            upper = center - 1
    
    return self.checkpoints[account][lower].votes
```

---

## 3. veToken Model (Vote-Escrow)

```vyper
# @version 0.4.0
# @title Vote-Escrow Token (veToken)
# @notice Lock tokens เพื่อรับ voting power ตามระยะเวลา

from vyper.interfaces import ERC20

event Deposit:
    provider: indexed(address)
    value: uint256
    locktime: indexed(uint256)
    timestamp: uint256

event Withdraw:
    provider: indexed(address)
    value: uint256
    timestamp: uint256

event Supply:
    prevSupply: uint256
    supply: uint256

struct LockedBalance:
    amount: int128
    end: uint256

# Constants
WEEK: constant(uint256) = 7 * 86400
MAXTIME: constant(uint256) = 4 * 365 * 86400  # 4 years
MULTIPLIER: constant(uint256) = 10 ** 18

# State
token: public(address)
supply: public(uint256)
locked: public(HashMap[address, LockedBalance])
epoch: public(uint256)
pointHistory: DynArray[int128[2], 100000]  # [bias, slope]
userPointHistory: HashMap[address, DynArray[int128[2], 1000]]
userPointEpoch: HashMap[address, uint256]
slopeChanges: HashMap[uint256, int128]

@deploy
def __init__(_token: address):
    self.token = _token
    self.pointHistory.append([0, 0])

@internal
@view
def _balanceOf(addr: address, t: uint256) -> uint256:
    """คำนวณ veToken balance (voting power) ณ เวลา t"""
    epoch: uint256 = self.userPointEpoch[addr]
    if epoch == 0:
        return 0
    
    lastPoint: int128[2] = self.userPointHistory[addr][epoch]
    lastPoint[0] -= lastPoint[1] * convert(t - 0, int128)  # simplified
    
    if lastPoint[0] < 0:
        return 0
    
    return convert(lastPoint[0], uint256)

@external
def createLock(value: uint256, unlockTime: uint256):
    """
    Lock tokens เพื่อรับ voting power
    Voting power = amount * (time_remaining / MAXTIME)
    """
    unlockTimeRounded: uint256 = (unlockTime / WEEK) * WEEK
    locked: LockedBalance = self.locked[msg.sender]
    
    assert value > 0, "Need non-zero value"
    assert locked.amount == 0, "Withdraw old tokens first"
    assert unlockTimeRounded > block.timestamp, "Can only lock until time in the future"
    assert unlockTimeRounded <= block.timestamp + MAXTIME, "Voting lock can be 4 years max"
    
    ERC20(self.token).transferFrom(msg.sender, self, value)
    
    self.locked[msg.sender] = LockedBalance({
        amount: convert(value, int128),
        end: unlockTimeRounded
    })
    
    prevSupply: uint256 = self.supply
    self.supply += value
    
    log Deposit(msg.sender, value, unlockTimeRounded, block.timestamp)
    log Supply(prevSupply, self.supply)

@external
def increaseLockAmount(value: uint256):
    """เพิ่มจำนวน tokens ใน lock"""
    locked: LockedBalance = self.locked[msg.sender]
    
    assert value > 0, "Need non-zero value"
    assert locked.amount > 0, "No existing lock found"
    assert locked.end > block.timestamp, "Cannot add to expired lock"
    
    ERC20(self.token).transferFrom(msg.sender, self, value)
    
    self.locked[msg.sender].amount += convert(value, int128)
    
    prevSupply: uint256 = self.supply
    self.supply += value
    
    log Deposit(msg.sender, value, locked.end, block.timestamp)
    log Supply(prevSupply, self.supply)

@external
def increaseLockTime(unlockTime: uint256):
    """ขยายระยะเวลา lock"""
    unlockTimeRounded: uint256 = (unlockTime / WEEK) * WEEK
    locked: LockedBalance = self.locked[msg.sender]
    
    assert locked.end > block.timestamp, "Lock expired"
    assert unlockTimeRounded > locked.end, "Can only increase lock duration"
    assert unlockTimeRounded <= block.timestamp + MAXTIME, "Max lock time exceeded"
    
    self.locked[msg.sender].end = unlockTimeRounded
    
    log Deposit(msg.sender, 0, unlockTimeRounded, block.timestamp)

@external
def withdraw():
    """ถอน tokens หลังจาก lock หมดอายุ"""
    locked: LockedBalance = self.locked[msg.sender]
    
    assert block.timestamp >= locked.end, "Lock not expired"
    
    value: uint256 = convert(locked.amount, uint256)
    
    self.locked[msg.sender] = LockedBalance({amount: 0, end: 0})
    
    prevSupply: uint256 = self.supply
    self.supply -= value
    
    ERC20(self.token).transfer(msg.sender, value)
    
    log Withdraw(msg.sender, value, block.timestamp)
    log Supply(prevSupply, self.supply)

@view
@external
def balanceOf(addr: address) -> uint256:
    """Voting power ปัจจุบัน"""
    locked: LockedBalance = self.locked[addr]
    if locked.end <= block.timestamp:
        return 0
    
    timeLeft: uint256 = locked.end - block.timestamp
    return convert(locked.amount, uint256) * timeLeft / MAXTIME

@view
@external
def totalSupply() -> uint256:
    """Total voting power"""
    return self.supply
```

---

## 4. Fee Distribution System

```vyper
# @version 0.4.0
# @title Fee Distributor
# @notice แจก protocol fees ให้ veToken holders

from vyper.interfaces import ERC20

interface IVeToken:
    def balanceOf(addr: address) -> uint256: view
    def totalSupply() -> uint256: view

event FeeDeposited:
    token: indexed(address)
    amount: uint256
    week: indexed(uint256)
    timestamp: uint256

event FeeClaimed:
    user: indexed(address)
    token: indexed(address)
    amount: uint256
    week: uint256

struct TokensPerWeek:
    amount: uint256
    claimedBy: HashMap[address, bool]

WEEK: constant(uint256) = 7 * 86400

# State
veToken: public(address)
tokens: public(DynArray[address, 20])
tokensPerWeek: HashMap[address, HashMap[uint256, uint256]]  # token => week => amount
lastClaim: HashMap[address, HashMap[address, uint256]]  # user => token => last claim week
owner: public(address)

@deploy
def __init__(_veToken: address):
    self.veToken = _veToken
    self.owner = msg.sender

@internal
@view
def _currentWeek() -> uint256:
    return block.timestamp / WEEK

@external
def depositFees(token: address, amount: uint256):
    """ฝาก fees สำหรับแจกใน week นี้"""
    ERC20(token).transferFrom(msg.sender, self, amount)
    
    week: uint256 = self._currentWeek()
    self.tokensPerWeek[token][week] += amount
    
    log FeeDeposited(token, amount, week, block.timestamp)

@external
def claimFees(token: address, weeksToCheck: uint256) -> uint256:
    """Claim accumulated fees"""
    totalClaimed: uint256 = 0
    currentWeek: uint256 = self._currentWeek()
    lastClaimed: uint256 = self.lastClaim[msg.sender][token]
    
    if lastClaimed == 0:
        lastClaimed = currentWeek - weeksToCheck
    
    for w in range(52):  # Max 52 weeks back
        week: uint256 = lastClaimed + convert(w, uint256)
        if week >= currentWeek:
            break
        
        weekFees: uint256 = self.tokensPerWeek[token][week]
        if weekFees == 0:
            continue
        
        # User's share = (user's veToken balance / total veToken supply) * week fees
        # Note: Use balance at start of week for fairness
        totalVe: uint256 = IVeToken(self.veToken).totalSupply()
        if totalVe == 0:
            continue
        
        userVe: uint256 = IVeToken(self.veToken).balanceOf(msg.sender)
        userShare: uint256 = weekFees * userVe / totalVe
        
        totalClaimed += userShare
    
    self.lastClaim[msg.sender][token] = currentWeek
    
    if totalClaimed > 0:
        ERC20(token).transfer(msg.sender, totalClaimed)
        log FeeClaimed(msg.sender, token, totalClaimed, currentWeek - 1)
    
    return totalClaimed

@view
@external
def claimableAmount(user: address, token: address) -> uint256:
    """ดูว่า claim ได้เท่าไหร่"""
    totalClaimable: uint256 = 0
    currentWeek: uint256 = self._currentWeek()
    lastClaimed: uint256 = self.lastClaim[user][token]
    
    if lastClaimed == 0:
        lastClaimed = currentWeek - 52
    
    totalVe: uint256 = IVeToken(self.veToken).totalSupply()
    userVe: uint256 = IVeToken(self.veToken).balanceOf(user)
    
    if totalVe == 0:
        return 0
    
    for w in range(52):
        week: uint256 = lastClaimed + convert(w, uint256)
        if week >= currentWeek:
            break
        
        weekFees: uint256 = self.tokensPerWeek[token][week]
        if weekFees > 0:
            totalClaimable += weekFees * userVe / totalVe
    
    return totalClaimable
```

---

## 5. Buyback and Burn Mechanism

```vyper
# @version 0.4.0
# @title Buyback and Burn
# @notice ใช้ protocol revenue ซื้อ token แล้วเผา

from vyper.interfaces import ERC20

interface IUniswapRouter:
    def swapExactTokensForTokens(
        amountIn: uint256,
        amountOutMin: uint256,
        path: DynArray[address, 3],
        to: address,
        deadline: uint256
    ) -> DynArray[uint256, 3]: nonpayable

event BuybackExecuted:
    revenueToken: indexed(address)
    revenueAmount: uint256
    tokensBought: uint256
    tokensBurned: uint256
    timestamp: uint256

event RevenueDeposited:
    token: indexed(address)
    amount: uint256
    timestamp: uint256

# State
governanceToken: public(address)
router: public(address)
treasury: public(address)
owner: public(address)

revenueTokens: public(DynArray[address, 20])
isRevenueToken: HashMap[address, bool]
totalRevenue: HashMap[address, uint256]
totalBurned: public(uint256)

buybackFrequency: public(uint256)  # minimum seconds between buybacks
lastBuyback: public(uint256)
buybackPercent: public(uint256)  # % ของ revenue ที่ใช้ buyback (basis points)
treasuryPercent: public(uint256)  # % ที่เข้า treasury

BASIS_POINTS: constant(uint256) = 10000

@deploy
def __init__(
    _token: address,
    _router: address,
    _treasury: address,
    _buybackPercent: uint256,
    _frequency: uint256
):
    assert _buybackPercent + 0 <= BASIS_POINTS, "Invalid percentages"
    
    self.governanceToken = _token
    self.router = _router
    self.treasury = _treasury
    self.owner = msg.sender
    self.buybackPercent = _buybackPercent
    self.treasuryPercent = BASIS_POINTS - _buybackPercent
    self.buybackFrequency = _frequency
    self.lastBuyback = block.timestamp

@external
def depositRevenue(token: address, amount: uint256):
    """ฝาก protocol revenue"""
    assert self.isRevenueToken[token], "Token not supported"
    
    ERC20(token).transferFrom(msg.sender, self, amount)
    self.totalRevenue[token] += amount
    
    log RevenueDeposited(token, amount, block.timestamp)

@external
def executeBuyback(token: address, minTokensOut: uint256):
    """
    Execute buyback:
    1. ส่ง % ไป treasury
    2. ใช้ % ซื้อ governance token
    3. เผา governance token ที่ซื้อมา
    """
    assert block.timestamp >= self.lastBuyback + self.buybackFrequency, "Too soon"
    assert self.isRevenueToken[token], "Token not supported"
    
    balance: uint256 = ERC20(token).balanceOf(self)
    assert balance > 0, "No revenue"
    
    # แบ่ง revenue
    buybackAmount: uint256 = balance * self.buybackPercent / BASIS_POINTS
    treasuryAmount: uint256 = balance - buybackAmount
    
    # ส่ง treasury portion
    if treasuryAmount > 0:
        ERC20(token).transfer(self.treasury, treasuryAmount)
    
    # Buyback และ burn
    tokensBought: uint256 = 0
    
    if buybackAmount > 0:
        ERC20(token).approve(self.router, buybackAmount)
        
        path: DynArray[address, 3] = [token, self.governanceToken]
        amounts: DynArray[uint256, 3] = IUniswapRouter(self.router).swapExactTokensForTokens(
            buybackAmount,
            minTokensOut,
            path,
            self,
            block.timestamp + 3600
        )
        
        tokensBought = amounts[len(amounts) - 1]
        
        # Burn tokens
        if tokensBought > 0:
            ERC20(self.governanceToken).transfer(empty(address), tokensBought)
            self.totalBurned += tokensBought
        
        log BuybackExecuted(
            token,
            buybackAmount,
            tokensBought,
            tokensBought,  # all bought tokens are burned
            block.timestamp
        )
    
    self.lastBuyback = block.timestamp

@external
def addRevenueToken(token: address):
    assert msg.sender == self.owner, "Not owner"
    assert not self.isRevenueToken[token], "Already added"
    
    self.revenueTokens.append(token)
    self.isRevenueToken[token] = True
```

---

## 6. Emission Schedule

```vyper
# @version 0.4.0
# @title Token Emission Controller
# @notice จัดการ emission schedule ตาม halving model

event Emission:
    recipient: indexed(address)
    amount: uint256
    epoch: indexed(uint256)
    timestamp: uint256

struct EmissionEpoch:
    startTime: uint256
    endTime: uint256
    emissionPerSecond: uint256
    totalEmitted: uint256

# State
token: public(address)
currentEpoch: public(uint256)
epochs: public(DynArray[EmissionEpoch, 20])
totalEmitted: public(uint256)
lastEmissionTime: public(uint256)
owner: public(address)

# Recipients
recipients: public(DynArray[address, 10])
shares: public(HashMap[address, uint256])  # basis points
totalShares: public(uint256)

@deploy
def __init__(
    _token: address,
    _recipients: DynArray[address, 10],
    _shares: DynArray[uint256, 10]
):
    assert len(_recipients) == len(_shares), "Length mismatch"
    
    self.token = _token
    self.owner = msg.sender
    self.lastEmissionTime = block.timestamp
    
    totalShares: uint256 = 0
    for i: uint256 in range(10):
        if i >= len(_recipients):
            break
        self.recipients.append(_recipients[i])
        self.shares[_recipients[i]] = _shares[i]
        totalShares += _shares[i]
    
    assert totalShares == 10000, "Shares must sum to 10000"
    self.totalShares = totalShares

@external
def addEpoch(
    durationDays: uint256,
    emissionPerDay: uint256
):
    """เพิ่ม emission epoch"""
    assert msg.sender == self.owner, "Not owner"
    
    epochCount: uint256 = len(self.epochs)
    startTime: uint256 = block.timestamp
    
    if epochCount > 0:
        lastEpoch: EmissionEpoch = self.epochs[epochCount - 1]
        startTime = lastEpoch.endTime
    
    duration: uint256 = durationDays * 86400
    
    self.epochs.append(EmissionEpoch({
        startTime: startTime,
        endTime: startTime + duration,
        emissionPerSecond: emissionPerDay / 86400,
        totalEmitted: 0
    }))

@external
def distribute():
    """แจก emission ตาม schedule"""
    assert len(self.epochs) > 0, "No epochs configured"
    
    currentEpochIdx: uint256 = 0
    for i: uint256 in range(20):
        if i >= len(self.epochs):
            break
        ep: EmissionEpoch = self.epochs[i]
        if block.timestamp >= ep.startTime and block.timestamp < ep.endTime:
            currentEpochIdx = i
            break
    
    epoch: EmissionEpoch = self.epochs[currentEpochIdx]
    
    timePassed: uint256 = block.timestamp - self.lastEmissionTime
    emissionAmount: uint256 = timePassed * epoch.emissionPerSecond
    
    if emissionAmount == 0:
        return
    
    self.lastEmissionTime = block.timestamp
    self.totalEmitted += emissionAmount
    self.epochs[currentEpochIdx].totalEmitted += emissionAmount
    
    # แจกให้ recipients ตาม shares
    for recipient in self.recipients:
        share: uint256 = self.shares[recipient]
        recipientAmount: uint256 = emissionAmount * share / 10000
        
        if recipientAmount > 0:
            ERC20(self.token).transfer(recipient, recipientAmount)
            log Emission(recipient, recipientAmount, currentEpochIdx, block.timestamp)

@view
@external
def pendingEmission() -> uint256:
    """ดูว่ามี pending emission เท่าไหร่"""
    if len(self.epochs) == 0:
        return 0
    
    for i: uint256 in range(20):
        if i >= len(self.epochs):
            break
        ep: EmissionEpoch = self.epochs[i]
        if block.timestamp >= ep.startTime and block.timestamp < ep.endTime:
            timePassed: uint256 = block.timestamp - self.lastEmissionTime
            return timePassed * ep.emissionPerSecond
    
    return 0
```

---

## 7. สรุป Tokenomics Best Practices

### Checklist สำหรับ Tokenomics Design

```
Supply:
□ Total supply ชัดเจนและสมเหตุสมผล
□ Inflation rate controlled และ predictable
□ Max supply cap ป้องกัน hyperinflation
□ Burn mechanism offset inflation

Distribution:
□ Team vesting (4 years, 1 year cliff)
□ Investor vesting (2-4 years)
□ Community allocation ใหญ่พอ (50%+)
□ Foundation/treasury สำหรับ sustainability

Value Accrual:
□ Protocol revenue ไปที่ token holders
□ Buyback mechanism หรือ direct distribution
□ Staking rewards competitive

Governance:
□ Token holders มี real power
□ Delegation เพื่อ participation
□ veToken สำหรับ long-term alignment
□ Quorum requirements เหมาะสม
```

### Red Flags

```
⚠️ ไม่มี utility นอกจาก speculation
⚠️ Team allocation > 30% ที่ไม่มี vesting
⚠️ Infinite supply โดยไม่มี burn
⚠️ Governance แต่ admin สามารถ override
⚠️ Revenue ไม่ไปที่ token holders เลย
⚠️ Unrealistic APY ที่ไม่ sustainable
```

---

## แบบฝึกหัด

1. ออกแบบ tokenomics สำหรับ lending protocol
2. Implement halving emission schedule (Bitcoin-like)
3. เขียน veToken ที่รองรับ multiple lock durations
4. สร้าง fee distribution ที่ weighted ตาม lock duration
5. Implement token migration mechanism จาก v1 ไป v2

---

*จบ Part 085: Protocol Economics และ Tokenomics*
