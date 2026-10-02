# Part 070: Cross-Chain Messaging Patterns (รูปแบบการส่งข้อความข้ามเชน)

## สารบัญ
1. [บทนำ Cross-Chain Messaging](#s1)
2. [LayerZero Protocol](#s2)
3. [Wormhole Message Format](#s3)
4. [Chainlink CCIP](#s4)
5. [Multi-Chain Token (OFT Pattern)](#s5)
6. [Cross-Chain Governance](#s6)
7. [CrossChainToken Contract](#s7)
8. [Security Considerations](#s8)

---

## 1. บทนำ Cross-Chain Messaging {#s1}

**Cross-chain messaging** ช่วยให้ smart contracts บน blockchain หนึ่งสื่อสารกับ blockchain อื่นได้
ใช้สำหรับ:
- Transfer tokens ข้าม chains
- Sync state ระหว่าง chains
- Cross-chain governance
- Multi-chain DeFi protocols

### ผู้ให้บริการหลัก

| Protocol | Mechanism | Security Model |
|---|---|---|
| LayerZero | Ultra Light Nodes | Oracle + Relayer |
| Wormhole | Guardian network | 19 validators (2/3 required) |
| Chainlink CCIP | DON + Risk Management | Defense in depth |
| Axelar | PoS validator set | AXL stakers |

---

## 2. LayerZero Protocol {#s2}

LayerZero ใช้ Oracle และ Relayer แยกกันเพื่อ verify messages

```python
# @version 0.4.0
# ILayerZero.vy
# LayerZero interface definitions

# Chain IDs (LayerZero specific, different from EVM chain IDs)
LZ_ETHEREUM: constant(uint16) = 101
LZ_BSC: constant(uint16) = 102
LZ_AVALANCHE: constant(uint16) = 106
LZ_POLYGON: constant(uint16) = 109
LZ_ARBITRUM: constant(uint16) = 110
LZ_OPTIMISM: constant(uint16) = 111
LZ_BASE: constant(uint16) = 184

interface ILayerZeroEndpoint:
    def send(
        dst_chain_id: uint16,
        destination: Bytes[40],   # abi.encodePacked(address, address)
        payload: Bytes[65536],
        refund_address: address,
        zero_pay_address: address,
        adapter_params: Bytes[65536]
    ) -> bool: payable
    
    def receivePayload(
        src_chain_id: uint16,
        src_address: Bytes[40],
        dst_address: address,
        nonce: uint64,
        gas_limit: uint256,
        payload: Bytes[65536]
    ): nonpayable
    
    def getInboundNonce(src_chain_id: uint16, src_address: Bytes[40]) -> uint64: view
    def getOutboundNonce(dst_chain_id: uint16, src_address: address) -> uint64: view
    def estimateFees(
        dst_chain_id: uint16,
        user_application: address,
        payload: Bytes[65536],
        pay_in_zro: bool,
        adapter_params: Bytes[65536]
    ) -> (uint256, uint256): view

interface ILayerZeroReceiver:
    def lzReceive(
        src_chain_id: uint16,
        src_address: Bytes[40],
        nonce: uint64,
        payload: Bytes[65536]
    ): nonpayable
```

### LayerZero Application Contract

```python
# @version 0.4.0
# LayerZeroApp.vy
# Base contract for LayerZero applications

interface ILayerZeroEndpoint:
    def send(
        dst_chain_id: uint16,
        destination: Bytes[40],
        payload: Bytes[65536],
        refund_address: address,
        zero_pay_address: address,
        adapter_params: Bytes[65536]
    ): payable
    
    def estimateFees(
        dst_chain_id: uint16,
        user_application: address,
        payload: Bytes[65536],
        pay_in_zro: bool,
        adapter_params: Bytes[65536]
    ) -> (uint256, uint256): view

LZ_ENDPOINT: immutable(address)

owner: public(address)

# Trusted remote addresses (chain_id -> remote contract address)
trusted_remotes: public(HashMap[uint16, Bytes[40]])

# Message nonce tracking
sent_nonces: public(HashMap[uint16, uint64])
received_nonces: public(HashMap[uint16, uint64])

# Failed messages for retry
failed_messages: public(HashMap[bytes32, bool])

event MessageSent:
    dst_chain_id: indexed(uint16)
    payload: Bytes[65536]
    nonce: uint64

event MessageReceived:
    src_chain_id: indexed(uint16)
    payload: Bytes[65536]
    nonce: uint64

event MessageFailed:
    src_chain_id: indexed(uint16)
    nonce: uint64
    reason: String[100]

@deploy
def __init__(endpoint: address):
    LZ_ENDPOINT = endpoint
    self.owner = msg.sender

@internal
def _lz_send(
    dst_chain_id: uint16,
    payload: Bytes[65536],
    refund_address: address,
    adapter_params: Bytes[65536]
):
    """Internal function to send LayerZero message"""
    destination: Bytes[40] = self.trusted_remotes[dst_chain_id]
    assert len(destination) == 40, "No trusted remote for chain"
    
    ILayerZeroEndpoint(LZ_ENDPOINT).send(
        dst_chain_id,
        destination,
        payload,
        refund_address,
        empty(address),
        adapter_params,
        value=msg.value
    )
    
    self.sent_nonces[dst_chain_id] += 1
    log MessageSent(dst_chain_id, payload, self.sent_nonces[dst_chain_id])

@external
def lzReceive(
    src_chain_id: uint16,
    src_address: Bytes[40],
    nonce: uint64,
    payload: Bytes[65536]
):
    """
    Receive a LayerZero message
    Called by the endpoint when a message arrives
    """
    assert msg.sender == LZ_ENDPOINT, "Not LayerZero endpoint"
    
    # Verify the source is trusted
    trusted: Bytes[40] = self.trusted_remotes[src_chain_id]
    assert trusted == src_address, "Untrusted source"
    
    # Process the message
    self._lz_receive(src_chain_id, src_address, nonce, payload)
    
    self.received_nonces[src_chain_id] = nonce
    log MessageReceived(src_chain_id, payload, nonce)

@internal
def _lz_receive(
    src_chain_id: uint16,
    src_address: Bytes[40],
    nonce: uint64,
    payload: Bytes[65536]
):
    """
    Override this in subcontracts to process messages
    """
    pass  # Override in subclass

@external
def set_trusted_remote(chain_id: uint16, remote: Bytes[40]):
    """Set a trusted remote contract for a chain"""
    assert msg.sender == self.owner, "Not owner"
    assert len(remote) == 40, "Invalid remote address"
    self.trusted_remotes[chain_id] = remote

@view
@external
def estimate_send_fee(
    dst_chain_id: uint16,
    payload: Bytes[65536]
) -> (uint256, uint256):
    """Estimate the fee for sending a message"""
    return ILayerZeroEndpoint(LZ_ENDPOINT).estimateFees(
        dst_chain_id,
        self,
        payload,
        False,
        b""
    )
```

---

## 3. Wormhole Message Format {#s3}

```python
# @version 0.4.0
# WormholeMessenger.vy
# Cross-chain messaging via Wormhole protocol

# Wormhole chain IDs (different from EVM chain IDs)
WH_ETHEREUM: constant(uint16) = 2
WH_BSC: constant(uint16) = 4
WH_POLYGON: constant(uint16) = 5
WH_AVALANCHE: constant(uint16) = 6
WH_ARBITRUM: constant(uint16) = 23
WH_OPTIMISM: constant(uint16) = 24
WH_BASE: constant(uint16) = 30

interface IWormhole:
    def publishMessage(
        nonce: uint32,
        payload: Bytes[65536],
        consistency_level: uint8
    ) -> uint64: payable
    
    def parseAndVerifyVM(
        encoded_vm: Bytes[65536]
    ) -> bool: view
    
    def messageFee() -> uint256: view
    
    def chainId() -> uint16: view

interface ITokenBridge:
    def transferTokens(
        token: address,
        amount: uint256,
        recipient_chain: uint16,
        recipient: bytes32,
        relayer_fee: uint256,
        nonce: uint32
    ) -> uint64: payable
    
    def completeTransfer(encoded_vm: Bytes[65536]): nonpayable
    def attestToken(token: address, nonce: uint32) -> uint64: payable

# Wormhole Core Bridge address
WORMHOLE: immutable(address)

# Wormhole Token Bridge address
TOKEN_BRIDGE: immutable(address)

owner: public(address)

# Track processed VAAs to prevent replay
processed_vaas: public(HashMap[bytes32, bool])

# Registered emitters (chain_id -> emitter address)
registered_emitters: public(HashMap[uint16, bytes32])

event WormholeMessageSent:
    sequence: uint64
    payload: Bytes[65536]

event WormholeMessageReceived:
    emitter_chain: uint16
    emitter_address: bytes32
    sequence: uint64

@deploy
def __init__(wormhole_address: address, token_bridge_address: address):
    WORMHOLE = wormhole_address
    TOKEN_BRIDGE = token_bridge_address
    self.owner = msg.sender

@external
@payable
def send_message(
    payload: Bytes[65536],
    nonce: uint32,
    consistency_level: uint8
) -> uint64:
    """
    Publish a message via Wormhole
    @param payload Message content
    @param nonce Unique nonce per emitter
    @param consistency_level 1=finalized, 32=instant
    @return sequence number
    """
    fee: uint256 = IWormhole(WORMHOLE).messageFee()
    assert msg.value >= fee, "Insufficient fee"
    
    sequence: uint64 = IWormhole(WORMHOLE).publishMessage(
        nonce,
        payload,
        consistency_level,
        value=fee
    )
    
    log WormholeMessageSent(sequence, payload)
    return sequence

@external
def receive_message(encoded_vaa: Bytes[65536]):
    """
    Receive and verify a Wormhole VAA (Verified Action Approval)
    VAA contains: emitter chain/address, sequence, payload
    """
    # Parse and verify the VAA
    is_valid: bool = IWormhole(WORMHOLE).parseAndVerifyVM(encoded_vaa)
    assert is_valid, "Invalid VAA"
    
    # Compute VAA hash for replay protection
    vaa_hash: bytes32 = keccak256(encoded_vaa)
    assert not self.processed_vaas[vaa_hash], "VAA already processed"
    
    # Mark as processed
    self.processed_vaas[vaa_hash] = True
    
    # Process the message payload
    # In real implementation, decode the VAA structure
    # self._process_payload(vaa.payload, vaa.emitter_chain_id)

@external
@payable
def transfer_tokens_via_wormhole(
    token: address,
    amount: uint256,
    recipient_chain: uint16,
    recipient: bytes32,
    nonce: uint32
) -> uint64:
    """
    Transfer tokens cross-chain via Wormhole Token Bridge
    """
    from vyper.interfaces import ERC20
    
    # Approve token bridge
    ERC20(token).approve(TOKEN_BRIDGE, amount)
    
    # Initiate transfer
    fee: uint256 = IWormhole(WORMHOLE).messageFee()
    assert msg.value >= fee, "Insufficient fee"
    
    sequence: uint64 = ITokenBridge(TOKEN_BRIDGE).transferTokens(
        token,
        amount,
        recipient_chain,
        recipient,
        0,      # No relayer fee
        nonce,
        value=fee
    )
    
    return sequence
```

---

## 4. Chainlink CCIP {#s4}

```python
# @version 0.4.0
# CCIPReceiver.vy
# Receive cross-chain messages via Chainlink CCIP

# CCIP Router interface
interface IRouter:
    def ccipSend(
        destination_chain_selector: uint64,
        message_receiver: address,
        message_data: Bytes[65536],
        fee_token_address: address
    ) -> bytes32: payable
    
    def getFee(
        destination_chain_selector: uint64,
        message_receiver: address,
        message_data: Bytes[65536],
        fee_token_address: address
    ) -> uint256: view

# Chain selectors (Chainlink CCIP specific)
CCIP_ETHEREUM_SELECTOR: constant(uint64) = 5009297550715157269
CCIP_OPTIMISM_SELECTOR: constant(uint64) = 3734403246176062136
CCIP_ARBITRUM_SELECTOR: constant(uint64) = 4949039107694359620
CCIP_POLYGON_SELECTOR: constant(uint64) = 4051577828743386545
CCIP_BASE_SELECTOR: constant(uint64) = 15971525489660198786

# Any2EVMMessage struct (received messages)
struct Any2EVMMessage:
    message_id: bytes32
    source_chain_selector: uint64
    sender: Bytes[40]
    data: Bytes[65536]
    token_amounts_token: DynArray[address, 10]
    token_amounts_amount: DynArray[uint256, 10]

CCIP_ROUTER: immutable(address)

owner: public(address)

# Allowlisted source chains and senders
allowlisted_chains: public(HashMap[uint64, bool])
allowlisted_senders: public(HashMap[address, bool])

# Processed messages
processed_messages: public(HashMap[bytes32, bool])

# Last received message
last_received_message_id: public(bytes32)
last_received_data: public(Bytes[65536])

event MessageSent:
    message_id: indexed(bytes32)
    destination_chain: uint64
    receiver: indexed(address)

event MessageReceived:
    message_id: indexed(bytes32)
    source_chain: uint64
    sender: Bytes[40]
    data: Bytes[65536]

@deploy
def __init__(router: address):
    CCIP_ROUTER = router
    self.owner = msg.sender

@external
@payable
def send_message(
    destination_chain_selector: uint64,
    receiver: address,
    data: Bytes[65536]
) -> bytes32:
    """
    Send a message via Chainlink CCIP
    @param destination_chain_selector CCIP chain selector
    @param receiver Address on destination chain
    @param data Message payload
    """
    # Estimate fee
    fee: uint256 = IRouter(CCIP_ROUTER).getFee(
        destination_chain_selector,
        receiver,
        data,
        empty(address)  # Pay in native token
    )
    
    assert msg.value >= fee, "Insufficient fee"
    
    # Send message
    message_id: bytes32 = IRouter(CCIP_ROUTER).ccipSend(
        destination_chain_selector,
        receiver,
        data,
        empty(address),
        value=fee
    )
    
    log MessageSent(message_id, destination_chain_selector, receiver)
    return message_id

@external
def ccipReceive(
    message_id: bytes32,
    source_chain_selector: uint64,
    sender: Bytes[40],
    data: Bytes[65536]
):
    """
    Receive a CCIP message
    Called by CCIP router when message arrives
    """
    # Only router can call
    assert msg.sender == CCIP_ROUTER, "Not CCIP router"
    
    # Check allowlist
    assert self.allowlisted_chains[source_chain_selector], "Chain not allowlisted"
    
    # Replay protection
    assert not self.processed_messages[message_id], "Message already processed"
    self.processed_messages[message_id] = True
    
    # Store received data
    self.last_received_message_id = message_id
    self.last_received_data = data
    
    # Process the message
    self._process_message(source_chain_selector, sender, data)
    
    log MessageReceived(message_id, source_chain_selector, sender, data)

@internal
def _process_message(
    source_chain: uint64,
    sender: Bytes[40],
    data: Bytes[65536]
):
    """Override in subcontracts to process messages"""
    pass

@external
def allowlist_chain(chain_selector: uint64, allowed: bool):
    """Add/remove a chain from allowlist"""
    assert msg.sender == self.owner, "Not owner"
    self.allowlisted_chains[chain_selector] = allowed

@external
def allowlist_sender(sender: address, allowed: bool):
    """Add/remove a sender from allowlist"""
    assert msg.sender == self.owner, "Not owner"
    self.allowlisted_senders[sender] = allowed
```

---

## 5. Multi-Chain Token (OFT Pattern) {#s5}

```python
# @version 0.4.0
# OFTToken.vy
# Omnichain Fungible Token (OFT) Pattern via LayerZero
# Allows same token to exist on multiple chains

from vyper.interfaces import ERC20

interface ILayerZeroEndpoint:
    def send(
        dst_chain_id: uint16,
        destination: Bytes[40],
        payload: Bytes[65536],
        refund_address: address,
        zero_pay_address: address,
        adapter_params: Bytes[65536]
    ): payable
    
    def estimateFees(
        dst_chain_id: uint16,
        user_application: address,
        payload: Bytes[65536],
        pay_in_zro: bool,
        adapter_params: Bytes[65536]
    ) -> (uint256, uint256): view

# Message types
PT_SEND: constant(uint8) = 0
PT_SEND_AND_CALL: constant(uint8) = 1

LZ_ENDPOINT: immutable(address)

# ERC-20 state
name: public(String[64])
symbol: public(String[32])
decimals: public(uint8)
total_supply: public(uint256)
balances: HashMap[address, uint256]
allowances: HashMap[address, HashMap[address, uint256]]

owner: public(address)

# LayerZero config
trusted_remotes: public(HashMap[uint16, Bytes[40]])
default_gas_limit: public(uint256)

# Cross-chain transfer limits
max_single_transfer: public(uint256)
daily_transfer_limit: public(uint256)
daily_transferred: public(HashMap[uint16, uint256])
daily_reset_time: public(HashMap[uint16, uint256])

event Transfer:
    sender: indexed(address)
    receiver: indexed(address)
    value: uint256

event Approval:
    owner: indexed(address)
    spender: indexed(address)
    value: uint256

event CrossChainTransferInitiated:
    from_: indexed(address)
    dst_chain_id: indexed(uint16)
    to: bytes32
    amount: uint256

event CrossChainTransferReceived:
    src_chain_id: indexed(uint16)
    to: indexed(address)
    amount: uint256

@deploy
def __init__(
    endpoint: address,
    token_name: String[64],
    token_symbol: String[32],
    initial_supply: uint256
):
    LZ_ENDPOINT = endpoint
    self.name = token_name
    self.symbol = token_symbol
    self.decimals = 18
    self.owner = msg.sender
    self.default_gas_limit = 200000
    self.max_single_transfer = 1_000_000 * 10**18
    self.daily_transfer_limit = 10_000_000 * 10**18
    
    if initial_supply > 0:
        self.total_supply = initial_supply
        self.balances[msg.sender] = initial_supply
        log Transfer(empty(address), msg.sender, initial_supply)

# ============================================================
# ERC-20 Functions
# ============================================================

@view
@external
def balanceOf(account: address) -> uint256:
    return self.balances[account]

@view
@external
def allowance(owner: address, spender: address) -> uint256:
    return self.allowances[owner][spender]

@external
def transfer(to: address, amount: uint256) -> bool:
    assert to != empty(address), "Transfer to zero"
    assert self.balances[msg.sender] >= amount, "Insufficient balance"
    self.balances[msg.sender] -= amount
    self.balances[to] += amount
    log Transfer(msg.sender, to, amount)
    return True

@external
def transferFrom(from_: address, to: address, amount: uint256) -> bool:
    assert to != empty(address), "Transfer to zero"
    assert self.balances[from_] >= amount, "Insufficient balance"
    assert self.allowances[from_][msg.sender] >= amount, "Insufficient allowance"
    self.allowances[from_][msg.sender] -= amount
    self.balances[from_] -= amount
    self.balances[to] += amount
    log Transfer(from_, to, amount)
    return True

@external
def approve(spender: address, amount: uint256) -> bool:
    self.allowances[msg.sender][spender] = amount
    log Approval(msg.sender, spender, amount)
    return True

# ============================================================
# OFT Cross-Chain Functions
# ============================================================

@external
@payable
def send_tokens(
    dst_chain_id: uint16,
    to: bytes32,
    amount: uint256,
    refund_address: address
):
    """
    Send tokens to another chain via LayerZero
    Burns tokens on this chain and mints on destination
    
    @param dst_chain_id Destination LayerZero chain ID
    @param to Recipient address on destination (as bytes32)
    @param amount Amount to transfer
    @param refund_address Address for excess fee refund
    """
    # Check limits
    assert amount <= self.max_single_transfer, "Exceeds single transfer limit"
    self._check_daily_limit(dst_chain_id, amount)
    
    # Check trusted remote exists
    assert len(self.trusted_remotes[dst_chain_id]) == 40, "No trusted remote"
    
    # Burn tokens from sender
    assert self.balances[msg.sender] >= amount, "Insufficient balance"
    self.balances[msg.sender] -= amount
    self.total_supply -= amount
    log Transfer(msg.sender, empty(address), amount)
    
    # Encode payload: [PT_SEND (1 byte)] [to (32 bytes)] [amount (32 bytes)]
    payload: Bytes[65] = concat(
        convert(PT_SEND, bytes1),
        to,
        convert(amount, bytes32)
    )
    
    # Send via LayerZero
    # Adapter params: [version (2 bytes)] [gas limit (32 bytes)]
    adapter_params: Bytes[34] = concat(
        convert(1, bytes2),  # version 1
        convert(self.default_gas_limit, bytes32)
    )
    
    ILayerZeroEndpoint(LZ_ENDPOINT).send(
        dst_chain_id,
        self.trusted_remotes[dst_chain_id],
        payload,
        refund_address,
        empty(address),
        adapter_params,
        value=msg.value
    )
    
    log CrossChainTransferInitiated(msg.sender, dst_chain_id, to, amount)

@external
def lzReceive(
    src_chain_id: uint16,
    src_address: Bytes[40],
    nonce: uint64,
    payload: Bytes[65536]
):
    """
    Receive cross-chain transfer from LayerZero
    Mints tokens to recipient
    """
    assert msg.sender == LZ_ENDPOINT, "Not endpoint"
    assert src_address == self.trusted_remotes[src_chain_id], "Untrusted source"
    
    # Decode payload
    pt: uint8 = convert(slice(payload, 0, 1), uint8)
    
    if pt == PT_SEND:
        # Extract recipient and amount
        to: address = convert(convert(slice(payload, 1, 32), bytes32), address)
        amount: uint256 = convert(slice(payload, 33, 32), uint256)
        
        # Mint tokens to recipient
        self.balances[to] += amount
        self.total_supply += amount
        
        log Transfer(empty(address), to, amount)
        log CrossChainTransferReceived(src_chain_id, to, amount)

@internal
def _check_daily_limit(chain_id: uint16, amount: uint256):
    """Check and update daily transfer limit"""
    # Reset if new day
    if block.timestamp >= self.daily_reset_time[chain_id] + 86400:
        self.daily_transferred[chain_id] = 0
        self.daily_reset_time[chain_id] = block.timestamp
    
    assert self.daily_transferred[chain_id] + amount <= self.daily_transfer_limit, \
        "Daily limit exceeded"
    
    self.daily_transferred[chain_id] += amount

@view
@external
def estimate_send_fee(
    dst_chain_id: uint16,
    to: bytes32,
    amount: uint256
) -> uint256:
    """Estimate LayerZero fee for cross-chain transfer"""
    payload: Bytes[65] = concat(
        convert(PT_SEND, bytes1),
        to,
        convert(amount, bytes32)
    )
    
    native_fee: uint256 = 0
    zro_fee: uint256 = 0
    
    native_fee, zro_fee = ILayerZeroEndpoint(LZ_ENDPOINT).estimateFees(
        dst_chain_id,
        self,
        payload,
        False,
        b""
    )
    
    return native_fee

# ============================================================
# Admin Functions
# ============================================================

@external
def set_trusted_remote(chain_id: uint16, remote: Bytes[40]):
    """Set trusted remote for a chain"""
    assert msg.sender == self.owner, "Not owner"
    assert len(remote) == 40, "Invalid remote"
    self.trusted_remotes[chain_id] = remote

@external
def set_transfer_limits(max_single: uint256, max_daily: uint256):
    """Update transfer limits"""
    assert msg.sender == self.owner, "Not owner"
    self.max_single_transfer = max_single
    self.daily_transfer_limit = max_daily
```

---

## 6. Cross-Chain Governance {#s6}

```python
# @version 0.4.0
# CrossChainGovernance.vy
# Governance that executes decisions across multiple chains

interface ILayerZeroEndpoint:
    def send(
        dst_chain_id: uint16,
        destination: Bytes[40],
        payload: Bytes[65536],
        refund_address: address,
        zero_pay_address: address,
        adapter_params: Bytes[65536]
    ): payable

# Governance message types
PROPOSAL_TYPE_PARAM_UPDATE: constant(uint8) = 1
PROPOSAL_TYPE_PAUSE: constant(uint8) = 2
PROPOSAL_TYPE_UPGRADE: constant(uint8) = 3
PROPOSAL_TYPE_TREASURY: constant(uint8) = 4

struct CrossChainProposal:
    id: uint256
    proposer: address
    target_chains: DynArray[uint16, 10]
    payload: Bytes[65536]
    vote_count: uint256
    required_votes: uint256
    executed: bool
    execution_deadline: uint256

LZ_ENDPOINT: immutable(address)

owner: public(address)

# Governance state
proposals: public(HashMap[uint256, CrossChainProposal])
proposal_count: public(uint256)
votes: public(HashMap[uint256, HashMap[address, bool]])

# Governance token (for voting)
governance_token: public(address)

# Chain configuration
trusted_remotes: HashMap[uint16, Bytes[40]]

# Executed proposals per chain
executed_proposals: public(HashMap[bytes32, bool])

event ProposalCreated:
    id: uint256
    proposer: indexed(address)
    target_chains: DynArray[uint16, 10]

event VoteCast:
    proposal_id: uint256
    voter: indexed(address)
    votes: uint256

event ProposalExecuted:
    proposal_id: uint256
    chains: DynArray[uint16, 10]

event CrossChainActionReceived:
    src_chain_id: indexed(uint16)
    proposal_id: uint256

@deploy
def __init__(endpoint: address, gov_token: address):
    LZ_ENDPOINT = endpoint
    self.owner = msg.sender
    self.governance_token = gov_token

@external
def create_proposal(
    target_chains: DynArray[uint16, 10],
    payload: Bytes[65536],
    voting_period: uint256
) -> uint256:
    """
    Create a cross-chain governance proposal
    """
    from vyper.interfaces import ERC20
    
    # Check proposer has enough tokens
    proposer_balance: uint256 = ERC20(self.governance_token).balanceOf(msg.sender)
    assert proposer_balance >= 10**18, "Insufficient governance tokens to propose"
    
    proposal_id: uint256 = self.proposal_count
    self.proposal_count += 1
    
    self.proposals[proposal_id] = CrossChainProposal(
        id=proposal_id,
        proposer=msg.sender,
        target_chains=target_chains,
        payload=payload,
        vote_count=0,
        required_votes=100 * 10**18,  # 100 tokens required
        executed=False,
        execution_deadline=block.timestamp + voting_period
    )
    
    log ProposalCreated(proposal_id, msg.sender, target_chains)
    return proposal_id

@external
def vote(proposal_id: uint256):
    """Vote on a cross-chain proposal"""
    from vyper.interfaces import ERC20
    
    assert proposal_id < self.proposal_count, "Invalid proposal"
    proposal: CrossChainProposal = self.proposals[proposal_id]
    
    assert block.timestamp < proposal.execution_deadline, "Voting ended"
    assert not self.votes[proposal_id][msg.sender], "Already voted"
    
    # Get voter's token balance
    voter_balance: uint256 = ERC20(self.governance_token).balanceOf(msg.sender)
    assert voter_balance > 0, "No voting power"
    
    # Record vote
    self.votes[proposal_id][msg.sender] = True
    self.proposals[proposal_id].vote_count += voter_balance
    
    log VoteCast(proposal_id, msg.sender, voter_balance)

@external
@payable
def execute_proposal(proposal_id: uint256):
    """
    Execute an approved proposal on all target chains
    """
    assert proposal_id < self.proposal_count, "Invalid proposal"
    proposal: CrossChainProposal = self.proposals[proposal_id]
    
    assert not proposal.executed, "Already executed"
    assert proposal.vote_count >= proposal.required_votes, "Insufficient votes"
    assert block.timestamp < proposal.execution_deadline, "Deadline passed"
    
    self.proposals[proposal_id].executed = True
    
    # Send governance message to each target chain
    for chain_id: uint16 in proposal.target_chains:
        remote: Bytes[40] = self.trusted_remotes[chain_id]
        if len(remote) != 40:
            continue
        
        # Encode governance message
        gov_message: Bytes[65572] = concat(
            convert(proposal_id, bytes32),
            proposal.payload
        )
        
        ILayerZeroEndpoint(LZ_ENDPOINT).send(
            chain_id,
            remote,
            gov_message,
            msg.sender,
            empty(address),
            b"",
            value=msg.value / len(proposal.target_chains)
        )
    
    log ProposalExecuted(proposal_id, proposal.target_chains)

@external
def lzReceive(
    src_chain_id: uint16,
    src_address: Bytes[40],
    nonce: uint64,
    payload: Bytes[65536]
):
    """Execute a governance action received from another chain"""
    assert msg.sender == LZ_ENDPOINT, "Not endpoint"
    assert src_address == self.trusted_remotes[src_chain_id], "Untrusted source"
    
    # Extract proposal ID
    proposal_id: uint256 = convert(slice(payload, 0, 32), uint256)
    action_payload: Bytes[65504] = slice(payload, 32, 65504)
    
    # Replay protection
    message_id: bytes32 = keccak256(
        concat(convert(src_chain_id, bytes2), convert(nonce, bytes8))
    )
    assert not self.executed_proposals[message_id], "Already executed"
    self.executed_proposals[message_id] = True
    
    # Execute the governance action
    # In real implementation, decode and execute action_payload
    log CrossChainActionReceived(src_chain_id, proposal_id)

@external
def set_trusted_remote(chain_id: uint16, remote: Bytes[40]):
    """Configure trusted remote"""
    assert msg.sender == self.owner, "Not owner"
    self.trusted_remotes[chain_id] = remote
```

---

## 7. CrossChainToken Contract {#s7}

```python
# @version 0.4.0
# CrossChainToken.vy
# Complete cross-chain token implementation
# Supports LayerZero OFT pattern with rate limiting and security

from vyper.interfaces import ERC20

interface ILayerZeroEndpoint:
    def send(
        dst_chain_id: uint16,
        destination: Bytes[40],
        payload: Bytes[65536],
        refund_address: address,
        zero_pay_address: address,
        adapter_params: Bytes[65536]
    ): payable
    
    def estimateFees(
        dst_chain_id: uint16,
        user_application: address,
        payload: Bytes[65536],
        pay_in_zro: bool,
        adapter_params: Bytes[65536]
    ) -> (uint256, uint256): view

# ============================================================
# Constants
# ============================================================

PT_SEND: constant(uint8) = 0
LZ_ENDPOINT: immutable(address)
MAX_DAILY_TRANSFER: constant(uint256) = 1_000_000 * 10**18
TRANSFER_COOLDOWN: constant(uint256) = 60  # 1 minute between large transfers

# ============================================================
# Storage
# ============================================================

name: public(String[64])
symbol: public(String[32])
decimals: public(uint8)
total_supply: public(uint256)

balances: HashMap[address, uint256]
allowances: HashMap[address, HashMap[address, uint256]]

owner: public(address)
paused: public(bool)

# Cross-chain config
trusted_remotes: public(HashMap[uint16, Bytes[40]])
chain_paused: public(HashMap[uint16, bool])

# Rate limiting
daily_outflow: public(HashMap[uint16, uint256])
daily_reset: public(HashMap[uint16, uint256])
user_last_transfer: public(HashMap[address, uint256])
large_transfer_threshold: public(uint256)

# Cross-chain tracking
total_sent: public(HashMap[uint16, uint256])
total_received: public(HashMap[uint16, uint256])
transfer_count: public(uint256)

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

event CrossChainTransfer:
    from_: indexed(address)
    dst_chain: indexed(uint16)
    to: bytes32
    amount: uint256
    transfer_id: uint256

event CrossChainReceive:
    src_chain: indexed(uint16)
    to: indexed(address)
    amount: uint256
    transfer_id: uint256

event ChainPaused:
    chain_id: indexed(uint16)

event ChainResumed:
    chain_id: indexed(uint16)

# ============================================================
# Constructor
# ============================================================

@deploy
def __init__(
    endpoint: address,
    token_name: String[64],
    token_symbol: String[32],
    initial_supply: uint256
):
    LZ_ENDPOINT = endpoint
    self.name = token_name
    self.symbol = token_symbol
    self.decimals = 18
    self.owner = msg.sender
    self.large_transfer_threshold = 10000 * 10**18  # 10k tokens
    
    if initial_supply > 0:
        self.total_supply = initial_supply
        self.balances[msg.sender] = initial_supply
        log Transfer(empty(address), msg.sender, initial_supply)

# ============================================================
# ERC-20 Standard Functions
# ============================================================

@view
@external
def balanceOf(account: address) -> uint256:
    return self.balances[account]

@view
@external
def allowance(owner: address, spender: address) -> uint256:
    return self.allowances[owner][spender]

@external
def transfer(to: address, amount: uint256) -> bool:
    assert not self.paused, "Token paused"
    assert to != empty(address), "Transfer to zero"
    assert self.balances[msg.sender] >= amount, "Insufficient balance"
    
    self.balances[msg.sender] -= amount
    self.balances[to] += amount
    log Transfer(msg.sender, to, amount)
    return True

@external
def transferFrom(from_: address, to: address, amount: uint256) -> bool:
    assert not self.paused, "Token paused"
    assert to != empty(address), "Transfer to zero"
    assert self.balances[from_] >= amount, "Insufficient balance"
    assert self.allowances[from_][msg.sender] >= amount, "Insufficient allowance"
    
    self.allowances[from_][msg.sender] -= amount
    self.balances[from_] -= amount
    self.balances[to] += amount
    log Transfer(from_, to, amount)
    return True

@external
def approve(spender: address, amount: uint256) -> bool:
    self.allowances[msg.sender][spender] = amount
    log Approval(msg.sender, spender, amount)
    return True

# ============================================================
# Cross-Chain Transfer Functions
# ============================================================

@external
@payable
def cross_chain_transfer(
    dst_chain_id: uint16,
    to: bytes32,
    amount: uint256
) -> uint256:
    """
    Transfer tokens to another chain
    Burns tokens here, mints on destination
    """
    assert not self.paused, "Token paused"
    assert not self.chain_paused[dst_chain_id], "Destination chain paused"
    assert amount > 0, "Amount must be positive"
    assert self.balances[msg.sender] >= amount, "Insufficient balance"
    
    # Check trusted remote
    assert len(self.trusted_remotes[dst_chain_id]) == 40, "Chain not supported"
    
    # Rate limiting checks
    self._check_rate_limits(dst_chain_id, amount)
    self._check_user_cooldown(amount)
    
    # Burn tokens
    self.balances[msg.sender] -= amount
    self.total_supply -= amount
    log Transfer(msg.sender, empty(address), amount)
    
    # Update tracking
    self.total_sent[dst_chain_id] += amount
    transfer_id: uint256 = self.transfer_count
    self.transfer_count += 1
    
    # Encode payload
    payload: Bytes[65] = concat(
        convert(PT_SEND, bytes1),
        to,
        convert(amount, bytes32)
    )
    
    # Encode adapter params with gas limit
    adapter_params: Bytes[34] = concat(
        convert(1, bytes2),
        convert(convert(300000, uint256), bytes32)
    )
    
    # Send via LayerZero
    ILayerZeroEndpoint(LZ_ENDPOINT).send(
        dst_chain_id,
        self.trusted_remotes[dst_chain_id],
        payload,
        msg.sender,
        empty(address),
        adapter_params,
        value=msg.value
    )
    
    log CrossChainTransfer(msg.sender, dst_chain_id, to, amount, transfer_id)
    return transfer_id

@external
def lzReceive(
    src_chain_id: uint16,
    src_address: Bytes[40],
    nonce: uint64,
    payload: Bytes[65536]
):
    """Receive cross-chain transfer from LayerZero"""
    assert msg.sender == LZ_ENDPOINT, "Not LayerZero"
    assert src_address == self.trusted_remotes[src_chain_id], "Untrusted"
    assert not self.chain_paused[src_chain_id], "Source chain paused"
    
    pt: uint8 = convert(slice(payload, 0, 1), uint8)
    
    if pt == PT_SEND:
        to: address = convert(convert(slice(payload, 1, 32), bytes32), address)
        amount: uint256 = convert(slice(payload, 33, 32), uint256)
        
        # Mint tokens on this chain
        self.balances[to] += amount
        self.total_supply += amount
        
        self.total_received[src_chain_id] += amount
        
        log Transfer(empty(address), to, amount)
        log CrossChainReceive(src_chain_id, to, amount, self.transfer_count)
        self.transfer_count += 1

@internal
def _check_rate_limits(chain_id: uint16, amount: uint256):
    """Check daily outflow rate limits"""
    if block.timestamp >= self.daily_reset[chain_id] + 86400:
        self.daily_outflow[chain_id] = 0
        self.daily_reset[chain_id] = block.timestamp
    
    assert self.daily_outflow[chain_id] + amount <= MAX_DAILY_TRANSFER, \
        "Daily limit exceeded"
    self.daily_outflow[chain_id] += amount

@internal
def _check_user_cooldown(amount: uint256):
    """Enforce cooldown for large transfers"""
    if amount >= self.large_transfer_threshold:
        assert block.timestamp >= self.user_last_transfer[msg.sender] + TRANSFER_COOLDOWN, \
            "Transfer cooldown active"
    self.user_last_transfer[msg.sender] = block.timestamp

# ============================================================
# Admin Functions
# ============================================================

@external
def set_trusted_remote(chain_id: uint16, remote: Bytes[40]):
    assert msg.sender == self.owner, "Not owner"
    self.trusted_remotes[chain_id] = remote

@external
def pause_chain(chain_id: uint16):
    assert msg.sender == self.owner, "Not owner"
    self.chain_paused[chain_id] = True
    log ChainPaused(chain_id)

@external
def resume_chain(chain_id: uint16):
    assert msg.sender == self.owner, "Not owner"
    self.chain_paused[chain_id] = False
    log ChainResumed(chain_id)

@view
@external
def estimate_fee(dst_chain_id: uint16, amount: uint256) -> uint256:
    to: bytes32 = empty(bytes32)
    payload: Bytes[65] = concat(convert(PT_SEND, bytes1), to, convert(amount, bytes32))
    fee: uint256 = 0
    zro: uint256 = 0
    fee, zro = ILayerZeroEndpoint(LZ_ENDPOINT).estimateFees(
        dst_chain_id, self, payload, False, b""
    )
    return fee
```

---

## 8. Security Considerations {#s8}

```python
# @version 0.4.0
# CrossChainSecurity.vy
# Security patterns for cross-chain protocols

# ============================================================
# 1. Message Ordering
# ============================================================

# LayerZero guarantees message ordering per channel
# But different channels can deliver out of order
# Use sequential nonces to verify ordering

last_received_nonce: public(HashMap[uint16, uint64])

@internal
def _enforce_message_order(src_chain: uint16, nonce: uint64):
    """Ensure messages are processed in order"""
    expected: uint64 = self.last_received_nonce[src_chain] + 1
    assert nonce == expected, "Out of order message"
    self.last_received_nonce[src_chain] = nonce

# ============================================================
# 2. Double-spend Prevention
# ============================================================

processed_nonces: public(HashMap[uint16, HashMap[uint64, bool]])

@internal
def _mark_nonce_processed(chain_id: uint16, nonce: uint64):
    """Prevent processing same message twice"""
    assert not self.processed_nonces[chain_id][nonce], "Nonce already used"
    self.processed_nonces[chain_id][nonce] = True

# ============================================================
# 3. Trusted Path Verification
# ============================================================

# NEVER trust msg.sender alone for cross-chain messages
# Always verify the complete path:
# 1. msg.sender == LayerZero endpoint
# 2. src_address == trusted remote for that chain
# 3. Nonce is sequential and unused

@internal
def _verify_lz_message(
    endpoint: address,
    src_chain_id: uint16,
    src_address: Bytes[40],
    trusted_remotes: HashMap[uint16, Bytes[40]]
):
    """Complete LayerZero message authentication"""
    # Must come from the official endpoint
    assert msg.sender == endpoint, "Not from endpoint"
    
    # Source must be our trusted remote on that chain
    assert len(trusted_remotes[src_chain_id]) == 40, "Unknown chain"
    assert src_address == trusted_remotes[src_chain_id], "Untrusted source"

# ============================================================
# 4. Rate Limiting and Circuit Breakers
# ============================================================

paused: bool
emergency_admin: public(address)
total_outflow_24h: public(uint256)
outflow_window_start: public(uint256)
max_outflow_24h: public(uint256)

@internal
def _check_outflow_limit(amount: uint256):
    """Circuit breaker: pause if too much flows out"""
    if block.timestamp >= self.outflow_window_start + 86400:
        self.total_outflow_24h = 0
        self.outflow_window_start = block.timestamp
    
    self.total_outflow_24h += amount
    
    # Auto-pause if limit exceeded
    if self.total_outflow_24h > self.max_outflow_24h:
        self.paused = True
        # Emit emergency event here

# ============================================================
# 5. Message Validation
# ============================================================

@internal
def _validate_payload(payload: Bytes[65536]) -> bool:
    """Validate cross-chain message payload"""
    # Check minimum length
    if len(payload) < 1:
        return False
    
    # Check message type is known
    pt: uint8 = convert(slice(payload, 0, 1), uint8)
    if pt != 0:  # Only PT_SEND supported
        return False
    
    # Check payload length matches expected
    if len(payload) != 65:  # 1 + 32 + 32
        return False
    
    return True
```

### สรุปความเสี่ยง Cross-Chain

| ความเสี่ยง | ผลกระทบ | การป้องกัน |
|---|---|---|
| Fake endpoint | Full control | Verify endpoint address |
| Untrusted sender | Fake messages | Verify trusted remote |
| Message replay | Double mint/spend | Track processed nonces |
| Out-of-order delivery | State inconsistency | Sequential nonces |
| Chain suspension | Loss of funds | Circuit breakers |
| Oracle manipulation | Wrong prices | Use multiple oracles |

---

## สรุป

Cross-chain messaging เป็นพื้นฐานของ multi-chain DeFi:
- **LayerZero**: Oracle + Relayer mechanism, OFT pattern
- **Wormhole**: Guardian network, VAA proofs
- **CCIP**: Chainlink DON, built-in rate limiting
- **Security**: Always verify complete message path
- **Rate Limits**: Protect against large-scale attacks

---
[← Previous Part](part_069_layer2_integration.md) | [→ Next Part](part_071_protocol_fees.md)
