# Part 083: On-chain Monitoring

## สารบัญ
1. ความสำคัญของ Monitoring
2. Event Design สำหรับ Monitoring
3. The Graph Protocol
4. Tenderly Alerts
5. OpenZeppelin Defender
6. Custom Monitoring Scripts
7. Alert Systems

---

## 1. ความสำคัญของ Monitoring

Monitoring smart contracts มีความสำคัญเพื่อ:
- ตรวจจับ attacks หรือ exploits ก่อนที่จะสาย
- Track protocol health metrics
- Debug issues ใน production
- Compliance และ audit trail

---

## 2. Contract ที่ออกแบบมาสำหรับ Monitoring

```vyper
# @version 0.4.0
# @title Monitoring-Friendly DeFi Protocol
# @notice Contract ที่ emit events ครบถ้วนสำหรับ monitoring

from vyper.interfaces import ERC20

# ===== Events ที่ออกแบบมาสำหรับ Monitoring =====

# Security Events
event OwnershipTransferred:
    previousOwner: indexed(address)
    newOwner: indexed(address)

event EmergencyPaused:
    by: indexed(address)
    reason: String[256]
    timestamp: uint256

event EmergencyUnpaused:
    by: indexed(address)
    timestamp: uint256

event SuspiciousActivityDetected:
    actor: indexed(address)
    activityType: indexed(uint8)
    amount: uint256
    timestamp: uint256

# Financial Events (สำคัญสำหรับ monitoring)
event LargeTransaction:
    user: indexed(address)
    tokenIn: indexed(address)
    tokenOut: indexed(address)
    amountIn: uint256
    amountOut: uint256
    timestamp: uint256

event LiquidityChanged:
    provider: indexed(address)
    action: indexed(uint8)  # 0=add, 1=remove
    token0Amount: uint256
    token1Amount: uint256
    totalLiquidity: uint256
    timestamp: uint256

event PriceUpdate:
    token: indexed(address)
    oldPrice: uint256
    newPrice: uint256
    source: indexed(address)
    timestamp: uint256

# System Events
event ParameterChanged:
    paramName: indexed(String[32])
    oldValue: uint256
    newValue: uint256
    changedBy: indexed(address)

event OracleUpdated:
    oracle: indexed(address)
    updater: indexed(address)

event FeeCollected:
    token: indexed(address)
    amount: uint256
    collector: indexed(address)
    timestamp: uint256

# ===== State =====
owner: public(address)
paused: public(bool)
token0: public(address)
token1: public(address)
reserve0: public(uint256)
reserve1: public(uint256)
feeRate: public(uint256)
oracle: public(address)

LARGE_TX_THRESHOLD: public(uint256)
BASIS_POINTS: constant(uint256) = 10000

@deploy
def __init__(
    _token0: address,
    _token1: address,
    _feeRate: uint256,
    _largeTxThreshold: uint256
):
    self.owner = msg.sender
    self.token0 = _token0
    self.token1 = _token1
    self.feeRate = _feeRate
    self.LARGE_TX_THRESHOLD = _largeTxThreshold

@internal
def _checkLargeTransaction(
    user: address,
    tokenIn: address,
    tokenOut: address,
    amountIn: uint256,
    amountOut: uint256
):
    """ตรวจสอบและ emit event สำหรับ large transactions"""
    if amountIn >= self.LARGE_TX_THRESHOLD or amountOut >= self.LARGE_TX_THRESHOLD:
        log LargeTransaction(
            user,
            tokenIn,
            tokenOut,
            amountIn,
            amountOut,
            block.timestamp
        )

@external
def swap(
    tokenIn: address,
    amountIn: uint256,
    minAmountOut: uint256
) -> uint256:
    """Swap ที่มี monitoring events"""
    assert not self.paused, "Paused"
    assert tokenIn == self.token0 or tokenIn == self.token1, "Invalid token"
    assert amountIn > 0, "Invalid amount"
    
    # คำนวณ output
    tokenOut: address = empty(address)
    reserveIn: uint256 = 0
    reserveOut: uint256 = 0
    
    if tokenIn == self.token0:
        tokenOut = self.token1
        reserveIn = self.reserve0
        reserveOut = self.reserve1
    else:
        tokenOut = self.token0
        reserveIn = self.reserve1
        reserveOut = self.reserve0
    
    amountInWithFee: uint256 = amountIn * (BASIS_POINTS - self.feeRate)
    amountOut: uint256 = amountInWithFee * reserveOut / (reserveIn * BASIS_POINTS + amountInWithFee)
    
    assert amountOut >= minAmountOut, "Slippage too high"
    
    # Transfer
    ERC20(tokenIn).transferFrom(msg.sender, self, amountIn)
    ERC20(tokenOut).transfer(msg.sender, amountOut)
    
    # Update reserves
    if tokenIn == self.token0:
        self.reserve0 += amountIn
        self.reserve1 -= amountOut
    else:
        self.reserve1 += amountIn
        self.reserve0 -= amountOut
    
    # Emit monitoring event for large txs
    self._checkLargeTransaction(msg.sender, tokenIn, tokenOut, amountIn, amountOut)
    
    return amountOut

@external
def updateFeeRate(newFee: uint256):
    assert msg.sender == self.owner, "Not owner"
    assert newFee <= 1000, "Fee too high"  # max 10%
    
    oldFee: uint256 = self.feeRate
    self.feeRate = newFee
    
    # Emit parameter change event สำหรับ monitoring
    log ParameterChanged("feeRate", oldFee, newFee, msg.sender)

@external
def pause(reason: String[256]):
    assert msg.sender == self.owner, "Not owner"
    self.paused = True
    log EmergencyPaused(msg.sender, reason, block.timestamp)

@external
def unpause():
    assert msg.sender == self.owner, "Not owner"
    self.paused = False
    log EmergencyUnpaused(msg.sender, block.timestamp)

@external
def updateOracle(newOracle: address):
    assert msg.sender == self.owner, "Not owner"
    
    old: address = self.oracle
    self.oracle = newOracle
    
    log OracleUpdated(newOracle, msg.sender)
    
    if old != empty(address):
        # Alert ว่า oracle เปลี่ยน (ควรตรวจสอบทุกครั้ง)
        log SuspiciousActivityDetected(
            msg.sender,
            1,  # type 1 = oracle change
            0,
            block.timestamp
        )
```

