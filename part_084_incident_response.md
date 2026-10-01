# Part 084: Incident Response สำหรับ Smart Contracts

## สารบัญ
1. Incident Response Planning
2. Emergency Procedures
3. Pause Mechanisms
4. Fund Recovery Patterns
5. EmergencyProtocol Contract
6. Post-incident Analysis
7. Communication Plan

---

## 1. Incident Response Planning

### ระดับของ Incidents

```
Level 1 - Low: ปัญหาเล็กน้อย ไม่กระทบ funds
  - Bug ใน non-critical functions
  - UI issues
  - Gas optimization problems

Level 2 - Medium: ปัญหาที่อาจกระทบ user experience
  - Incorrect calculations ที่เล็กน้อย
  - oracle deviation
  - Temporary service disruption

Level 3 - High: ปัญหาที่กระทบ funds บางส่วน
  - Funds at risk แต่ยังไม่ถูก drain
  - Economic attack ที่กำลังดำเนิน

Level 4 - Critical: Active exploit ที่กำลัง drain funds
  - Reentrancy attack
  - Oracle manipulation
  - Flash loan attack
```

### Pre-incident Checklist

```
□ Emergency contact list ของทีม
□ Multi-sig wallet พร้อมใช้งาน
□ Pause function ทดสอบแล้ว
□ Emergency withdraw สำหรับ users
□ Communication channels (Twitter, Discord)
□ Legal contact
□ Security firm contact (Trail of Bits, etc.)
□ Insurance information
□ Fund recovery scripts
```

---

## 2. EmergencyProtocol Contract

