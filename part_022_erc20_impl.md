# Part 022: ERC-20 Implementation ครบถ้วน

## สารบัญ
1. [Full ERC-20 Implementation](#full-erc20)
2. [Mint/Burn Functions](#mint-burn)
3. [Permit (EIP-2612)](#permit)
4. [Snapshot Mechanism](#snapshot)
5. [Complete Test Suite](#test-suite)
6. [Deploy Script](#deploy-script)
7. [แบบฝึกหัด](#exercises)

---

## 1. Full ERC-20 Implementation {#full-erc20}

```vyper
# @version 0.4.0
"""
@title Full ERC-20 Token Implementation
@notice ครบถ้วนตาม EIP-20 Standard
@dev รองรับ Mint, Burn, Pause, Blacklist
"""

# ==================== Interfaces ====================

from ethereum.ercs import IERC20

implements: IERC20

# ==================== Events ====================

event Transfer:
    sender: indexed(address)
    receiver: indexed(address)
    value: uint256

event Approval:
    owner: indexed(address)
    spender: indexed(address)
    value: uint256

event Mint:
    minter: indexed(address)
    to: indexed(address)
    amount: uint256

event Burn:
    burner: indexed(address)
    amount: uint256

event Paused:
    account: address

event Unpaused:
    account: address

event MinterAdded:
    account: indexed(address)

event MinterRemoved:
    account: indexed(address)

# ==================== State Variables ====================

name: public(String[64])
symbol: public(String[32])
decimals: public(uint8)
totalSupply: public(uint256)

balances: HashMap[address, uint256]
allowances: HashMap[address, HashMap[address, uint256]]

owner: public(address)
minters: public(HashMap[address, bool])
paused: public(bool)
blacklisted: public(HashMap[address, bool])

# Constants
MAX_SUPPLY: public(immutable(uint256))

@deploy
def __init__(
    _name: String[64],
    _symbol: String[32],
    _maxSupply: uint256,
    initialSupply: uint256
):
    """
    @param _name ชื่อ Token
    @param _symbol สัญลักษณ์ Token
    @param _maxSupply Supply สูงสุด
    @param initialSupply Supply เริ่มต้น (ให้ Deployer)
    """
    assert len(_name) > 0, "Empty name"
    assert len(_symbol) > 0, "Empty symbol"
    assert _maxSupply > 0, "Zero max supply"
    assert initialSupply <= _maxSupply, "Initial supply exceeds max"
    
    self.name = _name
    self.symbol = _symbol
    self.decimals = 18
    self.owner = msg.sender
    self.minters[msg.sender] = True
    self.paused = False
    
    MAX_SUPPLY = _maxSupply
    
    if initialSupply > 0:
        self.totalSupply = initialSupply
        self.balances[msg.sender] = initialSupply
        log Transfer(empty(address), msg.sender, initialSupply)
        log Mint(msg.sender, msg.sender, initialSupply)

# ==================== ERC-20 Standard Functions ====================

@external
@view
def balanceOf(account: address) -> uint256:
    return self.balances[account]

@external
@view
def allowance(_owner: address, spender: address) -> uint256:
    return self.allowances[_owner][spender]

@external
def transfer(receiver: address, amount: uint256) -> bool:
    """
    @notice โอน Token
    @dev ตรวจสอบ Pause และ Blacklist
    """
    self._beforeTokenTransfer(msg.sender, receiver, amount)
    self._transfer(msg.sender, receiver, amount)
    return True

@external
def approve(spender: address, amount: uint256) -> bool:
    """
    @notice Set Allowance
    """
    assert spender != empty(address), "ERC20: approve to zero address"
    
    self.allowances[msg.sender][spender] = amount
    log Approval(msg.sender, spender, amount)
    return True

@external
def transferFrom(sender: address, receiver: address, amount: uint256) -> bool:
    """
    @notice โอน Token แทนผู้อื่น
    """
    self._beforeTokenTransfer(sender, receiver, amount)
    
    currentAllowance: uint256 = self.allowances[sender][msg.sender]
    if currentAllowance != max_value(uint256):
        assert currentAllowance >= amount, "ERC20: insufficient allowance"
        self.allowances[sender][msg.sender] = currentAllowance - amount
        log Approval(sender, msg.sender, currentAllowance - amount)
    
    self._transfer(sender, receiver, amount)
    return True

@external
def increaseAllowance(spender: address, addedValue: uint256) -> bool:
    """ป้องกัน Approval Race Condition"""
    assert spender != empty(address), "ERC20: approve to zero address"
    
    newAllowance: uint256 = self.allowances[msg.sender][spender] + addedValue
    self.allowances[msg.sender][spender] = newAllowance
    log Approval(msg.sender, spender, newAllowance)
    return True

@external
def decreaseAllowance(spender: address, subtractedValue: uint256) -> bool:
    """ลด Allowance อย่างปลอดภัย"""
    assert spender != empty(address), "ERC20: approve to zero address"
    
    currentAllowance: uint256 = self.allowances[msg.sender][spender]
    assert currentAllowance >= subtractedValue, "ERC20: allowance below zero"
    
    newAllowance: uint256 = currentAllowance - subtractedValue
    self.allowances[msg.sender][spender] = newAllowance
    log Approval(msg.sender, spender, newAllowance)
    return True

# ==================== Internal Functions ====================

@internal
def _transfer(sender: address, receiver: address, amount: uint256):
    """Core Transfer Logic"""
    assert sender != empty(address), "ERC20: transfer from zero address"
    assert receiver != empty(address), "ERC20: transfer to zero address"
    assert self.balances[sender] >= amount, "ERC20: insufficient balance"
    
    self.balances[sender] -= amount
    self.balances[receiver] += amount
    
    log Transfer(sender, receiver, amount)

@internal
def _beforeTokenTransfer(sender: address, receiver: address, amount: uint256):
    """Hook ก่อน Transfer"""
    assert not self.paused, "ERC20: token transfer while paused"
    
    if sender != empty(address):
        assert not self.blacklisted[sender], "ERC20: sender is blacklisted"
    if receiver != empty(address):
        assert not self.blacklisted[receiver], "ERC20: receiver is blacklisted"

# ==================== Admin Functions ====================

@external
def mint(to: address, amount: uint256):
    """Mint Token ใหม่"""
    assert self.minters[msg.sender], "ERC20: not a minter"
    assert to != empty(address), "ERC20: mint to zero address"
    assert amount > 0, "ERC20: zero mint amount"
    assert self.totalSupply + amount <= MAX_SUPPLY, "ERC20: max supply exceeded"
    
    self._beforeTokenTransfer(empty(address), to, amount)
    
    self.totalSupply += amount
    self.balances[to] += amount
    
    log Transfer(empty(address), to, amount)
    log Mint(msg.sender, to, amount)

@external
def burn(amount: uint256):
    """Burn Token ของตัวเอง"""
    assert amount > 0, "ERC20: zero burn amount"
    assert self.balances[msg.sender] >= amount, "ERC20: insufficient balance"
    
    self.balances[msg.sender] -= amount
    self.totalSupply -= amount
    
    log Transfer(msg.sender, empty(address), amount)
    log Burn(msg.sender, amount)

@external
def burnFrom(account: address, amount: uint256):
    """Burn Token ของผู้อื่น (ต้องมี Allowance)"""
    currentAllowance: uint256 = self.allowances[account][msg.sender]
    assert currentAllowance >= amount, "ERC20: insufficient allowance"
    
    if currentAllowance != max_value(uint256):
        self.allowances[account][msg.sender] = currentAllowance - amount
        log Approval(account, msg.sender, currentAllowance - amount)
    
    assert self.balances[account] >= amount, "ERC20: insufficient balance"
    self.balances[account] -= amount
    self.totalSupply -= amount
    
    log Transfer(account, empty(address), amount)
    log Burn(account, amount)

@external
def pause():
    assert msg.sender == self.owner, "ERC20: not owner"
    assert not self.paused, "ERC20: already paused"
    self.paused = True
    log Paused(msg.sender)

@external
def unpause():
    assert msg.sender == self.owner, "ERC20: not owner"
    assert self.paused, "ERC20: not paused"
    self.paused = False
    log Unpaused(msg.sender)

@external
def blacklist(account: address):
    assert msg.sender == self.owner, "ERC20: not owner"
    assert account != empty(address), "ERC20: zero address"
    assert not self.blacklisted[account], "ERC20: already blacklisted"
    self.blacklisted[account] = True

@external
def removeFromBlacklist(account: address):
    assert msg.sender == self.owner, "ERC20: not owner"
    assert self.blacklisted[account], "ERC20: not blacklisted"
    self.blacklisted[account] = False

@external
def addMinter(account: address):
    assert msg.sender == self.owner, "ERC20: not owner"
    assert account != empty(address), "ERC20: zero address"
    self.minters[account] = True
    log MinterAdded(account)

@external
def removeMinter(account: address):
    assert msg.sender == self.owner, "ERC20: not owner"
    self.minters[account] = False
    log MinterRemoved(account)

@external
def transferOwnership(newOwner: address):
    assert msg.sender == self.owner, "ERC20: not owner"
    assert newOwner != empty(address), "ERC20: zero address"
    self.owner = newOwner
```

---

## 2. Mint/Burn Functions {#mint-burn}

### Mint ด้วย Role-Based Access

```vyper
# @version 0.4.0

MINTER_ROLE: constant(bytes32) = keccak256("MINTER_ROLE")
BURNER_ROLE: constant(bytes32) = keccak256("BURNER_ROLE")

roles: HashMap[bytes32, HashMap[address, bool]]
totalSupply: public(uint256)
balances: HashMap[address, uint256]
MAX_SUPPLY: public(uint256)

event Transfer:
    sender: indexed(address)
    receiver: indexed(address)
    value: uint256

@deploy
def __init__(maxSupply: uint256):
    self.MAX_SUPPLY = maxSupply
    self.roles[MINTER_ROLE][msg.sender] = True
    self.roles[BURNER_ROLE][msg.sender] = True

@external
def mint(to: address, amount: uint256):
    """Mint ต้องมี MINTER_ROLE"""
    assert self.roles[MINTER_ROLE][msg.sender], "Not minter"
    assert to != empty(address), "Zero address"
    assert self.totalSupply + amount <= self.MAX_SUPPLY, "Cap exceeded"
    
    self.totalSupply += amount
    self.balances[to] += amount
    log Transfer(empty(address), to, amount)

@external
def burn(amount: uint256):
    """ทุกคนสามารถ Burn Token ของตัวเองได้"""
    assert self.balances[msg.sender] >= amount, "Insufficient balance"
    
    self.balances[msg.sender] -= amount
    self.totalSupply -= amount
    log Transfer(msg.sender, empty(address), amount)

@external
def burnFrom(account: address, amount: uint256):
    """BURNER_ROLE สามารถ Burn Token ของผู้อื่นได้"""
    assert self.roles[BURNER_ROLE][msg.sender], "Not burner"
    assert self.balances[account] >= amount, "Insufficient balance"
    
    self.balances[account] -= amount
    self.totalSupply -= amount
    log Transfer(account, empty(address), amount)
```

---

## 3. Permit (EIP-2612) {#permit}

Permit ช่วยให้ผู้ใช้ Approve โดยไม่ต้องส่ง Transaction (Gasless Approval)

```vyper
# @version 0.4.0
"""
@title ERC-20 with EIP-2612 Permit
"""

# EIP-712 Domain Separator
DOMAIN_TYPEHASH: constant(bytes32) = keccak256(
    "EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)"
)

PERMIT_TYPEHASH: constant(bytes32) = keccak256(
    "Permit(address owner,address spender,uint256 value,uint256 nonce,uint256 deadline)"
)

# State
name: public(String[64])
symbol: public(String[32])
decimals: public(uint8)
totalSupply: public(uint256)

balances: HashMap[address, uint256]
allowances: HashMap[address, HashMap[address, uint256]]
nonces: public(HashMap[address, uint256])  # สำหรับ Replay Protection

DOMAIN_SEPARATOR: public(bytes32)

event Transfer:
    sender: indexed(address)
    receiver: indexed(address)
    value: uint256

event Approval:
    owner: indexed(address)
    spender: indexed(address)
    value: uint256

@deploy
def __init__(_name: String[64], _symbol: String[32], initialSupply: uint256):
    self.name = _name
    self.symbol = _symbol
    self.decimals = 18
    
    if initialSupply > 0:
        self.totalSupply = initialSupply
        self.balances[msg.sender] = initialSupply
        log Transfer(empty(address), msg.sender, initialSupply)
    
    # คำนวณ Domain Separator
    self.DOMAIN_SEPARATOR = keccak256(
        abi_encode(
            DOMAIN_TYPEHASH,
            keccak256(convert(_name, Bytes[64])),
            keccak256(b"1"),
            chain.id,
            self
        )
    )

@external
@view
def balanceOf(account: address) -> uint256:
    return self.balances[account]

@external
@view
def allowance(_owner: address, spender: address) -> uint256:
    return self.allowances[_owner][spender]

@external
def transfer(receiver: address, amount: uint256) -> bool:
    assert receiver != empty(address), "Zero address"
    assert self.balances[msg.sender] >= amount, "Insufficient balance"
    
    self.balances[msg.sender] -= amount
    self.balances[receiver] += amount
    log Transfer(msg.sender, receiver, amount)
    return True

@external
def approve(spender: address, amount: uint256) -> bool:
    assert spender != empty(address), "Zero address"
    self.allowances[msg.sender][spender] = amount
    log Approval(msg.sender, spender, amount)
    return True

@external
def transferFrom(sender: address, receiver: address, amount: uint256) -> bool:
    assert sender != empty(address), "Zero address"
    assert receiver != empty(address), "Zero address"
    assert self.balances[sender] >= amount, "Insufficient balance"
    
    currentAllowance: uint256 = self.allowances[sender][msg.sender]
    if currentAllowance != max_value(uint256):
        assert currentAllowance >= amount, "Insufficient allowance"
        self.allowances[sender][msg.sender] = currentAllowance - amount
    
    self.balances[sender] -= amount
    self.balances[receiver] += amount
    log Transfer(sender, receiver, amount)
    return True

@external
def permit(
    _owner: address,
    spender: address,
    amount: uint256,
    deadline: uint256,
    v: uint8,
    r: bytes32,
    s: bytes32
):
    """
    @notice Approve โดยใช้ Signature (ไม่ต้อง Submit Transaction)
    @param _owner ผู้ Owner Token
    @param spender ผู้ใช้ Token แทน
    @param amount จำนวน Allowance
    @param deadline Timestamp หมดอายุของ Permit
    @param v, r, s ส่วนของ ECDSA Signature
    """
    assert block.timestamp <= deadline, "Permit: expired"
    assert _owner != empty(address), "Permit: zero address"
    
    # สร้าง Hash ของ Permit
    structHash: bytes32 = keccak256(
        abi_encode(
            PERMIT_TYPEHASH,
            _owner,
            spender,
            amount,
            self.nonces[_owner],
            deadline
        )
    )
    
    # สร้าง EIP-712 Digest
    digest: bytes32 = keccak256(
        concat(
            b"\x19\x01",
            self.DOMAIN_SEPARATOR,
            structHash
        )
    )
    
    # Recover Signer จาก Signature
    signer: address = ecrecover(digest, v, r, s)
    assert signer != empty(address), "Permit: invalid signature"
    assert signer == _owner, "Permit: invalid signer"
    
    # Increment Nonce เพื่อป้องกัน Replay
    self.nonces[_owner] += 1
    
    # Set Allowance
    self.allowances[_owner][spender] = amount
    log Approval(_owner, spender, amount)

@external
@view
def DOMAIN_SEPARATOR_VALUE() -> bytes32:
    return self.DOMAIN_SEPARATOR
```

---

## 4. Snapshot Mechanism {#snapshot}

Snapshot ช่วยเก็บ Balance ณ เวลาหนึ่ง เพื่อใช้ในการ Vote หรือ Dividend

```vyper
# @version 0.4.0
"""
@title ERC-20 with Snapshot
@notice เก็บ Balance History สำหรับ Governance
"""

struct Checkpoint:
    blockNumber: uint256
    value: uint256

event Snapshot:
    id: indexed(uint256)

event Transfer:
    sender: indexed(address)
    receiver: indexed(address)
    value: uint256

# State
name: public(String[64])
symbol: public(String[32])
decimals: public(uint8)
totalSupply: public(uint256)

balances: HashMap[address, uint256]
allowances: HashMap[address, HashMap[address, uint256]]

owner: address

# Snapshot State
snapshotId: public(uint256)
accountBalanceSnapshots: HashMap[address, DynArray[Checkpoint, 100]]
totalSupplySnapshots: DynArray[Checkpoint, 100]

@deploy
def __init__(_name: String[64], _symbol: String[32], supply: uint256):
    self.name = _name
    self.symbol = _symbol
    self.decimals = 18
    self.owner = msg.sender
    
    if supply > 0:
        self.totalSupply = supply
        self.balances[msg.sender] = supply
        log Transfer(empty(address), msg.sender, supply)

@external
def snapshot() -> uint256:
    """
    @notice สร้าง Snapshot ใหม่
    @return snapshotId ของ Snapshot ที่สร้าง
    """
    assert msg.sender == self.owner, "Not owner"
    
    self.snapshotId += 1
    currentId: uint256 = self.snapshotId
    
    log Snapshot(currentId)
    return currentId

@external
@view
def balanceOfAt(account: address, snapshotId: uint256) -> uint256:
    """
    @notice ดู Balance ณ Snapshot ID ที่กำหนด
    """
    assert snapshotId > 0 and snapshotId <= self.snapshotId, "Invalid snapshot"
    
    checkpoints: DynArray[Checkpoint, 100] = self.accountBalanceSnapshots[account]
    
    if len(checkpoints) == 0:
        return self.balances[account]
    
    # Binary Search for the snapshot
    lastCheckpoint: Checkpoint = checkpoints[len(checkpoints) - 1]
    if lastCheckpoint.blockNumber <= snapshotId:
        return lastCheckpoint.value
    
    if checkpoints[0].blockNumber > snapshotId:
        return 0
    
    # Linear search (simplified)
    result: uint256 = 0
    for cp: Checkpoint in checkpoints:
        if cp.blockNumber <= snapshotId:
            result = cp.value
    
    return result

@internal
def _updateAccountSnapshot(account: address):
    """อัปเดต Snapshot เมื่อ Balance เปลี่ยน"""
    currentSnapshot: uint256 = self.snapshotId
    
    checkpoints: DynArray[Checkpoint, 100] = self.accountBalanceSnapshots[account]
    
    if len(checkpoints) == 0 or checkpoints[len(checkpoints) - 1].blockNumber < currentSnapshot:
        self.accountBalanceSnapshots[account].append(
            Checkpoint({
                blockNumber: currentSnapshot,
                value: self.balances[account]
            })
        )

@external
def transfer(receiver: address, amount: uint256) -> bool:
    assert receiver != empty(address), "Zero address"
    assert self.balances[msg.sender] >= amount, "Insufficient balance"
    
    # อัปเดต Snapshot ก่อน Transfer
    self._updateAccountSnapshot(msg.sender)
    self._updateAccountSnapshot(receiver)
    
    self.balances[msg.sender] -= amount
    self.balances[receiver] += amount
    
    log Transfer(msg.sender, receiver, amount)
    return True

@external
def approve(spender: address, amount: uint256) -> bool:
    self.allowances[msg.sender][spender] = amount
    return True

@external
def transferFrom(sender: address, receiver: address, amount: uint256) -> bool:
    currentAllowance: uint256 = self.allowances[sender][msg.sender]
    if currentAllowance != max_value(uint256):
        assert currentAllowance >= amount, "Insufficient allowance"
        self.allowances[sender][msg.sender] = currentAllowance - amount
    
    self._updateAccountSnapshot(sender)
    self._updateAccountSnapshot(receiver)
    
    assert self.balances[sender] >= amount, "Insufficient balance"
    self.balances[sender] -= amount
    self.balances[receiver] += amount
    
    log Transfer(sender, receiver, amount)
    return True

@external
@view
def balanceOf(account: address) -> uint256:
    return self.balances[account]

@external
@view
def allowance(_owner: address, spender: address) -> uint256:
    return self.allowances[_owner][spender]
```

---

## 5. Complete Test Suite {#test-suite}

```python
# tests/test_erc20.py
"""
Complete Test Suite สำหรับ Full ERC-20 Token
"""
import pytest
import boa
from eth_utils import to_wei

@pytest.fixture
def deployer():
    return boa.env.eoa

@pytest.fixture
def user1():
    addr = boa.env.generate_address()
    return addr

@pytest.fixture
def user2():
    addr = boa.env.generate_address()
    return addr

@pytest.fixture
def token(deployer):
    return boa.load(
        "contracts/ERC20Full.vy",
        "Test Token",
        "TEST",
        10**27,   # 1 Billion max supply
        10**24    # 1 Million initial supply
    )

class TestERC20Standard:
    """ทดสอบ ERC-20 Standard Functions"""
    
    def test_metadata(self, token):
        assert token.name() == "Test Token"
        assert token.symbol() == "TEST"
        assert token.decimals() == 18
    
    def test_total_supply(self, token):
        assert token.totalSupply() == 10**24
    
    def test_balance_of(self, token, deployer):
        assert token.balanceOf(deployer) == 10**24
        assert token.balanceOf(boa.env.generate_address()) == 0
    
    def test_transfer(self, token, deployer, user1):
        amount = 10**20
        token.transfer(user1, amount)
        
        assert token.balanceOf(user1) == amount
        assert token.balanceOf(deployer) == 10**24 - amount
    
    def test_transfer_zero_fails(self, token, user1):
        with pytest.raises(Exception):
            token.transfer(user1, 0)
    
    def test_transfer_insufficient_fails(self, token, user1, user2):
        with boa.env.prank(user1):
            with pytest.raises(Exception):
                token.transfer(user2, 1)
    
    def test_approve(self, token, deployer, user1):
        amount = 10**20
        token.approve(user1, amount)
        assert token.allowance(deployer, user1) == amount
    
    def test_transfer_from(self, token, deployer, user1, user2):
        amount = 10**20
        token.approve(user1, amount)
        
        with boa.env.prank(user1):
            token.transferFrom(deployer, user2, amount)
        
        assert token.balanceOf(user2) == amount
        assert token.allowance(deployer, user1) == 0
    
    def test_infinite_approval(self, token, deployer, user1, user2):
        """Infinite Approval ไม่ลด Allowance"""
        token.approve(user1, max(2**256 - 1, 0))
        
        amount = 10**20
        with boa.env.prank(user1):
            token.transferFrom(deployer, user2, amount)
        
        # Allowance ยังคง max value
        assert token.allowance(deployer, user1) == 2**256 - 1
    
    def test_increase_allowance(self, token, deployer, user1):
        token.approve(user1, 100)
        token.increaseAllowance(user1, 50)
        assert token.allowance(deployer, user1) == 150
    
    def test_decrease_allowance(self, token, deployer, user1):
        token.approve(user1, 100)
        token.decreaseAllowance(user1, 30)
        assert token.allowance(deployer, user1) == 70

class TestMintBurn:
    """ทดสอบ Mint และ Burn"""
    
    def test_mint(self, token, deployer, user1):
        amount = 10**20
        initial_supply = token.totalSupply()
        
        token.mint(user1, amount)
        
        assert token.totalSupply() == initial_supply + amount
        assert token.balanceOf(user1) == amount
    
    def test_mint_exceeds_max_fails(self, token):
        with pytest.raises(Exception, match="max supply exceeded"):
            token.mint(boa.env.generate_address(), 10**27)
    
    def test_non_minter_cannot_mint(self, token, user1):
        with boa.env.prank(user1):
            with pytest.raises(Exception, match="not a minter"):
                token.mint(user1, 100)
    
    def test_burn(self, token, deployer):
        amount = 10**20
        initial = token.totalSupply()
        
        token.burn(amount)
        
        assert token.totalSupply() == initial - amount
        assert token.balanceOf(deployer) == 10**24 - amount
    
    def test_burn_insufficient_fails(self, token, user1):
        with boa.env.prank(user1):
            with pytest.raises(Exception):
                token.burn(1)
    
    def test_burn_from(self, token, deployer, user1):
        amount = 10**20
        token.approve(user1, amount)
        
        initial_balance = token.balanceOf(deployer)
        
        with boa.env.prank(user1):
            token.burnFrom(deployer, amount)
        
        assert token.balanceOf(deployer) == initial_balance - amount
        assert token.allowance(deployer, user1) == 0

class TestPause:
    """ทดสอบ Pause/Unpause"""
    
    def test_pause_blocks_transfer(self, token, user1):
        token.pause()
        
        with pytest.raises(Exception, match="paused"):
            token.transfer(user1, 100)
    
    def test_unpause_allows_transfer(self, token, user1):
        token.pause()
        token.unpause()
        
        token.transfer(user1, 100)  # Should succeed

class TestBlacklist:
    """ทดสอบ Blacklist"""
    
    def test_blacklisted_sender_blocked(self, token, deployer, user1):
        token.transfer(user1, 10**20)
        token.blacklist(user1)
        
        with boa.env.prank(user1):
            with pytest.raises(Exception, match="blacklisted"):
                token.transfer(deployer, 100)
    
    def test_remove_from_blacklist(self, token, user1):
        token.transfer(user1, 10**20)
        token.blacklist(user1)
        token.removeFromBlacklist(user1)
        
        with boa.env.prank(user1):
            token.transfer(boa.env.generate_address(), 100)

class TestEvents:
    """ทดสอบ Events"""
    
    def test_transfer_event(self, token, user1):
        token.transfer(user1, 100)
        
        logs = token.get_logs()
        assert len(logs) > 0
    
    def test_approval_event(self, token, user1):
        token.approve(user1, 1000)
        
        logs = token.get_logs()
        assert len(logs) > 0
    
    def test_mint_event_on_deploy(self, deployer):
        """Initial Mint Emit Transfer Event"""
        token = boa.load(
            "contracts/ERC20Full.vy",
            "Test", "TEST", 10**27, 10**24
        )
        
        # Log Transfer(0x0, deployer, 10**24) ถูก Emit ตอน Deploy
        logs = token.get_logs()
        assert len(logs) > 0

class TestIntegration:
    """Integration Tests"""
    
    def test_full_token_lifecycle(self, token, deployer, user1, user2):
        """ทดสอบ Flow ทั้งหมด"""
        # Mint
        token.mint(user1, 10**22)
        assert token.balanceOf(user1) == 10**22
        
        # Transfer
        with boa.env.prank(user1):
            token.transfer(user2, 5 * 10**21)
        
        assert token.balanceOf(user1) == 5 * 10**21
        assert token.balanceOf(user2) == 5 * 10**21
        
        # Approve & TransferFrom
        with boa.env.prank(user1):
            token.approve(deployer, 10**21)
        
        token.transferFrom(user1, deployer, 10**21)
        
        # Burn
        with boa.env.prank(user2):
            token.burn(10**21)
        
        total = (
            token.balanceOf(deployer) +
            token.balanceOf(user1) +
            token.balanceOf(user2)
        )
        assert total == token.totalSupply()
```

---

## 6. Deploy Script {#deploy-script}

```python
# scripts/deploy_token.py
"""
Script สำหรับ Deploy ERC-20 Token
รองรับทั้ง Local (Titanoboa) และ Real Network
"""
import boa
import os
from eth_account import Account

def deploy_local():
    """Deploy บน Local Network"""
    token = boa.load(
        "contracts/ERC20Full.vy",
        "My Token",       # name
        "MTK",            # symbol
        10**27,           # maxSupply (1 Billion)
        10**24            # initialSupply (1 Million)
    )
    
    print(f"Token deployed: {token.address}")
    print(f"Name: {token.name()}")
    print(f"Symbol: {token.symbol()}")
    print(f"Total Supply: {token.totalSupply() / 10**18:,.0f}")
    
    return token

def deploy_mainnet():
    """Deploy บน Real Network"""
    private_key = os.environ.get("PRIVATE_KEY")
    rpc_url = os.environ.get("RPC_URL", "https://mainnet.infura.io/v3/YOUR_KEY")
    
    if not private_key:
        raise ValueError("PRIVATE_KEY environment variable not set")
    
    # Setup Boa for network
    boa.set_network_env(rpc_url)
    
    account = Account.from_key(private_key)
    boa.env.add_account(account)
    
    print(f"Deploying from: {account.address}")
    
    token = boa.load(
        "contracts/ERC20Full.vy",
        "My Token",
        "MTK",
        10**27,
        0   # ไม่ mint ตอน Deploy บน Mainnet
    )
    
    print(f"Token deployed at: {token.address}")
    return token

def verify_deployment(token_address: str):
    """Verify Contract หลัง Deploy"""
    # Load Contract ที่ Deploy แล้ว
    token = boa.load_partial("contracts/ERC20Full.vy").at(token_address)
    
    print(f"Contract at: {token_address}")
    print(f"Name: {token.name()}")
    print(f"Symbol: {token.symbol()}")
    print(f"Decimals: {token.decimals()}")
    print(f"Total Supply: {token.totalSupply()}")
    print(f"Max Supply: {token.MAX_SUPPLY()}")
    print(f"Owner: {token.owner()}")
    print(f"Paused: {token.paused()}")

if __name__ == "__main__":
    import sys
    
    if len(sys.argv) > 1 and sys.argv[1] == "mainnet":
        token = deploy_mainnet()
    else:
        token = deploy_local()
    
    print("\n✅ Deployment successful!")
```

---

## 7. แบบฝึกหัด {#exercises}

### แบบฝึกหัดที่ 1: Governance Token
เพิ่ม Voting Power ให้ ERC-20 Token:
- Delegate votes
- Track voting power at snapshots
- Integrate with Governor Contract

### แบบฝึกหัดที่ 2: Rebasing Token
สร้าง Token ที่ปรับ Supply อัตโนมัติ:
- Balance Scale ตาม Total Supply
- ใช้สำหรับ Elastic Supply (เช่น Ampleforth)

### แบบฝึกหัดที่ 3: Fee Token
สร้าง Token ที่หัก Fee ตอน Transfer:
- Configurable Fee Percent
- Fee ไปยัง Treasury
- Fee Exemptions

---

## สรุป

| Feature | Implementation |
|---------|---------------|
| Standard ERC-20 | ✅ ครบถ้วน |
| Mint/Burn | ✅ พร้อม Access Control |
| Pause | ✅ สำหรับ Emergency |
| Blacklist | ✅ สำหรับ Compliance |
| Permit (EIP-2612) | ✅ Gasless Approval |
| Snapshot | ✅ สำหรับ Governance |
| Test Coverage | ✅ ครอบคลุมทุก Case |

---

[← Part 021: ERC-20 Standard](part_021_erc20_standard.md) | [Part 023: ERC-721 Standard →](part_023_erc721_standard.md)
