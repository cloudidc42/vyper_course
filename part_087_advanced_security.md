# Part 087: Advanced Security Patterns

## สารบัญ
1. ERC-777 Reentrancy
2. Read-only Reentrancy
3. Cross-function Reentrancy
4. Price Oracle Manipulation
5. Governance Attacks
6. Flash Loan Attack Patterns
7. Mitigation Strategies

---

## 1. ERC-777 Reentrancy

ERC-777 มี hooks ที่เรียก receiver ก่อนที่จะ update state ทำให้เกิด reentrancy ได้

```vyper
# @version 0.4.0
# @title Vulnerable ERC-777 Vault (DO NOT USE)
# @notice ตัวอย่าง vulnerability จาก ERC-777 hooks

interface IERC777:
    def send(recipient: address, amount: uint256, data: Bytes[256]): nonpayable
    def balanceOf(who: address) -> uint256: view

interface IERC777Recipient:
    def tokensReceived(
        operator: address,
        from_addr: address,
        to: address,
        amount: uint256,
        userData: Bytes[256],
        operatorData: Bytes[256]
    ): nonpayable

# ===== VULNERABLE Contract =====

event Deposit:
    user: indexed(address)
    amount: uint256

event Withdrawal:
    user: indexed(address)
    amount: uint256

token: public(address)
balances: public(HashMap[address, uint256])

@deploy
def __init__(_token: address):
    self.token = _token

@external
def deposit(amount: uint256):
    IERC777(self.token).transferFrom(msg.sender, self, amount)  # conceptually
    self.balances[msg.sender] += amount
    log Deposit(msg.sender, amount)

@external
def withdraw(amount: uint256):
    """
    ⚠️ VULNERABLE: เรียก ERC-777.send() ก่อน update state
    ERC-777 จะเรียก tokensReceived() ซึ่ง attacker สามารถ reenter
    """
    assert self.balances[msg.sender] >= amount, "Insufficient balance"
    
    # BUG: ส่ง tokens ก่อน update state
    IERC777(self.token).send(msg.sender, amount, b"")  # ← attacker reenter here!
    
    # State update มาทีหลัง = too late!
    self.balances[msg.sender] -= amount  # ← attacker already re-entered multiple times
    
    log Withdrawal(msg.sender, amount)
```

---

## 2. Secure Contract ที่ป้องกัน ERC-777

```vyper
# @version 0.4.0
# @title Secure Vault (ERC-777 Protected)
# @notice แก้ไข reentrancy vulnerability

interface IERC777:
    def send(recipient: address, amount: uint256, data: Bytes[256]): nonpayable
    def balanceOf(who: address) -> uint256: view

event Deposit:
    user: indexed(address)
    amount: uint256

event Withdrawal:
    user: indexed(address)
    amount: uint256

token: public(address)
balances: public(HashMap[address, uint256])
_locked: bool  # reentrancy guard

@deploy
def __init__(_token: address):
    self.token = _token

@internal
def _nonReentrant():
    """Reentrancy guard"""
    assert not self._locked, "ReentrancyGuard: reentrant call"
    self._locked = True

@internal
def _unlock():
    self._locked = False

@external
def withdraw(amount: uint256):
    """
    ✅ SECURE: ใช้ CEI pattern + reentrancy guard
    """
    # Pattern: Reentrancy Guard
    self._nonReentrant()
    
    # 1. Checks
    assert self.balances[msg.sender] >= amount, "Insufficient balance"
    
    # 2. Effects (update state FIRST)
    self.balances[msg.sender] -= amount  # ← state updated BEFORE external call
    
    # 3. Interactions (external call LAST)
    IERC777(self.token).send(msg.sender, amount, b"")
    
    self._unlock()
    
    log Withdrawal(msg.sender, amount)
```

---

## 3. Read-only Reentrancy

Read-only reentrancy เกิดขึ้นเมื่อ contract อื่นอ่าน state ของเราในระหว่างที่เราอยู่ในสถานะ inconsistent

