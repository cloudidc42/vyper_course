# Part 077: Formal Verification สำหรับ Smart Contracts

## สารบัญ
1. Formal Verification คืออะไร?
2. Certora Prover และการใช้งาน
3. CVL (Certora Verification Language)
4. Invariants และ Rules
5. การตรวจสอบ ERC-20 Contract
6. การตรวจสอบ AMM Contract
7. ตัวอย่างขั้นสูง

---

## 1. Formal Verification คืออะไร?

Formal Verification คือกระบวนการพิสูจน์ทางคณิตศาสตร์ว่า smart contract ทำงานถูกต้องตามข้อกำหนดที่กำหนดไว้ ต่างจาก unit testing ที่ทดสอบเฉพาะกรณีที่เราคิดถึง formal verification จะพิสูจน์ว่าสำหรับ **ทุกๆ input ที่เป็นไปได้** contract จะทำงานถูกต้อง

### ประเภทของ Formal Verification

1. **Model Checking** - ตรวจสอบทุก state ที่เป็นไปได้
2. **Theorem Proving** - พิสูจน์โดยใช้ logic ทางคณิตศาสตร์
3. **Static Analysis** - วิเคราะห์โค้ดโดยไม่ต้องรันจริง
4. **Symbolic Execution** - รันโค้ดด้วย symbolic values แทน concrete values

### เครื่องมือที่นิยมใช้

- **Certora Prover** - เครื่องมือหลักที่ใช้สำหรับ EVM contracts
- **Halmos** - Symbolic testing framework สำหรับ Foundry
- **hevm** - Symbolic execution engine จาก DappHub
- **Manticore** - Symbolic execution จาก Trail of Bits
- **Echidna** - Fuzzer ที่ใช้ร่วมกับ property testing

---

## 2. Certora Prover

Certora Prover ใช้ภาษา CVL (Certora Verification Language) เพื่อเขียน specifications

### การติดตั้ง

```bash
# ติดตั้ง Certora Prover
pip install certora-cli

# ตรวจสอบ version
certoraRun --version

# ต้องการ API key จาก Certora
export CERTORAKEY="your_api_key"
```

### โครงสร้าง CVL Specification

```javascript
// specs/ERC20.spec - CVL specification file

// กำหนด methods ที่จะตรวจสอบ
methods {
    function balanceOf(address) external returns (uint256) envfree;
    function totalSupply() external returns (uint256) envfree;
    function transfer(address, uint256) external returns (bool);
    function allowance(address, address) external returns (uint256) envfree;
    function approve(address, uint256) external returns (bool);
    function transferFrom(address, address, uint256) external returns (bool);
}

// Invariant: ผลรวมของ balances ทั้งหมด = totalSupply
invariant totalSupplyEqualsSum(address a, address b)
    a != b => balanceOf(a) + balanceOf(b) <= totalSupply()
    {
        preserved {
            require a != 0 && b != 0;
        }
    }

// Rule: transfer ต้องลด balance ของ sender
rule transferReducesSenderBalance(address to, uint256 amount) {
    env e;
    address sender = e.msg.sender;
    
    require sender != to;
    require balanceOf(sender) >= amount;
    
    uint256 senderBalanceBefore = balanceOf(sender);
    
    transfer(e, to, amount);
    
    uint256 senderBalanceAfter = balanceOf(sender);
    
    assert senderBalanceAfter == senderBalanceBefore - amount,
        "Transfer should reduce sender balance";
}
```

---

## 3. ERC-20 Contract สำหรับ Formal Verification

