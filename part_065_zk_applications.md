# Part 065: ZK Proof Applications - Privacy and Verification On-Chain

## สารบัญ (Table of Contents)
1. บทนำ Zero-Knowledge Proofs
2. Groth16 Verifier
3. Private Voting System
4. Anonymous Credentials
5. ZK-Based Token Transfer (Privacy)
6. Tests

---

## 1. บทนำ Zero-Knowledge Proofs

**ZK Proof คืออะไร:**
- Prove ว่า "รู้ข้อมูลบางอย่าง" โดยไม่เปิดเผยข้อมูลนั้น
- ตัวอย่าง: Prove ว่าอายุ >= 18 โดยไม่บอกอายุจริง

**ประเภท ZK Systems:**
- **Groth16**: Efficient, small proof, trusted setup required
- **PLONK**: Universal setup, slightly larger proof
- **STARKs**: No trusted setup, quantum resistant, large proof

**On-chain ZK Use Cases:**
- Private voting
- Anonymous credentials
- Rollups (zkSync, StarkNet)
- Private DeFi

---

## 2. Groth16 Verifier

```vyper
# @version 0.4.0
# contracts/Groth16Verifier.vy
# Groth16 ZK-SNARK verification บน Ethereum
#
# Groth16 proof = (A, B, C) + public inputs
# Verify: e(A, B) = e(alpha, beta) * e(sum(inputs * gamma_abc), gamma) * e(C, delta)

# Pairing library interfaces (precompiles)
# BN128 curve operations available as precompiles

# Structs
struct G1Point:
    x: uint256
    y: uint256

struct G2Point:
    x: uint256[2]
    y: uint256[2]

struct Proof:
    a: G1Point
    b: G2Point
    c: G1Point

# Verification Key (set at deploy time from circuit)
struct VerifyingKey:
    alpha1: G1Point
    beta2: G2Point
    gamma2: G2Point
    delta2: G2Point
    ic: DynArray[G1Point, 64]  # IC[0] + IC[i] * input[i-1]

# State
verifyingKey: VerifyingKey
governance: public(address)
circuitId: public(bytes32)  # identifier ของ circuit

PRIME_Q: constant(uint256) = 21888242871839275222246405745257275088696311157297823662689037894645226208583

@deploy
def __init__(
    _governance: address,
    _circuitId: bytes32,
    alpha1_x: uint256, alpha1_y: uint256,
    beta2_x0: uint256, beta2_x1: uint256, beta2_y0: uint256, beta2_y1: uint256,
    gamma2_x0: uint256, gamma2_x1: uint256, gamma2_y0: uint256, gamma2_y1: uint256,
    delta2_x0: uint256, delta2_x1: uint256, delta2_y0: uint256, delta2_y1: uint256
):
    """
    Initialize verifier พร้อม verification key จาก circuit
    
    VK ได้จากการ compile circuit ด้วย snarkjs/circom
    """
    self.governance = _governance
    self.circuitId = _circuitId
    
    self.verifyingKey.alpha1 = G1Point({x: alpha1_x, y: alpha1_y})
    self.verifyingKey.beta2 = G2Point({
        x: [beta2_x0, beta2_x1],
        y: [beta2_y0, beta2_y1]
    })
    self.verifyingKey.gamma2 = G2Point({
        x: [gamma2_x0, gamma2_x1],
        y: [gamma2_y0, gamma2_y1]
    })
    self.verifyingKey.delta2 = G2Point({
        x: [delta2_x0, delta2_x1],
        y: [delta2_y0, delta2_y1]
    })

@external
def addIC(ic_x: uint256, ic_y: uint256):
    """เพิ่ม IC points ทีละตัว (called after deploy)"""
    assert msg.sender == self.governance
    self.verifyingKey.ic.append(G1Point({x: ic_x, y: ic_y}))

@internal
def _negate(p: G1Point) -> G1Point:
    """Negate G1 point"""
    if p.x == 0 and p.y == 0:
        return G1Point({x: 0, y: 0})
    return G1Point({x: p.x, y: PRIME_Q - (p.y % PRIME_Q)})

@internal
def _addition(p1: G1Point, p2: G1Point) -> G1Point:
    """Add two G1 points using bn128 precompile (0x06)"""
    input_data: Bytes[128] = abi.encode(p1.x, p1.y, p2.x, p2.y)
    result: Bytes[64] = b""
    success: bool = False
    result, success = raw_call(
        convert(6, address),  # bn128add precompile
        input_data,
        max_outsize=64,
        is_static_call=True,
        revert_on_failure=False
    )
    assert success, "G1 addition failed"
    x: uint256 = convert(slice(result, 0, 32), uint256)
    y: uint256 = convert(slice(result, 32, 32), uint256)
    return G1Point({x: x, y: y})

@internal
def _scalar_mul(p: G1Point, scalar: uint256) -> G1Point:
    """Scalar multiply G1 point by scalar using bn128 precompile (0x07)"""
    input_data: Bytes[96] = abi.encode(p.x, p.y, scalar)
    result: Bytes[64] = b""
    success: bool = False
    result, success = raw_call(
        convert(7, address),  # bn128mul precompile
        input_data,
        max_outsize=64,
        is_static_call=True,
        revert_on_failure=False
    )
    assert success, "G1 scalar mul failed"
    x: uint256 = convert(slice(result, 0, 32), uint256)
    y: uint256 = convert(slice(result, 32, 32), uint256)
    return G1Point({x: x, y: y})

@internal
def _pairing(p1: DynArray[G1Point, 4], p2: DynArray[G2Point, 4]) -> bool:
    """
    Multi-pairing check using bn128 precompile (0x08)
    Returns true if e(p1[0], p2[0]) * ... * e(p1[n], p2[n]) == 1
    """
    assert len(p1) == len(p2), "Length mismatch"
    
    # Pack all points into single bytes
    input_size: uint256 = convert(len(p1), uint256) * 192
    input_data: Bytes[768] = b""  # Max 4 pairs = 768 bytes
    
    for i: uint256 in range(4):
        if i >= len(p1):
            break
        # G1: 64 bytes (x, y)
        # G2: 128 bytes (x[0], x[1], y[0], y[1])
        # Note: G2 x/y coordinates are swapped for the precompile
        input_data = concat(
            input_data,
            abi.encode(p1[i].x, p1[i].y),
            abi.encode(
                p2[i].x[1], p2[i].x[0],
                p2[i].y[1], p2[i].y[0]
            )
        )
    
    result: Bytes[32] = b""
    success: bool = False
    result, success = raw_call(
        convert(8, address),  # bn128pairing precompile
        input_data,
        max_outsize=32,
        is_static_call=True,
        revert_on_failure=False
    )
    
    assert success, "Pairing failed"
    
    return convert(result, uint256) == 1

@external
@view
def verifyProof(
    proof: Proof,
    inputs: DynArray[uint256, 32]
) -> bool:
    """
    Verify Groth16 proof
    
    Algorithm:
    1. Compute vk_x = IC[0] + sum(IC[i] * inputs[i-1])
    2. Check pairing: e(A, B) = e(alpha, beta) * e(vk_x, gamma) * e(C, delta)
    
    Parameters:
        proof: ZK proof (A, B, C)
        inputs: public inputs ของ circuit
    
    Returns: true ถ้า proof valid
    """
    vk: VerifyingKey = self.verifyingKey
    
    assert len(inputs) + 1 == len(vk.ic), "Invalid inputs length"
    
    # ตรวจว่า inputs อยู่ใน field
    for inp: uint256 in inputs:
        assert inp < PRIME_Q, "Input out of field"
    
    # คำนวณ vk_x = IC[0] + sum(IC[i+1] * inputs[i])
    vk_x: G1Point = vk.ic[0]
    for i: uint256 in range(32):
        if i >= len(inputs):
            break
        vk_x = self._addition(vk_x, self._scalar_mul(vk.ic[i + 1], inputs[i]))
    
    # Check pairing equation
    # e(A, B) * e(-alpha, beta) * e(-vk_x, gamma) * e(-C, delta) == 1
    return self._pairing(
        [self._negate(proof.a), vk.alpha1, vk_x, proof.c],
        [proof.b, vk.beta2, vk.gamma2, vk.delta2]
    )
```

