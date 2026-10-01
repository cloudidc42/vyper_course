# Part 090: Protocol Launch Playbook

## สารบัญ
1. Pre-launch Checklist
2. Testnet → Mainnet Process
3. Bug Bounty Setup
4. Community Building
5. Gradual Launch Strategy
6. Post-launch Monitoring
7. Emergency Procedures

---

## 1. Pre-launch Checklist

```markdown
## Pre-launch Checklist (90 days before)

### Code Quality
□ All unit tests passing (>95% coverage)
□ Integration tests passing
□ Invariant/fuzzing tests passing
□ Formal verification (where applicable)
□ Code freeze 2 weeks before launch

### Security
□ Internal security review complete
□ External audit (minimum 1, ideally 2)
□ All Critical/High findings fixed
□ Audit report published
□ Bug bounty program active

### Infrastructure
□ Deployment scripts tested on testnet
□ Multi-sig signers confirmed
□ Timelock configured
□ Monitoring alerts set up
□ Incident response plan ready

### Legal/Compliance
□ Legal review (jurisdiction-specific)
□ Terms of service ready
□ Privacy policy ready
□ Token documentation

### Community
□ Documentation site live
□ Discord/Telegram active
□ Twitter/social active
□ Blog post drafts ready
□ FAQ document ready
```

---

## 2. Deployment Script

```python
# deploy_protocol.py
# Script สำหรับ deploy protocol ไปยัง mainnet

from web3 import Web3
from eth_account import Account
import json
import os
from pathlib import Path
from dataclasses import dataclass
from typing import Optional

@dataclass
class DeploymentConfig:
    # Network
    rpc_url: str
    chain_id: int
    
    # Deployer
    deployer_key: str  # จาก env var
    
    # Multi-sig
    multisig_address: str
    
    # Timelock
    timelock_delay: int  # seconds (e.g., 86400 = 1 day)
    
    # Parameters
    initial_fee_rate: int  # basis points
    initial_treasury: str

class ProtocolDeployer:
    def __init__(self, config: DeploymentConfig):
        self.config = config
        self.w3 = Web3(Web3.HTTPProvider(config.rpc_url))
        self.deployer = Account.from_key(config.deployer_key)
        self.deployments = {}

    def deploy_all(self):
        """Deploy ทุก contracts ตามลำดับ"""
        print("=== Starting Protocol Deployment ===")
        print(f"Network: Chain ID {self.config.chain_id}")
        print(f"Deployer: {self.deployer.address}")
        
        # 1. Deploy Timelock
        print("\n[1/6] Deploying Timelock...")
        timelock = self._deploy_timelock()
        
        # 2. Deploy Token
        print("\n[2/6] Deploying Governance Token...")
        token = self._deploy_token()
        
        # 3. Deploy Core Protocol
        print("\n[3/6] Deploying Core Protocol...")
        core = self._deploy_core(token)
        
        # 4. Deploy Treasury
        print("\n[4/6] Deploying Treasury...")
        treasury = self._deploy_treasury(core)
        
        # 5. Deploy Governance
        print("\n[5/6] Deploying Governance...")
        governance = self._deploy_governance(token, timelock)
        
        # 6. Initialize & Configure
        print("\n[6/6] Configuring Protocol...")
        self._configure_protocol(core, treasury, governance, timelock)
        
        # Save deployment info
        self._save_deployment()
        
        print("\n=== Deployment Complete ===")
        return self.deployments

    def _deploy_timelock(self):
        """Deploy timelock controller"""
        # Minimum delay: 24 hours for mainnet
        min_delay = max(self.config.timelock_delay, 86400)
        
        # เฉพาะ multi-sig สามารถ propose/execute ได้
        proposers = [self.config.multisig_address]
        executors = [self.config.multisig_address]
        
        tx_hash = self._send_deploy_tx(
            "TimelockController",
            min_delay,
            proposers,
            executors,
            self.deployer.address
        )
        
        address = self._get_deployed_address(tx_hash)
        self.deployments['timelock'] = address
        print(f"  Timelock deployed: {address}")
        return address

    def _deploy_token(self):
        """Deploy governance token"""
        tx_hash = self._send_deploy_tx(
            "GovernanceToken",
            "Protocol Token",
            "PROTO",
            self.config.multisig_address,  # initial mint to multisig
            10**6 * 10**18  # 1M tokens
        )
        
        address = self._get_deployed_address(tx_hash)
        self.deployments['token'] = address
        print(f"  Token deployed: {address}")
        return address

    def _send_deploy_tx(self, contract_name: str, *args):
        """Helper to send deployment transaction"""
        # Simplified - ในจริงใช้ compiled bytecode
        nonce = self.w3.eth.get_transaction_count(self.deployer.address)
        
        tx = {
            'chainId': self.config.chain_id,
            'nonce': nonce,
            'gasPrice': self.w3.eth.gas_price,
            'gas': 3000000,
        }
        
        signed = self.deployer.sign_transaction(tx)
        tx_hash = self.w3.eth.send_raw_transaction(signed.rawTransaction)
        
        receipt = self.w3.eth.wait_for_transaction_receipt(tx_hash)
        return tx_hash.hex()

    def _get_deployed_address(self, tx_hash: str) -> str:
        receipt = self.w3.eth.get_transaction_receipt(tx_hash)
        return receipt['contractAddress']

    def _configure_protocol(self, core, treasury, governance, timelock):
        """Configure protocol after deployment"""
        print("  - Setting treasury address...")
        print("  - Transferring ownership to timelock...")
        print("  - Setting up guardian...")
        
        # Transfer ownership to timelock
        # This means all admin actions require governance vote
        print(f"  Ownership transferred to timelock: {timelock}")

    def _save_deployment(self):
        """บันทึก deployment addresses"""
        deployment_info = {
            "chainId": self.config.chain_id,
            "timestamp": str(os.popen("date -u").read().strip()),
            "deployer": self.deployer.address,
            "contracts": self.deployments
        }
        
        filename = f"deployments/mainnet_{self.config.chain_id}.json"
        Path("deployments").mkdir(exist_ok=True)
        
        with open(filename, 'w') as f:
            json.dump(deployment_info, f, indent=2)
        
        print(f"\nDeployment saved to: {filename}")
```

