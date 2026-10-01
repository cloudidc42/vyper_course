# Part 063: Token Bridge Patterns - Cross-Chain Communication

## สารบัญ (Table of Contents)
1. บทนำ Token Bridges
2. Lock-and-Mint Bridge
3. Burn-and-Mint Bridge
4. Validator-Based Bridge
5. Cross-Chain Messaging
6. Tests

---

## 1. บทนำ Token Bridges

**ประเภท Bridges:**
- **Lock-and-Mint**: Lock token บน chain A, Mint wrapped token บน chain B
- **Burn-and-Mint**: Burn บน chain A, Mint บน chain B (native token)
- **Liquidity Pool**: AMM-based cross-chain swap

**Security Models:**
- Validator-based: multisig validators ยืนยัน
- Optimistic: assume valid until challenged
- ZK-based: ZK proof สำหรับ state transitions

---

## 2. Lock-and-Mint Bridge (Source Chain)

```vyper
# @version 0.4.0
# contracts/LockBridge.vy
# Source chain: Lock tokens here to mint on destination

from vyper.interfaces import ERC20

# Events
event Locked:
    sender: indexed(address)
    token: indexed(address)
    amount: uint256
    destinationChain: uint256
    recipient: address
    nonce: uint256

event Unlocked:
    recipient: indexed(address)
    token: indexed(address)
    amount: uint256
    sourceChain: uint256
    nonce: uint256

event ValidatorAdded:
    validator: indexed(address)

event ValidatorRemoved:
    validator: indexed(address)

# State
validators: public(HashMap[address, bool])
validatorCount: public(uint256)
requiredSignatures: public(uint256)

# Locks
lockedBalances: public(HashMap[address, HashMap[address, uint256]])  # token -> user -> amount
totalLocked: public(HashMap[address, uint256])

# Nonces
outboundNonce: public(uint256)
processedNonces: public(HashMap[bytes32, bool])

# Supported tokens
supportedTokens: public(HashMap[address, bool])
governance: public(address)

# Fee
bridgeFee: public(uint256)  # basis points
feeRecipient: public(address)

CHAIN_ID: immutable(uint256)

@deploy
def __init__(
    _chainId: uint256,
    _governance: address,
    _feeRecipient: address,
    _bridgeFee: uint256,
    _requiredSigs: uint256
):
    CHAIN_ID = _chainId
    self.governance = _governance
    self.feeRecipient = _feeRecipient
    self.bridgeFee = _bridgeFee
    self.requiredSignatures = _requiredSigs

# ===== Admin =====

@external
def addValidator(validator: address):
    assert msg.sender == self.governance, "Not governance"
    assert not self.validators[validator], "Already validator"
    self.validators[validator] = True
    self.validatorCount += 1
    log ValidatorAdded(validator)

@external
def removeValidator(validator: address):
    assert msg.sender == self.governance, "Not governance"
    assert self.validators[validator], "Not validator"
    self.validators[validator] = False
    self.validatorCount -= 1
    log ValidatorRemoved(validator)

@external
def addSupportedToken(token: address):
    assert msg.sender == self.governance, "Not governance"
    self.supportedTokens[token] = True

# ===== Lock =====

@external
def lockTokens(
    token: address,
    amount: uint256,
    destinationChain: uint256,
    recipient: address
) -> uint256:
    """
    Lock tokens บน source chain เพื่อ bridge ไป destination
    
    Parameters:
        token: token address บน source chain
        amount: จำนวน token
        destinationChain: chain ID ปลายทาง
        recipient: ผู้รับบน destination chain
    
    Returns: nonce สำหรับ tracking
    """
    assert self.supportedTokens[token], "Unsupported token"
    assert amount > 0, "Zero amount"
    assert recipient != empty(address), "Zero recipient"
    
    # คำนวณ fee
    feeAmount: uint256 = amount * self.bridgeFee / 10000
    netAmount: uint256 = amount - feeAmount
    
    assert ERC20(token).transferFrom(msg.sender, self, amount), "Transfer failed"
    
    if feeAmount > 0:
        assert ERC20(token).transfer(self.feeRecipient, feeAmount), "Fee failed"
    
    self.lockedBalances[token][msg.sender] += netAmount
    self.totalLocked[token] += netAmount
    
    nonce: uint256 = self.outboundNonce
    self.outboundNonce += 1
    
    log Locked(msg.sender, token, netAmount, destinationChain, recipient, nonce)
    
    return nonce

# ===== Unlock (from validators) =====

@external
def unlockTokens(
    token: address,
    recipient: address,
    amount: uint256,
    sourceChain: uint256,
    nonce: uint256,
    signatures: DynArray[bytes, 65 * 10]  # packed signatures
) -> bool:
    """
    Unlock tokens หลังจากได้รับ proof จาก validators
    
    Validators ต้อง sign message ยืนยันว่า lock เกิดขึ้นบน source chain
    
    Parameters:
        signatures: validator signatures (each 65 bytes: r, s, v)
    """
    msgHash: bytes32 = keccak256(
        abi.encode(token, recipient, amount, sourceChain, CHAIN_ID, nonce)
    )
    
    nonceKey: bytes32 = keccak256(abi.encode(sourceChain, nonce))
    assert not self.processedNonces[nonceKey], "Already processed"
    assert self.totalLocked[token] >= amount, "Insufficient locked"
    
    # ตรวจ signatures
    validSigs: uint256 = self._countValidSignatures(msgHash, signatures)
    assert validSigs >= self.requiredSignatures, "Insufficient signatures"
    
    self.processedNonces[nonceKey] = True
    self.totalLocked[token] -= amount
    
    assert ERC20(token).transfer(recipient, amount), "Unlock failed"
    
    log Unlocked(recipient, token, amount, sourceChain, nonce)
    
    return True

@internal
@view
def _countValidSignatures(
    msgHash: bytes32,
    signatures: DynArray[bytes, 65 * 10]
) -> uint256:
    """นับ validator signatures ที่ valid"""
    count: uint256 = 0
    seen: DynArray[address, 20] = []
    
    # Each signature is 65 bytes (r=32, s=32, v=1)
    sigLen: uint256 = len(signatures)
    i: uint256 = 0
    
    for _: uint256 in range(10):
        if i + 65 > sigLen:
            break
        
        # Extract r, s, v from packed bytes
        # In practice, pass as separate arrays for clarity
        # Simplified: assume each call has one sig for now
        break
    
    return count

@external
@view
def getLockedBalance(token: address, user: address) -> uint256:
    return self.lockedBalances[token][user]
```