---

## 3. Private Voting System

```vyper
# @version 0.4.0
# contracts/PrivateVoting.vy
# Anonymous voting ด้วย ZK proofs
#
# Users can vote without revealing their identity
# Only prove they have voting rights

from vyper.interfaces import ERC20

interface IZKVerifier:
    def verifyProof(
        proof_a: uint256[2],
        proof_b: uint256[2][2],
        proof_c: uint256[2],
        inputs: DynArray[uint256, 10]
    ) -> bool: view

# Events
event ProposalCreated:
    proposalId: indexed(uint256)
    creator: indexed(address)
    description: String[256]
    endTime: uint256

event VoteCast:
    proposalId: indexed(uint256)
    nullifier: indexed(bytes32)
    support: bool

event ProposalExecuted:
    proposalId: indexed(uint256)
    passed: bool

struct Proposal:
    description: String[256]
    endTime: uint256
    forVotes: uint256
    againstVotes: uint256
    executed: bool
    merkleRoot: bytes32  # Merkle root ของ eligible voters

# State
proposals: HashMap[uint256, Proposal]
nextProposalId: uint256

# Nullifiers: ป้องกัน double voting
usedNullifiers: HashMap[bytes32, bool]

verifier: public(address)
governance: public(address)

# Voter registration (Merkle tree managed off-chain)
voterMerkleRoots: HashMap[uint256, bytes32]  # proposalId -> merkle root

@deploy
def __init__(_verifier: address, _governance: address):
    self.verifier = _verifier
    self.governance = _governance

@external
def createProposal(
    description: String[256],
    votingPeriod: uint256,
    voterMerkleRoot: bytes32
) -> uint256:
    """
    สร้าง proposal ใหม่
    
    Parameters:
        voterMerkleRoot: Merkle root ของ list ผู้มีสิทธิ์โหวต
                        (คำนวณ off-chain จาก snapshot ของ token holders)
    """
    assert msg.sender == self.governance, "Not governance"
    
    proposalId: uint256 = self.nextProposalId
    self.nextProposalId += 1
    
    endTime: uint256 = block.timestamp + votingPeriod
    
    self.proposals[proposalId] = Proposal({
        description: description,
        endTime: endTime,
        forVotes: 0,
        againstVotes: 0,
        executed: False,
        merkleRoot: voterMerkleRoot
    })
    
    self.voterMerkleRoots[proposalId] = voterMerkleRoot
    
    log ProposalCreated(proposalId, msg.sender, description, endTime)
    
    return proposalId

@external
def castVoteWithProof(
    proposalId: uint256,
    support: bool,
    nullifier: bytes32,
    proof_a: uint256[2],
    proof_b: uint256[2][2],
    proof_c: uint256[2]
) -> bool:
    """
    ลงคะแนนแบบ anonymous ด้วย ZK proof
    
    ZK Circuit พิสูจน์ว่า:
    1. ผู้โหวตอยู่ใน voter Merkle tree (มีสิทธิ์โหวต)
    2. nullifier ถูกคำนวณจาก secret ของผู้โหวต (ป้องกัน double vote)
    3. ผู้โหวตเลือก support หรือ against (public input)
    
    Public inputs:
    - proposalId
    - merkleRoot
    - nullifier (hash ของ secret, ป้องกัน double voting)
    - support (0 or 1)
    
    Private inputs (ไม่เปิดเผย):
    - voterSecret
    - merkleProof (path จาก leaf ไป root)
    
    Parameters:
        nullifier: unique identifier สำหรับ vote นี้ (ป้องกัน double vote)
        proof: ZK proof
    """
    proposal: Proposal = self.proposals[proposalId]
    
    assert block.timestamp < proposal.endTime, "Voting ended"
    assert not self.usedNullifiers[nullifier], "Already voted"
    
    # สร้าง public inputs สำหรับ verifier
    supportUint: uint256 = 1 if support else 0
    
    inputs: DynArray[uint256, 10] = [
        proposalId,
        convert(proposal.merkleRoot, uint256),
        convert(nullifier, uint256),
        supportUint
    ]
    
    # Verify ZK proof
    isValid: bool = IZKVerifier(self.verifier).verifyProof(
        proof_a, proof_b, proof_c, inputs
    )
    assert isValid, "Invalid ZK proof"
    
    # บันทึก nullifier
    self.usedNullifiers[nullifier] = True
    
    # นับคะแนน
    if support:
        self.proposals[proposalId].forVotes += 1
    else:
        self.proposals[proposalId].againstVotes += 1
    
    log VoteCast(proposalId, nullifier, support)
    
    return True

@external
def executeProposal(proposalId: uint256) -> bool:
    """Execute proposal หลัง voting period"""
    proposal: Proposal = self.proposals[proposalId]
    
    assert block.timestamp >= proposal.endTime, "Still voting"
    assert not proposal.executed, "Already executed"
    
    self.proposals[proposalId].executed = True
    
    passed: bool = proposal.forVotes > proposal.againstVotes
    
    log ProposalExecuted(proposalId, passed)
    
    return passed

@external
@view
def getProposal(proposalId: uint256) -> Proposal:
    return self.proposals[proposalId]
```

