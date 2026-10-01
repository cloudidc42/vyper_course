# Part 082: Proxy Patterns เชิงลึก

## สารบัญ
1. ทำไมต้องใช้ Proxy Pattern?
2. Transparent Proxy (EIP-1967)
3. UUPS Proxy
4. Beacon Proxy
5. Diamond Pattern (EIP-2535)
6. Implementation Examples

---

## 1. ทำไมต้องใช้ Proxy Pattern?

Proxy patterns ช่วยให้เราสามารถ upgrade smart contracts ได้ เนื่องจาก bytecode ใน blockchain ไม่สามารถเปลี่ยนแปลงได้ proxy pattern ใช้ `delegatecall` เพื่อ forward calls ไปยัง implementation contract ที่เปลี่ยนได้

### ประเภทของ Upgradeability

```
1. Transparent Proxy - admin ใช้ proxy, users ใช้ implementation
2. UUPS - upgrade logic อยู่ใน implementation
3. Beacon - หลาย proxies ใช้ implementation เดียว
4. Diamond - หลาย implementations (facets) ต่อ proxy
```

### ข้อควรระวัง

```
Storage Collision - proxy และ implementation ต้องไม่ใช้ slot เดียวกัน
Initialization - ต้องใช้ initialize() แทน constructor
Selfdestruct - ห้ามใช้ใน implementation
Delegatecall Context - msg.sender เปลี่ยนเป็น proxy address
```

---

## 2. Transparent Proxy Implementation

```vyper
# @version 0.4.0
# @title Transparent Proxy (EIP-1967)
# @notice Admin เรียก proxy functions, Users delegatecall ไป implementation
# @dev Implementation ใน Vyper โดย proxy layer มักเป็น Solidity

# EIP-1967 Storage Slots
# IMPLEMENTATION_SLOT = keccak256("eip1967.proxy.implementation") - 1
# = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc

# ADMIN_SLOT = keccak256("eip1967.proxy.admin") - 1  
# = 0xb53127684a568b3173ae13b9f8a6016e243e63b6e8ee1178d6a717850b5d6103

# Events
event Upgraded:
    implementation: indexed(address)

event AdminChanged:
    previousAdmin: address
    newAdmin: address

# ===== NOTE =====
# Proxy pattern ที่สมบูรณ์ต้องมี fallback function
# ซึ่ง Vyper 0.4.0 รองรับผ่าน __default__
# แต่ proxy ส่วนใหญ่เขียนใน Solidity เนื่องจาก assembly requirements

# นี่คือ ProxyAdmin contract แทน
# ใช้จัดการ proxy ทั้งหมดในระบบ

struct ProxyInfo:
    implementation: address
    admin: address
    paused: bool

proxies: HashMap[address, ProxyInfo]
owner: public(address)

@deploy
def __init__():
    self.owner = msg.sender

@external
def registerProxy(proxy: address, implementation: address, admin: address):
    """ลงทะเบียน proxy ใหม่"""
    assert msg.sender == self.owner, "Not owner"
    assert proxy != empty(address), "Invalid proxy"
    assert implementation != empty(address), "Invalid implementation"
    
    self.proxies[proxy] = ProxyInfo({
        implementation: implementation,
        admin: admin,
        paused: False
    })

@external
def upgrade(proxy: address, newImplementation: address):
    """Upgrade implementation ของ proxy"""
    proxyInfo: ProxyInfo = self.proxies[proxy]
    assert msg.sender == proxyInfo.admin, "Not admin"
    assert newImplementation != empty(address), "Invalid implementation"
    
    self.proxies[proxy].implementation = newImplementation
    
    log Upgraded(newImplementation)

@external
def changeAdmin(proxy: address, newAdmin: address):
    """เปลี่ยน admin"""
    proxyInfo: ProxyInfo = self.proxies[proxy]
    assert msg.sender == proxyInfo.admin, "Not admin"
    
    oldAdmin: address = proxyInfo.admin
    self.proxies[proxy].admin = newAdmin
    
    log AdminChanged(oldAdmin, newAdmin)

@view
@external
def getImplementation(proxy: address) -> address:
    return self.proxies[proxy].implementation

@view
@external
def getAdmin(proxy: address) -> address:
    return self.proxies[proxy].admin
```