---

## 3. The Graph Subgraph

```typescript
// subgraph/schema.graphql
// GraphQL Schema สำหรับ The Graph

type Protocol @entity {
  id: ID!
  owner: Bytes!
  token0: Bytes!
  token1: Bytes!
  reserve0: BigInt!
  reserve1: BigInt!
  feeRate: BigInt!
  paused: Boolean!
  totalVolumeToken0: BigInt!
  totalVolumeToken1: BigInt!
  swapCount: BigInt!
  createdAt: BigInt!
  updatedAt: BigInt!
}

type Swap @entity {
  id: ID!
  protocol: Protocol!
  user: Bytes!
  tokenIn: Bytes!
  tokenOut: Bytes!
  amountIn: BigInt!
  amountOut: BigInt!
  timestamp: BigInt!
  blockNumber: BigInt!
  transactionHash: Bytes!
  isLargeTransaction: Boolean!
}

type LargeTransactionAlert @entity {
  id: ID!
  protocol: Protocol!
  user: Bytes!
  tokenIn: Bytes!
  tokenOut: Bytes!
  amountIn: BigInt!
  amountOut: BigInt!
  timestamp: BigInt!
}

type ParameterChange @entity {
  id: ID!
  protocol: Protocol!
  paramName: String!
  oldValue: BigInt!
  newValue: BigInt!
  changedBy: Bytes!
  timestamp: BigInt!
  blockNumber: BigInt!
}

type EmergencyEvent @entity {
  id: ID!
  protocol: Protocol!
  eventType: String!  # "pause" or "unpause"
  by: Bytes!
  reason: String
  timestamp: BigInt!
}

type DailyMetrics @entity {
  id: ID!  # date in YYYY-MM-DD format
  protocol: Protocol!
  date: String!
  volumeToken0: BigInt!
  volumeToken1: BigInt!
  swapCount: BigInt!
  uniqueUsers: BigInt!
  avgSwapSize: BigInt!
}
```

