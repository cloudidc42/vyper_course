# Part 093: Operational Monitoring

## สารบัญ
1. [ทำไมต้อง Monitor DeFi Protocol](#why-monitor)
2. [Tenderly Dashboards](#tenderly)
3. [OpenZeppelin Defender Monitors](#defender)
4. [การตั้ง Alerting บน Anomalies](#alerting)
5. [TVL Monitoring](#tvl)
6. [Emergency Response Procedures](#emergency)

---

## 1. ทำไมต้อง Monitor DeFi Protocol {#why-monitor}

การ monitoring ที่ดีช่วยให้:
- **ตรวจจับ exploit** ก่อนที่จะสายเกินไป
- **ติดตาม protocol health** แบบ real-time
- **รับรู้ anomalies** ที่อาจเป็นสัญญาณอันตราย
- **ตอบสนองเร็ว** เมื่อมีปัญหา

### สถิติ DeFi Hacks ที่ Monitor ช่วยได้
- Euler Finance (2023): $197M - ป้องกันได้ถ้ามี price anomaly detection
- Ronin Bridge (2022): $625M - validator monitoring
- Nomad Bridge (2022): $190M - unusual withdrawal patterns

---

## 2. Tenderly Dashboards {#tenderly}

### On-chain Monitoring Contract

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0

"""
@title MonitoringOracle - Contract สำหรับ on-chain monitoring
@notice บันทึก metrics สำคัญเพื่อใช้กับ monitoring tools
@dev ออกแบบให้ emit events ที่ Tenderly สามารถ index ได้
"""

from vyper.interfaces import ERC20

# ============================================================
# Events - ทุก event ที่ emit จะถูก Tenderly index
# ============================================================

event MetricRecorded:
    metricName: indexed(String[50])
    value: uint256
    timestamp: uint256
    context: String[100]

event ThresholdBreached:
    metricName: indexed(String[50])
    value: uint256
    threshold: uint256
    severity: String[10]  # "LOW", "MEDIUM", "HIGH", "CRITICAL"
    timestamp: uint256

event HealthCheckPerformed:
    checker: indexed(address)
    allHealthy: bool
    failedChecks: uint256
    timestamp: uint256

event TVLChange:
    token: indexed(address)
    oldTVL: uint256
    newTVL: uint256
    changePercent: int256  # basis points
    timestamp: uint256

event SuspiciousActivity:
    actor: indexed(address)
    activityType: String[50]
    details: String[200]
    timestamp: uint256

event OracleDeviation:
    oracle: indexed(address)
    expectedPrice: uint256
    actualPrice: uint256
    deviationBps: uint256
    timestamp: uint256

# ============================================================
# Structs
# ============================================================

struct MetricThreshold:
    name: String[50]
    lowThreshold: uint256
    highThreshold: uint256
    criticalThreshold: uint256
    lastValue: uint256
    lastUpdated: uint256
    enabled: bool

struct HealthStatus:
    name: String[50]
    isHealthy: bool
    lastChecked: uint256
    failureReason: String[200]

struct AlertConfig:
    recipient: address
    minSeverity: String[10]  # minimum severity to alert
    enabled: bool
    cooldownPeriod: uint256  # prevent spam
    lastAlerted: uint256

# ============================================================
# Constants
# ============================================================

MAX_METRICS: constant(uint256) = 50
MAX_HEALTH_CHECKS: constant(uint256) = 20
BASIS_POINTS: constant(uint256) = 10000

# ============================================================
# State Variables
# ============================================================

owner: public(address)
operators: public(HashMap[address, bool])

# Metrics tracking
metricNames: public(DynArray[String[50], MAX_METRICS])
metrics: public(HashMap[String[50], MetricThreshold])

# Health checks
healthCheckNames: public(DynArray[String[50], MAX_HEALTH_CHECKS])
healthStatus: public(HashMap[String[50], HealthStatus])

# TVL tracking
trackedTokens: public(DynArray[address, 20])
lastTVL: public(HashMap[address, uint256])
vaultAddress: public(address)

# Alert configurations
alertRecipients: public(DynArray[address, 10])
alertConfigs: public(HashMap[address, AlertConfig])

# Emergency state
emergencyMode: public(bool)
lastEmergencyTime: public(uint256)

# ============================================================
# Constructor
# ============================================================

@deploy
def __init__(_vault: address):
    """
    @notice Initialize monitoring oracle
    @param _vault Address ของ main vault contract
    """
    assert _vault != empty(address), "Invalid vault"
    
    self.owner = msg.sender
    self.vaultAddress = _vault

# ============================================================
# Metric Recording
# ============================================================

@external
def recordMetric(_name: String[50], _value: uint256, _context: String[100]):
    """
    @notice บันทึก metric value และตรวจสอบ thresholds
    @param _name ชื่อ metric
    @param _value ค่าที่วัดได้
    @param _context ข้อมูลเพิ่มเติม
    """
    assert self.operators[msg.sender] or msg.sender == self.owner, "Not authorized"
    
    threshold: MetricThreshold = self.metrics[_name]
    
    # Update last value
    self.metrics[_name].lastValue = _value
    self.metrics[_name].lastUpdated = block.timestamp
    
    # Emit metric event
    log MetricRecorded(_name, _value, block.timestamp, _context)
    
    # Check thresholds if metric is configured
    if threshold.enabled:
        if _value >= threshold.criticalThreshold:
            log ThresholdBreached(_name, _value, threshold.criticalThreshold, "CRITICAL", block.timestamp)
        elif _value >= threshold.highThreshold:
            log ThresholdBreached(_name, _value, threshold.highThreshold, "HIGH", block.timestamp)
        elif _value >= threshold.lowThreshold:
            log ThresholdBreached(_name, _value, threshold.lowThreshold, "LOW", block.timestamp)

@external
def recordTVLChange(_token: address):
    """
    @notice ตรวจสอบและบันทึกการเปลี่ยนแปลง TVL
    """
    assert self.operators[msg.sender] or msg.sender == self.owner, "Not authorized"
    
    currentTVL: uint256 = ERC20(_token).balanceOf(self.vaultAddress)
    previousTVL: uint256 = self.lastTVL[_token]
    
    if previousTVL == 0:
        self.lastTVL[_token] = currentTVL
        return
    
    # Calculate change in basis points
    changePercent: int256 = 0
    if currentTVL > previousTVL:
        change: uint256 = currentTVL - previousTVL
        changePercent = convert((change * BASIS_POINTS) / previousTVL, int256)
    elif currentTVL < previousTVL:
        change: uint256 = previousTVL - currentTVL
        changePercent = -convert((change * BASIS_POINTS) / previousTVL, int256)
    
    self.lastTVL[_token] = currentTVL
    
    log TVLChange(_token, previousTVL, currentTVL, changePercent, block.timestamp)
    
    # Alert on large drops (>10%)
    if changePercent < -1000:  # -10% in basis points
        log SuspiciousActivity(
            empty(address),
            "LARGE_TVL_DROP",
            "TVL dropped more than 10%",
            block.timestamp
        )
        
        if changePercent < -2000:  # -20%
            self.emergencyMode = True
            self.lastEmergencyTime = block.timestamp

@external
def reportSuspiciousActivity(
    _actor: address,
    _activityType: String[50],
    _details: String[200]
):
    """
    @notice รายงาน suspicious activity
    """
    assert self.operators[msg.sender] or msg.sender == self.owner, "Not authorized"
    
    log SuspiciousActivity(_actor, _activityType, _details, block.timestamp)

# ============================================================
# Health Checks
# ============================================================

@external
def performHealthCheck():
    """
    @notice ทำ health check ทั้งหมดและรายงานผล
    """
    failedChecks: uint256 = 0
    allHealthy: bool = True
    
    # ตรวจสอบ health checks ทั้งหมด
    for checkName: String[50] in self.healthCheckNames:
        status: HealthStatus = self.healthStatus[checkName]
        if not status.isHealthy:
            failedChecks += 1
            allHealthy = False
    
    log HealthCheckPerformed(msg.sender, allHealthy, failedChecks, block.timestamp)
    
    return

@external
def updateHealthStatus(
    _checkName: String[50],
    _isHealthy: bool,
    _failureReason: String[200]
):
    """อัปเดต health status"""
    assert self.operators[msg.sender] or msg.sender == self.owner, "Not authorized"
    
    self.healthStatus[_checkName] = HealthStatus({
        name: _checkName,
        isHealthy: _isHealthy,
        lastChecked: block.timestamp,
        failureReason: _failureReason
    })

# ============================================================
# Configuration
# ============================================================

@external
def configureMetric(
    _name: String[50],
    _lowThreshold: uint256,
    _highThreshold: uint256,
    _criticalThreshold: uint256
):
    """ตั้งค่า metric threshold"""
    assert msg.sender == self.owner, "Not owner"
    
    existing: MetricThreshold = self.metrics[_name]
    if not existing.enabled:
        self.metricNames.append(_name)
    
    self.metrics[_name] = MetricThreshold({
        name: _name,
        lowThreshold: _lowThreshold,
        highThreshold: _highThreshold,
        criticalThreshold: _criticalThreshold,
        lastValue: existing.lastValue,
        lastUpdated: existing.lastUpdated,
        enabled: True
    })

@external
def addTrackedToken(_token: address):
    """เพิ่ม token ที่ต้อง track TVL"""
    assert msg.sender == self.owner, "Not owner"
    self.trackedTokens.append(_token)

@external
def setOperator(_operator: address, _status: bool):
    """ตั้งค่า operator"""
    assert msg.sender == self.owner, "Not owner"
    self.operators[_operator] = _status

# ============================================================
# View Functions
# ============================================================

@external
@view
def getMetric(_name: String[50]) -> MetricThreshold:
    """ดู metric configuration และ latest value"""
    return self.metrics[_name]

@external
@view
def getAllMetrics() -> DynArray[String[50], MAX_METRICS]:
    """รายการ metrics ทั้งหมด"""
    return self.metricNames

@external
@view
def isEmergencyMode() -> bool:
    """ตรวจสอบ emergency mode"""
    return self.emergencyMode
```

---

## 3. OpenZeppelin Defender Monitors {#defender}

### Defender Monitor Configuration

OpenZeppelin Defender ช่วยให้ monitor events บน blockchain ได้อย่างมีประสิทธิภาพ

```javascript
// defender-monitor-config.js
// Configuration สำหรับ OZ Defender monitors

const monitors = [
  {
    name: "Large Withdrawal Alert",
    type: "BLOCK",
    network: "mainnet",
    addresses: ["0x..."],  // vault contract
    abi: vaultABI,
    
    // Monitor events
    eventConditions: [
      {
        eventSignature: "Withdrawal(address,address,uint256,uint256,uint256)",
        expression: "amount > 100000000000000000000000"  // > 100k tokens
      }
    ],
    
    // Alert configuration
    notification: {
      channels: ["telegram", "email", "pagerduty"],
      message: "🚨 Large withdrawal detected!\nUser: {{args.user}}\nAmount: {{args.amount}}\nTx: {{transaction.hash}}"
    }
  },
  
  {
    name: "Emergency Pause Monitor",
    type: "BLOCK",
    network: "mainnet",
    addresses: ["0x..."],
    
    eventConditions: [
      {
        eventSignature: "EmergencyPause(address,string,uint256)"
      }
    ],
    
    notification: {
      channels: ["pagerduty"],
      severity: "CRITICAL",
      message: "🆘 CONTRACT PAUSED!\nCaller: {{args.caller}}\nReason: {{args.reason}}"
    }
  },
  
  {
    name: "Oracle Price Deviation",
    type: "BLOCK",
    network: "mainnet",
    
    eventConditions: [
      {
        eventSignature: "OracleDeviation(address,uint256,uint256,uint256)",
        expression: "deviationBps > 500"  // > 5% deviation
      }
    ],
    
    notification: {
      channels: ["telegram", "pagerduty"],
      message: "⚠️ Oracle price deviation!\nOracle: {{args.oracle}}\nDeviation: {{args.deviationBps}} bps"
    }
  },
  
  {
    name: "Admin Action Monitor",
    type: "BLOCK",
    network: "mainnet",
    
    // Monitor any ownership or admin changes
    eventConditions: [
      {
        eventSignature: "OwnershipTransferred(address,address)"
      },
      {
        eventSignature: "FeeUpdated(string,uint256,uint256)"
      }
    ],
    
    notification: {
      channels: ["telegram"],
      message: "📢 Admin action detected!\nEvent: {{eventSignature}}\nTx: {{transaction.hash}}"
    }
  }
];
```

### Defender Autotask สำหรับ Automated Response

```javascript
// autotask-emergency-response.js
// Autotask ที่จะ execute เมื่อตรวจพบ emergency

const { ethers } = require("ethers");
const { DefenderRelayProvider, DefenderRelaySigner } = require("@openzeppelin/defender-relay-client/lib/ethers");

const VAULT_ADDRESS = "0x...";
const VAULT_ABI = [...]; // ABI

exports.handler = async function(credentials) {
  const provider = new DefenderRelayProvider(credentials);
  const signer = new DefenderRelaySigner(credentials, provider, { speed: 'fast' });
  
  const vault = new ethers.Contract(VAULT_ADDRESS, VAULT_ABI, signer);
  
  // ตรวจสอบสถานะก่อน pause
  const isPaused = await vault.isPaused();
  if (isPaused) {
    console.log("Already paused");
    return { status: "already_paused" };
  }
  
  // Emergency pause
  try {
    const tx = await vault.pause("Automated emergency pause - anomaly detected");
    const receipt = await tx.wait();
    
    console.log(`Emergency pause executed. Tx: ${receipt.transactionHash}`);
    
    return {
      status: "paused",
      txHash: receipt.transactionHash
    };
  } catch (error) {
    console.error("Failed to pause:", error);
    throw error;
  }
};
```

---

## 4. การตั้ง Alerting บน Anomalies {#alerting}

### Anomaly Detection Contract

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0

"""
@title AnomalyDetector - ตรวจจับ anomalies ใน on-chain data
@notice ใช้ statistical methods เพื่อตรวจสอบ unusual patterns
"""

from vyper.interfaces import ERC20

# ============================================================
# Events
# ============================================================

event AnomalyDetected:
    anomalyType: indexed(String[50])
    severity: String[10]
    description: String[200]
    value: uint256
    baseline: uint256
    timestamp: uint256

event BaselineUpdated:
    metricName: indexed(String[50])
    newBaseline: uint256
    dataPoints: uint256

event FlashLoanDetected:
    borrower: indexed(address)
    token: indexed(address)
    amount: uint256
    timestamp: uint256

# ============================================================
# Structs
# ============================================================

struct MetricBaseline:
    name: String[50]
    average: uint256
    standardDeviation: uint256
    dataPoints: uint256
    lastUpdated: uint256
    windowSize: uint256  # number of data points to consider
    
struct PriceSnapshot:
    price: uint256
    timestamp: uint256
    reporter: address

struct VolumeData:
    hourlyVolume: uint256
    dailyVolume: uint256
    weeklyAverage: uint256
    lastUpdated: uint256

# ============================================================
# Constants  
# ============================================================

MAX_PRICE_HISTORY: constant(uint256) = 100
DEVIATION_THRESHOLD_BPS: constant(uint256) = 500  # 5%
FLASH_LOAN_BLOCK_GAP: constant(uint256) = 1  # same block = flash loan suspect

# ============================================================
# State Variables
# ============================================================

owner: public(address)
operators: public(HashMap[address, bool])

# Price history for anomaly detection
priceHistory: public(DynArray[PriceSnapshot, MAX_PRICE_HISTORY])
baselines: public(HashMap[String[50], MetricBaseline])

# Volume tracking
tokenVolumes: public(HashMap[address, VolumeData])

# Flash loan detection
lastBalanceSnapshot: public(HashMap[address, HashMap[address, uint256]])
lastSnapshotBlock: public(HashMap[address, uint256])

# Anomaly log
anomalyCount: public(uint256)
lastAnomalyTime: public(uint256)

# ============================================================
# Constructor
# ============================================================

@deploy
def __init__():
    self.owner = msg.sender

# ============================================================
# Price Anomaly Detection
# ============================================================

@external
def recordPrice(_token: address, _price: uint256):
    """
    @notice บันทึก price และตรวจหา anomaly
    @param _token Token address
    @param _price Current price (in USD, 18 decimals)
    """
    assert self.operators[msg.sender] or msg.sender == self.owner, "Not authorized"
    
    snapshot: PriceSnapshot = PriceSnapshot({
        price: _price,
        timestamp: block.timestamp,
        reporter: msg.sender
    })
    
    # Check for price anomaly if we have history
    if len(self.priceHistory) > 0:
        lastPrice: uint256 = self.priceHistory[len(self.priceHistory) - 1].price
        
        if lastPrice > 0:
            deviation: uint256 = 0
            
            if _price > lastPrice:
                deviation = ((_price - lastPrice) * 10000) / lastPrice
            else:
                deviation = ((lastPrice - _price) * 10000) / lastPrice
            
            # Alert if deviation > 5%
            if deviation > DEVIATION_THRESHOLD_BPS:
                severity: String[10] = "HIGH"
                if deviation > 2000:  # >20%
                    severity = "CRITICAL"
                elif deviation > 1000:  # >10%
                    severity = "MEDIUM"
                
                log AnomalyDetected(
                    "PRICE_ANOMALY",
                    severity,
                    "Unusual price movement detected",
                    _price,
                    lastPrice,
                    block.timestamp
                )
                
                self.anomalyCount += 1
                self.lastAnomalyTime = block.timestamp
    
    # Add to history (keep last MAX_PRICE_HISTORY entries)
    if len(self.priceHistory) >= MAX_PRICE_HISTORY:
        # Shift history (simplified - remove oldest)
        newHistory: DynArray[PriceSnapshot, MAX_PRICE_HISTORY] = []
        for i: uint256 in range(MAX_PRICE_HISTORY):
            if i > 0 and i < len(self.priceHistory):
                newHistory.append(self.priceHistory[i])
        self.priceHistory = newHistory
    
    self.priceHistory.append(snapshot)

@external
def detectFlashLoan(_token: address, _vault: address):
    """
    @notice ตรวจจับ flash loan patterns
    """
    currentBalance: uint256 = ERC20(_token).balanceOf(_vault)
    lastBalance: uint256 = self.lastBalanceSnapshot[_vault][_token]
    lastBlock: uint256 = self.lastSnapshotBlock[_vault]
    
    # ถ้ามี balance เพิ่มขึ้นมากใน block เดียว
    if lastBlock == block.number and lastBalance > 0:
        if currentBalance > lastBalance * 2:  # doubled in same block
            log FlashLoanDetected(
                msg.sender,
                _token,
                currentBalance - lastBalance,
                block.timestamp
            )
            
            log AnomalyDetected(
                "FLASH_LOAN_SUSPECTED",
                "HIGH",
                "Large balance increase in single block",
                currentBalance,
                lastBalance,
                block.timestamp
            )
    
    # Update snapshot
    self.lastBalanceSnapshot[_vault][_token] = currentBalance
    self.lastSnapshotBlock[_vault] = block.number

@external
def checkVolumeAnomaly(_token: address, _newVolume: uint256):
    """
    @notice ตรวจสอบ volume anomaly
    """
    assert self.operators[msg.sender] or msg.sender == self.owner, "Not authorized"
    
    volumeData: VolumeData = self.tokenVolumes[_token]
    
    if volumeData.weeklyAverage > 0:
        # ถ้า hourly volume > 3x daily average
        dailyHourlyAvg: uint256 = volumeData.dailyVolume / 24
        
        if dailyHourlyAvg > 0 and _newVolume > dailyHourlyAvg * 3:
            log AnomalyDetected(
                "VOLUME_SPIKE",
                "MEDIUM",
                "Unusual volume spike detected",
                _newVolume,
                dailyHourlyAvg,
                block.timestamp
            )
    
    # Update volume data
    self.tokenVolumes[_token].hourlyVolume = _newVolume
    self.tokenVolumes[_token].lastUpdated = block.timestamp

# ============================================================
# View Functions
# ============================================================

@external
@view
def getPriceHistory() -> DynArray[PriceSnapshot, MAX_PRICE_HISTORY]:
    """ดูประวัติราคา"""
    return self.priceHistory

@external
@view
def getLatestPrice() -> uint256:
    """ดูราคาล่าสุด"""
    if len(self.priceHistory) == 0:
        return 0
    return self.priceHistory[len(self.priceHistory) - 1].price

@external
@view
def getAnomalyStats() -> (uint256, uint256):
    """ดูสถิติ anomalies"""
    return self.anomalyCount, self.lastAnomalyTime
```

---

## 5. TVL Monitoring {#tvl}

### TVL Tracking System

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0

"""
@title TVLMonitor - ระบบ monitoring Total Value Locked
@notice ติดตาม TVL แบบ real-time พร้อม historical data
"""

from vyper.interfaces import ERC20

# ============================================================
# Events
# ============================================================

event TVLSnapshot:
    timestamp: uint256
    totalTVL: uint256
    tokenCount: uint256

event TokenTVLUpdated:
    token: indexed(address)
    tvl: uint256
    priceUSD: uint256
    tvlUSD: uint256
    timestamp: uint256

event TVLAlert:
    alertType: String[50]
    currentTVL: uint256
    threshold: uint256
    severity: String[10]
    timestamp: uint256

event ProtocolMetrics:
    totalUsers: uint256
    dailyVolume: uint256
    totalFees: uint256
    timestamp: uint256

# ============================================================
# Structs
# ============================================================

struct TokenTVLData:
    token: address
    balance: uint256
    priceUSD: uint256       # 8 decimals
    tvlUSD: uint256         # 8 decimals
    lastUpdated: uint256
    change24h: int256       # basis points

struct TVLHistoryPoint:
    timestamp: uint256
    totalTVLUSD: uint256
    tokenCount: uint256

struct AlertThreshold:
    minTVL: uint256          # Alert if TVL drops below
    maxDropPercent: uint256   # Alert if drops by X% in 1 hour
    enabled: bool

# ============================================================
# Constants
# ============================================================

MAX_TOKENS: constant(uint256) = 30
TVL_HISTORY_SIZE: constant(uint256) = 168  # 7 days hourly

# ============================================================
# State Variables
# ============================================================

owner: public(address)
operators: public(HashMap[address, bool])

# Vault and oracle addresses
vaultAddress: public(address)
priceOracle: public(address)

# TVL data
trackedTokens: public(DynArray[address, MAX_TOKENS])
tokenTVL: public(HashMap[address, TokenTVLData])
totalTVLUSD: public(uint256)

# Historical data
tvlHistory: public(DynArray[TVLHistoryPoint, TVL_HISTORY_SIZE])
lastSnapshotHour: public(uint256)

# Alert configuration
alertThresholds: public(AlertThreshold)
tvlAtLastHour: public(uint256)

# Protocol metrics
totalUsers: public(uint256)
dailyVolume: public(uint256)
totalFeesCollected: public(uint256)

# ============================================================
# Constructor
# ============================================================

@deploy
def __init__(_vault: address, _oracle: address):
    """
    @notice Initialize TVL monitor
    @param _vault Main vault contract
    @param _oracle Price oracle contract
    """
    assert _vault != empty(address), "Invalid vault"
    
    self.owner = msg.sender
    self.vaultAddress = _vault
    self.priceOracle = _oracle
    
    # Default alert thresholds
    self.alertThresholds = AlertThreshold({
        minTVL: 100000 * 10**8,      # $100k minimum TVL
        maxDropPercent: 1000,          # Alert on 10% drop per hour
        enabled: True
    })

# ============================================================
# TVL Update Functions
# ============================================================

@external
def updateTokenTVL(_token: address, _priceUSD: uint256):
    """
    @notice อัปเดต TVL สำหรับ token
    @param _token Token address
    @param _priceUSD Current price in USD (8 decimals)
    """
    assert self.operators[msg.sender] or msg.sender == self.owner, "Not authorized"
    
    # Get current balance
    balance: uint256 = ERC20(_token).balanceOf(self.vaultAddress)
    
    # Calculate TVL in USD
    # Assuming token has 18 decimals, price has 8 decimals
    # TVL = balance * price / 10^18 (keeps 8 decimal precision)
    tvlUSD: uint256 = 0
    if balance > 0 and _priceUSD > 0:
        tvlUSD = (balance * _priceUSD) / (10**18)
    
    # Calculate 24h change
    oldTVL: TokenTVLData = self.tokenTVL[_token]
    change24h: int256 = 0
    
    if oldTVL.tvlUSD > 0:
        if tvlUSD > oldTVL.tvlUSD:
            change: uint256 = tvlUSD - oldTVL.tvlUSD
            change24h = convert((change * 10000) / oldTVL.tvlUSD, int256)
        elif tvlUSD < oldTVL.tvlUSD:
            change: uint256 = oldTVL.tvlUSD - tvlUSD
            change24h = -convert((change * 10000) / oldTVL.tvlUSD, int256)
    
    # Update token TVL
    self.tokenTVL[_token] = TokenTVLData({
        token: _token,
        balance: balance,
        priceUSD: _priceUSD,
        tvlUSD: tvlUSD,
        lastUpdated: block.timestamp,
        change24h: change24h
    })
    
    log TokenTVLUpdated(_token, balance, _priceUSD, tvlUSD, block.timestamp)
    
    # Recalculate total TVL
    self._updateTotalTVL()

@internal
def _updateTotalTVL():
    """คำนวณ total TVL ใหม่"""
    newTotal: uint256 = 0
    
    for token: address in self.trackedTokens:
        newTotal += self.tokenTVL[token].tvlUSD
    
    oldTotal: uint256 = self.totalTVLUSD
    self.totalTVLUSD = newTotal
    
    # Check alerts
    if self.alertThresholds.enabled:
        # Check minimum TVL
        if newTotal < self.alertThresholds.minTVL:
            log TVLAlert(
                "TVL_BELOW_MINIMUM",
                newTotal,
                self.alertThresholds.minTVL,
                "HIGH",
                block.timestamp
            )
        
        # Check hourly drop
        if self.tvlAtLastHour > 0:
            if newTotal < self.tvlAtLastHour:
                drop: uint256 = self.tvlAtLastHour - newTotal
                dropPercent: uint256 = (drop * 10000) / self.tvlAtLastHour
                
                if dropPercent >= self.alertThresholds.maxDropPercent:
                    severity: String[10] = "HIGH"
                    if dropPercent >= self.alertThresholds.maxDropPercent * 2:
                        severity = "CRITICAL"
                    
                    log TVLAlert(
                        "TVL_DROP_ALERT",
                        newTotal,
                        self.tvlAtLastHour,
                        severity,
                        block.timestamp
                    )
    
    # Take hourly snapshot
    currentHour: uint256 = block.timestamp / 3600
    if currentHour > self.lastSnapshotHour:
        self._takeSnapshot(newTotal)
        self.tvlAtLastHour = newTotal
        self.lastSnapshotHour = currentHour

@internal
def _takeSnapshot(_totalTVL: uint256):
    """บันทึก TVL snapshot"""
    point: TVLHistoryPoint = TVLHistoryPoint({
        timestamp: block.timestamp,
        totalTVLUSD: _totalTVL,
        tokenCount: len(self.trackedTokens)
    })
    
    # Maintain circular buffer
    if len(self.tvlHistory) >= TVL_HISTORY_SIZE:
        newHistory: DynArray[TVLHistoryPoint, TVL_HISTORY_SIZE] = []
        for i: uint256 in range(TVL_HISTORY_SIZE):
            if i > 0 and i < len(self.tvlHistory):
                newHistory.append(self.tvlHistory[i])
        self.tvlHistory = newHistory
    
    self.tvlHistory.append(point)
    
    log TVLSnapshot(block.timestamp, _totalTVL, len(self.trackedTokens))

# ============================================================
# Protocol Metrics
# ============================================================

@external
def updateProtocolMetrics(
    _totalUsers: uint256,
    _dailyVolume: uint256,
    _totalFees: uint256
):
    """อัปเดต protocol metrics"""
    assert self.operators[msg.sender] or msg.sender == self.owner, "Not authorized"
    
    self.totalUsers = _totalUsers
    self.dailyVolume = _dailyVolume
    self.totalFeesCollected = _totalFees
    
    log ProtocolMetrics(_totalUsers, _dailyVolume, _totalFees, block.timestamp)

# ============================================================
# Admin Functions
# ============================================================

@external
def addTrackedToken(_token: address):
    """เพิ่ม token ที่ต้อง track"""
    assert msg.sender == self.owner, "Not owner"
    assert len(self.trackedTokens) < MAX_TOKENS, "Too many tokens"
    self.trackedTokens.append(_token)

@external
def configureAlerts(
    _minTVL: uint256,
    _maxDropPercent: uint256,
    _enabled: bool
):
    """ตั้งค่า alert thresholds"""
    assert msg.sender == self.owner, "Not owner"
    
    self.alertThresholds = AlertThreshold({
        minTVL: _minTVL,
        maxDropPercent: _maxDropPercent,
        enabled: _enabled
    })

# ============================================================
# View Functions
# ============================================================

@external
@view
def getTVLHistory() -> DynArray[TVLHistoryPoint, TVL_HISTORY_SIZE]:
    """ดู TVL history"""
    return self.tvlHistory

@external
@view
def getAllTokenTVL() -> DynArray[TokenTVLData, MAX_TOKENS]:
    """ดู TVL ของทุก token"""
    result: DynArray[TokenTVLData, MAX_TOKENS] = []
    
    for token: address in self.trackedTokens:
        result.append(self.tokenTVL[token])
    
    return result

@external
@view
def getProtocolStats() -> (uint256, uint256, uint256, uint256):
    """ดู protocol statistics"""
    return (
        self.totalTVLUSD,
        self.totalUsers,
        self.dailyVolume,
        self.totalFeesCollected
    )
```

---

## 6. Emergency Response Procedures {#emergency}

### Emergency Response Contract

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0

"""
@title EmergencyResponse - ระบบตอบสนองเหตุฉุกเฉิน
@notice จัดการ emergency scenarios ต่างๆ อย่างเป็นระบบ
@dev Implements circuit breaker และ emergency withdrawal patterns
"""

from vyper.interfaces import ERC20

# ============================================================
# Events
# ============================================================

event EmergencyDeclared:
    severity: indexed(uint256)
    declaredBy: indexed(address)
    reason: String[200]
    timestamp: uint256

event CircuitBreakerTripped:
    reason: String[100]
    timestamp: uint256

event CircuitBreakerReset:
    resetBy: indexed(address)
    timestamp: uint256

event EmergencyWithdrawal:
    token: indexed(address)
    recipient: indexed(address)
    amount: uint256
    timestamp: uint256

event EmergencyActionExecuted:
    actionType: String[50]
    executor: indexed(address)
    timestamp: uint256

event IncidentResolved:
    incidentId: indexed(uint256)
    resolvedBy: indexed(address)
    resolution: String[200]
    timestamp: uint256

# ============================================================
# Structs
# ============================================================

struct Incident:
    id: uint256
    severity: uint256        # 1=LOW, 2=MEDIUM, 3=HIGH, 4=CRITICAL
    description: String[200]
    reportedBy: address
    reportedAt: uint256
    resolved: bool
    resolvedAt: uint256
    resolution: String[200]

struct CircuitBreakerState:
    isTripped: bool
    tripReason: String[100]
    tripTime: uint256
    tripCount: uint256
    lastReset: uint256
    autoResetAfter: uint256  # seconds, 0 = manual only

struct EmergencyContact:
    name: String[50]
    address_: address
    role: String[50]
    canPause: bool
    canWithdraw: bool

# ============================================================
# Constants
# ============================================================

MAX_INCIDENTS: constant(uint256) = 100
MAX_CONTACTS: constant(uint256) = 10
SEVERITY_LOW: constant(uint256) = 1
SEVERITY_MEDIUM: constant(uint256) = 2
SEVERITY_HIGH: constant(uint256) = 3
SEVERITY_CRITICAL: constant(uint256) = 4

# ============================================================
# State Variables
# ============================================================

owner: public(address)
protectedContract: public(address)

# Circuit breaker
circuitBreaker: public(CircuitBreakerState)

# Incidents
incidents: public(HashMap[uint256, Incident])
incidentCount: public(uint256)
activeIncidents: public(DynArray[uint256, MAX_INCIDENTS])

# Emergency contacts (on-call team)
contacts: public(DynArray[EmergencyContact, MAX_CONTACTS])
isEmergencyContact: public(HashMap[address, bool])

# Emergency funds
emergencyFundAddress: public(address)

# Response playbooks
playbooks: public(HashMap[uint256, String[500]])  # severity -> playbook

# ============================================================
# Constructor
# ============================================================

@deploy
def __init__(_protectedContract: address, _emergencyFund: address):
    """
    @notice Initialize emergency response system
    @param _protectedContract Contract to protect
    @param _emergencyFund Emergency fund address
    """
    assert _protectedContract != empty(address), "Invalid contract"
    assert _emergencyFund != empty(address), "Invalid fund"
    
    self.owner = msg.sender
    self.protectedContract = _protectedContract
    self.emergencyFundAddress = _emergencyFund
    
    # Initialize circuit breaker
    self.circuitBreaker = CircuitBreakerState({
        isTripped: False,
        tripReason: "",
        tripTime: 0,
        tripCount: 0,
        lastReset: block.timestamp,
        autoResetAfter: 24 * 3600  # 24 hours auto reset
    })
    
    # Set default playbooks
    self.playbooks[SEVERITY_LOW] = "1. Log incident 2. Monitor closely 3. Notify team lead"
    self.playbooks[SEVERITY_MEDIUM] = "1. Alert on-call team 2. Prepare pause 3. Investigate"
    self.playbooks[SEVERITY_HIGH] = "1. PAUSE CONTRACT IMMEDIATELY 2. Alert all stakeholders 3. Post-mortem"
    self.playbooks[SEVERITY_CRITICAL] = "1. PAUSE NOW 2. Emergency withdrawal 3. All hands incident"

# ============================================================
# Circuit Breaker
# ============================================================

@external
def tripCircuitBreaker(_reason: String[100]):
    """
    @notice ตัด circuit breaker หยุด operations
    """
    assert (
        self.isEmergencyContact[msg.sender] or 
        msg.sender == self.owner
    ), "Not authorized"
    
    self.circuitBreaker.isTripped = True
    self.circuitBreaker.tripReason = _reason
    self.circuitBreaker.tripTime = block.timestamp
    self.circuitBreaker.tripCount += 1
    
    log CircuitBreakerTripped(_reason, block.timestamp)

@external
def resetCircuitBreaker():
    """
    @notice Reset circuit breaker (เฉพาะ owner)
    """
    assert msg.sender == self.owner, "Not owner"
    
    cb: CircuitBreakerState = self.circuitBreaker
    
    # ถ้าไม่ได้ตั้ง auto reset ต้องรอ cooldown
    if cb.autoResetAfter > 0:
        assert block.timestamp >= cb.tripTime + cb.autoResetAfter, "Cooldown not passed"
    
    self.circuitBreaker.isTripped = False
    self.circuitBreaker.lastReset = block.timestamp
    
    log CircuitBreakerReset(msg.sender, block.timestamp)

@external
@view
def isCircuitBreakerTripped() -> bool:
    """ตรวจสอบสถานะ circuit breaker"""
    return self.circuitBreaker.isTripped

# ============================================================
# Incident Management
# ============================================================

@external
def reportIncident(
    _severity: uint256,
    _description: String[200]
) -> uint256:
    """
    @notice รายงาน incident ใหม่
    @return incidentId
    """
    assert (
        self.isEmergencyContact[msg.sender] or 
        msg.sender == self.owner
    ), "Not authorized"
    assert _severity >= 1 and _severity <= 4, "Invalid severity"
    
    incidentId: uint256 = self.incidentCount
    self.incidentCount += 1
    
    self.incidents[incidentId] = Incident({
        id: incidentId,
        severity: _severity,
        description: _description,
        reportedBy: msg.sender,
        reportedAt: block.timestamp,
        resolved: False,
        resolvedAt: 0,
        resolution: ""
    })
    
    self.activeIncidents.append(incidentId)
    
    log EmergencyDeclared(_severity, msg.sender, _description, block.timestamp)
    
    # Auto actions based on severity
    if _severity >= SEVERITY_HIGH:
        self.tripCircuitBreaker("Incident severity >= HIGH")
    
    return incidentId

@external
def resolveIncident(_incidentId: uint256, _resolution: String[200]):
    """
    @notice แก้ไข incident
    """
    assert msg.sender == self.owner, "Not owner"
    assert _incidentId < self.incidentCount, "Invalid incident"
    assert not self.incidents[_incidentId].resolved, "Already resolved"
    
    self.incidents[_incidentId].resolved = True
    self.incidents[_incidentId].resolvedAt = block.timestamp
    self.incidents[_incidentId].resolution = _resolution
    
    # Remove from active list
    newActive: DynArray[uint256, MAX_INCIDENTS] = []
    for id: uint256 in self.activeIncidents:
        if id != _incidentId:
            newActive.append(id)
    self.activeIncidents = newActive
    
    log IncidentResolved(_incidentId, msg.sender, _resolution, block.timestamp)

# ============================================================
# Emergency Actions
# ============================================================

@external
def emergencyWithdraw(_token: address, _amount: uint256, _recipient: address):
    """
    @notice ถอน funds ในกรณีฉุกเฉิน
    @dev เฉพาะ owner และต้องมี critical incident
    """
    assert msg.sender == self.owner, "Not owner"
    
    # ต้องมี critical incident ที่ยังไม่ resolved
    hasCritical: bool = False
    for id: uint256 in self.activeIncidents:
        if self.incidents[id].severity >= SEVERITY_CRITICAL:
            hasCritical = True
            break
    
    assert hasCritical, "No critical incident"
    assert _recipient != empty(address), "Invalid recipient"
    
    balance: uint256 = ERC20(_token).balanceOf(self)
    withdrawAmount: uint256 = min(_amount, balance)
    
    assert ERC20(_token).transfer(_recipient, withdrawAmount), "Transfer failed"
    
    log EmergencyWithdrawal(_token, _recipient, withdrawAmount, block.timestamp)

# ============================================================
# Contact Management
# ============================================================

@external
def addEmergencyContact(
    _name: String[50],
    _address: address,
    _role: String[50],
    _canPause: bool,
    _canWithdraw: bool
):
    """เพิ่ม emergency contact"""
    assert msg.sender == self.owner, "Not owner"
    assert len(self.contacts) < MAX_CONTACTS, "Too many contacts"
    
    self.contacts.append(EmergencyContact({
        name: _name,
        address_: _address,
        role: _role,
        canPause: _canPause,
        canWithdraw: _canWithdraw
    }))
    
    self.isEmergencyContact[_address] = True

# ============================================================
# View Functions
# ============================================================

@external
@view
def getActiveIncidents() -> DynArray[uint256, MAX_INCIDENTS]:
    """ดู active incidents"""
    return self.activeIncidents

@external
@view
def getIncident(_id: uint256) -> Incident:
    """ดูรายละเอียด incident"""
    return self.incidents[_id]

@external
@view
def getPlaybook(_severity: uint256) -> String[500]:
    """ดู playbook สำหรับ severity นั้น"""
    return self.playbooks[_severity]

@external
@view
def getEmergencyContacts() -> DynArray[EmergencyContact, MAX_CONTACTS]:
    """รายชื่อ emergency contacts"""
    return self.contacts

@external
@view
def getSystemStatus() -> (bool, bool, uint256, uint256):
    """
    ดูสถานะระบบโดยรวม
    Returns: (circuitBreakerTripped, hasActiveIncidents, activeIncidentCount, criticalCount)
    """
    criticalCount: uint256 = 0
    for id: uint256 in self.activeIncidents:
        if self.incidents[id].severity >= SEVERITY_CRITICAL:
            criticalCount += 1
    
    return (
        self.circuitBreaker.isTripped,
        len(self.activeIncidents) > 0,
        len(self.activeIncidents),
        criticalCount
    )
```

---

## สรุป: Emergency Response Runbook

```markdown
# Emergency Response Runbook

## Severity Levels

### CRITICAL (Severity 4)
- Active exploit in progress
- Funds at immediate risk
- Actions: PAUSE NOW → Emergency withdrawal → All hands

### HIGH (Severity 3) 
- Suspicious unusual activity
- Large unexpected TVL drop (>20%)
- Actions: Pause contract → Investigate → Notify stakeholders

### MEDIUM (Severity 2)
- Anomaly detected but no immediate risk
- Oracle price deviation
- Actions: Alert team → Monitor closely → Prepare response

### LOW (Severity 1)
- Unusual but not critical patterns
- Minor configuration issues
- Actions: Log → Monitor → Review

## Response Times
- CRITICAL: < 5 minutes
- HIGH: < 15 minutes  
- MEDIUM: < 1 hour
- LOW: < 24 hours

## Communication Channels
1. Internal Telegram group (immediate)
2. Discord announcement (community)
3. Twitter (public update)
4. Blog post (detailed post-mortem)
```

---

> **สิ่งที่จำไว้**: การ monitor ที่ดีคือการตรวจพบปัญหาก่อนที่จะเกิดความเสียหาย ลงทุนใน monitoring infrastructure ตั้งแต่วันแรก
