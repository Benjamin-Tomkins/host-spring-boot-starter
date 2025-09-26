# Host Spring Boot Starter — Product Requirements Document (PRD)

**Status:** Final

**Summary:** Minimal, modular, idiomatic integration of the OSHI library for Spring Boot applications. The starter exposes programmatic APIs only (no REST/Actuator/UI). It supplies beans and adapters that mirror a curated, stable data model of the host system (“ASCII model”), with configuration to load only the modules you enable (e.g., hardware, hardware.memory). The consuming application remains free to expose data via REST, GraphQL, CLI, RSocket, gRPC, etc.

## 1. Purpose

Provide host system facts to Spring Boot apps via a small, stable facade and DTO set, with conditional adapters (Sync / Reactor / Coroutines), optional caching and snapshotting, and native image guidance. The starter is transport-agnostic and avoids forcing runtime components.

## 2. Scope

### In Scope

*   **Auto-configuration:** System info access is exposed as beans/interfaces, not endpoints.
*   **Facade API:** Mirrors the ASCII model (`Host`, `Hardware`, `OperatingSystem` …) with selectively enabled submodules.
*   **Idiomatic Adapters (conditional):**
    *   Synchronous Java/Kotlin functions (default).
    *   Reactor (`Mono`/`Flux`) if Reactor present.
    *   Kotlin coroutines (`suspend`/`Flow`) if `kotlinx-coroutines` present.
*   **Performance:** Optional per-section caching and snapshotting to minimize OSHI call cost.
*   **GraalVM Compatibility:** `RuntimeHintsRegistrar` for starter DTOs and clear JNA/OSHI guidance.
*   **Developer Experience:** Configuration metadata, sane defaults, IDE hints, tests.

### Out of Scope

*   Any HTTP/Actuator/GraphQL/CLI endpoints.
*   Policy decisions about privacy/export (consumer decides; starter offers toggles).
*   Process management, schedulers, or background daemons (beyond optional internal snapshot timer).

## 3. Goals

*   **Modularity:** Load only what you enable (e.g., `hardware.memory`, `operating-system.file-system`).
*   **Ergonomics:** Concise, intuitive configuration; rich IDE hints.
*   **Performance:** Optional per-section TTL caches and opt-in snapshots.
*   **Idiomatic APIs:** Sync, Reactor, and Coroutine variants that mirror the same model, activated conditionally.
*   **Portability:** Works in fat JARs and native images; no mandatory runtime servers.
*   **Stability:** DTOs decoupled from OSHI; versioned “ASCII model” surface.

## 4. Non-Goals

*   No default network exposure.
*   No hard dependency on Reactor or coroutines.
*   No forced “giant model fetch” API.

## 5. Target Audience

Spring Boot developers who need host system facts with tight control over what loads and how it is consumed, while choosing their own transport/protocol.

## 6. Packaging & Versions

*   **Modules:**
    *   `host-spring-boot-autoconfigure`: auto-config, properties, DTOs, adapters, hints.
    *   `host-spring-boot-starter`: thin POM importing `…-autoconfigure` and aligning versions (no web deps).
*   **Runtime Targets:**
    *   JDK 21+
    *   Spring Boot 3.2+
    *   Kotlin 1.9+ (if used)
    *   OSHI 6.x
*   **API Compatibility:**
    *   Versioned DTO namespace: `com.yourcompany.host.api.v1`

## 7. Public API Surface

### 7.1 Facade & Providers

```kotlin
interface HostFacade {
  fun host(): Host?                          // null if Host module disabled
  fun hardware(): Hardware?
  fun operatingSystem(): OperatingSystem?
}

interface HardwareProvider {
  fun summary(): HostResult<HardwareSummary>
  fun memory(): HostResult<MemoryInfo>         // requires hardware.memory
  fun processor(): HostResult<ProcessorInfo>   // requires hardware.processor
}

interface OperatingSystemProvider {
  fun fileSystem(): HostResult<FileSystemInfo> // requires operating-system.file-system
  fun processes(limit: Int, sort: ProcessSort): HostResult<List<ProcessInfo>> // requires operating-system.processes
}
```

