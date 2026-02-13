# 04: Evasion and Bypass Techniques

The development of analysis tools and anti-cheat systems is a perpetual "cat-and-mouse" game. For every detection method
an anti-cheat employs, researchers develop a corresponding bypass technique. Understanding these techniques is crucial
for building more robust defenses.

## Bypassing Injection Detection

### The Anti-Cheat's Method: Module Registry Scanning

Anti-cheats use a simple but effective defense: checking the **module registry** (`link_map` on Linux).

```
Anti-Cheat Detection Flow:
┌─────────────────────────────────────┐
│ Scan the official module registry   │
└────────┬────────────────────────────┘
         │
         ↓
    ┌─────────────────────────────────┐
    │ Module Registry                 │
    ├─────────────────────────────────┤
    │ ✓ libUE4.so                     │
    │ ✓ libPhysics.so                 │
    │ ✓ libc.so                       │
    │ ✗ libMyTool.so ◄── UNAUTHORIZED!│
    └─────────────────────────────────┘
         │
         ↓
    CHEATS DETECTED!
```

**How It Works:**

1. Game loads libraries normally using `dlopen()`
2. OS automatically registers them in `link_map` (official module list)
3. AC periodically scans this list
4. Any unauthorized library is immediately flagged
5. Player gets banned

---

### The Bypass: Manual Mapping (The "Hidden Room" Approach)

Instead of using the front door (`dlopen()`), manual mapping builds a secret room outside the official registry.

```
Normal Loading (Detected):             Manual Mapping (Hidden):
┌──────────────────────┐              ┌──────────────────────┐
│ Application Code     │              │ Application Code     │
│ dlopen("lib.so")     │              │ (doesn't call dlopen)│
└──────────────────────┘              └──────────────────────┘
           │                                   │
           │                                   │
           ↓                                   ↓
    ┌────────────────┐                 ┌────────────────┐
    │ OS Linker      │                 │ Custom Code    │
    │ Registers in   │                 │ Allocates raw  │
    │ link_map       │                 │ memory         │
    └────────────────┘                 └────────────────┘
           │                                    │
           ↓                                    ↓
 ┌──────────────────────────┐      ┌──────────────────────────┐
 │ Module Registry          │      │ Module Registry          │
 ├──────────────────────────┤      ├──────────────────────────┤
 │ ✓ libUE4.so              │      │ ✓ libUE4.so              │
 │ ✓ libPhysics.so          │      │ ✓ libPhysics.so          │
 │ ✗ libMyTool.so DETECTED! │      │ (NO ENTRY - HIDDEN!) ✓   │
 └──────────────────────────┘      └──────────────────────────┘
         ❌ DETECTED                       ✅ SAFE
```

**Technical Breakdown:**

| Step       | Normal Loading                    | Manual Mapping                                   |
| ---------- | --------------------------------- | ------------------------------------------------ |
| 1          | Call `dlopen()`                   | Allocate memory with `malloc()`                  |
| 2          | OS loads `.so` from disk          | Manually copy `.so` bytes                        |
| 3          | **OS registers in link_map**      | **Skip registration (no dlopen call)**           |
| 4          | Linker fixes symbols              | Manually patch relocations                       |
| 5          | Execute code                      | Execute code                                     |
| **Result** | Visible in module list → Detected | Hidden from module list → Undetected (simple AC) |

**Key Insight:** Manual mapping bypasses the registry by never calling `dlopen()`. The `.so` file exists in memory and
runs, but the OS kernel has no record of it.

---

### Advanced Detection: Memory Scanning

More sophisticated anti-cheats don't rely on the module registry. Instead, they scan **all executable memory**.

```
Advanced Anti-Cheat Detection:
┌─────────────────────────────────────────┐
│ Scan ALL memory pages for executable    │
│ code that doesn't belong to known       │
│ modules                                 │
└────────────┬────────────────────────────┘
             │
             ↓
    ┌────────────────────────────────┐
    │ Memory Map:                    │
    ├────────────────────────────────┤
    │ 0x40000000 - libUE4.so    ✓    │
    │ 0x50000000 - libPhysics   ✓    │
    │ 0xDEADBEEF - ??? UNKNOWN!!!!   │◄─ Detected!
    │ 0xFEEDB000 - libc.so      ✓    │
    └────────────────────────────────┘
             │
             ↓
        BAN DETECTED!
```

**Why This Is Harder to Evade:**

- Requires scanning entire memory space (CPU intensive)
- Must intelligently identify code patterns
- Performance impact may limit frequency of checks
- Some AC systems don't use this method (older/mobile devices)