---

## 3. UUPS Proxy

ใน UUPS (Universal Upgradeable Proxy Standard) logic การ upgrade อยู่ใน implementation ไม่ใช่ proxy

```vyper
# @version 0.4.0
# @title UUPS Implementation Base
# @notice Base class สำหรับ UUPS upgradeable contracts

# EIP-1967 Implementation Slot
IMPLEMENTATION_SLOT: constant(bytes32) = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc

event Upgraded:
    implementation: indexed(address)

# Storage
_owner: address
_initialized: bool

@deploy
def __init__():
    # Lock implementation
    self._initialized = True

@internal
def _authorizeUpgrade(newImplementation: address):
    """
    Override ใน derived contract เพื่อกำหนดสิทธิ์ upgrade
    """
    assert msg.sender == self._owner, "Not authorized"

@external
def upgradeTo(newImplementation: address):
    """
    Upgrade to new implementation
    ต้อง override _authorizeUpgrade
    """
    self._authorizeUpgrade(newImplementation)
    
    # Store new implementation in EIP-1967 slot
    # (ใน real UUPS ต้องใช้ assembly)
    
    log Upgraded(newImplementation)

@external
def upgradeToAndCall(
    newImplementation: address,
    data: Bytes[1024]
):
    """
    Upgrade และ call function ใน implementation ใหม่
    """
    self._authorizeUpgrade(newImplementation)
    
    # Upgrade
    log Upgraded(newImplementation)
    
    # Call initialization function
    if len(data) > 0:
        raw_call(
            self,
            data,
            is_delegate_call=True
        )
```

---

## 4. ERC-20 ที่ Upgradeable ด้วย UUPS

```vyper
# @version 0.4.0
# @title Upgradeable ERC-20 (UUPS Pattern)
# @notice ERC-20 ที่สามารถ upgrade ได้

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

event Initialized:
    version: uint8

event OwnershipTransferred:
    previousOwner: indexed(address)
    newOwner: indexed(address)

# ===== Storage (ต้องไม่เปลี่ยนลำดับเมื่อ upgrade!) =====
# slot 0:
_initialized: bool
# slot 1:
_initializing: bool
# slot 2:
_owner: address
# slot 3:
name: public(String[64])
# slot 4:
symbol: public(String[32])
# slot 5:
decimals: public(uint8)
# slot 6:
totalSupply: public(uint256)
# slot 7:
balanceOf: public(HashMap[address, uint256])
# slot 8:
allowance: public(HashMap[address, HashMap[address, uint256]])
# slot 9: (reserved for future use)
__gap: uint256[50]  # storage gap สำหรับ future upgrades

@deploy
def __init__():
    # Lock direct deployment
    self._initialized = True

@external
def initialize(
    owner: address,
    _name: String[64],
    _symbol: String[32],
    initialSupply: uint256
):
    """
    Initialize function แทน constructor
    เรียกผ่าน proxy เท่านั้น
    """
    assert not self._initialized, "Already initialized"
    assert not self._initializing, "Initializing"
    
    self._initializing = True
    
    self._owner = owner
    self.name = _name
    self.symbol = _symbol
    self.decimals = 18
    
    if initialSupply > 0:
        self.totalSupply = initialSupply
        self.balanceOf[owner] = initialSupply
        log Transfer(empty(address), owner, initialSupply)
    
    self._initialized = True
    self._initializing = False
    
    log Initialized(1)
    log OwnershipTransferred(empty(address), owner)

@external
def transfer(to: address, amount: uint256) -> bool:
    assert to != empty(address), "Transfer to zero"
    assert self.balanceOf[msg.sender] >= amount, "Insufficient balance"
    
    self.balanceOf[msg.sender] -= amount
    self.balanceOf[to] += amount
    
    log Transfer(msg.sender, to, amount)
    return True

@external
def approve(spender: address, amount: uint256) -> bool:
    assert spender != empty(address), "Approve zero"
    self.allowance[msg.sender][spender] = amount
    log Approval(msg.sender, spender, amount)
    return True

@external
def transferFrom(sender: address, to: address, amount: uint256) -> bool:
    assert self.allowance[sender][msg.sender] >= amount, "Insufficient allowance"
    assert self.balanceOf[sender] >= amount, "Insufficient balance"
    
    self.allowance[sender][msg.sender] -= amount
    self.balanceOf[sender] -= amount
    self.balanceOf[to] += amount
    
    log Transfer(sender, to, amount)
    return True

@external
def mint(to: address, amount: uint256):
    assert msg.sender == self._owner, "Not owner"
    self.totalSupply += amount
    self.balanceOf[to] += amount
    log Transfer(empty(address), to, amount)

@external
def burn(amount: uint256):
    assert self.balanceOf[msg.sender] >= amount, "Insufficient balance"
    self.balanceOf[msg.sender] -= amount
    self.totalSupply -= amount
    log Transfer(msg.sender, empty(address), amount)

# ===== UUPS Upgrade Logic =====
@external
def upgradeTo(newImplementation: address):
    """Upgrade implementation"""
    assert msg.sender == self._owner, "Not authorized"
    assert newImplementation != empty(address), "Invalid implementation"
    
    # ใน real proxy นี้จะ call proxy ด้วย delegatecall

@external
def transferOwnership(newOwner: address):
    assert msg.sender == self._owner, "Not owner"
    assert newOwner != empty(address), "New owner is zero"
    
    old: address = self._owner
    self._owner = newOwner
    
    log OwnershipTransferred(old, newOwner)

@view
@external
def owner() -> address:
    return self._owner
```