**Reactive adapters (conditional):**
```kotlin
interface ReactiveHardwareProvider {
  fun memory(): reactor.core.publisher.Mono<HostResult<MemoryInfo>>
  fun logicalDisks(): reactor.core.publisher.Flux<HostResult<LogicalDisk>>
}
```

**Coroutine adapters (conditional):**
```kotlin
interface CoroutineHardwareProvider {
  suspend fun memory(): HostResult<MemoryInfo>
  fun logicalDisks(): kotlinx.coroutines.flow.Flow<HostResult<LogicalDisk>>
}
```

### 7.2 DTOs (ASCII Model Mirror)

*   **Package:** `com.yourcompany.host.api.v1.*`
*   **Immutable Data Classes:** Immutable Kotlin data classes, Jackson-friendly, decoupled from OSHI.
*   **Top-Level Anchors:** `Host`, `Hardware`, `OperatingSystem`, nested per ASCII model.
*   **No Side Effects:** DTOs are pure data snapshots.

### 7.3 HostResult<T> (Value Semantics)

```kotlin
sealed interface HostResult<out T> : java.io.Serializable {
  data class Success<T>(val value: T, val sampledAt: java.time.Instant) : HostResult<T>
  data class Unavailable(val reason: UnavailableReason, val detail: String? = null) : HostResult<Nothing>
  data class Failure(val code: FailureCode, val message: String, val cause: Throwable? = null) : HostResult<Nothing>
}

enum class UnavailableReason {
  MODULE_DISABLED, NOT_SUPPORTED, PERMISSION_DENIED, NATIVE_IMAGE_UNSUPPORTED, POLICY_REDACTED, NOT_APPLICABLE
}

enum class FailureCode {
  IO_ERROR, NATIVE_CALL_ERROR, PARSE_ERROR, TIMEOUT, TRANSIENT_FAILURE, INSUFFICIENT_RESOURCES, RATE_LIMITED
}
```
**Rule:** `Unavailable` represents expected, non-exceptional absence and must **not** be thrown as an error by providers or adapters.

## 8. Configuration

**Notation:** Examples use kebab-case (Spring relaxed binding also accepts camelCase/underscores).

```properties
# Enable the starter
host.starter.enabled=true

# --- Module selection ---
# Preset initializer (none|common|all). Presets set defaults only; explicit settings override.
host.modules.preset=none

# Additive list of enabled/disabled modules (fully-qualified module ids).
# Examples: hardware, hardware.memory, operating-system.file-system, operating-system.processes
host.modules.enabled=hardware,operating-system.file-system
host.modules.disabled=

# Boolean aliases (convenience). Explicit booleans override presets; last one wins per relaxed binding.
host.modules.hardware=true
host.modules.hardware.memory=true
host.modules.operating-system.file-system=true
host.modules.operating-system.processes=false

# --- Caching (per-section) ---
# Minimum enforced TTL = 50ms. Keys include method parameters (e.g., processes(limit, sort)).
host.cache.default=300ms
host.cache.processor=500ms
host.cache.memory=250ms
host.cache.processes=2s
host.cache.disks=5s

# --- Snapshotting (consistent point-in-time view, opt-in) ---
host.snapshot.enabled=false
host.snapshot.refresh-interval=500ms
host.snapshot.max-staleness=2s

# --- Async execution safeguards ---
# Applies to all blocking OSHI calls executed by adapters (Reactor/Coroutines) and internal offloading.
host.reactive.max-concurrent=4
host.reactive.scheduler=boundedElastic   # internal default; not exposed as a public bean

# --- Security toggles for potentially sensitive information ---
host.security.include.serials=true
host.security.include.mac=true
host.security.include.uuid=true

# --- Validation behavior ---
host.validation.conflicts=warn   # fail | warn

# --- Native image hardening ---
host.native.strict=false
```

### 8.1 Merge & Validation Rules

