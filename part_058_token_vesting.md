# Part 058: Token Vesting and Distribution

## สารบัญ (Table of Contents)
1. บทนำ Token Vesting
2. Linear Vesting
3. Cliff Vesting
4. Milestone Vesting
5. Revocable Grants
6. Vesting Factory
7. Tests

---

## 1. บทนำ Token Vesting

Token vesting คือกลไกที่ทำให้ tokens ถูก unlock ทีละน้อยตามเวลา:
- **ป้องกัน**: dump tokens ทันที
- **สร้าง**: alignment ระหว่าง team/investors กับ project
- **ประเภท**:
  - Linear: unlock ทุกวัน/เดือนสม่ำเสมอ
  - Cliff: ไม่ได้อะไรจนถึง cliff แล้วได้ทั้งก้อน
  - Cliff + Linear: cliff แล้วค่อยๆ unlock ต่อ
  - Milestone-based: unlock เมื่อทำ milestone สำเร็จ

---

## 2. Linear Vesting with Cliff

```vyper
# @version 0.4.0
# contracts/LinearVesting.vy
# Linear vesting พร้อม cliff period

from vyper.interfaces import ERC20

# Events
event VestingCreated:
    grantId: indexed(uint256)
    beneficiary: indexed(address)
    token: address
    totalAmount: uint256
    startTime: uint256
    cliffDuration: uint256
    vestingDuration: uint256

event TokensClaimed:
    grantId: indexed(uint256)
    beneficiary: indexed(address)
    amount: uint256

event VestingRevoked:
    grantId: indexed(uint256)
    beneficiary: indexed(address)
    refundAmount: uint256

event BeneficiaryChanged:
    grantId: indexed(uint256)
    oldBeneficiary: indexed(address)
    newBeneficiary: indexed(address)

# Struct
struct VestingGrant:
    beneficiary: address
    token: address
    totalAmount: uint256
    claimedAmount: uint256
    startTime: uint256
    cliffDuration: uint256   # seconds ก่อน tokens เริ่ม vest
    vestingDuration: uint256  # total vesting duration
    revocable: bool
    revoked: bool
    revokedAt: uint256
    grantedBy: address

# State
grants: HashMap[uint256, VestingGrant]
nextGrantId: uint256
beneficiaryGrants: HashMap[address, DynArray[uint256, 100]]

admin: public(address)
governance: public(address)

@deploy
def __init__(_admin: address, _governance: address):
    self.admin = _admin
    self.governance = _governance
    self.nextGrantId = 1

# ===== Create Grant =====

@external
def createGrant(
    beneficiary: address,
    token: address,
    totalAmount: uint256,
    startTime: uint256,
    cliffDuration: uint256,
    vestingDuration: uint256,
    revocable: bool
) -> uint256:
    """
    สร้าง vesting grant ใหม่
    
    Parameters:
        beneficiary: ผู้รับ tokens
        token: ERC20 token ที่จะ vest
        totalAmount: จำนวนทั้งหมด
        startTime: เวลาเริ่มต้น (0 = ตอนนี้)
        cliffDuration: ระยะ cliff (seconds)
        vestingDuration: ระยะ vesting ทั้งหมด (seconds)
        revocable: revoke ได้หรือไม่
    
    Returns:
        grantId: ID ของ grant
    """
    assert msg.sender == self.admin or msg.sender == self.governance, "Not authorized"
    assert beneficiary != empty(address), "Zero beneficiary"
    assert totalAmount > 0, "Zero amount"
    assert vestingDuration > 0, "Zero duration"
    assert cliffDuration <= vestingDuration, "Cliff > vesting"
    
    _startTime: uint256 = startTime
    if _startTime == 0:
        _startTime = block.timestamp
    
    # โอน tokens เข้า contract
    assert ERC20(token).transferFrom(msg.sender, self, totalAmount), "Transfer failed"
    
    grantId: uint256 = self.nextGrantId
    self.nextGrantId += 1
    
    self.grants[grantId] = VestingGrant({
        beneficiary: beneficiary,
        token: token,
        totalAmount: totalAmount,
        claimedAmount: 0,
        startTime: _startTime,
        cliffDuration: cliffDuration,
        vestingDuration: vestingDuration,
        revocable: revocable,
        revoked: False,
        revokedAt: 0,
        grantedBy: msg.sender
    })
    
    self.beneficiaryGrants[beneficiary].append(grantId)
    
    log VestingCreated(
        grantId,
        beneficiary,
        token,
        totalAmount,
        _startTime,
        cliffDuration,
        vestingDuration
    )
    
    return grantId

# ===== Vesting Calculation =====

@external
@view
def vestedAmount(grantId: uint256) -> uint256:
    """
    คำนวณ tokens ที่ vest แล้ว ณ ปัจจุบัน
    """
    return self._vestedAmount(grantId, block.timestamp)

@internal
@view
def _vestedAmount(grantId: uint256, timestamp: uint256) -> uint256:
    """
    Linear vesting formula:
    - ก่อน start: 0
    - ก่อน cliff end: 0
    - ระหว่าง cliff และ end: linear interpolation
    - หลัง end: totalAmount
    """
    grant: VestingGrant = self.grants[grantId]
    
    if grant.revoked:
        # ถ้า revoke แล้ว ใช้ timestamp ของ revoke
        return self._linearVesting(grant, grant.revokedAt)
    
    return self._linearVesting(grant, timestamp)

@internal
@pure
def _linearVesting(grant: VestingGrant, timestamp: uint256) -> uint256:
    if timestamp < grant.startTime + grant.cliffDuration:
        return 0  # ยังไม่ผ่าน cliff
    
    if timestamp >= grant.startTime + grant.vestingDuration:
        return grant.totalAmount  # vest ครบแล้ว
    
    # Linear interpolation
    elapsed: uint256 = timestamp - grant.startTime
    return grant.totalAmount * elapsed / grant.vestingDuration

@external
@view
def claimableAmount(grantId: uint256) -> uint256:
    """
    จำนวนที่ claim ได้ตอนนี้
    = vested - claimed
    """
    vested: uint256 = self._vestedAmount(grantId, block.timestamp)
    grant: VestingGrant = self.grants[grantId]
    
    if vested <= grant.claimedAmount:
        return 0
    
    return vested - grant.claimedAmount

# ===== Claim =====

@external
def claim(grantId: uint256) -> uint256:
    """
    Claim tokens ที่ vest แล้ว
    
    Returns:
        amount: จำนวนที่ได้รับ
    """
    grant: VestingGrant = self.grants[grantId]
    assert msg.sender == grant.beneficiary, "Not beneficiary"
    assert not grant.revoked, "Grant revoked"
    
    claimable: uint256 = self.claimableAmount(grantId)
    assert claimable > 0, "Nothing to claim"
    
    self.grants[grantId].claimedAmount += claimable
    
    assert ERC20(grant.token).transfer(grant.beneficiary, claimable), "Transfer failed"
    
    log TokensClaimed(grantId, grant.beneficiary, claimable)
    
    return claimable

@external
def claimMultiple(grantIds: DynArray[uint256, 20]) -> uint256:
    """Claim จากหลาย grants พร้อมกัน"""
    totalClaimed: uint256 = 0
    
    for grantId: uint256 in grantIds:
        grant: VestingGrant = self.grants[grantId]
        
        if grant.beneficiary != msg.sender or grant.revoked:
            continue
        
        claimable: uint256 = self.claimableAmount(grantId)
        
        if claimable == 0:
            continue
        
        self.grants[grantId].claimedAmount += claimable
        assert ERC20(grant.token).transfer(msg.sender, claimable), "Transfer failed"
        
        totalClaimed += claimable
        log TokensClaimed(grantId, msg.sender, claimable)
    
    return totalClaimed

# ===== Revoke =====

@external
def revokeGrant(grantId: uint256):
    """
    Revoke grant (เรียกคืน unvested tokens)
    
    Revoke ได้เฉพาะ:
    1. grant ต้อง revocable = true
    2. เรียกโดย grantedBy หรือ admin/governance
    """
    grant: VestingGrant = self.grants[grantId]
    assert grant.revocable, "Not revocable"
    assert not grant.revoked, "Already revoked"
    assert (
        msg.sender == grant.grantedBy or
        msg.sender == self.admin or
        msg.sender == self.governance
    ), "Not authorized"
    
    # คำนวณ vested amount ณ ตอนนี้
    vestedNow: uint256 = self._vestedAmount(grantId, block.timestamp)
    unvested: uint256 = grant.totalAmount - vestedNow
    refundAmount: uint256 = unvested - (grant.claimedAmount if vestedNow > grant.claimedAmount else 0)
    
    # Mark as revoked
    self.grants[grantId].revoked = True
    self.grants[grantId].revokedAt = block.timestamp
    
    # คืน unvested tokens ให้ grantedBy
    if unvested > 0:
        assert ERC20(grant.token).transfer(grant.grantedBy, unvested), "Transfer failed"
    
    log VestingRevoked(grantId, grant.beneficiary, unvested)

# ===== Transfer Grant =====

@external
def transferBeneficiary(grantId: uint256, newBeneficiary: address):
    """
    โอน grant ไปให้ address อื่น
    เรียกได้โดย beneficiary เท่านั้น
    """
    grant: VestingGrant = self.grants[grantId]
    assert msg.sender == grant.beneficiary, "Not beneficiary"
    assert newBeneficiary != empty(address), "Zero address"
    assert not grant.revoked, "Grant revoked"
    
    old: address = grant.beneficiary
    self.grants[grantId].beneficiary = newBeneficiary
    
    # Update beneficiaryGrants
    newGrants: DynArray[uint256, 100] = []
    for g: uint256 in self.beneficiaryGrants[old]:
        if g != grantId:
            newGrants.append(g)
    self.beneficiaryGrants[old] = newGrants
    
    self.beneficiaryGrants[newBeneficiary].append(grantId)
    
    log BeneficiaryChanged(grantId, old, newBeneficiary)

# ===== View =====

@external
@view
def getGrant(grantId: uint256) -> VestingGrant:
    return self.grants[grantId]

@external
@view
def getGrantsByBeneficiary(beneficiary: address) -> DynArray[uint256, 100]:
    return self.beneficiaryGrants[beneficiary]

@external
@view
def getVestingSchedule(grantId: uint256) -> (uint256, uint256, uint256, uint256, uint256):
    """
    ดู vesting schedule
    Returns: totalAmount, claimedAmount, vestedAmount, claimable, percentVested
    """
    grant: VestingGrant = self.grants[grantId]
    vested: uint256 = self._vestedAmount(grantId, block.timestamp)
    claimable: uint256 = 0
    if vested > grant.claimedAmount:
        claimable = vested - grant.claimedAmount
    
    percentVested: uint256 = 0
    if grant.totalAmount > 0:
        percentVested = vested * 10000 / grant.totalAmount
    
    return grant.totalAmount, grant.claimedAmount, vested, claimable, percentVested
```

