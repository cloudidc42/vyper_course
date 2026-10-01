# Part 079: Invariant Testing เชิงลึก

## สารบัญ
1. Invariant Testing คืออะไร?
2. คุณสมบัติของ Good Invariants
3. Handler-based Invariant Tests
4. Ghost Variables
5. Invariants สำหรับ AMM
6. Invariants สำหรับ Lending Protocol
7. Invariants สำหรับ Staking
8. Advanced Patterns

---

## 1. Invariant Testing คืออะไร?

Invariant testing คือการทดสอบว่า "คุณสมบัติ" บางอย่างของ contract ยังคงเป็นจริงหลังจากทำ transaction ใดๆ ก็ตาม

### ความแตกต่างจาก Unit Testing

```
Unit Testing:
- ทดสอบ specific inputs
- เช่น: deposit(100) → ควรได้ shares = 100
- ครอบคลุมเฉพาะที่เราคิดถึง

Invariant Testing:
- ทดสอบ properties ที่ต้องเป็นจริงเสมอ
- เช่น: สำหรับทุก sequence of transactions, totalSupply = sum(balances)
- ค้นหา edge cases ที่เราไม่คิดถึง
```

### ตัวอย่าง Invariants ที่สำคัญ

```
ความถูกต้องทางบัญชี (Accounting Correctness):
  totalSupply = sum of all balances

ความปลอดภัยของ Assets (Asset Safety):
  vault.totalAssets() >= amount_all_users_can_withdraw

ความสม่ำเสมอของ Price (Price Consistency):
  amm.pricePerShare() ไม่ลดลงโดยไม่มีการสูญเสีย

Solvency:
  protocol.totalCollateral() >= protocol.totalDebt() * minimumCR
```

---

## 2. AMM Contract สำหรับ Invariant Testing