```typescript
// subgraph/mappings/protocol.ts
// Event handlers สำหรับ The Graph

import {
  BigInt,
  Bytes,
  Address,
  log
} from "@graphprotocol/graph-ts";

import {
  LargeTransaction,
  ParameterChanged,
  EmergencyPaused,
  EmergencyUnpaused,
  SuspiciousActivityDetected
} from "../generated/Protocol/Protocol";

import {
  Protocol,
  Swap,
  LargeTransactionAlert,
  ParameterChange,
  EmergencyEvent,
  DailyMetrics
} from "../generated/schema";

export function handleLargeTransaction(event: LargeTransaction): void {
  let id = event.transaction.hash.toHex() + "-" + event.logIndex.toString();
  
  let alert = new LargeTransactionAlert(id);
  alert.protocol = event.address.toHex();
  alert.user = event.params.user;
  alert.tokenIn = event.params.tokenIn;
  alert.tokenOut = event.params.tokenOut;
  alert.amountIn = event.params.amountIn;
  alert.amountOut = event.params.amountOut;
  alert.timestamp = event.params.timestamp;
  
  alert.save();
  
  log.warning(
    "Large transaction detected: user={}, amountIn={}",
    [event.params.user.toHex(), event.params.amountIn.toString()]
  );
}

export function handleParameterChanged(event: ParameterChanged): void {
  let id = event.transaction.hash.toHex() + "-" + event.logIndex.toString();
  
  let change = new ParameterChange(id);
  change.protocol = event.address.toHex();
  change.paramName = event.params.paramName;
  change.oldValue = event.params.oldValue;
  change.newValue = event.params.newValue;
  change.changedBy = event.params.changedBy;
  change.timestamp = event.block.timestamp;
  change.blockNumber = event.block.number;
  
  change.save();
}

export function handleEmergencyPaused(event: EmergencyPaused): void {
  let id = event.transaction.hash.toHex();
  
  let emergencyEvent = new EmergencyEvent(id);
  emergencyEvent.protocol = event.address.toHex();
  emergencyEvent.eventType = "pause";
  emergencyEvent.by = event.params.by;
  emergencyEvent.reason = event.params.reason;
  emergencyEvent.timestamp = event.params.timestamp;
  
  emergencyEvent.save();
  
  // Update protocol state
  let protocol = Protocol.load(event.address.toHex());
  if (protocol) {
    protocol.paused = true;
    protocol.save();
  }
}

function getDailyMetricsId(timestamp: BigInt, protocol: Address): string {
  let date = new Date(timestamp.toI64() * 1000);
  return `${date.getFullYear()}-${date.getMonth()+1}-${date.getDate()}-${protocol.toHex()}`;
}

export function handleEmergencyUnpaused(event: EmergencyUnpaused): void {
  let protocol = Protocol.load(event.address.toHex());
  if (protocol) {
    protocol.paused = false;
    protocol.save();
  }
  
  let id = event.transaction.hash.toHex();
  let emergencyEvent = new EmergencyEvent(id);
  emergencyEvent.protocol = event.address.toHex();
  emergencyEvent.eventType = "unpause";
  emergencyEvent.by = event.params.by;
  emergencyEvent.timestamp = event.params.timestamp;
  
  emergencyEvent.save();
}
```

---

## 4. subgraph.yaml Configuration

```yaml
# subgraph/subgraph.yaml

specVersion: 0.0.5
schema:
  file: ./schema.graphql

dataSources:
  - kind: ethereum
    name: Protocol
    network: mainnet
    source:
      address: "0x1234567890123456789012345678901234567890"
      abi: Protocol
      startBlock: 18000000
    mapping:
      kind: ethereum/events
      apiVersion: 0.0.7
      language: wasm/assemblyscript
      entities:
        - Protocol
        - Swap
        - LargeTransactionAlert
        - ParameterChange
        - EmergencyEvent
      abis:
        - name: Protocol
          file: ./abis/Protocol.json
      eventHandlers:
        - event: LargeTransaction(indexed address,indexed address,indexed address,uint256,uint256,uint256)
          handler: handleLargeTransaction
        - event: ParameterChanged(indexed string,uint256,uint256,indexed address)
          handler: handleParameterChanged
        - event: EmergencyPaused(indexed address,string,uint256)
          handler: handleEmergencyPaused
        - event: EmergencyUnpaused(indexed address,uint256)
          handler: handleEmergencyUnpaused
        - event: SuspiciousActivityDetected(indexed address,indexed uint8,uint256,uint256)
          handler: handleSuspiciousActivity
      file: ./src/mappings/protocol.ts
```

---