## Bypassing Signature Detection

### The Anti-Cheat's Method: Fingerprint Matching

Anti-cheats maintain a database of known tool signatures (unique byte patterns) and scan memory for matches.

```
Anti-Cheat Signature Detection:
┌──────────────────────────┐
│ Known Tool Signatures    │
│ Database                 │
├──────────────────────────┤
│ Tool A: 0x48 0x89 0xE5.. │
│ Tool B: 0x55 0x48 0x8B.. │
│ Tool C: 0x41 0x54 0x41.. │
└──────────────┬───────────┘
               │
               ↓
    ┌─────────────────────────────┐
    │ Scan game memory every N ms │
    └──────┬──────────────────────┘
           │
           ↓
    Game Memory: [0x48 0x89 0xE5 ... ◄─ MATCH FOUND!

    Result: BAN!
```

**How It Works:**

1. AC maintains database of known tool byte patterns
2. Periodically scans game memory
3. Compares memory contents against known signatures
4. Any match triggers detection and ban

---

### The Bypass: Polymorphism & Obfuscation

Instead of having the same bytes every time, the tool changes its appearance on each load.

```
Without Protection (Detected):
Load 1: [0x48 0x89 0xE5 0x48...] ◄─ SIGNATURE MATCH!
Load 2: [0x48 0x89 0xE5 0x48...] ◄─ SIGNATURE MATCH!
Load 3: [0x48 0x89 0xE5 0x48...] ◄─ SIGNATURE MATCH!

❌ Detected on every load


With Polymorphic Obfuscation (Hidden):
Load 1: [0x90 0x48 0x89 0xE5...] ◄─ NO MATCH (junk code added)
Load 2: [0x48 0x99 0x48 0x89...] ◄─ NO MATCH (instruction substituted)
Load 3: [0x48 0x89 0x55 0xE5...] ◄─ NO MATCH (completely different)

✅ Never the same twice - Database signatures useless!
```

**Technical Implementation:**

```
STEP 1: Packing (Encryption)
┌─────────────────────────────┐
│ Real Tool Code              │
│ (The Payload)               │
└────────────┬────────────────┘
             │ Encrypt
             ↓
        ┌────────────────┐
        │ Encrypted Data │
        └────────────────┘
             │ Wrap
             ↓
┌─────────────────────────────┐
│ Stub Loader (Decryptor)     │
│ Encrypted Payload           │
│ (The "Package")             │
└─────────────────────────────┘
        ║ Injected into game
        ║
        ↓ ONLY stub executes first


STEP 2: In-Memory Decryption
┌──────────────────────────────┐
│ Running Stub Code:           │
│ 1. Decrypt payload in memory │
│ 2. Allocate new memory       │
│ 3. Copy decrypted code       │
│ 4. Jump to decrypted code    │
└──────────────────────────────┘
        │ Decryption makes new bytes
        ↓
Decrypted Code (Different each time!) ✅


STEP 3: Polymorphic Mutations
┌─────────────────────────────┐
│ Each Load Has:              │
│ • Different junk code       │
│ • Instruction substitution  │
│ • Register reordering       │
│ • Opcode variants           │
└─────────────────────────────┘
        ║ All performing same function
        ║ but with different bytes
        ↓
Database signatures never match!
```

**Comparison:**

| Method               | Signature           | Result                      |
| -------------------- | ------------------- | --------------------------- |
| Static `.so` on disk | Identical           | ✗ Easily detected           |
| Basic encryption     | All encrypted       | ✓ Hides disk signature      |
| Polymorphic packing  | Different each load | ✓✓ Hides both disk & memory |

**Why It's Effective:**

- Simple AC: Looks for exact byte matches → Can't find anything
- Advanced AC: Must identify code _behavior_ instead of patterns → Much harder
- Polymorphism ensures no two loads are identical

## Bypassing Behavioral Detection

### The Anti-Cheat's Method: Suspicious Activity Monitoring

Advanced anti-cheats operate at high privilege levels (sometimes in the kernel) and monitor **suspicious behavior
patterns**.

```
Behavioral Detection Pattern:
┌────────────────────────────────────────┐
│ Monitor memory access patterns:        │
│ • Which process is accessing data?     │
│ • How frequently? (e.g., 60 times/sec) │
│ • What data is being read?             │
│ • Is it normal game behavior?          │
└────────────┬───────────────────────────┘
             │
             ↓
    ┌──────────────────────────────┐
    │ Detection Rules:             │
    ├──────────────────────────────┤
    │ IF unknown module reads      │
    │ player data 60x/sec          │
    │ AND skips normal rendering   │
    │ AND accesses private memory  │
    │ THEN: BAN!                   │
    └──────────────────────────────┘

    Result: ❌ Detected!
```