```vyper
# @version 0.4.0
# @title Emergency Protocol
# @notice ระบบจัดการ emergency ที่ครบวงจร

from vyper.interfaces import ERC20

# ===== Events =====
event EmergencyDeclared:
    level: indexed(uint8)
    declaredBy: indexed(address)
    reason: String[512]
    timestamp: uint256

event EmergencyResolved:
    resolvedBy: indexed(address)
    resolution: String[256]
    timestamp: uint256

event FundsSeized:
    token: indexed(address)
    amount: uint256
    destination: indexed(address)
    authorizedBy: indexed(address)

event EmergencyWithdrawEnabled:
    enabledBy: indexed(address)
    timestamp: uint256

event UserEmergencyWithdraw:
    user: indexed(address)
    token: indexed(address)
    amount: uint256

event GuardianAdded:
    guardian: indexed(address)
    addedBy: indexed(address)

event GuardianRemoved:
    guardian: indexed(address)
    removedBy: indexed(address)

event TimelockSet:
    action: indexed(bytes32)
    executeAfter: uint256
    proposedBy: indexed(address)

event TimelockExecuted:
    action: indexed(bytes32)
    executedBy: indexed(address)

# ===== Structs =====
struct TimelockAction:
    proposedAt: uint256
    executeAfter: uint256
    executed: bool
    proposedBy: address
    data: Bytes[1024]

# ===== Constants =====
GUARDIAN_THRESHOLD: constant(uint256) = 3  # ต้องการ 3/5 guardians
EMERGENCY_TIMELOCK: constant(uint256) = 0  # 0 = immediate (ใน emergency)
NORMAL_TIMELOCK: constant(uint256) = 48 * 3600  # 48 hours
MAX_GUARDIANS: constant(uint256) = 10

# ===== State =====
owner: public(address)
emergencyLevel: public(uint8)  # 0=normal, 1=low, 2=medium, 3=high, 4=critical
emergencyTimestamp: public(uint256)
emergencyWithdrawEnabled: public(bool)
paused: public(bool)

# Guardians
guardians: public(DynArray[address, MAX_GUARDIANS])
isGuardian: public(HashMap[address, bool])
guardianVotes: HashMap[bytes32, DynArray[address, MAX_GUARDIANS]]

# Timelocked actions
timelockActions: HashMap[bytes32, TimelockAction]

# User balances (สำหรับ emergency withdraw)
userBalances: HashMap[address, HashMap[address, uint256]]  # user => token => amount
supportedTokens: public(DynArray[address, 50])
isSupported: HashMap[address, bool]

@deploy
def __init__(
    _owner: address,
    _guardians: DynArray[address, MAX_GUARDIANS]
):
    assert _owner != empty(address), "Invalid owner"
    assert len(_guardians) >= GUARDIAN_THRESHOLD, "Insufficient guardians"
    
    self.owner = _owner
    
    for guardian in _guardians:
        assert guardian != empty(address), "Invalid guardian"
        self.guardians.append(guardian)
        self.isGuardian[guardian] = True

# ===== EMERGENCY FUNCTIONS =====

@external
def declareEmergency(level: uint8, reason: String[512]):
    """
    ประกาศ emergency
    Level 4 (Critical) ต้องการ guardian threshold
    """
    assert level > 0 and level <= 4, "Invalid level"
    
    if level == 4:
        # Critical: ต้องการ guardian votes หรือ owner
        assert msg.sender == self.owner or self.isGuardian[msg.sender], "Not authorized"
        
        if msg.sender != self.owner:
            # Guardian vote
            voteKey: bytes32 = keccak256(
                _abi_encode(convert("declareEmergency4", Bytes[32]), block.number)
            )
            self._recordGuardianVote(voteKey)
            
            if not self._hasEnoughVotes(voteKey):
                return  # Not enough votes yet
    else:
        assert msg.sender == self.owner, "Not owner"
    
    self.emergencyLevel = level
    self.emergencyTimestamp = block.timestamp
    self.paused = True
    
    log EmergencyDeclared(level, msg.sender, reason, block.timestamp)

@external
def enableEmergencyWithdraw():
    """
    เปิด emergency withdraw สำหรับ users
    ต้องการ owner หรือ guardian threshold
    """
    assert self.emergencyLevel >= 3, "Not emergency"
    
    if msg.sender != self.owner:
        assert self.isGuardian[msg.sender], "Not guardian"
        
        voteKey: bytes32 = keccak256(
            _abi_encode(convert("enableEmergencyWithdraw", Bytes[32]))
        )
        self._recordGuardianVote(voteKey)
        
        if not self._hasEnoughVotes(voteKey):
            return
    
    self.emergencyWithdrawEnabled = True
    
    log EmergencyWithdrawEnabled(msg.sender, block.timestamp)

@external
def emergencyWithdraw(token: address):
    """
    User ถอน funds ทั้งหมดใน emergency
    """
    assert self.emergencyWithdrawEnabled, "Emergency withdraw not enabled"
    assert self.isSupported[token], "Token not supported"
    
    amount: uint256 = self.userBalances[msg.sender][token]
    assert amount > 0, "No balance"
    
    self.userBalances[msg.sender][token] = 0
    ERC20(token).transfer(msg.sender, amount)
    
    log UserEmergencyWithdraw(msg.sender, token, amount)

@external
def seizeAndDistribute(
    token: address,
    destination: address
):
    """
    ยึด funds และส่งไปยัง safe address (owner only)
    ใช้เมื่อต้องการ recover funds เร่งด่วน
    """
    assert msg.sender == self.owner, "Not owner"
    assert self.emergencyLevel >= 3, "Not critical emergency"
    
    amount: uint256 = ERC20(token).balanceOf(self)
    assert amount > 0, "No funds"
    
    ERC20(token).transfer(destination, amount)
    
    log FundsSeized(token, amount, destination, msg.sender)

@external
def resolveEmergency(resolution: String[256]):
    """
    ยุติ emergency
    """
    assert msg.sender == self.owner, "Not owner"
    assert self.emergencyLevel > 0, "No emergency"
    
    self.emergencyLevel = 0
    self.emergencyTimestamp = 0
    self.paused = False
    # emergencyWithdrawEnabled ยังคงเปิดไว้เพื่อให้ users มีเวลา withdraw
    
    log EmergencyResolved(msg.sender, resolution, block.timestamp)

# ===== GUARDIAN MANAGEMENT =====

@internal
def _recordGuardianVote(voteKey: bytes32):
    """บันทึก guardian vote"""
    voters: DynArray[address, MAX_GUARDIANS] = self.guardianVotes[voteKey]
    
    for voter in voters:
        if voter == msg.sender:
            return  # Already voted
    
    self.guardianVotes[voteKey].append(msg.sender)

@internal
@view
def _hasEnoughVotes(voteKey: bytes32) -> bool:
    """ตรวจสอบว่ามี votes เพียงพอ"""
    return len(self.guardianVotes[voteKey]) >= GUARDIAN_THRESHOLD

@external
def addGuardian(guardian: address):
    """เพิ่ม guardian (owner only)"""
    assert msg.sender == self.owner, "Not owner"
    assert not self.isGuardian[guardian], "Already guardian"
    assert len(self.guardians) < MAX_GUARDIANS, "Too many guardians"
    assert guardian != empty(address), "Invalid guardian"
    
    self.guardians.append(guardian)
    self.isGuardian[guardian] = True
    
    log GuardianAdded(guardian, msg.sender)

@external
def removeGuardian(guardian: address):
    """ลบ guardian"""
    assert msg.sender == self.owner, "Not owner"
    assert self.isGuardian[guardian], "Not a guardian"
    assert len(self.guardians) > GUARDIAN_THRESHOLD, "Would lose quorum"
    
    self.isGuardian[guardian] = False
    
    # Remove from array
    newGuardians: DynArray[address, MAX_GUARDIANS] = []
    for g in self.guardians:
        if g != guardian:
            newGuardians.append(g)
    self.guardians = newGuardians
    
    log GuardianRemoved(guardian, msg.sender)

# ===== TIMELOCK =====

@external
def proposeAction(actionId: bytes32, data: Bytes[1024]):
    """
    Propose action ที่ต้องรอ timelock
    """
    assert msg.sender == self.owner, "Not owner"
    assert self.timelockActions[actionId].proposedAt == 0, "Already proposed"
    
    delay: uint256 = NORMAL_TIMELOCK
    if self.emergencyLevel >= 3:
        delay = EMERGENCY_TIMELOCK
    
    self.timelockActions[actionId] = TimelockAction({
        proposedAt: block.timestamp,
        executeAfter: block.timestamp + delay,
        executed: False,
        proposedBy: msg.sender,
        data: data
    })
    
    log TimelockSet(actionId, block.timestamp + delay, msg.sender)

@external
def executeAction(actionId: bytes32):
    """
    Execute action หลังจาก timelock
    """
    action: TimelockAction = self.timelockActions[actionId]
    
    assert action.proposedAt > 0, "Not proposed"
    assert not action.executed, "Already executed"
    assert block.timestamp >= action.executeAfter, "Too early"
    
    self.timelockActions[actionId].executed = True
    
    # Execute the action
    raw_call(
        self,
        action.data,
        is_delegate_call=True
    )
    
    log TimelockExecuted(actionId, msg.sender)

# ===== USER BALANCE TRACKING =====

@external
def deposit(token: address, amount: uint256):
    """ฝาก tokens (tracking สำหรับ emergency withdraw)"""
    assert not self.paused, "Paused"
    assert self.isSupported[token], "Token not supported"
    assert amount > 0, "Invalid amount"
    
    ERC20(token).transferFrom(msg.sender, self, amount)
    self.userBalances[msg.sender][token] += amount

@external
def withdraw(token: address, amount: uint256):
    """ถอน tokens"""
    assert not self.paused, "Paused"
    assert self.userBalances[msg.sender][token] >= amount, "Insufficient balance"
    
    self.userBalances[msg.sender][token] -= amount
    ERC20(token).transfer(msg.sender, amount)

@external
def addSupportedToken(token: address):
    """เพิ่ม supported token"""
    assert msg.sender == self.owner, "Not owner"
    assert not self.isSupported[token], "Already supported"
    
    self.supportedTokens.append(token)
    self.isSupported[token] = True

# ===== VIEW FUNCTIONS =====

@view
@external
def getGuardians() -> DynArray[address, MAX_GUARDIANS]:
    return self.guardians

@view
@external
def getVoteCount(voteKey: bytes32) -> uint256:
    return len(self.guardianVotes[voteKey])

@view
@external
def getUserBalance(user: address, token: address) -> uint256:
    return self.userBalances[user][token]

@view
@external
def isEmergency() -> bool:
    return self.emergencyLevel > 0
```

