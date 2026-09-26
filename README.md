# WAVES — Game State Architecture and Interface Layer

Technical write-up of the systems I built for **WAVES**, a top-down stealth maze game made by a team
of five for CSD1451 at Singapore Institute of Technology and DigiPen Singapore.

You are trapped in a dark maze with restricted vision. Rocks generate waves of light and sound, so
you throw them to reveal the layout ahead and to pull patrolling enemies away from your route. Find
the keycards, reach the exit, do not get caught. Three levels, increasing difficulty.

**My role.** Programmer. I owned the game state architecture and the shared interface layer, which
means the state machine every screen in the game runs through, the fifteen screens built on it, the
three modal overlays, and the button, text and font handling used across all of them.

**Engine.** Built on AlphaEngine, the 2D C engine provided by DigiPen for this course, with FMOD for
audio and FreeType for font rasterisation. The engine was given to us. What is described below is
the game architecture built on top of it, not the engine itself.

---

## Contents

- [Architecture overview](#architecture-overview)
- [1. The state interface](#1-the-state-interface)
- [2. State teardown and ownership](#2-state-teardown-and-ownership)
- [3. States signal, the manager acts](#3-states-signal-the-manager-acts)
- [4. Global bootstrap and the game loop](#4-global-bootstrap-and-the-game-loop)
- [5. Overlays are not states](#5-overlays-are-not-states)
- [6. How a level consumes an overlay](#6-how-a-level-consumes-an-overlay)
- [7. Font caching](#7-font-caching)
- [8. Two coordinate spaces for hit testing](#8-two-coordinate-spaces-for-hit-testing)
- [9. Wall storage and manual growth](#9-wall-storage-and-manual-growth)
- [10. Transitions that compose](#10-transitions-that-compose)
- [What I would do differently](#what-i-would-do-differently)
- [File map](#file-map)
- [Credits](#credits)

---

## Architecture overview

```
Splash
  |
  v
Main Menu ----> Level Select ----> Transition ----> Preview ----> Level 1
  |                                                                 |
  |--> Instructions                                    win ---------|
  |--> Credits                                          |           |--> lose --> Lose State
  |--> Settings (overlay)                               v
  |--> Exit (overlay)                       Transition -> Preview -> Level 2 -> Level 3
                                                                                  |
                                                                                  v
                                                                              Win State
```

Everything on that diagram is a class implementing one interface, driven by one manager. The three
overlays, pause, settings and exit, are deliberately not on it, for reasons in section 5.

---

## 1. The state interface

Every screen in the game implements this.

```cpp
class BaseState {
public:
    virtual ~BaseState() {}                 // Destructor

    virtual void Load() = 0;                // Load assets
    virtual void Initialize() = 0;          // Setup initial state data
    virtual void Update(f32 deltaTime) = 0; // Handle logic updates
    virtual void Draw() = 0;                // Render game objects
    virtual void Free() = 0;                // Free runtime resources
    virtual void Unload() = 0;              // Unload assets
    virtual BaseState* GetNextState() = 0;  // Return the next state
};
```

Seven methods rather than the usual three, and the split is the point. `Load` and `Unload` touch
disk, meaning textures, meshes and audio. `Initialize` and `Free` touch per run data, meaning player
position, timers, flags. Restarting a level runs `Free` then `Initialize` and never re-reads a single
asset from disk.

Making them pure virtual rather than providing empty defaults was deliberate. Four other people on
the team were adding screens. If the base class had supplied a no-op `Unload`, a teammate could
forget to release a texture and nothing would complain until the memory profiler did. Pure virtual
turns that into a compile error.

## 2. State teardown and ownership

```cpp
void GameStateManager::SetState(BaseState* newState) {
    if (currentState) {
        currentState->Free();
        currentState->Unload();
        delete currentState;
    }

    currentState = newState;
    if (currentState) {
        if (auto mm = dynamic_cast<MainMenuState*>(currentState)) {
            mm->SetGameRunningPtr(pGameRunning);
        }

        currentState->Load();
        currentState->Initialize();
    }
}
```

The old state is fully dismantled before the new one is constructed, in the exact reverse of how it
was built. Runtime data first, then assets, then the object.

Ownership lives here and only here. No state deletes itself, and no state deletes another. That
single rule is why the destructor is four lines and why the leak check in `Main.cpp`, the
`_CRTDBG_LEAK_CHECK_DF` flag, came back clean.

The `dynamic_cast` in the middle is the one thing in this file I am not happy with. See
[what I would do differently](#what-i-would-do-differently).

## 3. States signal, the manager acts

This is the central decision of the whole architecture.

```cpp
void GameStateManager::Update(f32 deltaTime) {
    if (!currentState) return;

    currentState->Update(deltaTime);

    BaseState* nextState = currentState->GetNextState();
    if (nextState != nullptr) {
        SetState(nextState);
    }
}
```

A state never calls `SetState`. It returns the screen it wants to go to and the manager does the
switch on the next tick. The alternative, letting a state drive the transition itself, would mean
every state includes `GSM.h`, the manager includes every state, and you have a circular dependency
across fifteen files.

Instead the dependency runs one way. States know nothing about the manager. Here is a level
reporting three different outcomes without any knowledge of who is listening.

```cpp
BaseState* GameState1::GetNextState() {
    if (exitGame) {
        return new MainMenuState();
    }
    if (dead) {
        return new LoseState();
    }
    if (winGame) {
        return new gameTransition(new gamePreview(2));
    }
    return nullptr;  // Stay in the current state if nothing changes
}
```

Returning `nullptr` to mean stay put keeps the common case free. Most frames, for most states, this
function returns immediately.

## 4. Global bootstrap and the game loop

```cpp
int gGameRunning = 1;

// Initialize fonts globally
FontManager::InitializeFonts();

AEAudioGroup keyAudio = AEAudioCreateGroup();  // For background music
AudioManager::LoadAudio("GameOver", "Assets/Audio/GameOver.wav", false, keyAudio);
AudioManager::LoadAudio("Win", "Assets/Audio/Win.wav", false, keyAudio);

GameStateManager gameStateManager;
gameStateManager.SetGameRunningPtr(&gGameRunning); // Provide the pointer to GSM

// Start with the splash state
SplashState* splash = new SplashState();
gameStateManager.SetState(splash);

// reset the system modules
AESysReset();

// Game Loop
while (gGameRunning)
{
    // Informing the system about the loop's start
    AESysFrameStart();
    AEGfxSetRenderMode(AE_GFX_RM_COLOR);

    f32 dt = (f32)AEFrameRateControllerGetFrameTime();
    gameStateManager.Update(dt);
    gameStateManager.Draw();

    if (0 == AESysDoesWindowExist()) {
        gGameRunning = 0;
    }

    // Informing the system about the loop's end
    AESysFrameEnd();
}
```

Fonts and the two result sounds load once, before the loop, and outlive every state. Fonts
especially, because FreeType rasterises a TTF at a specific pixel size and WAVES uses seven of those.
Doing that work inside each state's `Load` would re-rasterise the same typeface every time a player
opened a menu.

The loop body stays at two meaningful calls because the manager absorbs all routing. Adding a screen
never touches this file.

## 5. Overlays are not states

Pause, settings and exit are namespaces, not classes implementing `BaseState`.

```cpp
// Internal variables (file scope)
namespace {
    bool sPaused = false;
    bool sExitToMenu = false;
    s8 sFont = -1;
    s8 mFont = -1;
    AEGfxVertexList* pauseScreenMesh = nullptr;
    AEGfxVertexList* sResumeMesh = nullptr;
    AEGfxVertexList* sMainMenuMesh = nullptr;

    const f32 kButtonW = 300.0f;
    const f32 kButtonH = 100.0f;
    const f32 kResumeY = -120.0f;
    const f32 kResumeX = 0.0f;
    const f32 kMainMenuY = -250.0f;
    const f32 kMainMenuX = 0.0f;
}

namespace PauseOverlay {
    void Init() {
        sPaused = false;
        sExitToMenu = false;

        sFont = FontManager::GetFont("NunitoBold50");
        mFont = FontManager::GetFont("NunitoBold100");

        pauseScreenMesh = CreateButtonMesh(0xFF404040);
        sResumeMesh = CreateButtonMesh(0xFF606060);
        sMainMenuMesh = CreateButtonMesh(0xFF606060);
    }

    void Update() {
        if (!sPaused) return;
        if (AEInputCheckTriggered(AEVK_LBUTTON)) {
            if (IsMouseInsideButton(kResumeX, kResumeY, kButtonW, kButtonH)) {
                sPaused = false;
            }
            else if (IsMouseInsideButton(kMainMenuX, kMainMenuY, kButtonW, kButtonH)) {
                sExitToMenu = true;
                sPaused = false;
            }
        }
    }
}
```

Pausing is not a transition. If pause had been a `BaseState`, entering it would have run
`SetState`, which calls `Free` and `Unload` on the level, destroying the maze, the enemies and the
player position in order to display a menu over them. Coming back would require reloading the level
from disk and losing the player's progress.

An overlay leaves the level fully resident and does two things instead. It gates input and it draws
on top. There is no state change, so there is nothing to rebuild.

The anonymous namespace wrapping the internals gives them internal linkage. `sPaused` and the mesh
pointers are invisible outside this translation unit, so the header exposes only the functions and
there are no extern globals for another file to reach into.

## 6. How a level consumes an overlay

```cpp
// ===========================================
// Pause Overlay
// ===========================================
if (AEInputCheckTriggered(AEVK_ESCAPE) && !PauseOverlay::IsPaused()) {
    PauseOverlay::SetPaused(true);
    AEInputReset();
}

if (PauseOverlay::IsPaused()) {
    PauseOverlay::Update();
    if (PauseOverlay::ShouldExitToMenu()) {
        exitGame = true;
    }
    return;
}
```

The level owns the decision to pause, and the early `return` freezes everything below it while the
state itself stays alive. Draw still runs on the next line of the frame, so the maze remains on
screen underneath the overlay.

`AEInputReset()` is there for a specific bug. Without it the same escape keypress that opened the
overlay was still sitting in the input buffer when the overlay's first `Update` ran, so the menu
opened and closed in a single frame.

## 7. Font caching

```cpp
namespace FontManager {
    // Internal cache for font handles
    static std::unordered_map<std::string, s8> fontCache;

    void InitializeFonts() {
        // Load Nunito-Bold at multiple sizes
        fontCache["NunitoBold50"] = AEGfxCreateFont("Assets/Fonts/Nunito/Nunito-Bold.ttf", 50);
        fontCache["NunitoBold70"] = AEGfxCreateFont("Assets/Fonts/Nunito/Nunito-Bold.ttf", 70);
        fontCache["NunitoBold100"] = AEGfxCreateFont("Assets/Fonts/Nunito/Nunito-Bold.ttf", 100);

        fontCache["NunitoSemiBoldItalic70"] = AEGfxCreateFont("Assets/Fonts/Nunito/Nunito-SemiBoldItalic.ttf", 70);
        fontCache["NunitoMediumItalic100"] = AEGfxCreateFont("Assets/Fonts/Nunito/Nunito-MediumItalic.ttf", 100);
        fontCache["Buggy80"] = AEGfxCreateFont("Assets/buggy-font.ttf", 80);
        fontCache["NosiferRegular150"] = AEGfxCreateFont("Assets/Fonts/Nosifer-Regular.ttf", 150);
    }

    s8 GetFont(const std::string& key) {
        auto it = fontCache.find(key);
        if (it != fontCache.end()) {
            return it->second;
        }
        return -1; // Not found
    }
}
```

An `unordered_map` keyed by string rather than an enum and a fixed array. The trade is a string hash
per lookup against readability at the call site, and `GetFont("NunitoBold50")` says what it wants
where `GetFont(FONT_NUNITO_BOLD_50)` needs an enum kept in sync by hand. Adding a new size is one
line here and nothing anywhere else. The cost is irrelevant at this volume, a handful of lookups per
frame during `Init`, not per glyph.

Returning `-1` for a miss rather than asserting matches AlphaEngine's own convention for an invalid
font handle, so a bad key degrades into text that does not render instead of a crash mid demo.

## 8. Two coordinate spaces for hit testing

```cpp
bool IsMouseInsideButton(f32 buttonX, f32 buttonY, f32 buttonWidth, f32 buttonHeight) {
    s32 mouseX, mouseY;
    AEInputGetCursorPosition(&mouseX, &mouseY);

    // Convert screen space to world space
    f32 worldMouseX = (f32)mouseX - (AEGfxGetWindowWidth() / 2);
    f32 worldMouseY = (AEGfxGetWindowHeight() / 2) - (f32)mouseY;

    // Check if mouse is inside the rectangle
    return (worldMouseX >= buttonX - buttonWidth / 2 &&
        worldMouseX <= buttonX + buttonWidth / 2 &&
        worldMouseY >= buttonY - buttonHeight / 2 &&
        worldMouseY <= buttonY + buttonHeight / 2);
}

bool IsMouseInsideButtonNDC(f32 centerX_NDC, f32 centerY_NDC,
    f32 width_NDC, f32 height_NDC)
{
    // 1. Get mouse in screen coords (0..windowWidth, 0..windowHeight).
    s32 mouseX, mouseY;
    AEInputGetCursorPosition(&mouseX, &mouseY);

    // 2. Convert mouse to normalized device coords (-1..+1).
    f32 halfW = AEGfxGetWindowWidth() * 0.5f;
    f32 halfH = AEGfxGetWindowHeight() * 0.5f;
    f32 mouseXNDC = (mouseX - halfW) / halfW;
    f32 mouseYNDC = (halfH - mouseY) / halfH;

    // 3. Compute the bounding box in NDC.
    f32 left = centerX_NDC - (width_NDC * 0.5f);
    f32 right = centerX_NDC + (width_NDC * 0.5f);
    f32 bottom = centerY_NDC - (height_NDC * 0.5f);
    f32 top = centerY_NDC + (height_NDC * 0.5f);

    // 4. Check if mouse is inside that bounding box.
    return (mouseXNDC >= left && mouseXNDC <= right &&
        mouseYNDC >= bottom && mouseYNDC <= top);
}
```

Both functions flip the Y axis, because AlphaEngine reports the cursor with the origin at the top
left while the game works from the centre outwards.

They exist as a pair because the two kinds of screen want different things. Menu text is laid out in
normalised device coordinates so that a button anchored at `0.0, 0.3` stays proportionally placed at
any window size, and the main menu uses the NDC version for all five of its buttons. Overlays place
their buttons in fixed world units because they sit at known pixel offsets over live gameplay.
Supplying both beats converting at every call site and getting the axis flip wrong in one of them.

## 9. Wall storage and manual growth

```cpp
void addWall(f32 x, f32 y, f32 scale_x, f32 scale_y) {
	wallBlock* temp = (wallBlock*)realloc(wallList, sizeof(wallBlock) * (wallCount + 1));
	if (temp == NULL) {
		// Handle memory allocation failure (e.g., log error and exit)
		fprintf(stderr, "Failed to allocate memory for wallList\n");
		return;
	}
	wallList = temp;

	wallList[wallCount].x = x;
	wallList[wallCount].y = y;
	wallList[wallCount].scale_x = scale_x;
	wallList[wallCount].scale_y = scale_y;
	wallCount++;
}
```

Walls are stored in a manually grown array rather than a `std::vector`, matching the C style of the
engine API this code sits directly on top of.

The `temp` pointer is the part worth pointing at. Writing `wallList = realloc(wallList, ...)`
directly is a common leak. If `realloc` fails it returns null without freeing the original block, so
assigning straight into `wallList` drops the only pointer to memory that is still allocated. Going
through `temp` means a failure leaves the existing wall list intact and the function returns without
corrupting anything.

This is still the system I would most want to revisit. See below.

## 10. Transitions that compose

```cpp
gameTransition::gameTransition(BaseState* nextState)
    : continueGame(false), quitGame(false), transitionFont(-1),
      continueButtonMesh(nullptr), quitButtonMesh(nullptr), nNextState(nextState) {}
```

The transition screen takes its destination as a constructor argument instead of hard coding where it
goes. One class then serves every route in the game rather than writing a separate transition per
pair of screens.

It composes, which is where it pays off. This is the line that runs when a player finishes level one.

```cpp
return new gameTransition(new gamePreview(2));
```

A transition wrapping a preview wrapping level two, assembled at the call site, with the manager
unwrapping one layer per switch.

## What I would do differently

**The `dynamic_cast` in `SetState`.** The manager special cases exactly one state type to hand it the
quit flag, which means generic routing code has to know that `MainMenuState` exists. Every new state
needing shared data would add another branch. The fix is a small context struct holding the quit flag
and anything else global, passed to every state through `Initialize`, so the manager stops knowing
any concrete type at all.

**`realloc` over `std::vector`.** The manual growth in `draw_map.cpp` works and the failure path is
handled, but it reallocates on every single wall added, which is quadratic over a level load, and it
is hand-rolled memory management in a C++ project for no benefit the engine actually required. A
`std::vector` with a `reserve` sized from the maze dimensions would be both faster and shorter.

**Three overlays that are nearly the same file.** Pause, settings and exit each declare their own
font handles, mesh pointers and button constants in near-identical anonymous namespaces. The shared
shape, a modal layer with buttons that gates input and draws over a live state, wants to be one small
overlay base with the three of them supplying only their button lists.

## File map

| File | What it is |
|---|---|
| `baseStates.h` | The state interface every screen implements |
| `GSM.cpp` / `GSM.h` | Game state manager, transitions and ownership |
| `Main.cpp` | Entry point, engine init, global bootstrap, game loop |
| `splash.cpp` | Splash screen, timed advance to the menu |
| `mainMenu.cpp` | Main menu, five buttons with hover states |
| `select.cpp` | Level select |
| `instructions.cpp` | How to play screen |
| `credits.cpp` | Credits screen |
| `gamePreview.cpp` | Per level preview before play |
| `gameTransition.cpp` | Reusable transition wrapping any destination |
| `game1.cpp` / `game2.cpp` / `game3.cpp` | The three playable levels |
| `winState.cpp` / `loseState.cpp` | End of run screens |
| `pauseOverlay.cpp` | Modal pause layer |
| `settingsOverlay.cpp` | Volume and settings layer |
| `exitOverlay.cpp` | Quit confirmation layer |
| `utility.cpp` / `utility.h` | Button hit testing, button meshes, text alignment, FontManager |
| `draw_map.cpp` / `draw_map.h` | Wall and boundary storage and rendering |

I am also credited at fifteen percent on `Audio.cpp` and `Enemy.cpp`.

## Credits

WAVES was built by five people. The systems above are mine. The rest of the game is not, and the
larger parts of it belong to others.

| Area | Author |
|---|---|
| Wave mechanic, the thing the game is named after | Chewn Thing Kwan |
| Maze loading, rendering and map data | Jordain Ng, with Guan Shao Jun |
| Player movement, collision and keycard pickup | Jordain Ng, with Guan Shao Jun |
| Rock throwing | Guan Shao Jun |
| Enemy behaviour | Benjamin Ban, with Jordain Ng and myself |
| Audio system | Jordain Ng, with myself |
| Team lead | Ban Kai Wei Benjamin |

Built on AlphaEngine, provided by DigiPen Institute of Technology Singapore.
Copyright (C) 2025 DigiPen Institute of Technology.
