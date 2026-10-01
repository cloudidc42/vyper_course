# Part 012: Events และ Logging

## สารบัญ
1. [Event Declaration](#declaration)
2. [indexed Parameters](#indexed)
3. [log Statement](#log-statement)
4. [Event ใน Testing](#testing)
5. [Event Best Practices](#best-practices)
6. [Event vs Storage](#event-vs-storage)
7. [ตัวอย่าง: Audit Trail Contract](#example)

---

## 1. Event Declaration {#declaration}

Events คือ Log ที่เก็บบน Blockchain ใช้สำหรับ off-chain monitoring

```python
# @version 0.4.0

# ════════════════════════════════════════
# Event Declaration
# ════════════════════════════════════════

# Simple event (no parameters)
event ContractPaused:
    pass

# Event with parameters
event Transfer:
    sender: address
    recipient: address
    amount: uint256

# Event with indexed parameters (สำหรับ filtering)
event Deposit:
    user: indexed(address)
    token: indexed(address)
    amount: uint256
    timestamp: uint256

# Event with all types
event FullEvent:
    addr: indexed(address)
    amount: uint256
    label: String[50]
    data: bytes32
    flag: bool
    timestamp: uint256

# Multiple events
event Minted:
    to: indexed(address)
    token_id: indexed(uint256)

event Burned:
    from_addr: indexed(address)
    token_id: indexed(uint256)

event OwnershipTransferred:
    previous_owner: indexed(address)
    new_owner: indexed(address)
```

### Event ที่มีหลาย Parameters

```python
# @version 0.4.0

# ERC-20 Standard Events
event Transfer:
    sender: indexed(address)
    recipient: indexed(address)
    value: uint256

event Approval:
    owner: indexed(address)
    spender: indexed(address)
    value: uint256

# ERC-721 Standard Events
event NFTTransfer:
    from_addr: indexed(address)
    to: indexed(address)
    token_id: indexed(uint256)

event NFTApproval:
    owner: indexed(address)
    approved: indexed(address)
    token_id: indexed(uint256)

event ApprovalForAll:
    owner: indexed(address)
    operator: indexed(address)
    approved: bool

# DeFi Events
event Swap:
    sender: indexed(address)
    amount_in: uint256
    amount_out: uint256
    token_in: indexed(address)
    token_out: indexed(address)

event LiquidityAdded:
    provider: indexed(address)
    token0_amount: uint256
    token1_amount: uint256
    lp_minted: uint256

event PriceUpdated:
    oracle: indexed(address)
    price: uint256
    timestamp: uint256
```

---

## 2. indexed Parameters {#indexed}

`indexed` ทำให้ filter event ได้จาก on-chain data (สูงสุด 3 indexed parameters)

```python
# @version 0.4.0

# ════════════════════════════════════════
# indexed Parameters
# ════════════════════════════════════════

# indexed ทำให้ parameter กลายเป็น topic ใน log
# สามารถ filter โดย frontend/backend ได้

event OrderPlaced:
    order_id: indexed(uint256)    # topic[1]
    buyer: indexed(address)       # topic[2]
    seller: indexed(address)      # topic[3]
    amount: uint256               # data (ไม่ indexed)
    price: uint256                # data
    timestamp: uint256            # data

# ข้อจำกัด: indexed ได้สูงสุด 3 parameters
# event signature เป็น topic[0] เสมอ

# ❌ ไม่ได้: เกิน 3 indexed
# event BadEvent:
#     a: indexed(address)
#     b: indexed(address)
#     c: indexed(address)
#     d: indexed(address)  # Error! Max 3 indexed

# ✅ String/Bytes ที่ indexed จะ hash เป็น bytes32
event DocumentSigned:
    signer: indexed(address)
    doc_hash: indexed(bytes32)   # bytes32 indexed ตรงๆ
    doc_name: String[100]        # String ไม่ indexed (เก็บ data)

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# เมื่อใด indexed? เมื่อใด ไม่?
# ━━━━━━━━━━━━━━━━━━━━━━━━━

# ✅ indexed: addresses, IDs ที่ query บ่อย
# ✅ indexed: ต้องการ filter โดย specific value
# ❌ ไม่ indexed: amounts, timestamps (มักใช้ range query)
# ❌ ไม่ indexed: strings (ใหญ่เกินไป)
# ❌ ไม่ indexed: calculated values
```

---

## 3. log Statement {#log-statement}

```python
# @version 0.4.0

# ════════════════════════════════════════
# log Statement
# ════════════════════════════════════════

event Transfer:
    sender: indexed(address)
    recipient: indexed(address)
    amount: uint256

event Paused:
    by: indexed(address)
    timestamp: uint256

owner: address
balances: HashMap[address, uint256]
paused: bool

@deploy
def __init__(initial_supply: uint256):
    self.owner = msg.sender
    self.balances[msg.sender] = initial_supply
    # Emit event on deployment
    log Transfer(empty(address), msg.sender, initial_supply)

@external
def transfer(to: address, amount: uint256) -> bool:
    assert self.balances[msg.sender] >= amount, "Insufficient"
    assert to != empty(address), "Zero address"
    self.balances[msg.sender] -= amount
    self.balances[to] += amount
    # log EventName(arg1, arg2, ...)
    log Transfer(msg.sender, to, amount)
    return True

@external
def pause():
    assert msg.sender == self.owner, "Not owner"
    self.paused = True
    log Paused(msg.sender, block.timestamp)

# Multiple events in one function
event Minted:
    to: indexed(address)
    amount: uint256

event SupplyIncreased:
    by: indexed(address)
    old_supply: uint256
    new_supply: uint256

total_supply: uint256

@external
def mint(to: address, amount: uint256):
    assert msg.sender == self.owner, "Not owner"
    old: uint256 = self.total_supply
    self.total_supply += amount
    self.balances[to] += amount
    # Emit multiple events
    log Transfer(empty(address), to, amount)
    log Minted(to, amount)
    log SupplyIncreased(msg.sender, old, self.total_supply)
```

### Conditional Logging

```python
# @version 0.4.0

event TierChanged:
    user: indexed(address)
    old_tier: uint8
    new_tier: uint8

event MilestoneReached:
    user: indexed(address)
    milestone: uint256

tiers: HashMap[address, uint8]
points: HashMap[address, uint256]

@deploy
def __init__():
    pass

@internal
def _check_tier_upgrade(user: address) -> bool:
    old_tier: uint8 = self.tiers[user]
    new_tier: uint8 = old_tier
    p: uint256 = self.points[user]

    if p >= 10000:
        new_tier = 4
    elif p >= 5000:
        new_tier = 3
    elif p >= 1000:
        new_tier = 2
    elif p >= 100:
        new_tier = 1

    if new_tier != old_tier:
        self.tiers[user] = new_tier
        log TierChanged(user, old_tier, new_tier)
        return True
    return False

@external
def add_points(user: address, amount: uint256):
    self.points[user] += amount

    # Milestone events
    if self.points[user] % 1000 == 0:
        log MilestoneReached(user, self.points[user])

    self._check_tier_upgrade(user)
```

---

## 4. Event ใน Testing {#testing}

```python
# tests/test_events.py
# Using pytest + brownie

import pytest
from brownie import EventDemo, accounts

@pytest.fixture
def contract(accounts):
    return EventDemo.deploy(1000000, {'from': accounts[0]})

class TestTransferEvents:
    def test_transfer_emits_event(self, contract, accounts):
        tx = contract.transfer(accounts[1], 100, {'from': accounts[0]})

        # Check event was emitted
        assert len(tx.events) == 1
        assert 'Transfer' in tx.events

        # Check event parameters
        event = tx.events['Transfer']
        assert event['sender'] == accounts[0]
        assert event['recipient'] == accounts[1]
        assert event['amount'] == 100

    def test_multiple_events(self, contract, accounts):
        tx = contract.mint(accounts[1], 500, {'from': accounts[0]})

        # Multiple events emitted
        assert 'Transfer' in tx.events
        assert 'Minted' in tx.events
        assert 'SupplyIncreased' in tx.events

    def test_event_filtering(self, contract, accounts):
        # Make several transfers
        contract.transfer(accounts[1], 100, {'from': accounts[0]})
        contract.transfer(accounts[2], 200, {'from': accounts[0]})

        # Filter by indexed parameter (using brownie's event filter)
        # In practice you'd use web3.py or ethers.js
        # contract.events.Transfer.filter(sender=accounts[0])

# Using web3.py for event filtering
"""
from web3 import Web3

w3 = Web3(Web3.HTTPProvider("http://localhost:8545"))
contract = w3.eth.contract(address=contract_address, abi=abi)

# Filter Transfer events
transfer_filter = contract.events.Transfer.create_filter(
    fromBlock=0,
    argument_filters={"sender": accounts[0]}
)
events = transfer_filter.get_all_entries()

# Latest events
latest = contract.events.Transfer.create_filter(
    fromBlock='latest'
)
"""
```

### Test ใน Hardhat/Foundry Style

```python
# @version 0.4.0

# Contract for testing event patterns
event ValueSet:
    setter: indexed(address)
    old_value: uint256
    new_value: uint256

event Initialized:
    owner: indexed(address)
    initial_value: uint256

value: uint256
owner: address

@deploy
def __init__(initial: uint256):
    self.owner = msg.sender
    self.value = initial
    log Initialized(msg.sender, initial)

@external
def set_value(new_val: uint256):
    old: uint256 = self.value
    self.value = new_val
    log ValueSet(msg.sender, old, new_val)
```

```javascript
// Hardhat test example
const { expect } = require("chai");
const { ethers } = require("hardhat");

describe("EventDemo", function() {
    it("Should emit ValueSet event", async function() {
        const [owner] = await ethers.getSigners();
        const Contract = await ethers.getContractFactory("EventDemo");
        const c = await Contract.deploy(100);

        await expect(c.set_value(200))
            .to.emit(c, "ValueSet")
            .withArgs(owner.address, 100, 200);
    });

    it("Should filter events by indexed param", async function() {
        const [owner, user1] = await ethers.getSigners();
        const Contract = await ethers.getContractFactory("EventDemo");
        const c = await Contract.deploy(100);

        await c.set_value(200);
        await c.connect(user1).set_value(300);

        // Filter by setter = owner
        const filter = c.filters.ValueSet(owner.address);
        const events = await c.queryFilter(filter);
        expect(events.length).to.equal(1);
        expect(events[0].args.new_value).to.equal(200);
    });
});
```

---

## 5. Event Best Practices {#best-practices}

```python
# @version 0.4.0

# ════════════════════════════════════════
# Event Best Practices
# ════════════════════════════════════════

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# ✅ Good: State-change events
# ━━━━━━━━━━━━━━━━━━━━━━━━━

event FundsDeposited:
    user: indexed(address)
    amount: uint256
    total_balance: uint256  # Include new state for easy tracking

event FundsWithdrawn:
    user: indexed(address)
    amount: uint256
    remaining: uint256

event OwnershipTransferred:
    previous_owner: indexed(address)
    new_owner: indexed(address)

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# ✅ Good: Include context
# ━━━━━━━━━━━━━━━━━━━━━━━━━

event TradeExecuted:
    trader: indexed(address)
    pair: indexed(bytes32)
    buy_amount: uint256
    sell_amount: uint256
    price: uint256
    fee: uint256
    block_number: uint256  # For ordering

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# ❌ Bad: Too little info
# ━━━━━━━━━━━━━━━━━━━━━━━━━
event BadEvent:
    amount: uint256  # ไม่รู้ว่าใคร ทำอะไร

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# ❌ Bad: Too much info (expensive gas)
# ━━━━━━━━━━━━━━━━━━━━━━━━━
# event ExpensiveEvent:
#     full_order: OrderStruct  # อย่าใส่ struct ขนาดใหญ่ใน event

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# ✅ Good: Error events (instead of just revert)
# ━━━━━━━━━━━━━━━━━━━━━━━━━
event OperationFailed:
    user: indexed(address)
    operation: String[30]
    reason: String[100]

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# ✅ Good: Admin events
# ━━━━━━━━━━━━━━━━━━━━━━━━━
event ParameterUpdated:
    param_name: indexed(bytes32)  # keccak256 of param name
    old_value: uint256
    new_value: uint256
    updated_by: indexed(address)
```

### Gas Cost ของ Events

```python
# @version 0.4.0

# Gas costs (approximate):
# Base log: 375 gas
# Per topic (indexed): 375 gas
# Per 32 bytes of data: 8 gas (non-zero), 32 gas overhead

# Event ที่ถูก (น้อย indexed, น้อย data):
event Cheap:
    amount: uint256  # ~750 gas total

# Event ที่แพงกว่า (3 indexed + data):
event Expensive:
    a: indexed(address)   # +375 gas
    b: indexed(address)   # +375 gas
    c: indexed(uint256)   # +375 gas
    d: uint256            # data
    e: String[100]        # more data bytes

# ✅ Optimization: ใส่ indexed เฉพาะ field ที่ต้อง filter
event OptimizedTransfer:
    from_addr: indexed(address)   # filter by sender
    to: indexed(address)           # filter by recipient
    amount: uint256               # ไม่ต้อง index amounts
```

---

## 6. Event vs Storage {#event-vs-storage}

```python
# @version 0.4.0

# ════════════════════════════════════════
# Events vs Storage: When to Use Which
# ════════════════════════════════════════

# Events:
# ✅ ถูกกว่า (log gas << storage gas)
# ✅ เหมาะสำหรับ historical data (ประวัติ)
# ✅ Frontend/Backend ดึงได้ง่าย
# ❌ Contract ไม่สามารถอ่าน event ของตัวเองได้
# ❌ ไม่เหมาะสำหรับ data ที่ต้องใช้ใน on-chain logic

# Storage:
# ✅ Contract อ่านได้
# ✅ เหมาะสำหรับ data ที่ต้องใช้ใน logic
# ❌ แพงกว่า (SSTORE ~20,000 gas)

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Pattern: Hybrid approach
# ━━━━━━━━━━━━━━━━━━━━━━━━━

balances: HashMap[address, uint256]  # Storage: ต้องใช้ใน transfer logic

event BalanceChanged:                 # Event: ประวัติทุกครั้งที่เปลี่ยน
    user: indexed(address)
    old_balance: uint256
    new_balance: uint256
    reason: String[30]

@external
def add_balance(user: address, amount: uint256, reason: String[30]):
    old: uint256 = self.balances[user]
    self.balances[user] += amount
    # เก็บทั้งใน storage และ emit event
    log BalanceChanged(user, old, self.balances[user], reason)

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# ✅ ใช้ Event (ไม่ต้องการใน logic)
# ━━━━━━━━━━━━━━━━━━━━━━━━━
event UserMessage:
    user: indexed(address)
    message: String[200]  # เก็บแค่ใน event log ไม่ต้อง storage

@external
def post_message(message: String[200]):
    # ไม่ต้อง store message บน chain
    log UserMessage(msg.sender, message)

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# ❌ ไม่ควรทำ: store แล้ว emit ซ้ำ
# ━━━━━━━━━━━━━━━━━━━━━━━━━
messages: DynArray[String[200], 1000]  # แพงมาก!

# ควรใช้ event แทน storage สำหรับ messages
```

---

## 7. ตัวอย่าง: Audit Trail Contract {#example}

Contract ที่บันทึก Audit Log สำหรับ Compliance

```python
# @version 0.4.0

# ════════════════════════════════════════════════════════════
# AuditTrail Contract
# ระบบ Audit Log แบบ On-Chain สำหรับ Compliance
# ════════════════════════════════════════════════════════════

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Action Types (constants)
# ━━━━━━━━━━━━━━━━━━━━━━━━━
ACTION_CREATE: constant(bytes4) = 0x01000000
ACTION_UPDATE: constant(bytes4) = 0x02000000
ACTION_DELETE: constant(bytes4) = 0x03000000
ACTION_TRANSFER: constant(bytes4) = 0x04000000
ACTION_APPROVE: constant(bytes4) = 0x05000000
ACTION_REJECT: constant(bytes4) = 0x06000000
ACTION_LOGIN: constant(bytes4) = 0x07000000
ACTION_ADMIN: constant(bytes4) = 0x08000000

# Severity Levels
SEVERITY_INFO: constant(uint8) = 1
SEVERITY_WARNING: constant(uint8) = 2
SEVERITY_ERROR: constant(uint8) = 3
SEVERITY_CRITICAL: constant(uint8) = 4

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Events (Primary audit mechanism)
# ━━━━━━━━━━━━━━━━━━━━━━━━━

# Main audit event - emitted for every action
event AuditLog:
    actor: indexed(address)           # Who performed the action
    action: indexed(bytes4)           # What action type
    target: indexed(address)          # What was affected (address/contract)
    target_id: uint256                # Resource ID (e.g., order_id)
    data_hash: bytes32                # Hash of additional data
    severity: uint8                   # 1-4
    timestamp: uint256

# Access control events
event AccessGranted:
    grantor: indexed(address)
    grantee: indexed(address)
    role: indexed(bytes32)
    valid_until: uint256

event AccessRevoked:
    revoker: indexed(address)
    revokee: indexed(address)
    role: indexed(bytes32)
    reason: String[200]

# Financial events
event FundsMovement:
    from_addr: indexed(address)
    to: indexed(address)
    amount: uint256
    currency: indexed(bytes32)
    reference: bytes32

# System events
event SystemStateChanged:
    changed_by: indexed(address)
    parameter: indexed(bytes32)
    old_value: bytes32
    new_value: bytes32

event AlertRaised:
    reporter: indexed(address)
    alert_type: indexed(bytes32)
    severity: uint8
    description: String[500]

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Structs
# ━━━━━━━━━━━━━━━━━━━━━━━━━

struct RoleInfo:
    role_hash: bytes32
    granted_by: address
    granted_at: uint256
    valid_until: uint256  # 0 = never expires
    is_active: bool

struct AuditSummary:
    total_actions: uint256
    last_action_at: uint256
    critical_count: uint256

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# State Variables
# ━━━━━━━━━━━━━━━━━━━━━━━━━

owner: address
auditors: HashMap[address, bool]

# Role-based access control
roles: HashMap[address, HashMap[bytes32, RoleInfo]]

# User summaries (minimal on-chain storage)
user_summaries: HashMap[address, AuditSummary]

# Alert threshold
alert_threshold: uint256
suspended_accounts: HashMap[address, bool]
suspension_reason: HashMap[address, String[200]]

# Sequential action counter for global ordering
action_counter: uint256

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Constants for roles
# ━━━━━━━━━━━━━━━━━━━━━━━━━
ROLE_ADMIN: constant(bytes32) = keccak256("ADMIN")
ROLE_OPERATOR: constant(bytes32) = keccak256("OPERATOR")
ROLE_VIEWER: constant(bytes32) = keccak256("VIEWER")
ROLE_AUDITOR: constant(bytes32) = keccak256("AUDITOR")

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Constructor
# ━━━━━━━━━━━━━━━━━━━━━━━━━

@deploy
def __init__(alert_threshold: uint256):
    self.owner = msg.sender
    self.alert_threshold = alert_threshold
    self.auditors[msg.sender] = True

    # Grant owner all roles
    self._grant_role(msg.sender, ROLE_ADMIN, 0)
    self._grant_role(msg.sender, ROLE_AUDITOR, 0)

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Internal Functions
# ━━━━━━━━━━━━━━━━━━━━━━━━━

@internal
def _has_role(user: address, role: bytes32) -> bool:
    info: RoleInfo = self.roles[user][role]
    if not info.is_active:
        return False
    if info.valid_until > 0 and block.timestamp > info.valid_until:
        return False
    return True

@internal
def _grant_role(grantee: address, role: bytes32, valid_until: uint256):
    self.roles[grantee][role] = RoleInfo({
        role_hash: role,
        granted_by: msg.sender,
        granted_at: block.timestamp,
        valid_until: valid_until,
        is_active: True
    })

@internal
def _emit_audit(
    actor: address,
    action: bytes4,
    target: address,
    target_id: uint256,
    data: bytes32,
    severity: uint8
):
    """Core audit logging function"""
    self.action_counter += 1
    self.user_summaries[actor].total_actions += 1
    self.user_summaries[actor].last_action_at = block.timestamp

    if severity >= SEVERITY_CRITICAL:
        self.user_summaries[actor].critical_count += 1

    log AuditLog(actor, action, target, target_id, data, severity, block.timestamp)

    # Auto-alert if too many critical actions
    if self.user_summaries[actor].critical_count >= self.alert_threshold:
        log AlertRaised(
            self,
            keccak256("EXCESSIVE_CRITICAL_ACTIONS"),
            SEVERITY_CRITICAL,
            "Actor exceeded critical action threshold"
        )

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Access Control Functions
# ━━━━━━━━━━━━━━━━━━━━━━━━━

@external
def grant_role(
    grantee: address,
    role: bytes32,
    valid_until: uint256
):
    """Grant role to an address"""
    assert self._has_role(msg.sender, ROLE_ADMIN), "Not admin"
    assert grantee != empty(address), "Invalid address"

    self._grant_role(grantee, role, valid_until)

    self._emit_audit(
        msg.sender,
        ACTION_ADMIN,
        grantee,
        0,
        role,
        SEVERITY_WARNING
    )

    log AccessGranted(msg.sender, grantee, role, valid_until)

@external
def revoke_role(grantee: address, role: bytes32, reason: String[200]):
    """Revoke role from an address"""
    assert self._has_role(msg.sender, ROLE_ADMIN), "Not admin"

    self.roles[grantee][role].is_active = False

    self._emit_audit(
        msg.sender,
        ACTION_ADMIN,
        grantee,
        0,
        role,
        SEVERITY_WARNING
    )

    log AccessRevoked(msg.sender, grantee, role, reason)

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Audit Logging Functions
# ━━━━━━━━━━━━━━━━━━━━━━━━━

@external
def log_action(
    action: bytes4,
    target: address,
    target_id: uint256,
    data: bytes32,
    severity: uint8
):
    """Log an auditable action"""
    assert not self.suspended_accounts[msg.sender], "Account suspended"
    assert severity >= 1 and severity <= 4, "Invalid severity"

    self._emit_audit(msg.sender, action, target, target_id, data, severity)

@external
def log_funds_movement(
    to: address,
    amount: uint256,
    currency: bytes32,
    reference: bytes32
):
    """Log financial movement"""
    assert self._has_role(msg.sender, ROLE_OPERATOR), "Not operator"
    assert not self.suspended_accounts[msg.sender], "Suspended"
    assert to != empty(address), "Invalid recipient"
    assert amount > 0, "Zero amount"

    log FundsMovement(msg.sender, to, amount, currency, reference)

    self._emit_audit(
        msg.sender,
        ACTION_TRANSFER,
        to,
        amount,
        reference,
        SEVERITY_INFO
    )

@external
def raise_alert(
    alert_type: bytes32,
    severity: uint8,
    description: String[500]
):
    """Manually raise an alert"""
    assert self.auditors[msg.sender], "Not auditor"

    log AlertRaised(msg.sender, alert_type, severity, description)

    if severity >= SEVERITY_CRITICAL:
        self._emit_audit(
            msg.sender,
            ACTION_ADMIN,
            self,
            0,
            alert_type,
            SEVERITY_CRITICAL
        )

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Admin Functions
# ━━━━━━━━━━━━━━━━━━━━━━━━━

@external
def suspend_account(account: address, reason: String[200]):
    """Suspend an account"""
    assert self._has_role(msg.sender, ROLE_ADMIN), "Not admin"
    assert account != self.owner, "Cannot suspend owner"

    self.suspended_accounts[account] = True
    self.suspension_reason[account] = reason

    log AlertRaised(
        msg.sender,
        keccak256("ACCOUNT_SUSPENDED"),
        SEVERITY_CRITICAL,
        reason
    )

    self._emit_audit(
        msg.sender,
        ACTION_ADMIN,
        account,
        0,
        keccak256(reason),
        SEVERITY_CRITICAL
    )

@external
def update_system_parameter(
    param_name: bytes32,
    old_value: bytes32,
    new_value: bytes32
):
    """Log system parameter change"""
    assert self._has_role(msg.sender, ROLE_ADMIN), "Not admin"

    log SystemStateChanged(msg.sender, param_name, old_value, new_value)

    self._emit_audit(
        msg.sender,
        ACTION_UPDATE,
        self,
        0,
        param_name,
        SEVERITY_WARNING
    )

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# View Functions
# ━━━━━━━━━━━━━━━━━━━━━━━━━

@view
@external
def get_user_summary(user: address) -> AuditSummary:
    return self.user_summaries[user]

@view
@external
def has_role(user: address, role: bytes32) -> bool:
    return self._has_role(user, role)

@view
@external
def is_suspended(account: address) -> bool:
    return self.suspended_accounts[account]

@view
@external
def get_suspension_reason(account: address) -> String[200]:
    return self.suspension_reason[account]

@view
@external
def get_total_actions() -> uint256:
    return self.action_counter
```

### Test Code

```python
# tests/test_audit_trail.py
import pytest
from brownie import AuditTrail, accounts

ROLE_ADMIN = bytes.fromhex("df8b4c520ffe197c5343c6f5aec59570151ef9a492f2c624fd45ddde6135ec42")

@pytest.fixture
def audit(accounts):
    return AuditTrail.deploy(5, {'from': accounts[0]})  # alert after 5 critical

class TestAuditLogging:
    def test_log_action_emits_event(self, audit, accounts):
        tx = audit.log_action(
            "0x01000000",  # ACTION_CREATE
            accounts[1],
            42,
            bytes(32),
            1,  # INFO
            {'from': accounts[0]}
        )
        assert 'AuditLog' in tx.events
        event = tx.events['AuditLog']
        assert event['actor'] == accounts[0]
        assert event['severity'] == 1

    def test_action_counter_increments(self, audit, accounts):
        audit.log_action("0x01000000", accounts[1], 1, bytes(32), 1,
                         {'from': accounts[0]})
        audit.log_action("0x01000000", accounts[1], 2, bytes(32), 1,
                         {'from': accounts[0]})
        assert audit.get_total_actions() == 2

    def test_suspended_account_cannot_log(self, audit, accounts):
        audit.suspend_account(accounts[1], "Test suspension",
                              {'from': accounts[0]})
        with pytest.raises(Exception):
            audit.log_action("0x01000000", accounts[2], 1, bytes(32), 1,
                             {'from': accounts[1]})

class TestRoleAccess:
    def test_grant_role(self, audit, accounts):
        OPERATOR_ROLE = bytes.fromhex("523a704056dcd17bcf83bed8b68c59416dac1119be77755efe3bde0a64e46e0c")
        audit.grant_role(accounts[1], OPERATOR_ROLE, 0, {'from': accounts[0]})
        assert audit.has_role(accounts[1], OPERATOR_ROLE) == True

    def test_grant_emits_access_event(self, audit, accounts):
        OPERATOR_ROLE = bytes.fromhex("523a704056dcd17bcf83bed8b68c59416dac1119be77755efe3bde0a64e46e0c")
        tx = audit.grant_role(accounts[1], OPERATOR_ROLE, 0,
                              {'from': accounts[0]})
        assert 'AccessGranted' in tx.events
```

---

## สรุป

- ✅ **Event declaration**: `event EventName: field: type`
- ✅ **indexed parameters**: สูงสุด 3 ทำให้ filter ได้
- ✅ **log statement**: `log EventName(arg1, arg2, ...)`
- ✅ **Events ถูกกว่า storage**: ใช้สำหรับ historical data
- ✅ **Event best practices**: ใส่ context เพียงพอ, indexed field ที่จะ filter
- ✅ **Hybrid approach**: Storage สำหรับ on-chain logic, Events สำหรับ history

## แบบฝึกหัด

1. **สร้าง** Event Aggregator ที่รวม events จากหลาย contracts
2. **เพิ่ม** Event ที่มี signature แบบ EIP-712 สำหรับ off-chain verification
3. **สร้าง** Dashboard ที่ดึง events ด้วย ethers.js
4. **ทดสอบ** Gas cost ระหว่าง event vs storage สำหรับ 100 records
5. **สร้าง** Compliance Report จาก event history

---

**ก่อนหน้า: [Part 011 - Structs](part_011_structs.md)**  
**ต่อไป: [Part 013 - Constructor และ Initialization](part_013_constructor.md)**