```vyper
# @version 0.4.0
# @title AMM ที่มี Read-only Reentrancy vulnerability
# @notice ตัวอย่าง attack vector ใน Curve-style AMMs

from vyper.interfaces import ERC20

interface IOracle:
    def getPrice(pool: address) -> uint256: view

# State
token0: public(address)
token1: public(address)
balance0: public(uint256)
balance1: public(uint256)
totalSupply: public(uint256)
lpBalances: HashMap[address, uint256]

@deploy
def __init__(_token0: address, _token1: address):
    self.token0 = _token0
    self.token1 = _token1

@external
def removeLiquidityETH(lpAmount: uint256, minEth: uint256):
    """
    ⚠️ Vulnerable: removeLiquidity ที่ส่ง ETH ระหว่างที่ state ยัง inconsistent
    
    Attack scenario:
    1. Attacker calls removeLiquidityETH
    2. Contract calculates amounts, updates internal state
    3. Contract sends ETH to attacker
    4. Attacker's receive() calls oracle.getPrice(this)
    5. Oracle reads our balances which are NOW inconsistent
       (ETH already sent but LP not burned yet)
    6. Oracle returns manipulated price
    7. Attacker uses manipulated price in other protocol
    """
    assert self.lpBalances[msg.sender] >= lpAmount, "Insufficient LP"
    
    eth_amount: uint256 = lpAmount * self.balance0 / self.totalSupply
    token_amount: uint256 = lpAmount * self.balance1 / self.totalSupply
    
    # ⚠️ State update - partial
    self.balance0 -= eth_amount
    # ← balance1 ยังไม่ update!
    
    # ⚠️ External call while state is inconsistent
    send(msg.sender, eth_amount)  # attacker can read inconsistent state here
    
    # State update - rest (too late!)
    self.balance1 -= token_amount
    self.lpBalances[msg.sender] -= lpAmount
    self.totalSupply -= lpAmount

@view
@external
def getVirtualPrice() -> uint256:
    """ราคาที่ oracle อ่านได้ - vulnerable ระหว่าง removeLiquidity"""
    if self.totalSupply == 0:
        return 10**18
    return (self.balance0 + self.balance1) * 10**18 / self.totalSupply
```

---

## 4. Secure AMM ที่ป้องกัน Read-only Reentrancy

```vyper
# @version 0.4.0
# @title Secure AMM (Read-only Reentrancy Protected)
# @notice ป้องกัน read-only reentrancy ด้วย lock ที่กระทบ view functions

from vyper.interfaces import ERC20

# Reentrancy lock
_locked: bool

@deploy
def __init__():
    pass

@view
@external
def getVirtualPrice() -> uint256:
    """
    ✅ SECURE: ตรวจสอบ lock ใน view function ด้วย
    ถ้า locked แสดงว่า state อาจ inconsistent
    """
    assert not self._locked, "Pool locked - potential read-only reentrancy"
    
    # ... calculate price
    return 0

@external
def removeLiquidity(lpAmount: uint256):
    """✅ SECURE: lock ตลอด function execution"""
    assert not self._locked
    self._locked = True
    
    # Update ALL state ก่อน
    # ...
    
    # External call (state is now consistent)
    # ...
    
    self._locked = False
```

---

## 5. Cross-function Reentrancy

```vyper
# @version 0.4.0
# @title Cross-function Reentrancy Example
# @notice ตัวอย่าง reentrancy ข้าม functions

# ===== VULNERABLE =====

balances: HashMap[address, uint256]
rewards: HashMap[address, uint256]

@external
def withdrawBalance():
    """Function 1"""
    amount: uint256 = self.balances[msg.sender]
    assert amount > 0
    
    # External call ก่อน update
    raw_call(msg.sender, b"", value=amount, revert_on_failure=False)
    
    self.balances[msg.sender] = 0  # Too late!

@external
def withdrawReward():
    """Function 2"""
    # balances ยัง > 0 เพราะ withdrawBalance ยังไม่ update
    assert self.balances[msg.sender] > 0, "No balance"
    
    reward: uint256 = self.rewards[msg.sender]
    self.rewards[msg.sender] = 0
    
    raw_call(msg.sender, b"", value=reward, revert_on_failure=False)

# ===== ATTACK =====
# 1. Attacker calls withdrawBalance()
# 2. Contract sends ETH to attacker
# 3. Attacker's receive() calls withdrawReward()
# 4. balances[attacker] still > 0 (not updated yet)
# 5. Attacker gets rewards without valid balance!
# 6. Original withdrawBalance() completes, sets balance = 0
```