```vyper
# @version 0.4.0
# @title Verifiable ERC20
# @notice ERC-20 ที่ออกแบบมาสำหรับ formal verification

from vyper.interfaces import ERC20
implements: ERC20

# Events
event Transfer:
    sender: indexed(address)
    receiver: indexed(address)
    value: uint256

event Approval:
    owner: indexed(address)
    spender: indexed(address)
    value: uint256

# State variables
name: public(String[64])
symbol: public(String[32])
decimals: public(uint8)

totalSupply: public(uint256)
balanceOf: public(HashMap[address, uint256])
allowance: public(HashMap[address, HashMap[address, uint256]])

owner: public(address)

@deploy
def __init__(
    _name: String[64],
    _symbol: String[32],
    _decimals: uint8,
    _initial_supply: uint256
):
    self.name = _name
    self.symbol = _symbol
    self.decimals = _decimals
    self.owner = msg.sender
    
    # mint initial supply to deployer
    self.totalSupply = _initial_supply
    self.balanceOf[msg.sender] = _initial_supply
    
    log Transfer(empty(address), msg.sender, _initial_supply)

@external
def transfer(to: address, amount: uint256) -> bool:
    """
    @notice โอน tokens ไปยัง address อื่น
    @dev สำหรับ formal verification: เราพิสูจน์ว่า
         1. sender balance ลดลง amount
         2. receiver balance เพิ่มขึ้น amount
         3. totalSupply ไม่เปลี่ยน
    """
    assert to != empty(address), "ERC20: transfer to zero address"
    assert self.balanceOf[msg.sender] >= amount, "ERC20: insufficient balance"
    
    self.balanceOf[msg.sender] -= amount
    self.balanceOf[to] += amount
    
    log Transfer(msg.sender, to, amount)
    return True

@external
def transferFrom(sender: address, to: address, amount: uint256) -> bool:
    """
    @notice โอน tokens จาก address หนึ่งไปยังอีก address หนึ่ง
    @dev สำหรับ formal verification: เราพิสูจน์ว่า
         1. allowance ลดลง amount
         2. sender balance ลดลง amount
         3. receiver balance เพิ่มขึ้น amount
    """
    assert sender != empty(address), "ERC20: transfer from zero address"
    assert to != empty(address), "ERC20: transfer to zero address"
    assert self.balanceOf[sender] >= amount, "ERC20: insufficient balance"
    assert self.allowance[sender][msg.sender] >= amount, "ERC20: insufficient allowance"
    
    self.allowance[sender][msg.sender] -= amount
    self.balanceOf[sender] -= amount
    self.balanceOf[to] += amount
    
    log Transfer(sender, to, amount)
    return True

@external
def approve(spender: address, amount: uint256) -> bool:
    """
    @notice อนุมัติให้ spender ใช้ tokens
    """
    assert spender != empty(address), "ERC20: approve to zero address"
    
    self.allowance[msg.sender][spender] = amount
    
    log Approval(msg.sender, spender, amount)
    return True

@external
def mint(to: address, amount: uint256):
    """
    @notice สร้าง tokens ใหม่ (owner เท่านั้น)
    @dev สำหรับ formal verification: เราพิสูจน์ว่า
         totalSupply เพิ่มขึ้น amount
    """
    assert msg.sender == self.owner, "ERC20: caller is not owner"
    assert to != empty(address), "ERC20: mint to zero address"
    
    self.totalSupply += amount
    self.balanceOf[to] += amount
    
    log Transfer(empty(address), to, amount)

@external
def burn(amount: uint256):
    """
    @notice ทำลาย tokens
    @dev สำหรับ formal verification: เราพิสูจน์ว่า
         totalSupply ลดลง amount
    """
    assert self.balanceOf[msg.sender] >= amount, "ERC20: burn amount exceeds balance"
    
    self.balanceOf[msg.sender] -= amount
    self.totalSupply -= amount
    
    log Transfer(msg.sender, empty(address), amount)
```

---

## 4. CVL Specification สำหรับ ERC-20

