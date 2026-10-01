# Part 062: Algorithmic Stablecoin - CDP-Based (MakerDAO Style)

## สารบัญ (Table of Contents)
1. บทนำ Algorithmic Stablecoin
2. Stablecoin Token (DAI-like)
3. CDP Vault System
4. Price Oracle
5. Liquidation Auction
6. Stability Fee & Savings Rate
7. Tests

---

## 1. บทนำ Algorithmic Stablecoin

**CDP (Collateralized Debt Position):**
- ผู้ใช้วาง collateral (เช่น ETH) เพื่อ mint stablecoin
- ต้อง over-collateralize (เช่น 150%)
- ถ้า collateral value ต่ำกว่า threshold → liquidation

**MakerDAO Model:**
- DAI backed by ETH, WBTC, RWA
- Stability fee = ดอกเบี้ย loan
- DSR (DAI Savings Rate) = ดอกเบี้ยสะสม

---

## 2. Stablecoin Token

```vyper
# @version 0.4.0
# contracts/StableCoin.vy
# USD-pegged stablecoin token

from vyper.interfaces import ERC20

event Transfer:
    sender: indexed(address)
    receiver: indexed(address)
    value: uint256

event Approval:
    owner: indexed(address)
    spender: indexed(address)
    value: uint256

event Minted:
    to: indexed(address)
    amount: uint256

event Burned:
    from_: indexed(address)
    amount: uint256

# ERC20 state
name: public(String[32])
symbol: public(String[8])
decimals: public(uint8)
totalSupply: public(uint256)
balanceOf: public(HashMap[address, uint256])
allowance: public(HashMap[address, HashMap[address, uint256]])

# Minters (vaults that can mint/burn)
minters: public(HashMap[address, bool])
governance: public(address)

@deploy
def __init__(_name: String[32], _symbol: String[8], _governance: address):
    self.name = _name
    self.symbol = _symbol
    self.decimals = 18
    self.governance = _governance

@external
def addMinter(minter: address):
    assert msg.sender == self.governance, "Not governance"
    self.minters[minter] = True

@external
def removeMinter(minter: address):
    assert msg.sender == self.governance, "Not governance"
    self.minters[minter] = False

@external
def mint(to: address, amount: uint256):
    assert self.minters[msg.sender], "Not minter"
    self.totalSupply += amount
    self.balanceOf[to] += amount
    log Transfer(empty(address), to, amount)
    log Minted(to, amount)

@external
def burn(from_: address, amount: uint256):
    assert self.minters[msg.sender], "Not minter"
    assert self.balanceOf[from_] >= amount, "Insufficient balance"
    self.totalSupply -= amount
    self.balanceOf[from_] -= amount
    log Transfer(from_, empty(address), amount)
    log Burned(from_, amount)

@external
def transfer(to: address, amount: uint256) -> bool:
    assert self.balanceOf[msg.sender] >= amount, "Insufficient"
    self.balanceOf[msg.sender] -= amount
    self.balanceOf[to] += amount
    log Transfer(msg.sender, to, amount)
    return True

@external
def transferFrom(from_: address, to: address, amount: uint256) -> bool:
    assert self.balanceOf[from_] >= amount, "Insufficient"
    assert self.allowance[from_][msg.sender] >= amount, "Not approved"
    self.balanceOf[from_] -= amount
    self.balanceOf[to] += amount
    self.allowance[from_][msg.sender] -= amount
    log Transfer(from_, to, amount)
    return True

@external
def approve(spender: address, amount: uint256) -> bool:
    self.allowance[msg.sender][spender] = amount
    log Approval(msg.sender, spender, amount)
    return True
```

---

## 3. CDP Vault System