## 5. Custom Monitoring Script

```python
# scripts/monitor.py
# Custom monitoring script ที่ polls blockchain

import asyncio
import json
import os
from web3 import Web3, AsyncWeb3
from web3.middleware import geth_poa_middleware
import aiohttp
from datetime import datetime
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Configuration
RPC_URL = os.environ.get("RPC_URL", "https://mainnet.infura.io/v3/KEY")
CONTRACT_ADDRESS = os.environ.get("CONTRACT_ADDRESS")
ALERT_WEBHOOK = os.environ.get("DISCORD_WEBHOOK")
LARGE_TX_THRESHOLD = int(os.environ.get("LARGE_TX_THRESHOLD", "100000")) * 10**18

# Load ABI
with open("artifacts/Protocol.json") as f:
    CONTRACT_ABI = json.load(f)["abi"]

class ProtocolMonitor:
    def __init__(self):
        self.w3 = Web3(Web3.HTTPProvider(RPC_URL))
        self.contract = self.w3.eth.contract(
            address=CONTRACT_ADDRESS,
            abi=CONTRACT_ABI
        )
        self.last_block = self.w3.eth.block_number
        self.alert_count = 0
    
    async def send_discord_alert(self, title: str, description: str, color: int = 0xff0000):
        """ส่ง alert ไปยัง Discord"""
        if not ALERT_WEBHOOK:
            logger.warning("No webhook configured")
            return
        
        payload = {
            "embeds": [{
                "title": title,
                "description": description,
                "color": color,
                "timestamp": datetime.utcnow().isoformat(),
                "footer": {"text": f"Protocol Monitor | Alert #{self.alert_count}"}
            }]
        }
        
        async with aiohttp.ClientSession() as session:
            async with session.post(ALERT_WEBHOOK, json=payload) as response:
                if response.status == 204:
                    logger.info(f"Alert sent: {title}")
                else:
                    logger.error(f"Failed to send alert: {response.status}")
        
        self.alert_count += 1
    
    async def check_large_transactions(self, from_block: int, to_block: int):
        """ตรวจสอบ large transactions"""
        events = self.contract.events.LargeTransaction.get_logs(
            fromBlock=from_block,
            toBlock=to_block
        )
        
        for event in events:
            amount_in = event["args"]["amountIn"] / 10**18
            amount_out = event["args"]["amountOut"] / 10**18
            user = event["args"]["user"]
            
            logger.warning(f"Large transaction: user={user}, amountIn={amount_in:.2f}")
            
            await self.send_discord_alert(
                "🚨 Large Transaction Detected",
                f"**User:** {user}\n"
                f"**Amount In:** {amount_in:,.2f}\n"
                f"**Amount Out:** {amount_out:,.2f}\n"
                f"**Block:** {event['blockNumber']}\n"
                f"**TX:** {event['transactionHash'].hex()}",
                color=0xff9900  # Orange
            )
    
    async def check_emergency_events(self, from_block: int, to_block: int):
        """ตรวจสอบ emergency events"""
        pause_events = self.contract.events.EmergencyPaused.get_logs(
            fromBlock=from_block,
            toBlock=to_block
        )
        
        for event in pause_events:
            logger.critical(f"EMERGENCY PAUSE: {event['args']['reason']}")
            
            await self.send_discord_alert(
                "🔴 EMERGENCY PAUSE",
                f"**By:** {event['args']['by']}\n"
                f"**Reason:** {event['args']['reason']}\n"
                f"**Time:** {datetime.fromtimestamp(event['args']['timestamp'])}\n"
                f"**Block:** {event['blockNumber']}",
                color=0xff0000  # Red
            )
    
    async def check_parameter_changes(self, from_block: int, to_block: int):
        """ตรวจสอบ parameter changes"""
        events = self.contract.events.ParameterChanged.get_logs(
            fromBlock=from_block,
            toBlock=to_block
        )
        
        for event in events:
            param = event["args"]["paramName"]
            old_val = event["args"]["oldValue"]
            new_val = event["args"]["newValue"]
            changed_by = event["args"]["changedBy"]
            
            logger.info(f"Parameter changed: {param} {old_val} -> {new_val}")
            
            await self.send_discord_alert(
                "⚙️ Parameter Changed",
                f"**Parameter:** {param}\n"
                f"**Old Value:** {old_val}\n"
                f"**New Value:** {new_val}\n"
                f"**Changed By:** {changed_by}\n"
                f"**Block:** {event['blockNumber']}",
                color=0x0099ff  # Blue
            )
    
    async def check_protocol_health(self):
        """ตรวจสอบ protocol health metrics"""
        try:
            paused = self.contract.functions.paused().call()
            reserve0 = self.contract.functions.reserve0().call()
            reserve1 = self.contract.functions.reserve1().call()
            fee_rate = self.contract.functions.feeRate().call()
            
            logger.info(f"Protocol health: paused={paused}, r0={reserve0/10**18:.2f}, r1={reserve1/10**18:.2f}")
            
            # Alert ถ้า reserves ต่ำมาก
            min_reserve = 1000 * 10**18
            if reserve0 < min_reserve or reserve1 < min_reserve:
                await self.send_discord_alert(
                    "⚠️ Low Liquidity Warning",
                    f"**Reserve 0:** {reserve0/10**18:,.2f}\n"
                    f"**Reserve 1:** {reserve1/10**18:,.2f}\n"
                    f"**Minimum Threshold:** {min_reserve/10**18:,.2f}",
                    color=0xffff00  # Yellow
                )
        
        except Exception as e:
            logger.error(f"Health check failed: {e}")
    
    async def monitor_loop(self):
        """Main monitoring loop"""
        logger.info(f"Starting monitor from block {self.last_block}")
        
        while True:
            try:
                current_block = self.w3.eth.block_number
                
                if current_block > self.last_block:
                    logger.info(f"Scanning blocks {self.last_block+1} to {current_block}")
                    
                    # Check events
                    await self.check_large_transactions(self.last_block + 1, current_block)
                    await self.check_emergency_events(self.last_block + 1, current_block)
                    await self.check_parameter_changes(self.last_block + 1, current_block)
                    
                    self.last_block = current_block
                
                # Check health every 10 blocks
                if current_block % 10 == 0:
                    await self.check_protocol_health()
                
                await asyncio.sleep(12)  # Wait for next block (~12 seconds)
                
            except Exception as e:
                logger.error(f"Monitor error: {e}")
                await asyncio.sleep(30)

async def main():
    monitor = ProtocolMonitor()
    await monitor.monitor_loop()

if __name__ == "__main__":
    asyncio.run(main())
```