```javascript
// specs/VerifiableERC20.spec

methods {
    // ประกาศ functions ทั้งหมด
    function balanceOf(address) external returns (uint256) envfree;
    function totalSupply() external returns (uint256) envfree;
    function transfer(address, uint256) external returns (bool);
    function transferFrom(address, address, uint256) external returns (bool);
    function approve(address, uint256) external returns (bool);
    function allowance(address, address) external returns (uint256) envfree;
    function mint(address, uint256) external;
    function burn(uint256) external;
}

// ===== INVARIANTS =====

// Invariant 1: totalSupply ต้องไม่เกิน max uint256
invariant totalSupplyBounded()
    totalSupply() <= max_uint256

// Invariant 2: สำหรับ 2 addresses ใดๆ ผลรวม balance ต้องไม่เกิน totalSupply
invariant solvency(address a, address b)
    a != b => balanceOf(a) + balanceOf(b) <= totalSupply()
    {
        preserved mint(address to, uint256 amount) with (env e) {
            require a != to || b != to;
        }
    }

// Invariant 3: zero address ต้องมี balance = 0
invariant zeroAddressBalance()
    balanceOf(0) == 0

// ===== RULES =====

// Rule 1: transfer ต้องทำงานถูกต้อง
rule transferCorrectness(address to, uint256 amount) {
    env e;
    address from = e.msg.sender;
    
    // Pre-conditions
    require from != 0 && to != 0;
    require from != to;
    
    uint256 fromBalBefore = balanceOf(from);
    uint256 toBalBefore = balanceOf(to);
    uint256 totalBefore = totalSupply();
    
    transfer(e, to, amount);
    
    // Post-conditions
    assert balanceOf(from) == fromBalBefore - amount,
        "Sender balance should decrease by amount";
    assert balanceOf(to) == toBalBefore + amount,
        "Receiver balance should increase by amount";
    assert totalSupply() == totalBefore,
        "Total supply should not change";
}

// Rule 2: transferFrom ต้องลด allowance
rule transferFromReducesAllowance(
    address from, 
    address to, 
    uint256 amount
) {
    env e;
    address spender = e.msg.sender;
    
    require from != 0 && to != 0 && spender != 0;
    
    uint256 allowanceBefore = allowance(from, spender);
    
    transferFrom(e, from, to, amount);
    
    assert allowance(from, spender) == allowanceBefore - amount,
        "Allowance should decrease by amount";
}

// Rule 3: approve ต้องตั้งค่า allowance ถูกต้อง
rule approveCorrectness(address spender, uint256 amount) {
    env e;
    address owner = e.msg.sender;
    
    require owner != 0 && spender != 0;
    
    approve(e, spender, amount);
    
    assert allowance(owner, spender) == amount,
        "Allowance should equal approved amount";
}

// Rule 4: mint ต้องเพิ่ม totalSupply
rule mintIncreasesTotalSupply(address to, uint256 amount) {
    env e;
    
    uint256 supplyBefore = totalSupply();
    uint256 balanceBefore = balanceOf(to);
    
    mint(e, to, amount);
    
    assert totalSupply() == supplyBefore + amount,
        "Total supply should increase by minted amount";
    assert balanceOf(to) == balanceBefore + amount,
        "Recipient balance should increase by minted amount";
}

// Rule 5: burn ต้องลด totalSupply
rule burnDecreasesTotalSupply(uint256 amount) {
    env e;
    address burner = e.msg.sender;
    
    uint256 supplyBefore = totalSupply();
    uint256 balanceBefore = balanceOf(burner);
    
    burn(e, amount);
    
    assert totalSupply() == supplyBefore - amount,
        "Total supply should decrease by burned amount";
    assert balanceOf(burner) == balanceBefore - amount,
        "Burner balance should decrease by burned amount";
}

// Rule 6: ไม่มีใครสามารถสร้าง tokens โดยไม่ได้รับอนุญาต
rule noUnauthorizedMinting(method f) {
    env e;
    uint256 supplyBefore = totalSupply();
    
    calldataarg args;
    f(e, args);
    
    uint256 supplyAfter = totalSupply();
    
    // totalSupply ควรเพิ่มได้เฉพาะจาก mint function เท่านั้น
    assert supplyAfter > supplyBefore => f.selector == sig:mint(address, uint256).selector,
        "Only mint can increase total supply";
}
```

---

## 5. AMM Contract สำหรับ Formal Verification

