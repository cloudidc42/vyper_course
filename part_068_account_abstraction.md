# Part 068: Account Abstraction ERC-4337 (Account Abstraction และ ERC-4337)

## สารบัญ
1. [บทนำ Account Abstraction](#s1)
2. [UserOperation Struct](#s2)
3. [EntryPoint Interface](#s3)
4. [SimpleAccount Wallet](#s4)
5. [Paymaster Contract](#s5)
6. [Social Recovery System](#s6)
7. [การทดสอบด้วย pytest](#s7)
8. [Security Considerations](#s8)

---

## 1. บทนำ Account Abstraction {#s1}

**Account Abstraction (ERC-4337)** คือมาตรฐานที่เปลี่ยน Ethereum account model ให้ยืดหยุ่นขึ้น
แทนที่จะมีแค่ EOA (Externally Owned Account) ผู้ใช้สามารถมี smart contract wallet ที่กำหนดเองได้

### ปัญหาของ EOA แบบเดิม

- ต้องมี private key เสมอ
- ไม่สามารถกำหนด validation logic เอง
- ไม่รองรับ multi-sig natively
- ไม่มี social recovery
- ต้องจ่าย gas ด้วย ETH

### ERC-4337 แก้ปัญหาอย่างไร

```
User -> UserOperation -> Bundler -> EntryPoint -> SmartAccount
                                  (validates &
                                   executes)
```

- **UserOperation**: แทน transaction ธรรมดา
- **Bundler**: รวบรวม UserOps และ submit
- **EntryPoint**: Contract กลางที่ validate และ execute
- **SmartAccount**: Wallet contract ของผู้ใช้
- **Paymaster**: จ่าย gas แทนผู้ใช้

---

## 2. UserOperation Struct {#s2}

```python
# @version 0.4.0
# UserOperationStructs.vy
# ERC-4337 UserOperation type definitions

# UserOperation is the ERC-4337 equivalent of a transaction
# It contains all information needed to execute an operation
# via the EntryPoint

struct UserOperation:
    sender: address          # The smart account address
    nonce: uint256           # Anti-replay nonce
    init_code: Bytes[65536]  # For account creation (empty if exists)
    call_data: Bytes[65536]  # Encoded function call
    call_gas_limit: uint256  # Gas for actual execution
    verification_gas_limit: uint256  # Gas for validation
    pre_verification_gas: uint256    # Gas overhead
    max_fee_per_gas: uint256         # EIP-1559 max fee
    max_priority_fee_per_gas: uint256  # EIP-1559 tip
    paymaster_and_data: Bytes[65536] # Paymaster info (optional)
    signature: Bytes[65536]          # User's signature

# Hash for signing the UserOperation
struct UserOperationHash:
    sender: address
    nonce: uint256
    init_code_hash: bytes32
    call_data_hash: bytes32
    call_gas_limit: uint256
    verification_gas_limit: uint256
    pre_verification_gas: uint256
    max_fee_per_gas: uint256
    max_priority_fee_per_gas: uint256
    paymaster_and_data_hash: bytes32
    entry_point: address
    chain_id: uint256

# Compute hash of a UserOperation for signing
@view
@internal
def _hash_user_op(op: UserOperation) -> bytes32:
    """
    Compute the hash of a UserOperation
    This is what the account owner signs
    """
    op_hash: bytes32 = keccak256(
        concat(
            convert(op.sender, bytes32),
            convert(op.nonce, bytes32),
            keccak256(op.init_code),
            keccak256(op.call_data),
            convert(op.call_gas_limit, bytes32),
            convert(op.verification_gas_limit, bytes32),
            convert(op.pre_verification_gas, bytes32),
            convert(op.max_fee_per_gas, bytes32),
            convert(op.max_priority_fee_per_gas, bytes32),
            keccak256(op.paymaster_and_data)
        )
    )
    
    # Include EntryPoint and chain ID for cross-chain protection
    return keccak256(
        concat(
            op_hash,
            convert(self, bytes32),      # EntryPoint address
            convert(chain.id, bytes32)   # Chain ID
        )
    )
```

---

## 3. EntryPoint Interface {#s3}

```python
# @version 0.4.0
# IEntryPoint.vy
# Interface definition for ERC-4337 EntryPoint

# The EntryPoint is the singleton contract that handles all UserOps
# Official EntryPoint: 0x5FF137D4b0FDCD49DcA30c7CF57E578a026d2789

# Return value from validateUserOp
# packaged as uint256: validAfter | validUntil | sigFailed
# sigFailed: bit 0 (1 = failed, 0 = success)
# validUntil: bits 1-48 (unix timestamp, 0 = infinite)
# validAfter: bits 49-96 (unix timestamp)
SIG_VALIDATION_FAILED: constant(uint256) = 1

interface IAccount:
    def validateUserOp(
        userOp_sender: address,
        userOp_nonce: uint256,
        userOp_init_code: Bytes[65536],
        userOp_call_data: Bytes[65536],
        userOp_call_gas_limit: uint256,
        userOp_verification_gas_limit: uint256,
        userOp_pre_verification_gas: uint256,
        userOp_max_fee_per_gas: uint256,
        userOp_max_priority_fee_per_gas: uint256,
        userOp_paymaster_and_data: Bytes[65536],
        userOp_signature: Bytes[65536],
        user_op_hash: bytes32,
        missing_account_funds: uint256
    ) -> uint256: nonpayable

interface IPaymaster:
    def validatePaymasterUserOp(
        userOp_sender: address,
        userOp_nonce: uint256,
        userOp_call_data: Bytes[65536],
        userOp_paymaster_and_data: Bytes[65536],
        userOp_signature: Bytes[65536],
        user_op_hash: bytes32,
        max_cost: uint256
    ) -> (bytes32, uint256): nonpayable
    
    def postOp(
        mode: uint256,
        context: bytes32,
        actual_gas_cost: uint256
    ): nonpayable

interface IAggregator:
    def validateSignatures(
        ops: DynArray[address, 100],  # Simplified
        signature: Bytes[65536]
    ): view
```

---

## 4. SimpleAccount Wallet {#s4}

```python
# @version 0.4.0
# SimpleAccount.vy
# ERC-4337 Simple Smart Contract Wallet
# Supports single-owner validation, execute, and executeBatch

# ============================================================
# Interfaces
# ============================================================

interface IEntryPoint:
    def getNonce(sender: address, key: uint192) -> uint256: view
    def getUserOpHash(
        sender: address,
        nonce: uint256,
        init_code_hash: bytes32,
        call_data_hash: bytes32,
        call_gas_limit: uint256,
        verification_gas_limit: uint256,
        pre_verification_gas: uint256,
        max_fee_per_gas: uint256,
        max_priority_fee_per_gas: uint256,
        paymaster_and_data_hash: bytes32
    ) -> bytes32: view
    def depositTo(account: address): payable
    def balanceOf(account: address) -> uint256: view
    def withdrawTo(withdrawAddress: address, withdrawAmount: uint256): nonpayable

# ============================================================
# Constants
# ============================================================

# ERC-4337 EntryPoint address (well-known singleton)
# In tests, use mock address
ENTRY_POINT: immutable(address)

# Signature validation failure return value
SIG_VALIDATION_FAILED: constant(uint256) = 1

# ============================================================
# Storage
# ============================================================

owner: public(address)
initialized: bool

# Nonce tracked by EntryPoint (not stored here)

# Guardian system for recovery
guardians: public(HashMap[address, bool])
guardian_count: public(uint256)
recovery_threshold: public(uint256)

# Pending recovery
pending_recovery: public(address)
recovery_votes: public(HashMap[address, bool])  # guardian => voted
recovery_vote_count: public(uint256)

# Execution history for auditing
execution_count: public(uint256)

# ============================================================
# Events
# ============================================================

event AccountInitialized:
    entry_point: indexed(address)
    owner: indexed(address)

event CallExecuted:
    target: indexed(address)
    value: uint256
    success: bool

event BatchExecuted:
    count: uint256

event OwnershipTransferred:
    old_owner: indexed(address)
    new_owner: indexed(address)

event GuardianAdded:
    guardian: indexed(address)

event GuardianRemoved:
    guardian: indexed(address)

event RecoveryInitiated:
    proposed_owner: indexed(address)
    initiated_by: indexed(address)

event RecoveryCompleted:
    new_owner: indexed(address)

# ============================================================
# Constructor
# ============================================================

@deploy
def __init__(entry_point: address):
    ENTRY_POINT = entry_point

@external
def initialize(owner: address):
    """
    Initialize the account (called by factory)
    Separate from constructor to support CREATE2 deployment
    """
    assert not self.initialized, "Already initialized"
    assert owner != empty(address), "Invalid owner"
    
    self.initialized = True
    self.owner = owner
    
    log AccountInitialized(ENTRY_POINT, owner)

# ============================================================
# Modifiers (implemented as internal functions)
# ============================================================

@internal
def _only_entry_point():
    """Restrict to EntryPoint or owner (for direct calls)"""
    assert msg.sender == ENTRY_POINT or msg.sender == self.owner, \
        "Not authorized"

@internal
def _only_owner():
    """Restrict to owner only"""
    assert msg.sender == self.owner or msg.sender == self, \
        "Not owner"

# ============================================================
# ERC-4337 Core Functions
# ============================================================

@external
def validateUserOp(
    sender: address,
    nonce: uint256,
    init_code: Bytes[65536],
    call_data: Bytes[65536],
    call_gas_limit: uint256,
    verification_gas_limit: uint256,
    pre_verification_gas: uint256,
    max_fee_per_gas: uint256,
    max_priority_fee_per_gas: uint256,
    paymaster_and_data: Bytes[65536],
    signature: Bytes[65536],
    user_op_hash: bytes32,
    missing_account_funds: uint256
) -> uint256:
    """
    Validate a UserOperation
    Called by EntryPoint during handleOps
    @return 0 if valid, 1 if signature failed
    Validation data can also include time range (validAfter/validUntil)
    """
    # Only EntryPoint can call this
    assert msg.sender == ENTRY_POINT, "Not entry point"
    
    # Validate signature
    validation_result: uint256 = self._validate_signature(
        user_op_hash,
        signature
    )
    
    if validation_result == SIG_VALIDATION_FAILED:
        return SIG_VALIDATION_FAILED
    
    # Validate nonce (EntryPoint handles this but we can add extra checks)
    self._validate_and_update_nonce(sender, nonce)
    
    # Pay prefund to EntryPoint if needed
    if missing_account_funds > 0:
        # Send funds to EntryPoint to cover gas
        raw_call(ENTRY_POINT, b"", value=missing_account_funds)
    
    return 0  # Validation successful

@internal
def _validate_signature(user_op_hash: bytes32, signature: Bytes[65536]) -> uint256:
    """
    Validate the signature on a UserOperation
    Default: ECDSA signature from owner
    Can be overridden for multi-sig or other schemes
    """
    if len(signature) != 65:
        return SIG_VALIDATION_FAILED
    
    # Extract v, r, s from signature bytes
    # Signature format: r (32) + s (32) + v (1)
    r: bytes32 = convert(slice(signature, 0, 32), bytes32)
    s: bytes32 = convert(slice(signature, 32, 32), bytes32)
    v: uint8 = convert(slice(signature, 64, 1), uint8)
    
    # Recover signer
    signer: address = ecrecover(user_op_hash, v, r, s)
    
    if signer == empty(address):
        return SIG_VALIDATION_FAILED
    
    if signer != self.owner:
        return SIG_VALIDATION_FAILED
    
    return 0  # Valid

@internal
def _validate_and_update_nonce(account: address, nonce: uint256):
    """
    Validate nonce is correct
    ERC-4337 manages nonces in EntryPoint, but accounts can add extra validation
    """
    # EntryPoint handles nonce validation
    # We just verify account matches
    assert account == self, "Wrong account"

# ============================================================
# Execute Functions
# ============================================================

@external
def execute(
    target: address,
    value: uint256,
    data: Bytes[65536]
) -> bool:
    """
    Execute a single call
    Called by EntryPoint after validation
    """
    self._only_entry_point()
    
    assert target != empty(address), "Invalid target"
    
    # Execute the call
    success: bool = False
    ret: Bytes[65536] = b""
    
    success, ret = raw_call(
        target,
        data,
        max_outsize=65536,
        value=value,
        revert_on_failure=False
    )
    
    self.execution_count += 1
    log CallExecuted(target, value, success)
    
    assert success, "Execution failed"
    return True

@external
def executeBatch(
    targets: DynArray[address, 20],
    values: DynArray[uint256, 20],
    datas: DynArray[Bytes[65536], 20]
) -> bool:
    """
    Execute multiple calls in a single UserOperation
    More efficient than multiple individual operations
    """
    self._only_entry_point()
    
    count: uint256 = len(targets)
    assert count == len(values), "Length mismatch"
    assert count == len(datas), "Length mismatch"
    assert count <= 20, "Max 20 calls per batch"
    
    for i: uint256 in range(20):
        if i >= count:
            break
        
        assert targets[i] != empty(address), "Invalid target"
        
        success: bool = False
        ret: Bytes[65536] = b""
        
        success, ret = raw_call(
            targets[i],
            datas[i],
            max_outsize=65536,
            value=values[i],
            revert_on_failure=False
        )
        
        assert success, "Batch call failed"
        log CallExecuted(targets[i], values[i], True)
    
    log BatchExecuted(count)
    return True

# ============================================================
# Ownership Management
# ============================================================

@external
def transferOwnership(new_owner: address):
    """Transfer ownership (must be called through EntryPoint)"""
    self._only_owner()
    assert new_owner != empty(address), "Invalid address"
    
    old_owner: address = self.owner
    self.owner = new_owner
    
    log OwnershipTransferred(old_owner, new_owner)

# ============================================================
# Guardian Management (Social Recovery)
# ============================================================

@external
def addGuardian(guardian: address):
    """Add a guardian for social recovery"""
    self._only_owner()
    assert guardian != empty(address), "Invalid guardian"
    assert not self.guardians[guardian], "Already guardian"
    
    self.guardians[guardian] = True
    self.guardian_count += 1
    
    # Update threshold to majority
    self.recovery_threshold = self.guardian_count / 2 + 1
    
    log GuardianAdded(guardian)

@external
def removeGuardian(guardian: address):
    """Remove a guardian"""
    self._only_owner()
    assert self.guardians[guardian], "Not a guardian"
    
    self.guardians[guardian] = False
    self.guardian_count -= 1
    
    if self.guardian_count > 0:
        self.recovery_threshold = self.guardian_count / 2 + 1
    else:
        self.recovery_threshold = 0
    
    log GuardianRemoved(guardian)

@external
def initiateRecovery(proposed_new_owner: address):
    """Guardian initiates recovery process"""
    assert self.guardians[msg.sender], "Not a guardian"
    assert proposed_new_owner != empty(address), "Invalid owner"
    
    # Reset if different proposal
    if self.pending_recovery != proposed_new_owner:
        self.pending_recovery = proposed_new_owner
        self.recovery_vote_count = 0
        # Reset existing votes would require different storage pattern
    
    # Cast vote if not already voted
    if not self.recovery_votes[msg.sender]:
        self.recovery_votes[msg.sender] = True
        self.recovery_vote_count += 1
        log RecoveryInitiated(proposed_new_owner, msg.sender)

@external
def executeRecovery():
    """Execute recovery if threshold reached"""
    assert self.pending_recovery != empty(address), "No pending recovery"
    assert self.recovery_vote_count >= self.recovery_threshold, "Insufficient votes"
    
    new_owner: address = self.pending_recovery
    
    # Reset recovery state
    self.pending_recovery = empty(address)
    self.recovery_vote_count = 0
    
    # Transfer ownership
    old_owner: address = self.owner
    self.owner = new_owner
    
    log RecoveryCompleted(new_owner)
    log OwnershipTransferred(old_owner, new_owner)

# ============================================================
# Utility Functions
# ============================================================

@view
@external
def getDeposit() -> uint256:
    """Get the account's deposit in EntryPoint"""
    return IEntryPoint(ENTRY_POINT).balanceOf(self)

@external
@payable
def addDeposit():
    """Add a deposit to EntryPoint for gas prepayment"""
    IEntryPoint(ENTRY_POINT).depositTo(self, value=msg.value)

@external
def withdrawDeposit(withdraw_address: address, amount: uint256):
    """Withdraw deposit from EntryPoint"""
    self._only_owner()
    IEntryPoint(ENTRY_POINT).withdrawTo(withdraw_address, amount)

@external
@payable
def __default__():
    """Receive ETH"""
    pass
```

---

## 5. Paymaster Contract {#s5}

Paymaster จ่าย gas แทนผู้ใช้ ช่วยให้ผู้ใช้ทำ transaction โดยไม่ต้องมี ETH

```python
# @version 0.4.0
# VerifyingPaymaster.vy
# ERC-4337 Paymaster that sponsors gas based on signature verification
# Can sponsor transactions for approved users/operations

# ============================================================
# Interfaces
# ============================================================

interface IEntryPoint:
    def depositTo(account: address): payable
    def withdrawTo(withdrawAddress: address, withdrawAmount: uint256): nonpayable
    def balanceOf(account: address) -> uint256: view
    def addStake(unstakeDelaySec: uint32): payable
    def unlockStake(): nonpayable
    def withdrawStake(withdrawAddress: address): nonpayable

# ============================================================
# Constants
# ============================================================

ENTRY_POINT: immutable(address)

# Paymaster operation modes
POST_OP_MODE_OP_SUCCEEDED: constant(uint256) = 0
POST_OP_MODE_OP_REVERTED: constant(uint256) = 1
POST_OP_MODE_POSTOP_REVERTED: constant(uint256) = 2

# ============================================================
# Structs
# ============================================================

struct PaymasterData:
    valid_until: uint48   # Timestamp when sponsorship expires
    valid_after: uint48   # Timestamp when sponsorship starts
    sponsor: address      # Who is sponsoring

# ============================================================
# Storage
# ============================================================

owner: public(address)
signer: public(address)  # Signs paymaster data

# Spending limits per user per day
daily_spending_limit: public(uint256)
user_daily_spent: public(HashMap[address, uint256])
user_last_reset: public(HashMap[address, uint256])

# Whitelisted contracts/functions that are free
whitelisted_targets: public(HashMap[address, bool])

# Total gas sponsored
total_gas_sponsored: public(uint256)

# ============================================================
# Events
# ============================================================

event GasSponsored:
    user: indexed(address)
    amount: uint256

event PostOpCompleted:
    mode: uint256
    actual_cost: uint256

event SpendingLimitUpdated:
    new_limit: uint256

# ============================================================
# Constructor
# ============================================================

@deploy
def __init__(entry_point: address, paymaster_signer: address):
    ENTRY_POINT = entry_point
    self.owner = msg.sender
    self.signer = paymaster_signer
    self.daily_spending_limit = 10**18  # 1 ETH per day default

# ============================================================
# Internal Functions
# ============================================================

@internal
def _parse_paymaster_data(paymaster_and_data: Bytes[65536]) -> PaymasterData:
    """
    Parse the paymaster data from UserOperation
    Format: [paymaster_address (20)] [valid_until (6)] [valid_after (6)] [signature (65)]
    """
    # Extract validity timestamps
    valid_until: uint48 = convert(
        slice(paymaster_and_data, 20, 6), 
        uint48
    )
    valid_after: uint48 = convert(
        slice(paymaster_and_data, 26, 6),
        uint48
    )
    
    return PaymasterData(
        valid_until=valid_until,
        valid_after=valid_after,
        sponsor=self
    )

@internal
def _verify_paymaster_signature(
    user_op_hash: bytes32,
    valid_until: uint48,
    valid_after: uint48,
    signature: Bytes[65]
) -> bool:
    """Verify the paymaster's signature on UserOperation data"""
    # Create hash that paymaster signs
    hash_to_sign: bytes32 = keccak256(
        concat(
            user_op_hash,
            convert(convert(valid_until, uint256), bytes32),
            convert(convert(valid_after, uint256), bytes32)
        )
    )
    
    # Extract signature components
    if len(signature) != 65:
        return False
    
    r: bytes32 = convert(slice(signature, 0, 32), bytes32)
    s: bytes32 = convert(slice(signature, 32, 32), bytes32)
    v: uint8 = convert(slice(signature, 64, 1), uint8)
    
    recovered: address = ecrecover(hash_to_sign, v, r, s)
    
    return recovered == self.signer

@internal
def _check_spending_limit(user: address, amount: uint256):
    """Check and update user's daily spending limit"""
    # Reset if new day
    if block.timestamp >= self.user_last_reset[user] + 86400:
        self.user_daily_spent[user] = 0
        self.user_last_reset[user] = block.timestamp
    
    assert self.user_daily_spent[user] + amount <= self.daily_spending_limit, \
        "Daily limit exceeded"

# ============================================================
# ERC-4337 Paymaster Interface
# ============================================================

@external
def validatePaymasterUserOp(
    sender: address,
    nonce: uint256,
    call_data: Bytes[65536],
    paymaster_and_data: Bytes[65536],
    signature: Bytes[65536],
    user_op_hash: bytes32,
    max_cost: uint256
) -> (bytes32, uint256):
    """
    Validate that we'll pay for this UserOperation
    @return (context, validationData)
    context: passed to postOp
    validationData: packaged validation result (sig failure, time range)
    """
    assert msg.sender == ENTRY_POINT, "Not entry point"
    
    # Parse paymaster data
    pm_data: PaymasterData = self._parse_paymaster_data(paymaster_and_data)
    
    # Extract paymaster signature (last 65 bytes)
    pm_signature: Bytes[65] = slice(paymaster_and_data, len(paymaster_and_data) - 65, 65)
    
    # Verify paymaster's signature
    sig_valid: bool = self._verify_paymaster_signature(
        user_op_hash,
        pm_data.valid_until,
        pm_data.valid_after,
        pm_signature
    )
    
    if not sig_valid:
        # Return sig failure
        return (empty(bytes32), 1)
    
    # Check spending limit
    self._check_spending_limit(sender, max_cost)
    
    # Pack context: store sender for postOp
    context: bytes32 = convert(sender, bytes32)
    
    # Pack validation data with time range
    # Format: validAfter | validUntil | sigFailed
    validation_data: uint256 = (
        convert(pm_data.valid_after, uint256) << 48 |
        convert(pm_data.valid_until, uint256) << 0
    )
    
    return (context, validation_data)

@external
def postOp(
    mode: uint256,
    context: bytes32,
    actual_gas_cost: uint256
):
    """
    Called after operation execution
    Used for accounting and refunds
    @param mode 0=success, 1=reverted, 2=postOp reverted
    @param context Data returned from validatePaymasterUserOp
    @param actual_gas_cost Actual gas used
    """
    assert msg.sender == ENTRY_POINT, "Not entry point"
    
    # Extract user from context
    user: address = convert(context, address)
    
    if mode == POST_OP_MODE_OP_SUCCEEDED:
        # Update spending tracker
        self.user_daily_spent[user] += actual_gas_cost
        self.total_gas_sponsored += actual_gas_cost
        log GasSponsored(user, actual_gas_cost)
    
    log PostOpCompleted(mode, actual_gas_cost)

# ============================================================
# Admin Functions
# ============================================================

@external
def setDailySpendingLimit(limit: uint256):
    """Update daily spending limit per user"""
    assert msg.sender == self.owner, "Not owner"
    self.daily_spending_limit = limit
    log SpendingLimitUpdated(limit)

@external
def addWhitelistedTarget(target: address):
    """Add a whitelisted contract (unlimited sponsorship)"""
    assert msg.sender == self.owner, "Not owner"
    self.whitelisted_targets[target] = True

@external
def updateSigner(new_signer: address):
    """Update the paymaster's signer key"""
    assert msg.sender == self.owner, "Not owner"
    assert new_signer != empty(address), "Invalid signer"
    self.signer = new_signer

@external
@payable
def deposit():
    """Add ETH to EntryPoint for sponsoring gas"""
    IEntryPoint(ENTRY_POINT).depositTo(self, value=msg.value)

@external
def withdrawDeposit(to: address, amount: uint256):
    """Withdraw deposited ETH"""
    assert msg.sender == self.owner, "Not owner"
    IEntryPoint(ENTRY_POINT).withdrawTo(to, amount)

@external
@payable
def __default__():
    """Receive ETH"""
    pass
```

### Token Paymaster (ERC-20 Gas Payment)

```python
# @version 0.4.0
# TokenPaymaster.vy
# Paymaster that accepts ERC-20 tokens as payment for gas

from vyper.interfaces import ERC20

interface IEntryPoint:
    def depositTo(account: address): payable
    def balanceOf(account: address) -> uint256: view

# Price oracle interface
interface IOracle:
    def getPrice() -> uint256: view  # Returns token price in ETH (18 decimals)

ENTRY_POINT: immutable(address)

# ERC-20 token accepted as payment
payment_token: public(immutable(address))

# Oracle for token/ETH price
oracle: public(address)

owner: public(address)

# Markup over ETH price (in basis points, 10000 = 100%)
price_markup: public(uint256)

# Amount collected in tokens
tokens_collected: public(uint256)

event GasPaidInToken:
    user: indexed(address)
    token_amount: uint256
    eth_equivalent: uint256

@deploy
def __init__(entry_point: address, token: address, price_oracle: address):
    ENTRY_POINT = entry_point
    payment_token = token
    self.oracle = price_oracle
    self.owner = msg.sender
    self.price_markup = 11000  # 110% of actual cost

@internal
def _get_token_amount(eth_amount: uint256) -> uint256:
    """Convert ETH amount to token amount using oracle"""
    eth_price: uint256 = IOracle(self.oracle).getPrice()
    # token_amount = eth_amount * markup / price
    return eth_amount * self.price_markup * 10**18 / (eth_price * 10000)

@external
def validatePaymasterUserOp(
    sender: address,
    nonce: uint256,
    call_data: Bytes[65536],
    paymaster_and_data: Bytes[65536],
    signature: Bytes[65536],
    user_op_hash: bytes32,
    max_cost: uint256
) -> (bytes32, uint256):
    """Validate and approve token payment"""
    assert msg.sender == ENTRY_POINT, "Not entry point"
    
    # Calculate token cost
    token_cost: uint256 = self._get_token_amount(max_cost)
    
    # Check allowance
    allowance: uint256 = ERC20(payment_token).allowance(sender, self)
    assert allowance >= token_cost, "Insufficient token allowance"
    
    # Transfer tokens now (pre-charge)
    ERC20(payment_token).transferFrom(sender, self, token_cost)
    self.tokens_collected += token_cost
    
    # Pack context with token cost for refund in postOp
    context: bytes32 = convert(token_cost, bytes32)
    
    log GasPaidInToken(sender, token_cost, max_cost)
    
    return (context, 0)  # 0 = no time restriction

@external
def postOp(mode: uint256, context: bytes32, actual_gas_cost: uint256):
    """Refund unused gas tokens"""
    assert msg.sender == ENTRY_POINT, "Not entry point"
    
    # In real implementation, calculate and refund excess tokens
    # Simplified here
    pass
```

---

## 6. Social Recovery System {#s6}

```python
# @version 0.4.0
# SocialRecoveryWallet.vy
# Smart wallet with social recovery and time-lock

# ============================================================
# Constants
# ============================================================

ENTRY_POINT: immutable(address)
RECOVERY_DELAY: constant(uint256) = 86400 * 3  # 3 days delay
MAX_GUARDIANS: constant(uint256) = 10

# ============================================================
# Structs
# ============================================================

struct RecoveryRequest:
    proposed_owner: address
    initiation_time: uint256
    vote_count: uint256
    executed: bool

# ============================================================
# Storage
# ============================================================

owner: public(address)
initialized: bool

# Guardian system
guardians: public(DynArray[address, 10])
is_guardian: public(HashMap[address, bool])
required_approvals: public(uint256)

# Recovery process
active_recovery: public(RecoveryRequest)
guardian_votes: public(HashMap[address, bool])  # guardian => voted for active recovery

# Emergency freeze
frozen: public(bool)
freeze_votes: public(HashMap[address, bool])
freeze_vote_count: public(uint256)

# ============================================================
# Events
# ============================================================

event WalletInitialized:
    owner: indexed(address)
    guardian_count: uint256

event RecoveryStarted:
    proposed_owner: indexed(address)
    by_guardian: indexed(address)
    unlock_time: uint256

event RecoveryVoted:
    guardian: indexed(address)
    votes_count: uint256

event RecoveryExecuted:
    new_owner: indexed(address)

event RecoveryCancelled:
    cancelled_by: indexed(address)

event WalletFrozen:
    by: indexed(address)

event WalletUnfrozen:
    by: indexed(address)

# ============================================================
# Constructor
# ============================================================

@deploy
def __init__(entry_point: address):
    ENTRY_POINT = entry_point

@external
def initialize(
    wallet_owner: address,
    initial_guardians: DynArray[address, 10],
    threshold: uint256
):
    """Initialize wallet with owner and guardians"""
    assert not self.initialized, "Already initialized"
    assert wallet_owner != empty(address), "Invalid owner"
    assert len(initial_guardians) >= threshold, "Threshold too high"
    assert threshold > 0, "Threshold must be positive"
    
    self.initialized = True
    self.owner = wallet_owner
    self.required_approvals = threshold
    
    for guardian: address in initial_guardians:
        assert guardian != empty(address), "Invalid guardian"
        assert not self.is_guardian[guardian], "Duplicate guardian"
        
        self.guardians.append(guardian)
        self.is_guardian[guardian] = True
    
    log WalletInitialized(wallet_owner, len(initial_guardians))

# ============================================================
# Core Wallet Functions
# ============================================================

@external
def validateUserOp(
    sender: address,
    nonce: uint256,
    init_code: Bytes[65536],
    call_data: Bytes[65536],
    call_gas_limit: uint256,
    verification_gas_limit: uint256,
    pre_verification_gas: uint256,
    max_fee_per_gas: uint256,
    max_priority_fee_per_gas: uint256,
    paymaster_and_data: Bytes[65536],
    signature: Bytes[65536],
    user_op_hash: bytes32,
    missing_account_funds: uint256
) -> uint256:
    """ERC-4337 validation"""
    assert msg.sender == ENTRY_POINT, "Not entry point"
    assert not self.frozen, "Wallet is frozen"
    
    # Validate owner signature
    if len(signature) != 65:
        return 1  # SIG_VALIDATION_FAILED
    
    r: bytes32 = convert(slice(signature, 0, 32), bytes32)
    s: bytes32 = convert(slice(signature, 32, 32), bytes32)
    v: uint8 = convert(slice(signature, 64, 1), uint8)
    
    signer: address = ecrecover(user_op_hash, v, r, s)
    
    if signer != self.owner or signer == empty(address):
        return 1  # SIG_VALIDATION_FAILED
    
    if missing_account_funds > 0:
        raw_call(ENTRY_POINT, b"", value=missing_account_funds)
    
    return 0

@external
def execute(target: address, value: uint256, data: Bytes[65536]):
    """Execute a transaction"""
    assert msg.sender == ENTRY_POINT or msg.sender == self.owner, "Unauthorized"
    assert not self.frozen, "Wallet frozen"
    
    success: bool = False
    ret: Bytes[65536] = b""
    success, ret = raw_call(target, data, max_outsize=65536, value=value, revert_on_failure=False)
    assert success, "Execution failed"

# ============================================================
# Recovery Functions
# ============================================================

@external
def startRecovery(proposed_owner: address):
    """
    Guardian initiates recovery
    Requires RECOVERY_DELAY before execution
    """
    assert self.is_guardian[msg.sender], "Not a guardian"
    assert proposed_owner != empty(address), "Invalid owner"
    assert not self.active_recovery.proposed_owner != empty(address) or \
           self.active_recovery.executed, "Recovery in progress"
    
    # Start new recovery
    self.active_recovery = RecoveryRequest(
        proposed_owner=proposed_owner,
        initiation_time=block.timestamp,
        vote_count=1,
        executed=False
    )
    
    # Record initiator's vote
    self.guardian_votes[msg.sender] = True
    
    unlock_time: uint256 = block.timestamp + RECOVERY_DELAY
    log RecoveryStarted(proposed_owner, msg.sender, unlock_time)

@external
def voteForRecovery():
    """Guardian votes for active recovery proposal"""
    assert self.is_guardian[msg.sender], "Not a guardian"
    assert self.active_recovery.proposed_owner != empty(address), "No active recovery"
    assert not self.active_recovery.executed, "Already executed"
    assert not self.guardian_votes[msg.sender], "Already voted"
    
    self.guardian_votes[msg.sender] = True
    self.active_recovery.vote_count += 1
    
    log RecoveryVoted(msg.sender, self.active_recovery.vote_count)

@external
def executeRecovery():
    """
    Execute recovery after delay and sufficient votes
    Anyone can call this once conditions are met
    """
    assert self.active_recovery.proposed_owner != empty(address), "No active recovery"
    assert not self.active_recovery.executed, "Already executed"
    assert self.active_recovery.vote_count >= self.required_approvals, \
        "Insufficient votes"
    assert block.timestamp >= self.active_recovery.initiation_time + RECOVERY_DELAY, \
        "Delay not passed"
    
    new_owner: address = self.active_recovery.proposed_owner
    
    # Mark as executed
    self.active_recovery.executed = True
    
    # Transfer ownership
    self.owner = new_owner
    self.frozen = False  # Unfreeze on recovery
    
    log RecoveryExecuted(new_owner)

@external
def cancelRecovery():
    """Owner or majority guardians cancel recovery"""
    if msg.sender == self.owner:
        # Owner can cancel any time
        self.active_recovery = RecoveryRequest(
            proposed_owner=empty(address),
            initiation_time=0,
            vote_count=0,
            executed=True
        )
        log RecoveryCancelled(msg.sender)
    else:
        # Non-owner cancel not supported in this simplified version
        raise "Only owner can cancel"

# ============================================================
# Emergency Freeze
# ============================================================

@external
def voteToFreeze():
    """Guardian votes to freeze the wallet"""
    assert self.is_guardian[msg.sender], "Not a guardian"
    assert not self.freeze_votes[msg.sender], "Already voted"
    
    self.freeze_votes[msg.sender] = True
    self.freeze_vote_count += 1
    
    # Auto-freeze if majority
    if self.freeze_vote_count >= self.required_approvals:
        self.frozen = True
        log WalletFrozen(msg.sender)

@external
def unfreeze():
    """Owner unfreezes the wallet"""
    assert msg.sender == self.owner, "Not owner"
    self.frozen = False
    self.freeze_vote_count = 0
    log WalletUnfrozen(msg.sender)

# ============================================================
# View Functions
# ============================================================

@view
@external
def getRecoveryStatus() -> (address, uint256, uint256, bool):
    """Get current recovery request status"""
    return (
        self.active_recovery.proposed_owner,
        self.active_recovery.vote_count,
        self.active_recovery.initiation_time,
        self.active_recovery.executed
    )

@view
@external
def getGuardians() -> DynArray[address, 10]:
    """Get list of all guardians"""
    return self.guardians

@external
@payable
def __default__():
    pass
```

---

## 7. การทดสอบด้วย pytest {#s7}

```python
# tests/test_account_abstraction.py
# Tests for ERC-4337 Account Abstraction contracts

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
def user_key():
    """Generate a test private key for user"""
    acc = Account.create()
    return acc

@pytest.fixture
def guardian1(accounts):
    return accounts[2]

@pytest.fixture
def guardian2(accounts):
    return accounts[3]

@pytest.fixture
def guardian3(accounts):
    return accounts[4]

@pytest.fixture
def mock_entry_point(deployer, project):
    """Deploy a mock EntryPoint for testing"""
    return deployer.deploy(project.MockEntryPoint)

@pytest.fixture
def simple_account(deployer, mock_entry_point, user_key, project):
    """Deploy and initialize a SimpleAccount"""
    wallet = deployer.deploy(project.SimpleAccount, mock_entry_point.address)
    wallet.initialize(user_key.address, sender=deployer)
    return wallet

@pytest.fixture
def social_wallet(
    deployer, mock_entry_point, user_key, 
    guardian1, guardian2, guardian3, project
):
    """Deploy SocialRecoveryWallet"""
    wallet = deployer.deploy(project.SocialRecoveryWallet, mock_entry_point.address)
    guardians = [guardian1.address, guardian2.address, guardian3.address]
    wallet.initialize(user_key.address, guardians, 2, sender=deployer)
    return wallet

# ============================================================
# Test SimpleAccount
# ============================================================

def test_simple_account_initialized(simple_account, user_key):
    """Test account is properly initialized"""
    assert simple_account.owner() == user_key.address

def test_validate_user_op_valid_sig(simple_account, user_key, mock_entry_point, deployer):
    """Test that valid signature passes validation"""
    # Create a mock user_op_hash
    user_op_hash = Web3.keccak(b"test user operation")
    
    # Sign it
    msg = encode_defunct(hexstr=user_op_hash.hex())
    signed = Account.sign_message(msg, private_key=user_key.key)
    
    # Pack signature as r+s+v
    r = signed.r.to_bytes(32, 'big')
    s = signed.s.to_bytes(32, 'big')
    v = signed.v.to_bytes(1, 'big')
    signature = r + s + v
    
    # Validate (called by mock entry point)
    result = simple_account.validateUserOp(
        simple_account.address,  # sender
        0,                        # nonce
        b"",                      # init_code
        b"",                      # call_data
        100000,                   # call_gas_limit
        100000,                   # verification_gas_limit
        50000,                    # pre_verification_gas
        10**9,                    # max_fee_per_gas
        10**9,                    # max_priority_fee_per_gas
        b"",                      # paymaster_and_data
        signature,                # signature
        user_op_hash,             # user_op_hash
        0,                        # missing_account_funds
        sender=mock_entry_point
    )
    
    assert result == 0  # 0 = valid

def test_validate_user_op_invalid_sig(simple_account, user_key, mock_entry_point):
    """Test that invalid signature fails validation"""
    user_op_hash = Web3.keccak(b"test operation")
    
    # Sign with wrong key
    wrong_account = Account.create()
    msg = encode_defunct(hexstr=user_op_hash.hex())
    signed = Account.sign_message(msg, private_key=wrong_account.key)
    
    r = signed.r.to_bytes(32, 'big')
    s = signed.s.to_bytes(32, 'big')
    v = signed.v.to_bytes(1, 'big')
    signature = r + s + v
    
    result = simple_account.validateUserOp(
        simple_account.address,
        0, b"", b"",
        100000, 100000, 50000,
        10**9, 10**9, b"",
        signature, user_op_hash, 0,
        sender=mock_entry_point
    )
    
    assert result == 1  # 1 = sig failed

def test_execute_call(simple_account, mock_entry_point, deployer, accounts):
    """Test executing a call via entry point"""
    target = accounts[9]
    target_initial_balance = target.balance
    
    # Fund the wallet
    deployer.transfer(simple_account.address, "1 ether")
    
    # Execute transfer via entry point
    simple_account.execute(
        target.address,
        10**17,  # 0.1 ETH
        b"",
        sender=mock_entry_point
    )
    
    assert target.balance == target_initial_balance + 10**17

def test_execute_batch(simple_account, mock_entry_point, deployer, accounts):
    """Test batch execution"""
    target1 = accounts[7]
    target2 = accounts[8]
    
    deployer.transfer(simple_account.address, "1 ether")
    
    simple_account.executeBatch(
        [target1.address, target2.address],
        [10**17, 2 * 10**17],
        [b"", b""],
        sender=mock_entry_point
    )
    
    assert simple_account.execution_count() == 2

# ============================================================
# Test Social Recovery
# ============================================================

def test_social_wallet_initialized(social_wallet, user_key, guardian1, guardian2, guardian3):
    """Test social wallet initialized correctly"""
    assert social_wallet.owner() == user_key.address
    assert social_wallet.is_guardian(guardian1.address) == True
    assert social_wallet.is_guardian(guardian2.address) == True
    assert social_wallet.required_approvals() == 2

def test_start_recovery(social_wallet, guardian1):
    """Test guardian can start recovery"""
    new_owner = Account.create().address
    social_wallet.startRecovery(new_owner, sender=guardian1)
    
    proposed, votes, init_time, executed = social_wallet.getRecoveryStatus()
    assert proposed == new_owner
    assert votes == 1
    assert not executed

def test_recovery_vote(social_wallet, guardian1, guardian2):
    """Test multiple guardians can vote"""
    new_owner = Account.create().address
    
    social_wallet.startRecovery(new_owner, sender=guardian1)
    social_wallet.voteForRecovery(sender=guardian2)
    
    _, votes, _, _ = social_wallet.getRecoveryStatus()
    assert votes == 2

def test_recovery_executes_after_delay(social_wallet, guardian1, guardian2, chain, user_key):
    """Test recovery executes after time delay and sufficient votes"""
    new_owner = Account.create()
    
    # Start recovery and vote
    social_wallet.startRecovery(new_owner.address, sender=guardian1)
    social_wallet.voteForRecovery(sender=guardian2)
    
    # Fast-forward time
    chain.pending_timestamp += 86400 * 4  # 4 days
    chain.mine()
    
    # Execute recovery
    social_wallet.executeRecovery(sender=guardian1)
    
    assert social_wallet.owner() == new_owner.address

def test_recovery_blocked_before_delay(social_wallet, guardian1, guardian2, chain):
    """Test recovery cannot execute before delay"""
    new_owner = Account.create().address
    
    social_wallet.startRecovery(new_owner, sender=guardian1)
    social_wallet.voteForRecovery(sender=guardian2)
    
    # Try immediately (without waiting)
    with pytest.raises(Exception, match="Delay not passed"):
        social_wallet.executeRecovery(sender=guardian1)

def test_freeze_wallet(social_wallet, guardian1, guardian2):
    """Test wallet can be frozen by guardians"""
    assert social_wallet.frozen() == False
    
    social_wallet.voteToFreeze(sender=guardian1)
    social_wallet.voteToFreeze(sender=guardian2)
    
    assert social_wallet.frozen() == True

def test_add_guardian(social_wallet, user_key, mock_entry_point, accounts):
    """Test adding a new guardian"""
    new_guardian = accounts[9]
    initial_count = len(social_wallet.getGuardians())
    
    # Execute through wallet (owner action)
    call_data = social_wallet.addGuardian.encode_input(new_guardian.address)
    social_wallet.execute(
        social_wallet.address,
        0,
        call_data,
        sender=mock_entry_point
    )
    
    assert social_wallet.is_guardian(new_guardian.address) == True
    assert len(social_wallet.getGuardians()) == initial_count + 1
```

---

## 8. Security Considerations {#s8}

### ข้อควรระวังสำหรับ ERC-4337

**1. EntryPoint Trust**
```python
# @version 0.4.0
# EntryPointSecurity.vy
# Only trust the canonical EntryPoint

ENTRY_POINT: immutable(address)

# Official ERC-4337 EntryPoint (MUST verify this matches deployment)
# Mainnet: 0x5FF137D4b0FDCD49DcA30c7CF57E578a026d2789

@deploy
def __init__(entry_point: address):
    # Always verify entry_point address before deployment!
    assert entry_point != empty(address), "Invalid EntryPoint"
    ENTRY_POINT = entry_point

@internal
def _require_from_entry_point():
    """CRITICAL: Only EntryPoint can call certain functions"""
    assert msg.sender == ENTRY_POINT, "account: not from EntryPoint"
```

**2. Nonce Validation**
```python
# ERC-4337 uses a 2D nonce system
# High 192 bits = "key" (allows parallel operations)
# Low 64 bits = sequential nonce within that key
# EntryPoint manages this, don't implement your own nonces
```

**3. Recovery Attack Vectors**
```python
# @version 0.4.0
# RecoverySecurity.vy
# Guard against guardian collusion and social engineering

# Time-lock delays recovery even if all guardians agree
RECOVERY_DELAY: constant(uint256) = 86400 * 7  # 7 days

# Allow owner to cancel recovery at any time
owner_can_cancel_until: public(uint256)

@external
def startRecovery(new_owner: address):
    # Set cancellation window
    self.owner_can_cancel_until = block.timestamp + RECOVERY_DELAY
    # ...

@external
def cancelRecovery():
    # Owner can cancel UNTIL execution time
    assert msg.sender == self.owner, "Not owner"
    assert block.timestamp < self.owner_can_cancel_until, "Too late"
    # Reset recovery state
```

### สรุปแนวทางความปลอดภัย

| จุดอ่อน | ผลกระทบ | การป้องกัน |
|---|---|---|
| Fake EntryPoint | Full control | Immutable EntryPoint |
| Guardian collusion | Steal wallet | Time-lock recovery |
| Replay UserOps | Double execution | EntryPoint nonces |
| Paymaster drain | Gas theft | Spending limits |
| Signature reuse | Unauthorized calls | Include nonce in sig |

---

## สรุป

ERC-4337 Account Abstraction เปลี่ยน model ของ Ethereum wallets:
- **UserOperation**: แทน transaction ธรรมดา
- **EntryPoint**: Singleton ที่ validate และ execute
- **SimpleAccount**: Wallet พื้นฐาน
- **Paymaster**: จ่าย gas แทนผู้ใช้ (ETH หรือ ERC-20)
- **Social Recovery**: Guardian-based key recovery

---
[← Previous Part](part_067_meta_transactions.md) | [→ Next Part](part_069_layer2_integration.md)
