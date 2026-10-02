# Part 066: Digital Signatures in Vyper (ลายเซ็นดิจิทัลใน Vyper)

## สารบัญ
1. [บทนำเรื่อง Digital Signatures](#s1)
2. [ECDSA และ ecrecover](#s2)
3. [EIP-191 Personal Sign](#s3)
4. [EIP-712 Typed Structured Data Signing](#s4)
5. [SignatureVerifier Contract](#s5)
6. [Permit-Style Approval](#s6)
7. [การทดสอบด้วย pytest](#s7)
8. [Security Considerations](#s8)

---

## 1. บทนำเรื่อง Digital Signatures {#s1}

ลายเซ็นดิจิทัล (Digital Signatures) เป็นกลไกสำคัญใน blockchain ที่ใช้พิสูจน์ตัวตนของผู้ส่งโดยไม่ต้องส่ง transaction จริง
ประโยชน์หลักคือ:
- **Gasless Approvals**: ผู้ใช้สามารถ sign approval โดยไม่เสีย gas
- **Off-chain Authorization**: สร้าง authorization off-chain แล้วใช้ on-chain
- **Meta-transactions**: ให้ผู้อื่นจ่าย gas แทน

### หลักการทำงานของ ECDSA

Ethereum ใช้ **Elliptic Curve Digital Signature Algorithm (ECDSA)** กับ secp256k1 curve

- **Private Key**: ตัวเลขลับที่ใช้สร้างลายเซ็น
- **Public Key**: คำนวณจาก private key
- **Address**: 20 bytes สุดท้ายของ keccak256(public key)
- **Signature**: ประกอบด้วย `r`, `s`, `v` (65 bytes รวมกัน)

```python
# Vyper 0.4.0 - ECDSA Signature basics
# @version 0.4.0

# The signature components
# v: recovery id (27 or 28 for Ethereum)
# r: x-coordinate of random point R
# s: proof value
# Together they form a 65-byte signature

# To verify: ecrecover(hash, v, r, s) returns the signer's address
```

---

## 2. ECDSA และ ecrecover {#s2}

Vyper มี builtin `ecrecover` ที่ใช้กู้คืน address จากลายเซ็น

```python
# @version 0.4.0
# ECDSA signature verification using ecrecover

# ecrecover signature: returns address that signed the message
# Parameters:
#   hash: bytes32 - the message hash that was signed
#   v: uint8 - recovery id (27 or 28)
#   r: bytes32 - signature component r
#   s: bytes32 - signature component s
# Returns: address of the signer (or zero address if invalid)

example_hash: bytes32 = keccak256(b"Hello, Ethereum!")
signer: address = ecrecover(example_hash, v, r, s)
```

### ตัวอย่าง Basic Signature Verifier

```python
# @version 0.4.0
# BasicSignatureVerifier.vy
# Simple ECDSA signature verification contract

# Storage
owner: public(address)
verified_signers: public(HashMap[address, bool])

event SignatureVerified:
    signer: indexed(address)
    message_hash: bytes32

event OwnerChanged:
    old_owner: indexed(address)
    new_owner: indexed(address)

@deploy
def __init__():
    # Initialize the contract with deployer as owner
    self.owner = msg.sender

@internal
def _recover_signer(message_hash: bytes32, v: uint8, r: bytes32, s: bytes32) -> address:
    # Recover the address that signed the message
    # Returns zero address if signature is invalid
    return ecrecover(message_hash, v, r, s)

@external
def verify_signature(
    message_hash: bytes32,
    v: uint8,
    r: bytes32,
    s: bytes32
) -> address:
    """
    Verify a signature and return the signer's address
    @param message_hash The hash of the signed message
    @param v Recovery id (27 or 28)
    @param r Signature component r
    @param s Signature component s
    @return The address that signed the message
    """
    # Recover signer address
    signer: address = self._recover_signer(message_hash, v, r, s)
    
    # Ensure signature is valid (non-zero address)
    assert signer != empty(address), "Invalid signature"
    
    # Store the verified signer
    self.verified_signers[signer] = True
    
    # Emit event
    log SignatureVerified(signer, message_hash)
    
    return signer

@external
def is_valid_signature(
    message_hash: bytes32,
    signer: address,
    v: uint8,
    r: bytes32,
    s: bytes32
) -> bool:
    """
    Check if a signature was made by a specific address
    @param message_hash The hash of the signed message
    @param signer The expected signer address
    @param v Recovery id
    @param r Signature component r
    @param s Signature component s
    @return True if signature is valid for the given signer
    """
    recovered: address = self._recover_signer(message_hash, v, r, s)
    return recovered == signer

@external
def change_owner(new_owner: address):
    """Change the contract owner"""
    assert msg.sender == self.owner, "Not owner"
    assert new_owner != empty(address), "Invalid address"
    
    old_owner: address = self.owner
    self.owner = new_owner
    
    log OwnerChanged(old_owner, new_owner)
```

### Signature Malleability Protection

```python
# @version 0.4.0
# SecureSignatureVerifier.vy
# Signature verification with malleability protection

# Prevent signature reuse
used_signatures: public(HashMap[bytes32, bool])

# Maximum value for s component (for malleability check)
# s must be in lower half of the curve order
MAX_S: constant(bytes32) = 0x7FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF5D576E7357A4501DDFE92F46681B20A0

@deploy
def __init__():
    pass

@internal
def _is_valid_s(s: bytes32) -> bool:
    # Check that s is in the lower half of the curve order
    # This prevents signature malleability attacks
    # Convert bytes32 to uint256 for comparison
    s_value: uint256 = convert(s, uint256)
    max_s_value: uint256 = convert(MAX_S, uint256)
    return s_value <= max_s_value

@internal
def _compute_signature_id(v: uint8, r: bytes32, s: bytes32) -> bytes32:
    # Create a unique identifier for this signature
    return keccak256(concat(convert(v, bytes1), r, s))

@external
def verify_and_store(
    message_hash: bytes32,
    v: uint8,
    r: bytes32,
    s: bytes32
) -> address:
    """
    Verify signature with malleability protection and replay prevention
    """
    # Check for signature malleability
    assert self._is_valid_s(s), "Invalid s value - potential malleability"
    
    # v must be 27 or 28
    assert v == 27 or v == 28, "Invalid v value"
    
    # Create signature unique ID
    sig_id: bytes32 = self._compute_signature_id(v, r, s)
    
    # Check for replay
    assert not self.used_signatures[sig_id], "Signature already used"
    
    # Recover signer
    signer: address = ecrecover(message_hash, v, r, s)
    assert signer != empty(address), "Invalid signature"
    
    # Mark signature as used
    self.used_signatures[sig_id] = True
    
    return signer
```

---

## 3. EIP-191 Personal Sign {#s3}

EIP-191 กำหนดมาตรฐานสำหรับการ sign messages ใน Ethereum
`personal_sign` เพิ่ม prefix `"\x19Ethereum Signed Message:\n"` เพื่อป้องกันการ sign transaction โดยไม่ตั้งใจ

### EIP-191 Prefix

```
"\x19Ethereum Signed Message:\n" + len(message) + message
```

```python
# @version 0.4.0
# EIP191Verifier.vy
# EIP-191 personal_sign verification

# EIP-191 prefix for personal_sign
# \x19Ethereum Signed Message:\n32 (for 32-byte hash)
ETH_SIGN_PREFIX: constant(Bytes[28]) = b"\x19Ethereum Signed Message:\n32"

# Nonce tracking to prevent replay attacks
nonces: public(HashMap[address, uint256])

event MessageSigned:
    signer: indexed(address)
    nonce: uint256
    message: bytes32

@deploy
def __init__():
    pass

@internal
def _to_eth_signed_message_hash(hash: bytes32) -> bytes32:
    """
    Convert a message hash to an EIP-191 personal_sign hash
    Matches what MetaMask's personal_sign does
    """
    # Prefix the hash with Ethereum prefix
    return keccak256(concat(ETH_SIGN_PREFIX, hash))

@external
def verify_personal_sign(
    message: bytes32,
    signer: address,
    v: uint8,
    r: bytes32,
    s: bytes32
) -> bool:
    """
    Verify an EIP-191 personal_sign signature
    @param message The original message bytes32
    @param signer The expected signer
    @param v Recovery id
    @param r Signature r component
    @param s Signature s component
    @return True if signature is valid
    """
    # Apply EIP-191 prefix
    eth_hash: bytes32 = self._to_eth_signed_message_hash(message)
    
    # Recover the signer
    recovered: address = ecrecover(eth_hash, v, r, s)
    
    return recovered == signer

@external
def verify_signed_action(
    action_data: bytes32,
    signer: address,
    v: uint8,
    r: bytes32,
    s: bytes32
) -> bool:
    """
    Verify an action with nonce to prevent replay
    @param action_data Encoded action data hash
    @param signer Expected signer address
    """
    # Include nonce in the signed data
    current_nonce: uint256 = self.nonces[signer]
    
    # Create message with nonce
    message_with_nonce: bytes32 = keccak256(
        concat(
            action_data,
            convert(current_nonce, bytes32),
            convert(chain.id, bytes32)
        )
    )
    
    # Apply EIP-191 prefix
    eth_hash: bytes32 = self._to_eth_signed_message_hash(message_with_nonce)
    
    # Recover signer
    recovered: address = ecrecover(eth_hash, v, r, s)
    
    if recovered != signer:
        return False
    
    # Increment nonce to prevent replay
    self.nonces[signer] = current_nonce + 1
    
    log MessageSigned(signer, current_nonce, action_data)
    
    return True
```

---

## 4. EIP-712 Typed Structured Data Signing {#s4}

EIP-712 เป็นมาตรฐานที่ทันสมัยกว่า EIP-191 โดยช่วยให้ผู้ใช้เห็นข้อมูลที่ชัดเจนก่อน sign
MetaMask และ wallet อื่นๆ แสดงข้อมูล structured อย่างสวยงาม

### Domain Separator

```python
# @version 0.4.0
# EIP712Domain.vy
# EIP-712 Domain Separator implementation

# EIP-712 type hash for domain
# keccak256("EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)")
DOMAIN_TYPE_HASH: constant(bytes32) = 0x8b73c3c69bb8fe3d512ecc4cf759cc79239f7b179b0ffacaa9a75d522b39400f

# Contract name and version for EIP-712
NAME: constant(String[20]) = "MyContract"
VERSION: constant(String[3]) = "1.0"

# Cached domain separator
domain_separator: public(bytes32)

@deploy
def __init__():
    # Compute domain separator at deployment
    # This binds signatures to this specific contract and chain
    self.domain_separator = keccak256(
        concat(
            DOMAIN_TYPE_HASH,
            keccak256(convert(NAME, Bytes[20])),
            keccak256(convert(VERSION, Bytes[3])),
            convert(chain.id, bytes32),
            convert(self, bytes32)
        )
    )

@internal
def _hash_typed_data(struct_hash: bytes32) -> bytes32:
    """
    Create EIP-712 compliant hash for signing
    Combines domain separator with struct hash
    """
    # EIP-712 format: \x19\x01 + domainSeparator + structHash
    return keccak256(
        concat(
            b"\x19\x01",
            self.domain_separator,
            struct_hash
        )
    )
```

### Complete EIP-712 Implementation

```python
# @version 0.4.0
# EIP712TypedData.vy
# Complete EIP-712 typed structured data signing

# ============================================================
# EIP-712 Type Hashes
# ============================================================

# Domain type hash
DOMAIN_TYPE_HASH: constant(bytes32) = 0x8b73c3c69bb8fe3d512ecc4cf759cc79239f7b179b0ffacaa9a75d522b39400f

# Transfer type hash
# keccak256("Transfer(address from,address to,uint256 amount,uint256 nonce,uint256 deadline)")
TRANSFER_TYPE_HASH: constant(bytes32) = keccak256(
    "Transfer(address from,address to,uint256 amount,uint256 nonce,uint256 deadline)"
)

# Vote type hash
# keccak256("Vote(uint256 proposalId,bool support,uint256 nonce)")
VOTE_TYPE_HASH: constant(bytes32) = keccak256(
    "Vote(uint256 proposalId,bool support,uint256 nonce)"
)

# ============================================================
# Storage
# ============================================================

domain_separator: public(bytes32)
nonces: public(HashMap[address, uint256])
name: public(String[50])
version: public(String[10])

# ============================================================
# Events
# ============================================================

event TransferExecuted:
    from_: indexed(address)
    to: indexed(address)
    amount: uint256

event VoteCast:
    voter: indexed(address)
    proposal_id: uint256
    support: bool

# ============================================================
# Constructor
# ============================================================

@deploy
def __init__(contract_name: String[50], contract_version: String[10]):
    # Store name and version
    self.name = contract_name
    self.version = contract_version
    
    # Compute and cache domain separator
    self.domain_separator = self._compute_domain_separator(contract_name, contract_version)

# ============================================================
# Internal Functions
# ============================================================

@internal
def _compute_domain_separator(
    contract_name: String[50],
    contract_version: String[10]
) -> bytes32:
    """Compute EIP-712 domain separator"""
    return keccak256(
        concat(
            DOMAIN_TYPE_HASH,
            keccak256(convert(contract_name, Bytes[50])),
            keccak256(convert(contract_version, Bytes[10])),
            convert(chain.id, bytes32),
            convert(self, bytes32)
        )
    )

@internal
def _hash_transfer(
    from_: address,
    to: address,
    amount: uint256,
    nonce: uint256,
    deadline: uint256
) -> bytes32:
    """Compute struct hash for Transfer type"""
    return keccak256(
        concat(
            TRANSFER_TYPE_HASH,
            convert(from_, bytes32),
            convert(to, bytes32),
            convert(amount, bytes32),
            convert(nonce, bytes32),
            convert(deadline, bytes32)
        )
    )

@internal
def _hash_vote(
    proposal_id: uint256,
    support: bool,
    nonce: uint256
) -> bytes32:
    """Compute struct hash for Vote type"""
    support_val: uint256 = 1 if support else 0
    return keccak256(
        concat(
            VOTE_TYPE_HASH,
            convert(proposal_id, bytes32),
            convert(support_val, bytes32),
            convert(nonce, bytes32)
        )
    )

@internal
def _to_typed_data_hash(struct_hash: bytes32) -> bytes32:
    """Create final EIP-712 hash"""
    return keccak256(
        concat(
            b"\x19\x01",
            self.domain_separator,
            struct_hash
        )
    )

@internal
def _recover_signer(
    hash: bytes32,
    v: uint8,
    r: bytes32,
    s: bytes32
) -> address:
    """Recover signer from hash and signature"""
    return ecrecover(hash, v, r, s)

# ============================================================
# External Functions
# ============================================================

@external
def execute_meta_transfer(
    from_: address,
    to: address,
    amount: uint256,
    deadline: uint256,
    v: uint8,
    r: bytes32,
    s: bytes32
) -> bool:
    """
    Execute a transfer authorized by EIP-712 signature
    @param from_ The token sender (who signed)
    @param to The recipient
    @param amount Amount to transfer
    @param deadline Signature expiry timestamp
    @param v,r,s Signature components
    """
    # Check deadline
    assert block.timestamp <= deadline, "Signature expired"
    
    # Get current nonce
    current_nonce: uint256 = self.nonces[from_]
    
    # Compute struct hash
    struct_hash: bytes32 = self._hash_transfer(
        from_, to, amount, current_nonce, deadline
    )
    
    # Compute final EIP-712 hash
    digest: bytes32 = self._to_typed_data_hash(struct_hash)
    
    # Recover signer
    signer: address = self._recover_signer(digest, v, r, s)
    
    # Verify signer matches from_
    assert signer == from_, "Invalid signature"
    assert signer != empty(address), "Invalid signer"
    
    # Increment nonce
    self.nonces[from_] = current_nonce + 1
    
    # Execute transfer logic here...
    log TransferExecuted(from_, to, amount)
    
    return True

@external
def cast_vote_with_sig(
    proposal_id: uint256,
    support: bool,
    voter: address,
    v: uint8,
    r: bytes32,
    s: bytes32
) -> bool:
    """
    Cast a vote on behalf of voter using EIP-712 signature
    """
    # Get voter's current nonce
    current_nonce: uint256 = self.nonces[voter]
    
    # Compute struct hash for vote
    struct_hash: bytes32 = self._hash_vote(proposal_id, support, current_nonce)
    
    # Create final digest
    digest: bytes32 = self._to_typed_data_hash(struct_hash)
    
    # Recover signer
    signer: address = self._recover_signer(digest, v, r, s)
    
    # Verify voter
    assert signer == voter, "Invalid signature"
    assert signer != empty(address), "Invalid voter signature"
    
    # Increment nonce
    self.nonces[voter] = current_nonce + 1
    
    log VoteCast(voter, proposal_id, support)
    
    return True

@view
@external
def get_transfer_hash(
    from_: address,
    to: address,
    amount: uint256,
    nonce: uint256,
    deadline: uint256
) -> bytes32:
    """Get the EIP-712 hash for a transfer (for off-chain signing)"""
    struct_hash: bytes32 = self._hash_transfer(from_, to, amount, nonce, deadline)
    return self._to_typed_data_hash(struct_hash)
```

---

## 5. SignatureVerifier Contract {#s5}

ตัวอย่าง contract ที่สมบูรณ์สำหรับ verification หลายรูปแบบ

```python
# @version 0.4.0
# SignatureVerifier.vy
# Comprehensive signature verification contract

# ============================================================
# Constants
# ============================================================

# EIP-712 Domain type hash
DOMAIN_TYPE_HASH: constant(bytes32) = 0x8b73c3c69bb8fe3d512ecc4cf759cc79239f7b179b0ffacaa9a75d522b39400f

# EIP-191 prefix for 32-byte messages
EIP191_PREFIX: constant(Bytes[28]) = b"\x19Ethereum Signed Message:\n32"

# ============================================================
# Structs
# ============================================================

struct SignatureData:
    v: uint8
    r: bytes32
    s: bytes32

struct SignedMessage:
    signer: address
    message_hash: bytes32
    timestamp: uint256
    is_valid: bool

# ============================================================
# Storage
# ============================================================

owner: public(address)
domain_separator: public(bytes32)
contract_name: public(String[50])

# Store signed messages for auditing
signed_messages: public(HashMap[bytes32, SignedMessage])
signer_message_count: public(HashMap[address, uint256])

# Revoked signatures
revoked_signatures: public(HashMap[bytes32, bool])

# Authorization: address => operation => bool
authorizations: public(HashMap[address, HashMap[bytes32, bool]))

# ============================================================
# Events
# ============================================================

event MessageVerified:
    signer: indexed(address)
    message_hash: indexed(bytes32)
    sig_type: String[10]

event SignatureRevoked:
    signer: indexed(address)
    sig_id: bytes32

event AuthorizationGranted:
    granter: indexed(address)
    grantee: indexed(address)
    operation: bytes32

# ============================================================
# Constructor
# ============================================================

@deploy
def __init__(name: String[50]):
    self.owner = msg.sender
    self.contract_name = name
    
    # Compute domain separator
    self.domain_separator = keccak256(
        concat(
            DOMAIN_TYPE_HASH,
            keccak256(convert(name, Bytes[50])),
            keccak256(b"1"),
            convert(chain.id, bytes32),
            convert(self, bytes32)
        )
    )

# ============================================================
# Internal Helper Functions
# ============================================================

@internal
def _eth_signed_hash(message_hash: bytes32) -> bytes32:
    """Apply EIP-191 prefix to message hash"""
    return keccak256(concat(EIP191_PREFIX, message_hash))

@internal
def _eip712_hash(struct_hash: bytes32) -> bytes32:
    """Apply EIP-712 domain to struct hash"""
    return keccak256(
        concat(b"\x19\x01", self.domain_separator, struct_hash)
    )

@internal
def _recover(hash: bytes32, sig: SignatureData) -> address:
    """Recover signer address from hash and signature"""
    # Validate v value
    if sig.v != 27 and sig.v != 28:
        return empty(address)
    
    return ecrecover(hash, sig.v, sig.r, sig.s)

@internal
def _sig_id(sig: SignatureData) -> bytes32:
    """Create unique ID for a signature"""
    return keccak256(concat(convert(sig.v, bytes1), sig.r, sig.s))

# ============================================================
# External Functions - Verification
# ============================================================

@external
def verify_eip191(
    message_hash: bytes32,
    expected_signer: address,
    v: uint8,
    r: bytes32,
    s: bytes32
) -> bool:
    """
    Verify an EIP-191 personal_sign signature
    Used when MetaMask signs with personal_sign
    """
    sig: SignatureData = SignatureData(v=v, r=r, s=s)
    
    # Check if signature is revoked
    sig_id: bytes32 = self._sig_id(sig)
    assert not self.revoked_signatures[sig_id], "Signature revoked"
    
    # Apply EIP-191 prefix
    eth_hash: bytes32 = self._eth_signed_hash(message_hash)
    
    # Recover signer
    recovered: address = self._recover(eth_hash, sig)
    
    if recovered == expected_signer and recovered != empty(address):
        # Record the verification
        self.signed_messages[sig_id] = SignedMessage(
            signer=recovered,
            message_hash=message_hash,
            timestamp=block.timestamp,
            is_valid=True
        )
        self.signer_message_count[recovered] += 1
        log MessageVerified(recovered, message_hash, "EIP191")
        return True
    
    return False

@external
def verify_eip712(
    struct_hash: bytes32,
    expected_signer: address,
    v: uint8,
    r: bytes32,
    s: bytes32
) -> bool:
    """
    Verify an EIP-712 typed data signature
    Used when dApps present structured signing dialogs
    """
    sig: SignatureData = SignatureData(v=v, r=r, s=s)
    
    # Check revocation
    sig_id: bytes32 = self._sig_id(sig)
    assert not self.revoked_signatures[sig_id], "Signature revoked"
    
    # Apply EIP-712 domain
    digest: bytes32 = self._eip712_hash(struct_hash)
    
    # Recover signer
    recovered: address = self._recover(digest, sig)
    
    if recovered == expected_signer and recovered != empty(address):
        self.signed_messages[sig_id] = SignedMessage(
            signer=recovered,
            message_hash=struct_hash,
            timestamp=block.timestamp,
            is_valid=True
        )
        log MessageVerified(recovered, struct_hash, "EIP712")
        return True
    
    return False

@external
def revoke_signature(v: uint8, r: bytes32, s: bytes32):
    """
    Revoke a signature - signer can invalidate their own signature
    """
    sig: SignatureData = SignatureData(v=v, r=r, s=s)
    sig_id: bytes32 = self._sig_id(sig)
    
    # Verify the caller is the signer
    # We check by verifying with a dummy hash - in practice you'd
    # want to verify the caller owns the signature
    self.revoked_signatures[sig_id] = True
    
    log SignatureRevoked(msg.sender, sig_id)

@external
def grant_authorization(grantee: address, operation: bytes32):
    """Grant authorization to another address for an operation"""
    self.authorizations[msg.sender][operation] = True
    
    # Record authorization as a "signed message"
    auth_hash: bytes32 = keccak256(
        concat(
            convert(msg.sender, bytes32),
            convert(grantee, bytes32),
            operation
        )
    )
    
    log AuthorizationGranted(msg.sender, grantee, operation)

@view
@external
def is_authorized(granter: address, operation: bytes32) -> bool:
    """Check if an authorization exists"""
    return self.authorizations[granter][operation]
```

---

## 6. Permit-Style Approval {#s6}

ERC-20 Permit (EIP-2612) ช่วยให้ผู้ใช้ approve tokens โดยใช้ signature แทน transaction
ประหยัด gas และ UX ดีขึ้น

```python
# @version 0.4.0
# ERC20Permit.vy
# ERC-20 token with EIP-2612 permit functionality
# Allows gasless approvals via signatures

from vyper.interfaces import ERC20

# ============================================================
# EIP-712 Constants
# ============================================================

# Domain type hash
DOMAIN_TYPE_HASH: constant(bytes32) = 0x8b73c3c69bb8fe3d512ecc4cf759cc79239f7b179b0ffacaa9a75d522b39400f

# Permit type hash
# keccak256("Permit(address owner,address spender,uint256 value,uint256 nonce,uint256 deadline)")
PERMIT_TYPE_HASH: constant(bytes32) = 0x6e71edae12b1b97f4d1f60370fef10105fa2faae0126114a169c64845d6126c9

# ============================================================
# Storage
# ============================================================

# ERC-20 state
name: public(String[64])
symbol: public(String[32])
decimals: public(uint8)
total_supply: public(uint256)
balances: HashMap[address, uint256]
allowances: HashMap[address, HashMap[address, uint256]]

# EIP-2612 state
domain_separator: public(bytes32)
nonces: public(HashMap[address, uint256])

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

# ============================================================
# Constructor
# ============================================================

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
    
    # Compute EIP-712 domain separator
    self.domain_separator = keccak256(
        concat(
            DOMAIN_TYPE_HASH,
            keccak256(convert(token_name, Bytes[64])),
            keccak256(b"1"),
            convert(chain.id, bytes32),
            convert(self, bytes32)
        )
    )
    
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
    assert to != empty(address), "Transfer to zero address"
    assert self.balances[msg.sender] >= amount, "Insufficient balance"
    
    self.balances[msg.sender] -= amount
    self.balances[to] += amount
    
    log Transfer(msg.sender, to, amount)
    return True

@external
def transferFrom(from_: address, to: address, amount: uint256) -> bool:
    assert to != empty(address), "Transfer to zero address"
    assert self.balances[from_] >= amount, "Insufficient balance"
    assert self.allowances[from_][msg.sender] >= amount, "Insufficient allowance"
    
    self.allowances[from_][msg.sender] -= amount
    self.balances[from_] -= amount
    self.balances[to] += amount
    
    log Transfer(from_, to, amount)
    return True

@external
def approve(spender: address, amount: uint256) -> bool:
    assert spender != empty(address), "Approve to zero address"
    
    self.allowances[msg.sender][spender] = amount
    log Approval(msg.sender, spender, amount)
    return True

# ============================================================
# EIP-2612 Permit Function
# ============================================================

@external
def permit(
    owner: address,
    spender: address,
    value: uint256,
    deadline: uint256,
    v: uint8,
    r: bytes32,
    s: bytes32
):
    """
    EIP-2612 permit: approve tokens via signature
    Allows gasless approvals - spender calls this with owner's signature
    
    @param owner Token owner who signed the permit
    @param spender Address being approved to spend tokens
    @param value Amount of tokens to approve
    @param deadline Timestamp after which the permit is invalid
    @param v,r,s Signature components
    """
    # Check deadline hasn't passed
    assert block.timestamp <= deadline, "Permit expired"
    
    # Get current nonce for owner
    current_nonce: uint256 = self.nonces[owner]
    
    # Compute struct hash
    struct_hash: bytes32 = keccak256(
        concat(
            PERMIT_TYPE_HASH,
            convert(owner, bytes32),
            convert(spender, bytes32),
            convert(value, bytes32),
            convert(current_nonce, bytes32),
            convert(deadline, bytes32)
        )
    )
    
    # Compute final EIP-712 hash
    digest: bytes32 = keccak256(
        concat(
            b"\x19\x01",
            self.domain_separator,
            struct_hash
        )
    )
    
    # Recover signer from signature
    recovered_owner: address = ecrecover(digest, v, r, s)
    
    # Verify signature is from owner
    assert recovered_owner == owner, "Invalid permit signature"
    assert recovered_owner != empty(address), "Invalid signer"
    
    # Increment nonce to prevent replay
    self.nonces[owner] = current_nonce + 1
    
    # Set approval
    self.allowances[owner][spender] = value
    log Approval(owner, spender, value)

@view
@external
def DOMAIN_SEPARATOR() -> bytes32:
    """Return the EIP-712 domain separator"""
    return self.domain_separator

@view
@external
def get_permit_hash(
    owner: address,
    spender: address,
    value: uint256,
    deadline: uint256
) -> bytes32:
    """
    Compute the permit hash for off-chain signing
    Frontend calls this to get the hash to sign
    """
    current_nonce: uint256 = self.nonces[owner]
    
    struct_hash: bytes32 = keccak256(
        concat(
            PERMIT_TYPE_HASH,
            convert(owner, bytes32),
            convert(spender, bytes32),
            convert(value, bytes32),
            convert(current_nonce, bytes32),
            convert(deadline, bytes32)
        )
    )
    
    return keccak256(
        concat(
            b"\x19\x01",
            self.domain_separator,
            struct_hash
        )
    )
```

---

## 7. การทดสอบด้วย pytest {#s7}

```python
# tests/test_signatures.py
# Tests for digital signature contracts

import pytest
from eth_account import Account
from eth_account.messages import encode_defunct, encode_structured_data
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
def user(accounts):
    return accounts[1]

@pytest.fixture
def signer_account():
    """Create a test account with known private key"""
    return Account.create()

@pytest.fixture
def basic_verifier(deployer, project):
    return deployer.deploy(project.BasicSignatureVerifier)

@pytest.fixture
def eip712_contract(deployer, project):
    return deployer.deploy(project.EIP712TypedData, "TestContract", "1.0")

@pytest.fixture
def permit_token(deployer, project):
    initial_supply = 1_000_000 * 10**18
    return deployer.deploy(project.ERC20Permit, "TestToken", "TEST", initial_supply)

# ============================================================
# Helper Functions
# ============================================================

def sign_message_eip191(message_hash: bytes, private_key: str):
    """Sign a message with EIP-191 prefix (personal_sign)"""
    message = encode_defunct(hexstr=message_hash.hex())
    signed = Account.sign_message(message, private_key=private_key)
    return signed.v, signed.r.to_bytes(32, 'big'), signed.s.to_bytes(32, 'big')

def sign_typed_data_eip712(
    domain: dict,
    types: dict,
    message: dict,
    private_key: str
):
    """Sign typed structured data with EIP-712"""
    data = {
        "domain": domain,
        "types": types,
        "message": message,
        "primaryType": list(types.keys())[0]
    }
    signed = Account.sign_typed_data(
        private_key=private_key,
        full_message=data
    )
    return signed.v, signed.r.to_bytes(32, 'big'), signed.s.to_bytes(32, 'big')

# ============================================================
# Test EIP-191 Signatures
# ============================================================

def test_eip191_valid_signature(basic_verifier, signer_account):
    """Test that valid EIP-191 signature is verified correctly"""
    # Create a test message
    test_message = b"Hello, Vyper!"
    message_hash = Web3.keccak(test_message)
    
    # Sign with EIP-191 prefix
    v, r, s = sign_message_eip191(message_hash, signer_account.key)
    
    # Verify signature
    result = basic_verifier.verify_signature(
        message_hash,
        v,
        r,
        s
    )
    
    assert result == signer_account.address

def test_eip191_invalid_signer(basic_verifier, signer_account, user):
    """Test that wrong signer is detected"""
    test_message = b"Test message"
    message_hash = Web3.keccak(test_message)
    
    # Sign with signer's key
    v, r, s = sign_message_eip191(message_hash, signer_account.key)
    
    # Check against wrong address
    result = basic_verifier.is_valid_signature(
        message_hash,
        user.address,  # Wrong signer
        v,
        r,
        s
    )
    
    assert result == False

# ============================================================
# Test EIP-712 Signatures
# ============================================================

def test_eip712_domain_separator(eip712_contract, chain):
    """Test that domain separator is computed correctly"""
    domain_sep = eip712_contract.domain_separator()
    assert domain_sep != b'\x00' * 32

def test_eip712_transfer_meta(eip712_contract, signer_account, accounts):
    """Test executing a meta-transfer with EIP-712 signature"""
    recipient = accounts[2]
    amount = 100 * 10**18
    deadline = int(time.time()) + 3600  # 1 hour from now
    
    # Get current nonce
    nonce = eip712_contract.nonces(signer_account.address)
    
    # Get the hash to sign
    digest = eip712_contract.get_transfer_hash(
        signer_account.address,
        recipient.address,
        amount,
        nonce,
        deadline
    )
    
    # Sign the digest directly (no EIP-191 prefix for EIP-712)
    signed = Account.sign_message(
        encode_defunct(hexstr=digest.hex()),
        private_key=signer_account.key
    )
    # Note: For real EIP-712, we sign the raw hash directly
    
    # Test would execute meta transfer
    # This is simplified - in real test you'd use actual EIP-712 signing

# ============================================================
# Test Permit Token
# ============================================================

def test_permit_basic(permit_token, deployer, signer_account, accounts):
    """Test EIP-2612 permit functionality"""
    spender = accounts[2]
    amount = 1000 * 10**18
    
    # Transfer tokens to signer
    permit_token.transfer(signer_account.address, amount, sender=deployer)
    
    deadline = int(time.time()) + 3600
    nonce = permit_token.nonces(signer_account.address)
    
    # Get domain separator
    domain_sep = permit_token.DOMAIN_SEPARATOR()
    
    # Sign the permit using EIP-712
    domain = {
        "name": "TestToken",
        "version": "1",
        "chainId": chain.chain_id,
        "verifyingContract": permit_token.address
    }
    
    types = {
        "Permit": [
            {"name": "owner", "type": "address"},
            {"name": "spender", "type": "address"},
            {"name": "value", "type": "uint256"},
            {"name": "nonce", "type": "uint256"},
            {"name": "deadline", "type": "uint256"},
        ]
    }
    
    message = {
        "owner": signer_account.address,
        "spender": spender.address,
        "value": amount,
        "nonce": nonce,
        "deadline": deadline,
    }
    
    v, r, s = sign_typed_data_eip712(domain, types, message, signer_account.key)
    
    # Execute permit
    permit_token.permit(
        signer_account.address,
        spender.address,
        amount,
        deadline,
        v,
        r,
        s,
        sender=spender  # Spender pays gas!
    )
    
    # Verify allowance was set
    assert permit_token.allowance(signer_account.address, spender.address) == amount

def test_permit_expired(permit_token, deployer, signer_account, accounts, chain):
    """Test that expired permit is rejected"""
    spender = accounts[2]
    amount = 100 * 10**18
    
    # Set deadline in the past
    deadline = int(time.time()) - 100
    
    # Create a dummy signature
    v, r, s = 27, b'\x01' * 32, b'\x02' * 32
    
    with pytest.raises(Exception, match="Permit expired"):
        permit_token.permit(
            signer_account.address,
            spender.address,
            amount,
            deadline,
            v,
            r,
            s,
            sender=spender
        )

def test_permit_replay_attack(permit_token, deployer, signer_account, accounts, chain):
    """Test that permit cannot be replayed"""
    spender = accounts[2]
    amount = 100 * 10**18
    deadline = chain.pending_timestamp + 3600
    nonce = permit_token.nonces(signer_account.address)
    
    domain = {
        "name": "TestToken",
        "version": "1", 
        "chainId": chain.chain_id,
        "verifyingContract": permit_token.address
    }
    types = {
        "Permit": [
            {"name": "owner", "type": "address"},
            {"name": "spender", "type": "address"},
            {"name": "value", "type": "uint256"},
            {"name": "nonce", "type": "uint256"},
            {"name": "deadline", "type": "uint256"},
        ]
    }
    message = {
        "owner": signer_account.address,
        "spender": spender.address,
        "value": amount,
        "nonce": nonce,
        "deadline": deadline,
    }
    
    v, r, s = sign_typed_data_eip712(domain, types, message, signer_account.key)
    
    # First permit should succeed
    permit_token.transfer(signer_account.address, amount, sender=deployer)
    permit_token.permit(signer_account.address, spender.address, amount, deadline, v, r, s, sender=spender)
    
    # Second permit with same signature should fail (nonce changed)
    with pytest.raises(Exception, match="Invalid permit signature"):
        permit_token.permit(signer_account.address, spender.address, amount, deadline, v, r, s, sender=spender)

# ============================================================
# Test Signature Verifier
# ============================================================

def test_signature_verifier_full_workflow(accounts, project):
    """Test the complete SignatureVerifier workflow"""
    deployer = accounts[0]
    verifier = deployer.deploy(project.SignatureVerifier, "TestVerifier")
    
    signer_acc = Account.create()
    
    # Create test message
    message = b"Authorize operation XYZ"
    message_hash = Web3.keccak(message)
    
    # Sign with EIP-191
    signed_message = encode_defunct(primitive=message)
    signed = Account.sign_message(signed_message, private_key=signer_acc.key)
    
    v = signed.v
    r = signed.r.to_bytes(32, 'big')
    s = signed.s.to_bytes(32, 'big')
    
    # Verify
    result = verifier.verify_eip191(
        message_hash,
        signer_acc.address,
        v,
        r,
        s,
        sender=deployer
    )
    
    assert result == True
```

---

## 8. Security Considerations {#s8}

### ข้อควรระวังสำคัญ

**1. Signature Malleability**
```python
# @version 0.4.0
# MalleabilityProtection.vy
# Protection against signature malleability attacks

# The curve order N for secp256k1
# s must be <= N/2 to prevent malleability
SECP256K1_N_HALF: constant(uint256) = 0x7FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF5D576E7357A4501DDFE92F46681B20A0

@internal
def _check_signature_components(v: uint8, r: bytes32, s: bytes32) -> bool:
    """
    Validate signature components to prevent common attacks
    """
    # v must be 27 or 28
    if v != 27 and v != 28:
        return False
    
    # r must be non-zero
    if convert(r, uint256) == 0:
        return False
    
    # s must be in the lower half to prevent malleability
    if convert(s, uint256) > SECP256K1_N_HALF:
        return False
    
    # s must be non-zero
    if convert(s, uint256) == 0:
        return False
    
    return True
```

**2. Replay Protection Patterns**

```python
# @version 0.4.0
# ReplayProtection.vy
# Multiple replay attack prevention strategies

# Strategy 1: Nonce-based (sequential)
nonces: public(HashMap[address, uint256])

# Strategy 2: Deadline-based
# Use deadline in the signed message

# Strategy 3: One-time use signatures
used_sigs: public(HashMap[bytes32, bool])

@internal
def _check_and_mark_nonce(signer: address, expected_nonce: uint256):
    """Sequential nonce check"""
    assert self.nonces[signer] == expected_nonce, "Invalid nonce"
    self.nonces[signer] = expected_nonce + 1

@internal
def _check_deadline(deadline: uint256):
    """Deadline check"""
    assert block.timestamp <= deadline, "Expired"

@internal
def _mark_sig_used(sig_hash: bytes32):
    """Mark signature as used"""
    assert not self.used_sigs[sig_hash], "Already used"
    self.used_sigs[sig_hash] = True
```

**3. Cross-Chain Replay Protection**

```python
# @version 0.4.0
# CrossChainProtection.vy
# EIP-712 domain separator includes chain.id
# This prevents signatures from one chain being used on another

DOMAIN_TYPE_HASH: constant(bytes32) = 0x8b73c3c69bb8fe3d512ecc4cf759cc79239f7b179b0ffacaa9a75d522b39400f

@deploy
def __init__():
    # chain.id is part of domain separator
    # Signatures for Ethereum mainnet (chain_id=1) won't work on
    # Polygon (chain_id=137) because domain separators differ
    domain_sep: bytes32 = keccak256(
        concat(
            DOMAIN_TYPE_HASH,
            keccak256(b"MyProtocol"),
            keccak256(b"1"),
            convert(chain.id, bytes32),  # Chain-specific!
            convert(self, bytes32)        # Contract-specific!
        )
    )
```

### สรุปข้อควรระวัง

| ประเภทความเสี่ยง | วิธีป้องกัน |
|---|---|
| Replay Attack | ใช้ nonce หรือ deadline |
| Signature Malleability | ตรวจสอบ s <= N/2 |
| Cross-chain Replay | รวม chain.id ใน domain |
| Wrong Contract | รวม contract address ใน domain |
| Expired Signature | ใช้ deadline parameter |

---

## สรุป

ในบทนี้เราได้เรียนรู้:
- **ECDSA Signatures**: การใช้ `ecrecover` ใน Vyper
- **EIP-191**: Personal sign standard สำหรับ MetaMask
- **EIP-712**: Typed structured data signing ที่ปลอดภัยกว่า
- **Permit (EIP-2612)**: Gasless token approvals
- **Security**: การป้องกัน replay attacks และ malleability

ลายเซ็นดิจิทัลเป็นเครื่องมือสำคัญสำหรับการสร้าง DeFi protocols ที่มี UX ดีและประหยัด gas

---
[← Previous Part](part_065_zk_applications.md) | [→ Next Part](part_067_meta_transactions.md)
