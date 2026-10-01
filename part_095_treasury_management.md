# Part 095: Treasury Management

## สารบัญ
1. [พื้นฐาน DAO Treasury](#treasury-basics)
2. [Diversification Strategies](#diversification)
3. [Yield on Treasury](#yield)
4. [Spending Policies](#spending)
5. [Contributor Payments](#contributors)
6. [Grant Programs](#grants)
7. [TreasuryPolicy Contract](#treasury-policy)

---

## 1. พื้นฐาน DAO Treasury {#treasury-basics}

DAO Treasury คือกองทุนส่วนกลางที่ community ควบคุม มีแหล่งที่มาจาก:
- **Protocol fees** - ค่าธรรมเนียมจาก protocol
- **Token sales** - รายได้จากการขาย tokens
- **Grants** - เงินทุนจาก ecosystem (Ethereum Foundation, etc.)
- **Investments** - ผลตอบแทนจาก treasury management

### หลักการ Treasury Management

1. **Runway First** - ต้องมี operational runway อย่างน้อย 2-3 ปี
2. **Diversification** - ไม่ถือ native token 100%
3. **Yield Generation** - ทำให้ treasury เติบโต
4. **Transparency** - รายงานสม่ำเสมอ
5. **Governance Oversight** - ทุก major spending ต้องผ่าน vote

---

## 2. Diversification Strategies {#diversification}

### ตัวอย่าง Treasury Allocation

```
Treasury Allocation Model (ตัวอย่าง)

Stablecoins (40%)
  - USDC: 20%
  - DAI: 10%  
  - USDT: 10%

Blue-chip Assets (30%)
  - ETH: 20%
  - WBTC: 10%

Protocol Token (20%)
  - Native token: 20%

Yield-generating (10%)
  - Stablecoin lending: 5%
  - LP positions: 5%
```

---

## 3. TreasuryPolicy Contract {#treasury-policy}

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0

"""
@title TreasuryPolicy - ระบบจัดการ Treasury ของ DAO
@notice จัดการ allocation, spending limits, contributor payments และ grants
@dev ทุก action ผ่าน governance หรือ multi-sig
"""

from vyper.interfaces import ERC20

# ============================================================
# Interfaces
# ============================================================

interface IYieldStrategy:
    def deposit(_token: address, _amount: uint256): nonpayable
    def withdraw(_token: address, _amount: uint256): nonpayable
    def getBalance(_token: address) -> uint256: view
    def getYield(_token: address) -> uint256: view

interface IPriceOracle:
    def getPrice(_token: address) -> uint256: view  # Returns USD price, 8 decimals

# ============================================================
# Events
# ============================================================

event TreasuryDeposit:
    token: indexed(address)
    amount: uint256
    source: String[50]
    timestamp: uint256

event TreasuryWithdrawal:
    token: indexed(address)
    amount: uint256
    recipient: indexed(address)
    reason: String[100]
    approvedBy: address
    timestamp: uint256

event AllocationUpdated:
    token: indexed(address)
    targetPercent: uint256
    timestamp: uint256

event ContributorPaid:
    contributor: indexed(address)
    token: indexed(address)
    amount: uint256
    period: String[50]
    timestamp: uint256

event GrantApproved:
    grantId: indexed(uint256)
    recipient: indexed(address)
    amount: uint256
    token: address
    timestamp: uint256

event GrantDisbursed:
    grantId: indexed(uint256)
    milestone: uint256
    amount: uint256
    timestamp: uint256

event YieldHarvested:
    strategy: indexed(address)
    token: indexed(address)
    amount: uint256
    timestamp: uint256

event SpendingLimitUpdated:
    category: String[50]
    oldLimit: uint256
    newLimit: uint256

event RebalanceExecuted:
    timestamp: uint256
    actions: uint256

# ============================================================
# Structs
# ============================================================

struct AssetAllocation:
    token: address
    targetPercent: uint256   # basis points (10000 = 100%)
    minPercent: uint256      # minimum allocation
    maxPercent: uint256      # maximum allocation
    currentValue: uint256    # in USD (8 decimals)
    isStablecoin: bool
    isYieldBearing: bool

struct ContributorInfo:
    name: String[50]
    address_: address
    role: String[50]
    monthlyUSD: uint256      # monthly payment in USD (8 decimals)
    token: address           # payment token
    startDate: uint256
    active: bool
    totalPaid: uint256

struct Grant:
    id: uint256
    title: String[100]
    recipient: address
    totalAmount: uint256
    token: address
    milestonesCount: uint256
    milestonesCompleted: uint256
    disbursedAmount: uint256
    approvedAt: uint256
    completedAt: uint256
    active: bool
    description: String[300]

struct GrantMilestone:
    grantId: uint256
    milestoneIndex: uint256
    amount: uint256
    description: String[200]
    completedAt: uint256
    paid: bool

struct SpendingLimit:
    category: String[50]
    dailyLimit: uint256      # USD (8 decimals)
    monthlyLimit: uint256    # USD (8 decimals)
    dailySpent: uint256
    monthlySpent: uint256
    lastDayReset: uint256
    lastMonthReset: uint256

struct YieldStrategy:
    strategy: address
    tokens: DynArray[address, 5]
    allocatedUSD: uint256
    earnedYield: uint256
    lastHarvest: uint256
    active: bool
    name: String[50]

# ============================================================
# Constants
# ============================================================

MAX_ASSETS: constant(uint256) = 20
MAX_CONTRIBUTORS: constant(uint256) = 100
MAX_GRANTS: constant(uint256) = 200
MAX_STRATEGIES: constant(uint256) = 10
BASIS_POINTS: constant(uint256) = 10000

# ============================================================
# State Variables
# ============================================================

# Access control
owner: public(address)
governance: public(address)
treasurer: public(address)
operators: public(HashMap[address, bool])

# Asset management
trackedAssets: public(DynArray[address, MAX_ASSETS])
assetAllocations: public(HashMap[address, AssetAllocation])
priceOracle: public(address)
totalTreasuryUSD: public(uint256)
lastValuationTime: public(uint256)

# Contributors
contributors: public(DynArray[address, MAX_CONTRIBUTORS])
contributorInfo: public(HashMap[address, ContributorInfo])
lastPaymentTime: public(HashMap[address, uint256])

# Grants
grants: public(HashMap[uint256, Grant])
grantCount: public(uint256)
grantMilestones: public(HashMap[uint256, HashMap[uint256, GrantMilestone]])

# Spending limits
spendingLimits: public(HashMap[String[50], SpendingLimit])

# Yield strategies
yieldStrategies: public(DynArray[address, MAX_STRATEGIES])
strategyInfo: public(HashMap[address, YieldStrategy])
totalYieldEarned: public(uint256)

# Rebalancing
lastRebalanceTime: public(uint256)
REBALANCE_THRESHOLD: public(uint256)  # deviation that triggers rebalance (basis points)

# Emergency
emergencyWithdrawalEnabled: public(bool)
emergencyRecipient: public(address)

# ============================================================
# Constructor
# ============================================================

@deploy
def __init__(
    _governance: address,
    _treasurer: address,
    _oracle: address
):
    """
    @notice Initialize treasury management system
    @param _governance Governance contract (multi-sig or Governor)
    @param _treasurer Treasurer address for day-to-day ops
    @param _oracle Price oracle for valuations
    """
    assert _governance != empty(address), "Invalid governance"
    assert _treasurer != empty(address), "Invalid treasurer"
    
    self.owner = msg.sender
    self.governance = _governance
    self.treasurer = _treasurer
    self.priceOracle = _oracle
    
    self.REBALANCE_THRESHOLD = 500  # 5% deviation triggers rebalance
    
    # Initialize spending limits
    self.spendingLimits["operations"] = SpendingLimit({
        category: "operations",
        dailyLimit: 10000 * 10**8,     # $10k/day
        monthlyLimit: 100000 * 10**8,  # $100k/month
        dailySpent: 0,
        monthlySpent: 0,
        lastDayReset: block.timestamp,
        lastMonthReset: block.timestamp
    })
    
    self.spendingLimits["grants"] = SpendingLimit({
        category: "grants",
        dailyLimit: 50000 * 10**8,     # $50k/day
        monthlyLimit: 500000 * 10**8,  # $500k/month
        dailySpent: 0,
        monthlySpent: 0,
        lastDayReset: block.timestamp,
        lastMonthReset: block.timestamp
    })

# ============================================================
# Asset Management
# ============================================================

@external
def addAsset(
    _token: address,
    _targetPercent: uint256,
    _minPercent: uint256,
    _maxPercent: uint256,
    _isStablecoin: bool,
    _isYieldBearing: bool
):
    """
    @notice เพิ่ม asset ที่ต้อง track
    """
    assert msg.sender == self.governance or msg.sender == self.owner, "Not authorized"
    assert len(self.trackedAssets) < MAX_ASSETS, "Too many assets"
    assert _targetPercent <= BASIS_POINTS, "Invalid target"
    assert _minPercent <= _targetPercent, "Min > target"
    assert _maxPercent >= _targetPercent, "Max < target"
    
    self.trackedAssets.append(_token)
    self.assetAllocations[_token] = AssetAllocation({
        token: _token,
        targetPercent: _targetPercent,
        minPercent: _minPercent,
        maxPercent: _maxPercent,
        currentValue: 0,
        isStablecoin: _isStablecoin,
        isYieldBearing: _isYieldBearing
    })
    
    log AllocationUpdated(_token, _targetPercent, block.timestamp)

@external
def updateValuation():
    """
    @notice อัปเดตมูลค่า treasury โดยใช้ oracle prices
    """
    oracle: IPriceOracle = IPriceOracle(self.priceOracle)
    
    totalUSD: uint256 = 0
    
    for token: address in self.trackedAssets:
        balance: uint256 = ERC20(token).balanceOf(self)
        price: uint256 = oracle.getPrice(token)  # 8 decimals USD
        
        # Calculate USD value
        # balance has 18 decimals, price has 8 decimals
        # valueUSD has 8 decimals
        valueUSD: uint256 = (balance * price) / (10**18)
        
        self.assetAllocations[token].currentValue = valueUSD
        totalUSD += valueUSD
    
    self.totalTreasuryUSD = totalUSD
    self.lastValuationTime = block.timestamp

@external
def checkRebalanceNeeded() -> bool:
    """
    @notice ตรวจสอบว่าต้อง rebalance หรือไม่
    @return True ถ้าต้อง rebalance
    """
    if self.totalTreasuryUSD == 0:
        return False
    
    for token: address in self.trackedAssets:
        allocation: AssetAllocation = self.assetAllocations[token]
        
        if allocation.currentValue == 0:
            continue
        
        # Calculate current percentage
        currentPercent: uint256 = (allocation.currentValue * BASIS_POINTS) / self.totalTreasuryUSD
        
        # Check deviation from target
        deviation: uint256 = 0
        if currentPercent > allocation.targetPercent:
            deviation = currentPercent - allocation.targetPercent
        else:
            deviation = allocation.targetPercent - currentPercent
        
        if deviation > self.REBALANCE_THRESHOLD:
            return True
    
    return False

# ============================================================
# Contributor Payment System
# ============================================================

@external
def addContributor(
    _address: address,
    _name: String[50],
    _role: String[50],
    _monthlyUSD: uint256,
    _paymentToken: address
):
    """
    @notice เพิ่ม contributor สำหรับรับ payment
    @param _monthlyUSD Monthly payment in USD (8 decimals)
    """
    assert msg.sender == self.governance or msg.sender == self.treasurer, "Not authorized"
    assert len(self.contributors) < MAX_CONTRIBUTORS, "Too many contributors"
    assert not self.contributorInfo[_address].active, "Already active"
    
    self.contributors.append(_address)
    self.contributorInfo[_address] = ContributorInfo({
        name: _name,
        address_: _address,
        role: _role,
        monthlyUSD: _monthlyUSD,
        token: _paymentToken,
        startDate: block.timestamp,
        active: True,
        totalPaid: 0
    })
    
    self.lastPaymentTime[_address] = block.timestamp

@external
def processContributorPayment(_contributor: address):
    """
    @notice จ่ายเงินให้ contributor (monthly)
    """
    assert msg.sender == self.treasurer or msg.sender == self.governance, "Not authorized"
    
    info: ContributorInfo = self.contributorInfo[_contributor]
    assert info.active, "Not active contributor"
    
    # ต้องผ่าน 30 วัน
    assert block.timestamp >= self.lastPaymentTime[_contributor] + 30 * 24 * 3600, "Too early"
    
    oracle: IPriceOracle = IPriceOracle(self.priceOracle)
    
    # คำนวณจำนวน tokens ที่ต้องจ่าย
    tokenPrice: uint256 = oracle.getPrice(info.token)
    assert tokenPrice > 0, "Invalid price"
    
    # tokenAmount = monthlyUSD / tokenPrice * 10^18
    # Both monthlyUSD and tokenPrice use 8 decimals
    tokenAmount: uint256 = (info.monthlyUSD * 10**18) / tokenPrice
    
    # ตรวจสอบ spending limit
    self._checkAndUpdateSpendingLimit("operations", info.monthlyUSD)
    
    # จ่ายเงิน
    assert ERC20(info.token).transfer(_contributor, tokenAmount), "Payment failed"
    
    self.lastPaymentTime[_contributor] = block.timestamp
    self.contributorInfo[_contributor].totalPaid += tokenAmount
    
    log ContributorPaid(
        _contributor,
        info.token,
        tokenAmount,
        "monthly",
        block.timestamp
    )

@external
def processBulkPayments():
    """
    @notice จ่ายเงินให้ contributors ทุกคนที่ครบกำหนด
    """
    assert msg.sender == self.treasurer or msg.sender == self.governance, "Not authorized"
    
    for contributor: address in self.contributors:
        info: ContributorInfo = self.contributorInfo[contributor]
        
        if not info.active:
            continue
        
        if block.timestamp < self.lastPaymentTime[contributor] + 30 * 24 * 3600:
            continue
        
        # Try to process payment (don't revert on failure)
        oracle: IPriceOracle = IPriceOracle(self.priceOracle)
        tokenPrice: uint256 = oracle.getPrice(info.token)
        
        if tokenPrice == 0:
            continue
        
        tokenAmount: uint256 = (info.monthlyUSD * 10**18) / tokenPrice
        
        # Check balance
        balance: uint256 = ERC20(info.token).balanceOf(self)
        if balance < tokenAmount:
            continue
        
        # Update spending limit
        self.spendingLimits["operations"].dailySpent += info.monthlyUSD
        self.spendingLimits["operations"].monthlySpent += info.monthlyUSD
        
        ERC20(info.token).transfer(contributor, tokenAmount)
        
        self.lastPaymentTime[contributor] = block.timestamp
        self.contributorInfo[contributor].totalPaid += tokenAmount
        
        log ContributorPaid(contributor, info.token, tokenAmount, "monthly", block.timestamp)

@external
def removeContributor(_address: address):
    """ลบ contributor"""
    assert msg.sender == self.governance or msg.sender == self.treasurer, "Not authorized"
    
    self.contributorInfo[_address].active = False

# ============================================================
# Grant Program
# ============================================================

@external
def createGrant(
    _title: String[100],
    _recipient: address,
    _totalAmount: uint256,
    _token: address,
    _milestonesCount: uint256,
    _description: String[300]
) -> uint256:
    """
    @notice สร้าง grant ใหม่
    @return grantId
    """
    assert msg.sender == self.governance, "Only governance"
    assert _recipient != empty(address), "Invalid recipient"
    assert _milestonesCount > 0 and _milestonesCount <= 10, "Invalid milestones"
    
    grantId: uint256 = self.grantCount
    self.grantCount += 1
    
    self.grants[grantId] = Grant({
        id: grantId,
        title: _title,
        recipient: _recipient,
        totalAmount: _totalAmount,
        token: _token,
        milestonesCount: _milestonesCount,
        milestonesCompleted: 0,
        disbursedAmount: 0,
        approvedAt: block.timestamp,
        completedAt: 0,
        active: True,
        description: _description
    })
    
    log GrantApproved(grantId, _recipient, _totalAmount, _token, block.timestamp)
    
    return grantId

@external
def setGrantMilestone(
    _grantId: uint256,
    _milestoneIndex: uint256,
    _amount: uint256,
    _description: String[200]
):
    """ตั้งค่า milestone"""
    assert msg.sender == self.governance, "Only governance"
    assert _grantId < self.grantCount, "Invalid grant"
    
    grant: Grant = self.grants[_grantId]
    assert _milestoneIndex < grant.milestonesCount, "Invalid milestone"
    
    self.grantMilestones[_grantId][_milestoneIndex] = GrantMilestone({
        grantId: _grantId,
        milestoneIndex: _milestoneIndex,
        amount: _amount,
        description: _description,
        completedAt: 0,
        paid: False
    })

@external
def disburseGrantMilestone(_grantId: uint256, _milestoneIndex: uint256):
    """
    @notice จ่ายเงิน grant milestone
    """
    assert msg.sender == self.governance or msg.sender == self.treasurer, "Not authorized"
    
    grant: Grant = self.grants[_grantId]
    assert grant.active, "Grant not active"
    assert _milestoneIndex < grant.milestonesCount, "Invalid milestone"
    
    milestone: GrantMilestone = self.grantMilestones[_grantId][_milestoneIndex]
    assert not milestone.paid, "Already paid"
    assert milestone.amount > 0, "Zero amount"
    
    # Update milestone
    self.grantMilestones[_grantId][_milestoneIndex].paid = True
    self.grantMilestones[_grantId][_milestoneIndex].completedAt = block.timestamp
    
    # Update grant
    self.grants[_grantId].disbursedAmount += milestone.amount
    self.grants[_grantId].milestonesCompleted += 1
    
    # Check if grant is complete
    if self.grants[_grantId].milestonesCompleted >= grant.milestonesCount:
        self.grants[_grantId].completedAt = block.timestamp
        self.grants[_grantId].active = False
    
    # Disburse payment
    assert ERC20(grant.token).transfer(grant.recipient, milestone.amount), "Transfer failed"
    
    log GrantDisbursed(_grantId, _milestoneIndex, milestone.amount, block.timestamp)

@external
def cancelGrant(_grantId: uint256, _reason: String[100]):
    """ยกเลิก grant"""
    assert msg.sender == self.governance, "Only governance"
    
    self.grants[_grantId].active = False

# ============================================================
# Yield Strategy Management
# ============================================================

@external
def addYieldStrategy(
    _strategy: address,
    _name: String[50],
    _tokens: DynArray[address, 5]
):
    """เพิ่ม yield strategy"""
    assert msg.sender == self.governance, "Only governance"
    assert len(self.yieldStrategies) < MAX_STRATEGIES, "Too many strategies"
    
    self.yieldStrategies.append(_strategy)
    self.strategyInfo[_strategy] = YieldStrategy({
        strategy: _strategy,
        tokens: _tokens,
        allocatedUSD: 0,
        earnedYield: 0,
        lastHarvest: block.timestamp,
        active: True,
        name: _name
    })

@external
def depositToStrategy(_strategy: address, _token: address, _amount: uint256):
    """ฝากเงินเข้า yield strategy"""
    assert msg.sender == self.treasurer or msg.sender == self.governance, "Not authorized"
    assert self.strategyInfo[_strategy].active, "Strategy not active"
    
    # Approve and deposit
    ERC20(_token).approve(_strategy, _amount)
    IYieldStrategy(_strategy).deposit(_token, _amount)
    
    oracle: IPriceOracle = IPriceOracle(self.priceOracle)
    price: uint256 = oracle.getPrice(_token)
    valueUSD: uint256 = (_amount * price) / (10**18)
    
    self.strategyInfo[_strategy].allocatedUSD += valueUSD

@external
def harvestYield(_strategy: address):
    """เก็บ yield จาก strategy"""
    assert self.strategyInfo[_strategy].active, "Strategy not active"
    
    info: YieldStrategy = self.strategyInfo[_strategy]
    
    for token: address in info.tokens:
        strategyContract: IYieldStrategy = IYieldStrategy(_strategy)
        yieldAmount: uint256 = strategyContract.getYield(token)
        
        if yieldAmount > 0:
            strategyContract.withdraw(token, yieldAmount)
            self.strategyInfo[_strategy].earnedYield += yieldAmount
            self.totalYieldEarned += yieldAmount
            
            log YieldHarvested(_strategy, token, yieldAmount, block.timestamp)
    
    self.strategyInfo[_strategy].lastHarvest = block.timestamp

# ============================================================
# Spending Limit Enforcement
# ============================================================

@internal
def _checkAndUpdateSpendingLimit(_category: String[50], _amountUSD: uint256):
    """ตรวจสอบและอัปเดต spending limit"""
    limit: SpendingLimit = self.spendingLimits[_category]
    
    # Reset daily limit
    if block.timestamp >= limit.lastDayReset + 24 * 3600:
        self.spendingLimits[_category].dailySpent = 0
        self.spendingLimits[_category].lastDayReset = block.timestamp
    
    # Reset monthly limit
    if block.timestamp >= limit.lastMonthReset + 30 * 24 * 3600:
        self.spendingLimits[_category].monthlySpent = 0
        self.spendingLimits[_category].lastMonthReset = block.timestamp
    
    # Check limits
    assert limit.dailySpent + _amountUSD <= limit.dailyLimit, "Daily limit exceeded"
    assert limit.monthlySpent + _amountUSD <= limit.monthlyLimit, "Monthly limit exceeded"
    
    # Update spent amounts
    self.spendingLimits[_category].dailySpent += _amountUSD
    self.spendingLimits[_category].monthlySpent += _amountUSD

@external
def updateSpendingLimit(
    _category: String[50],
    _dailyLimit: uint256,
    _monthlyLimit: uint256
):
    """อัปเดต spending limits"""
    assert msg.sender == self.governance, "Only governance"
    
    oldDaily: uint256 = self.spendingLimits[_category].dailyLimit
    
    self.spendingLimits[_category].dailyLimit = _dailyLimit
    self.spendingLimits[_category].monthlyLimit = _monthlyLimit
    
    log SpendingLimitUpdated(_category, oldDaily, _dailyLimit)

# ============================================================
# Emergency Functions
# ============================================================

@external
def enableEmergencyWithdrawal(_recipient: address):
    """เปิดใช้งาน emergency withdrawal"""
    assert msg.sender == self.governance, "Only governance"
    
    self.emergencyWithdrawalEnabled = True
    self.emergencyRecipient = _recipient

@external
def emergencyWithdraw(_token: address, _amount: uint256):
    """Emergency withdraw"""
    assert self.emergencyWithdrawalEnabled, "Not enabled"
    assert msg.sender == self.governance or msg.sender == self.owner, "Not authorized"
    
    recipient: address = self.emergencyRecipient
    assert recipient != empty(address), "No recipient"
    
    balance: uint256 = ERC20(_token).balanceOf(self)
    withdrawAmount: uint256 = min(_amount, balance)
    
    assert ERC20(_token).transfer(recipient, withdrawAmount), "Transfer failed"

# ============================================================
# View Functions
# ============================================================

@external
@view
def getTreasuryOverview() -> (uint256, uint256, uint256, uint256):
    """
    @notice ดูภาพรวม treasury
    @return (totalUSD, stablecoinUSD, volatileUSD, yieldEarned)
    """
    stablecoinUSD: uint256 = 0
    volatileUSD: uint256 = 0
    
    for token: address in self.trackedAssets:
        allocation: AssetAllocation = self.assetAllocations[token]
        if allocation.isStablecoin:
            stablecoinUSD += allocation.currentValue
        else:
            volatileUSD += allocation.currentValue
    
    return (self.totalTreasuryUSD, stablecoinUSD, volatileUSD, self.totalYieldEarned)

@external
@view
def getContributorPayrollCost() -> uint256:
    """
    @notice คำนวณค่าใช้จ่าย payroll รายเดือน (USD)
    """
    totalMonthly: uint256 = 0
    
    for contributor: address in self.contributors:
        info: ContributorInfo = self.contributorInfo[contributor]
        if info.active:
            totalMonthly += info.monthlyUSD
    
    return totalMonthly

@external
@view
def getActiveGrants() -> DynArray[uint256, MAX_GRANTS]:
    """รายการ grants ที่ยัง active"""
    active: DynArray[uint256, MAX_GRANTS] = []
    
    for i: uint256 in range(MAX_GRANTS):
        if i >= self.grantCount:
            break
        if self.grants[i].active:
            active.append(i)
    
    return active

@external
@view
def getGrant(_grantId: uint256) -> Grant:
    """ดูรายละเอียด grant"""
    return self.grants[_grantId]

@external
@view
def getRunwayMonths() -> uint256:
    """
    @notice คำนวณ runway ของ treasury (กี่เดือน)
    @dev ใช้ stablecoin balance หารด้วย monthly burn rate
    """
    monthlyBurn: uint256 = self.getContributorPayrollCost()
    
    if monthlyBurn == 0:
        return 999  # Infinite runway
    
    # Calculate stablecoin balance
    stablecoinBalance: uint256 = 0
    for token: address in self.trackedAssets:
        allocation: AssetAllocation = self.assetAllocations[token]
        if allocation.isStablecoin:
            stablecoinBalance += allocation.currentValue
    
    return stablecoinBalance / monthlyBurn

@external
@view
def getAllAssetAllocations() -> DynArray[AssetAllocation, MAX_ASSETS]:
    """ดู asset allocations ทั้งหมด"""
    result: DynArray[AssetAllocation, MAX_ASSETS] = []
    
    for token: address in self.trackedAssets:
        result.append(self.assetAllocations[token])
    
    return result

@external
@view
def getContributors() -> DynArray[ContributorInfo, MAX_CONTRIBUTORS]:
    """รายชื่อ contributors ทั้งหมด"""
    result: DynArray[ContributorInfo, MAX_CONTRIBUTORS] = []
    
    for contributor: address in self.contributors:
        result.append(self.contributorInfo[contributor])
    
    return result

@external
@view
def getGrantMilestone(_grantId: uint256, _milestoneIndex: uint256) -> GrantMilestone:
    """ดู grant milestone"""
    return self.grantMilestones[_grantId][_milestoneIndex]
```

---

## 4. Yield Strategies {#yield}

### Treasury Yield Strategy Examples

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0

"""
@title StablecoinYieldStrategy - Yield strategy สำหรับ stablecoins
@notice ฝาก stablecoins เข้า lending protocols เพื่อได้ yield
@dev รองรับ Aave, Compound, etc.
"""

from vyper.interfaces import ERC20

interface ILendingPool:
    def deposit(
        _asset: address,
        _amount: uint256,
        _onBehalfOf: address,
        _referralCode: uint16
    ): nonpayable
    
    def withdraw(
        _asset: address,
        _amount: uint256,
        _to: address
    ) -> uint256: nonpayable

interface IAToken:
    def balanceOf(_owner: address) -> uint256: view

# Events
event StrategyDeposit:
    token: indexed(address)
    amount: uint256
    timestamp: uint256

event StrategyWithdrawal:
    token: indexed(address)
    amount: uint256
    timestamp: uint256

event YieldCollected:
    token: indexed(address)
    principal: uint256
    current: uint256
    yield_: uint256
    timestamp: uint256

# State
owner: public(address)
treasury: public(address)
lendingPool: public(address)

# Token mappings
aTokens: public(HashMap[address, address])  # underlying -> aToken
principals: public(HashMap[address, uint256])  # deposited amounts

@deploy
def __init__(_lendingPool: address, _treasury: address):
    """
    @notice Initialize yield strategy
    @param _lendingPool Aave lending pool address
    @param _treasury Treasury address
    """
    assert _lendingPool != empty(address), "Invalid pool"
    assert _treasury != empty(address), "Invalid treasury"
    
    self.owner = msg.sender
    self.lendingPool = _lendingPool
    self.treasury = _treasury

@external
def deposit(_token: address, _amount: uint256):
    """ฝาก tokens เข้า lending pool"""
    assert msg.sender == self.treasury or msg.sender == self.owner, "Not authorized"
    
    # Transfer tokens from caller
    assert ERC20(_token).transferFrom(msg.sender, self, _amount), "Transfer failed"
    
    # Approve and deposit to Aave
    ERC20(_token).approve(self.lendingPool, _amount)
    ILendingPool(self.lendingPool).deposit(_token, _amount, self, 0)
    
    # Track principal
    self.principals[_token] += _amount
    
    log StrategyDeposit(_token, _amount, block.timestamp)

@external
def withdraw(_token: address, _amount: uint256):
    """ถอน tokens จาก lending pool"""
    assert msg.sender == self.treasury or msg.sender == self.owner, "Not authorized"
    
    withdrawn: uint256 = ILendingPool(self.lendingPool).withdraw(_token, _amount, msg.sender)
    
    # Update principal (can't go below 0)
    if withdrawn <= self.principals[_token]:
        self.principals[_token] -= withdrawn
    else:
        self.principals[_token] = 0
    
    log StrategyWithdrawal(_token, withdrawn, block.timestamp)

@external
@view
def getBalance(_token: address) -> uint256:
    """ดู balance ปัจจุบัน (principal + yield)"""
    aToken: address = self.aTokens[_token]
    if aToken == empty(address):
        return 0
    return IAToken(aToken).balanceOf(self)

@external
@view
def getYield(_token: address) -> uint256:
    """คำนวณ yield ที่ได้รับ"""
    currentBalance: uint256 = self.getBalance(_token)
    principal: uint256 = self.principals[_token]
    
    if currentBalance <= principal:
        return 0
    
    return currentBalance - principal

@external
def setAToken(_token: address, _aToken: address):
    """ตั้งค่า aToken mapping"""
    assert msg.sender == self.owner, "Not owner"
    self.aTokens[_token] = _aToken
```

---

## 5. Treasury Reporting {#reporting}

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0

"""
@title TreasuryReport - ระบบรายงาน treasury
@notice สร้างรายงาน treasury แบบ on-chain สำหรับ transparency
"""

# Events
event MonthlyReport:
    period: indexed(uint256)  # YYYYMM
    totalUSD: uint256
    inflows: uint256
    outflows: uint256
    yieldEarned: uint256
    timestamp: uint256

event QuarterlyReport:
    quarter: indexed(uint256)  # Q1=1, Q2=2, etc.
    year: uint256
    startBalance: uint256
    endBalance: uint256
    netChange: int256
    timestamp: uint256

# Structs
struct MonthlyData:
    period: uint256
    startBalance: uint256
    endBalance: uint256
    totalInflows: uint256
    totalOutflows: uint256
    payrollExpense: uint256
    grantExpense: uint256
    yieldEarned: uint256
    operationalExpense: uint256
    timestamp: uint256

# State
owner: public(address)
treasury: public(address)

monthlyReports: public(HashMap[uint256, MonthlyData])
reportPeriods: public(DynArray[uint256, 120])  # 10 years of monthly reports

@deploy
def __init__(_treasury: address):
    self.owner = msg.sender
    self.treasury = _treasury

@external
def submitMonthlyReport(
    _period: uint256,
    _startBalance: uint256,
    _endBalance: uint256,
    _inflows: uint256,
    _outflows: uint256,
    _payroll: uint256,
    _grants: uint256,
    _yield: uint256,
    _ops: uint256
):
    """
    @notice ส่ง monthly report
    @param _period YYYYMM format (e.g., 202401 = January 2024)
    """
    assert msg.sender == self.treasury or msg.sender == self.owner, "Not authorized"
    
    self.monthlyReports[_period] = MonthlyData({
        period: _period,
        startBalance: _startBalance,
        endBalance: _endBalance,
        totalInflows: _inflows,
        totalOutflows: _outflows,
        payrollExpense: _payroll,
        grantExpense: _grants,
        yieldEarned: _yield,
        operationalExpense: _ops,
        timestamp: block.timestamp
    })
    
    self.reportPeriods.append(_period)
    
    log MonthlyReport(_period, _endBalance, _inflows, _outflows, _yield, block.timestamp)

@external
@view
def getReport(_period: uint256) -> MonthlyData:
    """ดู monthly report"""
    return self.monthlyReports[_period]

@external
@view
def getReportPeriods() -> DynArray[uint256, 120]:
    """รายการ periods ที่มี report"""
    return self.reportPeriods
```

---

## สรุป: Treasury Management Best Practices

### Framework การตัดสินใจ

```
Treasury Decision Framework

1. Runway Check (ก่อนใช้จ่ายใดๆ)
   - ต้องมี stablecoin runway > 18 เดือน
   - ถ้า < 18 เดือน: หยุดใช้จ่ายที่ไม่จำเป็น
   - ถ้า < 12 เดือน: Emergency fundraise

2. Spending Approval Tiers
   < $10k/month: Treasurer
   $10k - $100k: Treasurer + 2 sig
   > $100k: Governance vote

3. Yield Strategy Risk Levels
   Conservative: USDC/DAI lending (Aave/Compound)
   Moderate: ETH staking
   Aggressive: LP positions (requires vote)

4. Rebalancing Triggers
   > 5% drift from target: Review
   > 10% drift: Rebalance
   > 20% drift: Emergency rebalance
```

### KPIs ที่ควร Track

| Metric | Description | Target |
|--------|-------------|--------|
| Runway | Months of runway | > 24 months |
| Diversification | % in non-native tokens | > 50% |
| Yield APY | Return on treasury assets | > 3% |
| Monthly Burn | Total monthly expenses | Trending down |
| Grant ROI | Value created / grant given | > 5x |

> **สำคัญ**: Treasury management ที่ดีคือ protocol สามารถ survive bear market ได้โดยไม่ต้องขาย native tokens ในราคาต่ำ
