# Part 078: Advanced Fuzzing สำหรับ Smart Contracts

## สารบัญ
1. Fuzzing คืออะไรและทำไมถึงสำคัญ
2. Echidna Fuzzer - ติดตั้งและใช้งาน
3. Property-based Fuzzing
4. Stateful Fuzzing
5. Foundry Invariant Testing
6. Fuzz Target Contracts
7. Advanced Echidna Configurations
8. การวิเคราะห์ผลลัพธ์

---

## 1. Fuzzing คืออะไร?

Fuzzing คือการทดสอบโดยการสุ่ม inputs จำนวนมากเพื่อหา bugs ที่ไม่คาดคิด

### ประเภทของ Fuzzing

1. **Black-box Fuzzing** - ไม่รู้โครงสร้างภายใน ส่ง random inputs
2. **Grey-box Fuzzing** - ใช้ code coverage เพื่อ guide การสุ่ม
3. **White-box Fuzzing** - วิเคราะห์โค้ดเพื่อสร้าง inputs ที่ cover ทุก paths
4. **Property-based Fuzzing** - กำหนด properties แล้วหา inputs ที่ทำลาย properties
5. **Stateful Fuzzing** - ทดสอบ sequence ของ transactions

### เครื่องมือ Fuzzing สำหรับ Smart Contracts

- **Echidna** - Haskell-based fuzzer จาก Trail of Bits
- **Foundry Fuzz** - Built-in fuzzing ใน Forge
- **Medusa** - Go-based fuzzer ที่ compatible กับ Echidna
- **Harvey** - Greybox fuzzer สำหรับ EVM
- **MythX** - Cloud-based fuzzer

---

## 2. Echidna - การติดตั้งและตั้งค่า

```bash
# ติดตั้ง Echidna ผ่าน Nix
nix-env -i echidna

# หรือผ่าน Docker
docker pull trailofbits/echidna

# ตรวจสอบ version
echidna --version
```

### โครงสร้าง Project

```
project/
├── contracts/
│   ├── Token.vy
│   └── AMM.vy
├── tests/
│   ├── echidna/
│   │   ├── EchidnaToken.sol
│   │   └── EchidnaAMM.sol
│   └── foundry/
│       ├── InvariantToken.t.sol
│       └── InvariantAMM.t.sol
├── echidna.yaml
└── foundry.toml
```

### echidna.yaml Configuration

```yaml
# echidna.yaml - Echidna configuration

# จำนวน transactions ต่อ test run
testLimit: 50000

# จำนวน workers (parallel)
workers: 4

# Corpus directory สำหรับเก็บ test cases
corpusDir: "echidna-corpus"

# Mode: property (เน้น invariants) หรือ assertion (เน้น assert statements)
testMode: "property"

# Coverage-guided fuzzing
coverage: true

# Timeout ต่อ test (วินาที)
timeout: 300

# Initial balance สำหรับ test accounts
balanceAddr: "0xffffffffffffffffffffffffffffffffffffffff"

# Gas limit
gasLimit: 12500000

# ตั้งค่า contract ที่จะ deploy ก่อน (dependencies)
deployContracts:
  - ["0x1234...", "MockERC20"]

# Network fork
rpc: "https://mainnet.infura.io/v3/YOUR_KEY"
blockNumber: 18000000
```

---

## 3. Contract หลักสำหรับ Fuzzing