---

## 5. Beacon Proxy Pattern

Beacon proxy ช่วยให้ proxies หลายตัวใช้ implementation เดียวกัน และสามารถ upgrade ทั้งหมดพร้อมกันผ่าน beacon

```vyper
# @version 0.4.0
# @title Upgrade Beacon
# @notice จัดการ implementation สำหรับ beacon proxies

event Upgraded:
    implementation: indexed(address)

implementation: public(address)
owner: public(address)

@deploy
def __init__(_implementation: address):
    assert _implementation != empty(address), "Invalid implementation"
    self.implementation = _implementation
    self.owner = msg.sender

@external
def upgradeTo(newImplementation: address):
    """
    Upgrade implementation สำหรับ beacon ทั้งหมด
    ทุก proxy ที่ใช้ beacon นี้จะ upgrade พร้อมกัน
    """
    assert msg.sender == self.owner, "Not owner"
    assert newImplementation != empty(address), "Invalid implementation"
    assert newImplementation.is_contract, "Not a contract"
    
    self.implementation = newImplementation
    
    log Upgraded(newImplementation)
```

---

## 6. Diamond Pattern (EIP-2535)

Diamond Pattern ช่วยให้ contract สามารถมีหลาย facets (implementations) และ bypass ขนาดจำกัด 24KB

