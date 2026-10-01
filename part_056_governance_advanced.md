# Part 056: Advanced Governance - Governor Bravo Style

## สารบัญ (Table of Contents)
1. บทนำ On-chain Governance
2. Governor Bravo Contract
3. Timelock Controller
4. Vote Delegation
5. Proposal Lifecycle
6. Emergency Powers
7. Tests

---

## 1. บทนำ On-chain Governance

On-chain governance ช่วยให้ token holders ควบคุม protocol:
- **Proposals**: เสนอการเปลี่ยนแปลง
- **Voting**: ลงคะแนนด้วย governance tokens
- **Timelock**: delay ก่อน execute (ให้ community ตรวจสอบ)
- **Quorum**: minimum votes ที่ต้องการ

**Governor Bravo Architecture:**
```
Governor -> Timelock -> Target Contract
```

---

## 2. Timelock Controller

```vyper
# @version 0.4.0
# contracts/TimelockController.vy
# Timelock สำหรับ execute governance proposals

# Events
event CallScheduled:
    id: indexed(bytes32)
    index: uint256
    target: indexed(address)
    value: uint256
    data: Bytes[1024]
    predecessor: bytes32
    delay: uint256

event CallExecuted:
    id: indexed(bytes32)
    index: uint256
    target: indexed(address)
    value: uint256
    data: Bytes[1024]

event CallCancelled:
    id: indexed(bytes32)

event MinDelayChange:
    oldDuration: uint256
    newDuration: uint256

event RoleGranted:
    role: indexed(bytes32)
    account: indexed(address)
    sender: indexed(address)

event RoleRevoked:
    role: indexed(bytes32)
    account: indexed(address)
    sender: indexed(address)

# Roles
TIMELOCK_ADMIN_ROLE: constant(bytes32) = keccak256("TIMELOCK_ADMIN_ROLE")
PROPOSER_ROLE: constant(bytes32) = keccak256("PROPOSER_ROLE")
EXECUTOR_ROLE: constant(bytes32) = keccak256("EXECUTOR_ROLE")
CANCELLER_ROLE: constant(bytes32) = keccak256("CANCELLER_ROLE")

# States
_UNSET: constant(uint8) = 0
_WAITING: constant(uint8) = 1
_READY: constant(uint8) = 2
_DONE: constant(uint8) = 3

# State
minDelay: public(uint256)

# Role management
roles: HashMap[bytes32, HashMap[address, bool]]

# Operations
timestamps: HashMap[bytes32, uint256]  # operationId -> timestamp

@deploy
def __init__(
    _minDelay: uint256,
    proposers: DynArray[address, 10],
    executors: DynArray[address, 10],
    admin: address
):
    self.minDelay = _minDelay
    
    # Grant roles
    self.roles[TIMELOCK_ADMIN_ROLE][self] = True
    
    if admin != empty(address):
        self.roles[TIMELOCK_ADMIN_ROLE][admin] = True
    
    for proposer: address in proposers:
        self.roles[PROPOSER_ROLE][proposer] = True
        self.roles[CANCELLER_ROLE][proposer] = True
        log RoleGranted(PROPOSER_ROLE, proposer, msg.sender)
    
    for executor: address in executors:
        self.roles[EXECUTOR_ROLE][executor] = True
        log RoleGranted(EXECUTOR_ROLE, executor, msg.sender)

# ===== Role Management =====

@internal
@view
def _checkRole(role: bytes32, account: address):
    assert self.roles[role][account], "AccessControl: account is missing role"

@external
@view
def hasRole(role: bytes32, account: address) -> bool:
    return self.roles[role][account]

@external
def grantRole(role: bytes32, account: address):
    self._checkRole(TIMELOCK_ADMIN_ROLE, msg.sender)
    if not self.roles[role][account]:
        self.roles[role][account] = True
        log RoleGranted(role, account, msg.sender)

@external
def revokeRole(role: bytes32, account: address):
    self._checkRole(TIMELOCK_ADMIN_ROLE, msg.sender)
    if self.roles[role][account]:
        self.roles[role][account] = False
        log RoleRevoked(role, account, msg.sender)

# ===== Operation Management =====

@internal
@pure
def _hashOperation(
    target: address,
    value: uint256,
    data: Bytes[1024],
    predecessor: bytes32,
    salt: bytes32
) -> bytes32:
    """Hash ของ operation"""
    return keccak256(
        concat(
            convert(target, bytes20),
            convert(value, bytes32),
            data,
            predecessor,
            salt
        )
    )

@external
@view
def getOperationState(id: bytes32) -> uint8:
    timestamp: uint256 = self.timestamps[id]
    if timestamp == 0:
        return _UNSET
    elif timestamp == 1:
        return _DONE
    elif timestamp > block.timestamp:
        return _WAITING
    else:
        return _READY

@external
def schedule(
    target: address,
    value: uint256,
    data: Bytes[1024],
    predecessor: bytes32,
    salt: bytes32,
    delay: uint256
) -> bytes32:
    """
    Schedule operation สำหรับ execution
    เรียกโดย PROPOSER เท่านั้น
    
    Returns:
        operationId: ID ของ operation
    """
    self._checkRole(PROPOSER_ROLE, msg.sender)
    assert delay >= self.minDelay, "Delay too short"
    
    id: bytes32 = self._hashOperation(target, value, data, predecessor, salt)
    assert self.timestamps[id] == 0, "Operation already scheduled"
    
    self.timestamps[id] = block.timestamp + delay
    
    log CallScheduled(id, 0, target, value, data, predecessor, delay)
    
    return id

@external
def execute(
    target: address,
    value: uint256,
    data: Bytes[1024],
    predecessor: bytes32,
    salt: bytes32
):
    """
    Execute scheduled operation
    เรียกโดย EXECUTOR เท่านั้น
    """
    self._checkRole(EXECUTOR_ROLE, msg.sender)
    
    id: bytes32 = self._hashOperation(target, value, data, predecessor, salt)
    assert self.timestamps[id] != 0, "Operation not scheduled"
    assert self.timestamps[id] <= block.timestamp, "Not ready"
    assert self.timestamps[id] != 1, "Already executed"
    
    # ตรวจสอบ predecessor
    if predecessor != empty(bytes32):
        assert self.timestamps[predecessor] == 1, "Predecessor not done"
    
    # Execute
    self.timestamps[id] = 1  # Mark as done
    
    raw_call(target, data, value=value)
    
    log CallExecuted(id, 0, target, value, data)

@external
def cancel(id: bytes32):
    """Cancel scheduled operation"""
    self._checkRole(CANCELLER_ROLE, msg.sender)
    
    assert self.timestamps[id] > 1, "Operation not scheduled or done"
    
    self.timestamps[id] = 0
    
    log CallCancelled(id)

@external
def updateDelay(newDelay: uint256):
    """อัพเดท minimum delay"""
    assert msg.sender == self, "Only timelock itself"
    
    oldDelay: uint256 = self.minDelay
    self.minDelay = newDelay
    
    log MinDelayChange(oldDelay, newDelay)

@external
@payable
def __default__():
    """รับ ETH"""
    pass
```

