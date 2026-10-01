# Part 086: DeFi Economics

## สารบัญ
1. Bonding Curves
2. Liquidity Bootstrapping Pools
3. Price Discovery Mechanisms
4. Game Theory ใน DeFi
5. Mechanism Design
6. MEV และ Arbitrage

---

## 1. Bonding Curves

Bonding curve คือ mathematical relationship ระหว่าง token price และ supply โดย price เพิ่มขึ้นเมื่อมีคน buy และลดลงเมื่อมีคน sell

### Types of Bonding Curves

```
Linear: price = m * supply + b
Exponential: price = a * e^(b * supply)
Polynomial: price = a * supply^n
Logarithmic: price = a * ln(supply) + b
Sigmoid: price = a / (1 + e^(-b*(supply-c)))
```

```vyper
# @version 0.4.0
# @title Bonding Curve Token
# @notice Token ที่ราคากำหนดโดย bonding curve

from vyper.interfaces import ERC20

event TokensPurchased:
    buyer: indexed(address)
    etherIn: uint256
    tokensOut: uint256
    newPrice: uint256

event TokensSold:
    seller: indexed(address)
    tokensIn: uint256
    etherOut: uint256
    newPrice: uint256

# State
name: public(String[64])
symbol: public(String[32])
decimals: public(uint8)
totalSupply: public(uint256)
balanceOf: public(HashMap[address, uint256])

# Bonding curve parameters
reserveRatio: public(uint256)  # Bancor formula reserve ratio (PPM)
reserveBalance: public(uint256)  # ETH in reserve

# Fee
fee: public(uint256)  # basis points
owner: public(address)

PRECISION: constant(uint256) = 10**18
PPM: constant(uint256) = 1000000  # Parts per million for reserve ratio

@deploy
def __init__(
    _name: String[64],
    _symbol: String[32],
    _reserveRatio: uint256,  # e.g., 500000 = 50%
    _fee: uint256
):
    assert _reserveRatio > 0 and _reserveRatio <= PPM, "Invalid reserve ratio"
    
    self.name = _name
    self.symbol = _symbol
    self.decimals = 18
    self.reserveRatio = _reserveRatio
    self.fee = _fee
    self.owner = msg.sender
    
    # Initial state: no tokens, no reserve

@internal
@pure
def _calculatePurchaseReturn(
    supply: uint256,
    reserveBalance: uint256,
    reserveRatio: uint256,
    depositAmount: uint256
) -> uint256:
    """
    Bancor Formula สำหรับ purchase:
    return = supply * ((1 + depositAmount/reserveBalance)^(reserveRatio/PPM) - 1)
    
    Simplified integer version:
    """
    if supply == 0 or reserveBalance == 0:
        # First buy: เหมือน initial price
        return depositAmount  # 1:1 initially
    
    # Simple approximation: return = supply * depositAmount / (reserveBalance / reserveRatio * PPM)
    # ใน production ต้องใช้ precise math library
    adjustedReserve: uint256 = reserveBalance * PPM / reserveRatio
    return supply * depositAmount / adjustedReserve

@internal
@pure
def _calculateSaleReturn(
    supply: uint256,
    reserveBalance: uint256,
    reserveRatio: uint256,
    sellAmount: uint256
) -> uint256:
    """
    Bancor Formula สำหรับ sale:
    return = reserveBalance * (1 - (1 - sellAmount/supply)^(PPM/reserveRatio))
    """
    if supply == 0:
        return 0
    
    # Simple approximation
    return reserveBalance * sellAmount / supply

@payable
@external
def buy(minTokensOut: uint256) -> uint256:
    """ซื้อ tokens ด้วย ETH"""
    assert msg.value > 0, "Send ETH to buy"
    
    # คำนวณ fee
    feeAmount: uint256 = msg.value * self.fee / 10000
    netAmount: uint256 = msg.value - feeAmount
    
    # คำนวณ tokens ที่ได้
    tokensOut: uint256 = self._calculatePurchaseReturn(
        self.totalSupply,
        self.reserveBalance,
        self.reserveRatio,
        netAmount
    )
    
    assert tokensOut >= minTokensOut, "Slippage too high"
    assert tokensOut > 0, "Zero output"
    
    # Update state
    self.reserveBalance += netAmount
    self.totalSupply += tokensOut
    self.balanceOf[msg.sender] += tokensOut
    
    # Send fee to owner
    if feeAmount > 0:
        send(self.owner, feeAmount)
    
    newPrice: uint256 = self.reserveBalance * PRECISION / self.totalSupply
    
    log TokensPurchased(msg.sender, msg.value, tokensOut, newPrice)
    
    return tokensOut

@external
def sell(tokensIn: uint256, minEthOut: uint256) -> uint256:
    """ขาย tokens รับ ETH"""
    assert self.balanceOf[msg.sender] >= tokensIn, "Insufficient balance"
    assert tokensIn > 0, "Zero tokens"
    
    # คำนวณ ETH ที่ได้
    ethOut: uint256 = self._calculateSaleReturn(
        self.totalSupply,
        self.reserveBalance,
        self.reserveRatio,
        tokensIn
    )
    
    # คำนวณ fee
    feeAmount: uint256 = ethOut * self.fee / 10000
    netEthOut: uint256 = ethOut - feeAmount
    
    assert netEthOut >= minEthOut, "Slippage too high"
    assert self.reserveBalance >= ethOut, "Insufficient reserve"
    
    # Update state
    self.balanceOf[msg.sender] -= tokensIn
    self.totalSupply -= tokensIn
    self.reserveBalance -= ethOut
    
    # Send ETH
    send(msg.sender, netEthOut)
    if feeAmount > 0:
        send(self.owner, feeAmount)
    
    newPrice: uint256 = self.reserveBalance * PRECISION / self.totalSupply if self.totalSupply > 0 else 0
    
    log TokensSold(msg.sender, tokensIn, netEthOut, newPrice)
    
    return netEthOut

@view
@external
def spotPrice() -> uint256:
    """ราคาปัจจุบัน (ETH per token)"""
    if self.totalSupply == 0:
        return PRECISION  # Initial price
    return self.reserveBalance * PRECISION / self.totalSupply

@view
@external
def marketCap() -> uint256:
    """Market cap ใน ETH"""
    if self.totalSupply == 0:
        return 0
    return self.totalSupply * self.spotPrice() / PRECISION
```

