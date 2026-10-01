# Part 036: Vesting Contract

## สารบัญ
1. [บทนำ Vesting](#บทนำ)
2. [Linear Vesting](#linear-vesting)
3. [Cliff Vesting](#cliff-vesting)
4. [Team Vesting](#team-vesting)
5. [Investor Vesting](#investor-vesting)
6. [ตัวอย่าง: Token Vesting Contract](#ตัวอย่าง-token-vesting-contract)
7. [Test Code](#test-code)

---

## บทนำ

**Vesting** คือกระบวนการปล่อย token ให้เป็นไปตามกำหนดเวลา เพื่อป้องกัน team/investor dump token ทั้งหมดพร้อมกัน

### ทำไมต้องมี Vesting?
- ป้องกัน rug pull
- สร้างแรงจูงใจระยะยาว
- แสดงความมุ่งมั่นของทีม
- มาตรฐานใน tokenomics

### Vesting Schedule ทั่วไป
- **Team**: 4 ปี vesting, 1 ปี cliff
- **Advisor**: 2 ปี vesting, 6 เดือน cliff
- **Investor**: 1-2 ปี vesting, 6 เดือน cliff
- **Community**: Immediate หรือ 1 ปี linear

---

## Linear Vesting

### Token ปล่อยสม่ำเสมอตามเวลา

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Linear Vesting

interface ERC20:
    def transfer(to: address, amount: uint256) -> bool: nonpayable
    def balanceOf(account: address) -> uint256: view

event VestingCreated:
    beneficiary: indexed(address)
    total_amount: uint256
    start: uint256
    duration: uint256

event TokensClaimed:
    beneficiary: indexed(address)
    amount: uint256

struct VestingSchedule:
    total_amount: uint256
    start_time: uint256
    duration: uint256
    claimed_amount: uint256

token: public(address)
vesting_schedules: public(HashMap[address, VestingSchedule])

@deploy
def __init__(_token: address):
    self.token = _token

@internal
def _vested_amount(schedule: VestingSchedule) -> uint256:
    """คำนวณจำนวนที่ vested แล้ว"""
    if block.timestamp < schedule.start_time:
        return 0
    
    elapsed: uint256 = block.timestamp - schedule.start_time
    
    if elapsed >= schedule.duration:
        return schedule.total_amount
    
    return schedule.total_amount * elapsed / schedule.duration

@external
def create_vesting(
    beneficiary: address,
    total_amount: uint256,
    start_time: uint256,
    duration: uint256
):
    """สร้าง vesting schedule"""
    assert beneficiary != empty(address)
    assert total_amount > 0
    assert duration > 0
    assert self.vesting_schedules[beneficiary].total_amount == 0, "Already exists"
    
    self.vesting_schedules[beneficiary] = VestingSchedule({
        total_amount: total_amount,
        start_time: start_time,
        duration: duration,
        claimed_amount: 0
    })
    
    log VestingCreated(beneficiary, total_amount, start_time, duration)

@external
def claim():
    """Claim vested tokens"""
    schedule: VestingSchedule = self.vesting_schedules[msg.sender]
    assert schedule.total_amount > 0, "No vesting schedule"
    
    vested: uint256 = self._vested_amount(schedule)
    claimable: uint256 = vested - schedule.claimed_amount
    
    assert claimable > 0, "Nothing to claim"
    
    self.vesting_schedules[msg.sender].claimed_amount += claimable
    
    ERC20(self.token).transfer(msg.sender, claimable)
    
    log TokensClaimed(msg.sender, claimable)

@view
@external
def vested_amount(beneficiary: address) -> uint256:
    return self._vested_amount(self.vesting_schedules[beneficiary])

@view
@external
def claimable_amount(beneficiary: address) -> uint256:
    schedule: VestingSchedule = self.vesting_schedules[beneficiary]
    vested: uint256 = self._vested_amount(schedule)
    return vested - schedule.claimed_amount
```

---

## Cliff Vesting

### ไม่ได้รับอะไรก่อน cliff แล้ว linear หลัง cliff

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Cliff Vesting

interface ERC20:
    def transfer(to: address, amount: uint256) -> bool: nonpayable

event VestingCreated:
    beneficiary: indexed(address)
    total_amount: uint256
    cliff_end: uint256
    vesting_end: uint256

event TokensClaimed:
    beneficiary: indexed(address)
    amount: uint256

struct CliffVestingSchedule:
    total_amount: uint256
    cliff_duration: uint256    # ระยะเวลา cliff (ไม่ได้อะไร)
    vesting_duration: uint256  # ระยะเวลา vesting หลัง cliff
    start_time: uint256
    claimed_amount: uint256

token: public(address)
owner: public(address)
schedules: public(HashMap[address, CliffVestingSchedule])

@deploy
def __init__(_token: address):
    self.token = _token
    self.owner = msg.sender

@internal
def _vested_amount(schedule: CliffVestingSchedule) -> uint256:
    """
    คำนวณ vested amount
    - ก่อน cliff: 0
    - หลัง cliff จนถึง end: linear
    - หลัง end: total_amount
    """
    cliff_end: uint256 = schedule.start_time + schedule.cliff_duration
    vesting_end: uint256 = cliff_end + schedule.vesting_duration
    
    if block.timestamp < cliff_end:
        return 0  # ยังอยู่ใน cliff period
    
    if block.timestamp >= vesting_end:
        return schedule.total_amount  # vested ทั้งหมด
    
    # Linear vesting หลัง cliff
    elapsed_after_cliff: uint256 = block.timestamp - cliff_end
    return schedule.total_amount * elapsed_after_cliff / schedule.vesting_duration

@external
def create_cliff_vesting(
    beneficiary: address,
    total_amount: uint256,
    cliff_months: uint256,    # เดือน
    vesting_months: uint256   # เดือน
):
    """
    สร้าง vesting พร้อม cliff
    ตัวอย่าง: cliff_months=12, vesting_months=36 = 1 ปี cliff + 3 ปี vesting
    """
    assert msg.sender == self.owner, "Not owner"
    assert beneficiary != empty(address)
    assert total_amount > 0
    assert vesting_months > 0
    assert self.schedules[beneficiary].total_amount == 0
    
    cliff_secs: uint256 = cliff_months * 30 * 86400
    vesting_secs: uint256 = vesting_months * 30 * 86400
    
    self.schedules[beneficiary] = CliffVestingSchedule({
        total_amount: total_amount,
        cliff_duration: cliff_secs,
        vesting_duration: vesting_secs,
        start_time: block.timestamp,
        claimed_amount: 0
    })
    
    cliff_end: uint256 = block.timestamp + cliff_secs
    vesting_end: uint256 = cliff_end + vesting_secs
    
    log VestingCreated(beneficiary, total_amount, cliff_end, vesting_end)

@external
def claim():
    schedule: CliffVestingSchedule = self.schedules[msg.sender]
    assert schedule.total_amount > 0, "No schedule"
    
    vested: uint256 = self._vested_amount(schedule)
    claimable: uint256 = vested - schedule.claimed_amount
    
    assert claimable > 0, "Nothing to claim"
    
    self.schedules[msg.sender].claimed_amount += claimable
    ERC20(self.token).transfer(msg.sender, claimable)
    
    log TokensClaimed(msg.sender, claimable)

@view
@external
def get_schedule_info(beneficiary: address) -> (uint256, uint256, uint256, uint256):
    """Return (total, vested, claimed, claimable)"""
    schedule: CliffVestingSchedule = self.schedules[beneficiary]
    vested: uint256 = self._vested_amount(schedule)
    claimable: uint256 = vested - schedule.claimed_amount
    
    return schedule.total_amount, vested, schedule.claimed_amount, claimable
```

---

## Team Vesting

### Vesting สำหรับทีม (4 ปี, 1 ปี cliff)

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Team Vesting

interface ERC20:
    def transfer(to: address, amount: uint256) -> bool: nonpayable
    def transferFrom(from_: address, to: address, amount: uint256) -> bool: nonpayable

event MemberAdded:
    member: indexed(address)
    role: String[50]
    allocation: uint256

event Terminated:
    member: indexed(address)
    vested_at_termination: uint256

event Claimed:
    member: indexed(address)
    amount: uint256

# Standard team vesting: 4 year total, 1 year cliff
CLIFF_DURATION: constant(uint256) = 365 * 86400     # 1 ปี
VESTING_DURATION: constant(uint256) = 3 * 365 * 86400  # 3 ปีหลัง cliff (รวม 4 ปี)

struct TeamMember:
    allocation: uint256
    start_time: uint256
    cliff_end: uint256
    vesting_end: uint256
    claimed: uint256
    terminated: bool
    vested_at_termination: uint256  # ถ้าถูก terminate

token: public(address)
company: public(address)   # บริษัท/DAO

members: public(HashMap[address, TeamMember])

@deploy
def __init__(_token: address, _company: address):
    self.token = _token
    self.company = _company

@external
def add_team_member(
    member: address,
    allocation: uint256,
    role: String[50]
):
    """
    @notice เพิ่ม team member
    @dev เฉพาะ company
    """
    assert msg.sender == self.company, "Not company"
    assert member != empty(address)
    assert allocation > 0
    assert self.members[member].allocation == 0, "Already added"
    
    start: uint256 = block.timestamp
    cliff_end: uint256 = start + CLIFF_DURATION
    vesting_end: uint256 = cliff_end + VESTING_DURATION
    
    self.members[member] = TeamMember({
        allocation: allocation,
        start_time: start,
        cliff_end: cliff_end,
        vesting_end: vesting_end,
        claimed: 0,
        terminated: False,
        vested_at_termination: 0
    })
    
    log MemberAdded(member, role, allocation)

@internal
def _calculate_vested(member: TeamMember) -> uint256:
    if member.terminated:
        return member.vested_at_termination
    
    if block.timestamp < member.cliff_end:
        return 0
    
    if block.timestamp >= member.vesting_end:
        return member.allocation
    
    elapsed: uint256 = block.timestamp - member.cliff_end
    return member.allocation * elapsed / VESTING_DURATION

@external
def claim():
    """Team member claim vested tokens"""
    member: TeamMember = self.members[msg.sender]
    assert member.allocation > 0, "Not a team member"
    
    vested: uint256 = self._calculate_vested(member)
    claimable: uint256 = vested - member.claimed
    
    assert claimable > 0, "Nothing to claim"
    
    self.members[msg.sender].claimed += claimable
    ERC20(self.token).transfer(msg.sender, claimable)
    
    log Claimed(msg.sender, claimable)

@external
def terminate(member: address):
    """
    @notice Terminate team member
    @dev Token ที่ vested แล้วยังเป็นของ member
         ส่วนที่ยังไม่ vested คืนบริษัท
    """
    assert msg.sender == self.company, "Not company"
    
    m: TeamMember = self.members[member]
    assert m.allocation > 0, "Not a member"
    assert not m.terminated, "Already terminated"
    
    vested: uint256 = self._calculate_vested(m)
    
    self.members[member].terminated = True
    self.members[member].vested_at_termination = vested
    
    # Return unvested tokens to company
    unvested: uint256 = m.allocation - vested
    if unvested > 0:
        ERC20(self.token).transfer(self.company, unvested)
    
    log Terminated(member, vested)

@view
@external
def get_member_info(member: address) -> (uint256, uint256, uint256, bool):
    """Return (allocation, vested, claimable, terminated)"""
    m: TeamMember = self.members[member]
    vested: uint256 = self._calculate_vested(m)
    claimable: uint256 = vested - m.claimed
    
    return m.allocation, vested, claimable, m.terminated
```

---

## Investor Vesting

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Investor Vesting

interface ERC20:
    def transfer(to: address, amount: uint256) -> bool: nonpayable
    def transferFrom(from_: address, to: address, amount: uint256) -> bool: nonpayable

event InvestorAdded:
    investor: indexed(address)
    round: String[20]
    amount: uint256
    price: uint256

event TokensClaimed:
    investor: indexed(address)
    amount: uint256

event TGEClaimed:
    investor: indexed(address)
    amount: uint256

# Investment rounds
SEED_ROUND: constant(String[20]) = "SEED"
PRIVATE_ROUND: constant(String[20]) = "PRIVATE"
PUBLIC_ROUND: constant(String[20]) = "PUBLIC"

struct InvestorVesting:
    round: String[20]
    total_tokens: uint256
    tge_percentage: uint256    # % ที่ได้ทันที (basis points, 1000 = 10%)
    cliff_months: uint256
    vesting_months: uint256
    start_time: uint256
    tge_claimed: bool
    claimed_after_cliff: uint256

owner: public(address)
token: public(address)

investors: public(HashMap[address, InvestorVesting])

# Round configurations
# Seed: 10% TGE, 6m cliff, 18m vesting
# Private: 15% TGE, 3m cliff, 12m vesting  
# Public: 25% TGE, 0m cliff, 6m vesting

@deploy
def __init__(_token: address):
    self.owner = msg.sender
    self.token = _token

@external
def add_investor(
    investor: address,
    total_tokens: uint256,
    tge_percentage: uint256,
    cliff_months: uint256,
    vesting_months: uint256,
    round_name: String[20]
):
    """เพิ่ม investor"""
    assert msg.sender == self.owner
    assert investor != empty(address)
    assert total_tokens > 0
    assert tge_percentage <= 10000  # max 100%
    assert self.investors[investor].total_tokens == 0
    
    self.investors[investor] = InvestorVesting({
        round: round_name,
        total_tokens: total_tokens,
        tge_percentage: tge_percentage,
        cliff_months: cliff_months,
        vesting_months: vesting_months,
        start_time: block.timestamp,
        tge_claimed: False,
        claimed_after_cliff: 0
    })
    
    log InvestorAdded(investor, round_name, total_tokens, 0)

@external
def claim_tge():
    """Claim TGE allocation ทันที"""
    inv: InvestorVesting = self.investors[msg.sender]
    assert inv.total_tokens > 0, "No allocation"
    assert not inv.tge_claimed, "TGE already claimed"
    
    tge_amount: uint256 = inv.total_tokens * inv.tge_percentage / 10000
    
    self.investors[msg.sender].tge_claimed = True
    
    if tge_amount > 0:
        ERC20(self.token).transfer(msg.sender, tge_amount)
        log TGEClaimed(msg.sender, tge_amount)

@internal
def _claimable_after_cliff(inv: InvestorVesting) -> uint256:
    cliff_end: uint256 = inv.start_time + inv.cliff_months * 30 * 86400
    
    if block.timestamp < cliff_end:
        return 0
    
    vesting_end: uint256 = cliff_end + inv.vesting_months * 30 * 86400
    
    tge_amount: uint256 = inv.total_tokens * inv.tge_percentage / 10000
    post_cliff_total: uint256 = inv.total_tokens - tge_amount
    
    if block.timestamp >= vesting_end:
        return post_cliff_total - inv.claimed_after_cliff
    
    elapsed: uint256 = block.timestamp - cliff_end
    vested: uint256 = post_cliff_total * elapsed / (inv.vesting_months * 30 * 86400)
    
    return vested - inv.claimed_after_cliff

@external
def claim_vested():
    """Claim vested tokens หลัง cliff"""
    inv: InvestorVesting = self.investors[msg.sender]
    assert inv.total_tokens > 0
    
    claimable: uint256 = self._claimable_after_cliff(inv)
    assert claimable > 0, "Nothing to claim"
    
    self.investors[msg.sender].claimed_after_cliff += claimable
    ERC20(self.token).transfer(msg.sender, claimable)
    
    log TokensClaimed(msg.sender, claimable)

@view
@external
def get_investor_info(investor: address) -> (uint256, uint256, bool, uint256):
    inv: InvestorVesting = self.investors[investor]
    tge_amount: uint256 = inv.total_tokens * inv.tge_percentage / 10000
    claimable: uint256 = self._claimable_after_cliff(inv)
    
    return inv.total_tokens, tge_amount, inv.tge_claimed, claimable
```

---

## ตัวอย่าง: Token Vesting Contract

```vyper
# SPDX-License-Identifier: MIT
# @version 0.4.0
# @title Complete Token Vesting Contract
# @notice Vesting ครบถ้วนรองรับทุก allocation type

interface ERC20:
    def transfer(to: address, amount: uint256) -> bool: nonpayable
    def transferFrom(from_: address, to: address, amount: uint256) -> bool: nonpayable
    def balanceOf(account: address) -> uint256: view

MAX_SCHEDULES: constant(uint256) = 100

# ==================== Events ====================

event ScheduleCreated:
    id: indexed(uint256)
    beneficiary: indexed(address)
    category: String[30]
    total: uint256
    cliff: uint256
    end: uint256

event TokensClaimed:
    id: indexed(uint256)
    beneficiary: indexed(address)
    amount: uint256

event ScheduleRevoked:
    id: indexed(uint256)
    unvested_returned: uint256

# ==================== Structs ====================

struct VestingSchedule:
    beneficiary: address
    token: address
    category: String[30]
    total_amount: uint256
    tge_bps: uint256          # % ที่ได้ทันที
    cliff_end: uint256
    vesting_end: uint256
    released: uint256
    tge_released: bool
    revoked: bool
    revocable: bool

# ==================== State Variables ====================

owner: public(address)
schedule_count: public(uint256)
schedules: public(HashMap[uint256, VestingSchedule])
beneficiary_schedules: HashMap[address, DynArray[uint256, 20]]

# ==================== Constructor ====================

@deploy
def __init__():
    self.owner = msg.sender

# ==================== Create Schedule ====================

@external
def create_schedule(
    beneficiary: address,
    token: address,
    category: String[30],
    total_amount: uint256,
    tge_bps: uint256,
    cliff_months: uint256,
    vesting_months: uint256,
    revocable: bool
) -> uint256:
    """
    @notice สร้าง vesting schedule
    @param tge_bps %, ทันทีใน bps (1000 = 10%)
    @param cliff_months จำนวนเดือน cliff (0 = ไม่มี cliff)
    @param vesting_months จำนวนเดือนหลัง cliff
    @param revocable owner สามารถ revoke ได้หรือไม่
    """
    assert msg.sender == self.owner, "Not owner"
    assert beneficiary != empty(address)
    assert total_amount > 0
    assert tge_bps <= 10000  # max 100%
    assert vesting_months > 0
    
    now: uint256 = block.timestamp
    cliff_secs: uint256 = cliff_months * 30 * 86400
    vesting_secs: uint256 = vesting_months * 30 * 86400
    
    schedule_id: uint256 = self.schedule_count
    
    self.schedules[schedule_id] = VestingSchedule({
        beneficiary: beneficiary,
        token: token,
        category: category,
        total_amount: total_amount,
        tge_bps: tge_bps,
        cliff_end: now + cliff_secs,
        vesting_end: now + cliff_secs + vesting_secs,
        released: 0,
        tge_released: False,
        revoked: False,
        revocable: revocable
    })
    
    self.beneficiary_schedules[beneficiary].append(schedule_id)
    self.schedule_count += 1
    
    log ScheduleCreated(
        schedule_id, beneficiary, category, total_amount,
        now + cliff_secs, now + cliff_secs + vesting_secs
    )
    return schedule_id

# ==================== Claim ====================

@internal
def _releasable_amount(s: VestingSchedule) -> uint256:
    """คำนวณ releasable amount"""
    if s.revoked:
        return 0
    
    total_vested: uint256 = 0
    tge_amount: uint256 = s.total_amount * s.tge_bps / 10000
    post_cliff_amount: uint256 = s.total_amount - tge_amount
    
    # TGE amount (พร้อมทันที)
    if not s.tge_released:
        total_vested += tge_amount
    
    # Post-cliff vesting
    if block.timestamp >= s.cliff_end:
        if block.timestamp >= s.vesting_end:
            total_vested += post_cliff_amount
        else:
            elapsed: uint256 = block.timestamp - s.cliff_end
            duration: uint256 = s.vesting_end - s.cliff_end
            total_vested += post_cliff_amount * elapsed / duration
    
    # หัก already released
    if total_vested <= s.released:
        return 0
    
    # แก้ไข: ถ้า TGE ถูก release แล้ว ไม่ต้องคิดซ้ำ
    already_counted: uint256 = s.released
    return total_vested - already_counted

@external
@nonreentrant
def release(schedule_id: uint256):
    """
    @notice Claim vested tokens
    """
    s: VestingSchedule = self.schedules[schedule_id]
    
    assert s.beneficiary == msg.sender, "Not beneficiary"
    assert not s.revoked, "Schedule revoked"
    
    releasable: uint256 = self._releasable_amount(s)
    assert releasable > 0, "Nothing to release"
    
    # Mark TGE as released ถ้ายังไม่ได้รับ
    if not s.tge_released and block.timestamp >= s.cliff_end - (s.cliff_end - (s.cliff_end - s.cliff_end)):
        tge_amount: uint256 = s.total_amount * s.tge_bps / 10000
        if releasable >= tge_amount:
            self.schedules[schedule_id].tge_released = True
    
    self.schedules[schedule_id].released += releasable
    
    ERC20(s.token).transfer(s.beneficiary, releasable)
    
    log TokensClaimed(schedule_id, s.beneficiary, releasable)

# ==================== Revoke ====================

@external
def revoke(schedule_id: uint256):
    """
    @notice Revoke vesting schedule
    @dev Owner สามารถ revoke ได้เฉพาะ revocable schedules
         Tokens ที่ vested แล้วยังเป็นของ beneficiary
    """
    assert msg.sender == self.owner, "Not owner"
    
    s: VestingSchedule = self.schedules[schedule_id]
    assert s.revocable, "Not revocable"
    assert not s.revoked, "Already revoked"
    
    # คำนวณ vested ณ ปัจจุบัน
    tge_amount: uint256 = s.total_amount * s.tge_bps / 10000
    post_cliff: uint256 = s.total_amount - tge_amount
    
    vested_post_cliff: uint256 = 0
    if block.timestamp >= s.cliff_end:
        if block.timestamp >= s.vesting_end:
            vested_post_cliff = post_cliff
        else:
            elapsed: uint256 = block.timestamp - s.cliff_end
            duration: uint256 = s.vesting_end - s.cliff_end
            vested_post_cliff = post_cliff * elapsed / duration
    
    total_vested: uint256 = tge_amount + vested_post_cliff
    unvested: uint256 = s.total_amount - total_vested
    
    self.schedules[schedule_id].revoked = True
    
    # Return unvested tokens to owner
    if unvested > 0:
        ERC20(s.token).transfer(msg.sender, unvested)
    
    log ScheduleRevoked(schedule_id, unvested)

# ==================== View Functions ====================

@view
@external
def get_schedule(schedule_id: uint256) -> VestingSchedule:
    return self.schedules[schedule_id]

@view
@external
def releasable(schedule_id: uint256) -> uint256:
    return self._releasable_amount(self.schedules[schedule_id])

@view
@external
def get_beneficiary_schedules(beneficiary: address) -> DynArray[uint256, 20]:
    return self.beneficiary_schedules[beneficiary]

@view
@external
def total_releasable_for_beneficiary(beneficiary: address) -> uint256:
    """รวม releasable จากทุก schedule"""
    total: uint256 = 0
    for sid: uint256 in self.beneficiary_schedules[beneficiary]:
        total += self._releasable_amount(self.schedules[sid])
    return total

@view
@external
def get_vesting_summary(schedule_id: uint256) -> (uint256, uint256, uint256, uint256):
    """Return (total, released, releasable, remaining_locked)"""
    s: VestingSchedule = self.schedules[schedule_id]
    releasable_now: uint256 = self._releasable_amount(s)
    remaining_locked: uint256 = s.total_amount - s.released - releasable_now
    
    return s.total_amount, s.released, releasable_now, remaining_locked
```

---

## Test Code

```python
# tests/test_vesting.py
import pytest

ONE_MONTH = 30 * 86400
ONE_YEAR = 12 * ONE_MONTH

@pytest.fixture
def owner(accounts):
    return accounts[0]

@pytest.fixture
def team_member(accounts):
    return accounts[1]

@pytest.fixture
def investor(accounts):
    return accounts[2]

@pytest.fixture
def token(owner, project):
    t = project.MockERC20.deploy("Token", "TKN", 18, sender=owner)
    t.mint(owner.address, 100_000_000 * 10**18, sender=owner)
    return t

@pytest.fixture
def vesting(owner, project):
    return project.CompleteVesting.deploy(sender=owner)

class TestLinearVesting:
    
    def test_create_schedule(self, vesting, owner, team_member, token):
        """ทดสอบสร้าง schedule"""
        amount = 1_000_000 * 10**18
        token.approve(vesting.address, amount, sender=owner)
        
        schedule_id = vesting.create_schedule(
            team_member.address,
            token.address,
            "TEAM",
            amount,
            0,      # 0% TGE
            12,     # 12 month cliff
            36,     # 36 month vesting
            True,   # revocable
            sender=owner
        )
        
        s = vesting.get_schedule(schedule_id)
        assert s.beneficiary == team_member.address
        assert s.total_amount == amount
    
    def test_no_claim_before_cliff(self, vesting, owner, team_member, token):
        """ทดสอบว่าไม่สามารถ claim ก่อน cliff"""
        amount = 1_000_000 * 10**18
        
        schedule_id = vesting.create_schedule(
            team_member.address, token.address, "TEAM",
            amount, 0, 12, 36, True, sender=owner
        )
        
        releasable = vesting.releasable(schedule_id)
        assert releasable == 0
    
    def test_tge_claim(self, vesting, owner, investor, token):
        """ทดสอบ TGE claim"""
        amount = 1_000_000 * 10**18
        tge_bps = 1000  # 10% TGE
        
        schedule_id = vesting.create_schedule(
            investor.address, token.address, "INVESTOR",
            amount, tge_bps, 0, 12, False, sender=owner
        )
        
        # ส่ง token ไป vesting contract
        token.transfer(vesting.address, amount, sender=owner)
        
        releasable = vesting.releasable(schedule_id)
        expected_tge = amount * tge_bps // 10000
        
        assert releasable == expected_tge
    
    def test_linear_vesting_after_cliff(self, vesting, owner, team_member, token, chain):
        """ทดสอบ linear vesting หลัง cliff"""
        amount = 1_200_000 * 10**18  # 1.2M tokens
        cliff_months = 12
        vesting_months = 12  # 100k per month after cliff
        
        schedule_id = vesting.create_schedule(
            team_member.address, token.address, "TEAM",
            amount, 0, cliff_months, vesting_months, True, sender=owner
        )
        
        token.transfer(vesting.address, amount, sender=owner)
        
        # เดิน time ข้าม cliff + 6 เดือน
        chain.mine(deltatime=cliff_months * ONE_MONTH + 6 * ONE_MONTH)
        
        releasable = vesting.releasable(schedule_id)
        expected = amount * 6 // vesting_months  # 50%
        
        # Allow 1% deviation
        assert abs(int(releasable) - int(expected)) < expected // 100
    
    def test_revoke_schedule(self, vesting, owner, team_member, token, chain):
        """ทดสอบ revoke"""
        amount = 1_200_000 * 10**18
        
        schedule_id = vesting.create_schedule(
            team_member.address, token.address, "TEAM",
            amount, 0, 12, 36, True, sender=owner  # revocable
        )
        
        token.transfer(vesting.address, amount, sender=owner)
        
        # เดิน time 18 เดือน (12 cliff + 6 vesting)
        chain.mine(deltatime=18 * ONE_MONTH)
        
        before_owner = token.balanceOf(owner.address)
        vesting.revoke(schedule_id, sender=owner)
        after_owner = token.balanceOf(owner.address)
        
        # Owner ได้รับ unvested tokens คืน
        assert after_owner > before_owner
    
    def test_cannot_revoke_non_revocable(self, vesting, owner, investor, token):
        """ทดสอบว่า non-revocable ยกเลิกไม่ได้"""
        schedule_id = vesting.create_schedule(
            investor.address, token.address, "INVESTOR",
            1_000_000 * 10**18, 0, 6, 18, False,  # NOT revocable
            sender=owner
        )
        
        with pytest.raises(Exception):
            vesting.revoke(schedule_id, sender=owner)

class TestMultipleSchedules:
    
    def test_multiple_schedules_for_one_beneficiary(
        self, vesting, owner, team_member, token
    ):
        """ทดสอบหลาย schedule สำหรับคนเดียว"""
        amount1 = 500_000 * 10**18
        amount2 = 300_000 * 10**18
        
        sid1 = vesting.create_schedule(
            team_member.address, token.address, "TEAM",
            amount1, 0, 12, 24, True, sender=owner
        )
        
        sid2 = vesting.create_schedule(
            team_member.address, token.address, "BONUS",
            amount2, 500, 6, 12, True, sender=owner  # 5% TGE
        )
        
        schedules = vesting.get_beneficiary_schedules(team_member.address)
        assert len(schedules) == 2
```

---

## สรุป

Token Vesting เป็นส่วนสำคัญใน Tokenomics:

| Category | TGE | Cliff | Vesting |
|---------|-----|-------|---------|
| Team | 0% | 12 เดือน | 36 เดือน |
| Advisor | 0% | 6 เดือน | 24 เดือน |
| Seed | 5% | 6 เดือน | 24 เดือน |
| Private | 10% | 3 เดือน | 18 เดือน |
| Public | 25% | 0 เดือน | 12 เดือน |
| Community | 100% | - | - |

---

[⬅️ Part 035: Staking Contract](part_035_staking.md) | [Part 037: Merkle Tree Proofs ➡️](part_037_merkle.md)