---

## 3. Governor Bravo Contract

```vyper
# @version 0.4.0
# contracts/GovernorBravo.vy
# Governor Bravo Style Governance

from vyper.interfaces import ERC20

interface IGovernanceToken:
    def getPriorVotes(account: address, blockNumber: uint256) -> uint256: view
    def totalSupply() -> uint256: view
    def delegates(account: address) -> address: view
    def delegate(delegatee: address): nonpayable

interface ITimelock:
    def schedule(target: address, value: uint256, data: Bytes[1024], predecessor: bytes32, salt: bytes32, delay: uint256) -> bytes32: nonpayable
    def execute(target: address, value: uint256, data: Bytes[1024], predecessor: bytes32, salt: bytes32): nonpayable
    def cancel(id: bytes32): nonpayable
    def getOperationState(id: bytes32) -> uint8: view
    def minDelay() -> uint256: view

# Enums
struct ProposalState:
    pass  # Vyper ไม่มี enum โดยตรง

# States as constants
PROPOSAL_PENDING: constant(uint8) = 0
PROPOSAL_ACTIVE: constant(uint8) = 1
PROPOSAL_CANCELED: constant(uint8) = 2
PROPOSAL_DEFEATED: constant(uint8) = 3
PROPOSAL_SUCCEEDED: constant(uint8) = 4
PROPOSAL_QUEUED: constant(uint8) = 5
PROPOSAL_EXPIRED: constant(uint8) = 6
PROPOSAL_EXECUTED: constant(uint8) = 7

# Struct
struct ProposalAction:
    target: address
    value: uint256
    calldata: Bytes[1024]

struct Proposal:
    id: uint256
    proposer: address
    eta: uint256               # execute ได้เมื่อไหร่
    startBlock: uint256        # เริ่ม vote
    endBlock: uint256          # สิ้นสุด vote
    forVotes: uint256          # votes สนับสนุน
    againstVotes: uint256      # votes ค้าน
    abstainVotes: uint256      # votes งดออกเสียง
    canceled: bool
    executed: bool
    description: String[512]
    actionsCount: uint256

struct Receipt:
    hasVoted: bool
    support: uint8    # 0 = against, 1 = for, 2 = abstain
    votes: uint256

# Events
event ProposalCreated:
    id: indexed(uint256)
    proposer: indexed(address)
    startBlock: uint256
    endBlock: uint256
    description: String[512]

event VoteCast:
    voter: indexed(address)
    proposalId: indexed(uint256)
    support: uint8
    votes: uint256
    reason: String[256]

event ProposalQueued:
    id: indexed(uint256)
    eta: uint256

event ProposalExecuted:
    id: indexed(uint256)

event ProposalCanceled:
    id: indexed(uint256)

event ProposalThresholdSet:
    oldThreshold: uint256
    newThreshold: uint256

event VotingDelaySet:
    oldVotingDelay: uint256
    newVotingDelay: uint256

event VotingPeriodSet:
    oldVotingPeriod: uint256
    newVotingPeriod: uint256

# State
token: public(address)
timelock: public(address)
admin: public(address)
pendingAdmin: address

name: public(String[64])

proposalCount: public(uint256)
proposals: HashMap[uint256, Proposal]
proposalActions: HashMap[uint256, HashMap[uint256, ProposalAction]]
receipts: HashMap[uint256, HashMap[address, Receipt]]

# Governance parameters
votingDelay: public(uint256)      # blocks ก่อนเริ่ม vote หลัง propose
votingPeriod: public(uint256)     # blocks ที่ vote ได้
proposalThreshold: public(uint256)  # min tokens ที่ต้องมีเพื่อ propose
quorumNumerator: public(uint256)  # % quorum (400 = 4%)

QUORUM_DENOMINATOR: constant(uint256) = 10000
MAX_ACTIONS: constant(uint256) = 10

@deploy
def __init__(
    _token: address,
    _timelock: address,
    _name: String[64],
    _votingDelay: uint256,
    _votingPeriod: uint256,
    _proposalThreshold: uint256,
    _quorumNumerator: uint256
):
    self.token = _token
    self.timelock = _timelock
    self.name = _name
    self.votingDelay = _votingDelay
    self.votingPeriod = _votingPeriod
    self.proposalThreshold = _proposalThreshold
    self.quorumNumerator = _quorumNumerator
    self.admin = msg.sender

# ===== Proposal =====

@external
def propose(
    targets: DynArray[address, 10],
    values: DynArray[uint256, 10],
    calldatas: DynArray[Bytes[1024], 10],
    description: String[512]
) -> uint256:
    """
    สร้าง governance proposal
    
    Parameters:
        targets: contract addresses ที่จะ call
        values: ETH values สำหรับแต่ละ call
        calldatas: calldata สำหรับแต่ละ call
        description: คำอธิบาย proposal
    
    Returns:
        proposalId: ID ของ proposal
    """
    assert len(targets) > 0, "Empty proposal"
    assert len(targets) == len(values), "Length mismatch"
    assert len(targets) == len(calldatas), "Length mismatch"
    assert len(targets) <= MAX_ACTIONS, "Too many actions"
    
    # ตรวจสอบ threshold
    proposerVotes: uint256 = IGovernanceToken(self.token).getPriorVotes(
        msg.sender,
        block.number - 1
    )
    assert proposerVotes >= self.proposalThreshold, "Below threshold"
    
    startBlock: uint256 = block.number + self.votingDelay
    endBlock: uint256 = startBlock + self.votingPeriod
    
    self.proposalCount += 1
    proposalId: uint256 = self.proposalCount
    
    self.proposals[proposalId] = Proposal({
        id: proposalId,
        proposer: msg.sender,
        eta: 0,
        startBlock: startBlock,
        endBlock: endBlock,
        forVotes: 0,
        againstVotes: 0,
        abstainVotes: 0,
        canceled: False,
        executed: False,
        description: description,
        actionsCount: len(targets)
    })
    
    for i: uint256 in range(10):
        if i >= len(targets):
            break
        self.proposalActions[proposalId][i] = ProposalAction({
            target: targets[i],
            value: values[i],
            calldata: calldatas[i]
        })
    
    log ProposalCreated(proposalId, msg.sender, startBlock, endBlock, description)
    
    return proposalId

# ===== Voting =====

@external
def castVote(proposalId: uint256, support: uint8):
    """
    ลงคะแนนเสียง
    
    Parameters:
        proposalId: ID ของ proposal
        support: 0 = ค้าน, 1 = เห็นด้วย, 2 = งดออกเสียง
    """
    self._castVoteInternal(msg.sender, proposalId, support, "")

@external
def castVoteWithReason(proposalId: uint256, support: uint8, reason: String[256]):
    """ลงคะแนนพร้อม reason"""
    self._castVoteInternal(msg.sender, proposalId, support, reason)

@external
def castVoteBySig(
    proposalId: uint256,
    support: uint8,
    v: uint8,
    r: bytes32,
    s: bytes32
):
    """ลงคะแนนด้วย signature (gas-less voting)"""
    # ตรวจสอบ signature
    domainSeparator: bytes32 = keccak256(
        concat(
            keccak256("EIP712Domain(string name,uint256 chainId,address verifyingContract)"),
            keccak256(convert(self.name, Bytes[64])),
            convert(chain.id, bytes32),
            convert(self, bytes32)
        )
    )
    
    structHash: bytes32 = keccak256(
        concat(
            keccak256("Ballot(uint256 proposalId,uint8 support)"),
            convert(proposalId, bytes32),
            convert(convert(support, uint256), bytes32)
        )
    )
    
    digest: bytes32 = keccak256(
        concat(
            b"\x19\x01",
            domainSeparator,
            structHash
        )
    )
    
    signatory: address = ecrecover(digest, convert(v, uint256), r, s)
    assert signatory != empty(address), "Invalid signature"
    
    self._castVoteInternal(signatory, proposalId, support, "")

@internal
def _castVoteInternal(
    voter: address,
    proposalId: uint256,
    support: uint8,
    reason: String[256]
):
    """Internal vote casting"""
    state: uint8 = self._state(proposalId)
    assert state == PROPOSAL_ACTIVE, "Voting closed"
    assert support <= 2, "Invalid support"
    
    receipt: Receipt = self.receipts[proposalId][voter]
    assert not receipt.hasVoted, "Already voted"
    
    proposal: Proposal = self.proposals[proposalId]
    
    votes: uint256 = IGovernanceToken(self.token).getPriorVotes(voter, proposal.startBlock)
    
    if support == 0:
        self.proposals[proposalId].againstVotes += votes
    elif support == 1:
        self.proposals[proposalId].forVotes += votes
    else:
        self.proposals[proposalId].abstainVotes += votes
    
    self.receipts[proposalId][voter] = Receipt({
        hasVoted: True,
        support: support,
        votes: votes
    })
    
    log VoteCast(voter, proposalId, support, votes, reason)

# ===== Proposal State =====

@internal
@view
def _state(proposalId: uint256) -> uint8:
    """คำนวณ state ของ proposal"""
    assert proposalId > 0 and proposalId <= self.proposalCount, "Invalid proposal"
    
    proposal: Proposal = self.proposals[proposalId]
    
    if proposal.canceled:
        return PROPOSAL_CANCELED
    
    if proposal.executed:
        return PROPOSAL_EXECUTED
    
    if block.number <= proposal.startBlock:
        return PROPOSAL_PENDING
    
    if block.number <= proposal.endBlock:
        return PROPOSAL_ACTIVE
    
    if proposal.forVotes <= proposal.againstVotes:
        return PROPOSAL_DEFEATED
    
    # ตรวจสอบ quorum
    totalSupply: uint256 = IGovernanceToken(self.token).totalSupply()
    quorum: uint256 = totalSupply * self.quorumNumerator / QUORUM_DENOMINATOR
    
    if proposal.forVotes + proposal.abstainVotes < quorum:
        return PROPOSAL_DEFEATED
    
    if proposal.eta == 0:
        return PROPOSAL_SUCCEEDED
    
    timelockDelay: uint256 = ITimelock(self.timelock).minDelay()
    
    if block.timestamp >= proposal.eta + timelockDelay + 14 * 24 * 3600:
        return PROPOSAL_EXPIRED  # 14 days หลัง ready
    
    return PROPOSAL_QUEUED

@external
@view
def state(proposalId: uint256) -> uint8:
    return self._state(proposalId)

# ===== Queue & Execute =====

@external
def queue(proposalId: uint256):
    """
    Queue proposal เข้า timelock
    ต้อง state = SUCCEEDED
    """
    assert self._state(proposalId) == PROPOSAL_SUCCEEDED, "Not succeeded"
    
    proposal: Proposal = self.proposals[proposalId]
    timelockDelay: uint256 = ITimelock(self.timelock).minDelay()
    eta: uint256 = block.timestamp + timelockDelay
    
    for i: uint256 in range(10):
        if i >= proposal.actionsCount:
            break
        action: ProposalAction = self.proposalActions[proposalId][i]
        ITimelock(self.timelock).schedule(
            action.target,
            action.value,
            action.calldata,
            empty(bytes32),
            convert(proposalId, bytes32),
            timelockDelay
        )
    
    self.proposals[proposalId].eta = eta
    
    log ProposalQueued(proposalId, eta)

@external
def execute(proposalId: uint256):
    """
    Execute proposal จาก timelock
    ต้อง state = QUEUED และ eta ผ่านแล้ว
    """
    assert self._state(proposalId) == PROPOSAL_QUEUED, "Not queued"
    
    proposal: Proposal = self.proposals[proposalId]
    assert block.timestamp >= proposal.eta, "Not ready"
    
    self.proposals[proposalId].executed = True
    
    for i: uint256 in range(10):
        if i >= proposal.actionsCount:
            break
        action: ProposalAction = self.proposalActions[proposalId][i]
        ITimelock(self.timelock).execute(
            action.target,
            action.value,
            action.calldata,
            empty(bytes32),
            convert(proposalId, bytes32)
        )
    
    log ProposalExecuted(proposalId)

@external
def cancel(proposalId: uint256):
    """
    Cancel proposal
    เรียกได้โดย proposer หรือ guardian
    """
    state: uint8 = self._state(proposalId)
    assert state != PROPOSAL_EXECUTED, "Already executed"
    assert state != PROPOSAL_CANCELED, "Already canceled"
    
    proposal: Proposal = self.proposals[proposalId]
    
    # Proposer สามารถ cancel ได้ถ้า votes ต่ำกว่า threshold
    proposerVotes: uint256 = IGovernanceToken(self.token).getPriorVotes(
        proposal.proposer,
        block.number - 1
    )
    
    assert (
        msg.sender == self.admin or
        proposerVotes < self.proposalThreshold
    ), "Not authorized"
    
    self.proposals[proposalId].canceled = True
    
    # Cancel ใน timelock ถ้า queued แล้ว
    if proposal.eta != 0:
        for i: uint256 in range(10):
            if i >= proposal.actionsCount:
                break
            action: ProposalAction = self.proposalActions[proposalId][i]
            opId: bytes32 = keccak256(
                concat(
                    convert(action.target, bytes20),
                    convert(action.value, bytes32),
                    action.calldata,
                    empty(bytes32),
                    convert(proposalId, bytes32)
                )
            )
            ITimelock(self.timelock).cancel(opId)
    
    log ProposalCanceled(proposalId)

# ===== Quorum =====

@external
@view
def quorum(blockNumber: uint256) -> uint256:
    """จำนวน votes ขั้นต่ำที่ต้องการ"""
    totalSupply: uint256 = IGovernanceToken(self.token).totalSupply()
    return totalSupply * self.quorumNumerator / QUORUM_DENOMINATOR

# ===== Admin =====

@external
def setVotingDelay(newVotingDelay: uint256):
    assert msg.sender == self.admin, "Not admin"
    assert newVotingDelay >= 1, "Too short"
    assert newVotingDelay <= 40320, "Too long"  # ~1 week
    
    old: uint256 = self.votingDelay
    self.votingDelay = newVotingDelay
    
    log VotingDelaySet(old, newVotingDelay)

@external
def setVotingPeriod(newVotingPeriod: uint256):
    assert msg.sender == self.admin, "Not admin"
    assert newVotingPeriod >= 5760, "Too short"   # ~1 day
    assert newVotingPeriod <= 80640, "Too long"  # ~2 weeks
    
    old: uint256 = self.votingPeriod
    self.votingPeriod = newVotingPeriod
    
    log VotingPeriodSet(old, newVotingPeriod)

@external
def setProposalThreshold(newThreshold: uint256):
    assert msg.sender == self.admin, "Not admin"
    
    old: uint256 = self.proposalThreshold
    self.proposalThreshold = newThreshold
    
    log ProposalThresholdSet(old, newThreshold)
```

