# Part 094: Governance Operations

## สารบัญ
1. [พื้นฐาน On-chain Governance](#governance-basics)
2. [Snapshot Voting Integration](#snapshot)
3. [On-chain Proposal Execution](#on-chain)
4. [Timelock Management](#timelock)
5. [GovernanceHelper Contract](#governance-helper)
6. [Emergency Governance](#emergency-governance)

---

## 1. พื้นฐาน On-chain Governance {#governance-basics}

Governance ใน DeFi หมายถึงกระบวนการที่ token holders สามารถมีส่วนร่วมในการตัดสินใจสำหรับ protocol โดยทั่วไปมีขั้นตอน:

1. **Discussion** - พูดคุยใน Forum (Discourse, Commonwealth)
2. **Temperature Check** - Snapshot vote ไม่ผูกพัน
3. **Formal Proposal** - On-chain proposal
4. **Voting Period** - โหวต (2-7 วัน)
5. **Timelock** - รอ execution (24-72 ชั่วโมง)
6. **Execution** - ดำเนินการตาม proposal

### Governance Token Contract

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0

"""
@title GovernanceToken - Token สำหรับ governance
@notice Implements ERC20 + voting power + delegation
@dev รองรับ on-chain governance ด้วย snapshot voting
"""

from vyper.interfaces import ERC20

# ============================================================
# Events
# ============================================================

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

# ============================================================
# Structs
# ============================================================

struct Checkpoint:
    fromBlock: uint256
    votes: uint256

# ============================================================
# Constants
# ============================================================

MAX_CHECKPOINTS: constant(uint256) = 1000
INITIAL_SUPPLY: constant(uint256) = 100_000_000 * 10**18  # 100M tokens

# ============================================================
# State Variables
# ============================================================

# ERC20 state
name: public(String[50])
symbol: public(String[10])
decimals: public(uint8)
totalSupply: public(uint256)
balances: HashMap[address, uint256]
allowances: HashMap[address, HashMap[address, uint256]]

# Delegation state
delegates_: HashMap[address, address]
checkpoints: HashMap[address, DynArray[Checkpoint, MAX_CHECKPOINTS]]
numCheckpoints: HashMap[address, uint256]

# Ownership
owner: public(address)
minter: public(address)

# ============================================================
# Constructor
# ============================================================

@deploy
def __init__(_name: String[50], _symbol: String[10], _initialHolder: address):
    """
    @notice Initialize governance token
    @param _name Token name
    @param _symbol Token symbol
    @param _initialHolder Initial token holder (treasury/team)
    """
    self.name = _name
    self.symbol = _symbol
    self.decimals = 18
    self.owner = msg.sender
    self.minter = msg.sender
    
    # Mint initial supply to holder
    self.totalSupply = INITIAL_SUPPLY
    self.balances[_initialHolder] = INITIAL_SUPPLY
    
    log Transfer(empty(address), _initialHolder, INITIAL_SUPPLY)

# ============================================================
# ERC20 Functions
# ============================================================

@external
@view
def balanceOf(_owner: address) -> uint256:
    return self.balances[_owner]

@external
@view
def allowance(_owner: address, _spender: address) -> uint256:
    return self.allowances[_owner][_spender]

@external
def transfer(_to: address, _value: uint256) -> bool:
    assert _to != empty(address), "Invalid recipient"
    assert self.balances[msg.sender] >= _value, "Insufficient balance"
    
    self.balances[msg.sender] -= _value
    self.balances[_to] += _value
    
    self._moveDelegates(
        self.delegates_[msg.sender],
        self.delegates_[_to],
        _value
    )
    
    log Transfer(msg.sender, _to, _value)
    return True

@external
def transferFrom(_from: address, _to: address, _value: uint256) -> bool:
    assert _to != empty(address), "Invalid recipient"
    assert self.balances[_from] >= _value, "Insufficient balance"
    assert self.allowances[_from][msg.sender] >= _value, "Insufficient allowance"
    
    self.balances[_from] -= _value
    self.allowances[_from][msg.sender] -= _value
    self.balances[_to] += _value
    
    self._moveDelegates(
        self.delegates_[_from],
        self.delegates_[_to],
        _value
    )
    
    log Transfer(_from, _to, _value)
    return True

@external
def approve(_spender: address, _value: uint256) -> bool:
    self.allowances[msg.sender][_spender] = _value
    log Approval(msg.sender, _spender, _value)
    return True

# ============================================================
# Delegation Functions
# ============================================================

@external
def delegate(_delegatee: address):
    """
    @notice มอบ voting power ให้ address อื่น
    @param _delegatee ผู้รับ voting power
    """
    self._delegate(msg.sender, _delegatee)

@internal
def _delegate(_delegator: address, _delegatee: address):
    currentDelegate: address = self.delegates_[_delegator]
    delegatorBalance: uint256 = self.balances[_delegator]
    
    self.delegates_[_delegator] = _delegatee
    
    log DelegateChanged(_delegator, currentDelegate, _delegatee)
    
    self._moveDelegates(currentDelegate, _delegatee, delegatorBalance)

@internal
def _moveDelegates(_srcRep: address, _dstRep: address, _amount: uint256):
    if _srcRep != _dstRep and _amount > 0:
        if _srcRep != empty(address):
            srcRepNum: uint256 = len(self.checkpoints[_srcRep])
            srcRepOld: uint256 = 0
            if srcRepNum > 0:
                srcRepOld = self.checkpoints[_srcRep][srcRepNum - 1].votes
            srcRepNew: uint256 = srcRepOld - _amount
            self._writeCheckpoint(_srcRep, srcRepNum, srcRepOld, srcRepNew)
        
        if _dstRep != empty(address):
            dstRepNum: uint256 = len(self.checkpoints[_dstRep])
            dstRepOld: uint256 = 0
            if dstRepNum > 0:
                dstRepOld = self.checkpoints[_dstRep][dstRepNum - 1].votes
            dstRepNew: uint256 = dstRepOld + _amount
            self._writeCheckpoint(_dstRep, dstRepNum, dstRepOld, dstRepNew)

@internal
def _writeCheckpoint(
    _delegatee: address,
    _nCheckpoints: uint256,
    _oldVotes: uint256,
    _newVotes: uint256
):
    if _nCheckpoints > 0 and self.checkpoints[_delegatee][_nCheckpoints - 1].fromBlock == block.number:
        self.checkpoints[_delegatee][_nCheckpoints - 1].votes = _newVotes
    else:
        self.checkpoints[_delegatee].append(Checkpoint({
            fromBlock: block.number,
            votes: _newVotes
        }))
    
    log DelegateVotesChanged(_delegatee, _oldVotes, _newVotes)

# ============================================================
# Voting Power Queries
# ============================================================

@external
@view
def getCurrentVotes(_account: address) -> uint256:
    """ดู voting power ปัจจุบัน"""
    nCheckpoints: uint256 = len(self.checkpoints[_account])
    if nCheckpoints == 0:
        return 0
    return self.checkpoints[_account][nCheckpoints - 1].votes

@external
@view
def getPriorVotes(_account: address, _blockNumber: uint256) -> uint256:
    """
    @notice ดู voting power ที่ block number ที่ระบุ (สำหรับ snapshot voting)
    @param _blockNumber Block ที่ต้องการดู voting power
    """
    assert _blockNumber < block.number, "Not yet determined"
    
    nCheckpoints: uint256 = len(self.checkpoints[_account])
    if nCheckpoints == 0:
        return 0
    
    # ตรวจสอบ checkpoint ล่าสุด
    if self.checkpoints[_account][nCheckpoints - 1].fromBlock <= _blockNumber:
        return self.checkpoints[_account][nCheckpoints - 1].votes
    
    # ตรวจสอบ checkpoint แรก
    if self.checkpoints[_account][0].fromBlock > _blockNumber:
        return 0
    
    # Binary search
    lower: uint256 = 0
    upper: uint256 = nCheckpoints - 1
    
    for _: uint256 in range(MAX_CHECKPOINTS):
        if lower >= upper:
            break
        center: uint256 = upper - (upper - lower) / 2
        cp: Checkpoint = self.checkpoints[_account][center]
        if cp.fromBlock == _blockNumber:
            return cp.votes
        elif cp.fromBlock < _blockNumber:
            lower = center
        else:
            upper = center - 1
    
    return self.checkpoints[_account][lower].votes

@external
@view
def delegates(_account: address) -> address:
    """ดูว่า delegate ให้ใคร"""
    return self.delegates_[_account]
```

---

## 2. On-chain Governor Contract {#on-chain}

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0

"""
@title Governor - On-chain governance contract
@notice Manages proposals, voting, and execution
@dev Based on Compound Governor Bravo pattern
"""

# ============================================================
# Interfaces
# ============================================================

interface IGovernanceToken:
    def getPriorVotes(_account: address, _blockNumber: uint256) -> uint256: view
    def totalSupply() -> uint256: view
    def getCurrentVotes(_account: address) -> uint256: view

interface ITimelock:
    def queueTransaction(
        _target: address,
        _value: uint256,
        _signature: String[50],
        _data: Bytes[1024],
        _eta: uint256
    ) -> bytes32: nonpayable
    
    def executeTransaction(
        _target: address,
        _value: uint256,
        _signature: String[50],
        _data: Bytes[1024],
        _eta: uint256
    ) -> Bytes[32]: nonpayable
    
    def cancelTransaction(
        _target: address,
        _value: uint256,
        _signature: String[50],
        _data: Bytes[1024],
        _eta: uint256
    ): nonpayable
    
    def delay() -> uint256: view

# ============================================================
# Events
# ============================================================

event ProposalCreated:
    id: indexed(uint256)
    proposer: indexed(address)
    targets: DynArray[address, 10]
    values: DynArray[uint256, 10]
    description: String[500]
    startBlock: uint256
    endBlock: uint256

event VoteCast:
    voter: indexed(address)
    proposalId: indexed(uint256)
    support: uint8
    votes: uint256
    reason: String[200]

event ProposalQueued:
    id: indexed(uint256)
    eta: uint256

event ProposalExecuted:
    id: indexed(uint256)

event ProposalCanceled:
    id: indexed(uint256)

# ============================================================
# Enums / Constants for Proposal States
# ============================================================

PROPOSAL_STATE_PENDING: constant(uint8) = 0
PROPOSAL_STATE_ACTIVE: constant(uint8) = 1
PROPOSAL_STATE_CANCELED: constant(uint8) = 2
PROPOSAL_STATE_DEFEATED: constant(uint8) = 3
PROPOSAL_STATE_SUCCEEDED: constant(uint8) = 4
PROPOSAL_STATE_QUEUED: constant(uint8) = 5
PROPOSAL_STATE_EXPIRED: constant(uint8) = 6
PROPOSAL_STATE_EXECUTED: constant(uint8) = 7

# ============================================================
# Structs
# ============================================================

struct Proposal:
    id: uint256
    proposer: address
    targets: DynArray[address, 10]
    values: DynArray[uint256, 10]
    signatures: DynArray[String[50], 10]
    calldatas: DynArray[Bytes[1024], 10]
    startBlock: uint256
    endBlock: uint256
    forVotes: uint256
    againstVotes: uint256
    abstainVotes: uint256
    canceled: bool
    executed: bool
    eta: uint256
    description: String[500]

struct Receipt:
    hasVoted: bool
    support: uint8
    votes: uint256

# ============================================================
# Constants
# ============================================================

MAX_PROPOSALS: constant(uint256) = 1000
QUORUM_DENOMINATOR: constant(uint256) = 100  # 4% = quorum

# ============================================================
# State Variables
# ============================================================

# Core
governanceToken: public(address)
timelock: public(address)
guardian: public(address)
admin: public(address)

# Governance parameters
votingDelay: public(uint256)          # blocks
votingPeriod: public(uint256)         # blocks
proposalThreshold: public(uint256)    # min votes to propose
quorumVotes: public(uint256)          # min votes for quorum

# Proposal storage
proposals: public(HashMap[uint256, Proposal])
proposalCount: public(uint256)
latestProposalIds: HashMap[address, uint256]
receipts: HashMap[uint256, HashMap[address, Receipt]]

# ============================================================
# Constructor
# ============================================================

@deploy
def __init__(
    _token: address,
    _timelock: address,
    _votingDelay: uint256,
    _votingPeriod: uint256,
    _proposalThreshold: uint256,
    _quorumVotes: uint256
):
    """
    @notice Initialize governor
    @param _token Governance token address
    @param _timelock Timelock contract address
    @param _votingDelay Delay before voting starts (blocks)
    @param _votingPeriod Voting period duration (blocks)
    @param _proposalThreshold Minimum votes to create proposal
    @param _quorumVotes Minimum votes for quorum
    """
    assert _token != empty(address), "Invalid token"
    assert _timelock != empty(address), "Invalid timelock"
    assert _votingDelay > 0, "Invalid delay"
    assert _votingPeriod > 0, "Invalid period"
    
    self.governanceToken = _token
    self.timelock = _timelock
    self.admin = msg.sender
    self.guardian = msg.sender
    
    self.votingDelay = _votingDelay
    self.votingPeriod = _votingPeriod
    self.proposalThreshold = _proposalThreshold
    self.quorumVotes = _quorumVotes

# ============================================================
# Proposal Functions
# ============================================================

@external
def propose(
    _targets: DynArray[address, 10],
    _values: DynArray[uint256, 10],
    _signatures: DynArray[String[50], 10],
    _calldatas: DynArray[Bytes[1024], 10],
    _description: String[500]
) -> uint256:
    """
    @notice สร้าง proposal ใหม่
    @return proposalId
    """
    token: IGovernanceToken = IGovernanceToken(self.governanceToken)
    
    # Check proposer has enough votes
    proposerVotes: uint256 = token.getPriorVotes(msg.sender, block.number - 1)
    assert proposerVotes >= self.proposalThreshold, "Insufficient votes to propose"
    
    # Check lengths match
    assert len(_targets) == len(_values), "Length mismatch"
    assert len(_targets) == len(_signatures), "Length mismatch"
    assert len(_targets) == len(_calldatas), "Length mismatch"
    assert len(_targets) > 0, "No actions"
    assert len(_targets) <= 10, "Too many actions"
    
    # Check proposer doesn't have pending proposal
    latestId: uint256 = self.latestProposalIds[msg.sender]
    if latestId > 0:
        state: uint8 = self._state(latestId)
        assert state != PROPOSAL_STATE_ACTIVE, "Active proposal exists"
        assert state != PROPOSAL_STATE_PENDING, "Pending proposal exists"
    
    # Create proposal
    proposalId: uint256 = self.proposalCount + 1
    self.proposalCount = proposalId
    
    startBlock: uint256 = block.number + self.votingDelay
    endBlock: uint256 = startBlock + self.votingPeriod
    
    self.proposals[proposalId] = Proposal({
        id: proposalId,
        proposer: msg.sender,
        targets: _targets,
        values: _values,
        signatures: _signatures,
        calldatas: _calldatas,
        startBlock: startBlock,
        endBlock: endBlock,
        forVotes: 0,
        againstVotes: 0,
        abstainVotes: 0,
        canceled: False,
        executed: False,
        eta: 0,
        description: _description
    })
    
    self.latestProposalIds[msg.sender] = proposalId
    
    log ProposalCreated(
        proposalId,
        msg.sender,
        _targets,
        _values,
        _description,
        startBlock,
        endBlock
    )
    
    return proposalId

@external
def castVote(_proposalId: uint256, _support: uint8):
    """
    @notice โหวต proposal
    @param _proposalId Proposal ID
    @param _support 0=Against, 1=For, 2=Abstain
    """
    self._castVote(msg.sender, _proposalId, _support, "")

@external
def castVoteWithReason(_proposalId: uint256, _support: uint8, _reason: String[200]):
    """โหวตพร้อมระบุเหตุผล"""
    self._castVote(msg.sender, _proposalId, _support, _reason)

@internal
def _castVote(
    _voter: address,
    _proposalId: uint256,
    _support: uint8,
    _reason: String[200]
):
    assert self._state(_proposalId) == PROPOSAL_STATE_ACTIVE, "Voting not active"
    assert _support <= 2, "Invalid vote"
    
    assert not self.receipts[_proposalId][_voter].hasVoted, "Already voted"
    
    proposal: Proposal = self.proposals[_proposalId]
    
    # Get voting power at start of voting
    token: IGovernanceToken = IGovernanceToken(self.governanceToken)
    votes: uint256 = token.getPriorVotes(_voter, proposal.startBlock)
    
    assert votes > 0, "No voting power"
    
    if _support == 0:
        self.proposals[_proposalId].againstVotes += votes
    elif _support == 1:
        self.proposals[_proposalId].forVotes += votes
    else:  # 2 = abstain
        self.proposals[_proposalId].abstainVotes += votes
    
    self.receipts[_proposalId][_voter] = Receipt({
        hasVoted: True,
        support: _support,
        votes: votes
    })
    
    log VoteCast(_voter, _proposalId, _support, votes, _reason)

@external
def queue(_proposalId: uint256):
    """
    @notice Queue proposal เข้า timelock หลัง voting ผ่าน
    """
    assert self._state(_proposalId) == PROPOSAL_STATE_SUCCEEDED, "Proposal not succeeded"
    
    proposal: Proposal = self.proposals[_proposalId]
    
    timelock: ITimelock = ITimelock(self.timelock)
    delay: uint256 = timelock.delay()
    eta: uint256 = block.timestamp + delay
    
    for i: uint256 in range(10):
        if i >= len(proposal.targets):
            break
        timelock.queueTransaction(
            proposal.targets[i],
            proposal.values[i],
            proposal.signatures[i],
            proposal.calldatas[i],
            eta
        )
    
    self.proposals[_proposalId].eta = eta
    
    log ProposalQueued(_proposalId, eta)

@external
def execute(_proposalId: uint256):
    """
    @notice Execute proposal หลัง timelock ผ่าน
    """
    assert self._state(_proposalId) == PROPOSAL_STATE_QUEUED, "Proposal not queued"
    
    proposal: Proposal = self.proposals[_proposalId]
    
    self.proposals[_proposalId].executed = True
    
    timelock: ITimelock = ITimelock(self.timelock)
    
    for i: uint256 in range(10):
        if i >= len(proposal.targets):
            break
        timelock.executeTransaction(
            proposal.targets[i],
            proposal.values[i],
            proposal.signatures[i],
            proposal.calldatas[i],
            proposal.eta
        )
    
    log ProposalExecuted(_proposalId)

@external
def cancel(_proposalId: uint256):
    """
    @notice ยกเลิก proposal
    @dev Proposer can cancel if they drop below threshold, guardian can always cancel
    """
    state: uint8 = self._state(_proposalId)
    assert state != PROPOSAL_STATE_EXECUTED, "Already executed"
    
    proposal: Proposal = self.proposals[_proposalId]
    
    # Guardian สามารถยกเลิกได้เสมอ
    if msg.sender != self.guardian:
        # Proposer ยกเลิกได้ถ้า votes ต่ำกว่า threshold
        token: IGovernanceToken = IGovernanceToken(self.governanceToken)
        assert msg.sender == proposal.proposer, "Not proposer"
        assert token.getPriorVotes(proposal.proposer, block.number - 1) < self.proposalThreshold, "Above threshold"
    
    self.proposals[_proposalId].canceled = True
    
    # Cancel from timelock if queued
    if proposal.eta > 0:
        timelock: ITimelock = ITimelock(self.timelock)
        for i: uint256 in range(10):
            if i >= len(proposal.targets):
                break
            timelock.cancelTransaction(
                proposal.targets[i],
                proposal.values[i],
                proposal.signatures[i],
                proposal.calldatas[i],
                proposal.eta
            )
    
    log ProposalCanceled(_proposalId)

# ============================================================
# State Functions
# ============================================================

@internal
@view
def _state(_proposalId: uint256) -> uint8:
    assert _proposalId > 0 and _proposalId <= self.proposalCount, "Invalid ID"
    
    proposal: Proposal = self.proposals[_proposalId]
    
    if proposal.canceled:
        return PROPOSAL_STATE_CANCELED
    elif block.number <= proposal.startBlock:
        return PROPOSAL_STATE_PENDING
    elif block.number <= proposal.endBlock:
        return PROPOSAL_STATE_ACTIVE
    elif proposal.forVotes <= proposal.againstVotes or proposal.forVotes < self.quorumVotes:
        return PROPOSAL_STATE_DEFEATED
    elif proposal.eta == 0:
        return PROPOSAL_STATE_SUCCEEDED
    elif proposal.executed:
        return PROPOSAL_STATE_EXECUTED
    elif block.timestamp >= proposal.eta + 14 * 24 * 3600:  # 14 day grace period
        return PROPOSAL_STATE_EXPIRED
    else:
        return PROPOSAL_STATE_QUEUED

@external
@view
def state(_proposalId: uint256) -> uint8:
    """ดูสถานะ proposal"""
    return self._state(_proposalId)

@external
@view
def getProposal(_proposalId: uint256) -> Proposal:
    """ดูรายละเอียด proposal"""
    return self.proposals[_proposalId]

@external
@view
def getReceipt(_proposalId: uint256, _voter: address) -> Receipt:
    """ดูผลการโหวตของ voter"""
    return self.receipts[_proposalId][_voter]
```

---

## 3. Timelock Management {#timelock}

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0

"""
@title Timelock - Timelock controller for governance
@notice ทุก governance action ต้องผ่าน timelock delay
@dev Based on Compound Timelock
"""

# ============================================================
# Events
# ============================================================

event NewAdmin:
    newAdmin: indexed(address)

event NewPendingAdmin:
    newPendingAdmin: indexed(address)

event NewDelay:
    newDelay: uint256

event CancelTransaction:
    txHash: indexed(bytes32)
    target: indexed(address)
    value: uint256
    eta: uint256

event ExecuteTransaction:
    txHash: indexed(bytes32)
    target: indexed(address)
    value: uint256
    eta: uint256

event QueueTransaction:
    txHash: indexed(bytes32)
    target: indexed(address)
    value: uint256
    eta: uint256

# ============================================================
# Constants
# ============================================================

GRACE_PERIOD: constant(uint256) = 14 * 24 * 3600  # 14 days
MINIMUM_DELAY: constant(uint256) = 2 * 24 * 3600   # 2 days
MAXIMUM_DELAY: constant(uint256) = 30 * 24 * 3600  # 30 days

# ============================================================
# State Variables
# ============================================================

admin: public(address)
pendingAdmin: public(address)
delay: public(uint256)

queuedTransactions: public(HashMap[bytes32, bool])

# ============================================================
# Constructor
# ============================================================

@deploy
def __init__(_admin: address, _delay: uint256):
    """
    @notice Initialize timelock
    @param _admin Admin address (should be governance contract)
    @param _delay Delay in seconds
    """
    assert _admin != empty(address), "Invalid admin"
    assert _delay >= MINIMUM_DELAY, "Delay too low"
    assert _delay <= MAXIMUM_DELAY, "Delay too high"
    
    self.admin = _admin
    self.delay = _delay

# ============================================================
# Admin Functions
# ============================================================

@external
def setDelay(_delay: uint256):
    """ปรับ delay"""
    assert msg.sender == self, "Must be self"  # ต้องผ่าน timelock เอง
    assert _delay >= MINIMUM_DELAY, "Delay too low"
    assert _delay <= MAXIMUM_DELAY, "Delay too high"
    
    self.delay = _delay
    log NewDelay(_delay)

@external
def acceptAdmin():
    """รับ admin role"""
    assert msg.sender == self.pendingAdmin, "Not pending admin"
    
    self.admin = self.pendingAdmin
    self.pendingAdmin = empty(address)
    
    log NewAdmin(self.admin)

@external
def setPendingAdmin(_pendingAdmin: address):
    """ตั้ง pending admin"""
    assert msg.sender == self, "Must be self"
    
    self.pendingAdmin = _pendingAdmin
    log NewPendingAdmin(_pendingAdmin)

# ============================================================
# Transaction Functions
# ============================================================

@external
def queueTransaction(
    _target: address,
    _value: uint256,
    _signature: String[50],
    _data: Bytes[1024],
    _eta: uint256
) -> bytes32:
    """
    @notice Queue transaction
    @param _eta Earliest execution time (unix timestamp)
    """
    assert msg.sender == self.admin, "Not admin"
    assert _eta >= block.timestamp + self.delay, "ETA too soon"
    
    txHash: bytes32 = keccak256(
        concat(
            convert(_target, bytes32),
            convert(_value, bytes32),
            convert(keccak256(convert(_signature, Bytes[50])), bytes32),
            keccak256(_data),
            convert(_eta, bytes32)
        )
    )
    
    self.queuedTransactions[txHash] = True
    
    log QueueTransaction(txHash, _target, _value, _eta)
    
    return txHash

@external
def cancelTransaction(
    _target: address,
    _value: uint256,
    _signature: String[50],
    _data: Bytes[1024],
    _eta: uint256
):
    """ยกเลิก transaction ที่ queue ไว้"""
    assert msg.sender == self.admin, "Not admin"
    
    txHash: bytes32 = keccak256(
        concat(
            convert(_target, bytes32),
            convert(_value, bytes32),
            convert(keccak256(convert(_signature, Bytes[50])), bytes32),
            keccak256(_data),
            convert(_eta, bytes32)
        )
    )
    
    self.queuedTransactions[txHash] = False
    
    log CancelTransaction(txHash, _target, _value, _eta)

@external
def executeTransaction(
    _target: address,
    _value: uint256,
    _signature: String[50],
    _data: Bytes[1024],
    _eta: uint256
) -> Bytes[32]:
    """
    @notice Execute transaction หลัง timelock ผ่าน
    """
    assert msg.sender == self.admin, "Not admin"
    
    txHash: bytes32 = keccak256(
        concat(
            convert(_target, bytes32),
            convert(_value, bytes32),
            convert(keccak256(convert(_signature, Bytes[50])), bytes32),
            keccak256(_data),
            convert(_eta, bytes32)
        )
    )
    
    assert self.queuedTransactions[txHash], "Not queued"
    assert block.timestamp >= _eta, "Timelock not passed"
    assert block.timestamp <= _eta + GRACE_PERIOD, "Transaction expired"
    
    self.queuedTransactions[txHash] = False
    
    callData: Bytes[1024] = _data
    
    success: bool = False
    response: Bytes[32] = b""
    success, response = raw_call(
        _target,
        callData,
        value=_value,
        max_outsize=32,
        revert_on_failure=False
    )
    
    assert success, "Transaction execution failed"
    
    log ExecuteTransaction(txHash, _target, _value, _eta)
    
    return response

@external
@payable
def receive():
    pass
```

---

## 4. GovernanceHelper Contract {#governance-helper}

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0

"""
@title GovernanceHelper - Helper contract สำหรับ governance operations
@notice ช่วยในการสร้าง proposals, ตรวจสอบสถานะ และ manage governance
@dev รวม utilities ที่ใช้บ่อยใน governance operations
"""

# ============================================================
# Interfaces
# ============================================================

interface IGovernor:
    def propose(
        _targets: DynArray[address, 10],
        _values: DynArray[uint256, 10],
        _signatures: DynArray[String[50], 10],
        _calldatas: DynArray[Bytes[1024], 10],
        _description: String[500]
    ) -> uint256: nonpayable
    
    def castVote(_proposalId: uint256, _support: uint8): nonpayable
    def castVoteWithReason(_proposalId: uint256, _support: uint8, _reason: String[200]): nonpayable
    def queue(_proposalId: uint256): nonpayable
    def execute(_proposalId: uint256): nonpayable
    def cancel(_proposalId: uint256): nonpayable
    def state(_proposalId: uint256) -> uint8: view
    def votingDelay() -> uint256: view
    def votingPeriod() -> uint256: view
    def proposalThreshold() -> uint256: view
    def quorumVotes() -> uint256: view
    def proposalCount() -> uint256: view

interface IGovernanceToken:
    def getCurrentVotes(_account: address) -> uint256: view
    def getPriorVotes(_account: address, _blockNumber: uint256) -> uint256: view
    def totalSupply() -> uint256: view
    def delegate(_delegatee: address): nonpayable

# ============================================================
# Events
# ============================================================

event ProposalTracked:
    proposalId: indexed(uint256)
    title: String[100]
    tracker: indexed(address)
    timestamp: uint256

event VoteReminder:
    proposalId: indexed(uint256)
    voter: indexed(address)
    endBlock: uint256

event BatchVoteCast:
    voter: indexed(address)
    proposalCount: uint256
    timestamp: uint256

event DelegationSetup:
    delegator: indexed(address)
    delegatee: indexed(address)
    power: uint256

# ============================================================
# Structs
# ============================================================

struct ProposalSummary:
    id: uint256
    title: String[100]
    state: uint8
    forVotes: uint256
    againstVotes: uint256
    abstainVotes: uint256
    endBlock: uint256
    quorumReached: bool

struct VoteRecord:
    proposalId: uint256
    voter: address
    support: uint8
    votes: uint256
    timestamp: uint256
    reason: String[200]

struct DelegationInfo:
    delegator: address
    delegatee: address
    power: uint256
    timestamp: uint256

# ============================================================
# Constants
# ============================================================

MAX_PROPOSALS_TRACK: constant(uint256) = 100
MAX_VOTE_HISTORY: constant(uint256) = 500

PROPOSAL_PENDING: constant(uint8) = 0
PROPOSAL_ACTIVE: constant(uint8) = 1
PROPOSAL_DEFEATED: constant(uint8) = 3
PROPOSAL_SUCCEEDED: constant(uint8) = 4

# ============================================================
# State Variables
# ============================================================

owner: public(address)
governor: public(address)
governanceToken: public(address)

# Proposal tracking
trackedProposals: public(DynArray[uint256, MAX_PROPOSALS_TRACK])
proposalTitles: public(HashMap[uint256, String[100]])
trackedAt: public(HashMap[uint256, uint256])

# Vote history
voteHistory: public(DynArray[VoteRecord, MAX_VOTE_HISTORY])
userVoteCount: public(HashMap[address, uint256])

# Delegation history
delegationHistory: public(DynArray[DelegationInfo, 1000])

# Statistics
totalProposalsCreated: public(uint256)
totalVotesCast: public(uint256)
totalDelegations: public(uint256)

# ============================================================
# Constructor
# ============================================================

@deploy
def __init__(_governor: address, _token: address):
    """
    @notice Initialize governance helper
    @param _governor Governor contract address
    @param _token Governance token address
    """
    assert _governor != empty(address), "Invalid governor"
    assert _token != empty(address), "Invalid token"
    
    self.owner = msg.sender
    self.governor = _governor
    self.governanceToken = _token

# ============================================================
# Proposal Tracking
# ============================================================

@external
def trackProposal(_proposalId: uint256, _title: String[100]):
    """
    @notice Track proposal สำหรับ monitoring
    """
    assert len(self.trackedProposals) < MAX_PROPOSALS_TRACK, "Too many tracked"
    
    self.trackedProposals.append(_proposalId)
    self.proposalTitles[_proposalId] = _title
    self.trackedAt[_proposalId] = block.timestamp
    
    log ProposalTracked(_proposalId, _title, msg.sender, block.timestamp)

@external
def castVoteAndTrack(
    _proposalId: uint256,
    _support: uint8,
    _reason: String[200]
):
    """
    @notice โหวตและบันทึกในประวัติ
    @param _proposalId Proposal ID
    @param _support 0=Against, 1=For, 2=Abstain
    @param _reason เหตุผลในการโหวต
    """
    # Cast vote
    IGovernor(self.governor).castVoteWithReason(_proposalId, _support, _reason)
    
    # Track vote
    token: IGovernanceToken = IGovernanceToken(self.governanceToken)
    votes: uint256 = token.getCurrentVotes(msg.sender)
    
    record: VoteRecord = VoteRecord({
        proposalId: _proposalId,
        voter: msg.sender,
        support: _support,
        votes: votes,
        timestamp: block.timestamp,
        reason: _reason
    })
    
    if len(self.voteHistory) < MAX_VOTE_HISTORY:
        self.voteHistory.append(record)
    
    self.userVoteCount[msg.sender] += 1
    self.totalVotesCast += 1
    
    log BatchVoteCast(msg.sender, 1, block.timestamp)

# ============================================================
# Delegation Management
# ============================================================

@external
def setupDelegation(_delegatee: address):
    """
    @notice ตั้งค่า delegation และบันทึกประวัติ
    """
    token: IGovernanceToken = IGovernanceToken(self.governanceToken)
    
    # Get current power before delegation
    power: uint256 = token.getCurrentVotes(msg.sender)
    
    # Delegate
    token.delegate(_delegatee)
    
    # Record delegation
    info: DelegationInfo = DelegationInfo({
        delegator: msg.sender,
        delegatee: _delegatee,
        power: power,
        timestamp: block.timestamp
    })
    
    if len(self.delegationHistory) < 1000:
        self.delegationHistory.append(info)
    
    self.totalDelegations += 1
    
    log DelegationSetup(msg.sender, _delegatee, power)

# ============================================================
# Analytics Functions
# ============================================================

@external
@view
def getGovernanceStats() -> (uint256, uint256, uint256, uint256):
    """
    @notice ดู governance statistics
    @return (totalProposals, activeProposals, totalVotes, participationRate)
    """
    governor: IGovernor = IGovernor(self.governor)
    token: IGovernanceToken = IGovernanceToken(self.governanceToken)
    
    totalProposals: uint256 = governor.proposalCount()
    
    activeProposals: uint256 = 0
    for pid: uint256 in self.trackedProposals:
        if governor.state(pid) == PROPOSAL_ACTIVE:
            activeProposals += 1
    
    totalVotes: uint256 = self.totalVotesCast
    
    # Participation rate (basis points)
    totalSupply: uint256 = token.totalSupply()
    participationRate: uint256 = 0
    if totalSupply > 0 and totalVotes > 0:
        # Simplified calculation
        participationRate = (totalVotes * 10000) / (totalSupply / 10**18)
    
    return (totalProposals, activeProposals, totalVotes, participationRate)

@external
@view
def getVoterActivity(_voter: address) -> (uint256, uint256):
    """
    @notice ดูกิจกรรม voting ของ address
    @return (voteCount, currentPower)
    """
    token: IGovernanceToken = IGovernanceToken(self.governanceToken)
    
    voteCount: uint256 = self.userVoteCount[_voter]
    currentPower: uint256 = token.getCurrentVotes(_voter)
    
    return (voteCount, currentPower)

@external
@view
def getActiveProposals() -> DynArray[uint256, MAX_PROPOSALS_TRACK]:
    """รายการ proposals ที่กำลัง active"""
    governor: IGovernor = IGovernor(self.governor)
    
    active: DynArray[uint256, MAX_PROPOSALS_TRACK] = []
    
    for pid: uint256 in self.trackedProposals:
        if governor.state(pid) == PROPOSAL_ACTIVE:
            active.append(pid)
    
    return active

@external
@view
def getTrackedProposals() -> DynArray[uint256, MAX_PROPOSALS_TRACK]:
    """รายการ proposals ที่ track"""
    return self.trackedProposals

@external
@view
def getVoteHistory() -> DynArray[VoteRecord, MAX_VOTE_HISTORY]:
    """ประวัติการโหวต"""
    return self.voteHistory
```

---

## 5. Emergency Governance {#emergency-governance}

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0

"""
@title EmergencyGovernance - Governance สำหรับสถานการณ์ฉุกเฉิน
@notice Fast-track governance actions ที่ไม่ต้องรอ normal governance
@dev ใช้สำหรับ critical security fixes เท่านั้น
"""

# ============================================================
# Events
# ============================================================

event EmergencyActionProposed:
    id: indexed(uint256)
    proposer: indexed(address)
    actionType: String[50]
    timestamp: uint256

event EmergencyActionApproved:
    id: indexed(uint256)
    approver: indexed(address)
    currentApprovals: uint256

event EmergencyActionExecuted:
    id: indexed(uint256)
    executor: indexed(address)
    timestamp: uint256

event EmergencyCouncilUpdated:
    member: indexed(address)
    status: bool

# ============================================================
# Structs
# ============================================================

struct EmergencyAction:
    id: uint256
    proposer: address
    target: address
    calldata_: Bytes[1024]
    description: String[200]
    actionType: String[50]
    proposedAt: uint256
    executed: bool
    canceled: bool
    approvals: uint256
    expiresAt: uint256

# ============================================================
# Constants
# ============================================================

MAX_COUNCIL: constant(uint256) = 9
REQUIRED_APPROVALS: constant(uint256) = 5  # 5-of-9 multisig
EMERGENCY_EXPIRY: constant(uint256) = 3 * 3600  # 3 hours
MAX_ACTIONS: constant(uint256) = 50

# ============================================================
# State Variables
# ============================================================

owner: public(address)

# Emergency council members
councilMembers: public(DynArray[address, MAX_COUNCIL])
isCouncilMember: public(HashMap[address, bool])

# Emergency actions
actions: public(HashMap[uint256, EmergencyAction])
actionCount: public(uint256)
approvals: public(HashMap[uint256, HashMap[address, bool]])

# Protected contracts
protectedContracts: public(DynArray[address, 20])

# Veto power (governance can veto emergency actions)
governanceContract: public(address)

# ============================================================
# Constructor
# ============================================================

@deploy
def __init__(
    _councilMembers: DynArray[address, MAX_COUNCIL],
    _governance: address
):
    """
    @notice Initialize emergency governance
    @param _councilMembers Emergency council members
    @param _governance Main governance contract (has veto power)
    """
    assert len(_councilMembers) >= 5, "Need at least 5 council members"
    assert _governance != empty(address), "Invalid governance"
    
    self.owner = msg.sender
    self.governanceContract = _governance
    
    for member: address in _councilMembers:
        assert member != empty(address), "Invalid member"
        self.councilMembers.append(member)
        self.isCouncilMember[member] = True

# ============================================================
# Emergency Action Functions
# ============================================================

@external
def proposeEmergencyAction(
    _target: address,
    _calldata: Bytes[1024],
    _description: String[200],
    _actionType: String[50]
) -> uint256:
    """
    @notice เสนอ emergency action
    @dev ต้องเป็น council member และต้องผ่านสถานการณ์จริง
    """
    assert self.isCouncilMember[msg.sender], "Not council member"
    assert _target != empty(address), "Invalid target"
    
    actionId: uint256 = self.actionCount
    self.actionCount += 1
    
    self.actions[actionId] = EmergencyAction({
        id: actionId,
        proposer: msg.sender,
        target: _target,
        calldata_: _calldata,
        description: _description,
        actionType: _actionType,
        proposedAt: block.timestamp,
        executed: False,
        canceled: False,
        approvals: 0,
        expiresAt: block.timestamp + EMERGENCY_EXPIRY
    })
    
    log EmergencyActionProposed(actionId, msg.sender, _actionType, block.timestamp)
    
    # Auto-approve by proposer
    self._approve(actionId)
    
    return actionId

@external
def approveEmergencyAction(_actionId: uint256):
    """อนุมัติ emergency action"""
    assert self.isCouncilMember[msg.sender], "Not council member"
    assert _actionId < self.actionCount, "Invalid action"
    
    action: EmergencyAction = self.actions[_actionId]
    assert not action.executed, "Already executed"
    assert not action.canceled, "Canceled"
    assert block.timestamp < action.expiresAt, "Expired"
    assert not self.approvals[_actionId][msg.sender], "Already approved"
    
    self._approve(_actionId)

@internal
def _approve(_actionId: uint256):
    self.approvals[_actionId][msg.sender] = True
    self.actions[_actionId].approvals += 1
    
    log EmergencyActionApproved(
        _actionId,
        msg.sender,
        self.actions[_actionId].approvals
    )

@external
def executeEmergencyAction(_actionId: uint256):
    """
    @notice Execute emergency action เมื่อได้รับ approvals ครบ
    """
    assert self.isCouncilMember[msg.sender], "Not council member"
    
    action: EmergencyAction = self.actions[_actionId]
    assert not action.executed, "Already executed"
    assert not action.canceled, "Canceled"
    assert block.timestamp < action.expiresAt, "Expired"
    assert action.approvals >= REQUIRED_APPROVALS, "Insufficient approvals"
    
    self.actions[_actionId].executed = True
    
    success: bool = False
    response: Bytes[32] = b""
    success, response = raw_call(
        action.target,
        action.calldata_,
        max_outsize=32,
        revert_on_failure=False
    )
    
    assert success, "Execution failed"
    
    log EmergencyActionExecuted(_actionId, msg.sender, block.timestamp)

@external
def cancelEmergencyAction(_actionId: uint256):
    """ยกเลิก emergency action (เฉพาะ proposer หรือ governance veto)"""
    action: EmergencyAction = self.actions[_actionId]
    
    assert (
        msg.sender == action.proposer or 
        msg.sender == self.governanceContract or
        msg.sender == self.owner
    ), "Not authorized"
    assert not action.executed, "Already executed"
    
    self.actions[_actionId].canceled = True

# ============================================================
# Council Management
# ============================================================

@external
def updateCouncilMember(_member: address, _status: bool):
    """อัปเดต council member (ผ่าน governance เท่านั้น)"""
    assert msg.sender == self.governanceContract or msg.sender == self.owner, "Not authorized"
    
    if _status and not self.isCouncilMember[_member]:
        assert len(self.councilMembers) < MAX_COUNCIL, "Too many members"
        self.councilMembers.append(_member)
        self.isCouncilMember[_member] = True
    elif not _status and self.isCouncilMember[_member]:
        self.isCouncilMember[_member] = False
        newMembers: DynArray[address, MAX_COUNCIL] = []
        for m: address in self.councilMembers:
            if m != _member:
                newMembers.append(m)
        self.councilMembers = newMembers
    
    log EmergencyCouncilUpdated(_member, _status)

# ============================================================
# View Functions
# ============================================================

@external
@view
def getAction(_actionId: uint256) -> EmergencyAction:
    """ดูรายละเอียด emergency action"""
    return self.actions[_actionId]

@external
@view
def getCouncilMembers() -> DynArray[address, MAX_COUNCIL]:
    """รายชื่อ council members"""
    return self.councilMembers

@external
@view
def hasApproved(_actionId: uint256, _member: address) -> bool:
    """ตรวจสอบว่า member อนุมัติแล้วหรือไม่"""
    return self.approvals[_actionId][_member]

@external
@view
def getPendingActions() -> DynArray[uint256, MAX_ACTIONS]:
    """รายการ actions ที่รอ execution"""
    pending: DynArray[uint256, MAX_ACTIONS] = []
    
    for i: uint256 in range(MAX_ACTIONS):
        if i >= self.actionCount:
            break
        action: EmergencyAction = self.actions[i]
        if not action.executed and not action.canceled and block.timestamp < action.expiresAt:
            pending.append(i)
    
    return pending
```

---

## สรุป: Governance Best Practices

### การออกแบบ Governance ที่ดี

1. **Progressive Decentralization** - เริ่มจาก centralized แล้วค่อยๆ กระจาย
2. **Timelock สำคัญมาก** - ให้เวลา community react กับ malicious proposals
3. **Quorum ที่เหมาะสม** - ไม่สูงเกินไป (ยากผ่าน) ไม่ต่ำเกินไป (ถูก manipulate)
4. **Emergency Mechanisms** - ต้องมีแต่ต้อง restrict การใช้งาน
5. **Transparency** - ทุก action ต้องมี on-chain record

### Governance Attack Vectors

- **Flash Loan Attacks** - ยืม tokens มา vote แล้วคืน
  - Solution: Snapshot voting (ดูยอดที่ block ก่อนหน้า)
  
- **Governance Capture** - คนรวยซื้อ votes
  - Solution: Time-weighted voting, quadratic voting
  
- **Low Participation** - Quorum ไม่ถึง
  - Solution: Vote delegation, ลด quorum requirements

> **คำแนะนำ**: เริ่มต้นด้วย governance ที่ conservative (high quorum, long timelock) แล้วค่อยปรับตาม participation rate ที่แท้จริง