---

## 6. Price Oracle Manipulation Attack

```vyper
# @version 0.4.0
# @title Oracle Manipulation Vulnerable Contract
# @notice แสดง vulnerability ของ spot price oracle

from vyper.interfaces import ERC20

interface IUniswapV2Pair:
    def getReserves() -> (uint112, uint112, uint32): view
    def token0() -> address: view
    def token1() -> address: view

interface IOracle:
    def getPrice(token: address) -> uint256: view

# ===== VULNERABLE LENDING =====

struct Position:
    collateral: uint256  # ใน token A
    debt: uint256        # ใน USD

pair: public(address)  # Uniswap pair สำหรับ oracle
collateralToken: public(address)

positions: HashMap[address, Position]

@deploy
def __init__(_pair: address, _collateral: address):
    self.pair = _pair
    self.collateralToken = _collateral

@internal
@view
def _getSpotPrice() -> uint256:
    """
    ⚠️ VULNERABLE: ใช้ Uniswap spot price โดยตรง
    Flash loan สามารถ manipulate ได้!
    """
    r0: uint112 = 0
    r1: uint112 = 0
    t: uint32 = 0
    r0, r1, t = IUniswapV2Pair(self.pair).getReserves()
    
    token0: address = IUniswapV2Pair(self.pair).token0()
    
    if token0 == self.collateralToken:
        # price = r1/r0 (USD per collateral)
        return convert(r1, uint256) * 10**18 / convert(r0, uint256)
    else:
        return convert(r0, uint256) * 10**18 / convert(r1, uint256)

@external
def borrow(collateralAmount: uint256, borrowAmount: uint256):
    """
    ⚠️ VULNERABLE: ราคาอาจถูก manipulate ด้วย flash loan
    
    Attack:
    1. Flash loan token A จาก pool
    2. Dump ใน Uniswap pair → ราคาตก
    3. Borrow เยอะมากโดยใช้ collateral น้อย
    4. Repay flash loan
    5. ได้กำไร!
    """
    price: uint256 = self._getSpotPrice()
    collateralValue: uint256 = collateralAmount * price / 10**18
    
    assert collateralValue >= borrowAmount * 150 / 100, "Undercollateralized"
    
    self.positions[msg.sender].collateral += collateralAmount
    self.positions[msg.sender].debt += borrowAmount

# ===== SECURE VERSION =====

# ใช้ TWAP (Time-Weighted Average Price) แทน spot price

struct TWAPObservation:
    timestamp: uint256
    price0Cumulative: uint256
    price1Cumulative: uint256

twapObservations: public(DynArray[TWAPObservation, 1000])
TWAP_PERIOD: constant(uint256) = 1800  # 30 minutes
TWAP_MIN_OBSERVATIONS: constant(uint256) = 2

@external
def updateTWAP():
    """บันทึก TWAP observation"""
    r0: uint112 = 0
    r1: uint112 = 0
    t: uint32 = 0
    r0, r1, t = IUniswapV2Pair(self.pair).getReserves()
    
    obs: TWAPObservation = TWAPObservation({
        timestamp: block.timestamp,
        price0Cumulative: convert(r1, uint256) * 10**18 / convert(r0, uint256),
        price1Cumulative: convert(r0, uint256) * 10**18 / convert(r1, uint256)
    })
    
    self.twapObservations.append(obs)

@internal
@view
def _getTWAPPrice() -> uint256:
    """
    ✅ SECURE: คำนวณ TWAP จาก observations หลายรอบ
    ยากต่อการ manipulate ด้วย flash loan
    """
    count: uint256 = len(self.twapObservations)
    assert count >= TWAP_MIN_OBSERVATIONS, "Insufficient observations"
    
    # หา observations ที่อยู่ใน time window
    oldestIdx: uint256 = 0
    for i: uint256 in range(1000):
        if i >= count:
            break
        if self.twapObservations[count - 1 - i].timestamp < block.timestamp - TWAP_PERIOD:
            oldestIdx = count - 1 - i
            break
    
    recent: TWAPObservation = self.twapObservations[count - 1]
    oldest: TWAPObservation = self.twapObservations[oldestIdx]
    
    timeDelta: uint256 = recent.timestamp - oldest.timestamp
    assert timeDelta >= TWAP_PERIOD / 2, "TWAP period too short"
    
    return (recent.price0Cumulative - oldest.price0Cumulative) / timeDelta
```