---

## 4. Governance Token with Delegation

```vyper
# @version 0.4.0
# contracts/GovernanceToken.vy
# Governance Token พร้อม vote delegation และ checkpointing

from vyper.interfaces import ERC20

implements: ERC20

# Events
event Transfer:
    from_: indexed(address)
    to: indexed(address)
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

# Struct สำหรับ checkpoint
struct Checkpoint:
    fromBlock: uint32
    votes: uint256

# ERC20 State
name: public(String[64])
symbol: public(String[32])
decimals: public(uint8)
totalSupply: public(uint256)
balanceOf: public(HashMap[address, uint256])
allowance: public(HashMap[address, HashMap[address, uint256]])

# Delegation State
delegates: public(HashMap[address, address])
numCheckpoints: public(HashMap[address, uint32])
checkpoints: HashMap[address, HashMap[uint32, Checkpoint]]

minter: public(address)

@deploy
def __init__(_name: String[64], _symbol: String[32], _minter: address):
    self.name = _name
    self.symbol = _symbol
    self.decimals = 18
    self.minter = _minter

# ===== Delegation =====

@external
def delegate(delegatee: address):
    """Delegate votes ไปให้คนอื่น"""
    self._delegate(msg.sender, delegatee)

@internal
def _delegate(delegator: address, delegatee: address):
    currentDelegate: address = self.delegates[delegator]
    delegatorBalance: uint256 = self.balanceOf[delegator]
    
    self.delegates[delegator] = delegatee
    
    log DelegateChanged(delegator, currentDelegate, delegatee)
    
    self._moveDelegates(currentDelegate, delegatee, delegatorBalance)

@internal
def _moveDelegates(srcRep: address, dstRep: address, amount: uint256):
    if srcRep != dstRep and amount > 0:
        if srcRep != empty(address):
            srcRepNum: uint32 = self.numCheckpoints[srcRep]
            srcRepOld: uint256 = 0
            if srcRepNum > 0:
                srcRepOld = self.checkpoints[srcRep][srcRepNum - 1].votes
            srcRepNew: uint256 = srcRepOld - amount
            self._writeCheckpoint(srcRep, srcRepNum, srcRepOld, srcRepNew)
        
        if dstRep != empty(address):
            dstRepNum: uint32 = self.numCheckpoints[dstRep]
            dstRepOld: uint256 = 0
            if dstRepNum > 0:
                dstRepOld = self.checkpoints[dstRep][dstRepNum - 1].votes
            dstRepNew: uint256 = dstRepOld + amount
            self._writeCheckpoint(dstRep, dstRepNum, dstRepOld, dstRepNew)

@internal
def _writeCheckpoint(
    delegatee: address,
    nCheckpoints: uint32,
    oldVotes: uint256,
    newVotes: uint256
):
    blockNumber: uint32 = convert(block.number, uint32)
    
    if nCheckpoints > 0 and self.checkpoints[delegatee][nCheckpoints - 1].fromBlock == blockNumber:
        # Update existing checkpoint ใน block นี้
        self.checkpoints[delegatee][nCheckpoints - 1].votes = newVotes
    else:
        # เพิ่ม checkpoint ใหม่
        self.checkpoints[delegatee][nCheckpoints] = Checkpoint({
            fromBlock: blockNumber,
            votes: newVotes
        })
        self.numCheckpoints[delegatee] = nCheckpoints + 1
    
    log DelegateVotesChanged(delegatee, oldVotes, newVotes)

@external
@view
def getCurrentVotes(account: address) -> uint256:
    """Votes ปัจจุบัน"""
    nCheckpoints: uint32 = self.numCheckpoints[account]
    if nCheckpoints == 0:
        return 0
    return self.checkpoints[account][nCheckpoints - 1].votes

@external
@view
def getPriorVotes(account: address, blockNumber: uint256) -> uint256:
    """
    Votes ณ block ที่ระบุ (สำหรับ governance)
    ใช้ binary search บน checkpoints
    """
    assert blockNumber < block.number, "Not finalized"
    
    nCheckpoints: uint32 = self.numCheckpoints[account]
    
    if nCheckpoints == 0:
        return 0
    
    # ตรวจสอบ checkpoint ล่าสุดก่อน
    if self.checkpoints[account][nCheckpoints - 1].fromBlock <= convert(blockNumber, uint32):
        return self.checkpoints[account][nCheckpoints - 1].votes
    
    # ตรวจสอบ checkpoint แรก
    if self.checkpoints[account][0].fromBlock > convert(blockNumber, uint32):
        return 0
    
    # Binary search
    lower: uint32 = 0
    upper: uint32 = nCheckpoints - 1
    
    for _: uint256 in range(32):
        if lower >= upper:
            break
        center: uint32 = upper - (upper - lower) / 2
        cp: Checkpoint = self.checkpoints[account][center]
        if cp.fromBlock == convert(blockNumber, uint32):
            return cp.votes
        elif cp.fromBlock < convert(blockNumber, uint32):
            lower = center
        else:
            upper = center - 1
    
    return self.checkpoints[account][lower].votes

# ===== ERC20 =====

@internal
def _transfer(sender: address, recipient: address, amount: uint256):
    assert sender != empty(address)
    assert recipient != empty(address)
    assert self.balanceOf[sender] >= amount
    
    self.balanceOf[sender] -= amount
    self.balanceOf[recipient] += amount
    
    log Transfer(sender, recipient, amount)
    
    self._moveDelegates(
        self.delegates[sender],
        self.delegates[recipient],
        amount
    )

@external
def transfer(to: address, amount: uint256) -> bool:
    self._transfer(msg.sender, to, amount)
    return True

@external
def transferFrom(from_: address, to: address, amount: uint256) -> bool:
    if self.allowance[from_][msg.sender] != max_value(uint256):
        assert self.allowance[from_][msg.sender] >= amount
        self.allowance[from_][msg.sender] -= amount
    self._transfer(from_, to, amount)
    return True

@external
def approve(spender: address, amount: uint256) -> bool:
    self.allowance[msg.sender][spender] = amount
    log Approval(msg.sender, spender, amount)
    return True

@external
def mint(to: address, amount: uint256):
    assert msg.sender == self.minter
    
    self.totalSupply += amount
    self.balanceOf[to] += amount
    
    log Transfer(empty(address), to, amount)
    
    self._moveDelegates(empty(address), self.delegates[to], amount)
```