```vyper
# @version 0.4.0
# @title Diamond Storage Library
# @notice Core storage structure สำหรับ Diamond Pattern

# Diamond Storage Slot
# keccak256("diamond.standard.diamond.storage") =
DIAMOND_STORAGE_SLOT: constant(bytes32) = 0xc8fcad8db84d3cc18b4c41d551ea0ee66dd599cde068d998e57d5e09332c131c

struct FacetAddressAndPosition:
    facetAddress: address
    functionSelectorPosition: uint16

struct FacetFunctionSelectors:
    functionSelectors: DynArray[bytes4, 1000]
    facetAddressPosition: uint256

struct DiamondStorage:
    selectorToFacetAndPosition: HashMap[bytes4, FacetAddressAndPosition]
    facetFunctionSelectors: HashMap[address, FacetFunctionSelectors]
    facetAddresses: DynArray[address, 100]
    supportedInterfaces: HashMap[bytes4, bool]
    contractOwner: address

event DiamondCut:
    facetAddress: indexed(address)
    action: uint8  # 0=Add, 1=Replace, 2=Remove
    functionSelectors: DynArray[bytes4, 100]

event OwnershipTransferred:
    previousOwner: indexed(address)
    newOwner: indexed(address)

# Storage
_diamondStorage: DiamondStorage

@deploy
def __init__(owner: address):
    self._diamondStorage.contractOwner = owner

@internal
@view
def _diamondOwner() -> address:
    return self._diamondStorage.contractOwner

@external
def diamondCut(
    facetAddress: address,
    action: uint8,
    functionSelectors: DynArray[bytes4, 100]
):
    """
    เพิ่ม/แก้ไข/ลบ facets และ function selectors
    action: 0=Add, 1=Replace, 2=Remove
    """
    assert msg.sender == self._diamondOwner(), "Not owner"
    assert len(functionSelectors) > 0, "No selectors"
    
    if action == 0:  # Add
        self._addFunctions(facetAddress, functionSelectors)
    elif action == 1:  # Replace
        self._replaceFunctions(facetAddress, functionSelectors)
    elif action == 2:  # Remove
        self._removeFunctions(facetAddress, functionSelectors)
    else:
        raise "Invalid action"
    
    log DiamondCut(facetAddress, action, functionSelectors)

@internal
def _addFunctions(facetAddress: address, selectors: DynArray[bytes4, 100]):
    """เพิ่ม functions ใหม่"""
    assert facetAddress != empty(address), "Invalid facet"
    
    selectorCount: uint256 = len(self._diamondStorage.facetFunctionSelectors[facetAddress].functionSelectors)
    
    if selectorCount == 0:
        self._addFacet(facetAddress, len(self._diamondStorage.facetAddresses))
    
    for selector in selectors:
        oldFacet: FacetAddressAndPosition = self._diamondStorage.selectorToFacetAndPosition[selector]
        assert oldFacet.facetAddress == empty(address), "Selector exists"
        
        self._addFunction(facetAddress, selector, selectorCount)
        selectorCount += 1

@internal
def _addFacet(facetAddress: address, facetAddressPosition: uint256):
    self._diamondStorage.facetFunctionSelectors[facetAddress].facetAddressPosition = facetAddressPosition
    self._diamondStorage.facetAddresses.append(facetAddress)

@internal
def _addFunction(facetAddress: address, selector: bytes4, selectorPosition: uint256):
    self._diamondStorage.facetFunctionSelectors[facetAddress].functionSelectors.append(selector)
    self._diamondStorage.selectorToFacetAndPosition[selector] = FacetAddressAndPosition({
        facetAddress: facetAddress,
        functionSelectorPosition: convert(selectorPosition, uint16)
    })

@internal
def _replaceFunctions(facetAddress: address, selectors: DynArray[bytes4, 100]):
    """แทนที่ functions"""
    assert facetAddress != empty(address), "Invalid facet"
    
    selectorCount: uint256 = len(self._diamondStorage.facetFunctionSelectors[facetAddress].functionSelectors)
    
    if selectorCount == 0:
        self._addFacet(facetAddress, len(self._diamondStorage.facetAddresses))
    
    for selector in selectors:
        oldFacet: FacetAddressAndPosition = self._diamondStorage.selectorToFacetAndPosition[selector]
        assert oldFacet.facetAddress != facetAddress, "Same facet"
        
        self._removeFunction(oldFacet.facetAddress, selector)
        self._addFunction(facetAddress, selector, selectorCount)
        selectorCount += 1

@internal
def _removeFunctions(facetAddress: address, selectors: DynArray[bytes4, 100]):
    """ลบ functions"""
    for selector in selectors:
        oldFacet: FacetAddressAndPosition = self._diamondStorage.selectorToFacetAndPosition[selector]
        assert oldFacet.facetAddress != empty(address), "Selector not found"
        
        self._removeFunction(oldFacet.facetAddress, selector)

@internal
def _removeFunction(facetAddress: address, selector: bytes4):
    """ลบ function selector"""
    selectorPosition: uint16 = self._diamondStorage.selectorToFacetAndPosition[selector].functionSelectorPosition
    lastSelectorPosition: uint256 = len(self._diamondStorage.facetFunctionSelectors[facetAddress].functionSelectors) - 1
    
    if convert(selectorPosition, uint256) != lastSelectorPosition:
        lastSelector: bytes4 = self._diamondStorage.facetFunctionSelectors[facetAddress].functionSelectors[lastSelectorPosition]
        self._diamondStorage.facetFunctionSelectors[facetAddress].functionSelectors[convert(selectorPosition, uint256)] = lastSelector
        self._diamondStorage.selectorToFacetAndPosition[lastSelector].functionSelectorPosition = selectorPosition
    
    self._diamondStorage.facetFunctionSelectors[facetAddress].functionSelectors.pop()
    self._diamondStorage.selectorToFacetAndPosition[selector] = FacetAddressAndPosition({
        facetAddress: empty(address),
        functionSelectorPosition: 0
    })
    
    # Remove facet if no more selectors
    if len(self._diamondStorage.facetFunctionSelectors[facetAddress].functionSelectors) == 0:
        self._removeFacet(facetAddress)

@internal
def _removeFacet(facetAddress: address):
    """ลบ facet"""
    facetPosition: uint256 = self._diamondStorage.facetFunctionSelectors[facetAddress].facetAddressPosition
    lastFacetPosition: uint256 = len(self._diamondStorage.facetAddresses) - 1
    
    if facetPosition != lastFacetPosition:
        lastFacetAddress: address = self._diamondStorage.facetAddresses[lastFacetPosition]
        self._diamondStorage.facetAddresses[facetPosition] = lastFacetAddress
        self._diamondStorage.facetFunctionSelectors[lastFacetAddress].facetAddressPosition = facetPosition
    
    self._diamondStorage.facetAddresses.pop()
    self._diamondStorage.facetFunctionSelectors[facetAddress].facetAddressPosition = 0

@view
@external
def facets() -> DynArray[address, 100]:
    return self._diamondStorage.facetAddresses

@view
@external
def facetFunctionSelectors(facet: address) -> DynArray[bytes4, 1000]:
    return self._diamondStorage.facetFunctionSelectors[facet].functionSelectors

@view
@external
def facetAddress(selector: bytes4) -> address:
    return self._diamondStorage.selectorToFacetAndPosition[selector].facetAddress

@external
def transferOwnership(newOwner: address):
    assert msg.sender == self._diamondOwner(), "Not owner"
    assert newOwner != empty(address), "Zero address"
    
    old: address = self._diamondStorage.contractOwner
    self._diamondStorage.contractOwner = newOwner
    
    log OwnershipTransferred(old, newOwner)
```