---

## 3. Fund Recovery Script

```python
# scripts/emergency_recovery.py
# Script สำหรับ fund recovery ใน emergency

import json
import asyncio
from web3 import Web3
from eth_account import Account
import os

# Configuration
RPC_URL = os.environ.get("RPC_URL")
CONTRACT_ADDRESS = os.environ.get("CONTRACT_ADDRESS")
SAFE_ADDRESS = os.environ.get("SAFE_ADDRESS")  # Multi-sig safe
PRIVATE_KEY = os.environ.get("EMERGENCY_PRIVATE_KEY")

class EmergencyRecovery:
    def __init__(self):
        self.w3 = Web3(Web3.HTTPProvider(RPC_URL))
        self.account = Account.from_key(PRIVATE_KEY)
        
        with open("artifacts/EmergencyProtocol.json") as f:
            abi = json.load(f)["abi"]
        
        self.contract = self.w3.eth.contract(
            address=CONTRACT_ADDRESS,
            abi=abi
        )
    
    def declare_emergency(self, level: int, reason: str):
        """ประกาศ emergency"""
        print(f"Declaring emergency level {level}: {reason}")
        
        tx = self.contract.functions.declareEmergency(
            level,
            reason
        ).build_transaction({
            'from': self.account.address,
            'nonce': self.w3.eth.get_transaction_count(self.account.address),
            'gas': 200000,
            'gasPrice': self.w3.eth.gas_price * 3  # 3x gas price เพื่อความเร็ว
        })
        
        signed = self.w3.eth.account.sign_transaction(tx, PRIVATE_KEY)
        tx_hash = self.w3.eth.send_raw_transaction(signed.rawTransaction)
        receipt = self.w3.eth.wait_for_transaction_receipt(tx_hash)
        
        print(f"Emergency declared: {tx_hash.hex()}")
        return receipt
    
    def enable_emergency_withdraw(self):
        """เปิด emergency withdraw"""
        print("Enabling emergency withdraw...")
        
        tx = self.contract.functions.enableEmergencyWithdraw().build_transaction({
            'from': self.account.address,
            'nonce': self.w3.eth.get_transaction_count(self.account.address),
            'gas': 100000,
            'gasPrice': self.w3.eth.gas_price * 3
        })
        
        signed = self.w3.eth.account.sign_transaction(tx, PRIVATE_KEY)
        tx_hash = self.w3.eth.send_raw_transaction(signed.rawTransaction)
        receipt = self.w3.eth.wait_for_transaction_receipt(tx_hash)
        
        print(f"Emergency withdraw enabled: {tx_hash.hex()}")
        return receipt
    
    def seize_funds(self, token_address: str, destination: str):
        """ยึด funds ไปยัง safe address"""
        balance = self.w3.eth.contract(
            address=token_address,
            abi=[{"inputs":[{"type":"address"}],"name":"balanceOf","outputs":[{"type":"uint256"}],"type":"function"}]
        ).functions.balanceOf(CONTRACT_ADDRESS).call()
        
        if balance == 0:
            print(f"No balance for token {token_address}")
            return
        
        print(f"Seizing {balance/10**18:.2f} tokens from {token_address}")
        
        tx = self.contract.functions.seizeAndDistribute(
            token_address,
            destination
        ).build_transaction({
            'from': self.account.address,
            'nonce': self.w3.eth.get_transaction_count(self.account.address),
            'gas': 200000,
            'gasPrice': self.w3.eth.gas_price * 3
        })
        
        signed = self.w3.eth.account.sign_transaction(tx, PRIVATE_KEY)
        tx_hash = self.w3.eth.send_raw_transaction(signed.rawTransaction)
        receipt = self.w3.eth.wait_for_transaction_receipt(tx_hash)
        
        print(f"Funds seized: {tx_hash.hex()}")
        return receipt
    
    def full_emergency_procedure(self, tokens: list, reason: str):
        """
        Full emergency procedure:
        1. Declare emergency level 4
        2. Enable emergency withdraw
        3. Seize remaining funds to safe
        """
        print("=== STARTING EMERGENCY PROCEDURE ===")
        
        # Step 1: Declare emergency
        self.declare_emergency(4, reason)
        print("Step 1: Emergency declared ✓")
        
        # Step 2: Enable emergency withdraw
        self.enable_emergency_withdraw()
        print("Step 2: Emergency withdraw enabled ✓")
        
        # Step 3: Notify users (send to monitoring)
        print("Step 3: Notifying users... (manual action required)")
        print(f"Post on Twitter/Discord: 'Emergency! Protocol paused. Use emergencyWithdraw() to retrieve funds.'")
        
        # Wait for users to withdraw (at least 30 minutes)
        print("Waiting 30 minutes for users to withdraw...")
        # In production: asyncio.sleep(1800)
        
        # Step 4: Seize remaining funds
        for token in tokens:
            self.seize_funds(token, SAFE_ADDRESS)
        print("Step 4: Remaining funds seized ✓")
        
        print("=== EMERGENCY PROCEDURE COMPLETE ===")

# ตัวอย่างการใช้งาน
if __name__ == "__main__":
    recovery = EmergencyRecovery()
    
    # Declare emergency
    recovery.declare_emergency(
        4,
        "Critical vulnerability detected. Active exploit in progress. Contract paused for user protection."
    )
```