---

## 4. Anonymous Credentials

```vyper
# @version 0.4.0
# contracts/AnonymousCredentials.vy
# Verify credentials (KYC, age, etc.) โดยไม่เปิดเผยข้อมูล

interface IZKVerifier:
    def verifyProof(
        proof_a: uint256[2],
        proof_b: uint256[2][2],
        proof_c: uint256[2],
        inputs: DynArray[uint256, 10]
    ) -> bool: view

# Events
event CredentialIssued:
    commitmentHash: indexed(bytes32)
    credentialType: indexed(bytes32)
    issuer: indexed(address)

event CredentialRevoked:
    commitmentHash: indexed(bytes32)

event CredentialVerified:
    nullifier: indexed(bytes32)
    credentialType: indexed(bytes32)
    action: String[64]

# Credential types
KYC_VERIFIED: constant(bytes32) = keccak256("KYC_VERIFIED")
AGE_18_PLUS: constant(bytes32) = keccak256("AGE_18_PLUS")
ACCREDITED_INVESTOR: constant(bytes32) = keccak256("ACCREDITED_INVESTOR")

# State
# Credential commitments: issuer posts commitment = hash(secret + userData)
credentialCommitments: HashMap[bytes32, bool]
credentialTypes: HashMap[bytes32, bytes32]  # commitment -> type
issuers: HashMap[address, bool]

# Revoked credentials
revokedCredentials: HashMap[bytes32, bool]

# Used nullifiers (per action type)
usedNullifiers: HashMap[bytes32, HashMap[bytes32, bool]]  # action -> nullifier -> used

verifier: public(address)
governance: public(address)

# Access-controlled actions
actionVerifiers: HashMap[bytes32, address]  # action -> specific verifier

@deploy
def __init__(_verifier: address, _governance: address):
    self.verifier = _verifier
    self.governance = _governance

@external
def addIssuer(issuer: address):
    assert msg.sender == self.governance
    self.issuers[issuer] = True

@external
def issueCredential(
    commitmentHash: bytes32,
    credentialType: bytes32
):
    """
    Issuer ออก credential
    
    commitmentHash = Poseidon(userSecret, credentialData, issuerSecret)
    
    ผู้ใช้รู้ userSecret และสามารถ prove ว่ามี commitment นี้
    โดยไม่เปิดเผย userSecret หรือ credentialData
    
    Parameters:
        commitmentHash: Pedersen/Poseidon hash ของ credential
        credentialType: ประเภท credential (KYC, AGE, etc.)
    """
    assert self.issuers[msg.sender], "Not issuer"
    assert not self.credentialCommitments[commitmentHash], "Already issued"
    
    self.credentialCommitments[commitmentHash] = True
    self.credentialTypes[commitmentHash] = credentialType
    
    log CredentialIssued(commitmentHash, credentialType, msg.sender)

@external
def revokeCredential(commitmentHash: bytes32):
    """Issuer revoke credential"""
    assert self.issuers[msg.sender], "Not issuer"
    self.revokedCredentials[commitmentHash] = True
    log CredentialRevoked(commitmentHash)

@external
def verifyCredential(
    credentialType: bytes32,
    action: String[64],
    nullifier: bytes32,
    proof_a: uint256[2],
    proof_b: uint256[2][2],
    proof_c: uint256[2],
    publicCommitmentRoot: uint256  # Merkle root ของ valid commitments
) -> bool:
    """
    ตรวจสอบว่าผู้ใช้มี credential ที่ต้องการ
    
    ZK Circuit พิสูจน์ว่า:
    1. รู้ secret ที่สอดคล้องกับ commitment ใน Merkle tree
    2. commitment ไม่ถูก revoke
    3. nullifier = hash(secret, action) เพื่อป้องกัน reuse
    
    Public inputs:
    - credentialType
    - action hash
    - nullifier
    - commitmentRoot (current Merkle root)
    
    Private inputs:
    - userSecret
    - credential commitment
    - Merkle proof
    
    Parameters:
        nullifier: unique ID สำหรับ (user, action) pair
        publicCommitmentRoot: current root ของ credential Merkle tree
    """
    actionHash: bytes32 = keccak256(convert(action, Bytes[64]))
    actionNullifierKey: bytes32 = keccak256(abi.encode(actionHash, nullifier))
    
    assert not self.usedNullifiers[actionHash][nullifier], "Already used"
    
    inputs: DynArray[uint256, 10] = [
        convert(credentialType, uint256),
        convert(actionHash, uint256),
        convert(nullifier, uint256),
        publicCommitmentRoot
    ]
    
    isValid: bool = IZKVerifier(self.verifier).verifyProof(
        proof_a, proof_b, proof_c, inputs
    )
    assert isValid, "Invalid proof"
    
    self.usedNullifiers[actionHash][nullifier] = True
    
    log CredentialVerified(nullifier, credentialType, action)
    
    return True
```