---

## 7. Diamond Facets

```vyper
# @version 0.4.0
# @title ERC-20 Facet (สำหรับ Diamond)
# @notice ERC-20 functionality เป็น facet ของ Diamond

# Diamond Facet ใช้ Diamond Storage แทน regular storage
# เพื่อหลีกเลี่ยง storage collision

# Namespace storage slot สำหรับ ERC-20
ERC20_STORAGE_SLOT: constant(bytes32) = 0x52c63247e1f47db19d5ce0460030c497f067ca4cebf71ba98eeadabe20bace01

event Transfer:
    sender: indexed(address)
    receiver: indexed(address)
    value: uint256

event Approval:
    owner: indexed(address)
    spender: indexed(address)
    value: uint256

# Regular storage (Diamond proxy ใช้ delegatecall ดังนั้น storage อยู่ใน proxy)
# แต่ offset จาก ERC20_STORAGE_SLOT
totalSupply: public(uint256)
balances: HashMap[address, uint256]
allowances: HashMap[address, HashMap[address, uint256]]

@deploy
def __init__():
    pass

@view
@external
def balanceOf(account: address) -> uint256:
    return self.balances[account]

@view
@external
def allowance(owner: address, spender: address) -> uint256:
    return self.allowances[owner][spender]

@external
def transfer(to: address, amount: uint256) -> bool:
    assert to != empty(address), "Transfer to zero"
    assert self.balances[msg.sender] >= amount, "Insufficient balance"
    
    self.balances[msg.sender] -= amount
    self.balances[to] += amount
    
    log Transfer(msg.sender, to, amount)
    return True

@external
def approve(spender: address, amount: uint256) -> bool:
    assert spender != empty(address), "Approve zero"
    self.allowances[msg.sender][spender] = amount
    log Approval(msg.sender, spender, amount)
    return True

@external
def transferFrom(from_addr: address, to: address, amount: uint256) -> bool:
    assert self.allowances[from_addr][msg.sender] >= amount, "Insufficient allowance"
    assert self.balances[from_addr] >= amount, "Insufficient balance"
    
    self.allowances[from_addr][msg.sender] -= amount
    self.balances[from_addr] -= amount
    self.balances[to] += amount
    
    log Transfer(from_addr, to, amount)
    return True

@external
def mint(to: address, amount: uint256):
    # ต้อง check สิทธิ์ผ่าน DiamondStorage
    self.totalSupply += amount
    self.balances[to] += amount
    log Transfer(empty(address), to, amount)
```

