# Part 033: Voting System

## สารบัญ
1. [บทนำ Voting](#บทนำ)
2. [Simple Voting](#simple-voting)
3. [Token-Weighted Voting](#token-weighted-voting)
4. [Delegation](#delegation)
5. [Quorum](#quorum)
6. [ตัวอย่าง: DAO Voting Contract](#ตัวอย่าง-dao-voting-contract)
7. [Test Code](#test-code)

---

## บทนำ

**Voting System** เป็นหัวใจสำคัญของ DAO (Decentralized Autonomous Organization) โดยให้ผู้ถือ token สามารถมีส่วนร่วมในการตัดสินใจ

### ประเภทของ Voting
- **Simple**: 1 address = 1 vote
- **Token-weighted**: 1 token = 1 vote
- **Quadratic**: sqrt(tokens) = votes
- **Conviction**: vote weight เพิ่มตามเวลา

---

## Simple Voting

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Simple Voting

event ProposalCreated:
    proposal_id: indexed(uint256)
    proposer: indexed(address)
    description: String[500]

event VoteCast:
    voter: indexed(address)
    proposal_id: indexed(uint256)
    support: bool

event ProposalExecuted:
    proposal_id: indexed(uint256)

struct Proposal:
    description: String[500]
    proposer: address
    start_time: uint256
    end_time: uint256
    votes_for: uint256
    votes_against: uint256
    executed: bool
    cancelled: bool

owner: public(address)
proposal_count: public(uint256)
proposals: public(HashMap[uint256, Proposal])
has_voted: HashMap[uint256, HashMap[address, bool]]

VOTING_PERIOD: constant(uint256) = 7 * 86400  # 7 วัน

@deploy
def __init__():
    self.owner = msg.sender

@external
def create_proposal(description: String[500]) -> uint256:
    """สร้าง proposal ใหม่"""
    proposal_id: uint256 = self.proposal_count
    
    self.proposals[proposal_id] = Proposal({
        description: description,
        proposer: msg.sender,
        start_time: block.timestamp,
        end_time: block.timestamp + VOTING_PERIOD,
        votes_for: 0,
        votes_against: 0,
        executed: False,
        cancelled: False
    })
    
    self.proposal_count += 1
    
    log ProposalCreated(proposal_id, msg.sender, description)
    return proposal_id

@external
def vote(proposal_id: uint256, support: bool):
    """ลงคะแนน"""
    assert proposal_id < self.proposal_count, "Invalid proposal"
    
    proposal: Proposal = self.proposals[proposal_id]
    assert block.timestamp >= proposal.start_time, "Voting not started"
    assert block.timestamp <= proposal.end_time, "Voting ended"
    assert not proposal.cancelled, "Proposal cancelled"
    assert not self.has_voted[proposal_id][msg.sender], "Already voted"
    
    self.has_voted[proposal_id][msg.sender] = True
    
    if support:
        self.proposals[proposal_id].votes_for += 1
    else:
        self.proposals[proposal_id].votes_against += 1
    
    log VoteCast(msg.sender, proposal_id, support)

@view
@external
def get_result(proposal_id: uint256) -> (uint256, uint256, bool):
    """ดู result (for, against, passed)"""
    proposal: Proposal = self.proposals[proposal_id]
    passed: bool = proposal.votes_for > proposal.votes_against
    return proposal.votes_for, proposal.votes_against, passed
```

---

## Token-Weighted Voting

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Token-Weighted Voting

interface GovernanceToken:
    def balanceOf(account: address) -> uint256: view
    def get_votes(account: address) -> uint256: view
    def get_past_votes(account: address, block_number: uint256) -> uint256: view

struct Proposal:
    description: String[500]
    proposer: address
    start_block: uint256
    end_block: uint256
    votes_for: uint256
    votes_against: uint256
    votes_abstain: uint256
    executed: bool
    eta: uint256  # timelock eta

VOTE_FOR: constant(uint8) = 0
VOTE_AGAINST: constant(uint8) = 1
VOTE_ABSTAIN: constant(uint8) = 2

governance_token: public(address)
proposal_count: public(uint256)
proposals: public(HashMap[uint256, Proposal])
receipts: HashMap[uint256, HashMap[address, uint8]]  # proposal -> voter -> vote type
vote_weights: HashMap[uint256, HashMap[address, uint256]]  # proposal -> voter -> weight

VOTING_PERIOD: constant(uint256) = 40320  # ~7 วัน ใน blocks (12 sec/block)
VOTING_DELAY: constant(uint256) = 1       # 1 block delay ก่อน vote

@deploy
def __init__(_token: address):
    self.governance_token = _token

@external
def propose(description: String[500]) -> uint256:
    """เสนอ proposal"""
    token: GovernanceToken = GovernanceToken(self.governance_token)
    assert token.get_votes(msg.sender) > 0, "Must have voting power"
    
    proposal_id: uint256 = self.proposal_count
    start_block: uint256 = block.number + VOTING_DELAY
    
    self.proposals[proposal_id] = Proposal({
        description: description,
        proposer: msg.sender,
        start_block: start_block,
        end_block: start_block + VOTING_PERIOD,
        votes_for: 0,
        votes_against: 0,
        votes_abstain: 0,
        executed: False,
        eta: 0
    })
    
    self.proposal_count += 1
    return proposal_id

@external
def cast_vote(proposal_id: uint256, support: uint8):
    """
    @notice ลงคะแนน
    @param support 0=for, 1=against, 2=abstain
    """
    assert support <= 2, "Invalid vote type"
    
    proposal: Proposal = self.proposals[proposal_id]
    assert block.number >= proposal.start_block, "Voting not started"
    assert block.number <= proposal.end_block, "Voting ended"
    
    # ตรวจสอบว่ายังไม่ได้ vote
    assert self.vote_weights[proposal_id][msg.sender] == 0, "Already voted"
    
    # ดึง voting power จาก snapshot block
    token: GovernanceToken = GovernanceToken(self.governance_token)
    weight: uint256 = token.get_past_votes(msg.sender, proposal.start_block)
    
    assert weight > 0, "No voting power"
    
    self.receipts[proposal_id][msg.sender] = support
    self.vote_weights[proposal_id][msg.sender] = weight
    
    if support == VOTE_FOR:
        self.proposals[proposal_id].votes_for += weight
    elif support == VOTE_AGAINST:
        self.proposals[proposal_id].votes_against += weight
    else:
        self.proposals[proposal_id].votes_abstain += weight

@view
@external
def get_votes_for_proposal(proposal_id: uint256) -> (uint256, uint256, uint256):
    """Return (for, against, abstain)"""
    p: Proposal = self.proposals[proposal_id]
    return p.votes_for, p.votes_against, p.votes_abstain
```

---

## Delegation

### Vote Delegation System

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Governance Token with Delegation

event DelegateChanged:
    delegator: indexed(address)
    from_delegate: indexed(address)
    to_delegate: indexed(address)

event DelegateVotesChanged:
    delegate: indexed(address)
    previous_votes: uint256
    new_votes: uint256

name: public(String[64])
symbol: public(String[8])
decimals: public(uint8)
total_supply: public(uint256)

balances: HashMap[address, uint256]
delegates: public(HashMap[address, address])  # account -> delegate
votes: public(HashMap[address, uint256])       # delegate -> voting power

@deploy
def __init__(_name: String[64], _symbol: String[8]):
    self.name = _name
    self.symbol = _symbol
    self.decimals = 18

@internal
def _move_delegates(
    src: address,
    dst: address,
    amount: uint256
):
    """โอน voting power"""
    if src != dst and amount > 0:
        if src != empty(address):
            old_votes: uint256 = self.votes[src]
            new_votes: uint256 = old_votes - amount
            self.votes[src] = new_votes
            log DelegateVotesChanged(src, old_votes, new_votes)
        
        if dst != empty(address):
            old_votes: uint256 = self.votes[dst]
            new_votes: uint256 = old_votes + amount
            self.votes[dst] = new_votes
            log DelegateVotesChanged(dst, old_votes, new_votes)

@external
def delegate(delegatee: address):
    """
    @notice มอบ voting power ให้ delegatee
    """
    old_delegate: address = self.delegates[msg.sender]
    new_delegate: address = delegatee
    
    if new_delegate == empty(address):
        new_delegate = msg.sender  # delegate ให้ตัวเอง
    
    self.delegates[msg.sender] = new_delegate
    
    # โอน voting power
    balance: uint256 = self.balances[msg.sender]
    self._move_delegates(old_delegate, new_delegate, balance)
    
    log DelegateChanged(msg.sender, old_delegate, new_delegate)

@external
def mint(to: address, amount: uint256):
    """Mint tokens พร้อม voting power"""
    self.balances[to] += amount
    self.total_supply += amount
    
    # อัปเดต voting power สำหรับ delegate
    delegate: address = self.delegates[to]
    if delegate == empty(address):
        delegate = to
    self._move_delegates(empty(address), delegate, amount)

@external
def transfer(to: address, amount: uint256) -> bool:
    """Transfer tokens พร้อมย้าย voting power"""
    assert self.balances[msg.sender] >= amount
    
    from_delegate: address = self.delegates[msg.sender]
    to_delegate: address = self.delegates[to]
    
    if from_delegate == empty(address):
        from_delegate = msg.sender
    if to_delegate == empty(address):
        to_delegate = to
    
    self.balances[msg.sender] -= amount
    self.balances[to] += amount
    
    self._move_delegates(from_delegate, to_delegate, amount)
    
    return True

@view
@external
def get_votes(account: address) -> uint256:
    """ดู voting power"""
    return self.votes[account]

@view
@external
def balance_of(account: address) -> uint256:
    return self.balances[account]
```

---

## Quorum

### Quorum Mechanism

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Voting with Quorum

interface IGovernanceToken:
    def total_supply() -> uint256: view
    def get_votes(account: address) -> uint256: view

struct Proposal:
    description: String[500]
    proposer: address
    end_time: uint256
    votes_for: uint256
    votes_against: uint256
    total_supply_at_creation: uint256
    executed: bool

PROPOSAL_THRESHOLD: constant(uint256) = 1000 * 10**18  # ต้องมี 1000 tokens ถึงเสนอได้
QUORUM_NUMERATOR: constant(uint256) = 4     # 4%
QUORUM_DENOMINATOR: constant(uint256) = 100
VOTING_PERIOD: constant(uint256) = 7 * 86400

token: public(address)
proposal_count: public(uint256)
proposals: public(HashMap[uint256, Proposal])
has_voted: HashMap[uint256, HashMap[address, bool]]

@deploy
def __init__(_token: address):
    self.token = _token

@internal
def _quorum(total_supply: uint256) -> uint256:
    """คำนวณ quorum"""
    return total_supply * QUORUM_NUMERATOR / QUORUM_DENOMINATOR

@internal
def _is_quorum_reached(proposal_id: uint256) -> bool:
    """ตรวจสอบว่าถึง quorum หรือไม่"""
    p: Proposal = self.proposals[proposal_id]
    quorum: uint256 = self._quorum(p.total_supply_at_creation)
    return (p.votes_for + p.votes_against + p.total_supply_at_creation * 0) >= quorum
    # simplified: ใช้แค่ votes_for + votes_against

@external
def propose(description: String[500]) -> uint256:
    token_contract: IGovernanceToken = IGovernanceToken(self.token)
    
    # ตรวจสอบ proposal threshold
    votes: uint256 = token_contract.get_votes(msg.sender)
    assert votes >= PROPOSAL_THRESHOLD, "Below proposal threshold"
    
    proposal_id: uint256 = self.proposal_count
    
    self.proposals[proposal_id] = Proposal({
        description: description,
        proposer: msg.sender,
        end_time: block.timestamp + VOTING_PERIOD,
        votes_for: 0,
        votes_against: 0,
        total_supply_at_creation: token_contract.total_supply(),
        executed: False
    })
    
    self.proposal_count += 1
    return proposal_id

@external
def vote(proposal_id: uint256, support: bool):
    token_contract: IGovernanceToken = IGovernanceToken(self.token)
    
    assert not self.has_voted[proposal_id][msg.sender]
    assert block.timestamp <= self.proposals[proposal_id].end_time
    
    weight: uint256 = token_contract.get_votes(msg.sender)
    assert weight > 0, "No voting power"
    
    self.has_voted[proposal_id][msg.sender] = True
    
    if support:
        self.proposals[proposal_id].votes_for += weight
    else:
        self.proposals[proposal_id].votes_against += weight

@view
@external
def has_passed(proposal_id: uint256) -> bool:
    """ตรวจสอบว่า proposal ผ่านหรือไม่"""
    p: Proposal = self.proposals[proposal_id]
    
    if block.timestamp <= p.end_time:
        return False  # ยังโหวตอยู่
    
    # ตรวจสอบ quorum
    quorum: uint256 = self._quorum(p.total_supply_at_creation)
    total_votes: uint256 = p.votes_for + p.votes_against
    
    if total_votes < quorum:
        return False  # ไม่ถึง quorum
    
    return p.votes_for > p.votes_against

@view
@external
def get_quorum(proposal_id: uint256) -> uint256:
    return self._quorum(self.proposals[proposal_id].total_supply_at_creation)
```

---

## ตัวอย่าง: DAO Voting Contract

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title DAO Governance
# @notice ระบบ Governance ครบถ้วนสำหรับ DAO

interface IVotingToken:
    def get_votes(account: address) -> uint256: view
    def total_supply() -> uint256: view

# ==================== Constants ====================

PROPOSAL_THRESHOLD: constant(uint256) = 100_000 * 10**18  # 100k tokens
QUORUM_BPS: constant(uint256) = 400                        # 4% of total supply
VOTING_DELAY: constant(uint256) = 86400                    # 1 day delay
VOTING_PERIOD: constant(uint256) = 7 * 86400               # 7 days
TIMELOCK_DELAY: constant(uint256) = 2 * 86400              # 2 days timelock

# Proposal States
PENDING: constant(uint8) = 0
ACTIVE: constant(uint8) = 1
CANCELLED: constant(uint8) = 2
DEFEATED: constant(uint8) = 3
SUCCEEDED: constant(uint8) = 4
QUEUED: constant(uint8) = 5
EXPIRED: constant(uint8) = 6
EXECUTED: constant(uint8) = 7

# Vote types
AGAINST: constant(uint8) = 0
FOR: constant(uint8) = 1
ABSTAIN: constant(uint8) = 2

# ==================== Structs ====================

struct ProposalCore:
    proposer: address
    start_time: uint256
    end_time: uint256
    eta: uint256
    canceled: bool
    executed: bool

struct ProposalVotes:
    against_votes: uint256
    for_votes: uint256
    abstain_votes: uint256

struct Receipt:
    has_voted: bool
    support: uint8
    votes: uint256

struct ProposalAction:
    target: address
    value: uint256
    calldata: Bytes[1024]
    description: String[500]

# ==================== Events ====================

event ProposalCreated:
    proposal_id: indexed(uint256)
    proposer: indexed(address)
    start_time: uint256
    end_time: uint256
    description: String[500]

event VoteCast:
    voter: indexed(address)
    proposal_id: indexed(uint256)
    support: uint8
    votes: uint256
    reason: String[256]

event ProposalQueued:
    proposal_id: indexed(uint256)
    eta: uint256

event ProposalExecuted:
    proposal_id: indexed(uint256)

event ProposalCanceled:
    proposal_id: indexed(uint256)

# ==================== State Variables ====================

token: public(address)
timelock: public(address)

proposal_count: public(uint256)
proposal_core: public(HashMap[uint256, ProposalCore])
proposal_votes: public(HashMap[uint256, ProposalVotes])
proposal_actions: public(HashMap[uint256, ProposalAction])
receipts: HashMap[uint256, HashMap[address, Receipt]]

# Snapshot of total supply per proposal
total_supply_snapshots: HashMap[uint256, uint256]

# ==================== Constructor ====================

@deploy
def __init__(_token: address, _timelock: address):
    self.token = _token
    self.timelock = _timelock

# ==================== Internal Functions ====================

@internal
def _get_state(proposal_id: uint256) -> uint8:
    core: ProposalCore = self.proposal_core[proposal_id]
    
    if core.canceled:
        return CANCELLED
    
    if core.executed:
        return EXECUTED
    
    if block.timestamp < core.start_time:
        return PENDING
    
    if block.timestamp <= core.end_time:
        return ACTIVE
    
    votes_data: ProposalVotes = self.proposal_votes[proposal_id]
    
    # ตรวจสอบ quorum
    quorum: uint256 = self.total_supply_snapshots[proposal_id] * QUORUM_BPS / 10000
    total_votes: uint256 = votes_data.for_votes + votes_data.against_votes + votes_data.abstain_votes
    
    if total_votes < quorum:
        return DEFEATED
    
    if votes_data.for_votes <= votes_data.against_votes:
        return DEFEATED
    
    if core.eta == 0:
        return SUCCEEDED
    
    if block.timestamp >= core.eta + TIMELOCK_DELAY + 14 * 86400:
        return EXPIRED
    
    return QUEUED

# ==================== Propose ====================

@external
def propose(
    target: address,
    value: uint256,
    calldata: Bytes[1024],
    description: String[500]
) -> uint256:
    """
    @notice สร้าง proposal
    """
    token_contract: IVotingToken = IVotingToken(self.token)
    
    # ตรวจสอบ threshold
    proposer_votes: uint256 = token_contract.get_votes(msg.sender)
    assert proposer_votes >= PROPOSAL_THRESHOLD, \
        "Governor: proposer votes below threshold"
    
    proposal_id: uint256 = self.proposal_count
    start_time: uint256 = block.timestamp + VOTING_DELAY
    end_time: uint256 = start_time + VOTING_PERIOD
    
    self.proposal_core[proposal_id] = ProposalCore({
        proposer: msg.sender,
        start_time: start_time,
        end_time: end_time,
        eta: 0,
        canceled: False,
        executed: False
    })
    
    self.proposal_actions[proposal_id] = ProposalAction({
        target: target,
        value: value,
        calldata: calldata,
        description: description
    })
    
    # Snapshot total supply
    self.total_supply_snapshots[proposal_id] = token_contract.total_supply()
    
    self.proposal_count += 1
    
    log ProposalCreated(proposal_id, msg.sender, start_time, end_time, description)
    return proposal_id

# ==================== Voting ====================

@external
def cast_vote(
    proposal_id: uint256,
    support: uint8
) -> uint256:
    """
    @notice ลงคะแนน
    @param support 0=against, 1=for, 2=abstain
    """
    return self._cast_vote(msg.sender, proposal_id, support, "")

@external
def cast_vote_with_reason(
    proposal_id: uint256,
    support: uint8,
    reason: String[256]
) -> uint256:
    """ลงคะแนนพร้อมเหตุผล"""
    return self._cast_vote(msg.sender, proposal_id, support, reason)

@internal
def _cast_vote(
    voter: address,
    proposal_id: uint256,
    support: uint8,
    reason: String[256]
) -> uint256:
    assert self._get_state(proposal_id) == ACTIVE, "Voting is closed"
    assert support <= 2, "Invalid vote type"
    assert not self.receipts[proposal_id][voter].has_voted, "Already voted"
    
    token_contract: IVotingToken = IVotingToken(self.token)
    weight: uint256 = token_contract.get_votes(voter)
    assert weight > 0, "No voting power"
    
    self.receipts[proposal_id][voter] = Receipt({
        has_voted: True,
        support: support,
        votes: weight
    })
    
    if support == AGAINST:
        self.proposal_votes[proposal_id].against_votes += weight
    elif support == FOR:
        self.proposal_votes[proposal_id].for_votes += weight
    else:
        self.proposal_votes[proposal_id].abstain_votes += weight
    
    log VoteCast(voter, proposal_id, support, weight, reason)
    return weight

# ==================== Queue & Execute ====================

@external
def queue(proposal_id: uint256):
    """Queue proposal สำหรับ timelock"""
    assert self._get_state(proposal_id) == SUCCEEDED, \
        "Proposal not succeeded"
    
    eta: uint256 = block.timestamp + TIMELOCK_DELAY
    self.proposal_core[proposal_id].eta = eta
    
    log ProposalQueued(proposal_id, eta)

@external
def execute(proposal_id: uint256):
    """Execute queued proposal"""
    assert self._get_state(proposal_id) == QUEUED, "Not queued"
    
    core: ProposalCore = self.proposal_core[proposal_id]
    assert block.timestamp >= core.eta, "Timelock not expired"
    
    self.proposal_core[proposal_id].executed = True
    
    action: ProposalAction = self.proposal_actions[proposal_id]
    
    if len(action.calldata) > 0:
        raw_call(action.target, action.calldata, value=action.value)
    elif action.value > 0:
        send(action.target, action.value)
    
    log ProposalExecuted(proposal_id)

@external
def cancel(proposal_id: uint256):
    """
    @notice ยกเลิก proposal
    @dev proposer สามารถยกเลิกได้ถ้า voting power ต่ำกว่า threshold
    """
    state: uint8 = self._get_state(proposal_id)
    assert state != EXECUTED and state != CANCELLED, "Cannot cancel"
    
    core: ProposalCore = self.proposal_core[proposal_id]
    
    token_contract: IVotingToken = IVotingToken(self.token)
    proposer_votes: uint256 = token_contract.get_votes(core.proposer)
    
    assert msg.sender == core.proposer or proposer_votes < PROPOSAL_THRESHOLD, \
        "Cannot cancel"
    
    self.proposal_core[proposal_id].canceled = True
    
    log ProposalCanceled(proposal_id)

# ==================== View Functions ====================

@view
@external
def state(proposal_id: uint256) -> uint8:
    return self._get_state(proposal_id)

@view
@external
def get_votes(proposal_id: uint256) -> (uint256, uint256, uint256):
    """Return (for, against, abstain)"""
    v: ProposalVotes = self.proposal_votes[proposal_id]
    return v.for_votes, v.against_votes, v.abstain_votes

@view
@external
def get_receipt(proposal_id: uint256, voter: address) -> Receipt:
    return self.receipts[proposal_id][voter]

@view
@external
def quorum(proposal_id: uint256) -> uint256:
    return self.total_supply_snapshots[proposal_id] * QUORUM_BPS / 10000

@view
@external
def has_voted(proposal_id: uint256, voter: address) -> bool:
    return self.receipts[proposal_id][voter].has_voted
```

---

## Test Code

```python
# tests/test_voting.py
import pytest

AGAINST = 0
FOR = 1
ABSTAIN = 2

@pytest.fixture
def owner(accounts):
    return accounts[0]

@pytest.fixture
def voters(accounts):
    return accounts[1:5]

@pytest.fixture
def token(owner, project):
    token = project.GovernanceToken.deploy(
        "DAO Token", "DAO",
        sender=owner
    )
    return token

@pytest.fixture
def governor(owner, token, project):
    # Deploy a simple timelock or use zero address
    return project.DAOGovernor.deploy(
        token.address,
        owner.address,  # timelock (simplified)
        sender=owner
    )

class TestSimpleVoting:
    
    def test_create_proposal(self, governor, owner, token):
        """ทดสอบสร้าง proposal"""
        # Mint enough tokens
        token.mint(owner.address, 200_000 * 10**18, sender=owner)
        token.delegate(owner.address, sender=owner)
        
        # Skip voting delay
        # proposal_id = governor.propose(...)
        pass
    
    def test_vote_for(self, governor, token, voters):
        """ทดสอบลงคะแนน FOR"""
        pass
    
    def test_quorum_not_reached(self, governor, token):
        """ทดสอบ proposal ที่ไม่ถึง quorum"""
        pass

class TestTokenWeightedVoting:
    
    def test_larger_holder_has_more_power(self, governor, token, accounts):
        """ผู้ถือ token มากกว่า มีอำนาจมากกว่า"""
        big_holder = accounts[1]
        small_holder = accounts[2]
        
        token.mint(big_holder.address, 1_000_000 * 10**18, sender=accounts[0])
        token.mint(small_holder.address, 1000 * 10**18, sender=accounts[0])
        
        token.delegate(big_holder.address, sender=big_holder)
        token.delegate(small_holder.address, sender=small_holder)
        
        from eth_utils import keccak
        assert token.get_votes(big_holder.address) > token.get_votes(small_holder.address)

class TestDelegation:
    
    def test_delegate_votes(self, token, accounts):
        """ทดสอบ delegation"""
        alice = accounts[0]
        bob = accounts[1]
        
        token.mint(alice.address, 1000 * 10**18, sender=alice)
        token.delegate(bob.address, sender=alice)
        
        assert token.get_votes(bob.address) == 1000 * 10**18
        assert token.get_votes(alice.address) == 0
    
    def test_undelegate(self, token, accounts):
        """ทดสอบการ undelegate"""
        alice = accounts[0]
        bob = accounts[1]
        
        token.mint(alice.address, 1000 * 10**18, sender=alice)
        token.delegate(bob.address, sender=alice)
        token.delegate(alice.address, sender=alice)  # delegate กลับให้ตัวเอง
        
        assert token.get_votes(alice.address) == 1000 * 10**18
        assert token.get_votes(bob.address) == 0
```

---

## สรุป

ระบบ Voting สำคัญสำหรับ DAO:

| Feature | Description |
|---------|-------------|
| Simple Voting | 1 address = 1 vote |
| Token-Weighted | 1 token = 1 vote |
| Delegation | มอบ vote ให้คนอื่น |
| Quorum | ต้องมีผู้ร่วมโหวตขั้นต่ำ |
| Timelock | delay ก่อน execute |

---

[⬅️ Part 032: Escrow Contract](part_032_escrow.md) | [Part 034: Auction Contract ➡️](part_034_auction.md)