---

## 5. ZK-Protected Token Transfer

```vyper
# @version 0.4.0
# contracts/PrivateTransfer.vy
# Tornado Cash-like privacy pool (educational)
# Note: อาจมีข้อจำกัดทางกฎหมาย - ใช้เพื่อการศึกษาเท่านั้น

interface IZKVerifier:
    def verifyProof(
        proof_a: uint256[2],
        proof_b: uint256[2][2],
        proof_c: uint256[2],
        inputs: DynArray[uint256, 10]
    ) -> bool: view

# Events
event Deposit:
    commitment: indexed(bytes32)
    leafIndex: uint256
    timestamp: uint256

event Withdrawal:
    nullifier: indexed(bytes32)
    to: indexed(address)
    relayer: address
    fee: uint256

# Merkle tree for commitments
TREE_DEPTH: constant(uint256) = 20
FIELD_SIZE: constant(uint256) = 21888242871839275222246405745257275088548364400416034343698204186575808495617

struct MerkleTree:
    leaves: DynArray[bytes32, 1048576]  # 2^20 leaves
    nextLeaf: uint256
    roots: DynArray[bytes32, 100]       # history ของ roots
    currentRoot: bytes32

# State
verifier: public(address)
denomination: public(uint256)   # fixed amount (e.g. 1 ETH)
token: public(address)          # ERC20 token (empty = native ETH)

tree: MerkleTree
nullifiers: HashMap[bytes32, bool]

# Zeros for empty tree levels
zeros: DynArray[bytes32, 21]

governance: public(address)

@deploy
def __init__(
    _verifier: address,
    _denomination: uint256,
    _token: address,
    _governance: address
):
    self.verifier = _verifier
    self.denomination = _denomination
    self.token = _token
    self.governance = _governance
    
    # Pre-compute zero hashes
    zero: bytes32 = keccak256(b"\x00")
    self.zeros.append(zero)
    for i: uint256 in range(20):
        zero = keccak256(abi.encode(zero, zero))
        self.zeros.append(zero)
    
    self.tree.currentRoot = self.zeros[TREE_DEPTH]

@external
@payable
def deposit(commitment: bytes32):
    """
    ฝาก token + commitment hash เข้า pool
    
    commitment = Poseidon(nullifier, secret)
    
    ผู้ฝากเก็บ nullifier และ secret ไว้
    ไม่มีใครรู้ว่า commitment ไหนเป็นของใคร
    
    Parameters:
        commitment: Pedersen commitment ของ (nullifier, secret)
    """
    if self.token == empty(address):
        assert msg.value == self.denomination, "Wrong ETH amount"
    else:
        assert msg.value == 0
        from vyper.interfaces import ERC20
        assert ERC20(self.token).transferFrom(msg.sender, self, self.denomination), "Transfer failed"
    
    # เพิ่ม commitment เข้า Merkle tree
    leafIndex: uint256 = self.tree.nextLeaf
    self.tree.leaves.append(commitment)
    self.tree.nextLeaf += 1
    
    # อัพเดท root
    self._updateRoot(commitment, leafIndex)
    
    log Deposit(commitment, leafIndex, block.timestamp)

@external
def withdraw(
    recipient: address,
    relayer: address,
    fee: uint256,
    nullifier: bytes32,
    root: bytes32,
    proof_a: uint256[2],
    proof_b: uint256[2][2],
    proof_c: uint256[2]
) -> bool:
    """
    ถอน token โดยไม่เปิดเผย identity
    
    ZK Circuit พิสูจน์ว่า:
    1. รู้ (nullifier, secret) ที่ hash เป็น commitment ใน tree
    2. commitment อยู่ใน Merkle tree (root ที่ระบุ)
    3. nullifier ยังไม่ถูกใช้
    
    Public inputs:
    - root (Merkle root ณ เวลา deposit)
    - nullifier hash
    - recipient
    - fee
    
    Private inputs:
    - nullifier
    - secret
    - Merkle proof (path ไป root)
    
    Parameters:
        relayer: relayer ที่ส่ง tx (รับ fee)
        nullifier: nullifier ของ deposit
        root: Merkle root ที่ commitment อยู่ใน
    """
    assert not self.nullifiers[nullifier], "Already withdrawn"
    assert self._isKnownRoot(root), "Unknown root"
    assert fee <= self.denomination, "Fee too high"
    
    inputs: DynArray[uint256, 10] = [
        convert(root, uint256),
        convert(nullifier, uint256),
        convert(recipient, uint256),
        fee
    ]
    
    isValid: bool = IZKVerifier(self.verifier).verifyProof(
        proof_a, proof_b, proof_c, inputs
    )
    assert isValid, "Invalid proof"
    
    self.nullifiers[nullifier] = True
    
    payout: uint256 = self.denomination - fee
    
    if self.token == empty(address):
        send(recipient, payout)
        if fee > 0:
            send(relayer, fee)
    else:
        from vyper.interfaces import ERC20
        assert ERC20(self.token).transfer(recipient, payout), "Transfer failed"
        if fee > 0:
            assert ERC20(self.token).transfer(relayer, fee), "Fee failed"
    
    log Withdrawal(nullifier, recipient, relayer, fee)
    
    return True

@internal
def _updateRoot(leaf: bytes32, index: uint256):
    """อัพเดท Merkle root หลัง deposit"""
    currentHash: bytes32 = leaf
    currentIndex: uint256 = index
    
    for level: uint256 in range(TREE_DEPTH):
        if currentIndex % 2 == 0:
            # Left node
            sibling: bytes32 = self.zeros[level]
            if currentIndex + 1 < len(self.tree.leaves):
                # sibling exists - would need full tree storage for real impl
                pass
            currentHash = keccak256(abi.encode(currentHash, sibling))
        else:
            # Right node
            sibling: bytes32 = self.zeros[level]
            currentHash = keccak256(abi.encode(sibling, currentHash))
        
        currentIndex = currentIndex / 2
    
    self.tree.currentRoot = currentHash
    self.tree.roots.append(currentHash)

@internal
@view
def _isKnownRoot(root: bytes32) -> bool:
    """ตรวจว่า root เคยเป็น valid root"""
    if root == self.tree.currentRoot:
        return True
    for r: bytes32 in self.tree.roots:
        if r == root:
            return True
    return False

@external
@view
def getRoot() -> bytes32:
    return self.tree.currentRoot

@external
@view
def isNullifierUsed(nullifier: bytes32) -> bool:
    return self.nullifiers[nullifier]
```

