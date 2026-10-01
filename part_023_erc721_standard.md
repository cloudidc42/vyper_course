# Part 023: ERC-721 NFT Standard

## สารบัญ
1. [ERC-721 คืออะไร?](#what-is-erc721)
2. [ERC-721 Specification](#specification)
3. [tokenURI และ Metadata](#metadata)
4. [safeTransfer](#safe-transfer)
5. [Approval Flows](#approval-flows)
6. [Operator Approval](#operator-approval)
7. [ERC-165 supportsInterface](#erc165)
8. [แบบฝึกหัด](#exercises)

---

## 1. ERC-721 คืออะไร? {#what-is-erc721}

ERC-721 คือมาตรฐานสำหรับ **Non-Fungible Token (NFT)** บน Ethereum

### ความแตกต่างจาก ERC-20

```
ERC-20 (Fungible):
  Alice: 100 USDT
  Bob: 100 USDT
  → ทั้งสองมี USDT เท่ากัน (แลกกันได้)

ERC-721 (Non-Fungible):
  Alice: NFT #1 (ภาพแมว สีฟ้า หมวก ทอง)
  Bob: NFT #2 (ภาพแมว สีดำ หมวก เงิน)
  → แต่ละ NFT ไม่เหมือนกัน ไม่แลกกันได้
```

### Use Cases ของ NFT

```
1. Digital Art - CryptoPunks, BAYC
2. Gaming Items - Axie Infinity, Gods Unchained
3. Music & Media - NFT Albums
4. Virtual Real Estate - Decentraland
5. Membership - DAO Membership Cards
6. Certificates - Degrees, Licenses
7. Tickets - Event Tickets
8. Domain Names - ENS (.eth)
```

### โครงสร้างของ NFT

```
ERC-721 Contract
├── tokenId: uint256 (ไม่ซ้ำกัน)
│   ├── ownerOf(tokenId) → address
│   ├── tokenURI(tokenId) → string
│   └── getApproved(tokenId) → address
│
└── Owner (address)
    ├── balanceOf(owner) → uint256 (จำนวน NFT ที่มี)
    └── isApprovedForAll(owner, operator) → bool
```

---

## 2. ERC-721 Specification {#specification}

### Interface ฉบับสมบูรณ์

```vyper
# @version 0.4.0

interface IERC721:
    # ==================== Events ====================
    event Transfer:
        sender: indexed(address)
        receiver: indexed(address)
        tokenId: indexed(uint256)
    
    event Approval:
        owner: indexed(address)
        approved: indexed(address)
        tokenId: indexed(uint256)
    
    event ApprovalForAll:
        owner: indexed(address)
        operator: indexed(address)
        approved: bool
    
    # ==================== Required Functions ====================
    
    # Query Functions
    def balanceOf(owner: address) -> uint256: view
    def ownerOf(tokenId: uint256) -> address: view
    
    # Transfer Functions
    def safeTransferFrom(
        sender: address,
        receiver: address,
        tokenId: uint256,
        data: Bytes[1024]
    ): nonpayable
    
    def safeTransferFrom(
        sender: address,
        receiver: address,
        tokenId: uint256
    ): nonpayable
    
    def transferFrom(
        sender: address,
        receiver: address,
        tokenId: uint256
    ): nonpayable
    
    # Approval Functions
    def approve(approved: address, tokenId: uint256): nonpayable
    def setApprovalForAll(operator: address, approved: bool): nonpayable
    def getApproved(tokenId: uint256) -> address: view
    def isApprovedForAll(owner: address, operator: address) -> bool: view
    
    # ERC-165
    def supportsInterface(interfaceId: bytes4) -> bool: view

# ERC-721 Metadata Extension (Optional แต่มักใช้เสมอ)
interface IERC721Metadata:
    def name() -> String[100]: view
    def symbol() -> String[32]: view
    def tokenURI(tokenId: uint256) -> String[512]: view

# ERC-721 Enumerable Extension (Optional)
interface IERC721Enumerable:
    def totalSupply() -> uint256: view
    def tokenByIndex(index: uint256) -> uint256: view
    def tokenOfOwnerByIndex(owner: address, index: uint256) -> uint256: view
```

---

## 3. tokenURI และ Metadata {#metadata}

### NFT Metadata Format (JSON)

```json
{
  "name": "CryptoKitty #42",
  "description": "A unique digital cat with special traits",
  "image": "ipfs://QmXxx.../42.png",
  "external_url": "https://mycollection.com/token/42",
  "attributes": [
    {
      "trait_type": "Color",
      "value": "Blue"
    },
    {
      "trait_type": "Rarity",
      "value": "Rare"
    },
    {
      "trait_type": "Level",
      "display_type": "number",
      "value": 5
    }
  ]
}
```

### tokenURI Implementation Patterns

```vyper
# @version 0.4.0

# Pattern 1: Base URI + tokenId
baseURI: String[256]

@external
@view
def tokenURI(tokenId: uint256) -> String[512]:
    assert self._exists(tokenId), "Token does not exist"
    return concat(self.baseURI, uint2str(tokenId))

# Pattern 2: Individual URI per Token
tokenURIs: HashMap[uint256, String[512]]

@external
@view
def tokenURIIndividual(tokenId: uint256) -> String[512]:
    assert self._exists(tokenId), "Token does not exist"
    uri: String[512] = self.tokenURIs[tokenId]
    if len(uri) == 0:
        return concat(self.baseURI, uint2str(tokenId))
    return uri

# Pattern 3: On-chain SVG
@external
@view
def tokenURIOnChain(tokenId: uint256) -> String[1024]:
    assert self._exists(tokenId), "Token does not exist"
    
    # สร้าง SVG on-chain
    svg: String[512] = concat(
        '<svg xmlns="http://www.w3.org/2000/svg" width="200" height="200">',
        '<rect width="200" height="200" fill="blue"/>',
        '<text x="100" y="100" text-anchor="middle" fill="white">',
        uint2str(tokenId),
        '</text></svg>'
    )
    
    return concat(
        "data:image/svg+xml;base64,",
        # In production: base64 encode the svg
        svg
    )

owners: HashMap[uint256, address]

@internal
@view
def _exists(tokenId: uint256) -> bool:
    return self.owners[tokenId] != empty(address)
```

### IPFS Metadata

```vyper
# @version 0.4.0

# ใช้ IPFS CID เป็น Base URI
# ipfs://QmbaseHash/ + tokenId.json

BASE_URI: immutable(String[256])

@deploy
def __init__(baseURI: String[256]):
    BASE_URI = baseURI

@external
@view
def tokenURI(tokenId: uint256) -> String[512]:
    # ipfs://QmXxx.../ + 42 + .json
    return concat(BASE_URI, uint2str(tokenId), ".json")
```

---

## 4. safeTransfer {#safe-transfer}

### ทำไมต้องมี safeTransfer?

```
ปัญหา: ถ้า Transfer NFT ไปยัง Smart Contract ที่ไม่รองรับ NFT
  → NFT จะถูก Lock อยู่ใน Contract ตลอดไป!

safeTransfer แก้ด้วย:
  ถ้า Receiver เป็น Contract → ต้อง Implement IERC721Receiver
  ถ้า Implement → ยืนยันการรับ → Transfer สำเร็จ
  ถ้าไม่ Implement → Revert → NFT ปลอดภัย
```

### IERC721Receiver Interface

```vyper
# @version 0.4.0

# Interface สำหรับ Contract ที่ต้องการรับ NFT
interface IERC721Receiver:
    def onERC721Received(
        operator: address,
        sender: address,
        tokenId: uint256,
        data: Bytes[1024]
    ) -> bytes4: nonpayable

# Magic Return Value ที่ต้องคืน
ERC721_RECEIVED: constant(bytes4) = 0x150b7a02
```

### safeTransfer Implementation

```vyper
# @version 0.4.0

interface IERC721Receiver:
    def onERC721Received(
        operator: address,
        sender: address,
        tokenId: uint256,
        data: Bytes[1024]
    ) -> bytes4: nonpayable

ERC721_RECEIVED: constant(bytes4) = 0x150b7a02

owners: HashMap[uint256, address]
balances: HashMap[address, uint256]

event Transfer:
    sender: indexed(address)
    receiver: indexed(address)
    tokenId: indexed(uint256)

@internal
def _safeTransfer(
    sender: address,
    receiver: address,
    tokenId: uint256,
    data: Bytes[1024]
):
    """Core Safe Transfer Logic"""
    # ทำ Transfer ปกติก่อน
    self._transfer(sender, receiver, tokenId)
    
    # ถ้า Receiver เป็น Contract ต้อง Check
    if receiver.code_size > 0:
        returnValue: bytes4 = IERC721Receiver(receiver).onERC721Received(
            msg.sender,
            sender,
            tokenId,
            data
        )
        assert returnValue == ERC721_RECEIVED, "ERC721: transfer to non ERC721Receiver"

@internal
def _transfer(sender: address, receiver: address, tokenId: uint256):
    """Core Transfer (ไม่ Check Contract)"""
    assert self.owners[tokenId] == sender, "Not token owner"
    assert receiver != empty(address), "Transfer to zero address"
    
    self.balances[sender] -= 1
    self.balances[receiver] += 1
    self.owners[tokenId] = receiver
    
    log Transfer(sender, receiver, tokenId)
```

### Contract ที่ Implement IERC721Receiver

```vyper
# @version 0.4.0
"""
@title NFT Vault
@notice Contract ที่รับ NFT ได้ (Implement IERC721Receiver)
"""

interface IERC721Receiver:
    def onERC721Received(
        operator: address,
        sender: address,
        tokenId: uint256,
        data: Bytes[1024]
    ) -> bytes4: nonpayable

ERC721_RECEIVED: constant(bytes4) = 0x150b7a02

# เก็บ NFT ที่ได้รับ
receivedNFTs: DynArray[(address, uint256), 100]

event NFTReceived:
    operator: indexed(address)
    from_: indexed(address)
    tokenId: indexed(uint256)

@external
def onERC721Received(
    operator: address,
    sender: address,
    tokenId: uint256,
    data: Bytes[1024]
) -> bytes4:
    """
    @notice Handle การรับ NFT
    @dev ต้องคืน ERC721_RECEIVED (0x150b7a02)
    """
    self.receivedNFTs.append((msg.sender, tokenId))
    log NFTReceived(operator, sender, tokenId)
    
    return ERC721_RECEIVED
```

---

## 5. Approval Flows {#approval-flows}

### Single Token Approval

```
Alice: nft.approve(Bob, tokenId=42)
  → getApproved(42) = Bob
  → Bob สามารถ transferFrom(Alice, ..., 42) ได้

Bob: nft.transferFrom(Alice, Charlie, 42)
  → Approval ถูก Clear หลัง Transfer
  → getApproved(42) = 0x0 (ไม่มีคนอนุมัติ)
```

```vyper
# @version 0.4.0

owners: HashMap[uint256, address]
approvals: HashMap[uint256, address]  # tokenId → approved address

event Approval:
    owner: indexed(address)
    approved: indexed(address)
    tokenId: indexed(uint256)

@external
def approve(approved: address, tokenId: uint256):
    """
    @notice อนุมัติให้ approved จัดการ tokenId
    """
    tokenOwner: address = self.owners[tokenId]
    assert tokenOwner != empty(address), "Token does not exist"
    assert msg.sender == tokenOwner or self.isApprovedForAll(tokenOwner, msg.sender), \
        "Not owner or approved for all"
    assert approved != tokenOwner, "Approve to owner"
    
    self.approvals[tokenId] = approved
    log Approval(tokenOwner, approved, tokenId)

@external
@view
def getApproved(tokenId: uint256) -> address:
    assert self.owners[tokenId] != empty(address), "Token does not exist"
    return self.approvals[tokenId]

operatorApprovals: HashMap[address, HashMap[address, bool]]

@external
@view
def isApprovedForAll(owner: address, operator: address) -> bool:
    return self.operatorApprovals[owner][operator]
```

### Approval ถูก Clear หลัง Transfer

```vyper
# @version 0.4.0

approvals: HashMap[uint256, address]
owners: HashMap[uint256, address]
balances: HashMap[address, uint256]

event Transfer:
    sender: indexed(address)
    receiver: indexed(address)
    tokenId: indexed(uint256)

@internal
def _transfer(sender: address, receiver: address, tokenId: uint256):
    assert self.owners[tokenId] == sender
    assert receiver != empty(address)
    
    # Clear Approval เมื่อ Transfer
    if self.approvals[tokenId] != empty(address):
        self.approvals[tokenId] = empty(address)
    
    self.balances[sender] -= 1
    self.balances[receiver] += 1
    self.owners[tokenId] = receiver
    
    log Transfer(sender, receiver, tokenId)
```

---

## 6. Operator Approval {#operator-approval}

### setApprovalForAll

Operator Approval ให้ Contract/Address หนึ่งจัดการ NFT **ทั้งหมด** ของ Owner

```
Use Cases:
  - Marketplace Contract (OpenSea, Blur)
  - Gaming Contract (จัดการ Item ทั้งหมด)
  - Staking Contract
```

```vyper
# @version 0.4.0

operatorApprovals: HashMap[address, HashMap[address, bool]]

event ApprovalForAll:
    owner: indexed(address)
    operator: indexed(address)
    approved: bool

@external
def setApprovalForAll(operator: address, approved: bool):
    """
    @notice อนุมัติ/ยกเลิก Operator สำหรับ NFT ทั้งหมด
    """
    assert operator != msg.sender, "Approve to self"
    assert operator != empty(address), "Zero address"
    
    self.operatorApprovals[msg.sender][operator] = approved
    log ApprovalForAll(msg.sender, operator, approved)

@external
@view
def isApprovedForAll(owner: address, operator: address) -> bool:
    return self.operatorApprovals[owner][operator]
```

### Authorization Check

```vyper
# @version 0.4.0

owners: HashMap[uint256, address]
approvals: HashMap[uint256, address]
operatorApprovals: HashMap[address, HashMap[address, bool]]

@internal
@view
def _isApprovedOrOwner(spender: address, tokenId: uint256) -> bool:
    """
    @notice ตรวจสอบว่า spender มีสิทธิ์จัดการ tokenId
    """
    tokenOwner: address = self.owners[tokenId]
    
    # เป็น Owner ตรงๆ
    if spender == tokenOwner:
        return True
    
    # ได้รับ Approval สำหรับ Token นี้โดยตรง
    if self.approvals[tokenId] == spender:
        return True
    
    # เป็น Operator ที่ได้รับ ApprovalForAll
    if self.operatorApprovals[tokenOwner][spender]:
        return True
    
    return False

@external
def transferFrom(sender: address, receiver: address, tokenId: uint256):
    """Transfer ด้วยการ Check Authorization"""
    assert self._isApprovedOrOwner(msg.sender, tokenId), "Not authorized"
    
    # ทำ Transfer...
    self.owners[tokenId] = receiver
    self.balances[sender] -= 1
    self.balances[receiver] += 1
```

---

## 7. ERC-165 supportsInterface {#erc165}

ERC-165 ช่วยให้ Contract ประกาศว่า Implement Interface อะไรบ้าง

```vyper
# @version 0.4.0

# Interface IDs
ERC165_INTERFACE_ID: constant(bytes4) = 0x01ffc9a7
ERC721_INTERFACE_ID: constant(bytes4) = 0x80ac58cd
ERC721_METADATA_INTERFACE_ID: constant(bytes4) = 0x5b5e139f
ERC721_ENUMERABLE_INTERFACE_ID: constant(bytes4) = 0x780e9d63

@external
@view
def supportsInterface(interfaceId: bytes4) -> bool:
    """
    @notice ตรวจสอบว่า Contract รองรับ Interface ที่ระบุ
    @dev ตาม ERC-165 Standard
    """
    return (
        interfaceId == ERC165_INTERFACE_ID or
        interfaceId == ERC721_INTERFACE_ID or
        interfaceId == ERC721_METADATA_INTERFACE_ID
    )
```

### วิธีคำนวณ Interface ID

```
Interface ID = XOR ของ Function Selectors ทั้งหมดใน Interface

ERC-721 functions:
  balanceOf(address) = 0x70a08231
  ownerOf(uint256) = 0x6352211e
  ...
  
ERC721_INTERFACE_ID = 0x70a08231 XOR 0x6352211e XOR ... = 0x80ac58cd
```

---

## สรุป ERC-721 Standard

### Functions

| Function | Description |
|----------|-------------|
| `balanceOf(owner)` | จำนวน NFT ที่ owner มี |
| `ownerOf(tokenId)` | ผู้ครอง NFT |
| `safeTransferFrom(from, to, tokenId)` | โอนอย่างปลอดภัย |
| `transferFrom(from, to, tokenId)` | โอนตรงๆ |
| `approve(to, tokenId)` | อนุมัติ Single Token |
| `setApprovalForAll(operator, approved)` | อนุมัติ Operator |
| `getApproved(tokenId)` | ดู Approved Address |
| `isApprovedForAll(owner, operator)` | ดู Operator Status |
| `supportsInterface(interfaceId)` | ERC-165 Check |

### Events

| Event | When |
|-------|------|
| `Transfer(from, to, tokenId)` | เมื่อ Transfer, Mint, Burn |
| `Approval(owner, approved, tokenId)` | เมื่อ approve() |
| `ApprovalForAll(owner, operator, approved)` | เมื่อ setApprovalForAll() |

### Metadata Extension

| Function | Description |
|----------|-------------|
| `name()` | ชื่อ Collection |
| `symbol()` | สัญลักษณ์ |
| `tokenURI(tokenId)` | URI ของ Token Metadata |

---

## 8. แบบฝึกหัด {#exercises}

### แบบฝึกหัดที่ 1: NFT Staking
สร้าง Contract ที่:
- รับ NFT เข้า Staking
- ให้ Reward Token ตามระยะเวลา
- ถอน NFT คืนได้

### แบบฝึกหัดที่ 2: NFT Marketplace
สร้าง Marketplace ที่:
- List NFT ขาย
- Buy NFT ด้วย ETH
- Cancel Listing

### แบบฝึกหัดที่ 3: NFT Rental
สร้าง System ที่:
- ให้เช่า NFT
- กำหนดระยะเวลาและค่าเช่า
- ส่งคืน NFT หลังหมดสัญญา

---

[← Part 022: ERC-20 Implementation](part_022_erc20_impl.md) | [Part 024: ERC-721 Implementation →](part_024_erc721_impl.md)