**What Anti-Cheats Monitor:**

- **Unknown Code Execution:** Code not in known modules accessing game memory
- **Access Patterns:** Reading entire entity lists every frame (unnatural)
- **Timing:** Reading player positions too frequently or at odd times
- **Data Correlation:** Reading data used for aiming/prediction (suspicious)

```
Normal Player Behavior vs. ESP Behavior:

NORMAL RENDERING:
Game Engine → Render visible players → Draw to screen
(Accesses data only for nearby, visible entities)

ESP BEHAVIOR (Suspicious):
Unknown Code → Read ALL player data → Send externally
(Accesses hidden data, reads everything 60x/sec)
   │
   └─► Easy to detect when monitored!
```

---

### The Bypass: Kernel-Level Operation (The Surveillance Satellite)

To evade monitoring from Ring 3 (user-mode), move the analysis tool to Ring 0 (kernel-mode). From there, memory access
is invisible.

```
Privilege Ring Architecture:

Ring 3 (User Mode - MONITORED):
┌─────────────────────────────────┐
│ Game Process                    │
├─────────────────────────────────┤
│ Anti-Cheat (User-Mode)          │
│ • Can see memory accesses       │
│ • Can monitor behavior          │
└─────────────────────────────────┘
    Can't monitor from here!


Ring 0 (Kernel Mode - UNMONITORED):
┌───────────────────────────────────────┐
│ Operating System Kernel               │
├───────────────────────────────────────┤
│ Custom Malicious Driver               │
│ • Reads memory DIRECTLY (NO SYSCALLS!)│
│ • Bypasses all Ring 3 monitors        │
│ • User-mode AC can't see it           │
└───────────────────────────────────────┘


User-Mode AC Perspective:
┌─────────────────────────────────────┐
│ Monitor memory accesses             │
│ [No suspicious activity detected]   │
│                                     │
│ Why?                                │
│ The kernel is reading memory,       │
│ not the user-mode game!             │
└─────────────────────────────────────┘
    ║
    ║ Meanwhile in Ring 0:
    ║ Kernel driver is silently reading
    ║ all player data!
    ║
     ✅ UNDETECTED!
```

**Technical Details: CPU Privilege Rings**

```
Privilege Escalation:
┌──────────────────────────────────┐
│ Ring 3 (Least Privileged)        │
│ • User applications              │
│ • Game process                   │
│ • User-mode AC                   │
│ • Can't read kernel memory       │
│ • Can't disable protections      │
└──────────────┬───────────────────┘
               │ Kernel module injection
               ↓
┌──────────────────────────────────┐
│ Ring 0 (Most Privileged)         │
│ • Operating system kernel        │
│ • Custom driver code             │
│ • Can read ANY memory            │
│ • Can disable any protection     │
│ • Can't be monitored by Ring 3   │
└──────────────────────────────────┘
```

**Information Flow Comparison:**

```
User-Mode Access (MONITORED):
Game Process
    ↓
Read request: "Get health at 0xAAAA1000"
    ↓ Syscall (System Call - MONITORED)
Kernel
    ↓
Return data
    ↓ Monitored AC detects this!


Kernel-Mode Access (UNMONITORED):
Custom Kernel Driver
    ↓
Direct memory access: "Get health at 0xAAAA1000"
    ↓ No syscall - direct hardware access!
Kernel Memory Manager
    ↓
Return data
    ↓ No syscall means NO DETECTION! ✅
```

**Why This Is the "Ultimate" Evasion:**

| Detection Type      | How It Works                   | Kernel-Mode Status              |
| ------------------- | ------------------------------ | ------------------------------- |
| Module Registry     | Checks `link_map`              | Invisible to `link_map`         |
| Memory Scanning     | Finds executable code          | Code is in kernel (hidden)      |
| Syscall Hooking     | Monitors `read/write` calls    | No syscalls made                |
| Behavioral Analysis | Watches memory access patterns | Access from Ring 0 is invisible |

**Practical Reality:**

- Kernel-level access = Absolute victory in the evasion arms race (for user-space AC)
- However: Modern systems have protections against kernel module injection
- On mobile (Android): SELinux and verified boot make kernel modification nearly impossible
- This explains why mobile anti-cheats are particularly challenging to evade
