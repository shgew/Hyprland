# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What is Hyprland?

Hyprland is a 100% independent, dynamic tiling Wayland compositor written in C++26. It's a full window manager/compositor that doesn't use wlroots, libweston, kwin, or mutter. Key features include animations, gradient borders, blur, powerful plugin support, and extensive customization.

## Building and Testing

### Building the project

**Standard release build:**
```bash
make release
```

**Debug build (includes test support):**
```bash
make debug
```

**Build without precompiled headers:**
```bash
make nopch
```

**Clean build:**
```bash
make clear  # Remove build artifacts and generated protocol files
make all    # Clean + release build
```

**Install:**
```bash
sudo make install
```

### Installing headers for plugin development

After building, install headers needed for plugin development:
```bash
sudo make installheaders
```

### Running tests

Build and run the test suite:
```bash
make test
```

This builds in debug mode and runs hyprtester with the test configuration.

### Build system details

- Uses **CMake 3.30+** as the build system
- Requires C++26 standard
- Build flags include `-Wall -Wextra -Wpedantic` with specific warning suppressions
- **Important:** LTO is disabled (`-fno-lto`) to prevent plugin breakage
- Debug builds can enable ASan with `-DWITH_ASAN=True`
- Tracy profiling support available with `-DUSE_TRACY=True`

## High-Level Architecture

### Manager-Based Architecture

Hyprland uses a **centralized manager pattern** where specialized manager classes handle distinct responsibilities. These are instantiated as global singletons accessible via `g_pCompositor`, `g_pInputManager`, etc.

**Key Managers (in `src/managers/`):**
- `Compositor.cpp/hpp` - Central orchestrator managing monitors, windows, workspaces, and the Wayland display loop
- `InputManager` - Low-level input device handling (keyboard, mouse, tablet, touch)
- `KeybindManager` - Keybindings, submaps, mouse bindings
- `SeatManager` - Wayland seat protocol (focus, capabilities)
- `PointerManager` - Mouse cursor management
- `LayoutManager` - Layout engine selection (Dwindle, Master)
- `ProtocolManager` - Dynamic protocol lifecycle management
- `EventManager` - Custom event dispatching system
- `HookSystemManager` - Plugin hook callbacks
- `ANRManager` - Application Not Responding detection
- `SessionLockManager` - Screen locking

### Desktop Hierarchy

```
CCompositor
  ├─ Monitors (via Aquamarine backend)
  │   ├─ Layer surfaces (panels, backgrounds, overlays)
  │   └─ Active workspace
  ├─ Workspaces
  │   ├─ Windows list
  │   ├─ Layout engine (Dwindle/Master)
  │   └─ Animation state
  └─ Windows
      ├─ Surface (wl_surface)
      ├─ Position/size (logical + animated)
      ├─ Decorations (borders, shadows)
      └─ Popups and subsurfaces
```

### Core Subsystems

**Window Management (`src/desktop/`):**
- `Window.cpp/hpp` - Window representation with position, animations, fullscreen state, groups
- `Workspace.cpp/hpp` - Workspace with layout and window collections
- `LayerSurface.cpp/hpp` - Layer shell surfaces (panels, backgrounds)

**Layouts (`src/layout/`):**
- `IHyprLayout` - Base interface for all layouts
- `DwindleLayout` - Dynamic binary tree partitioning
- `MasterLayout` - Master-slave layout

**Rendering (`src/render/`):**
- `Renderer.cpp/hpp` - Main rendering orchestrator with damage tracking
- `OpenGL.cpp/hpp` - OpenGL ES 3 implementation
- `pass/` - Render pass system (surfaces, textures, borders, shadows, blur)
- `decorations/` - Window decoration rendering

**Input (`src/managers/input/`):**
- Event routing from OS through Aquamarine backend
- Keybind processing and dispatch
- Gesture recognition (touchpad swipes)
- Text input and IME support

**Protocols (`src/protocols/`):**
- Wayland protocol implementations (100+ protocols)
- Core protocols: `Compositor`, `Seat`, `DataDevice`, `Output`
- Window protocols: `XDGShell`, `LayerShell`
- Modern features: color management, tearing control, fractional scaling

**Configuration (`src/config/`):**
- `ConfigManager.cpp/hpp` - Parses hyprland.conf using hyprlang library
- `ConfigWatcher` - Hot-reload on file changes
- Window rules, workspace rules, keybinds, animations, etc.

### Signal/Event System

Hyprland uses a custom C++ signal system (`helpers/signal/Signal.hpp`) for event-driven communication:
- Windows, surfaces, and devices emit signals on state changes
- Managers subscribe to relevant signals
- Plugin hooks attach to these signals
- Prevents tight coupling between subsystems

### Animated Variables