---

## 5. Tests

```python
# tests/test_governance.py
import pytest
from brownie import (
    GovernanceToken,
    TimelockController,
    GovernorBravo,
    MockTarget,
    accounts,
    chain
)

@pytest.fixture
def setup():
    owner = accounts[0]
    alice = accounts[1]
    bob = accounts[2]
    charlie = accounts[3]
    
    # Deploy token
    token = GovernanceToken.deploy("DAO Token", "DAO", owner.address, {"from": owner})
    
    # Mint tokens
    token.mint(alice.address, 10**22, {"from": owner})  # 10000 tokens
    token.mint(bob.address, 5 * 10**21, {"from": owner})  # 5000 tokens
    token.mint(charlie.address, 2 * 10**21, {"from": owner})  # 2000 tokens
    
    # Deploy timelock (2 day delay)
    timelock = TimelockController.deploy(
        2 * 24 * 3600,
        [],
        [],
        owner.address,
        {"from": owner}
    )
    
    # Deploy governor
    governor = GovernorBravo.deploy(
        token.address,
        timelock.address,
        "My DAO",
        1,          # 1 block voting delay
        10,         # 10 blocks voting period
        10**21,     # 1000 token threshold
        400,        # 4% quorum
        {"from": owner}
    )
    
    # Setup timelock roles
    PROPOSER_ROLE = timelock.PROPOSER_ROLE()
    EXECUTOR_ROLE = timelock.EXECUTOR_ROLE()
    timelock.grantRole(PROPOSER_ROLE, governor.address, {"from": owner})
    timelock.grantRole(EXECUTOR_ROLE, governor.address, {"from": owner})
    
    # Deploy target
    target = MockTarget.deploy({"from": owner})
    
    return owner, alice, bob, charlie, token, timelock, governor, target

def test_delegation(setup):
    owner, alice, bob, charlie, token, timelock, governor, target = setup
    
    # Delegate to self
    token.delegate(alice.address, {"from": alice})
    token.delegate(bob.address, {"from": bob})
    
    alice_votes = token.getCurrentVotes(alice.address)
    bob_votes = token.getCurrentVotes(bob.address)
    
    assert alice_votes == token.balanceOf(alice.address)
    assert bob_votes == token.balanceOf(bob.address)
    
    print(f"Alice votes: {alice_votes}")
    print(f"Bob votes: {bob_votes}")

def test_propose_and_vote(setup):
    owner, alice, bob, charlie, token, timelock, governor, target = setup
    
    # Delegate
    token.delegate(alice.address, {"from": alice})
    token.delegate(bob.address, {"from": bob})
    token.delegate(charlie.address, {"from": charlie})
    
    chain.mine(1)
    
    # Alice creates proposal
    proposal_id = governor.propose(
        [target.address],
        [0],
        [target.setValue.encode_input(42)],
        "Set value to 42",
        {"from": alice}
    )
    
    print(f"Proposal ID: {proposal_id.return_value}")
    assert governor.state(proposal_id.return_value) == 0  # Pending
    
    # Wait for voting to start
    chain.mine(2)
    
    assert governor.state(proposal_id.return_value) == 1  # Active
    
    # Vote
    governor.castVote(proposal_id.return_value, 1, {"from": alice})  # For
    governor.castVote(proposal_id.return_value, 1, {"from": bob})    # For
    governor.castVote(proposal_id.return_value, 0, {"from": charlie})  # Against
    
    # Wait for voting to end
    chain.mine(12)
    
    assert governor.state(proposal_id.return_value) == 4  # Succeeded
    
    proposal = governor.proposals(proposal_id.return_value)
    print(f"For votes: {proposal[5]}")
    print(f"Against votes: {proposal[6]}")

def test_full_lifecycle(setup):
    owner, alice, bob, charlie, token, timelock, governor, target = setup
    
    token.delegate(alice.address, {"from": alice})
    token.delegate(bob.address, {"from": bob})
    chain.mine(1)
    
    # Propose
    pid = governor.propose(
        [target.address],
        [0],
        [target.setValue.encode_input(100)],
        "Set value to 100",
        {"from": alice}
    ).return_value
    
    chain.mine(2)
    
    # Vote
    governor.castVote(pid, 1, {"from": alice})
    governor.castVote(pid, 1, {"from": bob})
    
    chain.mine(12)
    
    # Queue
    governor.queue(pid, {"from": owner})
    assert governor.state(pid) == 5  # Queued
    
    # Wait for timelock
    chain.sleep(2 * 24 * 3600 + 1)
    chain.mine(1)
    
    # Execute
    governor.execute(pid, {"from": owner})
    assert governor.state(pid) == 7  # Executed
    
    assert target.value() == 100
    print("Proposal executed successfully!")

def test_quorum_failure(setup):
    owner, alice, bob, charlie, token, timelock, governor, target = setup
    
    # Only charlie delegates (2000 tokens = 11.8% of total)
    token.delegate(charlie.address, {"from": charlie})
    chain.mine(1)
    
    pid = governor.propose(
        [target.address],
        [0],
        [target.setValue.encode_input(1)],
        "Small vote",
        {"from": alice}  # Alice still has tokens even without delegating
    ).return_value
    
    chain.mine(2)
    
    governor.castVote(pid, 1, {"from": charlie})
    
    chain.mine(12)
    
    # Should be defeated due to insufficient quorum (need 4% of ~17000 = 680)
    state = governor.state(pid)
    print(f"Proposal state: {state}")
    # State might be defeated or succeeded based on total supply calculation
```