1.  **Preset:** Initializes defaults only. Explicit properties (lists or booleans) always override.
2.  **Enabled/Disabled lists:** Resolve to a final set by `(preset defaults + enabled) - disabled`.
3.  **Booleans:** Apply after list resolution for the targeted module (last write wins).
4.  **Conflicts:**
    *   Parent disabled but child enabled → conflict (e.g., `hardware=false` & `hardware.memory=true`).
    *   OS parent disabled but submodule enabled → conflict.
    *   Native-strict mode forbids features requiring unhinted reflection/JNA → conflict.
    *   Mutually exclusive performance modes (reserved).
5.  **Behavior:**
    *   **fail** → throw `HostConfigurationException` with code, paths, actionable fix.
    *   **warn** → structured log; auto-disable conflicting children.

## 9. Auto-Configuration Behavior

*   **Registers:**
    *   A single `SystemInfo` instance.
    *   `HostFacade` and enabled module providers (sync).
    *   `Reactive*Provider` beans *only if Reactor present*.
    *   `Coroutine*Provider` beans *only if kotlinx-coroutines present*.
*   **Never registers:**
    *   Controllers/endpoints.
    *   Public schedulers or `TaskScheduler` beans.
*   **May create internally:**
    *   Bounded worker pools for offloading blocking calls (configurable via `host.reactive.*`), not exposed as beans.

## 10. Execution Model & Asynchronous Guarantees

OSHI calls are blocking. Adapters ensure they do not run on event loops:
*   **Reactor:** offload to `Schedulers.boundedElastic()` internally.
*   **Coroutines:** execute on `Dispatchers.IO`.
*   **Concurrency cap:** `host.reactive.max-concurrent` bounds parallelism across all adapter work.
*   **Sync calls:** Execute on caller threads; benefit from cache/snapshot if enabled.

## 11. Caching & Snapshot Strategy

*   Per-section caches keyed by module and method parameters (e.g., `processes(limit, sort)`).
*   TTL floor: values < 50ms are coerced to 50ms.
*   Snapshot mode: optional timer maintains a consistent view for its TTL window.
*   DTO freshness: DTOs carry `sampledAt` timestamps.

## 12. GraalVM Native Image

*   Provide a `RuntimeHintsRegistrar` covering DTOs and basic reflection.
*   Document OSHI/JNA requirements and offer a playbook:
    *   Add reachability metadata where needed.
    *   Toggle off fragile sub-features (e.g., `hardware.displays`) if builds fail.
    *   Use `host.native.strict=true` to forbid features lacking hints (fail early).
*   Modular design allows feature-based workarounds.

## 13. Security & Privacy

*   The starter does not expose data over the network.
*   Field-level policy by default: when `host.security.include.*=false`, omit/null only the sensitive fields in `Success`.
*   Use `Unavailable(POLICY_REDACTED)` only when an entire section must be suppressed.
*   Surface policy decisions as values (not exceptions) to keep semantics predictable.

## 14. Observability (Optional)

Optionally publish a minimal set of Micrometer meters (CPU load, memory used/total, disk used/total). This is disabled by default; enabling it does not imply HTTP exposure or add registries.

```yaml
host.metrics.enabled: false
host.metrics.set: basic   # basic|minimal|none
```

## 15. Testing & Quality

*   **Auto-config tests** (`ApplicationContextRunner`):
    *   Module enable/disable matrix
    *   Presence/absence of Reactor & coroutines
    *   Cache and snapshot toggles
    *   Validation behavior (fail vs warn)
*   **Contract tests** for DTO stability (JSON snapshots).
*   **OS matrix CI:** Linux/macOS/Windows; JDK 21 (and 25 when available).
*   **Native image smoke test** with minimal module set.
*   **Determinism:** Inject an internal `Clock` so `sampledAt` can be stabilized in tests.
*   **Configuration metadata:** Ensure `spring-configuration-metadata.json` is generated with value hints.

## 16. Example Usage (Transport-Agnostic)

Providers/adapters return `HostResult<T>`; `Unavailable` is a value, not an error.

### 16.1 REST (Spring Web)