```vyper
# @version 0.4.0
# contracts/CDPVault.vy
# Collateralized Debt Position Vault

from vyper.interfaces import ERC20

interface IStableCoin:
    def mint(to: address, amount: uint256): nonpayable
    def burn(from_: address, amount: uint256): nonpayable

interface IPriceOracle:
    def getPrice(token: address) -> uint256: view  # USD price scaled 1e18

# Events
event VaultOpened:
    owner: indexed(address)
    vaultId: indexed(uint256)
    collateralType: address

event CollateralDeposited:
    vaultId: indexed(uint256)
    amount: uint256

event CollateralWithdrawn:
    vaultId: indexed(uint256)
    amount: uint256

event DebtGenerated:
    vaultId: indexed(uint256)
    amount: uint256

event DebtRepaid:
    vaultId: indexed(uint256)
    amount: uint256

event VaultLiquidated:
    vaultId: indexed(uint256)
    liquidator: indexed(address)
    collateralSeized: uint256
    debtRepaid: uint256

# Structs
struct CollateralType:
    token: address
    liquidationRatio: uint256   # min collateral ratio (150% = 15000)
    stabilityFee: uint256       # annual fee rate (2% = 200) basis points
    liquidationPenalty: uint256 # penalty % (13% = 1300)
    debtCeiling: uint256        # max debt for this collateral
    totalDebt: uint256          # current total debt

struct Vault:
    owner: address
    collateralType: bytes32
    collateralAmount: uint256   # collateral deposited
    debtAmount: uint256         # stablecoin borrowed
    lastInterestTime: uint256   # for accrual

# State
vaults: HashMap[uint256, Vault]
nextVaultId: uint256
collateralTypes: HashMap[bytes32, CollateralType]

stablecoin: public(address)
oracle: public(address)
governance: public(address)

totalDebt: public(uint256)
globalDebtCeiling: public(uint256)

# Savings
savingsRate: public(uint256)  # DSR: annual % (3% = 300)
savingsBalances: HashMap[address, uint256]  # stablecoin in savings
savingsLastTime: HashMap[address, uint256]
totalSavings: public(uint256)

SCALE: constant(uint256) = 10**18
BASIS: constant(uint256) = 10000
YEAR: constant(uint256) = 365 * 24 * 3600

@deploy
def __init__(
    _stablecoin: address,
    _oracle: address,
    _governance: address,
    _globalDebtCeiling: uint256
):
    self.stablecoin = _stablecoin
    self.oracle = _oracle
    self.governance = _governance
    self.globalDebtCeiling = _globalDebtCeiling
    self.nextVaultId = 1

# ===== Admin =====

@external
def addCollateralType(
    token: address,
    liquidationRatio: uint256,
    stabilityFee: uint256,
    liquidationPenalty: uint256,
    debtCeiling: uint256
):
    assert msg.sender == self.governance, "Not governance"
    key: bytes32 = keccak256(abi.encode(token))
    self.collateralTypes[key] = CollateralType({
        token: token,
        liquidationRatio: liquidationRatio,
        stabilityFee: stabilityFee,
        liquidationPenalty: liquidationPenalty,
        debtCeiling: debtCeiling,
        totalDebt: 0
    })

# ===== Vault Management =====

@external
def openVault(collateralToken: address) -> uint256:
    """เปิด vault ใหม่ใช้ collateral ที่ระบุ"""
    key: bytes32 = keccak256(abi.encode(collateralToken))
    assert self.collateralTypes[key].token != empty(address), "Unknown collateral"
    
    vaultId: uint256 = self.nextVaultId
    self.nextVaultId += 1
    
    self.vaults[vaultId] = Vault({
        owner: msg.sender,
        collateralType: key,
        collateralAmount: 0,
        debtAmount: 0,
        lastInterestTime: block.timestamp
    })
    
    log VaultOpened(msg.sender, vaultId, collateralToken)
    
    return vaultId

@external
def depositCollateral(vaultId: uint256, amount: uint256):
    """ฝาก collateral เพิ่มใน vault"""
    vault: Vault = self.vaults[vaultId]
    assert vault.owner == msg.sender, "Not owner"
    
    colType: CollateralType = self.collateralTypes[vault.collateralType]
    
    assert ERC20(colType.token).transferFrom(msg.sender, self, amount), "Transfer failed"
    
    self.vaults[vaultId].collateralAmount += amount
    
    log CollateralDeposited(vaultId, amount)

@external
def withdrawCollateral(vaultId: uint256, amount: uint256):
    """ถอน collateral ออกจาก vault (ต้อง healthy หลังถอน)"""
    vault: Vault = self.vaults[vaultId]
    assert vault.owner == msg.sender, "Not owner"
    assert vault.collateralAmount >= amount, "Insufficient collateral"
    
    self._accrueInterest(vaultId)
    
    newCollateral: uint256 = vault.collateralAmount - amount
    
    # ตรวจ collateral ratio หลังถอน
    if vault.debtAmount > 0:
        assert self._isHealthy(vaultId, newCollateral, vault.debtAmount), "Below liquidation ratio"
    
    self.vaults[vaultId].collateralAmount = newCollateral
    
    colType: CollateralType = self.collateralTypes[vault.collateralType]
    assert ERC20(colType.token).transfer(msg.sender, amount), "Transfer failed"
    
    log CollateralWithdrawn(vaultId, amount)

@external
def generateDebt(vaultId: uint256, amount: uint256):
    """
    Mint stablecoin โดยใช้ collateral ใน vault
    
    Collateral ratio = collateralValue / debtAmount >= liquidationRatio
    """
    vault: Vault = self.vaults[vaultId]
    assert vault.owner == msg.sender, "Not owner"
    
    self._accrueInterest(vaultId)
    
    colType: CollateralType = self.collateralTypes[vault.collateralType]
    
    newDebt: uint256 = vault.debtAmount + amount
    
    # ตรวจ debt ceiling
    assert colType.totalDebt + amount <= colType.debtCeiling, "Debt ceiling reached"
    assert self.totalDebt + amount <= self.globalDebtCeiling, "Global ceiling"
    
    # ตรวจ collateral ratio
    assert self._isHealthy(vaultId, vault.collateralAmount, newDebt), "Insufficient collateral"
    
    self.vaults[vaultId].debtAmount = newDebt
    self.collateralTypes[vault.collateralType].totalDebt += amount
    self.totalDebt += amount
    
    IStableCoin(self.stablecoin).mint(msg.sender, amount)
    
    log DebtGenerated(vaultId, amount)

@external
def repayDebt(vaultId: uint256, amount: uint256):
    """ชำระหนี้และเผา stablecoin"""
    vault: Vault = self.vaults[vaultId]
    
    self._accrueInterest(vaultId)
    
    repayAmount: uint256 = amount
    if repayAmount > vault.debtAmount:
        repayAmount = vault.debtAmount
    
    IStableCoin(self.stablecoin).burn(msg.sender, repayAmount)
    
    self.vaults[vaultId].debtAmount -= repayAmount
    self.collateralTypes[vault.collateralType].totalDebt -= repayAmount
    self.totalDebt -= repayAmount
    
    log DebtRepaid(vaultId, repayAmount)

# ===== Interest Accrual =====

@internal
def _accrueInterest(vaultId: uint256):
    """
    คำนวณและเพิ่ม stability fee (ดอกเบี้ย)
    
    ดอกเบี้ย = debt * rate * time / YEAR
    """
    vault: Vault = self.vaults[vaultId]
    
    if vault.debtAmount == 0 or vault.lastInterestTime == block.timestamp:
        return
    
    colType: CollateralType = self.collateralTypes[vault.collateralType]
    
    elapsed: uint256 = block.timestamp - vault.lastInterestTime
    
    # Simple interest approximation
    interest: uint256 = vault.debtAmount * colType.stabilityFee * elapsed / BASIS / YEAR
    
    self.vaults[vaultId].debtAmount += interest
    self.collateralTypes[vault.collateralType].totalDebt += interest
    self.totalDebt += interest
    self.vaults[vaultId].lastInterestTime = block.timestamp

# ===== Liquidation =====

@external
def liquidate(vaultId: uint256, debtToRepay: uint256) -> uint256:
    """
    Liquidate vault ที่ collateral ratio ต่ำกว่า threshold
    
    Liquidator จ่าย stablecoin -> ได้ collateral + bonus
    
    Returns: collateralReceived
    """
    vault: Vault = self.vaults[vaultId]
    assert vault.debtAmount > 0, "No debt"
    
    self._accrueInterest(vaultId)
    
    # ตรวจว่า unsafe
    assert not self._isHealthy(vaultId, vault.collateralAmount, vault.debtAmount), "Vault healthy"
    
    colType: CollateralType = self.collateralTypes[vault.collateralType]
    
    # จำกัด debt ที่ repay
    actualRepay: uint256 = debtToRepay
    if actualRepay > vault.debtAmount:
        actualRepay = vault.debtAmount
    
    # คำนวณ collateral ที่ liquidator ได้
    collateralPrice: uint256 = IPriceOracle(self.oracle).getPrice(colType.token)
    
    # collateral = (debtRepaid * penalty%) / price
    collateralAmount: uint256 = actualRepay * (BASIS + colType.liquidationPenalty) / BASIS
    collateralAmount = collateralAmount * SCALE / collateralPrice
    
    if collateralAmount > vault.collateralAmount:
        collateralAmount = vault.collateralAmount
    
    # Burn stablecoin จาก liquidator
    IStableCoin(self.stablecoin).burn(msg.sender, actualRepay)
    
    # อัพเดท vault
    self.vaults[vaultId].debtAmount -= actualRepay
    self.vaults[vaultId].collateralAmount -= collateralAmount
    self.collateralTypes[vault.collateralType].totalDebt -= actualRepay
    self.totalDebt -= actualRepay
    
    # ส่ง collateral ให้ liquidator
    assert ERC20(colType.token).transfer(msg.sender, collateralAmount), "Transfer failed"
    
    log VaultLiquidated(vaultId, msg.sender, collateralAmount, actualRepay)
    
    return collateralAmount

# ===== Helpers =====

@internal
@view
def _isHealthy(vaultId: uint256, collateralAmt: uint256, debtAmt: uint256) -> bool:
    """
    ตรวจสอบว่า vault healthy หรือไม่
    
    Collateral Ratio = (collateral * price) / debt >= liquidationRatio
    """
    if debtAmt == 0:
        return True
    
    vault: Vault = self.vaults[vaultId]
    colType: CollateralType = self.collateralTypes[vault.collateralType]
    
    collateralPrice: uint256 = IPriceOracle(self.oracle).getPrice(colType.token)
    collateralValue: uint256 = collateralAmt * collateralPrice / SCALE
    
    # Ratio ใน basis points
    ratio: uint256 = collateralValue * BASIS / debtAmt
    
    return ratio >= colType.liquidationRatio

@external
@view
def getCollateralRatio(vaultId: uint256) -> uint256:
    """
    ดู collateral ratio ของ vault (basis points)
    15000 = 150%
    """
    vault: Vault = self.vaults[vaultId]
    if vault.debtAmount == 0:
        return max_value(uint256)
    
    colType: CollateralType = self.collateralTypes[vault.collateralType]
    collateralPrice: uint256 = IPriceOracle(self.oracle).getPrice(colType.token)
    collateralValue: uint256 = vault.collateralAmount * collateralPrice / SCALE
    
    return collateralValue * BASIS / vault.debtAmount

@external
@view
def getVault(vaultId: uint256) -> Vault:
    return self.vaults[vaultId]

# ===== Savings Rate =====

@external
def depositSavings(amount: uint256):
    """ฝาก stablecoin เพื่อรับ savings rate (DSR)"""
    self._accrueSavings(msg.sender)
    
    IStableCoin(self.stablecoin).burn(msg.sender, amount)
    self.savingsBalances[msg.sender] += amount
    self.totalSavings += amount

@external
def withdrawSavings(amount: uint256):
    """ถอน savings พร้อมดอกเบี้ย"""
    self._accrueSavings(msg.sender)
    
    bal: uint256 = self.savingsBalances[msg.sender]
    withdrawAmt: uint256 = amount
    if withdrawAmt > bal:
        withdrawAmt = bal
    
    self.savingsBalances[msg.sender] -= withdrawAmt
    self.totalSavings -= withdrawAmt
    
    IStableCoin(self.stablecoin).mint(msg.sender, withdrawAmt)

@internal
def _accrueSavings(user: address):
    """เพิ่มดอกเบี้ย savings"""
    if self.savingsBalances[user] == 0:
        self.savingsLastTime[user] = block.timestamp
        return
    
    elapsed: uint256 = block.timestamp - self.savingsLastTime[user]
    if elapsed == 0:
        return
    
    interest: uint256 = self.savingsBalances[user] * self.savingsRate * elapsed / BASIS / YEAR
    
    self.savingsBalances[user] += interest
    self.totalSavings += interest
    self.totalDebt += interest
    self.savingsLastTime[user] = block.timestamp
```