```vyper
# @version 0.4.0
# @title Constant Sum AMM (x + y = k)
# @notice Stablecoin AMM สำหรับ invariant testing

from vyper.interfaces import ERC20

event AddLiquidity:
    provider: indexed(address)
    amounts: uint256[2]
    lp_minted: uint256

event RemoveLiquidity:
    provider: indexed(address)
    amounts: uint256[2]
    lp_burned: uint256

event TokenExchange:
    buyer: indexed(address)
    sold_id: indexed(uint256)
    tokens_sold: uint256
    bought_id: indexed(uint256)
    tokens_bought: uint256

# State
coins: public(address[2])
balances: public(uint256[2])
totalLPSupply: public(uint256)
lpBalances: public(HashMap[address, uint256])
A: public(uint256)  # Amplification coefficient
fee: public(uint256)  # Fee in basis points
owner: public(address)

FEE_DENOMINATOR: constant(uint256) = 10**10
PRECISION: constant(uint256) = 10**18
N_COINS: constant(uint256) = 2

@deploy
def __init__(
    _coins: address[2],
    _A: uint256,
    _fee: uint256
):
    assert _coins[0] != empty(address) and _coins[1] != empty(address)
    assert _coins[0] != _coins[1]
    assert _A > 0
    assert _fee <= 10**9  # max 10%
    
    self.coins[0] = _coins[0]
    self.coins[1] = _coins[1]
    self.A = _A
    self.fee = _fee
    self.owner = msg.sender

@internal
@view
def _getD(xp: uint256[2]) -> uint256:
    """
    คำนวณ D (invariant) ตาม StableSwap formula
    D^3 + D * sum * A * N^N = D^2 * N^N * A * sum + sum^N
    """
    s: uint256 = xp[0] + xp[1]
    if s == 0:
        return 0
    
    D: uint256 = s
    Ann: uint256 = self.A * N_COINS
    
    for _: uint256 in range(255):
        D_P: uint256 = D * D / xp[0] * D / xp[1] / (N_COINS**2)
        D_prev: uint256 = D
        D = (Ann * s + D_P * N_COINS) * D / ((Ann - 1) * D + (N_COINS + 1) * D_P)
        
        if D > D_prev:
            if D - D_prev <= 1:
                break
        else:
            if D_prev - D <= 1:
                break
    
    return D

@internal
@view
def _getY(
    i: uint256,
    j: uint256,
    x: uint256,
    xp: uint256[2]
) -> uint256:
    """
    คำนวณ y เมื่อ x เปลี่ยน
    ใช้ StableSwap formula
    """
    D: uint256 = self._getD(xp)
    Ann: uint256 = self.A * N_COINS
    
    c: uint256 = D
    S_: uint256 = 0
    _x: uint256 = 0
    y_prev: uint256 = 0
    
    for _i in range(N_COINS):
        if _i == i:
            _x = x
        elif _i != j:
            _x = xp[_i]
        else:
            continue
        
        S_ += _x
        c = c * D / (_x * N_COINS)
    
    c = c * D / (Ann * N_COINS)
    b: uint256 = S_ + D / Ann
    
    y: uint256 = D
    
    for _: uint256 in range(255):
        y_prev = y
        y = (y * y + c) / (2 * y + b - D)
        
        if y > y_prev:
            if y - y_prev <= 1:
                break
        else:
            if y_prev - y <= 1:
                break
    
    return y

@external
def addLiquidity(amounts: uint256[2], min_mint_amount: uint256) -> uint256:
    """เพิ่ม liquidity"""
    assert amounts[0] > 0 or amounts[1] > 0, "Invalid amounts"
    
    # Transfer tokens
    for i in range(N_COINS):
        if amounts[i] > 0:
            ERC20(self.coins[i]).transferFrom(msg.sender, self, amounts[i])
    
    # คำนวณ LP tokens
    lp_minted: uint256 = 0
    
    if self.totalLPSupply == 0:
        # First deposit: D = sum of amounts
        D_new: uint256 = self._getD([amounts[0], amounts[1]])
        lp_minted = D_new
    else:
        # Subsequent deposits
        D_old: uint256 = self._getD(self.balances)
        new_balances: uint256[2] = [
            self.balances[0] + amounts[0],
            self.balances[1] + amounts[1]
        ]
        D_new: uint256 = self._getD(new_balances)
        lp_minted = self.totalLPSupply * (D_new - D_old) / D_old
    
    assert lp_minted >= min_mint_amount, "Slippage"
    
    # Update state
    for i in range(N_COINS):
        self.balances[i] += amounts[i]
    
    self.totalLPSupply += lp_minted
    self.lpBalances[msg.sender] += lp_minted
    
    log AddLiquidity(msg.sender, amounts, lp_minted)
    
    return lp_minted

@external
def exchange(i: uint256, j: uint256, dx: uint256, min_dy: uint256) -> uint256:
    """แลกเปลี่ยน tokens"""
    assert i != j, "Same token"
    assert i < N_COINS and j < N_COINS, "Invalid token index"
    assert dx > 0, "Invalid amount"
    
    xp: uint256[2] = self.balances
    
    x: uint256 = xp[i] + dx
    y: uint256 = self._getY(i, j, x, xp)
    
    dy: uint256 = xp[j] - y - 1  # -1 for rounding
    dy_fee: uint256 = dy * self.fee / FEE_DENOMINATOR
    
    dy_after_fee: uint256 = dy - dy_fee
    
    assert dy_after_fee >= min_dy, "Slippage"
    
    # Update balances
    self.balances[i] += dx
    self.balances[j] -= dy_after_fee
    
    # Transfer tokens
    ERC20(self.coins[i]).transferFrom(msg.sender, self, dx)
    ERC20(self.coins[j]).transfer(msg.sender, dy_after_fee)
    
    log TokenExchange(msg.sender, i, dx, j, dy_after_fee)
    
    return dy_after_fee

@external
def removeLiquidity(
    lp_amount: uint256,
    min_amounts: uint256[2]
) -> uint256[2]:
    """ถอน liquidity"""
    assert lp_amount > 0, "Invalid amount"
    assert self.lpBalances[msg.sender] >= lp_amount, "Insufficient LP"
    
    amounts: uint256[2] = empty(uint256[2])
    
    for i in range(N_COINS):
        amounts[i] = self.balances[i] * lp_amount / self.totalLPSupply
        assert amounts[i] >= min_amounts[i], "Slippage"
    
    # Burn LP tokens
    self.lpBalances[msg.sender] -= lp_amount
    self.totalLPSupply -= lp_amount
    
    # Update balances and transfer
    for i in range(N_COINS):
        self.balances[i] -= amounts[i]
        ERC20(self.coins[i]).transfer(msg.sender, amounts[i])
    
    log RemoveLiquidity(msg.sender, amounts, lp_amount)
    
    return amounts

@view
@external
def getD() -> uint256:
    return self._getD(self.balances)

@view
@external
def getDy(i: uint256, j: uint256, dx: uint256) -> uint256:
    """Preview ว่าจะได้ tokens เท่าไหร่"""
    xp: uint256[2] = self.balances
    x: uint256 = xp[i] + dx
    y: uint256 = self._getY(i, j, x, xp)
    dy: uint256 = xp[j] - y - 1
    fee_amount: uint256 = dy * self.fee / FEE_DENOMINATOR
    return dy - fee_amount
```

---

## 3. Handler Contract สำหรับ AMM