---

## 6. Tenderly Alerts Configuration

```json
// tenderly/alerts.json
// Configuration สำหรับ Tenderly Alerts

{
  "project": "my-defi-protocol",
  "network": "mainnet",
  "alerts": [
    {
      "name": "Large Transaction Alert",
      "description": "Alert when transaction > 100K tokens",
      "type": "event",
      "contract": "0x1234...",
      "event": "LargeTransaction",
      "conditions": [],
      "notifications": [
        {
          "type": "discord",
          "webhook": "https://discord.com/api/webhooks/...",
          "severity": "high"
        },
        {
          "type": "email",
          "address": "security@protocol.com",
          "severity": "critical"
        }
      ]
    },
    {
      "name": "Emergency Pause Alert",
      "description": "Alert when contract is paused",
      "type": "event",
      "contract": "0x1234...",
      "event": "EmergencyPaused",
      "conditions": [],
      "notifications": [
        {
          "type": "pagerduty",
          "integration_key": "...",
          "severity": "critical"
        }
      ]
    },
    {
      "name": "Oracle Update Alert",
      "description": "Alert when oracle is changed",
      "type": "event",
      "contract": "0x1234...",
      "event": "OracleUpdated",
      "notifications": [
        {
          "type": "slack",
          "webhook": "https://hooks.slack.com/...",
          "channel": "#security-alerts"
        }
      ]
    },
    {
      "name": "Failed Transaction Alert",
      "description": "Alert on failed transactions",
      "type": "failed_transaction",
      "contract": "0x1234...",
      "threshold": 10,
      "time_window": 300,
      "notifications": [
        {
          "type": "discord",
          "webhook": "https://discord.com/api/webhooks/..."
        }
      ]
    }
  ]
}
```

---

## 7. OpenZeppelin Defender Autotask