---

## 6. Circuit Example (Circom)

```circom
// circuits/membership.circom
// Prove membership ใน Merkle tree โดยไม่เปิดเผย leaf

pragma circom 2.0.0;

include "circomlib/circuits/poseidon.circom";
include "circomlib/circuits/mux1.circom";

// Merkle tree membership proof
template MembershipProof(levels) {
    // Private inputs (ไม่เปิดเผย)
    signal input secret;
    signal input nullifier;
    signal input pathElements[levels];  // Merkle proof elements
    signal input pathIndices[levels];   // 0 = left, 1 = right
    
    // Public inputs
    signal input root;
    signal input nullifierHash;
    signal input recipient;
    signal input fee;
    
    // Compute commitment = Poseidon(nullifier, secret)
    component commitmentHasher = Poseidon(2);
    commitmentHasher.inputs[0] <== nullifier;
    commitmentHasher.inputs[1] <== secret;
    signal commitment <== commitmentHasher.out;
    
    // Compute nullifier hash = Poseidon(nullifier)
    component nullifierHasher = Poseidon(1);
    nullifierHasher.inputs[0] <== nullifier;
    nullifierHasher.out === nullifierHash;  // ต้องตรงกับ public input
    
    // Verify Merkle proof
    component hashers[levels];
    component muxes[levels];
    
    signal levelHashes[levels + 1];
    levelHashes[0] <== commitment;
    
    for (var i = 0; i < levels; i++) {
        hashers[i] = Poseidon(2);
        muxes[i] = MultiMux1(2);
        
        muxes[i].c[0][0] <== levelHashes[i];
        muxes[i].c[0][1] <== pathElements[i];
        muxes[i].c[1][0] <== pathElements[i];
        muxes[i].c[1][1] <== levelHashes[i];
        muxes[i].s <== pathIndices[i];
        
        hashers[i].inputs[0] <== muxes[i].out[0];
        hashers[i].inputs[1] <== muxes[i].out[1];
        levelHashes[i + 1] <== hashers[i].out;
    }
    
    // Root ต้องตรงกับ public input
    root === levelHashes[levels];
}

component main {public [root, nullifierHash, recipient, fee]} = MembershipProof(20);
```