---

## 2. Liquidity Bootstrapping Pool (LBP)

LBP ใช้เพื่อ launch tokens ใหม่อย่างยุติธรรม โดย weight เปลี่ยนแปลงตามเวลา ทำให้ราคาเริ่มต้นสูงแล้วค่อยๆ ลง

```vyper
# @version 0.4.0
# @title Liquidity Bootstrapping Pool
# @notice Price discovery ด้วย dynamic weights

from vyper.interfaces import ERC20

event WeightUpdated:
    projectWeight: uint256
    fundingWeight: uint256
    timestamp: uint256

event Swap:
    trader: indexed(address)
    tokenIn: indexed(address)
    amountIn: uint256
    tokenOut: indexed(address)
    amountOut: uint256

# LBP Configuration
projectToken: public(address)  # token ที่ project ขาย
fundingToken: public(address)  # USDC/ETH ที่รับ

# Starting weights (project token heavy at start)
startWeightProject: public(uint256)  # e.g., 9000 = 90%
endWeightProject: public(uint256)    # e.g., 5000 = 50%

# Time bounds
startTime: public(uint256)
endTime: public(uint256)

# Reserves
balanceProject: public(uint256)
balanceFunding: public(uint256)

# Trading
tradingEnabled: public(bool)
fee: public(uint256)
owner: public(address)

BASIS_POINTS: constant(uint256) = 10000
PRECISION: constant(uint256) = 10**18

@deploy
def __init__(
    _projectToken: address,
    _fundingToken: address,
    _startWeightProject: uint256,
    _endWeightProject: uint256,
    _startTime: uint256,
    _endTime: uint256,
    _fee: uint256
):
    assert _startWeightProject > _endWeightProject, "Start weight must be > end"
    assert _startWeightProject <= 9500, "Too high start weight"
    assert _endWeightProject >= 500, "Too low end weight"
    assert _endTime > _startTime, "Invalid time range"
    
    self.projectToken = _projectToken
    self.fundingToken = _fundingToken
    self.startWeightProject = _startWeightProject
    self.endWeightProject = _endWeightProject
    self.startTime = _startTime
    self.endTime = _endTime
    self.fee = _fee
    self.owner = msg.sender

@internal
@view
def _currentWeightProject() -> uint256:
    """คำนวณ weight ปัจจุบันของ project token"""
    if block.timestamp <= self.startTime:
        return self.startWeightProject
    if block.timestamp >= self.endTime:
        return self.endWeightProject
    
    elapsed: uint256 = block.timestamp - self.startTime
    duration: uint256 = self.endTime - self.startTime
    
    # Linear interpolation
    weightDiff: uint256 = self.startWeightProject - self.endWeightProject
    reduction: uint256 = weightDiff * elapsed / duration
    
    return self.startWeightProject - reduction

@internal
@view
def _currentWeightFunding() -> uint256:
    return BASIS_POINTS - self._currentWeightProject()

@view
@external
def currentWeights() -> (uint256, uint256):
    """Return (projectWeight, fundingWeight)"""
    wp: uint256 = self._currentWeightProject()
    return wp, BASIS_POINTS - wp

@external
def initialize(projectAmount: uint256, fundingAmount: uint256):
    """Initialize LBP ด้วย initial liquidity"""
    assert msg.sender == self.owner, "Not owner"
    assert self.balanceProject == 0, "Already initialized"
    
    ERC20(self.projectToken).transferFrom(msg.sender, self, projectAmount)
    ERC20(self.fundingToken).transferFrom(msg.sender, self, fundingAmount)
    
    self.balanceProject = projectAmount
    self.balanceFunding = fundingAmount

@external
def enableTrading():
    """เปิด trading"""
    assert msg.sender == self.owner, "Not owner"
    assert block.timestamp >= self.startTime, "Not started"
    self.tradingEnabled = True

@internal
@view
def _getSpotPrice(
    balIn: uint256,
    weightIn: uint256,
    balOut: uint256,
    weightOut: uint256
) -> uint256:
    """
    Balancer formula spot price:
    spotPrice = (balanceIn / weightIn) / (balanceOut / weightOut)
    """
    return (balIn * PRECISION / weightIn) * weightOut / balOut

@internal
@view
def _getAmountOut(
    amountIn: uint256,
    balIn: uint256,
    weightIn: uint256,
    balOut: uint256,
    weightOut: uint256,
    fee: uint256
) -> uint256:
    """
    Balancer formula output:
    amountOut = balOut * (1 - (balIn/(balIn + amountIn*(1-fee)))^(weightIn/weightOut))
    Simplified integer version
    """
    amountInWithFee: uint256 = amountIn * (BASIS_POINTS - fee) / BASIS_POINTS
    
    # Simple approximation ที่ใช้ linear formula
    # Production ต้องใช้ precise power function
    numerator: uint256 = amountInWithFee * balOut
    denominator: uint256 = balIn + amountInWithFee
    
    return numerator / denominator

@external
def swapFundingForProject(
    amountIn: uint256,
    minOut: uint256
) -> uint256:
    """ซื้อ project token ด้วย funding token"""
    assert self.tradingEnabled, "Trading not enabled"
    assert block.timestamp <= self.endTime, "LBP ended"
    assert amountIn > 0, "Zero input"
    
    weightP: uint256 = self._currentWeightProject()
    weightF: uint256 = BASIS_POINTS - weightP
    
    amountOut: uint256 = self._getAmountOut(
        amountIn,
        self.balanceFunding,
        weightF,
        self.balanceProject,
        weightP,
        self.fee
    )
    
    assert amountOut >= minOut, "Slippage"
    assert amountOut < self.balanceProject, "Insufficient project tokens"
    
    ERC20(self.fundingToken).transferFrom(msg.sender, self, amountIn)
    ERC20(self.projectToken).transfer(msg.sender, amountOut)
    
    self.balanceFunding += amountIn
    self.balanceProject -= amountOut
    
    log Swap(msg.sender, self.fundingToken, amountIn, self.projectToken, amountOut)
    
    return amountOut

@external
def endLBP():
    """เก็บ funds หลัง LBP สิ้นสุด"""
    assert msg.sender == self.owner, "Not owner"
    assert block.timestamp >= self.endTime, "LBP not ended"
    
    # คืน project tokens ที่เหลือ
    projectLeft: uint256 = self.balanceProject
    if projectLeft > 0:
        ERC20(self.projectToken).transfer(msg.sender, projectLeft)
        self.balanceProject = 0
    
    # ส่ง funding tokens
    fundingRaised: uint256 = self.balanceFunding
    if fundingRaised > 0:
        ERC20(self.fundingToken).transfer(msg.sender, fundingRaised)
        self.balanceFunding = 0

@view
@external
def currentPrice() -> uint256:
    """ราคา project token ปัจจุบัน ใน funding token"""
    if self.balanceProject == 0:
        return 0
    
    weightP: uint256 = self._currentWeightProject()
    weightF: uint256 = BASIS_POINTS - weightP
    
    return self._getSpotPrice(
        self.balanceFunding,
        weightF,
        self.balanceProject,
        weightP
    )
```