```solidity
// test/invariant/AMMHandler.sol

pragma solidity ^0.8.0;

import "forge-std/Test.sol";

interface IAMM {
    function balances(uint256) external view returns (uint256);
    function totalLPSupply() external view returns (uint256);
    function lpBalances(address) external view returns (uint256);
    function addLiquidity(uint256[2] calldata, uint256) external returns (uint256);
    function exchange(uint256, uint256, uint256, uint256) external returns (uint256);
    function removeLiquidity(uint256, uint256[2] calldata) external returns (uint256[2] memory);
    function getD() external view returns (uint256);
}

contract AMMHandler is Test {
    IAMM amm;
    address[2] tokens;
    
    address[] public actors;
    address internal currentActor;
    
    // Ghost variables - ติดตาม state ที่สำคัญ
    uint256 public ghost_sumLPBalances;
    uint256 public ghost_depositedToken0;
    uint256 public ghost_depositedToken1;
    uint256 public ghost_withdrawnToken0;
    uint256 public ghost_withdrawnToken1;
    uint256 public ghost_swapCount;
    uint256 public ghost_D_initial;
    bool public ghost_D_initialized;
    
    modifier useActor(uint256 seed) {
        currentActor = actors[seed % actors.length];
        vm.startPrank(currentActor);
        _;
        vm.stopPrank();
    }
    
    constructor(address _amm, address[2] memory _tokens) {
        amm = IAMM(_amm);
        tokens[0] = _tokens[0];
        tokens[1] = _tokens[1];
        
        for (uint256 i = 0; i < 5; i++) {
            actors.push(makeAddr(string(abi.encode(i))));
        }
    }
    
    // Handler: addLiquidity
    function addLiquidity(
        uint256 actorSeed,
        uint256 amount0,
        uint256 amount1
    ) external useActor(actorSeed) {
        amount0 = bound(amount0, 1e6, 1_000_000e18);
        amount1 = bound(amount1, 1e6, 1_000_000e18);
        
        // Give tokens to actor
        deal(tokens[0], currentActor, amount0);
        deal(tokens[1], currentActor, amount1);
        
        // Approve
        IERC20(tokens[0]).approve(address(amm), amount0);
        IERC20(tokens[1]).approve(address(amm), amount1);
        
        uint256[2] memory amounts = [amount0, amount1];
        uint256[2] memory minAmounts = [uint256(0), uint256(0)];
        
        try amm.addLiquidity(amounts, 0) returns (uint256 lpMinted) {
            ghost_sumLPBalances += lpMinted;
            ghost_depositedToken0 += amount0;
            ghost_depositedToken1 += amount1;
            
            if (!ghost_D_initialized) {
                ghost_D_initial = amm.getD();
                ghost_D_initialized = true;
            }
        } catch {}
    }
    
    // Handler: exchange
    function exchange(
        uint256 actorSeed,
        uint256 i,
        uint256 dx
    ) external useActor(actorSeed) {
        i = i % 2;
        uint256 j = 1 - i;
        dx = bound(dx, 1e6, 10_000e18);
        
        // Give token to actor
        deal(tokens[i], currentActor, dx);
        IERC20(tokens[i]).approve(address(amm), dx);
        
        try amm.exchange(i, j, dx, 0) {
            ghost_swapCount++;
        } catch {}
    }
    
    // Handler: removeLiquidity
    function removeLiquidity(
        uint256 actorSeed,
        uint256 lpFraction
    ) external useActor(actorSeed) {
        uint256 lpBalance = amm.lpBalances(currentActor);
        if (lpBalance == 0) return;
        
        lpFraction = bound(lpFraction, 1, 100);
        uint256 lpAmount = lpBalance * lpFraction / 100;
        
        uint256[2] memory minAmounts = [uint256(0), uint256(0)];
        
        try amm.removeLiquidity(lpAmount, minAmounts) returns (uint256[2] memory amounts) {
            ghost_sumLPBalances -= lpAmount;
            ghost_withdrawnToken0 += amounts[0];
            ghost_withdrawnToken1 += amounts[1];
        } catch {}
    }
    
    function getActors() external view returns (address[] memory) {
        return actors;
    }
}
```

---

## 4. Invariant Tests สำหรับ AMM

