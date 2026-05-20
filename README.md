# Save Future — 2D Platformer Game

<div align="center">

![Save Future Banner](https://img.shields.io/badge/Unity-2D%20Platformer-black?style=for-the-badge&logo=unity&logoColor=white)
![C#](https://img.shields.io/badge/C%23-Game%20Scripts-239120?style=for-the-badge&logo=c-sharp&logoColor=white)
![Platform](https://img.shields.io/badge/Platform-Android%20%7C%20PC-informational?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-In%20Development-yellow?style=for-the-badge)

**A 2D side-scrolling platformer built in Unity — run, dodge, checkpoint, and conquer!**

[▶ View Gameplay Screenshot](https://github.com/Chandan-Baskey/G6-SaveFuture/blob/4fdd91c2778ca0e04b6994404ba588217d9b57e8/Game%20View.jpg) · [📂 Browse Code](https://github.com/Chandan-Baskey/G6-SaveFuture) · [🐛 Report Bug](https://github.com/Chandan-Baskey/G6-SaveFuture/issues)

</div>
![image alt](https://github.com/Chandan-Baskey/SaveFuture-2dGame/blob/ac0e509bcd78b6d6122453bf1601dae654e62ecf/Game%20View.jpg)

## 📖 Table of Contents

- [About the Game](#-about-the-game)
- [Gameplay Features](#-gameplay-features)
- [Game Architecture](#-game-architecture)
- [Script Reference](#-script-reference)
- [Scene Breakdown](#-scene-breakdown)
- [Controls](#-controls)
- [Installation & Setup](#-installation--setup)
- [Project Structure](#-project-structure)
- [System Design](#-system-design)
- [Known Issues & Roadmap](#-known-issues--roadmap)
- [Team](#-team)

---

## 🎮 About the Game

**Save Future** is a 2D platformer game developed in Unity as part of the **G6 team project**. The player controls a character named **Newton**, navigating through a side-scrolling world filled with obstacles, platforms, moving hazards, portals, and checkpoints.

The core mechanic is a **hold-to-run** system — holding the screen/mouse moves the player forward, while releasing decelerates them. The player auto-flips direction on wall contact, creating a ping-pong style traversal challenge that rewards timing and spatial awareness.

The game also includes a **bonus mini-game** (the Apple Catcher) that serves as a standalone arcade mode, accessible from the main menu.

---

## ✨ Gameplay Features

| Feature | Description |
|---|---|
| 🏃 Hold-to-Run Movement | Player accelerates on hold, decelerates on release — smooth velocity blending |
| 🔄 Auto-Wall Flip | Character auto-flips direction upon touching a wall layer |
| 🏁 Checkpoint System | Touch a checkpoint flag to save your position; checkpoint turns active and locks to prevent double-triggering |
| 🌀 Portal Teleportation | Step into a portal and teleport instantly to its paired destination |
| 💀 Die & Respawn | Hit an obstacle → shrink-and-vanish death → respawn at last checkpoint |
| 🎯 Level Progression | Reach the Finish trigger to load the next scene automatically |
| 📷 Smooth Camera | Camera smoothly follows the player with configurable offset and boundary clamps |
| 🍎 Apple Mini-Game | Separate arcade mode — catch/dodge falling apples, track score, quit to main menu |
| 🖥️ Main Menu | Start / Quit buttons with a clean scene-managed UI |

---

## 🏗️ Game Architecture

The game is organized around **five core systems**, each handled by a dedicated MonoBehaviour script:

```
┌─────────────────────────────────────────────────────────┐
│                     SAVE FUTURE                         │
│                  Game Architecture                      │
├──────────────┬──────────────┬──────────────┬────────────┤
│  Input Layer │ Physics Layer│  Game State  │    UI      │
│              │              │   Layer      │   Layer    │
│ PlayerControl│ Rigidbody2D  │ GameControl  │  Canvas    │
│   (hold btn) │  (velocity)  │ (checkpoint/ │  (score /  │
│              │              │  respawn)    │  tap/quit) │
├──────────────┴──────────────┴──────────────┴────────────┤
│              Scene / World Layer                        │
│   Checkpoint · Portal · CameraControl · Gamemanager    │
└─────────────────────────────────────────────────────────┘
```

### Data Flow

```
Mouse/Touch Input
      │
      ▼
PlayerControl.Update()
  btnPressed = GetMouseButton(0)
      │
      ▼
PlayerControl.FixedUpdate()
  speedMultiplier ─── Mathf.MoveTowards ──► velocity applied to Rigidbody2D
      │
      ▼
  WallCheck (OverlapBox)
      │ wall detected
      ▼
  Flip() ─── localScale.x *= -1
      │
      ▼
GameControl.OnTriggerEnter2D()
  ┌── "Obstacle" → Die() → Respawn coroutine → teleport to checkpointPos
  └── "Finish"   → SceneManager.LoadScene(next)
```

---

## 📜 Script Reference

### `PlayerControl.cs` — Player Movement

The heart of the player experience. Implements smooth acceleration/deceleration and automatic wall bouncing.

**Key Fields**

| Field | Type | Purpose |
|---|---|---|
| `speed` | `int` | Base horizontal movement speed |
| `acceleration` | `float` (1–10) | How fast the player speeds up or slows down |
| `wallLayer` | `LayerMask` | Which layer is treated as a wall |
| `wallCheckPoint` | `Transform` | Origin of the wall-detection overlap box |
| `wallCheckSize` | `Vector2` | Dimensions of the overlap box (default `0.06 × 0.8`) |
| `isOnPlatform` | `bool` | Set externally when player is riding a moving platform |
| `platformRb` | `Rigidbody2D` | Platform's Rigidbody; velocity is added to player's when on platform |

**Core Logic**

```csharp
// Smooth speed multiplier: 0 (stopped) → 1 (full speed)
float target = btnPressed ? 1f : 0f;
speedMultiplier = Mathf.MoveTowards(speedMultiplier, target, acceleration * Time.fixedDeltaTime);

// Direction is encoded in the sprite's localScale.x sign
float dir = Mathf.Sign(transform.localScale.x);
float targetSpeed = speed * speedMultiplier * dir;

// Platform velocity stacking
rb.velocity = isOnPlatform
    ? new Vector2(targetSpeed + platformRb.velocity.x, rb.velocity.y)
    : new Vector2(targetSpeed, rb.velocity.y);
```

> ⚠️ **Note:** `UpdateSpeedMultiplier()` is called *and* `Mathf.MoveTowards` is applied in `FixedUpdate`, causing the multiplier to update twice per physics tick. Consider removing the redundant `UpdateSpeedMultiplier()` call for precision.

---

### `GameControl.cs` — Health, Death & Checkpoints

Attached to the **Player** GameObject. Manages the player's checkpoint position, obstacle detection, death animation, and scene transitions.

**Respawn Flow**

```
OnTriggerEnter2D("Obstacle")
        │
        ▼
      Die()
        │
        ▼
  StartCoroutine(Respawn(0.5f))
        │
        ├─ Scale player to (0, 0, 0)   ← "pop" death effect
        ├─ Wait 0.5 seconds
        ├─ Move to checkpointPos
        └─ Scale back to (1, 1, 1)
```

**Public API**

```csharp
// Called by Checkpoint when player steps on it
public void UpdateCheckpoint(Vector2 pos)
{
    checkpointPos = pos;
}
```

---

### `Checkpoint.cs` — Save Points

Placed throughout the level. When triggered by the player:

1. Calls `gameController.UpdateCheckpoint(respawnPoint.position)` — saves the `respawnPoint` transform (not the flag itself) as the respawn position, giving accurate foot-level placement.
2. Swaps the sprite to the **active** visual.
3. Disables its own `Collider2D` to prevent repeated triggering.

```csharp
private void OnTriggerEnter2D(Collider2D collision)
{
    if (collision.CompareTag("Player"))
    {
        gameController.UpdateCheckpoint(respawnPoint.position);
        spriteRenderer.sprite = active;
        coll.enabled = false;
    }
}
```

**Inspector Setup Required**

- `respawnPoint` — assign an empty child Transform positioned where the player should spawn
- `passive` / `active` — two sprites representing the checkpoint states

---

### `Portal.cs` — Teleportation

Pairs two Portal GameObjects. Uses a `HashSet<GameObject>` to prevent the infinite-loop teleport bug (where the object immediately re-enters the destination portal).

```csharp
// On Enter: teleport and register this object at the destination
private void OnTriggerEnter2D(Collider2D collision)
{
    if (portalObjects.Contains(collision.gameObject)) return; // cooldown check

    if (destination.TryGetComponent(out Portal destinationPortal))
        destinationPortal.portalObjects.Add(collision.gameObject); // mark as "just arrived"

    collision.transform.position = destination.position;
}

// On Exit: remove cooldown, allow re-entry
private void OnTriggerExit2D(Collider2D collision)
{
    portalObjects.Remove(collision.gameObject);
}
```

**Inspector Setup Required**

- `destination` — assign the partner portal's `Transform`

---

### `CameraControl.cs` — Smooth Follow Camera

Smoothly follows the player using `Vector3.SmoothDamp` with configurable position offsets and axis boundary clamps.

```csharp
Vector3 targetPosition = target.position + positionOffset;
targetPosition = new Vector3(
    Mathf.Clamp(targetPosition.x, xLimits.x, xLimits.y),
    Mathf.Clamp(targetPosition.y, yLimits.x, yLimits.y),
    -10
);
transform.position = Vector3.SmoothDamp(transform.position, targetPosition, ref velocity, smoothTime);
```

| Inspector Field | Purpose |
|---|---|
| `smoothTime` | Damping factor — 0 = instant snap, 1 = very slow follow |
| `positionOffset` | Offset the camera from the player (e.g. look-ahead bias) |
| `xLimits` / `yLimits` | World-space bounds to prevent camera from leaving the level |

---

### `Gamemanager.cs` — Apple Mini-Game Controller

Manages the standalone Apple Catcher mini-game (Scene: `LvL1`).

- Waits for the first mouse click to start the game
- Hides the "Tap to Start" UI prompt
- Calls `InvokeRepeating` to spawn apples at a configurable `spawnRate`
- Randomly offsets spawn X position between `-maxX` and `+maxX`
- Increments and displays a live score counter

```csharp
private void StartSpawning()
{
    InvokeRepeating("SpawnPos", 0.5f, spawnRate);
}

private void SpawnPos()
{
    Vector3 spawnPos = spawnPoint.position;
    spawnPos.x = Random.Range(-maxX, maxX);
    Instantiate(apple, spawnPos, Quaternion.identity);
    score++;
    scoreText.text = score.ToString();
}
```

---

### `Player.cs` — Apple Catcher Player Controller

Used exclusively in the mini-game. Moves the catcher left/right based on which screen half is tapped.

```csharp
Vector3 touchPos = Camera.main.ScreenToWorldPoint(Input.mousePosition);
if (touchPos.x < 0)
    rb.AddForce(Vector2.left * moveSpeed * Time.deltaTime);
else
    rb.AddForce(Vector2.right * moveSpeed * Time.deltaTime);
```

Colliding with an Apple tagged object reloads the current scene (game over / restart).

---

### `Apple.cs` — Falling Apple Cleanup

Simple self-destruction script on each spawned apple. When `transform.position.y < -3.5f`, the apple destroys itself to keep the scene clean and prevent memory buildup.

---

### `mainscene.cs` — Main Menu Controller

Handles the two buttons on the Main Menu scene:

```csharp
public void startgame() => SceneManager.LoadScene(1); // Load mini-game level
public void stopgame() => Application.Quit();          // Exit application
```

---

## 🗺️ Scene Breakdown

### `MainScene` — Main Menu

| Object | Component | Purpose |
|---|---|---|
| Canvas | `mainscene` script | Hosts START and QUIT buttons |
| Button (Legacy) | Button onClick → `startgame()` | Loads Scene index 1 |
| Button (Legacy) (1) | Button onClick → `stopgame()` | Quits the application |
| Main Camera | Orthographic, size 5 | Renders the menu |

### `LvL1` — Apple Catcher Mini-Game

| Object | Component | Purpose |
|---|---|---|
| Gamemange | `Gamemanager` | Spawn controller and score tracker |
| Newton (Player) | `Player`, Rigidbody2D, CapsuleCollider2D | The catcher basket |
| Apple (prefab) | `Apple` | Falling projectile with self-destruct |
| SpawnPoint | Transform | Spawn origin for apples |
| Canvas | Score TMP, Tap UI, Quit button | HUD |
| Wall | Parent of two BoxCollider2D children | Invisible side walls |
| Main Camera | URP camera, Orthographic | 2D render view |

---

## 🕹️ Controls

| Action | Input |
|---|---|
| Move / Accelerate | Hold Left Mouse Button (or tap on mobile) |
| Stop / Decelerate | Release Mouse Button |
| Navigate Menu | Click START / QUIT buttons |

> The player does **not** have manual jump — vertical physics are handled by gravity and platform collisions. The auto-wall-flip mechanic means the player bounces off walls automatically while the button is held.

---

## 🛠️ Installation & Setup

### Prerequisites

- **Unity 2021.3 LTS** or newer (URP — Universal Render Pipeline)
- **TextMeshPro** package (included in Unity via Package Manager)
- Git (for cloning)

### Steps

```bash
# 1. Clone the repository
git clone https://github.com/Chandan-Baskey/G6-SaveFuture.git

# 2. Open Unity Hub → Add → select the cloned folder

# 3. Unity will import assets and compile scripts automatically

# 4. Open Build Settings (File → Build Settings)
#    Add scenes in this order:
#      0: MainScene
#      1: LvL1

# 5. Press Play in the Editor to test, or Build for your target platform
```

### Scene Order (Build Settings)

| Index | Scene | Description |
|---|---|---|
| 0 | `MainScene` | Main menu with Start / Quit |
| 1 | `LvL1` | Apple Catcher mini-game |

> Additional platformer levels should be added at index 2+ and triggered by `SceneManager.LoadScene(SceneManager.GetActiveScene().buildIndex + 1)` already present in `GameControl`.

---

## 📂 Project Structure

```
G6-SaveFuture/
├── Assets/
│   ├── Scenes/
│   │   ├── MainScene.unity          # Main menu scene
│   │   └── LvL1.unity               # Apple catcher level
│   ├── Scripts/
│   │   ├── PlayerControl.cs         # Platformer player movement
│   │   ├── GameControl.cs           # Death, respawn, checkpoints, scene load
│   │   ├── Checkpoint.cs            # Save point trigger
│   │   ├── Portal.cs                # Teleportation system
│   │   ├── CameraControl.cs         # Smooth camera follow
│   │   ├── Gamemanager.cs           # Apple mini-game controller
│   │   ├── Player.cs                # Mini-game player controller
│   │   ├── Apple.cs                 # Apple self-destruct
│   │   └── mainscene.cs             # Main menu button logic
│   ├── Prefabs/
│   │   └── Apple.prefab             # Falling apple prefab
│   └── Sprites/                     # Game art assets
├── ProjectSettings/
└── README.md
```

---

## 🔧 System Design

### Checkpoint Architecture

```
Checkpoint GameObject
├── SpriteRenderer          (passive sprite by default)
├── Collider2D (IsTrigger)  (disabled after first activation)
└── Checkpoint.cs
    ├── respawnPoint: Transform   (child empty — foot position)
    ├── passive: Sprite
    └── active: Sprite

On Trigger:
    → GameControl.UpdateCheckpoint(respawnPoint.position)
    → sprite = active
    → collider.enabled = false
```

### Portal Pairing

```
Portal A                    Portal B
├── Collider2D (Trigger)    ├── Collider2D (Trigger)
└── Portal.cs               └── Portal.cs
    destination → B.Transform   destination → A.Transform

Anti-loop protection:
  HashSet<GameObject> portalObjects
  OnEnter → add to DESTINATION's set → teleport
  OnExit  → remove from THIS set
```

### Death & Respawn Sequence

```
[Player hits Obstacle]
      │  OnTriggerEnter2D
      ▼
GameControl.Die()
      │  StartCoroutine
      ▼
Respawn(0.5f)
  ├─ localScale = (0,0,0)    ← immediate visual pop
  ├─ yield WaitForSeconds(0.5f)
  ├─ position = checkpointPos
  └─ localScale = (1,1,1)    ← reappear at checkpoint
```

---

## 🐛 Known Issues & Roadmap

### Known Issues

| Issue | Location | Notes |
|---|---|---|
| Double speed multiplier update | `PlayerControl.cs` | `UpdateSpeedMultiplier()` and `MoveTowards` both run in `FixedUpdate` — remove one |
| Score increments on spawn, not on catch | `Gamemanager.cs` | Score goes up when an apple appears, not when caught |
| Apple game player does not stop precisely | `Player.cs` | `rb.velocity = Vector2.zero` creates abrupt stop; consider drag instead |
| Checkpoint uses `transform.position` in commented code | `Checkpoint.cs` | Old commented code — clean up for clarity |

### Roadmap

- [ ] Full platformer levels (beyond the mini-game)
- [ ] Jump mechanic for the main platformer character
- [ ] Animated death / respawn particles
- [ ] Moving platforms with proper `isOnPlatform` detection script
- [ ] Background music and SFX (AudioSource placeholders already commented in code)
- [ ] Mobile touch support improvements
- [ ] Score persistence between sessions (PlayerPrefs)
- [ ] Level select screen

---

## 👥 Team

**Group 6 — Save Future**

| Role | Contributor |
|---|---|
| Project Lead & Core Systems | [Chandan Baskey](https://github.com/Chandan-Baskey) |
| Game Design | G6 Team |
| Level Design | G6 Team |
| UI / UX | G6 Team |

---

## 📄 License

This project was created as an academic/learning game project. All code is open for reference and learning purposes.

---

<div align="center">

Made with ❤️ using **Unity** · **C#** · **URP**

⭐ Star this repo if you found it helpful!

</div>