```vyper
# @version 0.4.0
# @title Fuzz Target: Token with Staking
# @notice Contract ที่ซับซ้อนสำหรับ fuzzing

from vyper.interfaces import ERC20

event Transfer:
    sender: indexed(address)
    receiver: indexed(address)
    value: uint256

event Staked:
    staker: indexed(address)
    amount: uint256

event Unstaked:
    staker: indexed(address)
    amount: uint256

event RewardClaimed:
    staker: indexed(address)
    reward: uint256

# State variables
name: public(String[64])
symbol: public(String[32])
decimals: public(uint8)
totalSupply: public(uint256)
balanceOf: public(HashMap[address, uint256])

# Staking state
stakedBalance: public(HashMap[address, uint256])
totalStaked: public(uint256)
rewardPerTokenStored: public(uint256)
userRewardPerTokenPaid: public(HashMap[address, uint256])
rewards: public(HashMap[address, uint256])
rewardRate: public(uint256)
lastUpdateTime: public(uint256)
rewardsDuration: public(uint256)
periodFinish: public(uint256)
owner: public(address)

PRECISION: constant(uint256) = 10**18

@deploy
def __init__():
    self.name = "Fuzz Token"
    self.symbol = "FUZZ"
    self.decimals = 18
    self.owner = msg.sender
    self.rewardsDuration = 7 * 24 * 3600  # 7 days

@internal
def _updateReward(account: address):
    """อัพเดต reward state"""
    self.rewardPerTokenStored = self._rewardPerToken()
    self.lastUpdateTime = self._lastTimeRewardApplicable()
    
    if account != empty(address):
        self.rewards[account] = self._earned(account)
        self.userRewardPerTokenPaid[account] = self.rewardPerTokenStored

@internal
@view
def _lastTimeRewardApplicable() -> uint256:
    return min(block.timestamp, self.periodFinish)

@internal
@view
def _rewardPerToken() -> uint256:
    if self.totalStaked == 0:
        return self.rewardPerTokenStored
    
    time_delta: uint256 = self._lastTimeRewardApplicable() - self.lastUpdateTime
    return self.rewardPerTokenStored + (
        time_delta * self.rewardRate * PRECISION / self.totalStaked
    )

@internal
@view
def _earned(account: address) -> uint256:
    return (
        self.stakedBalance[account] * 
        (self._rewardPerToken() - self.userRewardPerTokenPaid[account]) / 
        PRECISION + 
        self.rewards[account]
    )

@external
def transfer(to: address, amount: uint256) -> bool:
    assert to != empty(address), "Transfer to zero"
    assert self.balanceOf[msg.sender] >= amount, "Insufficient balance"
    
    self.balanceOf[msg.sender] -= amount
    self.balanceOf[to] += amount
    
    log Transfer(msg.sender, to, amount)
    return True

@external
def mint(to: address, amount: uint256):
    assert msg.sender == self.owner, "Not owner"
    self.totalSupply += amount
    self.balanceOf[to] += amount
    log Transfer(empty(address), to, amount)

@external
def stake(amount: uint256):
    """Stake tokens"""
    assert amount > 0, "Cannot stake 0"
    assert self.balanceOf[msg.sender] >= amount, "Insufficient balance"
    
    self._updateReward(msg.sender)
    
    self.balanceOf[msg.sender] -= amount
    self.stakedBalance[msg.sender] += amount
    self.totalStaked += amount
    
    log Staked(msg.sender, amount)

@external
def unstake(amount: uint256):
    """Unstake tokens"""
    assert amount > 0, "Cannot unstake 0"
    assert self.stakedBalance[msg.sender] >= amount, "Insufficient staked balance"
    
    self._updateReward(msg.sender)
    
    self.stakedBalance[msg.sender] -= amount
    self.totalStaked -= amount
    self.balanceOf[msg.sender] += amount
    
    log Unstaked(msg.sender, amount)

@external
def claimReward():
    """รับ rewards"""
    self._updateReward(msg.sender)
    
    reward: uint256 = self.rewards[msg.sender]
    if reward > 0:
        self.rewards[msg.sender] = 0
        self.balanceOf[msg.sender] += reward
        self.totalSupply += reward
        
        log RewardClaimed(msg.sender, reward)

@external
def notifyRewardAmount(reward: uint256):
    """เพิ่ม rewards (owner only)"""
    assert msg.sender == self.owner, "Not owner"
    
    self._updateReward(empty(address))
    
    if block.timestamp >= self.periodFinish:
        self.rewardRate = reward / self.rewardsDuration
    else:
        remaining: uint256 = self.periodFinish - block.timestamp
        leftover: uint256 = remaining * self.rewardRate
        self.rewardRate = (reward + leftover) / self.rewardsDuration
    
    self.lastUpdateTime = block.timestamp
    self.periodFinish = block.timestamp + self.rewardsDuration
    
    # mint reward tokens to contract
    self.totalSupply += reward
    self.balanceOf[self] += reward

@view
@external
def earned(account: address) -> uint256:
    return self._earned(account)

@view
@external
def rewardPerToken() -> uint256:
    return self._rewardPerToken()
```

---

## 4. Echidna Test Contract

```solidity
// tests/echidna/EchidnaStaking.sol
// Echidna properties สำหรับ Staking Contract

pragma solidity ^0.8.0;

interface IStakingToken {
    function balanceOf(address) external view returns (uint256);
    function totalSupply() external view returns (uint256);
    function stakedBalance(address) external view returns (uint256);
    function totalStaked() external view returns (uint256);
    function transfer(address, uint256) external returns (bool);
    function stake(uint256) external;
    function unstake(uint256) external;
    function claimReward() external;
    function earned(address) external view returns (uint256);
    function mint(address, uint256) external;
    function notifyRewardAmount(uint256) external;
}

contract EchidnaStakingTest {
    IStakingToken token;
    
    address constant USER1 = address(0x10000);
    address constant USER2 = address(0x20000);
    address constant USER3 = address(0x30000);
    
    constructor() {
        // Setup: deploy token and give users some tokens
        // token = new StakingToken();
        // token.mint(USER1, 1000 ether);
        // token.mint(USER2, 1000 ether);
        // token.mint(USER3, 1000 ether);
    }
    
    // Property 1: totalStaked ต้องเท่ากับผลรวม staked balances
    function echidna_total_staked_equals_sum() public view returns (bool) {
        uint256 sum = token.stakedBalance(USER1) + 
                      token.stakedBalance(USER2) + 
                      token.stakedBalance(USER3) +
                      token.stakedBalance(address(this));
        return token.totalStaked() == sum;
    }
    
    // Property 2: totalSupply ต้องไม่เกิน uint256 max
    function echidna_supply_not_overflow() public view returns (bool) {
        return token.totalSupply() < type(uint256).max;
    }
    
    // Property 3: ไม่มีใครมี balance มากกว่า totalSupply
    function echidna_balance_lte_supply() public view returns (bool) {
        return token.balanceOf(address(this)) <= token.totalSupply() &&
               token.balanceOf(USER1) <= token.totalSupply() &&
               token.balanceOf(USER2) <= token.totalSupply();
    }
    
    // Property 4: staked balance + liquid balance ต้อง <= initial balance (รวม rewards)
    function echidna_no_free_tokens() public view returns (bool) {
        // คำนวณ total holdings ของ USER1
        uint256 user1Total = token.balanceOf(USER1) + token.stakedBalance(USER1);
        // ต้องไม่มากกว่า initial amount + earned rewards
        return user1Total <= 1000 ether + token.earned(USER1);
    }
    
    // Helper functions สำหรับ Echidna ให้ call
    function stake_user1(uint256 amount) external {
        amount = amount % (token.balanceOf(USER1) + 1);
        vm_prank(USER1);
        if (amount > 0) token.stake(amount);
    }
    
    function unstake_user1(uint256 amount) external {
        amount = amount % (token.stakedBalance(USER1) + 1);
        vm_prank(USER1);
        if (amount > 0) token.unstake(amount);
    }
    
    function claim_user1() external {
        vm_prank(USER1);
        token.claimReward();
    }
    
    function transfer_user1_to_user2(uint256 amount) external {
        amount = amount % (token.balanceOf(USER1) + 1);
        vm_prank(USER1);
        if (amount > 0) token.transfer(USER2, amount);
    }
    
    // Cheat code simulation
    function vm_prank(address sender) internal {
        // This would use Hevm cheatcodes in real implementation
    }
}
```