```kotlin
@RestController
class MyController(private val hardware: HardwareProvider) {

  @GetMapping("/memory")
  fun memory() = hardware.memory().toResponseEntity()

  @GetMapping("/memory-summary")
  fun memorySummary() = hardware.memory().toResponseEntity(
    successMapper = { memory -> mapOf("usedPercent" to (memory.used * 100.0 / memory.total)) }
  )

  // Full control
  @GetMapping("/memory-detailed")
  fun memoryDetailed(): ResponseEntity<*> =
    when (val r = hardware.memory()) {
      is HostResult.Success -> ResponseEntity.ok(r.value)
      is HostResult.Unavailable -> ResponseEntity.status(503)
        .body(mapOf("status" to "UNAVAILABLE", "reason" to r.reason.name, "detail" to r.detail))
      is HostResult.Failure -> ResponseEntity.status(500)
        .body(mapOf("error" to r.message, "code" to r.code.name))
    }
}
```

### 16.2 GraphQL (Union Result; no throwing on Unavailable)

```kotlin
@Component
class SystemGraphQLResolver(private val hardware: HardwareProvider) {

  @QueryMapping
  fun systemMemory(): MemoryResult =
    when (val r = hardware.memory()) {
      is HostResult.Success     -> MemoryData(r.value)
      is HostResult.Unavailable -> MemoryUnavailable(reason = r.reason.name, detail = r.detail)
      is HostResult.Failure     -> MemoryError(code = r.code.name, message = r.message)
    }
}

// schema: union MemoryResult = MemoryData | MemoryUnavailable | MemoryError
```

### 16.3 RSocket (Reactive)

```kotlin
@Controller
class SystemRSocketController(private val hw: ReactiveHardwareProvider) {

  @MessageMapping("system.memory")
  fun memory(): Mono<MemoryResponse> =
    hw.memory().map { r ->
      when (r) {
        is HostResult.Success     -> MemoryResponse.success(r.value)
        is HostResult.Unavailable -> MemoryResponse.unavailable(r.reason)
        is HostResult.Failure     -> MemoryResponse.error(r.code)
      }
    }

  @MessageMapping("system.memory.stream")
  fun memoryStream(): Flux<HostResult<MemoryInfo>> =
    Flux.interval(Duration.ofSeconds(1)).flatMap { hw.memory() }
}
```

### 16.4 gRPC

```kotlin
@GrpcService
class SystemGrpcService(private val hardware: HardwareProvider) : SystemServiceGrpc {
  override fun getMemory(request: Empty): MemoryResponse =
    when (val r = hardware.memory()) {
      is HostResult.Success -> MemoryResponse.newBuilder().setSuccess(true).setMemoryInfo(r.value.toProto()).build()
      is HostResult.Unavailable -> MemoryResponse.newBuilder().setSuccess(false)
        .setError(ErrorInfo.newBuilder().setCode(ErrorCode.UNAVAILABLE).setReason(r.reason.toString()).build()).build()
      is HostResult.Failure -> MemoryResponse.newBuilder().setSuccess(false)
        .setError(ErrorInfo.newBuilder().setCode(ErrorCode.INTERNAL).setMessage(r.message).build()).build()
    }
}
```

### 16.5 Convenience Extensions

```kotlin
// Core extensions
fun <T> HostResult<T>.getOrNull(): T?
fun <T> HostResult<T>.getOrDefault(default: T): T
fun <T> HostResult<T>.getOrThrow(): T
fun <T, R> HostResult<T>.map(transform: (T) -> R): HostResult<R>
fun <T, R> HostResult<T>.flatMap(transform: (T) -> HostResult<R>): HostResult<R>

// REST-specific
fun <T> HostResult<T>.toResponseEntity(
  successMapper: (T) -> Any = { it },
  unavailableHandler: (HostResult.Unavailable) -> ResponseEntity<*> = { u ->
    ResponseEntity.status(503).body(mapOf("status" to "UNAVAILABLE", "reason" to u.reason.name, "detail" to u.detail))
  },
  failureHandler: (HostResult.Failure) -> ResponseEntity<*> = { f ->
    ResponseEntity.status(500).body(mapOf("error" to f.message, "code" to f.code.name))
  }
): ResponseEntity<*>
```