---

## 7. Tests

```python
# tests/test_zk_applications.py
import pytest
from brownie import Groth16Verifier, PrivateVoting, AnonymousCredentials, accounts
from py_ecc.bn128 import G1, G2, add, multiply, pairing, neg, FQ, FQ2

@pytest.fixture
def setup():
    owner = accounts[0]
    alice = accounts[1]
    issuer = accounts[2]
    
    # In real tests, would use actual ZK verifier keys from snarkjs
    # For testing, use mock verifier
    
    return owner, alice, issuer

def test_groth16_verifier_structure(setup):
    """Test that verifier accepts valid proof structure"""
    owner, alice, issuer = setup
    
    # These would be real values from snarkjs trusted setup
    # For educational purposes, using dummy values
    alpha_x = 1
    alpha_y = 2
    
    # Note: Real deployment would use actual BN128 curve points
    print("ZK Verifier structure test - would use real circuit in production")

def test_private_voting_flow(setup):
    """
    Demonstrate private voting flow
    
    In real system:
    1. User computes: commitment = Poseidon(secret, voterAddress)
    2. User adds commitment to Merkle tree (off-chain)
    3. Admin creates proposal with Merkle root
    4. User generates ZK proof off-chain using snarkjs
    5. User submits proof on-chain
    """
    owner, alice, issuer = setup
    
    print("""
    Private Voting Flow:
    
    1. Setup Phase:
       - Compile circuit: circom membership.circom --r1cs --wasm
       - Generate trusted setup: snarkjs groth16 setup
       - Generate verification key
       
    2. Registration:
       - Each voter: secret = random(), nullifier = random()
       - commitment = Poseidon(nullifier, secret)
       - Submit commitment to off-chain tree
       
    3. Proposal:
       - Admin takes snapshot of commitments
       - Compute Merkle root
       - Create proposal with root on-chain
       
    4. Voting:
       - Voter creates witness: wasmFile, commitment, merkleProof
       - Generate proof: snarkjs groth16 prove
       - Submit proof on-chain with nullifier hash
       
    5. Execution:
       - Count votes from events
       - Execute based on result
    """)

def test_anonymous_credentials_flow(setup):
    """
    Demonstrate anonymous credential verification
    """
    owner, alice, issuer = setup
    
    print("""
    Anonymous Credentials Flow:
    
    1. KYC Process (off-chain):
       - User submits ID to KYC provider
       - KYC provider verifies identity
       
    2. Credential Issuance:
       - User generates: secret = random()
       - KYC provider computes: commitment = Poseidon(userAddress, kycData, secret)
       - Provider posts commitment on-chain
       - Provider gives user: credential = (commitment, merkleProof, secret)
       
    3. Credential Use (e.g., DeFi access):
       - User wants to use service requiring KYC
       - User generates ZK proof:
         * Knows secret for a commitment in the tree
         * commitment is not revoked
         * nullifier = Poseidon(secret, serviceAddress)
       - Submit proof on-chain
       - Service verifies without knowing user identity
       
    4. Revocation:
       - KYC provider revokes if fraud detected
       - User cannot generate new proofs for revoked commitment
    """)

def generate_sample_proof():
    """
    Show how to generate proof with snarkjs (Python integration)
    """
    import subprocess
    import json
    
    # 1. Compute witness
    witness_input = {
        "secret": "12345678",
        "nullifier": "87654321",
        "pathElements": ["0"] * 20,
        "pathIndices": [0] * 20,
        "root": "0",
        "nullifierHash": "0",
        "recipient": "0x0000000000000000000000000000000000000001",
        "fee": "0"
    }
    
    print("Sample proof generation with snarkjs:")
    print(f"Input: {json.dumps(witness_input, indent=2)}")
    print("""
    Commands:
    # 1. Compute witness
    node build/membership_js/generate_witness.js \\
        build/membership_js/membership.wasm \\
        input.json \\
        witness.wtns
    
    # 2. Generate proof
    snarkjs groth16 prove \\
        membership_0001.zkey \\
        witness.wtns \\
        proof.json \\
        public.json
    
    # 3. Verify locally  
    snarkjs groth16 verify \\
        verification_key.json \\
        public.json \\
        proof.json
    
    # 4. Generate Solidity/Vyper calldata
    snarkjs zkey export solidityverifier membership_0001.zkey verifier.sol
    snarkjs generatecall public.json proof.json
    """)
```