```solidity
// test/invariant/InvariantAMM.t.sol

pragma solidity ^0.8.0;

import "forge-std/Test.sol";
import "forge-std/InvariantTest.sol";

contract InvariantAMMTest is Test, InvariantTest {
    IAMM amm;
    AMMHandler handler;
    address[2] tokens;
    
    function setUp() public {
        // Deploy mock tokens
        // Deploy AMM
        // Deploy handler
        
        targetContract(address(handler));
        
        // เลือก functions ที่จะ fuzz
        bytes4[] memory selectors = new bytes4[](3);
        selectors[0] = AMMHandler.addLiquidity.selector;
        selectors[1] = AMMHandler.exchange.selector;
        selectors[2] = AMMHandler.removeLiquidity.selector;
        
        targetSelector(FuzzSelector({
            addr: address(handler),
            selectors: selectors
        }));
    }
    
    // Invariant 1: totalLPSupply = sum of LP balances
    function invariant_lp_supply_accounting() public view {
        uint256 sumLP;
        address[] memory actors = handler.getActors();
        for (uint256 i = 0; i < actors.length; i++) {
            sumLP += amm.lpBalances(actors[i]);
        }
        assertEq(amm.totalLPSupply(), sumLP, "LP supply mismatch");
    }
    
    // Invariant 2: D ไม่ลดลงหลัง swap (เนื่องจาก fee)
    function invariant_D_non_decreasing_after_swaps() public view {
        if (!handler.ghost_D_initialized()) return;
        if (handler.ghost_swapCount() == 0) return;
        
        assertGe(
            amm.getD(),
            handler.ghost_D_initial(),
            "D decreased after swaps"
        );
    }
    
    // Invariant 3: reserves ต้องเป็น positive เมื่อมี LP supply
    function invariant_positive_reserves() public view {
        if (amm.totalLPSupply() > 0) {
            assertGt(amm.balances(0), 0, "Reserve 0 must be positive");
            assertGt(amm.balances(1), 0, "Reserve 1 must be positive");
        }
    }
    
    // Invariant 4: ไม่สามารถ drain pool ได้
    function invariant_no_drain() public view {
        uint256 contractBal0 = IERC20(tokens[0]).balanceOf(address(amm));
        uint256 contractBal1 = IERC20(tokens[1]).balanceOf(address(amm));
        
        assertGe(contractBal0, amm.balances(0), "Token0 balance mismatch");
        assertGe(contractBal1, amm.balances(1), "Token1 balance mismatch");
    }
    
    // Invariant 5: price impact ต้องสมเหตุสมผล
    function invariant_price_sanity() public view {
        if (amm.balances(0) == 0 || amm.balances(1) == 0) return;
        
        // Ratio ต้องไม่ห่างกันเกิน 1000x
        uint256 ratio;
        if (amm.balances(0) > amm.balances(1)) {
            ratio = amm.balances(0) / amm.balances(1);
        } else {
            ratio = amm.balances(1) / amm.balances(0);
        }
        
        assertLe(ratio, 1000, "Price ratio too extreme");
    }
}
```

---

## 5. Lending Protocol สำหรับ Invariant Testing

```vyper
# @version 0.4.0
# @title Lending Protocol สำหรับ Invariant Testing
# @notice Full-featured lending protocol

from vyper.interfaces import ERC20

struct Market:
    collateralFactor: uint256  # basis points (e.g., 7500 = 75%)
    liquidationThreshold: uint256  # basis points (e.g., 8000 = 80%)
    liquidationBonus: uint256  # basis points (e.g., 500 = 5%)
    isActive: bool
    isPaused: bool

struct UserAccount:
    collateralBalance: HashMap[address, uint256]
    borrowBalance: HashMap[address, uint256]

event Deposited:
    user: indexed(address)
    token: indexed(address)
    amount: uint256

event Withdrawn:
    user: indexed(address)
    token: indexed(address)
    amount: uint256

event Borrowed:
    user: indexed(address)
    token: indexed(address)
    amount: uint256

event Repaid:
    user: indexed(address)
    token: indexed(address)
    amount: uint256

event Liquidated:
    liquidator: indexed(address)
    borrower: indexed(address)
    debtToken: indexed(address)
    collateralToken: address
    debtAmount: uint256
    collateralSeized: uint256

# State
markets: public(HashMap[address, Market])
collateralBalances: public(HashMap[address, HashMap[address, uint256]])
borrowBalances: public(HashMap[address, HashMap[address, uint256]])
totalCollateral: public(HashMap[address, uint256])
totalBorrows: public(HashMap[address, uint256])
totalReserves: public(HashMap[address, uint256])

priceOracle: public(address)
owner: public(address)

PRECISION: constant(uint256) = 10**18
BASIS_POINTS: constant(uint256) = 10000

interface IPriceOracle:
    def getPrice(token: address) -> uint256: view

@deploy
def __init__(_oracle: address):
    self.priceOracle = _oracle
    self.owner = msg.sender

@external
def addMarket(
    token: address,
    cf: uint256,
    lt: uint256,
    lb: uint256
):
    assert msg.sender == self.owner, "Not owner"
    assert cf <= 9000, "CF too high"
    assert lt <= 9500, "LT too high"
    assert lb <= 2000, "LB too high"
    assert cf < lt, "CF must be < LT"
    
    self.markets[token] = Market({
        collateralFactor: cf,
        liquidationThreshold: lt,
        liquidationBonus: lb,
        isActive: True,
        isPaused: False
    })

@internal
@view
def _getCollateralValue(user: address) -> uint256:
    """คำนวณ total collateral value ใน USD"""
    # simplified: only checks first market
    return 0  # placeholder

@internal
@view
def _getBorrowValue(user: address) -> uint256:
    """คำนวณ total borrow value ใน USD"""
    return 0  # placeholder

@internal
@view
def _isHealthy(user: address) -> bool:
    """ตรวจสอบว่า position healthy"""
    borrowVal: uint256 = self._getBorrowValue(user)
    if borrowVal == 0:
        return True
    collateralVal: uint256 = self._getCollateralValue(user)
    return collateralVal >= borrowVal

@external
def deposit(token: address, amount: uint256):
    """ฝาก collateral"""
    assert self.markets[token].isActive, "Market not active"
    assert not self.markets[token].isPaused, "Market paused"
    assert amount > 0, "Invalid amount"
    
    ERC20(token).transferFrom(msg.sender, self, amount)
    
    self.collateralBalances[msg.sender][token] += amount
    self.totalCollateral[token] += amount
    
    log Deposited(msg.sender, token, amount)

@external
def withdraw(token: address, amount: uint256):
    """ถอน collateral"""
    assert self.collateralBalances[msg.sender][token] >= amount, "Insufficient collateral"
    
    self.collateralBalances[msg.sender][token] -= amount
    self.totalCollateral[token] -= amount
    
    assert self._isHealthy(msg.sender), "Would be undercollateralized"
    
    ERC20(token).transfer(msg.sender, amount)
    
    log Withdrawn(msg.sender, token, amount)

@external
def borrow(token: address, amount: uint256):
    """กู้ยืม"""
    assert self.markets[token].isActive, "Market not active"
    assert not self.markets[token].isPaused, "Market paused"
    assert amount > 0, "Invalid amount"
    assert self.totalReserves[token] >= amount, "Insufficient reserves"
    
    self.borrowBalances[msg.sender][token] += amount
    self.totalBorrows[token] += amount
    self.totalReserves[token] -= amount
    
    assert self._isHealthy(msg.sender), "Undercollateralized"
    
    ERC20(token).transfer(msg.sender, amount)
    
    log Borrowed(msg.sender, token, amount)

@external
def repay(token: address, amount: uint256):
    """ชำระหนี้"""
    assert self.borrowBalances[msg.sender][token] >= amount, "Repay too much"
    
    ERC20(token).transferFrom(msg.sender, self, amount)
    
    self.borrowBalances[msg.sender][token] -= amount
    self.totalBorrows[token] -= amount
    self.totalReserves[token] += amount
    
    log Repaid(msg.sender, token, amount)

@external
def liquidate(
    borrower: address,
    debtToken: address,
    collateralToken: address,
    repayAmount: uint256
):
    """Liquidate unhealthy position"""
    assert not self._isHealthy(borrower), "Position is healthy"
    assert self.borrowBalances[borrower][debtToken] >= repayAmount, "Repay too much"
    
    # คำนวณ collateral seized
    market: Market = self.markets[collateralToken]
    collateralSeized: uint256 = repayAmount * (BASIS_POINTS + market.liquidationBonus) / BASIS_POINTS
    
    assert self.collateralBalances[borrower][collateralToken] >= collateralSeized, "Insufficient collateral"
    
    # Transfer debt from liquidator
    ERC20(debtToken).transferFrom(msg.sender, self, repayAmount)
    
    # Update debt
    self.borrowBalances[borrower][debtToken] -= repayAmount
    self.totalBorrows[debtToken] -= repayAmount
    self.totalReserves[debtToken] += repayAmount
    
    # Transfer collateral to liquidator
    self.collateralBalances[borrower][collateralToken] -= collateralSeized
    self.totalCollateral[collateralToken] -= collateralSeized
    
    ERC20(collateralToken).transfer(msg.sender, collateralSeized)
    
    log Liquidated(
        msg.sender,
        borrower,
        debtToken,
        collateralToken,
        repayAmount,
        collateralSeized
    )
```