```vyper
# @version 0.4.0
# @title Verifiable AMM (Constant Product)
# @notice AMM ที่ออกแบบมาสำหรับ formal verification
# @dev ใช้ x * y = k formula

from vyper.interfaces import ERC20

# Events
event AddLiquidity:
    provider: indexed(address)
    token_amounts: uint256[2]
    lp_tokens_minted: uint256

event RemoveLiquidity:
    provider: indexed(address)
    token_amounts: uint256[2]
    lp_tokens_burned: uint256

event Swap:
    buyer: indexed(address)
    sold_id: indexed(uint256)
    tokens_sold: uint256
    bought_id: indexed(uint256)
    tokens_bought: uint256

# Constants
FEE_DENOMINATOR: constant(uint256) = 10000
FEE: constant(uint256) = 30  # 0.3% fee

# State variables
tokens: public(address[2])
balances: public(uint256[2])
totalLPSupply: public(uint256)
lpBalances: public(HashMap[address, uint256])

@deploy
def __init__(token0: address, token1: address):
    assert token0 != empty(address), "Invalid token0"
    assert token1 != empty(address), "Invalid token1"
    assert token0 != token1, "Identical tokens"
    
    self.tokens[0] = token0
    self.tokens[1] = token1

@internal
def _sqrt(x: uint256) -> uint256:
    """คำนวณ square root โดยใช้ Newton's method"""
    if x == 0:
        return 0
    
    z: uint256 = (x + 1) / 2
    y: uint256 = x
    
    for _: uint256 in range(256):
        if z >= y:
            break
        y = z
        z = (x / z + z) / 2
    
    return y

@external
def addLiquidity(
    amount0: uint256,
    amount1: uint256,
    min_lp: uint256
) -> uint256:
    """
    @notice เพิ่ม liquidity เข้า pool
    @dev สำหรับ formal verification:
         invariant: k = reserves[0] * reserves[1] ต้องเพิ่มขึ้นหรือคงที่
    """
    assert amount0 > 0 and amount1 > 0, "Invalid amounts"
    
    lp_minted: uint256 = 0
    
    if self.totalLPSupply == 0:
        # Initial liquidity
        lp_minted = self._sqrt(amount0 * amount1)
    else:
        # Proportional liquidity
        lp0: uint256 = amount0 * self.totalLPSupply / self.balances[0]
        lp1: uint256 = amount1 * self.totalLPSupply / self.balances[1]
        lp_minted = min(lp0, lp1)
    
    assert lp_minted >= min_lp, "Slippage too high"
    
    # Transfer tokens
    ERC20(self.tokens[0]).transferFrom(msg.sender, self, amount0)
    ERC20(self.tokens[1]).transferFrom(msg.sender, self, amount1)
    
    # Update state
    self.balances[0] += amount0
    self.balances[1] += amount1
    self.totalLPSupply += lp_minted
    self.lpBalances[msg.sender] += lp_minted
    
    log AddLiquidity(msg.sender, [amount0, amount1], lp_minted)
    
    return lp_minted

@external
def removeLiquidity(
    lp_amount: uint256,
    min_amounts: uint256[2]
) -> uint256[2]:
    """
    @notice ถอน liquidity ออกจาก pool
    @dev สำหรับ formal verification:
         LP holder ต้องได้รับ tokens ตามสัดส่วน
    """
    assert lp_amount > 0, "Invalid LP amount"
    assert self.lpBalances[msg.sender] >= lp_amount, "Insufficient LP balance"
    
    amount0: uint256 = lp_amount * self.balances[0] / self.totalLPSupply
    amount1: uint256 = lp_amount * self.balances[1] / self.totalLPSupply
    
    assert amount0 >= min_amounts[0], "Insufficient token0 received"
    assert amount1 >= min_amounts[1], "Insufficient token1 received"
    
    # Burn LP tokens
    self.lpBalances[msg.sender] -= lp_amount
    self.totalLPSupply -= lp_amount
    
    # Update reserves
    self.balances[0] -= amount0
    self.balances[1] -= amount1
    
    # Transfer tokens
    ERC20(self.tokens[0]).transfer(msg.sender, amount0)
    ERC20(self.tokens[1]).transfer(msg.sender, amount1)
    
    log RemoveLiquidity(msg.sender, [amount0, amount1], lp_amount)
    
    return [amount0, amount1]

@external
def swap(
    sold_id: uint256,
    amount_in: uint256,
    min_amount_out: uint256
) -> uint256:
    """
    @notice แลกเปลี่ยน tokens
    @dev สำหรับ formal verification:
         invariant: k ต้องเพิ่มขึ้น (เนื่องจาก fee)
    """
    assert sold_id < 2, "Invalid token ID"
    assert amount_in > 0, "Invalid amount"
    
    bought_id: uint256 = 1 - sold_id
    
    reserve_in: uint256 = self.balances[sold_id]
    reserve_out: uint256 = self.balances[bought_id]
    
    # Calculate amount out with fee
    amount_in_with_fee: uint256 = amount_in * (FEE_DENOMINATOR - FEE)
    amount_out: uint256 = (amount_in_with_fee * reserve_out) / (
        reserve_in * FEE_DENOMINATOR + amount_in_with_fee
    )
    
    assert amount_out >= min_amount_out, "Slippage too high"
    assert amount_out < reserve_out, "Insufficient liquidity"
    
    # Transfer tokens
    ERC20(self.tokens[sold_id]).transferFrom(msg.sender, self, amount_in)
    ERC20(self.tokens[bought_id]).transfer(msg.sender, amount_out)
    
    # Update reserves
    self.balances[sold_id] += amount_in
    self.balances[bought_id] -= amount_out
    
    log Swap(msg.sender, sold_id, amount_in, bought_id, amount_out)
    
    return amount_out

@view
@external
def getReserves() -> uint256[2]:
    return [self.balances[0], self.balances[1]]

@view
@external
def getK() -> uint256:
    """คำนวณ constant product k"""
    return self.balances[0] * self.balances[1]
```

---

## 6. CVL Specification สำหรับ AMM