---

## 5. Stateful Fuzzing ด้วย Echidna

```solidity
// tests/echidna/EchidnaAMMStateful.sol
// Stateful fuzzing สำหรับ AMM

pragma solidity ^0.8.0;

interface IAMM {
    function balances(uint256) external view returns (uint256);
    function totalLPSupply() external view returns (uint256);
    function lpBalances(address) external view returns (uint256);
    function addLiquidity(uint256, uint256, uint256) external returns (uint256);
    function removeLiquidity(uint256, uint256[] calldata) external returns (uint256[] memory);
    function swap(uint256, uint256, uint256) external returns (uint256);
    function getK() external view returns (uint256);
}

contract EchidnaAMMStateful {
    IAMM amm;
    
    // Track initial state
    uint256 initialK;
    bool isInitialized;
    
    // สถิติสำหรับ debug
    uint256 swapCount;
    uint256 addLiqCount;
    uint256 removeLiqCount;
    
    constructor() {
        // Deploy AMM
    }
    
    // ===== PROPERTIES =====
    
    // Property 1: k ต้องไม่ลดลงหลัง swap
    function echidna_k_never_decreases() public returns (bool) {
        if (!isInitialized) return true;
        uint256 currentK = amm.getK();
        return currentK >= initialK;
    }
    
    // Property 2: totalLPSupply > 0 เมื่อมี reserves
    function echidna_lp_supply_consistency() public view returns (bool) {
        uint256 r0 = amm.balances(0);
        uint256 r1 = amm.balances(1);
        uint256 lp = amm.totalLPSupply();
        
        if (r0 > 0 && r1 > 0) {
            return lp > 0;
        }
        return lp == 0;
    }
    
    // Property 3: ไม่มีใครสามารถ drain pool ได้ทั้งหมด
    function echidna_pool_not_drained() public view returns (bool) {
        if (amm.totalLPSupply() == 0) return true;
        return amm.balances(0) > 0 && amm.balances(1) > 0;
    }
    
    // Property 4: LP share invariant
    function echidna_lp_share_valid() public view returns (bool) {
        uint256 myLP = amm.lpBalances(address(this));
        uint256 totalLP = amm.totalLPSupply();
        return myLP <= totalLP;
    }
    
    // ===== ACTIONS =====
    
    function action_add_liquidity(uint256 a0, uint256 a1) external {
        a0 = (a0 % 1000 ether) + 1;
        a1 = (a1 % 1000 ether) + 1;
        
        try amm.addLiquidity(a0, a1, 0) {
            if (!isInitialized) {
                initialK = amm.getK();
                isInitialized = true;
            }
            addLiqCount++;
        } catch {}
    }
    
    function action_swap(uint256 soldId, uint256 amount) external {
        soldId = soldId % 2;
        amount = (amount % 100 ether) + 1;
        
        uint256 kBefore = amm.getK();
        
        try amm.swap(soldId, amount, 0) {
            swapCount++;
            // k ต้องไม่ลดลง
            assert(amm.getK() >= kBefore);
        } catch {}
    }
    
    function action_remove_liquidity(uint256 lpAmount) external {
        uint256 myLP = amm.lpBalances(address(this));
        if (myLP == 0) return;
        
        lpAmount = (lpAmount % myLP) + 1;
        
        uint256[] memory minAmounts = new uint256[](2);
        minAmounts[0] = 0;
        minAmounts[1] = 0;
        
        try amm.removeLiquidity(lpAmount, minAmounts) {
            removeLiqCount++;
        } catch {}
    }
}
```

---

