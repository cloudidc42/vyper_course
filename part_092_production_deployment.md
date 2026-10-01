# Part 092: Production Deployment

## สารบัญ
1. [การเตรียมความพร้อมก่อน Mainnet Deployment](#pre-deployment)
2. [Mainnet Deployment Checklist](#mainnet-checklist)
3. [Multi-sig Deployment](#multisig)
4. [Deployment ด้วย Gnosis Safe](#gnosis-safe)
5. [Proxy Deployment Patterns](#proxy)
6. [Contract Verification](#verification)
7. [Post-Deployment Procedures](#post-deployment)

---

## 1. การเตรียมความพร้อมก่อน Mainnet Deployment {#pre-deployment}

การ deploy smart contract ไปยัง Ethereum mainnet เป็นขั้นตอนที่ต้องการความระมัดระวังสูง เพราะ:
- **ไม่สามารถแก้ไขได้** (Immutable) - เมื่อ deploy แล้วต้องใช้ Proxy Pattern เพื่อ upgrade
- **ค่าใช้จ่ายสูง** - Gas cost บน mainnet สูงกว่า testnet หลายเท่า
- **ความปลอดภัยสำคัญสูงสุด** - มี real value ที่เสี่ยงถูก exploit

### Pre-deployment Checklist ฉบับสมบูรณ์

```
## Security Audit
[ ] Internal code review เสร็จสมบูรณ์
[ ] External audit โดย reputable firm (อย่างน้อย 1 audit)
[ ] Bug bounty program เปิดตัวและผ่านระยะเวลาที่กำหนด
[ ] ตรวจสอบ known vulnerabilities ทั้งหมด
[ ] Slither static analysis ผ่าน
[ ] Mythril analysis ผ่าน

## Testing
[ ] Unit tests coverage > 95%
[ ] Integration tests ครอบคลุมทุก edge cases
[ ] Fork testing บน mainnet fork
[ ] Stress testing และ fuzzing
[ ] Testnet deployment และ testing (Goerli/Sepolia)
[ ] Public testnet testing พร้อม real users

## Economic Security
[ ] Token economics review
[ ] Flash loan attack analysis
[ ] Price manipulation scenarios tested
[ ] Liquidity analysis

## Operational
[ ] Multi-sig wallet setup เสร็จสิ้น
[ ] Emergency pause mechanism พร้อม
[ ] Monitoring และ alerting setup
[ ] Incident response plan เตรียมพร้อม
[ ] Documentation ครบถ้วน
```

---

## 2. Mainnet Deployment Checklist {#mainnet-checklist}

### สัญญาหลักสำหรับ Production System

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0

"""
@title ProductionVault - ตัวอย่าง Vault contract สำหรับ Production
@notice สัญญานี้แสดงให้เห็น patterns สำคัญสำหรับ production deployment
@dev ใช้ Checks-Effects-Interactions pattern และมี emergency controls
"""

from vyper.interfaces import ERC20

# ============================================================
# Interfaces
# ============================================================

interface IWETH:
    def deposit(): payable
    def withdraw(_amount: uint256): nonpayable
    def balanceOf(_owner: address) -> uint256: view

interface IPriceOracle:
    def getPrice(_token: address) -> uint256: view
    def isValid() -> bool: view

# ============================================================
# Events
# ============================================================

event Deposit:
    user: indexed(address)
    token: indexed(address)
    amount: uint256
    shares: uint256
    timestamp: uint256

event Withdrawal:
    user: indexed(address)
    token: indexed(address)
    amount: uint256
    shares: uint256
    timestamp: uint256

event EmergencyPause:
    caller: indexed(address)
    reason: String[200]
    timestamp: uint256

event EmergencyUnpause:
    caller: indexed(address)
    timestamp: uint256

event OwnershipTransferred:
    previousOwner: indexed(address)
    newOwner: indexed(address)

event FeeUpdated:
    feeType: String[50]
    oldFee: uint256
    newFee: uint256

event Harvest:
    token: indexed(address)
    amount: uint256
    timestamp: uint256

# ============================================================
# Structs
# ============================================================

struct TokenInfo:
    isSupported: bool
    totalDeposited: uint256
    totalShares: uint256
    depositFee: uint256      # basis points (1 = 0.01%)
    withdrawFee: uint256     # basis points
    lastHarvestTime: uint256
    minDeposit: uint256
    maxDeposit: uint256

struct UserPosition:
    shares: uint256
    lastDepositTime: uint256
    totalDeposited: uint256
    totalWithdrawn: uint256

# ============================================================
# State Variables
# ============================================================

# Access control
owner: public(address)
pendingOwner: public(address)
guardian: public(address)          # สามารถ pause ได้ แต่ unpause ต้องเป็น owner
operators: public(HashMap[address, bool])

# Pause mechanism
isPaused: public(bool)
pauseReason: public(String[200])
pauseTimestamp: public(uint256)

# Token management
supportedTokens: public(DynArray[address, 50])
tokenInfo: public(HashMap[address, TokenInfo])
userPositions: public(HashMap[address, HashMap[address, UserPosition]])

# Fee management
treasury: public(address)
MAX_FEE: constant(uint256) = 1000  # 10% max fee

# Reentrancy guard
_locked: bool

# Version control
VERSION: constant(String[10]) = "1.0.0"
DEPLOYMENT_TIMESTAMP: public(uint256)

# ============================================================
# Constructor
# ============================================================

@deploy
def __init__(
    _owner: address,
    _guardian: address,
    _treasury: address
):
    """
    @notice Initialize the vault
    @param _owner Contract owner (should be multi-sig)
    @param _guardian Emergency guardian address
    @param _treasury Fee recipient address
    """
    assert _owner != empty(address), "Invalid owner"
    assert _guardian != empty(address), "Invalid guardian"
    assert _treasury != empty(address), "Invalid treasury"
    
    self.owner = _owner
    self.guardian = _guardian
    self.treasury = _treasury
    self.DEPLOYMENT_TIMESTAMP = block.timestamp
    self._locked = False
    self.isPaused = False
    
    log OwnershipTransferred(empty(address), _owner)

# ============================================================
# Modifiers (implemented as internal functions)
# ============================================================

@internal
def _onlyOwner():
    assert msg.sender == self.owner, "Not owner"

@internal
def _onlyGuardianOrOwner():
    assert msg.sender == self.owner or msg.sender == self.guardian, "Not authorized"

@internal
def _onlyOperator():
    assert self.operators[msg.sender] or msg.sender == self.owner, "Not operator"

@internal
def _notPaused():
    assert not self.isPaused, "Contract is paused"

@internal
def _nonReentrant():
    assert not self._locked, "Reentrant call"
    self._locked = True

@internal
def _unlock():
    self._locked = False

# ============================================================
# Core Functions
# ============================================================

@external
def deposit(_token: address, _amount: uint256) -> uint256:
    """
    @notice ฝากเงินเข้า vault
    @param _token Token address
    @param _amount Amount to deposit
    @return shares Shares received
    """
    self._nonReentrant()
    self._notPaused()
    
    # Checks
    info: TokenInfo = self.tokenInfo[_token]
    assert info.isSupported, "Token not supported"
    assert _amount >= info.minDeposit, "Below minimum deposit"
    assert _amount <= info.maxDeposit, "Exceeds maximum deposit"
    
    # Calculate shares before transfer (prevent inflation attack)
    shares: uint256 = self._calculateShares(_token, _amount)
    
    # Calculate fee
    fee: uint256 = (_amount * info.depositFee) / 10000
    depositAmount: uint256 = _amount - fee
    
    # Effects - update state before external calls
    self.tokenInfo[_token].totalDeposited += depositAmount
    self.tokenInfo[_token].totalShares += shares
    
    position: UserPosition = self.userPositions[msg.sender][_token]
    self.userPositions[msg.sender][_token].shares = position.shares + shares
    self.userPositions[msg.sender][_token].lastDepositTime = block.timestamp
    self.userPositions[msg.sender][_token].totalDeposited = position.totalDeposited + _amount
    
    # Interactions - external calls last
    assert ERC20(_token).transferFrom(msg.sender, self, _amount), "Transfer failed"
    if fee > 0:
        assert ERC20(_token).transfer(self.treasury, fee), "Fee transfer failed"
    
    log Deposit(msg.sender, _token, _amount, shares, block.timestamp)
    
    self._unlock()
    return shares

@external
def withdraw(_token: address, _shares: uint256) -> uint256:
    """
    @notice ถอนเงินจาก vault
    @param _token Token address
    @param _shares Shares to burn
    @return amount Amount received
    """
    self._nonReentrant()
    self._notPaused()
    
    # Checks
    position: UserPosition = self.userPositions[msg.sender][_token]
    assert position.shares >= _shares, "Insufficient shares"
    
    info: TokenInfo = self.tokenInfo[_token]
    amount: uint256 = self._calculateAmount(_token, _shares)
    
    fee: uint256 = (amount * info.withdrawFee) / 10000
    withdrawAmount: uint256 = amount - fee
    
    # Effects
    self.tokenInfo[_token].totalShares -= _shares
    self.tokenInfo[_token].totalDeposited -= amount
    
    self.userPositions[msg.sender][_token].shares = position.shares - _shares
    self.userPositions[msg.sender][_token].totalWithdrawn = position.totalWithdrawn + withdrawAmount
    
    # Interactions
    if fee > 0:
        assert ERC20(_token).transfer(self.treasury, fee), "Fee transfer failed"
    assert ERC20(_token).transfer(msg.sender, withdrawAmount), "Transfer failed"
    
    log Withdrawal(msg.sender, _token, withdrawAmount, _shares, block.timestamp)
    
    self._unlock()
    return withdrawAmount

# ============================================================
# Internal Calculation Functions
# ============================================================

@internal
@view
def _calculateShares(_token: address, _amount: uint256) -> uint256:
    """คำนวณ shares จาก amount"""
    info: TokenInfo = self.tokenInfo[_token]
    
    if info.totalShares == 0:
        # First deposit - 1:1 ratio
        return _amount
    
    # shares = amount * totalShares / totalDeposited
    return (_amount * info.totalShares) / info.totalDeposited

@internal
@view
def _calculateAmount(_token: address, _shares: uint256) -> uint256:
    """คำนวณ amount จาก shares"""
    info: TokenInfo = self.tokenInfo[_token]
    
    if info.totalShares == 0:
        return 0
    
    # amount = shares * totalDeposited / totalShares
    return (_shares * info.totalDeposited) / info.totalShares

# ============================================================
# View Functions
# ============================================================

@external
@view
def getUserShares(_user: address, _token: address) -> uint256:
    """ดู shares ของ user"""
    return self.userPositions[_user][_token].shares

@external
@view
def getUserValue(_user: address, _token: address) -> uint256:
    """คำนวณมูลค่า position ของ user"""
    shares: uint256 = self.userPositions[_user][_token].shares
    return self._calculateAmount(_token, shares)

@external
@view
def getTokenTVL(_token: address) -> uint256:
    """ดู Total Value Locked ของ token"""
    return self.tokenInfo[_token].totalDeposited

@external
@view
def getSupportedTokens() -> DynArray[address, 50]:
    """รายการ tokens ที่ support"""
    return self.supportedTokens

# ============================================================
# Admin Functions
# ============================================================

@external
def addSupportedToken(
    _token: address,
    _depositFee: uint256,
    _withdrawFee: uint256,
    _minDeposit: uint256,
    _maxDeposit: uint256
):
    """เพิ่ม token ที่รองรับ"""
    self._onlyOwner()
    
    assert not self.tokenInfo[_token].isSupported, "Already supported"
    assert _depositFee <= MAX_FEE, "Deposit fee too high"
    assert _withdrawFee <= MAX_FEE, "Withdraw fee too high"
    assert _minDeposit < _maxDeposit, "Invalid deposit range"
    
    self.supportedTokens.append(_token)
    self.tokenInfo[_token] = TokenInfo({
        isSupported: True,
        totalDeposited: 0,
        totalShares: 0,
        depositFee: _depositFee,
        withdrawFee: _withdrawFee,
        lastHarvestTime: block.timestamp,
        minDeposit: _minDeposit,
        maxDeposit: _maxDeposit
    })

@external
def pause(_reason: String[200]):
    """หยุดการทำงานฉุกเฉิน"""
    self._onlyGuardianOrOwner()
    
    self.isPaused = True
    self.pauseReason = _reason
    self.pauseTimestamp = block.timestamp
    
    log EmergencyPause(msg.sender, _reason, block.timestamp)

@external
def unpause():
    """เปิดการทำงานอีกครั้ง (เฉพาะ owner)"""
    self._onlyOwner()
    
    self.isPaused = False
    log EmergencyUnpause(msg.sender, block.timestamp)

@external
def updateFee(_token: address, _feeType: String[50], _newFee: uint256):
    """อัปเดตค่าธรรมเนียม"""
    self._onlyOwner()
    assert _newFee <= MAX_FEE, "Fee too high"
    
    info: TokenInfo = self.tokenInfo[_token]
    
    if keccak256(_feeType) == keccak256("deposit"):
        oldFee: uint256 = info.depositFee
        self.tokenInfo[_token].depositFee = _newFee
        log FeeUpdated(_feeType, oldFee, _newFee)
    elif keccak256(_feeType) == keccak256("withdraw"):
        oldFee: uint256 = info.withdrawFee
        self.tokenInfo[_token].withdrawFee = _newFee
        log FeeUpdated(_feeType, oldFee, _newFee)

@external
def transferOwnership(_newOwner: address):
    """โอน ownership (two-step process)"""
    self._onlyOwner()
    assert _newOwner != empty(address), "Invalid address"
    
    self.pendingOwner = _newOwner

@external
def acceptOwnership():
    """รับ ownership"""
    assert msg.sender == self.pendingOwner, "Not pending owner"
    
    oldOwner: address = self.owner
    self.owner = self.pendingOwner
    self.pendingOwner = empty(address)
    
    log OwnershipTransferred(oldOwner, self.owner)

@external
def setOperator(_operator: address, _status: bool):
    """ตั้งค่า operator"""
    self._onlyOwner()
    self.operators[_operator] = _status
```

---

## 3. Multi-sig Deployment {#multisig}

### ทำไมต้อง Multi-sig?

Multi-signature wallet คือ wallet ที่ต้องการลายเซ็นจากหลาย private keys เพื่อทำธุรกรรม เหมาะสำหรับ:
- **ป้องกัน single point of failure** - ถ้า key หนึ่งหาย ยังมี key อื่น
- **ป้องกัน insider attack** - ต้องการความเห็นชอบหลายคน
- **Transparency** - สามารถตรวจสอบได้ว่าใครอนุมัติอะไร

### Simple Multi-sig Contract

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0

"""
@title SimpleMultiSig - Multi-signature wallet อย่างง่าย
@notice ต้องการลายเซ็น M-of-N เพื่อ execute transaction
@dev ใช้สำหรับ governance และ fund management
"""

# ============================================================
# Events
# ============================================================

event SubmitTransaction:
    txId: indexed(uint256)
    submitter: indexed(address)
    to: address
    value: uint256
    data: Bytes[1024]

event ConfirmTransaction:
    txId: indexed(uint256)
    confirmer: indexed(address)

event RevokeConfirmation:
    txId: indexed(uint256)
    revoker: indexed(address)

event ExecuteTransaction:
    txId: indexed(uint256)
    executor: indexed(address)
    success: bool

event OwnerAdded:
    owner: indexed(address)

event OwnerRemoved:
    owner: indexed(address)

event RequirementChanged:
    required: uint256

# ============================================================
# Structs
# ============================================================

struct Transaction:
    to: address
    value: uint256
    data: Bytes[1024]
    executed: bool
    numConfirmations: uint256
    submitter: address
    submittedAt: uint256
    description: String[200]

# ============================================================
# Constants
# ============================================================

MAX_OWNERS: constant(uint256) = 20
MAX_TRANSACTIONS: constant(uint256) = 1000

# ============================================================
# State Variables
# ============================================================

owners: public(DynArray[address, MAX_OWNERS])
isOwner: public(HashMap[address, bool])
required: public(uint256)

transactions: public(HashMap[uint256, Transaction])
confirmations: public(HashMap[uint256, HashMap[address, bool]])
transactionCount: public(uint256)

# ============================================================
# Constructor
# ============================================================

@deploy
def __init__(_owners: DynArray[address, MAX_OWNERS], _required: uint256):
    """
    @notice สร้าง multi-sig wallet
    @param _owners รายชื่อ owners
    @param _required จำนวน signatures ที่ต้องการ
    """
    assert len(_owners) > 0, "No owners"
    assert _required > 0 and _required <= len(_owners), "Invalid required"
    
    for owner: address in _owners:
        assert owner != empty(address), "Invalid owner"
        assert not self.isOwner[owner], "Duplicate owner"
        
        self.isOwner[owner] = True
        self.owners.append(owner)
        log OwnerAdded(owner)
    
    self.required = _required

# ============================================================
# Core Functions
# ============================================================

@external
@payable
def submitTransaction(
    _to: address,
    _value: uint256,
    _data: Bytes[1024],
    _description: String[200]
) -> uint256:
    """
    @notice เสนอ transaction ใหม่
    @return txId Transaction ID
    """
    assert self.isOwner[msg.sender], "Not owner"
    assert _to != empty(address), "Invalid target"
    
    txId: uint256 = self.transactionCount
    
    self.transactions[txId] = Transaction({
        to: _to,
        value: _value,
        data: _data,
        executed: False,
        numConfirmations: 0,
        submitter: msg.sender,
        submittedAt: block.timestamp,
        description: _description
    })
    
    self.transactionCount += 1
    
    log SubmitTransaction(txId, msg.sender, _to, _value, _data)
    
    # Auto-confirm by submitter
    self._confirm(txId)
    
    return txId

@external
def confirmTransaction(_txId: uint256):
    """ยืนยัน transaction"""
    assert self.isOwner[msg.sender], "Not owner"
    assert _txId < self.transactionCount, "Invalid txId"
    assert not self.transactions[_txId].executed, "Already executed"
    assert not self.confirmations[_txId][msg.sender], "Already confirmed"
    
    self._confirm(_txId)

@internal
def _confirm(_txId: uint256):
    self.confirmations[_txId][msg.sender] = True
    self.transactions[_txId].numConfirmations += 1
    
    log ConfirmTransaction(_txId, msg.sender)

@external
def revokeConfirmation(_txId: uint256):
    """ยกเลิกการยืนยัน"""
    assert self.isOwner[msg.sender], "Not owner"
    assert self.confirmations[_txId][msg.sender], "Not confirmed"
    assert not self.transactions[_txId].executed, "Already executed"
    
    self.confirmations[_txId][msg.sender] = False
    self.transactions[_txId].numConfirmations -= 1
    
    log RevokeConfirmation(_txId, msg.sender)

@external
def executeTransaction(_txId: uint256):
    """
    @notice Execute transaction เมื่อได้รับ confirmations ครบ
    """
    assert self.isOwner[msg.sender], "Not owner"
    assert _txId < self.transactionCount, "Invalid txId"
    
    tx: Transaction = self.transactions[_txId]
    
    assert not tx.executed, "Already executed"
    assert tx.numConfirmations >= self.required, "Insufficient confirmations"
    
    self.transactions[_txId].executed = True
    
    success: bool = False
    response: Bytes[32] = b""
    success, response = raw_call(
        tx.to,
        tx.data,
        value=tx.value,
        max_outsize=32,
        revert_on_failure=False
    )
    
    log ExecuteTransaction(_txId, msg.sender, success)
    
    if not success:
        # Revert state if failed
        self.transactions[_txId].executed = False
        raise "Execution failed"

# ============================================================
# View Functions
# ============================================================

@external
@view
def getOwners() -> DynArray[address, MAX_OWNERS]:
    """รายชื่อ owners ทั้งหมด"""
    return self.owners

@external
@view
def getTransaction(_txId: uint256) -> Transaction:
    """ดูรายละเอียด transaction"""
    return self.transactions[_txId]

@external
@view
def isConfirmed(_txId: uint256, _owner: address) -> bool:
    """ตรวจสอบว่า owner ยืนยันแล้วหรือไม่"""
    return self.confirmations[_txId][_owner]

@external
@view
def getPendingTransactions() -> DynArray[uint256, MAX_TRANSACTIONS]:
    """รายการ transactions ที่รอ execution"""
    pending: DynArray[uint256, MAX_TRANSACTIONS] = []
    
    for i: uint256 in range(MAX_TRANSACTIONS):
        if i >= self.transactionCount:
            break
        if not self.transactions[i].executed:
            pending.append(i)
    
    return pending

# ============================================================
# Admin Functions  
# ============================================================

@external
def addOwner(_owner: address):
    """เพิ่ม owner ใหม่ (ต้องผ่าน multi-sig)"""
    assert msg.sender == self, "Must be through multisig"
    assert _owner != empty(address), "Invalid owner"
    assert not self.isOwner[_owner], "Already owner"
    assert len(self.owners) < MAX_OWNERS, "Too many owners"
    
    self.isOwner[_owner] = True
    self.owners.append(_owner)
    
    log OwnerAdded(_owner)

@external
def removeOwner(_owner: address):
    """ลบ owner (ต้องผ่าน multi-sig)"""
    assert msg.sender == self, "Must be through multisig"
    assert self.isOwner[_owner], "Not owner"
    assert len(self.owners) > self.required, "Would break requirement"
    
    self.isOwner[_owner] = False
    
    # Remove from array
    newOwners: DynArray[address, MAX_OWNERS] = []
    for owner: address in self.owners:
        if owner != _owner:
            newOwners.append(owner)
    self.owners = newOwners
    
    log OwnerRemoved(_owner)

@external
@payable
def receive():
    pass
```

---

## 4. Deployment ด้วย Gnosis Safe {#gnosis-safe}

### Integration กับ Gnosis Safe

Gnosis Safe เป็น multi-sig wallet ที่ใช้งานกันแพร่หลายที่สุดใน DeFi

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0

"""
@title GnosisSafeIntegration - สัญญาที่ integrate กับ Gnosis Safe
@notice แสดงวิธี deploy และ manage contracts ผ่าน Gnosis Safe
"""

from vyper.interfaces import ERC20

# ============================================================
# Interfaces สำหรับ Gnosis Safe
# ============================================================

interface IGnosisSafe:
    def execTransaction(
        to: address,
        value: uint256,
        data: Bytes[1024],
        operation: uint8,
        safeTxGas: uint256,
        baseGas: uint256,
        gasPrice: uint256,
        gasToken: address,
        refundReceiver: address,
        signatures: Bytes[65]
    ) -> bool: nonpayable
    
    def getOwners() -> DynArray[address, 100]: view
    def getThreshold() -> uint256: view
    def isOwner(_owner: address) -> bool: view
    def nonce() -> uint256: view

interface IProxyFactory:
    def createProxy(
        _masterCopy: address,
        _data: Bytes[256]
    ) -> address: nonpayable

# ============================================================
# Events
# ============================================================

event SafeTransactionProposed:
    txHash: indexed(bytes32)
    proposer: indexed(address)
    to: address
    value: uint256

event SafeTransactionExecuted:
    txHash: indexed(bytes32)
    executor: indexed(address)
    success: bool

# ============================================================
# Structs
# ============================================================

struct SafeProposal:
    to: address
    value: uint256
    data: Bytes[1024]
    description: String[200]
    proposer: address
    proposedAt: uint256
    executed: bool
    txHash: bytes32

# ============================================================
# State Variables
# ============================================================

gnosisSafe: public(address)
owner: public(address)

proposals: public(HashMap[bytes32, SafeProposal])
proposalList: public(DynArray[bytes32, 1000])

# ============================================================
# Constructor
# ============================================================

@deploy
def __init__(_gnosisSafe: address):
    """
    @notice เชื่อมต่อกับ Gnosis Safe ที่มีอยู่แล้ว
    @param _gnosisSafe Address ของ Gnosis Safe
    """
    assert _gnosisSafe != empty(address), "Invalid safe"
    
    self.gnosisSafe = _gnosisSafe
    self.owner = msg.sender

# ============================================================
# Core Functions
# ============================================================

@external
def proposeTransaction(
    _to: address,
    _value: uint256,
    _data: Bytes[1024],
    _description: String[200]
) -> bytes32:
    """
    @notice เสนอ transaction ให้ Safe ดำเนินการ
    @return txHash Hash ของ transaction
    """
    safe: IGnosisSafe = IGnosisSafe(self.gnosisSafe)
    assert safe.isOwner(msg.sender), "Not safe owner"
    
    # คำนวณ tx hash
    txHash: bytes32 = keccak256(
        concat(
            convert(_to, bytes32),
            convert(_value, bytes32),
            keccak256(_data),
            convert(block.timestamp, bytes32)
        )
    )
    
    self.proposals[txHash] = SafeProposal({
        to: _to,
        value: _value,
        data: _data,
        description: _description,
        proposer: msg.sender,
        proposedAt: block.timestamp,
        executed: False,
        txHash: txHash
    })
    
    self.proposalList.append(txHash)
    
    log SafeTransactionProposed(txHash, msg.sender, _to, _value)
    
    return txHash

@external
@view
def getSafeInfo() -> (uint256, DynArray[address, 100]):
    """ดูข้อมูล Safe"""
    safe: IGnosisSafe = IGnosisSafe(self.gnosisSafe)
    threshold: uint256 = safe.getThreshold()
    owners: DynArray[address, 100] = safe.getOwners()
    return threshold, owners

@external
@view
def getProposal(_txHash: bytes32) -> SafeProposal:
    """ดูรายละเอียด proposal"""
    return self.proposals[_txHash]
```

---

## 5. Proxy Deployment Patterns {#proxy}

### Transparent Proxy Pattern

Proxy patterns ทำให้ contract สามารถ upgrade ได้ โดยแยก logic ออกจาก storage

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0

"""
@title TransparentProxy - Transparent Upgradeable Proxy
@notice Proxy contract ที่แยก admin functions จาก user functions
@dev ใช้ EIP-1967 storage slots
"""

# ============================================================
# EIP-1967 Storage Slots
# ============================================================

# bytes32(uint256(keccak256("eip1967.proxy.implementation")) - 1)
IMPLEMENTATION_SLOT: constant(bytes32) = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc

# bytes32(uint256(keccak256("eip1967.proxy.admin")) - 1)
ADMIN_SLOT: constant(bytes32) = 0xb53127684a568b3173ae13b9f8a6016e243e63b6e8ee1178d6a717850b5d6103

# ============================================================
# Events
# ============================================================

event Upgraded:
    implementation: indexed(address)

event AdminChanged:
    previousAdmin: indexed(address)
    newAdmin: indexed(address)

# ============================================================
# Constructor
# ============================================================

@deploy
def __init__(_logic: address, _admin: address, _data: Bytes[1024]):
    """
    @notice Deploy proxy
    @param _logic Implementation contract address
    @param _admin Proxy admin address
    @param _data Initialization calldata
    """
    assert _logic != empty(address), "Invalid logic"
    assert _admin != empty(address), "Invalid admin"
    
    # Store implementation
    raw_storage_write(IMPLEMENTATION_SLOT, convert(_logic, bytes32))
    
    # Store admin
    raw_storage_write(ADMIN_SLOT, convert(_admin, bytes32))
    
    log Upgraded(_logic)
    log AdminChanged(empty(address), _admin)
    
    # Initialize if data provided
    if len(_data) > 0:
        raw_call(_logic, _data, is_delegate_call=True)

# ============================================================
# Admin Functions
# ============================================================

@external
def upgradeTo(_newImplementation: address):
    """อัปเกรด implementation"""
    adminSlot: bytes32 = raw_storage_read(ADMIN_SLOT)
    admin: address = convert(adminSlot, address)
    assert msg.sender == admin, "Not admin"
    assert _newImplementation != empty(address), "Invalid implementation"
    
    raw_storage_write(IMPLEMENTATION_SLOT, convert(_newImplementation, bytes32))
    
    log Upgraded(_newImplementation)

@external
def upgradeToAndCall(_newImplementation: address, _data: Bytes[1024]):
    """อัปเกรดและ initialize"""
    adminSlot: bytes32 = raw_storage_read(ADMIN_SLOT)
    admin: address = convert(adminSlot, address)
    assert msg.sender == admin, "Not admin"
    assert _newImplementation != empty(address), "Invalid implementation"
    
    raw_storage_write(IMPLEMENTATION_SLOT, convert(_newImplementation, bytes32))
    
    log Upgraded(_newImplementation)
    
    if len(_data) > 0:
        raw_call(_newImplementation, _data, is_delegate_call=True)

@external
def changeAdmin(_newAdmin: address):
    """เปลี่ยน admin"""
    adminSlot: bytes32 = raw_storage_read(ADMIN_SLOT)
    admin: address = convert(adminSlot, address)
    assert msg.sender == admin, "Not admin"
    assert _newAdmin != empty(address), "Invalid admin"
    
    raw_storage_write(ADMIN_SLOT, convert(_newAdmin, bytes32))
    
    log AdminChanged(admin, _newAdmin)

@external
@view
def implementation() -> address:
    """ดู implementation address"""
    adminSlot: bytes32 = raw_storage_read(ADMIN_SLOT)
    admin: address = convert(adminSlot, address)
    assert msg.sender == admin, "Not admin"
    
    implSlot: bytes32 = raw_storage_read(IMPLEMENTATION_SLOT)
    return convert(implSlot, address)

@external
@view
def admin() -> address:
    """ดู admin address"""
    adminSlot: bytes32 = raw_storage_read(ADMIN_SLOT)
    admin: address = convert(adminSlot, address)
    assert msg.sender == admin, "Not admin"
    
    return admin

# ============================================================
# Fallback - delegate calls to implementation
# ============================================================

@external
@payable
def __default__():
    """Delegate calls ไปยัง implementation"""
    implSlot: bytes32 = raw_storage_read(IMPLEMENTATION_SLOT)
    impl: address = convert(implSlot, address)
    
    raw_call(impl, msg.data, is_delegate_call=True, max_outsize=0)
```

### ProxyAdmin Contract

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0

"""
@title ProxyAdmin - จัดการ Proxy contracts
@notice Contract สำหรับจัดการการ upgrade ผ่าน Timelock
"""

interface ITransparentProxy:
    def upgradeTo(_newImpl: address): nonpayable
    def upgradeToAndCall(_newImpl: address, _data: Bytes[1024]): nonpayable
    def changeAdmin(_newAdmin: address): nonpayable
    def implementation() -> address: view
    def admin() -> address: view

event ProxyUpgraded:
    proxy: indexed(address)
    oldImpl: indexed(address)
    newImpl: indexed(address)

event UpgradeScheduled:
    proxy: indexed(address)
    newImpl: indexed(address)
    executeAfter: uint256

struct PendingUpgrade:
    proxy: address
    newImpl: address
    data: Bytes[1024]
    proposedAt: uint256
    executeAfter: uint256
    executed: bool

# Timelock delay (48 hours)
TIMELOCK_DELAY: constant(uint256) = 48 * 3600

owner: public(address)
pendingUpgrades: public(HashMap[bytes32, PendingUpgrade])

@deploy
def __init__():
    self.owner = msg.sender

@external
def scheduleUpgrade(
    _proxy: address,
    _newImpl: address,
    _data: Bytes[1024]
) -> bytes32:
    """กำหนดการ upgrade (ต้องรอ timelock)"""
    assert msg.sender == self.owner, "Not owner"
    
    upgradeId: bytes32 = keccak256(
        concat(
            convert(_proxy, bytes32),
            convert(_newImpl, bytes32),
            convert(block.timestamp, bytes32)
        )
    )
    
    executeAfter: uint256 = block.timestamp + TIMELOCK_DELAY
    
    self.pendingUpgrades[upgradeId] = PendingUpgrade({
        proxy: _proxy,
        newImpl: _newImpl,
        data: _data,
        proposedAt: block.timestamp,
        executeAfter: executeAfter,
        executed: False
    })
    
    log UpgradeScheduled(_proxy, _newImpl, executeAfter)
    
    return upgradeId

@external
def executeUpgrade(_upgradeId: bytes32):
    """Execute upgrade หลังจาก timelock ผ่านแล้ว"""
    assert msg.sender == self.owner, "Not owner"
    
    upgrade: PendingUpgrade = self.pendingUpgrades[_upgradeId]
    assert upgrade.proxy != empty(address), "Invalid upgrade"
    assert not upgrade.executed, "Already executed"
    assert block.timestamp >= upgrade.executeAfter, "Timelock not passed"
    
    self.pendingUpgrades[_upgradeId].executed = True
    
    proxy: ITransparentProxy = ITransparentProxy(upgrade.proxy)
    oldImpl: address = proxy.implementation()
    
    if len(upgrade.data) > 0:
        proxy.upgradeToAndCall(upgrade.newImpl, upgrade.data)
    else:
        proxy.upgradeTo(upgrade.newImpl)
    
    log ProxyUpgraded(upgrade.proxy, oldImpl, upgrade.newImpl)
```

---

## 6. Contract Verification {#verification}

### การ Verify Contract บน Etherscan

Contract verification สำคัญมากเพราะ:
- ทำให้ผู้ใช้สามารถอ่าน source code ได้
- เพิ่มความน่าเชื่อถือ
- ทำให้ auditors และ researchers สามารถตรวจสอบได้

```python
# verification_script.py - Script สำหรับ verify contract
# ใช้กับ Vyper 0.4.0

import requests
import json
import os

ETHERSCAN_API_KEY = os.environ.get("ETHERSCAN_API_KEY")
ETHERSCAN_API_URL = "https://api.etherscan.io/api"

def verify_vyper_contract(
    contract_address: str,
    contract_name: str,
    source_code: str,
    compiler_version: str = "v0.4.0",
    optimization_used: bool = True,
    constructor_abi: str = ""
):
    """
    Verify Vyper contract บน Etherscan
    """
    
    payload = {
        "apikey": ETHERSCAN_API_KEY,
        "module": "contract",
        "action": "verifysourcecode",
        "contractaddress": contract_address,
        "sourceCode": source_code,
        "codeformat": "vyper-single-file",
        "contractname": contract_name,
        "compilerversion": compiler_version,
        "optimizationUsed": "1" if optimization_used else "0",
        "constructorArguements": constructor_abi,
        "licenseType": "3",  # MIT
    }
    
    response = requests.post(ETHERSCAN_API_URL, data=payload)
    result = response.json()
    
    if result["status"] == "1":
        guid = result["result"]
        print(f"Verification submitted. GUID: {guid}")
        return check_verification_status(guid)
    else:
        print(f"Error: {result['result']}")
        return False

def check_verification_status(guid: str) -> bool:
    """
    ตรวจสอบสถานะ verification
    """
    import time
    
    for _ in range(10):  # ลอง 10 ครั้ง
        time.sleep(5)
        
        payload = {
            "apikey": ETHERSCAN_API_KEY,
            "module": "contract",
            "action": "checkverifystatus",
            "guid": guid
        }
        
        response = requests.post(ETHERSCAN_API_URL, data=payload)
        result = response.json()
        
        if result["result"] == "Pass - Verified":
            print("Contract verified successfully!")
            return True
        elif "Pending" in result["result"]:
            print("Verification pending...")
            continue
        else:
            print(f"Verification failed: {result['result']}")
            return False
    
    return False

# Usage
if __name__ == "__main__":
    with open("ProductionVault.vy", "r") as f:
        source_code = f.read()
    
    verify_vyper_contract(
        contract_address="0x...",
        contract_name="ProductionVault",
        source_code=source_code
    )
```

### Deployment Script ฉบับสมบูรณ์

```python
# deploy_production.py - Production deployment script

from web3 import Web3
from eth_account import Account
import json
import os
import vyper

def deploy_with_multisig_check():
    """
    Script สำหรับ production deployment
    - ตรวจสอบ environment
    - Deploy ผ่าน multi-sig
    - Verify contract
    - บันทึก deployment info
    """
    
    # 1. ตรวจสอบ environment
    print("=== Checking Environment ===")
    
    required_env = [
        "MAINNET_RPC_URL",
        "DEPLOYER_PRIVATE_KEY",
        "MULTISIG_ADDRESS",
        "ETHERSCAN_API_KEY"
    ]
    
    for env in required_env:
        assert os.environ.get(env), f"Missing {env}"
    print("✓ Environment variables OK")
    
    # 2. Connect to network
    w3 = Web3(Web3.HTTPProvider(os.environ["MAINNET_RPC_URL"]))
    assert w3.is_connected(), "Cannot connect to network"
    
    chain_id = w3.eth.chain_id
    assert chain_id == 1, f"Wrong network! Expected mainnet (1), got {chain_id}"
    print(f"✓ Connected to mainnet (chain_id: {chain_id})")
    
    # 3. Load deployer account
    deployer = Account.from_key(os.environ["DEPLOYER_PRIVATE_KEY"])
    balance = w3.eth.get_balance(deployer.address)
    print(f"✓ Deployer: {deployer.address}")
    print(f"  Balance: {w3.from_wei(balance, 'ether'):.4f} ETH")
    
    # ต้องมี ETH เพียงพอ
    assert balance > w3.to_wei(0.1, 'ether'), "Insufficient ETH for deployment"
    
    # 4. Compile contract
    print("\n=== Compiling Contract ===")
    with open("ProductionVault.vy", "r") as f:
        source = f.read()
    
    # ใช้ vyper compiler
    result = vyper.compile_code(source, output_formats=['abi', 'bytecode'])
    abi = result['abi']
    bytecode = result['bytecode']
    print("✓ Contract compiled")
    
    # 5. Deploy contract
    print("\n=== Deploying Contract ===")
    
    multisig = os.environ["MULTISIG_ADDRESS"]
    treasury = os.environ.get("TREASURY_ADDRESS", multisig)
    guardian = os.environ.get("GUARDIAN_ADDRESS", multisig)
    
    Contract = w3.eth.contract(abi=abi, bytecode=bytecode)
    
    # Estimate gas
    gas_estimate = Contract.constructor(
        multisig,  # owner = multisig
        guardian,
        treasury
    ).estimate_gas({'from': deployer.address})
    
    print(f"  Estimated gas: {gas_estimate:,}")
    
    # Get gas price
    gas_price = w3.eth.gas_price
    estimated_cost = w3.from_wei(gas_estimate * gas_price, 'ether')
    print(f"  Estimated cost: {estimated_cost:.6f} ETH")
    
    # Build transaction
    tx = Contract.constructor(
        multisig,
        guardian,
        treasury
    ).build_transaction({
        'from': deployer.address,
        'gas': int(gas_estimate * 1.2),  # 20% buffer
        'gasPrice': int(gas_price * 1.1),  # 10% above current
        'nonce': w3.eth.get_transaction_count(deployer.address),
        'chainId': 1
    })
    
    # Sign and send
    signed_tx = w3.eth.account.sign_transaction(tx, deployer.key)
    tx_hash = w3.eth.send_raw_transaction(signed_tx.rawTransaction)
    
    print(f"  Transaction sent: {tx_hash.hex()}")
    print("  Waiting for confirmation...")
    
    # Wait for receipt
    receipt = w3.eth.wait_for_transaction_receipt(tx_hash, timeout=300)
    
    assert receipt['status'] == 1, "Transaction failed!"
    
    contract_address = receipt['contractAddress']
    print(f"✓ Contract deployed at: {contract_address}")
    
    # 6. Save deployment info
    deployment_info = {
        "contract": "ProductionVault",
        "address": contract_address,
        "deployer": deployer.address,
        "owner": multisig,
        "network": "mainnet",
        "chain_id": chain_id,
        "tx_hash": tx_hash.hex(),
        "block_number": receipt['blockNumber'],
        "gas_used": receipt['gasUsed'],
        "timestamp": w3.eth.get_block(receipt['blockNumber'])['timestamp']
    }
    
    with open("deployment_mainnet.json", "w") as f:
        json.dump(deployment_info, f, indent=2)
    
    print("✓ Deployment info saved")
    
    return contract_address, abi

if __name__ == "__main__":
    deploy_with_multisig_check()
```

---

## 7. Post-Deployment Procedures {#post-deployment}

### Post-Deployment Checklist

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0

"""
@title DeploymentRegistry - บันทึก deployment information
@notice เก็บข้อมูลการ deploy สำหรับ tracking และ audit
"""

event ContractRegistered:
    name: indexed(String[50])
    contractAddress: indexed(address)
    version: String[10]
    timestamp: uint256

struct DeploymentRecord:
    name: String[50]
    contractAddress: address
    version: String[10]
    deployer: address
    deployedAt: uint256
    verified: bool
    auditReport: String[200]  # IPFS hash of audit report
    notes: String[500]

MAX_DEPLOYMENTS: constant(uint256) = 100

owner: public(address)
deployments: public(HashMap[String[50], DeploymentRecord])
deploymentList: public(DynArray[String[50], MAX_DEPLOYMENTS])

@deploy
def __init__():
    self.owner = msg.sender

@external
def registerDeployment(
    _name: String[50],
    _address: address,
    _version: String[10],
    _auditReport: String[200],
    _notes: String[500]
):
    """บันทึก deployment ใหม่"""
    assert msg.sender == self.owner, "Not owner"
    assert _address != empty(address), "Invalid address"
    
    self.deployments[_name] = DeploymentRecord({
        name: _name,
        contractAddress: _address,
        version: _version,
        deployer: msg.sender,
        deployedAt: block.timestamp,
        verified: False,
        auditReport: _auditReport,
        notes: _notes
    })
    
    self.deploymentList.append(_name)
    
    log ContractRegistered(_name, _address, _version, block.timestamp)

@external
def markVerified(_name: String[50]):
    """mark contract ว่า verified แล้ว"""
    assert msg.sender == self.owner, "Not owner"
    assert self.deployments[_name].contractAddress != empty(address), "Not found"
    
    self.deployments[_name].verified = True

@external
@view
def getDeployment(_name: String[50]) -> DeploymentRecord:
    """ดู deployment record"""
    return self.deployments[_name]

@external
@view
def getAllDeployments() -> DynArray[String[50], MAX_DEPLOYMENTS]:
    """รายการ deployments ทั้งหมด"""
    return self.deploymentList
```

### Post-Deployment Testing Script

```python
# post_deployment_tests.py

from web3 import Web3
import json

def run_post_deployment_tests(contract_address: str, abi: list, w3: Web3):
    """
    ทดสอบ contract หลัง deployment
    """
    contract = w3.eth.contract(address=contract_address, abi=abi)
    
    print("=== Running Post-Deployment Tests ===\n")
    
    tests_passed = 0
    tests_failed = 0
    
    def test(name: str, condition: bool, detail: str = ""):
        nonlocal tests_passed, tests_failed
        if condition:
            print(f"  ✓ {name}")
            tests_passed += 1
        else:
            print(f"  ✗ {name}: {detail}")
            tests_failed += 1
    
    # Test 1: Contract is deployed
    code = w3.eth.get_code(contract_address)
    test("Contract deployed", len(code) > 0)
    
    # Test 2: Owner is set correctly
    owner = contract.functions.owner().call()
    expected_owner = "0x..."  # multisig address
    test("Owner is multisig", owner.lower() == expected_owner.lower(),
         f"Got {owner}, expected {expected_owner}")
    
    # Test 3: Contract is not paused
    is_paused = contract.functions.isPaused().call()
    test("Contract is not paused", not is_paused)
    
    # Test 4: Version is correct
    # version = contract.functions.VERSION().call()
    # test("Version is correct", version == "1.0.0")
    
    print(f"\nResults: {tests_passed} passed, {tests_failed} failed")
    
    return tests_failed == 0

# Run tests after deployment
if __name__ == "__main__":
    with open("deployment_mainnet.json") as f:
        deployment = json.load(f)
    
    with open("ProductionVault_abi.json") as f:
        abi = json.load(f)
    
    w3 = Web3(Web3.HTTPProvider("https://mainnet.infura.io/v3/..."))
    
    success = run_post_deployment_tests(deployment["address"], abi, w3)
    
    if success:
        print("\n✓ All post-deployment tests passed!")
    else:
        print("\n✗ Some tests failed. Investigate before proceeding.")
```

---

## สรุป

การ deploy สู่ production ต้องการ:

1. **Security First** - Audit, testing, bug bounty
2. **Multi-sig Control** - ไม่ใช้ EOA สำหรับ admin functions
3. **Verification** - Verify บน Etherscan เสมอ
4. **Monitoring** - Setup monitoring ก่อน go-live
5. **Incident Response** - มีแผนรับมือเหตุฉุกเฉิน
6. **Documentation** - บันทึกทุกอย่างให้ครบถ้วน

> **หลักการสำคัญ**: "Deploy slow, verify everything, monitor always"