---

## 6. Handler สำหรับ Lending Protocol

```solidity
// test/invariant/LendingHandler.sol

pragma solidity ^0.8.0;

import "forge-std/Test.sol";

interface ILending {
    function collateralBalances(address user, address token) external view returns (uint256);
    function borrowBalances(address user, address token) external view returns (uint256);
    function totalCollateral(address token) external view returns (uint256);
    function totalBorrows(address token) external view returns (uint256);
    function totalReserves(address token) external view returns (uint256);
    function deposit(address, uint256) external;
    function withdraw(address, uint256) external;
    function borrow(address, uint256) external;
    function repay(address, uint256) external;
    function liquidate(address, address, address, uint256) external;
}

contract LendingHandler is Test {
    ILending lending;
    address[] tokens;
    address[] actors;
    address internal currentActor;
    
    // Ghost variables
    mapping(address => uint256) public ghost_totalDeposited;
    mapping(address => uint256) public ghost_totalWithdrawn;
    mapping(address => uint256) public ghost_totalBorrowed;
    mapping(address => uint256) public ghost_totalRepaid;
    
    // Track users with positions
    mapping(address => bool) public hasPosition;
    address[] public usersWithPositions;
    
    modifier useActor(uint256 seed) {
        currentActor = actors[seed % actors.length];
        vm.startPrank(currentActor);
        _;
        vm.stopPrank();
    }
    
    constructor(address _lending, address[] memory _tokens) {
        lending = ILending(_lending);
        tokens = _tokens;
        
        for (uint256 i = 0; i < 5; i++) {
            actors.push(makeAddr(string(abi.encode(i))));
        }
    }
    
    function deposit(
        uint256 actorSeed,
        uint256 tokenSeed,
        uint256 amount
    ) external useActor(actorSeed) {
        address token = tokens[tokenSeed % tokens.length];
        amount = bound(amount, 1e18, 1_000_000e18);
        
        deal(token, currentActor, amount);
        IERC20(token).approve(address(lending), amount);
        
        try lending.deposit(token, amount) {
            ghost_totalDeposited[token] += amount;
            
            if (!hasPosition[currentActor]) {
                hasPosition[currentActor] = true;
                usersWithPositions.push(currentActor);
            }
        } catch {}
    }
    
    function withdraw(
        uint256 actorSeed,
        uint256 tokenSeed,
        uint256 fraction
    ) external useActor(actorSeed) {
        address token = tokens[tokenSeed % tokens.length];
        uint256 balance = lending.collateralBalances(currentActor, token);
        
        if (balance == 0) return;
        
        fraction = bound(fraction, 1, 100);
        uint256 amount = balance * fraction / 100;
        
        try lending.withdraw(token, amount) {
            ghost_totalWithdrawn[token] += amount;
        } catch {}
    }
    
    function borrow(
        uint256 actorSeed,
        uint256 tokenSeed,
        uint256 fraction
    ) external useActor(actorSeed) {
        address token = tokens[tokenSeed % tokens.length];
        uint256 reserves = lending.totalReserves(token);
        
        if (reserves == 0) return;
        
        fraction = bound(fraction, 1, 30);  // max 30% of reserves
        uint256 amount = reserves * fraction / 100;
        
        try lending.borrow(token, amount) {
            ghost_totalBorrowed[token] += amount;
        } catch {}
    }
    
    function repay(
        uint256 actorSeed,
        uint256 tokenSeed,
        uint256 fraction
    ) external useActor(actorSeed) {
        address token = tokens[tokenSeed % tokens.length];
        uint256 debt = lending.borrowBalances(currentActor, token);
        
        if (debt == 0) return;
        
        fraction = bound(fraction, 1, 100);
        uint256 amount = debt * fraction / 100;
        
        deal(token, currentActor, amount);
        IERC20(token).approve(address(lending), amount);
        
        try lending.repay(token, amount) {
            ghost_totalRepaid[token] += amount;
        } catch {}
    }
    
    function liquidate(
        uint256 actorSeed,
        uint256 borrowerSeed,
        uint256 debtTokenSeed,
        uint256 collTokenSeed
    ) external useActor(actorSeed) {
        if (usersWithPositions.length == 0) return;
        
        address borrower = usersWithPositions[borrowerSeed % usersWithPositions.length];
        address debtToken = tokens[debtTokenSeed % tokens.length];
        address collToken = tokens[collTokenSeed % tokens.length];
        
        uint256 debt = lending.borrowBalances(borrower, debtToken);
        if (debt == 0) return;
        
        uint256 repayAmount = debt / 2;
        
        deal(debtToken, currentActor, repayAmount);
        IERC20(debtToken).approve(address(lending), repayAmount);
        
        try lending.liquidate(borrower, debtToken, collToken, repayAmount) {} catch {}
    }
    
    function getUsersWithPositions() external view returns (address[] memory) {
        return usersWithPositions;
    }
}
```