## 6. Foundry Invariant Testing

```toml
# foundry.toml
[fuzz]
runs = 10000
max_test_rejects = 65536
seed = "0x1234"
dictionary_weight = 40
include_storage = true
include_push_bytes = true

[invariant]
runs = 256
depth = 500
fail_on_revert = false
call_override = false
dictionary_weight = 80
include_storage = true
include_push_bytes = true
shrink_sequence = true
```

```solidity
// tests/foundry/InvariantToken.t.sol
// Foundry Invariant Tests

pragma solidity ^0.8.0;

import "forge-std/Test.sol";
import "forge-std/InvariantTest.sol";

// Handler contract สำหรับ guided fuzzing
contract TokenHandler is Test {
    IStakingToken token;
    
    address[] public actors;
    address internal currentActor;
    
    // Ghost variables
    uint256 public ghost_totalMinted;
    uint256 public ghost_totalBurned;
    uint256 public ghost_totalStaked;
    uint256 public ghost_totalUnstaked;
    
    modifier useActor(uint256 actorIndexSeed) {
        currentActor = actors[actorIndexSeed % actors.length];
        vm.startPrank(currentActor);
        _;
        vm.stopPrank();
    }
    
    constructor(address _token) {
        token = IStakingToken(_token);
        
        // Create actors
        for (uint256 i = 0; i < 5; i++) {
            actors.push(address(uint160(uint256(keccak256(abi.encode(i))))));
        }
        
        // Give actors initial tokens
        for (uint256 i = 0; i < actors.length; i++) {
            vm.prank(address(this));
            token.mint(actors[i], 10000 ether);
            ghost_totalMinted += 10000 ether;
        }
    }
    
    // Handler: transfer
    function transfer(
        uint256 actorSeed,
        uint256 recipientSeed,
        uint256 amount
    ) external useActor(actorSeed) {
        address recipient = actors[recipientSeed % actors.length];
        amount = bound(amount, 0, token.balanceOf(currentActor));
        
        if (amount == 0) return;
        
        token.transfer(recipient, amount);
    }
    
    // Handler: stake
    function stake(
        uint256 actorSeed, 
        uint256 amount
    ) external useActor(actorSeed) {
        amount = bound(amount, 1, token.balanceOf(currentActor));
        
        if (token.balanceOf(currentActor) < amount) return;
        
        token.stake(amount);
        ghost_totalStaked += amount;
    }
    
    // Handler: unstake
    function unstake(
        uint256 actorSeed,
        uint256 amount
    ) external useActor(actorSeed) {
        uint256 staked = token.stakedBalance(currentActor);
        amount = bound(amount, 1, staked);
        
        if (staked < amount) return;
        
        token.unstake(amount);
        ghost_totalUnstaked += amount;
    }
    
    // Handler: claimReward
    function claimReward(uint256 actorSeed) external useActor(actorSeed) {
        uint256 earned = token.earned(currentActor);
        if (earned > 0) {
            token.claimReward();
            ghost_totalMinted += earned;
        }
    }
    
    function getActors() external view returns (address[] memory) {
        return actors;
    }
}

// Main invariant test contract
contract InvariantTokenTest is Test, InvariantTest {
    IStakingToken token;
    TokenHandler handler;
    
    function setUp() public {
        // Deploy token
        // token = new StakingToken();
        
        // Deploy handler
        handler = new TokenHandler(address(token));
        
        // Target the handler for fuzzing
        targetContract(address(handler));
        
        // Target specific functions
        bytes4[] memory selectors = new bytes4[](4);
        selectors[0] = TokenHandler.transfer.selector;
        selectors[1] = TokenHandler.stake.selector;
        selectors[2] = TokenHandler.unstake.selector;
        selectors[3] = TokenHandler.claimReward.selector;
        
        targetSelector(FuzzSelector({
            addr: address(handler),
            selectors: selectors
        }));
    }
    
    // Invariant 1: totalSupply = totalMinted - totalBurned
    function invariant_supply_accounting() public view {
        assertEq(
            token.totalSupply(),
            handler.ghost_totalMinted() - handler.ghost_totalBurned(),
            "Supply accounting broken"
        );
    }
    
    // Invariant 2: totalStaked = sum of all staked balances
    function invariant_staked_accounting() public view {
        address[] memory actors = handler.getActors();
        uint256 sumStaked;
        for (uint256 i = 0; i < actors.length; i++) {
            sumStaked += token.stakedBalance(actors[i]);
        }
        assertEq(token.totalStaked(), sumStaked, "Staked accounting broken");
    }
    
    // Invariant 3: ไม่มีใคร lose tokens โดยไม่ได้ตั้งใจ
    function invariant_no_token_loss() public view {
        address[] memory actors = handler.getActors();
        uint256 sumHoldings;
        for (uint256 i = 0; i < actors.length; i++) {
            sumHoldings += token.balanceOf(actors[i]);
            sumHoldings += token.stakedBalance(actors[i]);
            sumHoldings += token.earned(actors[i]);
        }
        // ต้องไม่มากกว่า totalSupply (รวมที่ยังไม่ได้ claim)
        assertLe(sumHoldings, token.totalSupply(), "Token loss detected");
    }
    
    // Invariant 4: ไม่มีใครมี balance < 0 (implicit in uint256)
    function invariant_non_negative_balances() public view {
        address[] memory actors = handler.getActors();
        for (uint256 i = 0; i < actors.length; i++) {
            // uint256 ไม่สามารถ < 0 ได้ แต่ตรวจสอบว่าไม่ overflow
            assertLe(token.balanceOf(actors[i]), type(uint256).max / 2);
        }
    }
}
```