```javascript
// defender/autotask.js
// OpenZeppelin Defender Autotask สำหรับ automated monitoring

const { DefenderRelayProvider } = require('@openzeppelin/defender-relay-client/lib/ethers');
const { ethers } = require('ethers');

// Autotask handler
exports.handler = async function(credentials) {
  const provider = new DefenderRelayProvider(credentials);
  const signer = provider.getSigner();
  
  // Contract setup
  const contractABI = require('./Protocol.abi.json');
  const contractAddress = process.env.CONTRACT_ADDRESS;
  const contract = new ethers.Contract(contractAddress, contractABI, signer);
  
  // Check protocol health
  const [paused, reserve0, reserve1] = await Promise.all([
    contract.paused(),
    contract.reserve0(),
    contract.totalSupply ? contract.totalSupply() : ethers.constants.Zero
  ]);
  
  const health = {
    paused,
    reserve0: ethers.utils.formatEther(reserve0),
    reserve1: ethers.utils.formatEther(reserve1),
    timestamp: new Date().toISOString()
  };
  
  console.log('Protocol health:', JSON.stringify(health));
  
  // Alert ถ้า paused
  if (paused) {
    await sendAlert(
      'CRITICAL: Protocol Paused',
      `Protocol at ${contractAddress} is PAUSED`
    );
  }
  
  // Alert ถ้า reserves ต่ำ
  const minReserve = ethers.utils.parseEther('1000');
  if (reserve0.lt(minReserve) || reserve1.lt(minReserve)) {
    await sendAlert(
      'WARNING: Low Reserves',
      `Reserve0: ${health.reserve0}, Reserve1: ${health.reserve1}`
    );
  }
  
  return health;
};

async function sendAlert(title, message) {
  const webhookUrl = process.env.DISCORD_WEBHOOK;
  if (!webhookUrl) return;
  
  const response = await fetch(webhookUrl, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      embeds: [{
        title,
        description: message,
        color: 0xff0000,
        timestamp: new Date().toISOString()
      }]
    })
  });
  
  console.log('Alert sent:', response.status);
}
```

---

## 8. Grafana Dashboard

```json
// grafana/dashboard.json
// Grafana Dashboard configuration

{
  "title": "DeFi Protocol Monitor",
  "panels": [
    {
      "title": "TVL (Total Value Locked)",
      "type": "graph",
      "datasource": "The Graph",
      "targets": [
        {
          "query": "{ protocol(id: \"0x1234...\") { reserve0 reserve1 } }"
        }
      ]
    },
    {
      "title": "24h Volume",
      "type": "stat",
      "datasource": "The Graph",
      "targets": [
        {
          "query": "{ dailyMetrics(orderBy: date, orderDirection: desc, first: 1) { volumeToken0 volumeToken1 } }"
        }
      ]
    },
    {
      "title": "Large Transactions",
      "type": "table",
      "datasource": "The Graph",
      "targets": [
        {
          "query": "{ largeTransactionAlerts(orderBy: timestamp, orderDirection: desc, first: 10) { user amountIn amountOut timestamp } }"
        }
      ]
    },
    {
      "title": "Emergency Events",
      "type": "logs",
      "datasource": "The Graph",
      "targets": [
        {
          "query": "{ emergencyEvents(orderBy: timestamp, orderDirection: desc) { eventType by reason timestamp } }"
        }
      ]
    }
  ]
}
```

---

## 9. Alerting Best Practices

### Severity Levels

```
CRITICAL (PagerDuty/immediate call):
- Contract paused unexpectedly
- Exploit detected (unusual drain)
- Oracle manipulation detected
- All funds at risk

HIGH (Discord/Slack immediate):
- Large transaction > 10% TVL
- Parameter changes without timelock
- Oracle address changed
- Admin key used unexpectedly

MEDIUM (Email/Slack):
- Large transaction > 1% TVL
- Unusual swap patterns
- Fee changes
- User count drops significantly

LOW (Daily digest):
- Normal large transactions
- Routine parameter updates
- Statistics anomalies
```

### Anti-alert-fatigue Tips

```
1. ตั้ง threshold ให้เหมาะสม - ไม่มากเกินไป ไม่น้อยเกินไป
2. Group similar alerts - aggregate ก่อน notify
3. ใช้ alert cooling - ไม่ส่ง alert เดิมซ้ำใน 15 นาที
4. Escalation path - ถ้าไม่ response ใน X นาที escalate
5. On-call rotation - กระจาย responsibility
6. Run books - มี document บอกว่าต้องทำอะไรเมื่อ alert fires
```

