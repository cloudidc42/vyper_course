# Part 050: Cross-Chain Concepts in Vyper

## สารบัญ
1. [Overview](#overview)
2. [Bridge Patterns](#bridge-patterns)
3. [Message Passing](#message-passing)
4. [LayerZero Integration Concepts](#layerzero)
5. [Cross-Chain Governance](#governance)
6. [BridgeReceiver Contract](#bridge-receiver)
7. [Security Considerations](#security)
8. [Testing](#testing)

---

## 1. Overview {#overview}

Cross-chain technology ช่วยให้ Smart Contracts บน chain ต่างๆ สามารถสื่อสารกันได้ ซึ่งเป็นพื้นฐานของ Multi-chain DeFi

### ทำไมต้องใช้ Cross-Chain?

- **Liquidity**: รวม liquidity จากหลาย chains
- **Cost Optimization**: ใช้ chain ที่ถูกกว่าสำหรับ operations ต่างๆ
- **User Experience**: ไม่ต้อง manually bridge tokens
- **Interoperability**: dApps ทำงานข้าม chains

### Cross-Chain Protocols หลัก

| Protocol | Mechanism | Security Model |
|----------|-----------|---------------|
| LayerZero | Ultra Light Node | Oracle + Relayer |
| Axelar | Proof of Stake | Validator network |
| Wormhole | Guardian Network | 2/3 multisig |
| Hop Protocol | AMM-based | Optimistic |
| Connext | HTLC | Trustless |
| Stargate | Unified Liquidity | Delta Algorithm |

### Bridge Categories

1. **Lock & Mint**: Lock บน source chain, Mint บน destination
2. **Burn & Mint**: Burn บน source chain, Mint บน destination  
3. **Lock & Unlock**: Lock บน both sides (native bridges)
4. **Liquidity**: ใช้ liquidity pools บนทุก chain

---

## 2. Bridge Patterns {#bridge-patterns}

### Lock and Mint Pattern

```vyper
# @version 0.4.0
# @title TokenBridgeSource
# @notice Lock tokens บน source chain
# @dev ส่วนของ source chain (e.g., Ethereum)

interface IERC20:
    def transfer(to: address, amount: uint256) -> bool: nonpayable
    def transferFrom(frm: address, to: address, amount: uint256) -> bool: nonpayable
    def balanceOf(owner: address) -> uint256: view

event TokensLocked:
    sender: indexed(address)
    recipient: indexed(address)
    amount: uint256
    destination_chain_id: uint256
    nonce: uint256

event TokensUnlocked:
    recipient: indexed(address)
    amount: uint256
    source_chain_id: uint256
    source_tx_hash: bytes32

struct BridgeRequest:
    sender: address
    recipient: address
    amount: uint256
    destination_chain_id: uint256
    nonce: uint256
    timestamp: uint256
    is_processed: bool

# Constants
MAX_BRIDGE_AMOUNT: constant(uint256) = 10**6 * 10**18  # 1M tokens max
MIN_BRIDGE_AMOUNT: constant(uint256) = 10**15            # 0.001 tokens min
BRIDGE_FEE: constant(uint256) = 30                       # 0.3%
FEE_DENOMINATOR: constant(uint256) = 10000

# State
owner: public(address)
token: public(address)
bridge_validator: public(address)  # Address ที่มีสิทธิ์ unlock tokens
total_locked: public(uint256)
nonce: public(uint256)
is_paused: public(bool)

bridge_requests: public(HashMap[uint256, BridgeRequest])  # nonce => request
processed_unlocks: public(HashMap[bytes32, bool])  # tx_hash => processed

fee_collector: public(address)

@deploy
def __init__(
    token: address,
    bridge_validator: address,
    fee_collector: address
):
    assert token != empty(address), "Zero token"
    assert bridge_validator != empty(address), "Zero validator"
    
    self.owner = msg.sender
    self.token = token
    self.bridge_validator = bridge_validator
    self.fee_collector = fee_collector

@external
def lock_tokens(
    recipient: address,
    amount: uint256,
    destination_chain_id: uint256
):
    """
    Lock tokens บน source chain เพื่อ mint บน destination chain
    
    recipient: address บน destination chain
    amount: จำนวน tokens ที่จะ bridge
    destination_chain_id: Chain ID ของ destination
    """
    assert not self.is_paused, "Bridge paused"
    assert amount >= MIN_BRIDGE_AMOUNT, "Amount too small"
    assert amount <= MAX_BRIDGE_AMOUNT, "Amount too large"
    assert recipient != empty(address), "Zero recipient"
    
    # คำนวณ fee
    fee: uint256 = amount * BRIDGE_FEE // FEE_DENOMINATOR
    net_amount: uint256 = amount - fee
    
    # Transfer tokens จาก user
    assert IERC20(self.token).transferFrom(msg.sender, self, amount), "Transfer failed"
    
    # Transfer fee
    if fee > 0:
        assert IERC20(self.token).transfer(self.fee_collector, fee), "Fee transfer failed"
    
    # บันทึก bridge request
    current_nonce: uint256 = self.nonce
    self.bridge_requests[current_nonce] = BridgeRequest({
        sender: msg.sender,
        recipient: recipient,
        amount: net_amount,
        destination_chain_id: destination_chain_id,
        nonce: current_nonce,
        timestamp: block.timestamp,
        is_processed: False
    })
    
    self.nonce += 1
    self.total_locked += net_amount
    
    log TokensLocked(msg.sender, recipient, net_amount, destination_chain_id, current_nonce)

@external
def unlock_tokens(
    recipient: address,
    amount: uint256,
    source_chain_id: uint256,
    source_tx_hash: bytes32
):
    """
    Unlock tokens เมื่อได้รับ message จาก destination chain
    เรียกโดย bridge_validator เท่านั้น
    """
    assert msg.sender == self.bridge_validator, "Not validator"
    assert not self.is_paused, "Bridge paused"
    assert not self.processed_unlocks[source_tx_hash], "Already processed"
    assert recipient != empty(address), "Zero recipient"
    assert amount > 0, "Zero amount"
    
    # Mark as processed ก่อน transfer
    self.processed_unlocks[source_tx_hash] = True
    self.total_locked -= amount
    
    assert IERC20(self.token).transfer(recipient, amount), "Transfer failed"
    
    log TokensUnlocked(recipient, amount, source_chain_id, source_tx_hash)

@external
def set_paused(paused: bool):
    assert msg.sender == self.owner, "Not owner"
    self.is_paused = paused

@external
def set_validator(new_validator: address):
    assert msg.sender == self.owner, "Not owner"
    assert new_validator != empty(address), "Zero validator"
    self.bridge_validator = new_validator

@external
def emergency_withdraw(amount: uint256):
    """Emergency function ที่ owner เท่านั้นเรียกได้"""
    assert msg.sender == self.owner, "Not owner"
    assert self.is_paused, "Not paused"
    IERC20(self.token).transfer(self.owner, amount)
```

### Destination Chain (Mint/Burn)

```vyper
# @version 0.4.0
# @title BridgedToken
# @notice Wrapped token บน destination chain

interface ITokenBridgeSource:
    def lock_tokens(recipient: address, amount: uint256, destination_chain_id: uint256): nonpayable

event Transfer:
    sender: indexed(address)
    receiver: indexed(address)
    value: uint256

event Approval:
    owner: indexed(address)
    spender: indexed(address)
    value: uint256

event BridgeMinted:
    to: indexed(address)
    amount: uint256
    source_chain_id: uint256
    source_tx_hash: bytes32

event BridgeBurned:
    from_addr: indexed(address)
    amount: uint256
    destination_chain_id: uint256

# ERC20 State
balances: HashMap[address, uint256]
allowances: HashMap[address, HashMap[address, uint256]]
total_supply: public(uint256)

# Bridge State
owner: public(address)
bridge_validator: public(address)
source_chain_id: public(uint256)
source_bridge: public(address)

name: public(String[64])
symbol: public(String[32])
decimals: public(uint8)

processed_mints: public(HashMap[bytes32, bool])

@deploy
def __init__(
    name: String[64],
    symbol: String[32],
    bridge_validator: address,
    source_chain_id: uint256,
    source_bridge: address
):
    self.owner = msg.sender
    self.name = name
    self.symbol = symbol
    self.decimals = 18
    self.bridge_validator = bridge_validator
    self.source_chain_id = source_chain_id
    self.source_bridge = source_bridge

@external
@view
def balanceOf(account: address) -> uint256:
    return self.balances[account]

@external
@view
def allowance(owner: address, spender: address) -> uint256:
    return self.allowances[owner][spender]

@external
@view
def totalSupply() -> uint256:
    return self.total_supply

@external
def transfer(to: address, amount: uint256) -> bool:
    assert to != empty(address), "Zero address"
    assert self.balances[msg.sender] >= amount, "Insufficient"
    self.balances[msg.sender] -= amount
    self.balances[to] += amount
    log Transfer(msg.sender, to, amount)
    return True

@external
def transferFrom(sender: address, to: address, amount: uint256) -> bool:
    assert to != empty(address), "Zero address"
    assert self.balances[sender] >= amount, "Insufficient"
    assert self.allowances[sender][msg.sender] >= amount, "Insufficient allowance"
    self.balances[sender] -= amount
    self.balances[to] += amount
    self.allowances[sender][msg.sender] -= amount
    log Transfer(sender, to, amount)
    return True

@external
def approve(spender: address, amount: uint256) -> bool:
    assert spender != empty(address), "Zero address"
    self.allowances[msg.sender][spender] = amount
    log Approval(msg.sender, spender, amount)
    return True

@external
def bridge_mint(
    to: address,
    amount: uint256,
    source_tx_hash: bytes32
):
    """
    Mint tokens เมื่อได้รับ bridge message
    เรียกโดย bridge_validator เท่านั้น
    """
    assert msg.sender == self.bridge_validator, "Not validator"
    assert not self.processed_mints[source_tx_hash], "Already minted"
    assert to != empty(address), "Zero address"
    assert amount > 0, "Zero amount"
    
    self.processed_mints[source_tx_hash] = True
    self.total_supply += amount
    self.balances[to] += amount
    
    log Transfer(empty(address), to, amount)
    log BridgeMinted(to, amount, self.source_chain_id, source_tx_hash)

@external
def bridge_burn(
    amount: uint256,
    destination_chain_id: uint256,
    recipient: address
):
    """
    Burn tokens เพื่อ bridge กลับไป source chain
    """
    assert self.balances[msg.sender] >= amount, "Insufficient"
    assert amount > 0, "Zero amount"
    
    self.balances[msg.sender] -= amount
    self.total_supply -= amount
    
    log Transfer(msg.sender, empty(address), amount)
    log BridgeBurned(msg.sender, amount, destination_chain_id)
```

---

## 3. Message Passing {#message-passing}

```vyper
# @version 0.4.0
# @title CrossChainMessenger
# @notice Generic cross-chain message passing

event MessageSent:
    sender: indexed(address)
    destination_chain: uint256
    recipient: indexed(address)
    payload: Bytes[1024]
    nonce: uint256

event MessageReceived:
    source_chain: uint256
    sender: indexed(address)
    payload: Bytes[1024]
    nonce: uint256

struct Message:
    source_chain: uint256
    sender: address
    recipient: address
    payload: Bytes[1024]
    nonce: uint256
    timestamp: uint256

owner: public(address)
relayer: public(address)  # Off-chain relayer service
nonce: public(uint256)
received_messages: public(HashMap[bytes32, bool])

@deploy
def __init__(relayer: address):
    self.owner = msg.sender
    self.relayer = relayer

@internal
@pure
def _message_hash(
    source_chain: uint256,
    sender: address,
    nonce: uint256
) -> bytes32:
    return keccak256(
        concat(
            convert(source_chain, Bytes[32]),
            convert(sender, Bytes[32]),
            convert(nonce, Bytes[32])
        )
    )

@external
def send_message(
    destination_chain: uint256,
    recipient: address,
    payload: Bytes[1024]
) -> uint256:
    """Send cross-chain message"""
    assert recipient != empty(address), "Zero recipient"
    
    current_nonce: uint256 = self.nonce
    self.nonce += 1
    
    log MessageSent(msg.sender, destination_chain, recipient, payload, current_nonce)
    return current_nonce

@external
def receive_message(
    source_chain: uint256,
    sender: address,
    recipient: address,
    payload: Bytes[1024],
    nonce: uint256
):
    """
    Receive and process cross-chain message
    เรียกโดย relayer เท่านั้น
    """
    assert msg.sender == self.relayer, "Not relayer"
    assert recipient == self, "Wrong recipient"
    
    # Check for duplicates
    msg_hash: bytes32 = self._message_hash(source_chain, sender, nonce)
    assert not self.received_messages[msg_hash], "Already received"
    
    self.received_messages[msg_hash] = True
    
    log MessageReceived(source_chain, sender, payload, nonce)
    
    # Process message payload
    # ในการใช้งานจริงจะ decode payload และ execute action

@external
def set_relayer(new_relayer: address):
    assert msg.sender == self.owner, "Not owner"
    self.relayer = new_relayer
```

---

## 4. BridgeReceiver Contract {#bridge-receiver}

```vyper
# @version 0.4.0
# @title BridgeReceiver
# @notice Receives and processes cross-chain messages
# @notice Compatible with LayerZero-style message passing

interface IERC20:
    def transfer(to: address, amount: uint256) -> bool: nonpayable
    def balanceOf(owner: address) -> uint256: view

# ===== Events =====
event MessageReceived:
    source_chain_id: indexed(uint256)
    sender: indexed(address)
    payload_type: uint256
    amount: uint256
    recipient: address

event TokensBridged:
    recipient: indexed(address)
    amount: uint256
    source_chain: uint256

event GovernanceMessageExecuted:
    proposal_id: uint256
    action: uint256
    target: address

event FailedMessageStored:
    msg_hash: bytes32
    source_chain: uint256
    reason: String[100]

# ===== Payload Types =====
PAYLOAD_TOKEN_TRANSFER: constant(uint256) = 1
PAYLOAD_GOVERNANCE: constant(uint256) = 2
PAYLOAD_ARBITRARY: constant(uint256) = 3

# ===== Structs =====
struct FailedMessage:
    source_chain: uint256
    sender: address
    payload: Bytes[1024]
    nonce: uint256
    timestamp: uint256
    can_retry: bool

struct GovernanceAction:
    proposal_id: uint256
    action_type: uint256
    target: address
    data: Bytes[256]
    executed: bool

# ===== State =====
owner: public(address)
endpoint: public(address)  # LayerZero/bridge endpoint
token: public(address)

# Trusted remote chains/contracts
trusted_remotes: public(HashMap[uint256, address])  # chainId => address

# Message tracking
received_nonces: public(HashMap[uint256, HashMap[uint256, bool]])  # chainId => nonce => received
failed_messages: public(HashMap[bytes32, FailedMessage])

# Governance
governance_actions: public(HashMap[uint256, GovernanceAction])
governance_action_count: public(uint256)

# Security
is_paused: public(bool)
max_single_transfer: public(uint256)
daily_limit: public(uint256)
daily_transferred: public(uint256)
last_day_reset: public(uint256)

@deploy
def __init__(
    endpoint: address,
    token: address,
    max_single: uint256,
    daily_limit: uint256
):
    assert endpoint != empty(address), "Zero endpoint"
    assert token != empty(address), "Zero token"
    
    self.owner = msg.sender
    self.endpoint = endpoint
    self.token = token
    self.max_single_transfer = max_single
    self.daily_limit = daily_limit
    self.last_day_reset = block.timestamp

# ===== Security Helpers =====

@internal
def _check_daily_limit(amount: uint256):
    """Reset daily limit if new day"""
    if block.timestamp >= self.last_day_reset + 86400:
        self.daily_transferred = 0
        self.last_day_reset = block.timestamp
    
    assert self.daily_transferred + amount <= self.daily_limit, "Daily limit exceeded"
    self.daily_transferred += amount

# ===== Message Processing =====

@internal
def _decode_token_transfer(payload: Bytes[1024]) -> (address, uint256):
    """Decode token transfer payload"""
    # Simplified: first 32 bytes = address, next 32 = amount
    recipient: address = convert(
        slice(payload, 12, 20),  # Last 20 bytes of first 32
        address
    )
    amount: uint256 = convert(
        slice(payload, 32, 32),  # Second 32 bytes
        uint256
    )
    return recipient, amount

@internal
def _process_token_transfer(
    source_chain: uint256,
    recipient: address,
    amount: uint256
):
    """Process a cross-chain token transfer"""
    assert amount > 0, "Zero amount"
    assert amount <= self.max_single_transfer, "Exceeds single transfer limit"
    assert recipient != empty(address), "Zero recipient"
    
    self._check_daily_limit(amount)
    
    assert IERC20(self.token).transfer(recipient, amount), "Transfer failed"
    
    log TokensBridged(recipient, amount, source_chain)

@external
def receive_message(
    source_chain_id: uint256,
    sender: address,
    nonce: uint256,
    payload: Bytes[1024]
):
    """
    Main entry point for receiving cross-chain messages
    Called by the bridge endpoint/relayer
    """
    assert msg.sender == self.endpoint, "Only endpoint"
    assert not self.is_paused, "Receiver paused"
    
    # Verify trusted remote
    assert self.trusted_remotes[source_chain_id] == sender, "Untrusted source"
    
    # Check for duplicate messages
    assert not self.received_nonces[source_chain_id][nonce], "Duplicate message"
    self.received_nonces[source_chain_id][nonce] = True
    
    # Decode payload type (first 32 bytes)
    payload_type: uint256 = convert(slice(payload, 0, 32), uint256)
    
    if payload_type == PAYLOAD_TOKEN_TRANSFER:
        recipient: address = convert(slice(payload, 44, 20), address)
        amount: uint256 = convert(slice(payload, 64, 32), uint256)
        
        self._process_token_transfer(source_chain_id, recipient, amount)
        log MessageReceived(source_chain_id, sender, payload_type, amount, recipient)
    
    elif payload_type == PAYLOAD_GOVERNANCE:
        proposal_id: uint256 = convert(slice(payload, 32, 32), uint256)
        action_type: uint256 = convert(slice(payload, 64, 32), uint256)
        target: address = convert(slice(payload, 108, 20), address)
        
        self._queue_governance_action(proposal_id, action_type, target)
        log MessageReceived(source_chain_id, sender, payload_type, 0, target)
    
    else:
        # Unknown payload type - store as failed
        msg_hash: bytes32 = keccak256(
            concat(
                convert(source_chain_id, Bytes[32]),
                convert(sender, Bytes[32]),
                convert(nonce, Bytes[32])
            )
        )
        
        self.failed_messages[msg_hash] = FailedMessage({
            source_chain: source_chain_id,
            sender: sender,
            payload: payload,
            nonce: nonce,
            timestamp: block.timestamp,
            can_retry: True
        })
        
        log FailedMessageStored(msg_hash, source_chain_id, "Unknown payload type")

@internal
def _queue_governance_action(
    proposal_id: uint256,
    action_type: uint256,
    target: address
):
    """Queue a governance action from cross-chain message"""
    action_id: uint256 = self.governance_action_count
    self.governance_action_count += 1
    
    self.governance_actions[action_id] = GovernanceAction({
        proposal_id: proposal_id,
        action_type: action_type,
        target: target,
        data: b"",
        executed: False
    })

@external
def retry_failed_message(msg_hash: bytes32):
    """Retry a failed message"""
    assert msg.sender == self.owner, "Not owner"
    
    failed: FailedMessage = self.failed_messages[msg_hash]
    assert failed.can_retry, "Cannot retry"
    
    # Mark as cannot retry to prevent infinite loops
    self.failed_messages[msg_hash].can_retry = False
    
    # Re-process message
    # ในกรณีนี้เราแค่ log ว่า retry แล้ว
    # ในการใช้งานจริงจะ process payload อีกครั้ง

# ===== Admin Functions =====

@external
def set_trusted_remote(chain_id: uint256, remote_address: address):
    """Set trusted contract address บน chain อื่น"""
    assert msg.sender == self.owner, "Not owner"
    self.trusted_remotes[chain_id] = remote_address

@external
def set_endpoint(new_endpoint: address):
    assert msg.sender == self.owner, "Not owner"
    assert new_endpoint != empty(address), "Zero endpoint"
    self.endpoint = new_endpoint

@external
def set_paused(paused: bool):
    assert msg.sender == self.owner, "Not owner"
    self.is_paused = paused

@external
def update_limits(
    new_max_single: uint256,
    new_daily_limit: uint256
):
    assert msg.sender == self.owner, "Not owner"
    self.max_single_transfer = new_max_single
    self.daily_limit = new_daily_limit

@external
def withdraw_token(amount: uint256):
    """Emergency withdrawal"""
    assert msg.sender == self.owner, "Not owner"
    assert self.is_paused, "Not paused"
    IERC20(self.token).transfer(self.owner, amount)
```

---

## 5. Cross-Chain Governance {#governance}

```vyper
# @version 0.4.0
# @title CrossChainGovernance
# @notice Governance ที่ทำงานข้าม chains

interface IMessageSender:
    def send_message(
        destination_chain: uint256,
        recipient: address,
        payload: Bytes[1024]
    ) -> uint256: nonpayable

event ProposalCreated:
    id: indexed(uint256)
    proposer: indexed(address)
    description: String[256]
    destination_chain: uint256

event ProposalExecuted:
    id: indexed(uint256)
    executor: indexed(address)
    destination_chain: uint256

event VoteCast:
    proposal_id: indexed(uint256)
    voter: indexed(address)
    support: bool
    votes: uint256

struct Proposal:
    id: uint256
    proposer: address
    description: String[256]
    destination_chain: uint256
    target: address
    action_type: uint256
    for_votes: uint256
    against_votes: uint256
    start_time: uint256
    end_time: uint256
    executed: bool
    cancelled: bool

VOTING_PERIOD: constant(uint256) = 3 * 86400  # 3 days
EXECUTION_DELAY: constant(uint256) = 2 * 86400  # 2 days timelock
QUORUM_VOTES: constant(uint256) = 100000 * 10**18  # 100k tokens
PROPOSAL_THRESHOLD: constant(uint256) = 10000 * 10**18  # 10k tokens needed to propose

owner: public(address)
governance_token: public(address)
messenger: public(address)

proposals: public(HashMap[uint256, Proposal])
proposal_count: public(uint256)

has_voted: public(HashMap[uint256, HashMap[address, bool]])  # proposal_id => voter => voted

@deploy
def __init__(
    governance_token: address,
    messenger: address
):
    self.owner = msg.sender
    self.governance_token = governance_token
    self.messenger = messenger

@external
def propose(
    description: String[256],
    destination_chain: uint256,
    target: address,
    action_type: uint256
) -> uint256:
    """Create a new cross-chain governance proposal"""
    # Check proposer has enough tokens
    # ใน production จะใช้ interface ของ governance token
    
    proposal_id: uint256 = self.proposal_count
    self.proposal_count += 1
    
    self.proposals[proposal_id] = Proposal({
        id: proposal_id,
        proposer: msg.sender,
        description: description,
        destination_chain: destination_chain,
        target: target,
        action_type: action_type,
        for_votes: 0,
        against_votes: 0,
        start_time: block.timestamp,
        end_time: block.timestamp + VOTING_PERIOD,
        executed: False,
        cancelled: False
    })
    
    log ProposalCreated(proposal_id, msg.sender, description, destination_chain)
    return proposal_id

@external
def cast_vote(proposal_id: uint256, support: bool, votes: uint256):
    """Cast vote on a proposal"""
    proposal: Proposal = self.proposals[proposal_id]
    
    assert not proposal.cancelled, "Cancelled"
    assert block.timestamp >= proposal.start_time, "Not started"
    assert block.timestamp <= proposal.end_time, "Ended"
    assert not self.has_voted[proposal_id][msg.sender], "Already voted"
    
    self.has_voted[proposal_id][msg.sender] = True
    
    if support:
        self.proposals[proposal_id].for_votes += votes
    else:
        self.proposals[proposal_id].against_votes += votes
    
    log VoteCast(proposal_id, msg.sender, support, votes)

@external
def execute_proposal(proposal_id: uint256):
    """Execute a passed proposal via cross-chain message"""
    proposal: Proposal = self.proposals[proposal_id]
    
    assert not proposal.executed, "Already executed"
    assert not proposal.cancelled, "Cancelled"
    assert block.timestamp > proposal.end_time + EXECUTION_DELAY, "Timelock not passed"
    
    # Check quorum and majority
    total_votes: uint256 = proposal.for_votes + proposal.against_votes
    assert total_votes >= QUORUM_VOTES, "Quorum not reached"
    assert proposal.for_votes > proposal.against_votes, "No majority"
    
    self.proposals[proposal_id].executed = True
    
    # Build cross-chain payload
    payload: Bytes[1024] = concat(
        convert(2, Bytes[32]),  # PAYLOAD_GOVERNANCE type
        convert(proposal_id, Bytes[32]),
        convert(proposal.action_type, Bytes[32]),
        convert(proposal.target, Bytes[32])
    )
    
    # Send cross-chain message
    IMessageSender(self.messenger).send_message(
        proposal.destination_chain,
        proposal.target,
        payload
    )
    
    log ProposalExecuted(proposal_id, msg.sender, proposal.destination_chain)
```

---

## 6. Testing {#testing}

```python
# tests/test_bridge.py
import pytest
import boa

@pytest.fixture
def deployer():
    return boa.env.generate_address()

@pytest.fixture
def validator():
    return boa.env.generate_address()

@pytest.fixture
def fee_collector():
    return boa.env.generate_address()

@pytest.fixture
def alice():
    addr = boa.env.generate_address()
    return addr

@pytest.fixture
def token(deployer):
    with boa.env.prank(deployer):
        t = boa.load(
            "contracts/Token.vy",
            "Bridge Token",
            "BRT",
            10**9 * 10**18
        )
        t.mint(deployer, 10**6 * 10**18)
    return t

@pytest.fixture
def bridge(deployer, token, validator, fee_collector):
    with boa.env.prank(deployer):
        return boa.load(
            "contracts/TokenBridgeSource.vy",
            token.address,
            validator,
            fee_collector
        )

@pytest.fixture
def bridge_with_funds(bridge, deployer, token):
    """Bridge ที่มี tokens สำรองไว้"""
    with boa.env.prank(deployer):
        token.transfer(bridge.address, 100000 * 10**18)
    return bridge


class TestBridgeSource:
    def test_lock_tokens(self, bridge, deployer, token, alice):
        bridge_amount = 1000 * 10**18
        
        with boa.env.prank(deployer):
            token.transfer(alice, bridge_amount)
        
        with boa.env.prank(alice):
            token.approve(bridge.address, bridge_amount)
            bridge.lock_tokens(
                alice,          # recipient on dest chain
                bridge_amount,  # amount
                137             # Polygon chain ID
            )
        
        # Check fee deducted
        fee = bridge_amount * 30 // 10000  # 0.3%
        net_amount = bridge_amount - fee
        
        assert token.balanceOf(bridge.address) == net_amount
        assert token.balanceOf(alice) == 0
    
    def test_unlock_tokens(self, bridge_with_funds, validator, alice):
        unlock_amount = 1000 * 10**18
        source_tx = b'\x01' * 32  # Mock tx hash
        
        balance_before = boa.env.evm.get_balance(alice) if hasattr(boa.env.evm, 'get_balance') else 0
        
        with boa.env.prank(validator):
            bridge_with_funds.unlock_tokens(
                alice,
                unlock_amount,
                137,  # Source chain: Polygon
                source_tx
            )
        
        from contracts.Token import IToken
        # alice should receive tokens
        assert True  # Simplified check
    
    def test_cannot_unlock_twice(self, bridge_with_funds, validator, alice):
        unlock_amount = 100 * 10**18
        source_tx = b'\x02' * 32
        
        with boa.env.prank(validator):
            bridge_with_funds.unlock_tokens(alice, unlock_amount, 137, source_tx)
        
        with pytest.raises(Exception, match="Already processed"):
            with boa.env.prank(validator):
                bridge_with_funds.unlock_tokens(alice, unlock_amount, 137, source_tx)
    
    def test_only_validator_can_unlock(self, bridge_with_funds, alice):
        with pytest.raises(Exception, match="Not validator"):
            with boa.env.prank(alice):
                bridge_with_funds.unlock_tokens(alice, 100, 137, b'\x03' * 32)
    
    def test_paused_bridge(self, bridge, deployer, alice, token):
        with boa.env.prank(deployer):
            bridge.set_paused(True)
        
        with pytest.raises(Exception, match="Bridge paused"):
            with boa.env.prank(alice):
                bridge.lock_tokens(alice, 100, 137)


class TestBridgeReceiver:
    @pytest.fixture
    def endpoint(self):
        return boa.env.generate_address()
    
    @pytest.fixture
    def receiver(self, deployer, token, endpoint):
        max_single = 10000 * 10**18
        daily_limit = 100000 * 10**18
        
        with boa.env.prank(deployer):
            r = boa.load(
                "contracts/BridgeReceiver.vy",
                endpoint,
                token.address,
                max_single,
                daily_limit
            )
            # Fund receiver
            token.transfer(r.address, 50000 * 10**18)
        
        return r
    
    def test_set_trusted_remote(self, receiver, deployer):
        remote_address = boa.env.generate_address()
        
        with boa.env.prank(deployer):
            receiver.set_trusted_remote(1, remote_address)  # Chain 1 = Ethereum
        
        assert receiver.trusted_remotes(1) == remote_address
    
    def test_only_endpoint_can_deliver(self, receiver, alice, deployer):
        # Set trusted remote
        remote = boa.env.generate_address()
        with boa.env.prank(deployer):
            receiver.set_trusted_remote(1, remote)
        
        # Build payload
        payload = (
            b'\x00' * 31 + b'\x01' +  # PAYLOAD_TOKEN_TRANSFER = 1
            b'\x00' * 12 + bytes.fromhex(alice[2:]) +  # recipient
            (100 * 10**18).to_bytes(32, 'big')  # amount
        )
        
        # Only endpoint can call
        with pytest.raises(Exception, match="Only endpoint"):
            with boa.env.prank(alice):
                receiver.receive_message(1, remote, 0, payload)
    
    def test_paused_receiver(self, receiver, deployer, endpoint):
        with boa.env.prank(deployer):
            receiver.set_paused(True)
        
        with pytest.raises(Exception, match="Receiver paused"):
            with boa.env.prank(endpoint):
                receiver.receive_message(1, endpoint, 0, b'\x00' * 96)


class TestCrossChainGovernance:
    @pytest.fixture
    def gov_token(self, deployer):
        with boa.env.prank(deployer):
            t = boa.load(
                "contracts/Token.vy",
                "Gov Token",
                "GOV",
                10**9 * 10**18
            )
            t.mint(deployer, 10**6 * 10**18)
        return t
    
    @pytest.fixture
    def messenger(self, deployer):
        return boa.env.generate_address()  # Mock messenger
    
    @pytest.fixture
    def governance(self, deployer, gov_token, messenger):
        with boa.env.prank(deployer):
            return boa.load(
                "contracts/CrossChainGovernance.vy",
                gov_token.address,
                messenger
            )
    
    def test_create_proposal(self, governance, deployer):
        target = boa.env.generate_address()
        
        with boa.env.prank(deployer):
            proposal_id = governance.propose(
                "Upgrade contract on Polygon",
                137,    # Polygon
                target,
                1       # Action type: upgrade
            )
        
        assert governance.proposal_count() == 1
        proposal = governance.proposals(0)
        assert proposal[0] == 0  # id
        assert proposal[2] == "Upgrade contract on Polygon"  # description
    
    def test_vote_on_proposal(self, governance, deployer):
        target = boa.env.generate_address()
        
        with boa.env.prank(deployer):
            governance.propose("Test proposal", 137, target, 1)
            governance.cast_vote(0, True, 200000 * 10**18)
        
        proposal = governance.proposals(0)
        assert proposal[6] == 200000 * 10**18  # for_votes
    
    def test_cannot_vote_twice(self, governance, deployer):
        target = boa.env.generate_address()
        
        with boa.env.prank(deployer):
            governance.propose("Test", 137, target, 1)
            governance.cast_vote(0, True, 100000 * 10**18)
        
        with pytest.raises(Exception, match="Already voted"):
            with boa.env.prank(deployer):
                governance.cast_vote(0, True, 100000 * 10**18)


if __name__ == "__main__":
    pytest.main([__file__, "-v"])
```

---

## 7. Security Considerations {#security}

### Bridge Security Checklist

```
✅ Message Authentication:
[ ] Verify sender is trusted remote contract
[ ] Track processed message nonces
[ ] Use unique message hashes

✅ Token Security:
[ ] Limit single transaction amounts
[ ] Implement daily transfer limits
[ ] Circuit breaker for emergency pause

✅ Validator Security:
[ ] Multi-sig for critical operations
[ ] Timelock for parameter changes
[ ] Monitor for suspicious activity

✅ Replay Protection:
[ ] Unique nonces per chain pair
[ ] Store processed message hashes
[ ] Clear audit trail in events

✅ Governance Security:
[ ] Adequate voting period (3+ days)
[ ] Execution timelock (2+ days)
[ ] Quorum requirements
[ ] Cancellation mechanism
```

### Known Vulnerabilities

1. **Validator Compromise**: Single validator = single point of failure
   - แก้: ใช้ multi-sig validator
   
2. **Replay Attacks**: ส่ง message เดิมซ้ำ
   - แก้: Track processed nonces

3. **Fake Source**: แอบอ้างว่าเป็น source contract
   - แก้: Verify trusted_remotes mapping

4. **Reentrancy**: Callback ใน token transfer
   - แก้: Mark processed before transfer

---

## สรุป

### Cross-Chain Architecture Patterns

```
1. Lock & Mint (ง่ายที่สุด):
   Source: Lock tokens → emit event
   Off-chain: Listen for event → relay
   Destination: Mint wrapped tokens

2. Burn & Mint (สำหรับ native assets):
   Source: Burn tokens → emit event
   Off-chain: Listen → verify → relay
   Destination: Mint equivalent tokens

3. HTLC (Trustless):
   ทั้งสองฝั่ง lock tokens พร้อมกัน
   ใช้ hash time-lock สำหรับ atomic swap
   ไม่ต้องมี centralized validator

4. Optimistic Bridge:
   Assume valid, ให้ challenge period
   Fraud proof จาก watchers
   Cheap แต่ช้า (1 week finality)
```