---

## 7. Complex Fuzz Target: Flash Loan Contract

```vyper
# @version 0.4.0
# @title Flash Loan Provider
# @notice สำหรับทดสอบ flash loan invariants

from vyper.interfaces import ERC20

interface FlashLoanReceiver:
    def executeOperation(
        asset: address,
        amount: uint256,
        fee: uint256,
        initiator: address,
        params: Bytes[1024]
    ) -> bool: nonpayable

event FlashLoan:
    receiver: indexed(address)
    asset: indexed(address)
    amount: uint256
    fee: uint256

# State
token: public(address)
totalReserves: public(uint256)
flashLoanFeeRate: public(uint256)  # basis points
FLASH_LOAN_FEE_PRECISION: constant(uint256) = 10000

# Flash loan protection
_flashLoanInProgress: bool

@deploy
def __init__(_token: address, _feeRate: uint256):
    assert _token != empty(address)
    assert _feeRate <= 1000  # max 10%
    self.token = _token
    self.flashLoanFeeRate = _feeRate

@external
def deposit(amount: uint256):
    """ฝาก tokens เข้า pool"""
    assert amount > 0, "Invalid amount"
    ERC20(self.token).transferFrom(msg.sender, self, amount)
    self.totalReserves += amount

@external
def withdraw(amount: uint256):
    """ถอน tokens ออกจาก pool"""
    assert amount > 0, "Invalid amount"
    assert self.totalReserves >= amount, "Insufficient reserves"
    assert not self._flashLoanInProgress, "Flash loan in progress"
    
    self.totalReserves -= amount
    ERC20(self.token).transfer(msg.sender, amount)

@external
def flashLoan(
    receiver: address,
    amount: uint256,
    params: Bytes[1024]
) -> bool:
    """
    Flash loan
    Invariant: หลัง flash loan เสร็จ reserves ต้องเพิ่มขึ้น (จาก fee)
    """
    assert not self._flashLoanInProgress, "No re-entrancy"
    assert amount > 0, "Invalid amount"
    assert self.totalReserves >= amount, "Insufficient reserves"
    
    fee: uint256 = amount * self.flashLoanFeeRate / FLASH_LOAN_FEE_PRECISION
    
    reservesBefore: uint256 = self.totalReserves
    
    # Mark flash loan in progress
    self._flashLoanInProgress = True
    
    # Transfer amount to receiver
    ERC20(self.token).transfer(receiver, amount)
    
    # Call receiver's executeOperation
    result: bool = FlashLoanReceiver(receiver).executeOperation(
        self.token,
        amount,
        fee,
        msg.sender,
        params
    )
    
    assert result, "Flash loan failed"
    
    # Transfer amount + fee back
    ERC20(self.token).transferFrom(receiver, self, amount + fee)
    
    # Update reserves
    self.totalReserves = reservesBefore + fee
    
    # Clear flag
    self._flashLoanInProgress = False
    
    log FlashLoan(receiver, self.token, amount, fee)
    
    return True

@view
@external
def getBalance() -> uint256:
    return ERC20(self.token).balanceOf(self)
```

---

## 8. Echidna Properties สำหรับ Flash Loan

```solidity
// tests/echidna/EchidnaFlashLoan.sol

pragma solidity ^0.8.0;

interface IFlashLoanProvider {
    function totalReserves() external view returns (uint256);
    function deposit(uint256) external;
    function withdraw(uint256) external;
    function flashLoan(address, uint256, bytes calldata) external returns (bool);
}

contract MaliciousReceiver {
    IFlashLoanProvider provider;
    bool public tryReentrance;
    uint256 public stoleAmount;
    
    constructor(address _provider) {
        provider = IFlashLoanProvider(_provider);
    }
    
    function executeOperation(
        address asset,
        uint256 amount,
        uint256 fee,
        address initiator,
        bytes calldata params
    ) external returns (bool) {
        if (tryReentrance) {
            // พยายาม re-entrance flash loan
            try provider.flashLoan(address(this), amount / 2, "") {
                stoleAmount += amount / 2;
            } catch {}
        }
        
        // ชำระคืน amount + fee
        // IERC20(asset).approve(address(provider), amount + fee);
        return true;
    }
    
    function setTryReentrance(bool _try) external {
        tryReentrance = _try;
    }
}

contract EchidnaFlashLoanTest {
    IFlashLoanProvider provider;
    MaliciousReceiver maliciousReceiver;
    
    uint256 initialDeposit = 1000 ether;
    
    constructor() {
        // Deploy contracts
    }
    
    // Property 1: reserves ต้องไม่ลดลงหลัง flash loan
    function echidna_reserves_never_decrease() public view returns (bool) {
        return provider.totalReserves() >= initialDeposit;
    }
    
    // Property 2: malicious receiver ต้องไม่ขโมยได้
    function echidna_no_theft() public view returns (bool) {
        return maliciousReceiver.stoleAmount() == 0;
    }
    
    // Property 3: re-entrance ต้อง fail
    function echidna_no_reentrance() public returns (bool) {
        maliciousReceiver.setTryReentrance(true);
        
        uint256 reservesBefore = provider.totalReserves();
        
        try provider.flashLoan(
            address(maliciousReceiver),
            100 ether,
            ""
        ) returns (bool result) {
            // ถ้าสำเร็จ reserves ต้องเพิ่มขึ้น
            return provider.totalReserves() >= reservesBefore;
        } catch {
            // ถ้า revert reserves ต้องไม่เปลี่ยน
            return provider.totalReserves() == reservesBefore;
        }
    }
}
```