---

## 3. Dutch Auction สำหรับ Token Launch

```vyper
# @version 0.4.0
# @title Dutch Auction Token Sale
# @notice ราคาเริ่มต้นสูงแล้วค่อยๆ ลงจนมีคนซื้อ

from vyper.interfaces import ERC20

event TokenSold:
    buyer: indexed(address)
    amount: uint256
    price: uint256
    totalRaised: uint256

event AuctionEnded:
    finalPrice: uint256
    totalSold: uint256
    totalRaised: uint256

# Auction parameters
token: public(address)
startPrice: public(uint256)    # ราคาเริ่มต้น (ETH per token)
endPrice: public(uint256)      # ราคาต่ำสุด
startTime: public(uint256)
endTime: public(uint256)
totalTokens: public(uint256)   # จำนวน tokens สำหรับขาย

# State
tokensSold: public(uint256)
totalRaised: public(uint256)
finalPrice: public(uint256)
auctionEnded: public(bool)
allocations: HashMap[address, uint256]
owner: public(address)

PRECISION: constant(uint256) = 10**18

@deploy
def __init__(
    _token: address,
    _startPrice: uint256,
    _endPrice: uint256,
    _startTime: uint256,
    _duration: uint256,
    _totalTokens: uint256
):
    assert _startPrice > _endPrice, "Start must be > end"
    assert _duration > 0, "Invalid duration"
    assert _totalTokens > 0, "Invalid total"
    
    self.token = _token
    self.startPrice = _startPrice
    self.endPrice = _endPrice
    self.startTime = _startTime
    self.endTime = _startTime + _duration
    self.totalTokens = _totalTokens
    self.owner = msg.sender

@view
@external
def currentPrice() -> uint256:
    """ราคาปัจจุบัน (ลดลงตามเวลา)"""
    if block.timestamp < self.startTime:
        return self.startPrice
    if block.timestamp >= self.endTime:
        return self.endPrice
    if self.auctionEnded:
        return self.finalPrice
    
    elapsed: uint256 = block.timestamp - self.startTime
    duration: uint256 = self.endTime - self.startTime
    
    priceDiff: uint256 = self.startPrice - self.endPrice
    priceDecline: uint256 = priceDiff * elapsed / duration
    
    return self.startPrice - priceDecline

@payable
@external
def bid():
    """
    วาง bid ที่ราคาปัจจุบัน
    จ่ายตอน bid แต่ refund ถ้าราคาสุดท้ายต่ำกว่า
    """
    assert block.timestamp >= self.startTime, "Not started"
    assert block.timestamp < self.endTime, "Auction ended"
    assert not self.auctionEnded, "Already ended"
    assert msg.value > 0, "Send ETH"
    
    price: uint256 = self.currentPrice()
    tokensWanted: uint256 = msg.value * PRECISION / price
    tokensAvailable: uint256 = self.totalTokens - self.tokensSold
    
    tokensToGet: uint256 = min(tokensWanted, tokensAvailable)
    assert tokensToGet > 0, "No tokens available"
    
    ethUsed: uint256 = tokensToGet * price / PRECISION
    ethRefund: uint256 = msg.value - ethUsed
    
    # Record allocation
    self.allocations[msg.sender] += tokensToGet
    self.tokensSold += tokensToGet
    self.totalRaised += ethUsed
    
    # Refund excess
    if ethRefund > 0:
        send(msg.sender, ethRefund)
    
    log TokenSold(msg.sender, tokensToGet, price, self.totalRaised)
    
    # Check if all sold out
    if self.tokensSold >= self.totalTokens:
        self.auctionEnded = True
        self.finalPrice = price
        log AuctionEnded(price, self.tokensSold, self.totalRaised)

@external
def claimTokens():
    """รับ tokens หลัง auction สิ้นสุด"""
    assert self.auctionEnded or block.timestamp >= self.endTime, "Not ended"
    
    amount: uint256 = self.allocations[msg.sender]
    assert amount > 0, "Nothing to claim"
    
    self.allocations[msg.sender] = 0
    ERC20(self.token).transfer(msg.sender, amount)

@external
def endAuction():
    """สิ้นสุด auction"""
    assert block.timestamp >= self.endTime, "Not ended"
    assert not self.auctionEnded, "Already ended"
    
    self.auctionEnded = True
    self.finalPrice = self.endPrice
    
    log AuctionEnded(self.endPrice, self.tokensSold, self.totalRaised)

@external
def withdrawRaised():
    """เก็บ ETH ที่ raise ได้"""
    assert msg.sender == self.owner, "Not owner"
    assert self.auctionEnded, "Not ended"
    
    amount: uint256 = self.balance
    send(self.owner, amount)
```