---

## 4. Liquidation Auction (Dutch Auction)

```vyper
# @version 0.4.0
# contracts/LiquidationAuction.vy
# Dutch Auction สำหรับ liquidated collateral

from vyper.interfaces import ERC20

interface IStableCoin:
    def mint(to: address, amount: uint256): nonpayable
    def burn(from_: address, amount: uint256): nonpayable

# Events
event AuctionStarted:
    auctionId: indexed(uint256)
    collateralToken: address
    collateralAmount: uint256
    startPrice: uint256
    debt: uint256

event AuctionTaken:
    auctionId: indexed(uint256)
    buyer: indexed(address)
    collateralBought: uint256
    pricePaid: uint256

event AuctionSettled:
    auctionId: indexed(uint256)

struct Auction:
    collateralToken: address
    collateralAmount: uint256
    startingPrice: uint256    # เริ่มต้นสูง, ลดตามเวลา
    minimumBid: uint256       # debt ที่ต้องชำระ
    startTime: uint256
    vaultOwner: address
    settled: bool

# State
auctions: HashMap[uint256, Auction]
nextAuctionId: uint256
stablecoin: public(address)
vault: public(address)

# Dutch auction params
priceDecayRate: uint256   # ราคาลดลง % ต่อนาที (scaled 1e18)
auctionDuration: uint256  # นานแค่ไหน

SCALE: constant(uint256) = 10**18

@deploy
def __init__(_stablecoin: address, _vault: address):
    self.stablecoin = _stablecoin
    self.vault = _vault
    self.priceDecayRate = 995 * 10**15  # 0.5% per minute decay (0.995^1)
    self.auctionDuration = 3600 * 4      # 4 hours max

@external
def startAuction(
    collateralToken: address,
    collateralAmount: uint256,
    debt: uint256,
    startPrice: uint256,
    vaultOwner: address
) -> uint256:
    assert msg.sender == self.vault, "Not vault"
    
    auctionId: uint256 = self.nextAuctionId
    self.nextAuctionId += 1
    
    assert ERC20(collateralToken).transferFrom(msg.sender, self, collateralAmount), "Transfer failed"
    
    self.auctions[auctionId] = Auction({
        collateralToken: collateralToken,
        collateralAmount: collateralAmount,
        startingPrice: startPrice,
        minimumBid: debt,
        startTime: block.timestamp,
        vaultOwner: vaultOwner,
        settled: False
    })
    
    log AuctionStarted(auctionId, collateralToken, collateralAmount, startPrice, debt)
    
    return auctionId

@external
@view
def getCurrentPrice(auctionId: uint256) -> uint256:
    """
    ราคา Dutch auction ที่ลดลงตามเวลา
    
    price = startPrice * decayRate^(elapsed_minutes)
    
    Simplified: linear decay สำหรับ on-chain efficiency
    """
    auction: Auction = self.auctions[auctionId]
    elapsed: uint256 = block.timestamp - auction.startTime
    
    if elapsed >= self.auctionDuration:
        return auction.minimumBid * SCALE / auction.collateralAmount  # minimum price
    
    # Linear decay: price = start - (start - min) * elapsed / duration
    startPricePerUnit: uint256 = auction.startingPrice
    minPricePerUnit: uint256 = auction.minimumBid * SCALE / auction.collateralAmount
    
    if startPricePerUnit <= minPricePerUnit:
        return minPricePerUnit
    
    priceRange: uint256 = startPricePerUnit - minPricePerUnit
    decay: uint256 = priceRange * elapsed / self.auctionDuration
    
    return startPricePerUnit - decay

@external
def takeAuction(auctionId: uint256, maxPrice: uint256, collateralWanted: uint256):
    """
    ซื้อ collateral ใน auction
    
    Parameters:
        maxPrice: ราคาสูงสุดต่อหน่วยที่ยอมรับ
        collateralWanted: จำนวน collateral ที่ต้องการ
    """
    auction: Auction = self.auctions[auctionId]
    assert not auction.settled, "Settled"
    
    currentPrice: uint256 = self.getCurrentPrice(auctionId)
    assert currentPrice <= maxPrice, "Price too high"
    
    buyAmount: uint256 = collateralWanted
    if buyAmount > auction.collateralAmount:
        buyAmount = auction.collateralAmount
    
    # จ่าย stablecoin
    payment: uint256 = buyAmount * currentPrice / SCALE
    IStableCoin(self.stablecoin).burn(msg.sender, payment)
    
    # ส่ง collateral
    assert ERC20(auction.collateralToken).transfer(msg.sender, buyAmount), "Transfer failed"
    
    self.auctions[auctionId].collateralAmount -= buyAmount
    
    if self.auctions[auctionId].collateralAmount == 0:
        self.auctions[auctionId].settled = True
        log AuctionSettled(auctionId)
    
    log AuctionTaken(auctionId, msg.sender, buyAmount, payment)

@external
def settleExpiredAuction(auctionId: uint256):
    """
    Settle auction ที่หมดเวลาโดยคืน collateral ให้ vault owner
    """
    auction: Auction = self.auctions[auctionId]
    assert not auction.settled, "Already settled"
    assert block.timestamp >= auction.startTime + self.auctionDuration, "Not expired"
    
    self.auctions[auctionId].settled = True
    
    if auction.collateralAmount > 0:
        assert ERC20(auction.collateralToken).transfer(
            auction.vaultOwner, auction.collateralAmount
        ), "Return failed"
    
    log AuctionSettled(auctionId)
```

