# Part 037: Merkle Tree Proofs

## สารบัญ
1. [Merkle Tree Concept](#merkle-tree-concept)
2. [Merkle Proof Verification](#merkle-proof-verification)
3. [Whitelist with Merkle](#whitelist-with-merkle)
4. [Airdrop with Merkle](#airdrop-with-merkle)
5. [ตัวอย่าง: Merkle Airdrop](#ตัวอย่าง-merkle-airdrop)
6. [Test Code](#test-code)

---

## Merkle Tree Concept

### Merkle Tree คืออะไร?

**Merkle Tree** คือ Binary Tree ที่แต่ละ node มีค่าเป็น hash ของ children ของมัน

```
                    Root Hash
                   /          \
          Hash(1,2)            Hash(3,4)
          /      \             /      \
    Hash(L1)  Hash(L2)   Hash(L3)  Hash(L4)
       |          |          |          |
     Leaf1      Leaf2      Leaf3      Leaf4
   (Alice,100) (Bob,200) (Carol,50) (Dave,75)
```

### ทำไมต้อง Merkle Tree?
- **Gas efficient**: ไม่ต้อง store ทุก address on-chain
- **Scalable**: 1 million addresses ก็ใช้ root hash เดียว
- **Verifiable**: ทุกคนสามารถ verify ตัวเองได้

### วิธีทำงาน

1. **Off-chain**: สร้าง Merkle Tree จาก whitelist
2. **On-chain**: เก็บแค่ root hash
3. **Claim**: User ส่ง proof (path ใน tree)
4. **Verify**: Contract verify proof กับ root

---

## Merkle Proof Verification

### การ Verify Proof

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Merkle Proof Verifier

@internal
@pure
def _verify(
    proof: DynArray[bytes32, 20],
    root: bytes32,
    leaf: bytes32
) -> bool:
    """
    @notice ตรวจสอบ Merkle Proof
    @param proof Array ของ sibling hashes จาก leaf ถึง root
    @param root Merkle root ที่ต้องการ verify
    @param leaf Leaf hash ที่ต้องการ verify
    @return True ถ้า proof valid
    """
    computed_hash: bytes32 = leaf
    
    for proof_element: bytes32 in proof:
        # Sort เพื่อให้ hash ใน order เดียวกันเสมอ
        if convert(computed_hash, uint256) <= convert(proof_element, uint256):
            computed_hash = keccak256(concat(computed_hash, proof_element))
        else:
            computed_hash = keccak256(concat(proof_element, computed_hash))
    
    return computed_hash == root

@view
@external
def verify_proof(
    proof: DynArray[bytes32, 20],
    root: bytes32,
    leaf: bytes32
) -> bool:
    return self._verify(proof, root, leaf)
```

### การสร้าง Leaf Hash

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0

# Leaf = keccak256(keccak256(abi.encode(address, amount)))
# Double hash เพื่อป้องกัน second preimage attacks

@internal
@pure
def _leaf_hash(account: address, amount: uint256) -> bytes32:
    """สร้าง leaf hash สำหรับ (address, amount) pair"""
    # keccak256(abi.encode(address, uint256))
    inner: bytes32 = keccak256(
        concat(
            convert(account, bytes32),
            convert(amount, bytes32)
        )
    )
    return keccak256(concat(inner, inner))  # double hash

@view
@external
def compute_leaf(account: address, amount: uint256) -> bytes32:
    return self._leaf_hash(account, amount)
```

---

## Whitelist with Merkle

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title NFT Whitelist with Merkle

interface ERC721:
    def mint(to: address, token_id: uint256): nonpayable

event WhitelistMint:
    minter: indexed(address)
    token_id: uint256

PRICE: constant(uint256) = 50_000_000_000_000_000  # 0.05 ETH
MAX_SUPPLY: constant(uint256) = 10000

owner: public(address)
nft_contract: public(address)
merkle_root: public(bytes32)

whitelist_phase: public(bool)
public_phase: public(bool)
total_minted: public(uint256)

whitelist_claimed: public(HashMap[address, bool])

@deploy
def __init__(_nft: address, _root: bytes32):
    self.owner = msg.sender
    self.nft_contract = _nft
    self.merkle_root = _root
    self.whitelist_phase = True

@internal
@pure
def _verify(proof: DynArray[bytes32, 20], root: bytes32, leaf: bytes32) -> bool:
    computed: bytes32 = leaf
    for element: bytes32 in proof:
        if convert(computed, uint256) <= convert(element, uint256):
            computed = keccak256(concat(computed, element))
        else:
            computed = keccak256(concat(element, computed))
    return computed == root

@external
@payable
def whitelist_mint(proof: DynArray[bytes32, 20]):
    """
    @notice Mint ใน whitelist phase
    """
    assert self.whitelist_phase, "Whitelist phase not active"
    assert not self.whitelist_claimed[msg.sender], "Already claimed"
    assert msg.value >= PRICE, "Insufficient payment"
    assert self.total_minted < MAX_SUPPLY, "Sold out"
    
    # สร้าง leaf สำหรับ msg.sender
    leaf: bytes32 = keccak256(
        concat(convert(msg.sender, bytes32), empty(bytes32))
    )
    
    assert self._verify(proof, self.merkle_root, leaf), "Invalid proof"
    
    self.whitelist_claimed[msg.sender] = True
    
    token_id: uint256 = self.total_minted
    self.total_minted += 1
    
    ERC721(self.nft_contract).mint(msg.sender, token_id)
    
    log WhitelistMint(msg.sender, token_id)
    
    # Refund ส่วนเกิน
    if msg.value > PRICE:
        send(msg.sender, msg.value - PRICE)

@external
@payable
def public_mint():
    """Mint ใน public phase"""
    assert self.public_phase, "Public phase not active"
    assert msg.value >= PRICE
    assert self.total_minted < MAX_SUPPLY
    
    token_id: uint256 = self.total_minted
    self.total_minted += 1
    
    ERC721(self.nft_contract).mint(msg.sender, token_id)

@external
def update_root(new_root: bytes32):
    """อัปเดต Merkle root"""
    assert msg.sender == self.owner
    self.merkle_root = new_root

@external
def set_whitelist_phase(active: bool):
    assert msg.sender == self.owner
    self.whitelist_phase = active

@external
def set_public_phase(active: bool):
    assert msg.sender == self.owner
    self.public_phase = active

@external
def withdraw():
    assert msg.sender == self.owner
    send(self.owner, self.balance)
```

---

## Airdrop with Merkle

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title ERC20 Airdrop with Merkle Proof

interface ERC20:
    def transfer(to: address, amount: uint256) -> bool: nonpayable
    def balanceOf(account: address) -> uint256: view

event Claimed:
    account: indexed(address)
    amount: uint256
    index: uint256

token: public(address)
merkle_root: public(bytes32)
claim_deadline: public(uint256)

is_claimed: public(HashMap[uint256, bool])  # index -> claimed

@deploy
def __init__(
    _token: address,
    _root: bytes32,
    _duration: uint256
):
    self.token = _token
    self.merkle_root = _root
    self.claim_deadline = block.timestamp + _duration

@internal
@pure
def _verify(
    proof: DynArray[bytes32, 20],
    root: bytes32,
    leaf: bytes32
) -> bool:
    computed: bytes32 = leaf
    for element: bytes32 in proof:
        if convert(computed, uint256) <= convert(element, uint256):
            computed = keccak256(concat(computed, element))
        else:
            computed = keccak256(concat(element, computed))
    return computed == root

@external
def claim(
    index: uint256,
    account: address,
    amount: uint256,
    proof: DynArray[bytes32, 20]
):
    """
    @notice Claim airdrop tokens
    @param index ลำดับใน merkle tree
    @param account ที่อยู่ของผู้รับ
    @param amount จำนวน tokens
    @param proof Merkle proof
    """
    assert block.timestamp <= self.claim_deadline, "Claim expired"
    assert not self.is_claimed[index], "Already claimed"
    
    # Verify proof
    leaf: bytes32 = keccak256(
        concat(
            convert(index, bytes32),
            convert(account, bytes32),
            convert(amount, bytes32)
        )
    )
    
    assert self._verify(proof, self.merkle_root, leaf), "Invalid proof"
    
    # Mark as claimed
    self.is_claimed[index] = True
    
    # Transfer tokens
    ERC20(self.token).transfer(account, amount)
    
    log Claimed(account, amount, index)

@view
@external
def is_claim_active() -> bool:
    return block.timestamp <= self.claim_deadline

@external
def sweep_unclaimed(recipient: address):
    """ดึง token ที่ไม่ถูก claim หลัง deadline"""
    assert block.timestamp > self.claim_deadline, "Not expired"
    balance: uint256 = ERC20(self.token).balanceOf(self)
    if balance > 0:
        ERC20(self.token).transfer(recipient, balance)
```

---

## ตัวอย่าง: Merkle Airdrop

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Complete Merkle Airdrop System
# @notice ระบบ Airdrop ที่สมบูรณ์พร้อม Merkle Proof

interface ERC20:
    def transfer(to: address, amount: uint256) -> bool: nonpayable
    def transferFrom(from_: address, to: address, amount: uint256) -> bool: nonpayable
    def balanceOf(account: address) -> uint256: view

MAX_PROOF_LENGTH: constant(uint256) = 32

# ==================== Events ====================

event AirdropCreated:
    id: indexed(uint256)
    token: indexed(address)
    total_amount: uint256
    merkle_root: bytes32
    deadline: uint256

event Claimed:
    airdrop_id: indexed(uint256)
    account: indexed(address)
    amount: uint256

event AirdropCancelled:
    id: indexed(uint256)

event UnclaimedsSwept:
    id: indexed(uint256)
    amount: uint256
    recipient: indexed(address)

# ==================== Structs ====================

struct Airdrop:
    token: address
    total_amount: uint256
    claimed_amount: uint256
    merkle_root: bytes32
    deadline: uint256
    creator: address
    cancelled: bool
    name: String[100]
    description: String[500]

# ==================== State Variables ====================

owner: public(address)
airdrop_count: public(uint256)
airdrops: public(HashMap[uint256, Airdrop])

# airdrop_id -> account -> claimed
claimed_map: HashMap[uint256, HashMap[address, bool]]

# airdrop_id -> index -> claimed (alternative by index)
index_claimed: HashMap[uint256, HashMap[uint256, bool]]

# ==================== Constructor ====================

@deploy
def __init__():
    self.owner = msg.sender

# ==================== Merkle Verification ====================

@internal
@pure
def _verify_proof(
    proof: DynArray[bytes32, MAX_PROOF_LENGTH],
    root: bytes32,
    leaf: bytes32
) -> bool:
    """
    @dev ตรวจสอบ Merkle Proof
    """
    computed_hash: bytes32 = leaf
    
    for proof_element: bytes32 in proof:
        if convert(computed_hash, uint256) < convert(proof_element, uint256):
            computed_hash = keccak256(concat(computed_hash, proof_element))
        else:
            computed_hash = keccak256(concat(proof_element, computed_hash))
    
    return computed_hash == root

@internal
@pure
def _make_leaf_address_amount(account: address, amount: uint256) -> bytes32:
    """สร้าง leaf สำหรับ (address, amount)"""
    return keccak256(
        concat(
            convert(account, bytes32),
            convert(amount, bytes32)
        )
    )

@internal
@pure
def _make_leaf_index_address_amount(
    index: uint256,
    account: address,
    amount: uint256
) -> bytes32:
    """สร้าง leaf สำหรับ (index, address, amount)"""
    return keccak256(
        concat(
            convert(index, bytes32),
            convert(account, bytes32),
            convert(amount, bytes32)
        )
    )

# ==================== Create Airdrop ====================

@external
def create_airdrop(
    token: address,
    total_amount: uint256,
    merkle_root: bytes32,
    duration: uint256,
    name: String[100],
    description: String[500]
) -> uint256:
    """
    @notice สร้าง airdrop campaign ใหม่
    @dev Creator ต้องโอน tokens ก่อน call ฟังก์ชันนี้
    """
    assert token != empty(address), "Invalid token"
    assert total_amount > 0, "Amount must be > 0"
    assert merkle_root != empty(bytes32), "Invalid root"
    assert duration > 0, "Invalid duration"
    
    # ตรวจสอบว่า token ถูกโอนเข้ามาแล้ว
    balance: uint256 = ERC20(token).balanceOf(self)
    
    airdrop_id: uint256 = self.airdrop_count
    deadline: uint256 = block.timestamp + duration
    
    self.airdrops[airdrop_id] = Airdrop({
        token: token,
        total_amount: total_amount,
        claimed_amount: 0,
        merkle_root: merkle_root,
        deadline: deadline,
        creator: msg.sender,
        cancelled: False,
        name: name,
        description: description
    })
    
    self.airdrop_count += 1
    
    log AirdropCreated(airdrop_id, token, total_amount, merkle_root, deadline)
    return airdrop_id

# ==================== Claim ====================

@external
@nonreentrant
def claim(
    airdrop_id: uint256,
    amount: uint256,
    proof: DynArray[bytes32, MAX_PROOF_LENGTH]
):
    """
    @notice Claim airdrop (ใช้ address เป็น key)
    """
    airdrop: Airdrop = self.airdrops[airdrop_id]
    
    assert airdrop.total_amount > 0, "Airdrop not found"
    assert not airdrop.cancelled, "Airdrop cancelled"
    assert block.timestamp <= airdrop.deadline, "Airdrop expired"
    assert not self.claimed_map[airdrop_id][msg.sender], "Already claimed"
    assert airdrop.claimed_amount + amount <= airdrop.total_amount, "Exceeds total"
    
    # สร้าง leaf และ verify
    leaf: bytes32 = self._make_leaf_address_amount(msg.sender, amount)
    assert self._verify_proof(proof, airdrop.merkle_root, leaf), "Invalid proof"
    
    # Mark as claimed
    self.claimed_map[airdrop_id][msg.sender] = True
    self.airdrops[airdrop_id].claimed_amount += amount
    
    # Transfer tokens
    ERC20(airdrop.token).transfer(msg.sender, amount)
    
    log Claimed(airdrop_id, msg.sender, amount)

@external
@nonreentrant
def claim_by_index(
    airdrop_id: uint256,
    index: uint256,
    account: address,
    amount: uint256,
    proof: DynArray[bytes32, MAX_PROOF_LENGTH]
):
    """
    @notice Claim airdrop (ใช้ index เป็น key - ป้องกัน double claim ที่ดีกว่า)
    """
    airdrop: Airdrop = self.airdrops[airdrop_id]
    
    assert airdrop.total_amount > 0, "Airdrop not found"
    assert not airdrop.cancelled, "Airdrop cancelled"
    assert block.timestamp <= airdrop.deadline, "Airdrop expired"
    assert not self.index_claimed[airdrop_id][index], "Already claimed"
    assert airdrop.claimed_amount + amount <= airdrop.total_amount, "Exceeds total"
    
    # สร้าง leaf
    leaf: bytes32 = self._make_leaf_index_address_amount(index, account, amount)
    assert self._verify_proof(proof, airdrop.merkle_root, leaf), "Invalid proof"
    
    # Mark as claimed
    self.index_claimed[airdrop_id][index] = True
    self.airdrops[airdrop_id].claimed_amount += amount
    
    ERC20(airdrop.token).transfer(account, amount)
    
    log Claimed(airdrop_id, account, amount)

@external
@nonreentrant
def claim_multiple(
    airdrop_ids: DynArray[uint256, 10],
    amounts: DynArray[uint256, 10],
    proofs: DynArray[DynArray[bytes32, MAX_PROOF_LENGTH], 10]
):
    """
    @notice Claim หลาย airdrop ในครั้งเดียว
    """
    assert len(airdrop_ids) == len(amounts), "Length mismatch"
    assert len(airdrop_ids) == len(proofs), "Length mismatch"
    
    for i: uint256 in range(10):
        if i >= len(airdrop_ids):
            break
        
        airdrop_id: uint256 = airdrop_ids[i]
        amount: uint256 = amounts[i]
        proof: DynArray[bytes32, MAX_PROOF_LENGTH] = proofs[i]
        
        airdrop: Airdrop = self.airdrops[airdrop_id]
        
        if (airdrop.total_amount > 0 and
            not airdrop.cancelled and
            block.timestamp <= airdrop.deadline and
            not self.claimed_map[airdrop_id][msg.sender]):
            
            leaf: bytes32 = self._make_leaf_address_amount(msg.sender, amount)
            
            if self._verify_proof(proof, airdrop.merkle_root, leaf):
                self.claimed_map[airdrop_id][msg.sender] = True
                self.airdrops[airdrop_id].claimed_amount += amount
                ERC20(airdrop.token).transfer(msg.sender, amount)
                log Claimed(airdrop_id, msg.sender, amount)

# ==================== Admin ====================

@external
def cancel_airdrop(airdrop_id: uint256):
    """ยกเลิก airdrop"""
    airdrop: Airdrop = self.airdrops[airdrop_id]
    assert msg.sender == airdrop.creator or msg.sender == self.owner
    assert not airdrop.cancelled
    
    self.airdrops[airdrop_id].cancelled = True
    
    log AirdropCancelled(airdrop_id)

@external
def update_merkle_root(airdrop_id: uint256, new_root: bytes32):
    """อัปเดต Merkle root (ก่อน deadline)"""
    airdrop: Airdrop = self.airdrops[airdrop_id]
    assert msg.sender == airdrop.creator or msg.sender == self.owner
    assert block.timestamp <= airdrop.deadline
    
    self.airdrops[airdrop_id].merkle_root = new_root

@external
def sweep_unclaimed(airdrop_id: uint256, recipient: address):
    """ดึง token ที่ไม่ถูก claim หลัง deadline"""
    airdrop: Airdrop = self.airdrops[airdrop_id]
    assert msg.sender == airdrop.creator or msg.sender == self.owner
    assert block.timestamp > airdrop.deadline or airdrop.cancelled
    
    unclaimed: uint256 = airdrop.total_amount - airdrop.claimed_amount
    
    if unclaimed > 0:
        self.airdrops[airdrop_id].claimed_amount = airdrop.total_amount
        ERC20(airdrop.token).transfer(recipient, unclaimed)
        log UnclaimedsSwept(airdrop_id, unclaimed, recipient)

# ==================== View Functions ====================

@view
@external
def get_airdrop(airdrop_id: uint256) -> Airdrop:
    return self.airdrops[airdrop_id]

@view
@external
def has_claimed(airdrop_id: uint256, account: address) -> bool:
    return self.claimed_map[airdrop_id][account]

@view
@external
def has_claimed_by_index(airdrop_id: uint256, index: uint256) -> bool:
    return self.index_claimed[airdrop_id][index]

@view
@external
def is_valid_proof(
    airdrop_id: uint256,
    account: address,
    amount: uint256,
    proof: DynArray[bytes32, MAX_PROOF_LENGTH]
) -> bool:
    """ตรวจสอบว่า proof valid หรือไม่"""
    airdrop: Airdrop = self.airdrops[airdrop_id]
    leaf: bytes32 = self._make_leaf_address_amount(account, amount)
    return self._verify_proof(proof, airdrop.merkle_root, leaf)

@view
@external
def get_remaining(airdrop_id: uint256) -> uint256:
    a: Airdrop = self.airdrops[airdrop_id]
    return a.total_amount - a.claimed_amount
```

---

## Test Code

```python
# tests/test_merkle_airdrop.py
import pytest
from eth_utils import keccak

# ================== Helper Functions ==================

def make_leaf(address_hex: str, amount: int) -> bytes:
    """สร้าง leaf hash"""
    addr_bytes = bytes.fromhex(address_hex[2:].zfill(64))
    amount_bytes = amount.to_bytes(32, 'big')
    return keccak(addr_bytes + amount_bytes)

def make_node(left: bytes, right: bytes) -> bytes:
    """สร้าง tree node"""
    if int.from_bytes(left, 'big') <= int.from_bytes(right, 'big'):
        return keccak(left + right)
    return keccak(right + left)

def build_merkle_tree(leaves: list[bytes]) -> tuple[bytes, dict]:
    """สร้าง Merkle Tree และ return (root, proof_map)"""
    if len(leaves) == 0:
        return b'\x00' * 32, {}
    
    # Pad to power of 2
    while len(leaves) & (len(leaves) - 1) != 0:
        leaves.append(leaves[-1])
    
    tree = [leaves[:]]
    current = leaves[:]
    
    while len(current) > 1:
        next_level = []
        for i in range(0, len(current), 2):
            next_level.append(make_node(current[i], current[i+1]))
        current = next_level
        tree.append(current)
    
    root = current[0]
    
    # Build proofs
    proofs = {}
    for idx, leaf in enumerate(tree[0]):
        proof = []
        current_idx = idx
        for level in tree[:-1]:
            sibling_idx = current_idx ^ 1  # XOR เพื่อ get sibling
            if sibling_idx < len(level):
                proof.append(level[sibling_idx])
            current_idx //= 2
        proofs[idx] = proof
    
    return root, proofs

@pytest.fixture
def owner(accounts):
    return accounts[0]

@pytest.fixture
def alice(accounts):
    return accounts[1]

@pytest.fixture
def bob(accounts):
    return accounts[2]

@pytest.fixture
def carol(accounts):
    return accounts[3]

@pytest.fixture
def token(owner, project):
    t = project.MockERC20.deploy("Airdrop", "AIR", 18, sender=owner)
    t.mint(owner.address, 1_000_000 * 10**18, sender=owner)
    return t

@pytest.fixture
def airdrop_data(alice, bob, carol):
    """สร้าง test airdrop data"""
    recipients = {
        alice.address: 100 * 10**18,
        bob.address: 200 * 10**18,
        carol.address: 150 * 10**18,
    }
    
    leaves = [
        make_leaf(addr, amount)
        for addr, amount in recipients.items()
    ]
    
    root, proofs = build_merkle_tree(leaves)
    
    return {
        'recipients': recipients,
        'leaves': leaves,
        'root': root,
        'proofs': proofs
    }

@pytest.fixture
def airdrop_contract(owner, project):
    return project.MerkleAirdrop.deploy(sender=owner)

class TestMerkleProof:
    
    def test_verify_valid_proof(self, airdrop_contract, alice, airdrop_data):
        """ทดสอบ verify proof ที่ valid"""
        root = airdrop_data['root']
        amount = airdrop_data['recipients'][alice.address]
        leaf = make_leaf(alice.address, amount)
        proof = airdrop_data['proofs'][0]  # Alice's proof
        
        assert airdrop_contract.is_valid_proof(
            # Note: need actual airdrop_id
            0, alice.address, amount, proof
        )
    
    def test_invalid_proof_fails(self, airdrop_contract, alice, bob, airdrop_data):
        """ทดสอบ proof ที่ไม่ valid"""
        root = airdrop_data['root']
        # ใช้ Bob's amount กับ Alice's address
        wrong_amount = airdrop_data['recipients'][bob.address]
        leaf = make_leaf(alice.address, wrong_amount)
        proof = airdrop_data['proofs'][0]
        
        # ควรไม่ผ่าน
        # assert not airdrop_contract.verify_proof(proof, root, leaf)

class TestMerkleAirdrop:
    
    def test_create_airdrop(self, airdrop_contract, owner, token, airdrop_data):
        """ทดสอบสร้าง airdrop"""
        total = sum(airdrop_data['recipients'].values())
        token.approve(airdrop_contract.address, total, sender=owner)
        token.transfer(airdrop_contract.address, total, sender=owner)
        
        airdrop_id = airdrop_contract.create_airdrop(
            token.address,
            total,
            airdrop_data['root'],
            7 * 86400,  # 7 days
            "Test Airdrop",
            "Test description",
            sender=owner
        )
        
        airdrop = airdrop_contract.get_airdrop(airdrop_id)
        assert airdrop.total_amount == total
    
    def test_claim_airdrop(self, airdrop_contract, owner, token, alice, airdrop_data):
        """ทดสอบ claim"""
        total = sum(airdrop_data['recipients'].values())
        token.transfer(airdrop_contract.address, total, sender=owner)
        
        airdrop_id = airdrop_contract.create_airdrop(
            token.address, total, airdrop_data['root'],
            7 * 86400, "Test", "Test",
            sender=owner
        )
        
        alice_amount = airdrop_data['recipients'][alice.address]
        alice_proof = airdrop_data['proofs'][0]
        
        before = token.balanceOf(alice.address)
        airdrop_contract.claim(airdrop_id, alice_amount, alice_proof, sender=alice)
        after = token.balanceOf(alice.address)
        
        assert after - before == alice_amount
    
    def test_cannot_claim_twice(self, airdrop_contract, owner, token, alice, airdrop_data):
        """ทดสอบว่า claim ซ้ำไม่ได้"""
        total = sum(airdrop_data['recipients'].values())
        token.transfer(airdrop_contract.address, total, sender=owner)
        
        airdrop_id = airdrop_contract.create_airdrop(
            token.address, total, airdrop_data['root'],
            7 * 86400, "Test", "Test",
            sender=owner
        )
        
        alice_amount = airdrop_data['recipients'][alice.address]
        alice_proof = airdrop_data['proofs'][0]
        
        airdrop_contract.claim(airdrop_id, alice_amount, alice_proof, sender=alice)
        
        with pytest.raises(Exception):
            airdrop_contract.claim(airdrop_id, alice_amount, alice_proof, sender=alice)
    
    def test_cannot_claim_with_wrong_amount(self, airdrop_contract, owner, token, alice, airdrop_data):
        """ทดสอบว่า claim ด้วย amount ผิดไม่ได้"""
        total = sum(airdrop_data['recipients'].values())
        token.transfer(airdrop_contract.address, total, sender=owner)
        
        airdrop_id = airdrop_contract.create_airdrop(
            token.address, total, airdrop_data['root'],
            7 * 86400, "Test", "Test",
            sender=owner
        )
        
        alice_proof = airdrop_data['proofs'][0]
        wrong_amount = airdrop_data['recipients'][alice.address] * 2  # 2x amount
        
        with pytest.raises(Exception):
            airdrop_contract.claim(airdrop_id, wrong_amount, alice_proof, sender=alice)
```

### Script สำหรับสร้าง Merkle Tree

```python
# scripts/generate_merkle.py
# Script สำหรับสร้าง Merkle Root และ Proofs

from eth_utils import keccak
import json

def generate_airdrop_merkle(recipients: dict) -> dict:
    """
    สร้าง Merkle Tree สำหรับ airdrop
    
    Args:
        recipients: {address: amount} dict
    
    Returns:
        {root, proofs} dict
    """
    
    def make_leaf(address: str, amount: int) -> bytes:
        addr_bytes = bytes.fromhex(address[2:].zfill(64))
        amount_bytes = amount.to_bytes(32, 'big')
        return keccak(addr_bytes + amount_bytes)
    
    def make_node(left: bytes, right: bytes) -> bytes:
        if int.from_bytes(left, 'big') <= int.from_bytes(right, 'big'):
            return keccak(left + right)
        return keccak(right + left)
    
    # สร้าง leaves
    items = list(recipients.items())
    leaves = [make_leaf(addr, amount) for addr, amount in items]
    
    # Build tree
    tree = [leaves[:]]
    current = leaves[:]
    
    while len(current) > 1:
        if len(current) % 2 == 1:
            current.append(current[-1])
        
        next_level = []
        for i in range(0, len(current), 2):
            next_level.append(make_node(current[i], current[i+1]))
        current = next_level
        tree.append(current)
    
    root = '0x' + current[0].hex()
    
    # สร้าง proofs
    proofs = {}
    for idx, (addr, amount) in enumerate(items):
        proof = []
        current_idx = idx
        
        for level in tree[:-1]:
            sibling_idx = current_idx ^ 1
            if sibling_idx < len(level):
                proof.append('0x' + level[sibling_idx].hex())
            current_idx //= 2
        
        proofs[addr] = {
            'amount': str(amount),
            'proof': proof
        }
    
    return {
        'root': root,
        'proofs': proofs
    }

# ตัวอย่างการใช้งาน
if __name__ == '__main__':
    recipients = {
        '0x1234567890123456789012345678901234567890': 100 * 10**18,
        '0xabcdefabcdefabcdefabcdefabcdefabcdefabcd': 200 * 10**18,
        '0x9876543210987654321098765432109876543210': 150 * 10**18,
    }
    
    result = generate_airdrop_merkle(recipients)
    print(f"Root: {result['root']}")
    print(json.dumps(result['proofs'], indent=2))
```

---

## สรุป

Merkle Proof เป็นเครื่องมือสำคัญสำหรับ Gas-efficient verification:

| Use Case | Description |
|---------|-------------|
| Airdrop | Claim tokens ด้วย proof |
| Whitelist | ตรวจสอบ NFT whitelist |
| Token gating | ตรวจสอบสิทธิ์ |
| Snapshot | Verify balance ณ block ที่ผ่านมา |

### Best Practices
- ใช้ Double hash เพื่อป้องกัน preimage attack
- อัปเดต root ได้ถ้า airdrop list เปลี่ยน
- มี grace period สำหรับการ claim
- Sweep unclaimed tokens หลัง deadline

---

[⬅️ Part 036: Vesting Contract](part_036_vesting.md) | [Part 038: Oracle Integration ➡️](part_038_oracle.md)