---

## 3. Bug Bounty Setup

```markdown
## Bug Bounty Program Setup

### Immunefi Program Template

**Protocol Name:** My DeFi Protocol
**Live Since:** [Date]
**Total Value Locked:** $X

### Assets in Scope
Primary:
- Core.vy: https://etherscan.io/address/0x...
- Token.vy: https://etherscan.io/address/0x...

Out of Scope:
- Governance token price manipulation
- Frontrunning attacks (expected behavior)
- Off-chain components

### Rewards

| Severity | Smart Contract | Website |
|----------|---------------|---------|
| Critical | Up to $100,000 | Up to $5,000 |
| High     | Up to $20,000  | Up to $2,000 |
| Medium   | Up to $5,000   | Up to $1,000 |
| Low      | Up to $1,000   | Up to $500  |

### Critical Severity Definition
- Direct theft of user funds
- Permanent protocol DoS
- Minting unauthorized tokens
- Bypassing governance

### Submission Process
1. Email: security@protocol.xyz
2. Include: Description, PoC, Impact
3. Response within: 48 hours
4. Fix timeline: 30 days for critical

### Safe Harbor
White-hat researchers who follow responsible disclosure
will not be prosecuted.
```

---

## 4. Gradual Launch Strategy (Progressive Decentralization)

