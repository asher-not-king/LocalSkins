# LocalSkins

A purely **client-side** Fabric mod for Minecraft **1.21.4** that lets you (and other players on the same machine) load fully custom PNG skins directly from your local filesystem — no server installation required, works in singleplayer and on any multiplayer server.

---

## Table of Contents

- [Features](#features)
- [Requirements](#requirements)
- [Installation](#installation)
- [Setting Up Custom Skins](#setting-up-custom-skins)
  - [Skin File Format](#skin-file-format)
  - [Skin Model Types](#skin-model-types)
  - [Directory Structure](#directory-structure)
- [How It Works](#how-it-works)
- [Fallback Behaviour](#fallback-behaviour)
- [Multiplayer Compatibility](#multiplayer-compatibility)
- [Building from Source](#building-from-source)
- [Project Structure](#project-structure)
- [Troubleshooting](#troubleshooting)
- [Known Limitations](#known-limitations)
- [License](#license)

---

## Features

- 🎨 Load any standard Minecraft-format PNG skin from disk at runtime
- 👤 Per-player skin overrides keyed by in-game username
- ⚡ Textures are cached after the first load — zero repeated disk I/O per frame
- 🌐 Fully client-side — works on any server, vanilla or modded, with zero server-side installation
- 🔁 Automatic fallback to the player's default Mojang skin when no local file is found
- 🔒 No network requests, no external services — your custom skin never leaves your machine

---

## Requirements

| Dependency | Version |
|---|---|
| Minecraft | **1.21.4** |
| Fabric Loader | **≥ 0.16.9** |
| Fabric API | **0.119.4+1.21.4** (or newer compatible) |
| Java | **21** |

---

## Installation

1. Make sure you have **Fabric Loader ≥ 0.16.9** installed for Minecraft 1.21.4.  
   → [fabric.io/use](https://fabricmc.net/use/)

2. Download **Fabric API** (`fabric-api-0.119.4+1.21.4.jar` or newer) and place it in your `mods/` folder.  
   → [Fabric API on Modrinth](https://modrinth.com/mod/fabric-api)

3. Place the `localskins-1.0.0.jar` file into your `.minecraft/mods/` folder.

4. Launch the game once to let it generate the config directory, then proceed to [Setting Up Custom Skins](#setting-up-custom-skins).

---

## Setting Up Custom Skins

### Skin File Format

Custom skin files must be:

| Property | Requirement |
|---|---|
| Format | **PNG** |
| Dimensions | **64 × 64 pixels** (modern layout) |
| File name | Exactly the player's **in-game username** (case-sensitive), e.g. `Notch.png` |
| Location | `.minecraft/config/localskins/` |

> ⚠️ **The filename is case-sensitive and must match the player's username exactly.**  
> `notch.png` will NOT match a player named `Notch`.

### Skin Model Types

Minecraft supports two skin models:

| Model | Description |
|---|---|
| `default` (Steve) | Classic 4-pixel wide arms |
| `slim` (Alex) | Thinner 3-pixel wide arms |

The mod preserves whatever model type is already configured on the player's Mojang account. If you want a `slim`-model skin, make sure your Mojang account is set to the slim model at [minecraft.net](https://www.minecraft.net/en-us/profile/skin), as the local PNG only replaces the texture — not the model selection.

### Directory Structure

After installation the folder layout should look like this:

```
.minecraft/
├── mods/
│   ├── localskins-1.0.0.jar
│   └── fabric-api-0.119.4+1.21.4.jar
└── config/
    └── localskins/
        ├── Notch.png          ← overrides the skin for a player named "Notch"
        ├── Steve.png          ← overrides the skin for a player named "Steve"
        └── YourUsername.png   ← overrides your own skin
```

> The `config/localskins/` directory is **not** created automatically on first launch.  
> You must create it manually if it does not appear after the first run.

**Step-by-step:**

1. Navigate to your `.minecraft` folder  
   - **Windows:** `%APPDATA%\.minecraft`  
   - **macOS:** `~/Library/Application Support/minecraft`  
   - **Linux:** `~/.minecraft`

2. Open (or create) the `config/` folder.

3. Inside `config/`, create a new folder named exactly `localskins`.

4. Drop your PNG skin file(s) inside, named after the target player's username.

5. Launch Minecraft — the skin loads automatically on first render.

---

## How It Works

LocalSkins hooks into the client-side skin texture pipeline using a **Mixin** on `AbstractClientPlayerEntity#getSkinTextures()`.

When a player entity is rendered for the first time:

1. The mixin intercepts the `getSkinTextures()` return value.
2. `LocalSkinManager` checks whether a PNG file exists at  
   `config/localskins/<playerName>.png`.
3. If found, the PNG is read into a `NativeImage`, wrapped in a  
   `NativeImageBackedTexture`, and registered with Minecraft's `TextureManager`  
   under the identifier `localskins:skin/<playername>`.
4. The `SkinTextures` record is reconstructed with the new texture identifier  
   substituted in — cape, elytra, and model settings are preserved from the  
   original.
5. The result is cached in memory. Subsequent frames use the cached `Identifier`  
   with no further disk access.

If no file is found, the original `SkinTextures` object is returned untouched and Mojang's skin is used as normal.

```
getSkinTextures() called
        │
        ▼
  LocalSkinManager.getLocalSkin(name)
        │
   ┌────┴────┐
   │ cached? │──yes──► return cached Identifier → inject into SkinTextures
   └────┬────┘
        │ no
        ▼
  config/localskins/<name>.png exists?
        │
   ┌────┴────┐
   │  found  │──yes──► load PNG → register texture → cache → inject
   └────┬────┘
        │ no
        ▼
  mark as checked (no file) → return original SkinTextures unchanged
```

---

## Fallback Behaviour

| Situation | Result |
|---|---|
| `config/localskins/<name>.png` exists and is valid | Custom skin is used |
| File does not exist | Default Mojang account skin is used |
| File exists but is not a valid PNG / is corrupted | Warning logged, default skin used |
| Player has no Mojang skin (e.g. offline mode default) | Default Steve/Alex skin used |

Fallback decisions are cached per session. If you add a new skin file while the game is running, **you must restart Minecraft** for it to take effect — the mod records which names it has already checked and does not re-poll the filesystem after that.

---

## Multiplayer Compatibility

LocalSkins is entirely **client-side**. It does not:

- Send any data to servers
- Require any server-side mod or plugin
- Interfere with server-enforced skins or resource packs

It will work transparently on:

- ✅ Vanilla servers
- ✅ Fabric servers (with or without this mod installed server-side)
- ✅ Paper / Spigot / Purpur servers
- ✅ Hypixel and other large networks
- ✅ Singleplayer and LAN worlds

Other players on the same server will **not** see your custom skin — they see whatever skin your Mojang account has. Only the instance of Minecraft running LocalSkins applies the local overrides, and only for that client's rendering.

> If you want **other players** to see a specific skin on a specific person, every player who wants to see that override must install LocalSkins and have the corresponding PNG file in their own `config/localskins/` folder.

---

## Building from Source

**Prerequisites:** JDK 21, Git

```bash
# Clone the repository
git clone https://github.com/asher-not-king/localskins.git
cd localskins

# Build (produces the jar in build/libs/)
./gradlew build          # Linux / macOS
gradlew.bat build        # Windows
```

The output jar will be at:

```
build/libs/localskins-1.0.0.jar
```

**Other useful Gradle tasks:**

| Task | Description |
|---|---|
| `./gradlew build` | Compile and produce the release jar |
| `./gradlew runClient` | Launch a dev client with the mod loaded |
| `./gradlew genSources` | Decompile Minecraft sources for IDE navigation |

---

## Project Structure

```
localskins/
├── build.gradle                          # Fabric Loom build configuration
├── gradle.properties                     # Version pins (MC, Yarn, Loader, Fabric API)
├── settings.gradle                       # Gradle project name
└── src/
    ├── main/
    │   ├── java/com/example/localskins/
    │   │   └── LocalSkins.java           # ModInitializer (no-op, logic is client-only)
    │   └── resources/
    │       ├── fabric.mod.json           # Mod metadata and entrypoint declarations
    │       └── localskins.mixins.json    # Mixin configuration
    └── client/
        └── java/com/example/localskins/
            ├── LocalSkinsClient.java     # ClientModInitializer entrypoint
            ├── LocalSkinManager.java     # Disk I/O, texture registration, caching
            └── mixin/
                └── AbstractClientPlayerEntityMixin.java  # getSkinTextures() intercept
```

---

## Troubleshooting

**My skin isn't loading.**

- Confirm the file is named **exactly** after your in-game username (case-sensitive), e.g. `Notch.png` not `notch.png`.
- Confirm the file is inside `.minecraft/config/localskins/` — not inside `mods/` or `config/` directly.
- Confirm the image is a valid **64 × 64 PNG**.
- Check the game log (`latest.log`) for lines beginning with `[LocalSkins]` — a successful load prints `Loaded custom skin for '...'`; a failure prints a warning.

**The log says "Failed to load skin file".**

- The PNG may be corrupted or saved in an incompatible format (e.g. indexed-color PNG). Re-export it as a standard **RGBA PNG** from an image editor.

**I added a new skin file while the game was running but it doesn't appear.**

- Restart the game. The manager caches "no file found" results for the session and does not re-check the filesystem after the initial lookup.

**The skin loads but the arms look wrong / clipping.**

- You are using a `slim` (Alex) skin texture on a `default` (Steve) model or vice versa. Change your account's model type at [minecraft.net/profile/skin](https://www.minecraft.net/en-us/profile/skin) to match your PNG.

**Other players on the server see my old Mojang skin.**

- This is expected — see [Multiplayer Compatibility](#multiplayer-compatibility). LocalSkins only affects local rendering.

---

## Known Limitations

- Skin files are loaded once per session; adding or changing files requires a restart.
- The mod replaces only the **skin texture**. Cape and elytra textures remain under Mojang's control.
- The skin model (`default` vs `slim`) is determined by the player's Mojang account setting, not the local PNG.
- Custom skins are only visible to the client running this mod — other players on a server see the Mojang-registered skin.

---

## License

This project is licensed under the **MIT License**.  
You are free to use, modify, and redistribute it with attribution.