---

## 8. สรุป

### ZK Proof Concepts:

**1. สิ่งที่ ZK Proofs ทำได้:**
- Prove ownership ของ secret โดยไม่เปิดเผย
- Prove membership ใน set โดยไม่บอกว่าอยู่ตำแหน่งไหน
- Prove computation ถูกต้องโดยไม่แสดง input

**2. Groth16 vs PLONK vs STARKs:**
- Groth16: ขนาด proof เล็กสุด (~200 bytes), เร็ว verify, ต้อง trusted setup ต่อ circuit
- PLONK: Universal setup, ใหญ่กว่า Groth16 นิดหน่อย
- STARKs: ไม่ต้อง trusted setup, proof ใหญ่มาก (~100KB)

**3. Trusted Setup:**
- Groth16 ต้องการ ceremony สร้าง proving/verification keys
- Perpetual Powers of Tau: reusable setup สำหรับ phase 1
- Circuit-specific setup: phase 2 ต้องทำต่อ circuit

**4. ZK Applications ใน DeFi:**
- **zkRollups**: compress many txs into one proof (zkSync, StarkNet)
- **Private payments**: Tornado Cash model (regulatory issues)
- **Identity**: Proof of KYC without revealing PII
- **Governance**: Anonymous voting

**5. Nullifier Pattern:**
- สำคัญมากสำหรับ privacy-preserving systems
- nullifier = Poseidon(secret, context)
- ป้องกัน double-spend/double-vote
- ต้อง unique ต่อ (user, action) pair

**6. Merkle Trees ใน ZK:**
- เก็บ commitments อย่าง efficient
- Proof membership ด้วย O(log n) hashes
- Poseidon hash function efficient กว่า keccak256 ใน circuits
