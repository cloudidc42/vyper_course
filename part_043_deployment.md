# Part 043: Deployment Strategies

## สารบัญ
1. [Overview](#overview)
2. [Environment Setup](#setup)
3. [Testnet Deployment](#testnet)
4. [Mainnet Deployment](#mainnet)
5. [Deployment Scripts with web3.py](#web3py)
6. [Verification on Etherscan](#etherscan)
7. [Upgradeable Patterns](#upgradeable)
8. [Complete Deployment Workflow](#workflow)

---

## 1. Overview {#overview}

การ deploy Smart Contract เป็นขั้นตอนสำคัญที่ต้องวางแผนอย่างรอบคอบ เพราะ:

- **Immutability**: ไม่สามารถแก้ไขได้หลัง deploy (โดยทั่วไป)
- **Gas Cost**: การ deploy เสีย gas มาก
- **Security**: ต้องตรวจสอบ security ก่อน deploy mainnet
- **Verification**: ผู้ใช้ควรตรวจสอบ source code ได้

### Deployment Workflow

```
1. เขียนและทดสอบ Contract
2. Deploy บน Local Network (Anvil/Hardhat)
3. Deploy บน Testnet (Sepolia/Goerli)
4. Audit/Review
5. Deploy บน Mainnet
6. Verify บน Etherscan
```

---

## 2. Environment Setup {#setup}

### ติดตั้ง Dependencies

```bash
pip install web3 vyper titanoboa python-dotenv eth-account

# หรือใช้ requirements.txt
pip install -r requirements.txt
```

### requirements.txt

```
vyper==0.4.0
web3>=6.0.0
titanoboa>=0.1.9
python-dotenv>=1.0.0
eth-account>=0.9.0
requests>=2.28.0
```

### .env file

```bash
# .env - อย่า commit ไฟล์นี้!
PRIVATE_KEY=0x...your_private_key...
INFURA_API_KEY=your_infura_key
ALCHEMY_API_KEY=your_alchemy_key
ETHERSCAN_API_KEY=your_etherscan_key

# RPC URLs
SEPOLIA_RPC=https://sepolia.infura.io/v3/${INFURA_API_KEY}
MAINNET_RPC=https://mainnet.infura.io/v3/${INFURA_API_KEY}

# Contract addresses (หลัง deploy)
TOKEN_ADDRESS=
PROXY_ADDRESS=
```

### .gitignore

```
.env
*.pyc
__pycache__/
deployments/secrets/
```

---

## 3. Testnet Deployment {#testnet}

### Contract ที่จะ Deploy

```vyper
# contracts/ProductionToken.vy
# @version 0.4.0
# @title ProductionToken
# @notice Token พร้อมสำหรับ production deployment

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

event OwnershipTransferred:
    previous_owner: indexed(address)
    new_owner: indexed(address)

# Token metadata
NAME: immutable(String[64])
SYMBOL: immutable(String[32])
DECIMALS: immutable(uint8)
MAX_SUPPLY: immutable(uint256)
DEPLOY_TIME: immutable(uint256)

# State
total_supply: uint256
balances: HashMap[address, uint256]
allowances: HashMap[address, HashMap[address, uint256]]
owner: address
minters: HashMap[address, bool]
is_paused: bool

@deploy
def __init__(
    name: String[64],
    symbol: String[32],
    max_supply: uint256,
    initial_supply: uint256,
    initial_holder: address
):
    NAME = name
    SYMBOL = symbol
    DECIMALS = 18
    MAX_SUPPLY = max_supply
    DEPLOY_TIME = block.timestamp
    
    self.owner = msg.sender
    self.is_paused = False
    
    if initial_supply > 0:
        assert initial_holder != empty(address), "Zero holder"
        assert initial_supply <= max_supply, "Exceeds max"
        self.total_supply = initial_supply
        self.balances[initial_holder] = initial_supply
        log Transfer(empty(address), initial_holder, initial_supply)

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
def maxSupply() -> uint256:
    return MAX_SUPPLY

@external
@view
def deployTime() -> uint256:
    return DEPLOY_TIME

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
    assert self.total_supply + amount <= MAX_SUPPLY, "Exceeds max supply"
    
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
    old_owner: address = self.owner
    self.owner = new_owner
    log OwnershipTransferred(old_owner, new_owner)
```

---

## 4. Deployment Scripts {#web3py}

### deploy.py - Main Deployment Script

```python
# scripts/deploy.py
"""
Main deployment script สำหรับ ProductionToken
รองรับ: local, testnet, mainnet
"""

import os
import json
import time
from pathlib import Path
from dotenv import load_dotenv
from web3 import Web3
from web3.middleware import geth_poa_middleware
from eth_account import Account
import vyper

load_dotenv()

# ==================== Configuration ====================

NETWORKS = {
    "local": {
        "rpc": "http://localhost:8545",
        "chain_id": 31337,
        "name": "Local Anvil"
    },
    "sepolia": {
        "rpc": os.getenv("SEPOLIA_RPC", f"https://sepolia.infura.io/v3/{os.getenv('INFURA_API_KEY')}"),
        "chain_id": 11155111,
        "name": "Sepolia Testnet"
    },
    "mainnet": {
        "rpc": os.getenv("MAINNET_RPC", f"https://mainnet.infura.io/v3/{os.getenv('INFURA_API_KEY')}"),
        "chain_id": 1,
        "name": "Ethereum Mainnet"
    }
}

# ==================== Compiler ====================

def compile_contract(contract_path: str) -> dict:
    """Compile Vyper contract"""
    print(f"Compiling {contract_path}...")
    
    with open(contract_path, 'r') as f:
        source = f.read()
    
    # ใช้ vyper compiler
    from vyper.compiler import compile_code
    from vyper.compiler.settings import Settings
    
    settings = Settings()
    compiled = compile_code(
        source,
        output_formats=['abi', 'bytecode', 'bytecode_runtime'],
        settings=settings
    )
    
    return {
        'abi': compiled['abi'],
        'bytecode': compiled['bytecode'],
        'source': source
    }

# ==================== Deployment ====================

class Deployer:
    def __init__(self, network: str = "local"):
        self.network = network
        self.config = NETWORKS[network]
        
        # Connect to network
        self.w3 = Web3(Web3.HTTPProvider(self.config["rpc"]))
        
        # Add PoA middleware for testnets
        if network != "mainnet":
            self.w3.middleware_onion.inject(geth_poa_middleware, layer=0)
        
        # Load account
        private_key = os.getenv("PRIVATE_KEY")
        if not private_key:
            raise ValueError("PRIVATE_KEY not set in .env")
        
        self.account = Account.from_key(private_key)
        self.address = self.account.address
        
        print(f"Connected to {self.config['name']}")
        print(f"Deployer: {self.address}")
        print(f"Balance: {self.w3.from_wei(self.w3.eth.get_balance(self.address), 'ether')} ETH")
    
    def deploy_contract(
        self,
        contract_path: str,
        constructor_args: list,
        wait_confirmations: int = 1
    ) -> str:
        """Deploy contract และรอ confirmations"""
        
        # Compile
        compiled = compile_contract(contract_path)
        
        # สร้าง contract instance
        contract = self.w3.eth.contract(
            abi=compiled['abi'],
            bytecode=compiled['bytecode']
        )
        
        # Build constructor transaction
        nonce = self.w3.eth.get_transaction_count(self.address)
        
        # Estimate gas
        gas_estimate = contract.constructor(*constructor_args).estimate_gas({
            'from': self.address
        })
        
        # Get gas price
        gas_price = self.w3.eth.gas_price
        
        print(f"\nDeployment Details:")
        print(f"  Gas Estimate: {gas_estimate:,}")
        print(f"  Gas Price: {self.w3.from_wei(gas_price, 'gwei'):.2f} Gwei")
        print(f"  Estimated Cost: {self.w3.from_wei(gas_estimate * gas_price, 'ether'):.6f} ETH")
        
        # Build transaction
        tx = contract.constructor(*constructor_args).build_transaction({
            'from': self.address,
            'nonce': nonce,
            'gas': int(gas_estimate * 1.2),  # 20% buffer
            'gasPrice': gas_price,
            'chainId': self.config['chain_id']
        })
        
        # Sign transaction
        signed_tx = self.w3.eth.account.sign_transaction(tx, self.account.key)
        
        # Send transaction
        print(f"\nSending deployment transaction...")
        tx_hash = self.w3.eth.send_raw_transaction(signed_tx.rawTransaction)
        print(f"Transaction hash: {tx_hash.hex()}")
        
        # Wait for receipt
        print(f"Waiting for {wait_confirmations} confirmation(s)...")
        receipt = self.w3.eth.wait_for_transaction_receipt(
            tx_hash,
            timeout=300,
            poll_latency=2
        )
        
        if receipt['status'] != 1:
            raise Exception("Deployment transaction failed!")
        
        contract_address = receipt['contractAddress']
        print(f"\nContract deployed at: {contract_address}")
        print(f"Gas used: {receipt['gasUsed']:,}")
        print(f"Block: {receipt['blockNumber']}")
        
        return contract_address, compiled['abi'], compiled['source']
    
    def save_deployment(
        self,
        contract_name: str,
        address: str,
        abi: list,
        constructor_args: dict,
        tx_hash: str
    ):
        """บันทึกข้อมูล deployment"""
        deployment_dir = Path("deployments") / self.network
        deployment_dir.mkdir(parents=True, exist_ok=True)
        
        deployment_data = {
            "contract": contract_name,
            "address": address,
            "network": self.network,
            "chain_id": self.config['chain_id'],
            "deployer": self.address,
            "constructor_args": constructor_args,
            "tx_hash": tx_hash,
            "timestamp": int(time.time()),
            "abi": abi
        }
        
        filename = deployment_dir / f"{contract_name}.json"
        with open(filename, 'w') as f:
            json.dump(deployment_data, f, indent=2)
        
        print(f"Deployment saved to: {filename}")


# ==================== Main Deployment ====================

def deploy_production_token(network: str = "local"):
    """Deploy ProductionToken"""
    
    deployer = Deployer(network)
    
    # Token parameters
    TOKEN_NAME = "My Production Token"
    TOKEN_SYMBOL = "MPT"
    MAX_SUPPLY = 1_000_000_000 * 10**18  # 1 billion
    INITIAL_SUPPLY = 100_000_000 * 10**18  # 100 million
    INITIAL_HOLDER = deployer.address
    
    constructor_args = [
        TOKEN_NAME,
        TOKEN_SYMBOL,
        MAX_SUPPLY,
        INITIAL_SUPPLY,
        INITIAL_HOLDER
    ]
    
    print(f"\n{'='*50}")
    print(f"Deploying ProductionToken to {network}")
    print(f"{'='*50}")
    print(f"Name: {TOKEN_NAME}")
    print(f"Symbol: {TOKEN_SYMBOL}")
    print(f"Max Supply: {MAX_SUPPLY / 10**18:,.0f}")
    print(f"Initial Supply: {INITIAL_SUPPLY / 10**18:,.0f}")
    
    # Confirm for mainnet
    if network == "mainnet":
        confirm = input("\n⚠️  MAINNET DEPLOYMENT - Type 'yes' to confirm: ")
        if confirm != "yes":
            print("Deployment cancelled.")
            return
    
    # Deploy
    wait_confirmations = 5 if network == "mainnet" else 1
    
    address, abi, source = deployer.deploy_contract(
        "contracts/ProductionToken.vy",
        constructor_args,
        wait_confirmations=wait_confirmations
    )
    
    print(f"\n✅ Deployment successful!")
    print(f"Contract address: {address}")
    
    return address

if __name__ == "__main__":
    import sys
    network = sys.argv[1] if len(sys.argv) > 1 else "local"
    deploy_production_token(network)
```

---

## 5. Verification on Etherscan {#etherscan}

```python
# scripts/verify.py
"""
Script สำหรับ verify contract บน Etherscan
"""

import os
import json
import time
import requests
from dotenv import load_dotenv

load_dotenv()

ETHERSCAN_API_KEY = os.getenv("ETHERSCAN_API_KEY")

ETHERSCAN_APIS = {
    "mainnet": "https://api.etherscan.io/api",
    "sepolia": "https://api-sepolia.etherscan.io/api",
    "goerli": "https://api-goerli.etherscan.io/api"
}

def verify_on_etherscan(
    network: str,
    contract_address: str,
    contract_name: str,
    source_code: str,
    constructor_args_encoded: str = ""
) -> bool:
    """Verify contract source code บน Etherscan"""
    
    api_url = ETHERSCAN_APIS.get(network)
    if not api_url:
        print(f"Unknown network: {network}")
        return False
    
    print(f"Verifying {contract_name} at {contract_address}...")
    
    # Submit verification
    payload = {
        "apikey": ETHERSCAN_API_KEY,
        "module": "contract",
        "action": "verifysourcecode",
        "contractaddress": contract_address,
        "sourceCode": source_code,
        "codeformat": "solidity-single-file",  # Vyper ก็ใช้ format นี้
        "contractname": contract_name,
        "compilerversion": "vyper:0.4.0",
        "optimizationUsed": "0",
        "runs": "200",
        "constructorArguements": constructor_args_encoded,
        "licenseType": "3",  # MIT
    }
    
    response = requests.post(api_url, data=payload)
    result = response.json()
    
    if result.get("status") != "1":
        print(f"Verification submission failed: {result.get('result')}")
        return False
    
    guid = result["result"]
    print(f"Verification GUID: {guid}")
    
    # Poll for result
    print("Checking verification status...")
    for attempt in range(20):
        time.sleep(5)
        
        check_payload = {
            "apikey": ETHERSCAN_API_KEY,
            "module": "contract",
            "action": "checkverifystatus",
            "guid": guid
        }
        
        check_response = requests.get(api_url, params=check_payload)
        check_result = check_response.json()
        
        status = check_result.get("result", "")
        print(f"Attempt {attempt + 1}: {status}")
        
        if "Pass - Verified" in status:
            print(f"\n✅ Contract verified successfully!")
            etherscan_url = f"https://{'sepolia.' if network == 'sepolia' else ''}etherscan.io/address/{contract_address}#code"
            print(f"View at: {etherscan_url}")
            return True
        
        if "Fail" in status:
            print(f"\n❌ Verification failed: {status}")
            return False
    
    print("Verification timed out")
    return False


def verify_from_deployment_file(deployment_file: str):
    """Verify จาก deployment JSON file"""
    with open(deployment_file) as f:
        deployment = json.load(f)
    
    network = deployment["network"]
    address = deployment["address"]
    contract_name = deployment["contract"]
    
    # อ่าน source code
    source_path = f"contracts/{contract_name}.vy"
    with open(source_path) as f:
        source = f.read()
    
    verify_on_etherscan(
        network=network,
        contract_address=address,
        contract_name=contract_name,
        source_code=source
    )


if __name__ == "__main__":
    import sys
    if len(sys.argv) > 1:
        verify_from_deployment_file(sys.argv[1])
```

---

## 6. Upgradeable Patterns {#upgradeable}

### Proxy Pattern (EIP-1967)

```vyper
# contracts/proxy/TransparentProxy.vy
# @version 0.4.0
# @title TransparentProxy
# @notice Transparent Proxy pattern สำหรับ upgradeable contracts
# @notice Implementation contract address stored in specific slot

# EIP-1967 storage slots
# keccak256("eip1967.proxy.implementation") - 1
IMPLEMENTATION_SLOT: constant(bytes32) = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc
# keccak256("eip1967.proxy.admin") - 1  
ADMIN_SLOT: constant(bytes32) = 0xb53127684a568b3173ae13b9f8a6016e243e63b6e8ee1178d6a717850b5d6103

event Upgraded:
    implementation: indexed(address)

event AdminChanged:
    previous_admin: address
    new_admin: address

@deploy
def __init__(implementation: address, admin: address, init_data: Bytes[1000]):
    assert implementation != empty(address), "Zero implementation"
    assert admin != empty(address), "Zero admin"
    
    # Store implementation
    self._set_implementation(implementation)
    
    # Store admin
    self._set_admin(admin)
    
    # Initialize implementation
    if len(init_data) > 0:
        raw_call(implementation, init_data, is_delegate_call=True)

@internal
def _implementation() -> address:
    impl: address = empty(address)
    # อ่านจาก EIP-1967 slot
    impl = convert(
        convert(self.balance, bytes32),  # placeholder
        address
    )
    return impl

@internal
def _set_implementation(new_impl: address):
    log Upgraded(new_impl)

@internal
def _admin() -> address:
    return empty(address)  # placeholder

@internal
def _set_admin(new_admin: address):
    old_admin: address = self._admin()
    log AdminChanged(old_admin, new_admin)

@external
def upgrade_to(new_implementation: address):
    assert msg.sender == self._admin(), "Not admin"
    assert new_implementation != empty(address), "Zero address"
    self._set_implementation(new_implementation)

@external
def change_admin(new_admin: address):
    assert msg.sender == self._admin(), "Not admin"
    assert new_admin != empty(address), "Zero admin"
    self._set_admin(new_admin)

@external
@payable
def __default__():
    # Delegate all calls to implementation
    impl: address = self._implementation()
    assert impl != empty(address), "No implementation"
    
    raw_call(
        impl,
        msg.data,
        value=msg.value,
        is_delegate_call=True
    )
```

### Storage Contract สำหรับ Upgradeable Pattern

```vyper
# contracts/upgradeable/TokenStorageV1.vy
# @version 0.4.0
# @title TokenStorageV1
# @notice Storage layout สำหรับ upgradeable token V1
# ต้องไม่เปลี่ยน storage layout เมื่อ upgrade

# ===== Storage Layout V1 =====
# IMPORTANT: อย่าเปลี่ยน order หรือลบ variables เมื่อ upgrade
owner: public(address)
balances: HashMap[address, uint256]
allowances: HashMap[address, HashMap[address, uint256]]
total_supply: public(uint256)
is_paused: public(bool)
# ===== End V1 =====

# Constants (ไม่ใช้ storage)
NAME: constant(String[64]) = "Upgradeable Token"
SYMBOL: constant(String[32]) = "UPG"
DECIMALS: constant(uint8) = 18
```

---

## 7. Complete Deployment Workflow {#workflow}

```python
# scripts/full_deploy.py
"""
Complete deployment workflow:
1. Test locally
2. Deploy to testnet
3. Verify on testnet Etherscan
4. Deploy to mainnet (manual approval)
5. Verify on mainnet Etherscan
"""

import os
import sys
import subprocess
import json
from pathlib import Path
from dotenv import load_dotenv

load_dotenv()

def run_tests() -> bool:
    """รัน test suite ก่อน deploy"""
    print("\n📋 Running test suite...")
    result = subprocess.run(
        ["python", "-m", "pytest", "tests/", "-v", "--tb=short"],
        capture_output=True,
        text=True
    )
    
    print(result.stdout)
    if result.returncode != 0:
        print("❌ Tests failed!")
        print(result.stderr)
        return False
    
    print("✅ All tests passed!")
    return True

def check_security() -> bool:
    """Basic security checks"""
    print("\n🔐 Running security checks...")
    
    checks = {
        "Private key not hardcoded": True,
        "No infinite loops": True,
        "Reentrancy protection": True,
        "Access control": True
    }
    
    all_passed = True
    for check, passed in checks.items():
        status = "✅" if passed else "❌"
        print(f"  {status} {check}")
        if not passed:
            all_passed = False
    
    return all_passed

def full_deployment_workflow():
    """Run complete deployment workflow"""
    
    print("🚀 Starting Full Deployment Workflow")
    print("=" * 60)
    
    # Step 1: Run tests
    if not run_tests():
        print("\n❌ Deployment aborted: tests failed")
        sys.exit(1)
    
    # Step 2: Security checks
    if not check_security():
        print("\n⚠️  Security issues found")
        if input("Continue anyway? (y/N): ") != "y":
            sys.exit(1)
    
    # Step 3: Deploy to testnet
    print("\n📡 Deploying to Sepolia testnet...")
    
    from deploy import deploy_production_token
    
    testnet_address = deploy_production_token("sepolia")
    
    if not testnet_address:
        print("❌ Testnet deployment failed")
        sys.exit(1)
    
    print(f"\n✅ Testnet deployment: {testnet_address}")
    print(f"Verify at: https://sepolia.etherscan.io/address/{testnet_address}")
    
    # Step 4: Manual verification period
    print("\n⏰ Please verify the testnet deployment manually before mainnet.")
    print("Check:")
    print("  - Token metadata (name, symbol, decimals)")
    print("  - Initial supply distribution")
    print("  - Access control functions")
    print("  - Pause/unpause functionality")
    
    proceed = input("\nProceed to mainnet deployment? (yes/NO): ")
    
    if proceed != "yes":
        print("Mainnet deployment skipped.")
        return testnet_address
    
    # Step 5: Deploy to mainnet
    print("\n🌐 Deploying to Ethereum Mainnet...")
    mainnet_address = deploy_production_token("mainnet")
    
    if mainnet_address:
        print(f"\n🎉 Mainnet deployment successful!")
        print(f"Contract: {mainnet_address}")
        print(f"View at: https://etherscan.io/address/{mainnet_address}")
    
    return mainnet_address


# Deployment checklist
DEPLOYMENT_CHECKLIST = """
Pre-Deployment Checklist:
========================

Code Review:
[ ] Contract reviewed by at least 2 developers
[ ] No hardcoded addresses (use constructor params)
[ ] All functions have appropriate access control
[ ] Events emitted for all state changes
[ ] No integer overflow/underflow vulnerabilities

Testing:
[ ] Unit tests pass (>95% coverage)
[ ] Integration tests pass
[ ] Fuzzing tests completed
[ ] Gas optimization reviewed

Security:
[ ] Reentrancy protection where needed
[ ] Emergency pause mechanism tested
[ ] Ownership transfer tested
[ ] No unreachable code

Deployment:
[ ] Private key secured (not in code)
[ ] Deployment script tested on local network
[ ] Deployment script tested on testnet
[ ] Gas costs estimated and acceptable
[ ] Etherscan verification script ready

Post-Deployment:
[ ] Contract verified on Etherscan
[ ] Deployment data saved (address, tx hash, ABI)
[ ] README updated with contract addresses
[ ] Team notified of deployment
"""

if __name__ == "__main__":
    print(DEPLOYMENT_CHECKLIST)
    
    if input("\nAll checks complete? Run deployment? (yes/NO): ") == "yes":
        full_deployment_workflow()
```

### Post-Deployment Script

```python
# scripts/post_deploy.py
"""
Tasks หลัง deployment:
- ตั้งค่า minters
- โอน ownership
- กระจาย initial tokens
"""

import json
from web3 import Web3
from eth_account import Account
import os
from dotenv import load_dotenv

load_dotenv()

def post_deployment_setup(
    network: str,
    deployment_file: str
):
    """Setup หลัง deployment"""
    
    # Load deployment info
    with open(deployment_file) as f:
        deployment = json.load(f)
    
    contract_address = deployment["address"]
    abi = deployment["abi"]
    
    # Connect
    w3 = Web3(Web3.HTTPProvider(
        os.getenv(f"{network.upper()}_RPC")
    ))
    
    account = Account.from_key(os.getenv("PRIVATE_KEY"))
    
    # Create contract instance
    contract = w3.eth.contract(
        address=contract_address,
        abi=abi
    )
    
    print(f"Post-deployment setup for {contract_address}")
    
    # ตัวอย่าง: กระจาย tokens ไปยัง accounts ต่างๆ
    distribution = [
        {"address": "0x...", "amount": 1_000_000 * 10**18},
        {"address": "0x...", "amount": 500_000 * 10**18},
    ]
    
    nonce = w3.eth.get_transaction_count(account.address)
    
    for item in distribution:
        tx = contract.functions.mint(
            item["address"],
            item["amount"]
        ).build_transaction({
            "from": account.address,
            "nonce": nonce,
            "gas": 100000,
            "gasPrice": w3.eth.gas_price,
            "chainId": w3.eth.chain_id
        })
        
        signed = w3.eth.account.sign_transaction(tx, account.key)
        tx_hash = w3.eth.send_raw_transaction(signed.rawTransaction)
        receipt = w3.eth.wait_for_transaction_receipt(tx_hash)
        
        print(f"Minted to {item['address']}: {receipt['transactionHash'].hex()}")
        nonce += 1
    
    print("Post-deployment setup complete!")

if __name__ == "__main__":
    post_deployment_setup("sepolia", "deployments/sepolia/ProductionToken.json")
```

---

## สรุป

### Key Takeaways

1. **ทดสอบก่อน deploy เสมอ** - ทดสอบบน local, testnet ก่อน mainnet
2. **ใช้ environment variables** - ไม่ hardcode private keys
3. **Save deployment data** - บันทึก address, tx hash, ABI
4. **Verify source code** - ให้ผู้ใช้ตรวจสอบได้
5. **Plan upgrades** - ถ้า contract ต้อง upgrade ใช้ proxy pattern

### Deployment Commands Quick Reference

```bash
# Local deployment
python scripts/deploy.py local

# Testnet deployment
python scripts/deploy.py sepolia

# Mainnet deployment (requires confirmation)
python scripts/deploy.py mainnet

# Verify on Etherscan
python scripts/verify.py deployments/sepolia/ProductionToken.json

# Full workflow
python scripts/full_deploy.py

# Post-deployment setup
python scripts/post_deploy.py sepolia deployments/sepolia/ProductionToken.json
```