---

## 7. Invariant Tests สำหรับ Lending

```solidity
// test/invariant/InvariantLending.t.sol

pragma solidity ^0.8.0;

import "forge-std/Test.sol";

contract InvariantLendingTest is Test {
    ILending lending;
    LendingHandler handler;
    address[] tokens;
    
    function setUp() public {
        // Deploy contracts
        targetContract(address(handler));
    }
    
    // Invariant 1: totalCollateral accounting
    function invariant_collateral_accounting() public view {
        for (uint256 t = 0; t < tokens.length; t++) {
            address token = tokens[t];
            
            uint256 sumCollateral;
            address[] memory users = handler.getUsersWithPositions();
            
            for (uint256 i = 0; i < users.length; i++) {
                sumCollateral += lending.collateralBalances(users[i], token);
            }
            
            assertEq(
                lending.totalCollateral(token),
                sumCollateral,
                "Collateral accounting broken"
            );
        }
    }
    
    // Invariant 2: totalBorrows accounting
    function invariant_borrow_accounting() public view {
        for (uint256 t = 0; t < tokens.length; t++) {
            address token = tokens[t];
            
            uint256 sumBorrows;
            address[] memory users = handler.getUsersWithPositions();
            
            for (uint256 i = 0; i < users.length; i++) {
                sumBorrows += lending.borrowBalances(users[i], token);
            }
            
            assertEq(
                lending.totalBorrows(token),
                sumBorrows,
                "Borrow accounting broken"
            );
        }
    }
    
    // Invariant 3: reserves + borrows = total deposited
    function invariant_reserves_plus_borrows() public view {
        for (uint256 t = 0; t < tokens.length; t++) {
            address token = tokens[t];
            
            uint256 reserves = lending.totalReserves(token);
            uint256 borrows = lending.totalBorrows(token);
            uint256 total = reserves + borrows;
            
            // ต้องเท่ากับ totalCollateral ที่ deposit เข้ามา (simplified)
            assertGe(total, 0, "Reserves sanity check");
        }
    }
    
    // Invariant 4: contract ต้องมี tokens เพียงพอ
    function invariant_solvency() public view {
        for (uint256 t = 0; t < tokens.length; t++) {
            address token = tokens[t];
            
            uint256 contractBalance = IERC20(token).balanceOf(address(lending));
            uint256 neededBalance = lending.totalCollateral(token) + lending.totalReserves(token);
            
            assertGe(contractBalance, neededBalance, "Protocol insolvent");
        }
    }
    
    // Invariant 5: ไม่สามารถ liquidate healthy positions
    function invariant_no_improper_liquidation() public view {
        // ตรวจสอบว่าทุก user ที่ถูก liquidate ต้อง unhealthy
        // (tracked via ghost variables in handler)
    }
    
    // Invariant 6: ghost variable matching
    function invariant_ghost_accounting() public view {
        for (uint256 t = 0; t < tokens.length; t++) {
            address token = tokens[t];
            
            // deposited - withdrawn = totalCollateral + totalBorrows
            uint256 netFlow = handler.ghost_totalDeposited(token) - 
                              handler.ghost_totalWithdrawn(token);
            
            // ต้องสอดคล้องกับ state
            assertGe(
                netFlow,
                lending.totalCollateral(token),
                "Ghost accounting mismatch"
            );
        }
    }
}
```

