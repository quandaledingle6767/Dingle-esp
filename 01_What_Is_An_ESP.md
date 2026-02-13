# 01: What Is An ESP?

An ESP (Extra Sensory Perception), in the context of software analysis, is a tool designed to read a host application's
internal data and render it visually as a separate layer. The goal for a researcher is to understand the two fundamental
components of this process: **Data Gathering** and **Visualization**.

## The Two Halves of an ESP

An ESP is best understood as two separate programs working in concert:

1.  **The Reader (`.so` file):** A low-level C++ library that is injected into the game's memory. Its sole purpose is to
    read the game's data, perform calculations (like the World-to-Screen transformation), and then send this processed
    information completely outside of the game.
2.  **The Renderer (Overlay App):** A completely separate, independent, and isolated high-level application. Its only
    job is to receive the stream of data from "The Reader" and draw the appropriate visuals on the screen.

## The Principle of the Detached Overlay

The core design of a modern ESP is based on a clean separation of duties. The high-risk data gathering happens inside
the game, but the visual rendering happens safely outside of it.

**Analogy: Drawing on a Transparency Sheet** Imagine the game is a detailed, physical map spread out on a table. The
overlay app is like taking a clear, plastic transparency sheet and laying it on top of the map. The "Reader" `.so` file
is an agent on the inside who is looking at the map and whispering coordinates and information to you. You then take a
marker and draw on the plastic sheet based on the intel you receive. You are adding visual information without ever
altering the original map underneath.

**The Information Flow**

1.  The injected `.so` file (the "Reader") acts as an internal spy. It reads the raw player data from the game's memory.
2.  It then sends this information "across the border" out of the game's process to the external Renderer app. This
    communication is a form of Inter-Process Communication (IPC).
3.  The Renderer app, which is a completely isolated and independent function, receives this stream of coordinates and
    data.
4.  It then handles the task of drawing the ESP overlays (boxes, lines, health bars) onto its own transparent window,
    which sits on top of the game.

This separation is key. The game process is only compromised by the small, efficient "Reader" whose only job is to
gather data. All the complex and resource-intensive logic for drawing and managing the UI is handled by a separate,
safer application.