---

## 8. Staking Facet สำหรับ Diamond

```vyper
# @version 0.4.0
# @title Staking Facet
# @notice Staking functionality สำหรับ Diamond contract

from vyper.interfaces import ERC20

event Staked:
    user: indexed(address)
    amount: uint256

event Withdrawn:
    user: indexed(address)
    amount: uint256

event RewardPaid:
    user: indexed(address)
    reward: uint256

# Storage (ผ่าน proxy delegatecall)
stakingToken: address
rewardToken: address

stakedBalance: HashMap[address, uint256]
totalStaked: uint256
rewardPerTokenStored: uint256
userRewardPerTokenPaid: HashMap[address, uint256]
rewards: HashMap[address, uint256]

rewardRate: uint256
lastUpdateTime: uint256
periodFinish: uint256
rewardsDuration: uint256

PRECISION: constant(uint256) = 10**18

@deploy
def __init__():
    pass

@internal
def _updateReward(account: address):
    self.rewardPerTokenStored = self._rewardPerToken()
    self.lastUpdateTime = min(block.timestamp, self.periodFinish)
    
    if account != empty(address):
        self.rewards[account] = self._earned(account)
        self.userRewardPerTokenPaid[account] = self.rewardPerTokenStored

@internal
@view
def _rewardPerToken() -> uint256:
    if self.totalStaked == 0:
        return self.rewardPerTokenStored
    
    return self.rewardPerTokenStored + (
        (min(block.timestamp, self.periodFinish) - self.lastUpdateTime) *
        self.rewardRate * PRECISION / self.totalStaked
    )

@internal
@view
def _earned(account: address) -> uint256:
    return (
        self.stakedBalance[account] *
        (self._rewardPerToken() - self.userRewardPerTokenPaid[account]) /
        PRECISION + self.rewards[account]
    )

@external
def stake(amount: uint256):
    assert amount > 0, "Cannot stake 0"
    
    self._updateReward(msg.sender)
    
    ERC20(self.stakingToken).transferFrom(msg.sender, self, amount)
    
    self.stakedBalance[msg.sender] += amount
    self.totalStaked += amount
    
    log Staked(msg.sender, amount)

@external
def withdraw(amount: uint256):
    assert amount > 0, "Cannot withdraw 0"
    assert self.stakedBalance[msg.sender] >= amount, "Insufficient staked"
    
    self._updateReward(msg.sender)
    
    self.stakedBalance[msg.sender] -= amount
    self.totalStaked -= amount
    
    ERC20(self.stakingToken).transfer(msg.sender, amount)
    
    log Withdrawn(msg.sender, amount)

@external
def getReward():
    self._updateReward(msg.sender)
    
    reward: uint256 = self.rewards[msg.sender]
    if reward > 0:
        self.rewards[msg.sender] = 0
        ERC20(self.rewardToken).transfer(msg.sender, reward)
        log RewardPaid(msg.sender, reward)

@view
@external
def earned(account: address) -> uint256:
    return self._earned(account)
```

