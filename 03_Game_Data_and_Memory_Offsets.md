# 03: Game Data and Memory Offsets

Once an analysis tool is running inside the game's memory, its core task begins: finding and understanding the game's
data. This data isn't a chaotic mess; it's highly organized, and comprehending that organization is the key to reading
it.

## Game Data Structures: The Blueprint and the Houses

Games are built using Object-Oriented Programming (OOP). Every entity in the game—a player, a vehicle, a weapon—is an
"object." Each object is created from a C++ `class` or `struct`, which acts as its blueprint.

**Analogy: The Architect's Blueprint**

- A `Player` class is like an architect's blueprint for a house. The blueprint defines the layout: "The front door is at
  the start, a hallway extends 10 feet, the kitchen is at +10 feet, and the master bedroom is at +30 feet."
- Every time a new player joins the match, the game engine acts as a construction company, using the `Player` blueprint
  to build a new house in an empty lot in memory.
- Every house built from this blueprint has the exact same internal layout. The bedroom is always 30 feet from the front
  door. This consistency is what makes offsets work.

## Base Addresses and the Entity List: The Address Book

While every "house" (Player object) has the same internal layout, they are all built in different locations in memory.

- Each player object has its own unique **Base Address**. This is the "street address" of their house (e.g.,
  `0xAAAA1000`).
- The game needs a way to keep track of all the players in the match. It does this using a master list, commonly called
  the **Entity List**.
- This Entity List is not a list of the houses themselves, but a list of their street addresses. It's the **address
  book**. In C++ terms, it's a list of _pointers_ to the player objects (`std::vector<Player*>`).

When the analysis tool wants to find all the players, its first goal is to find this address book.

**Visualizing the Memory Layout**

```
// The Entity List itself lives at a known address (found via signature scan)
Entity List (e.g., at address 0x12345678):
+----------------+
| [0]: 0xAAAA1000 |  <-- This is a POINTER to Player A's memory block
| [1]: 0xBBBB2000 |  <-- This is a POINTER to Player B's memory block
| [2]: 0xCCCC3000 |  <-- This is a POINTER to Player C's memory block
+----------------+
     |
     +----------------------------------------+
                                              |
// Player A's actual data in memory           //
Memory at Base Address 0xAAAA1000:            //
+-----------------------------------+         //
| (Start of the object)             |         //
| ... other data ...                |         //
| + 0x9A0 (health_offset): 100      |         //
| + 0x9A4 (teamID_offset): 4        |         //
+-----------------------------------+         //
                                              |
// Player B's actual data in memory          //
Memory at Base Address 0xBBBB2000: <----------+
+-----------------------------------+
| (Start of the object)             |
| ... other data ...                |
| + 0x9A0 (health_offset): 85       |
| + 0x9A4 (teamID_offset): 7        |
+-----------------------------------+
```

## Offsets and Signatures: The Tools for the Treasure Hunt

A researcher needs two things to read a player's health: the base address of that player and the offset for health.

- **Offsets:** An offset is the distance, in bytes, from the base address of an object to one of its member variables.
  It's the "+30 feet" in the blueprint. The C++ compiler determines these offsets when the game is made. The job of the
  reverse engineer is to analyze the game's executable in Ghidra to re-discover these offsets.
  `Player's Health = Player_Base_Address + health_offset`.

- **Signatures:** Since the address of the Entity List changes every time the game starts, you need a reliable way to
  find it. You can't hardcode its address. However, the game's own code that accesses this list is static. A
  **signature** is a unique "fingerprint" or sequence of bytes from that code. Your tool can search the game's memory
  for this fingerprint. Once found, the code around the signature can be analyzed to reveal the location of the Entity
  List. This is a robust method because code fingerprints are less likely to change between game patches than data
  addresses.

---

## The Process in Action: Finding a Player

This is the practical, two-step process a tool uses to find the dynamic addresses of every player.

### Step 1: Find the "Address Book" (The Entity List) - Done Once

The first action the tool takes upon injection is to find the master Entity List.

1.  The tool comes with a pre-defined "signature" (a byte pattern) that corresponds to a piece of the game's code known
    to be near the Entity List.
2.  It scans the game's memory for this signature. This is a one-time, relatively heavy operation.
3.  Once it finds the signature, it performs a calculation based on the signature's location to get the current,
    real-time memory address of the Entity List.

This complex step is done only once when the tool loads. It now has the address of the "address book" for the rest of
the game session.

### Step 2: Read the "Addresses" from the "Address Book" - Done Every Frame

Now that the tool knows where the Entity List is, its main loop becomes very simple and fast. It no longer needs to scan
memory. Instead, it just reads the contents of the list.

The ESP's main loop continuously repeats the following:

1.  **Go to the known Entity List address.**
2.  **Read the list of pointers.** It iterates through the array, reading each entry.
    - The first entry is Player A's base address.
    - The second entry is Player B's base address.
    - ...and so on for every player in the match.
3.  **Store the fresh addresses.** At the end of the iteration, the tool has an up-to-date list of the exact base
    address for every active player.
4.  **Add offsets.** The tool then uses this list of base addresses, adding the static offsets to each one to read the
    final data (health, position, etc.).

### Summary Flowchart

```
[TOOL INJECTS]
      |
      V
[1. Signature Scan (Heavy Operation)] --> Finds address of Entity List.
(Done only ONCE per match)
      |
      V
[START OF MAIN ESP LOOP]
      |
      V
[2. Read Pointers from Entity List (Lightweight)] --> Gets a fresh list of all player base addresses.
      |
      V
[3. For Each Player Address...] --> Add offsets to read health, position, etc.
      |
      V
[4. Send data to Overlay for Drawing]
      |
      V
[Loop back to Step 2]
```
