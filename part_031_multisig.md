# Part 031: Multisig Wallet

## สารบัญ
1. [บทนำ Multisig](#บทนำ)
2. [M-of-N Signature Scheme](#m-of-n-signature-scheme)
3. [Transaction Proposal](#transaction-proposal)
4. [Approval Mechanism](#approval-mechanism)
5. [Execution](#execution)
6. [Full Multisig Implementation](#full-multisig-implementation)
7. [Test Code](#test-code)

---

## บทนำ

**Multisig (Multi-Signature) Wallet** คือ Wallet ที่ต้องการลายเซ็น (approval) จากหลายคนก่อนดำเนินการ เช่น "2 จาก 3 คน" ต้องอนุมัติก่อนส่งเงิน

### ทำไมต้อง Multisig?
- ป้องกัน single point of failure
- ป้องกัน insider attack
- ต้องการความโปร่งใส
- มาตรฐานสำหรับ Protocol Treasury

### ใช้งานจริงที่ไหน?
- Gnosis Safe (มีมูลค่าหลายล้านล้านบาท)
- Protocol Treasuries
- Team funds
- DAO operations

---

## M-of-N Signature Scheme

### แนวคิด

```
N = จำนวน signer ทั้งหมด
M = จำนวนที่ต้องอนุมัติขั้นต่ำ

ตัวอย่าง:
- 2-of-3: Alice, Bob, Carol - ต้องการ 2 คนในการดำเนินการ
- 3-of-5: Team ใหญ่ - ต้องการ 3 คน
- 1-of-1: เหมือน normal wallet (ไม่ปลอดภัยกว่า)
```

### การกำหนด Signers

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0

MAX_SIGNERS: constant(uint256) = 20

# M-of-N configuration
required: public(uint256)           # M: จำนวนที่ต้องการ
signer_count: public(uint256)       # N: จำนวน signer ทั้งหมด
is_signer: public(HashMap[address, bool])  # ตรวจว่าเป็น signer หรือไม่
signers: public(DynArray[address, MAX_SIGNERS])  # รายชื่อ signers

@deploy
def __init__(
    _signers: DynArray[address, MAX_SIGNERS],
    _required: uint256
):
    assert len(_signers) > 0, "Must have signers"
    assert _required > 0, "Required must be > 0"
    assert _required <= len(_signers), "Required cannot exceed signers"
    
    for signer: address in _signers:
        assert signer != empty(address), "Invalid signer"
        assert not self.is_signer[signer], "Duplicate signer"
        
        self.is_signer[signer] = True
        self.signers.append(signer)
    
    self.signer_count = len(_signers)
    self.required = _required
```

---

## Transaction Proposal

### โครงสร้าง Transaction

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0

MAX_SIGNERS: constant(uint256) = 20
MAX_DATA_SIZE: constant(uint256) = 1024

struct Transaction:
    to: address              # ปลายทาง
    value: uint256           # จำนวน ETH
    data: Bytes[MAX_DATA_SIZE]  # calldata
    executed: bool           # ดำเนินการแล้วหรือไม่
    approvals: uint256       # จำนวน approval

event TransactionSubmitted:
    tx_id: indexed(uint256)
    submitter: indexed(address)
    to: address
    value: uint256

event TransactionApproved:
    tx_id: indexed(uint256)
    approver: indexed(address)

event TransactionRevoked:
    tx_id: indexed(uint256)
    revoker: indexed(address)

event TransactionExecuted:
    tx_id: indexed(uint256)
    executor: indexed(address)
    success: bool

required: public(uint256)
signer_count: public(uint256)
is_signer: public(HashMap[address, bool])

tx_count: public(uint256)
transactions: public(HashMap[uint256, Transaction])
approvals: HashMap[uint256, HashMap[address, bool]]

@deploy
def __init__(
    _signers: DynArray[address, MAX_SIGNERS],
    _required: uint256
):
    assert len(_signers) >= _required
    assert _required > 0
    
    for s: address in _signers:
        assert not self.is_signer[s]
        self.is_signer[s] = True
    
    self.signer_count = len(_signers)
    self.required = _required

@external
@payable
def submit_transaction(
    to: address,
    value: uint256,
    data: Bytes[MAX_DATA_SIZE]
) -> uint256:
    """
    @notice เสนอ transaction ใหม่
    @return tx_id
    """
    assert self.is_signer[msg.sender], "Not a signer"
    assert to != empty(address), "Invalid destination"
    
    tx_id: uint256 = self.tx_count
    
    self.transactions[tx_id] = Transaction({
        to: to,
        value: value,
        data: data,
        executed: False,
        approvals: 0
    })
    
    self.tx_count += 1
    
    log TransactionSubmitted(tx_id, msg.sender, to, value)
    return tx_id
```

---

## Approval Mechanism

### ระบบ Approve และ Revoke

```vyper
# ต่อจากโค้ดข้างบน...

@external
def approve_transaction(tx_id: uint256):
    """
    @notice อนุมัติ transaction
    """
    assert self.is_signer[msg.sender], "Not a signer"
    assert tx_id < self.tx_count, "Invalid tx_id"
    
    tx: Transaction = self.transactions[tx_id]
    assert not tx.executed, "Already executed"
    assert not self.approvals[tx_id][msg.sender], "Already approved"
    
    self.approvals[tx_id][msg.sender] = True
    self.transactions[tx_id].approvals += 1
    
    log TransactionApproved(tx_id, msg.sender)

@external
def revoke_approval(tx_id: uint256):
    """
    @notice เพิกถอน approval ของตัวเอง
    """
    assert self.is_signer[msg.sender], "Not a signer"
    assert tx_id < self.tx_count, "Invalid tx_id"
    
    tx: Transaction = self.transactions[tx_id]
    assert not tx.executed, "Already executed"
    assert self.approvals[tx_id][msg.sender], "Not approved"
    
    self.approvals[tx_id][msg.sender] = False
    self.transactions[tx_id].approvals -= 1
    
    log TransactionRevoked(tx_id, msg.sender)

@view
@external
def get_approval_count(tx_id: uint256) -> uint256:
    return self.transactions[tx_id].approvals

@view
@external
def has_approved(tx_id: uint256, signer: address) -> bool:
    return self.approvals[tx_id][signer]

@view
@external
def is_ready_to_execute(tx_id: uint256) -> bool:
    tx: Transaction = self.transactions[tx_id]
    return (not tx.executed and 
            tx.approvals >= self.required)
```

---

## Execution

### Execute Transaction

```vyper
# ต่อจากโค้ดข้างบน...

@external
def execute_transaction(tx_id: uint256):
    """
    @notice Execute transaction เมื่อได้รับ approval เพียงพอ
    """
    assert self.is_signer[msg.sender], "Not a signer"
    assert tx_id < self.tx_count, "Invalid tx_id"
    
    tx: Transaction = self.transactions[tx_id]
    assert not tx.executed, "Already executed"
    assert tx.approvals >= self.required, "Insufficient approvals"
    assert self.balance >= tx.value, "Insufficient ETH"
    
    # Mark as executed ก่อน (CEI pattern)
    self.transactions[tx_id].executed = True
    
    # Execute
    success: bool = False
    if len(tx.data) > 0:
        success = raw_call(tx.to, tx.data, value=tx.value, revert_on_failure=False)
    else:
        send(tx.to, tx.value)
        success = True
    
    log TransactionExecuted(tx_id, msg.sender, success)
    
    if not success:
        # Revert execution status ถ้าล้มเหลว
        self.transactions[tx_id].executed = False
        raise "Transaction execution failed"
```

---

## Full Multisig Implementation

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Multisig Wallet
# @notice M-of-N Multisig Wallet สมบูรณ์

MAX_SIGNERS: constant(uint256) = 20
MAX_DATA_SIZE: constant(uint256) = 1024

# ==================== Structs ====================

struct Transaction:
    to: address
    value: uint256
    data: Bytes[MAX_DATA_SIZE]
    executed: bool
    approvals: uint256
    submitter: address
    description: String[256]

# ==================== Events ====================

event Deposit:
    sender: indexed(address)
    amount: uint256
    balance: uint256

event TransactionSubmitted:
    tx_id: indexed(uint256)
    submitter: indexed(address)
    to: indexed(address)
    value: uint256
    description: String[256]

event TransactionApproved:
    tx_id: indexed(uint256)
    approver: indexed(address)
    total_approvals: uint256

event TransactionRevoked:
    tx_id: indexed(uint256)
    revoker: indexed(address)

event TransactionExecuted:
    tx_id: indexed(uint256)
    executor: indexed(address)

event TransactionFailed:
    tx_id: indexed(uint256)

event SignerAdded:
    signer: indexed(address)

event SignerRemoved:
    signer: indexed(address)

event RequirementChanged:
    old_required: uint256
    new_required: uint256

# ==================== State Variables ====================

required: public(uint256)
signer_count: public(uint256)
is_signer: public(HashMap[address, bool])
signers: public(DynArray[address, MAX_SIGNERS])

tx_count: public(uint256)
transactions: public(HashMap[uint256, Transaction])
approvals: HashMap[uint256, HashMap[address, bool]]

# ==================== Constructor ====================

@deploy
def __init__(
    _signers: DynArray[address, MAX_SIGNERS],
    _required: uint256
):
    """
    @notice Deploy Multisig Wallet
    @param _signers รายชื่อ signers (M-of-N)
    @param _required จำนวนที่ต้องการ (M)
    """
    n: uint256 = len(_signers)
    assert n > 0, "Must have at least 1 signer"
    assert n <= MAX_SIGNERS, "Too many signers"
    assert _required > 0, "Required must be > 0"
    assert _required <= n, "Required cannot exceed signer count"
    
    for signer: address in _signers:
        assert signer != empty(address), "Invalid signer address"
        assert not self.is_signer[signer], "Duplicate signer"
        
        self.is_signer[signer] = True
        self.signers.append(signer)
    
    self.signer_count = n
    self.required = _required

# ==================== Receive ETH ====================

@external
@payable
def __default__():
    """รับ ETH"""
    log Deposit(msg.sender, msg.value, self.balance)

# ==================== Internal Functions ====================

@internal
def _only_signer():
    assert self.is_signer[msg.sender], "Not a signer"

@internal
def _only_wallet():
    assert msg.sender == self, "Can only be called by wallet itself"

@internal
def _tx_exists(tx_id: uint256):
    assert tx_id < self.tx_count, "Transaction does not exist"

@internal
def _not_executed(tx_id: uint256):
    assert not self.transactions[tx_id].executed, "Already executed"

# ==================== Transaction Functions ====================

@external
def submit_transaction(
    to: address,
    value: uint256,
    data: Bytes[MAX_DATA_SIZE],
    description: String[256]
) -> uint256:
    """
    @notice เสนอ transaction ใหม่
    @param to ปลายทาง
    @param value จำนวน ETH
    @param data calldata (empty สำหรับ ETH transfer)
    @param description คำอธิบาย
    @return tx_id
    """
    self._only_signer()
    assert to != empty(address), "Invalid destination"
    
    tx_id: uint256 = self.tx_count
    
    self.transactions[tx_id] = Transaction({
        to: to,
        value: value,
        data: data,
        executed: False,
        approvals: 0,
        submitter: msg.sender,
        description: description
    })
    
    self.tx_count += 1
    
    log TransactionSubmitted(tx_id, msg.sender, to, value, description)
    return tx_id

@external
def approve_transaction(tx_id: uint256):
    """
    @notice อนุมัติ transaction
    """
    self._only_signer()
    self._tx_exists(tx_id)
    self._not_executed(tx_id)
    
    assert not self.approvals[tx_id][msg.sender], "Already approved"
    
    self.approvals[tx_id][msg.sender] = True
    self.transactions[tx_id].approvals += 1
    
    log TransactionApproved(tx_id, msg.sender, self.transactions[tx_id].approvals)

@external
def revoke_approval(tx_id: uint256):
    """
    @notice เพิกถอน approval
    """
    self._only_signer()
    self._tx_exists(tx_id)
    self._not_executed(tx_id)
    
    assert self.approvals[tx_id][msg.sender], "Not approved"
    
    self.approvals[tx_id][msg.sender] = False
    self.transactions[tx_id].approvals -= 1
    
    log TransactionRevoked(tx_id, msg.sender)

@external
def execute_transaction(tx_id: uint256):
    """
    @notice Execute transaction
    """
    self._only_signer()
    self._tx_exists(tx_id)
    self._not_executed(tx_id)
    
    tx: Transaction = self.transactions[tx_id]
    assert tx.approvals >= self.required, "Not enough approvals"
    assert self.balance >= tx.value, "Insufficient balance"
    
    # Mark as executed ก่อน (CEI pattern)
    self.transactions[tx_id].executed = True
    
    # Execute
    if len(tx.data) > 0:
        raw_call(tx.to, tx.data, value=tx.value)
    else:
        send(tx.to, tx.value)
    
    log TransactionExecuted(tx_id, msg.sender)

# ==================== Signer Management (ต้อง execute ผ่าน multisig) ====================

@external
def add_signer(new_signer: address):
    """
    @notice เพิ่ม signer ใหม่
    @dev ต้องเรียกผ่าน execute_transaction เท่านั้น
    """
    self._only_wallet()
    assert new_signer != empty(address), "Invalid address"
    assert not self.is_signer[new_signer], "Already a signer"
    assert self.signer_count < MAX_SIGNERS, "Max signers reached"
    
    self.is_signer[new_signer] = True
    self.signers.append(new_signer)
    self.signer_count += 1
    
    log SignerAdded(new_signer)

@external
def remove_signer(signer: address):
    """
    @notice ลบ signer
    @dev ต้องเรียกผ่าน execute_transaction เท่านั้น
    """
    self._only_wallet()
    assert self.is_signer[signer], "Not a signer"
    assert self.signer_count > self.required, "Cannot go below required"
    
    self.is_signer[signer] = False
    self.signer_count -= 1
    
    # ลบจาก array
    new_signers: DynArray[address, MAX_SIGNERS] = []
    for s: address in self.signers:
        if s != signer:
            new_signers.append(s)
    self.signers = new_signers
    
    log SignerRemoved(signer)

@external
def change_requirement(new_required: uint256):
    """
    @notice เปลี่ยนจำนวน required approvals
    @dev ต้องเรียกผ่าน execute_transaction เท่านั้น
    """
    self._only_wallet()
    assert new_required > 0, "Required must be > 0"
    assert new_required <= self.signer_count, "Required exceeds signer count"
    
    old_required: uint256 = self.required
    self.required = new_required
    
    log RequirementChanged(old_required, new_required)

# ==================== View Functions ====================

@view
@external
def get_transaction(tx_id: uint256) -> Transaction:
    return self.transactions[tx_id]

@view
@external
def get_signers() -> DynArray[address, MAX_SIGNERS]:
    return self.signers

@view
@external
def has_approved(tx_id: uint256, signer: address) -> bool:
    return self.approvals[tx_id][signer]

@view
@external
def is_ready(tx_id: uint256) -> bool:
    tx: Transaction = self.transactions[tx_id]
    return not tx.executed and tx.approvals >= self.required

@view
@external
def get_pending_transactions() -> DynArray[uint256, 100]:
    """ดึง list ของ pending transactions"""
    pending: DynArray[uint256, 100] = []
    for i: uint256 in range(100):
        if i >= self.tx_count:
            break
        if not self.transactions[i].executed:
            pending.append(i)
    return pending

@view
@external
def get_balance() -> uint256:
    return self.balance
```

---

## Test Code

```python
# tests/test_multisig.py
import pytest

@pytest.fixture
def signers(accounts):
    return accounts[:3]

@pytest.fixture
def alice(accounts):
    return accounts[0]

@pytest.fixture
def bob(accounts):
    return accounts[1]

@pytest.fixture
def carol(accounts):
    return accounts[2]

@pytest.fixture
def outsider(accounts):
    return accounts[4]

@pytest.fixture
def recipient(accounts):
    return accounts[5]

@pytest.fixture
def multisig(alice, bob, carol, project):
    return project.MultisigWallet.deploy(
        [alice.address, bob.address, carol.address],  # 3 signers
        2,  # 2-of-3 required
        sender=alice
    )

class TestMultisigSetup:
    
    def test_initial_signers(self, multisig, alice, bob, carol):
        """ตรวจสอบ signers เริ่มต้น"""
        assert multisig.is_signer(alice.address)
        assert multisig.is_signer(bob.address)
        assert multisig.is_signer(carol.address)
        assert multisig.signer_count() == 3
    
    def test_required_approvals(self, multisig):
        """ตรวจสอบ required approvals"""
        assert multisig.required() == 2
    
    def test_non_signer_detected(self, multisig, outsider):
        """ตรวจสอบว่า non-signer ถูก detect"""
        assert not multisig.is_signer(outsider.address)
    
    def test_invalid_config_fails(self, project, alice, bob):
        """ทดสอบ config ที่ผิด"""
        with pytest.raises(Exception):
            project.MultisigWallet.deploy(
                [alice.address, bob.address],
                3,  # required > signers
                sender=alice
            )

class TestMultisigTransactions:
    
    def test_submit_transaction(self, multisig, alice, recipient):
        """ทดสอบ submit transaction"""
        # Fund the wallet
        alice.transfer(multisig.address, 1 * 10**18)
        
        tx_id = multisig.submit_transaction(
            recipient.address,
            1 * 10**18,
            b"",
            "Test transfer",
            sender=alice
        )
        
        tx = multisig.get_transaction(tx_id)
        assert tx.to == recipient.address
        assert not tx.executed
        assert tx.approvals == 0
    
    def test_approve_transaction(self, multisig, alice, bob, recipient):
        """ทดสอบ approve transaction"""
        alice.transfer(multisig.address, 1 * 10**18)
        
        tx_id = multisig.submit_transaction(
            recipient.address, 10**18, b"", "Transfer",
            sender=alice
        )
        
        multisig.approve_transaction(tx_id, sender=alice)
        multisig.approve_transaction(tx_id, sender=bob)
        
        tx = multisig.get_transaction(tx_id)
        assert tx.approvals == 2
        assert multisig.has_approved(tx_id, alice.address)
        assert multisig.has_approved(tx_id, bob.address)
    
    def test_execute_transaction(self, multisig, alice, bob, recipient):
        """ทดสอบ execute transaction"""
        amount = 10**18
        alice.transfer(multisig.address, amount)
        
        tx_id = multisig.submit_transaction(
            recipient.address, amount, b"", "Transfer",
            sender=alice
        )
        
        multisig.approve_transaction(tx_id, sender=alice)
        multisig.approve_transaction(tx_id, sender=bob)
        
        before = recipient.balance
        multisig.execute_transaction(tx_id, sender=alice)
        after = recipient.balance
        
        assert after - before == amount
        assert multisig.get_transaction(tx_id).executed
    
    def test_cannot_execute_without_approvals(self, multisig, alice, recipient):
        """ทดสอบว่าไม่สามารถ execute โดยไม่มี approval พอ"""
        alice.transfer(multisig.address, 10**18)
        
        tx_id = multisig.submit_transaction(
            recipient.address, 10**18, b"", "Transfer",
            sender=alice
        )
        
        multisig.approve_transaction(tx_id, sender=alice)  # เพียง 1 approval
        
        with pytest.raises(Exception):
            multisig.execute_transaction(tx_id, sender=alice)
    
    def test_cannot_execute_twice(self, multisig, alice, bob, recipient):
        """ทดสอบว่าไม่สามารถ execute ซ้ำ"""
        alice.transfer(multisig.address, 2 * 10**18)
        
        tx_id = multisig.submit_transaction(
            recipient.address, 10**18, b"", "Transfer",
            sender=alice
        )
        
        multisig.approve_transaction(tx_id, sender=alice)
        multisig.approve_transaction(tx_id, sender=bob)
        multisig.execute_transaction(tx_id, sender=alice)
        
        with pytest.raises(Exception):
            multisig.execute_transaction(tx_id, sender=alice)
    
    def test_revoke_approval(self, multisig, alice, bob, carol, recipient):
        """ทดสอบ revoke approval"""
        alice.transfer(multisig.address, 10**18)
        
        tx_id = multisig.submit_transaction(
            recipient.address, 10**18, b"", "Transfer",
            sender=alice
        )
        
        multisig.approve_transaction(tx_id, sender=alice)
        multisig.approve_transaction(tx_id, sender=bob)
        
        # Bob revokes
        multisig.revoke_approval(tx_id, sender=bob)
        
        tx = multisig.get_transaction(tx_id)
        assert tx.approvals == 1
        assert not multisig.has_approved(tx_id, bob.address)
    
    def test_non_signer_cannot_submit(self, multisig, outsider, recipient):
        """Non-signer ไม่สามารถ submit ได้"""
        with pytest.raises(Exception):
            multisig.submit_transaction(
                recipient.address, 0, b"", "Test",
                sender=outsider
            )

class TestMultisigManagement:
    
    def test_add_signer_via_multisig(self, multisig, alice, bob, outsider, project):
        """ทดสอบเพิ่ม signer ผ่าน multisig"""
        # สร้าง calldata สำหรับ add_signer
        # ในการใช้งานจริงจะต้อง encode calldata
        pass
```

---

## สรุป

Multisig Wallet เป็น Pattern สำคัญ:

| Scheme | Use Case |
|--------|----------|
| 1-of-1 | Personal wallet |
| 2-of-3 | Team wallet |
| 3-of-5 | Protocol treasury |
| 5-of-9 | DAO operations |

### ข้อควรระวัง
- กำหนด required ให้เหมาะสม (ไม่สูงหรือต่ำเกินไป)
- มีแผน key recovery
- ทดสอบ flow ก่อน deploy จริง

---

[⬅️ Part 030: Timelock](part_030_timelock.md) | [Part 032: Escrow Contract ➡️](part_032_escrow.md)