---

## 10. Vyper Events สำหรับ Specific Monitoring Use Cases

```vyper
# @version 0.4.0
# @title AMM with Comprehensive Monitoring Events
# @notice ตัวอย่าง AMM ที่มี events ครบถ้วนสำหรับ monitoring

from vyper.interfaces import ERC20

# Health metric events
event ReservesSnapshot:
    reserve0: uint256
    reserve1: uint256
    k: uint256
    blockNumber: uint256
    timestamp: uint256

event SwapExecuted:
    user: indexed(address)
    tokenIn: indexed(address)
    amountIn: uint256
    amountOut: uint256
    priceImpact: uint256  # basis points
    reserveUtilization: uint256  # percentage
    timestamp: uint256

event FeeAccrued:
    token: indexed(address)
    amount: uint256
    cumulativeFees: uint256
    timestamp: uint256

event ArbitrageDetected:
    arbitrageur: indexed(address)
    profitEstimate: uint256
    timestamp: uint256

# State
token0: public(address)
token1: public(address)
reserve0: public(uint256)
reserve1: public(uint256)
cumulativeFees0: public(uint256)
cumulativeFees1: public(uint256)
FEE_RATE: constant(uint256) = 30  # 0.3%
FEE_BASE: constant(uint256) = 10000

@deploy
def __init__(_token0: address, _token1: address):
    self.token0 = _token0
    self.token1 = _token1

@internal
def _calculatePriceImpact(
    amountIn: uint256,
    reserveIn: uint256
) -> uint256:
    """คำนวณ price impact เป็น basis points"""
    return amountIn * FEE_BASE / (reserveIn + amountIn)

@external
def swap(
    tokenIn: address,
    amountIn: uint256,
    minOut: uint256
) -> uint256:
    assert tokenIn == self.token0 or tokenIn == self.token1, "Invalid token"
    assert amountIn > 0, "Zero amount"
    
    isToken0: bool = tokenIn == self.token0
    reserveIn: uint256 = self.reserve0 if isToken0 else self.reserve1
    reserveOut: uint256 = self.reserve1 if isToken0 else self.reserve0
    tokenOut: address = self.token1 if isToken0 else self.token0
    
    amountInWithFee: uint256 = amountIn * (FEE_BASE - FEE_RATE)
    amountOut: uint256 = amountInWithFee * reserveOut / (reserveIn * FEE_BASE + amountInWithFee)
    fee: uint256 = amountIn * FEE_RATE / FEE_BASE
    
    assert amountOut >= minOut, "Slippage"
    
    ERC20(tokenIn).transferFrom(msg.sender, self, amountIn)
    ERC20(tokenOut).transfer(msg.sender, amountOut)
    
    # Update reserves
    if isToken0:
        self.reserve0 += amountIn
        self.reserve1 -= amountOut
        self.cumulativeFees0 += fee
    else:
        self.reserve1 += amountIn
        self.reserve0 -= amountOut
        self.cumulativeFees1 += fee
    
    priceImpact: uint256 = self._calculatePriceImpact(amountIn, reserveIn)
    reserveUtil: uint256 = amountOut * 100 / reserveOut
    
    # Rich monitoring event
    log SwapExecuted(
        msg.sender,
        tokenIn,
        amountIn,
        amountOut,
        priceImpact,
        reserveUtil,
        block.timestamp
    )
    
    # Fee accrual event
    log FeeAccrued(
        tokenIn,
        fee,
        self.cumulativeFees0 if isToken0 else self.cumulativeFees1,
        block.timestamp
    )
    
    # Snapshot reserves สำหรับ tracking
    if block.number % 100 == 0:  # Every 100 blocks
        log ReservesSnapshot(
            self.reserve0,
            self.reserve1,
            self.reserve0 * self.reserve1,
            block.number,
            block.timestamp
        )
    
    return amountOut
```

---

## แบบฝึกหัด

1. สร้าง The Graph subgraph สำหรับ staking contract
2. เขียน monitoring script ที่ใช้ WebSocket แทน polling
3. ออกแบบ alert system สำหรับ lending protocol
4. สร้าง Grafana dashboard จาก The Graph data
5. Implement automated circuit breaker ที่ pause contract เมื่อตรวจพบ anomaly

---

*จบ Part 083: On-chain Monitoring*