---

## 7. Governance Attack Prevention

```vyper
# @version 0.4.0
# @title Flash Loan Governance Attack Prevention
# @notice ป้องกัน flash loan governance attack

from vyper.interfaces import ERC20

event ProposalCreated:
    proposalId: indexed(uint256)
    proposer: indexed(address)
    description: String[256]

event VoteCast:
    voter: indexed(address)
    proposalId: indexed(uint256)
    support: bool
    votes: uint256

struct Proposal:
    proposer: address
    description: String[256]
    forVotes: uint256
    againstVotes: uint256
    startBlock: uint256
    endBlock: uint256
    executed: bool
    canceled: bool
    quorumRequired: uint256
    snapshotBlock: uint256  # ← สำคัญ! ใช้ balance ณ block นี้

struct Receipt:
    hasVoted: bool
    support: bool
    votes: uint256

# State
token: public(address)
proposals: public(DynArray[Proposal, 1000])
receipts: HashMap[uint256, HashMap[address, Receipt]]  # proposalId => voter => receipt

VOTING_PERIOD: constant(uint256) = 50400  # ~7 days in blocks
VOTING_DELAY: constant(uint256) = 1       # 1 block delay before voting
PROPOSAL_THRESHOLD: constant(uint256) = 1000 * 10**18  # 1000 tokens to propose
QUORUM_VOTES: constant(uint256) = 400000 * 10**18     # 400k tokens = 4% quorum

@deploy
def __init__(_token: address):
    self.token = _token

@external
def propose(description: String[256]) -> uint256:
    """
    สร้าง proposal
    ✅ SECURE: snapshot block ถูกกำหนดไว้ล่วงหน้า
    """
    # ต้องมี enough tokens ที่ block ปัจจุบัน
    proposerVotes: uint256 = self._getVotesAtBlock(
        msg.sender,
        block.number - 1
    )
    assert proposerVotes >= PROPOSAL_THRESHOLD, "Insufficient votes to propose"
    
    snapshotBlock: uint256 = block.number  # ← snapshot ณ ตอน propose
    
    proposalId: uint256 = len(self.proposals)
    
    self.proposals.append(Proposal({
        proposer: msg.sender,
        description: description,
        forVotes: 0,
        againstVotes: 0,
        startBlock: snapshotBlock + VOTING_DELAY,
        endBlock: snapshotBlock + VOTING_DELAY + VOTING_PERIOD,
        executed: False,
        canceled: False,
        quorumRequired: QUORUM_VOTES,
        snapshotBlock: snapshotBlock  # ← ใช้ balance ณ block นี้
    }))
    
    log ProposalCreated(proposalId, msg.sender, description)
    
    return proposalId

@external
def castVote(proposalId: uint256, support: bool):
    """
    Vote บน proposal
    ✅ SECURE: ใช้ balance ณ snapshot block ไม่ใช่ current balance
    """
    assert proposalId < len(self.proposals), "Invalid proposal"
    
    proposal: Proposal = self.proposals[proposalId]
    
    assert block.number >= proposal.startBlock, "Voting not started"
    assert block.number <= proposal.endBlock, "Voting ended"
    assert not self.receipts[proposalId][msg.sender].hasVoted, "Already voted"
    
    # ✅ ใช้ votes ณ snapshot block - ป้องกัน flash loan attack!
    votes: uint256 = self._getVotesAtBlock(msg.sender, proposal.snapshotBlock)
    assert votes > 0, "No voting power at snapshot"
    
    self.receipts[proposalId][msg.sender] = Receipt({
        hasVoted: True,
        support: support,
        votes: votes
    })
    
    if support:
        self.proposals[proposalId].forVotes += votes
    else:
        self.proposals[proposalId].againstVotes += votes
    
    log VoteCast(msg.sender, proposalId, support, votes)

@internal
@view
def _getVotesAtBlock(account: address, blockNumber: uint256) -> uint256:
    """
    อ่าน voting power ณ block number ที่กำหนด
    ใช้ checkpoints จาก governance token
    """
    # ใน implementation จริงจะ call token.getPriorVotes()
    return ERC20(self.token).balanceOf(account)  # simplified

@view
@external
def getProposalState(proposalId: uint256) -> uint8:
    """
    0 = Pending
    1 = Active
    2 = Canceled
    3 = Defeated
    4 = Succeeded
    5 = Queued
    6 = Expired
    7 = Executed
    """
    proposal: Proposal = self.proposals[proposalId]
    
    if proposal.canceled:
        return 2
    if proposal.executed:
        return 7
    if block.number <= proposal.startBlock:
        return 0  # Pending
    if block.number <= proposal.endBlock:
        return 1  # Active
    
    # Voting ended - check results
    if proposal.forVotes <= proposal.againstVotes:
        return 3  # Defeated
    if proposal.forVotes < proposal.quorumRequired:
        return 3  # Defeated (no quorum)
    
    return 4  # Succeeded
```

