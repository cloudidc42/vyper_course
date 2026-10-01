# Part 057: DAO Treasury Management

## สารบัญ (Table of Contents)
1. บทนำ DAO Treasury
2. Multi-Asset Treasury
3. Allocation Policies
4. Spending Proposals
5. Revenue Distribution
6. Investment Strategies
7. Tests

---

## 1. บทนำ DAO Treasury

DAO Treasury คือกระเป๋าเงินของ DAO ที่:
- เก็บ protocol revenues
- จ่าย grants และ expenses
- ลงทุนเพื่อ grow treasury
- กระจาย profits ให้ token holders

---

## 2. Multi-Asset Treasury

```vyper
# @version 0.4.0
# contracts/DAOTreasury.vy
# DAO Treasury สำหรับจัดการ multi-asset portfolio

from vyper.interfaces import ERC20

# Events
event Deposit:
    token: indexed(address)
    sender: indexed(address)
    amount: uint256

event Withdrawal:
    token: indexed(address)
    recipient: indexed(address)
    amount: uint256
    reason: String[256]

event AllocationUpdated:
    token: indexed(address)
    allocation: uint256

event SpendingProposalCreated:
    proposalId: indexed(uint256)
    requester: indexed(address)
    token: address
    amount: uint256

event SpendingProposalExecuted:
    proposalId: indexed(uint256)
    executor: indexed(address)

event SpendingProposalCanceled:
    proposalId: indexed(uint256)

event RevenueReceived:
    token: indexed(address)
    amount: uint256
    source: String[128]

# Structs
struct TokenBalance:
    token: address
    balance: uint256
    allocation: uint256  # Target allocation % (10000 = 100%)

struct SpendingProposal:
    id: uint256
    proposer: address
    recipient: address
    token: address
    amount: uint256
    description: String[512]
    executesAt: uint256  # timestamp ที่ execute ได้
    executed: bool
    canceled: bool
    approvals: uint256

# State
governance: public(address)
timelock: public(address)

# Supported tokens
supportedTokens: DynArray[address, 50]
isSupported: HashMap[address, bool]
tokenAllocations: HashMap[address, uint256]  # target % (10000 = 100%)

# Spending proposals
spendingProposals: HashMap[uint256, SpendingProposal]
nextProposalId: uint256
proposalApprovers: HashMap[uint256, HashMap[address, bool]]

# Multi-sig
signers: DynArray[address, 20]
isSigner: HashMap[address, bool]
requiredApprovals: uint256

# Spending limits
dailySpendingLimit: HashMap[address, uint256]  # token -> limit per day
spentToday: HashMap[address, uint256]
lastSpendingDay: HashMap[address, uint256]

EXECUTION_DELAY: constant(uint256) = 2 * 24 * 3600  # 2 days

@deploy
def __init__(
    _governance: address,
    _timelock: address,
    _signers: DynArray[address, 20],
    _requiredApprovals: uint256
):
    self.governance = _governance
    self.timelock = _timelock
    
    for signer: address in _signers:
        self.signers.append(signer)
        self.isSigner[signer] = True
    
    self.requiredApprovals = _requiredApprovals
    self.nextProposalId = 1

# ===== Token Management =====

@external
def addSupportedToken(token: address, targetAllocation: uint256):
    """เพิ่ม token ที่ treasury รองรับ"""
    assert msg.sender == self.governance or msg.sender == self.timelock, "Not authorized"
    assert not self.isSupported[token], "Already supported"
    
    self.supportedTokens.append(token)
    self.isSupported[token] = True
    self.tokenAllocations[token] = targetAllocation

@external
def updateAllocation(token: address, newAllocation: uint256):
    """อัพเดท target allocation"""
    assert msg.sender == self.governance or msg.sender == self.timelock, "Not authorized"
    assert self.isSupported[token], "Not supported"
    
    self.tokenAllocations[token] = newAllocation
    
    log AllocationUpdated(token, newAllocation)

@external
@view
def getBalance(token: address) -> uint256:
    """ดู balance ของ token ใน treasury"""
    if token == empty(address):
        return self.balance  # ETH
    return ERC20(token).balanceOf(self)

@external
@view
def getTotalValueInToken(baseToken: address) -> uint256:
    """
    คำนวณมูลค่า treasury ทั้งหมดใน base token
    Simplified: ไม่มี price oracle
    """
    total: uint256 = 0
    for token: address in self.supportedTokens:
        balance: uint256 = ERC20(token).balanceOf(self)
        total += balance  # Simplified: assume all tokens are 1:1
    return total

@external
@view
def getPortfolio() -> DynArray[TokenBalance, 50]:
    """ดู portfolio ทั้งหมด"""
    portfolio: DynArray[TokenBalance, 50] = []
    
    for token: address in self.supportedTokens:
        portfolio.append(TokenBalance({
            token: token,
            balance: ERC20(token).balanceOf(self),
            allocation: self.tokenAllocations[token]
        }))
    
    return portfolio

# ===== Deposit =====

@external
def deposit(token: address, amount: uint256):
    """ฝาก tokens เข้า treasury"""
    assert self.isSupported[token], "Not supported"
    
    assert ERC20(token).transferFrom(msg.sender, self, amount), "Transfer failed"
    
    log Deposit(token, msg.sender, amount)

@external
@payable
def depositETH():
    """ฝาก ETH เข้า treasury"""
    log Deposit(empty(address), msg.sender, msg.value)

# ===== Revenue Distribution =====

@external
def receiveRevenue(token: address, amount: uint256, source: String[128]):
    """บันทึก revenue ที่ได้รับ (สมมติว่า transfer เกิดขึ้นแล้ว)"""
    log RevenueReceived(token, amount, source)

struct RevenueShare:
    recipient: address
    bps: uint256  # basis points (10000 = 100%)

@external
def distributeRevenue(
    token: address,
    recipients: DynArray[address, 50],
    amounts: DynArray[uint256, 50]
):
    """
    กระจาย revenue ให้ recipients
    เรียกโดย governance หรือ timelock เท่านั้น
    """
    assert msg.sender == self.governance or msg.sender == self.timelock, "Not authorized"
    assert len(recipients) == len(amounts), "Length mismatch"
    
    for i: uint256 in range(50):
        if i >= len(recipients):
            break
        
        assert ERC20(token).transfer(recipients[i], amounts[i]), "Transfer failed"
        log Withdrawal(token, recipients[i], amounts[i], "Revenue distribution")

# ===== Spending Proposals =====

@external
def createSpendingProposal(
    recipient: address,
    token: address,
    amount: uint256,
    description: String[512]
) -> uint256:
    """
    สร้าง spending proposal
    ต้องผ่าน multi-sig approval
    """
    assert self.isSigner[msg.sender], "Not signer"
    assert recipient != empty(address), "Zero recipient"
    assert self.isSupported[token], "Token not supported"
    assert amount > 0, "Zero amount"
    
    proposalId: uint256 = self.nextProposalId
    self.nextProposalId += 1
    
    self.spendingProposals[proposalId] = SpendingProposal({
        id: proposalId,
        proposer: msg.sender,
        recipient: recipient,
        token: token,
        amount: amount,
        description: description,
        executesAt: block.timestamp + EXECUTION_DELAY,
        executed: False,
        canceled: False,
        approvals: 1  # Proposer auto-approves
    })
    
    self.proposalApprovers[proposalId][msg.sender] = True
    
    log SpendingProposalCreated(proposalId, msg.sender, token, amount)
    
    return proposalId

@external
def approveSpendingProposal(proposalId: uint256):
    """Approve spending proposal"""
    assert self.isSigner[msg.sender], "Not signer"
    
    proposal: SpendingProposal = self.spendingProposals[proposalId]
    assert not proposal.executed, "Already executed"
    assert not proposal.canceled, "Canceled"
    assert not self.proposalApprovers[proposalId][msg.sender], "Already approved"
    
    self.proposalApprovers[proposalId][msg.sender] = True
    self.spendingProposals[proposalId].approvals += 1

@external
def executeSpendingProposal(proposalId: uint256):
    """Execute spending proposal หลัง delay"""
    proposal: SpendingProposal = self.spendingProposals[proposalId]
    assert not proposal.executed, "Already executed"
    assert not proposal.canceled, "Canceled"
    assert proposal.approvals >= self.requiredApprovals, "Insufficient approvals"
    assert block.timestamp >= proposal.executesAt, "Too early"
    
    # ตรวจสอบ daily limit
    self._checkDailyLimit(proposal.token, proposal.amount)
    
    # ตรวจสอบ balance
    tokenBalance: uint256 = ERC20(proposal.token).balanceOf(self)
    assert tokenBalance >= proposal.amount, "Insufficient balance"
    
    self.spendingProposals[proposalId].executed = True
    
    assert ERC20(proposal.token).transfer(proposal.recipient, proposal.amount), "Transfer failed"
    
    log SpendingProposalExecuted(proposalId, msg.sender)
    log Withdrawal(proposal.token, proposal.recipient, proposal.amount, proposal.description)

@external
def cancelSpendingProposal(proposalId: uint256):
    """Cancel spending proposal"""
    proposal: SpendingProposal = self.spendingProposals[proposalId]
    assert not proposal.executed, "Already executed"
    assert msg.sender == proposal.proposer or self.isSigner[msg.sender], "Not authorized"
    
    self.spendingProposals[proposalId].canceled = True
    
    log SpendingProposalCanceled(proposalId)

@internal
def _checkDailyLimit(token: address, amount: uint256):
    """ตรวจสอบว่าไม่เกิน daily spending limit"""
    limit: uint256 = self.dailySpendingLimit[token]
    if limit == 0:
        return  # No limit
    
    today: uint256 = block.timestamp / 86400
    
    if self.lastSpendingDay[token] < today:
        self.spentToday[token] = 0
        self.lastSpendingDay[token] = today
    
    assert self.spentToday[token] + amount <= limit, "Daily limit exceeded"
    self.spentToday[token] += amount

@external
def setDailySpendingLimit(token: address, limit: uint256):
    """ตั้ง daily spending limit"""
    assert msg.sender == self.governance or msg.sender == self.timelock, "Not authorized"
    self.dailySpendingLimit[token] = limit

# ===== Large Grants (via Governance) =====

@external
def executeGrant(
    recipient: address,
    token: address,
    amount: uint256,
    vestingDuration: uint256,
    description: String[256]
):
    """
    Execute grant ขนาดใหญ่ ต้องผ่าน governance
    """
    assert msg.sender == self.timelock, "Only governance"
    
    assert ERC20(token).transfer(recipient, amount), "Transfer failed"
    
    log Withdrawal(token, recipient, amount, description)

# ===== Investment =====

@external
def invest(
    token: address,
    strategy: address,
    amount: uint256
):
    """ลงทุน assets ใน strategy"""
    assert msg.sender == self.timelock, "Only governance"
    
    assert ERC20(token).approve(strategy, amount), "Approve failed"
    
    # Call strategy deposit (simplified)
    # IStrategy(strategy).deposit(amount)

@external
def divestFromStrategy(
    token: address,
    strategy: address,
    amount: uint256
):
    """ถอน assets จาก strategy"""
    assert msg.sender == self.timelock, "Only governance"
    
    # IStrategy(strategy).withdraw(amount)

# ===== Signer Management =====

@external
def addSigner(newSigner: address):
    """เพิ่ม signer"""
    assert msg.sender == self.timelock, "Only governance"
    assert not self.isSigner[newSigner], "Already signer"
    assert len(self.signers) < 20, "Too many signers"
    
    self.signers.append(newSigner)
    self.isSigner[newSigner] = True

@external
def removeSigner(signer: address):
    """ลบ signer"""
    assert msg.sender == self.timelock, "Only governance"
    assert self.isSigner[signer], "Not signer"
    
    self.isSigner[signer] = False
    newSigners: DynArray[address, 20] = []
    for s: address in self.signers:
        if s != signer:
            newSigners.append(s)
    self.signers = newSigners
    
    # ตรวจสอบว่า required approvals ยังสมเหตุสมผล
    assert self.requiredApprovals <= len(self.signers), "Required approvals too high"

@external
def setRequiredApprovals(newRequired: uint256):
    assert msg.sender == self.timelock, "Only governance"
    assert newRequired > 0, "Zero"
    assert newRequired <= len(self.signers), "Too high"
    self.requiredApprovals = newRequired

# ===== View =====

@external
@view
def getSpendingProposal(proposalId: uint256) -> SpendingProposal:
    return self.spendingProposals[proposalId]

@external
@view
def getSigners() -> DynArray[address, 20]:
    return self.signers
```

