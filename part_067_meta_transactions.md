# Part 067: Meta-Transactions (Gasless Transactions) (Meta-Transactions และ Gasless Transactions)

## สารบัญ
1. [บทนำ Meta-Transactions](#s1)
2. [EIP-2771 Trusted Forwarder Pattern](#s2)
3. [MinimalForwarder Contract](#s3)
4. [Recipient Contract](#s4)
5. [Relayer Off-chain Logic (Python)](#s5)
6. [Nonce Management และ Replay Protection](#s6)
7. [การทดสอบด้วย pytest](#s7)
8. [Security Considerations](#s8)

---

## 1. บทนำ Meta-Transactions {#s1}

**Meta-transactions** คือรูปแบบที่ช่วยให้ผู้ใช้ทำ transaction โดยไม่ต้องมี ETH เพื่อจ่าย gas
แทนที่ผู้ใช้จ่าย gas เอง จะมี **relayer** จ่ายแทน และได้รับ ETH หรือ token เป็นค่าตอบแทน

### วิธีทำงาน

```
ผู้ใช้ -> sign message -> Relayer -> submit transaction -> Smart Contract
                                     (จ่าย gas)
```

1. ผู้ใช้สร้างและ sign message ที่ระบุ action ที่ต้องการ
2. ส่ง signed message ให้ relayer
3. Relayer สร้าง transaction จริงและ submit (จ่าย gas)
4. Smart contract ตรวจสอบ signature และ execute ตาม request ของผู้ใช้

### ประโยชน์หลัก

- **Onboarding ง่ายขึ้น**: ผู้ใช้ใหม่ไม่ต้องมี ETH ก่อน
- **Better UX**: ไม่ต้องจัดการ gas
- **Sponsorship**: Protocol/dApp จ่าย gas ให้ผู้ใช้
- **Batching**: Relayer สามารถรวม transactions หลายๆ อัน

---

## 2. EIP-2771 Trusted Forwarder Pattern {#s2}

EIP-2771 กำหนดมาตรฐานสำหรับ meta-transactions ที่ smart contracts สามารถ trust ได้

### Key Concepts

- **Forwarder**: Contract ที่รับ signed requests และส่งต่อไปยัง recipient
- **Trusted Forwarder**: Forwarder ที่ recipient contract ยอมรับ
- **_msgSender()**: Function ที่คืน original sender (ไม่ใช่ forwarder)

```python
# @version 0.4.0
# EIP2771Context.vy
# Context contract for EIP-2771 meta-transactions

# The trusted forwarder address
trusted_forwarder: public(immutable(address))

@deploy
def __init__(forwarder: address):
    trusted_forwarder = forwarder

@internal
def _msg_sender() -> address:
    """
    Get the actual sender of the transaction
    If called by trusted forwarder, extract original sender from calldata
    Otherwise return msg.sender directly
    """
    if msg.sender == trusted_forwarder:
        # Extract original sender from the last 20 bytes of calldata
        # EIP-2771: forwarder appends original sender to calldata
        return convert(
            slice(msg.data, len(msg.data) - 20, 20),
            address
        )
    return msg.sender

@internal
def _msg_data() -> Bytes[65536]:
    """
    Get the actual calldata
    If called by trusted forwarder, strip the appended sender address
    """
    if msg.sender == trusted_forwarder:
        # Remove last 20 bytes (the appended sender)
        return slice(msg.data, 0, len(msg.data) - 20)
    return msg.data

@view
@external
def is_trusted_forwarder(forwarder: address) -> bool:
    """Check if an address is the trusted forwarder"""
    return forwarder == trusted_forwarder
```

---

## 3. MinimalForwarder Contract {#s3}

MinimalForwarder รับ signed requests และส่งต่อไปยัง recipient contracts

```python
# @version 0.4.0
# MinimalForwarder.vy
# EIP-2771 compliant Minimal Forwarder
# Accepts signed meta-transaction requests and forwards them

# ============================================================
# EIP-712 Constants
# ============================================================

# Domain type hash
DOMAIN_TYPE_HASH: constant(bytes32) = 0x8b73c3c69bb8fe3d512ecc4cf759cc79239f7b179b0ffacaa9a75d522b39400f

# ForwardRequest type hash
# keccak256("ForwardRequest(address from,address to,uint256 value,uint256 gas,uint256 nonce,bytes data)")
REQUEST_TYPE_HASH: constant(bytes32) = keccak256(
    "ForwardRequest(address from,address to,uint256 value,uint256 gas,uint256 nonce,bytes data)"
)

# ============================================================
# Structs
# ============================================================

struct ForwardRequest:
    from_: address    # Original sender
    to: address       # Target contract
    value: uint256    # ETH value to send
    gas: uint256      # Gas limit for the call
    nonce: uint256    # Sender's current nonce
    data: Bytes[65536] # Encoded function call

# ============================================================
# Storage
# ============================================================

domain_separator: public(bytes32)
nonces: public(HashMap[address, uint256])

# ============================================================
# Events
# ============================================================

event MetaTxExecuted:
    from_: indexed(address)
    to: indexed(address)
    nonce: uint256
    success: bool

event BatchExecuted:
    executor: indexed(address)
    count: uint256

# ============================================================
# Constructor
# ============================================================

@deploy
def __init__():
    # Initialize EIP-712 domain separator
    self.domain_separator = keccak256(
        concat(
            DOMAIN_TYPE_HASH,
            keccak256(b"MinimalForwarder"),
            keccak256(b"0.0.1"),
            convert(chain.id, bytes32),
            convert(self, bytes32)
        )
    )

# ============================================================
# Internal Functions
# ============================================================

@internal
def _hash_request(req: ForwardRequest) -> bytes32:
    """
    Hash a ForwardRequest struct for EIP-712 signing
    """
    return keccak256(
        concat(
            REQUEST_TYPE_HASH,
            convert(req.from_, bytes32),
            convert(req.to, bytes32),
            convert(req.value, bytes32),
            convert(req.gas, bytes32),
            convert(req.nonce, bytes32),
            keccak256(req.data)
        )
    )

@internal
def _verify_signature(
    req: ForwardRequest,
    v: uint8,
    r: bytes32,
    s: bytes32
) -> bool:
    """
    Verify the signature on a ForwardRequest
    """
    # Compute struct hash
    struct_hash: bytes32 = self._hash_request(req)
    
    # Compute EIP-712 digest
    digest: bytes32 = keccak256(
        concat(
            b"\x19\x01",
            self.domain_separator,
            struct_hash
        )
    )
    
    # Recover signer
    signer: address = ecrecover(digest, v, r, s)
    
    # Verify signer matches request sender
    return signer == req.from_ and signer != empty(address)

# ============================================================
# External Functions
# ============================================================

@view
@external
def get_nonce(from_: address) -> uint256:
    """Get the current nonce for an address"""
    return self.nonces[from_]

@view
@external
def verify(
    req_from: address,
    req_to: address,
    req_value: uint256,
    req_gas: uint256,
    req_nonce: uint256,
    req_data: Bytes[65536],
    v: uint8,
    r: bytes32,
    s: bytes32
) -> bool:
    """
    Verify a meta-transaction request without executing it
    @return True if the signature is valid and nonce is correct
    """
    req: ForwardRequest = ForwardRequest(
        from_=req_from,
        to=req_to,
        value=req_value,
        gas=req_gas,
        nonce=req_nonce,
        data=req_data
    )
    
    # Check nonce
    if req.nonce != self.nonces[req.from_]:
        return False
    
    # Verify signature
    return self._verify_signature(req, v, r, s)

@external
@payable
def execute(
    req_from: address,
    req_to: address,
    req_value: uint256,
    req_gas: uint256,
    req_nonce: uint256,
    req_data: Bytes[65536],
    v: uint8,
    r: bytes32,
    s: bytes32
) -> (bool, Bytes[65536]):
    """
    Execute a meta-transaction
    @param req_from Original sender (signer)
    @param req_to Target contract address
    @param req_value ETH value to send
    @param req_gas Gas limit for the call
    @param req_nonce Expected nonce
    @param req_data Encoded function call data
    @param v,r,s Signature components
    @return (success, returndata)
    """
    req: ForwardRequest = ForwardRequest(
        from_=req_from,
        to=req_to,
        value=req_value,
        gas=req_gas,
        nonce=req_nonce,
        data=req_data
    )
    
    # Verify signature
    assert self._verify_signature(req, v, r, s), "Invalid signature"
    
    # Verify nonce
    assert req.nonce == self.nonces[req.from_], "Invalid nonce"
    
    # Increment nonce before execution (reentrancy protection)
    self.nonces[req.from_] += 1
    
    # Append original sender to calldata (EIP-2771)
    call_data: Bytes[65556] = concat(req.data, convert(req.from_, bytes20))
    
    # Execute the call
    success: bool = False
    ret: Bytes[65536] = b""
    
    # Call the target contract with appended sender
    # Note: In production you'd use raw_call with gas parameter
    success, ret = raw_call(
        req.to,
        call_data,
        max_outsize=65536,
        value=req.value,
        revert_on_failure=False
    )
    
    log MetaTxExecuted(req.from_, req.to, req.nonce - 1, success)
    
    return success, ret

@view
@external  
def get_request_hash(
    req_from: address,
    req_to: address,
    req_value: uint256,
    req_gas: uint256,
    req_nonce: uint256,
    req_data: Bytes[65536]
) -> bytes32:
    """
    Get the EIP-712 hash for a ForwardRequest (for off-chain signing)
    """
    req: ForwardRequest = ForwardRequest(
        from_=req_from,
        to=req_to,
        value=req_value,
        gas=req_gas,
        nonce=req_nonce,
        data=req_data
    )
    
    struct_hash: bytes32 = self._hash_request(req)
    
    return keccak256(
        concat(
            b"\x19\x01",
            self.domain_separator,
            struct_hash
        )
    )
```

---

## 4. Recipient Contract {#s4}

Contract ที่รองรับ meta-transactions ต้องใช้ `_msgSender()` แทน `msg.sender`

```python
# @version 0.4.0
# MetaTxRecipient.vy
# Example contract that supports EIP-2771 meta-transactions
# Uses _msgSender() to get the real user address

# ============================================================
# Storage
# ============================================================

# The trusted forwarder that can forward meta-transactions
trusted_forwarder: public(immutable(address))

# Owner of the contract
owner: public(address)

# Simple counter for demonstration
user_actions: public(HashMap[address, uint256])
total_actions: public(uint256)

# User balances (for token-like functionality)
balances: public(HashMap[address, uint256])

# ============================================================
# Events
# ============================================================

event ActionPerformed:
    user: indexed(address)
    action_id: uint256
    timestamp: uint256

event TokensMinted:
    to: indexed(address)
    amount: uint256
    via_meta_tx: bool

event OwnershipTransferred:
    previous_owner: indexed(address)
    new_owner: indexed(address)

# ============================================================
# Constructor
# ============================================================

@deploy
def __init__(forwarder: address):
    trusted_forwarder = forwarder
    self.owner = msg.sender

# ============================================================
# EIP-2771 Context Functions
# ============================================================

@internal
def _msg_sender() -> address:
    """
    Get the actual message sender
    EIP-2771: If called by trusted forwarder, original sender is
    appended to calldata as last 20 bytes
    """
    if msg.sender == trusted_forwarder:
        # Extract original sender from calldata
        # The forwarder appends it as bytes20 at the end
        return convert(
            slice(msg.data, len(msg.data) - 20, 20),
            address
        )
    return msg.sender

@view
@external
def is_trusted_forwarder(forwarder: address) -> bool:
    """Check if an address is the trusted forwarder"""
    return forwarder == trusted_forwarder

# ============================================================
# Business Logic (using _msgSender instead of msg.sender)
# ============================================================

@external
def perform_action():
    """
    Perform an action - works both directly and via meta-tx
    The user doesn't need ETH if using a relayer
    """
    # Get the real user (not the forwarder)
    user: address = self._msg_sender()
    
    # Increment user's action count
    self.user_actions[user] += 1
    self.total_actions += 1
    
    log ActionPerformed(user, self.user_actions[user], block.timestamp)

@external
def mint_tokens(amount: uint256):
    """
    Mint tokens to the caller
    Can be called via meta-transaction (gasless)
    """
    user: address = self._msg_sender()
    
    # Business logic check
    assert amount > 0, "Amount must be positive"
    assert amount <= 1000 * 10**18, "Max 1000 tokens"
    
    # Mint tokens
    self.balances[user] += amount
    
    # Check if this was a meta-transaction
    via_meta: bool = msg.sender == trusted_forwarder
    
    log TokensMinted(user, amount, via_meta)

@external
def transfer_tokens(to: address, amount: uint256):
    """
    Transfer tokens - works via meta-tx too
    """
    from_: address = self._msg_sender()
    
    assert to != empty(address), "Transfer to zero address"
    assert self.balances[from_] >= amount, "Insufficient balance"
    
    self.balances[from_] -= amount
    self.balances[to] += amount

@external
def transfer_ownership(new_owner: address):
    """Transfer contract ownership"""
    caller: address = self._msg_sender()
    assert caller == self.owner, "Not owner"
    assert new_owner != empty(address), "Invalid address"
    
    old_owner: address = self.owner
    self.owner = new_owner
    
    log OwnershipTransferred(old_owner, new_owner)

@view
@external
def get_user_stats(user: address) -> (uint256, uint256):
    """Get user's action count and token balance"""
    return self.user_actions[user], self.balances[user]
```

### ตัวอย่าง Batch Meta-Transactions

```python
# @version 0.4.0
# BatchMetaTxRecipient.vy
# Recipient that supports multiple actions in one meta-tx

trusted_forwarder: public(immutable(address))
owner: public(address)

# Action types
ACTION_MINT: constant(uint256) = 1
ACTION_TRANSFER: constant(uint256) = 2
ACTION_BURN: constant(uint256) = 3

balances: public(HashMap[address, uint256])

event BatchExecuted:
    user: indexed(address)
    num_actions: uint256

@deploy
def __init__(forwarder: address):
    trusted_forwarder = forwarder
    self.owner = msg.sender

@internal
def _msg_sender() -> address:
    if msg.sender == trusted_forwarder:
        return convert(
            slice(msg.data, len(msg.data) - 20, 20),
            address
        )
    return msg.sender

@external
def execute_batch(
    action_types: DynArray[uint256, 10],
    targets: DynArray[address, 10],
    amounts: DynArray[uint256, 10]
):
    """
    Execute multiple actions in a single meta-transaction
    Saves gas for relayers, better UX for users
    """
    user: address = self._msg_sender()
    num_actions: uint256 = len(action_types)
    
    assert num_actions == len(targets), "Array length mismatch"
    assert num_actions == len(amounts), "Array length mismatch"
    assert num_actions <= 10, "Max 10 actions"
    
    for i: uint256 in range(10):
        if i >= num_actions:
            break
            
        action: uint256 = action_types[i]
        target: address = targets[i]
        amount: uint256 = amounts[i]
        
        if action == ACTION_MINT:
            # Mint to user
            self.balances[user] += amount
        elif action == ACTION_TRANSFER:
            # Transfer to target
            assert self.balances[user] >= amount, "Insufficient balance"
            self.balances[user] -= amount
            self.balances[target] += amount
        elif action == ACTION_BURN:
            # Burn tokens
            assert self.balances[user] >= amount, "Insufficient balance"
            self.balances[user] -= amount
    
    log BatchExecuted(user, num_actions)
```

---

## 5. Relayer Off-chain Logic (Python) {#s5}

Relayer เป็น off-chain service ที่รับ signed requests และ submit ไป blockchain

```python
# relayer.py
# Off-chain relayer service for meta-transactions
# This runs as a Python service, NOT a Vyper contract

import json
import time
from web3 import Web3
from eth_account import Account
from eth_account.messages import encode_defunct
import asyncio

# ============================================================
# Relayer Configuration
# ============================================================

# Connect to node
w3 = Web3(Web3.HTTPProvider("http://localhost:8545"))

# Relayer account (pays gas)
RELAYER_PRIVATE_KEY = "0x..."  # Load from secure env
relayer_account = Account.from_key(RELAYER_PRIVATE_KEY)

# Contract addresses
FORWARDER_ADDRESS = "0x..."
RECIPIENT_ADDRESS = "0x..."

# Load ABIs
with open("MinimalForwarder.abi.json") as f:
    FORWARDER_ABI = json.load(f)
    
with open("MetaTxRecipient.abi.json") as f:
    RECIPIENT_ABI = json.load(f)

forwarder = w3.eth.contract(address=FORWARDER_ADDRESS, abi=FORWARDER_ABI)
recipient = w3.eth.contract(address=RECIPIENT_ADDRESS, abi=RECIPIENT_ABI)

# ============================================================
# EIP-712 Signing Helpers
# ============================================================

def get_domain_separator():
    """Get the domain separator from the forwarder contract"""
    return forwarder.functions.domain_separator().call()

def create_forward_request(
    from_address: str,
    to_address: str,
    value: int,
    gas: int,
    data: bytes
) -> dict:
    """Create a ForwardRequest dictionary"""
    nonce = forwarder.functions.get_nonce(from_address).call()
    
    return {
        "from_": from_address,
        "to": to_address,
        "value": value,
        "gas": gas,
        "nonce": nonce,
        "data": data
    }

def sign_forward_request(request: dict, private_key: str) -> dict:
    """
    Sign a ForwardRequest with EIP-712
    Returns the signature components (v, r, s)
    """
    # EIP-712 domain
    domain = {
        "name": "MinimalForwarder",
        "version": "0.0.1",
        "chainId": w3.eth.chain_id,
        "verifyingContract": FORWARDER_ADDRESS
    }
    
    # EIP-712 types
    types = {
        "ForwardRequest": [
            {"name": "from", "type": "address"},
            {"name": "to", "type": "address"},
            {"name": "value", "type": "uint256"},
            {"name": "gas", "type": "uint256"},
            {"name": "nonce", "type": "uint256"},
            {"name": "data", "type": "bytes"},
        ]
    }
    
    # Message data
    message = {
        "from": request["from_"],
        "to": request["to"],
        "value": request["value"],
        "gas": request["gas"],
        "nonce": request["nonce"],
        "data": request["data"]
    }
    
    # Sign with EIP-712
    full_message = {
        "domain": domain,
        "types": types,
        "message": message,
        "primaryType": "ForwardRequest"
    }
    
    account = Account.from_key(private_key)
    signed = account.sign_typed_data(full_message=full_message)
    
    return {
        "v": signed.v,
        "r": signed.r.to_bytes(32, 'big'),
        "s": signed.s.to_bytes(32, 'big')
    }

# ============================================================
# Request Queue and Processing
# ============================================================

class MetaTxQueue:
    """Queue for pending meta-transactions"""
    
    def __init__(self):
        self.pending = []
        self.processed = []
    
    def add_request(self, request: dict, signature: dict):
        """Add a meta-tx request to the queue"""
        self.pending.append({
            "request": request,
            "signature": signature,
            "timestamp": time.time()
        })
    
    def get_pending(self) -> list:
        """Get all pending requests"""
        return self.pending.copy()

class Relayer:
    """
    Off-chain relayer that processes meta-transactions
    """
    
    def __init__(self, queue: MetaTxQueue):
        self.queue = queue
        self.running = False
    
    def verify_request(self, request: dict, signature: dict) -> bool:
        """Verify a meta-tx request before submitting"""
        try:
            is_valid = forwarder.functions.verify(
                request["from_"],
                request["to"],
                request["value"],
                request["gas"],
                request["nonce"],
                request["data"],
                signature["v"],
                signature["r"],
                signature["s"]
            ).call()
            return is_valid
        except Exception as e:
            print(f"Verification failed: {e}")
            return False
    
    def estimate_gas(self, request: dict, signature: dict) -> int:
        """Estimate gas needed for execution"""
        try:
            gas = forwarder.functions.execute(
                request["from_"],
                request["to"],
                request["value"],
                request["gas"],
                request["nonce"],
                request["data"],
                signature["v"],
                signature["r"],
                signature["s"]
            ).estimate_gas({"from": relayer_account.address})
            return gas
        except Exception as e:
            print(f"Gas estimation failed: {e}")
            return 500000  # Default gas limit
    
    def submit_transaction(self, request: dict, signature: dict) -> str:
        """Submit the meta-transaction to blockchain"""
        # Build transaction
        tx = forwarder.functions.execute(
            request["from_"],
            request["to"],
            request["value"],
            request["gas"],
            request["nonce"],
            request["data"],
            signature["v"],
            signature["r"],
            signature["s"]
        ).build_transaction({
            "from": relayer_account.address,
            "gas": request["gas"] + 100000,  # Extra gas for forwarder overhead
            "gasPrice": w3.eth.gas_price,
            "nonce": w3.eth.get_transaction_count(relayer_account.address)
        })
        
        # Sign and send
        signed_tx = w3.eth.account.sign_transaction(tx, private_key=RELAYER_PRIVATE_KEY)
        tx_hash = w3.eth.send_raw_transaction(signed_tx.rawTransaction)
        
        return tx_hash.hex()
    
    async def process_queue(self):
        """Process pending meta-transaction requests"""
        self.running = True
        
        while self.running:
            pending = self.queue.get_pending()
            
            for item in pending:
                request = item["request"]
                signature = item["signature"]
                
                # Verify before submitting
                if not self.verify_request(request, signature):
                    print(f"Invalid request from {request['from_']}, skipping")
                    continue
                
                # Submit
                try:
                    tx_hash = self.submit_transaction(request, signature)
                    print(f"Submitted meta-tx: {tx_hash}")
                    
                    # Remove from pending
                    self.queue.pending.remove(item)
                    self.queue.processed.append({
                        **item,
                        "tx_hash": tx_hash,
                        "status": "submitted"
                    })
                    
                except Exception as e:
                    print(f"Failed to submit: {e}")
            
            await asyncio.sleep(1)  # Process every second

# ============================================================
# Client-side Helper (User Script)
# ============================================================

class MetaTxClient:
    """
    Client for users to create and submit meta-transaction requests
    """
    
    def __init__(self, user_private_key: str):
        self.user_account = Account.from_key(user_private_key)
        self.relayer_url = "http://localhost:3000"  # Relayer API endpoint
    
    def create_action_request(self) -> tuple:
        """
        Create a request to call perform_action() via meta-tx
        Returns (request, signature)
        """
        # Encode the function call
        call_data = recipient.encode_abi("perform_action", [])
        
        # Create request
        request = create_forward_request(
            from_address=self.user_account.address,
            to_address=RECIPIENT_ADDRESS,
            value=0,
            gas=200000,
            data=call_data
        )
        
        # Sign the request
        signature = sign_forward_request(request, self.user_account.key.hex())
        
        return request, signature
    
    def create_mint_request(self, amount: int) -> tuple:
        """Create a request to mint tokens via meta-tx"""
        call_data = recipient.encode_abi(
            "mint_tokens",
            [amount]
        )
        
        request = create_forward_request(
            from_address=self.user_account.address,
            to_address=RECIPIENT_ADDRESS,
            value=0,
            gas=200000,
            data=call_data
        )
        
        signature = sign_forward_request(request, self.user_account.key.hex())
        
        return request, signature

# ============================================================
# Example Usage
# ============================================================

def example_gasless_mint():
    """Example: User mints tokens without ETH"""
    # Create user account (no ETH needed)
    user = Account.create()
    client = MetaTxClient(user.key.hex())
    
    # Create mint request
    amount = 100 * 10**18  # 100 tokens
    request, signature = client.create_mint_request(amount)
    
    print(f"User {user.address} wants to mint {amount} tokens")
    print(f"Nonce: {request['nonce']}")
    
    # Send to relayer (HTTP request in real scenario)
    queue = MetaTxQueue()
    queue.add_request(request, signature)
    
    # Relayer processes it
    relayer = Relayer(queue)
    is_valid = relayer.verify_request(request, signature)
    
    if is_valid:
        tx_hash = relayer.submit_transaction(request, signature)
        print(f"Meta-tx submitted: {tx_hash}")
    else:
        print("Invalid request!")
    
    return tx_hash

if __name__ == "__main__":
    asyncio.run(example_gasless_mint())
```

---

## 6. Nonce Management และ Replay Protection {#s6}

```python
# @version 0.4.0
# NonceManager.vy
# Advanced nonce management for meta-transactions

# ============================================================
# Nonce Types
# ============================================================

# Sequential nonces (default) - simple but ordered
sequential_nonces: public(HashMap[address, uint256])

# Bitmap nonces - allow out-of-order execution
# Each bit represents a nonce (0=unused, 1=used)
nonce_bitmaps: public(HashMap[address, HashMap[uint256, uint256]])

# Channel-based nonces - different channels for parallel execution
channel_nonces: public(HashMap[address, HashMap[uint256, uint256]])

# ============================================================
# Events
# ============================================================

event NonceUsed:
    user: indexed(address)
    nonce_type: String[10]
    nonce: uint256

event BitmapNonceInvalidated:
    user: indexed(address)
    word_pos: uint256
    mask: uint256

# ============================================================
# Constructor
# ============================================================

@deploy
def __init__():
    pass

# ============================================================
# Sequential Nonce Functions
# ============================================================

@view
@external
def get_sequential_nonce(user: address) -> uint256:
    """Get the current sequential nonce for a user"""
    return self.sequential_nonces[user]

@internal
def _use_sequential_nonce(user: address, expected_nonce: uint256):
    """Use a sequential nonce"""
    assert self.sequential_nonces[user] == expected_nonce, "Invalid nonce"
    self.sequential_nonces[user] = expected_nonce + 1
    log NonceUsed(user, "sequential", expected_nonce)

# ============================================================
# Bitmap Nonce Functions (Unordered)
# ============================================================

@view
@external
def is_nonce_used(user: address, nonce: uint256) -> bool:
    """Check if a bitmap nonce has been used"""
    word_pos: uint256 = nonce / 256
    bit_pos: uint256 = nonce % 256
    word: uint256 = self.nonce_bitmaps[user][word_pos]
    return (word >> bit_pos) & 1 == 1

@internal
def _use_bitmap_nonce(user: address, nonce: uint256):
    """Use a bitmap nonce (allows out-of-order execution)"""
    word_pos: uint256 = nonce / 256
    bit_pos: uint256 = nonce % 256
    
    # Create bitmask for this nonce
    mask: uint256 = 1 << bit_pos
    
    # Check nonce is unused
    current_word: uint256 = self.nonce_bitmaps[user][word_pos]
    assert current_word & mask == 0, "Nonce already used"
    
    # Mark as used
    self.nonce_bitmaps[user][word_pos] = current_word | mask
    log NonceUsed(user, "bitmap", nonce)

@external
def invalidate_nonces(word_pos: uint256, mask: uint256):
    """
    Batch invalidate nonces using bitmap
    Useful for cancelling pending requests
    """
    user: address = msg.sender
    self.nonce_bitmaps[user][word_pos] = (
        self.nonce_bitmaps[user][word_pos] | mask
    )
    log BitmapNonceInvalidated(user, word_pos, mask)

# ============================================================
# Channel Nonce Functions (Parallel Execution)
# ============================================================

@view
@external
def get_channel_nonce(user: address, channel: uint256) -> uint256:
    """Get nonce for a specific channel"""
    return self.channel_nonces[user][channel]

@internal
def _use_channel_nonce(user: address, channel: uint256, expected_nonce: uint256):
    """
    Use a channel-specific nonce
    Different channels can execute in parallel
    """
    assert self.channel_nonces[user][channel] == expected_nonce, "Invalid nonce"
    self.channel_nonces[user][channel] = expected_nonce + 1
    log NonceUsed(user, "channel", expected_nonce)

# ============================================================
# Demo: Contract using all nonce types
# ============================================================

@external
def execute_sequential(
    from_: address,
    action_data: bytes32,
    nonce: uint256,
    v: uint8,
    r: bytes32,
    s: bytes32
):
    """Execute with sequential nonce"""
    # Verify signature
    msg_hash: bytes32 = keccak256(
        concat(action_data, convert(nonce, bytes32))
    )
    signer: address = ecrecover(msg_hash, v, r, s)
    assert signer == from_, "Invalid signature"
    
    # Use nonce (prevents replay)
    self._use_sequential_nonce(from_, nonce)
    
    # Execute action...

@external
def execute_unordered(
    from_: address,
    action_data: bytes32,
    nonce: uint256,
    v: uint8,
    r: bytes32,
    s: bytes32
):
    """Execute with bitmap nonce (unordered, non-sequential)"""
    # Verify signature
    msg_hash: bytes32 = keccak256(
        concat(action_data, convert(nonce, bytes32))
    )
    signer: address = ecrecover(msg_hash, v, r, s)
    assert signer == from_, "Invalid signature"
    
    # Use nonce (prevents replay, allows out-of-order)
    self._use_bitmap_nonce(from_, nonce)
    
    # Execute action...
```

---

## 7. การทดสอบด้วย pytest {#s7}

```python
# tests/test_meta_transactions.py
# Tests for meta-transaction contracts

import pytest
from eth_account import Account
from eth_account.messages import encode_defunct
from web3 import Web3
from ape import accounts, project, chain
import time

# ============================================================
# Fixtures
# ============================================================

@pytest.fixture
def deployer(accounts):
    return accounts[0]

@pytest.fixture
def relayer(accounts):
    """The relayer who pays gas"""
    return accounts[1]

@pytest.fixture
def user_account():
    """User who doesn't have ETH"""
    return Account.create()

@pytest.fixture
def forwarder(deployer, project):
    return deployer.deploy(project.MinimalForwarder)

@pytest.fixture
def recipient(deployer, project, forwarder):
    return deployer.deploy(project.MetaTxRecipient, forwarder.address)

# ============================================================
# Helper Functions
# ============================================================

def encode_action_call(contract, function_name: str, args: list = []) -> bytes:
    """Encode a function call for meta-tx data"""
    return contract.encode_input(function_name, args)

def create_and_sign_request(
    forwarder_contract,
    from_account,
    to_address: str,
    value: int,
    gas: int,
    data: bytes,
    chain_id: int
) -> tuple:
    """Create and sign a ForwardRequest"""
    nonce = forwarder_contract.get_nonce(from_account.address)
    
    domain = {
        "name": "MinimalForwarder",
        "version": "0.0.1",
        "chainId": chain_id,
        "verifyingContract": forwarder_contract.address
    }
    
    types = {
        "ForwardRequest": [
            {"name": "from", "type": "address"},
            {"name": "to", "type": "address"},
            {"name": "value", "type": "uint256"},
            {"name": "gas", "type": "uint256"},
            {"name": "nonce", "type": "uint256"},
            {"name": "data", "type": "bytes"},
        ]
    }
    
    message = {
        "from": from_account.address,
        "to": to_address,
        "value": value,
        "gas": gas,
        "nonce": nonce,
        "data": data
    }
    
    full_message = {
        "domain": domain,
        "types": types,
        "message": message,
        "primaryType": "ForwardRequest"
    }
    
    signed = from_account.sign_typed_data(full_message=full_message)
    
    return (
        from_account.address,
        to_address,
        value,
        gas,
        nonce,
        data,
        signed.v,
        signed.r.to_bytes(32, 'big'),
        signed.s.to_bytes(32, 'big')
    )

# ============================================================
# Tests
# ============================================================

def test_forwarder_deploys(forwarder):
    """Test forwarder deploys with correct domain separator"""
    domain_sep = forwarder.domain_separator()
    assert domain_sep != b'\x00' * 32

def test_initial_nonce_is_zero(forwarder, user_account):
    """Test initial nonce is 0"""
    nonce = forwarder.get_nonce(user_account.address)
    assert nonce == 0

def test_verify_valid_request(forwarder, recipient, user_account, chain):
    """Test that valid request passes verification"""
    data = b""  # Empty data for this test
    
    params = create_and_sign_request(
        forwarder,
        user_account,
        recipient.address,
        0,  # No ETH
        200000,
        data,
        chain.chain_id
    )
    
    is_valid = forwarder.verify(*params)
    assert is_valid == True

def test_meta_tx_perform_action(forwarder, recipient, user_account, relayer, chain, project):
    """Test user performs action via meta-transaction"""
    # Encode the perform_action call
    action_data = project.MetaTxRecipient.perform_action.encode_input()
    
    params = create_and_sign_request(
        forwarder,
        user_account,
        recipient.address,
        0,
        200000,
        action_data,
        chain.chain_id
    )
    
    # Initial action count
    initial_count = recipient.user_actions(user_account.address)
    
    # Relayer submits the meta-tx
    forwarder.execute(*params, sender=relayer)
    
    # Action count should increment for the user (not relayer)
    new_count = recipient.user_actions(user_account.address)
    assert new_count == initial_count + 1
    
    # Relayer's action count should be unchanged
    relayer_count = recipient.user_actions(relayer.address)
    assert relayer_count == initial_count

def test_nonce_increments_after_meta_tx(forwarder, recipient, user_account, relayer, chain, project):
    """Test nonce increments after successful meta-tx"""
    initial_nonce = forwarder.get_nonce(user_account.address)
    
    action_data = project.MetaTxRecipient.perform_action.encode_input()
    
    params = create_and_sign_request(
        forwarder,
        user_account,
        recipient.address,
        0,
        200000,
        action_data,
        chain.chain_id
    )
    
    forwarder.execute(*params, sender=relayer)
    
    new_nonce = forwarder.get_nonce(user_account.address)
    assert new_nonce == initial_nonce + 1

def test_replay_attack_fails(forwarder, recipient, user_account, relayer, chain, project):
    """Test that same signature cannot be used twice"""
    action_data = project.MetaTxRecipient.perform_action.encode_input()
    
    params = create_and_sign_request(
        forwarder,
        user_account,
        recipient.address,
        0,
        200000,
        action_data,
        chain.chain_id
    )
    
    # First execution should succeed
    forwarder.execute(*params, sender=relayer)
    
    # Second execution with same signature should fail
    with pytest.raises(Exception, match="Invalid nonce"):
        forwarder.execute(*params, sender=relayer)

def test_wrong_signature_fails(forwarder, user_account, relayer, accounts, chain, project):
    """Test that wrong signature is rejected"""
    recipient_contract = accounts[5]
    data = b"test"
    nonce = forwarder.get_nonce(user_account.address)
    
    # Sign with wrong account
    wrong_account = Account.create()
    
    # This should fail verification
    is_valid = forwarder.verify(
        user_account.address,  # Claiming to be user
        recipient_contract.address,
        0,
        200000,
        nonce,
        data,
        27,
        b'\x01' * 32,  # Invalid signature
        b'\x02' * 32
    )
    
    assert is_valid == False

def test_meta_tx_with_value(forwarder, recipient, user_account, relayer, chain, project):
    """Test meta-tx that sends ETH value"""
    action_data = b""
    
    # Send 0.01 ETH via meta-tx
    value = 10**16  # 0.01 ETH
    
    params = create_and_sign_request(
        forwarder,
        user_account,
        recipient.address,
        value,
        200000,
        action_data,
        chain.chain_id
    )
    
    # Relayer must provide the ETH value
    forwarder.execute(*params, sender=relayer, value=value)

def test_nonce_bitmap(project, accounts):
    """Test bitmap nonce functionality"""
    deployer = accounts[0]
    nonce_manager = deployer.deploy(project.NonceManager)
    
    user = Account.create()
    
    # Nonce should not be used initially
    assert nonce_manager.is_nonce_used(user.address, 0) == False
    assert nonce_manager.is_nonce_used(user.address, 255) == False
    
    # Invalidate some nonces
    # bit 0 and bit 3 -> mask = 0b1001 = 9
    nonce_manager.invalidate_nonces(0, 9, sender=accounts[1])
    
    # The user who called invalidate_nonces
    assert nonce_manager.is_nonce_used(accounts[1].address, 0) == True
    assert nonce_manager.is_nonce_used(accounts[1].address, 3) == True
    assert nonce_manager.is_nonce_used(accounts[1].address, 1) == False
```

---

## 8. Security Considerations {#s8}

### ความเสี่ยงและการป้องกัน

**1. Relayer Front-running**
```python
# @version 0.4.0
# AntiReorder.vy
# Protection against relayer reordering/front-running

# Include sequence information to prevent reordering
struct OrderedRequest:
    from_: address
    to: address
    data: Bytes[65536]
    nonce: uint256
    max_gas_price: uint256  # User sets max gas price they accept
    deadline: uint256

@internal
def _check_request_freshness(req: OrderedRequest):
    """Ensure request is still fresh"""
    # Deadline check
    assert block.timestamp <= req.deadline, "Request expired"
    
    # Gas price check (protect against relayer extracting value)
    assert tx.gasprice <= req.max_gas_price, "Gas price too high"
```

**2. Reentrancy via Meta-tx**
```python
# @version 0.4.0
# ReentrancyProtectedRecipient.vy
# Use reentrancy guard with meta-transactions

locked: bool

@internal
def _msg_sender() -> address:
    if msg.sender == trusted_forwarder:
        return convert(
            slice(msg.data, len(msg.data) - 20, 20),
            address
        )
    return msg.sender

@external
def sensitive_action():
    """Protected action that prevents reentrancy"""
    assert not self.locked, "Reentrancy detected"
    self.locked = True
    
    user: address = self._msg_sender()
    
    # Perform sensitive operation
    # ...
    
    self.locked = False
```

**3. Trusted Forwarder Verification**
```python
# @version 0.4.0
# ForwarderCheck.vy
# Always verify the trusted forwarder is legitimate

# Use immutable for security (cannot be changed after deploy)
trusted_forwarder: public(immutable(address))

@deploy
def __init__(forwarder: address):
    # Verify the forwarder implements IForwarder interface
    assert forwarder != empty(address), "Invalid forwarder"
    trusted_forwarder = forwarder

@external
def change_trusted_forwarder(new_forwarder: address):
    # NEVER allow changing trusted forwarder after deploy!
    # This would be a critical vulnerability
    # The immutable pattern prevents this
    raise "Cannot change trusted forwarder"
```

### ตารางสรุปความเสี่ยง

| ความเสี่ยง | ผลกระทบ | การป้องกัน |
|---|---|---|
| Replay Attack | ใช้ signature ซ้ำ | Nonce management |
| Relayer Front-running | Reorder transactions | Max gas price in sig |
| Wrong _msgSender | Incorrect auth | Careful EIP-2771 impl |
| Reentrancy | Drain funds | Reentrancy guard |
| Forwarder Swap | Trust wrong forwarder | Immutable forwarder |

---

## สรุป

Meta-transactions เป็นเทคนิคสำคัญสำหรับการ improve UX ของ DeFi:
- **EIP-2771**: มาตรฐาน trusted forwarder pattern
- **MinimalForwarder**: Contract ที่ forward meta-txs
- **_msgSender()**: วิธีหา original sender
- **Nonce Management**: ป้องกัน replay attacks
- **Relayer Service**: Off-chain service ที่ submit transactions

---
[← Previous Part](part_066_signatures.md) | [→ Next Part](part_068_account_abstraction.md)