---

## 8. Flash Loan Attack Simulation

```vyper
# @version 0.4.0
# @title Flash Loan Attack Simulator
# @notice ใช้เพื่อ test ว่า contract vulnerable ต่อ flash loan attacks

from vyper.interfaces import ERC20

interface IFlashLoanProvider:
    def flashLoan(receiver: address, token: address, amount: uint256, data: Bytes[1024]) -> bool: nonpayable

interface IVulnerableContract:
    def deposit(amount: uint256): nonpayable
    def borrow(collateral: uint256, amount: uint256): nonpayable
    def getCollateralValue(amount: uint256) -> uint256: view

event AttackExecuted:
    profit: uint256
    borrowed: uint256
    timestamp: uint256

attacker: address
flashLoanProvider: address
targetContract: address
profitToken: address
PRECISION: constant(uint256) = 10**18

@deploy
def __init__(
    _flashLoanProvider: address,
    _targetContract: address,
    _profitToken: address
):
    self.attacker = msg.sender
    self.flashLoanProvider = _flashLoanProvider
    self.targetContract = _targetContract
    self.profitToken = _profitToken

@external
def executeAttack(flashAmount: uint256, minProfit: uint256):
    """
    Execute price oracle manipulation attack:
    1. Flash loan large amount
    2. Manipulate price
    3. Borrow from vulnerable contract
    4. Repay flash loan
    5. Keep profit
    """
    assert msg.sender == self.attacker, "Not attacker"
    
    balanceBefore: uint256 = ERC20(self.profitToken).balanceOf(self)
    
    # Encode attack parameters
    data: Bytes[1024] = _abi_encode(flashAmount, minProfit)
    
    IFlashLoanProvider(self.flashLoanProvider).flashLoan(
        self,
        self.profitToken,
        flashAmount,
        data
    )
    
    balanceAfter: uint256 = ERC20(self.profitToken).balanceOf(self)
    profit: uint256 = balanceAfter - balanceBefore
    
    assert profit >= minProfit, "Insufficient profit"
    
    log AttackExecuted(profit, flashAmount, block.timestamp)

@external
def onFlashLoan(
    initiator: address,
    token: address,
    amount: uint256,
    fee: uint256,
    data: Bytes[1024]
) -> bytes32:
    """Flash loan callback"""
    assert msg.sender == self.flashLoanProvider, "Unauthorized"
    
    # Step 1: ใช้ flash loan เพื่อ manipulate price oracle
    # (dump tokens ใน pool)
    
    # Step 2: Exploit vulnerable contract ที่ใช้ manipulated price
    # IVulnerableContract(self.targetContract).borrow(...)
    
    # Step 3: Repay flash loan + fee
    ERC20(token).approve(self.flashLoanProvider, amount + fee)
    
    return keccak256("ERC3156FlashBorrower.onFlashLoan")
```