---

## 4. Game Theory: Prisoner's Dilemma ใน DeFi

```vyper
# @version 0.4.0
# @title Coordination Game
# @notice ตัวอย่าง coordination game ในรูปแบบ on-chain

event PlayerRegistered:
    player: indexed(address)
    choice: indexed(uint8)  # 0=cooperate, 1=defect

event RoundEnded:
    round: indexed(uint256)
    cooperators: uint256
    defectors: uint256
    cooperatorReward: uint256
    defectorReward: uint256

struct Round:
    startTime: uint256
    endTime: uint256
    totalStake: uint256
    cooperators: uint256
    defectors: uint256
    ended: bool

# Constants
COOPERATION_BONUS: constant(uint256) = 200   # 2x bonus ถ้า majority cooperate
DEFECTION_BONUS: constant(uint256) = 150     # 1.5x ถ้า defect ใน round ที่ majority cooperate
BASELINE_REWARD: constant(uint256) = 100     # 1x baseline

# State
rounds: public(DynArray[Round, 1000])
playerChoices: HashMap[uint256, HashMap[address, uint8]]  # round => player => choice
playerStakes: HashMap[uint256, HashMap[address, uint256]]  # round => player => stake
owner: public(address)
stakeToken: public(address)

@deploy
def __init__(_stakeToken: address):
    self.stakeToken = _stakeToken
    self.owner = msg.sender

@external
def startNewRound(duration: uint256):
    assert msg.sender == self.owner, "Not owner"
    
    self.rounds.append(Round({
        startTime: block.timestamp,
        endTime: block.timestamp + duration,
        totalStake: 0,
        cooperators: 0,
        defectors: 0,
        ended: False
    }))

@external
def participate(roundIndex: uint256, choice: uint8, stakeAmount: uint256):
    """เข้าร่วม round โดย choose cooperate (0) หรือ defect (1)"""
    assert roundIndex < len(self.rounds), "Invalid round"
    assert choice <= 1, "Invalid choice"
    
    round: Round = self.rounds[roundIndex]
    assert block.timestamp < round.endTime, "Round ended"
    assert self.playerChoices[roundIndex][msg.sender] == 0, "Already participated"
    
    ERC20(self.stakeToken).transferFrom(msg.sender, self, stakeAmount)
    
    self.playerChoices[roundIndex][msg.sender] = choice + 1  # 1=cooperate, 2=defect
    self.playerStakes[roundIndex][msg.sender] = stakeAmount
    
    self.rounds[roundIndex].totalStake += stakeAmount
    
    if choice == 0:
        self.rounds[roundIndex].cooperators += 1
    else:
        self.rounds[roundIndex].defectors += 1
    
    log PlayerRegistered(msg.sender, choice)

@external
def endRound(roundIndex: uint256):
    """สิ้นสุด round และ distribute rewards"""
    assert roundIndex < len(self.rounds), "Invalid round"
    round: Round = self.rounds[roundIndex]
    
    assert block.timestamp >= round.endTime, "Round not ended"
    assert not round.ended, "Already ended"
    
    self.rounds[roundIndex].ended = True
    
    # Game Theory: Nash Equilibrium calculation
    cooperators: uint256 = round.cooperators
    defectors: uint256 = round.defectors
    total: uint256 = cooperators + defectors
    
    cooperatorReward: uint256 = 0
    defectorReward: uint256 = 0
    
    if total == 0:
        return
    
    # Majority cooperates scenario
    if cooperators * 2 > total:
        # Nash: cooperators get bonus, defectors exploit but less
        cooperatorReward = 200  # 2x
        defectorReward = 150    # 1.5x (free rider)
    else:
        # Majority defects: prisoners dilemma outcome
        cooperatorReward = 80   # lose stake
        defectorReward = 110    # slight gain

    log RoundEnded(roundIndex, cooperators, defectors, cooperatorReward, defectorReward)

@view
@external
def getRoundInfo(roundIndex: uint256) -> Round:
    return self.rounds[roundIndex]
```