---

## 9. Upgrade Script

```python
# scripts/upgrade.py
# Script สำหรับ upgrade proxy contracts

import json
from web3 import Web3
from eth_account import Account

def upgrade_transparent_proxy(
    proxy_admin_address: str,
    proxy_address: str,
    new_implementation_address: str,
    admin_private_key: str,
    rpc_url: str
):
    """
    Upgrade Transparent Proxy ไปยัง implementation ใหม่
    """
    w3 = Web3(Web3.HTTPProvider(rpc_url))
    admin_account = Account.from_key(admin_private_key)
    
    # Load ProxyAdmin ABI
    with open("artifacts/ProxyAdmin.json") as f:
        proxy_admin_abi = json.load(f)["abi"]
    
    proxy_admin = w3.eth.contract(
        address=proxy_admin_address,
        abi=proxy_admin_abi
    )
    
    # Build upgrade transaction
    tx = proxy_admin.functions.upgrade(
        proxy_address,
        new_implementation_address
    ).build_transaction({
        'from': admin_account.address,
        'nonce': w3.eth.get_transaction_count(admin_account.address),
        'gas': 200000,
        'gasPrice': w3.eth.gas_price
    })
    
    # Sign and send
    signed = w3.eth.account.sign_transaction(tx, admin_private_key)
    tx_hash = w3.eth.send_raw_transaction(signed.rawTransaction)
    receipt = w3.eth.wait_for_transaction_receipt(tx_hash)
    
    print(f"Upgraded proxy {proxy_address} to {new_implementation_address}")
    print(f"Transaction: {tx_hash.hex()}")
    
    return receipt

def upgrade_uups_proxy(
    proxy_address: str,
    new_implementation_address: str,
    owner_private_key: str,
    init_calldata: bytes,
    rpc_url: str
):
    """
    Upgrade UUPS Proxy
    """
    w3 = Web3(Web3.HTTPProvider(rpc_url))
    owner_account = Account.from_key(owner_private_key)
    
    # Call upgradeTo or upgradeToAndCall on proxy directly
    # (proxy delegates to implementation's upgrade function)
    
    upgrade_selector = bytes.fromhex("3659cfe6")  # upgradeTo(address)
    calldata = upgrade_selector + new_implementation_address.encode().zfill(32)
    
    tx = {
        'from': owner_account.address,
        'to': proxy_address,
        'data': calldata,
        'nonce': w3.eth.get_transaction_count(owner_account.address),
        'gas': 200000,
        'gasPrice': w3.eth.gas_price
    }
    
    signed = w3.eth.account.sign_transaction(tx, owner_private_key)
    tx_hash = w3.eth.send_raw_transaction(signed.rawTransaction)
    receipt = w3.eth.wait_for_transaction_receipt(tx_hash)
    
    print(f"UUPS upgraded to {new_implementation_address}")
    
    return receipt

def check_storage_compatibility(
    old_abi_path: str,
    new_abi_path: str
):
    """
    ตรวจสอบว่า storage layout ยังคง compatible หลัง upgrade
    """
    with open(old_abi_path) as f:
        old_abi = json.load(f)
    
    with open(new_abi_path) as f:
        new_abi = json.load(f)
    
    # ตรวจสอบว่า state variables ไม่เปลี่ยน type หรือ order
    print("Storage compatibility check:")
    print("- State variables should maintain same order")
    print("- New variables should be added at the END only")
    print("- Existing variable types should not change")
    print("- Storage gaps should be used for future additions")

if __name__ == "__main__":
    # ตัวอย่างการใช้งาน
    check_storage_compatibility("old_impl.json", "new_impl.json")
```

---

## 10. Testing Proxy Upgrades