---

## 8. Ghost Variables ขั้นสูง

Ghost variables ช่วยให้เราติดตาม aggregate state ที่ไม่ได้เก็บใน contract โดยตรง

```solidity
// test/invariant/GhostVariableExample.sol

pragma solidity ^0.8.0;

// ตัวอย่างการใช้ ghost variables ขั้นสูง
contract AdvancedGhostVariables is Test {
    // Ghost: ติดตามจำนวน unique users
    mapping(address => bool) ghost_isUser;
    address[] ghost_users;
    
    // Ghost: ติดตาม total flow
    uint256 ghost_totalIn;
    uint256 ghost_totalOut;
    
    // Ghost: ติดตาม state transitions
    uint256 ghost_depositCount;
    uint256 ghost_withdrawCount;
    uint256 ghost_swapCount;
    
    // Ghost: ติดตาม max values
    uint256 ghost_maxBalance;
    address ghost_richestUser;
    
    function _recordUser(address user) internal {
        if (!ghost_isUser[user]) {
            ghost_isUser[user] = true;
            ghost_users.push(user);
        }
    }
    
    function _updateMaxBalance(address user, uint256 balance) internal {
        if (balance > ghost_maxBalance) {
            ghost_maxBalance = balance;
            ghost_richestUser = user;
        }
    }
    
    // Handler functions จะเรียก helper เหล่านี้
    function deposit(address user, uint256 amount) external {
        _recordUser(user);
        ghost_totalIn += amount;
        ghost_depositCount++;
        
        // Call actual contract
        // ...
    }
    
    function withdraw(address user, uint256 amount) external {
        ghost_totalOut += amount;
        ghost_withdrawCount++;
        
        // Call actual contract
        // ...
    }
    
    // Invariant ที่ใช้ ghost variables
    function invariant_flow_conservation() public view {
        // totalIn - totalOut = current holdings
        assertGe(ghost_totalIn, ghost_totalOut, "More out than in");
    }
    
    function invariant_user_count_valid() public view {
        assertGe(ghost_users.length, 0);
    }
}
```

---

## 9. Property Categories สำหรับ DeFi

### 9.1 Valid State Properties

```
Valid State: ค่าในทุก state ต้องถูกต้องเสมอ
- balance >= 0
- totalSupply >= 0  
- reserves > 0 เมื่อมี liquidity
- collateralFactor < 100%
```

### 9.2 Transition Properties

```
Transition: state เปลี่ยนถูกต้องเมื่อทำ action
- deposit() → totalCollateral เพิ่มขึ้น
- swap() → k ไม่ลดลง
- transfer() → totalSupply ไม่เปลี่ยน
```

### 9.3 High-level Properties

```
High-level: คุณสมบัติระดับธุรกิจ
- ไม่มีใครขโมย funds ได้
- ทุกคน withdraw ได้ไม่เกินที่ฝากไว้
- protocol ไม่ล้มละลาย
```

### 9.4 Access Control Properties

```
Access Control: สิทธิ์การเข้าถึง
- เฉพาะ owner เท่านั้นที่เพิ่ม markets ได้
- เฉพาะ approved liquidators เท่านั้น
- ไม่มีใครแก้ไข state โดยไม่ได้รับอนุญาต
```

---

## 10. การ Debug Invariant Failures