```javascript
// specs/VerifiableAMM.spec

methods {
    function balances(uint256) external returns (uint256) envfree;
    function totalLPSupply() external returns (uint256) envfree;
    function lpBalances(address) external returns (uint256) envfree;
    function getK() external returns (uint256) envfree;
    function addLiquidity(uint256, uint256, uint256) external returns (uint256);
    function removeLiquidity(uint256, uint256[2]) external returns (uint256[2]);
    function swap(uint256, uint256, uint256) external returns (uint256);
}

// ===== GHOST VARIABLES =====
// Ghost variables ช่วยให้เราติดตาม state ระหว่าง transactions

ghost mathint sumOfLPBalances {
    init_state axiom sumOfLPBalances == 0;
}

hook Sstore lpBalances[KEY address a] uint256 newValue (uint256 oldValue) {
    sumOfLPBalances = sumOfLPBalances + newValue - oldValue;
}

// ===== INVARIANTS =====

// Invariant 1: ผลรวม LP balances = totalLPSupply
invariant lpSolvency()
    to_mathint(totalLPSupply()) == sumOfLPBalances

// Invariant 2: k ต้องมากกว่า 0 เมื่อมี liquidity
invariant positiveK()
    totalLPSupply() > 0 => balances(0) > 0 && balances(1) > 0

// ===== RULES =====

// Rule 1: swap ต้องเพิ่ม k (เนื่องจาก fee)
rule swapIncreasesK(uint256 sold_id, uint256 amount_in, uint256 min_out) {
    env e;
    
    uint256 kBefore = getK();
    
    require kBefore > 0; // มี liquidity อยู่แล้ว
    
    swap(e, sold_id, amount_in, min_out);
    
    uint256 kAfter = getK();
    
    assert kAfter >= kBefore,
        "K should not decrease after swap";
}

// Rule 2: addLiquidity ต้องเพิ่ม LP tokens
rule addLiquidityMintsLP(uint256 a0, uint256 a1, uint256 min_lp) {
    env e;
    
    uint256 lpBefore = lpBalances(e.msg.sender);
    uint256 totalBefore = totalLPSupply();
    
    uint256 lp_minted = addLiquidity(e, a0, a1, min_lp);
    
    assert lp_minted > 0, "Should mint LP tokens";
    assert lpBalances(e.msg.sender) == lpBefore + lp_minted,
        "LP balance should increase";
    assert totalLPSupply() == totalBefore + lp_minted,
        "Total LP supply should increase";
}

// Rule 3: removeLiquidity ต้องเผา LP tokens ถูกต้อง
rule removeLiquidityBurnsLP(uint256 lp_amount, uint256[2] min_amounts) {
    env e;
    
    uint256 lpBefore = lpBalances(e.msg.sender);
    uint256 totalBefore = totalLPSupply();
    
    require lpBefore >= lp_amount;
    require lp_amount > 0;
    
    removeLiquidity(e, lp_amount, min_amounts);
    
    assert lpBalances(e.msg.sender) == lpBefore - lp_amount,
        "LP balance should decrease";
    assert totalLPSupply() == totalBefore - lp_amount,
        "Total LP supply should decrease";
}

// Rule 4: ไม่มีใครได้ tokens มากกว่าที่ฝากไว้ (no free money)
rule noFreeTokens(method f) {
    env e;
    
    uint256 r0Before = balances(0);
    uint256 r1Before = balances(1);
    
    calldataarg args;
    f(e, args);
    
    uint256 r0After = balances(0);
    uint256 r1After = balances(1);
    
    // ถ้า reserve ลดลง ต้องมีการ remove liquidity หรือ swap
    assert (r0After < r0Before || r1After < r1Before) =>
        (f.selector == sig:removeLiquidity(uint256, uint256[2]).selector ||
         f.selector == sig:swap(uint256, uint256, uint256).selector),
        "Only authorized operations can reduce reserves";
}
```

---

## 7. การรัน Certora Prover

```bash
# certora.conf - configuration file

# รัน verification สำหรับ ERC-20
certoraRun \
    contracts/VerifiableERC20.vy:VerifiableERC20 \
    --verify VerifiableERC20:specs/VerifiableERC20.spec \
    --msg "ERC20 Formal Verification" \
    --compiler_map "VerifiableERC20=vyper" \
    --vyper_version 0.4.0

# รัน verification สำหรับ AMM
certoraRun \
    contracts/VerifiableAMM.vy:VerifiableAMM \
    contracts/MockERC20.vy:MockToken \
    --verify VerifiableAMM:specs/VerifiableAMM.spec \
    --msg "AMM Formal Verification" \
    --compiler_map "VerifiableAMM=vyper,MockToken=vyper" \
    --vyper_version 0.4.0
```

---

## 8. Halmos - Symbolic Testing ใน Foundry

Halmos เป็นเครื่องมือที่ใช้ symbolic execution ร่วมกับ Foundry

```python
# test/HalmosTest.py - Python-based symbolic tests

# ติดตั้ง
# pip install halmos

# รัน halmos
# halmos --contract HalmosERC20Test
```