---

## 5. MEV Protection Mechanisms

```vyper
# @version 0.4.0
# @title MEV-Protected AMM
# @notice AMM ที่มี mechanisms ป้องกัน MEV

from vyper.interfaces import ERC20

event CommitHash:
    committer: indexed(address)
    commitHash: indexed(bytes32)
    timestamp: uint256

event RevealSwap:
    swapper: indexed(address)
    amountIn: uint256
    amountOut: uint256

struct SwapCommit:
    blockNumber: uint256
    timestamp: uint256
    revealed: bool

# State
token0: public(address)
token1: public(address)
reserve0: public(uint256)
reserve1: public(uint256)
commitments: HashMap[bytes32, SwapCommit]
COMMIT_REVEAL_DELAY: constant(uint256) = 2  # blocks

owner: public(address)

@deploy
def __init__(_token0: address, _token1: address):
    self.token0 = _token0
    self.token1 = _token1
    self.owner = msg.sender

@external
def commitSwap(commitHash: bytes32):
    """
    Commit ว่าจะทำ swap
    Hash = keccak256(swapper, tokenIn, amountIn, minOut, nonce)
    """
    self.commitments[commitHash] = SwapCommit({
        blockNumber: block.number,
        timestamp: block.timestamp,
        revealed: False
    })
    
    log CommitHash(msg.sender, commitHash, block.timestamp)

@external
def revealSwap(
    tokenIn: address,
    amountIn: uint256,
    minOut: uint256,
    nonce: bytes32
) -> uint256:
    """
    Reveal swap หลังจาก commit
    ต้องรออย่างน้อย COMMIT_REVEAL_DELAY blocks
    """
    # Verify commit
    commitHash: bytes32 = keccak256(
        _abi_encode(msg.sender, tokenIn, amountIn, minOut, nonce)
    )
    
    commit: SwapCommit = self.commitments[commitHash]
    assert commit.blockNumber > 0, "No commitment found"
    assert not commit.revealed, "Already revealed"
    assert block.number >= commit.blockNumber + COMMIT_REVEAL_DELAY, "Too early"
    
    self.commitments[commitHash].revealed = True
    
    # Execute swap
    return self._executeSwap(msg.sender, tokenIn, amountIn, minOut)

@internal
def _executeSwap(
    user: address,
    tokenIn: address,
    amountIn: uint256,
    minOut: uint256
) -> uint256:
    """Execute the actual swap"""
    assert tokenIn == self.token0 or tokenIn == self.token1, "Invalid token"
    
    isToken0: bool = tokenIn == self.token0
    reserveIn: uint256 = self.reserve0 if isToken0 else self.reserve1
    reserveOut: uint256 = self.reserve1 if isToken0 else self.reserve0
    tokenOut: address = self.token1 if isToken0 else self.token0
    
    amountOut: uint256 = amountIn * 997 * reserveOut / (reserveIn * 1000 + amountIn * 997)
    assert amountOut >= minOut, "Slippage"
    
    ERC20(tokenIn).transferFrom(user, self, amountIn)
    ERC20(tokenOut).transfer(user, amountOut)
    
    if isToken0:
        self.reserve0 += amountIn
        self.reserve1 -= amountOut
    else:
        self.reserve1 += amountIn
        self.reserve0 -= amountOut
    
    log RevealSwap(user, amountIn, amountOut)
    
    return amountOut
```