---

## 4. Post-incident Analysis Template

```markdown
# Incident Post-Mortem Report

## Summary
- **Date:** 2024-01-15
- **Severity:** Level 4 - Critical
- **Duration:** 2 hours 15 minutes
- **Funds at Risk:** $2.5M USDC
- **Funds Lost:** $150K USDC
- **Root Cause:** Reentrancy vulnerability in withdraw function

## Timeline

| Time (UTC) | Event |
|------------|-------|
| 14:00 | First suspicious transaction detected |
| 14:05 | Monitoring alert fired |
| 14:12 | Team notified via PagerDuty |
| 14:18 | Team assembled on war room call |
| 14:25 | Root cause identified |
| 14:30 | Emergency pause executed |
| 14:45 | Emergency withdraw enabled |
| 15:30 | Remaining funds seized to safe |
| 16:15 | Public communication posted |

## Root Cause Analysis

The vulnerability was a classic reentrancy attack in the `withdraw` function.
The function transferred ETH before updating the user's balance...

## Impact Analysis

- Total funds drained: $150K
- Users affected: 47
- Protocol TVL impact: -6%

## Remediation

### Immediate Actions
1. Pause contract
2. Enable emergency withdraw
3. Deploy patch

### Long-term Fixes
1. Add reentrancy guard to all state-changing functions
2. Follow checks-effects-interactions pattern
3. Increase test coverage
4. Add formal verification

## Lessons Learned

1. Always audit before mainnet deployment
2. Emergency procedures should be tested regularly
3. Monitoring should have shorter alert latency
4. Post-launch bug bounty is essential

## Communication Log

- 14:30: Twitter post about pause
- 14:45: Discord announcement
- 15:00: Email to major LPs
- 16:15: Full post-mortem published
```