---

## 9. Medusa - Go-based Fuzzer

```yaml
# medusa.json - Medusa configuration

{
    "fuzzing": {
        "workers": 10,
        "workerResetLimit": 50,
        "timeout": 0,
        "testLimit": 0,
        "callSequenceLength": 100,
        "corpusDirectory": "corpus",
        "coverageEnabled": true,
        "targetContracts": ["EchidnaFlashLoanTest"],
        "targetContractsBalances": [],
        "constructorArgs": {},
        "deploymentOrder": [],
        "excludeContracts": [],
        "targetContractsMaintainOrder": false,
        "maxBlockNumberDelay": 60480,
        "maxBlockTimestampDelay": 604800,
        "blockNumberDelayMax": 60480,
        "blockTimestampDelayMax": 604800,
        "transactionGasLimit": 12500000,
        "testing": {
            "stopOnFailedTest": true,
            "stopOnFailedContractMatching": false,
            "stopOnNoTests": true,
            "testAllContracts": false,
            "traceAll": false,
            "assertionTesting": {
                "enabled": true,
                "testViewMethods": false,
                "panicCodeConfig": {
                    "failOnCompilerInsertedPanic": false,
                    "failOnAssertion": true,
                    "failOnArithmeticUnderflow": false,
                    "failOnDivideByZero": false,
                    "failOnEnumTypeConversionOutOfBounds": false,
                    "failOnIncorrectStorageAccess": false,
                    "failOnPopEmptyArray": false,
                    "failOnOutOfBoundsArrayAccess": false,
                    "failOnAllocateTooMuchMemory": false,
                    "failOnCallUninitializedVariable": false
                }
            },
            "propertyTesting": {
                "enabled": true,
                "testPrefixes": ["echidna_", "fuzz_", "property_"]
            },
            "optimizationTesting": {
                "enabled": false,
                "testPrefixes": ["optimize_"]
            }
        }
    }
}
```

---

## 10. Advanced Vyper Contract สำหรับ Fuzzing