---

## 6. Mechanism Design: Vickrey Auction

```vyper
# @version 0.4.0
# @title Vickrey (Second-Price) Sealed-Bid Auction
# @notice ผู้ชนะจ่ายราคา bid สูงสุดอันดับสอง

event Bid:
    bidder: indexed(address)
    commitHash: indexed(bytes32)

event Revealed:
    bidder: indexed(address)
    amount: uint256

event AuctionEnded:
    winner: indexed(address)
    winningBid: uint256
    secondPrice: uint256

struct BidData:
    commitHash: bytes32
    revealedAmount: uint256
    revealed: bool
    deposited: uint256

# State
auctionItem: public(String[256])
bidPhaseEnd: public(uint256)
revealPhaseEnd: public(uint256)
bids: public(HashMap[address, BidData])
bidders: public(DynArray[address, 1000])

winner: public(address)
highestBid: public(uint256)
secondHighestBid: public(uint256)
auctionEnded: public(bool)
owner: public(address)

@deploy
def __init__(
    _item: String[256],
    _bidDuration: uint256,
    _revealDuration: uint256
):
    self.auctionItem = _item
    self.bidPhaseEnd = block.timestamp + _bidDuration
    self.revealPhaseEnd = self.bidPhaseEnd + _revealDuration
    self.owner = msg.sender

@payable
@external
def commitBid(commitHash: bytes32):
    """
    Commit bid (sealed)
    commitHash = keccak256(amount, secret)
    ต้อง deposit ETH เพื่อ guarantee
    """
    assert block.timestamp < self.bidPhaseEnd, "Bid phase ended"
    assert msg.value > 0, "Must deposit"
    assert self.bids[msg.sender].commitHash == empty(bytes32), "Already bid"
    
    self.bids[msg.sender] = BidData({
        commitHash: commitHash,
        revealedAmount: 0,
        revealed: False,
        deposited: msg.value
    })
    
    self.bidders.append(msg.sender)
    
    log Bid(msg.sender, commitHash)

@external
def revealBid(amount: uint256, secret: bytes32):
    """Reveal bid ใน reveal phase"""
    assert block.timestamp >= self.bidPhaseEnd, "Bid phase not ended"
    assert block.timestamp < self.revealPhaseEnd, "Reveal phase ended"
    
    bid: BidData = self.bids[msg.sender]
    assert not bid.revealed, "Already revealed"
    
    # Verify commitment
    expectedHash: bytes32 = keccak256(_abi_encode(amount, secret))
    assert expectedHash == bid.commitHash, "Incorrect reveal"
    
    self.bids[msg.sender].revealed = True
    self.bids[msg.sender].revealedAmount = amount
    
    log Revealed(msg.sender, amount)

@external
def endAuction():
    """สิ้นสุด auction และหา winner"""
    assert block.timestamp >= self.revealPhaseEnd, "Reveal phase not ended"
    assert not self.auctionEnded, "Already ended"
    
    self.auctionEnded = True
    
    # หา top 2 bids
    for bidder in self.bidders:
        bid: BidData = self.bids[bidder]
        if not bid.revealed:
            continue
        
        amount: uint256 = bid.revealedAmount
        
        if amount > self.highestBid:
            self.secondHighestBid = self.highestBid
            self.highestBid = amount
            self.winner = bidder
        elif amount > self.secondHighestBid:
            self.secondHighestBid = amount
    
    log AuctionEnded(self.winner, self.highestBid, self.secondHighestBid)

@external
def withdraw():
    """
    ถอน deposit:
    - Winner: จ่าย second-price refund ส่วนต่าง
    - ผู้แพ้: คืนทั้งหมด
    """
    assert self.auctionEnded, "Auction not ended"
    
    bid: BidData = self.bids[msg.sender]
    assert bid.deposited > 0, "Nothing to withdraw"
    
    amount: uint256 = bid.deposited
    self.bids[msg.sender].deposited = 0
    
    if msg.sender == self.winner:
        # Winner จ่าย second price (ส่วนที่เกิน refund)
        toRefund: uint256 = 0
        if amount > self.secondHighestBid:
            toRefund = amount - self.secondHighestBid
        
        # ส่งต่อ second price ให้ owner
        payment: uint256 = amount - toRefund
        if payment > 0:
            send(self.owner, payment)
        
        if toRefund > 0:
            send(msg.sender, toRefund)
    else:
        # ผู้แพ้ได้คืนทั้งหมด
        send(msg.sender, amount)
```