```solidity
// test/HalmosERC20Test.sol - Solidity test สำหรับ Halmos
// ใช้เพื่อทดสอบ Vyper contract ผ่าน ABI

pragma solidity ^0.8.0;

import "forge-std/Test.sol";

interface IVerifiableERC20 {
    function transfer(address to, uint256 amount) external returns (bool);
    function balanceOf(address) external view returns (uint256);
    function totalSupply() external view returns (uint256);
    function approve(address spender, uint256 amount) external returns (bool);
    function allowance(address owner, address spender) external view returns (uint256);
    function transferFrom(address from, address to, uint256 amount) external returns (bool);
}

contract HalmosERC20Test is Test {
    IVerifiableERC20 token;
    
    function setUp() public {
        // Deploy Vyper contract
        // token = IVerifiableERC20(deployVyperContract());
    }
    
    // Symbolic test: transfer ต้องทำงานถูกต้องสำหรับทุก input
    function check_transfer(
        address sender,
        address receiver,
        uint256 amount
    ) public {
        vm.assume(sender != address(0));
        vm.assume(receiver != address(0));
        vm.assume(sender != receiver);
        
        uint256 senderBal = token.balanceOf(sender);
        uint256 receiverBal = token.balanceOf(receiver);
        
        vm.assume(senderBal >= amount);
        
        vm.prank(sender);
        bool success = token.transfer(receiver, amount);
        
        assert(success);
        assert(token.balanceOf(sender) == senderBal - amount);
        assert(token.balanceOf(receiver) == receiverBal + amount);
    }
    
    // Symbolic test: totalSupply ต้องไม่เปลี่ยนหลัง transfer
    function check_transfer_preserves_supply(
        address sender,
        address receiver,
        uint256 amount
    ) public {
        vm.assume(sender != address(0));
        vm.assume(receiver != address(0));
        
        uint256 supplyBefore = token.totalSupply();
        uint256 senderBal = token.balanceOf(sender);
        
        vm.assume(senderBal >= amount);
        
        vm.prank(sender);
        token.transfer(receiver, amount);
        
        assert(token.totalSupply() == supplyBefore);
    }
}
```

---

## 9. Mutation Testing เพื่อตรวจสอบ Specifications

Mutation testing ช่วยตรวจสอบว่า specifications ของเราแข็งแกร่งพอ

```vyper
# @version 0.4.0
# @title Mutant ERC20 - สำหรับ mutation testing
# @notice Version ที่มี bugs สำหรับทดสอบว่า specs จับได้

from vyper.interfaces import ERC20
implements: ERC20

# State
name: public(String[64])
symbol: public(String[32])
decimals: public(uint8)
totalSupply: public(uint256)
balanceOf: public(HashMap[address, uint256])
allowance: public(HashMap[address, HashMap[address, uint256]])

event Transfer:
    sender: indexed(address)
    receiver: indexed(address)
    value: uint256

event Approval:
    owner: indexed(address)
    spender: indexed(address)
    value: uint256

@deploy
def __init__(_name: String[64], _symbol: String[32]):
    self.name = _name
    self.symbol = _symbol
    self.decimals = 18

# MUTATION 1: ไม่ลด sender balance (bug!)
@external
def transfer_mutant1(to: address, amount: uint256) -> bool:
    # BUG: ไม่มีการลด balance ของ sender
    # self.balanceOf[msg.sender] -= amount  <-- missing!
    self.balanceOf[to] += amount
    log Transfer(msg.sender, to, amount)
    return True

# MUTATION 2: ลด balance แต่ไม่เพิ่ม (bug!)
@external
def transfer_mutant2(to: address, amount: uint256) -> bool:
    self.balanceOf[msg.sender] -= amount
    # BUG: ไม่เพิ่ม balance ของ receiver
    # self.balanceOf[to] += amount  <-- missing!
    log Transfer(msg.sender, to, amount)
    return True

# MUTATION 3: ลด totalSupply หลัง transfer (bug!)
@external
def transfer_mutant3(to: address, amount: uint256) -> bool:
    self.balanceOf[msg.sender] -= amount
    self.balanceOf[to] += amount
    # BUG: ลด totalSupply โดยไม่ควร
    self.totalSupply -= amount
    log Transfer(msg.sender, to, amount)
    return True

# CORRECT version
@external
def transfer(to: address, amount: uint256) -> bool:
    assert self.balanceOf[msg.sender] >= amount
    self.balanceOf[msg.sender] -= amount
    self.balanceOf[to] += amount
    log Transfer(msg.sender, to, amount)
    return True

@external
def approve(spender: address, amount: uint256) -> bool:
    self.allowance[msg.sender][spender] = amount
    log Approval(msg.sender, spender, amount)
    return True

@external
def transferFrom(from_addr: address, to: address, amount: uint256) -> bool:
    assert self.balanceOf[from_addr] >= amount
    assert self.allowance[from_addr][msg.sender] >= amount
    self.allowance[from_addr][msg.sender] -= amount
    self.balanceOf[from_addr] -= amount
    self.balanceOf[to] += amount
    log Transfer(from_addr, to, amount)
    return True
```

