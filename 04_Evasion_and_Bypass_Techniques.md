# 04: Evasion and Bypass Techniques

The development of analysis tools and anti-cheat systems is a perpetual "cat-and-mouse" game. For every detection method
an anti-cheat employs, researchers develop a corresponding bypass technique. Understanding these techniques is crucial
for building more robust defenses.

## Bypassing Injection Detection

**The Anti-Cheat's Method:** The AC acts as a security guard, watching the primary entrances. It monitors the standard
function for loading libraries (`dlopen` on Linux) and regularly checks the building's official tenant registry (the
process's list of loaded modules). If an unauthorized library appears, the alarm is raised.

**The Bypass (Manual Mapping):** This technique is like ignoring the front door and teleporting an agent directly into a
custom-built, hidden room.

- **Analogy:** The AC guard is watching the main entrance and lobby. A researcher using manual mapping doesn't use the
  door. Instead, they use a teleporter to materialize their agent inside a newly created, undocumented room. The guard,
  only watching the official entrances and registry, is completely unaware.
- **Technical Detail:** The injector allocates a new page of memory, copies the `.so` file into it, and executes it.
  Because the standard `dlopen` function was never called, the operating system's linker never adds the library to the
  official list of modules (e.g., the `link_map` in Linux). A simple anti-cheat that just enumerates this list will see
  nothing. While a more advanced AC can still detect this by scanning all memory pages for executable code not belonging
  to a known module, this is a more complex and resource-intensive check.

## Bypassing Signature Detection

**The Anti-Cheat's Method:** The AC works like an antivirus program. It has a large database of "fingerprints" (byte
patterns or signatures) from known tools. It regularly scans the game's memory, and if it finds a match, it triggers a
detection.

**The Bypass (Polymorphism & Obfuscation):** This technique ensures the tool never looks the same way twice.

- **Analogy:** The AC has a photograph of a specific suspect. Obfuscation is like the suspect wearing a simple disguise,
  like a hat and sunglasses. Polymorphism is like the suspect having access to a Hollywood-level makeup artist who gives
  them a completely new, unique face every single day. The original photograph is useless.
- **Technical Detail:** The tool's `.so` file is "packed." The real, functional code (the payload) is encrypted. It is
  wrapped inside a "stub" loader. When the tool is injected, only the stub code runs. Its job is to decrypt the payload
  in memory and then transfer execution to it. This defeats static analysis of the file on disk. A polymorphic packer
  takes this further by ensuring that the stub/decryptor code itself is different on every single load, using techniques
  like junk code insertion and instruction substitution. This makes even in-memory signature scanning incredibly
  difficult.

## Bypassing Behavioral Detection

**The Anti-Cheat's Method:** The most advanced anti-cheats operate at a high privilege level (sometimes in the kernel)
and look for suspicious _behavior_. For example, they can monitor which parts of memory are being read and written to.
If an unknown code module suddenly starts reading player data 60 times a second, it's a massive red flag.

**The Bypass (Kernel-Level Operation):** To bypass a powerful guard, you need to have even more power. This means moving
the analysis tool to the OS kernel.

- **Analogy:** The user-mode AC is a high-tech security guard patrolling _inside_ the building, monitoring all activity.
  The kernel-level tool is a spy watching the building from a satellite with thermal imaging. The guard inside the
  building has no logs, no sensors, and no possible way of knowing that he is being observed from above.
- **Technical Detail:** Modern CPUs have "privilege rings." User applications, including the game and the user-mode part
  of the AC, run in the least privileged ring (Ring 3). The operating system's core, the kernel, runs in the most
  privileged ring (Ring 0). Code in Ring 0 has absolute authority over code in Ring 3. By loading a custom, malicious
  kernel driver, the analysis tool's code can also run in Ring 0. From there, it can read the game's memory directly by
  making requests to the kernel's memory manager. From the perspective of the game process in Ring 3, no suspicious
  memory access ever occurs, rendering the AC's behavioral detection blind. This is the heart of the most powerful
  software-based analysis tools.