```vyper
# @version 0.4.0
# @title Vault with Complex Logic
# @notice Vault contract สำหรับทดสอบ advanced fuzzing

from vyper.interfaces import ERC20

event Deposit:
    user: indexed(address)
    amount: uint256
    shares: uint256

event Withdraw:
    user: indexed(address)
    shares: uint256
    amount: uint256

# State
asset: public(address)
totalAssets: public(uint256)
totalShares: public(uint256)
shares: public(HashMap[address, uint256])
owner: public(address)

# Fees
depositFee: public(uint256)    # basis points
withdrawFee: public(uint256)   # basis points
performanceFee: public(uint256) # basis points
FEE_BASE: constant(uint256) = 10000

# Emergency
paused: public(bool)
emergencyWithdrawEnabled: public(bool)

@deploy
def __init__(
    _asset: address,
    _depositFee: uint256,
    _withdrawFee: uint256
):
    assert _asset != empty(address)
    assert _depositFee <= 500  # max 5%
    assert _withdrawFee <= 500  # max 5%
    
    self.asset = _asset
    self.depositFee = _depositFee
    self.withdrawFee = _withdrawFee
    self.owner = msg.sender

@internal
@view
def _sharesToAssets(shares_amount: uint256) -> uint256:
    """แปลง shares เป็น assets"""
    if self.totalShares == 0:
        return shares_amount
    return shares_amount * self.totalAssets / self.totalShares

@internal
@view
def _assetsToShares(assets_amount: uint256) -> uint256:
    """แปลง assets เป็น shares"""
    if self.totalShares == 0:
        return assets_amount
    return assets_amount * self.totalShares / self.totalAssets

@external
def deposit(amount: uint256) -> uint256:
    """ฝาก assets รับ shares"""
    assert not self.paused, "Paused"
    assert amount > 0, "Invalid amount"
    
    # คำนวณ fee
    fee: uint256 = amount * self.depositFee / FEE_BASE
    netAmount: uint256 = amount - fee
    
    # คำนวณ shares
    shares_minted: uint256 = self._assetsToShares(netAmount)
    assert shares_minted > 0, "Too small deposit"
    
    # Transfer assets
    ERC20(self.asset).transferFrom(msg.sender, self, amount)
    
    # Update state
    self.totalAssets += netAmount
    self.totalShares += shares_minted
    self.shares[msg.sender] += shares_minted
    
    log Deposit(msg.sender, amount, shares_minted)
    
    return shares_minted

@external
def withdraw(shares_amount: uint256) -> uint256:
    """เผา shares รับ assets"""
    assert not self.paused or self.emergencyWithdrawEnabled, "Paused"
    assert shares_amount > 0, "Invalid shares"
    assert self.shares[msg.sender] >= shares_amount, "Insufficient shares"
    
    # คำนวณ assets
    assets: uint256 = self._sharesToAssets(shares_amount)
    
    # คำนวณ fee (ยกเว้น emergency)
    fee: uint256 = 0
    if not self.emergencyWithdrawEnabled:
        fee = assets * self.withdrawFee / FEE_BASE
    
    netAssets: uint256 = assets - fee
    
    assert netAssets > 0, "Too small withdrawal"
    assert self.totalAssets >= assets, "Insufficient assets"
    
    # Update state
    self.shares[msg.sender] -= shares_amount
    self.totalShares -= shares_amount
    self.totalAssets -= assets
    
    # Transfer assets
    ERC20(self.asset).transfer(msg.sender, netAssets)
    
    log Withdraw(msg.sender, shares_amount, netAssets)
    
    return netAssets

@external
def reportProfit(profit: uint256):
    """รายงาน profit (owner only)"""
    assert msg.sender == self.owner, "Not owner"
    
    # 20% performance fee
    fee: uint256 = profit * self.performanceFee / FEE_BASE
    
    ERC20(self.asset).transferFrom(msg.sender, self, profit)
    self.totalAssets += profit - fee

@external
def emergencyPause():
    """หยุด contract ฉุกเฉิน"""
    assert msg.sender == self.owner, "Not owner"
    self.paused = True

@external
def enableEmergencyWithdraw():
    """เปิดให้ withdraw ฉุกเฉิน"""
    assert msg.sender == self.owner, "Not owner"
    assert self.paused, "Not paused"
    self.emergencyWithdrawEnabled = True

@view
@external
def pricePerShare() -> uint256:
    """ราคา 1 share เป็น assets"""
    if self.totalShares == 0:
        return 10**18
    return self.totalAssets * 10**18 / self.totalShares

@view
@external
def previewDeposit(amount: uint256) -> uint256:
    """ดูว่าจะได้ shares เท่าไหร่"""
    fee: uint256 = amount * self.depositFee / FEE_BASE
    netAmount: uint256 = amount - fee
    return self._assetsToShares(netAmount)

@view
@external
def previewWithdraw(shares_amount: uint256) -> uint256:
    """ดูว่าจะได้ assets เท่าไหร่"""
    assets: uint256 = self._sharesToAssets(shares_amount)
    fee: uint256 = assets * self.withdrawFee / FEE_BASE
    return assets - fee
```

---

## 11. Foundry Invariant Tests สำหรับ Vault

```solidity
// tests/foundry/InvariantVault.t.sol

pragma solidity ^0.8.0;

import "forge-std/Test.sol";

interface IVault {
    function deposit(uint256) external returns (uint256);
    function withdraw(uint256) external returns (uint256);
    function shares(address) external view returns (uint256);
    function totalShares() external view returns (uint256);
    function totalAssets() external view returns (uint256);
    function pricePerShare() external view returns (uint256);
    function previewDeposit(uint256) external view returns (uint256);
    function previewWithdraw(uint256) external view returns (uint256);
}

contract VaultHandler is Test {
    IVault vault;
    address token;
    
    address[] actors;
    
    // Ghost variables
    uint256 public ghost_depositedAssets;
    uint256 public ghost_withdrawnAssets;
    uint256 public ghost_mintedShares;
    uint256 public ghost_burnedShares;
    
    mapping(address => uint256) public actorDeposited;
    mapping(address => uint256) public actorWithdrawn;
    
    constructor(address _vault, address _token) {
        vault = IVault(_vault);
        token = _token;
        
        // Create actors
        for (uint256 i = 0; i < 3; i++) {
            address actor = address(uint160(i + 1));
            actors.push(actor);
        }
    }
    
    function deposit(uint256 actorSeed, uint256 amount) external {
        address actor = actors[actorSeed % actors.length];
        amount = bound(amount, 1 ether, 100 ether);
        
        // Give actor tokens
        deal(token, actor, amount);
        
        vm.startPrank(actor);
        IERC20(token).approve(address(vault), amount);
        
        uint256 sharesBefore = vault.shares(actor);
        uint256 sharesReceived = vault.deposit(amount);
        
        // Verify preview matches actual
        // assertApproxEqAbs(sharesReceived, vault.previewDeposit(amount), 1);
        
        vm.stopPrank();
        
        ghost_depositedAssets += amount;
        ghost_mintedShares += sharesReceived;
        actorDeposited[actor] += amount;
    }
    
    function withdraw(uint256 actorSeed, uint256 sharesFraction) external {
        address actor = actors[actorSeed % actors.length];
        uint256 actorShares = vault.shares(actor);
        
        if (actorShares == 0) return;
        
        sharesFraction = bound(sharesFraction, 1, 100);
        uint256 sharesToWithdraw = actorShares * sharesFraction / 100;
        
        vm.startPrank(actor);
        uint256 assetsReceived = vault.withdraw(sharesToWithdraw);
        vm.stopPrank();
        
        ghost_withdrawnAssets += assetsReceived;
        ghost_burnedShares += sharesToWithdraw;
        actorWithdrawn[actor] += assetsReceived;
    }
}

contract InvariantVaultTest is Test {
    IVault vault;
    VaultHandler handler;
    
    function setUp() public {
        // Deploy contracts
        // vault = new Vault(...);
        
        handler = new VaultHandler(address(vault), address(0));
        targetContract(address(handler));
    }
    
    // Invariant 1: pricePerShare ต้องไม่ลดลง (ยกเว้น loss event)
    function invariant_price_per_share_non_decreasing() public view {
        // Store initial price in setUp
        uint256 currentPrice = vault.pricePerShare();
        assertGe(currentPrice, 1 ether, "Price per share should not be below 1");
    }
    
    // Invariant 2: totalShares = sum of all actor shares
    function invariant_shares_accounting() public view {
        uint256 sumShares;
        address[] memory actors = handler.getActors();
        for (uint256 i = 0; i < actors.length; i++) {
            sumShares += vault.shares(actors[i]);
        }
        // (ไม่นับ shares ที่ contract อื่นถือ)
        // assertEq(vault.totalShares(), sumShares);
    }
    
    // Invariant 3: vault ต้องมี assets เพียงพอสำหรับ shares ทั้งหมด
    function invariant_vault_solvency() public view {
        if (vault.totalShares() == 0) return;
        assertGt(vault.totalAssets(), 0, "Vault must have assets if shares exist");
    }
    
    // Invariant 4: ไม่สามารถ deposit แล้วทันที withdraw ได้กำไร (เนื่องจาก fee)
    function invariant_no_instant_profit() public {
        // ไม่ทดสอบ invariant นี้โดยตรง แต่ verified ผ่าน handler
    }
}
```