---

## 10. hevm - Symbolic Execution

```bash
# ใช้ hevm สำหรับ symbolic execution

# ติดตั้ง hevm
nix-env -iA nixpkgs.hevm

# Symbolic execution ตรวจสอบ equivalence ระหว่าง 2 implementations
hevm equivalence \
    --code-a "$(cat build/VerifiableERC20.bin)" \
    --code-b "$(cat build/OptimizedERC20.bin)" \
    --sig "transfer(address,uint256)"

# ตรวจสอบ safety properties
hevm symbolic \
    --code "$(cat build/VerifiableERC20.bin)" \
    --sig "transfer(address,uint256)" \
    --get-models
```

---

## 11. Scribble - Specification Language สำหรับ Solidity/Vyper

```javascript
// scribble annotations (ใช้ใน comments)

// invariant: totalSupply ต้อง >= sum of all balances
/// #if_updated invariant {:msg "total supply conservation"} 
///     old(totalSupply) + amount == totalSupply;

// postcondition: transfer ต้องลด balance ของ sender
/// @notice postcondition {:msg "sender balance decreases"} 
///     old(balanceOf[msg.sender]) - amount == balanceOf[msg.sender];
```

---

## 12. ตัวอย่าง: Lending Protocol Invariants

```vyper
# @version 0.4.0
# @title Verifiable Lending Protocol
# @notice Simplified lending protocol สำหรับ formal verification

struct Position:
    collateral: uint256
    debt: uint256

# State
collateralToken: public(address)
debtToken: public(address)
positions: public(HashMap[address, Position])
totalCollateral: public(uint256)
totalDebt: public(uint256)
collateralFactor: public(uint256)  # in basis points (e.g., 7500 = 75%)
liquidationBonus: public(uint256)  # e.g., 500 = 5% bonus

PRECISION: constant(uint256) = 10000

@deploy
def __init__(
    _collateral: address,
    _debt: address,
    _cf: uint256,
    _lb: uint256
):
    self.collateralToken = _collateral
    self.debtToken = _debt
    self.collateralFactor = _cf
    self.liquidationBonus = _lb

@internal
@view
def _isHealthy(user: address) -> bool:
    """
    ตรวจสอบว่า position ยังแข็งแรงหรือไม่
    collateral * CF >= debt
    """
    pos: Position = self.positions[user]
    if pos.debt == 0:
        return True
    return pos.collateral * self.collateralFactor / PRECISION >= pos.debt

@external
def deposit(amount: uint256):
    """ฝาก collateral"""
    assert amount > 0, "Invalid amount"
    
    ERC20(self.collateralToken).transferFrom(msg.sender, self, amount)
    
    self.positions[msg.sender].collateral += amount
    self.totalCollateral += amount

@external
def borrow(amount: uint256):
    """กู้ยืม tokens"""
    assert amount > 0, "Invalid amount"
    
    self.positions[msg.sender].debt += amount
    
    assert self._isHealthy(msg.sender), "Position would be undercollateralized"
    
    self.totalDebt += amount
    ERC20(self.debtToken).transfer(msg.sender, amount)

@external
def repay(amount: uint256):
    """ชำระหนี้"""
    assert amount > 0, "Invalid amount"
    assert self.positions[msg.sender].debt >= amount, "Repay too much"
    
    ERC20(self.debtToken).transferFrom(msg.sender, self, amount)
    
    self.positions[msg.sender].debt -= amount
    self.totalDebt -= amount

@external
def withdraw(amount: uint256):
    """ถอน collateral"""
    assert amount > 0, "Invalid amount"
    assert self.positions[msg.sender].collateral >= amount, "Insufficient collateral"
    
    self.positions[msg.sender].collateral -= amount
    
    assert self._isHealthy(msg.sender), "Would become undercollateralized"
    
    self.totalCollateral -= amount
    ERC20(self.collateralToken).transfer(msg.sender, amount)

@external
def liquidate(borrower: address, repay_amount: uint256):
    """
    Liquidate unhealthy position
    สำหรับ formal verification: invariant ว่า liquidation ต้องทำให้ position healthy
    """
    assert not self._isHealthy(borrower), "Position is healthy"
    assert repay_amount > 0, "Invalid repay amount"
    assert self.positions[borrower].debt >= repay_amount, "Repay too much"
    
    # คำนวณ collateral ที่ liquidator ได้รับ (รวม bonus)
    collateral_received: uint256 = repay_amount * (PRECISION + self.liquidationBonus) / PRECISION
    assert self.positions[borrower].collateral >= collateral_received, "Insufficient collateral"
    
    # อัพเดต state
    ERC20(self.debtToken).transferFrom(msg.sender, self, repay_amount)
    
    self.positions[borrower].debt -= repay_amount
    self.positions[borrower].collateral -= collateral_received
    self.totalDebt -= repay_amount
    self.totalCollateral -= collateral_received
    
    ERC20(self.collateralToken).transfer(msg.sender, collateral_received)
```

