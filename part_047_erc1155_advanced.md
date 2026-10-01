# Part 047: Advanced ERC-1155

## สารบัญ
1. [Overview](#overview)
2. [ERC-1155 Standard](#standard)
3. [Semi-Fungible Tokens](#semi-fungible)
4. [Game Items Contract](#game-items)
5. [Marketplace Integration](#marketplace)
6. [Batch Operations](#batch)
7. [Testing](#testing)

---

## 1. Overview {#overview}

ERC-1155 เป็น "Multi Token Standard" ที่รวมข้อดีของ ERC-20 และ ERC-721 เข้าด้วยกัน

### เปรียบเทียบ Token Standards

| Feature | ERC-20 | ERC-721 | ERC-1155 |
|---------|--------|---------|---------|
| Token types | 1 | 1 per contract | หลายประเภท |
| Fungibility | Fungible | Non-fungible | ทั้งสองอย่าง |
| Batch transfer | ไม่มี | ไม่มี | ✅ |
| Gas efficiency | ปานกลาง | ต่ำ | สูง |
| Use case | Currency | Digital art | Games, DeFi |

### Use Cases

- **Gaming**: Sword (100 units), Shield (50 units), Rare Dragon (1 unit)
- **Tickets**: VIP tickets (fungible), Special tickets (non-fungible)
- **DeFi**: LP tokens + NFT positions
- **Collectibles**: Series ที่มีหลาย editions

---

## 2. ERC-1155 Standard {#standard}

```vyper
# @version 0.4.0
# @title BasicERC1155
# @notice ERC-1155 implementation พื้นฐาน

interface IERC1155Receiver:
    def onERC1155Received(
        operator: address,
        frm: address,
        id: uint256,
        value: uint256,
        data: Bytes[1024]
    ) -> bytes4: nonpayable
    
    def onERC1155BatchReceived(
        operator: address,
        frm: address,
        ids: DynArray[uint256, 100],
        values: DynArray[uint256, 100],
        data: Bytes[1024]
    ) -> bytes4: nonpayable

event TransferSingle:
    operator: indexed(address)
    sender: indexed(address)
    receiver: indexed(address)
    id: uint256
    value: uint256

event TransferBatch:
    operator: indexed(address)
    sender: indexed(address)
    receiver: indexed(address)
    ids: DynArray[uint256, 100]
    values: DynArray[uint256, 100]

event ApprovalForAll:
    owner: indexed(address)
    operator: indexed(address)
    approved: bool

event URI:
    value: String[512]
    id: indexed(uint256)

# Interface IDs
INTERFACE_ID_ERC165: constant(bytes4) = 0x01ffc9a7
INTERFACE_ID_ERC1155: constant(bytes4) = 0xd9b67a26
INTERFACE_ID_ERC1155_METADATA: constant(bytes4) = 0x0e89341c

# State
balances: HashMap[uint256, HashMap[address, uint256]]  # id => account => balance
operator_approvals: HashMap[address, HashMap[address, bool]]  # owner => operator => approved
uri: String[512]

@deploy
def __init__(uri: String[512]):
    self.uri = uri

@external
@view
def supportsInterface(interfaceId: bytes4) -> bool:
    return interfaceId in [
        INTERFACE_ID_ERC165,
        INTERFACE_ID_ERC1155,
        INTERFACE_ID_ERC1155_METADATA
    ]

@external
@view
def balanceOf(account: address, id: uint256) -> uint256:
    assert account != empty(address), "Zero address"
    return self.balances[id][account]

@external
@view
def balanceOfBatch(
    accounts: DynArray[address, 100],
    ids: DynArray[uint256, 100]
) -> DynArray[uint256, 100]:
    assert len(accounts) == len(ids), "Length mismatch"
    
    result: DynArray[uint256, 100] = []
    for i: uint256 in range(100):
        if i >= len(accounts):
            break
        result.append(self.balances[ids[i]][accounts[i]])
    
    return result

@external
@view
def isApprovedForAll(owner: address, operator: address) -> bool:
    return self.operator_approvals[owner][operator]

@external
def setApprovalForAll(operator: address, approved: bool):
    assert operator != msg.sender, "Cannot approve self"
    self.operator_approvals[msg.sender][operator] = approved
    log ApprovalForAll(msg.sender, operator, approved)

@internal
def _is_approved_or_owner(operator: address, owner: address) -> bool:
    return operator == owner or self.operator_approvals[owner][operator]

@external
def safeTransferFrom(
    sender: address,
    to: address,
    id: uint256,
    amount: uint256,
    data: Bytes[1024]
):
    assert self._is_approved_or_owner(msg.sender, sender), "Not approved"
    assert to != empty(address), "Zero address"
    assert self.balances[id][sender] >= amount, "Insufficient balance"
    
    self.balances[id][sender] -= amount
    self.balances[id][to] += amount
    
    log TransferSingle(msg.sender, sender, to, id, amount)
    
    if to.is_contract:
        result: bytes4 = IERC1155Receiver(to).onERC1155Received(
            msg.sender, sender, id, amount, data
        )
        assert result == 0xf23a6e61, "Invalid receiver"

@external
def safeBatchTransferFrom(
    sender: address,
    to: address,
    ids: DynArray[uint256, 100],
    amounts: DynArray[uint256, 100],
    data: Bytes[1024]
):
    assert self._is_approved_or_owner(msg.sender, sender), "Not approved"
    assert to != empty(address), "Zero address"
    assert len(ids) == len(amounts), "Length mismatch"
    
    for i: uint256 in range(100):
        if i >= len(ids):
            break
        
        id: uint256 = ids[i]
        amount: uint256 = amounts[i]
        
        assert self.balances[id][sender] >= amount, "Insufficient balance"
        
        self.balances[id][sender] -= amount
        self.balances[id][to] += amount
    
    log TransferBatch(msg.sender, sender, to, ids, amounts)
    
    if to.is_contract:
        result: bytes4 = IERC1155Receiver(to).onERC1155BatchReceived(
            msg.sender, sender, ids, amounts, data
        )
        assert result == 0xbc197c81, "Invalid receiver"
```

---

## 3. Semi-Fungible Tokens {#semi-fungible}

```vyper
# @version 0.4.0
# @title SemiFungibleToken
# @notice Semi-fungible tokens: เป็นทั้ง fungible และ non-fungible

# ===== Token Types =====
# IDs 1-999: Fungible tokens (เหมือน ERC-20)
# IDs 1000-9999: Non-fungible tokens (เหมือน ERC-721)

# Constants
FUNGIBLE_THRESHOLD: constant(uint256) = 1000  # IDs < 1000 are fungible
MAX_NFT_SUPPLY: constant(uint256) = 1000       # Max 1000 NFTs per type

event TransferSingle:
    operator: indexed(address)
    sender: indexed(address)
    receiver: indexed(address)
    id: uint256
    value: uint256

event TransferBatch:
    operator: indexed(address)
    sender: indexed(address)
    receiver: indexed(address)
    ids: DynArray[uint256, 100]
    values: DynArray[uint256, 100]

event ApprovalForAll:
    owner: indexed(address)
    operator: indexed(address)
    approved: bool

struct TokenType:
    is_fungible: bool
    max_supply: uint256
    current_supply: uint256
    name: String[64]
    uri: String[256]

balances: HashMap[uint256, HashMap[address, uint256]]
operator_approvals: HashMap[address, HashMap[address, bool]]
token_types: HashMap[uint256, TokenType]
owner: address
next_nft_id: uint256

@deploy
def __init__():
    self.owner = msg.sender
    self.next_nft_id = FUNGIBLE_THRESHOLD  # NFTs start at 1000

@external
@view
def isFungible(id: uint256) -> bool:
    return id < FUNGIBLE_THRESHOLD

@external
@view
def isNFT(id: uint256) -> bool:
    return id >= FUNGIBLE_THRESHOLD

@external
def createFungibleType(
    id: uint256,
    name: String[64],
    max_supply: uint256,
    uri: String[256]
):
    assert msg.sender == self.owner, "Not owner"
    assert id < FUNGIBLE_THRESHOLD, "Use NFT range"
    assert self.token_types[id].max_supply == 0, "Already exists"
    
    self.token_types[id] = TokenType({
        is_fungible: True,
        max_supply: max_supply,
        current_supply: 0,
        name: name,
        uri: uri
    })

@external
def createNFTType(name: String[64], uri: String[256]) -> uint256:
    assert msg.sender == self.owner, "Not owner"
    
    id: uint256 = self.next_nft_id
    self.next_nft_id += 1
    
    self.token_types[id] = TokenType({
        is_fungible: False,
        max_supply: MAX_NFT_SUPPLY,
        current_supply: 0,
        name: name,
        uri: uri
    })
    
    return id

@external
def mint(to: address, id: uint256, amount: uint256):
    assert msg.sender == self.owner, "Not owner"
    assert to != empty(address), "Zero address"
    
    token_type: TokenType = self.token_types[id]
    assert token_type.max_supply > 0, "Token type not created"
    
    if not token_type.is_fungible:
        assert amount == 1, "NFT: amount must be 1"
    
    assert token_type.current_supply + amount <= token_type.max_supply, "Exceeds max supply"
    
    self.token_types[id].current_supply += amount
    self.balances[id][to] += amount
    
    log TransferSingle(msg.sender, empty(address), to, id, amount)

@external
@view
def balanceOf(account: address, id: uint256) -> uint256:
    return self.balances[id][account]

@external
def safeTransferFrom(
    sender: address,
    to: address,
    id: uint256,
    amount: uint256,
    data: Bytes[1024]
):
    assert msg.sender == sender or self.operator_approvals[sender][msg.sender], "Not approved"
    assert to != empty(address), "Zero address"
    assert self.balances[id][sender] >= amount, "Insufficient"
    
    # NFTs: amount must be 1
    token_type: TokenType = self.token_types[id]
    if not token_type.is_fungible:
        assert amount == 1, "NFT: amount must be 1"
    
    self.balances[id][sender] -= amount
    self.balances[id][to] += amount
    
    log TransferSingle(msg.sender, sender, to, id, amount)

@external
def setApprovalForAll(operator: address, approved: bool):
    assert operator != msg.sender, "Self approval"
    self.operator_approvals[msg.sender][operator] = approved
    log ApprovalForAll(msg.sender, operator, approved)

@external
@view
def isApprovedForAll(owner: address, operator: address) -> bool:
    return self.operator_approvals[owner][operator]
```

---

## 4. GameItems Contract {#game-items}

```vyper
# @version 0.4.0
# @title GameItems
# @notice Full ERC-1155 game items contract

interface IERC1155Receiver:
    def onERC1155Received(
        operator: address,
        frm: address,
        id: uint256,
        value: uint256,
        data: Bytes[1024]
    ) -> bytes4: nonpayable
    
    def onERC1155BatchReceived(
        operator: address,
        frm: address,
        ids: DynArray[uint256, 100],
        values: DynArray[uint256, 100],
        data: Bytes[1024]
    ) -> bytes4: nonpayable

# ===== Events =====
event TransferSingle:
    operator: indexed(address)
    sender: indexed(address)
    receiver: indexed(address)
    id: uint256
    value: uint256

event TransferBatch:
    operator: indexed(address)
    sender: indexed(address)
    receiver: indexed(address)
    ids: DynArray[uint256, 100]
    values: DynArray[uint256, 100]

event ApprovalForAll:
    owner: indexed(address)
    operator: indexed(address)
    approved: bool

event ItemCreated:
    id: indexed(uint256)
    name: String[64]
    item_type: uint256
    max_supply: uint256

event ItemMinted:
    to: indexed(address)
    id: indexed(uint256)
    amount: uint256

event ItemCrafted:
    crafter: indexed(address)
    output_id: indexed(uint256)
    amount: uint256

# ===== Item Types =====
ITEM_TYPE_WEAPON: constant(uint256) = 1
ITEM_TYPE_ARMOR: constant(uint256) = 2
ITEM_TYPE_POTION: constant(uint256) = 3
ITEM_TYPE_MATERIAL: constant(uint256) = 4
ITEM_TYPE_RARE: constant(uint256) = 5

# ===== Interface IDs =====
INTERFACE_ID_ERC165: constant(bytes4) = 0x01ffc9a7
INTERFACE_ID_ERC1155: constant(bytes4) = 0xd9b67a26
INTERFACE_ID_ERC1155_METADATA: constant(bytes4) = 0x0e89341c

# ===== Structs =====
struct ItemConfig:
    name: String[64]
    item_type: uint256
    max_supply: uint256
    current_supply: uint256
    is_tradeable: bool
    is_burnable: bool

struct CraftingRecipe:
    input_ids: DynArray[uint256, 5]
    input_amounts: DynArray[uint256, 5]
    output_id: uint256
    output_amount: uint256
    is_active: bool

# ===== State =====
owner: public(address)
game_master: public(HashMap[address, bool])

balances: HashMap[uint256, HashMap[address, uint256]]
operator_approvals: HashMap[address, HashMap[address, bool]]

items: public(HashMap[uint256, ItemConfig])
item_count: public(uint256)

recipes: HashMap[uint256, CraftingRecipe]  # recipe_id => recipe
recipe_count: public(uint256)

base_uri: public(String[256])

# Player stats
player_level: public(HashMap[address, uint256])
player_experience: public(HashMap[address, uint256])

# ===== Pre-defined Item IDs =====
IRON_SWORD: constant(uint256) = 1
STEEL_SWORD: constant(uint256) = 2
DRAGON_SWORD: constant(uint256) = 3
LEATHER_ARMOR: constant(uint256) = 4
IRON_ARMOR: constant(uint256) = 5
HEALTH_POTION: constant(uint256) = 6
MANA_POTION: constant(uint256) = 7
IRON_ORE: constant(uint256) = 8
STEEL_BAR: constant(uint256) = 9
DRAGON_SCALE: constant(uint256) = 10

@deploy
def __init__(base_uri: String[256]):
    self.owner = msg.sender
    self.game_master[msg.sender] = True
    self.base_uri = base_uri
    self._initialize_default_items()

@internal
def _initialize_default_items():
    """Create default game items"""
    # Weapons
    self.items[IRON_SWORD] = ItemConfig({
        name: "Iron Sword",
        item_type: ITEM_TYPE_WEAPON,
        max_supply: 1000000,
        current_supply: 0,
        is_tradeable: True,
        is_burnable: True
    })
    
    self.items[STEEL_SWORD] = ItemConfig({
        name: "Steel Sword",
        item_type: ITEM_TYPE_WEAPON,
        max_supply: 100000,
        current_supply: 0,
        is_tradeable: True,
        is_burnable: True
    })
    
    self.items[DRAGON_SWORD] = ItemConfig({
        name: "Dragon Sword",
        item_type: ITEM_TYPE_WEAPON,
        max_supply: 1000,  # Rare!
        current_supply: 0,
        is_tradeable: True,
        is_burnable: False  # Too rare to burn
    })
    
    # Armor
    self.items[LEATHER_ARMOR] = ItemConfig({
        name: "Leather Armor",
        item_type: ITEM_TYPE_ARMOR,
        max_supply: 1000000,
        current_supply: 0,
        is_tradeable: True,
        is_burnable: True
    })
    
    self.items[IRON_ARMOR] = ItemConfig({
        name: "Iron Armor",
        item_type: ITEM_TYPE_ARMOR,
        max_supply: 100000,
        current_supply: 0,
        is_tradeable: True,
        is_burnable: True
    })
    
    # Potions
    self.items[HEALTH_POTION] = ItemConfig({
        name: "Health Potion",
        item_type: ITEM_TYPE_POTION,
        max_supply: 10000000,  # Very common
        current_supply: 0,
        is_tradeable: True,
        is_burnable: True
    })
    
    self.items[MANA_POTION] = ItemConfig({
        name: "Mana Potion",
        item_type: ITEM_TYPE_POTION,
        max_supply: 10000000,
        current_supply: 0,
        is_tradeable: True,
        is_burnable: True
    })
    
    # Materials
    self.items[IRON_ORE] = ItemConfig({
        name: "Iron Ore",
        item_type: ITEM_TYPE_MATERIAL,
        max_supply: 100000000,
        current_supply: 0,
        is_tradeable: True,
        is_burnable: True
    })
    
    self.items[STEEL_BAR] = ItemConfig({
        name: "Steel Bar",
        item_type: ITEM_TYPE_MATERIAL,
        max_supply: 50000000,
        current_supply: 0,
        is_tradeable: True,
        is_burnable: True
    })
    
    self.items[DRAGON_SCALE] = ItemConfig({
        name: "Dragon Scale",
        item_type: ITEM_TYPE_RARE,
        max_supply: 5000,
        current_supply: 0,
        is_tradeable: True,
        is_burnable: False
    })
    
    self.item_count = 10

# ===== ERC-165 =====

@external
@view
def supportsInterface(interfaceId: bytes4) -> bool:
    return interfaceId in [
        INTERFACE_ID_ERC165,
        INTERFACE_ID_ERC1155,
        INTERFACE_ID_ERC1155_METADATA
    ]

# ===== ERC-1155 View Functions =====

@external
@view
def balanceOf(account: address, id: uint256) -> uint256:
    assert account != empty(address), "Zero address"
    return self.balances[id][account]

@external
@view
def balanceOfBatch(
    accounts: DynArray[address, 100],
    ids: DynArray[uint256, 100]
) -> DynArray[uint256, 100]:
    assert len(accounts) == len(ids), "Length mismatch"
    
    result: DynArray[uint256, 100] = []
    for i: uint256 in range(100):
        if i >= len(accounts):
            break
        result.append(self.balances[ids[i]][accounts[i]])
    
    return result

@external
@view
def isApprovedForAll(owner: address, operator: address) -> bool:
    return self.operator_approvals[owner][operator]

@external
@view
def uri(id: uint256) -> String[512]:
    # ใน production จะ concat base_uri + id
    return self.base_uri

# ===== Transfer Functions =====

@internal
def _do_safe_transfer_check(
    operator: address,
    sender: address,
    to: address,
    id: uint256,
    amount: uint256,
    data: Bytes[1024]
):
    if to.is_contract:
        result: bytes4 = IERC1155Receiver(to).onERC1155Received(
            operator, sender, id, amount, data
        )
        assert result == 0xf23a6e61, "Unsafe receiver"

@external
def safeTransferFrom(
    sender: address,
    to: address,
    id: uint256,
    amount: uint256,
    data: Bytes[1024]
):
    assert msg.sender == sender or self.operator_approvals[sender][msg.sender], \
        "Not approved"
    assert to != empty(address), "Zero address"
    assert self.balances[id][sender] >= amount, "Insufficient balance"
    
    item: ItemConfig = self.items[id]
    assert item.is_tradeable, "Item not tradeable"
    
    self.balances[id][sender] -= amount
    self.balances[id][to] += amount
    
    log TransferSingle(msg.sender, sender, to, id, amount)
    self._do_safe_transfer_check(msg.sender, sender, to, id, amount, data)

@external
def safeBatchTransferFrom(
    sender: address,
    to: address,
    ids: DynArray[uint256, 100],
    amounts: DynArray[uint256, 100],
    data: Bytes[1024]
):
    assert msg.sender == sender or self.operator_approvals[sender][msg.sender], \
        "Not approved"
    assert to != empty(address), "Zero address"
    assert len(ids) == len(amounts), "Length mismatch"
    
    for i: uint256 in range(100):
        if i >= len(ids):
            break
        
        id: uint256 = ids[i]
        amount: uint256 = amounts[i]
        
        assert self.balances[id][sender] >= amount, "Insufficient balance"
        assert self.items[id].is_tradeable, "Item not tradeable"
        
        self.balances[id][sender] -= amount
        self.balances[id][to] += amount
    
    log TransferBatch(msg.sender, sender, to, ids, amounts)

@external
def setApprovalForAll(operator: address, approved: bool):
    assert operator != msg.sender, "Cannot approve self"
    self.operator_approvals[msg.sender][operator] = approved
    log ApprovalForAll(msg.sender, operator, approved)

# ===== Minting =====

@external
def mint(to: address, id: uint256, amount: uint256):
    assert self.game_master[msg.sender], "Not game master"
    assert to != empty(address), "Zero address"
    
    item: ItemConfig = self.items[id]
    assert item.max_supply > 0, "Item not created"
    assert item.current_supply + amount <= item.max_supply, "Exceeds max supply"
    
    self.items[id].current_supply += amount
    self.balances[id][to] += amount
    
    log TransferSingle(msg.sender, empty(address), to, id, amount)
    log ItemMinted(to, id, amount)

@external
def batchMint(
    to: address,
    ids: DynArray[uint256, 20],
    amounts: DynArray[uint256, 20]
):
    assert self.game_master[msg.sender], "Not game master"
    assert to != empty(address), "Zero address"
    assert len(ids) == len(amounts), "Length mismatch"
    
    for i: uint256 in range(20):
        if i >= len(ids):
            break
        
        id: uint256 = ids[i]
        amount: uint256 = amounts[i]
        
        item: ItemConfig = self.items[id]
        assert item.max_supply > 0, "Item not created"
        assert item.current_supply + amount <= item.max_supply, "Exceeds max supply"
        
        self.items[id].current_supply += amount
        self.balances[id][to] += amount
    
    log TransferBatch(msg.sender, empty(address), to, ids, amounts)

# ===== Burning =====

@external
def burn(id: uint256, amount: uint256):
    assert self.balances[id][msg.sender] >= amount, "Insufficient"
    assert self.items[id].is_burnable, "Item not burnable"
    
    self.balances[id][msg.sender] -= amount
    self.items[id].current_supply -= amount
    
    log TransferSingle(msg.sender, msg.sender, empty(address), id, amount)

# ===== Crafting System =====

@external
def addRecipe(
    input_ids: DynArray[uint256, 5],
    input_amounts: DynArray[uint256, 5],
    output_id: uint256,
    output_amount: uint256
) -> uint256:
    assert msg.sender == self.owner, "Not owner"
    assert len(input_ids) == len(input_amounts), "Length mismatch"
    assert len(input_ids) > 0, "No inputs"
    
    recipe_id: uint256 = self.recipe_count
    self.recipes[recipe_id] = CraftingRecipe({
        input_ids: input_ids,
        input_amounts: input_amounts,
        output_id: output_id,
        output_amount: output_amount,
        is_active: True
    })
    self.recipe_count += 1
    return recipe_id

@external
def craft(recipe_id: uint256, quantity: uint256):
    """Craft items using a recipe"""
    assert quantity > 0, "Quantity must be positive"
    
    recipe: CraftingRecipe = self.recipes[recipe_id]
    assert recipe.is_active, "Recipe not active"
    
    # Check and consume inputs
    for i: uint256 in range(5):
        if i >= len(recipe.input_ids):
            break
        
        input_id: uint256 = recipe.input_ids[i]
        required: uint256 = recipe.input_amounts[i] * quantity
        
        assert self.balances[input_id][msg.sender] >= required, "Insufficient materials"
        self.balances[input_id][msg.sender] -= required
        self.items[input_id].current_supply -= required
        
        log TransferSingle(msg.sender, msg.sender, empty(address), input_id, required)
    
    # Produce output
    output_id: uint256 = recipe.output_id
    output_amount: uint256 = recipe.output_amount * quantity
    
    output_item: ItemConfig = self.items[output_id]
    assert output_item.current_supply + output_amount <= output_item.max_supply, \
        "Output exceeds max supply"
    
    self.items[output_id].current_supply += output_amount
    self.balances[output_id][msg.sender] += output_amount
    
    log TransferSingle(msg.sender, empty(address), msg.sender, output_id, output_amount)
    log ItemCrafted(msg.sender, output_id, output_amount)

# ===== Admin =====

@external
def setGameMaster(account: address, status: bool):
    assert msg.sender == self.owner, "Not owner"
    self.game_master[account] = status

@external
def createItem(
    id: uint256,
    name: String[64],
    item_type: uint256,
    max_supply: uint256,
    is_tradeable: bool,
    is_burnable: bool
):
    assert msg.sender == self.owner, "Not owner"
    assert self.items[id].max_supply == 0, "Item already exists"
    
    self.items[id] = ItemConfig({
        name: name,
        item_type: item_type,
        max_supply: max_supply,
        current_supply: 0,
        is_tradeable: is_tradeable,
        is_burnable: is_burnable
    })
    
    if id >= self.item_count:
        self.item_count = id + 1
    
    log ItemCreated(id, name, item_type, max_supply)

@external
def grantExperience(player: address, amount: uint256):
    assert self.game_master[msg.sender], "Not game master"
    self.player_experience[player] += amount
    
    # Simple leveling: every 1000 XP = 1 level
    new_level: uint256 = self.player_experience[player] // 1000 + 1
    if new_level > self.player_level[player]:
        self.player_level[player] = new_level
```

---

## 5. Testing {#testing}

```python
# tests/test_game_items.py
import pytest
import boa

@pytest.fixture
def deployer():
    return boa.env.generate_address()

@pytest.fixture
def game_master():
    return boa.env.generate_address()

@pytest.fixture
def player1():
    return boa.env.generate_address()

@pytest.fixture
def player2():
    return boa.env.generate_address()

@pytest.fixture
def game(deployer):
    with boa.env.prank(deployer):
        return boa.load(
            "contracts/GameItems.vy",
            "https://api.game.com/items/"
        )

@pytest.fixture
def game_with_master(game, deployer, game_master):
    with boa.env.prank(deployer):
        game.setGameMaster(game_master, True)
    return game

# Item IDs
IRON_SWORD = 1
STEEL_SWORD = 2
DRAGON_SWORD = 3
HEALTH_POTION = 6
IRON_ORE = 8
STEEL_BAR = 9
DRAGON_SCALE = 10


class TestGameItems:
    def test_deployment(self, game, deployer):
        assert game.owner() == deployer
    
    def test_mint_item(self, game_with_master, game_master, player1):
        with boa.env.prank(game_master):
            game_with_master.mint(player1, IRON_SWORD, 5)
        
        assert game_with_master.balanceOf(player1, IRON_SWORD) == 5
    
    def test_batch_mint(self, game_with_master, game_master, player1):
        ids = [IRON_SWORD, HEALTH_POTION, IRON_ORE]
        amounts = [2, 10, 100]
        
        with boa.env.prank(game_master):
            game_with_master.batchMint(player1, ids, amounts)
        
        assert game_with_master.balanceOf(player1, IRON_SWORD) == 2
        assert game_with_master.balanceOf(player1, HEALTH_POTION) == 10
        assert game_with_master.balanceOf(player1, IRON_ORE) == 100
    
    def test_transfer_item(self, game_with_master, game_master, player1, player2):
        with boa.env.prank(game_master):
            game_with_master.mint(player1, IRON_SWORD, 3)
        
        with boa.env.prank(player1):
            game_with_master.safeTransferFrom(player1, player2, IRON_SWORD, 2, b"")
        
        assert game_with_master.balanceOf(player1, IRON_SWORD) == 1
        assert game_with_master.balanceOf(player2, IRON_SWORD) == 2
    
    def test_batch_transfer(self, game_with_master, game_master, player1, player2):
        with boa.env.prank(game_master):
            game_with_master.batchMint(player1, [IRON_SWORD, HEALTH_POTION], [5, 10])
        
        with boa.env.prank(player1):
            game_with_master.safeBatchTransferFrom(
                player1, player2,
                [IRON_SWORD, HEALTH_POTION],
                [2, 5],
                b""
            )
        
        assert game_with_master.balanceOf(player1, IRON_SWORD) == 3
        assert game_with_master.balanceOf(player2, IRON_SWORD) == 2
        assert game_with_master.balanceOf(player2, HEALTH_POTION) == 5
    
    def test_balance_of_batch(self, game_with_master, game_master, player1, player2):
        with boa.env.prank(game_master):
            game_with_master.mint(player1, IRON_SWORD, 5)
            game_with_master.mint(player2, HEALTH_POTION, 10)
        
        balances = game_with_master.balanceOfBatch(
            [player1, player2],
            [IRON_SWORD, HEALTH_POTION]
        )
        
        assert balances[0] == 5
        assert balances[1] == 10
    
    def test_burn_item(self, game_with_master, game_master, player1):
        with boa.env.prank(game_master):
            game_with_master.mint(player1, HEALTH_POTION, 10)
        
        with boa.env.prank(player1):
            game_with_master.burn(HEALTH_POTION, 5)
        
        assert game_with_master.balanceOf(player1, HEALTH_POTION) == 5
    
    def test_cannot_burn_unburnable(self, game_with_master, game_master, player1):
        with boa.env.prank(game_master):
            game_with_master.mint(player1, DRAGON_SWORD, 1)
        
        with pytest.raises(Exception):
            with boa.env.prank(player1):
                game_with_master.burn(DRAGON_SWORD, 1)
    
    def test_crafting_recipe(self, game_with_master, deployer, game_master, player1):
        # Add recipe: 5 Iron Ore + 2 Coal = 1 Steel Bar (recipe_id = 0)
        with boa.env.prank(deployer):
            game_with_master.addRecipe(
                [IRON_ORE],     # inputs
                [5],            # input amounts
                STEEL_BAR,      # output
                1               # output amount
            )
        
        # Give player materials
        with boa.env.prank(game_master):
            game_with_master.mint(player1, IRON_ORE, 15)
        
        # Craft 3 Steel Bars
        with boa.env.prank(player1):
            game_with_master.craft(0, 3)  # recipe_id=0, quantity=3
        
        assert game_with_master.balanceOf(player1, IRON_ORE) == 0
        assert game_with_master.balanceOf(player1, STEEL_BAR) == 3
    
    def test_approval_for_all(self, game_with_master, game_master, player1, player2, deployer):
        with boa.env.prank(game_master):
            game_with_master.mint(player1, IRON_SWORD, 5)
        
        with boa.env.prank(player1):
            game_with_master.setApprovalForAll(deployer, True)
        
        assert game_with_master.isApprovedForAll(player1, deployer) == True
        
        with boa.env.prank(deployer):
            game_with_master.safeTransferFrom(player1, player2, IRON_SWORD, 3, b"")
        
        assert game_with_master.balanceOf(player2, IRON_SWORD) == 3
    
    def test_experience_and_leveling(self, game_with_master, game_master, player1):
        with boa.env.prank(game_master):
            game_with_master.grantExperience(player1, 2500)
        
        assert game_with_master.player_experience(player1) == 2500
        assert game_with_master.player_level(player1) == 3  # 2500 // 1000 + 1 = 3
    
    def test_supports_erc1155_interface(self, game):
        # ERC-1155
        assert game.supportsInterface(bytes.fromhex("d9b67a26"))
        # ERC-165
        assert game.supportsInterface(bytes.fromhex("01ffc9a7"))
    
    def test_non_tradeable_item_cannot_transfer(self, game_with_master, deployer, game_master, player1, player2):
        # Create non-tradeable item
        with boa.env.prank(deployer):
            game_with_master.createItem(
                100, "Soulbound Sword", 1, 1000, False, False
            )
        
        with boa.env.prank(game_master):
            game_with_master.mint(player1, 100, 1)
        
        with pytest.raises(Exception):
            with boa.env.prank(player1):
                game_with_master.safeTransferFrom(player1, player2, 100, 1, b"")


if __name__ == "__main__":
    pytest.main([__file__, "-v"])
```

---

## สรุป

### Key Features ของ GameItems

1. **Full ERC-1155**: ครบถ้วนตาม standard
2. **Multiple Item Types**: weapon, armor, potion, material, rare
3. **Batch Operations**: batch mint, batch transfer, batch balance check
4. **Crafting System**: สร้างไอเทมใหม่จากวัตถุดิบ
5. **Item Properties**: tradeable/non-tradeable, burnable/non-burnable
6. **Player System**: experience และ leveling
7. **Game Master Role**: ผู้ที่มีสิทธิ์ mint items

### Gas Comparison: ERC-1155 vs ERC-721

```
Transfer 10 NFTs:
ERC-721 individual: ~300,000 gas
ERC-1155 batch:     ~100,000 gas
Savings:             ~66%

Mint 100 items:
ERC-721 individual: ~3,000,000 gas
ERC-1155 batch:     ~500,000 gas
Savings:             ~83%
```