## 17. Lifecycle & Thread-Safety

*   Single `SystemInfo` bean reused; OSHI types are treated as read-only within the starter.
*   Internal caches/snapshots are thread-safe; concurrency bounded by configuration.
*   `Clock` injectable to aid deterministic tests.

## 18. Risks & Mitigations

*   **Platform variability:** Covered by OSHI; CI matrix across Linux/macOS/Windows.
*   **Native image fragility (JNA):** Hints + modular toggles + `host.native.strict`.
*   **Event-loop safety:** Internal offloading + concurrency caps.
*   **Configuration complexity:** Presets + lists + boolean aliases; config metadata for IDE hints.

## 19. Acceptance Criteria

*   No HTTP/Actuator/GraphQL/CLI beans are auto-registered.
*   Enabling `host.modules.*` yields only the corresponding provider beans.
*   Reactor/coroutine adapters appear only when their libraries are present.
*   DTOs are stable and versioned; contract tests pass.
*   Native image sample builds with a basic module set and returns expected values.
*   Validation behaves per `host.validation.conflicts` (fail or warn) with clear diagnostics.

---

## Appendix A — Suggested Property Aliases

```yaml
# Shorthand booleans (aliases to the enabled list)
host.modules.hardware: true
host.modules.hardware.memory: true
host.modules.hardware.processor: true
host.modules.operating-system.file-system: true
```

## Appendix B — Minimal POM (Starter)

```xml
<dependencyManagement>
  <!-- aligns versions; no web deps -->
</dependencyManagement>

<dependencies>
  <dependency>
    <groupId>com.yourcompany</groupId>
    <artifactId>host-spring-boot-autoconfigure</artifactId>
  </dependency>
  <dependency>
    <groupId>com.github.oshi</groupId>
    <artifactId>oshi-core</artifactId>
  </dependency>
  <!-- Reactor / Coroutines optional; adapters load conditionally -->
</dependencies>
```

## Appendix C — ASCII Data Model

This model illustrates the DTO structure and its mapping from the underlying OSHI library.