```vyper
# @version 0.4.0
# @title Launch Controller
# @notice จัดการ gradual launch ด้วย caps และ limits

event CapUpdated:
    old_cap: uint256
    new_cap: uint256
    timestamp: uint256

event ProtocolEnabled:
    feature: String[64]
    timestamp: uint256

owner: public(address)
multisig: public(address)

# Launch parameters
tvl_cap: public(uint256)
per_user_cap: public(uint256)
whitelist_only: public(bool)
paused: public(bool)

# Features enabled
features: public(HashMap[String[64], bool])

# TVL tracking
total_tvl: public(uint256)
user_deposits: public(HashMap[address, uint256])

# Whitelist
whitelist: public(HashMap[address, bool])

@deploy
def __init__(_multisig: address):
    self.owner = msg.sender
    self.multisig = _multisig
    
    # Phase 1: Very conservative limits
    self.tvl_cap = 100_000 * 10**18      # $100k initial cap
    self.per_user_cap = 10_000 * 10**18  # $10k per user
    self.whitelist_only = True            # Whitelist only initially
    self.paused = False

@external
def deposit(amount: uint256):
    """Deposit with launch controls"""
    assert not self.paused, "Protocol paused"
    
    if self.whitelist_only:
        assert self.whitelist[msg.sender], "Not whitelisted"
    
    assert self.user_deposits[msg.sender] + amount <= self.per_user_cap, "Per-user cap exceeded"
    assert self.total_tvl + amount <= self.tvl_cap, "TVL cap exceeded"
    
    self.user_deposits[msg.sender] += amount
    self.total_tvl += amount

@external
def increaseCap(new_cap: uint256):
    """Gradually increase TVL cap"""
    assert msg.sender == self.multisig, "Only multisig"
    
    # Safety: max 2x increase at a time
    assert new_cap <= self.tvl_cap * 2, "Increase too large"
    
    old_cap: uint256 = self.tvl_cap
    self.tvl_cap = new_cap
    
    log CapUpdated(old_cap, new_cap, block.timestamp)

@external
def disableWhitelist():
    """Phase 2: Open to public"""
    assert msg.sender == self.multisig, "Only multisig"
    self.whitelist_only = False
    log ProtocolEnabled("public_access", block.timestamp)

@external
def enableFeature(feature: String[64]):
    """เปิด features ทีละอย่างตาม schedule"""
    assert msg.sender == self.multisig, "Only multisig"
    self.features[feature] = True
    log ProtocolEnabled(feature, block.timestamp)
```

---

## 5. Launch Timeline

```markdown
## Launch Timeline

### T-90 days: Final Audit Starts
- Code freeze
- Submit to auditors
- Start internal testing

### T-60 days: Audit Complete
- Fix all Critical/High findings
- Re-audit critical fixes
- Publish audit report

### T-30 days: Testnet Launch
- Deploy to Goerli/Sepolia
- Community testing
- Bug bounty on testnet (smaller rewards)
- Final integration testing

### T-14 days: Bug Bounty Live
- Immunefi listing active
- Announce on social media
- $50k initial bounty pool

### T-7 days: Mainnet Prep
- Final checklist review
- Multi-sig signers briefed
- Incident response team on standby
- Monitoring alerts configured

### T-0: LAUNCH
- Deploy contracts
- Verify on Etherscan
- Publish deploy script + addresses
- Enable whitelist deposits
- Monitor closely for 24 hours

### T+7 days: Phase 2
- Review launch metrics
- Increase TVL cap if safe
- Open whitelist to more users

### T+30 days: Public Launch
- Remove whitelist restriction
- Increase TVL cap significantly
- Marketing push

### T+90 days: Full Decentralization
- Transfer ownership to governance
- Remove emergency powers (if safe)
- Governance vote for major changes only
```

---

## 6. Post-launch Monitoring

