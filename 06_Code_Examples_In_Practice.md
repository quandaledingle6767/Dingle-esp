# 06: Code Examples In Practice

This chapter provides a simplified, heavily commented pseudo-code example to illustrate how the concepts from the
previous chapters translate into actual code.

## Core Principles in Code

Before reading the code, remember these two foundational principles:

- **Adding vs. Modifying:** The analysis tool does **not** replace the game's code. It **adds** new code (the `.so`)
  that runs alongside the original code in the same memory space.
- **Reading vs. Leaking:** The tool does not cause a "leak." It actively **reads** data that is already present in the
  client's memory because the game's own rendering engine needs it to be there.

---

## The Game's Code (`Game.cpp`)

This pseudo-code represents the game itself. Its purpose is to establish predictable data structures in memory that an
analysis tool could theoretically target.

```cpp
#include <iostream>
#include <vector>   // The C++ Standard Library for dynamic arrays.
#include <thread>   // The C++ Standard Library for managing threads.

// In C++, a 'struct' is a simple way to group related variables into a single, organized unit.
// This is the "blueprint" for a 3D coordinate.
struct Vector3 {
    float x, y, z;
};

// This is the "blueprint" for a Player object. A reverse engineer must
// discover this exact layout. If they get the order or size of any
// variable wrong, all their memory reads will be incorrect.
struct Player {
    int id;
    int health;
    Vector3 position;
};

// --- THE CRITICAL DATA ---
// We declare a global variable to represent the list of all players.
// 'std::vector' is a dynamic array that can grow as new players join.
// In a real game, this would be a list of POINTERS (Player*), not the objects themselves.
// We imagine that when the game runs, the OS places this list in memory at address 0xABCD0000.
// This runtime address is the primary target of the analysis tool.
std::vector<Player> g_entityList;

int main() {
    // When a match starts, the game populates the list with data from the server.
    g_entityList.push_back({1, 100, {10.f, 20.f, 30.f}}); // The local player
    g_entityList.push_back({2, 80,  {55.f, -12.f, 32.f}}); // An opponent

    // The game then enters its main loop, running its own logic and rendering,
    // completely oblivious to any external tools that may be watching.
    while (true) { /* Game logic runs here... */ }
    return 0;
}
```

---

## The Analysis Tool's Code (`Tool.so.cpp`)

This code is compiled into an `.so` file. After being injected into the game's memory space, it has the same memory
access rights as the game itself.

```cpp
#include <iostream>
#include <vector>
#include <thread>

// The tool MUST replicate the game's data structures exactly. It is defining
// its own local "blueprints" that it will use to interpret the game's memory.
struct Vector3 { float x, y, z; };
struct Player { int id; int health; Vector3 position; };

// This is the main analysis loop. It will run continuously in the background.
void AnalysisLoop() {
    // --- THE CORE OF MEMORY HACKING ---
    // A pointer is a variable that holds a memory address. We declare a pointer
    // named 'p_entityList' that is designed to hold the address of a vector of Players.
    // The C-style cast `(std::vector<Player>*)` is a powerful and dangerous instruction.
    // We are telling the compiler: "I know more than you. I am certain that the raw
    // number 0xABCD0000 is the memory address of a `std::vector<Player>`. Please
    // trust me and treat it as such."
    // In a real tool, 0xABCD0000 would be replaced by the result of a signature scan.
    std::vector<Player>* p_entityList = (std::vector<Player>*)0xABCD0000;

    // The tool enters its own infinite loop, invisible to the game code.
    while (true) {
        if (p_entityList) { // Always check if the pointer is valid!
            // The '*' before 'p_entityList' is the dereference operator. It means
            // "access the actual data that this pointer points to."
            // This loop iterates through the REAL game's entity list.
            for (const auto& player : *p_entityList) {
                // Example logic: ignore the local player and read enemies.
                if (player.id != 1) {
                    // We are now reading memory that belongs to the game process.
                    std::cout << "[TOOL] Reading data for Player " << player.id << std::endl;
                    // In a real ESP, this player data would be sent to an
                    // overlay for rendering.
                }
            }
        }
        // Sleep for a short duration to avoid consuming 100% of the CPU.
        std::this_thread::sleep_for(std::chrono::milliseconds(50));
    }
}

// This is the entry point of the .so file, similar to `main()`. This function
// is the first piece of code to run after the injector successfully loads the library.
extern "C" void StartTool() {
    // This is a CRITICAL step. We must create a new thread for our analysis loop.
    // If we just called AnalysisLoop() directly, we would hijack the game thread that
    // the injector used, which would immediately freeze the game.
    std::thread toolThread(AnalysisLoop);

    // We "detach" the thread. This means we are letting it run completely on its
    // own in the background. We release control of it and don't need to wait for
    // it to finish. It is now a fully independent parasite inside the game process.
    toolThread.detach();
}
```
