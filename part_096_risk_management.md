# Part 096: Protocol Risk Management

## สารบัญ
1. [Overview of DeFi Risks](#overview)
2. [Smart Contract Risk](#smart-contract-risk)
3. [Liquidity Risk](#liquidity-risk)
4. [Oracle Risk](#oracle-risk)
5. [Governance Risk](#governance-risk)
6. [Operational Risk](#operational-risk)
7. [Risk Scoring Framework](#risk-scoring)
8. [Risk Mitigation Contracts](#mitigation)

---

## 1. Overview of DeFi Risks {#overview}

DeFi protocols เผชิญกับความเสี่ยงหลากหลายประเภทที่แตกต่างจาก Traditional Finance:

```
Risk Categories in DeFi

Technical Risks
├── Smart Contract Bugs
├── Oracle Manipulation
├── Economic Design Flaws
└── Integration Risks (composability)

Market Risks  
├── Liquidity Risk
├── Price Volatility
├── Correlation Risk
└── Black Swan Events

Operational Risks
├── Key Management
├── Infrastructure Failures
├── Upgrade Risks
└── Regulatory Risks

Governance Risks
├── Governance Attacks
├── Voter Apathy
└── Malicious Proposals
```

### Risk Framework Overview

| Risk Type | Likelihood | Impact | Priority |
|-----------|-----------|--------|----------|
| Smart Contract Bug | Medium | Critical | P0 |
| Oracle Manipulation | Medium | High | P1 |
| Liquidity Crisis | Low | High | P1 |
| Governance Attack | Low | Critical | P1 |
| Regulatory Action | Medium | Medium | P2 |
| Key Compromise | Low | Critical | P1 |

---

## 2. Smart Contract Risk {#smart-contract-risk}

### RiskRegistry Contract

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0

"""
@title RiskRegistry - ระบบ track และ manage protocol risks
@notice บันทึก risk assessments, mitigations และ status ของทุก risk
@dev ใช้สำหรับ on-chain risk tracking และ transparency
"""

from vyper.interfaces import ERC20

# ============================================================
# Events
# ============================================================

event RiskRegistered:
    riskId: indexed(uint256)
    category: String[30]
    severity: uint8
    timestamp: uint256

event RiskMitigated:
    riskId: indexed(uint256)
    mitigationId: uint256
    timestamp: uint256

event RiskStatusChanged:
    riskId: indexed(uint256)
    oldStatus: String[20]
    newStatus: String[20]

event IncidentLogged:
    incidentId: indexed(uint256)
    riskId: indexed(uint256)
    severity: uint8
    timestamp: uint256

event RiskScoreUpdated:
    riskId: indexed(uint256)
    oldScore: uint256
    newScore: uint256

# ============================================================
# Structs
# ============================================================

struct Risk:
    id: uint256
    category: String[30]         # "smart_contract", "liquidity", "oracle", etc.
    title: String[100]
    description: String[300]
    severity: uint8              # 1=Low, 2=Medium, 3=High, 4=Critical
    likelihood: uint8            # 1=Rare, 2=Unlikely, 3=Possible, 4=Likely, 5=Almost Certain
    riskScore: uint256           # severity * likelihood (max 20)
    status: String[20]           # "open", "mitigated", "accepted", "closed"
    registeredAt: uint256
    lastUpdated: uint256
    mitigationsCount: uint256
    triggeredIncidents: uint256

struct Mitigation:
    id: uint256
    riskId: uint256
    description: String[300]
    status: String[20]           # "planned", "in_progress", "completed"
    priority: uint8
    implementedAt: uint256
    implementedBy: address
    effectivenessScore: uint8    # 1-5 scale post-implementation

struct Incident:
    id: uint256
    riskId: uint256
    description: String[300]
    severity: uint8
    fundsAtRisk: uint256         # USD value
    fundsLost: uint256           # USD value  
    occurredAt: uint256
    resolvedAt: uint256
    resolved: bool
    rootCause: String[300]
    lessonsLearned: String[300]

struct RiskCategory:
    name: String[30]
    totalRisks: uint256
    openRisks: uint256
    averageScore: uint256

# ============================================================
# Constants
# ============================================================

MAX_RISKS: constant(uint256) = 200
MAX_MITIGATIONS: constant(uint256) = 500
MAX_INCIDENTS: constant(uint256) = 100
MAX_CATEGORIES: constant(uint256) = 10

# Risk categories
CAT_SMART_CONTRACT: constant(String[30]) = "smart_contract"
CAT_LIQUIDITY: constant(String[30]) = "liquidity"
CAT_ORACLE: constant(String[30]) = "oracle"
CAT_GOVERNANCE: constant(String[30]) = "governance"
CAT_OPERATIONAL: constant(String[30]) = "operational"
CAT_MARKET: constant(String[30]) = "market"
CAT_REGULATORY: constant(String[30]) = "regulatory"
CAT_COUNTERPARTY: constant(String[30]) = "counterparty"

# ============================================================
# State Variables
# ============================================================

owner: public(address)
riskCommittee: public(DynArray[address, 10])
isCommitteeMember: public(HashMap[address, bool])

# Risk storage
risks: public(HashMap[uint256, Risk])
riskCount: public(uint256)
risksByCategory: public(HashMap[String[30], DynArray[uint256, MAX_RISKS]])

# Mitigation storage
mitigations: public(HashMap[uint256, Mitigation])
mitigationCount: public(uint256)
riskMitigations: public(HashMap[uint256, DynArray[uint256, 20]])

# Incident storage
incidents: public(HashMap[uint256, Incident])
incidentCount: public(uint256)
riskIncidents: public(HashMap[uint256, DynArray[uint256, 20]])

# Statistics
totalFundsAtRisk: public(uint256)
totalFundsLost: public(uint256)
openRisksCount: public(uint256)
criticalRisksCount: public(uint256)

# ============================================================
# Constructor
# ============================================================

@deploy
def __init__(_committee: DynArray[address, 10]):
    """
    @notice Initialize risk registry
    @param _committee Initial risk committee members
    """
    assert len(_committee) >= 3, "Need at least 3 committee members"
    
    self.owner = msg.sender
    
    for member: address in _committee:
        self.riskCommittee.append(member)
        self.isCommitteeMember[member] = True

# ============================================================
# Risk Management Functions
# ============================================================

@external
def registerRisk(
    _category: String[30],
    _title: String[100],
    _description: String[300],
    _severity: uint8,
    _likelihood: uint8
) -> uint256:
    """
    @notice ลงทะเบียน risk ใหม่
    @param _severity 1=Low, 2=Medium, 3=High, 4=Critical
    @param _likelihood 1=Rare to 5=Almost Certain
    @return riskId
    """
    assert self.isCommitteeMember[msg.sender] or msg.sender == self.owner, "Not authorized"
    assert _severity >= 1 and _severity <= 4, "Invalid severity"
    assert _likelihood >= 1 and _likelihood <= 5, "Invalid likelihood"
    
    riskId: uint256 = self.riskCount
    self.riskCount += 1
    
    riskScore: uint256 = convert(_severity, uint256) * convert(_likelihood, uint256)
    
    self.risks[riskId] = Risk({
        id: riskId,
        category: _category,
        title: _title,
        description: _description,
        severity: _severity,
        likelihood: _likelihood,
        riskScore: riskScore,
        status: "open",
        registeredAt: block.timestamp,
        lastUpdated: block.timestamp,
        mitigationsCount: 0,
        triggeredIncidents: 0
    })
    
    # Track by category
    if len(self.risksByCategory[_category]) < MAX_RISKS:
        self.risksByCategory[_category].append(riskId)
    
    # Update statistics
    self.openRisksCount += 1
    if _severity == 4:  # Critical
        self.criticalRisksCount += 1
    
    log RiskRegistered(riskId, _category, _severity, block.timestamp)
    
    return riskId

@external
def addMitigation(
    _riskId: uint256,
    _description: String[300],
    _priority: uint8
) -> uint256:
    """
    @notice เพิ่ม mitigation สำหรับ risk
    """
    assert self.isCommitteeMember[msg.sender] or msg.sender == self.owner, "Not authorized"
    assert _riskId < self.riskCount, "Invalid risk"
    
    mitigationId: uint256 = self.mitigationCount
    self.mitigationCount += 1
    
    self.mitigations[mitigationId] = Mitigation({
        id: mitigationId,
        riskId: _riskId,
        description: _description,
        status: "planned",
        priority: _priority,
        implementedAt: 0,
        implementedBy: empty(address),
        effectivenessScore: 0
    })
    
    if len(self.riskMitigations[_riskId]) < 20:
        self.riskMitigations[_riskId].append(mitigationId)
    
    self.risks[_riskId].mitigationsCount += 1
    self.risks[_riskId].lastUpdated = block.timestamp
    
    log RiskMitigated(_riskId, mitigationId, block.timestamp)
    
    return mitigationId

@external
def implementMitigation(_mitigationId: uint256, _effectivenessScore: uint8):
    """
    @notice Mark mitigation เป็น implemented
    @param _effectivenessScore 1-5 score ของ effectiveness
    """
    assert self.isCommitteeMember[msg.sender] or msg.sender == self.owner, "Not authorized"
    assert _mitigationId < self.mitigationCount, "Invalid mitigation"
    assert _effectivenessScore >= 1 and _effectivenessScore <= 5, "Invalid score"
    
    self.mitigations[_mitigationId].status = "completed"
    self.mitigations[_mitigationId].implementedAt = block.timestamp
    self.mitigations[_mitigationId].implementedBy = msg.sender
    self.mitigations[_mitigationId].effectivenessScore = _effectivenessScore
    
    # Update risk score based on mitigation effectiveness
    riskId: uint256 = self.mitigations[_mitigationId].riskId
    oldScore: uint256 = self.risks[riskId].riskScore
    
    # Reduce risk score based on effectiveness (1-5 maps to 10-50% reduction)
    reduction: uint256 = convert(_effectivenessScore, uint256) * 10  # 10-50%
    newScore: uint256 = oldScore * (100 - reduction) / 100
    
    self.risks[riskId].riskScore = newScore
    self.risks[riskId].lastUpdated = block.timestamp
    
    log RiskScoreUpdated(riskId, oldScore, newScore)

@external
def logIncident(
    _riskId: uint256,
    _description: String[300],
    _severity: uint8,
    _fundsAtRisk: uint256
) -> uint256:
    """
    @notice บันทึก incident ที่เกิดขึ้น
    """
    assert self.isCommitteeMember[msg.sender] or msg.sender == self.owner, "Not authorized"
    
    incidentId: uint256 = self.incidentCount
    self.incidentCount += 1
    
    self.incidents[incidentId] = Incident({
        id: incidentId,
        riskId: _riskId,
        description: _description,
        severity: _severity,
        fundsAtRisk: _fundsAtRisk,
        fundsLost: 0,
        occurredAt: block.timestamp,
        resolvedAt: 0,
        resolved: False,
        rootCause: "",
        lessonsLearned: ""
    })
    
    if len(self.riskIncidents[_riskId]) < 20:
        self.riskIncidents[_riskId].append(incidentId)
    
    self.risks[_riskId].triggeredIncidents += 1
    self.totalFundsAtRisk += _fundsAtRisk
    
    log IncidentLogged(incidentId, _riskId, _severity, block.timestamp)
    
    return incidentId

@external
def resolveIncident(
    _incidentId: uint256,
    _fundsLost: uint256,
    _rootCause: String[300],
    _lessonsLearned: String[300]
):
    """
    @notice แก้ไข incident และบันทึก post-mortem
    """
    assert self.isCommitteeMember[msg.sender] or msg.sender == self.owner, "Not authorized"
    assert _incidentId < self.incidentCount, "Invalid incident"
    
    self.incidents[_incidentId].resolved = True
    self.incidents[_incidentId].resolvedAt = block.timestamp
    self.incidents[_incidentId].fundsLost = _fundsLost
    self.incidents[_incidentId].rootCause = _rootCause
    self.incidents[_incidentId].lessonsLearned = _lessonsLearned
    
    self.totalFundsLost += _fundsLost

@external
def updateRiskStatus(_riskId: uint256, _newStatus: String[20]):
    """
    @notice อัปเดตสถานะ risk
    @param _newStatus "open", "mitigated", "accepted", "closed"
    """
    assert self.isCommitteeMember[msg.sender] or msg.sender == self.owner, "Not authorized"
    assert _riskId < self.riskCount, "Invalid risk"
    
    oldStatus: String[20] = self.risks[_riskId].status
    self.risks[_riskId].status = _newStatus
    self.risks[_riskId].lastUpdated = block.timestamp
    
    # Update statistics
    if oldStatus == "open" and _newStatus != "open":
        if self.openRisksCount > 0:
            self.openRisksCount -= 1
        if self.risks[_riskId].severity == 4 and self.criticalRisksCount > 0:
            self.criticalRisksCount -= 1
    elif oldStatus != "open" and _newStatus == "open":
        self.openRisksCount += 1
        if self.risks[_riskId].severity == 4:
            self.criticalRisksCount += 1
    
    log RiskStatusChanged(_riskId, oldStatus, _newStatus)

# ============================================================
# View Functions
# ============================================================

@external
@view
def getRisk(_riskId: uint256) -> Risk:
    """ดูรายละเอียด risk"""
    return self.risks[_riskId]

@external
@view
def getRiskMitigations(_riskId: uint256) -> DynArray[uint256, 20]:
    """รายการ mitigations ของ risk"""
    return self.riskMitigations[_riskId]

@external
@view
def getIncident(_incidentId: uint256) -> Incident:
    """ดูรายละเอียด incident"""
    return self.incidents[_incidentId]

@external
@view
def getRisksByCategory(_category: String[30]) -> DynArray[uint256, MAX_RISKS]:
    """รายการ risks ตาม category"""
    return self.risksByCategory[_category]

@external
@view
def getPortfolioRiskScore() -> uint256:
    """
    @notice คำนวณ risk score รวมของ protocol
    @dev ผลรวม weighted risk scores ของ open risks
    """
    totalScore: uint256 = 0
    
    for i: uint256 in range(MAX_RISKS):
        if i >= self.riskCount:
            break
        
        risk: Risk = self.risks[i]
        if risk.status == "open" or risk.status == "mitigated":
            totalScore += risk.riskScore
    
    return totalScore

@external
@view
def getCriticalRisks() -> DynArray[uint256, 50]:
    """รายการ critical risks ที่ยัง open"""
    critical: DynArray[uint256, 50] = []
    
    for i: uint256 in range(MAX_RISKS):
        if i >= self.riskCount:
            break
        
        risk: Risk = self.risks[i]
        if risk.severity == 4 and risk.status == "open":
            if len(critical) < 50:
                critical.append(i)
    
    return critical

@external
@view
def getRiskStats() -> (uint256, uint256, uint256, uint256):
    """
    @notice ดูสถิติ risks
    @return (totalRisks, openRisks, criticalRisks, portfolioScore)
    """
    return (
        self.riskCount,
        self.openRisksCount,
        self.criticalRisksCount,
        self.getPortfolioRiskScore()
    )
```

---

## 3. Oracle Risk Management {#oracle-risk}

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0

"""
@title OracleRiskManager - จัดการความเสี่ยงจาก Price Oracles
@notice ป้องกัน oracle manipulation ด้วย circuit breakers และ multi-oracle
"""

# ============================================================
# Interfaces
# ============================================================

interface IChainlinkOracle:
    def latestRoundData() -> (uint80, int256, uint256, uint256, uint80): view
    def decimals() -> uint8: view

interface IUniswapV3Pool:
    def observe(_secondsAgos: DynArray[uint32, 2]) -> (int56[2], uint160[2]): view
    def slot0() -> (uint160, int24, uint16, uint16, uint16, uint8, bool): view

# ============================================================
# Events
# ============================================================

event OracleAdded:
    oracle: indexed(address)
    oracleType: String[20]
    token: indexed(address)

event PriceDeviationAlert:
    token: indexed(address)
    chainlinkPrice: uint256
    twapPrice: uint256
    deviationBps: uint256
    timestamp: uint256

event OracleCircuitBreakerTripped:
    token: indexed(address)
    reason: String[100]
    timestamp: uint256

event StaleOracleDetected:
    oracle: indexed(address)
    lastUpdate: uint256
    maxAge: uint256

# ============================================================
# Structs
# ============================================================

struct OracleConfig:
    chainlinkFeed: address
    uniswapV3Pool: address
    twapPeriod: uint32           # seconds for TWAP
    maxDeviationBps: uint256     # max allowed deviation between oracles
    maxStaleness: uint256        # max age of price data (seconds)
    circuitBreakerTripped: bool
    lastValidPrice: uint256
    lastPriceUpdate: uint256

struct PriceData:
    price: uint256
    timestamp: uint256
    source: String[20]
    isValid: bool

# ============================================================
# Constants
# ============================================================

MAX_STALENESS: constant(uint256) = 3600  # 1 hour max staleness
DEFAULT_DEVIATION_BPS: constant(uint256) = 500  # 5% default max deviation
TWAP_PERIOD: constant(uint32) = 1800  # 30 minute TWAP

# ============================================================
# State Variables
# ============================================================

owner: public(address)
guardian: public(address)

# Oracle configurations per token
oracleConfigs: public(HashMap[address, OracleConfig])
trackedTokens: public(DynArray[address, 30])

# Price circuit breakers
globalCircuitBreaker: public(bool)
circuitBreakerReason: public(String[100])

# Price sanity bounds (to prevent extreme values)
priceSanityMin: public(HashMap[address, uint256])   # minimum acceptable price
priceSanityMax: public(HashMap[address, uint256])   # maximum acceptable price

# ============================================================
# Constructor
# ============================================================

@deploy
def __init__(_guardian: address):
    self.owner = msg.sender
    self.guardian = _guardian

# ============================================================
# Oracle Functions
# ============================================================

@external
def addOracleConfig(
    _token: address,
    _chainlink: address,
    _uniswapPool: address,
    _maxDeviationBps: uint256,
    _maxStaleness: uint256
):
    """
    @notice เพิ่ม oracle configuration สำหรับ token
    """
    assert msg.sender == self.owner, "Not owner"
    
    self.oracleConfigs[_token] = OracleConfig({
        chainlinkFeed: _chainlink,
        uniswapV3Pool: _uniswapPool,
        twapPeriod: TWAP_PERIOD,
        maxDeviationBps: _maxDeviationBps,
        maxStaleness: _maxStaleness,
        circuitBreakerTripped: False,
        lastValidPrice: 0,
        lastPriceUpdate: 0
    })
    
    self.trackedTokens.append(_token)
    
    log OracleAdded(_chainlink, "chainlink", _token)

@external
@view
def getPrice(_token: address) -> uint256:
    """
    @notice ดู validated price ของ token
    @return price in USD (8 decimals)
    """
    assert not self.globalCircuitBreaker, "Global circuit breaker active"
    
    config: OracleConfig = self.oracleConfigs[_token]
    assert not config.circuitBreakerTripped, "Token circuit breaker active"
    
    # Get Chainlink price
    chainlinkPrice: uint256 = self._getChainlinkPrice(config.chainlinkFeed)
    
    # Validate price
    assert chainlinkPrice > 0, "Invalid Chainlink price"
    
    # Check sanity bounds
    if self.priceSanityMin[_token] > 0:
        assert chainlinkPrice >= self.priceSanityMin[_token], "Below sanity min"
    if self.priceSanityMax[_token] > 0:
        assert chainlinkPrice <= self.priceSanityMax[_token], "Above sanity max"
    
    return chainlinkPrice

@internal
@view
def _getChainlinkPrice(_feed: address) -> uint256:
    """ดูราคาจาก Chainlink"""
    if _feed == empty(address):
        return 0
    
    oracle: IChainlinkOracle = IChainlinkOracle(_feed)
    
    roundId: uint80 = 0
    answer: int256 = 0
    startedAt: uint256 = 0
    updatedAt: uint256 = 0
    answeredInRound: uint80 = 0
    
    (roundId, answer, startedAt, updatedAt, answeredInRound) = oracle.latestRoundData()
    
    # Validate data
    if answer <= 0:
        return 0
    
    if block.timestamp - updatedAt > MAX_STALENESS:
        return 0
    
    decimals: uint8 = oracle.decimals()
    
    # Normalize to 8 decimals
    price: uint256 = convert(answer, uint256)
    
    if decimals < 8:
        price = price * (10 ** convert(8 - decimals, uint256))
    elif decimals > 8:
        price = price / (10 ** convert(decimals - 8, uint256))
    
    return price

@external
def validateAndUpdatePrice(_token: address):
    """
    @notice Validate price คือ consistent ระหว่าง oracles
    @dev ถ้า deviation สูงเกินไป ให้ trip circuit breaker
    """
    config: OracleConfig = self.oracleConfigs[_token]
    
    chainlinkPrice: uint256 = self._getChainlinkPrice(config.chainlinkFeed)
    
    if chainlinkPrice == 0:
        log StaleOracleDetected(config.chainlinkFeed, block.timestamp, config.maxStaleness)
        return
    
    # Check if price has moved dramatically from last known good price
    if config.lastValidPrice > 0:
        deviation: uint256 = 0
        if chainlinkPrice > config.lastValidPrice:
            deviation = ((chainlinkPrice - config.lastValidPrice) * 10000) / config.lastValidPrice
        else:
            deviation = ((config.lastValidPrice - chainlinkPrice) * 10000) / config.lastValidPrice
        
        if deviation > config.maxDeviationBps:
            log PriceDeviationAlert(
                _token,
                chainlinkPrice,
                config.lastValidPrice,
                deviation,
                block.timestamp
            )
            
            # Trip circuit breaker for extreme movements
            if deviation > config.maxDeviationBps * 2:
                self.oracleConfigs[_token].circuitBreakerTripped = True
                log OracleCircuitBreakerTripped(
                    _token,
                    "Extreme price deviation detected",
                    block.timestamp
                )
                return
    
    # Update last valid price
    self.oracleConfigs[_token].lastValidPrice = chainlinkPrice
    self.oracleConfigs[_token].lastPriceUpdate = block.timestamp

@external
def resetCircuitBreaker(_token: address):
    """Reset circuit breaker (เฉพาะ owner หรือ guardian)"""
    assert msg.sender == self.owner or msg.sender == self.guardian, "Not authorized"
    
    self.oracleConfigs[_token].circuitBreakerTripped = False

@external
def tripGlobalCircuitBreaker(_reason: String[100]):
    """Trip global circuit breaker"""
    assert msg.sender == self.owner or msg.sender == self.guardian, "Not authorized"
    
    self.globalCircuitBreaker = True
    self.circuitBreakerReason = _reason

@external
def resetGlobalCircuitBreaker():
    """Reset global circuit breaker"""
    assert msg.sender == self.owner, "Not owner"
    self.globalCircuitBreaker = False

@external
def setPriceSanityBounds(_token: address, _min: uint256, _max: uint256):
    """ตั้งค่า price sanity bounds"""
    assert msg.sender == self.owner, "Not owner"
    assert _min < _max, "Invalid bounds"
    
    self.priceSanityMin[_token] = _min
    self.priceSanityMax[_token] = _max
```

---

## 4. Liquidity Risk Management {#liquidity-risk}

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0

"""
@title LiquidityRiskManager - จัดการความเสี่ยงด้าน liquidity
@notice Monitor และป้องกัน bank run scenarios
"""

from vyper.interfaces import ERC20

# Events
event LiquidityAlert:
    alertType: String[50]
    currentRatio: uint256
    threshold: uint256
    timestamp: uint256

event WithdrawalRateLimited:
    user: indexed(address)
    requestedAmount: uint256
    allowedAmount: uint256
    timestamp: uint256

event LiquidityBufferReplenished:
    token: indexed(address)
    amount: uint256
    timestamp: uint256

# Structs
struct LiquidityConfig:
    token: address
    minLiquidityRatio: uint256   # minimum % of deposits to keep liquid (basis points)
    targetLiquidityRatio: uint256
    maxWithdrawalRate: uint256   # max % of liquidity that can be withdrawn per hour
    withdrawalWindowHours: uint256
    withdrawnThisWindow: uint256
    windowStartTime: uint256
    isActive: bool

struct UserWithdrawalTracker:
    dailyWithdrawn: uint256
    dailyLimit: uint256
    lastReset: uint256

# Constants
BASIS_POINTS: constant(uint256) = 10000
SECONDS_PER_HOUR: constant(uint256) = 3600

# State
owner: public(address)
vault: public(address)

liquidityConfigs: public(HashMap[address, LiquidityConfig])
trackedTokens: public(DynArray[address, 20])

userTrackers: public(HashMap[address, HashMap[address, UserWithdrawalTracker]])

emergencyMode: public(bool)
emergencyWithdrawalOnly: public(bool)

@deploy
def __init__(_vault: address):
    self.owner = msg.sender
    self.vault = _vault

@external
def configureLiquidity(
    _token: address,
    _minRatio: uint256,
    _targetRatio: uint256,
    _maxWithdrawalRate: uint256
):
    """ตั้งค่า liquidity parameters"""
    assert msg.sender == self.owner, "Not owner"
    
    self.liquidityConfigs[_token] = LiquidityConfig({
        token: _token,
        minLiquidityRatio: _minRatio,
        targetLiquidityRatio: _targetRatio,
        maxWithdrawalRate: _maxWithdrawalRate,
        withdrawalWindowHours: 1,
        withdrawnThisWindow: 0,
        windowStartTime: block.timestamp,
        isActive: True
    })
    
    self.trackedTokens.append(_token)

@external
def checkAndEnforceWithdrawalLimit(
    _token: address,
    _user: address,
    _amount: uint256
) -> uint256:
    """
    @notice ตรวจสอบและจำกัด withdrawal amount
    @return allowed amount (อาจน้อยกว่า requested)
    """
    config: LiquidityConfig = self.liquidityConfigs[_token]
    
    if not config.isActive:
        return _amount
    
    # Reset window if needed
    if block.timestamp >= config.windowStartTime + config.withdrawalWindowHours * SECONDS_PER_HOUR:
        self.liquidityConfigs[_token].withdrawnThisWindow = 0
        self.liquidityConfigs[_token].windowStartTime = block.timestamp
    
    # Calculate current liquidity
    totalBalance: uint256 = ERC20(_token).balanceOf(self.vault)
    
    # Check minimum liquidity ratio
    if totalBalance == 0:
        return 0
    
    # Max withdrawal for this window
    maxWindowWithdrawal: uint256 = (totalBalance * config.maxWithdrawalRate) / BASIS_POINTS
    alreadyWithdrawn: uint256 = self.liquidityConfigs[_token].withdrawnThisWindow
    remainingAllowance: uint256 = 0
    
    if maxWindowWithdrawal > alreadyWithdrawn:
        remainingAllowance = maxWindowWithdrawal - alreadyWithdrawn
    
    allowedAmount: uint256 = min(_amount, remainingAllowance)
    
    # Check if this would drop below minimum ratio
    afterWithdrawal: uint256 = 0
    if totalBalance > allowedAmount:
        afterWithdrawal = totalBalance - allowedAmount
    
    # If would drop below minimum, reduce allowed amount
    minRequired: uint256 = (totalBalance * config.minLiquidityRatio) / BASIS_POINTS
    if afterWithdrawal < minRequired:
        if totalBalance > minRequired:
            allowedAmount = totalBalance - minRequired
        else:
            allowedAmount = 0
    
    # Update tracking
    self.liquidityConfigs[_token].withdrawnThisWindow += allowedAmount
    
    if allowedAmount < _amount:
        log WithdrawalRateLimited(_user, _amount, allowedAmount, block.timestamp)
    
    # Emit alert if liquidity is getting low
    currentRatio: uint256 = (totalBalance * BASIS_POINTS) / totalBalance
    if totalBalance > 0:
        currentRatio = ((totalBalance - allowedAmount) * BASIS_POINTS) / totalBalance
    
    if currentRatio < config.minLiquidityRatio * 2:
        log LiquidityAlert("LOW_LIQUIDITY", currentRatio, config.minLiquidityRatio, block.timestamp)
    
    return allowedAmount

@external
@view
def getLiquidityRatio(_token: address) -> uint256:
    """ดู liquidity ratio ปัจจุบัน"""
    balance: uint256 = ERC20(_token).balanceOf(self.vault)
    if balance == 0:
        return 0
    
    # Simplified: ratio = liquid balance / total TVL
    return BASIS_POINTS  # 100% if all in vault

@external
@view
def getLiquidityConfig(_token: address) -> LiquidityConfig:
    """ดู liquidity configuration"""
    return self.liquidityConfigs[_token]
```

---

## 5. Risk Scoring Framework {#risk-scoring}

### Comprehensive Risk Score Calculator

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0

"""
@title RiskScoreCalculator - คำนวณ composite risk score ของ protocol
@notice รวม risk factors หลายอย่างเป็น single score
"""

# Events
event RiskScoreCalculated:
    score: uint256
    components: DynArray[uint256, 10]
    timestamp: uint256

event RiskLevelChanged:
    oldLevel: String[10]
    newLevel: String[10]
    score: uint256
    timestamp: uint256

# Structs
struct RiskComponents:
    smartContractRisk: uint256    # 0-100
    liquidityRisk: uint256        # 0-100
    oracleRisk: uint256           # 0-100
    governanceRisk: uint256       # 0-100
    operationalRisk: uint256      # 0-100
    marketRisk: uint256           # 0-100
    compositeScore: uint256       # weighted average 0-100
    riskLevel: String[10]         # "LOW", "MEDIUM", "HIGH", "CRITICAL"
    calculatedAt: uint256

# Weights (basis points, must sum to 10000)
SMART_CONTRACT_WEIGHT: constant(uint256) = 3000   # 30%
LIQUIDITY_WEIGHT: constant(uint256) = 2000         # 20%
ORACLE_WEIGHT: constant(uint256) = 2000            # 20%
GOVERNANCE_WEIGHT: constant(uint256) = 1500        # 15%
OPERATIONAL_WEIGHT: constant(uint256) = 1000       # 10%
MARKET_WEIGHT: constant(uint256) = 500             # 5%

# Thresholds
LOW_RISK_MAX: constant(uint256) = 30
MEDIUM_RISK_MAX: constant(uint256) = 60
HIGH_RISK_MAX: constant(uint256) = 80
# > 80 = CRITICAL

# State
owner: public(address)
operators: public(HashMap[address, bool])

currentRiskComponents: public(RiskComponents)
riskHistory: public(DynArray[RiskComponents, 100])
lastLevel: public(String[10])

@deploy
def __init__():
    self.owner = msg.sender
    self.lastLevel = "UNKNOWN"

@external
def updateRiskScores(
    _smartContract: uint256,
    _liquidity: uint256,
    _oracle: uint256,
    _governance: uint256,
    _operational: uint256,
    _market: uint256
):
    """
    @notice อัปเดต risk component scores
    @dev แต่ละ score คือ 0-100 (0=no risk, 100=maximum risk)
    """
    assert self.operators[msg.sender] or msg.sender == self.owner, "Not authorized"
    
    # Validate all scores are 0-100
    assert _smartContract <= 100, "Invalid SC score"
    assert _liquidity <= 100, "Invalid liquidity score"
    assert _oracle <= 100, "Invalid oracle score"
    assert _governance <= 100, "Invalid governance score"
    assert _operational <= 100, "Invalid operational score"
    assert _market <= 100, "Invalid market score"
    
    # Calculate weighted composite score
    composite: uint256 = (
        _smartContract * SMART_CONTRACT_WEIGHT +
        _liquidity * LIQUIDITY_WEIGHT +
        _oracle * ORACLE_WEIGHT +
        _governance * GOVERNANCE_WEIGHT +
        _operational * OPERATIONAL_WEIGHT +
        _market * MARKET_WEIGHT
    ) / 10000
    
    # Determine risk level
    level: String[10] = "CRITICAL"
    if composite <= LOW_RISK_MAX:
        level = "LOW"
    elif composite <= MEDIUM_RISK_MAX:
        level = "MEDIUM"
    elif composite <= HIGH_RISK_MAX:
        level = "HIGH"
    
    components: RiskComponents = RiskComponents({
        smartContractRisk: _smartContract,
        liquidityRisk: _liquidity,
        oracleRisk: _oracle,
        governanceRisk: _governance,
        operationalRisk: _operational,
        marketRisk: _market,
        compositeScore: composite,
        riskLevel: level,
        calculatedAt: block.timestamp
    })
    
    self.currentRiskComponents = components
    
    if len(self.riskHistory) < 100:
        self.riskHistory.append(components)
    
    # Alert on level change
    if keccak256(level) != keccak256(self.lastLevel):
        log RiskLevelChanged(self.lastLevel, level, composite, block.timestamp)
        self.lastLevel = level
    
    scoreArray: DynArray[uint256, 10] = [
        _smartContract, _liquidity, _oracle,
        _governance, _operational, _market,
        composite
    ]
    log RiskScoreCalculated(composite, scoreArray, block.timestamp)

@external
@view
def getCurrentRiskScore() -> (uint256, String[10]):
    """ดู risk score และ level ปัจจุบัน"""
    return (
        self.currentRiskComponents.compositeScore,
        self.currentRiskComponents.riskLevel
    )

@external
@view
def getRiskBreakdown() -> RiskComponents:
    """ดูรายละเอียด risk components"""
    return self.currentRiskComponents

@external
@view
def getRiskHistory() -> DynArray[RiskComponents, 100]:
    """ดูประวัติ risk scores"""
    return self.riskHistory
```

---

## สรุป: Risk Management Framework

### Risk Response Matrix

```
Risk Score Matrix

         Likelihood
Severity  Rare  Unlikely  Possible  Likely  Almost Certain
Critical   5      8        12       16          20
High       4      6         9       12          15
Medium     3      4         6        8          10
Low        2      2         3        4           5

Score Interpretation:
1-4:   Low Risk     → Monitor, standard controls
5-9:   Medium Risk  → Active mitigation needed
10-15: High Risk    → Urgent action required
16-20: Critical     → Immediate action, consider pause
```

### Protocol Security Score Template

```
Protocol: [Name]
Date: [Date]
Version: [Version]

SECURITY SCORES (0-100, higher = more risk)

1. Smart Contract Risk: __/100
   - Audit coverage: __%
   - Bug bounty active: Yes/No
   - Code age: _ months
   - Complexity score: _

2. Liquidity Risk: __/100
   - TVL concentration: __%
   - Withdrawal queue: __%  
   - Stress test results: Pass/Fail

3. Oracle Risk: __/100
   - Oracle count: _
   - TWAP protection: Yes/No
   - Staleness threshold: _ min

4. Governance Risk: __/100
   - Token concentration: __%
   - Timelock delay: _ days
   - Voter participation: __%

COMPOSITE SCORE: __/100
RISK LEVEL: LOW / MEDIUM / HIGH / CRITICAL
```

> **คำแนะนำ**: Risk management ไม่ใช่เรื่องที่ทำครั้งเดียวแล้วจบ ต้อง review อย่างสม่ำเสมอ โดยเฉพาะหลัง major protocol changes