---

## 3. Mint Bridge (Destination Chain)

```vyper
# @version 0.4.0
# contracts/MintBridge.vy
# Destination chain: Mint wrapped tokens

from vyper.interfaces import ERC20

interface IWrappedToken:
    def mint(to: address, amount: uint256): nonpayable
    def burn(from_: address, amount: uint256): nonpayable
    def balanceOf(owner: address) -> uint256: view

# Events
event Minted:
    recipient: indexed(address)
    wrappedToken: indexed(address)
    amount: uint256
    sourceChain: uint256
    nonce: uint256

event BurnedForBridge:
    sender: indexed(address)
    wrappedToken: indexed(address)
    amount: uint256
    destinationChain: uint256
    recipient: address
    nonce: uint256

# State
validators: public(HashMap[address, bool])
requiredSignatures: public(uint256)
governance: public(address)

# Wrapped token mapping: original -> wrapped
wrappedTokens: public(HashMap[bytes32, address])  # hash(sourceChain, originalToken) -> wrappedToken

# Nonces
processedMints: public(HashMap[bytes32, bool])
burnNonce: public(uint256)

CHAIN_ID: immutable(uint256)

@deploy
def __init__(
    _chainId: uint256,
    _governance: address,
    _requiredSigs: uint256
):
    CHAIN_ID = _chainId
    self.governance = _governance
    self.requiredSignatures = _requiredSigs

@external
def addValidator(validator: address):
    assert msg.sender == self.governance
    self.validators[validator] = True

@external
def registerWrappedToken(
    sourceChain: uint256,
    originalToken: address,
    wrappedToken: address
):
    """Register wrapped token สำหรับ original token จาก source chain"""
    assert msg.sender == self.governance
    key: bytes32 = keccak256(abi.encode(sourceChain, originalToken))
    self.wrappedTokens[key] = wrappedToken

@external
def mintWrapped(
    sourceChain: uint256,
    originalToken: address,
    recipient: address,
    amount: uint256,
    nonce: uint256,
    v: DynArray[uint8, 20],
    r: DynArray[bytes32, 20],
    s: DynArray[bytes32, 20]
) -> bool:
    """
    Mint wrapped tokens หลัง validators ยืนยัน lock บน source chain
    
    ต้องมี >= requiredSignatures จาก validators
    """
    nonceKey: bytes32 = keccak256(abi.encode(sourceChain, nonce))
    assert not self.processedMints[nonceKey], "Already minted"
    
    wrappedKey: bytes32 = keccak256(abi.encode(sourceChain, originalToken))
    wrappedToken: address = self.wrappedTokens[wrappedKey]
    assert wrappedToken != empty(address), "No wrapped token"
    
    # Message hash
    msgHash: bytes32 = keccak256(
        abi.encode(sourceChain, CHAIN_ID, originalToken, recipient, amount, nonce)
    )
    
    # Verify signatures
    validCount: uint256 = 0
    seen: DynArray[address, 20] = []
    
    for i: uint256 in range(20):
        if i >= len(v):
            break
        
        signer: address = ecrecover(msgHash, convert(v[i], uint256), r[i], s[i])
        
        if self.validators[signer] and signer not in seen:
            validCount += 1
            seen.append(signer)
    
    assert validCount >= self.requiredSignatures, "Insufficient signatures"
    
    self.processedMints[nonceKey] = True
    
    IWrappedToken(wrappedToken).mint(recipient, amount)
    
    log Minted(recipient, wrappedToken, amount, sourceChain, nonce)
    
    return True

@external
def burnForBridge(
    wrappedToken: address,
    amount: uint256,
    destinationChain: uint256,
    recipient: address
) -> uint256:
    """
    Burn wrapped token เพื่อ bridge กลับไป source chain
    Validators จะเห็น event นี้และ unlock บน source chain
    """
    assert amount > 0
    
    IWrappedToken(wrappedToken).burn(msg.sender, amount)
    
    nonce: uint256 = self.burnNonce
    self.burnNonce += 1
    
    log BurnedForBridge(msg.sender, wrappedToken, amount, destinationChain, recipient, nonce)
    
    return nonce
```