---

## 12. การวิเคราะห์ Fuzzing Results

```bash
# ดู coverage report จาก Echidna
echidna . --contract EchidnaStakingTest \
    --config echidna.yaml \
    --coverage \
    --format text

# ดู corpus ที่ Echidna สร้าง
ls echidna-corpus/

# รัน Foundry invariant tests ด้วย verbose output
forge test --match-contract Invariant -vvvv

# ดู call sequence เมื่อ invariant ถูก break
forge test --match-contract InvariantVaultTest -vvvv --ffi

# Shrink failing sequence
forge test --match-contract InvariantVaultTest \
    --match-test invariant_vault_solvency \
    -vvvv
```

### ตัวอย่าง Output เมื่อพบ Bug

```
Failing tests:
[FAIL. Reason: Invariant violated: vault_solvency]
Sequence:
  [0] deposit(0, 100000000000000000000) (actor=0x0000...0001)
  [1] reportProfit(999999999999999999999) (by owner)
  [2] withdraw(0, 100) (actor=0x0000...0001)
  [3] deposit(1, 50000000000000000000) (actor=0x0000...0002)
  ...
  [47] withdraw(1, 100) → revert: Insufficient assets
  
Counter-example:
  totalAssets: 0
  totalShares: 1000000
  → vault_solvency violated!
```

---

## 13. Tips สำหรับ Fuzzing ที่มีประสิทธิภาพ

### การเขียน Properties ที่ดี

```
1. Invariants ที่ดีต้องเป็นจริง "เสมอ" ไม่ใช่แค่ "ส่วนใหญ่"
2. เริ่มจาก simple properties แล้วเพิ่มความซับซ้อน
3. ใช้ ghost variables เพื่อติดตาม state ที่ไม่ได้เก็บใน contract
4. ทดสอบ pre-conditions ด้วย (ไม่ใช่แค่ post-conditions)
5. ระวัง off-by-one errors ใน bounds
```

### Common Invariants สำหรับ DeFi

```
ERC-20:
- totalSupply = sum(balances)
- balanceOf(user) <= totalSupply
- transfer ไม่เปลี่ยน totalSupply

AMM:
- k = reserve0 * reserve1 ไม่ลดลงหลัง swap
- totalLPSupply > 0 ⟺ reserves > 0

Lending:
- totalCollateral = sum(collaterals)
- healthy position: collateral * CF >= debt
- liquidation ต้องทำให้ position healthy

Vault:
- pricePerShare ไม่ลดลง
- totalAssets / totalShares = pricePerShare
```

---

## แบบฝึกหัด

1. เขียน Echidna properties สำหรับ multi-token staking contract
2. สร้าง Foundry invariant tests สำหรับ lending protocol
3. ค้นหา bugs ในตัวอย่างข้างต้นโดยใช้ fuzzing
4. เขียน stateful fuzzing สำหรับ governance contract
5. ใช้ coverage-guided fuzzing เพื่อ maximize code coverage

---

*จบ Part 078: Advanced Fuzzing สำหรับ Smart Contracts*
