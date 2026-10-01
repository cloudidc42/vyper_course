# Part 048: Merkle Trees in Vyper

## สารบัญ
1. [Overview](#overview)
2. [Merkle Tree Concepts](#concepts)
3. [Proof Generation (Python)](#proof-gen)
4. [MerkleAirdrop Contract](#airdrop)
5. [Whitelist Minting](#whitelist)
6. [Batch Claims](#batch-claims)
7. [Complete Tests](#tests)

---

## 1. Overview {#overview}

Merkle Tree เป็น data structure ที่ใช้ verify ข้อมูลขนาดใหญ่ด้วย proof ขนาดเล็ก ในบริบทของ Smart Contract ใช้สำหรับ:

- **Airdrop**: แจก tokens ให้กับ eligible addresses
- **Whitelist**: ตรวจสอบว่า address อยู่ใน whitelist
- **Allowlist**: จัดการรายการสิทธิ์โดยไม่ต้องเก็บทุก address ใน storage

### ทำไมต้องใช้ Merkle Tree?

**แบบปกติ** (เก็บ mapping):
- เก็บ 10,000 addresses = ใช้ gas ~200M (เก็บ 10,000 SSTOREs)
- อัปเดทข้อมูล = ราคาแพงมาก

**แบบ Merkle Tree**:
- เก็บแค่ merkle root (1 bytes32) = gas ถูกมาก
- ผู้ใช้ส่ง proof มาพร้อม transaction
- Contract verify ว่า address อยู่ใน tree

### Gas Comparison

| Approach | Storage Gas | Verification Gas |
|----------|------------|-----------------|
| Mapping 10,000 entries | ~200M | ~2,100 |
| Merkle Root | ~22,000 | ~5,000-50,000 |

---

## 2. Merkle Tree Concepts {#concepts}

```
         Root Hash
        /          \
    H(A+B)          H(C+D)
    /    \          /    \
  H(A)  H(B)     H(C)   H(D)
   |     |        |       |
  A[0]  A[1]    A[2]   A[3]

A = leaf data (address + amount)
H = hash function (keccak256)
```

### การ Verify

เพื่อ prove ว่า `A[0]` อยู่ใน tree:
1. คำนวณ `H(A[0])`
2. รับ proof: `[H(B), H(C+D)]`
3. คำนวณ: `H(H(A[0]) + H(B))` = `H(A+B)`
4. คำนวณ: `H(H(A+B) + H(C+D))` = Root
5. ถ้า Root ตรงกัน = verified!

---

## 3. Proof Generation (Python) {#proof-gen}

```python
# scripts/merkle.py
"""
Python script สำหรับสร้าง Merkle Tree และ proofs
"""

import hashlib
import json
from eth_abi import encode
from eth_utils import keccak

def keccak256(data: bytes) -> bytes:
    """คำนวณ keccak256 hash"""
    return keccak(data)

def hash_leaf(address: str, amount: int) -> bytes:
    """Hash ข้อมูล leaf node"""
    # Encode address + amount เหมือนกับที่ Vyper ทำ
    encoded = encode(
        ['address', 'uint256'],
        [address, amount]
    )
    # Double hash เพื่อป้องกัน second preimage attacks
    return keccak256(keccak256(encoded))

class MerkleTree:
    def __init__(self, leaves: list[tuple[str, int]]):
        """
        leaves: list of (address, amount) tuples
        """
        self.leaves = leaves
        self.leaf_hashes = [hash_leaf(addr, amount) for addr, amount in leaves]
        self.tree = self._build_tree(self.leaf_hashes)
    
    def _build_tree(self, leaves: list[bytes]) -> list[list[bytes]]:
        """Build Merkle Tree จาก leaf hashes"""
        if len(leaves) == 0:
            return []
        
        # ถ้า odd number, duplicate last leaf
        current_level = list(leaves)
        if len(current_level) % 2 != 0:
            current_level.append(current_level[-1])
        
        tree = [current_level]
        
        while len(current_level) > 1:
            next_level = []
            for i in range(0, len(current_level), 2):
                left = current_level[i]
                right = current_level[i + 1] if i + 1 < len(current_level) else left
                
                # Sort hashes ก่อน hash เพื่อให้ order ไม่สำคัญ
                if left <= right:
                    parent = keccak256(left + right)
                else:
                    parent = keccak256(right + left)
                
                next_level.append(parent)
            
            if len(next_level) % 2 != 0 and len(next_level) > 1:
                next_level.append(next_level[-1])
            
            tree.append(next_level)
            current_level = next_level
        
        return tree
    
    @property
    def root(self) -> bytes:
        """Get Merkle Root"""
        if not self.tree:
            return b'\x00' * 32
        return self.tree[-1][0]
    
    @property
    def root_hex(self) -> str:
        """Get Merkle Root as hex string"""
        return '0x' + self.root.hex()
    
    def get_proof(self, index: int) -> list[bytes]:
        """Get proof สำหรับ leaf ที่ index"""
        if index >= len(self.leaf_hashes):
            raise ValueError(f"Index {index} out of range")
        
        proof = []
        current_index = index
        
        for level in self.tree[:-1]:  # ไม่รวม root
            # หา sibling
            if current_index % 2 == 0:
                sibling_index = current_index + 1
            else:
                sibling_index = current_index - 1
            
            if sibling_index < len(level):
                proof.append(level[sibling_index])
            
            # ขึ้น level ถัดไป
            current_index //= 2
        
        return proof
    
    def get_proof_hex(self, index: int) -> list[str]:
        """Get proof เป็น hex strings"""
        return ['0x' + p.hex() for p in self.get_proof(index)]
    
    def verify(self, leaf_hash: bytes, proof: list[bytes], root: bytes) -> bool:
        """Verify proof"""
        computed = leaf_hash
        
        for proof_element in proof:
            if computed <= proof_element:
                computed = keccak256(computed + proof_element)
            else:
                computed = keccak256(proof_element + computed)
        
        return computed == root


def generate_airdrop_data(recipients: list[tuple[str, int]]) -> dict:
    """
    Generate complete airdrop data including tree and proofs
    
    recipients: list of (address, amount) tuples
    Returns dict with root and proofs for each recipient
    """
    tree = MerkleTree(recipients)
    
    result = {
        "root": tree.root_hex,
        "recipients": {}
    }
    
    for i, (address, amount) in enumerate(recipients):
        proof = tree.get_proof_hex(i)
        leaf = '0x' + hash_leaf(address, amount).hex()
        
        result["recipients"][address] = {
            "amount": amount,
            "proof": proof,
            "leaf": leaf,
            "index": i
        }
    
    return result


# ===== ตัวอย่างการใช้งาน =====

if __name__ == "__main__":
    # สร้างรายการ recipients
    recipients = [
        ("0x1111111111111111111111111111111111111111", 100 * 10**18),
        ("0x2222222222222222222222222222222222222222", 200 * 10**18),
        ("0x3333333333333333333333333333333333333333", 150 * 10**18),
        ("0x4444444444444444444444444444444444444444", 300 * 10**18),
        ("0x5555555555555555555555555555555555555555", 250 * 10**18),
    ]
    
    # สร้าง Merkle Tree
    tree = MerkleTree(recipients)
    
    print(f"Merkle Root: {tree.root_hex}")
    print(f"Number of recipients: {len(recipients)}")
    print()
    
    # แสดง proof สำหรับแต่ละ recipient
    for i, (address, amount) in enumerate(recipients):
        proof = tree.get_proof_hex(i)
        print(f"Recipient {i}: {address}")
        print(f"  Amount: {amount / 10**18} tokens")
        print(f"  Proof: {proof}")
        print()
    
    # Generate complete airdrop data
    airdrop_data = generate_airdrop_data(recipients)
    
    # บันทึก JSON สำหรับใช้ใน frontend
    with open("airdrop_data.json", "w") as f:
        json.dump(airdrop_data, f, indent=2)
    
    print("Airdrop data saved to airdrop_data.json")
```

---

## 4. MerkleAirdrop Contract {#airdrop}

```vyper
# @version 0.4.0
# @title MerkleAirdrop
# @notice Airdrop contract ใช้ Merkle Tree สำหรับ verification

interface IERC20:
    def transfer(to: address, amount: uint256) -> bool: nonpayable
    def balanceOf(owner: address) -> uint256: view

event Claimed:
    account: indexed(address)
    amount: uint256

event AirdropFunded:
    amount: uint256

event RootUpdated:
    old_root: bytes32
    new_root: bytes32

# State
owner: public(address)
token: public(address)
merkle_root: public(bytes32)
is_claimed: public(HashMap[address, bool])
total_claimed: public(uint256)
total_funded: public(uint256)
claim_deadline: public(uint256)
is_active: public(bool)

@deploy
def __init__(
    token: address,
    merkle_root: bytes32,
    claim_period_days: uint256
):
    assert token != empty(address), "Zero token"
    assert merkle_root != empty(bytes32), "Empty root"
    
    self.owner = msg.sender
    self.token = token
    self.merkle_root = merkle_root
    self.claim_deadline = block.timestamp + claim_period_days * 86400
    self.is_active = True

@internal
@view
def _verify_proof(
    proof: DynArray[bytes32, 20],
    leaf: bytes32
) -> bool:
    """
    Verify Merkle proof
    proof: list ของ sibling hashes
    leaf: hash ของ data ที่ต้องการ verify
    """
    computed_hash: bytes32 = leaf
    
    for proof_element: bytes32 in proof:
        if convert(computed_hash, uint256) <= convert(proof_element, uint256):
            # computed_hash เป็น left node
            computed_hash = keccak256(
                concat(convert(computed_hash, Bytes[32]), convert(proof_element, Bytes[32]))
            )
        else:
            # computed_hash เป็น right node
            computed_hash = keccak256(
                concat(convert(proof_element, Bytes[32]), convert(computed_hash, Bytes[32]))
            )
    
    return computed_hash == self.merkle_root

@internal
@pure
def _leaf_hash(account: address, amount: uint256) -> bytes32:
    """
    สร้าง leaf hash จาก account + amount
    Double hash เพื่อป้องกัน second preimage attack
    """
    return keccak256(
        keccak256(
            concat(
                convert(account, Bytes[32]),
                convert(amount, Bytes[32])
            )
        )
    )

@external
@view
def can_claim(
    account: address,
    amount: uint256,
    proof: DynArray[bytes32, 20]
) -> bool:
    """ตรวจสอบว่า account สามารถ claim ได้"""
    if self.is_claimed[account]:
        return False
    
    if not self.is_active:
        return False
    
    if block.timestamp > self.claim_deadline:
        return False
    
    leaf: bytes32 = self._leaf_hash(account, amount)
    return self._verify_proof(proof, leaf)

@external
def claim(
    amount: uint256,
    proof: DynArray[bytes32, 20]
):
    """
    Claim airdrop tokens
    amount: จำนวน tokens ที่ eligible
    proof: Merkle proof
    """
    assert self.is_active, "Airdrop not active"
    assert block.timestamp <= self.claim_deadline, "Claim period ended"
    assert not self.is_claimed[msg.sender], "Already claimed"
    
    # Verify proof
    leaf: bytes32 = self._leaf_hash(msg.sender, amount)
    assert self._verify_proof(proof, leaf), "Invalid proof"
    
    # Mark as claimed BEFORE transfer (reentrancy protection)
    self.is_claimed[msg.sender] = True
    self.total_claimed += amount
    
    # Transfer tokens
    assert IERC20(self.token).transfer(msg.sender, amount), "Transfer failed"
    
    log Claimed(msg.sender, amount)

@external
def fund(amount: uint256):
    """เติม tokens เข้า airdrop contract"""
    # ต้องให้ contract approve ก่อนเรียก function นี้
    from_addr: address = msg.sender
    assert IERC20(self.token).balanceOf(from_addr) >= amount, "Insufficient balance"
    
    self.total_funded += amount
    log AirdropFunded(amount)

@external
@view
def remaining_balance() -> uint256:
    """จำนวน tokens ที่เหลือใน contract"""
    return IERC20(self.token).balanceOf(self)

@external
def update_root(new_root: bytes32):
    """อัปเดท merkle root (เช่น เพิ่ม recipients)"""
    assert msg.sender == self.owner, "Not owner"
    assert new_root != empty(bytes32), "Empty root"
    
    old_root: bytes32 = self.merkle_root
    self.merkle_root = new_root
    
    log RootUpdated(old_root, new_root)

@external
def set_active(active: bool):
    assert msg.sender == self.owner, "Not owner"
    self.is_active = active

@external
def recover_unclaimed():
    """Owner สามารถ recover tokens ที่ไม่ได้ claim หลังจาก deadline"""
    assert msg.sender == self.owner, "Not owner"
    assert block.timestamp > self.claim_deadline, "Claim period not ended"
    
    balance: uint256 = IERC20(self.token).balanceOf(self)
    if balance > 0:
        IERC20(self.token).transfer(self.owner, balance)
```

---

## 5. Whitelist Minting {#whitelist}

```vyper
# @version 0.4.0
# @title MerkleWhitelist
# @notice NFT contract ใช้ Merkle Tree สำหรับ whitelist verification

interface IERC721Receiver:
    def onERC721Received(
        operator: address,
        frm: address,
        tokenId: uint256,
        data: Bytes[1024]
    ) -> bytes4: nonpayable

event Transfer:
    sender: indexed(address)
    receiver: indexed(address)
    tokenId: indexed(uint256)

event Approval:
    owner: indexed(address)
    approved: indexed(address)
    tokenId: indexed(uint256)

event ApprovalForAll:
    owner: indexed(address)
    operator: indexed(address)
    approved: bool

MAX_SUPPLY: constant(uint256) = 10000
MAX_PER_WALLET: constant(uint256) = 3

owner: public(address)
name: public(String[64])
symbol: public(String[32])
base_uri: public(String[256])

owner_of: HashMap[uint256, address]
balances: HashMap[address, uint256]
token_approvals: HashMap[uint256, address]
operator_approvals: HashMap[address, HashMap[address, bool]]

total_supply: public(uint256)
next_id: uint256

# Sale state
whitelist_merkle_root: public(bytes32)
public_sale_active: public(bool)
whitelist_sale_active: public(bool)
mint_price: public(uint256)
whitelist_price: public(uint256)

whitelist_minted: public(HashMap[address, uint256])
public_minted: public(HashMap[address, uint256])

@deploy
def __init__(
    name: String[64],
    symbol: String[32],
    base_uri: String[256],
    whitelist_root: bytes32,
    mint_price: uint256,
    whitelist_price: uint256
):
    self.owner = msg.sender
    self.name = name
    self.symbol = symbol
    self.base_uri = base_uri
    self.whitelist_merkle_root = whitelist_root
    self.mint_price = mint_price
    self.whitelist_price = whitelist_price
    self.next_id = 1

@internal
@view
def _verify_whitelist(
    account: address,
    proof: DynArray[bytes32, 20]
) -> bool:
    """Verify ว่า account อยู่ใน whitelist"""
    # Leaf = keccak256(keccak256(abi.encodePacked(account)))
    leaf: bytes32 = keccak256(
        keccak256(convert(account, Bytes[32]))
    )
    
    computed: bytes32 = leaf
    for elem: bytes32 in proof:
        if convert(computed, uint256) <= convert(elem, uint256):
            computed = keccak256(
                concat(convert(computed, Bytes[32]), convert(elem, Bytes[32]))
            )
        else:
            computed = keccak256(
                concat(convert(elem, Bytes[32]), convert(computed, Bytes[32]))
            )
    
    return computed == self.whitelist_merkle_root

@internal
def _mint(to: address):
    token_id: uint256 = self.next_id
    self.next_id += 1
    self.total_supply += 1
    self.balances[to] += 1
    self.owner_of[token_id] = to
    log Transfer(empty(address), to, token_id)

@external
@payable
def whitelistMint(
    quantity: uint256,
    proof: DynArray[bytes32, 20]
):
    """Mint สำหรับ whitelisted addresses"""
    assert self.whitelist_sale_active, "Whitelist sale not active"
    assert self._verify_whitelist(msg.sender, proof), "Not in whitelist"
    assert quantity > 0, "Quantity must be > 0"
    assert self.whitelist_minted[msg.sender] + quantity <= MAX_PER_WALLET, \
        "Exceeds whitelist limit"
    assert self.total_supply + quantity <= MAX_SUPPLY, "Sold out"
    
    total_price: uint256 = self.whitelist_price * quantity
    assert msg.value >= total_price, "Insufficient ETH"
    
    self.whitelist_minted[msg.sender] += quantity
    
    for i: uint256 in range(3):
        if i >= quantity:
            break
        self._mint(msg.sender)
    
    if msg.value > total_price:
        send(msg.sender, msg.value - total_price)

@external
@payable
def publicMint(quantity: uint256):
    assert self.public_sale_active, "Public sale not active"
    assert quantity > 0, "Must mint > 0"
    assert self.public_minted[msg.sender] + quantity <= MAX_PER_WALLET, \
        "Exceeds public limit"
    assert self.total_supply + quantity <= MAX_SUPPLY, "Sold out"
    
    total_price: uint256 = self.mint_price * quantity
    assert msg.value >= total_price, "Insufficient ETH"
    
    self.public_minted[msg.sender] += quantity
    
    for i: uint256 in range(3):
        if i >= quantity:
            break
        self._mint(msg.sender)
    
    if msg.value > total_price:
        send(msg.sender, msg.value - total_price)

@external
def setSaleState(whitelist: bool, public_sale: bool):
    assert msg.sender == self.owner, "Not owner"
    self.whitelist_sale_active = whitelist
    self.public_sale_active = public_sale

@external
def updateWhitelistRoot(new_root: bytes32):
    assert msg.sender == self.owner, "Not owner"
    self.whitelist_merkle_root = new_root

@external
def balanceOf(account: address) -> uint256:
    return self.balances[account]

@external
def ownerOf(tokenId: uint256) -> address:
    owner: address = self.owner_of[tokenId]
    assert owner != empty(address), "Nonexistent"
    return owner

@external
def transferFrom(sender: address, to: address, tokenId: uint256):
    owner: address = self.owner_of[tokenId]
    assert msg.sender == owner or \
           self.operator_approvals[owner][msg.sender] or \
           self.token_approvals[tokenId] == msg.sender, "Not approved"
    assert sender == owner, "Wrong owner"
    assert to != empty(address), "Zero address"
    
    self.token_approvals[tokenId] = empty(address)
    self.balances[sender] -= 1
    self.balances[to] += 1
    self.owner_of[tokenId] = to
    log Transfer(sender, to, tokenId)

@external
def approve(approved: address, tokenId: uint256):
    owner: address = self.owner_of[tokenId]
    assert msg.sender == owner or self.operator_approvals[owner][msg.sender], \
        "Not authorized"
    self.token_approvals[tokenId] = approved
    log Approval(owner, approved, tokenId)

@external
def setApprovalForAll(operator: address, approved: bool):
    assert operator != msg.sender, "Self approval"
    self.operator_approvals[msg.sender][operator] = approved
    log ApprovalForAll(msg.sender, operator, approved)

@external
def isApprovedForAll(owner: address, operator: address) -> bool:
    return self.operator_approvals[owner][operator]

@external
def withdraw():
    assert msg.sender == self.owner, "Not owner"
    send(self.owner, self.balance)
```

---

## 6. Complete Tests {#tests}

```python
# tests/test_merkle_airdrop.py
import pytest
import boa
from eth_abi import encode
from eth_utils import keccak

def keccak256(data: bytes) -> bytes:
    return keccak(data)

def hash_leaf(address: str, amount: int) -> bytes:
    """Hash leaf node เหมือนกับใน contract"""
    # pad address to 32 bytes and amount to 32 bytes
    addr_bytes = bytes.fromhex(address[2:]).rjust(32, b'\x00')
    amount_bytes = amount.to_bytes(32, 'big')
    return keccak256(keccak256(addr_bytes + amount_bytes))

class SimpleMerkleTree:
    """Simple Merkle Tree implementation สำหรับ tests"""
    
    def __init__(self, leaves_data: list):
        self.leaves_data = leaves_data
        self.leaf_hashes = [
            hash_leaf(addr, amount) for addr, amount in leaves_data
        ]
        self.layers = self._build()
    
    def _build(self):
        current = list(self.leaf_hashes)
        if len(current) % 2 != 0:
            current.append(current[-1])
        
        layers = [current]
        while len(current) > 1:
            next_layer = []
            for i in range(0, len(current), 2):
                l, r = current[i], current[i+1] if i+1 < len(current) else current[i]
                if l <= r:
                    parent = keccak256(l + r)
                else:
                    parent = keccak256(r + l)
                next_layer.append(parent)
            
            if len(next_layer) % 2 != 0 and len(next_layer) > 1:
                next_layer.append(next_layer[-1])
            
            layers.append(next_layer)
            current = next_layer
        
        return layers
    
    @property
    def root(self):
        return self.layers[-1][0]
    
    def get_proof(self, index: int):
        proof = []
        idx = index
        for layer in self.layers[:-1]:
            sibling = idx + 1 if idx % 2 == 0 else idx - 1
            if sibling < len(layer):
                proof.append(layer[sibling])
            idx //= 2
        return proof


# Test data
RECIPIENTS = [
    ("0x1111111111111111111111111111111111111111", 100 * 10**18),
    ("0x2222222222222222222222222222222222222222", 200 * 10**18),
    ("0x3333333333333333333333333333333333333333", 150 * 10**18),
    ("0x4444444444444444444444444444444444444444", 300 * 10**18),
]


@pytest.fixture
def deployer():
    return boa.env.generate_address()

@pytest.fixture
def tree():
    return SimpleMerkleTree(RECIPIENTS)

@pytest.fixture
def token(deployer):
    with boa.env.prank(deployer):
        return boa.load(
            "contracts/Token.vy",
            "Airdrop Token",
            "ADR",
            10**9 * 10**18
        )

@pytest.fixture
def airdrop(deployer, token, tree):
    with boa.env.prank(deployer):
        # Mint tokens to deployer
        token.mint(deployer, 10**6 * 10**18)
        
        # Deploy airdrop
        contract = boa.load(
            "contracts/MerkleAirdrop.vy",
            token.address,
            tree.root,
            30  # 30 days
        )
        
        # Fund airdrop contract
        total = sum(amount for _, amount in RECIPIENTS)
        token.transfer(contract.address, total)
    
    return contract


class TestMerkleAirdrop:
    def test_merkle_root_set(self, airdrop, tree):
        assert airdrop.merkle_root() == tree.root
    
    def test_valid_claim(self, airdrop, tree):
        # Recipient 0 claims
        recipient_addr = RECIPIENTS[0][0]
        recipient_amount = RECIPIENTS[0][1]
        proof = tree.get_proof(0)
        
        # Set boa to use recipient's address
        boa.env.alias(recipient_addr, "recipient_0")
        
        can_claim = airdrop.can_claim(recipient_addr, recipient_amount, proof)
        assert can_claim == True
    
    def test_invalid_proof(self, airdrop, tree):
        recipient_addr = RECIPIENTS[0][0]
        recipient_amount = RECIPIENTS[0][1]
        
        # ใช้ proof ผิด
        wrong_proof = [b'\x00' * 32]
        
        can_claim = airdrop.can_claim(recipient_addr, recipient_amount, wrong_proof)
        assert can_claim == False
    
    def test_wrong_amount(self, airdrop, tree):
        recipient_addr = RECIPIENTS[0][0]
        wrong_amount = 999 * 10**18  # Amount ไม่ตรง
        proof = tree.get_proof(0)
        
        can_claim = airdrop.can_claim(recipient_addr, wrong_amount, proof)
        assert can_claim == False
    
    def test_all_recipients_can_claim(self, airdrop, tree):
        for i, (addr, amount) in enumerate(RECIPIENTS):
            proof = tree.get_proof(i)
            can_claim = airdrop.can_claim(addr, amount, proof)
            assert can_claim == True, f"Recipient {i} cannot claim"
    
    def test_double_claim_prevented(self, airdrop, tree, token):
        recipient_addr = RECIPIENTS[1][0]
        recipient_amount = RECIPIENTS[1][1]
        proof = tree.get_proof(1)
        
        # Set balance for recipient
        boa.env.set_balance(recipient_addr, 10**18)
        
        with boa.env.prank(recipient_addr):
            airdrop.claim(recipient_amount, proof)
        
        # ไม่สามารถ claim ซ้ำได้
        with pytest.raises(Exception, match="Already claimed"):
            with boa.env.prank(recipient_addr):
                airdrop.claim(recipient_amount, proof)
    
    def test_after_deadline(self, airdrop, tree):
        recipient_addr = RECIPIENTS[0][0]
        recipient_amount = RECIPIENTS[0][1]
        proof = tree.get_proof(0)
        
        # Fast forward past deadline
        boa.env.time_travel(seconds=31 * 86400)
        
        can_claim = airdrop.can_claim(recipient_addr, recipient_amount, proof)
        assert can_claim == False
    
    def test_is_claimed_after_claim(self, airdrop, tree, token):
        recipient_addr = RECIPIENTS[2][0]
        recipient_amount = RECIPIENTS[2][1]
        proof = tree.get_proof(2)
        
        boa.env.set_balance(recipient_addr, 10**18)
        
        assert airdrop.is_claimed(recipient_addr) == False
        
        with boa.env.prank(recipient_addr):
            airdrop.claim(recipient_amount, proof)
        
        assert airdrop.is_claimed(recipient_addr) == True
    
    def test_update_root(self, airdrop, deployer, tree):
        new_recipients = [
            ("0x5555555555555555555555555555555555555555", 500 * 10**18),
        ]
        new_tree = SimpleMerkleTree(new_recipients)
        
        with boa.env.prank(deployer):
            airdrop.update_root(new_tree.root)
        
        assert airdrop.merkle_root() == new_tree.root
    
    def test_recover_unclaimed(self, airdrop, deployer, token, tree):
        # Fast forward past deadline
        boa.env.time_travel(seconds=31 * 86400)
        
        balance_before = token.balanceOf(deployer)
        contract_balance = token.balanceOf(airdrop.address)
        
        with boa.env.prank(deployer):
            airdrop.recover_unclaimed()
        
        assert token.balanceOf(deployer) == balance_before + contract_balance
        assert token.balanceOf(airdrop.address) == 0


class TestMerkleWhitelist:
    """ทดสอบ Whitelist Minting ด้วย Merkle Tree"""
    
    @pytest.fixture
    def whitelist_accounts(self):
        return [boa.env.generate_address() for _ in range(5)]
    
    @pytest.fixture
    def whitelist_tree(self, whitelist_accounts):
        """Whitelist tree (ใช้แค่ address ไม่มี amount)"""
        class WhitelistTree:
            def __init__(self, accounts):
                self.accounts = accounts
                self.leaf_hashes = [
                    keccak256(keccak256(
                        bytes.fromhex(acc[2:]).rjust(32, b'\x00')
                    ))
                    for acc in accounts
                ]
                self.layers = self._build()
            
            def _build(self):
                current = list(self.leaf_hashes)
                if len(current) % 2 != 0:
                    current.append(current[-1])
                
                layers = [current]
                while len(current) > 1:
                    next_layer = []
                    for i in range(0, len(current), 2):
                        l, r = current[i], current[i+1]
                        parent = keccak256(l + r) if l <= r else keccak256(r + l)
                        next_layer.append(parent)
                    if len(next_layer) % 2 != 0 and len(next_layer) > 1:
                        next_layer.append(next_layer[-1])
                    layers.append(next_layer)
                    current = next_layer
                
                return layers
            
            @property
            def root(self):
                return self.layers[-1][0]
            
            def get_proof(self, index):
                proof = []
                idx = index
                for layer in self.layers[:-1]:
                    sibling = idx + 1 if idx % 2 == 0 else idx - 1
                    if sibling < len(layer):
                        proof.append(layer[sibling])
                    idx //= 2
                return proof
        
        return WhitelistTree(whitelist_accounts)
    
    @pytest.fixture
    def nft(self, deployer, whitelist_tree):
        mint_price = int(0.05 * 10**18)
        
        with boa.env.prank(deployer):
            contract = boa.load(
                "contracts/MerkleWhitelist.vy",
                "Merkle NFT",
                "MNFT",
                "ipfs://hash/",
                whitelist_tree.root,
                mint_price,
                int(0.04 * 10**18)  # 20% discount for whitelist
            )
        
        return contract
    
    def test_whitelist_mint(self, nft, deployer, whitelist_accounts, whitelist_tree):
        with boa.env.prank(deployer):
            nft.setSaleState(True, False)
        
        minter = whitelist_accounts[0]
        boa.env.set_balance(minter, 10**18)
        proof = whitelist_tree.get_proof(0)
        
        with boa.env.prank(minter):
            nft.whitelistMint(2, proof, value=int(0.04 * 10**18) * 2)
        
        assert nft.balanceOf(minter) == 2
        assert nft.whitelist_minted(minter) == 2
    
    def test_non_whitelist_cannot_mint(self, nft, deployer, whitelist_tree):
        with boa.env.prank(deployer):
            nft.setSaleState(True, False)
        
        non_whitelisted = boa.env.generate_address()
        boa.env.set_balance(non_whitelisted, 10**18)
        
        # ใช้ proof ของคนอื่น
        fake_proof = whitelist_tree.get_proof(0)
        
        with pytest.raises(Exception):
            with boa.env.prank(non_whitelisted):
                nft.whitelistMint(1, fake_proof, value=int(0.04 * 10**18))


if __name__ == "__main__":
    pytest.main([__file__, "-v"])
```

---

## สรุป

### Merkle Tree Use Cases ใน Smart Contracts

1. **Airdrop**: แจก tokens ให้ eligible addresses โดยไม่ต้องเก็บทุก address ใน storage
2. **Whitelist**: NFT whitelist minting
3. **Vote Eligibility**: ตรวจสอบว่า address มีสิทธิ์โหวต
4. **Allowlist**: จัดการ permissions

### Security Considerations

1. **Double Hash**: ใช้ `keccak256(keccak256(data))` ป้องกัน second preimage attack
2. **Sort Pairs**: sort sibling pairs ก่อน hash ป้องกัน order manipulation
3. **Deadline**: ตั้ง deadline สำหรับ claim เพื่อ recover unclaimed tokens
4. **Reentrancy**: Mark claimed ก่อน transfer

### Gas Savings

```
Traditional mapping (10,000 addresses):
- Deploy: ~200M gas
- Add address: ~22,000 gas each

Merkle Tree (10,000 addresses):
- Deploy: ~22,000 gas (just root)
- Update root: ~22,000 gas
- Verify proof: ~5,000-50,000 gas per claim
- Savings: 99.9% on storage!
```