---

## 4. Cross-Chain Messaging

```vyper
# @version 0.4.0
# contracts/CrossChainMessenger.vy
# ส่ง arbitrary messages ข้าม chain

# Events
event MessageSent:
    messageId: indexed(bytes32)
    sender: indexed(address)
    destinationChain: uint256
    recipient: address
    data: Bytes[1024]
    nonce: uint256

event MessageReceived:
    messageId: indexed(bytes32)
    sourceChain: uint256
    sender: address
    recipient: indexed(address)
    data: Bytes[1024]

event MessageExecuted:
    messageId: indexed(bytes32)
    success: bool

# Interfaces
interface IMessageReceiver:
    def receiveMessage(
        sourceChain: uint256,
        sourceSender: address,
        data: Bytes[1024]
    ) -> bool: nonpayable

# State
validators: public(HashMap[address, bool])
requiredSignatures: public(uint256)

outboundNonce: public(uint256)
executedMessages: public(HashMap[bytes32, bool])
pendingMessages: public(HashMap[bytes32, bool])

governance: public(address)

CHAIN_ID: immutable(uint256)

@deploy
def __init__(_chainId: uint256, _governance: address, _reqSigs: uint256):
    CHAIN_ID = _chainId
    self.governance = _governance
    self.requiredSignatures = _reqSigs

@external
def addValidator(v: address):
    assert msg.sender == self.governance
    self.validators[v] = True

@external
def sendMessage(
    destinationChain: uint256,
    recipient: address,
    data: Bytes[1024]
) -> bytes32:
    """
    ส่ง message ไปยัง chain อื่น
    
    Validators จะ relay message นี้ไปยัง destination
    
    Returns: messageId สำหรับ tracking
    """
    nonce: uint256 = self.outboundNonce
    self.outboundNonce += 1
    
    messageId: bytes32 = keccak256(
        abi.encode(CHAIN_ID, destinationChain, msg.sender, recipient, data, nonce)
    )
    
    log MessageSent(messageId, msg.sender, destinationChain, recipient, data, nonce)
    
    return messageId

@external
def receiveMessage(
    messageId: bytes32,
    sourceChain: uint256,
    sourceSender: address,
    recipient: address,
    data: Bytes[1024],
    nonce: uint256,
    v: DynArray[uint8, 20],
    r: DynArray[bytes32, 20],
    s: DynArray[bytes32, 20]
) -> bool:
    """
    รับและ execute message จาก source chain
    
    Requires validator signatures
    """
    assert not self.executedMessages[messageId], "Already executed"
    
    expectedId: bytes32 = keccak256(
        abi.encode(sourceChain, CHAIN_ID, sourceSender, recipient, data, nonce)
    )
    assert messageId == expectedId, "Invalid message ID"
    
    # Verify validator signatures
    validCount: uint256 = 0
    seen: DynArray[address, 20] = []
    
    for i: uint256 in range(20):
        if i >= len(v):
            break
        
        signer: address = ecrecover(messageId, convert(v[i], uint256), r[i], s[i])
        
        if self.validators[signer] and signer not in seen:
            validCount += 1
            seen.append(signer)
    
    assert validCount >= self.requiredSignatures, "Insufficient signatures"
    
    self.executedMessages[messageId] = True
    
    log MessageReceived(messageId, sourceChain, sourceSender, recipient, data)
    
    # Execute on recipient contract
    success: bool = False
    if recipient.is_contract:
        success = IMessageReceiver(recipient).receiveMessage(sourceChain, sourceSender, data)
    
    log MessageExecuted(messageId, success)
    
    return success

@external
@view
def isExecuted(messageId: bytes32) -> bool:
    return self.executedMessages[messageId]
```

