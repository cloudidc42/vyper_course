# Part 030: Timelock

## สารบัญ
1. [บทนำ Timelock](#บทนำ)
2. [Time Delays และ block.timestamp](#time-delays-และ-blocktimestamp)
3. [Timelock Patterns](#timelock-patterns)
4. [Governance Timelock](#governance-timelock)
5. [ตัวอย่าง: Timelock Controller](#ตัวอย่าง-timelock-controller)
6. [Test Code](#test-code)

---

## บทนำ

**Timelock** เป็น Pattern ที่บังคับให้มีการ delay ก่อนดำเนินการสำคัญ เพื่อให้ผู้ใช้มีเวลาตรวจสอบและตัดสินใจก่อนที่การเปลี่ยนแปลงจะมีผล

### ทำไมต้องใช้ Timelock?
- ป้องกัน admin abuse
- ให้เวลาผู้ใช้ถอน funds ถ้าไม่เห็นด้วย
- สร้างความโปร่งใส
- มาตรฐานใน DeFi protocols

### ใช้งานจริงที่ไหนบ้าง?
- Compound Finance (2 วัน)
- Uniswap (7 วัน)
- Aave (24 ชั่วโมง - 7 วัน)
- MakerDAO (variable)

---

## Time Delays และ block.timestamp

### ความเข้าใจ block.timestamp

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0

# block.timestamp = Unix timestamp (วินาที)
# ค่า approximate เพราะ miner สามารถปรับเล็กน้อยได้

# Time constants
ONE_MINUTE: constant(uint256) = 60
ONE_HOUR: constant(uint256) = 3600
ONE_DAY: constant(uint256) = 86400
ONE_WEEK: constant(uint256) = 604800
ONE_MONTH: constant(uint256) = 2592000  # 30 วัน
ONE_YEAR: constant(uint256) = 31536000  # 365 วัน

@view
@external
def get_current_time() -> uint256:
    return block.timestamp

@view
@external
def get_time_until(future_timestamp: uint256) -> uint256:
    if future_timestamp > block.timestamp:
        return future_timestamp - block.timestamp
    return 0

@view
@external
def is_past(timestamp: uint256) -> bool:
    return block.timestamp >= timestamp
```

### Timelock พื้นฐาน

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Basic Timelock

event ActionQueued:
    action_id: indexed(bytes32)
    eta: uint256

event ActionExecuted:
    action_id: indexed(bytes32)

event ActionCancelled:
    action_id: indexed(bytes32)

MINIMUM_DELAY: constant(uint256) = 86400      # 1 วัน
MAXIMUM_DELAY: constant(uint256) = 2592000    # 30 วัน

owner: public(address)
delay: public(uint256)

# action_id -> eta (earliest time to execute)
queued_actions: public(HashMap[bytes32, uint256])

@deploy
def __init__(_delay: uint256):
    assert _delay >= MINIMUM_DELAY, "Delay too short"
    assert _delay <= MAXIMUM_DELAY, "Delay too long"
    self.owner = msg.sender
    self.delay = _delay

@internal
def _only_owner():
    assert msg.sender == self.owner, "Not owner"

@external
def queue_action(action_id: bytes32) -> uint256:
    """
    @notice Queue action พร้อม delay
    @return eta เวลาที่สามารถ execute ได้
    """
    self._only_owner()
    
    assert self.queued_actions[action_id] == 0, "Action already queued"
    
    eta: uint256 = block.timestamp + self.delay
    self.queued_actions[action_id] = eta
    
    log ActionQueued(action_id, eta)
    return eta

@external
def execute_action(action_id: bytes32):
    """
    @notice Execute action หลังจาก delay ผ่านไป
    """
    self._only_owner()
    
    eta: uint256 = self.queued_actions[action_id]
    assert eta != 0, "Action not queued"
    assert block.timestamp >= eta, "Timelock not expired"
    
    # Grace period: 14 วัน หลัง eta
    assert block.timestamp <= eta + 14 * 86400, "Action expired"
    
    # Clear queue
    self.queued_actions[action_id] = 0
    
    log ActionExecuted(action_id)

@external
def cancel_action(action_id: bytes32):
    """
    @notice ยกเลิก action ที่ queue ไว้
    """
    self._only_owner()
    
    assert self.queued_actions[action_id] != 0, "Action not queued"
    
    self.queued_actions[action_id] = 0
    
    log ActionCancelled(action_id)
```

---

## Timelock Patterns

### Transaction Queue with Parameters

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Timelock with Transaction Queue

struct Transaction:
    target: address       # Contract ที่จะ call
    value: uint256        # ETH amount
    data: Bytes[1024]     # Calldata
    eta: uint256          # Earliest execution time
    executed: bool        # ถูก execute แล้วหรือไม่

event TransactionQueued:
    tx_hash: indexed(bytes32)
    target: address
    value: uint256
    eta: uint256

event TransactionExecuted:
    tx_hash: indexed(bytes32)

event TransactionCancelled:
    tx_hash: indexed(bytes32)

MINIMUM_DELAY: constant(uint256) = 172800   # 2 วัน
MAXIMUM_DELAY: constant(uint256) = 2592000  # 30 วัน
GRACE_PERIOD: constant(uint256) = 1209600   # 14 วัน

admin: public(address)
pending_admin: public(address)
delay: public(uint256)

transactions: HashMap[bytes32, Transaction]

@deploy
def __init__(_admin: address, _delay: uint256):
    assert _delay >= MINIMUM_DELAY
    assert _delay <= MAXIMUM_DELAY
    self.admin = _admin
    self.delay = _delay

@internal
def _only_timelock():
    assert msg.sender == self, "Timelock: not timelock"

@internal
def _only_admin():
    assert msg.sender == self.admin, "Timelock: not admin"

@internal
def _get_tx_hash(
    target: address,
    value: uint256,
    data: Bytes[1024],
    eta: uint256
) -> bytes32:
    return keccak256(
        concat(
            convert(target, bytes32),
            convert(value, bytes32),
            data,
            convert(eta, bytes32)
        )
    )

@external
def queue_transaction(
    target: address,
    value: uint256,
    data: Bytes[1024],
    eta: uint256
) -> bytes32:
    """
    @notice Queue transaction
    @param eta ต้องมากกว่า block.timestamp + delay
    """
    self._only_admin()
    assert eta >= block.timestamp + self.delay, "ETA too soon"
    
    tx_hash: bytes32 = self._get_tx_hash(target, value, data, eta)
    assert not self.transactions[tx_hash].executed, "Already queued"
    
    self.transactions[tx_hash] = Transaction({
        target: target,
        value: value,
        data: data,
        eta: eta,
        executed: False
    })
    
    log TransactionQueued(tx_hash, target, value, eta)
    return tx_hash

@external
def execute_transaction(
    target: address,
    value: uint256,
    data: Bytes[1024],
    eta: uint256
):
    """
    @notice Execute queued transaction
    """
    self._only_admin()
    
    tx_hash: bytes32 = self._get_tx_hash(target, value, data, eta)
    tx: Transaction = self.transactions[tx_hash]
    
    assert tx.eta > 0, "Transaction not queued"
    assert not tx.executed, "Already executed"
    assert block.timestamp >= tx.eta, "Timelock not expired"
    assert block.timestamp <= tx.eta + GRACE_PERIOD, "Grace period expired"
    
    self.transactions[tx_hash].executed = True
    
    # Execute the transaction
    raw_call(target, data, value=value)
    
    log TransactionExecuted(tx_hash)

@external
def cancel_transaction(
    target: address,
    value: uint256,
    data: Bytes[1024],
    eta: uint256
):
    """
    @notice ยกเลิก transaction
    """
    self._only_admin()
    
    tx_hash: bytes32 = self._get_tx_hash(target, value, data, eta)
    assert self.transactions[tx_hash].eta > 0, "Not queued"
    
    self.transactions[tx_hash] = Transaction({
        target: empty(address),
        value: 0,
        data: b"",
        eta: 0,
        executed: False
    })
    
    log TransactionCancelled(tx_hash)

@view
@external
def get_transaction(tx_hash: bytes32) -> Transaction:
    return self.transactions[tx_hash]

@view
@external
def is_queued(tx_hash: bytes32) -> bool:
    return self.transactions[tx_hash].eta > 0

@view
@external
def can_execute(tx_hash: bytes32) -> bool:
    tx: Transaction = self.transactions[tx_hash]
    if tx.eta == 0 or tx.executed:
        return False
    return (block.timestamp >= tx.eta and 
            block.timestamp <= tx.eta + GRACE_PERIOD)
```

---

## Governance Timelock

### Timelock สำหรับ DAO Governance

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Governance Timelock
# @notice Timelock Controller สำหรับ DAO

event CallScheduled:
    id: indexed(bytes32)
    index: uint256
    target: address
    value: uint256
    predecessor: bytes32
    delay: uint256

event CallExecuted:
    id: indexed(bytes32)
    index: uint256
    target: address
    value: uint256

event Cancelled:
    id: indexed(bytes32)

event MinDelayChange:
    old_duration: uint256
    new_duration: uint256

# Role constants
TIMELOCK_ADMIN_ROLE: constant(bytes32) = keccak256("TIMELOCK_ADMIN_ROLE")
PROPOSER_ROLE: constant(bytes32) = keccak256("PROPOSER_ROLE")
EXECUTOR_ROLE: constant(bytes32) = keccak256("EXECUTOR_ROLE")
CANCELLER_ROLE: constant(bytes32) = keccak256("CANCELLER_ROLE")

# Operation states
_UNSET: constant(uint8) = 0
_WAITING: constant(uint8) = 1
_READY: constant(uint8) = 2
_DONE: constant(uint8) = 3

# State
min_delay: public(uint256)
roles: HashMap[bytes32, HashMap[address, bool]]
timestamps: HashMap[bytes32, uint256]  # operation_id -> ready timestamp

@deploy
def __init__(
    _min_delay: uint256,
    proposers: DynArray[address, 10],
    executors: DynArray[address, 10],
    admin: address
):
    """
    @param _min_delay ระยะเวลาขั้นต่ำก่อน execute
    @param proposers รายชื่อผู้มีสิทธิ์เสนอ
    @param executors รายชื่อผู้มีสิทธิ์ execute
    @param admin admin address (ควรเป็น address(0) หลัง setup)
    """
    self.min_delay = _min_delay
    log MinDelayChange(0, _min_delay)
    
    # Setup roles
    self.roles[TIMELOCK_ADMIN_ROLE][self] = True
    self.roles[TIMELOCK_ADMIN_ROLE][msg.sender] = True
    
    if admin != empty(address):
        self.roles[TIMELOCK_ADMIN_ROLE][admin] = True
    
    for proposer: address in proposers:
        self.roles[PROPOSER_ROLE][proposer] = True
        self.roles[CANCELLER_ROLE][proposer] = True
    
    for executor: address in executors:
        self.roles[EXECUTOR_ROLE][executor] = True

@internal
def _check_role(role: bytes32):
    assert self.roles[role][msg.sender], "AccessControl: missing role"

@internal
def _get_operation_id(
    target: address,
    value: uint256,
    data: Bytes[1024],
    predecessor: bytes32,
    salt: bytes32
) -> bytes32:
    return keccak256(
        concat(
            convert(target, bytes32),
            convert(value, bytes32),
            data,
            predecessor,
            salt
        )
    )

@internal
def _get_operation_state(id: bytes32) -> uint8:
    timestamp: uint256 = self.timestamps[id]
    if timestamp == 0:
        return _UNSET
    elif timestamp == 1:
        return _DONE
    elif timestamp > block.timestamp:
        return _WAITING
    else:
        return _READY

@external
def schedule(
    target: address,
    value: uint256,
    data: Bytes[1024],
    predecessor: bytes32,
    salt: bytes32,
    delay: uint256
) -> bytes32:
    """
    @notice Schedule operation
    @param predecessor operation ก่อนหน้าที่ต้อง execute ก่อน
    @param salt random value เพื่อสร้าง unique ID
    @param delay ระยะเวลา (ต้องมากกว่า min_delay)
    """
    self._check_role(PROPOSER_ROLE)
    assert delay >= self.min_delay, "Timelock: insufficient delay"
    
    id: bytes32 = self._get_operation_id(target, value, data, predecessor, salt)
    assert self._get_operation_state(id) == _UNSET, "Operation already scheduled"
    
    eta: uint256 = block.timestamp + delay
    self.timestamps[id] = eta
    
    log CallScheduled(id, 0, target, value, predecessor, delay)
    return id

@external
def execute(
    target: address,
    value: uint256,
    data: Bytes[1024],
    predecessor: bytes32,
    salt: bytes32
):
    """
    @notice Execute ready operation
    """
    self._check_role(EXECUTOR_ROLE)
    
    id: bytes32 = self._get_operation_id(target, value, data, predecessor, salt)
    
    # ตรวจสอบ predecessor ถ้ามี
    if predecessor != empty(bytes32):
        assert self._get_operation_state(predecessor) == _DONE, \
            "Predecessor not done"
    
    assert self._get_operation_state(id) == _READY, "Operation not ready"
    
    self.timestamps[id] = 1  # DONE state
    
    raw_call(target, data, value=value)
    
    log CallExecuted(id, 0, target, value)

@external
def cancel(id: bytes32):
    """
    @notice ยกเลิก operation ที่ pending
    """
    self._check_role(CANCELLER_ROLE)
    
    state: uint8 = self._get_operation_state(id)
    assert state == _WAITING or state == _READY, "Cannot cancel"
    
    self.timestamps[id] = 0
    
    log Cancelled(id)

@external
def update_delay(new_delay: uint256):
    """
    @notice อัปเดต min delay (ต้อง execute ผ่าน timelock เอง)
    """
    assert msg.sender == self, "Timelock: not timelock"
    
    log MinDelayChange(self.min_delay, new_delay)
    self.min_delay = new_delay

@external
def grant_role(role: bytes32, account: address):
    assert self.roles[TIMELOCK_ADMIN_ROLE][msg.sender], "Not admin"
    self.roles[role][account] = True

@external
def revoke_role(role: bytes32, account: address):
    assert self.roles[TIMELOCK_ADMIN_ROLE][msg.sender], "Not admin"
    self.roles[role][account] = False

@view
@external
def get_operation_state(id: bytes32) -> uint8:
    return self._get_operation_state(id)

@view
@external
def is_operation_ready(id: bytes32) -> bool:
    return self._get_operation_state(id) == _READY

@view
@external
def is_operation_done(id: bytes32) -> bool:
    return self._get_operation_state(id) == _DONE

@view
@external
def has_role(role: bytes32, account: address) -> bool:
    return self.roles[role][account]

@view
@external
def get_timestamp(id: bytes32) -> uint256:
    return self.timestamps[id]
```

---

## ตัวอย่าง: Timelock Controller

### ระบบ Timelock สมบูรณ์พร้อมการใช้งาน

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Complete Timelock Controller
# @notice ระบบ Timelock ที่ใช้งานได้จริงพร้อม multi-sig support

struct PendingAction:
    action_hash: bytes32
    target: address
    calldata: Bytes[1024]
    value: uint256
    eta: uint256
    proposer: address
    approvals: uint256
    executed: bool
    cancelled: bool

event ActionProposed:
    action_id: indexed(uint256)
    target: indexed(address)
    eta: uint256
    proposer: indexed(address)

event ActionApproved:
    action_id: indexed(uint256)
    approver: indexed(address)

event ActionExecuted:
    action_id: indexed(uint256)
    executor: indexed(address)

event ActionCancelled:
    action_id: indexed(uint256)

event DelayUpdated:
    old_delay: uint256
    new_delay: uint256

# Configuration
MIN_DELAY: constant(uint256) = 86400      # 1 วัน
MAX_DELAY: constant(uint256) = 2592000    # 30 วัน
GRACE_PERIOD: constant(uint256) = 1209600 # 14 วัน

# State
owner: public(address)
delay: public(uint256)
required_approvals: public(uint256)

proposers: public(HashMap[address, bool])
approvers: public(HashMap[address, bool])
executors: public(HashMap[address, bool])

action_count: public(uint256)
actions: public(HashMap[uint256, PendingAction])
action_approvals: HashMap[uint256, HashMap[address, bool]]

@deploy
def __init__(
    _delay: uint256,
    _required_approvals: uint256
):
    assert _delay >= MIN_DELAY and _delay <= MAX_DELAY
    assert _required_approvals > 0
    
    self.owner = msg.sender
    self.delay = _delay
    self.required_approvals = _required_approvals
    
    # Owner มีทุก role
    self.proposers[msg.sender] = True
    self.approvers[msg.sender] = True
    self.executors[msg.sender] = True

@internal
def _only_owner():
    assert msg.sender == self.owner, "Not owner"

@external
def propose_action(
    target: address,
    calldata: Bytes[1024],
    value: uint256
) -> uint256:
    """
    @notice เสนอ action พร้อม delay
    @return action_id
    """
    assert self.proposers[msg.sender], "Not proposer"
    
    eta: uint256 = block.timestamp + self.delay
    action_id: uint256 = self.action_count
    
    action_hash: bytes32 = keccak256(
        concat(
            convert(target, bytes32),
            convert(value, bytes32),
            calldata,
            convert(eta, bytes32)
        )
    )
    
    self.actions[action_id] = PendingAction({
        action_hash: action_hash,
        target: target,
        calldata: calldata,
        value: value,
        eta: eta,
        proposer: msg.sender,
        approvals: 0,
        executed: False,
        cancelled: False
    })
    
    self.action_count += 1
    
    log ActionProposed(action_id, target, eta, msg.sender)
    return action_id

@external
def approve_action(action_id: uint256):
    """
    @notice อนุมัติ action
    """
    assert self.approvers[msg.sender], "Not approver"
    
    action: PendingAction = self.actions[action_id]
    assert action.eta > 0, "Action not found"
    assert not action.executed, "Already executed"
    assert not action.cancelled, "Cancelled"
    assert not self.action_approvals[action_id][msg.sender], "Already approved"
    
    self.action_approvals[action_id][msg.sender] = True
    self.actions[action_id].approvals += 1
    
    log ActionApproved(action_id, msg.sender)

@external
def execute_action(action_id: uint256):
    """
    @notice Execute action หลัง delay และได้รับ approval เพียงพอ
    """
    assert self.executors[msg.sender], "Not executor"
    
    action: PendingAction = self.actions[action_id]
    assert action.eta > 0, "Action not found"
    assert not action.executed, "Already executed"
    assert not action.cancelled, "Cancelled"
    assert block.timestamp >= action.eta, "Timelock not expired"
    assert block.timestamp <= action.eta + GRACE_PERIOD, "Grace period expired"
    assert action.approvals >= self.required_approvals, "Insufficient approvals"
    
    self.actions[action_id].executed = True
    
    raw_call(action.target, action.calldata, value=action.value)
    
    log ActionExecuted(action_id, msg.sender)

@external
def cancel_action(action_id: uint256):
    """
    @notice ยกเลิก action (proposer หรือ owner)
    """
    action: PendingAction = self.actions[action_id]
    assert not action.executed, "Already executed"
    assert not action.cancelled, "Already cancelled"
    assert msg.sender == action.proposer or msg.sender == self.owner, \
        "Not authorized"
    
    self.actions[action_id].cancelled = True
    
    log ActionCancelled(action_id)

@external
def update_delay(new_delay: uint256):
    """
    @notice อัปเดต delay (เฉพาะ owner)
    """
    self._only_owner()
    assert new_delay >= MIN_DELAY and new_delay <= MAX_DELAY
    
    log DelayUpdated(self.delay, new_delay)
    self.delay = new_delay

@external
def add_proposer(account: address):
    self._only_owner()
    self.proposers[account] = True

@external
def remove_proposer(account: address):
    self._only_owner()
    self.proposers[account] = False

@external
def add_approver(account: address):
    self._only_owner()
    self.approvers[account] = True

@external
def add_executor(account: address):
    self._only_owner()
    self.executors[account] = True

@view
@external
def get_action(action_id: uint256) -> PendingAction:
    return self.actions[action_id]

@view
@external
def can_execute(action_id: uint256) -> bool:
    action: PendingAction = self.actions[action_id]
    if action.eta == 0 or action.executed or action.cancelled:
        return False
    if action.approvals < self.required_approvals:
        return False
    return (block.timestamp >= action.eta and 
            block.timestamp <= action.eta + GRACE_PERIOD)

@view
@external
def has_approved(action_id: uint256, account: address) -> bool:
    return self.action_approvals[action_id][account]
```

---

## Test Code

```python
# tests/test_timelock.py
import pytest
import ape

ONE_DAY = 86400

@pytest.fixture
def owner(accounts):
    return accounts[0]

@pytest.fixture
def proposer(accounts):
    return accounts[1]

@pytest.fixture
def approver1(accounts):
    return accounts[2]

@pytest.fixture
def approver2(accounts):
    return accounts[3]

@pytest.fixture
def executor(accounts):
    return accounts[4]

@pytest.fixture
def timelock(owner, project):
    return project.TimelockController.deploy(
        ONE_DAY,    # 1 day delay
        2,          # 2 approvals required
        sender=owner
    )

class TestTimelockBasic:
    
    def test_initial_state(self, timelock, owner):
        """ตรวจสอบ state เริ่มต้น"""
        assert timelock.delay() == ONE_DAY
        assert timelock.required_approvals() == 2
        assert timelock.owner() == owner.address
    
    def test_propose_action(self, timelock, owner):
        """ทดสอบ propose action"""
        action_id = timelock.propose_action(
            owner.address,  # target
            b"",           # calldata
            0,             # value
            sender=owner
        )
        
        action = timelock.get_action(action_id)
        assert action.target == owner.address
        assert not action.executed
        assert not action.cancelled
    
    def test_cannot_execute_before_delay(self, timelock, owner):
        """ทดสอบว่าไม่สามารถ execute ก่อน delay"""
        action_id = timelock.propose_action(
            owner.address, b"", 0,
            sender=owner
        )
        
        # Approve 2 times
        timelock.approve_action(action_id, sender=owner)
        # ต้องการ approver อีก แต่ในที่นี้จะ fail เพราะ delay ยังไม่ผ่าน
        
        with pytest.raises(Exception):
            timelock.execute_action(action_id, sender=owner)
    
    def test_can_execute_after_delay(self, timelock, owner, chain):
        """ทดสอบ execute หลัง delay"""
        action_id = timelock.propose_action(
            owner.address, b"", 0,
            sender=owner
        )
        
        timelock.approve_action(action_id, sender=owner)
        
        # เดิน time ข้ามไป 1 วัน + 1 วินาที
        chain.mine(deltatime=ONE_DAY + 1)
        
        # ยังต้องการ approval เพิ่ม (required = 2)
        # timelock.execute_action(action_id, sender=owner)
    
    def test_cancel_action(self, timelock, owner):
        """ทดสอบ cancel action"""
        action_id = timelock.propose_action(
            owner.address, b"", 0,
            sender=owner
        )
        
        timelock.cancel_action(action_id, sender=owner)
        
        action = timelock.get_action(action_id)
        assert action.cancelled

class TestTimelockApprovals:
    
    def test_require_sufficient_approvals(self, timelock, owner, approver1, approver2, chain):
        """ทดสอบว่าต้องมี approval เพียงพอ"""
        # Add approvers
        timelock.add_approver(approver1.address, sender=owner)
        timelock.add_approver(approver2.address, sender=owner)
        timelock.add_executor(owner.address, sender=owner)
        
        action_id = timelock.propose_action(
            owner.address, b"", 0,
            sender=owner
        )
        
        # เดิน time
        chain.mine(deltatime=ONE_DAY + 1)
        
        # ยังไม่มี approval - ควร fail
        with pytest.raises(Exception):
            timelock.execute_action(action_id, sender=owner)
        
        # Add 1 approval - ยังไม่พอ
        timelock.approve_action(action_id, sender=approver1)
        with pytest.raises(Exception):
            timelock.execute_action(action_id, sender=owner)
        
        # Add 2nd approval - พอแล้ว
        timelock.approve_action(action_id, sender=approver2)
        timelock.execute_action(action_id, sender=owner)
    
    def test_cannot_approve_twice(self, timelock, owner):
        """ทดสอบว่า approve ซ้ำไม่ได้"""
        action_id = timelock.propose_action(
            owner.address, b"", 0,
            sender=owner
        )
        
        timelock.approve_action(action_id, sender=owner)
        
        with pytest.raises(Exception):
            timelock.approve_action(action_id, sender=owner)
    
    def test_update_delay(self, timelock, owner):
        """ทดสอบ update delay"""
        new_delay = ONE_DAY * 2
        timelock.update_delay(new_delay, sender=owner)
        assert timelock.delay() == new_delay
    
    def test_delay_bounds(self, timelock, owner):
        """ทดสอบขอบเขต delay"""
        # Too short
        with pytest.raises(Exception):
            timelock.update_delay(3600, sender=owner)  # 1 hour
        
        # Too long
        with pytest.raises(Exception):
            timelock.update_delay(31536000, sender=owner)  # 1 year
```

---

## สรุป

Timelock เป็น Pattern ที่สำคัญมากใน DeFi:

| Delay | ใช้สำหรับ |
|-------|---------|
| 1 วัน | Protocol parameter changes |
| 2 วัน | Fee changes |
| 7 วัน | Major upgrades |
| 14 วัน | Critical security changes |
| 30 วัน | Token migration |

### Best Practices
- ใช้ร่วมกับ multisig
- มี grace period เสมอ
- แจ้ง community ก่อน execute
- ออกแบบให้ cancel ได้เสมอ

---

[⬅️ Part 029: ReentrancyGuard](part_029_reentrancy.md) | [Part 031: Multisig Wallet ➡️](part_031_multisig.md)