---

## 9. Security Checklist

```python
# security_checklist.py
# Checklist สำหรับ audit smart contracts

SECURITY_CHECKLIST = """
=== ADVANCED SECURITY CHECKLIST ===

REENTRANCY:
□ CEI (Checks-Effects-Interactions) pattern ใช้อย่างถูกต้อง?
□ Reentrancy guard บน ทุก state-changing functions?
□ Read-only reentrancy lock บน view functions ที่ sensitive?
□ Cross-function reentrancy ถูก track?
□ ERC-777 hooks considered?

ORACLE:
□ ใช้ TWAP แทน spot price?
□ TWAP period นานพอ (>= 30 min)?
□ Multiple oracle sources?
□ Sanity checks บน oracle responses?
□ Circuit breakers เมื่อ price deviation สูง?

GOVERNANCE:
□ Snapshot voting เพื่อป้องกัน flash loan attacks?
□ Timelock บน critical changes?
□ Quorum requirements เหมาะสม?
□ Proposal threshold ป้องกัน spam?
□ Emergency veto mechanism?

FLASH LOANS:
□ Spot price ไม่ถูกใช้ใน critical calculations?
□ State ไม่ inconsistent ระหว่าง flash loan execution?
□ Multi-block observations สำหรับ key metrics?

PRICE MANIPULATION:
□ Uniswap V2/V3 TWAP ถูกใช้?
□ Chainlink price feeds backup?
□ Price bounds/sanity checks?
□ Deviation thresholds?

ACCESS CONTROL:
□ Role-based access control?
□ Multi-sig สำหรับ admin functions?
□ Timelocks?
□ Emergency pause?

ARITHMETIC:
□ Overflow/underflow checked?
□ Precision loss minimized?
□ Rounding direction favorable?
□ Division before multiplication avoided?
"""

print(SECURITY_CHECKLIST)
```

---

## 10. สรุป Advanced Vulnerabilities

### ERC-777 Reentrancy
- ใช้ CEI pattern เสมอ
- เพิ่ม reentrancy guard
- ระวัง hooks ที่ hidden

### Read-only Reentrancy
- Lock view functions ด้วย
- Ensure state consistent ก่อน external calls
- ระวัง composability กับ protocols อื่น

### Cross-function Reentrancy
- ใช้ single reentrancy guard ที่ใช้ร่วมกัน
- ไม่ assume state ของ functions อื่น

### Oracle Manipulation
- ไม่ใช้ spot price สำหรับ critical logic
- TWAP minimum 30 minutes
- Multiple oracle sources
- Circuit breakers

### Governance Attacks
- Snapshot voting power ล่วงหน้า
- Timelock สำหรับ execution
- Quorum requirements ที่เหมาะสม

---

## แบบฝึกหัด

1. ค้นหา read-only reentrancy ใน Curve Finance contracts
2. เขียน unit test ที่จำลอง flash loan oracle attack
3. Implement TWAP oracle แบบ on-chain
4. ออกแบบ governance ที่ resistant ต่อ attacks
5. Audit code ที่ให้มาและหา vulnerabilities ทั้งหมด

---

*จบ Part 087: Advanced Security Patterns*