---

## 5. Tests

```python
# tests/test_stablecoin.py
import pytest
from brownie import StableCoin, CDPVault, LiquidationAuction, MockERC20, MockOracle, accounts, chain

SCALE = 10**18
BASIS = 10000

@pytest.fixture
def setup():
    owner = accounts[0]
    alice = accounts[1]
    bob = accounts[2]
    
    # Deploy tokens
    weth = MockERC20.deploy("WETH", "WETH", 18, {"from": owner})
    stable = StableCoin.deploy("USD Stablecoin", "USDS", owner.address, {"from": owner})
    oracle = MockOracle.deploy({"from": owner})
    
    # ETH = $2000
    oracle.setPrice(weth.address, 2000 * SCALE, {"from": owner})
    
    vault = CDPVault.deploy(
        stable.address,
        oracle.address,
        owner.address,
        10**24,  # $1M global ceiling
        {"from": owner}
    )
    
    stable.addMinter(vault.address, {"from": owner})
    
    # Add ETH collateral type: 150% ratio, 2% fee, 13% penalty
    vault.addCollateralType(
        weth.address,
        15000,  # 150% liquidation ratio
        200,    # 2% stability fee
        1300,   # 13% liquidation penalty
        10**24, # $1M debt ceiling
        {"from": owner}
    )
    
    weth.mint(alice, 100 * SCALE, {"from": owner})
    weth.mint(bob, 100 * SCALE, {"from": owner})
    
    return owner, alice, bob, weth, stable, oracle, vault

def test_open_vault_and_borrow(setup):
    owner, alice, bob, weth, stable, oracle, vault = setup
    
    # Alice opens vault
    vault_id = vault.openVault(weth.address, {"from": alice}).return_value
    
    # Deposit 10 ETH ($20,000 collateral)
    weth.approve(vault.address, 10 * SCALE, {"from": alice})
    vault.depositCollateral(vault_id, 10 * SCALE, {"from": alice})
    
    # Borrow $10,000 USDS (50% of max at 150% ratio = $13,333 max)
    vault.generateDebt(vault_id, 10000 * SCALE, {"from": alice})
    
    ratio = vault.getCollateralRatio(vault_id)
    print(f"Collateral ratio: {ratio/100:.1f}%")
    assert ratio == 20000  # 200%
    
    alice_stable = stable.balanceOf(alice.address)
    print(f"Alice's USDS: {alice_stable / SCALE:.2f}")
    assert alice_stable == 10000 * SCALE

def test_liquidation(setup):
    owner, alice, bob, weth, stable, oracle, vault = setup
    
    # Alice opens vault with 150% collateral ratio (danger zone)
    vault_id = vault.openVault(weth.address, {"from": alice}).return_value
    
    # Deposit 1 ETH ($2000)
    weth.approve(vault.address, SCALE, {"from": alice})
    vault.depositCollateral(vault_id, SCALE, {"from": alice})
    
    # Borrow $1300 (just above 150% line: $2000 * 100/150 = $1333)
    vault.generateDebt(vault_id, 1300 * SCALE, {"from": alice})
    
    ratio = vault.getCollateralRatio(vault_id)
    print(f"Initial ratio: {ratio/100:.1f}%")
    
    # ETH price drops to $1800 -> ratio drops
    oracle.setPrice(weth.address, 1800 * SCALE, {"from": owner})
    
    ratio_after = vault.getCollateralRatio(vault_id)
    print(f"Ratio after price drop: {ratio_after/100:.1f}%")
    
    # Give Bob stablecoin to liquidate
    stable_mock_mint = 2000 * SCALE
    # Bob needs stable - give him some directly (in real test, he'd buy it)
    # For test purposes, owner mints via vault
    vault2 = vault.openVault(weth.address, {"from": bob}).return_value
    weth.approve(vault.address, 10 * SCALE, {"from": bob})
    vault.depositCollateral(vault2, 10 * SCALE, {"from": bob})
    vault.generateDebt(vault2, 2000 * SCALE, {"from": bob})
    
    bob_stable_before = stable.balanceOf(bob.address)
    stable.approve(vault.address, 2000 * SCALE, {"from": bob})
    
    if ratio_after < 15000:  # Below liquidation threshold
        bob_weth_before = weth.balanceOf(bob.address)
        vault.liquidate(vault_id, 1300 * SCALE, {"from": bob})
        bob_weth_after = weth.balanceOf(bob.address)
        
        print(f"Bob received {(bob_weth_after - bob_weth_before) / SCALE:.4f} ETH")
    else:
        print("Vault still healthy, need bigger price drop")

def test_stability_fee(setup):
    owner, alice, bob, weth, stable, oracle, vault = setup
    
    vault_id = vault.openVault(weth.address, {"from": alice}).return_value
    weth.approve(vault.address, 10 * SCALE, {"from": alice})
    vault.depositCollateral(vault_id, 10 * SCALE, {"from": alice})
    vault.generateDebt(vault_id, 5000 * SCALE, {"from": alice})
    
    initial_debt = vault.getVault(vault_id).debtAmount
    
    # Fast forward 1 year
    chain.sleep(365 * 24 * 3600)
    chain.mine(1)
    
    # Trigger accrual by depositing 0
    vault.repayDebt(vault_id, 0, {"from": alice})
    
    final_debt = vault.getVault(vault_id).debtAmount
    interest = final_debt - initial_debt
    
    print(f"Interest after 1 year: ${interest / SCALE:.2f}")
    print(f"Annual rate: {interest * 10000 / initial_debt / 100:.2f}%")

def test_savings_rate(setup):
    owner, alice, bob, weth, stable, oracle, vault = setup
    
    # Alice mints some stable
    vault_id = vault.openVault(weth.address, {"from": alice}).return_value
    weth.approve(vault.address, 10 * SCALE, {"from": alice})
    vault.depositCollateral(vault_id, 10 * SCALE, {"from": alice})
    vault.generateDebt(vault_id, 3000 * SCALE, {"from": alice})
    
    # Set savings rate to 3%
    # vault.setSavingsRate(300, {"from": owner})  # need governance
    
    # Deposit to savings
    stable.approve(vault.address, 1000 * SCALE, {"from": alice})
    vault.depositSavings(1000 * SCALE, {"from": alice})
    
    chain.sleep(365 * 24 * 3600)
    chain.mine(1)
    
    vault.withdrawSavings(max_value(uint256), {"from": alice})
    
    final_balance = stable.balanceOf(alice.address)
    print(f"Alice's stable after savings: {final_balance / SCALE:.2f}")
```

---

## 6. สรุป

### CDP Stablecoin Concepts:

**1. Over-Collateralization**
- ต้อง lock collateral มากกว่าที่ mint
- 150% ratio หมายถึง $150 collateral per $100 stablecoin

**2. Stability Fee**
- ดอกเบี้ยของ "loan" ที่ mint stablecoin
- เพิ่มขึ้นตามเวลา (compound)
- เป็นรายได้ของ protocol

**3. Liquidation Cascade**
- ราคา collateral ต่ำ → ratio ต่ำ
- ถ้า < liquidation threshold → liquidation
- Liquidator จ่าย debt → รับ collateral + bonus

**4. Price Stability**
- ถ้า stable > $1: ผู้คนเปิด vault มากขึ้น (supply เพิ่ม)
- ถ้า stable < $1: ผู้คนซื้อและชำระหนี้ (supply ลด)
- Savings rate ช่วย attract demand

**5. Dutch Auction vs English Auction**
- Dutch: ราคาสูงแล้วลด (เร็วกว่า, liquidation efficient)
- English: ราคาต่ำแล้วเพิ่ม (ช้า, ยุ่งยากกว่า)