Core animation framework using `PHLANIMVAR<T>` template:
```cpp
PHLANIMVAR<Vector2D> m_realPosition;  // Animates smoothly
PHLANIMVAR<float> m_alpha;            // For opacity
```
Used throughout for smooth window movements, workspace transitions, and effects.

### Memory Management

Uses shared pointers for safe memory management:
- `SP<>` - Shared pointer (e.g., `PHLWINDOW`, `PHLMONITOR`)
- `WP<>` - Weak pointer
- RAII principles throughout to prevent use-after-free

## Important Development Concepts

### Protocol Generation

Wayland protocols are generated from XML using `hyprwayland-scanner`:
- XML definitions in `protocols/` and `WAYLAND_PROTOCOLS_DIR`
- Generated `.cpp/.hpp` files in `protocols/`
- Don't manually edit generated files
- Regenerate with `./scripts/generateShaderIncludes.sh` (runs automatically during cmake)

### Render Pipeline Flow

```
renderMonitor(monitor)
  ├─ Determine damage (what needs redrawing)
  ├─ beginRender() - Setup framebuffer, clear damage
  ├─ For each workspace:
  │   ├─ Render fullscreen windows (if any)
  │   ├─ Render tiled/floating windows
  │   │   └─ Multi-pass per window:
  │   │       ├─ Surface pass (window content)
  │   │       ├─ Blur pass
  │   │       ├─ Shadow pass
  │   │       └─ Border/decoration passes
  │   └─ Render layer surfaces
  ├─ Render cursor
  └─ endRender() - Swap buffers, send frame callbacks
```

### Window Lifecycle

1. App creates XDG toplevel surface via Wayland protocol
2. `XDGShell.cpp` creates `CWindow` object
3. Compositor adds window to active workspace
4. Layout engine repositions all windows (animated)
5. Renderer draws window in next frame
6. Window receives input events, can be moved/resized
7. On close: Window emits destroy signal, removed from lists

### Input Event Flow

```
OS Event (key press, mouse move)
  ↓
Aquamarine Backend
  ↓
InputManager::onKeyEvent() / onMouseEvent()
  ↓
KeybindManager checks for matching keybind
  ├─ If matched: Execute handler (e.g., moveFocus, workspace)
  └─ If not: Pass to focused surface via Wayland protocol
```

## Tools and Utilities

### hyprctl - Runtime Control Interface

Unix socket at `$XDG_RUNTIME_DIR/hypr/<instance>/.socket.sock` for runtime control.

**Query commands:**
```bash
hyprctl monitors      # List all monitors
hyprctl clients       # List all windows
hyprctl workspaces    # List workspaces
hyprctl devices       # List input devices
hyprctl keybinds      # List configured keybinds
```

**Control commands:**
```bash
hyprctl dispatch focuswindow <window>
hyprctl dispatch workspace <id>
hyprctl dispatch movefocus <direction>
hyprctl keyword <option> <value>  # Change config at runtime
```

**Output formats:**
- Default: Human-readable
- `-j`: JSON output
- `-b`: Batch mode for multiple commands

Implementation: `src/debug/HyprCtl.cpp` (93K lines)

### hyprpm - Plugin Manager

Manages the Hyprland plugin ecosystem (located in `hyprpm/`):
- Downloads and builds plugins from git repos
- Verifies ABI compatibility with current Hyprland version
- Loads/unloads plugins at runtime
- Plugin headers installed via `sudo make installheaders`

### hyprtester - Testing Framework

Test runner for Hyprland (located in `hyprtester/`):
- Runs integration tests
- Can load test plugins
- Used by `make test`

## Dependencies

**Core Hyprland libraries (hypr.org ecosystem):**
- `aquamarine>=0.9.3` - DRM/KMS backend abstraction
- `hyprlang>=0.3.2` - Configuration parser
- `hyprcursor>=0.1.7` - Cursor theme support
- `hyprutils>=0.10.2` - Utility functions
- `hyprgraphics>=0.1.6` - Graphics utilities
- `hyprland-protocols>=0.6.4` - Custom Wayland protocols

**System dependencies:**
- `wayland-server>=1.22.90`, `wayland-protocols>=1.45`
- OpenGL ES 3
- `xkbcommon>=1.11.0` - Keyboard layouts
- `libinput>=1.28` - Input device handling
- `cairo`, `pango`, `pangocairo` - Text rendering
- `pixman-1` - Pixel manipulation
- `libdrm`, `gbm` - Direct rendering
- `re2` - Regular expressions
- `muparser` - Math expression parser

**Optional:**
- XWayland dependencies (disable with `-DNO_XWAYLAND=ON`)
- systemd (disable with `-DNO_SYSTEMD=ON`)

## Configuration Files

