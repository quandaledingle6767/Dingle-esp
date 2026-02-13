# 05: The Complete A-Z Roadmap

This chapter provides a more detailed project plan for a researcher to analyze a game and develop a proof-of-concept
tool. This is a marathon, not a sprint. Each phase is a significant undertaking that builds upon the last.

## Phase 1: The Laboratory - Environment Setup

**Goal:** To establish a stable and efficient environment for both static and dynamic analysis.

Before a mechanic can fix a car, they must organize their garage and tools.

- **Rooted Android Device:** Your "patient." This provides the privileged access needed for analysis.
- **Android Debug Bridge (ADB):** Your "command and control center." This is how you will install applications, run
  shell commands, pull files, and interact with the device from your computer.
- **Ghidra:** Your "microscope." You will use it to examine the "dead" code—the game's `.so` files—to understand its
  anatomy.
- **Android NDK & C++ IDE:** Your "fabrication workshop." This is where you will write and compile your own C++ analysis
  tool (`.so` file).
- **Android Studio:** Your "design studio" for creating the user-facing overlay application in Java or Kotlin.

## Phase 2: Static Reconnaissance - Deconstructing the Application

**Goal:** To build a "mental model" of the game's code and data structures before ever running it live.

1.  **Extract and Triage Files:** After pulling and unzipping the APK, you will find many `.so` files. Your primary
    target is `libUE4.so`, as it contains the core engine logic. Other libraries may contain game-specific logic.
2.  **Initial Analysis in Ghidra:** This is your primary intelligence-gathering phase. The goal is not to understand
    everything, but to find starting points.
    - **The Symbol Tree:** Look for non-random function names that the developers may have accidentally left in the
      build (e.g., `Player::TakeDamage`). These are gold mines.
    - **String Searches:** Searching for strings like "GWorld", "Health", "PlayerController" can lead you directly to
      the code that uses these critical concepts.
    - **Cross-References (`X-refs`):** Once you find an interesting string or function, you can use Ghidra to find every
      other piece of code that calls or references it. This allows you to build a map of how data flows through the
      application.
    - **The Decompiler:** The decompiler view, which shows C-like pseudo-code, is your best friend. While not perfect,
      it is infinitely more readable than pure Assembly. Your job is to read this output and deduce the original C++
      `class` and `struct` layouts. This is a manual, painstaking process.

## Phase 3: Dynamic Reconnaissance - Live Memory Analysis

**Goal:** To verify the theories from your static analysis with live data from the running game.

1.  **Finding `GWorld`:** Your static analysis should have given you a candidate signature for a function that accesses
    `GWorld`. Now, you write a small C++ program that scans the live game's memory for this signature to find the
    real-time address of `GWorld`. This is your first major breakthrough.
2.  **Verification with a Debugger (GDB):** This is where theory meets reality.
    - You attach GDB to the game process.
    - Using the `GWorld` address you found, you can begin to "walk" through the memory structures. You navigate from
      `GWorld` to the entity list, then to a specific player object.
    - You compare the memory layout you see in the debugger's memory view with the C++ `struct` you designed in your
      header file. Does `health` really appear at `+0x9A0`? Is the next variable a 4-byte float representing the X
      coordinate? This step is filled with trial, error, and refinement.

## Phase 4: Development - Building the Analysis Tool (`.so`)

**Goal:** To write a stable, efficient C++ program that can perform the data gathering automatically.

This is a significant software engineering challenge. Your code must be robust. A single bad pointer or incorrect offset
can crash the entire game.

- **Project Structure:** A clean project might have several files: `proc_mem.cpp` for memory reading/writing functions,
  `scanner.cpp` for signature scanning, `game_structs.h` to define all your reversed structures, and `main.cpp` for the
  entry point.
- **Core Logic:** The main loop must be efficient. It needs to find `GWorld`, get the entity list, and iterate through
  potentially hundreds of objects every frame without causing noticeable performance drops in the game.
- **Mathematical Implementation:** You will implement the 3D math for the World-to-Screen transformation. This involves
  not only the matrix multiplication itself but also finding the View Matrix in memory, which is another dynamic address
  that needs to be found and updated every frame.

## Phase 5: Development - The Overlay

**Goal:** To create a user-friendly and efficient renderer for the data your C++ tool is gathering.

- **IPC Protocol Design:** The biggest challenge here is designing the Inter-Process Communication. How will the C++
  `.so` and the Java/Kotlin app talk? You need to define a "protocol." Will you send a simple comma-separated string? A
  more structured JSON object? Or raw binary structs for maximum performance? This protocol must be fast enough to send
  data for dozens of players every frame without lag.
- **Efficient Drawing:** The `onDraw` method of your overlay needs to be highly optimized. Drawing dozens of boxes,
  lines of text, and health bars 60 times per second can be resource-intensive. Inefficient drawing code will make the
  ESP feel sluggish and unusable.

## Phase 6: Deployment and Execution

**Goal:** To successfully launch all components in the correct order.

- **The Injector:** Writing the `ptrace` injector is a low-level challenge. It must correctly handle different Android
  versions and security features (like SELinux). It needs to find a suitable location in the game's memory to copy the
  `.so` and correctly set memory permissions.
- **The Launch Sequence:** The final process must be reliable: 1) Start the overlay app (which waits for a
  connection). 2) Start the game. 3) Run the injector. If any step fails, the whole system fails.
