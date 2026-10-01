# Part 009: Arrays และ DynArray

## สารบัญ
1. [Fixed-Size Arrays](#fixed-arrays)
2. [DynArray: Dynamic Arrays](#dynarrays)
3. [Array Operations: append, pop](#operations)
4. [Multi-Dimensional Arrays](#multidim)
5. [Array as Function Parameter/Return](#func-params)
6. [Array Slicing](#slicing)
7. [ตัวอย่าง: Leaderboard Contract](#example)

---

## 1. Fixed-Size Arrays {#fixed-arrays}

Array ขนาดคงที่ที่กำหนดขนาดตอน Compile time

```python
# @version 0.4.0

# ════════════════════════════════════════
# Fixed-Size Arrays: Type[N]
# ════════════════════════════════════════

# Declaration
scores: uint256[5]           # 5 uint256 values
addresses: address[10]       # 10 addresses
flags: bool[3]               # 3 booleans
matrix: uint256[3]           # row of 3

@deploy
def __init__():
    # Initialize with literal values
    self.scores = [100, 95, 88, 72, 60]
    self.addresses[0] = msg.sender
    self.flags = [True, False, True]

@view
@external
def get_score(index: uint256) -> uint256:
    assert index < 5, "Index out of bounds"
    return self.scores[index]

@external
def set_score(index: uint256, value: uint256):
    assert index < 5, "Index out of bounds"
    self.scores[index] = value

@view
@external
def get_all_scores() -> uint256[5]:
    return self.scores

@pure
@external
def sum_fixed(arr: uint256[10]) -> uint256:
    total: uint256 = 0
    for v: uint256 in arr:
        total += v
    return total
```

### Fixed Array ใน Storage vs Memory

```python
# @version 0.4.0

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Storage Arrays
# ━━━━━━━━━━━━━━━━━━━━━━━━━
top_scores: uint256[10]
voters: address[100]

@deploy
def __init__():
    # Initialize storage array
    for i: uint256 in range(10):
        self.top_scores[i] = 0

@external
def update_score(rank: uint256, score: uint256):
    assert rank < 10, "Invalid rank"
    self.top_scores[rank] = score

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Memory (Stack) Arrays
# ━━━━━━━━━━━━━━━━━━━━━━━━━
@pure
@external
def process_locally() -> uint256:
    # Memory array - ไม่ต้อง self.
    temp: uint256[5] = [1, 2, 3, 4, 5]
    total: uint256 = 0
    for v: uint256 in temp:
        total += v
    # Modify
    temp[0] = 100
    return temp[0] + total  # 100 + 15 = 115

@view
@external
def sorted_scores() -> uint256[10]:
    # Copy storage to memory, sort, return
    temp: uint256[10] = self.top_scores

    # Bubble sort
    for i: uint256 in range(10):
        for j: uint256 in range(9, bound=9):
            if j >= 9 - i:
                break
            if temp[j] < temp[j + 1]:
                swap: uint256 = temp[j]
                temp[j] = temp[j + 1]
                temp[j + 1] = swap

    return temp
```

### Array Default Values

```python
# @version 0.4.0

# Default values สำหรับ types ต่างๆ
uint_arr: uint256[5]      # Default: [0, 0, 0, 0, 0]
int_arr: int256[3]        # Default: [0, 0, 0]
bool_arr: bool[4]         # Default: [False, False, False, False]
addr_arr: address[2]      # Default: [0x000...0, 0x000...0]
bytes_arr: bytes32[2]     # Default: [0x000...0, 0x000...0]

@view
@external
def all_zero() -> bool:
    for v: uint256 in self.uint_arr:
        if v != 0:
            return False
    return True

@view
@external
def get_default_uint() -> uint256[5]:
    return self.uint_arr  # [0, 0, 0, 0, 0]
```

---

## 2. DynArray: Dynamic Arrays {#dynarrays}

Array ที่เปลี่ยนขนาดได้ระหว่าง Runtime (แต่มี max size)

```python
# @version 0.4.0

# ════════════════════════════════════════
# DynArray[T, MaxSize]
# ════════════════════════════════════════

# Declaration
participants: DynArray[address, 1000]
prices: DynArray[uint256, 500]
names: DynArray[String[50], 100]

@deploy
def __init__():
    # Start empty - ต่างจาก Fixed Array
    pass

@external
def register():
    assert len(self.participants) < 1000, "Max participants reached"
    assert not self._is_registered(msg.sender), "Already registered"
    self.participants.append(msg.sender)

@view
@internal
def _is_registered(addr: address) -> bool:
    for p: address in self.participants:
        if p == addr:
            return True
    return False

@view
@external
def participant_count() -> uint256:
    return convert(len(self.participants), uint256)

@view
@external
def get_participants() -> DynArray[address, 1000]:
    return self.participants

@view
@external
def get_participant(index: uint256) -> address:
    assert index < convert(len(self.participants), uint256), "Out of bounds"
    return self.participants[index]
```

### len() Function

```python
# @version 0.4.0

items: DynArray[uint256, 100]

@deploy
def __init__():
    self.items.append(10)
    self.items.append(20)
    self.items.append(30)

@view
@external
def get_length() -> uint256:
    return convert(len(self.items), uint256)  # len() returns int128

@view
@external
def is_empty() -> bool:
    return len(self.items) == 0

@view
@external
def is_full() -> bool:
    return len(self.items) == 100

@view
@external
def last_item() -> uint256:
    assert len(self.items) > 0, "Empty array"
    return self.items[len(self.items) - 1]

@view
@external
def slice_last_n(n: uint256) -> DynArray[uint256, 100]:
    assert n <= 100, "n too large"
    result: DynArray[uint256, 100] = []
    total: uint256 = convert(len(self.items), uint256)
    if n > total:
        return self.items
    start: uint256 = total - n
    for i: uint256 in range(start, start + n, bound=100):
        result.append(self.items[i])
    return result
```

---

## 3. Array Operations: append, pop {#operations}

```python
# @version 0.4.0

# ════════════════════════════════════════
# append() and pop()
# ════════════════════════════════════════

queue: DynArray[uint256, 100]
stack: DynArray[uint256, 50]

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# append(): เพิ่มที่ท้าย array
# ━━━━━━━━━━━━━━━━━━━━━━━━━

@deploy
def __init__():
    pass

@external
def enqueue(value: uint256):
    assert len(self.queue) < 100, "Queue full"
    self.queue.append(value)

@external
def push(value: uint256):
    assert len(self.stack) < 50, "Stack full"
    self.stack.append(value)

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# pop(): เอาออกจากท้าย array
# ━━━━━━━━━━━━━━━━━━━━━━━━━

@external
def pop_queue() -> uint256:
    assert len(self.queue) > 0, "Queue empty"
    # pop() เอา element สุดท้ายออก
    last: uint256 = self.queue.pop()
    return last

@external
def pop_stack() -> uint256:
    assert len(self.stack) > 0, "Stack empty"
    return self.stack.pop()

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Queue Simulation (FIFO)
# ━━━━━━━━━━━━━━━━━━━━━━━━━
# หมายเหตุ: Vyper's DynArray.pop() removes the LAST element
# สำหรับ FIFO queue จริงๆ ต้องเก็บ index

actual_queue: DynArray[uint256, 100]
queue_head: uint256  # Index ของหัว queue

@external
def fifo_enqueue(value: uint256):
    assert len(self.actual_queue) < 100, "Queue full"
    self.actual_queue.append(value)

@external
def fifo_dequeue() -> uint256:
    assert self.queue_head < convert(len(self.actual_queue), uint256), "Queue empty"
    value: uint256 = self.actual_queue[self.queue_head]
    self.queue_head += 1
    return value

@view
@external
def queue_size() -> uint256:
    total: uint256 = convert(len(self.actual_queue), uint256)
    if total <= self.queue_head:
        return 0
    return total - self.queue_head
```

### Delete Element (Swap-and-Pop)

```python
# @version 0.4.0

# ════════════════════════════════════════
# Deleting Elements from DynArray
# Swap-and-Pop pattern (O(1) delete)
# ════════════════════════════════════════

items: DynArray[uint256, 200]
item_index: HashMap[uint256, uint256]  # value -> index

@deploy
def __init__():
    pass

@external
def add(value: uint256):
    assert len(self.items) < 200, "Full"
    self.item_index[value] = convert(len(self.items), uint256)
    self.items.append(value)

@external
def remove(value: uint256):
    n: uint256 = convert(len(self.items), uint256)
    assert n > 0, "Empty"

    idx: uint256 = self.item_index[value]
    assert idx < n, "Not found"

    # Swap with last element
    last_val: uint256 = self.items[n - 1]
    self.items[idx] = last_val
    self.item_index[last_val] = idx

    # Pop last
    self.items.pop()
    self.item_index[value] = 0

@view
@external
def contains(value: uint256) -> bool:
    n: uint256 = convert(len(self.items), uint256)
    if n == 0:
        return False
    idx: uint256 = self.item_index[value]
    if idx >= n:
        return False
    return self.items[idx] == value

@view
@external
def get_all() -> DynArray[uint256, 200]:
    return self.items
```

---

## 4. Multi-Dimensional Arrays {#multidim}

```python
# @version 0.4.0

# ════════════════════════════════════════
# Multi-Dimensional Arrays
# ════════════════════════════════════════

# 2D Fixed Array: matrix[rows][cols]
matrix_3x3: uint256[3][3]   # 3 rows, 3 columns
board_8x8: bool[8][8]       # Chess board

@deploy
def __init__():
    # Initialize 3x3 matrix
    self.matrix_3x3 = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]

@view
@external
def get_cell(row: uint256, col: uint256) -> uint256:
    assert row < 3 and col < 3, "Out of bounds"
    return self.matrix_3x3[row][col]

@external
def set_cell(row: uint256, col: uint256, value: uint256):
    assert row < 3 and col < 3, "Out of bounds"
    self.matrix_3x3[row][col] = value

@pure
@external
def multiply_matrices(
    a: uint256[2][2],
    b: uint256[2][2]
) -> uint256[2][2]:
    result: uint256[2][2] = empty(uint256[2][2])
    for i: uint256 in range(2):
        for j: uint256 in range(2):
            sum_val: uint256 = 0
            for k: uint256 in range(2):
                sum_val += a[i][k] * b[k][j]
            result[i][j] = sum_val
    return result

@view
@external
def matrix_diagonal_sum() -> uint256:
    # Sum main diagonal: (0,0) + (1,1) + (2,2)
    return (
        self.matrix_3x3[0][0] +
        self.matrix_3x3[1][1] +
        self.matrix_3x3[2][2]
    )

@view
@external
def matrix_row_sum(row: uint256) -> uint256:
    assert row < 3, "Invalid row"
    total: uint256 = 0
    for v: uint256 in self.matrix_3x3[row]:
        total += v
    return total
```

### 2D DynArray

```python
# @version 0.4.0

# ════════════════════════════════════════
# 2D DynArray
# ════════════════════════════════════════

# Game board: rooms x items
game_inventory: DynArray[DynArray[uint256, 10], 5]  # 5 rooms, 10 items each

@deploy
def __init__():
    # Initialize empty rooms
    for i: uint256 in range(5):
        self.game_inventory.append([])

@external
def add_item_to_room(room: uint256, item_id: uint256):
    assert room < 5, "Invalid room"
    assert len(self.game_inventory[room]) < 10, "Room full"
    self.game_inventory[room].append(item_id)

@view
@external
def get_room_items(room: uint256) -> DynArray[uint256, 10]:
    assert room < 5, "Invalid room"
    return self.game_inventory[room]

@view
@external
def total_items() -> uint256:
    total: uint256 = 0
    for room: DynArray[uint256, 10] in self.game_inventory:
        total += convert(len(room), uint256)
    return total
```

---

## 5. Array as Function Parameter/Return {#func-params}

```python
# @version 0.4.0

# ════════════════════════════════════════
# Arrays as Parameters and Return Values
# ════════════════════════════════════════

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Fixed Arrays as Parameters
# ━━━━━━━━━━━━━━━━━━━━━━━━━

@pure
@external
def sum_five(values: uint256[5]) -> uint256:
    total: uint256 = 0
    for v: uint256 in values:
        total += v
    return total

@pure
@external
def all_nonzero(values: uint256[10]) -> bool:
    for v: uint256 in values:
        if v == 0:
            return False
    return True

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# DynArray as Parameters
# ━━━━━━━━━━━━━━━━━━━━━━━━━

@pure
@external
def sum_dynamic(values: DynArray[uint256, 100]) -> uint256:
    total: uint256 = 0
    for v: uint256 in values:
        total += v
    return total

@pure
@external
def filter_above(
    values: DynArray[uint256, 100],
    threshold: uint256
) -> DynArray[uint256, 100]:
    result: DynArray[uint256, 100] = []
    for v: uint256 in values:
        if v > threshold:
            result.append(v)
    return result

@pure
@external
def unique_addresses(
    addrs: DynArray[address, 50]
) -> DynArray[address, 50]:
    seen: DynArray[address, 50] = []
    for addr: address in addrs:
        found: bool = False
        for s: address in seen:
            if s == addr:
                found = True
                break
        if not found:
            seen.append(addr)
    return seen

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Array Return Values
# ━━━━━━━━━━━━━━━━━━━━━━━━━

prices: DynArray[uint256, 50]

@view
@external
def get_prices() -> DynArray[uint256, 50]:
    return self.prices

@view
@external
def get_price_range() -> (uint256, uint256):
    if len(self.prices) == 0:
        return 0, 0
    min_p: uint256 = self.prices[0]
    max_p: uint256 = self.prices[0]
    for p: uint256 in self.prices:
        if p < min_p:
            min_p = p
        if p > max_p:
            max_p = p
    return min_p, max_p

@view
@external
def get_sorted_prices() -> DynArray[uint256, 50]:
    if len(self.prices) == 0:
        return []
    # Copy to memory
    temp: DynArray[uint256, 50] = self.prices
    n: uint256 = convert(len(temp), uint256)
    # Bubble sort
    for i: uint256 in range(50, bound=50):
        if i >= n:
            break
        for j: uint256 in range(50, bound=50):
            if j >= n - 1 - i:
                break
            if temp[j] > temp[j + 1]:
                swap: uint256 = temp[j]
                temp[j] = temp[j + 1]
                temp[j + 1] = swap
    return temp
```

---

## 6. Array Slicing {#slicing}

```python
# @version 0.4.0

# ════════════════════════════════════════
# Array Slicing
# ════════════════════════════════════════

data: DynArray[uint256, 500]

@deploy
def __init__():
    for i: uint256 in range(100):
        self.data.append(i)

# Vyper ไม่มี slice syntax เหมือน Python (arr[1:5])
# ต้องสร้าง slice function เอง

@view
@external
def slice_array(
    start: uint256,
    length: uint256
) -> DynArray[uint256, 100]:
    assert length <= 100, "Slice too large"
    n: uint256 = convert(len(self.data), uint256)
    assert start < n, "Start out of bounds"

    result: DynArray[uint256, 100] = []
    end: uint256 = start + length
    if end > n:
        end = n

    for i: uint256 in range(start, start + 100, bound=100):
        if i >= end:
            break
        result.append(self.data[i])
    return result

@view
@external
def get_page(page: uint256, page_size: uint256) -> DynArray[uint256, 50]:
    assert page_size <= 50, "Page too large"
    n: uint256 = convert(len(self.data), uint256)
    start: uint256 = page * page_size
    if start >= n:
        return []

    result: DynArray[uint256, 50] = []
    end: uint256 = start + page_size
    if end > n:
        end = n

    for i: uint256 in range(start, start + 50, bound=50):
        if i >= end:
            break
        result.append(self.data[i])
    return result

# Bytes slicing (Vyper has built-in slice for Bytes)
@pure
@external
def extract_bytes(data: Bytes[100], start: uint256, length: uint256) -> Bytes[32]:
    # slice() works on Bytes type
    return slice(data, start, length)
```

---

## 7. ตัวอย่าง: Leaderboard Contract {#example}

Contract จัดการ Leaderboard สำหรับ Game หรือ Competition

```python
# @version 0.4.0

# ════════════════════════════════════════════════════════════
# Leaderboard Contract
# ระบบ Leaderboard แบบ On-Chain สำหรับ Competition
# ════════════════════════════════════════════════════════════

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Constants
# ━━━━━━━━━━━━━━━━━━━━━━━━━
MAX_PLAYERS: constant(uint256) = 1000
TOP_N: constant(uint256) = 10   # Top 10 leaderboard
NAME_MAX_LEN: constant(uint256) = 32

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Structs
# ━━━━━━━━━━━━━━━━━━━━━━━━━
struct PlayerEntry:
    player: address
    score: uint256
    name: String[32]
    games_played: uint256
    last_updated: uint256

struct LeaderboardSnapshot:
    entries: DynArray[PlayerEntry, 10]
    timestamp: uint256
    total_players: uint256

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# State Variables
# ━━━━━━━━━━━━━━━━━━━━━━━━━
owner: address
game_master: address

# Player data
all_players: DynArray[address, 1000]
player_data: HashMap[address, PlayerEntry]
player_exists: HashMap[address, bool]

# Leaderboard cache (sorted top 10)
top_players: DynArray[address, 10]
is_season_active: bool
season_end: uint256

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Events
# ━━━━━━━━━━━━━━━━━━━━━━━━━
event ScoreUpdated:
    player: indexed(address)
    old_score: uint256
    new_score: uint256

event LeaderboardUpdated:
    top_player: indexed(address)
    top_score: uint256

event PlayerRegistered:
    player: indexed(address)
    name: String[32]

event SeasonEnded:
    winner: indexed(address)
    winning_score: uint256

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Constructor
# ━━━━━━━━━━━━━━━━━━━━━━━━━
@deploy
def __init__(game_master: address, season_duration: uint256):
    self.owner = msg.sender
    self.game_master = game_master
    self.is_season_active = True
    self.season_end = block.timestamp + season_duration

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# Internal Functions
# ━━━━━━━━━━━━━━━━━━━━━━━━━

@internal
def _update_top_players(player: address, new_score: uint256):
    """Update the top 10 leaderboard after a score change"""
    # Remove player from top list if present
    temp: DynArray[address, 10] = []
    for p: address in self.top_players:
        if p != player:
            temp.append(p)

    # Find insertion position
    inserted: bool = False
    new_top: DynArray[address, 10] = []

    for p: address in temp:
        if not inserted and new_score >= self.player_data[p].score:
            if len(new_top) < 10:
                new_top.append(player)
            inserted = True
        if len(new_top) < 10:
            new_top.append(p)

    # Append at end if not yet inserted and list has room
    if not inserted and len(new_top) < 10:
        new_top.append(player)

    self.top_players = new_top

    if len(self.top_players) > 0:
        top_addr: address = self.top_players[0]
        log LeaderboardUpdated(top_addr, self.player_data[top_addr].score)

@pure
@internal
def _is_in_top(player: address, top: DynArray[address, 10]) -> bool:
    for p: address in top:
        if p == player:
            return True
    return False

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# External Functions
# ━━━━━━━━━━━━━━━━━━━━━━━━━

@external
def register(name: String[32]):
    """Register as a player"""
    assert self.is_season_active, "Season is over"
    assert not self.player_exists[msg.sender], "Already registered"
    assert len(name) > 0, "Name required"
    assert convert(len(self.all_players), uint256) < MAX_PLAYERS, "Leaderboard full"

    self.player_data[msg.sender] = PlayerEntry({
        player: msg.sender,
        score: 0,
        name: name,
        games_played: 0,
        last_updated: block.timestamp
    })
    self.player_exists[msg.sender] = True
    self.all_players.append(msg.sender)

    log PlayerRegistered(msg.sender, name)

@external
def submit_score(player: address, score: uint256):
    """Submit a score for a player (game master only)"""
    assert msg.sender == self.game_master, "Not game master"
    assert self.is_season_active, "Season is over"
    assert self.player_exists[player], "Player not registered"

    old_score: uint256 = self.player_data[player].score

    # Only update if new score is higher
    if score > old_score:
        self.player_data[player].score = score
        self.player_data[player].last_updated = block.timestamp
        self._update_top_players(player, score)
        log ScoreUpdated(player, old_score, score)

    self.player_data[player].games_played += 1

@external
def batch_submit_scores(
    players: DynArray[address, 50],
    scores: DynArray[uint256, 50]
):
    """Submit scores for multiple players at once"""
    assert msg.sender == self.game_master, "Not game master"
    assert self.is_season_active, "Season is over"
    assert len(players) == len(scores), "Length mismatch"
    assert len(players) <= 50, "Too many players"

    for i: uint256 in range(50, bound=50):
        if i >= convert(len(players), uint256):
            break
        player: address = players[i]
        score: uint256 = scores[i]
        if self.player_exists[player] and score > self.player_data[player].score:
            old: uint256 = self.player_data[player].score
            self.player_data[player].score = score
            self.player_data[player].last_updated = block.timestamp
            self.player_data[player].games_played += 1
            self._update_top_players(player, score)
            log ScoreUpdated(player, old, score)

@external
def end_season():
    """End the current season"""
    assert msg.sender == self.owner, "Not owner"
    assert self.is_season_active, "Season already ended"
    assert block.timestamp >= self.season_end or msg.sender == self.owner

    self.is_season_active = False

    if len(self.top_players) > 0:
        winner: address = self.top_players[0]
        log SeasonEnded(winner, self.player_data[winner].score)

# ━━━━━━━━━━━━━━━━━━━━━━━━━
# View Functions
# ━━━━━━━━━━━━━━━━━━━━━━━━━

@view
@external
def get_leaderboard() -> DynArray[PlayerEntry, 10]:
    """Get top 10 leaderboard"""
    result: DynArray[PlayerEntry, 10] = []
    for addr: address in self.top_players:
        result.append(self.player_data[addr])
    return result

@view
@external
def get_player_rank(player: address) -> uint256:
    """Get player's rank (1-based, 0 if not in top 10)"""
    for i: uint256 in range(10, bound=10):
        if i >= convert(len(self.top_players), uint256):
            break
        if self.top_players[i] == player:
            return i + 1
    return 0  # Not in top 10

@view
@external
def get_snapshot() -> LeaderboardSnapshot:
    """Get current leaderboard snapshot"""
    entries: DynArray[PlayerEntry, 10] = []
    for addr: address in self.top_players:
        entries.append(self.player_data[addr])
    return LeaderboardSnapshot({
        entries: entries,
        timestamp: block.timestamp,
        total_players: convert(len(self.all_players), uint256)
    })

@view
@external
def get_players_above(threshold: uint256) -> DynArray[address, 100]:
    """Get all players above a score threshold"""
    result: DynArray[address, 100] = []
    for addr: address in self.all_players:
        if self.player_data[addr].score >= threshold:
            if len(result) < 100:
                result.append(addr)
    return result

@view
@external
def get_player_info(player: address) -> PlayerEntry:
    assert self.player_exists[player], "Player not found"
    return self.player_data[player]

@view
@external
def total_players() -> uint256:
    return convert(len(self.all_players), uint256)

@view
@external
def get_score_distribution(
    buckets: uint256,
    max_score: uint256
) -> DynArray[uint256, 10]:
    """Count players in each score bucket"""
    assert buckets <= 10 and buckets > 0, "Invalid buckets"
    bucket_size: uint256 = max_score / buckets
    counts: DynArray[uint256, 10] = []

    for i: uint256 in range(10, bound=10):
        if i >= buckets:
            break
        counts.append(0)

    for addr: address in self.all_players:
        score: uint256 = self.player_data[addr].score
        if score < max_score:
            bucket: uint256 = (score * buckets) / max_score
            if bucket < buckets:
                counts[bucket] += 1

    return counts
```

### Test Code

```python
# tests/test_leaderboard.py
import pytest
from brownie import Leaderboard, accounts, chain

SEASON_DURATION = 7 * 24 * 3600  # 1 week

@pytest.fixture
def board(accounts):
    return Leaderboard.deploy(
        accounts[9],  # game master
        SEASON_DURATION,
        {'from': accounts[0]}
    )

@pytest.fixture
def board_with_players(board, accounts):
    for i in range(1, 6):
        board.register(f"Player{i}", {'from': accounts[i]})
    return board

class TestRegistration:
    def test_register(self, board, accounts):
        board.register("Alice", {'from': accounts[1]})
        info = board.get_player_info(accounts[1])
        assert info[2] == "Alice"
        assert info[1] == 0  # score

    def test_duplicate_register(self, board, accounts):
        board.register("Alice", {'from': accounts[1]})
        with pytest.raises(Exception):
            board.register("Alice2", {'from': accounts[1]})

class TestScoring:
    def test_submit_score(self, board_with_players, accounts):
        board_with_players.submit_score(accounts[1], 100, {'from': accounts[9]})
        info = board_with_players.get_player_info(accounts[1])
        assert info[1] == 100

    def test_only_higher_score_counts(self, board_with_players, accounts):
        board_with_players.submit_score(accounts[1], 100, {'from': accounts[9]})
        board_with_players.submit_score(accounts[1], 50, {'from': accounts[9]})
        info = board_with_players.get_player_info(accounts[1])
        assert info[1] == 100  # Still 100, not overwritten by 50

    def test_leaderboard_ordering(self, board_with_players, accounts):
        board_with_players.submit_score(accounts[1], 300, {'from': accounts[9]})
        board_with_players.submit_score(accounts[2], 500, {'from': accounts[9]})
        board_with_players.submit_score(accounts[3], 100, {'from': accounts[9]})

        lb = board_with_players.get_leaderboard()
        # accounts[2] should be first (500 score)
        assert lb[0][0] == accounts[2]  # player address
        assert lb[0][1] == 500  # score

    def test_batch_scores(self, board_with_players, accounts):
        players = [accounts[i] for i in range(1, 4)]
        scores = [100, 200, 300]
        board_with_players.batch_submit_scores(players, scores, {'from': accounts[9]})

        assert board_with_players.get_player_info(accounts[3])[1] == 300

class TestRanking:
    def test_rank_tracking(self, board_with_players, accounts):
        board_with_players.submit_score(accounts[1], 100, {'from': accounts[9]})
        rank = board_with_players.get_player_rank(accounts[1])
        assert rank == 1  # Only player with a score

    def test_not_in_top_10(self, board_with_players, accounts):
        rank = board_with_players.get_player_rank(accounts[1])
        assert rank == 0  # No score yet
```

---

## สรุป

- ✅ **Fixed Array T[N]**: ขนาดคงที่ รู้ตอน Compile time
- ✅ **DynArray[T, MaxN]**: ขนาดเปลี่ยนได้ แต่มี upper bound
- ✅ **append()**: เพิ่ม element ที่ท้าย DynArray
- ✅ **pop()**: เอา element สุดท้ายออก
- ✅ **len()**: ขนาดปัจจุบัน (คืน int128)
- ✅ **Multi-dimensional**: `uint256[3][3]` หรือ `DynArray[DynArray[T, N], M]`
- ✅ **Swap-and-pop**: Pattern สำหรับ O(1) delete จาก array

## แบบฝึกหัด

1. **สร้าง** Priority Queue ที่ใช้ DynArray โดยมี score-based ordering
2. **ปรับปรุง** Leaderboard ให้รองรับ multiple seasons
3. **สร้าง** Merge function ที่รวม 2 sorted arrays
4. **ทดสอบ** Gas ของ append vs direct index assignment
5. **สร้าง** Ring Buffer ด้วย Fixed Array

---

**ก่อนหน้า: [Part 008 - Loops: For Loop](part_008_loops.md)**  
**ต่อไป: [Part 010 - HashMap (Mapping)](part_010_hashmap.md)**
