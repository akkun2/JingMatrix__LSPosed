# Vector Daemon

The Vector `daemon` is a highly privileged, standalone executable that runs as `root`.
It acts as the central coordinator and backend for the entire Vector framework.

Unlike the injected framework code, the daemon does not hook methods directly. Instead, it manages state, provides IPC endpoints to hooked apps and modules, handles AOT compilation evasion, and interacts safely with Android system services.

## Architecture Overview

The daemon relies on a dual-IPC architecture and extensive use of Android Binder mechanisms to orchestrate the framework lifecycle without triggering SELinux denials or breaking system stability.

1. **Bootstrapping & Bridge (`core/`)**: The daemon starts early in the boot process. It forces its primary Binder (`VectorService`) into `system_server` by hijacking transactions on the Android `activity` service.
2. **System Server MITM (`ipc/SystemServerService.kt`)**: To distribute IPC endpoints safely, the daemon registers a callback to intercept the registration of the hardware `serial` (or `serial_vector`) service. It proxies this service, intercepting framework-specific requests from the injected `system_server` while passing normal requests through.
3. **Application IPC (`ipc/ApplicationService.kt`)**: When a target app process forks, the Zygisk native hook contacts the daemon. The daemon provides the framework DEX via `SharedMemory`, obfuscation maps, and the list of active modules applicable to that specific process.
4. **State Management (`data/`)**: To ensure IPC calls resolve in microseconds, the daemon caches SQLite database records (modules, scopes, preferences) in memory. This cache is lazily loaded only after Android's `PackageManager` becomes available.
5. **Native Environment (`env/` & JNI)**: Background threads (C++ and Kotlin Coroutines) handle low-level system subversion, including `dex2oat` compilation hijacking and logcat monitoring.

## Directory Layout

```text
src/main/
├── kotlin/org/matrix/vector/daemon/
│   ├── core/       # Entry point (Main), looper setup, and OS broadcast receivers
│   ├── ipc/        # AIDL implementations (Manager, Module, App, SystemServer endpoints)
│   ├── data/       # SQLite DB, in-memory ConcurrentHashMap cache, File & ZIP parsing
│   ├── system/     # System binder wrappers, UID observers, Notification UI
│   ├── env/        # Socket servers and monitors communicating with JNI (dex2oat, logcat)
│   └── utils/      # OEM-specific workarounds, FakeContext, JNI bindings
└── jni/            # Native C++ layer (dex2oat wrapper, logcat watcher, slicer obfuscation)
```

## Core Technical Mechanisms

### 1. IPC Routing (The Two Doors)
* **Door 1 (`SystemServerService`)**: Used **only** by `system_server`. The Zygisk hook inside `system_server` sends a raw `_VEC` transaction to the hijacked `serial` service. The daemon intercepts this and returns the `ApplicationService` binder.
* **Door 2 (`VectorService`)**: Used by normal apps. The Java hook in normal apps sends an `Action.GET_BINDER` transaction to the `VectorService` binder (which was cached system-wide during the initial bridge injection).

### 2. AOT Compilation Hijacking (`dex2oat`)
To prevent Android's ART from inlining hooked methods (which makes them unhookable), Vector hijacks the Ahead-of-Time (AOT) compiler.
* **Mechanism**: The daemon (`Dex2OatServer`) mounts a C++ wrapper binary (`bin/dex2oatXX`) over the system's actual `dex2oat` binaries in the `/apex` mount namespace.
* **FD Passing**: When the wrapper executes, to read the original compiler or the `liboat_hook.so`, it opens a UNIX domain socket to the daemon. The daemon (running as root) opens the files and passes the File Descriptors (FDs) back to the wrapper via `SCM_RIGHTS`.
* **Execution**: The wrapper uses `memfd_create` and `sendfile` to load the hook, bypassing execute restrictions, and uses `LD_PRELOAD` to inject the hook into the real `dex2oat` process while appending `--inline-max-code-units=0`.

### 3. Dex Obfuscation & Zero-Copy Memory
The framework DEX is passed to target apps via Android's `SharedMemory` API. 
To protect Xposed API from reflections, `ObfuscationManager.kt` passes this memory buffer to JNI (`obfuscation.cpp`), which uses `slicer` to mutate Dalvik string pools in-place using `MAP_SHARED`. This ensures zero-copy manipulation; the Java side immediately sees the obfuscated DEX without reallocating buffers.

### 4. Lifecycle & State Tracking
The daemon must precisely know which apps are installed and which processes are running.
* **Broadcasts**: `VectorService` registers a hidden `IIntentReceiver` to listen for `ACTION_PACKAGE_ADDED`, `REMOVED`, and `ACTION_LOCKED_BOOT_COMPLETED`.
* **UID Observers**: `IUidObserver` tracks `onUidActive` and `onUidGone`. When a process becomes active, the daemon uses a forged `ContentProvider` call to proactively push the `IXposedService` binder into the target process, bypassing standard `bindService` limitations.

## Development & Maintenance Guidelines

When modifying the daemon, strictly adhere to the following principles:

1. **Never Block IPC Threads**: AIDL `onTransact` methods are called synchronously by the Android framework and target apps. Blocking these threads (e.g., by executing raw SQL queries or heavy I/O directly) will cause Application Not Responding (ANR) crashes system-wide. Always read from the in-memory `ConfigCache`.
2. **Resource Determinism**: The daemon runs indefinitely. Leaking a single `Cursor`, `ParcelFileDescriptor`, or `SharedMemory` instance will eventually exhaust system limits and crash the OS. Always use Kotlin's `.use { }` blocks or explicit C++ RAII wrappers for native resources.
3. **Isolate OEM Quirks**: Android OS behavior varies wildly between manufacturers (e.g., Lenovo hiding cloned apps in user IDs 900-909, MIUI killing background dual-apps). Place all OEM-specific logic in `utils/Workarounds.kt` to prevent core logic pollution.
4. **Context Forgery (`FakeContext`)**: The daemon does not have a real Android `Context`. To interact with system APIs that require one (like building Notifications or querying packages), use `FakeContext`. Be aware that standard `Context` methods may crash if not explicitly mocked.