---

## 3. Revenue Splitter

```vyper
# @version 0.4.0
# contracts/RevenueSplitter.vy
# แบ่ง protocol revenue ให้ stakeholders

from vyper.interfaces import ERC20

# Events
event SharesUpdated:
    recipient: indexed(address)
    shares: uint256

event RevenueDistributed:
    token: indexed(address)
    totalAmount: uint256
    recipients: uint256

# Struct
struct Recipient:
    account: address
    shares: uint256  # relative shares

# State
recipients: DynArray[Recipient, 50]
totalShares: uint256
governance: public(address)

# Accumulated revenue per token per share
accRevenuePerShare: HashMap[address, uint256]  # token -> accumulated revenue per share
rewardDebt: HashMap[address, HashMap[address, uint256]]  # token -> recipient -> debt
pendingRevenue: HashMap[address, HashMap[address, uint256]]  # token -> recipient -> pending

SCALE: constant(uint256) = 10**18

@deploy
def __init__(
    _governance: address,
    _recipients: DynArray[address, 50],
    _shares: DynArray[uint256, 50]
):
    assert len(_recipients) == len(_shares), "Length mismatch"
    
    self.governance = _governance
    
    for i: uint256 in range(50):
        if i >= len(_recipients):
            break
        
        self.recipients.append(Recipient({
            account: _recipients[i],
            shares: _shares[i]
        }))
        self.totalShares += _shares[i]

@external
def notifyRevenue(token: address, amount: uint256):
    """
    แจ้ง revenue ที่ได้รับ
    ต้องโอน tokens เข้าก่อนเรียก function นี้
    """
    if self.totalShares == 0 or amount == 0:
        return
    
    revenuePerShare: uint256 = amount * SCALE / self.totalShares
    self.accRevenuePerShare[token] += revenuePerShare
    
    log RevenueDistributed(token, amount, len(self.recipients))

@external
def claimRevenue(token: address):
    """Claim pending revenue"""
    pendingAmount: uint256 = self._getPendingRevenue(msg.sender, token)
    
    if pendingAmount > 0:
        # Update reward debt
        for recipient: Recipient in self.recipients:
            if recipient.account == msg.sender:
                self.rewardDebt[token][msg.sender] = self.accRevenuePerShare[token] * recipient.shares / SCALE
                break
        
        assert ERC20(token).transfer(msg.sender, pendingAmount), "Transfer failed"

@internal
@view
def _getPendingRevenue(account: address, token: address) -> uint256:
    """คำนวณ pending revenue"""
    for recipient: Recipient in self.recipients:
        if recipient.account == account:
            return (
                recipient.shares * self.accRevenuePerShare[token] / SCALE -
                self.rewardDebt[token][account]
            )
    return 0

@external
@view
def getPendingRevenue(account: address, token: address) -> uint256:
    return self._getPendingRevenue(account, token)

@external
def updateShares(recipient: address, newShares: uint256):
    """อัพเดท shares ของ recipient"""
    assert msg.sender == self.governance, "Not governance"
    
    for i: uint256 in range(50):
        if i >= len(self.recipients):
            break
        if self.recipients[i].account == recipient:
            self.totalShares = self.totalShares - self.recipients[i].shares + newShares
            self.recipients[i].shares = newShares
            log SharesUpdated(recipient, newShares)
            return
    
    raise "Recipient not found"
```