```
Host
├── Manufacturer              (oshi.hardware.ComputerSystem#getManufacturer)
├── ProductName               (oshi.hardware.ComputerSystem#getModel)
├── SerialNumber              (oshi.hardware.ComputerSystem#getSerialNumber)
├── HardwareUUID              (oshi.hardware.ComputerSystem#getHardwareUUID)
├── Hardware
│   ├── Motherboard           (oshi.hardware.ComputerSystem#getBaseboard)
│   │   ├── Manufacturer      (oshi.hardware.Baseboard#getManufacturer)
│   │   ├── ProductName       (oshi.hardware.Baseboard#getModel)
│   │   ├── Version           (oshi.hardware.Baseboard#getVersion)
│   │   └── SerialNumber      (oshi.hardware.Baseboard#getSerialNumber)
│   │
│   ├── Firmware              (oshi.hardware.ComputerSystem#getFirmware)
│   │   ├── Manufacturer      (oshi.hardware.Firmware#getManufacturer)
│   │   ├── Name              (oshi.hardware.Firmware#getName)
│   │   ├── Description       (oshi.hardware.Firmware#getDescription)
│   │   ├── Version           (oshi.hardware.Firmware#getVersion)
│   │   └── ReleaseDate       (oshi.hardware.Firmware#getReleaseDate)
│   │
│   ├── Processor             (oshi.hardware.HardwareAbstractionLayer#getProcessor)
│   │   ├── Name              (oshi.hardware.CentralProcessor.ProcessorIdentifier#getName)
│   │   ├── Manufacturer      (oshi.hardware.CentralProcessor.ProcessorIdentifier#getVendor)
│   │   ├── MaxClockSpeed (MHz)(oshi.hardware.CentralProcessor#getMaxFreq)
│   │   ├── PhysicalCoreCount (oshi.hardware.CentralProcessor#getPhysicalProcessorCount)
│   │   ├── LogicalProcessorCount(oshi.hardware.CentralProcessor#getLogicalProcessorCount)
│   │   ├── Identifier        (oshi.hardware.CentralProcessor.ProcessorIdentifier#getIdentifier)
│   │   ├── ProcessorId       (oshi.hardware.CentralProcessor.ProcessorIdentifier#getProcessorID)
│   │   ├── ContextSwitches   (oshi.hardware.CentralProcessor#getContextSwitches)
│   │   ├── Interrupts        (oshi.hardware.CentralProcessor#getInterrupts)
│   │   ├── Caches [List]     (oshi.hardware.CentralProcessor#getProcessorCaches)
│   │   │   └── ProcessorCache (nested)
│   │   │       ├── Level         (oshi.hardware.CentralProcessor.ProcessorCache#getLevel)
│   │   │       ├── Type          (oshi.hardware.CentralProcessor.ProcessorCache#getType)
│   │   │       ├── Size          (oshi.hardware.CentralProcessor.ProcessorCache#getCacheSize)
│   │   │       └── Associativity (oshi.hardware.CentralProcessor.ProcessorCache#getAssociativity)
│   │   └── LogicalProcessors [List](oshi.hardware.CentralProcessor#getLogicalProcessors)
│   │       └── LogicalProcessor (nested)
│   │           ├── ProcessorNumber (oshi.hardware.CentralProcessor.LogicalProcessor#getProcessorNumber)
│   │           ├── CoreNumber    (oshi.hardware.CentralProcessor.LogicalProcessor#getPhysicalProcessorNumber)
│   │           └── PackageNumber (oshi.hardware.CentralProcessor.LogicalProcessor#getPhysicalPackageNumber)
│   │
│   ├── Memory                (oshi.hardware.HardwareAbstractionLayer#getMemory)
│   │   ├── Total             (oshi.hardware.GlobalMemory#getTotal)
│   │   ├── Available         (oshi.hardware.GlobalMemory#getAvailable)
│   │   ├── PageSize          (oshi.hardware.GlobalMemory#getPageSize)
│   │   └── VirtualMemory (nested)(oshi.hardware.GlobalMemory#getVirtualMemory)
│   │       ├── SwapTotal     (oshi.hardware.VirtualMemory#getSwapTotal)
│   │       ├── SwapUsed      (oshi.hardware.VirtualMemory#getSwapUsed)
│   │       ├── SwapPagesIn   (oshi.hardware.VirtualMemory#getSwapPagesIn)
│   │       └── SwapPagesOut  (oshi.hardware.VirtualMemory#getSwapPagesOut)
│   │
│   ├── PhysicalMemory [List] (oshi.hardware.GlobalMemory#getPhysicalMemory)
│   │   └── PhysicalMemory
│   │       ├── BankLabel     (oshi.hardware.PhysicalMemory#getBankLabel)
│   │       ├── Capacity      (oshi.hardware.PhysicalMemory#getCapacity)
│   │       ├── ClockSpeed    (oshi.hardware.PhysicalMemory#getClockSpeed)
│   │       ├── Manufacturer  (oshi.hardware.PhysicalMemory#getManufacturer)
│   │       └── MemoryType    (oshi.hardware.PhysicalMemory#getMemoryType)
│   │
│   ├── DiskDrives [List]     (oshi.hardware.HardwareAbstractionLayer#getDiskStores)
│   │   └── DiskDrive
│   │       ├── Name          (oshi.hardware.HWDiskStore#getName)
│   │       ├── Model         (oshi.hardware.HWDiskStore#getModel)
│   │       ├── SerialNumber  (oshi.hardware.HWDiskStore#getSerial)
│   │       ├── Size          (oshi.hardware.HWDiskStore#getSize)
│   │       ├── Reads         (oshi.hardware.HWDiskStore#getReads)
│   │       ├── Writes        (oshi.hardware.HWDiskStore#getWrites)
│   │       └── Partitions [List](oshi.hardware.HWDiskStore#getPartitions)
│   │           └── Partition (nested)
│   │               ├── Identification (oshi.hardware.HWPartition#getIdentification)
│   │               ├── Name      (oshi.hardware.HWPartition#getName)
│   │               ├── Type      (oshi.hardware.HWPartition#getType)
│   │               ├── MountPoint(oshi.hardware.HWPartition#getMountPoint)
│   │               ├── Size      (oshi.hardware.HWPartition#getSize)
│   │               └── UUID      (oshi.hardware.HWPartition#getUuid)
│   │
│   ├── NetworkAdapters [List](oshi.hardware.HardwareAbstractionLayer#getNetworkIFs)
│   │   └── NetworkAdapter
│   │       ├── Name          (oshi.hardware.NetworkIF#getName)
│   │       ├── DisplayName   (oshi.hardware.NetworkIF#getDisplayName)
│   │       ├── MACAddress    (oshi.hardware.NetworkIF#getMacaddr)
│   │       ├── IPv4Addresses [List of String](oshi.hardware.NetworkIF#getIPv4addr)
│   │       ├── IPv6Addresses [List of String](oshi.hardware.NetworkIF#getIPv6addr)
│   │       ├── Speed         (oshi.hardware.NetworkIF#getSpeed)
│   │       ├── BytesSent     (oshi.hardware.NetworkIF#getBytesSent)
│   │       ├── BytesReceived (oshi.hardware.NetworkIF#getBytesRecv)
│   │       └── OperStatus    (oshi.hardware.NetworkIF#getIfOperStatus)
│   │
│   ├── PowerSources [List]   (oshi.hardware.HardwareAbstractionLayer#getPowerSources)
│   │   └── PowerSource
│   │       ├── Name          (oshi.hardware.PowerSource#getName)
│   │       ├── DeviceName    (oshi.hardware.PowerSource#getDeviceName)
│   │       ├── RemainingCapacityPercent (oshi.hardware.PowerSource#getRemainingCapacityPercent)
│   │       ├── TimeRemainingEstimated (oshi.hardware.PowerSource#getTimeRemainingEstimated)
│   │       ├── PowerOnLine   (oshi.hardware.PowerSource#isPowerOnLine)
│   │       ├── Charging      (oshi.hardware.PowerSource#isCharging)
│   │       ├── Discharging   (oshi.hardware.PowerSource#isDischarging)
│   │       ├── Voltage       (oshi.hardware.PowerSource#getVoltage)
│   │       └── Amperage      (oshi.hardware.PowerSource#getAmperage)
│   │
│   ├── Displays [List]       (oshi.hardware.HardwareAbstractionLayer#getDisplays)
│   │   └── Display
│   │       ├── DisplayID     (Construct from oshi.util.EdidUtil)
│   │       └── EDID          (oshi.hardware.Display#getEdid)
│   │
│   ├── UsbDevices [List]     (oshi.hardware.HardwareAbstractionLayer#getUsbDevices)
│   │   └── UsbDevice
│   │       ├── Name          (oshi.hardware.UsbDevice#getName)
│   │       ├── Vendor        (oshi.hardware.UsbDevice#getVendor)
│   │       ├── VendorId      (oshi.hardware.UsbDevice#getVendorId)
│   │       ├── ProductId     (oshi.hardware.UsbDevice#getProductId)
│   │       ├── SerialNumber  (oshi.hardware.UsbDevice#getSerialNumber)
│   │       └── ConnectedDevices [List of UsbDevice] (recursive) (oshi.hardware.UsbDevice#getConnectedDevices)
│   │
│   └── Sensors               (oshi.hardware.HardwareAbstractionLayer#getSensors)
│       ├── CpuTemperature    (oshi.hardware.Sensors#getCpuTemperature)
│       └── FanSpeeds [List of Integer] (oshi.hardware.Sensors#getFanSpeeds)
│
└── OperatingSystem           (oshi.SystemInfo#getOperatingSystem)
    ├── Family                (oshi.software.os.OperatingSystem#getFamily)
    ├── Manufacturer          (oshi.software.os.OperatingSystem#getManufacturer)
    ├── Architecture          (oshi.software.os.OperatingSystem#getBitness)
    ├── SystemUptime          (oshi.software.os.OperatingSystem#getSystemUptime)
    ├── BootTime              (oshi.software.os.OperatingSystem#getSystemBootTime)
    ├── ProcessCount          (oshi.software.os.OperatingSystem#getProcessCount)
    ├── ThreadCount           (oshi.software.os.OperatingSystem#getThreadCount)
    ├── VersionInfo (nested)  (oshi.software.os.OperatingSystem#getVersionInfo)
    │   ├── Version           (oshi.software.os.OperatingSystem.OSVersionInfo#getVersion)
    │   ├── CodeName          (oshi.software.os.OperatingSystem.OSVersionInfo#getCodeName)
    │   └── BuildNumber       (oshi.software.os.OperatingSystem.OSVersionInfo#getBuildNumber)
    │
    ├── FileSystem            (oshi.software.os.OperatingSystem#getFileSystem)
    │   ├── MaxFileDescriptors (oshi.software.os.FileSystem#getMaxFileDescriptors)
    │   └── OpenFileDescriptors(oshi.software.os.FileSystem#getOpenFileDescriptors)
    │
    ├── LogicalDisks [List]   (oshi.software.os.FileSystem#getFileStores)
    │   └── LogicalDisk
    │       ├── Name          (oshi.software.os.OSFileStore#getName)
    │       ├── Volume        (oshi.software.os.OSFileStore#getVolume)
    │       ├── Mount         (oshi.software.os.OSFileStore#getMount)
    │       ├── Description   (oshi.software.os.OSFileStore#getDescription)
    │       ├── Type          (oshi.software.os.OSFileStore#getType)
    │       ├── TotalSpace    (oshi.software.os.OSFileStore#getTotalSpace)
    │       ├── FreeSpace     (oshi.software.os.OSFileStore#getFreeSpace)
    │       ├── UsableSpace   (oshi.software.os.OSFileStore#getUsableSpace)
    │       └── TotalInodes   (oshi.software.os.OSFileStore#getTotalInodes)
    │
    ├── NetworkParameters     (oshi.software.os.OperatingSystem#getNetworkParams)
    │   ├── HostName          (oshi.software.os.NetworkParams#getHostName)
    │   ├── DomainName        (oshi.software.os.NetworkParams#getDomainName)
    │   └── DnsServers [List of String] (oshi.software.os.NetworkParams#getDnsServers)
    │
    ├── Processes [List]      (oshi.software.os.OperatingSystem#getProcesses)
    │   └── Process
    │       ├── Name          (oshi.software.os.OSProcess#getName)
    │       ├── ProcessID     (oshi.software.os.OSProcess#getProcessID)
    │       ├── ParentProcessID(oshi.software.os.OSProcess#getParentProcessID)
    │       ├── Path          (oshi.software.os.OSProcess#getPath)
    │       ├── State         (oshi.software.os.OSProcess#getState)
    │       ├── User          (oshi.software.os.OSProcess#getUser)
    │       ├── UserID        (oshi.software.os.OSProcess#getUserID)
    │       ├── Group         (oshi.software.os.OSProcess#getGroup)
    │       ├── GroupID       (oshi.software.os.OSProcess#getGroupID)
    │       ├── KernelTime    (oshi.software.os.OSProcess#getKernelTime)
    │       ├── UserTime      (oshi.software.os.OSProcess#getUserTime)
    │       ├── UpTime        (oshi.software.os.OSProcess#getUpTime)
    │       ├── StartTime     (oshi.software.os.OSProcess#getStartTime)
    │       ├── VirtualSize   (oshi.software.os.OSProcess#getVirtualSize)
    │       └── ResidentSetSize(oshi.software.os.OSProcess#getResidentSetSize)
    │
    └── Services [List]       (oshi.software.os.OperatingSystem#getServices)
        └── Service
            ├── Name          (oshi.software.os.OSService#getName)
            ├── ProcessID     (oshi.software.os.OSService#getProcessID)
            └── State         (oshi.software.os.OSService#getState)
```
