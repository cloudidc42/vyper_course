# Part 044: Hardhat Integration with Vyper

## สารบัญ
1. [Overview](#overview)
2. [Project Setup](#setup)
3. [Hardhat Configuration](#config)
4. [TypeScript Tests](#tests)
5. [Deploy Scripts](#deploy)
6. [Verify on Etherscan](#verify)
7. [Complete Examples](#examples)

---

## 1. Overview {#overview}

Hardhat เป็น development environment ยอดนิยมสำหรับ Ethereum ซึ่งรองรับการ compile Vyper contracts ผ่าน plugin `@nomiclabs/hardhat-vyper`

### ข้อดีของ Hardhat

- **TypeScript support**: เขียน test และ script ด้วย TypeScript
- **Network management**: จัดการ network หลายเครื่องได้ง่าย
- **Plugin ecosystem**: มี plugins จำนวนมาก
- **Etherscan integration**: verify contract ได้โดยตรง
- **console.log**: debug ได้ง่าย

---

## 2. Project Setup {#setup}

### การติดตั้ง

```bash
# สร้าง project ใหม่
mkdir vyper-hardhat-project
cd vyper-hardhat-project

# Initialize npm project
npm init -y

# ติดตั้ง Hardhat
npm install --save-dev hardhat

# ติดตั้ง TypeScript dependencies
npm install --save-dev \
  typescript \
  ts-node \
  @types/node \
  @types/chai \
  @types/mocha

# ติดตั้ง Hardhat plugins
npm install --save-dev \
  @nomiclabs/hardhat-ethers \
  @nomiclabs/hardhat-vyper \
  @nomiclabs/hardhat-etherscan \
  ethers \
  chai

# ติดตั้ง testing tools
npm install --save-dev \
  @nomicfoundation/hardhat-chai-matchers \
  @nomicfoundation/hardhat-network-helpers

# Initialize Hardhat (เลือก TypeScript project)
npx hardhat init
```

### Project Structure

```
vyper-hardhat-project/
├── contracts/
│   ├── Token.vy
│   ├── Staking.vy
│   └── interfaces/
│       └── IToken.vy
├── scripts/
│   ├── deploy.ts
│   └── verify.ts
├── test/
│   ├── Token.test.ts
│   ├── Staking.test.ts
│   └── helpers/
│       └── index.ts
├── deployments/
│   ├── sepolia.json
│   └── mainnet.json
├── hardhat.config.ts
├── tsconfig.json
├── package.json
└── .env
```

### tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "strict": true,
    "esModuleInterop": true,
    "outDir": "dist",
    "rootDir": ".",
    "resolveJsonModule": true,
    "skipLibCheck": true
  },
  "include": [
    "**/*.ts"
  ],
  "exclude": [
    "node_modules",
    "dist"
  ]
}
```

---

## 3. Hardhat Configuration {#config}

### hardhat.config.ts

```typescript
import { HardhatUserConfig } from "hardhat/config";
import "@nomiclabs/hardhat-ethers";
import "@nomiclabs/hardhat-vyper";
import "@nomiclabs/hardhat-etherscan";
import "@nomicfoundation/hardhat-chai-matchers";
import "@nomicfoundation/hardhat-network-helpers";
import * as dotenv from "dotenv";

dotenv.config();

const PRIVATE_KEY = process.env.PRIVATE_KEY || "0x" + "0".repeat(64);
const INFURA_API_KEY = process.env.INFURA_API_KEY || "";
const ETHERSCAN_API_KEY = process.env.ETHERSCAN_API_KEY || "";

const config: HardhatUserConfig = {
  // Vyper compiler configuration
  vyper: {
    version: "0.4.0",
    settings: {
      optimize: "gas",  // or "none" or "codesize"
    }
  },
  
  // Network configuration
  networks: {
    // Local development network
    hardhat: {
      chainId: 31337,
      mining: {
        auto: true,
        interval: 0
      },
      // Fork mainnet for testing with real protocols
      // forking: {
      //   url: `https://mainnet.infura.io/v3/${INFURA_API_KEY}`,
      //   blockNumber: 18000000
      // }
    },
    
    // Local Anvil
    localhost: {
      url: "http://127.0.0.1:8545",
      chainId: 31337
    },
    
    // Sepolia Testnet
    sepolia: {
      url: `https://sepolia.infura.io/v3/${INFURA_API_KEY}`,
      accounts: [PRIVATE_KEY],
      chainId: 11155111,
      gasMultiplier: 1.2
    },
    
    // Ethereum Mainnet
    mainnet: {
      url: `https://mainnet.infura.io/v3/${INFURA_API_KEY}`,
      accounts: [PRIVATE_KEY],
      chainId: 1,
      gasMultiplier: 1.1
    }
  },
  
  // Etherscan verification
  etherscan: {
    apiKey: {
      mainnet: ETHERSCAN_API_KEY,
      sepolia: ETHERSCAN_API_KEY
    }
  },
  
  // Paths configuration
  paths: {
    sources: "./contracts",
    tests: "./test",
    cache: "./cache",
    artifacts: "./artifacts"
  }
};

export default config;
```

---

## 4. Vyper Contracts

### contracts/Token.vy

```vyper
# @version 0.4.0
# @title HardhatToken
# @notice ERC20 token สำหรับทดสอบกับ Hardhat

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

event Mint:
    to: indexed(address)
    amount: uint256

NAME: immutable(String[64])
SYMBOL: immutable(String[32])
DECIMALS: immutable(uint8)
MAX_SUPPLY: immutable(uint256)

total_supply: uint256
balances: HashMap[address, uint256]
allowances: HashMap[address, HashMap[address, uint256]]
owner: address
is_paused: bool

@deploy
def __init__(
    name: String[64],
    symbol: String[32],
    max_supply: uint256
):
    NAME = name
    SYMBOL = symbol
    DECIMALS = 18
    MAX_SUPPLY = max_supply
    self.owner = msg.sender
    self.is_paused = False

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
    assert msg.sender == self.owner, "Not owner"
    assert to != empty(address), "Zero address"
    assert self.total_supply + amount <= MAX_SUPPLY, "Exceeds max"
    
    self.total_supply += amount
    self.balances[to] += amount
    log Mint(to, amount)
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
def transfer_ownership(new_owner: address):
    assert msg.sender == self.owner, "Not owner"
    assert new_owner != empty(address), "Zero address"
    self.owner = new_owner
```

### contracts/Staking.vy

```vyper
# @version 0.4.0
# @title SimpleStaking
# @notice Staking contract สำหรับทดสอบกับ Hardhat

interface IERC20:
    def transfer(to: address, amount: uint256) -> bool: nonpayable
    def transferFrom(frm: address, to: address, amount: uint256) -> bool: nonpayable
    def balanceOf(owner: address) -> uint256: view

event Staked:
    user: indexed(address)
    amount: uint256

event Unstaked:
    user: indexed(address)
    amount: uint256

event RewardClaimed:
    user: indexed(address)
    amount: uint256

struct UserInfo:
    amount: uint256
    reward_debt: uint256
    last_stake_time: uint256

STAKING_TOKEN: immutable(address)
REWARD_TOKEN: immutable(address)
REWARD_PER_SECOND: immutable(uint256)

users: public(HashMap[address, UserInfo])
total_staked: public(uint256)
owner: public(address)

@deploy
def __init__(
    staking_token: address,
    reward_token: address,
    reward_per_second: uint256
):
    STAKING_TOKEN = staking_token
    REWARD_TOKEN = reward_token
    REWARD_PER_SECOND = reward_per_second
    self.owner = msg.sender

@internal
@view
def _pending_reward(user: address) -> uint256:
    info: UserInfo = self.users[user]
    if info.amount == 0:
        return 0
    elapsed: uint256 = block.timestamp - info.last_stake_time
    return info.amount * elapsed * REWARD_PER_SECOND // 10**18

@external
@view
def pendingReward(user: address) -> uint256:
    return self._pending_reward(user)

@external
def stake(amount: uint256):
    assert amount > 0, "Cannot stake 0"
    
    info: UserInfo = self.users[msg.sender]
    
    if info.amount > 0:
        pending: uint256 = self._pending_reward(msg.sender)
        info.reward_debt += pending
    
    info.amount += amount
    info.last_stake_time = block.timestamp
    self.users[msg.sender] = info
    
    self.total_staked += amount
    IERC20(STAKING_TOKEN).transferFrom(msg.sender, self, amount)
    log Staked(msg.sender, amount)

@external
def unstake(amount: uint256):
    info: UserInfo = self.users[msg.sender]
    assert info.amount >= amount, "Insufficient staked"
    
    pending: uint256 = self._pending_reward(msg.sender)
    total_reward: uint256 = pending + info.reward_debt
    
    info.amount -= amount
    info.reward_debt = 0
    info.last_stake_time = block.timestamp
    self.users[msg.sender] = info
    
    self.total_staked -= amount
    IERC20(STAKING_TOKEN).transfer(msg.sender, amount)
    
    if total_reward > 0:
        IERC20(REWARD_TOKEN).transfer(msg.sender, total_reward)
    
    log Unstaked(msg.sender, amount)
    log RewardClaimed(msg.sender, total_reward)

@external
def claim_reward():
    pending: uint256 = self._pending_reward(msg.sender)
    total_reward: uint256 = pending + self.users[msg.sender].reward_debt
    
    assert total_reward > 0, "No reward"
    
    self.users[msg.sender].reward_debt = 0
    self.users[msg.sender].last_stake_time = block.timestamp
    
    IERC20(REWARD_TOKEN).transfer(msg.sender, total_reward)
    log RewardClaimed(msg.sender, total_reward)
```

---

## 5. TypeScript Tests {#tests}

### test/helpers/index.ts

```typescript
import { ethers } from "hardhat";
import { BigNumber, Contract, Signer } from "ethers";

export const ZERO_ADDRESS = ethers.constants.AddressZero;
export const MAX_UINT256 = ethers.constants.MaxUint256;

// Token amounts
export const parseToken = (amount: string | number) =>
  ethers.utils.parseEther(amount.toString());

export const formatToken = (amount: BigNumber) =>
  ethers.utils.formatEther(amount);

// Time utilities
export async function increaseTime(seconds: number): Promise<void> {
  await ethers.provider.send("evm_increaseTime", [seconds]);
  await ethers.provider.send("evm_mine", []);
}

export async function getLatestBlockTime(): Promise<number> {
  const block = await ethers.provider.getBlock("latest");
  return block.timestamp;
}

// Token deployment helper
export async function deployToken(
  signer: Signer,
  name: string = "Test Token",
  symbol: string = "TEST",
  maxSupply: BigNumber = parseToken("1000000000")
): Promise<Contract> {
  const TokenFactory = await ethers.getContractFactory("Token", signer);
  const token = await TokenFactory.deploy(name, symbol, maxSupply);
  await token.deployed();
  return token;
}

// Funding helper
export async function fundAccount(
  token: Contract,
  owner: Signer,
  account: string,
  amount: BigNumber
): Promise<void> {
  await token.connect(owner).mint(account, amount);
}
```

### test/Token.test.ts

```typescript
import { expect } from "chai";
import { ethers } from "hardhat";
import { Contract, Signer, BigNumber } from "ethers";
import { loadFixture, time } from "@nomicfoundation/hardhat-network-helpers";
import {
  ZERO_ADDRESS,
  MAX_UINT256,
  parseToken,
  deployToken,
  fundAccount
} from "./helpers";

describe("Token Contract", function () {
  // ==================== Fixtures ====================

  async function deployTokenFixture() {
    const [deployer, alice, bob, charlie, minter] =
      await ethers.getSigners();

    const token = await deployToken(
      deployer,
      "Test Token",
      "TEST",
      parseToken("1000000000")
    );

    return { token, deployer, alice, bob, charlie, minter };
  }

  async function deployTokenWithSupplyFixture() {
    const fixture = await deployTokenFixture();
    const { token, deployer, alice, bob } = fixture;

    await fundAccount(token, deployer, alice.address, parseToken("1000"));
    await fundAccount(token, deployer, bob.address, parseToken("1000"));
    await fundAccount(token, deployer, deployer.address, parseToken("8000"));

    return fixture;
  }

  // ==================== Deployment Tests ====================

  describe("Deployment", function () {
    it("Should set the correct name", async function () {
      const { token } = await loadFixture(deployTokenFixture);
      expect(await token.name()).to.equal("Test Token");
    });

    it("Should set the correct symbol", async function () {
      const { token } = await loadFixture(deployTokenFixture);
      expect(await token.symbol()).to.equal("TEST");
    });

    it("Should set the correct decimals", async function () {
      const { token } = await loadFixture(deployTokenFixture);
      expect(await token.decimals()).to.equal(18);
    });

    it("Should set the deployer as owner", async function () {
      const { token, deployer } = await loadFixture(deployTokenFixture);
      expect(await token.owner()).to.equal(deployer.address);
    });

    it("Should have zero initial supply", async function () {
      const { token } = await loadFixture(deployTokenFixture);
      expect(await token.totalSupply()).to.equal(0);
    });

    it("Should not be paused initially", async function () {
      const { token } = await loadFixture(deployTokenFixture);
      expect(await token.is_paused()).to.equal(false);
    });
  });

  // ==================== Transfer Tests ====================

  describe("Transfer", function () {
    it("Should transfer tokens between accounts", async function () {
      const { token, deployer, alice, bob } =
        await loadFixture(deployTokenWithSupplyFixture);

      const amount = parseToken("100");
      const aliceBalanceBefore = await token.balanceOf(alice.address);

      await token.connect(alice).transfer(bob.address, amount);

      expect(await token.balanceOf(alice.address)).to.equal(
        aliceBalanceBefore.sub(amount)
      );
      expect(await token.balanceOf(bob.address)).to.equal(
        parseToken("1000").add(amount)
      );
    });

    it("Should emit Transfer event", async function () {
      const { token, alice, bob } =
        await loadFixture(deployTokenWithSupplyFixture);

      const amount = parseToken("100");

      await expect(token.connect(alice).transfer(bob.address, amount))
        .to.emit(token, "Transfer")
        .withArgs(alice.address, bob.address, amount);
    });

    it("Should fail when sender has insufficient balance", async function () {
      const { token, alice, bob } =
        await loadFixture(deployTokenFixture);

      await expect(
        token.connect(alice).transfer(bob.address, parseToken("1"))
      ).to.be.reverted;
    });

    it("Should fail when transferring to zero address", async function () {
      const { token, alice } =
        await loadFixture(deployTokenWithSupplyFixture);

      await expect(
        token.connect(alice).transfer(ZERO_ADDRESS, parseToken("1"))
      ).to.be.reverted;
    });

    it("Should fail when paused", async function () {
      const { token, deployer, alice, bob } =
        await loadFixture(deployTokenWithSupplyFixture);

      await token.connect(deployer).set_paused(true);

      await expect(
        token.connect(alice).transfer(bob.address, parseToken("1"))
      ).to.be.reverted;
    });
  });

  // ==================== Approve & TransferFrom ====================

  describe("Approve and TransferFrom", function () {
    it("Should set correct allowance", async function () {
      const { token, alice, bob } =
        await loadFixture(deployTokenFixture);

      const amount = parseToken("500");
      await token.connect(alice).approve(bob.address, amount);

      expect(await token.allowance(alice.address, bob.address)).to.equal(
        amount
      );
    });

    it("Should emit Approval event", async function () {
      const { token, alice, bob } =
        await loadFixture(deployTokenFixture);

      const amount = parseToken("500");

      await expect(token.connect(alice).approve(bob.address, amount))
        .to.emit(token, "Approval")
        .withArgs(alice.address, bob.address, amount);
    });

    it("Should transfer using allowance", async function () {
      const { token, deployer, alice, bob, charlie } =
        await loadFixture(deployTokenWithSupplyFixture);

      await token.connect(alice).approve(bob.address, parseToken("500"));

      await token
        .connect(bob)
        .transferFrom(alice.address, charlie.address, parseToken("200"));

      expect(await token.balanceOf(charlie.address)).to.equal(
        parseToken("200")
      );
      expect(await token.allowance(alice.address, bob.address)).to.equal(
        parseToken("300")
      );
    });

    it("Should fail with insufficient allowance", async function () {
      const { token, alice, bob, charlie } =
        await loadFixture(deployTokenWithSupplyFixture);

      await token.connect(alice).approve(bob.address, parseToken("50"));

      await expect(
        token
          .connect(bob)
          .transferFrom(alice.address, charlie.address, parseToken("100"))
      ).to.be.reverted;
    });
  });

  // ==================== Mint & Burn ====================

  describe("Mint and Burn", function () {
    it("Should mint tokens", async function () {
      const { token, deployer, alice } =
        await loadFixture(deployTokenFixture);

      const amount = parseToken("1000");
      await token.connect(deployer).mint(alice.address, amount);

      expect(await token.balanceOf(alice.address)).to.equal(amount);
      expect(await token.totalSupply()).to.equal(amount);
    });

    it("Should emit Mint event", async function () {
      const { token, deployer, alice } =
        await loadFixture(deployTokenFixture);

      const amount = parseToken("1000");

      await expect(token.connect(deployer).mint(alice.address, amount))
        .to.emit(token, "Mint")
        .withArgs(alice.address, amount);
    });

    it("Should fail when minting exceeds max supply", async function () {
      const { token, deployer, alice } =
        await loadFixture(deployTokenFixture);

      const maxSupply = await token.MAX_SUPPLY();

      await expect(
        token.connect(deployer).mint(alice.address, maxSupply.add(1))
      ).to.be.reverted;
    });

    it("Should burn tokens", async function () {
      const { token, alice } =
        await loadFixture(deployTokenWithSupplyFixture);

      const amount = parseToken("100");
      const balanceBefore = await token.balanceOf(alice.address);
      const supplyBefore = await token.totalSupply();

      await token.connect(alice).burn(amount);

      expect(await token.balanceOf(alice.address)).to.equal(
        balanceBefore.sub(amount)
      );
      expect(await token.totalSupply()).to.equal(supplyBefore.sub(amount));
    });
  });

  // ==================== Access Control ====================

  describe("Access Control", function () {
    it("Owner can pause", async function () {
      const { token, deployer } = await loadFixture(deployTokenFixture);
      await token.connect(deployer).set_paused(true);
      expect(await token.is_paused()).to.equal(true);
    });

    it("Non-owner cannot pause", async function () {
      const { token, alice } = await loadFixture(deployTokenFixture);
      await expect(token.connect(alice).set_paused(true)).to.be.reverted;
    });

    it("Should transfer ownership", async function () {
      const { token, deployer, alice } =
        await loadFixture(deployTokenFixture);

      await token.connect(deployer).transfer_ownership(alice.address);
      expect(await token.owner()).to.equal(alice.address);
    });

    it("Old owner cannot act after transfer", async function () {
      const { token, deployer, alice } =
        await loadFixture(deployTokenFixture);

      await token.connect(deployer).transfer_ownership(alice.address);

      await expect(
        token.connect(deployer).mint(deployer.address, parseToken("1"))
      ).to.be.reverted;
    });
  });
});
```

### test/Staking.test.ts

```typescript
import { expect } from "chai";
import { ethers } from "hardhat";
import { loadFixture, time } from "@nomicfoundation/hardhat-network-helpers";
import { parseToken, deployToken, fundAccount } from "./helpers";

describe("Staking Contract", function () {
  async function deployStakingFixture() {
    const [deployer, alice, bob] = await ethers.getSigners();

    // Deploy tokens
    const stakingToken = await deployToken(
      deployer,
      "Staking Token",
      "STK",
      parseToken("100000000")
    );

    const rewardToken = await deployToken(
      deployer,
      "Reward Token",
      "RWD",
      parseToken("100000000")
    );

    // Deploy staking contract
    const rewardPerSecond = parseToken("0.001"); // 0.001 tokens per second per token staked
    const StakingFactory = await ethers.getContractFactory("Staking");
    const staking = await StakingFactory.deploy(
      stakingToken.address,
      rewardToken.address,
      rewardPerSecond
    );
    await staking.deployed();

    // Fund staking contract with reward tokens
    await rewardToken
      .connect(deployer)
      .mint(staking.address, parseToken("1000000"));

    // Fund users with staking tokens
    await fundAccount(stakingToken, deployer, alice.address, parseToken("10000"));
    await fundAccount(stakingToken, deployer, bob.address, parseToken("10000"));

    // Approve staking contract
    await stakingToken
      .connect(alice)
      .approve(staking.address, ethers.constants.MaxUint256);
    await stakingToken
      .connect(bob)
      .approve(staking.address, ethers.constants.MaxUint256);

    return { staking, stakingToken, rewardToken, deployer, alice, bob, rewardPerSecond };
  }

  describe("Staking", function () {
    it("Should stake tokens", async function () {
      const { staking, stakingToken, alice } =
        await loadFixture(deployStakingFixture);

      const amount = parseToken("1000");
      await staking.connect(alice).stake(amount);

      const userInfo = await staking.users(alice.address);
      expect(userInfo.amount).to.equal(amount);
      expect(await staking.total_staked()).to.equal(amount);
    });

    it("Should emit Staked event", async function () {
      const { staking, alice } = await loadFixture(deployStakingFixture);

      const amount = parseToken("1000");

      await expect(staking.connect(alice).stake(amount))
        .to.emit(staking, "Staked")
        .withArgs(alice.address, amount);
    });

    it("Should accrue rewards over time", async function () {
      const { staking, alice } = await loadFixture(deployStakingFixture);

      await staking.connect(alice).stake(parseToken("1000"));

      // Fast forward time by 100 seconds
      await time.increase(100);

      const pending = await staking.pendingReward(alice.address);
      expect(pending).to.be.gt(0);
    });

    it("Should unstake and receive rewards", async function () {
      const { staking, stakingToken, rewardToken, alice } =
        await loadFixture(deployStakingFixture);

      const stakeAmount = parseToken("1000");
      await staking.connect(alice).stake(stakeAmount);

      const stakingBalanceBefore = await stakingToken.balanceOf(alice.address);
      const rewardBalanceBefore = await rewardToken.balanceOf(alice.address);

      // Fast forward 100 seconds
      await time.increase(100);

      await staking.connect(alice).unstake(stakeAmount);

      const stakingBalanceAfter = await stakingToken.balanceOf(alice.address);
      const rewardBalanceAfter = await rewardToken.balanceOf(alice.address);

      // Got back staked tokens
      expect(stakingBalanceAfter).to.equal(
        stakingBalanceBefore.add(stakeAmount)
      );

      // Got rewards
      expect(rewardBalanceAfter).to.be.gt(rewardBalanceBefore);
    });

    it("Should fail to stake 0", async function () {
      const { staking, alice } = await loadFixture(deployStakingFixture);
      await expect(staking.connect(alice).stake(0)).to.be.reverted;
    });
  });
});
```

---

## 6. Deploy Scripts {#deploy}

### scripts/deploy.ts

```typescript
import { ethers } from "hardhat";
import * as fs from "fs";
import * as path from "path";

interface DeploymentData {
  network: string;
  chainId: number;
  deployer: string;
  contracts: {
    [name: string]: {
      address: string;
      txHash: string;
      blockNumber: number;
    };
  };
  timestamp: number;
}

async function main() {
  const [deployer] = await ethers.getSigners();
  const network = await ethers.provider.getNetwork();

  console.log("=".repeat(50));
  console.log("Deploying contracts...");
  console.log(`Network: ${network.name} (chainId: ${network.chainId})`);
  console.log(`Deployer: ${deployer.address}`);
  console.log(
    `Balance: ${ethers.utils.formatEther(
      await deployer.getBalance()
    )} ETH`
  );
  console.log("=".repeat(50));

  const deploymentData: DeploymentData = {
    network: network.name,
    chainId: network.chainId,
    deployer: deployer.address,
    contracts: {},
    timestamp: Math.floor(Date.now() / 1000),
  };

  // Deploy Token
  console.log("\nDeploying Token...");
  const TokenFactory = await ethers.getContractFactory("Token");
  const token = await TokenFactory.deploy(
    "My Token",
    "MTK",
    ethers.utils.parseEther("1000000000") // 1 billion
  );
  await token.deployed();

  console.log(`Token deployed to: ${token.address}`);

  deploymentData.contracts["Token"] = {
    address: token.address,
    txHash: token.deployTransaction.hash,
    blockNumber: (await token.deployTransaction.wait()).blockNumber,
  };

  // Deploy Staking
  console.log("\nDeploying Staking...");
  const rewardPerSecond = ethers.utils.parseEther("0.0001");
  const StakingFactory = await ethers.getContractFactory("Staking");
  const staking = await StakingFactory.deploy(
    token.address,
    token.address, // Using same token for simplicity
    rewardPerSecond
  );
  await staking.deployed();

  console.log(`Staking deployed to: ${staking.address}`);

  deploymentData.contracts["Staking"] = {
    address: staking.address,
    txHash: staking.deployTransaction.hash,
    blockNumber: (await staking.deployTransaction.wait()).blockNumber,
  };

  // Post-deployment setup
  console.log("\nSetting up contracts...");

  // Mint initial supply to deployer
  await token.mint(
    deployer.address,
    ethers.utils.parseEther("100000000") // 100 million
  );

  console.log("Setup complete!");

  // Save deployment data
  const deploymentDir = path.join(__dirname, "../deployments");
  if (!fs.existsSync(deploymentDir)) {
    fs.mkdirSync(deploymentDir, { recursive: true });
  }

  const filename = path.join(deploymentDir, `${network.name}.json`);
  fs.writeFileSync(filename, JSON.stringify(deploymentData, null, 2));
  console.log(`\nDeployment data saved to: ${filename}`);

  console.log("\n" + "=".repeat(50));
  console.log("Deployment Summary:");
  console.log("=".repeat(50));
  for (const [name, data] of Object.entries(deploymentData.contracts)) {
    console.log(`${name}: ${data.address}`);
  }
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```

---

## 7. Verification {#verify}

### scripts/verify.ts

```typescript
import { run } from "hardhat";
import * as fs from "fs";
import * as path from "path";

async function main() {
  const network = process.env.NETWORK || "sepolia";
  const deploymentFile = path.join(
    __dirname,
    `../deployments/${network}.json`
  );

  if (!fs.existsSync(deploymentFile)) {
    throw new Error(`Deployment file not found: ${deploymentFile}`);
  }

  const deployment = JSON.parse(fs.readFileSync(deploymentFile, "utf8"));

  console.log(`Verifying contracts on ${network}...`);

  // Verify Token
  if (deployment.contracts.Token) {
    console.log(`\nVerifying Token at ${deployment.contracts.Token.address}...`);
    try {
      await run("verify:verify", {
        address: deployment.contracts.Token.address,
        constructorArguments: [
          "My Token",
          "MTK",
          "1000000000000000000000000000", // 1 billion * 10^18
        ],
      });
      console.log("Token verified successfully!");
    } catch (error: any) {
      if (error.message.includes("Already Verified")) {
        console.log("Token already verified.");
      } else {
        console.error("Token verification failed:", error.message);
      }
    }
  }

  console.log("\nVerification complete!");
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```

### Package.json scripts

```json
{
  "scripts": {
    "compile": "hardhat compile",
    "test": "hardhat test",
    "test:coverage": "hardhat coverage",
    "deploy:local": "hardhat run scripts/deploy.ts --network localhost",
    "deploy:sepolia": "hardhat run scripts/deploy.ts --network sepolia",
    "deploy:mainnet": "hardhat run scripts/deploy.ts --network mainnet",
    "verify:sepolia": "NETWORK=sepolia hardhat run scripts/verify.ts --network sepolia",
    "verify:mainnet": "NETWORK=mainnet hardhat run scripts/verify.ts --network mainnet",
    "node": "hardhat node",
    "clean": "hardhat clean"
  }
}
```

---

## สรุป

### Quick Reference Commands

```bash
# Compile contracts
npx hardhat compile

# Run tests
npx hardhat test

# Run specific test
npx hardhat test test/Token.test.ts

# Deploy locally
npx hardhat run scripts/deploy.ts --network localhost

# Deploy to sepolia
npx hardhat run scripts/deploy.ts --network sepolia

# Verify
npx hardhat verify --network sepolia <CONTRACT_ADDRESS> "Token Name" "SYMBOL" "1000000000000000000000000000"

# Check sizes
npx hardhat size-contracts
```
