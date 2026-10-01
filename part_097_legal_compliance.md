# Part 097: Legal and Compliance Considerations

## สารบัญ
1. [Regulatory Landscape สำหรับ DeFi](#regulatory)
2. [KYC/AML Considerations](#kyc-aml)
3. [Compliance by Design](#compliance-design)
4. [DAO Legal Structures](#dao-legal)
5. [Securities Law Basics](#securities-law)
6. [Compliance Contract Patterns](#compliance-contracts)

---

> **Disclaimer**: เนื้อหาในบทนี้เป็นข้อมูลทั่วไปเพื่อการศึกษาเท่านั้น ไม่ใช่คำปรึกษาทางกฎหมาย สำหรับการตัดสินใจทางธุรกิจจริงควรปรึกษาทนายความที่มีความเชี่ยวชาญด้าน crypto และ DeFi

---

## 1. Regulatory Landscape {#regulatory}

### ภูมิทัศน์กฎระเบียบ DeFi ทั่วโลก

**สหรัฐอเมริกา (USA)**
- SEC (Securities and Exchange Commission) - กำกับ securities
- CFTC (Commodity Futures Trading Commission) - กำกับ derivatives
- FinCEN - กำกับ AML/KYC
- IRS - กำกับภาษี (crypto = property)

**สหภาพยุโรป (EU)**
- MiCA (Markets in Crypto-Assets Regulation) - กฎระเบียบ crypto ฉบับใหม่
- AMLD6 - Anti-Money Laundering Directive
- GDPR - ความเป็นส่วนตัวของข้อมูล

**เอเชีย**
- ญี่ปุ่น: FSA regulated, exchange licensing
- สิงคโปร์: MAS - Payment Services Act
- ฮ่องกง: SFC licensing regime
- ไทย: SEC กำกับ digital assets

### Howey Test - เมื่อไหร่ Token ถือเป็น Security?

Howey Test ใช้ตัดสินว่า instrument ใดคือ "investment contract" (security):
1. **Investment of Money** - มีการลงทุนเงิน
2. **Common Enterprise** - ใน enterprise ร่วม
3. **Expectation of Profits** - คาดหวัง profit
4. **From Efforts of Others** - จากความพยายามของคนอื่น

ถ้า token ผ่าน Howey Test ทั้ง 4 ข้อ = อาจถือเป็น Security

---

## 2. KYC/AML Considerations {#kyc-aml}

### On-chain Compliance Contract

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0

"""
@title ComplianceModule - โมดูล compliance สำหรับ DeFi protocol
@notice จัดการ KYC/AML requirements โดยไม่เก็บ sensitive data on-chain
@dev ใช้ zero-knowledge proofs หรือ attestations จาก trusted providers
"""

# ============================================================
# Events
# ============================================================

event UserVerified:
    user: indexed(address)
    verificationLevel: uint8
    verifiedBy: indexed(address)
    expiry: uint256
    timestamp: uint256

event UserRevoked:
    user: indexed(address)
    reason: String[100]
    revokedBy: indexed(address)
    timestamp: uint256

event CountryBlocked:
    countryCode: String[3]
    reason: String[100]
    timestamp: uint256

event TransactionFlagged:
    user: indexed(address)
    txHash: bytes32
    reason: String[100]
    timestamp: uint256

event VerifierAdded:
    verifier: indexed(address)
    verifierName: String[50]

event ComplianceCheckPassed:
    user: indexed(address)
    checkType: String[30]
    timestamp: uint256

# ============================================================
# Structs
# ============================================================

struct UserCompliance:
    verificationLevel: uint8     # 0=none, 1=basic, 2=enhanced, 3=accredited
    verifiedBy: address          # KYC provider address
    verifiedAt: uint256
    expiresAt: uint256
    countryCode: String[3]       # ISO 3166-1 alpha-3
    isBlacklisted: bool
    blacklistReason: String[100]
    blacklistedAt: uint256
    totalTransactions: uint256
    highValueTxCount: uint256

struct VerifierInfo:
    name: String[50]
    address_: address
    isActive: bool
    verificationCount: uint256
    addedAt: uint256

struct ComplianceConfig:
    requireKYC: bool             # ต้องมี KYC หรือไม่
    minVerificationLevel: uint8  # ระดับ KYC ขั้นต่ำ
    maxTransactionLimit: uint256 # ขีดจำกัดต่อ transaction (0 = no limit)
    dailyTransactionLimit: uint256
    highValueThreshold: uint256  # กำหนด high value transaction
    verificationExpiry: uint256  # อายุ verification (seconds)

# ============================================================
# Constants
# ============================================================

MAX_VERIFIERS: constant(uint256) = 20
VERIFICATION_BASIC: constant(uint8) = 1
VERIFICATION_ENHANCED: constant(uint8) = 2
VERIFICATION_ACCREDITED: constant(uint8) = 3

# ============================================================
# State Variables
# ============================================================

owner: public(address)
governance: public(address)

# User compliance data
userCompliance: public(HashMap[address, UserCompliance])

# Verifiers (KYC providers)
verifiers: public(DynArray[address, MAX_VERIFIERS])
verifierInfo: public(HashMap[address, VerifierInfo])

# Blocked countries
blockedCountries: public(HashMap[String[3], bool])
blockedCountryReasons: public(HashMap[String[3], String[100]])

# Configuration
config: public(ComplianceConfig)

# Statistics
totalVerifiedUsers: public(uint256)
totalBlacklistedUsers: public(uint256)
totalFlaggedTransactions: public(uint256)

# ============================================================
# Constructor
# ============================================================

@deploy
def __init__(_governance: address):
    """
    @notice Initialize compliance module
    @param _governance Governance contract for updates
    """
    assert _governance != empty(address), "Invalid governance"
    
    self.owner = msg.sender
    self.governance = _governance
    
    # Default configuration
    self.config = ComplianceConfig({
        requireKYC: False,           # เริ่มต้นไม่บังคับ
        minVerificationLevel: 1,
        maxTransactionLimit: 0,      # ไม่จำกัด
        dailyTransactionLimit: 0,    # ไม่จำกัด
        highValueThreshold: 10000 * 10**18,  # $10k
        verificationExpiry: 365 * 24 * 3600  # 1 year
    })

# ============================================================
# Verification Functions
# ============================================================

@external
def verifyUser(
    _user: address,
    _level: uint8,
    _countryCode: String[3],
    _expiryOverride: uint256
):
    """
    @notice Verify user - สามารถเรียกได้เฉพาะ approved verifiers
    @param _user User address
    @param _level Verification level (1=basic, 2=enhanced, 3=accredited)
    @param _countryCode ISO 3166-1 alpha-3 country code
    @param _expiryOverride Custom expiry (0 = use default)
    """
    assert self.verifierInfo[msg.sender].isActive, "Not active verifier"
    assert _level >= 1 and _level <= 3, "Invalid level"
    assert _user != empty(address), "Invalid user"
    
    # ตรวจสอบว่าประเทศไม่ถูก block
    assert not self.blockedCountries[_countryCode], "Country blocked"
    
    expiresAt: uint256 = block.timestamp + self.config.verificationExpiry
    if _expiryOverride > block.timestamp:
        expiresAt = _expiryOverride
    
    isNewUser: bool = self.userCompliance[_user].verificationLevel == 0
    
    self.userCompliance[_user] = UserCompliance({
        verificationLevel: _level,
        verifiedBy: msg.sender,
        verifiedAt: block.timestamp,
        expiresAt: expiresAt,
        countryCode: _countryCode,
        isBlacklisted: False,
        blacklistReason: "",
        blacklistedAt: 0,
        totalTransactions: self.userCompliance[_user].totalTransactions,
        highValueTxCount: self.userCompliance[_user].highValueTxCount
    })
    
    if isNewUser:
        self.totalVerifiedUsers += 1
    
    self.verifierInfo[msg.sender].verificationCount += 1
    
    log UserVerified(_user, _level, msg.sender, expiresAt, block.timestamp)

@external
def revokeVerification(_user: address, _reason: String[100]):
    """
    @notice ยกเลิก verification ของ user
    """
    assert (
        self.verifierInfo[msg.sender].isActive or 
        msg.sender == self.owner or 
        msg.sender == self.governance
    ), "Not authorized"
    
    self.userCompliance[_user].verificationLevel = 0
    
    log UserRevoked(_user, _reason, msg.sender, block.timestamp)

@external
def blacklistUser(_user: address, _reason: String[100]):
    """
    @notice Blacklist user (AML action)
    """
    assert msg.sender == self.owner or msg.sender == self.governance, "Not authorized"
    
    self.userCompliance[_user].isBlacklisted = True
    self.userCompliance[_user].blacklistReason = _reason
    self.userCompliance[_user].blacklistedAt = block.timestamp
    
    self.totalBlacklistedUsers += 1
    
    log UserRevoked(_user, _reason, msg.sender, block.timestamp)

# ============================================================
# Compliance Checks
# ============================================================

@external
@view
def checkCompliance(_user: address, _amount: uint256) -> (bool, String[100]):
    """
    @notice ตรวจสอบ compliance ก่อน transaction
    @return (isCompliant, reason)
    """
    compliance: UserCompliance = self.userCompliance[_user]
    
    # Check blacklist
    if compliance.isBlacklisted:
        return False, "User is blacklisted"
    
    # Check KYC requirement
    if self.config.requireKYC:
        if compliance.verificationLevel < self.config.minVerificationLevel:
            return False, "Insufficient verification level"
        
        if compliance.expiresAt < block.timestamp:
            return False, "Verification expired"
    
    # Check transaction limit
    if self.config.maxTransactionLimit > 0:
        if _amount > self.config.maxTransactionLimit:
            return False, "Exceeds transaction limit"
    
    return True, "Compliant"

@external
def isUserVerified(_user: address) -> bool:
    """ตรวจสอบว่า user ผ่าน KYC แล้วหรือไม่"""
    compliance: UserCompliance = self.userCompliance[_user]
    
    if compliance.isBlacklisted:
        return False
    
    if compliance.verificationLevel < self.config.minVerificationLevel:
        return False
    
    if compliance.expiresAt < block.timestamp:
        return False
    
    return True

@external
def flagTransaction(_user: address, _txHash: bytes32, _reason: String[100]):
    """
    @notice Flag suspicious transaction สำหรับ review
    """
    assert (
        self.verifierInfo[msg.sender].isActive or 
        msg.sender == self.owner
    ), "Not authorized"
    
    self.totalFlaggedTransactions += 1
    
    log TransactionFlagged(_user, _txHash, _reason, block.timestamp)

# ============================================================
# Country Management
# ============================================================

@external
def blockCountry(_countryCode: String[3], _reason: String[100]):
    """บล็อก country"""
    assert msg.sender == self.owner or msg.sender == self.governance, "Not authorized"
    
    self.blockedCountries[_countryCode] = True
    self.blockedCountryReasons[_countryCode] = _reason
    
    log CountryBlocked(_countryCode, _reason, block.timestamp)

@external
def unblockCountry(_countryCode: String[3]):
    """ยกเลิกการบล็อก country"""
    assert msg.sender == self.owner or msg.sender == self.governance, "Not authorized"
    
    self.blockedCountries[_countryCode] = False

# ============================================================
# Verifier Management
# ============================================================

@external
def addVerifier(_verifier: address, _name: String[50]):
    """เพิ่ม KYC verifier"""
    assert msg.sender == self.owner or msg.sender == self.governance, "Not authorized"
    assert len(self.verifiers) < MAX_VERIFIERS, "Too many verifiers"
    
    self.verifiers.append(_verifier)
    self.verifierInfo[_verifier] = VerifierInfo({
        name: _name,
        address_: _verifier,
        isActive: True,
        verificationCount: 0,
        addedAt: block.timestamp
    })
    
    log VerifierAdded(_verifier, _name)

@external
def deactivateVerifier(_verifier: address):
    """ปิดการใช้งาน verifier"""
    assert msg.sender == self.owner or msg.sender == self.governance, "Not authorized"
    self.verifierInfo[_verifier].isActive = False

# ============================================================
# Configuration
# ============================================================

@external
def updateConfig(
    _requireKYC: bool,
    _minLevel: uint8,
    _maxTxLimit: uint256,
    _dailyLimit: uint256,
    _highValueThreshold: uint256
):
    """อัปเดต compliance configuration"""
    assert msg.sender == self.governance or msg.sender == self.owner, "Not authorized"
    
    self.config = ComplianceConfig({
        requireKYC: _requireKYC,
        minVerificationLevel: _minLevel,
        maxTransactionLimit: _maxTxLimit,
        dailyTransactionLimit: _dailyLimit,
        highValueThreshold: _highValueThreshold,
        verificationExpiry: self.config.verificationExpiry
    })

# ============================================================
# View Functions
# ============================================================

@external
@view
def getUserCompliance(_user: address) -> UserCompliance:
    """ดู compliance status ของ user"""
    return self.userCompliance[_user]

@external
@view
def getComplianceStats() -> (uint256, uint256, uint256):
    """ดูสถิติ compliance"""
    return (
        self.totalVerifiedUsers,
        self.totalBlacklistedUsers,
        self.totalFlaggedTransactions
    )
```

---

## 3. DAO Legal Structures {#dao-legal}

### ประเภท Legal Wrapper สำหรับ DAO

#### 1. Wyoming DAO LLC (สหรัฐ)
- DAO ที่จดทะเบียนใน Wyoming ได้รับการรับรองทางกฎหมาย
- Members = token holders
- ข้อดี: limited liability, legal recognition
- ข้อเสีย: ต้องมี registered agent ใน Wyoming

#### 2. Marshall Islands DAO (Pacific)
- Non-profit DAO structure
- Token holders = members
- ใช้กันมาก: Maker DAO, etc.

#### 3. Cayman Islands Foundation
- Foundation Company structure
- ออกแบบมาเพื่อ decentralized organizations
- ใช้โดย: many DeFi protocols

#### 4. Swiss Association (Verein)
- ไม่ต้องมี profit motive
- Members มี voting rights
- ใช้โดย: Ethereum Foundation

### On-chain DAO Registry

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0

"""
@title DAOLegalRegistry - บันทึก legal information ของ DAO on-chain
@notice Transparency สำหรับ legal structure ของ DAO
"""

# Events
event LegalInfoUpdated:
    dao: indexed(address)
    jurisdiction: String[50]
    entityType: String[50]
    timestamp: uint256

event TermsUpdated:
    version: uint256
    contentHash: bytes32
    timestamp: uint256

event DisputeFiledOnChain:
    disputeId: indexed(uint256)
    filer: indexed(address)
    description: String[200]
    timestamp: uint256

# Structs
struct LegalInfo:
    entityName: String[100]
    entityType: String[50]        # "LLC", "Foundation", "Association", etc.
    jurisdiction: String[50]
    registrationNumber: String[50]
    registrationDate: uint256
    legalContact: String[100]     # email or postal address (not stored for privacy)
    legalDocumentsIPFS: String[100]  # IPFS hash of legal documents
    lastUpdated: uint256

struct TermsOfService:
    version: uint256
    contentHash: bytes32          # keccak256 of ToS document
    ipfsHash: String[100]         # IPFS link to full document
    effectiveDate: uint256
    createdAt: uint256

struct DisputeRecord:
    id: uint256
    filer: address
    description: String[200]
    filedAt: uint256
    resolved: bool
    resolution: String[200]

# State
owner: public(address)
governance: public(address)

legalInfo: public(LegalInfo)
currentToSVersion: public(uint256)
termsHistory: public(HashMap[uint256, TermsOfService])

disputes: public(HashMap[uint256, DisputeRecord])
disputeCount: public(uint256)

# User ToS acceptance
userAcceptedToS: public(HashMap[address, uint256])  # address -> ToS version
userAcceptedAt: public(HashMap[address, HashMap[uint256, uint256]])  # address -> version -> timestamp

@deploy
def __init__(_governance: address):
    self.owner = msg.sender
    self.governance = _governance

@external
def updateLegalInfo(
    _entityName: String[100],
    _entityType: String[50],
    _jurisdiction: String[50],
    _regNumber: String[50],
    _regDate: uint256,
    _ipfsHash: String[100]
):
    """อัปเดต legal information"""
    assert msg.sender == self.governance or msg.sender == self.owner, "Not authorized"
    
    self.legalInfo = LegalInfo({
        entityName: _entityName,
        entityType: _entityType,
        jurisdiction: _jurisdiction,
        registrationNumber: _regNumber,
        registrationDate: _regDate,
        legalContact: "",
        legalDocumentsIPFS: _ipfsHash,
        lastUpdated: block.timestamp
    })
    
    log LegalInfoUpdated(self, _jurisdiction, _entityType, block.timestamp)

@external
def publishTermsOfService(_contentHash: bytes32, _ipfsHash: String[100]):
    """เผยแพร่ Terms of Service ใหม่"""
    assert msg.sender == self.governance or msg.sender == self.owner, "Not authorized"
    
    newVersion: uint256 = self.currentToSVersion + 1
    self.currentToSVersion = newVersion
    
    self.termsHistory[newVersion] = TermsOfService({
        version: newVersion,
        contentHash: _contentHash,
        ipfsHash: _ipfsHash,
        effectiveDate: block.timestamp + 14 * 24 * 3600,  # 14 day notice
        createdAt: block.timestamp
    })
    
    log TermsUpdated(newVersion, _contentHash, block.timestamp)

@external
def acceptTermsOfService():
    """ยอมรับ Terms of Service ปัจจุบัน"""
    version: uint256 = self.currentToSVersion
    assert version > 0, "No ToS published"
    
    tos: TermsOfService = self.termsHistory[version]
    assert block.timestamp >= tos.effectiveDate, "ToS not yet effective"
    
    self.userAcceptedToS[msg.sender] = version
    self.userAcceptedAt[msg.sender][version] = block.timestamp

@external
@view
def hasAcceptedCurrentToS(_user: address) -> bool:
    """ตรวจสอบว่า user ยอมรับ ToS ปัจจุบันแล้วหรือไม่"""
    return self.userAcceptedToS[_user] >= self.currentToSVersion

@external
def fileDispute(_description: String[200]):
    """ยื่น dispute on-chain"""
    disputeId: uint256 = self.disputeCount
    self.disputeCount += 1
    
    self.disputes[disputeId] = DisputeRecord({
        id: disputeId,
        filer: msg.sender,
        description: _description,
        filedAt: block.timestamp,
        resolved: False,
        resolution: ""
    })
    
    log DisputeFiledOnChain(disputeId, msg.sender, _description, block.timestamp)
```

---

## 4. Regulatory-Compliant Token Contract {#compliance-contracts}

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0

"""
@title RegulatoryCompliantToken - Token ที่ออกแบบรองรับ regulations
@notice ผสาน compliance features เข้ากับ ERC20 token
@dev สำหรับ protocols ที่ต้องการ compliance
"""

from vyper.interfaces import ERC20

# Events
event Transfer:
    sender: indexed(address)
    receiver: indexed(address)
    value: uint256

event Approval:
    owner: indexed(address)
    spender: indexed(address)
    value: uint256

event ComplianceTransferBlocked:
    from_: indexed(address)
    to: indexed(address)
    amount: uint256
    reason: String[100]

event RegulatorAdded:
    regulator: indexed(address)

event ForcedTransfer:
    from_: indexed(address)
    to: indexed(address)
    amount: uint256
    reason: String[100]

# Interfaces
interface IComplianceModule:
    def checkCompliance(_user: address, _amount: uint256) -> (bool, String[100]): view
    def isUserVerified(_user: address) -> bool: view

# State
name: public(String[50])
symbol: public(String[10])
decimals: public(uint8)
totalSupply: public(uint256)

balances: HashMap[address, uint256]
allowances: HashMap[address, HashMap[address, uint256]]

owner: public(address)
complianceModule: public(address)
regulators: public(HashMap[address, bool])

# Transfer restrictions
transfersEnabled: public(bool)
requireVerifiedSender: public(bool)
requireVerifiedReceiver: public(bool)

@deploy
def __init__(
    _name: String[50],
    _symbol: String[10],
    _compliance: address
):
    self.name = _name
    self.symbol = _symbol
    self.decimals = 18
    self.owner = msg.sender
    self.complianceModule = _compliance
    self.transfersEnabled = True
    self.requireVerifiedSender = False
    self.requireVerifiedReceiver = False

@internal
@view
def _checkTransferCompliance(_from: address, _to: address, _amount: uint256) -> bool:
    """ตรวจสอบ compliance สำหรับ transfer"""
    if self.complianceModule == empty(address):
        return True
    
    compliance: IComplianceModule = IComplianceModule(self.complianceModule)
    
    if self.requireVerifiedSender:
        if not compliance.isUserVerified(_from):
            return False
    
    if self.requireVerifiedReceiver:
        if not compliance.isUserVerified(_to):
            return False
    
    senderOk: bool = False
    reason: String[100] = ""
    senderOk, reason = compliance.checkCompliance(_from, _amount)
    if not senderOk:
        return False
    
    return True

@external
def transfer(_to: address, _value: uint256) -> bool:
    assert self.transfersEnabled, "Transfers disabled"
    assert _to != empty(address), "Invalid recipient"
    assert self.balances[msg.sender] >= _value, "Insufficient balance"
    
    if not self._checkTransferCompliance(msg.sender, _to, _value):
        log ComplianceTransferBlocked(msg.sender, _to, _value, "Compliance check failed")
        raise "Transfer blocked: compliance"
    
    self.balances[msg.sender] -= _value
    self.balances[_to] += _value
    
    log Transfer(msg.sender, _to, _value)
    return True

@external
def transferFrom(_from: address, _to: address, _value: uint256) -> bool:
    assert self.transfersEnabled, "Transfers disabled"
    assert self.balances[_from] >= _value, "Insufficient balance"
    assert self.allowances[_from][msg.sender] >= _value, "Insufficient allowance"
    
    if not self._checkTransferCompliance(_from, _to, _value):
        log ComplianceTransferBlocked(_from, _to, _value, "Compliance check failed")
        raise "Transfer blocked: compliance"
    
    self.balances[_from] -= _value
    self.allowances[_from][msg.sender] -= _value
    self.balances[_to] += _value
    
    log Transfer(_from, _to, _value)
    return True

@external
def approve(_spender: address, _value: uint256) -> bool:
    self.allowances[msg.sender][_spender] = _value
    log Approval(msg.sender, _spender, _value)
    return True

@external
@view
def balanceOf(_owner: address) -> uint256:
    return self.balances[_owner]

@external
@view
def allowance(_owner: address, _spender: address) -> uint256:
    return self.allowances[_owner][_spender]

@external
def forcedTransfer(
    _from: address,
    _to: address,
    _amount: uint256,
    _reason: String[100]
):
    """
    @notice Forced transfer สำหรับ regulatory compliance
    @dev เฉพาะ regulators - สำหรับ freeze/confiscation orders
    """
    assert self.regulators[msg.sender], "Not regulator"
    assert self.balances[_from] >= _amount, "Insufficient balance"
    
    self.balances[_from] -= _amount
    self.balances[_to] += _amount
    
    log ForcedTransfer(_from, _to, _amount, _reason)
    log Transfer(_from, _to, _amount)

@external
def mint(_to: address, _amount: uint256):
    """Mint tokens"""
    assert msg.sender == self.owner, "Not owner"
    
    self.totalSupply += _amount
    self.balances[_to] += _amount
    
    log Transfer(empty(address), _to, _amount)

@external
def burn(_amount: uint256):
    """Burn tokens"""
    assert self.balances[msg.sender] >= _amount, "Insufficient"
    
    self.balances[msg.sender] -= _amount
    self.totalSupply -= _amount
    
    log Transfer(msg.sender, empty(address), _amount)

@external
def setTransfersEnabled(_enabled: bool):
    """Enable/disable transfers"""
    assert msg.sender == self.owner, "Not owner"
    self.transfersEnabled = _enabled

@external
def addRegulator(_regulator: address):
    """เพิ่ม regulator"""
    assert msg.sender == self.owner, "Not owner"
    self.regulators[_regulator] = True
    log RegulatorAdded(_regulator)

@external
def updateComplianceModule(_newModule: address):
    """อัปเดต compliance module"""
    assert msg.sender == self.owner, "Not owner"
    self.complianceModule = _newModule
```

---

## 5. สรุปข้อควรระวังทางกฎหมาย

### Red Flags ที่ควรหลีกเลี่ยง

```
กิจกรรมที่อาจมีปัญหาทางกฎหมาย:

1. Marketing ที่เน้น "investment returns"
   ❌ "Token จะมีมูลค่าสูงขึ้นแน่นอน"
   ✓ "Token ใช้สำหรับ governance ใน protocol"

2. Token Distribution ที่ไม่ยุติธรรม
   ❌ Team ถือ 80% โดยไม่มี lock-up
   ✓ มี vesting schedule ที่ชัดเจน

3. ขาด Disclosure
   ❌ ไม่บอก risks ให้ผู้ใช้รู้
   ✓ มี comprehensive risk disclosure

4. Marketing ในประเทศที่ banned
   ❌ Target users จาก sanctioned countries
   ✓ Geo-block หรือ Terms of Service ที่ชัดเจน
```

### Legal Checklist สำหรับ DeFi Protocol

```
Pre-Launch Legal Checklist

Entity Structure
[ ] เลือก legal structure (LLC, Foundation, etc.)
[ ] จดทะเบียนในเขตอำนาจที่เหมาะสม
[ ] ตั้ง registered agent

Documentation
[ ] Terms of Service ร่างพร้อม
[ ] Privacy Policy ร่างพร้อม (GDPR compliance)
[ ] Risk Disclosure Document
[ ] Token Sale Agreement (ถ้ามี)

Technical Compliance
[ ] Geo-blocking สำหรับ restricted jurisdictions
[ ] KYC/AML framework (ถ้าจำเป็น)
[ ] Transaction monitoring
[ ] Sanctions screening

IP Protection
[ ] Trademark ชื่อ protocol
[ ] Source code licensing
[ ] Patent considerations (ถ้ามี novel technology)

Regulatory Engagement
[ ] Legal opinion เรื่อง token classification
[ ] Consult ด้วย lawyers ในเขตอำนาจหลัก
[ ] Monitor regulatory developments
```

> **คำเตือน**: กฎหมาย crypto เปลี่ยนแปลงรวดเร็ว สิ่งที่ถูกกฎหมายวันนี้อาจไม่ถูกกฎหมายพรุ่งนี้ ควรมี legal counsel อย่างต่อเนื่อง และ stay updated กับ regulatory developments
