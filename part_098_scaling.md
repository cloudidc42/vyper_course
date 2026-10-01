# Part 098: Scaling DeFi Protocols

## สารบัญ
1. [Gas Optimization ระดับ Protocol](#gas-optimization)
2. [Layer 2 Deployment](#l2-deployment)
3. [Multi-chain Strategy](#multichain)
4. [Cross-chain Consistency](#cross-chain)
5. [State Channel Patterns](#state-channels)

---

## 1. Gas Optimization ระดับ Protocol {#gas-optimization}

### หลักการ Gas Optimization

Gas optimization ระดับ protocol แตกต่างจากระดับ contract เพราะมองภาพรวมทั้งระบบ:

1. **Batch Operations** - รวมหลาย operations เป็นหนึ่ง
2. **Storage Optimization** - ลดการ read/write storage
3. **Event-driven Architecture** - ใช้ events แทน storage ที่ไม่ต้องการ
4. **Lazy Evaluation** - คำนวณเฉพาะตอนที่จำเป็น

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0

"""
@title GasOptimizedVault - Vault ที่ optimize gas consumption
@notice แสดง patterns สำหรับ gas optimization ระดับ protocol
@dev ใช้ packed storage, batch operations และ lazy evaluation
"""

from vyper.interfaces import ERC20

# ============================================================
# Events
# ============================================================

event BatchDeposit:
    depositor: indexed(address)
    tokenCount: uint256
    totalGasSaved: uint256
    timestamp: uint256

event BatchWithdrawal:
    withdrawer: indexed(address)
    tokenCount: uint256
    timestamp: uint256

event StorageCompacted:
    oldSlots: uint256
    newSlots: uint256
    timestamp: uint256

# ============================================================
# Storage Packing
# ============================================================
# เก็บ multiple values ใน single storage slot เพื่อประหยัด gas

struct PackedUserData:
    # Pack หลาย values ไว้ใน single uint256
    # bits 0-63: balance (uint64 = max 18 ETH in wei, ปรับตาม use case)
    # bits 64-127: lastActionBlock (uint64)
    # bits 128-191: shares (uint64)
    # bits 192-255: flags + misc data
    data: uint256

struct TokenBatch:
    tokens: DynArray[address, 20]
    amounts: DynArray[uint256, 20]

# ============================================================
# Constants
# ============================================================

BALANCE_MASK: constant(uint256) = 0xFFFFFFFFFFFFFFFF  # 64 bits
BLOCK_SHIFT: constant(uint256) = 64
SHARES_SHIFT: constant(uint256) = 128
FLAGS_SHIFT: constant(uint256) = 192

MAX_BATCH_SIZE: constant(uint256) = 20

# ============================================================
# State Variables - Optimized Layout
# ============================================================

owner: public(address)

# Packed storage - ประหยัด storage slots
packedUserData: HashMap[address, HashMap[address, PackedUserData]]

# Token info packed อย่างมีประสิทธิภาพ
tokenTotalShares: HashMap[address, uint256]  # แยก เพราะอัปเดตบ่อย
tokenTotalDeposited: HashMap[address, uint256]
supportedTokens: DynArray[address, 50]
isSupported: HashMap[address, bool]

# Fee settings (packed)
# depositFee (uint16) + withdrawFee (uint16) + reserved (uint224) = 1 slot
tokenFeesPacked: HashMap[address, uint256]

# Admin
isPaused: public(bool)

# Gas tracking
totalGasSaved: public(uint256)  # Estimated gas saved from optimizations

# ============================================================
# Constructor
# ============================================================

@deploy
def __init__():
    self.owner = msg.sender

# ============================================================
# Packed Storage Helpers
# ============================================================

@internal
@pure
def _packUserData(
    _balance: uint256,
    _lastBlock: uint256,
    _shares: uint256,
    _flags: uint256
) -> uint256:
    """Pack user data into single uint256"""
    balance: uint256 = _balance & BALANCE_MASK
    lastBlock: uint256 = (_lastBlock & BALANCE_MASK) << BLOCK_SHIFT
    shares: uint256 = (_shares & BALANCE_MASK) << SHARES_SHIFT
    flags: uint256 = (_flags & BALANCE_MASK) << FLAGS_SHIFT
    return balance | lastBlock | shares | flags

@internal
@pure
def _unpackBalance(_data: uint256) -> uint256:
    return _data & BALANCE_MASK

@internal
@pure
def _unpackLastBlock(_data: uint256) -> uint256:
    return (_data >> BLOCK_SHIFT) & BALANCE_MASK

@internal
@pure
def _unpackShares(_data: uint256) -> uint256:
    return (_data >> SHARES_SHIFT) & BALANCE_MASK

# ============================================================
# Batch Operations - Core Gas Savings
# ============================================================

@external
def batchDeposit(
    _tokens: DynArray[address, MAX_BATCH_SIZE],
    _amounts: DynArray[uint256, MAX_BATCH_SIZE]
) -> DynArray[uint256, MAX_BATCH_SIZE]:
    """
    @notice ฝากหลาย tokens ในครั้งเดียว - ประหยัด gas มาก
    @dev แต่ละ deposit ปกติ ~100k gas, batch = overhead 1 transaction
    @return shares array of shares received per token
    """
    assert not self.isPaused, "Paused"
    assert len(_tokens) == len(_amounts), "Length mismatch"
    assert len(_tokens) <= MAX_BATCH_SIZE, "Too many tokens"
    
    shares: DynArray[uint256, MAX_BATCH_SIZE] = []
    
    # Gas saving: cache msg.sender ใน memory (ถูกกว่า repeat CALLER opcode)
    caller: address = msg.sender
    
    for i: uint256 in range(MAX_BATCH_SIZE):
        if i >= len(_tokens):
            break
        
        token: address = _tokens[i]
        amount: uint256 = _amounts[i]
        
        assert self.isSupported[token], "Token not supported"
        assert amount > 0, "Zero amount"
        
        # Calculate shares
        tokenShares: uint256 = self._calculateSharesOptimized(token, amount)
        
        # Update packed storage (1 SSTORE instead of multiple)
        currentData: PackedUserData = self.packedUserData[caller][token]
        currentBalance: uint256 = self._unpackBalance(currentData.data)
        currentShares: uint256 = self._unpackShares(currentData.data)
        
        newData: uint256 = self._packUserData(
            currentBalance + amount,
            block.number,
            currentShares + tokenShares,
            0
        )
        self.packedUserData[caller][token] = PackedUserData({data: newData})
        
        # Update token totals
        self.tokenTotalDeposited[token] += amount
        self.tokenTotalShares[token] += tokenShares
        
        # Transfer (external call)
        assert ERC20(token).transferFrom(caller, self, amount), "Transfer failed"
        
        shares.append(tokenShares)
    
    # Gas saved ≈ (n-1) * 21000 (base tx cost) + (n-1) * overhead
    gasSaved: uint256 = (len(_tokens) - 1) * 21000
    self.totalGasSaved += gasSaved
    
    log BatchDeposit(caller, len(_tokens), gasSaved, block.timestamp)
    
    return shares

@external
def batchWithdraw(
    _tokens: DynArray[address, MAX_BATCH_SIZE],
    _shares: DynArray[uint256, MAX_BATCH_SIZE]
) -> DynArray[uint256, MAX_BATCH_SIZE]:
    """
    @notice ถอนหลาย tokens ในครั้งเดียว
    """
    assert not self.isPaused, "Paused"
    assert len(_tokens) == len(_shares), "Length mismatch"
    
    amounts: DynArray[uint256, MAX_BATCH_SIZE] = []
    caller: address = msg.sender
    
    for i: uint256 in range(MAX_BATCH_SIZE):
        if i >= len(_tokens):
            break
        
        token: address = _tokens[i]
        sharesToWithdraw: uint256 = _shares[i]
        
        # Get current packed data
        currentData: PackedUserData = self.packedUserData[caller][token]
        currentShares: uint256 = self._unpackShares(currentData.data)
        currentBalance: uint256 = self._unpackBalance(currentData.data)
        
        assert currentShares >= sharesToWithdraw, "Insufficient shares"
        
        # Calculate amount
        amount: uint256 = self._calculateAmountOptimized(token, sharesToWithdraw)
        
        # Update packed storage
        newData: uint256 = self._packUserData(
            currentBalance - amount,
            block.number,
            currentShares - sharesToWithdraw,
            0
        )
        self.packedUserData[caller][token] = PackedUserData({data: newData})
        
        # Update totals
        self.tokenTotalShares[token] -= sharesToWithdraw
        self.tokenTotalDeposited[token] -= amount
        
        # Transfer
        assert ERC20(token).transfer(caller, amount), "Transfer failed"
        
        amounts.append(amount)
    
    log BatchWithdrawal(caller, len(_tokens), block.timestamp)
    
    return amounts

# ============================================================
# Optimized Calculations
# ============================================================

@internal
@view
def _calculateSharesOptimized(_token: address, _amount: uint256) -> uint256:
    """คำนวณ shares - optimized version"""
    totalShares: uint256 = self.tokenTotalShares[_token]
    totalDeposited: uint256 = self.tokenTotalDeposited[_token]
    
    if totalShares == 0:
        return _amount
    
    # ใช้ unchecked math ถ้าสามารถ prove ได้ว่าไม่ overflow
    return (_amount * totalShares) / totalDeposited

@internal
@view
def _calculateAmountOptimized(_token: address, _shares: uint256) -> uint256:
    """คำนวณ amount - optimized version"""
    totalShares: uint256 = self.tokenTotalShares[_token]
    totalDeposited: uint256 = self.tokenTotalDeposited[_token]
    
    if totalShares == 0:
        return 0
    
    return (_shares * totalDeposited) / totalShares

# ============================================================
# Storage Optimization - Compaction
# ============================================================

@external
def cleanupEmptyPositions(_users: DynArray[address, 50], _token: address):
    """
    @notice Clean up empty positions เพื่อ free up storage
    @dev คืน gas refund จากการ clearing storage slots
    """
    assert msg.sender == self.owner, "Not owner"
    
    cleared: uint256 = 0
    
    for user: address in _users:
        data: PackedUserData = self.packedUserData[user][_token]
        shares: uint256 = self._unpackShares(data.data)
        
        if shares == 0:
            # Clear storage - gets gas refund
            self.packedUserData[user][_token] = PackedUserData({data: 0})
            cleared += 1
    
    log StorageCompacted(len(_users), len(_users) - cleared, block.timestamp)

# ============================================================
# View Functions
# ============================================================

@external
@view
def getUserShares(_user: address, _token: address) -> uint256:
    data: PackedUserData = self.packedUserData[_user][_token]
    return self._unpackShares(data.data)

@external
@view
def getUserBalance(_user: address, _token: address) -> uint256:
    data: PackedUserData = self.packedUserData[_user][_token]
    return self._unpackBalance(data.data)

@external
def addToken(_token: address):
    assert msg.sender == self.owner, "Not owner"
    self.isSupported[_token] = True
    self.supportedTokens.append(_token)
```

---

## 2. Layer 2 Deployment {#l2-deployment}

### L2 Deployment Considerations

Layer 2 networks มีความแตกต่างจาก mainnet:
- **Optimistic Rollups** (Optimism, Arbitrum): ใช้ fraud proofs, 7 day withdrawal window
- **ZK Rollups** (zkSync, Starknet, Polygon zkEVM): ใช้ validity proofs, fast withdrawal

### L2-Optimized Contract

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0

"""
@title L2OptimizedProtocol - Protocol ที่ optimize สำหรับ L2
@notice L2 มี gas ถูกกว่า แต่ต้องระวัง L2-specific issues
@dev ออกแบบสำหรับ Arbitrum/Optimism deployment
"""

from vyper.interfaces import ERC20

# L2 Specific Interfaces
interface IL2Bridge:
    def sendMessage(_to: address, _data: Bytes[1024]): nonpayable
    def getL1GasPrice() -> uint256: view

interface IL2CrossDomainMessenger:
    def xDomainMessageSender() -> address: view

# Events
event L2OperationExecuted:
    opType: String[30]
    user: indexed(address)
    amount: uint256
    l2Block: uint256

event L1MessageReceived:
    sender: indexed(address)
    data: Bytes[256]
    timestamp: uint256

event SequencerHealthChecked:
    isHealthy: bool
    blockNumber: uint256

# State
owner: public(address)
l2Bridge: public(address)
l1Contract: public(address)  # Corresponding L1 contract
crossDomainMessenger: public(address)

# L2-specific tracking
l2BlockNumber: public(uint256)
lastSequencerCheck: public(uint256)
sequencerHealthy: public(bool)

# High-frequency operations (cheaper on L2)
userOperationCount: public(HashMap[address, uint256])
totalOperations: public(uint256)

# Token balances
balances: HashMap[address, HashMap[address, uint256]]
supportedTokens: DynArray[address, 50]

@deploy
def __init__(
    _bridge: address,
    _messenger: address,
    _l1Contract: address
):
    """
    @notice Initialize L2 contract
    @param _bridge L2 bridge contract
    @param _messenger Cross-domain messenger
    @param _l1Contract L1 counterpart address
    """
    self.owner = msg.sender
    self.l2Bridge = _bridge
    self.crossDomainMessenger = _messenger
    self.l1Contract = _l1Contract
    self.sequencerHealthy = True

@external
def checkSequencerHealth():
    """
    @notice ตรวจสอบว่า sequencer ทำงานปกติ
    @dev สำคัญมากสำหรับ Optimistic rollups
    """
    # Check if block number is incrementing (basic health check)
    isHealthy: bool = block.number > self.l2BlockNumber
    
    self.l2BlockNumber = block.number
    self.lastSequencerCheck = block.timestamp
    self.sequencerHealthy = isHealthy
    
    log SequencerHealthChecked(isHealthy, block.number)

@external
def deposit(_token: address, _amount: uint256):
    """
    @notice ฝาก token (บน L2 ถูก gas มาก)
    """
    assert _amount > 0, "Zero amount"
    assert ERC20(_token).transferFrom(msg.sender, self, _amount), "Transfer failed"
    
    self.balances[msg.sender][_token] += _amount
    self.userOperationCount[msg.sender] += 1
    self.totalOperations += 1
    
    log L2OperationExecuted("deposit", msg.sender, _amount, block.number)

@external
def withdraw(_token: address, _amount: uint256):
    """
    @notice ถอน token (ต้องระวัง withdrawal window บน Optimistic)
    """
    assert self.balances[msg.sender][_token] >= _amount, "Insufficient"
    
    self.balances[msg.sender][_token] -= _amount
    self.userOperationCount[msg.sender] += 1
    self.totalOperations += 1
    
    assert ERC20(_token).transfer(msg.sender, _amount), "Transfer failed"
    
    log L2OperationExecuted("withdraw", msg.sender, _amount, block.number)

@external
def receiveL1Message(_data: Bytes[256]):
    """
    @notice รับ message จาก L1
    @dev ต้อง verify ว่า message มาจาก cross-domain messenger
    """
    messenger: IL2CrossDomainMessenger = IL2CrossDomainMessenger(self.crossDomainMessenger)
    assert msg.sender == self.crossDomainMessenger, "Not messenger"
    assert messenger.xDomainMessageSender() == self.l1Contract, "Not L1 contract"
    
    log L1MessageReceived(self.l1Contract, _data, block.timestamp)

@external
@view
def getUserBalance(_user: address, _token: address) -> uint256:
    return self.balances[_user][_token]

@external
@view
def isSequencerHealthy() -> bool:
    return self.sequencerHealthy
```

---

## 3. Multi-chain Strategy {#multichain}

### Cross-chain Message Passing

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0

"""
@title CrossChainCoordinator - จัดการ cross-chain operations
@notice ประสาน state ระหว่าง chains ต่างๆ
@dev ใช้ LayerZero หรือ Axelar สำหรับ cross-chain messaging
"""

from vyper.interfaces import ERC20

# ============================================================
# Interfaces
# ============================================================

interface ILayerZeroEndpoint:
    def send(
        _dstChainId: uint16,
        _destination: Bytes[40],
        _payload: Bytes[1024],
        _refundAddress: address,
        _zroPaymentAddress: address,
        _adapterParams: Bytes[200]
    ): payable
    
    def estimateFees(
        _dstChainId: uint16,
        _userApplication: address,
        _payload: Bytes[1024],
        _payInZRO: bool,
        _adapterParams: Bytes[200]
    ) -> (uint256, uint256): view

interface IAxelarGateway:
    def callContract(
        _destinationChain: String[30],
        _contractAddress: String[42],
        _payload: Bytes[1024]
    ): nonpayable
    
    def sendToken(
        _destinationChain: String[30],
        _destinationAddress: String[42],
        _symbol: String[10],
        _amount: uint256
    ): nonpayable

# ============================================================
# Events
# ============================================================

event CrossChainMessageSent:
    dstChain: indexed(uint16)
    sender: indexed(address)
    messageType: String[30]
    payload: Bytes[256]
    timestamp: uint256

event CrossChainMessageReceived:
    srcChain: indexed(uint16)
    sender: bytes32
    messageType: String[30]
    timestamp: uint256

event ChainRegistered:
    chainId: indexed(uint16)
    chainName: String[30]
    contractAddress: bytes32

event BridgeTransferInitiated:
    user: indexed(address)
    srcChain: uint16
    dstChain: indexed(uint16)
    token: address
    amount: uint256
    nonce: uint256

# ============================================================
# Structs
# ============================================================

struct ChainConfig:
    chainId: uint16
    chainName: String[30]
    contractAddress: bytes32  # Destination contract as bytes32
    isActive: bool
    messageCount: uint256

struct PendingTransfer:
    nonce: uint256
    sender: address
    dstChain: uint16
    token: address
    amount: uint256
    initiatedAt: uint256
    completed: bool

# ============================================================
# Constants
# ============================================================

# Chain IDs (LayerZero IDs)
CHAIN_ETH_MAINNET: constant(uint16) = 1
CHAIN_ARBITRUM: constant(uint16) = 110
CHAIN_OPTIMISM: constant(uint16) = 111
CHAIN_POLYGON: constant(uint16) = 109
CHAIN_BSC: constant(uint16) = 102
CHAIN_AVALANCHE: constant(uint16) = 106

MAX_CHAINS: constant(uint256) = 20

# Message types
MSG_DEPOSIT: constant(uint8) = 1
MSG_WITHDRAWAL: constant(uint8) = 2
MSG_SYNC_STATE: constant(uint8) = 3
MSG_EMERGENCY: constant(uint8) = 4

# ============================================================
# State Variables
# ============================================================

owner: public(address)
lzEndpoint: public(address)
axelarGateway: public(address)

# Chain registry
registeredChains: public(DynArray[uint16, MAX_CHAINS])
chainConfigs: public(HashMap[uint16, ChainConfig])
currentChainId: public(uint16)

# Cross-chain state
pendingTransfers: public(HashMap[uint256, PendingTransfer])
transferNonce: public(uint256)
completedTransfers: public(HashMap[uint256, bool])

# Trusted remotes (LayerZero)
trustedRemotes: public(HashMap[uint16, bytes32])

# Total cross-chain stats
totalCrossChainMessages: public(uint256)
totalBridgedVolume: public(HashMap[address, uint256])

# ============================================================
# Constructor
# ============================================================

@deploy
def __init__(
    _currentChainId: uint16,
    _lzEndpoint: address,
    _axelarGateway: address
):
    """
    @notice Initialize cross-chain coordinator
    @param _currentChainId This chain's LayerZero chain ID
    @param _lzEndpoint LayerZero endpoint address
    @param _axelarGateway Axelar gateway address
    """
    self.owner = msg.sender
    self.currentChainId = _currentChainId
    self.lzEndpoint = _lzEndpoint
    self.axelarGateway = _axelarGateway

# ============================================================
# Chain Management
# ============================================================

@external
def registerChain(
    _chainId: uint16,
    _chainName: String[30],
    _contractAddress: bytes32
):
    """
    @notice ลงทะเบียน destination chain
    """
    assert msg.sender == self.owner, "Not owner"
    assert len(self.registeredChains) < MAX_CHAINS, "Too many chains"
    
    self.registeredChains.append(_chainId)
    self.chainConfigs[_chainId] = ChainConfig({
        chainId: _chainId,
        chainName: _chainName,
        contractAddress: _contractAddress,
        isActive: True,
        messageCount: 0
    })
    
    # Set trusted remote for LayerZero
    self.trustedRemotes[_chainId] = _contractAddress
    
    log ChainRegistered(_chainId, _chainName, _contractAddress)

# ============================================================
# Cross-chain Messaging
# ============================================================

@external
@payable
def sendCrossChainMessage(
    _dstChainId: uint16,
    _messageType: uint8,
    _payload: Bytes[512]
):
    """
    @notice ส่ง message ไปยัง chain อื่น ผ่าน LayerZero
    """
    assert self.chainConfigs[_dstChainId].isActive, "Chain not active"
    
    # Encode full payload
    fullPayload: Bytes[1024] = concat(
        convert(_messageType, bytes1),
        convert(msg.sender, bytes32),
        _payload
    )
    
    # Send via LayerZero
    endpoint: ILayerZeroEndpoint = ILayerZeroEndpoint(self.lzEndpoint)
    
    # Destination address (packed as bytes40: chainId + address)
    dstContract: bytes32 = self.chainConfigs[_dstChainId].contractAddress
    
    endpoint.send(
        _dstChainId,
        convert(dstContract, Bytes[40]),
        fullPayload,
        msg.sender,  # refund address
        empty(address),
        b""
    , value=msg.value)
    
    self.chainConfigs[_dstChainId].messageCount += 1
    self.totalCrossChainMessages += 1
    
    log CrossChainMessageSent(
        _dstChainId,
        msg.sender,
        "message",
        _payload[:256],
        block.timestamp
    )

@external
@payable
def initiateBridgeTransfer(
    _dstChainId: uint16,
    _token: address,
    _amount: uint256
):
    """
    @notice เริ่ม token bridge transfer ไปยัง chain อื่น
    """
    assert self.chainConfigs[_dstChainId].isActive, "Chain not active"
    assert _amount > 0, "Zero amount"
    
    # Lock tokens on source chain
    assert ERC20(_token).transferFrom(msg.sender, self, _amount), "Transfer failed"
    
    nonce: uint256 = self.transferNonce
    self.transferNonce += 1
    
    self.pendingTransfers[nonce] = PendingTransfer({
        nonce: nonce,
        sender: msg.sender,
        dstChain: _dstChainId,
        token: _token,
        amount: _amount,
        initiatedAt: block.timestamp,
        completed: False
    })
    
    # Encode bridge message
    payload: Bytes[512] = concat(
        convert(nonce, bytes32),
        convert(msg.sender, bytes32),
        convert(_token, bytes32),
        convert(_amount, bytes32)
    )
    
    # Send cross-chain message
    endpoint: ILayerZeroEndpoint = ILayerZeroEndpoint(self.lzEndpoint)
    dstContract: bytes32 = self.chainConfigs[_dstChainId].contractAddress
    
    endpoint.send(
        _dstChainId,
        convert(dstContract, Bytes[40]),
        payload,
        msg.sender,
        empty(address),
        b""
    , value=msg.value)
    
    self.totalBridgedVolume[_token] += _amount
    
    log BridgeTransferInitiated(msg.sender, self.currentChainId, _dstChainId, _token, _amount, nonce)

@external
def lzReceive(
    _srcChainId: uint16,
    _srcAddress: bytes32,
    _nonce: uint64,
    _payload: Bytes[1024]
):
    """
    @notice รับ message จาก LayerZero
    @dev ต้องมาจาก LZ endpoint เท่านั้น
    """
    assert msg.sender == self.lzEndpoint, "Not LZ endpoint"
    
    # Verify trusted remote
    assert self.trustedRemotes[_srcChainId] == _srcAddress, "Untrusted remote"
    
    # Parse payload
    # First byte = message type
    # Next 32 bytes = sender address
    messageType: uint8 = convert(slice(_payload, 0, 1), uint8)
    
    log CrossChainMessageReceived(_srcChainId, _srcAddress, "received", block.timestamp)
    
    # Process based on message type
    if messageType == MSG_DEPOSIT:
        self._processCrossChainDeposit(_srcChainId, _payload)
    elif messageType == MSG_WITHDRAWAL:
        self._processCrossChainWithdrawal(_srcChainId, _payload)

@internal
def _processCrossChainDeposit(_srcChainId: uint16, _payload: Bytes[1024]):
    """Process cross-chain deposit"""
    # Parse: nonce (32) + sender (32) + token (32) + amount (32)
    # Skip first byte (message type)
    nonce: uint256 = convert(slice(_payload, 1, 33), uint256)
    sender: address = convert(slice(_payload, 33, 65), address)
    token: address = convert(slice(_payload, 65, 97), address)
    amount: uint256 = convert(slice(_payload, 97, 129), uint256)
    
    # Mark as completed
    self.completedTransfers[nonce] = True

@internal
def _processCrossChainWithdrawal(_srcChainId: uint16, _payload: Bytes[1024]):
    """Process cross-chain withdrawal"""
    pass  # Implementation specific to use case

# ============================================================
# Fee Estimation
# ============================================================

@external
@view
def estimateCrossChainFee(
    _dstChainId: uint16,
    _payloadSize: uint256
) -> uint256:
    """
    @notice คำนวณค่าใช้จ่ายสำหรับ cross-chain message
    """
    endpoint: ILayerZeroEndpoint = ILayerZeroEndpoint(self.lzEndpoint)
    
    # Create dummy payload of specified size
    dummyPayload: Bytes[1024] = b"\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00"
    
    nativeFee: uint256 = 0
    zroFee: uint256 = 0
    nativeFee, zroFee = endpoint.estimateFees(
        _dstChainId,
        self,
        dummyPayload,
        False,
        b""
    )
    
    return nativeFee

# ============================================================
# View Functions
# ============================================================

@external
@view
def getRegisteredChains() -> DynArray[uint16, MAX_CHAINS]:
    """รายการ chains ที่ลงทะเบียน"""
    return self.registeredChains

@external
@view
def getPendingTransfer(_nonce: uint256) -> PendingTransfer:
    """ดู pending transfer"""
    return self.pendingTransfers[_nonce]

@external
@view
def getChainConfig(_chainId: uint16) -> ChainConfig:
    """ดู chain configuration"""
    return self.chainConfigs[_chainId]
```

---

## 4. Cross-chain Consistency {#cross-chain}

### State Synchronization Pattern

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0

"""
@title StateSync - Synchronize state across multiple chains
@notice ทำให้ state consistent ระหว่าง chains ต่างๆ
"""

# Events
event StateSynced:
    sourceChain: uint16
    stateRoot: bytes32
    blockNumber: uint256
    timestamp: uint256

event StateConflict:
    chain1: uint16
    chain2: uint16
    stateRoot1: bytes32
    stateRoot2: bytes32
    timestamp: uint256

# Structs
struct ChainState:
    chainId: uint16
    stateRoot: bytes32
    blockNumber: uint256
    timestamp: uint256
    isCanonical: bool

struct StateUpdate:
    fromChain: uint16
    stateRoot: bytes32
    blockNumber: uint256
    proofData: Bytes[256]

# State
owner: public(address)
canonicalChainId: public(uint16)

chainStates: public(HashMap[uint16, ChainState])
stateHistory: public(DynArray[ChainState, 1000])
trackedChains: public(DynArray[uint16, 20])

# Dispute resolution
pendingDisputes: public(uint256)
resolvedDisputes: public(uint256)

@deploy
def __init__(_canonicalChain: uint16):
    self.owner = msg.sender
    self.canonicalChainId = _canonicalChain

@external
def submitStateUpdate(_update: StateUpdate):
    """
    @notice ส่ง state update จาก chain อื่น
    """
    assert msg.sender == self.owner, "Not owner"  # In practice: verify cross-chain message
    
    # Check for conflicts
    existing: ChainState = self.chainStates[_update.fromChain]
    if existing.stateRoot != _update.stateRoot and existing.blockNumber >= _update.blockNumber:
        log StateConflict(
            _update.fromChain,
            self.canonicalChainId,
            _update.stateRoot,
            existing.stateRoot,
            block.timestamp
        )
        self.pendingDisputes += 1
        return
    
    # Update state
    newState: ChainState = ChainState({
        chainId: _update.fromChain,
        stateRoot: _update.stateRoot,
        blockNumber: _update.blockNumber,
        timestamp: block.timestamp,
        isCanonical: _update.fromChain == self.canonicalChainId
    })
    
    self.chainStates[_update.fromChain] = newState
    
    if len(self.stateHistory) < 1000:
        self.stateHistory.append(newState)
    
    log StateSynced(_update.fromChain, _update.stateRoot, _update.blockNumber, block.timestamp)

@external
@view
def areStatesSynced(_chain1: uint16, _chain2: uint16) -> bool:
    """ตรวจสอบว่า 2 chains sync กัน"""
    state1: ChainState = self.chainStates[_chain1]
    state2: ChainState = self.chainStates[_chain2]
    
    return state1.stateRoot == state2.stateRoot

@external
@view
def getChainState(_chainId: uint16) -> ChainState:
    return self.chainStates[_chainId]

@external
def addTrackedChain(_chainId: uint16):
    assert msg.sender == self.owner, "Not owner"
    self.trackedChains.append(_chainId)
```

---

## สรุป: Scaling Strategy

### Decision Framework

```
Scaling Decision Tree

Protocol TVL > $100M?
├── YES → Consider L2 deployment
│   ├── Use case: high-frequency?
│   │   ├── YES → Arbitrum/Optimism (EVM compatible)
│   │   └── NO  → Polygon (good balance)
│   └── Need fast finality?
│       ├── YES → ZK Rollup (zkSync, Starknet)
│       └── NO  → Optimistic Rollup
└── NO → Optimize gas on mainnet first

Multi-chain deployment?
├── Security critical? → Start single chain
├── User base fragmented? → Multi-chain
└── Competition pressure? → Follow users
```

### Gas Savings Summary

| Technique | Gas Saved | Complexity |
|-----------|-----------|------------|
| Batch transactions | 30-60% | Low |
| Packed storage | 20-40% | Medium |
| Lazy evaluation | 10-30% | Medium |
| L2 deployment | 90-99% | High |
| State channels | 99%+ | Very High |

> **คำแนะนำ**: เริ่มจาก L1 mainnet เพื่อ test product-market fit แล้วค่อย scale ไป L2 เมื่อ user base โตพอ การย้ายไป L2 เร็วเกินไปอาจทำให้ miss users ที่ยังอยู่บน mainnet