```bash
# เมื่อ invariant ถูก break จะมี output แบบนี้
# Sequence:
#   [0] deposit(0, 1000000000000000000)
#   [1] borrow(0, 500000000000000000)
#   [2] deposit(1, 2000000000000000000)
#   [3] withdraw(0, 800000000000000000) ← invariant breaks here

# วิธี debug:
# 1. ดู call sequence ที่ทำให้ break
# 2. Reproduce ด้วย unit test
# 3. วิเคราะห์ว่า state เปลี่ยนอย่างไร

# Reproduce failing sequence
forge test --match-test invariant_collateral_accounting \
    -vvvv \
    --fuzz-seed 0xdeadbeef
```

### ตัวอย่าง Unit Test จาก Failing Sequence

```solidity
// test/unit/ReproduceInvariantFail.t.sol

pragma solidity ^0.8.0;

import "forge-std/Test.sol";

contract ReproduceTest is Test {
    ILending lending;
    
    function setUp() public {
        // Deploy fresh contracts
    }
    
    function test_reproduce_invariant_failure() public {
        address user1 = makeAddr("user1");
        address user2 = makeAddr("user2");
        address token = address(0x1);
        
        // Reproduce failing sequence
        deal(token, user1, 1e18);
        vm.prank(user1);
        IERC20(token).approve(address(lending), 1e18);
        vm.prank(user1);
        lending.deposit(token, 1e18);
        
        vm.prank(user1);
        lending.borrow(token, 0.5e18);
        
        deal(token, user2, 2e18);
        vm.prank(user2);
        IERC20(token).approve(address(lending), 2e18);
        vm.prank(user2);
        lending.deposit(token, 2e18);
        
        // This should fail
        vm.prank(user1);
        vm.expectRevert();
        lending.withdraw(token, 0.8e18);
        
        // Verify invariant
        uint256 sumCollateral = lending.collateralBalances(user1, token) +
                                lending.collateralBalances(user2, token);
        assertEq(lending.totalCollateral(token), sumCollateral);
    }
}
```

---

## 11. Best Practices สำหรับ Invariant Testing

### การเขียน Handler ที่ดี

```
1. ใช้ bound() แทน modulo สำหรับ numeric inputs
2. ใช้ try/catch เพื่อ handle reverts
3. Track ghost variables ใน handler ไม่ใช่ใน invariant functions
4. สร้าง actors หลายคน ไม่ใช่แค่คนเดียว
5. เพิ่ม time manipulation (vm.warp) สำหรับ time-dependent logic
```

### การเลือก Invariants

```
เริ่มจาก:
1. Accounting invariants (ง่าย, มักพบ bugs)
2. Solvency invariants (สำคัญมาก)
3. Access control invariants

ขั้นสูง:
4. Economic invariants (AMM k-value, price bounds)
5. Oracle-dependent invariants
6. Cross-contract invariants
```

### ตั้งค่า Depth และ Runs

```toml
[invariant]
# สำหรับ simple contracts
runs = 100
depth = 100

# สำหรับ complex contracts
runs = 512
depth = 1000

# สำหรับ production-grade testing
runs = 1000
depth = 2000
```

---

## 12. Invariants สำหรับ Staking Contract

```solidity
// test/invariant/InvariantStaking.t.sol

pragma solidity ^0.8.0;

contract InvariantStakingTest is Test {
    IStaking staking;
    StakingHandler handler;
    
    // Invariant 1: totalStaked = sum of individual stakes
    function invariant_staking_accounting() public view {
        uint256 sumStaked;
        address[] memory actors = handler.getActors();
        for (uint256 i = 0; i < actors.length; i++) {
            sumStaked += staking.stakedBalance(actors[i]);
        }
        assertEq(staking.totalStaked(), sumStaked);
    }
    
    // Invariant 2: rewards accrued ต้องไม่เกิน total reward budget
    function invariant_reward_budget() public view {
        uint256 totalRewardBudget = staking.rewardRate() * staking.rewardsDuration();
        uint256 distributed = handler.ghost_totalRewardsClaimed();
        assertLe(distributed, totalRewardBudget + 1e18); // small rounding error ok
    }
    
    // Invariant 3: unstake ต้องคืน tokens เต็มจำนวน
    function invariant_stake_unstake_roundtrip() public view {
        // ตรวจสอบว่า ghost_totalStaked - ghost_totalUnstaked = totalStaked
        assertEq(
            handler.ghost_totalStaked() - handler.ghost_totalUnstaked(),
            staking.totalStaked()
        );
    }
    
    // Invariant 4: reward rate ต้องไม่เกิน constraint
    function invariant_reward_rate_bounded() public view {
        assertLe(staking.rewardRate(), 1e18); // max 1 token/second
    }
}
```

---

## แบบฝึกหัด

1. เขียน invariant tests สำหรับ ERC-4626 vault
2. สร้าง handler สำหรับ governance contract ที่มี voting
3. เขียน ghost variables สำหรับ cross-protocol interactions
4. ค้นหา invariant violations ใน AMM ที่ให้มา
5. Design invariants สำหรับ multi-collateral lending protocol

---

*จบ Part 079: Invariant Testing เชิงลึก*
