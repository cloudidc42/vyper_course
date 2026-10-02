# Part 069: Layer 2 Integration (การ Deploy บน Layer 2)

## สารบัญ
1. [บทนำ Layer 2](#s1)
2. [Optimism vs Arbitrum](#s2)
3. [L1 to L2 Messaging](#s3)
4. [Sequencer Uptime Check](#s4)
5. [Optimism-specific Opcodes](#s5)
6. [Multi-chain Deployment](#s6)
7. [Cross-Layer State Sync](#s7)
8. [Security Considerations](#s8)

---

## 1. บทนำ Layer 2 {#s1}

**Layer 2 (L2)** คือ blockchain ที่ทำงานบน Ethereum (L1) โดย:
- ประมวลผล transactions ได้เร็วกว่า
- ค่า gas ถูกกว่ามาก
- ใช้ L1 เป็น security layer

### ประเภทของ L2

**Optimistic Rollups** (Optimism, Arbitrum):
- สันนิษฐานว่า transactions ถูกต้อง
- มี "fraud proof" period (~7 วัน) สำหรับ withdrawal
- Compatible กับ EVM

**ZK Rollups** (zkSync, Polygon zkEVM, Starknet):
- ใช้ zero-knowledge proofs
- การ verify รวดเร็วกว่า
- Faster withdrawals

```python
# @version 0.4.0
# ChainConfig.vy
# Configuration for different networks

# Chain IDs
ETHEREUM_MAINNET: constant(uint256) = 1
OPTIMISM_MAINNET: constant(uint256) = 10
ARBITRUM_ONE: constant(uint256) = 42161
POLYGON_MAINNET: constant(uint256) = 137
BASE_MAINNET: constant(uint256) = 8453
ZKSYNC_ERA: constant(uint256) = 324

# Check current chain
@view
@external
def is_l2() -> bool:
    """Check if running on an L2 network"""
    cid: uint256 = chain.id
    return (
        cid == OPTIMISM_MAINNET or
        cid == ARBITRUM_ONE or
        cid == POLYGON_MAINNET or
        cid == BASE_MAINNET or
        cid == ZKSYNC_ERA
    )

@view
@external
def is_optimism() -> bool:
    """Check if running on Optimism"""
    return chain.id == OPTIMISM_MAINNET

@view
@external
def is_arbitrum() -> bool:
    """Check if running on Arbitrum"""
    return chain.id == ARBITRUM_ONE
```

---

## 2. Optimism vs Arbitrum {#s2}

### ความแตกต่างหลัก

| Feature | Optimism | Arbitrum |
|---|---|---|
| L1 Gas Price | OVM_GasPriceOracle | ArbGasInfo |
| L1 Block Number | block.number (อาจต่างกัน) | ArbSys precompile |
| Fraud Proof | Single-round | Multi-round |
| Precompiles | OVM-specific | Arb-specific |

```python
# @version 0.4.0
# OptimismGasOracle.vy
# Interact with Optimism's L1 gas price oracle
# Address: 0x420000000000000000000000000000000000000F

interface IOptimismGasPriceOracle:
    def l1BaseFee() -> uint256: view
    def overhead() -> uint256: view
    def scalar() -> uint256: view
    def decimals() -> uint256: view
    def getL1Fee(data: Bytes[65536]) -> uint256: view
    def getL1GasUsed(data: Bytes[65536]) -> uint256: view

# Optimism L1 Gas Price Oracle (predeploy address)
OPTIMISM_GAS_ORACLE: constant(address) = 0x420000000000000000000000000000000000000F

@view
@external
def get_l1_gas_fee(tx_data: Bytes[65536]) -> uint256:
    """
    Get the L1 data fee for a transaction on Optimism
    This is the fee for posting calldata to Ethereum L1
    """
    if chain.id == 10:  # Optimism
        return IOptimismGasPriceOracle(OPTIMISM_GAS_ORACLE).getL1Fee(tx_data)
    return 0

@view
@external
def get_l1_base_fee() -> uint256:
    """Get current L1 base fee from Optimism oracle"""
    if chain.id == 10:
        return IOptimismGasPriceOracle(OPTIMISM_GAS_ORACLE).l1BaseFee()
    return 0
```

```python
# @version 0.4.0
# ArbitrumUtils.vy
# Utilities for Arbitrum-specific features
# ArbSys precompile: 0x0000000000000000000000000000000000000064

interface IArbSys:
    def arbBlockNumber() -> uint256: view
    def arbBlockHash(arbBlockNum: uint256) -> bytes32: view
    def arbChainID() -> uint256: view

interface IArbGasInfo:
    def getL1BaseFeeEstimate() -> uint256: view
    def getMinimumGasPrice() -> uint256: view
    def getL1GasPrice() -> uint256: view

ARBSYS: constant(address) = 0x0000000000000000000000000000000000000064
ARB_GAS_INFO: constant(address) = 0x000000000000000000000000000000000000006C

@view
@external
def get_arb_block_number() -> uint256:
    """
    Get Arbitrum-specific block number
    Different from L1 block number
    """
    if chain.id == 42161:  # Arbitrum One
        return IArbSys(ARBSYS).arbBlockNumber()
    return block.number

@view
@external
def get_l1_gas_price_estimate() -> uint256:
    """Get estimated L1 gas price on Arbitrum"""
    if chain.id == 42161:
        return IArbGasInfo(ARB_GAS_INFO).getL1GasPrice()
    return 0
```

---

## 3. L1 to L2 Messaging {#s3}

### Optimism Messenger

```python
# @version 0.4.0
# OptimismL1Sender.vy
# Send messages from L1 Ethereum to Optimism L2
# Deploy this on L1 (Ethereum mainnet)

# Optimism Cross Domain Messenger
interface IL1CrossDomainMessenger:
    def sendMessage(
        target: address,       # L2 contract address
        message: Bytes[65536], # Encoded function call
        gas_limit: uint32      # Gas for L2 execution
    ): nonpayable
    
    def xDomainMessageSender() -> address: view

# L1 Cross Domain Messenger address
L1_CROSS_DOMAIN_MESSENGER: constant(address) = 0x25ace71c97B33Cc4729CF772ae268934F7ab5fA1  # Mainnet

owner: public(address)
l2_contract: public(address)

event MessageSentToL2:
    target: indexed(address)
    data_hash: bytes32
    gas_limit: uint32

@deploy
def __init__(l2_target: address):
    self.owner = msg.sender
    self.l2_contract = l2_target

@external
def send_to_l2(data: Bytes[65536], gas_limit: uint32):
    """
    Send a message from L1 to L2 via Optimism messenger
    @param data Encoded function call for L2 contract
    @param gas_limit Gas to provide for L2 execution
    """
    assert msg.sender == self.owner, "Not owner"
    
    IL1CrossDomainMessenger(L1_CROSS_DOMAIN_MESSENGER).sendMessage(
        self.l2_contract,
        data,
        gas_limit
    )
    
    log MessageSentToL2(self.l2_contract, keccak256(data), gas_limit)

@external
def send_update_to_l2(new_value: uint256):
    """
    Send a state update to L2
    Encodes the updateState(uint256) function call
    """
    # Encode function call: updateState(new_value)
    # Function selector: keccak256("updateState(uint256)")[:4]
    call_data: Bytes[36] = concat(
        b"\x12\x34\x56\x78",  # Replace with actual 4-byte selector
        convert(new_value, bytes32)
    )
    
    IL1CrossDomainMessenger(L1_CROSS_DOMAIN_MESSENGER).sendMessage(
        self.l2_contract,
        call_data,
        200000  # 200k gas
    )
```

```python
# @version 0.4.0
# OptimismL2Receiver.vy
# Receive messages from L1 on Optimism L2
# Deploy this on Optimism

# L2 Cross Domain Messenger
interface IL2CrossDomainMessenger:
    def xDomainMessageSender() -> address: view

# L2 Messenger predeploy address
L2_CROSS_DOMAIN_MESSENGER: constant(address) = 0x4200000000000000000000000000000000000007

owner: public(address)
l1_contract: public(address)  # The trusted L1 sender

state: public(uint256)
last_l1_message: public(bytes32)

event StateUpdatedFromL1:
    new_value: uint256
    l1_sender: indexed(address)

@deploy
def __init__(l1_sender: address):
    self.owner = msg.sender
    self.l1_contract = l1_sender

@internal
def _only_from_l1():
    """
    Verify message came from our trusted L1 contract
    Critical security check for L1->L2 messaging
    """
    # Must be called by L2 messenger
    assert msg.sender == L2_CROSS_DOMAIN_MESSENGER, "Not L2 messenger"
    
    # The original L1 sender must be our contract
    l1_sender: address = IL2CrossDomainMessenger(L2_CROSS_DOMAIN_MESSENGER).xDomainMessageSender()
    assert l1_sender == self.l1_contract, "Unauthorized L1 sender"

@external
def updateState(new_value: uint256):
    """
    Update state from L1 message
    Only callable via the L2 messenger from our L1 contract
    """
    self._only_from_l1()
    
    old_value: uint256 = self.state
    self.state = new_value
    self.last_l1_message = keccak256(
        concat(convert(new_value, bytes32), convert(block.timestamp, bytes32))
    )
    
    log StateUpdatedFromL1(new_value, self.l1_contract)
```

### Arbitrum L1 to L2

```python
# @version 0.4.0
# ArbitrumL1Bridge.vy
# Send messages from L1 to Arbitrum L2

interface IArbitrumInbox:
    def createRetryableTicket(
        to: address,
        l2_call_value: uint256,
        max_submission_cost: uint256,
        excess_fee_refund_address: address,
        call_value_refund_address: address,
        gas_limit: uint256,
        max_fee_per_gas: uint256,
        data: Bytes[65536]
    ) -> uint256: payable

# Arbitrum Inbox address on L1
ARBITRUM_INBOX: constant(address) = 0x4Dbd4fc535Ac27206064B68FfCf827b0A60BAB3f  # Mainnet

owner: public(address)
l2_target: public(address)

@deploy
def __init__(l2_address: address):
    self.owner = msg.sender
    self.l2_target = l2_address

@external
@payable
def send_to_arbitrum(
    call_data: Bytes[65536],
    gas_limit: uint256,
    max_fee_per_gas: uint256,
    submission_cost: uint256
):
    """
    Send a message to Arbitrum L2 via retryable ticket
    @param call_data Encoded function call
    @param gas_limit Gas for L2 execution
    @param max_fee_per_gas Max gas price on L2
    @param submission_cost Cost for submitting the ticket
    """
    assert msg.sender == self.owner, "Not owner"
    
    # Must provide enough ETH: submission_cost + gas_limit * max_fee_per_gas
    required_eth: uint256 = submission_cost + gas_limit * max_fee_per_gas
    assert msg.value >= required_eth, "Insufficient ETH"
    
    IArbitrumInbox(ARBITRUM_INBOX).createRetryableTicket(
        self.l2_target,
        0,               # No ETH to L2 contract
        submission_cost,
        msg.sender,      # Refund excess to sender
        msg.sender,
        gas_limit,
        max_fee_per_gas,
        call_data,
        value=msg.value
    )
```

---

## 4. Sequencer Uptime Check {#s4}

เมื่อ sequencer ล้ม contract ควรหยุดทำงานบางอย่าง เช่น liquidation

```python
# @version 0.4.0
# SequencerUptimeChecker.vy
# Check Chainlink L2 Sequencer Uptime Feed
# Critical for DeFi protocols on L2 to prevent bad liquidations

# Chainlink Sequencer Uptime Feed interface
interface IChainlinkAggregator:
    def latestRoundData() -> (uint80, int256, uint256, uint256, uint80): view
    def decimals() -> uint8: view

# Sequencer Uptime Feed addresses
# Optimism: 0x371EAD81c9102C9BF4874A9075FFFf170F2Ee389
# Arbitrum: 0xFdB631F5EE196F0ed6FAa767959853A9F217697D
# Base: 0xBCF85224fc0756B9Fa45aA7892530B47e10b6058

# Grace period after sequencer restart before trusting prices
GRACE_PERIOD: constant(uint256) = 3600  # 1 hour

owner: public(address)
sequencer_feed: public(address)
grace_period: public(uint256)

# Status tracking
last_sequencer_down: public(uint256)
is_sequencer_grace_period: public(bool)

event SequencerDownDetected:
    timestamp: uint256

event SequencerBackUp:
    timestamp: uint256
    grace_period_end: uint256

@deploy
def __init__(seq_feed: address):
    self.owner = msg.sender
    self.sequencer_feed = seq_feed
    self.grace_period = GRACE_PERIOD

@view
@internal
def _is_sequencer_up() -> (bool, uint256):
    """
    Check if the L2 sequencer is operational
    @return (is_up, started_at) - whether up and when it started
    """
    round_id: uint80 = 0
    answer: int256 = 0
    started_at: uint256 = 0
    updated_at: uint256 = 0
    answered_in_round: uint80 = 0
    
    round_id, answer, started_at, updated_at, answered_in_round = \
        IChainlinkAggregator(self.sequencer_feed).latestRoundData()
    
    # answer == 0 means sequencer is UP
    # answer == 1 means sequencer is DOWN
    is_up: bool = answer == 0
    
    return is_up, started_at

@view
@external
def check_sequencer() -> (bool, bool, uint256):
    """
    Check sequencer status and grace period
    @return (is_up, is_in_grace_period, grace_period_ends)
    """
    is_up: bool = False
    started_at: uint256 = 0
    is_up, started_at = self._is_sequencer_up()
    
    if not is_up:
        return False, False, 0
    
    # Check if we're in grace period after restart
    grace_end: uint256 = started_at + self.grace_period
    in_grace: bool = block.timestamp < grace_end
    
    return True, in_grace, grace_end

@internal
def _require_sequencer_up():
    """
    Revert if sequencer is down or in grace period
    Call this before any price-sensitive operations
    """
    is_up: bool = False
    started_at: uint256 = 0
    is_up, started_at = self._is_sequencer_up()
    
    assert is_up, "Sequencer is down"
    
    # Also check grace period
    assert block.timestamp >= started_at + self.grace_period, \
        "Sequencer just restarted - in grace period"

@view
@external
def is_safe_to_operate() -> bool:
    """
    Check if it's safe to perform operations
    Returns False if sequencer is down or in grace period
    """
    is_up: bool = False
    started_at: uint256 = 0
    is_up, started_at = self._is_sequencer_up()
    
    if not is_up:
        return False
    
    return block.timestamp >= started_at + self.grace_period
```

### DeFi Protocol with Sequencer Check

```python
# @version 0.4.0
# L2SafeProtocol.vy
# DeFi protocol that checks sequencer before price-sensitive operations

interface IChainlinkAggregator:
    def latestRoundData() -> (uint80, int256, uint256, uint256, uint80): view

interface ISequencerUptimeFeed:
    def latestRoundData() -> (uint80, int256, uint256, uint256, uint80): view

# Contract addresses (Optimism example)
SEQUENCER_UPTIME_FEED: immutable(address)
ETH_USD_PRICE_FEED: immutable(address)

GRACE_PERIOD: constant(uint256) = 3600
PRICE_STALENESS_THRESHOLD: constant(uint256) = 3600  # 1 hour

owner: public(address)
paused: public(bool)

# User positions
collateral: public(HashMap[address, uint256])
debt: public(HashMap[address, uint256])

event Liquidation:
    user: indexed(address)
    collateral_seized: uint256
    debt_repaid: uint256

event SequencerPause:
    reason: String[50]

@deploy
def __init__(seq_feed: address, price_feed: address):
    SEQUENCER_UPTIME_FEED = seq_feed
    ETH_USD_PRICE_FEED = price_feed
    self.owner = msg.sender

@internal
def _check_sequencer_and_price() -> uint256:
    """
    Check sequencer is up and get fresh price
    @return Current ETH price in USD (8 decimals)
    """
    # 1. Check sequencer
    seq_round_id: uint80 = 0
    seq_answer: int256 = 0
    seq_started_at: uint256 = 0
    seq_updated_at: uint256 = 0
    seq_answered_in_round: uint80 = 0
    
    seq_round_id, seq_answer, seq_started_at, seq_updated_at, seq_answered_in_round = \
        ISequencerUptimeFeed(SEQUENCER_UPTIME_FEED).latestRoundData()
    
    # 0 = sequencer up, 1 = down
    assert seq_answer == 0, "Sequencer is down"
    
    # Check grace period
    assert block.timestamp >= seq_started_at + GRACE_PERIOD, "Sequencer in grace period"
    
    # 2. Get price with staleness check
    price_round_id: uint80 = 0
    price_answer: int256 = 0
    price_started_at: uint256 = 0
    price_updated_at: uint256 = 0
    price_answered_in_round: uint80 = 0
    
    price_round_id, price_answer, price_started_at, price_updated_at, price_answered_in_round = \
        IChainlinkAggregator(ETH_USD_PRICE_FEED).latestRoundData()
    
    # Check price staleness
    assert block.timestamp - price_updated_at <= PRICE_STALENESS_THRESHOLD, \
        "Price feed stale"
    
    assert price_answer > 0, "Invalid price"
    
    return convert(price_answer, uint256)

@external
def liquidate(user: address):
    """
    Liquidate undercollateralized position
    ONLY works when sequencer is up with fresh prices
    """
    assert not self.paused, "Protocol paused"
    
    # Get verified price (reverts if sequencer is down)
    eth_price: uint256 = self._check_sequencer_and_price()
    
    # Check if position is liquidatable
    collateral_value: uint256 = self.collateral[user] * eth_price / 10**8
    
    # 150% collateralization ratio required
    required_collateral: uint256 = self.debt[user] * 150 / 100
    
    assert collateral_value < required_collateral, "Position is healthy"
    
    # Execute liquidation
    seized: uint256 = self.collateral[user]
    debt_amount: uint256 = self.debt[user]
    
    self.collateral[user] = 0
    self.debt[user] = 0
    
    # Transfer collateral to liquidator (simplified)
    self.collateral[msg.sender] += seized
    
    log Liquidation(user, seized, debt_amount)

@external
def emergency_pause():
    """Pause protocol if sequencer issues detected"""
    assert msg.sender == self.owner, "Not owner"
    self.paused = True
    log SequencerPause("Manual pause by owner")
```

---

## 5. Optimism-specific Opcodes {#s5}

```python
# @version 0.4.0
# OptimismSpecific.vy
# Optimism-specific functionality and opcodes

# Optimism predeploy contracts
L1_BLOCK_INFO: constant(address) = 0x4200000000000000000000000000000000000015
L2_TO_L1_MESSAGE_PASSER: constant(address) = 0x4200000000000000000000000000000000000016
GAS_PRICE_ORACLE: constant(address) = 0x420000000000000000000000000000000000000F

# L1 Block Info interface (Optimism specific)
interface IL1Block:
    def number() -> uint64: view
    def timestamp() -> uint64: view
    def basefee() -> uint256: view
    def hash_() -> bytes32: view
    def sequence_number() -> uint64: view
    def batcher_hash() -> bytes32: view
    def l1_fee_overhead() -> uint256: view
    def l1_fee_scalar() -> uint256: view

interface IGasPriceOracle:
    def l1BaseFee() -> uint256: view
    def gasPrice() -> uint256: view
    def baseFee() -> uint256: view
    def overhead() -> uint256: view
    def scalar() -> uint256: view

owner: public(address)

@deploy
def __init__():
    self.owner = msg.sender

@view
@external
def get_l1_block_number() -> uint64:
    """
    Get the L1 block number corresponding to this L2 block
    Only works on Optimism
    """
    assert chain.id == 10, "Not Optimism"
    return IL1Block(L1_BLOCK_INFO).number()

@view
@external
def get_l1_timestamp() -> uint64:
    """Get the L1 timestamp for this L2 block"""
    assert chain.id == 10, "Not Optimism"
    return IL1Block(L1_BLOCK_INFO).timestamp()

@view
@external
def get_l1_base_fee() -> uint256:
    """Get current L1 base fee from the oracle"""
    assert chain.id == 10, "Not Optimism"
    return IL1Block(L1_BLOCK_INFO).basefee()

@view
@external
def get_l1_block_hash() -> bytes32:
    """Get the hash of the corresponding L1 block"""
    assert chain.id == 10, "Not Optimism"
    return IL1Block(L1_BLOCK_INFO).hash_()

@view
@external
def calculate_l1_data_fee(tx_data: Bytes[65536]) -> uint256:
    """
    Calculate the L1 data fee for calldata
    This is what gets charged on top of L2 execution cost
    """
    assert chain.id == 10, "Not Optimism"
    
    # Count zero and non-zero bytes
    zero_bytes: uint256 = 0
    nonzero_bytes: uint256 = 0
    
    for i: uint256 in range(65536):
        if i >= len(tx_data):
            break
        if slice(tx_data, i, 1) == b"\x00":
            zero_bytes += 1
        else:
            nonzero_bytes += 1
    
    # EIP-4844 calldata costs: 4 gas/zero byte, 16 gas/nonzero byte
    calldata_gas: uint256 = zero_bytes * 4 + nonzero_bytes * 16
    
    # Get current L1 base fee and parameters
    l1_base_fee: uint256 = IL1Block(L1_BLOCK_INFO).basefee()
    overhead: uint256 = IGasPriceOracle(GAS_PRICE_ORACLE).overhead()
    scalar: uint256 = IGasPriceOracle(GAS_PRICE_ORACLE).scalar()
    
    # L1 fee = (calldata_gas + overhead) * l1_base_fee * scalar / 1000000
    return (calldata_gas + overhead) * l1_base_fee * scalar / 1000000
```

---

## 6. Multi-chain Deployment {#s6}

```python
# @version 0.4.0
# MultiChainToken.vy
# Token that can be deployed identically on multiple chains
# Uses chain-specific configurations

from vyper.interfaces import ERC20

# Chain-specific configuration
struct ChainConfig:
    name: String[30]
    is_l2: bool
    has_sequencer_feed: bool
    sequencer_feed: address

# Supported chains
ETHEREUM: constant(uint256) = 1
OPTIMISM: constant(uint256) = 10
ARBITRUM: constant(uint256) = 42161
BASE: constant(uint256) = 8453

# Token state
name: public(String[64])
symbol: public(String[32])
decimals: public(uint8)
total_supply: public(uint256)
balances: HashMap[address, uint256]
allowances: HashMap[address, HashMap[address, uint256]]

owner: public(address)

# Chain-specific data
deployment_chain: public(uint256)
deployment_block: public(uint256)
deployment_timestamp: public(uint256)

event Transfer:
    sender: indexed(address)
    receiver: indexed(address)
    value: uint256

event Approval:
    owner: indexed(address)
    spender: indexed(address)
    value: uint256

event CrossChainBurn:
    from_: indexed(address)
    amount: uint256
    destination_chain: uint256

event CrossChainMint:
    to: indexed(address)
    amount: uint256
    source_chain: uint256

@deploy
def __init__(
    token_name: String[64],
    token_symbol: String[32],
    initial_supply: uint256
):
    self.name = token_name
    self.symbol = token_symbol
    self.decimals = 18
    self.total_supply = initial_supply
    self.balances[msg.sender] = initial_supply
    self.owner = msg.sender
    
    # Record deployment information
    self.deployment_chain = chain.id
    self.deployment_block = block.number
    self.deployment_timestamp = block.timestamp
    
    log Transfer(empty(address), msg.sender, initial_supply)

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

@view
@external
def get_chain_info() -> (uint256, String[30]):
    """Get information about the chain this contract is on"""
    cid: uint256 = chain.id
    
    if cid == ETHEREUM:
        return cid, "Ethereum Mainnet"
    elif cid == OPTIMISM:
        return cid, "Optimism"
    elif cid == ARBITRUM:
        return cid, "Arbitrum One"
    elif cid == BASE:
        return cid, "Base"
    else:
        return cid, "Unknown Chain"
```

### Deterministic Deployment Script

```python
# deploy_multichain.py
# Script to deploy contract on multiple chains with same address
# Using CREATE2 for deterministic addresses

from web3 import Web3
import json

# Networks to deploy on
NETWORKS = {
    "ethereum": {
        "rpc": "https://mainnet.infura.io/v3/YOUR_KEY",
        "chain_id": 1,
        "factory": "0x..."  # Create2 Factory address
    },
    "optimism": {
        "rpc": "https://mainnet.optimism.io",
        "chain_id": 10,
        "factory": "0x..."
    },
    "arbitrum": {
        "rpc": "https://arb1.arbitrum.io/rpc",
        "chain_id": 42161,
        "factory": "0x..."
    }
}

# Salt for deterministic address
DEPLOYMENT_SALT = b"MultiChainToken_v1_2024"

def compute_create2_address(factory: str, salt: bytes, init_code_hash: bytes) -> str:
    """Compute the CREATE2 address"""
    from eth_abi.packed import encode_packed
    data = encode_packed(
        ["bytes1", "address", "bytes32", "bytes32"],
        [b"\xff", factory, salt, init_code_hash]
    )
    return "0x" + Web3.keccak(data)[12:].hex()

def deploy_on_network(network_config: dict, bytecode: bytes, constructor_args: bytes):
    """Deploy contract on a specific network"""
    w3 = Web3(Web3.HTTPProvider(network_config["rpc"]))
    
    # Use deployer key from environment
    deployer_key = "0x..."  # Load from env
    deployer = w3.eth.account.from_key(deployer_key)
    
    # Load factory ABI
    factory = w3.eth.contract(
        address=network_config["factory"],
        abi=[{
            "inputs": [
                {"name": "salt", "type": "bytes32"},
                {"name": "bytecode", "type": "bytes"}
            ],
            "name": "deploy",
            "outputs": [{"name": "addr", "type": "address"}],
            "type": "function"
        }]
    )
    
    # Deploy
    salt = Web3.keccak(DEPLOYMENT_SALT)
    init_code = bytecode + constructor_args
    
    tx = factory.functions.deploy(salt, init_code).build_transaction({
        "from": deployer.address,
        "gas": 3000000,
        "gasPrice": w3.eth.gas_price,
        "nonce": w3.eth.get_transaction_count(deployer.address)
    })
    
    signed = w3.eth.account.sign_transaction(tx, deployer_key)
    tx_hash = w3.eth.send_raw_transaction(signed.rawTransaction)
    receipt = w3.eth.wait_for_transaction_receipt(tx_hash)
    
    return receipt

if __name__ == "__main__":
    # Load contract bytecode
    with open("MultiChainToken.abi.json") as f:
        abi = json.load(f)
    
    with open("MultiChainToken.bin") as f:
        bytecode = bytes.fromhex(f.read())
    
    # Deploy on all networks
    for name, config in NETWORKS.items():
        print(f"Deploying on {name}...")
        try:
            receipt = deploy_on_network(config, bytecode, b"")
            print(f"Deployed at {receipt['contractAddress']}")
        except Exception as e:
            print(f"Failed: {e}")
```

---

## 7. Cross-Layer State Sync {#s7}

```python
# @version 0.4.0
# CrossLayerStateSync.vy
# Synchronize state between L1 and L2
# L2 version of the contract

interface IL2CrossDomainMessenger:
    def xDomainMessageSender() -> address: view
    def sendMessage(target: address, message: Bytes[65536], gas_limit: uint32): nonpayable

L2_MESSENGER: constant(address) = 0x4200000000000000000000000000000000000007
L1_MESSENGER: constant(address) = 0x25ace71c97B33Cc4729CF772ae268934F7ab5fA1

# State that mirrors L1
struct L1State:
    block_number: uint256
    state_root: bytes32
    timestamp: uint256
    verified: bool

owner: public(address)
l1_bridge: public(address)

# Cached L1 state
cached_l1_state: public(L1State)
last_sync_time: public(uint256)

# Sync window: only sync if data is fresh
SYNC_MAX_AGE: constant(uint256) = 3600  # 1 hour

event L1StateReceived:
    block_number: uint256
    state_root: bytes32
    timestamp: uint256

event StateSyncRequested:
    requested_by: indexed(address)

@deploy
def __init__(l1_bridge_address: address):
    self.owner = msg.sender
    self.l1_bridge = l1_bridge_address

@internal
def _only_from_l1():
    """Verify call came from our L1 bridge"""
    assert msg.sender == L2_MESSENGER, "Not L2 messenger"
    l1_sender: address = IL2CrossDomainMessenger(L2_MESSENGER).xDomainMessageSender()
    assert l1_sender == self.l1_bridge, "Not L1 bridge"

@external
def updateL1State(block_number: uint256, state_root: bytes32, timestamp: uint256):
    """
    Receive L1 state update
    Only callable via L1->L2 message from bridge
    """
    self._only_from_l1()
    
    # Update cached state
    self.cached_l1_state = L1State(
        block_number=block_number,
        state_root=state_root,
        timestamp=timestamp,
        verified=True
    )
    self.last_sync_time = block.timestamp
    
    log L1StateReceived(block_number, state_root, timestamp)

@view
@external
def get_l1_state() -> (uint256, bytes32, uint256, bool):
    """Get the latest synced L1 state"""
    state: L1State = self.cached_l1_state
    is_fresh: bool = block.timestamp - self.last_sync_time < SYNC_MAX_AGE
    
    return (
        state.block_number,
        state.state_root,
        state.timestamp,
        state.verified and is_fresh
    )

@view
@external
def is_state_fresh() -> bool:
    """Check if the L1 state is fresh enough to use"""
    return block.timestamp - self.last_sync_time < SYNC_MAX_AGE

@external
def request_sync():
    """Request L1 to send updated state (sends L2->L1 message)"""
    # Send message to L1 requesting sync
    # This would trigger the L1 bridge to send back current state
    request_data: Bytes[4] = b"\x12\x34\x56\x78"  # syncState() selector
    
    IL2CrossDomainMessenger(L2_MESSENGER).sendMessage(
        self.l1_bridge,
        request_data,
        200000
    )
    
    log StateSyncRequested(msg.sender)
```

---

## 8. Security Considerations {#s8}

```python
# @version 0.4.0
# L2SecurityPatterns.vy
# Security patterns specific to L2 deployments

# ============================================================
# 1. Block timestamp safety on L2
# ============================================================

# On L2s, block.timestamp is generally reliable
# but can differ from L1 in subtle ways
# Use block.timestamp for deadline checks, 
# but NOT for randomness

@view
@internal
def _check_deadline(deadline: uint256):
    """Safe deadline check on L2"""
    assert block.timestamp <= deadline, "Expired"

# ============================================================
# 2. L1 block.number vs L2 block.number
# ============================================================

# On Arbitrum: use ArbSys.arbBlockNumber() for L2 block
# On Optimism: block.number IS the L2 block number
# L1 block number available via L1Block predeploy

# DANGER: Don't use block.number for time calculations on L2
# L2 blocks are minted much faster than L1 blocks!
# Use block.timestamp instead

# ============================================================
# 3. Sequencer censorship resistance
# ============================================================

# L2 sequencers CAN censor transactions
# Mitigation: Force inclusion via L1 inbox after delay

# ============================================================
# 4. Cross-chain message authentication
# ============================================================

@internal
def _verify_cross_chain_sender(
    messenger: address,
    expected_l1_sender: address
):
    """
    Verify the original sender of a cross-chain message
    CRITICAL: Must check BOTH messenger AND original sender
    """
    # Step 1: Message MUST come from the L2 messenger
    assert msg.sender == messenger, "Not from messenger"
    
    # Step 2: The L1 sender MUST be our trusted contract
    # This prevents anyone else from sending messages
    l1_sender: address = IL2CrossDomainMessenger(messenger).xDomainMessageSender()
    assert l1_sender == expected_l1_sender, "Unauthorized L1 sender"

# ============================================================
# 5. Withdrawal attack prevention
# ============================================================

# On Optimism, withdrawals from L2 to L1 have a 7-day challenge period
# During this time, the withdrawal can be challenged via fraud proofs
# Design your protocol to handle this delay

struct PendingWithdrawal:
    amount: uint256
    l1_recipient: address
    l2_block: uint256
    claimable_after: uint256

pending_withdrawals: public(HashMap[bytes32, PendingWithdrawal])

WITHDRAWAL_DELAY: constant(uint256) = 86400 * 7  # 7 days on Optimism

@external
def initiate_withdrawal(amount: uint256, l1_recipient: address) -> bytes32:
    """Start the L2->L1 withdrawal process"""
    withdrawal_id: bytes32 = keccak256(
        concat(
            convert(msg.sender, bytes32),
            convert(amount, bytes32),
            convert(block.number, bytes32)
        )
    )
    
    self.pending_withdrawals[withdrawal_id] = PendingWithdrawal(
        amount=amount,
        l1_recipient=l1_recipient,
        l2_block=block.number,
        claimable_after=block.timestamp + WITHDRAWAL_DELAY
    )
    
    return withdrawal_id
```

### สรุปความแตกต่าง L2

| ประเด็น | Optimism | Arbitrum |
|---|---|---|
| Gas Oracle | OVM_GasPriceOracle | ArbGasInfo |
| L1 Block | IL1Block.number() | ArbSys.arbBlockNumber() |
| Fraud Proof | Single round | Multi-round (interactive) |
| Finality | 7 วัน | ~7 วัน |
| Precompiles | 0x4200... series | 0x0000...0064 series |

---

## สรุป

Layer 2 integration ต้องพิจารณา:
- **Chain-specific APIs**: Oracle และ precompile ต่างกัน
- **Sequencer health**: ตรวจสอบ uptime ก่อน operations สำคัญ
- **Cross-layer messaging**: ต้อง verify sender อย่างรัดกุม
- **Block timing**: L2 blocks เร็วกว่า L1 มาก
- **Withdrawal delays**: 7 วันสำหรับ optimistic rollups

---
[← Previous Part](part_068_account_abstraction.md) | [→ Next Part](part_070_cross_chain_messaging.md)
