# Part 045: Foundry Integration with Vyper

## สารบัญ
1. [Overview](#overview)
2. [Project Setup](#setup)
3. [forge.toml Configuration](#forgetoml)
4. [Writing Forge Tests](#tests)
5. [Forge Script Deployment](#scripts)
6. [Cast Commands](#cast)
7. [Fuzz Testing](#fuzz)
8. [Complete Test Contract](#complete)

---

## 1. Overview {#overview}

Foundry เป็น fast, portable, modular toolkit สำหรับ Ethereum application development ที่เขียนด้วย Rust รองรับการ test Vyper contracts ผ่าน `vyper` compiler

### ข้อดีของ Foundry

- **เร็วมาก**: Compile และ run tests เร็วกว่า Hardhat หลายเท่า
- **Solidity tests**: เขียน tests ด้วย Solidity (ง่ายสำหรับ on-chain testing)
- **Built-in fuzzing**: Fuzzing มาในตัวโดยไม่ต้องติดตั้ง library เพิ่ม
- **Cheatcodes**: `vm.prank`, `vm.warp`, `vm.roll` สำหรับ manipulate state
- **Gas snapshots**: เปรียบเทียบ gas ระหว่าง versions

---

## 2. Project Setup {#setup}

### ติดตั้ง Foundry

```bash
# ติดตั้ง foundryup
curl -L https://foundry.paradigm.xyz | bash

# ติดตั้ง forge, cast, anvil
foundryup

# ตรวจสอบ version
forge --version
cast --version
anvil --version
```

### สร้าง Project

```bash
# สร้าง project ใหม่
forge init vyper-foundry-project
cd vyper-foundry-project

# ติดตั้ง vyper
pip install vyper==0.4.0

# ตรวจสอบ vyper
vyper --version
```

### Project Structure

```
vyper-foundry-project/
├── src/                    # Vyper source contracts
│   ├── Token.vy
│   ├── Staking.vy
│   └── NFT.vy
├── test/                   # Solidity test files
│   ├── Token.t.sol
│   ├── Staking.t.sol
│   └── utils/
│       └── Helper.sol
├── script/                 # Forge scripts
│   ├── Deploy.s.sol
│   └── Interact.s.sol
├── lib/                    # Dependencies (git submodules)
│   └── forge-std/
├── out/                    # Compiled output
├── cache/                  # Build cache
├── foundry.toml            # Configuration
└── .env
```

---

## 3. forge.toml Configuration {#forgetoml}

### foundry.toml

```toml
[profile.default]
# Source and output directories
src = "src"
out = "out"
libs = ["lib"]
test = "test"
script = "script"

# Vyper compiler settings
vyper_path = "vyper"    # path to vyper compiler

# Compiler settings
optimizer = true
optimizer_runs = 200
via_ir = false

# Test settings
verbosity = 2           # 0-5, higher = more output
fuzz_runs = 256         # Number of fuzz runs per test
fuzz_seed = 12345       # Seed for reproducible fuzzing

# Gas settings
gas_reports = ["Token", "Staking", "NFT"]

# Remappings
remappings = [
  "forge-std/=lib/forge-std/src/",
  "@openzeppelin/=lib/openzeppelin-contracts/"
]

# Formatter
[fmt]
line_length = 100
tab_width = 4
bracket_spacing = true

# Fuzz settings
[fuzz]
runs = 1000
seed = "0x3e8"
max_test_rejects = 65536

# Invariant test settings
[invariant]
runs = 256
depth = 15
```

---

## 4. Vyper Contracts for Testing

### src/Token.vy

```vyper
# @version 0.4.0
# @title FoundryToken
# @notice ERC20 สำหรับทดสอบกับ Foundry

from vyper.interfaces import ERC20
implements: ERC20

event Transfer:
    sender: indexed(address)
    receiver: indexed(address)
    value: uint256

event Approval:
    owner: indexed(address)
    spender: indexed(address)
    value: uint256

NAME: public(immutable(String[64]))
SYMBOL: public(immutable(String[32]))
DECIMALS: public(immutable(uint8))
MAX_SUPPLY: public(immutable(uint256))

total_supply: public(uint256)
balances: HashMap[address, uint256]
allowances: HashMap[address, HashMap[address, uint256]]
owner: public(address)
is_paused: public(bool)
minters: public(HashMap[address, bool])

@deploy
def __init__(name: String[64], symbol: String[32], max_supply: uint256):
    NAME = name
    SYMBOL = symbol
    DECIMALS = 18
    MAX_SUPPLY = max_supply
    self.owner = msg.sender

@external
@view
def name() -> String[64]:
    return NAME

@external
@view
def symbol() -> String[32]:
    return SYMBOL

@external
@view
def decimals() -> uint8:
    return DECIMALS

@external
@view
def totalSupply() -> uint256:
    return self.total_supply

@external
@view
def balanceOf(account: address) -> uint256:
    return self.balances[account]

@external
@view
def allowance(owner: address, spender: address) -> uint256:
    return self.allowances[owner][spender]

@external
def transfer(to: address, amount: uint256) -> bool:
    assert not self.is_paused, "Paused"
    assert to != empty(address), "Zero address"
    assert self.balances[msg.sender] >= amount, "Insufficient balance"
    self.balances[msg.sender] -= amount
    self.balances[to] += amount
    log Transfer(msg.sender, to, amount)
    return True

@external
def transferFrom(sender: address, to: address, amount: uint256) -> bool:
    assert not self.is_paused, "Paused"
    assert to != empty(address), "Zero address"
    assert self.balances[sender] >= amount, "Insufficient balance"
    assert self.allowances[sender][msg.sender] >= amount, "Insufficient allowance"
    self.balances[sender] -= amount
    self.balances[to] += amount
    self.allowances[sender][msg.sender] -= amount
    log Transfer(sender, to, amount)
    return True

@external
def approve(spender: address, amount: uint256) -> bool:
    assert spender != empty(address), "Zero address"
    self.allowances[msg.sender][spender] = amount
    log Approval(msg.sender, spender, amount)
    return True

@external
def mint(to: address, amount: uint256):
    assert msg.sender == self.owner or self.minters[msg.sender], "Unauthorized"
    assert to != empty(address), "Zero address"
    assert self.total_supply + amount <= MAX_SUPPLY, "Exceeds max"
    self.total_supply += amount
    self.balances[to] += amount
    log Transfer(empty(address), to, amount)

@external
def burn(amount: uint256):
    assert self.balances[msg.sender] >= amount, "Insufficient"
    self.balances[msg.sender] -= amount
    self.total_supply -= amount
    log Transfer(msg.sender, empty(address), amount)

@external
def set_paused(paused: bool):
    assert msg.sender == self.owner, "Not owner"
    self.is_paused = paused

@external
def set_minter(minter: address, status: bool):
    assert msg.sender == self.owner, "Not owner"
    self.minters[minter] = status

@external
def transfer_ownership(new_owner: address):
    assert msg.sender == self.owner, "Not owner"
    assert new_owner != empty(address), "Zero address"
    self.owner = new_owner
```

---

## 5. Writing Forge Tests {#tests}

### test/Token.t.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "forge-std/Test.sol";
import "forge-std/console.sol";

// Interface for Vyper Token
interface IToken {
    function name() external view returns (string memory);
    function symbol() external view returns (string memory);
    function decimals() external view returns (uint8);
    function totalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function allowance(address owner, address spender) external view returns (uint256);
    function transfer(address to, uint256 amount) external returns (bool);
    function transferFrom(address sender, address to, uint256 amount) external returns (bool);
    function approve(address spender, uint256 amount) external returns (bool);
    function mint(address to, uint256 amount) external;
    function burn(uint256 amount) external;
    function set_paused(bool paused) external;
    function set_minter(address minter, bool status) external;
    function transfer_ownership(address new_owner) external;
    function owner() external view returns (address);
    function is_paused() external view returns (bool);
    function MAX_SUPPLY() external view returns (uint256);
    function minters(address account) external view returns (bool);
}

contract TokenTest is Test {
    IToken public token;
    
    address public deployer = address(this);
    address public alice = address(0xA11CE);
    address public bob = address(0xB0B);
    address public charlie = address(0xC0C0);
    
    uint256 public constant INITIAL_SUPPLY = 1_000_000 * 1e18;
    uint256 public constant MAX_SUPPLY = 1_000_000_000 * 1e18;
    
    event Transfer(address indexed sender, address indexed receiver, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
    
    function setUp() public {
        // Deploy Vyper contract using forge
        // Note: Vyper contracts need to be compiled separately
        // In practice, you'd use vm.getCode() with the compiled bytecode
        
        // For demonstration, using mock deployment
        bytes memory bytecode = abi.encodePacked(
            vm.getCode("Token.vy"),
            abi.encode("Test Token", "TEST", MAX_SUPPLY)
        );
        
        address tokenAddr;
        assembly {
            tokenAddr := create(0, add(bytecode, 0x20), mload(bytecode))
        }
        
        require(tokenAddr != address(0), "Deployment failed");
        token = IToken(tokenAddr);
        
        // Mint initial supply to deployer
        token.mint(deployer, INITIAL_SUPPLY);
    }
    
    // ==================== Deployment Tests ====================
    
    function test_name() public {
        assertEq(token.name(), "Test Token");
    }
    
    function test_symbol() public {
        assertEq(token.symbol(), "TEST");
    }
    
    function test_decimals() public {
        assertEq(token.decimals(), 18);
    }
    
    function test_owner() public {
        assertEq(token.owner(), deployer);
    }
    
    function test_initialSupply() public {
        assertEq(token.totalSupply(), INITIAL_SUPPLY);
        assertEq(token.balanceOf(deployer), INITIAL_SUPPLY);
    }
    
    // ==================== Transfer Tests ====================
    
    function test_transfer() public {
        uint256 amount = 100 * 1e18;
        
        vm.expectEmit(true, true, false, true, address(token));
        emit Transfer(deployer, alice, amount);
        
        bool success = token.transfer(alice, amount);
        
        assertTrue(success);
        assertEq(token.balanceOf(deployer), INITIAL_SUPPLY - amount);
        assertEq(token.balanceOf(alice), amount);
    }
    
    function test_transfer_insufficientBalance() public {
        vm.prank(alice);
        vm.expectRevert();
        token.transfer(bob, 1);
    }
    
    function test_transfer_toZeroAddress() public {
        vm.expectRevert();
        token.transfer(address(0), 1);
    }
    
    function test_transfer_whenPaused() public {
        token.set_paused(true);
        
        vm.expectRevert();
        token.transfer(alice, 100);
    }
    
    // ==================== Approve Tests ====================
    
    function test_approve() public {
        uint256 amount = 500 * 1e18;
        
        vm.expectEmit(true, true, false, true, address(token));
        emit Approval(deployer, alice, amount);
        
        bool success = token.approve(alice, amount);
        
        assertTrue(success);
        assertEq(token.allowance(deployer, alice), amount);
    }
    
    function test_transferFrom() public {
        uint256 amount = 100 * 1e18;
        
        token.approve(alice, 500 * 1e18);
        
        vm.prank(alice);
        token.transferFrom(deployer, bob, amount);
        
        assertEq(token.balanceOf(bob), amount);
        assertEq(token.allowance(deployer, alice), 400 * 1e18);
    }
    
    // ==================== Mint & Burn Tests ====================
    
    function test_mint() public {
        uint256 amount = 1000 * 1e18;
        uint256 supplyBefore = token.totalSupply();
        
        token.mint(alice, amount);
        
        assertEq(token.balanceOf(alice), amount);
        assertEq(token.totalSupply(), supplyBefore + amount);
    }
    
    function test_mint_exceedsMaxSupply() public {
        uint256 currentSupply = token.totalSupply();
        uint256 remaining = MAX_SUPPLY - currentSupply;
        
        vm.expectRevert();
        token.mint(alice, remaining + 1);
    }
    
    function test_burn() public {
        uint256 amount = 100 * 1e18;
        uint256 supplyBefore = token.totalSupply();
        uint256 balanceBefore = token.balanceOf(deployer);
        
        token.burn(amount);
        
        assertEq(token.balanceOf(deployer), balanceBefore - amount);
        assertEq(token.totalSupply(), supplyBefore - amount);
    }
    
    // ==================== Access Control Tests ====================
    
    function test_onlyOwner_setMinter() public {
        vm.prank(alice);
        vm.expectRevert();
        token.set_minter(bob, true);
    }
    
    function test_minter_canMint() public {
        token.set_minter(alice, true);
        
        vm.prank(alice);
        token.mint(bob, 1000 * 1e18);
        
        assertEq(token.balanceOf(bob), 1000 * 1e18);
    }
    
    function test_transferOwnership() public {
        token.transfer_ownership(alice);
        
        assertEq(token.owner(), alice);
        
        // Old owner cannot perform admin actions
        vm.expectRevert();
        token.mint(bob, 1);
    }
    
    // ==================== Gas Tests ====================
    
    function test_gas_transfer() public {
        uint256 gasBefore = gasleft();
        token.transfer(alice, 100 * 1e18);
        uint256 gasUsed = gasBefore - gasleft();
        
        console.log("Gas used for transfer:", gasUsed);
        assertLt(gasUsed, 100000); // Should use less than 100k gas
    }
}
```

---

## 6. Fuzz Testing {#fuzz}

### test/TokenFuzz.t.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "forge-std/Test.sol";
import "./Token.t.sol";

contract TokenFuzzTest is Test {
    IToken public token;
    
    address public owner = address(this);
    
    uint256 public constant MAX_SUPPLY = 1_000_000_000 * 1e18;
    
    function setUp() public {
        bytes memory bytecode = abi.encodePacked(
            vm.getCode("Token.vy"),
            abi.encode("Fuzz Token", "FUZZ", MAX_SUPPLY)
        );
        
        address tokenAddr;
        assembly {
            tokenAddr := create(0, add(bytecode, 0x20), mload(bytecode))
        }
        
        token = IToken(tokenAddr);
        token.mint(owner, MAX_SUPPLY / 2);
    }
    
    // ==================== Fuzz Tests ====================
    
    /// @notice Transfer ใดๆ ต้องไม่เปลี่ยน total supply
    function testFuzz_transfer_preservesTotalSupply(
        address to,
        uint256 amount
    ) public {
        vm.assume(to != address(0));
        vm.assume(to != owner);
        vm.assume(amount > 0);
        vm.assume(amount <= token.balanceOf(owner));
        
        uint256 totalBefore = token.totalSupply();
        token.transfer(to, amount);
        uint256 totalAfter = token.totalSupply();
        
        assertEq(totalBefore, totalAfter);
    }
    
    /// @notice Balance ของ sender ต้องลดลง amount หลัง transfer
    function testFuzz_transfer_decreasesSenderBalance(
        address to,
        uint256 amount
    ) public {
        vm.assume(to != address(0));
        vm.assume(to != owner);
        vm.assume(amount > 0);
        vm.assume(amount <= token.balanceOf(owner));
        
        uint256 balanceBefore = token.balanceOf(owner);
        token.transfer(to, amount);
        
        assertEq(token.balanceOf(owner), balanceBefore - amount);
    }
    
    /// @notice Balance ของ receiver ต้องเพิ่มขึ้น amount หลัง transfer
    function testFuzz_transfer_increasesReceiverBalance(
        address to,
        uint256 amount
    ) public {
        vm.assume(to != address(0));
        vm.assume(to != owner);
        vm.assume(amount > 0);
        vm.assume(amount <= token.balanceOf(owner));
        
        uint256 balanceBefore = token.balanceOf(to);
        token.transfer(to, amount);
        
        assertEq(token.balanceOf(to), balanceBefore + amount);
    }
    
    /// @notice Approve ใดๆ ต้อง set allowance ถูกต้อง
    function testFuzz_approve_setsCorrectAllowance(
        address spender,
        uint256 amount
    ) public {
        vm.assume(spender != address(0));
        
        token.approve(spender, amount);
        
        assertEq(token.allowance(owner, spender), amount);
    }
    
    /// @notice Mint ต้องเพิ่ม total supply และ balance
    function testFuzz_mint_increasesSupplyAndBalance(
        address to,
        uint256 amount
    ) public {
        vm.assume(to != address(0));
        vm.assume(amount > 0);
        vm.assume(token.totalSupply() + amount <= MAX_SUPPLY);
        
        uint256 supplyBefore = token.totalSupply();
        uint256 balanceBefore = token.balanceOf(to);
        
        token.mint(to, amount);
        
        assertEq(token.totalSupply(), supplyBefore + amount);
        assertEq(token.balanceOf(to), balanceBefore + amount);
    }
    
    /// @notice Burn ต้องลด total supply และ balance
    function testFuzz_burn_decreasesSupplyAndBalance(
        uint256 amount
    ) public {
        vm.assume(amount > 0);
        vm.assume(amount <= token.balanceOf(owner));
        
        uint256 supplyBefore = token.totalSupply();
        uint256 balanceBefore = token.balanceOf(owner);
        
        token.burn(amount);
        
        assertEq(token.totalSupply(), supplyBefore - amount);
        assertEq(token.balanceOf(owner), balanceBefore - amount);
    }
    
    /// @notice Transfer ที่ไม่มี balance เพียงพอต้อง revert
    function testFuzz_transfer_revertsWhenInsufficientBalance(
        address to,
        uint256 amount
    ) public {
        vm.assume(to != address(0));
        
        address poorUser = address(0xDEAD);
        vm.assume(amount > token.balanceOf(poorUser));
        
        vm.prank(poorUser);
        vm.expectRevert();
        token.transfer(to, amount);
    }
    
    /// @notice Mint ที่เกิน max supply ต้อง revert
    function testFuzz_mint_revertsWhenExceedsMaxSupply(
        address to,
        uint256 excess
    ) public {
        vm.assume(to != address(0));
        vm.assume(excess > 0);
        
        uint256 remaining = MAX_SUPPLY - token.totalSupply();
        uint256 tooMuch = remaining + excess;
        
        // Prevent overflow
        vm.assume(tooMuch > remaining);
        
        vm.expectRevert();
        token.mint(to, tooMuch);
    }
}
```

### test/TokenInvariant.t.sol - Invariant Tests

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "forge-std/Test.sol";
import "./Token.t.sol";

/// @notice Handler contract สำหรับ invariant testing
contract TokenHandler is Test {
    IToken public token;
    
    address[] public actors;
    
    constructor(address _token, address[] memory _actors) {
        token = IToken(_token);
        actors = _actors;
    }
    
    function transfer(
        uint256 actorSeed,
        uint256 recipientSeed,
        uint256 amount
    ) external {
        address actor = actors[actorSeed % actors.length];
        address recipient = actors[recipientSeed % actors.length];
        
        if (actor == recipient) return;
        
        uint256 balance = token.balanceOf(actor);
        if (balance == 0) return;
        
        amount = bound(amount, 1, balance);
        
        vm.prank(actor);
        token.transfer(recipient, amount);
    }
    
    function mint(
        uint256 actorSeed,
        uint256 amount
    ) external {
        address actor = actors[actorSeed % actors.length];
        
        uint256 remaining = token.MAX_SUPPLY() - token.totalSupply();
        if (remaining == 0) return;
        
        amount = bound(amount, 1, remaining);
        
        // Only owner can mint (index 0)
        vm.prank(actors[0]);
        token.mint(actor, amount);
    }
    
    function burn(
        uint256 actorSeed,
        uint256 amount
    ) external {
        address actor = actors[actorSeed % actors.length];
        uint256 balance = token.balanceOf(actor);
        
        if (balance == 0) return;
        
        amount = bound(amount, 1, balance);
        
        vm.prank(actor);
        token.burn(amount);
    }
}

contract TokenInvariantTest is Test {
    IToken public token;
    TokenHandler public handler;
    
    address[] public actors;
    address public owner = address(0x1);
    
    uint256 public constant MAX_SUPPLY = 1_000_000_000 * 1e18;
    
    function setUp() public {
        // Deploy token
        vm.prank(owner);
        bytes memory bytecode = abi.encodePacked(
            vm.getCode("Token.vy"),
            abi.encode("Invariant Token", "INV", MAX_SUPPLY)
        );
        
        address tokenAddr;
        assembly {
            tokenAddr := create(0, add(bytecode, 0x20), mload(bytecode))
        }
        token = IToken(tokenAddr);
        
        // Create actors
        for (uint256 i = 0; i < 5; i++) {
            actors.push(address(uint160(i + 2)));
        }
        actors.push(owner);
        
        // Deploy handler
        handler = new TokenHandler(address(token), actors);
        
        // Target handler for invariant testing
        targetContract(address(handler));
    }
    
    /// @notice Total supply ต้องไม่เกิน MAX_SUPPLY เสมอ
    function invariant_totalSupplyNeverExceedsMax() public {
        assertLe(token.totalSupply(), MAX_SUPPLY);
    }
    
    /// @notice Total supply ต้องเท่ากับผลรวม balances เสมอ
    function invariant_totalSupplyEqualsSumOfBalances() public {
        uint256 sumBalances = 0;
        for (uint256 i = 0; i < actors.length; i++) {
            sumBalances += token.balanceOf(actors[i]);
        }
        
        // Note: ยังมี balance ของ handler contract เอง ด้วย
        sumBalances += token.balanceOf(address(handler));
        
        assertLe(sumBalances, token.totalSupply());
    }
    
    /// @notice Total supply ต้องไม่เป็น negative (ไม่มี underflow)
    function invariant_totalSupplyNonNegative() public {
        assertGe(token.totalSupply(), 0);
    }
    
    function afterInvariant() public view {
        console.log("Total Supply:", token.totalSupply());
        console.log("Max Supply:", MAX_SUPPLY);
    }
}
```

---

## 7. Forge Script Deployment {#scripts}

### script/Deploy.s.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "forge-std/Script.sol";
import "./Token.t.sol";

contract DeployScript is Script {
    function run() external {
        uint256 deployerPrivateKey = vm.envUint("PRIVATE_KEY");
        address deployer = vm.addr(deployerPrivateKey);
        
        console.log("Deploying from:", deployer);
        console.log("Balance:", deployer.balance);
        
        vm.startBroadcast(deployerPrivateKey);
        
        // Deploy Token
        bytes memory tokenBytecode = abi.encodePacked(
            vm.getCode("Token.vy"),
            abi.encode(
                "My Token",
                "MTK",
                uint256(1_000_000_000 * 1e18)
            )
        );
        
        address tokenAddr;
        assembly {
            tokenAddr := create(0, add(tokenBytecode, 0x20), mload(tokenBytecode))
        }
        
        require(tokenAddr != address(0), "Token deployment failed");
        console.log("Token deployed at:", tokenAddr);
        
        // Mint initial supply
        IToken token = IToken(tokenAddr);
        token.mint(deployer, 100_000_000 * 1e18);
        console.log("Minted 100M tokens to deployer");
        
        vm.stopBroadcast();
        
        // Log deployment summary
        console.log("\n=== Deployment Summary ===");
        console.log("Token:", tokenAddr);
        console.log("Total Supply:", token.totalSupply());
    }
}
```

---

## 8. Cast Commands {#cast}

```bash
# ==================== Cast Command Examples ====================

# ตั้งค่า environment
export TOKEN_ADDRESS="0x..."
export PRIVATE_KEY="0x..."
export RPC_URL="http://localhost:8545"

# อ่าน token info
cast call $TOKEN_ADDRESS "name()" --rpc-url $RPC_URL | cast --to-ascii
cast call $TOKEN_ADDRESS "symbol()" --rpc-url $RPC_URL | cast --to-ascii
cast call $TOKEN_ADDRESS "totalSupply()" --rpc-url $RPC_URL | cast --to-dec

# ตรวจสอบ balance
cast call $TOKEN_ADDRESS "balanceOf(address)(uint256)" 0x... --rpc-url $RPC_URL

# ตรวจสอบ allowance
cast call $TOKEN_ADDRESS "allowance(address,address)(uint256)" 0x... 0x... --rpc-url $RPC_URL

# ส่ง transaction (mint)
cast send $TOKEN_ADDRESS "mint(address,uint256)" 0x... 1000000000000000000000 \
    --private-key $PRIVATE_KEY \
    --rpc-url $RPC_URL

# Transfer tokens
cast send $TOKEN_ADDRESS "transfer(address,uint256)(bool)" 0x... 100000000000000000000 \
    --private-key $PRIVATE_KEY \
    --rpc-url $RPC_URL

# Approve
cast send $TOKEN_ADDRESS "approve(address,uint256)(bool)" 0x... 115792089237316195423570985008687907853269984665640564039457584007913129639935 \
    --private-key $PRIVATE_KEY \
    --rpc-url $RPC_URL

# Get transaction receipt
cast tx 0x...tx_hash... --rpc-url $RPC_URL

# Get block info
cast block latest --rpc-url $RPC_URL

# ABI decode
cast 4byte "transfer(address,uint256)"
cast pretty-calldata 0xa9059cbb...

# Convert units
cast --to-wei 1.5 ether
cast --from-wei 1500000000000000000
cast --to-hex 255
cast --to-dec 0xff

# Run local anvil
anvil --block-time 1 --chain-id 31337
anvil --fork-url https://mainnet.infura.io/v3/... --fork-block-number 18000000
```

---

## Complete Test Suite runner

```bash
# รัน tests ทั้งหมด
forge test

# รัน tests แบบ verbose
forge test -vvvv

# รัน specific test file
forge test --match-path test/Token.t.sol

# รัน specific test function
forge test --match-test test_transfer

# รัน fuzz tests
forge test --match-test testFuzz_ -v

# รัน invariant tests
forge test --match-contract Invariant -v

# Gas report
forge test --gas-report

# Gas snapshot (สร้าง baseline)
forge snapshot

# เปรียบเทียบ gas กับ snapshot ก่อนหน้า
forge snapshot --check

# Coverage
forge coverage

# Coverage report (HTML)
forge coverage --report lcov
genhtml lcov.info --output-directory coverage
```

### Makefile

```makefile
# Makefile สำหรับ Foundry project

.PHONY: all build test clean deploy

all: build test

build:
	forge build

test:
	forge test -vv

test-verbose:
	forge test -vvvv

fuzz:
	forge test --match-test testFuzz_ -v

invariant:
	forge test --match-contract Invariant -v

gas:
	forge test --gas-report

snapshot:
	forge snapshot

coverage:
	forge coverage

clean:
	forge clean

deploy-local:
	forge script script/Deploy.s.sol --rpc-url http://localhost:8545 --broadcast

deploy-sepolia:
	forge script script/Deploy.s.sol --rpc-url $(SEPOLIA_RPC) --broadcast --verify

format:
	forge fmt

lint:
	forge fmt --check
```