---

## 7. สรุป DeFi Economics

### Key Concepts

```
Bonding Curves:
- ราคากำหนดโดย supply/demand function
- Automated market maker ที่ไม่ต้องการ liquidity
- ใช้สำหรับ token launches, NFT pricing

Liquidity Bootstrapping:
- Dynamic weights ทำให้ราคาเริ่มสูงแล้วลง
- ป้องกัน front-running
- สร้าง fair launch

Game Theory:
- ผู้เล่น rational จะ optimize ผลลัพธ์ตัวเอง
- Nash Equilibrium = สภาวะที่ไม่มีใครอยากเปลี่ยน strategy
- Mechanism design สร้าง incentives ที่ align individual และ group

MEV Protection:
- Commit-reveal scheme ป้องกัน front-running
- Transaction ordering ควบคุมได้
- Private mempool (Flashbots)
```

### Vickrey Auction Properties

```
Truth-telling is dominant strategy:
- Bidding ราคาจริงเป็น optimal strategy เสมอ
- ไม่มีประโยชน์ที่จะ overbid หรือ underbid
- Efficient allocation ไปที่ผู้ที่ value สูงสุด
```

---

## แบบฝึกหัด

1. Implement Sigmoid bonding curve
2. สร้าง LBP ที่มี 3 tokens
3. เขียน mechanism ที่จูงใจให้ liquidity providers อยู่นาน
4. Analyze Nash Equilibrium ของ liquidity provision game
5. Implement batch auction ที่ป้องกัน MEV ได้ดีกว่า

---

*จบ Part 086: DeFi Economics*