---

## 13. CVL สำหรับ Lending Protocol

```javascript
// specs/LendingProtocol.spec

methods {
    function totalCollateral() external returns (uint256) envfree;
    function totalDebt() external returns (uint256) envfree;
    function deposit(uint256) external;
    function borrow(uint256) external;
    function repay(uint256) external;
    function withdraw(uint256) external;
    function liquidate(address, uint256) external;
    function collateralFactor() external returns (uint256) envfree;
}

// Ghost: ติดตาม total positions
ghost mathint sumCollateral {
    init_state axiom sumCollateral == 0;
}

ghost mathint sumDebt {
    init_state axiom sumDebt == 0;
}

// Invariant 1: totalCollateral ต้องสอดคล้องกับ individual positions
invariant collateralAccounting()
    to_mathint(totalCollateral()) == sumCollateral

// Invariant 2: ไม่มีการสร้าง debt จากอากาศ
invariant debtAccounting()
    to_mathint(totalDebt()) == sumDebt

// Rule: liquidation ต้องเกิดขึ้นเฉพาะกับ unhealthy positions
rule liquidationRequiresUnhealthyPosition(
    address borrower, 
    uint256 amount
) {
    env e;
    
    // ถ้า liquidation สำเร็จ แสดงว่า position ต้องไม่ healthy ก่อน
    liquidate@withrevert(e, borrower, amount);
    
    bool reverted = lastReverted;
    
    // ถ้าไม่ revert แสดงว่า position ต้องไม่ healthy
    assert !reverted => !isHealthy(borrower),
        "Liquidation should only work on unhealthy positions";
}

// Rule: deposit ต้องเพิ่ม collateral
rule depositIncreasesCollateral(uint256 amount) {
    env e;
    
    uint256 before = totalCollateral();
    
    deposit(e, amount);
    
    assert totalCollateral() == before + amount,
        "Deposit should increase total collateral";
}
```

---

## 14. สรุป Best Practices สำหรับ Formal Verification

### หลักการเขียน Specifications ที่ดี

1. **เริ่มจาก invariants** - หาคุณสมบัติที่ต้องเป็นจริงตลอดเวลา
2. **เขียน rules สำหรับแต่ละ function** - พิสูจน์ว่าแต่ละ function ทำงานถูกต้อง
3. **ใช้ ghost variables** - ติดตาม aggregate state ที่ไม่ได้เก็บใน contract
4. **ทดสอบ specifications ด้วย mutation testing** - ตรวจสอบว่า specs จับ bugs ได้
5. **เริ่มจาก simple properties แล้วเพิ่มความซับซ้อน**

### Common Invariants สำหรับ DeFi

```javascript
// ERC-20 invariants
totalSupply() >= sum(balances)  // solvency
balanceOf(x) >= 0 for all x    // non-negative
transfer doesn't change totalSupply

// AMM invariants  
k = reserve0 * reserve1 ต้องไม่ลดลงหลัง swap
sum(LP shares) == totalLPSupply
reserves > 0 เมื่อมี liquidity

// Lending invariants
totalCollateral >= sum(collateral positions)
totalDebt >= sum(debt positions)  
healthy position: collateral * CF >= debt
```

### เครื่องมือตามระดับความยาก

| ระดับ | เครื่องมือ | ใช้สำหรับ |
|-------|-----------|-----------|
| เริ่มต้น | Echidna | Property-based fuzzing |
| กลาง | Halmos | Symbolic testing ใน Foundry |
| สูง | Certora | Production-grade verification |
| ผู้เชี่ยวชาญ | hevm | Low-level symbolic execution |

---

## แบบฝึกหัด

1. เขียน CVL specification สำหรับ staking contract ที่มี reward distribution
2. พิสูจน์ว่า multi-sig wallet ต้องการ N/M signatures เสมอ
3. เขียน invariants สำหรับ lending protocol ที่มี multiple collateral types
4. ใช้ Halmos ทดสอบ symbolic properties ของ ERC-20
5. ทำ mutation testing บน specifications ที่คุณเขียน

---

*จบ Part 077: Formal Verification สำหรับ Smart Contracts*