---

## 5. Tests

```python
# tests/test_bridges.py
import pytest
from eth_abi import encode
from eth_account import Account
from brownie import LockBridge, MintBridge, WrappedToken, MockERC20, accounts, chain, web3

SCALE = 10**18
SOURCE_CHAIN = 1   # Ethereum
DEST_CHAIN = 137   # Polygon

@pytest.fixture
def setup():
    owner = accounts[0]
    alice = accounts[1]
    validator1 = accounts[2]
    validator2 = accounts[3]
    validator3 = accounts[4]
    
    # Deploy tokens
    usdc = MockERC20.deploy("USDC", "USDC", 6, {"from": owner})
    
    # Deploy bridge contracts
    lock_bridge = LockBridge.deploy(
        SOURCE_CHAIN, owner.address, owner.address, 10, 2,
        {"from": owner}
    )
    
    mint_bridge = MintBridge.deploy(
        DEST_CHAIN, owner.address, 2,
        {"from": owner}
    )
    
    # Deploy wrapped token on dest chain
    wrapped_usdc = WrappedToken.deploy(
        "Wrapped USDC", "wUSDC", mint_bridge.address,
        {"from": owner}
    )
    
    # Setup validators
    lock_bridge.addValidator(validator1.address, {"from": owner})
    lock_bridge.addValidator(validator2.address, {"from": owner})
    lock_bridge.addValidator(validator3.address, {"from": owner})
    
    mint_bridge.addValidator(validator1.address, {"from": owner})
    mint_bridge.addValidator(validator2.address, {"from": owner})
    mint_bridge.addValidator(validator3.address, {"from": owner})
    
    lock_bridge.addSupportedToken(usdc.address, {"from": owner})
    mint_bridge.registerWrappedToken(SOURCE_CHAIN, usdc.address, wrapped_usdc.address, {"from": owner})
    
    usdc.mint(alice, 10000 * 10**6, {"from": owner})
    
    return owner, alice, validator1, validator2, validator3, usdc, lock_bridge, mint_bridge, wrapped_usdc

def test_lock_tokens(setup):
    owner, alice, v1, v2, v3, usdc, lock_bridge, mint_bridge, wusdc = setup
    
    amount = 1000 * 10**6  # $1000 USDC
    
    usdc.approve(lock_bridge.address, amount, {"from": alice})
    
    tx = lock_bridge.lockTokens(
        usdc.address,
        amount,
        DEST_CHAIN,
        alice.address,
        {"from": alice}
    )
    
    nonce = tx.return_value
    locked_event = tx.events["Locked"][0]
    
    print(f"Locked nonce: {nonce}")
    print(f"Net amount locked: {locked_event['amount'] / 10**6:.2f} USDC")
    
    total = lock_bridge.totalLocked(usdc.address)
    print(f"Total locked: {total / 10**6:.2f} USDC")

def test_mint_with_signatures(setup):
    """
    Test minting on destination chain with validator signatures
    (Simulated since we're on single chain)
    """
    owner, alice, v1, v2, v3, usdc, lock_bridge, mint_bridge, wusdc = setup
    
    source_chain = SOURCE_CHAIN
    original_token = usdc.address
    recipient = alice.address
    amount = 990 * 10**6  # after fee
    nonce = 0
    
    dest_chain = DEST_CHAIN
    
    msg_hash = web3.keccak(
        encode(
            ["uint256", "uint256", "address", "address", "uint256", "uint256"],
            [source_chain, dest_chain, original_token, recipient, amount, nonce]
        )
    )
    
    # Sign with validators
    sig1 = Account.sign_message({"messageHash": msg_hash}, v1.private_key)
    sig2 = Account.sign_message({"messageHash": msg_hash}, v2.private_key)
    
    v_arr = [sig1.v, sig2.v]
    r_arr = [sig1.r.to_bytes(32, 'big'), sig2.r.to_bytes(32, 'big')]
    s_arr = [sig1.s.to_bytes(32, 'big'), sig2.s.to_bytes(32, 'big')]
    
    before = wusdc.balanceOf(alice.address)
    
    mint_bridge.mintWrapped(
        source_chain,
        original_token,
        recipient,
        amount,
        nonce,
        v_arr, r_arr, s_arr,
        {"from": owner}
    )
    
    after = wusdc.balanceOf(alice.address)
    print(f"Minted: {(after - before) / 10**6:.2f} wUSDC")
    assert after - before == amount

def test_burn_and_bridge_back(setup):
    """Test burning wrapped tokens to bridge back"""
    owner, alice, v1, v2, v3, usdc, lock_bridge, mint_bridge, wusdc = setup
    
    # First mint some wUSDC (simplified)
    # In real scenario, would go through full bridge flow
    wusdc.mint(alice.address, 500 * 10**6, {"from": owner})
    
    # Burn to bridge back
    wusdc.approve(mint_bridge.address, 500 * 10**6, {"from": alice})
    
    tx = mint_bridge.burnForBridge(
        wusdc.address,
        500 * 10**6,
        SOURCE_CHAIN,
        alice.address,
        {"from": alice}
    )
    
    burn_nonce = tx.return_value
    print(f"Burn nonce: {burn_nonce}")
    print(f"wUSDC balance after burn: {wusdc.balanceOf(alice.address) / 10**6:.2f}")
```

---

## 6. สรุป

### Bridge Security Considerations:

**1. Validator Security**
- Multi-sig threshold ป้องกัน single point of failure
- Key management สำคัญมาก (hardware security modules)
- ต้องมี decentralized validator set

**2. Race Conditions**
- Replay attacks: ใช้ nonce และ chain ID
- Double spending: processedNonces mapping
- Reorg protection: รอ N confirmations ก่อน relay

**3. Fee Structure**
- Bridge fees ชดเชย gas costs ของ validators
- Flash loan protection: fee ควรสูงกว่า profit จาก arbitrage

**4. Liquidity**
- Lock-and-Mint: ต้องมี token locked บน source
- Liquidity pool bridges: ต้องมี liquidity ทั้ง 2 chains
- Optimistic bridges: 7-day challenge period

**5. Bridge Hacks**
- Ronin ($625M): validator key compromise
- Wormhole ($320M): signature verification bug
- Nomad ($190M): improper initialization
- ทุก bug มักเกิดจาก signature/proof verification