---

## 4. Budget Management

```vyper
# @version 0.4.0
# contracts/BudgetManager.vy
# จัดการ budget สำหรับ departments/teams

from vyper.interfaces import ERC20

# Events
event BudgetAllocated:
    department: indexed(bytes32)
    token: indexed(address)
    amount: uint256
    period: uint256

event BudgetSpent:
    department: indexed(bytes32)
    token: indexed(address)
    recipient: indexed(address)
    amount: uint256

event BudgetReclaimed:
    department: indexed(bytes32)
    token: indexed(address)
    amount: uint256

# Struct
struct Budget:
    token: address
    totalAmount: uint256
    spentAmount: uint256
    periodStart: uint256
    periodEnd: uint256
    manager: address  # ใครจัดการ budget นี้
    active: bool

# State
governance: public(address)
treasury: public(address)

budgets: HashMap[bytes32, Budget]  # department -> budget
departmentManagers: HashMap[bytes32, address]

@deploy
def __init__(_governance: address, _treasury: address):
    self.governance = _governance
    self.treasury = _treasury

@external
def allocateBudget(
    department: bytes32,
    token: address,
    amount: uint256,
    periodDuration: uint256,
    manager: address
):
    """
    จัดสรร budget ให้ department
    เรียกโดย governance
    """
    assert msg.sender == self.governance, "Not governance"
    
    assert ERC20(token).transferFrom(self.treasury, self, amount), "Transfer failed"
    
    period: uint256 = block.timestamp + periodDuration
    
    self.budgets[department] = Budget({
        token: token,
        totalAmount: amount,
        spentAmount: 0,
        periodStart: block.timestamp,
        periodEnd: period,
        manager: manager,
        active: True
    })
    
    log BudgetAllocated(department, token, amount, period)

@external
def spendBudget(
    department: bytes32,
    recipient: address,
    amount: uint256,
    reason: String[256]
):
    """
    ใช้ budget ของ department
    เรียกโดย department manager
    """
    budget: Budget = self.budgets[department]
    assert budget.active, "Budget not active"
    assert msg.sender == budget.manager, "Not manager"
    assert block.timestamp <= budget.periodEnd, "Period ended"
    assert budget.spentAmount + amount <= budget.totalAmount, "Exceeds budget"
    
    self.budgets[department].spentAmount += amount
    
    assert ERC20(budget.token).transfer(recipient, amount), "Transfer failed"
    
    log BudgetSpent(department, budget.token, recipient, amount)

@external
def reclaimUnspentBudget(department: bytes32):
    """ดึง unspent budget กลับ treasury"""
    budget: Budget = self.budgets[department]
    assert msg.sender == self.governance, "Not governance"
    assert budget.active, "Not active"
    assert block.timestamp > budget.periodEnd, "Period not ended"
    
    unspent: uint256 = budget.totalAmount - budget.spentAmount
    
    if unspent > 0:
        assert ERC20(budget.token).transfer(self.treasury, unspent), "Transfer failed"
    
    self.budgets[department].active = False
    
    log BudgetReclaimed(department, budget.token, unspent)

@external
@view
def getRemainingBudget(department: bytes32) -> uint256:
    budget: Budget = self.budgets[department]
    if not budget.active or block.timestamp > budget.periodEnd:
        return 0
    return budget.totalAmount - budget.spentAmount
```

