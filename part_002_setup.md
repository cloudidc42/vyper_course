# Part 002: ติดตั้งและตั้งค่า Development Environment

## สารบัญ
1. [Requirements](#requirements)
2. [ติดตั้ง Python](#python)
3. [ติดตั้ง Vyper Compiler](#vyper)
4. [ติดตั้ง Node.js](#nodejs)
5. [ตั้งค่า VS Code](#vscode)
6. [ติดตั้ง Titanoboa](#titanoboa)
7. [ติดตั้ง Brownie](#brownie)
8. [ติดตั้ง Hardhat](#hardhat)
9. [Remix IDE](#remix)
10. [ทดสอบการติดตั้ง](#test)
11. [Project Structure](#structure)

---

## 1. Requirements {#requirements}

### Software ที่ต้องการ

| Software | Version | เหตุผล |
|----------|---------|--------|
| Python | 3.10+ | Vyper ต้องการ Python |
| pip | latest | Package Manager |
| Node.js | 18+ | Hardhat, Frontend |
| npm/yarn | latest | JS Package Manager |
| Git | any | Version Control |
| VS Code | latest | IDE แนะนำ |

### Hardware ขั้นต่ำ
- RAM: 4GB (แนะนำ 8GB+)
- Storage: 5GB ว่าง
- OS: Windows 10+, macOS 10.15+, Ubuntu 20.04+

---

## 2. ติดตั้ง Python {#python}

### macOS

```bash
# วิธีที่ 1: Homebrew (แนะนำ)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
brew install python@3.11

# ตรวจสอบ
python3 --version  # Python 3.11.x

# วิธีที่ 2: pyenv (จัดการหลาย Version)
brew install pyenv
pyenv install 3.11.0
pyenv global 3.11.0
```

### Ubuntu/Debian Linux

```bash
# อัปเดต Package List
sudo apt update && sudo apt upgrade -y

# ติดตั้ง Python 3.11
sudo apt install python3.11 python3.11-venv python3.11-dev -y

# ติดตั้ง pip
sudo apt install python3-pip -y

# ตรวจสอบ
python3.11 --version
pip3 --version
```

### Windows

```powershell
# วิธีที่ 1: Official Installer
# ดาวน์โหลดจาก https://python.org/downloads/
# ✅ ติ๊ก "Add Python to PATH"

# วิธีที่ 2: Windows Package Manager (winget)
winget install Python.Python.3.11

# วิธีที่ 3: Microsoft Store
# ค้นหา "Python 3.11" ใน Microsoft Store

# ตรวจสอบ (PowerShell หรือ Command Prompt)
python --version
pip --version
```

### ตั้งค่า Virtual Environment

```bash
# สร้าง Virtual Environment (แนะนำมาก!)
python3 -m venv vyper_env

# Activate
# macOS/Linux:
source vyper_env/bin/activate

# Windows (PowerShell):
vyper_env\Scripts\Activate.ps1

# Windows (Command Prompt):
vyper_env\Scripts\activate.bat

# ตอนนี้ prompt จะแสดง (vyper_env)
(vyper_env) $
```

---

## 3. ติดตั้ง Vyper Compiler {#vyper}

### ติดตั้งด้วย pip

```bash
# ตรวจสอบว่า Virtual Environment ทำงานอยู่
(vyper_env) $ pip install vyper

# ติดตั้งเวอร์ชันเฉพาะ
(vyper_env) $ pip install vyper==0.4.0

# ตรวจสอบ
(vyper_env) $ vyper --version
# Vyper 0.4.0

# อัปเดต
(vyper_env) $ pip install --upgrade vyper
```

### ทดสอบ Compiler ครั้งแรก

```bash
# สร้างไฟล์ทดสอบ
cat > test_contract.vy << 'EOF'
# @version 0.4.0

stored_number: uint256

@deploy
def __init__():
    self.stored_number = 42

@view
@external
def get_number() -> uint256:
    return self.stored_number
EOF

# Compile
vyper test_contract.vy

# ดู Bytecode
vyper -f bytecode test_contract.vy

# ดู ABI
vyper -f abi test_contract.vy

# ดูทั้งหมด
vyper -f combined_json test_contract.vy
```

### Output Formats

```bash
# Bytecode เท่านั้น
vyper -f bytecode contract.vy

# ABI
vyper -f abi contract.vy

# AST (Abstract Syntax Tree)
vyper -f ast contract.vy

# Opcodes
vyper -f opcodes contract.vy

# Source Map
vyper -f source_map contract.vy

# Layout (Storage slots)
vyper -f layout contract.vy

# Combined JSON (ทุกอย่าง)
vyper -f combined_json contract.vy
```

---

## 4. ติดตั้ง Node.js {#nodejs}

### macOS

```bash
# วิธีที่ 1: nvm (Node Version Manager) - แนะนำ
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash

# Reload shell
source ~/.bashrc  # หรือ ~/.zshrc

# ติดตั้ง Node.js LTS
nvm install --lts
nvm use --lts

# วิธีที่ 2: Homebrew
brew install node

# ตรวจสอบ
node --version  # v20.x.x
npm --version   # 10.x.x
```

### Ubuntu/Linux

```bash
# ใช้ nvm (แนะนำ)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
source ~/.bashrc
nvm install 20
nvm use 20

# หรือ NodeSource repository
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs

node --version
npm --version
```

### Windows

```powershell
# nvm-windows
# ดาวน์โหลด: https://github.com/coreybutler/nvm-windows/releases

# หรือ winget
winget install OpenJS.NodeJS.LTS

# ตรวจสอบ
node --version
npm --version
```

---

## 5. ตั้งค่า VS Code {#vscode}

### ติดตั้ง VS Code

```bash
# macOS: Homebrew
brew install --cask visual-studio-code

# Ubuntu
sudo snap install code --classic

# Windows: winget
winget install Microsoft.VisualStudioCode

# หรือดาวน์โหลดจาก https://code.visualstudio.com/
```

### Extensions ที่แนะนำ

```bash
# ติดตั้ง Extensions ผ่าน Command Line
code --install-extension vyperlang.vyper  # Vyper syntax highlighting

# หรือค้นหาใน VS Code Marketplace:
```

**Extensions ที่ต้องมี:**

| Extension | Publisher | ฟังก์ชัน |
|-----------|-----------|---------|
| Vyper | vyperlang | Syntax Highlighting, Snippets |
| Python | Microsoft | Python Support |
| Solidity | NomicFoundation | ถ้าใช้ Solidity ด้วย |
| GitLens | GitKraken | Git Integration |
| Prettier | Prettier | Code Formatting |
| Error Lens | Alexander | แสดง Error inline |

### settings.json สำหรับ Vyper Development

```json
{
    // Python
    "python.defaultInterpreterPath": "./vyper_env/bin/python",
    
    // Editor
    "editor.tabSize": 4,
    "editor.insertSpaces": true,
    "editor.formatOnSave": true,
    "editor.rulers": [88, 120],
    
    // Vyper
    "[vyper]": {
        "editor.defaultFormatter": "vyperlang.vyper",
        "editor.tabSize": 4
    },
    
    // Terminal
    "terminal.integrated.defaultProfile.linux": "bash",
    "terminal.integrated.env.linux": {
        "VIRTUAL_ENV": "${workspaceFolder}/vyper_env"
    },
    
    // Files
    "files.associations": {
        "*.vy": "vyper"
    },
    "files.exclude": {
        "**/__pycache__": true,
        "**/*.pyc": true,
        "**/node_modules": true
    }
}
```

### Keyboard Shortcuts ที่มีประโยชน์

| Shortcut (Mac) | Shortcut (Win) | Action |
|----------------|----------------|--------|
| `Cmd+Shift+P` | `Ctrl+Shift+P` | Command Palette |
| `Cmd+`` ` | `Ctrl+`` ` | Toggle Terminal |
| `Cmd+/` | `Ctrl+/` | Comment/Uncomment |
| `Opt+Shift+F` | `Alt+Shift+F` | Format Document |
| `F2` | `F2` | Rename Symbol |
| `Cmd+Click` | `Ctrl+Click` | Go to Definition |

---

## 6. ติดตั้ง Titanoboa {#titanoboa}

Titanoboa คือ Python Testing Framework สำหรับ Vyper ที่ทันสมัยที่สุด

```bash
# ติดตั้ง
(vyper_env) $ pip install titanoboa

# ติดตั้งพร้อม pytest
(vyper_env) $ pip install titanoboa pytest pytest-cov

# ตรวจสอบ
(vyper_env) $ python -c "import boa; print(boa.__version__)"
```

### ตัวอย่างการใช้งาน Titanoboa

```python
# test_with_titanoboa.py
import boa
import pytest

# โหลด Contract จากไฟล์
@pytest.fixture
def storage_contract():
    return boa.load("contracts/storage.vy")

def test_initial_value(storage_contract):
    assert storage_contract.get_number() == 42

def test_set_value(storage_contract):
    storage_contract.set_number(100)
    assert storage_contract.get_number() == 100

# รัน Test
# pytest test_with_titanoboa.py -v
```

### Titanoboa Interactive Shell

```python
# python REPL
import boa

# Deploy Contract
contract = boa.load("contracts/storage.vy")
print(type(contract))  # <class 'boa.contracts.vyper.vyper_contract.VyperContract'>

# เรียก Functions
contract.get_number()     # 42
contract.set_number(100)
contract.get_number()     # 100

# ดู State
contract._storage  # ดู Storage โดยตรง
```

---

## 7. ติดตั้ง Brownie {#brownie}

Brownie คือ Python-based Framework สำหรับ Ethereum Development

```bash
# ติดตั้ง
(vyper_env) $ pip install eth-brownie

# ตรวจสอบ
(vyper_env) $ brownie --version

# สร้าง Project ใหม่
mkdir my_vyper_project
cd my_vyper_project
brownie init

# โครงสร้าง Brownie Project
my_vyper_project/
├── build/              # Compiled artifacts
│   ├── contracts/
│   └── deployments/
├── contracts/          # Smart Contracts (.vy, .sol)
├── interfaces/         # Contract Interfaces
├── reports/            # Test Reports
├── scripts/            # Deployment Scripts
├── tests/              # Test Files
└── brownie-config.yaml # Configuration
```

### brownie-config.yaml สำหรับ Vyper

```yaml
# brownie-config.yaml
compiler:
  vyper:
    version: "0.4.0"

networks:
  default: development
  development:
    gas_limit: 12000000
    gas_price: 0
    accounts: 10
    mnemonic: "brownie"

pytest:
  gas_limit: 12000000
```

### ตัวอย่าง Deploy Script

```python
# scripts/deploy.py
from brownie import SimpleStorage, accounts

def main():
    deployer = accounts[0]
    
    # Deploy Contract
    contract = SimpleStorage.deploy(
        {'from': deployer}
    )
    
    print(f"Deployed at: {contract.address}")
    
    # เรียก Function
    print(f"Value: {contract.get_number()}")
    
    # Set Value
    tx = contract.set_number(100, {'from': deployer})
    tx.wait(1)  # รอ 1 block confirmation
    
    print(f"New Value: {contract.get_number()}")
```

---

## 8. ติดตั้ง Hardhat {#hardhat}

Hardhat คือ JavaScript-based Framework ที่นิยมใช้กับ Vyper

```bash
# สร้าง Project
mkdir hardhat_vyper_project
cd hardhat_vyper_project

# Initialize npm
npm init -y

# ติดตั้ง Hardhat
npm install --save-dev hardhat

# ติดตั้ง Vyper Plugin
npm install --save-dev @nomiclabs/hardhat-vyper

# Setup Hardhat
npx hardhat init
# เลือก "Create a JavaScript project"
```

### hardhat.config.js สำหรับ Vyper

```javascript
require("@nomiclabs/hardhat-vyper");
require("@nomiclabs/hardhat-ethers");

module.exports = {
  vyper: {
    version: "0.4.0",
    settings: {
      optimize: true,
      runs: 200
    }
  },
  
  networks: {
    hardhat: {
      chainId: 1337,
      gas: 12000000,
      blockGasLimit: 12000000,
    },
    
    localhost: {
      url: "http://127.0.0.1:8545",
    },
    
    sepolia: {
      url: process.env.SEPOLIA_RPC_URL || "",
      accounts: process.env.PRIVATE_KEY ? [process.env.PRIVATE_KEY] : [],
    },
    
    mainnet: {
      url: process.env.MAINNET_RPC_URL || "",
      accounts: process.env.PRIVATE_KEY ? [process.env.PRIVATE_KEY] : [],
    }
  },
  
  etherscan: {
    apiKey: process.env.ETHERSCAN_API_KEY
  }
};
```

### ตัวอย่าง Hardhat Test

```javascript
// test/SimpleStorage.test.js
const { expect } = require("chai");
const { ethers } = require("hardhat");

describe("SimpleStorage", function() {
    let simpleStorage;
    let owner;
    
    beforeEach(async function() {
        [owner] = await ethers.getSigners();
        
        const SimpleStorage = await ethers.getContractFactory("SimpleStorage");
        simpleStorage = await SimpleStorage.deploy();
        await simpleStorage.deployed();
    });
    
    it("Should return initial value of 42", async function() {
        expect(await simpleStorage.get_number()).to.equal(42);
    });
    
    it("Should update the value", async function() {
        const tx = await simpleStorage.set_number(100);
        await tx.wait();
        
        expect(await simpleStorage.get_number()).to.equal(100);
    });
    
    it("Should emit event when value changes", async function() {
        await expect(simpleStorage.set_number(200))
            .to.emit(simpleStorage, "NumberUpdated")
            .withArgs(200);
    });
});
```

---

## 9. Remix IDE {#remix}

Remix เป็น Online IDE ที่ไม่ต้องติดตั้งอะไร เหมาะสำหรับเริ่มต้น

### เข้าใช้งาน

```
URL: https://remix.ethereum.org
```

### ตั้งค่า Remix สำหรับ Vyper

1. เปิด Remix IDE
2. ไปที่ **Plugin Manager** (ไอคอน Puzzle)
3. ค้นหา "**Vyper**"
4. คลิก **Activate**
5. ตอนนี้ Panel ด้านซ้ายจะมีไอคอน Vyper

### การใช้งาน Remix กับ Vyper

```
1. สร้างไฟล์ .vy ใน File Explorer
2. เขียน Code
3. ไปที่ Vyper Compiler tab
4. เลือก Version
5. คลิก Compile
6. ไปที่ Deploy & Run tab
7. เลือก Environment (Remix VM สำหรับทดสอบ)
8. Deploy Contract
9. ทดสอบ Functions
```

### Remix + Local File System

```bash
# ติดตั้ง remixd เพื่อเชื่อมต่อกับ Local Files
npm install -g @remix-project/remixd

# รัน remixd
remixd -s /path/to/project --remix-ide https://remix.ethereum.org

# ใน Remix: Workspaces → connect to localhost
```

---

## 10. ทดสอบการติดตั้ง {#test}

### สร้าง Test Contract

```bash
# สร้าง Directory
mkdir ~/vyper_test
cd ~/vyper_test

# สร้าง Contract
cat > hello_vyper.vy << 'EOF'
# @version 0.4.0
# SPDX-License-Identifier: MIT

# Simple Hello World Contract

owner: address
message: String[100]

event MessageUpdated:
    sender: indexed(address)
    new_message: String[100]

@deploy
def __init__():
    self.owner = msg.sender
    self.message = "Hello, Vyper!"

@view
@external
def get_message() -> String[100]:
    return self.message

@external
def set_message(new_msg: String[100]):
    self.message = new_msg
    log MessageUpdated(msg.sender, new_msg)

@view
@external
def get_owner() -> address:
    return self.owner
EOF
```

### Compile Test

```bash
# Activate virtual environment
source vyper_env/bin/activate

# Compile
vyper hello_vyper.vy

# ถ้าไม่มี Error แสดงว่าการติดตั้งสำเร็จ
# Output ควรเป็น bytecode hex string

# ดู ABI
vyper -f abi hello_vyper.vy
```

### สร้าง Pytest Test

```python
# test_hello.py
import boa
import pytest

@pytest.fixture
def hello_contract():
    return boa.load("hello_vyper.vy")

def test_initial_message(hello_contract):
    assert hello_contract.get_message() == "Hello, Vyper!"

def test_set_message(hello_contract):
    hello_contract.set_message("Hello World!")
    assert hello_contract.get_message() == "Hello World!"

def test_owner(hello_contract):
    # owner ควรเป็น deployer address
    owner = hello_contract.get_owner()
    assert owner != "0x0000000000000000000000000000000000000000"
```

```bash
# รัน Tests
pytest test_hello.py -v

# Output ที่ต้องการ:
# test_hello.py::test_initial_message PASSED
# test_hello.py::test_set_message PASSED  
# test_hello.py::test_owner PASSED
```

### ตรวจสอบ Version ทั้งหมด

```bash
#!/bin/bash
# check_setup.sh

echo "=== Vyper Development Setup Check ==="
echo ""

echo "Python:"
python3 --version

echo ""
echo "pip:"
pip3 --version

echo ""
echo "Vyper:"
vyper --version

echo ""
echo "Node.js:"
node --version

echo ""
echo "npm:"
npm --version

echo ""
echo "Titanoboa:"
python3 -c "import boa; print(f'Titanoboa: {boa.__version__}')"

echo ""
echo "=== All checks complete ==="
```

---

## 11. Project Structure แนะนำ {#structure}

### โครงสร้าง Standard Vyper Project

```
my_vyper_project/
│
├── contracts/                    # Smart Contracts
│   ├── interfaces/               # Interface definitions
│   │   └── IERC20.vy
│   ├── libraries/                # Reusable modules (ถ้ามี)
│   ├── tokens/                   # Token contracts
│   │   └── MyToken.vy
│   ├── defi/                     # DeFi contracts
│   │   └── Staking.vy
│   └── utils/                    # Utility contracts
│
├── tests/                        # Test files
│   ├── conftest.py               # Pytest configuration
│   ├── test_token.py
│   └── test_staking.py
│
├── scripts/                      # Deployment & utility scripts
│   ├── deploy.py
│   └── interact.py
│
├── artifacts/                    # Compiled contracts (auto-generated)
│
├── docs/                         # Documentation
│   ├── architecture.md
│   └── deployment.md
│
├── .env                          # Environment variables (ไม่ commit!)
├── .env.example                  # Template สำหรับ .env
├── .gitignore
├── brownie-config.yaml           # ถ้าใช้ Brownie
├── hardhat.config.js             # ถ้าใช้ Hardhat
├── pyproject.toml                # Python project config
└── README.md
```

### .gitignore สำหรับ Vyper Project

```gitignore
# Python
__pycache__/
*.py[cod]
*$py.class
*.so
.Python
build/
dist/
*.egg-info/
.eggs/

# Virtual Environment
venv/
vyper_env/
env/
.venv/

# IDE
.vscode/settings.json
.idea/
*.swp
*.swo

# Node.js
node_modules/
npm-debug.log*

# Environment Variables (สำคัญมาก!)
.env
.env.local
.env.*.local

# Build artifacts
artifacts/
build/
cache/

# Test coverage
.coverage
htmlcov/
.pytest_cache/

# Brownie
build/
reports/

# Hardhat
cache/
artifacts/

# OS
.DS_Store
Thumbs.db
```

### pyproject.toml

```toml
[tool.pytest.ini_options]
minversion = "6.0"
testpaths = ["tests"]
addopts = "-v --tb=short"

[tool.coverage.run]
source = ["contracts"]
omit = ["tests/*", "scripts/*"]

[tool.coverage.report]
show_missing = true
skip_covered = false

[build-system]
requires = ["setuptools", "wheel"]
build-backend = "setuptools.backends.legacy:build"
```

### requirements.txt

```
# Core
vyper==0.4.0

# Testing
titanoboa>=0.1.0
pytest>=7.0.0
pytest-cov>=4.0.0
hypothesis>=6.0.0

# Ethereum Interaction
web3>=6.0.0
eth-account>=0.9.0

# Development Tools
eth-brownie>=1.19.0  # ถ้าใช้ Brownie

# Optional: Brownie
# eth-brownie>=1.19.0

# Utilities
python-dotenv>=1.0.0
click>=8.0.0
rich>=13.0.0
```

---

## สรุป Part 002

ในส่วนนี้คุณได้ติดตั้ง:
- ✅ Python 3.10+ และ Virtual Environment
- ✅ Vyper Compiler
- ✅ Node.js
- ✅ VS Code พร้อม Extensions
- ✅ Titanoboa Testing Framework
- ✅ Brownie Framework
- ✅ Hardhat Framework
- ✅ Remix IDE

## แบบฝึกหัด

1. **ติดตั้ง** ทุก Tool ตามขั้นตอน
2. **สร้าง** Virtual Environment และ activate
3. **Compile** `hello_vyper.vy` สำเร็จ
4. **รัน** pytest tests ผ่านทั้งหมด
5. **สร้าง** Project Structure ตามที่แนะนำ

## Troubleshooting

**ปัญหา: `vyper: command not found`**
```bash
# ตรวจสอบว่า activate virtual environment แล้ว
source vyper_env/bin/activate
which vyper
```

**ปัญหา: Import Error ใน Python**
```bash
# ตรวจสอบ Python path
python3 -c "import sys; print(sys.path)"
pip list | grep vyper
```

**ปัญหา: Permission Error บน Linux**
```bash
# อย่าใช้ sudo กับ pip ใน virtual environment
# ถ้าต้องการ global install:
sudo pip3 install vyper  # ไม่แนะนำ
```

---

**ก่อนหน้า: [Part 001 - แนะนำ Vyper](part_001_introduction.md)**  
**ต่อไป: [Part 003 - Smart Contract แรกของคุณ](part_003_first_contract.md)**
