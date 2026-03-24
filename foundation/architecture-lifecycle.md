[← Back to Index](../README.md)

# React Native Architecture & Lifecycle Deep Dive

A comprehensive guide to React Native internals: architecture, threading, rendering,
Hermes engine, platform lifecycles, and the transition from Bridge to Bridgeless.

**Target audience**: Intermediate to advanced React Native / Expo developers who want
to understand what happens beneath the JavaScript layer.

---

<details>
<summary><strong>TL;DR</strong></summary>

- React Native has 6 threads: JS, Main/UI, Shadow, Native Modules, Render, and Bridge (old arch)
- New Architecture (JSI/Fabric/TurboModules) replaces the async Bridge with synchronous C++ calls
- Hermes compiles JS to bytecode at build time — no JIT, uses GenGC for memory management
- Android cold start: Zygote → SoLoader → Bridge → JS → Render. iOS: dyld → AppDelegate → RCTBridge → Bundle → Render
- As of RN 0.82, the old Bridge is fully removed — New Architecture is the only option

</details>

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Threading Model](#2-threading-model)
3. [Rendering Pipeline](#3-rendering-pipeline)
4. [Hermes Engine Internals](#4-hermes-engine-internals)
5. [Android App Lifecycle (Cold Start Sequence)](#5-android-app-lifecycle-cold-start-sequence)
6. [iOS App Lifecycle (Cold Start Sequence)](#6-ios-app-lifecycle-cold-start-sequence)
7. [Bridge vs Bridgeless (New Architecture Deep Dive)](#7-bridge-vs-bridgeless-new-architecture-deep-dive)
8. [Event System](#8-event-system)
9. [Memory Model](#9-memory-model)
10. [Key Architecture Decisions & Their Impact](#10-key-architecture-decisions--their-impact)
11. [Sources & References](#11-sources--references)

---

## 1. Architecture Overview

React Native runs JavaScript that describes a UI, then maps that description to
real platform-native views. The way JS communicates with native code is the
single most important architectural decision in the framework -- and it changed
fundamentally with the New Architecture.

### 1.1 Old Architecture (Bridge-Based)

The original React Native architecture (2015-2023 default) has three realms that
communicate through an asynchronous JSON bridge:

```
 OLD ARCHITECTURE (Bridge-Based)
 ================================

 +---------------------+       +---------------------+       +---------------------+
 |                     |       |                     |       |                     |
 |     JS THREAD       |       |      BRIDGE         |       |   NATIVE / UI       |
 |                     |       |                     |       |     THREAD           |
 |  - React tree       | JSON  |  - Serialization    | JSON  |  - UIKit / Android   |
 |  - Business logic   |------>|  - Batched queue     |------>|    View hierarchy    |
 |  - State mgmt       |       |  - Async only       |       |  - Touch dispatch    |
 |  - Event handlers   |<------|  - Single threaded   |<------|  - Animations        |
 |                     |       |    message queue     |       |  - Native APIs       |
 +---------------------+       +---------------------+       +---------------------+
                                        |
                                        v
                               +---------------------+
                               |   SHADOW THREAD      |
                               |                     |
                               |  - Yoga layout      |
                               |  - Tree diffing     |
                               |  - Layout calc      |
                               +---------------------+

 Data flow:
 JS setState() -> JSON serialize -> Bridge queue -> Shadow Thread (Yoga layout)
 -> Bridge queue -> Main Thread (native view mutations)
```

**Key components of the Old Architecture:**

| Component | Role |
|-----------|------|
| **JS Thread** | Runs the JavaScript bundle (React, business logic, state management). Single-threaded via the JS engine (JavaScriptCore or Hermes). |
| **Bridge** | An asynchronous, batched, serialized message queue. All communication between JS and native passes through it as JSON strings. |
| **Shadow Thread** | Runs Yoga (the layout engine). Receives the virtual DOM diff from JS, computes flexbox layout, and sends layout results to the main thread. |
| **Main/UI Thread** | The platform's primary thread. Creates and mutates actual native views (UIView on iOS, android.view.View on Android). Handles touch events, animations, and all OS interactions. |
| **Native Modules Thread** | A dedicated thread (or thread pool) where native module method calls execute, preventing them from blocking the UI thread. |

**Bridge bottlenecks:**

1. **Serialization overhead** -- Every message is JSON.stringify'd on one side and JSON.parse'd on the other.
2. **Asynchronous only** -- No synchronous calls possible. Even reading a value from native requires a round trip.
3. **Batching delays** -- Messages are batched and flushed at ~5ms intervals, introducing latency.
4. **Single queue** -- All modules share one message queue; a slow module blocks everything behind it.
5. **No shared memory** -- JS and native cannot share data structures; everything is copied.

### 1.2 New Architecture (JSI, Fabric, TurboModules, Codegen)

The New Architecture (default since React Native 0.76) replaces the bridge with
four interconnected pillars:

```
 NEW ARCHITECTURE (Bridgeless)
 ================================

 +---------------------+    JSI (C++)    +---------------------+
 |                     |<===============>|                     |
 |     JS THREAD       |  Direct refs    |   NATIVE / UI       |
 |                     |  No serialize   |     THREAD           |
 |  - React 18 tree    |  Sync + Async   |                     |
 |  - Concurrent mode  |                 |  - Fabric Renderer   |
 |  - TurboModule      |                 |  - Native views      |
 |    JS bindings      |                 |  - TurboModule       |
 |                     |                 |    host objects       |
 +---------------------+                 +---------------------+
         |                                        |
         |  Immutable                              |
         |  Shadow Trees                           |
         v                                        v
 +---------------------+       +---------------------+
 |   RENDER PHASE      |       |   MOUNT PHASE        |
 |   (JS Thread)       |       |   (Main Thread)      |
 |                     |       |                     |
 |  - Create new tree  |       |  - Tree diff         |
 |  - Clone + mutate   |       |  - View mutations    |
 |                     |       |  - Event listeners   |
 +--------+------------+       +---------------------+
          |                              ^
          v                              |
 +---------------------+                 |
 |   COMMIT PHASE      |-----------------+
 |   (Background)      |
 |                     |
 |  - Yoga layout      |
 |  - Tree promotion   |
 +---------------------+

 CODEGEN (Build Time)
 +-----------------------------------------------------+
 | TypeScript/Flow specs -> C++ interfaces              |
 | Generates type-safe native <-> JS bridge code        |
 | Catches type mismatches at compile time              |
 +-----------------------------------------------------+
```

**The Four Pillars:**

| Pillar | What It Does |
|--------|-------------|
| **JSI (JavaScript Interface)** | A C++ API that lets JS hold direct references to C++ host objects (and vice versa). No serialization, no bridge. JS can call native methods synchronously or asynchronously through these references. |
| **Fabric** | The new rendering system. Replaces the old renderer with an immutable shadow tree architecture. Rendering happens in three explicit phases (render, commit, mount) enabling React 18 concurrent features. |
| **TurboModules** | The replacement for legacy Native Modules. Lazily initialized (only loaded when first accessed), type-safe (via Codegen), and callable both synchronously and asynchronously through JSI. |
| **Codegen** | A build-time code generator. Takes TypeScript or Flow type definitions and generates C++ (and Java/ObjC) interface code. This ensures type safety at the JS-native boundary and eliminates an entire class of runtime type errors. |

### 1.3 Side-by-Side Comparison

```
 OLD ARCHITECTURE                          NEW ARCHITECTURE
 ================                          ================

 JS  ---[JSON]---> Bridge ---[JSON]---> Native
     <--[JSON]---        <--[JSON]---

 - All data serialized as JSON             JS  <---[JSI C++ refs]---> Native
 - Always asynchronous
 - Batched message queue                   - Direct memory references
 - All modules loaded at startup           - Sync AND async calls
 - Mutable shadow tree                     - No serialization needed
                                           - Lazy module initialization
                                           - Immutable shadow trees
```

---

## 2. Threading Model

React Native is a multi-threaded framework. Understanding which thread does what
is critical for debugging performance issues, ANRs, and deadlocks.

### 2.1 Thread Inventory

| Thread Name | Platform Label | Purpose | What Happens When Blocked |
|-------------|---------------|---------|--------------------------|
| **Main / UI Thread** | `main` (iOS), `UI Thread` (Android) | Renders native views, handles touch events, runs platform animations, manages the view hierarchy. This is the OS's primary thread. | App freezes. ANR dialog on Android (5s). Watchdog kill on iOS. User sees a hung UI with no response to touches. |
| **JS Thread** | `mqt_js` (old arch) / `mqt_v_js` (Hermes, old arch) / `com.facebook.react.runtime.JavaScript` (new arch) | Executes the entire JS bundle: React rendering, business logic, event handlers, state management, timers. | UI stops updating. Touch events queue up. Animations driven by JS stutter. `setTimeout`/`setInterval` callbacks delayed. React re-renders halt. |
| **Shadow Thread** | `mqt_native_modules` (old arch, shared) | Runs Yoga layout calculations. In old arch, often shares a thread with native modules. In new arch, layout runs in the commit phase (background thread). | Layout is delayed. Views may appear unsized or in wrong positions. Cascading delay to mount phase. |
| **Native Modules Thread** | `mqt_native_modules` (old arch) / per-module threads (new arch) | Executes native module methods (camera, file system, network, etc.). | That specific native operation stalls. In old arch, can block other modules sharing the queue. |
| **Render Thread** | `RenderThread` (Android only) | Android's GPU rendering thread (hardware-accelerated). Separate from the UI thread on Android 5.0+. Draws display lists produced by the UI thread. | Dropped frames, visual glitches. UI thread can still process events but frames won't paint. |
| **React Concurrent Thread** | `create_react_co` | Used by React 18 concurrent features (useTransition, Suspense). Handles interruptible rendering work. | Low-priority renders delayed; high-priority updates still proceed on JS thread. Graceful degradation by design. |

### 2.2 Old Architecture Threading

```
 OLD ARCHITECTURE THREADS
 ========================

 +------------------+     +------------------+     +------------------+
 |   Main Thread    |     |    JS Thread     |     |  Native Modules  |
 |                  |     |   (mqt_js)       |     |   Thread         |
 |  - View render   |     |                  |     |   (mqt_native_   |
 |  - Touch events  |     |  - React         |     |    modules)      |
 |  - Native anim   |     |  - Redux/state   |     |                  |
 |  - OS callbacks   |     |  - Timers        |     |  - Camera        |
 |                  |     |  - Handlers      |     |  - File I/O      |
 +--------+---------+     +--------+---------+     |  - Yoga layout   |
          ^                        ^                +--------+---------+
          |                        |                         ^
          |    +----------+        |    +----------+         |
          +----|  BRIDGE   |-------+----|  BRIDGE   |---------+
               | (async    |            | (async    |
               |  JSON)    |            |  JSON)    |
               +----------+            +----------+

 All communication is async JSON through the bridge.
 Shadow/Yoga work often runs on the native modules thread.
```

### 2.3 New Architecture Threading

```
 NEW ARCHITECTURE THREADS
 ========================

 +------------------+     +------------------+     +------------------+
 |   Main Thread    |     |    JS Thread     |     | Background Pool  |
 |                  |     |                  |     |                  |
 |  - Mount phase   |<----|  - Render phase  |---->|  - Commit phase  |
 |  - View create   | JSI |  - React 18      |     |  - Yoga layout   |
 |  - View update   |<--->|  - TurboModule   |     |  - Tree diff     |
 |  - Events        |     |    JS bindings   |     |  - Promotion     |
 |  - Native anim   |     |  - Concurrent    |     |                  |
 +------------------+     +------------------+     +------------------+
                                   |
                                   | JSI (sync or async)
                                   v
                          +------------------+
                          |  TurboModule     |
                          |  Host Objects    |
                          |                  |
                          |  Each module     |
                          |  can specify its |
                          |  own thread      |
                          +------------------+

 JSI allows direct, synchronous C++ calls between threads.
 No serialization. Shared memory references possible.
```

### 2.4 Key Threading Differences

| Aspect | Old Architecture | New Architecture |
|--------|-----------------|-----------------|
| JS-to-Native communication | Async JSON bridge only | JSI: sync or async, no serialization |
| Layout thread | Often shares `mqt_native_modules` | Dedicated background thread (commit phase) |
| Native module execution | Shared thread pool | Per-module thread control |
| Concurrent rendering | Not possible | Supported via React 18 + immutable trees |
| Event delivery | Async through bridge | Can be synchronous via JSI |
| Module initialization | All modules loaded at startup (blocking) | Lazy: loaded on first access |

---

## 3. Rendering Pipeline

### 3.1 Old Renderer Pipeline

The old renderer uses a mutable shadow tree and communicates exclusively through
the bridge:

```
 OLD RENDERING PIPELINE
 ======================

 Step 1: JS Thread                    Step 2: Bridge
 +---------------------------+        +---------------------------+
 |                           |        |                           |
 |  React reconciler diffs   | JSON   |  Serialized as JSON       |
 |  virtual DOM              |------->|  Batched messages          |
 |  Produces a list of       |        |  Queued for shadow thread |
 |  "create view" / "update  |        |                           |
 |  view" operations         |        +-------------+-------------+
 |                           |                      |
 +---------------------------+                      v

 Step 3: Shadow Thread                Step 4: Main Thread
 +---------------------------+        +---------------------------+
 |                           |        |                           |
 |  Yoga processes layout    | JSON   |  Creates/updates actual   |
 |  Computes x, y, width,   |------->|  native views             |
 |  height for each node     |        |  Applies layout values    |
 |  Mutates shadow tree      |        |  Commits to screen        |
 |  in place                 |        |                           |
 +---------------------------+        +---------------------------+

 Total bridge crossings per render: 2 (JS->Shadow, Shadow->Main)
 Each crossing = JSON serialize + queue + deserialize
```

**Problems with this pipeline:**

- Two bridge crossings per frame means double serialization overhead.
- Mutable shadow tree means you cannot interrupt and restart work -- once you
  start mutating, you must finish.
- Layout and view creation happen on separate threads with only async
  communication, making it impossible to read layout synchronously.
- No way to prioritize urgent updates (e.g., user input) over background updates.

### 3.2 Fabric Rendering Pipeline (New Architecture)

Fabric introduces a three-phase pipeline with immutable shadow trees:

```
 FABRIC RENDERING PIPELINE
 =========================

 Phase 1: RENDER (JS Thread)
 +-----------------------------------------------------------+
 |                                                           |
 |  React 18 reconciler runs on JS thread                    |
 |                                                           |
 |  1. React calls render()                                  |
 |  2. Creates a NEW immutable shadow tree (clone + mutate)  |
 |  3. Old tree is untouched (still being displayed)         |
 |  4. This phase is INTERRUPTIBLE by higher-priority work   |
 |                                                           |
 |  Output: New immutable shadow tree (not yet laid out)     |
 |                                                           |
 +-----------------------------+-----------------------------+
                               |
                               v
 Phase 2: COMMIT (Background Thread)
 +-----------------------------------------------------------+
 |                                                           |
 |  1. Yoga layout engine runs on the new tree               |
 |  2. Computes all x, y, width, height values               |
 |  3. Tree diffing: compare new tree vs. currently mounted  |
 |  4. Produces a list of mutations (create, update, delete) |
 |  5. "Promotes" new tree to be the next tree to mount      |
 |                                                           |
 |  Output: Laid-out, diffed tree + mutation instructions    |
 |                                                           |
 +-----------------------------+-----------------------------+
                               |
                               v
 Phase 3: MOUNT (Main/UI Thread)
 +-----------------------------------------------------------+
 |                                                           |
 |  1. Applies mutation instructions to native view tree     |
 |  2. Creates new native views as needed                    |
 |  3. Updates properties on existing views                  |
 |  4. Removes deleted views                                 |
 |  5. Synchronous -- must not be interrupted                |
 |                                                           |
 |  Output: Updated screen                                   |
 |                                                           |
 +-----------------------------------------------------------+
```

### 3.3 Yoga Layout Engine

Yoga is a cross-platform flexbox layout engine written in C/C++ by Meta. It is
used in both the old and new architectures.

```
 YOGA LAYOUT ENGINE
 ==================

 Input: Shadow tree with style props (flexDirection, padding, margin, etc.)
 Output: Computed layout (x, y, width, height) for every node

 Example:

 Shadow Tree (input)              Computed Layout (output)
 +------------------+             +------------------+
 | View             |             | View             |
 | flex: 1          |             | x:0 y:0          |
 | flexDir: column  |             | w:375 h:812      |
 |   +------------+ |    Yoga     |   +------------+ |
 |   | Text       | |  =======>  |   | Text       | |
 |   | fontSize:16| |             |   | x:0 y:0    | |
 |   +------------+ |             |   | w:375 h:22 | |
 |   +------------+ |             |   +------------+ |
 |   | View       | |             |   | View       | |
 |   | flex: 1    | |             |   | x:0 y:22   | |
 |   +------------+ |             |   | w:375 h:790| |
 +------------------+             +------------------+

 Yoga implements the W3C Flexbox spec (with minor deviations):
 - flexDirection defaults to 'column' (not 'row' as in CSS)
 - flex means flexGrow (simplified)
 - All dimensions are density-independent pixels
```

**Key Yoga behaviors:**

- Written in C++ for performance -- runs natively on both platforms.
- Layout is computed top-down (parent constraints propagate to children).
- Text measurement requires a platform callback (iOS: TextKit, Android: BoringLayout/StaticLayout).
- Supports absolute positioning, percentage values, and aspect ratios.

### 3.4 Why Immutable Shadow Trees Enable Concurrent Rendering

```
 MUTABLE (Old) vs IMMUTABLE (Fabric) TREES
 ==========================================

 MUTABLE (Old Architecture):
 +---------+      +---------+      +---------+
 | Tree v1 | ---> | Tree v1'| ---> | Tree v1"|   Mutated in place.
 +---------+      +---------+      +---------+   Can't go back.
                                                  Can't interrupt.

 IMMUTABLE (Fabric):
 +---------+
 | Tree v1 |  <--- Still displayed, still valid
 +---------+
      |
      | clone()
      v
 +---------+
 | Tree v2 |  <--- Being built (render phase)
 +---------+       Can be ABANDONED if higher-priority
      |            update arrives. Tree v1 is untouched.
      | (if committed)
      v
 +---------+
 | Tree v2 |  <--- Now becomes the displayed tree
 +---------+       Tree v1 can be garbage collected

 This is what enables:
 - useTransition: low-priority re-renders can be interrupted
 - Suspense: render can pause and resume
 - startTransition: mark updates as non-urgent
 - No tearing: old tree is always consistent while new one is built
```

The immutability guarantee means:
1. The render phase (creating the new tree) can be interrupted at any point.
2. If a higher-priority update arrives, React abandons the in-progress tree
   and starts fresh -- the displayed tree (old tree) was never touched.
3. Multiple versions of the tree can exist simultaneously, enabling
   concurrent rendering without visual inconsistencies (tearing).

---

## 4. Hermes Engine Internals

Hermes is Meta's JavaScript engine, purpose-built for React Native. It became the
default engine in React Native 0.70 and is the only supported engine in the New
Architecture.

### 4.1 Bytecode Precompilation

Unlike V8 (Chrome/Node.js) or JavaScriptCore (Safari), Hermes compiles JavaScript
to bytecode at **build time**, not at runtime.

```
 HERMES BUILD-TIME COMPILATION
 ==============================

 Traditional JS Engine (V8, JSC):
 +----------+    Runtime     +----------+    Runtime     +----------+
 | JS Source| =============> | Parse &  | =============> | Execute  |
 | (.js)    |   (on device)  | Compile  |   (on device)  | bytecode |
 +----------+                +----------+                +----------+
                              ^ This is slow on mobile devices

 Hermes:
 +----------+   Build Time   +----------+   Runtime      +----------+
 | JS Source| =============> | Hermes   | =============> | Execute  |
 | (.js)    |  (on CI/dev    | Bytecode |   (on device)  | bytecode |
 +----------+   machine)     | (.hbc)   |   FAST START   | directly |
                              +----------+                +----------+
                              ^ Done on powerful build machine
                                Device just loads and runs

 .hbc file = Hermes ByteCode
 - Memory-mappable (mmap): loaded into memory without parsing
 - Pages loaded on demand (lazy loading)
 - ~50% smaller than equivalent minified JS
```

**Impact on startup:**

| Metric | JSC (old default) | Hermes |
|--------|-------------------|--------|
| TTI (Time to Interactive) | Slower (parse + compile on device) | Faster (precompiled bytecode, mmap) |
| Bundle size on disk | Minified JS (~larger) | Bytecode (~smaller) |
| Memory for code | Full source + compiled code in memory | mmap'd bytecode, page-fault loaded |
| Startup parse time | Significant (full parse) | Near zero (bytecode is ready) |

### 4.2 Register-Based Bytecode

Hermes uses a **register-based** bytecode VM, in contrast to V8's stack-based
approach:

```
 STACK-BASED vs REGISTER-BASED
 ==============================

 Example: a + b

 STACK-BASED (like V8 bytecode, JVM):
   LOAD a        ; push a onto stack
   LOAD b        ; push b onto stack
   ADD           ; pop two, push result
                 ; 3 instructions

 REGISTER-BASED (Hermes):
   ADD r0, r1, r2   ; r0 = r1 + r2
                     ; 1 instruction (but wider encoding)

 Trade-offs:
 - Register-based = fewer instructions dispatched = fewer dispatch overheads
 - Stack-based = simpler encoding = smaller bytecode per instruction
 - For an interpreter (no JIT), fewer dispatches matters more
   because each dispatch has overhead (fetch, decode, jump)
```

Register-based bytecode is a deliberate choice for an **interpreter-only**
engine: it reduces the number of bytecode instructions that need to be dispatched,
which is the primary performance bottleneck when there is no JIT compiler.

### 4.3 Garbage Collection: GenGC

Hermes uses a generational garbage collector called **GenGC** (and the newer
**Hades** collector for concurrent GC):

```
 HERMES GenGC
 ============

 +---------------------------------------------+
 |              HERMES HEAP                     |
 |                                              |
 |  +----------------+  +-------------------+   |
 |  | YOUNG GEN      |  | OLD GEN           |   |
 |  | (Nursery)      |  |                   |   |
 |  |                |  |                   |   |
 |  | - Small, fast  |  | - Large           |   |
 |  | - Bump alloc   |  | - Mark-compact    |   |
 |  | - Scavenged    |  | - Less frequent   |   |
 |  |   frequently   |  | - Compacts to     |   |
 |  | - Most objects  |  |   reduce frag     |   |
 |  |   die here     |  |                   |   |
 |  +-------+--------+  +-------------------+   |
 |          |                     ^              |
 |          | survives N          |              |
 |          | collections         |              |
 |          +------> promoted --->+              |
 |                                              |
 +---------------------------------------------+

 GenGC Algorithm:
 1. New objects allocated in Young Gen (bump pointer = extremely fast)
 2. When Young Gen fills up, scavenge (minor GC):
    - Copy surviving objects to a second semi-space
    - Objects that survive N scavenges get promoted to Old Gen
 3. When Old Gen fills up, mark-compact (major GC):
    - Mark all reachable objects
    - Compact (slide) them to remove fragmentation
    - More expensive but happens less often

 Hades (newer, concurrent collector):
 - Old Gen collection runs concurrently with JS execution
 - Reduces GC pause times from ~50ms to ~1-5ms
 - Young Gen still uses stop-the-world scavenge (fast, <5ms)
```

### 4.4 No JIT Compiler (By Design)

Hermes deliberately does **not** include a Just-In-Time compiler. This is not a
limitation -- it is a conscious design decision:

**Why no JIT:**

1. **iOS restriction**: Apple's App Store policy prohibits writable-executable
   memory pages (W^X). JIT compilers need to write machine code and then execute
   it. While JavaScriptCore has a special entitlement from Apple, third-party
   engines do not.

2. **Memory savings**: JIT compilers need to keep both the bytecode and the
   generated machine code in memory. On memory-constrained mobile devices, this
   is significant. Hermes only keeps the bytecode (which is mmap'd and can be
   paged out).

3. **Predictable performance**: JIT compilation introduces warmup time (code
   runs slowly until the JIT kicks in) and can cause unpredictable pauses when
   the JIT decides to recompile (deoptimization). Hermes provides consistent
   performance from the first instruction.

4. **Startup time**: JIT compilation competes with application startup for CPU
   time. By doing all compilation at build time, Hermes avoids this entirely.

| Feature | V8 (JIT) | JavaScriptCore (JIT) | Hermes (No JIT) |
|---------|----------|---------------------|----------------|
| Peak throughput | Highest | High | Lower |
| Startup time | Slow (parse+compile) | Medium | Fastest (precompiled) |
| Memory usage | Highest (source+bytecode+machine code) | High | Lowest (mmap'd bytecode) |
| Performance consistency | Variable (warmup, deopts) | Variable | Consistent |
| iOS compatible | No (W^X) | Yes (Apple entitlement) | Yes |
| GC pause times | <1ms (Orinoco) | <5ms (Riptide) | <5ms (Hades) |

### 4.5 How to Verify Hermes Bytecode Is Working

In development, Hermes runs JS source directly (for fast reload). In release
builds, the Metro bundler invokes the Hermes compiler to produce `.hbc` files.

**Verification methods:**

```bash
# 1. Check if Hermes is enabled in your app
# In JS console or at runtime:
# global.HermesInternal will be defined if Hermes is active
console.log(typeof HermesInternal !== 'undefined'); // true = Hermes

# 2. Check the bundle file format (release build)
# Hermes bytecode files start with specific magic bytes
xxd your-app-bundle.hbc | head -1
# Should show: c61f bc03  (Hermes bytecode magic number)

# 3. For Expo / EAS builds, check the build logs
# Look for: "Compiling JS bundle to Hermes bytecode"

# 4. React Native DevMenu
# Open the dev menu (shake device or Cmd+D)
# Look for "Hermes" badge in the menu

# 5. Check Podfile (iOS) or gradle (Android)
# iOS (Podfile):
#   :hermes_enabled => true
# Android (gradle.properties):
#   hermesEnabled=true
```

---

## 5. Android App Lifecycle (Cold Start Sequence)

Understanding the cold start sequence is essential for optimizing startup time
and debugging initialization crashes.

### 5.1 Start Types

| Start Type | Definition | What Is Preserved |
|------------|-----------|-------------------|
| **Cold Start** | Process does not exist. OS must fork from Zygote, load the app, initialize everything from scratch. | Nothing. Full initialization. |
| **Warm Start** | Process exists but the Activity was destroyed. Activity must be recreated, but the Application object and native libraries are still in memory. | Application, loaded .so files, initialized native modules. |
| **Hot Start** | Process and Activity exist. Activity just needs to be brought to foreground. | Everything. Just `onResume()`. |

### 5.2 Cold Start Sequence

```
 ANDROID COLD START SEQUENCE
 ============================

 Step 1: OS Process Creation
 +-----------------------------------------------------------+
 |                                                           |
 |  Linux Kernel                                              |
 |  1. App icon tapped / Intent received                     |
 |  2. ActivityManagerService resolves the app               |
 |  3. Zygote process fork()                                 |
 |     - Zygote is a pre-warmed process with ART runtime     |
 |       and core Android classes already loaded              |
 |     - fork() is fast: copy-on-write memory pages          |
 |  4. New process gets its own ART virtual machine           |
 |  5. Process calls RuntimeInit.commonInit()                 |
 |                                                           |
 |  Time: ~10-50ms                                            |
 +-----------------------------------------------------------+
                               |
                               v
 Step 2: Application.onCreate()
 +-----------------------------------------------------------+
 |                                                           |
 |  Your Application subclass runs:                           |
 |                                                           |
 |  1. SoLoader.init(this)                                   |
 |     - Initializes the native library loader               |
 |     - Unpacks .so files from APK if needed                |
 |     - Required BEFORE any JNI calls                       |
 |                                                           |
 |  2. Native module registration                             |
 |     OLD ARCH: ReactNativeHost.getPackages()               |
 |       - Returns list of all ReactPackage instances         |
 |       - Each package lists its NativeModules               |
 |       - ALL modules instantiated eagerly                   |
 |     NEW ARCH: ReactNativeHost.getReactPackageTurbo()      |
 |       - Returns TurboReactPackage instances                |
 |       - Modules are NOT instantiated yet (lazy)            |
 |       - Only the module spec (name/type) is registered    |
 |                                                           |
 |  3. Other SDK initialization (analytics, crash reporting) |
 |                                                           |
 |  Time: 50-300ms (depends on number of native modules)     |
 +-----------------------------------------------------------+
                               |
                               v
 Step 3: Activity.onCreate()
 +-----------------------------------------------------------+
 |                                                           |
 |  MainActivity (extends ReactActivity) runs:                |
 |                                                           |
 |  1. Creates ReactRootView                                  |
 |     - This is the Android ViewGroup that hosts all         |
 |       React Native views                                   |
 |                                                           |
 |  2. Creates ReactInstanceManager (old arch) or             |
 |     ReactHost (new arch)                                   |
 |     OLD ARCH: ReactInstanceManager                        |
 |       - Manages the bridge lifecycle                       |
 |       - Holds reference to JSC/Hermes instance             |
 |     NEW ARCH: ReactHost                                   |
 |       - Manages the bridgeless runtime                     |
 |       - Creates ReactInstance (JSI-based)                  |
 |                                                           |
 |  3. Calls startReactApplication() or ReactHost.start()    |
 |                                                           |
 |  Time: 10-50ms                                             |
 +-----------------------------------------------------------+
                               |
                               v
 Step 4: Bridge/Runtime Initialization
 +-----------------------------------------------------------+
 |                                                           |
 |  1. Load native libraries:                                 |
 |     SoLoader loads:                                        |
 |     - libjsi.so (JavaScript Interface)                     |
 |     - libhermes.so (Hermes engine)                         |
 |     - libreactnativejni.so (React Native core)            |
 |     - libfabricjni.so (Fabric renderer, new arch)         |
 |     - libturbomodulejsijni.so (TurboModules, new arch)   |
 |                                                           |
 |  2. Initialize JS engine:                                  |
 |     - Create Hermes runtime instance                       |
 |     - Set up JSI bindings                                  |
 |     - Register global objects (console, setTimeout, etc.) |
 |                                                           |
 |  3. Load JS bundle:                                        |
 |     DEBUG:  Load from Metro dev server (HTTP, slow)       |
 |     RELEASE: Load .hbc from APK assets (fast, mmap)       |
 |     RELEASE+HERMES: Memory-map bytecode directly           |
 |     RELEASE+JSC: Parse JS source at runtime (slower)       |
 |                                                           |
 |  Time: 100-500ms (Hermes release) / 1-3s (JSC release)   |
 +-----------------------------------------------------------+
                               |
                               v
 Step 5: React Rendering
 +-----------------------------------------------------------+
 |                                                           |
 |  1. AppRegistry.registerComponent('AppName', () => App)   |
 |     - This was called when the bundle loaded               |
 |                                                           |
 |  2. AppRegistry.runApplication('AppName', {               |
 |       rootTag: reactRootView.getId()                      |
 |     })                                                    |
 |     - React begins rendering the component tree            |
 |     - First render produces shadow tree                    |
 |     - Layout computed (Yoga)                               |
 |     - Native views created and mounted                     |
 |                                                           |
 |  3. First meaningful paint                                 |
 |     - User sees actual app content                         |
 |     - Before this: splash screen                           |
 |                                                           |
 |  Time: 100-500ms (depends on component complexity)        |
 +-----------------------------------------------------------+
                               |
                               v
 Step 6: Activity.onResume()
 +-----------------------------------------------------------+
 |                                                           |
 |  - Activity is now in foreground                           |
 |  - Touch events begin flowing                              |
 |  - AppState = "active" in JS                               |
 |  - App is fully interactive                                |
 |                                                           |
 +-----------------------------------------------------------+
```

### 5.3 SoLoader Explained

SoLoader is Meta's replacement for `System.loadLibrary()`. It solves several
Android-specific problems:

```
 SoLoader vs System.loadLibrary
 ===============================

 System.loadLibrary("hermes"):
   - Looks for libhermes.so in standard linker paths
   - Cannot load from compressed APK assets
   - Cannot handle custom dependency ordering
   - Limited error reporting

 SoLoader.loadLibrary("hermes"):
   - Can extract .so from APK assets on first run
   - Handles transitive native dependencies automatically
   - Supports "super-packed" .so files (compressed in APK, extracted on install)
   - Provides detailed error messages on load failure
   - Handles 32/64-bit ABI selection
   - Thread-safe with proper initialization ordering
```

SoLoader.init() must be called **before** any React Native code runs. If you
see crashes with `UnsatisfiedLinkError` or `SoLoader not initialized`, this
initialization is missing or out of order.

### 5.4 Bundle Loading: Debug vs Release

| Mode | Bundle Source | JS Engine | Load Method |
|------|-------------|-----------|-------------|
| Debug | Metro dev server (HTTP) | Hermes (interpreting source) | Fetch over HTTP, parse source |
| Release + Hermes | `index.android.bundle` (Hermes bytecode in APK assets) | Hermes | `mmap()` bytecode file, execute directly |
| Release + JSC | `index.android.bundle` (minified JS in APK assets) | JavaScriptCore | Read file, parse source, compile to bytecode, execute |

### 5.5 Native Module Initialization Order

```
 OLD ARCHITECTURE:
 Application.onCreate()
   -> ReactNativeHost.getPackages()
     -> new MainReactPackage()     // Core modules
     -> new MyCustomPackage()      // Your modules
       -> createNativeModules()    // ALL modules instantiated NOW
         -> new CameraModule()     // Even if never used
         -> new BluetoothModule()  // Still instantiated
         -> new 50MoreModules()    // All of them, eagerly

 NEW ARCHITECTURE (TurboModules):
 Application.onCreate()
   -> ReactNativeHost.getReactPackageTurbo()
     -> TurboReactPackage (only registers module names)

 [Later, when JS code first calls NativeModules.Camera...]
   -> TurboModuleManager.getModule("Camera")
     -> CameraModule instantiated NOW (lazy, on demand)
     -> CameraModule.initialize() via JSI binding
```

---

## 6. iOS App Lifecycle (Cold Start Sequence)

### 6.1 Cold Start Sequence

```
 iOS COLD START SEQUENCE
 ========================

 Step 1: dyld (Dynamic Linker)
 +-----------------------------------------------------------+
 |                                                           |
 |  1. Kernel maps the app executable into memory             |
 |  2. dyld (dynamic linker/loader) takes over               |
 |  3. Loads all dependent dylibs (frameworks):               |
 |     - UIKit.framework                                      |
 |     - Foundation.framework                                 |
 |     - React.framework (or static lib)                      |
 |     - hermes.framework                                     |
 |     - All CocoaPods dependencies                           |
 |  4. Rebases and binds symbols (ASLR fixups)               |
 |  5. Runs initializers for all loaded dylibs                |
 |                                                           |
 |  Note: Static linking (default for RN pods) reduces       |
 |  dylib count and speeds up this phase significantly.      |
 |                                                           |
 |  Time: 50-200ms (depends on number of dylibs)             |
 +-----------------------------------------------------------+
                               |
                               v
 Step 2: +[load] Methods and Static Initializers
 +-----------------------------------------------------------+
 |                                                           |
 |  Before main() is called, ObjC runtime runs:              |
 |                                                           |
 |  1. +[load] methods on all classes (in dependency order)  |
 |     - Some RN modules use +load for registration          |
 |     - This is discouraged in modern RN code               |
 |                                                           |
 |  2. C++ static constructors (__attribute__((constructor))) |
 |     - Some native modules register themselves here        |
 |                                                           |
 |  Time: 5-50ms                                              |
 +-----------------------------------------------------------+
                               |
                               v
 Step 3: main() -> UIApplicationMain()
 +-----------------------------------------------------------+
 |                                                           |
 |  int main(int argc, char *argv[]) {                       |
 |    @autoreleasepool {                                     |
 |      return UIApplicationMain(argc, argv, nil,            |
 |        NSStringFromClass([AppDelegate class]));           |
 |    }                                                      |
 |  }                                                        |
 |                                                           |
 |  UIApplicationMain:                                        |
 |  1. Creates UIApplication singleton                        |
 |  2. Creates AppDelegate instance                           |
 |  3. Sets up the main run loop                              |
 |  4. Loads main storyboard/nib (if any)                    |
 |  5. Calls AppDelegate methods                              |
 |                                                           |
 |  Time: ~10ms                                               |
 +-----------------------------------------------------------+
                               |
                               v
 Step 4: AppDelegate.didFinishLaunchingWithOptions
 +-----------------------------------------------------------+
 |                                                           |
 |  OLD ARCHITECTURE:                                        |
 |  1. Creates RCTBridge                                     |
 |     - RCTBridge is the central object managing the        |
 |       bridge, JS runtime, and module registry             |
 |     - Calls [bridge setUp] which:                         |
 |       a. Creates RCTCxxBridge (the actual implementation) |
 |       b. Collects all RCTBridgeModule classes              |
 |       c. Instantiates ALL native modules eagerly           |
 |                                                           |
 |  2. Creates RCTRootView with the bridge                   |
 |     rootView = [[RCTRootView alloc]                       |
 |       initWithBridge:bridge                               |
 |       moduleName:@"AppName"                               |
 |       initialProperties:nil];                             |
 |                                                           |
 |  NEW ARCHITECTURE:                                        |
 |  1. Creates RCTHost                                       |
 |     - RCTHost replaces RCTBridge                           |
 |     - Manages ReactInstance (JSI-based, bridgeless)       |
 |     - Native modules registered via TurboModule system    |
 |     - Lazy initialization of modules                       |
 |                                                           |
 |  2. Creates RCTFabricSurfaceHostingProxyRootView          |
 |     - Fabric-aware root view                               |
 |     - Hosts the Fabric surface                             |
 |                                                           |
 |  3. Sets rootView as window's rootViewController.view     |
 |                                                           |
 |  Time: 50-200ms                                            |
 +-----------------------------------------------------------+
                               |
                               v
 Step 5: Bridge/Runtime Initialization
 +-----------------------------------------------------------+
 |                                                           |
 |  (Runs on a background thread, concurrent with UI setup)  |
 |                                                           |
 |  1. Initialize JS engine                                   |
 |     - Create Hermes runtime (or JSC runtime)               |
 |     - Set up JSI bindings                                  |
 |     - Register global objects                              |
 |                                                           |
 |  2. Load JS bundle                                         |
 |     DEBUG:  Fetch from Metro dev server (HTTP, slow)      |
 |     RELEASE+HERMES: Load .hbc from app bundle (mmap)      |
 |     RELEASE+JSC: Load .jsbundle, parse at runtime          |
 |                                                           |
 |  3. Execute the bundle                                     |
 |     - Runs all top-level code                              |
 |     - Registers components via AppRegistry                 |
 |                                                           |
 |  Time: 100-500ms (Hermes) / 500ms-2s (JSC)               |
 +-----------------------------------------------------------+
                               |
                               v
 Step 6: React Rendering
 +-----------------------------------------------------------+
 |                                                           |
 |  1. AppRegistry.runApplication() called                    |
 |  2. React component tree renders                           |
 |  3. Shadow tree created, Yoga layout computed              |
 |  4. Native views (UIView hierarchy) created and mounted   |
 |  5. First meaningful paint                                 |
 |                                                           |
 |  Time: 100-500ms                                           |
 +-----------------------------------------------------------+
                               |
                               v
 Step 7: applicationDidBecomeActive
 +-----------------------------------------------------------+
 |                                                           |
 |  - App is now fully in foreground                          |
 |  - Touch events flow                                       |
 |  - AppState = "active" in JS                               |
 |  - App is fully interactive                                |
 |                                                           |
 +-----------------------------------------------------------+
```

### 6.2 RCTBridge vs RCTHost

| Aspect | RCTBridge (Old Arch) | RCTHost (New Arch) |
|--------|---------------------|-------------------|
| Communication | Async JSON bridge | JSI (direct C++ bindings) |
| Module registry | Eager: all modules instantiated at startup | Lazy: modules loaded on first use |
| JS runtime | Managed by RCTCxxBridge | Managed by ReactInstance |
| Renderer | Old renderer (mutable shadow tree) | Fabric (immutable shadow trees) |
| Module registration | `RCT_EXPORT_MODULE()` macro | Codegen-generated C++ spec |
| Thread model | Bridge thread + JS thread | JSI thread (shared runtime) |

### 6.3 Native Module Registration: Old vs New

```
 OLD ARCHITECTURE (RCT_EXPORT_MODULE):

 // MyModule.m
 @implementation MyModule
 RCT_EXPORT_MODULE();  // Macro that registers this class with the bridge

 RCT_EXPORT_METHOD(doSomething:(NSString *)value) {
   // Called asynchronously from JS via bridge
   // Arguments are deserialized from JSON
 }
 @end

 // Registration happens via ObjC runtime class scanning:
 // RCTBridge scans all classes for RCTBridgeModule protocol conformance
 // All matching classes are instantiated eagerly at startup

 -------------------------------------------------------

 NEW ARCHITECTURE (Codegen + TurboModule):

 // 1. Define spec in JS/TS:
 // NativeMyModule.ts
 import { TurboModuleRegistry } from 'react-native';
 export interface Spec extends TurboModule {
   doSomething(value: string): Promise<void>;
 }
 export default TurboModuleRegistry.getEnforcing<Spec>('MyModule');

 // 2. Codegen generates C++ interface:
 // MyModuleSpec.h (auto-generated)
 class JSI_EXPORT NativeMyModuleSpecJSI : public TurboModule {
   virtual jsi::Value doSomething(jsi::Runtime &rt,
     const jsi::String &value) = 0;
 };

 // 3. Implement in ObjC/C++:
 // MyModule.mm
 @interface MyModule : NSObject <NativeMyModuleSpec>
 @end
 @implementation MyModule
 - (void)doSomething:(NSString *)value {
   // Called via JSI -- can be sync or async
   // Arguments are C++ types, not JSON
 }
 @end

 // Registration:
 // Module is registered in TurboModuleManager
 // Instantiated ONLY when JS first calls TurboModuleRegistry.get('MyModule')
```

---

## 7. Bridge vs Bridgeless (New Architecture Deep Dive)

### 7.1 Bridge Bottlenecks

The bridge was the single communication channel between JavaScript and native.
Every piece of data flowing between realms had to pass through it:

```
 BRIDGE BOTTLENECKS
 ==================

 1. SERIALIZATION
    JS object -> JSON.stringify() -> UTF-8 string -> [bridge] ->
    UTF-8 string -> JSON.parse() -> Native object
    Cost: CPU time for serialization, memory for string copies

 2. ASYNCHRONOUS ONLY
    JS: "What is the current scroll position?"
    JS -> [bridge] -> Native (reads scrollY) -> [bridge] -> JS callback
    Minimum: 2 bridge crossings + 2 serializations
    Cannot be done synchronously

 3. BATCHING
    Messages are not sent immediately.
    They are queued and flushed every ~5ms (or on specific triggers).
    +-----+-----+-----+-----+
    | msg1 | msg2 | msg3 | msg4 |  <- batch flushed together
    +-----+-----+-----+-----+
    This improves throughput but adds latency to individual messages.

 4. SINGLE QUEUE
    All modules share one message queue:
    [CameraModule.takePicture, MapModule.setRegion, TextInput.onChange, ...]
    A slow operation blocks everything behind it.

 5. NO SHARED MEMORY
    JS and Native cannot share data structures.
    A large array must be COPIED across the bridge.
    NativeArray(1000 items) -> JSON(large string) -> JSArray(1000 items)
    Two copies of the data exist in memory.
```

### 7.2 How Bridgeless Works

```
 BRIDGELESS (JSI) ARCHITECTURE
 ==============================

 Instead of serialized messages, JS holds direct C++ object references:

 // JS side (conceptual):
 const cameraModule = global.__turboModuleProxy('Camera');
 // cameraModule is a JSI HostObject -- a C++ object callable from JS

 // When you call a method:
 cameraModule.takePicture({ quality: 0.8 });

 // What actually happens:
 // 1. JSI resolves the method on the C++ HostObject
 // 2. Arguments are passed as jsi::Value references (not JSON)
 // 3. The C++ method executes (can be sync or async)
 // 4. Return value is a jsi::Value reference (not JSON)

 JS Thread                          Native
 +------------------+               +------------------+
 |                  |    C++ ref    |                  |
 | jsi::Object  ----+-------------->| C++ HostObject   |
 | (camera proxy)   |               | (CameraModule)   |
 |                  |    Direct     |                  |
 | camera.snap() ---+--  call  --->| snap() {         |
 |                  |               |   // native code |
 |                  |<--  return ---| }                |
 |                  |    value      |                  |
 +------------------+               +------------------+

 No serialization. No queue. No batching. Direct C++ function call.
```

### 7.3 Fabric vs Old Renderer Comparison

| Feature | Old Renderer | Fabric |
|---------|-------------|--------|
| Shadow tree | Mutable (modified in place) | Immutable (clone-on-write) |
| Rendering phases | Unstructured (JS -> bridge -> shadow -> bridge -> native) | 3 explicit phases (Render, Commit, Mount) |
| Thread communication | Async bridge (JSON) | JSI (C++ direct calls) |
| Concurrent rendering | Not possible (mutable tree cannot be interrupted) | Supported (immutable trees can be abandoned) |
| React 18 features | Not supported | useTransition, Suspense, startTransition |
| Layout calculation | Shadow thread (shared with native modules) | Background thread (commit phase) |
| View creation | Async (JS describes, bridge transmits, native creates) | Can be synchronous when needed |
| Event handling | Async through bridge | Can be synchronous via JSI |
| View flattening | Limited | Aggressive (removes unnecessary wrapper views) |
| Typed interface | No (JSON any) | Yes (Codegen C++ specs) |

### 7.4 TurboModules vs Legacy Native Modules Comparison

| Feature | Legacy Native Modules | TurboModules |
|---------|---------------------|-------------|
| Initialization | Eager (all loaded at startup) | Lazy (loaded on first access) |
| Communication | Async JSON bridge only | JSI: sync or async, no serialization |
| Type safety | None at bridge boundary (JSON) | Codegen-enforced C++ types |
| Argument passing | JSON serialization/deserialization | jsi::Value references (zero-copy for strings) |
| Return values | Only via async callbacks/promises | Sync return values possible |
| Thread control | Runs on native modules thread | Module can specify its own thread/queue |
| Code generation | None (manual RCT_EXPORT_METHOD) | Codegen from TypeScript/Flow specs |
| Backward compatibility | N/A | Interop layer supports legacy modules |
| Platform code | ObjC (iOS) / Java (Android) | C++ shared + thin ObjC/Java/Kotlin layer |

### 7.5 React 18 Concurrent Features in React Native

The New Architecture enables React 18 concurrent features that were impossible
with the old bridge:

```
 CONCURRENT FEATURES IN REACT NATIVE
 ====================================

 1. useTransition
    +-------------------------------------------------+
    | const [isPending, startTransition] = useTransition();
    |
    | // Mark a state update as non-urgent:
    | startTransition(() => {
    |   setSearchResults(filterLargeList(query));
    | });
    |
    | // React can interrupt this render if a more urgent
    | // update arrives (e.g., user types another character)
    +-------------------------------------------------+

 2. Suspense
    +-------------------------------------------------+
    | <Suspense fallback={<Spinner />}>
    |   <LazyLoadedScreen />
    | </Suspense>
    |
    | // Component can "suspend" during render
    | // React shows fallback, then resumes when ready
    | // Possible because immutable trees can be paused
    +-------------------------------------------------+

 3. startTransition (standalone)
    +-------------------------------------------------+
    | import { startTransition } from 'react';
    |
    | startTransition(() => {
    |   navigation.navigate('HeavyScreen');
    | });
    |
    | // Navigation renders at low priority
    | // Current screen remains responsive
    +-------------------------------------------------+

 WHY THESE REQUIRE THE NEW ARCHITECTURE:
 - Interruptible rendering needs immutable shadow trees (Fabric)
 - Priority-based scheduling needs synchronous access to the
   renderer (JSI, not async bridge)
 - Suspense streaming needs direct communication between
   React and the native renderer (no bridge batching delay)
```

---

## 8. Event System

### 8.1 Touch Handling Flow

```
 TOUCH EVENT FLOW
 =================

 Step 1: Native Touch Detection
 +---------------------------+
 | OS detects touch event    |
 | (UITouch on iOS,          |
 |  MotionEvent on Android)  |
 +-------------+-------------+
               |
               v
 Step 2: Hit Testing
 +---------------------------+
 | Native view hierarchy     |
 | performs hit testing       |
 | to find the target view   |
 |                           |
 | iOS: hitTest:withEvent:   |
 | Android: dispatchTouch    |
 |   Event()                 |
 +-------------+-------------+
               |
               v
 Step 3: Event Dispatch to JS
 +---------------------------+
 | OLD ARCH:                 |
 | Touch serialized as JSON  |
 | Sent through bridge       |
 | (async, batched)          |
 |                           |
 | NEW ARCH:                 |
 | Touch dispatched via JSI  |
 | (can be synchronous)      |
 +-------------+-------------+
               |
               v
 Step 4: JS Event Handler
 +---------------------------+
 | React's event system      |
 | processes the touch       |
 |                           |
 | onPress, onLongPress,     |
 | onPressIn, onPressOut     |
 | handlers execute          |
 |                           |
 | May trigger setState()    |
 +-------------+-------------+
               |
               v
 Step 5: Re-render (if state changed)
 +---------------------------+
 | React re-renders          |
 | New shadow tree created   |
 | Layout computed           |
 | Native views updated      |
 +---------------------------+

 OLD ARCH total touch latency: ~32-48ms (2-3 frame delays)
   - Bridge async delay: ~5ms (batching)
   - JS processing: ~5-16ms
   - Bridge return: ~5ms
   - Native apply: ~5-16ms

 NEW ARCH total touch latency: ~16-32ms (1-2 frame delays)
   - JSI dispatch: <1ms (synchronous possible)
   - JS processing: ~5-16ms
   - Fabric mount: ~5-16ms
```

### 8.2 Gesture Handling

The `react-native-gesture-handler` library (by Software Mansion) moves gesture
recognition from JS to the native thread:

```
 BUILT-IN GESTURE SYSTEM (PanResponder):
 ========================================
 Native touch -> Bridge -> JS decides gesture -> Bridge -> Native response
                 ^^^^                             ^^^^
                 Latency on every touch move event

 REACT-NATIVE-GESTURE-HANDLER:
 ===============================
 Native touch -> Native gesture recognizer -> Native response
                    |
                    | (only final events sent to JS)
                    v
                 JS handler (for state update only)

 Key difference:
 - Built-in: JS is in the hot path for EVERY touch move
 - RNGH: Native handles gesture recognition, JS only gets results
 - This eliminates bridge/JSI overhead for continuous gestures
```

This is why `react-native-gesture-handler` with `react-native-reanimated` can
achieve 60fps animations that would be impossible with PanResponder on the old
architecture.

### 8.3 Synchronous vs Asynchronous Native Module Calls

```
 OLD ARCHITECTURE (Async Only):

 JS: NativeModules.Clipboard.getString()
     -> returns Promise (always)
     -> bridge serializes call
     -> native module thread executes
     -> bridge serializes result
     -> JS Promise resolves

 Even for trivial operations like reading a string, this requires
 a full async round trip through the bridge.

 -------------------------------------------------------

 NEW ARCHITECTURE (Sync + Async via JSI):

 SYNC CALL:
 JS: const value = MyTurboModule.getStringSync()
     -> JSI directly calls C++ method
     -> C++ method executes on JS thread
     -> Returns value immediately
     -> No thread hop, no serialization

 ASYNC CALL:
 JS: const value = await MyTurboModule.getStringAsync()
     -> JSI creates C++ promise
     -> Work dispatched to background thread
     -> Result delivered via JSI callback
     -> Still no serialization (jsi::Value references)

 WHEN TO USE SYNC:
 - Reading small cached values (user preferences, feature flags)
 - Layout measurements needed during render
 - Anything where async latency causes UI jank

 WHEN TO USE ASYNC:
 - I/O operations (network, disk, camera)
 - Long computations (image processing)
 - Anything that would block the JS thread
```

### 8.4 Batched Bridge Calls and Why New Architecture Does Not Need Batching

```
 OLD ARCHITECTURE BATCHING:

 Without batching:
 JS: setColor('red')     -> bridge call 1
 JS: setSize(20)         -> bridge call 2
 JS: setText('hello')    -> bridge call 3
 = 3 bridge crossings, 3 serializations

 With batching:
 JS: setColor('red')     -> queued
 JS: setSize(20)         -> queued
 JS: setText('hello')    -> queued
 [~5ms flush interval]   -> bridge call 1 (all 3 operations)
 = 1 bridge crossing, 1 serialization (but 5ms latency added)

 Batching trades latency for throughput. It was necessary because
 each bridge crossing was expensive (JSON serialization).

 -------------------------------------------------------

 NEW ARCHITECTURE (no batching needed):

 JS: setColor('red')     -> JSI call (C++ direct, ~0.01ms)
 JS: setSize(20)         -> JSI call (C++ direct, ~0.01ms)
 JS: setText('hello')    -> JSI call (C++ direct, ~0.01ms)

 Each call is so cheap that batching would add unnecessary complexity.
 No serialization means no overhead to amortize.
```

---

## 9. Memory Model

React Native apps have a complex memory landscape because they run code in
multiple runtimes simultaneously.

### 9.1 Android Memory Layout

```
 ANDROID MEMORY MODEL
 =====================

 +-----------------------------------------------------------+
 |                    PROCESS MEMORY                          |
 |                                                           |
 |  +-------------------+    +-------------------+           |
 |  | JAVA/KOTLIN HEAP  |    | NATIVE HEAP (C++) |           |
 |  | (ART managed)     |    |                   |           |
 |  |                   |    | - Hermes runtime   |           |
 |  | - Activity        |    | - JSI objects      |           |
 |  | - View objects    |    | - Yoga nodes       |           |
 |  | - Drawables       |    | - Image decode     |           |
 |  | - Java NativeModule|    |   buffers          |           |
 |  |   instances       |    | - TurboModule      |           |
 |  |                   |    |   host objects      |           |
 |  | GC: ART (generational, |                   |           |
 |  |  concurrent, compacting)| malloc/free       |           |
 |  +-------------------+    +-------------------+           |
 |                                                           |
 |  +-------------------+    +-------------------+           |
 |  | HERMES HEAP       |    | GRAPHICS MEMORY   |           |
 |  | (Hermes managed)  |    |                   |           |
 |  |                   |    | - GPU textures     |           |
 |  | - JS objects      |    | - Render buffers   |           |
 |  | - Closures        |    | - Surface flinger  |           |
 |  | - Strings         |    |   buffers          |           |
 |  | - ArrayBuffers    |    | - Bitmap hardware  |           |
 |  |                   |    |   acceleration     |           |
 |  | GC: GenGC/Hades   |    |                   |           |
 |  | (generational)    |    | Managed by GPU     |           |
 |  +-------------------+    |   driver           |           |
 |                           +-------------------+           |
 +-----------------------------------------------------------+

 Total memory = Java Heap + Native Heap + Hermes Heap + Graphics
 Android "PSS" (Proportional Set Size) includes all of these.
 OOM killer considers total process memory, not just Java heap.
```

### 9.2 iOS Memory Layout

```
 iOS MEMORY MODEL
 ================

 +-----------------------------------------------------------+
 |                    PROCESS MEMORY                          |
 |                                                           |
 |  +-------------------+    +-------------------+           |
 |  | ObjC/SWIFT HEAP   |    | NATIVE HEAP (C++) |           |
 |  | (ARC managed)     |    |                   |           |
 |  |                   |    | - Hermes runtime   |           |
 |  | - UIView objects  |    | - JSI objects      |           |
 |  | - UIImage         |    | - Yoga nodes       |           |
 |  | - NSString        |    | - Image decode     |           |
 |  | - ObjC NativeModule|    |   buffers          |           |
 |  |   instances       |    | - TurboModule      |           |
 |  |                   |    |   host objects      |           |
 |  | Memory: ARC       |    |                   |           |
 |  | (ref counting,    |    | malloc/free        |           |
 |  |  no GC)           |    +-------------------+           |
 |  +-------------------+                                    |
 |                                                           |
 |  +-------------------+    +-------------------+           |
 |  | HERMES HEAP       |    | GPU / METAL        |           |
 |  | (Hermes managed)  |    |                   |           |
 |  |                   |    | - Metal textures   |           |
 |  | - JS objects      |    | - CALayer backing  |           |
 |  | - Closures        |    |   stores           |           |
 |  | - Strings         |    | - Core Animation   |           |
 |  | - ArrayBuffers    |    |   buffers          |           |
 |  |                   |    |                   |           |
 |  | GC: GenGC/Hades   |    | Managed by Metal   |           |
 |  | (generational)    |    |   driver           |           |
 |  +-------------------+    +-------------------+           |
 |                                                           |
 +-----------------------------------------------------------+

 iOS does NOT have a traditional garbage collector for ObjC/Swift.
 ARC (Automatic Reference Counting) inserts retain/release at compile time.
 Retain cycles = memory leaks (use weak references to break cycles).
 iOS Jetsam kills apps that exceed memory limits (~1.5-3GB depending on device).
```

### 9.3 Hermes GenGC vs JSC Riptide GC

| Feature | Hermes GenGC/Hades | JSC Riptide GC |
|---------|-------------------|----------------|
| Type | Generational (young + old gen) | Generational + concurrent |
| Young gen strategy | Semi-space scavenge (copying) | Eden + Nursery, copying |
| Old gen strategy | Mark-compact (GenGC) or concurrent mark-sweep (Hades) | Concurrent mark-sweep |
| Compaction | Yes (eliminates fragmentation) | No (uses free-list allocation) |
| Max pause time (GenGC) | ~50ms (major GC, stop-the-world) | ~5ms (concurrent) |
| Max pause time (Hades) | ~1-5ms (concurrent old gen) | ~5ms |
| Memory overhead | Lower (bytecode is mmap'd, not duplicated) | Higher (source + compiled code) |
| Allocation speed | Very fast (bump pointer in young gen) | Fast (free-list in eden) |
| Write barriers | Yes (for generational tracking) | Yes |
| Weak references | Supported | Supported |

### 9.4 Memory Sharing Between JS and Native

```
 OLD ARCHITECTURE (Bridge):
 ============================
 JS Array [1,2,3]  ->  JSON.stringify  ->  "[1,2,3]"  ->  bridge
                                                            |
 Native Array {1,2,3}  <-  JSON.parse  <-  "[1,2,3]"  <---+

 Two SEPARATE copies of the data exist.
 The JSON string is a THIRD temporary copy.
 For large data (images, audio buffers), this is devastating.

 -------------------------------------------------------

 NEW ARCHITECTURE (JSI):
 ========================
 JS ArrayBuffer  ->  jsi::ArrayBuffer  ->  C++ can access the
                                            SAME memory directly

 // C++ TurboModule can return a shared buffer:
 jsi::ArrayBuffer buf = runtime.global()
   .getPropertyAsFunction(runtime, "ArrayBuffer")
   .callAsConstructor(runtime, 1024);

 // Both JS and C++ see the same bytes. Zero copy.
 // This enables efficient image processing, audio manipulation,
 // and large data transfer.

 For objects (not buffers), JSI still creates proxy objects.
 But there is no serialization -- JSI passes C++ references
 that JS can interact with directly.
```

---

## 10. Key Architecture Decisions & Their Impact

### 10.1 Why Hermes Over JavaScriptCore

| Criterion | Hermes Advantage | Impact |
|-----------|-----------------|--------|
| Startup time | Precompiled bytecode eliminates runtime parsing and compilation | 40-70% faster TTI on mid-range Android devices |
| Memory usage | Bytecode is mmap'd (shared with OS page cache, can be evicted under pressure) | 30-50% less memory for code representation |
| Bundle size | Bytecode is more compact than minified JS | Smaller APK/IPA (10-20% smaller bundles) |
| Performance predictability | No JIT warmup or deoptimization | Consistent frame rates from first interaction |
| GC tuning | GenGC/Hades designed specifically for React Native patterns | Fewer and shorter GC pauses during UI interaction |

**When JSC might still be preferred:**
- Compute-heavy workloads that benefit from JIT (rare in typical RN apps).
- Apps that spend most time in long-running computations rather than UI.
- Note: Since RN 0.70, Hermes is the default; since RN 0.76 (New Architecture default), Hermes is the only officially supported engine.

### 10.2 Why No JIT

```
 JIT COMPILER TRADE-OFFS ON MOBILE
 ==================================

 WITH JIT (V8-style):
 +----------+     +----------+     +----------+     +----------+
 | Parse    | --> | Bytecode | --> | Profile  | --> | Machine  |
 | source   |     | interpret|     | hot code |     | code gen |
 +----------+     +----------+     +----------+     +----------+
   ~100ms           Slow              +10-50ms         Fast!
   startup          execution         ongoing          execution
                                      overhead         (but only for hot code)

 Peak throughput is HIGHER with JIT.
 But: startup is slower, memory is higher, behavior is unpredictable.

 WITHOUT JIT (Hermes):
 +----------+     +----------+
 | Load     | --> | Execute  |
 | bytecode |     | bytecode |
 +----------+     +----------+
   ~10ms           Consistent
   (mmap)          speed

 React Native apps typically:
 - Are UI-heavy, not compute-heavy
 - Need fast startup (cold start matters for UX)
 - Run on memory-constrained devices
 - Need consistent frame timing (not spiky JIT behavior)

 For these patterns, no-JIT is the correct trade-off.
```

The iOS restriction (Apple prohibits W^X memory pages for third-party code)
makes JIT impossible on iOS regardless of preference. By not having a JIT on
either platform, Hermes maintains consistent behavior across iOS and Android.

### 10.3 Why Immutable Shadow Trees

```
 MUTABLE TREE PROBLEM:
 =====================

 Render starts...
   Tree: [A] -> [B] -> [C]
   Mutating B to B'...
     Tree: [A] -> [B'] -> [C]    <-- PARTIALLY MUTATED

   ** USER TYPES! High priority update needed! **

   But we can't stop -- tree is in inconsistent state.
   Must finish current render before handling user input.
   Result: dropped frames, janky typing.

 IMMUTABLE TREE SOLUTION:
 ========================

 Render starts...
   Current tree: [A] -> [B] -> [C]     (displayed, untouched)
   New tree:     [A'] -> [B'] ->        (building...)

   ** USER TYPES! High priority update needed! **

   Abandon new tree. It was never displayed.
   Current tree [A]->[B]->[C] is still perfectly valid.
   Start fresh with high-priority update.
   Result: immediate response to user input.

 This is React 18 concurrent rendering.
 It is IMPOSSIBLE with mutable trees.
```

### 10.4 Why Codegen

```
 WITHOUT CODEGEN (Old Architecture):
 ====================================

 // JS side:
 NativeModules.MyModule.calculate(42, "hello");
 // Types: number, string -- but bridge doesn't check

 // Native side (Java):
 @ReactMethod
 public void calculate(int value, String text) { ... }

 // What happens if JS passes wrong types?
 NativeModules.MyModule.calculate("not a number", 42);
 // Bridge serializes as JSON: ["not a number", 42]
 // Java tries to cast "not a number" to int
 // RUNTIME CRASH (or silent wrong behavior)

 -------------------------------------------------------

 WITH CODEGEN (New Architecture):
 ==================================

 // 1. TypeScript spec:
 export interface Spec extends TurboModule {
   calculate(value: number, text: string): number;
 }

 // 2. Codegen generates C++ interface (BUILD TIME):
 virtual jsi::Value calculate(
   jsi::Runtime &rt,
   double value,           // enforced: must be number
   const jsi::String &text // enforced: must be string
 ) = 0;

 // 3. If native implementation doesn't match:
 // BUILD ERROR -- caught at compile time, not runtime

 // 4. If JS passes wrong types:
 // JSI type checking catches it immediately
 // Clear error message instead of mysterious crash

 Codegen eliminates an entire class of bugs:
 - Type mismatches between JS and native
 - Missing method implementations
 - Wrong number of arguments
 - Wrong return types
```

### 10.5 When to Use Sync vs Async TurboModule Methods

```
 DECISION FRAMEWORK FOR SYNC vs ASYNC
 ======================================

 Use SYNC when:                        Use ASYNC when:
 +-------------------------------+     +-------------------------------+
 | - Reading cached values       |     | - I/O operations              |
 |   (feature flags, prefs)      |     |   (network, disk, camera)     |
 | - Computing small values      |     | - Long computations           |
 |   (string format, math)       |     |   (image processing)          |
 | - Measurement/layout needs    |     | - Operations that may fail    |
 |   (text measurement)          |     |   (permissions, user prompt)  |
 | - Execution time < 1ms        |     | - Execution time > 1ms        |
 +-------------------------------+     +-------------------------------+

 WARNING: Sync methods execute on the JS THREAD.
 A sync method that takes 100ms will block ALL JS execution
 for 100ms -- no renders, no event handling, no timers.

 Rule of thumb:
 - If you can't guarantee < 1ms execution: use async
 - If it involves ANY I/O: use async
 - If it might block on a lock: use async
 - If you need the value during render and it's cached: sync is OK
```

---

## 11. Sources & References

### Official Documentation

- **React Native Architecture Overview**: [reactnative.dev/architecture/overview](https://reactnative.dev/architecture/overview)
- **React Native Rendering Pipeline**: [reactnative.dev/architecture/render-pipeline](https://reactnative.dev/architecture/render-pipeline)
- **React Native Threading Model**: [reactnative.dev/architecture/threading-model](https://reactnative.dev/architecture/threading-model)
- **Fabric Renderer**: [reactnative.dev/architecture/fabric-renderer](https://reactnative.dev/architecture/fabric-renderer)
- **Hermes Engine**: [hermesengine.dev](https://hermesengine.dev)
- **Hermes GitHub**: [github.com/facebook/hermes](https://github.com/facebook/hermes)
- **Yoga Layout Engine**: [yogalayout.dev](https://yogalayout.dev)

### Meta Engineering Blog

- "Hermes: An open source JavaScript engine optimized for mobile apps" -- Meta Engineering Blog (2019)
- "React Native's New Architecture" -- Meta Engineering Blog (2022)
- "Fabric: React Native's New Rendering System" -- Meta Engineering Blog
- "Building Hermes: a runtime optimized for React Native" -- Chain React 2019

### React Native New Architecture Working Group

- [github.com/reactwg/react-native-new-architecture](https://github.com/reactwg/react-native-new-architecture)
- Migration guides, discussions, and architecture decision records
- TurboModules and Fabric adoption guides

### Conference Talks

- **Nicola Corti** -- "React Native New Architecture" (React Native EU, App.js Conf)
- **Samuel Susla** -- "Fabric: React Native's New Rendering System" (React Native EU 2021)
- **Tzvetan Mikov** -- "Hermes Engine Internals" (Chain React 2019)
- **Joshua Gross** -- "React Native's New Architecture" (React Conf 2021)
- **Lorenzo Sciandra** -- "Pillar of the New Architecture: TurboModules" (React Native EU)

### Community Blogs & Resources

- **Oscar Franco** -- [ospfranco.com](https://ospfranco.com) -- Deep dives into JSI, Hermes internals, native module development, and React Native performance optimization
- **Nicola Corti** -- [ncorti.com](https://ncorti.com) -- Android-specific React Native internals, build systems, and New Architecture migration
- **Software Mansion** -- [blog.swmansion.com](https://blog.swmansion.com) -- Creators of react-native-reanimated, react-native-gesture-handler, and react-native-screens; deep technical articles on animation, gestures, and native integration
- **Callstack** -- [callstack.com/blog](https://callstack.com/blog) -- Creators of react-native-paper, react-native-builder-bob; articles on performance, architecture, and tooling
- **Infinite Red** -- [shift.infinite.red](https://shift.infinite.red) -- React Native consultancy blog with architecture guides
- **Margelo (Marc Rousavy)** -- [margelo.io](https://margelo.io) -- Creator of react-native-vision-camera, react-native-mmkv; articles on JSI, C++ modules, and performance

### Additional References

- **React 18 Working Group**: [github.com/reactwg/react-18](https://github.com/reactwg/react-18) -- Concurrent features that power Fabric
- **SoLoader (Android)**: [github.com/facebook/SoLoader](https://github.com/facebook/SoLoader) -- Native library loading for Android
- **Apple App Store Review Guidelines**: Section 2.5.2 (code execution restrictions relevant to JIT)
- **ART (Android Runtime)**: [source.android.com/docs/core/runtime](https://source.android.com/docs/core/runtime) -- Android's managed runtime and GC
- **dyld (Apple)**: Apple's open-source dynamic linker documentation

---

*Last updated: March 2026. Architecture details reflect React Native 0.76+ with New Architecture as default.*

---

## Related Guides

| Guide | Relationship |
|-------|-------------|
| [New Architecture Deep Dive](../optimization/new-architecture-migration.md) | Migration playbook with Shopify/Discord production metrics — practical application of the architecture concepts covered here |
| [SIGABRT & libc Debugging](../debugging/crash-analysis.md) | When architecture-level issues cause native crashes — signal analysis and root cause framework |
| [Native-Layer Debugging](../debugging/native-layer-debugging.md) | Profiling tools for the native layer described in this guide — Android Studio, Xcode, Perfetto |
| [Profiling Tools Deep Dive](../debugging/profiling-tools-deep-dive.md) | Decision framework for choosing the right profiling tool at each architecture layer |