---

## 5. Tests

```python
# tests/test_treasury.py
import pytest
from brownie import DAOTreasury, RevenueSplitter, MockERC20, accounts, chain

@pytest.fixture
def setup():
    owner = accounts[0]
    alice = accounts[1]
    bob = accounts[2]
    charlie = accounts[3]
    
    usdc = MockERC20.deploy("USDC", "USDC", 6, {"from": owner})
    weth = MockERC20.deploy("WETH", "WETH", 18, {"from": owner})
    
    treasury = DAOTreasury.deploy(
        owner.address,  # governance
        owner.address,  # timelock (simplified)
        [alice.address, bob.address, charlie.address],
        2,  # 2 of 3 multisig
        {"from": owner}
    )
    
    treasury.addSupportedToken(usdc.address, 7000, {"from": owner})  # 70%
    treasury.addSupportedToken(weth.address, 3000, {"from": owner})  # 30%
    
    # Fund treasury
    usdc.mint(owner, 10**10, {"from": owner})
    usdc.approve(treasury.address, 10**10, {"from": owner})
    treasury.deposit(usdc.address, 10**10, {"from": owner})
    
    return owner, alice, bob, charlie, usdc, weth, treasury

def test_deposit(setup):
    owner, alice, bob, charlie, usdc, weth, treasury = setup
    
    balance = treasury.getBalance(usdc.address)
    assert balance == 10**10
    print(f"Treasury USDC balance: {balance}")

def test_spending_proposal(setup):
    owner, alice, bob, charlie, usdc, weth, treasury = setup
    
    # Alice creates proposal
    pid = treasury.createSpendingProposal(
        bob.address,
        usdc.address,
        10**8,  # 100 USDC
        "Grant for development",
        {"from": alice}
    ).return_value
    
    # Bob approves
    treasury.approveSpendingProposal(pid, {"from": bob})
    
    # Wait for delay
    chain.sleep(2 * 24 * 3600 + 1)
    chain.mine(1)
    
    before = usdc.balanceOf(bob.address)
    treasury.executeSpendingProposal(pid, {"from": owner})
    after = usdc.balanceOf(bob.address)
    
    assert after - before == 10**8
    print(f"Bob received: {after - before} USDC")

def test_revenue_distribution(setup):
    owner, alice, bob, charlie, usdc, weth, treasury = setup
    
    recipients = [alice.address, bob.address]
    amounts = [5 * 10**8, 5 * 10**8]  # 500 USDC each
    
    before_alice = usdc.balanceOf(alice.address)
    before_bob = usdc.balanceOf(bob.address)
    
    treasury.distributeRevenue(usdc.address, recipients, amounts, {"from": owner})
    
    assert usdc.balanceOf(alice.address) - before_alice == 5 * 10**8
    assert usdc.balanceOf(bob.address) - before_bob == 5 * 10**8

def test_revenue_splitter(setup):
    owner, alice, bob, charlie, usdc, weth, treasury = setup
    
    splitter = RevenueSplitter.deploy(
        owner.address,
        [alice.address, bob.address],
        [7000, 3000],  # 70/30 split
        {"from": owner}
    )
    
    # Send revenue
    revenue = 10**8  # 100 USDC
    usdc.mint(owner, revenue, {"from": owner})
    usdc.transfer(splitter.address, revenue, {"from": owner})
    splitter.notifyRevenue(usdc.address, revenue, {"from": owner})
    
    # Check pending
    alice_pending = splitter.getPendingRevenue(alice.address, usdc.address)
    bob_pending = splitter.getPendingRevenue(bob.address, usdc.address)
    
    print(f"Alice pending: {alice_pending}")
    print(f"Bob pending: {bob_pending}")
    
    assert alice_pending > bob_pending  # Alice has 70%
```

---

## 6. สรุป

### Treasury Best Practices:

**1. Multi-sig for Security**
- ต้องการ N-of-M signatures สำหรับ spending
- ป้องกัน single point of failure

**2. Spending Limits**
- Daily limits ป้องกัน large unexpected withdrawals
- Tiered limits ตาม amount

**3. Budget Allocation**
- จัดสรร budget ให้ teams ล่วงหน้า
- ป้องกัน ad-hoc spending

**4. Revenue Distribution**
- Automated distribution ตาม shares
- Transparent สำหรับ token holders

**5. Investment**
- Diversify treasury assets
- ลงทุนใน DeFi protocols เพื่อ yield
- Risk management: maintain liquid reserves