---

## 3. Milestone Vesting

```vyper
# @version 0.4.0
# contracts/MilestoneVesting.vy
# Vesting แบบ milestone-based

from vyper.interfaces import ERC20

# Events
event MilestoneVestingCreated:
    grantId: indexed(uint256)
    beneficiary: indexed(address)
    totalMilestones: uint256
    totalAmount: uint256

event MilestoneApproved:
    grantId: indexed(uint256)
    milestoneIndex: uint256
    unlockAmount: uint256

event MilestoneClaimed:
    grantId: indexed(uint256)
    beneficiary: indexed(address)
    milestoneIndex: uint256
    amount: uint256

# Struct
struct Milestone:
    description: String[256]
    unlockAmount: uint256
    approved: bool
    approvedAt: uint256
    claimed: bool

struct MilestoneGrant:
    beneficiary: address
    token: address
    totalAmount: uint256
    claimedAmount: uint256
    milestoneCount: uint256
    revocable: bool
    revoked: bool
    grantedBy: address

# State
grants: HashMap[uint256, MilestoneGrant]
milestones: HashMap[uint256, HashMap[uint256, Milestone]]  # grantId -> index -> milestone
nextGrantId: uint256

admin: public(address)
governance: public(address)

# Approvers สำหรับ milestone approval
approvers: HashMap[address, bool]
requiredApprovals: uint256
milestoneApprovals: HashMap[uint256, HashMap[uint256, HashMap[address, bool]]]  # grantId -> milestone -> approver -> voted
milestoneApprovalCount: HashMap[uint256, HashMap[uint256, uint256]]  # grantId -> milestone -> count

@deploy
def __init__(_admin: address, _governance: address, _requiredApprovals: uint256):
    self.admin = _admin
    self.governance = _governance
    self.requiredApprovals = _requiredApprovals

@external
def addApprover(approver: address):
    assert msg.sender == self.governance, "Not governance"
    self.approvers[approver] = True

@external
def createMilestoneGrant(
    beneficiary: address,
    token: address,
    milestoneDescriptions: DynArray[String[256], 20],
    milestoneAmounts: DynArray[uint256, 20],
    revocable: bool
) -> uint256:
    """
    สร้าง milestone vesting
    
    Parameters:
        milestoneDescriptions: คำอธิบายแต่ละ milestone
        milestoneAmounts: จำนวน tokens ที่ unlock แต่ละ milestone
    """
    assert msg.sender == self.admin or msg.sender == self.governance, "Not authorized"
    assert len(milestoneDescriptions) == len(milestoneAmounts), "Length mismatch"
    assert len(milestoneDescriptions) > 0, "No milestones"
    
    totalAmount: uint256 = 0
    for amount: uint256 in milestoneAmounts:
        totalAmount += amount
    
    assert ERC20(token).transferFrom(msg.sender, self, totalAmount), "Transfer failed"
    
    grantId: uint256 = self.nextGrantId
    self.nextGrantId += 1
    
    self.grants[grantId] = MilestoneGrant({
        beneficiary: beneficiary,
        token: token,
        totalAmount: totalAmount,
        claimedAmount: 0,
        milestoneCount: len(milestoneDescriptions),
        revocable: revocable,
        revoked: False,
        grantedBy: msg.sender
    })
    
    for i: uint256 in range(20):
        if i >= len(milestoneDescriptions):
            break
        self.milestones[grantId][i] = Milestone({
            description: milestoneDescriptions[i],
            unlockAmount: milestoneAmounts[i],
            approved: False,
            approvedAt: 0,
            claimed: False
        })
    
    log MilestoneVestingCreated(grantId, beneficiary, len(milestoneDescriptions), totalAmount)
    
    return grantId

@external
def approveMilestone(grantId: uint256, milestoneIndex: uint256):
    """
    Approve milestone completion
    ต้องการ N approvers
    """
    assert self.approvers[msg.sender], "Not approver"
    
    grant: MilestoneGrant = self.grants[grantId]
    assert not grant.revoked, "Grant revoked"
    assert milestoneIndex < grant.milestoneCount, "Invalid milestone"
    
    milestone: Milestone = self.milestones[grantId][milestoneIndex]
    assert not milestone.approved, "Already approved"
    assert not self.milestoneApprovals[grantId][milestoneIndex][msg.sender], "Already voted"
    
    self.milestoneApprovals[grantId][milestoneIndex][msg.sender] = True
    self.milestoneApprovalCount[grantId][milestoneIndex] += 1
    
    # Check if enough approvals
    if self.milestoneApprovalCount[grantId][milestoneIndex] >= self.requiredApprovals:
        self.milestones[grantId][milestoneIndex].approved = True
        self.milestones[grantId][milestoneIndex].approvedAt = block.timestamp
        
        log MilestoneApproved(grantId, milestoneIndex, milestone.unlockAmount)

@external
def claimMilestone(grantId: uint256, milestoneIndex: uint256):
    """
    Claim tokens จาก approved milestone
    """
    grant: MilestoneGrant = self.grants[grantId]
    assert msg.sender == grant.beneficiary, "Not beneficiary"
    assert not grant.revoked, "Grant revoked"
    assert milestoneIndex < grant.milestoneCount, "Invalid milestone"
    
    milestone: Milestone = self.milestones[grantId][milestoneIndex]
    assert milestone.approved, "Not approved"
    assert not milestone.claimed, "Already claimed"
    
    self.milestones[grantId][milestoneIndex].claimed = True
    self.grants[grantId].claimedAmount += milestone.unlockAmount
    
    assert ERC20(grant.token).transfer(grant.beneficiary, milestone.unlockAmount), "Transfer failed"
    
    log MilestoneClaimed(grantId, grant.beneficiary, milestoneIndex, milestone.unlockAmount)

@external
def claimAllApprovedMilestones(grantId: uint256) -> uint256:
    """Claim ทุก milestone ที่ approved แล้ว"""
    grant: MilestoneGrant = self.grants[grantId]
    assert msg.sender == grant.beneficiary, "Not beneficiary"
    assert not grant.revoked, "Grant revoked"
    
    totalClaimed: uint256 = 0
    
    for i: uint256 in range(20):
        if i >= grant.milestoneCount:
            break
        
        milestone: Milestone = self.milestones[grantId][i]
        
        if milestone.approved and not milestone.claimed:
            self.milestones[grantId][i].claimed = True
            totalClaimed += milestone.unlockAmount
            
            log MilestoneClaimed(grantId, grant.beneficiary, i, milestone.unlockAmount)
    
    if totalClaimed > 0:
        self.grants[grantId].claimedAmount += totalClaimed
        assert ERC20(grant.token).transfer(grant.beneficiary, totalClaimed), "Transfer failed"
    
    return totalClaimed

@external
@view
def getMilestoneStatus(grantId: uint256) -> (uint256, uint256, uint256, uint256):
    """
    ดูสถานะ milestones
    Returns: total, approved, claimed, claimable
    """
    grant: MilestoneGrant = self.grants[grantId]
    
    approved: uint256 = 0
    claimed: uint256 = 0
    claimable: uint256 = 0
    
    for i: uint256 in range(20):
        if i >= grant.milestoneCount:
            break
        
        m: Milestone = self.milestones[grantId][i]
        if m.approved:
            approved += 1
            if m.claimed:
                claimed += 1
            else:
                claimable += m.unlockAmount
    
    return grant.milestoneCount, approved, claimed, claimable
```