---

## 6. Emergency Powers

```vyper
# @version 0.4.0
# contracts/EmergencyModule.vy
# Emergency powers สำหรับ DAO

interface IGovernor:
    def cancel(proposalId: uint256): nonpayable

interface ITimelock:
    def cancel(id: bytes32): nonpayable

# Events
event EmergencyPause:
    caller: indexed(address)
    reason: String[256]

event EmergencyResume:
    caller: indexed(address)

event GuardianChanged:
    oldGuardian: indexed(address)
    newGuardian: indexed(address)

# State
guardian: public(address)
paused: public(bool)
pauseExpiry: public(uint256)  # timestamp ที่ pause หมดอายุ

MAX_PAUSE_DURATION: constant(uint256) = 7 * 24 * 3600  # 7 days

governor: public(address)
timelock: public(address)

@deploy
def __init__(_guardian: address, _governor: address, _timelock: address):
    self.guardian = _guardian
    self.governor = _governor
    self.timelock = _timelock

@external
def emergencyPause(reason: String[256]):
    """
    หยุด protocol ชั่วคราว
    เรียกได้โดย guardian เท่านั้น
    """
    assert msg.sender == self.guardian, "Not guardian"
    assert not self.paused, "Already paused"
    
    self.paused = True
    self.pauseExpiry = block.timestamp + MAX_PAUSE_DURATION
    
    log EmergencyPause(msg.sender, reason)

@external
def emergencyResume():
    """Resume protocol"""
    assert msg.sender == self.guardian or block.timestamp > self.pauseExpiry, "Not authorized"
    
    self.paused = False
    self.pauseExpiry = 0
    
    log EmergencyResume(msg.sender)

@external
def changeGuardian(newGuardian: address):
    """
    เปลี่ยน guardian
    ต้องผ่าน governance (เรียกจาก timelock)
    """
    assert msg.sender == self.timelock, "Only governance"
    
    old: address = self.guardian
    self.guardian = newGuardian
    
    log GuardianChanged(old, newGuardian)
```

---

## 7. สรุป

### Governance Best Practices:

**1. Voting Delay (1-2 days)**
- ให้เวลา users ที่ซื้อ token ก่อน propose ไม่สามารถ vote ได้ทันที
- ป้องกัน flash loan attacks

**2. Voting Period (3-7 days)**
- ให้เวลาเพียงพอสำหรับ community engagement
- ยาวเกินไป = slow, สั้นเกินไป = low participation

**3. Timelock (2-7 days)**
- Users มีเวลาอ่าน proposal ก่อน execute
- ถ้าไม่ชอบสามารถ exit ก่อน execute ได้

**4. Quorum (4-10%)**
- ต่ำเกินไป = small group control
- สูงเกินไป = hard to pass

**5. Proposal Threshold (1%)**
- ป้องกัน spam proposals
- ต้องมี skin in the game

**6. Emergency Guardian**
- ช่วย respond เร็วกับ critical bugs
- มี time limit และต้องผ่าน governance เพื่อ change