**User config:** `~/.config/hypr/hyprland.conf`
**Example config:** `example/hyprland.conf`
**Runtime socket:** `$XDG_RUNTIME_DIR/hypr/<instance>/`

## Code Style

- **C++26** standard
- Use clang-format (`.clang-format` in root)
- Check with clang-tidy (`.clang-tidy` in root)
- Run `clang-format` on modified files before committing
- Naming: PascalCase for classes, camelCase for methods/variables
- Use `SP<>` for shared ownership, `WP<>` for weak references
- Emit signals for state changes instead of direct callbacks

## Plugin Development

Plugins are compiled `.so` shared libraries that can:
- Add custom keybind handlers
- Register new config options
- Hook into events (window open/close, focus change, etc.)
- Create custom Wayland protocols
- Modify rendering behavior

**Requirements:**
- Must match Hyprland version/ABI exactly
- Build against installed headers (`sudo make installheaders`)
- Link against Hyprland symbols
- Export `APICALL EXPORT std::string PLUGIN_API_VERSION()`
- Export `APICALL EXPORT PLUGIN_DESCRIPTION_INFO PLUGIN_INIT(HANDLE handle)`

See hyprpm source for plugin loading implementation.

## XWayland Integration

Located in `src/xwayland/`:
- Runs XWayland server for X11 app compatibility
- `XWaylandManager` manages X11 window lifecycle
- `XWaylandSurface` wraps X11 windows as Wayland surfaces
- Can be disabled at build time with `-DNO_XWAYLAND=ON`

## Debugging

**Debug build:**
```bash
make debug
```

**ASan (Address Sanitizer) build:**
```bash
make asan  # Run in TTY only, not within Hyprland session
```

**Logging:**
- Logs to stderr/stdout
- Check systemd journal: `journalctl -u hyprland` (if using systemd session)
- Enable debug logging in config: `debug { disable_logs = false }`

**Crash reports:**
- Crash reporter generates reports with backtraces
- Located in `/tmp/hypr/`
- Disable with `HYPRLAND_NO_CRASHREPORTER=1`

## Version Information

Version defined in `VERSION` file in root directory. Git information is embedded during build via CMake in `src/version.h` (generated from `src/version.h.in`).

## Important Code Patterns

### Finding Windows
```cpp
// Get window under cursor
g_pCompositor->vectorToWindowUnified(cursor_pos);

// Get focused window
g_pCompositor->m_pLastWindow.lock();

// Iterate all windows
for (auto& w : g_pCompositor->m_vWindows) { ... }
```

### Damage Tracking
```cpp
// Mark window as needing redraw
g_pHyprRenderer->damageWindow(pWindow);

// Mark entire monitor
g_pHyprRenderer->damageMonitor(pMonitor);

// Mark specific region
g_pHyprRenderer->damageBox(&box);
```

### Adding Keybinds (in KeybindManager)
```cpp
// Dispatcher functions must be static members or standalone
static void moveWindow(std::string args);
```

### Configuration Access
```cpp
// Get config value
static auto* const PVAR = (Hyprlang::INT* const*)g_pConfigManager->getConfigValuePtr("general:border_size");
int borderSize = **PVAR;
```

## Common File Locations for Specific Features

- **Window focus logic:** `src/Compositor.cpp` (`focusWindow`, `focusSurface`)
- **Keybind handlers:** `src/managers/KeybindManager.cpp` (dispatcher functions)
- **Window rules:** `src/desktop/rule/WindowRules.cpp`
- **Workspace switching:** `src/desktop/Workspace.cpp` + `src/Compositor.cpp`
- **Animation config:** `src/managers/animation/AnimationManager.cpp`
- **Blur implementation:** `src/render/pass/BlurPass.cpp`
- **Layer shell (panels):** `src/desktop/LayerSurface.cpp`
- **Protocol implementations:** `src/protocols/<protocol-name>.cpp`

## When Working on Rendering

- Rendering uses OpenGL ES 3 exclusively
- Shaders in `src/render/shaders/` with `.frag.glsl` and `.vert.glsl` extensions
- Shader includes generated by `scripts/generateShaderIncludes.sh`
- Damage tracking is critical for performance - always damage what you change
- Render passes compose effects (see `src/render/pass/`)

## When Working on Input

- All input flows through `InputManager` first
- Keybinds processed before forwarding to clients
- Seat manages focus (keyboard, pointer, touch separately)
- Virtual devices supported (for remote control tools)
- Tablet support includes pressure and tilt

## When Working on Window Management

- Windows can be tiled, floating, fullscreen, or in groups (tabs)
- Layout engines control tiled window positioning
- Window rules apply on window creation (see `src/desktop/rule/`)
- Workspaces can be per-monitor or global
- Special workspaces (scratchpads) handled differently
- Use nix build to build the project