---

## 4. Vesting Factory

```vyper
# @version 0.4.0
# contracts/VestingFactory.vy
# Factory สำหรับ deploy vesting contracts

# Events
event VestingDeployed:
    vestingContract: indexed(address)
    vestingType: String[32]
    beneficiary: indexed(address)
    amount: uint256

# State
admin: public(address)
governance: public(address)
vestingContracts: DynArray[address, 10000]
vestingByBeneficiary: HashMap[address, DynArray[address, 100]]

@deploy
def __init__(_admin: address, _governance: address):
    self.admin = _admin
    self.governance = _governance

@external
@view
def getVestingContracts(beneficiary: address) -> DynArray[address, 100]:
    return self.vestingByBeneficiary[beneficiary]

@external
@view  
def allVestingContracts() -> DynArray[address, 10000]:
    return self.vestingContracts
```

---

## 5. Tests

```python
# tests/test_vesting.py
import pytest
from brownie import LinearVesting, MilestoneVesting, MockERC20, accounts, chain

@pytest.fixture
def setup():
    owner = accounts[0]
    alice = accounts[1]
    bob = accounts[2]
    
    token = MockERC20.deploy("Test Token", "TEST", 18, {"from": owner})
    token.mint(owner, 10**24, {"from": owner})
    
    vesting = LinearVesting.deploy(owner.address, owner.address, {"from": owner})
    
    return owner, alice, bob, token, vesting

def test_linear_vesting(setup):
    owner, alice, bob, token, vesting = setup
    
    amount = 10**21  # 1000 tokens
    start = chain.time()
    cliff = 6 * 30 * 24 * 3600   # 6 months
    duration = 4 * 365 * 24 * 3600  # 4 years
    
    token.approve(vesting.address, amount, {"from": owner})
    
    grantId = vesting.createGrant(
        alice.address,
        token.address,
        amount,
        start,
        cliff,
        duration,
        True,  # revocable
        {"from": owner}
    ).return_value
    
    # Before cliff: 0
    assert vesting.vestedAmount(grantId) == 0
    
    # After cliff (6 months)
    chain.sleep(cliff + 1)
    chain.mine(1)
    
    vested = vesting.vestedAmount(grantId)
    assert vested > 0
    print(f"Vested after cliff: {vested / 10**18:.2f} tokens")
    
    # After full vesting (4 years)
    chain.sleep(duration - cliff)
    chain.mine(1)
    
    vested_full = vesting.vestedAmount(grantId)
    assert vested_full == amount
    print(f"Fully vested: {vested_full / 10**18:.2f} tokens")

def test_claim(setup):
    owner, alice, bob, token, vesting = setup
    
    amount = 10**21
    start = chain.time()
    cliff = 30 * 24 * 3600  # 1 month
    duration = 12 * 30 * 24 * 3600  # 12 months
    
    token.approve(vesting.address, amount, {"from": owner})
    grantId = vesting.createGrant(
        alice.address, token.address, amount,
        start, cliff, duration, False,
        {"from": owner}
    ).return_value
    
    # Skip 3 months
    chain.sleep(3 * 30 * 24 * 3600 + 1)
    chain.mine(1)
    
    before = token.balanceOf(alice.address)
    vesting.claim(grantId, {"from": alice})
    after = token.balanceOf(alice.address)
    
    claimed = after - before
    expected = amount * 3 // 12  # ~25%
    
    print(f"Claimed: {claimed / 10**18:.2f} tokens (expected ~{expected / 10**18:.2f})")
    assert abs(claimed - expected) < 10**18  # Within 1 token

def test_revoke(setup):
    owner, alice, bob, token, vesting = setup
    
    amount = 10**21
    start = chain.time()
    cliff = 0
    duration = 12 * 30 * 24 * 3600
    
    token.approve(vesting.address, amount, {"from": owner})
    grantId = vesting.createGrant(
        alice.address, token.address, amount,
        start, cliff, duration, True,  # revocable
        {"from": owner}
    ).return_value
    
    # Skip 6 months
    chain.sleep(6 * 30 * 24 * 3600 + 1)
    chain.mine(1)
    
    # Alice claims vested
    vesting.claim(grantId, {"from": alice})
    
    owner_before = token.balanceOf(owner.address)
    
    # Revoke
    vesting.revokeGrant(grantId, {"from": owner})
    
    owner_after = token.balanceOf(owner.address)
    
    # Owner should get back unvested tokens
    refunded = owner_after - owner_before
    expected = amount // 2  # ~50% unvested
    
    print(f"Refunded to owner: {refunded / 10**18:.2f} tokens")
    assert refunded > 0

def test_milestone_vesting(setup):
    owner, alice, bob, token, vesting = setup
    
    # Create milestone vesting
    milestone_vesting = MilestoneVesting.deploy(
        owner.address, owner.address, 1,  # 1 of 1 approval
        {"from": owner}
    )
    
    milestone_vesting.addApprover(owner.address, {"from": owner})
    
    amounts = [10**20, 10**20, 3 * 10**20, 5 * 10**20]  # 4 milestones
    descriptions = ["MVP Launch", "100 Users", "Protocol Launch", "100K TVL"]
    
    total = sum(amounts)
    token.approve(milestone_vesting.address, total, {"from": owner})
    
    grantId = milestone_vesting.createMilestoneGrant(
        alice.address,
        token.address,
        descriptions,
        amounts,
        True,
        {"from": owner}
    ).return_value
    
    # Approve first milestone
    milestone_vesting.approveMilestone(grantId, 0, {"from": owner})
    
    # Claim
    before = token.balanceOf(alice.address)
    milestone_vesting.claimMilestone(grantId, 0, {"from": alice})
    after = token.balanceOf(alice.address)
    
    assert after - before == amounts[0]
    print(f"Claimed milestone 0: {(after - before) / 10**18:.2f} tokens")
    
    # Status
    total_m, approved, claimed, claimable = milestone_vesting.getMilestoneStatus(grantId)
    print(f"Milestones: {total_m} total, {approved} approved, {claimed} claimed")
```

---

## 6. สรุป

### Vesting Best Practices:

**1. Standard Schedule (Startup)**
- 25% cliff ที่ 1 ปี
- Linear vesting อีก 3 ปี
- รวม 4 ปี

**2. Revocable vs Non-revocable**
- Revocable: ใช้สำหรับ employee grants
- Non-revocable: ใช้สำหรับ investor/founder grants

**3. Milestone Vesting**
- ดีสำหรับ grants ที่ผูกกับ performance
- ต้องการ oracle หรือ multi-sig approval

**4. Gas Optimization**
- Claim multiple grants พร้อมกัน
- Store เฉพาะ claimedAmount, คำนวณ vested on-the-fly

**5. Security**
- ตรวจสอบ beneficiary ก่อน claim
- Guard against reentrancy
- ทดสอบ edge cases: cliff boundary, full vesting