---

## 5. Whitehack Communication Template

```python
# scripts/communication.py
# Templates สำหรับ communication ระหว่าง incident

TWITTER_PAUSE_TEMPLATE = """
🚨 IMPORTANT UPDATE

We have temporarily paused Protocol operations due to a potential security concern.

User funds are safe. Emergency withdrawal is available at:
{contract_address}

Call emergencyWithdraw({token}) to retrieve your funds.

More updates to follow. We apologize for the inconvenience.
"""

DISCORD_DETAILED_UPDATE = """
**⚠️ Protocol Security Update**

We have identified a potential vulnerability and have taken immediate action to protect user funds.

**Current Status:**
• Protocol paused: ✅
• User funds safe: ✅
• Emergency withdraw available: ✅

**How to Withdraw Your Funds:**
1. Go to Etherscan: {etherscan_link}
2. Connect your wallet
3. Call `emergencyWithdraw({token})` for each token you have deposited

**What Happened:**
[Brief description without revealing exploit details]

**What We're Doing:**
- Investigating root cause
- Working with security experts
- Preparing fix

**Timeline:**
- Emergency withdrawal will remain open for 7 days
- We will update you as more information becomes available

For support, open a ticket in #support

We sincerely apologize for this disruption.
"""

COMPENSATION_ANNOUNCEMENT = """
**Protocol Compensation Plan**

We are committed to making affected users whole.

**Affected Users:** {count} addresses
**Total Amount:** ${amount}

**Compensation Details:**
- Users will receive 100% of lost funds
- Compensation paid in USDC
- Distribution within 14 days

**How to Claim:**
Visit {compensation_url} and connect your wallet.

We take responsibility for this incident and are committed to learning from it.
"""

def post_twitter_update(message: str):
    """Post to Twitter (production would use Tweepy)"""
    print(f"[TWITTER] {message}")

def post_discord_update(webhook_url: str, message: str):
    """Post to Discord"""
    import requests
    
    data = {"content": message}
    response = requests.post(webhook_url, json=data)
    print(f"Discord update posted: {response.status_code}")
```

---

## 6. Automated Incident Response

