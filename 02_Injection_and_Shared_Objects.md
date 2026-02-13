# 02: Injection and Shared Objects

To analyze a game, a researcher must first get their own code to run inside the target game's process. This is achieved
through **code injection**. The code to be injected is packaged as a **Shared Object (`.so`)** file.

## The Shared Object: A Modular Toolbox

An `.so` file, or **Shared Object**, is a compiled library of C++ code. Think of it as a specialized toolbox. A complex
application like a game is not built as one single, monolithic program. Instead, it's composed of many `.so` files
working together.

- `libUE4.so`: The main Unreal Engine toolbox.
- `libPhysics.so`: A toolbox for handling physics calculations.
- `libRenderer.so`: A toolbox for drawing graphics.

The game loads these toolboxes into memory to perform its functions. The researcher's goal is to create their own custom
toolbox—a new `.so` file containing their analysis code—and force the game to load and use it.

## The Injection Process: Becoming a Parasite

On a standard phone, Android's security model isolates every app in its own "apartment." App A cannot see or affect App
B.

**Analogy: The Master Key and the Secret Room**

1.  **Gaining Root Access:** Getting root on your device is like acquiring the master key to every apartment in the
    building. You now have the authority to bypass the normal security.
2.  **Targeting the Process:** The game is running in its own apartment. The researcher's "loader" tool uses the master
    key to unlock the door to the game's apartment.
3.  **The Injection:** The loader doesn't just enter the apartment; it becomes an architect. It finds an unused
    space—like a thick wall—and secretly builds a new, hidden room inside the game's apartment. It then moves its own
    agent (the analysis `.so` file) into this secret room.

The game continues to operate in its own apartment, completely unaware that there is a new room and a new agent now
living within its walls. This "agent" can now see and hear everything that happens inside the apartment.

### The Technical Principle: Abusing the Debugger's Tool

The "master key" used for this process is a powerful Linux system call called `ptrace` (process trace).

- **Intended Purpose:** `ptrace` is a legitimate tool for developers. It's designed for debugging. It allows a "tracer"
  (the debugger) to attach to a "tracee" (the program being debugged). Once attached, the tracer can pause the program,
  inspect its memory and CPU state, modify it, and then tell it to continue running.
- **Malicious Use:** Code injection is a creative abuse of this debugging functionality. The loader program acts as a
  "debugger" to attach to the game. It uses its `ptrace` privileges not to debug, but to perform the architectural work
  of building the "secret room":
    1.  It pauses the game.
    2.  It commands the game to ask the OS for a new block of memory, marking it as executable.
    3.  It copies the analysis `.so` file byte-by-byte into this new memory block.
    4.  It commands the game to execute the entry point of the new `.so` file, bringing it to life.
    5.  It detaches, leaving the "agent" running inside the game, hidden from plain view.

This process, especially when done without using standard library-loading functions, is known as **Manual Mapping** and
is a cornerstone of advanced analysis and evasion techniques.