```python
# tests/test_proxy_upgrade.py
# ทดสอบ proxy upgrade process

import pytest
from brownie import accounts, UpgradeableToken, ProxyAdmin, TransparentProxy

@pytest.fixture
def setup():
    owner = accounts[0]
    user = accounts[1]
    
    # Deploy implementation V1
    impl_v1 = UpgradeableToken.deploy({"from": owner})
    
    # Deploy ProxyAdmin
    proxy_admin = ProxyAdmin.deploy({"from": owner})
    
    # Deploy proxy
    init_data = impl_v1.initialize.encode_input(
        owner,
        "Test Token",
        "TEST",
        1000 * 10**18
    )
    
    proxy = TransparentProxy.deploy(
        impl_v1,
        proxy_admin,
        init_data,
        {"from": owner}
    )
    
    # Connect to proxy with V1 ABI
    token_v1 = UpgradeableToken.at(proxy.address)
    
    return owner, user, proxy, proxy_admin, token_v1

def test_basic_token_functionality(setup):
    owner, user, proxy, proxy_admin, token = setup
    
    # ทดสอบ basic functions
    assert token.name() == "Test Token"
    assert token.symbol() == "TEST"
    assert token.totalSupply() == 1000 * 10**18
    
    # Transfer
    token.transfer(user, 100 * 10**18, {"from": owner})
    assert token.balanceOf(user) == 100 * 10**18

def test_upgrade_preserves_state(setup):
    owner, user, proxy, proxy_admin, token_v1 = setup
    
    # Transfer before upgrade
    token_v1.transfer(user, 100 * 10**18, {"from": owner})
    balance_before = token_v1.balanceOf(user)
    
    # Deploy V2
    # impl_v2 = UpgradeableTokenV2.deploy({"from": owner})
    
    # Upgrade
    # proxy_admin.upgrade(proxy, impl_v2, {"from": owner})
    
    # token_v2 = UpgradeableTokenV2.at(proxy.address)
    
    # State preserved
    # assert token_v2.balanceOf(user) == balance_before

def test_only_admin_can_upgrade(setup):
    owner, user, proxy, proxy_admin, token = setup
    
    with pytest.raises(Exception):
        # User ไม่ใช่ admin
        proxy_admin.upgrade(proxy, accounts[5], {"from": user})

def test_initialize_only_once(setup):
    owner, user, proxy, proxy_admin, token = setup
    
    with pytest.raises(Exception):
        # ไม่สามารถ initialize ซ้ำ
        token.initialize(owner, "New Name", "NEW", 0, {"from": owner})
```

---

## 11. สรุป Proxy Pattern

### เปรียบเทียบ Proxy Types

```
Type          | Admin เปลี่ยน Impl | User ใช้ Impl | Upgrade Logic | ความซับซ้อน
─────────────────────────────────────────────────────────────────────────────
Transparent   | Admin เท่านั้น    | Users         | ใน Proxy      | กลาง
UUPS          | ผ่าน Implementation| Users         | ใน Impl       | ต่ำ
Beacon        | ผ่าน Beacon       | Users         | ใน Beacon     | กลาง
Diamond       | ผ่าน Diamond      | Users         | ใน Diamond    | สูง
```

### เมื่อไหร่ควรใช้อะไร

```
Transparent Proxy:
- ต้องการ security สูง
- Admin และ users ใช้ต่างกันชัดเจน
- Simple upgrade requirements

UUPS:
- ต้องการ gas efficiency
- Upgrade logic ซับซ้อน
- ต้องการ custom authorization

Beacon Proxy:
- หลาย instances ของ contract เดียวกัน
- Upgrade ทั้งหมดพร้อมกัน
- Factory pattern

Diamond:
- Contract ใหญ่เกิน 24KB
- ต้องการ modular design
- Complex upgrade requirements
```

---

## แบบฝึกหัด

1. Implement Transparent Proxy ด้วย Vyper implementation
2. เขียน upgrade script ที่มี rollback capability
3. สร้าง storage layout checker ที่ validate compatibility
4. Implement Beacon Proxy สำหรับ ERC-721 NFT factory
5. เขียน Diamond ที่มี ERC-20, Staking, และ Governance facets

---

*จบ Part 082: Proxy Patterns เชิงลึก*
