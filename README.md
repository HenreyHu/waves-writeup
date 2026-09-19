# Wave Maze — Game State Architecture, Keycard System & File I/O

Technical write-up for **Wave Maze**, a top-down maze game built solo/small-team in Year 1 of my
Computer Science degree.

**This is a documentation-only repo.** Wave Maze was a school project, and I don't have clearance
to publish the original coursework source. What's here is my own write-up of the systems I built.

▶ [Portfolio case study for this project](https://HenreyHu.github.io/projects/wave-maze.html) — same content, video-first.

---

## My role

**Programmer.** I built the full game state architecture: the menu flow, the keycard-gated
progression system, collision and spawn logic, and file I/O for saving and loading progress.

## Game state architecture

A single state manager owns transitions between menu, gameplay and pause states, so input
handling, rendering and update logic each only run for the state that's actually active — pausing
the game doesn't require every system to individually check an `isPaused` flag. This was the first
time I'd structured a game this way rather than letting states leak into each other, and it's the
pattern I carried into [STRETCH's](https://github.com/HenreyHu/stretch-writeup) larger system
a year later.

## Keycard progression system

Doors and gates check the player's inventory against a required keycard ID before allowing
passage. Collision and spawn logic for keycards, doors and the player run through the same spatial
system, so adding a new gated area is a data change — placing an object and setting its required
key — rather than new code.

## File I/O and persistence

Save and load covers player position, collected keycards and maze progress, written to disk in a
simple flat format. It's a smaller-scope version of the same problem STRETCH's save system solved
later — deciding what counts as state worth persisting versus what the level itself already
encodes.

## What I'd change

The save format was hand-rolled per project rather than built as something reusable — writing it
again from scratch for STRETCH a year later is what pushed me toward treating serialization as its
own small system rather than page-specific glue code.

---

**Stack:** C++
**Timeframe:** Year 1