```vyper
# @version 0.4.0
# @title Automated Circuit Breaker
# @notice Contract ที่ pause ตัวเองเมื่อตรวจพบ anomaly

from vyper.interfaces import ERC20

event CircuitBreakerTripped:
    trigger: indexed(String[64])
    value: uint256
    timestamp: uint256

event CircuitBreakerReset:
    by: indexed(address)

# State
paused: public(bool)
owner: public(address)

# Circuit breaker thresholds
maxDailyWithdraw: public(uint256)   # Max ที่ withdraw ได้ต่อวัน
dailyWithdrawn: public(uint256)      # ที่ withdraw ไปแล้วในวันนี้
lastResetDay: public(uint256)        # วันที่ reset ล่าสุด

maxSingleWithdraw: public(uint256)   # Max ต่อ transaction เดียว
maxPriceDeviation: public(uint256)   # Max price deviation ที่ยอมรับ (basis points)

lastOraclePrice: public(uint256)     # ราคาล่าสุดจาก oracle
oracle: public(address)

@deploy
def __init__(
    _maxDailyWithdraw: uint256,
    _maxSingleWithdraw: uint256,
    _maxPriceDeviation: uint256,
    _oracle: address
):
    self.owner = msg.sender
    self.maxDailyWithdraw = _maxDailyWithdraw
    self.maxSingleWithdraw = _maxSingleWithdraw
    self.maxPriceDeviation = _maxPriceDeviation
    self.oracle = _oracle
    self.lastResetDay = block.timestamp / 86400

@internal
def _resetDailyCounterIfNeeded():
    """Reset counter เมื่อขึ้นวันใหม่"""
    currentDay: uint256 = block.timestamp / 86400
    if currentDay > self.lastResetDay:
        self.dailyWithdrawn = 0
        self.lastResetDay = currentDay

@internal
def _checkAndTripCircuitBreaker(amount: uint256) -> bool:
    """
    ตรวจสอบ circuit breaker conditions
    Return True ถ้า breaker trip (และ pause)
    """
    self._resetDailyCounterIfNeeded()
    
    # Check 1: Single transaction limit
    if amount > self.maxSingleWithdraw:
        self.paused = True
        log CircuitBreakerTripped(
            "single_tx_limit",
            amount,
            block.timestamp
        )
        return True
    
    # Check 2: Daily limit
    if self.dailyWithdrawn + amount > self.maxDailyWithdraw:
        self.paused = True
        log CircuitBreakerTripped(
            "daily_limit",
            self.dailyWithdrawn + amount,
            block.timestamp
        )
        return True
    
    return False

@external
def withdraw(token: address, amount: uint256):
    """ถอน tokens พร้อม circuit breaker"""
    assert not self.paused, "Circuit breaker tripped"
    assert amount > 0, "Invalid amount"
    
    # Check circuit breaker
    if self._checkAndTripCircuitBreaker(amount):
        raise "Circuit breaker tripped - withdrawal paused"
    
    self.dailyWithdrawn += amount
    
    ERC20(token).transfer(msg.sender, amount)

@external
def resetCircuitBreaker():
    """Reset circuit breaker (owner only หลัง investigation)"""
    assert msg.sender == self.owner, "Not owner"
    
    self.paused = False
    self.dailyWithdrawn = 0
    
    log CircuitBreakerReset(msg.sender)

@external
def updateThresholds(
    _maxDaily: uint256,
    _maxSingle: uint256
):
    assert msg.sender == self.owner, "Not owner"
    self.maxDailyWithdraw = _maxDaily
    self.maxSingleWithdraw = _maxSingle
```

---

## 7. สรุป Incident Response

### Emergency Response Checklist

```
T+0 (Alert fires):
□ Acknowledge alert
□ Assess severity
□ Notify team lead

T+5 minutes:
□ Convene war room call
□ Begin root cause investigation
□ Monitor exploit in real-time

T+15 minutes:
□ If critical: execute pause
□ Post brief Twitter/Discord notice
□ Contact security firm (if needed)

T+30 minutes:
□ Enable emergency withdraw (if needed)
□ Identify full scope of impact
□ Prepare detailed communication

T+1 hour:
□ Post detailed update
□ Contact major users directly
□ Begin patch development

T+24 hours:
□ Publish full post-mortem
□ Announce compensation plan
□ Submit fix for audit

T+7 days:
□ Audit completed
□ Deploy fix
□ Resume operations
□ Complete compensation distribution
```

---

## แบบฝึกหัด

1. Implement multi-sig guardian system สำหรับ EmergencyProtocol
2. เขียน automated monitor ที่ trip circuit breaker โดยอัตโนมัติ
3. สร้าง fund recovery script ที่รองรับ multiple tokens
4. เขียน post-mortem template สำหรับ flash loan attack
5. Implement timelock ที่มี guardian veto power

---

*จบ Part 084: Incident Response สำหรับ Smart Contracts*