```python
# post_launch_monitor.py
# Monitoring script สำหรับหลัง launch

import asyncio
import aiohttp
from web3 import Web3
from datetime import datetime

class LaunchMonitor:
    def __init__(self, rpc_url: str, contract_address: str):
        self.w3 = Web3(Web3.HTTPProvider(rpc_url))
        self.contract_address = contract_address
        self.alerts = []
        
        # Alert thresholds
        self.MAX_SINGLE_DEPOSIT = 50_000  # $50k
        self.MAX_HOURLY_WITHDRAWALS = 200_000  # $200k
        self.MIN_TVL_DROP_PERCENT = 10  # alert if TVL drops 10% in 1 hour
        
        self.last_tvl = 0
        self.hourly_withdrawals = 0
        self.last_hour_reset = datetime.now()

    async def monitor_forever(self):
        """Monitor lifeforce"""
        print("=== Launch Monitor Started ===")
        
        while True:
            try:
                await self._check_tvl()
                await self._check_unusual_activity()
                await self._check_price_impact()
                await asyncio.sleep(30)  # Check every 30 seconds
                
            except Exception as e:
                await self._alert(f"Monitor error: {e}", severity="HIGH")
                await asyncio.sleep(60)

    async def _check_tvl(self):
        """ตรวจสอบ TVL ลดลงผิดปกติ"""
        current_tvl = await self._get_tvl()
        
        if self.last_tvl > 0:
            change_percent = abs(current_tvl - self.last_tvl) / self.last_tvl * 100
            
            if current_tvl < self.last_tvl and change_percent > self.MIN_TVL_DROP_PERCENT:
                await self._alert(
                    f"CRITICAL: TVL dropped {change_percent:.1f}% in last check\n"
                    f"From: ${self.last_tvl:,.0f} → ${current_tvl:,.0f}",
                    severity="CRITICAL"
                )
        
        self.last_tvl = current_tvl

    async def _check_unusual_activity(self):
        """ตรวจสอบ transactions ที่ผิดปกติ"""
        recent_txs = await self._get_recent_transactions(limit=50)
        
        for tx in recent_txs:
            if tx['value'] > self.MAX_SINGLE_DEPOSIT:
                await self._alert(
                    f"Large deposit: ${tx['value']:,.0f} from {tx['from']}",
                    severity="MEDIUM"
                )
            
            if tx['type'] == 'withdrawal':
                self.hourly_withdrawals += tx['value']
        
        if self.hourly_withdrawals > self.MAX_HOURLY_WITHDRAWALS:
            await self._alert(
                f"High withdrawal rate: ${self.hourly_withdrawals:,.0f}/hour",
                severity="HIGH"
            )

    async def _alert(self, message: str, severity: str = "INFO"):
        """ส่ง alert ไปยัง team"""
        timestamp = datetime.now().isoformat()
        alert_msg = f"[{severity}] {timestamp}\n{message}"
        
        print(alert_msg)
        
        # ส่งไปยัง Discord/Telegram/PagerDuty
        if severity in ["HIGH", "CRITICAL"]:
            await self._send_discord_alert(alert_msg)
            await self._send_pagerduty_alert(alert_msg)

    async def _send_discord_alert(self, message: str):
        webhook_url = "DISCORD_WEBHOOK_URL"
        async with aiohttp.ClientSession() as session:
            await session.post(webhook_url, json={"content": f"🚨 {message}"})

    async def _get_tvl(self) -> float:
        # Query contract for current TVL
        return 0.0

    async def _get_recent_transactions(self, limit: int = 50):
        return []

    async def _send_pagerduty_alert(self, message: str):
        pass


if __name__ == "__main__":
    monitor = LaunchMonitor(
        rpc_url="https://mainnet.infura.io/v3/YOUR_KEY",
        contract_address="0x..."
    )
    asyncio.run(monitor.monitor_forever())
```

---

## 7. Post-launch Communication Templates

```markdown
## Communication Templates

### Launch Announcement
🚀 [Protocol Name] is live on Ethereum mainnet!

After [X] months of development and [X] security audits,
we're thrilled to open deposits.

Phase 1 Details:
- TVL Cap: $100k (will increase progressively)
- Whitelist: First 100 users
- Audit: [Link]
- Code: [GitHub]

Stay safe, start small. 🙏

---

### Incident Communication (Template)

IMPORTANT: Security Issue Detected

We have identified [brief description] affecting [scope].

Current Status:
- Protocol is [paused/affected/monitoring]
- User funds are [safe/at risk]
- Team is [investigating/working on fix]

Immediate Actions:
1. [Action 1]
2. [Action 2]

We will update every 2 hours until resolved.

More info: [Discord/Forum link]

---

### All-Clear After Incident

✅ Protocol Resumed

The issue has been resolved:
- Root cause: [brief explanation]
- Fix applied: [brief description]
- User impact: [none/X affected]

Full post-mortem: [link]

Thank you for your patience and trust.
```

---

## 8. สรุป Protocol Launch

### สิ่งที่ต้องทำ ก่อน Launch
1. ✅ Security audit (2+ auditors)
2. ✅ Bug bounty program active
3. ✅ Multi-sig configured
4. ✅ Monitoring alerts ready
5. ✅ Incident response plan written

### สิ่งที่ต้องทำ หลัง Launch
1. Monitor 24/7 ใน 30 วันแรก
2. Gradual TVL cap increases
3. Community communication regular
4. Iterate based on feedback
5. Work toward full decentralization

### Key Metrics ที่ต้อง Track
- TVL growth rate
- User adoption
- Transaction volume
- Bug bounty submissions
- Community sentiment

---

## แบบฝึกหัด

1. สร้าง deployment script สำหรับ testnet
2. เขียน bug bounty program policy
3. ออกแบบ gradual launch strategy สำหรับ protocol ของคุณ
4. Setup monitoring dashboard
5. Rehearse incident response drill

---

*จบ Part 090: Protocol Launch Playbook*

*จบ Series: Vyper Smart Contract Development (Parts 077-090)*
