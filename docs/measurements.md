# Measurement guide

## Clock, precision and statistics

Software intervals use QueryPerformanceCounter (QPC). The optional probe uses the matching kernel counter, and kernel ETW is requested with raw QPC timestamps. Counter frequency is exported; clock resolution does not imply equal measurement accuracy. Thread scheduling, API buffering, instrumentation overhead and missing events can affect observations.

Each series retains its latest 1,024 samples. Percentiles use linear interpolation within that window. Session counts cover all accepted samples since the current capture started, while exported samples and statistics cover only the retained window. No-sample statistics are JSON `null`. Negative, nonfinite, malformed and out-of-bounds samples are rejected. A process/target change starts a new capture instead of mixing different targets.

Raw Input, XInput, ETW, PresentMon and the optional kernel probe observe different stages. Their values are not additive: subtracting unrelated samples does not produce a valid end-to-end latency. Do not sum p99 values across sources. Correlate timestamps and workloads before drawing conclusions.

## Input devices

**Raw Input:** time between messages arriving on a dedicated observer thread, plus partial message-handler cost. Device discovery does not require touching a controller. Move a device to generate intervals. An idle period can produce a long interval; this is not necessarily a dropped report or a slow device. Windows may batch or suppress messages.

**XInput:** state API duration and elapsed time between observed packet-number changes. Connected slots are queried about every fourth 1 ms timer wake; actual sampling depends on Windows scheduling and API execution. Empty slots are retried at most every 250 ms. Intervals include this sampling uncertainty. XInput exposes a cached API view, not a guaranteed physical device identity or hardware polling timestamp. A slot can refer to the same controller as a HID view. GIP and XUSB transport observations are available only where the Windows stack/filter exposes them; unsupported paths remain unmeasured.

**Optional probe:** interval from forwarding a read/USB IN request until its successful completion, intervals between successful completions at that node, partial metadata-recording cost, and failure/cancellation flags. A pending request can simply be waiting for a future input. Multiple transport endpoints can contribute to one node; the completion interval is not guaranteed hardware polling rate. No packet contents are used to guess a kernel-to-application match.

Device IDs are capture-local views. PnP paths identify Windows nodes, not a guaranteed count of physical controllers. Parent transport nodes, HID collections and XInput slots may refer to one physical device. Bluetooth/USB firmware, radio and physical-switch delays before a Windows observation are outside this measurement.

## Frames, CPU and GPU

### Application stage markers

The Game pipeline view consumes existing PCL/Reflex TraceLogging and Intel PresentMon application timing events. It never injects into a game or generates input. PCL event markers 0..5 identify simulation start/end, render submission start/end and Present start/end. Intel events can additionally provide sleep start/end and an application input-sample marker. PCL synthetic latency-ping markers are deliberately excluded from physical-input measurements.

The collector does not generate diagnostic pings itself. A game's own PC Latency Stats instrumentation can generate its SDK's ping messages while statistics are enabled; those are not treated as physical mouse, keyboard or controller reports. A PCL initialization event resets that process/provider's pending pairs so markers are not paired across instrumentation reinitialization.

Pairing requires the same process ID, provider and frame ID. PCL frame IDs retain all 64 bits; exported frame IDs are decimal strings to avoid JSON/JavaScript integer rounding. Providers and processes are never merged solely because timestamps are close. Missing, unsupported or reversed boundaries do not create a zero-duration sample. Identical duplicate timestamps are ignored; contradictory duplicates flag the frame as conflicting. The first valid historical pairs may remain in aggregate statistics, so review conflicts/coverage before interpreting them. Pending frame state is bounded to 512 records and the report keeps the latest 64 observed frame records.

The view shows simulation, simulation-to-submission gap, render submission, submission-to-Present gap, Present, sleep, input-sample-to-Present and an aggregate simulation-start-to-Present-end interval when the required boundaries exist. These are elapsed wall-clock intervals around application-defined work and waits. They are not exclusive CPU execution times, and overlapping aggregate/child intervals must not be added.

An application input-sample marker is not a USB reception timestamp and does not identify a particular physical report. This view does not correlate a marker frame with a specific PresentMon GPU/display record. USB-to-game latency, each third-party filter's per-input latency, physical display response and uninstrumented internal game functions remain unmeasured. Application instrumentation must be correctly placed by its author; registering a provider does not verify that placement. The pipeline collector has been validated with an owned marker emitter and schema/identity tests, not Fortnite or every commercial game.

PresentMon uses graphics ETW events, separated by PID and swap chain. Some durations are reconstructed and require supported drivers, display events or application instrumentation. Blank fields remain unmeasured. A process may expose several swap chains. A reported display event is not a light sensor reading; monitor scan-out and pixel transition behavior cannot be independently verified here.

If the full graphics pipeline produces no rows for ten seconds, capture automatically retries with CPU/Present-only tracing. This helps graphics stacks that do not provide the required GPU/display events, including some virtual machines. In that mode only CPU busy/wait, presentation interval and Present API duration are collected; GPU/display/input-to-display fields are excluded. Report & coverage identifies the fallback. Selecting a game or starting a new session retries full tracing.

Kernel ISR/DPC durations describe system activity. ReadyThread/CSwitch pairs describe observed waits between a thread becoming ready and running. Only paired, known transitions contribute. Events lost by ETW or unobserved transitions limit coverage. These numbers do not by themselves establish that a game or input was delayed. Thread filtering uses known thread ownership; unowned events are not guessed.

Driver attribution resolves the DPC/ISR entry routine address to a loaded kernel image range. An initial driver snapshot is supplemented by supported kernel image load/unload events. Unknown addresses remain unresolved and contribute to an unresolved-attribution flag. Per-driver values are inclusive elapsed routine intervals in microseconds and can include nested interrupts/preemption. They are not exclusive driver CPU time, not request residence time and not a critical-path allocation to the selected game or a particular input. Framework entry routines can resolve to the framework module instead of the vendor callback it invokes. Drivers that process input without a DPC/ISR are not covered by this breakdown.

Timer wake lateness measures the inspector worker's lateness against a nominal 1 ms timer schedule. It includes collector execution and host scheduling effects. It is not the selected game's scheduling delay. CPU percentages are normalized across logical processors. Inspector CPU/memory measurements exclude its separate PresentMon process and some kernel trace overhead.

## Network, storage and audio

ICMP RTT is Windows' millisecond-granularity echo result. API elapsed time also includes call/wakeup overhead. Variation is the absolute difference between consecutive successful RTTs. Unanswered probes are a rolling percentage over at most 60 probes; ICMP filtering/rate limiting can explain them. TCP setup is a separate synthetic connection to port 443, not a game request. No automatic game-server attribution, one-way latency or game packet-loss claim is made.

Disk timing uses DiskIo HighResResponseTime: the observed I/O response interval including queueing. It is not solely media access time. Per-process attribution requires a known issuing thread. Process I/O volume includes cached/non-disk operations and is not a latency. Hard-fault completion is system scoped.

WASAPI endpoint periods are reported without opening a playback or recording stream. Default/minimum supported periods and the current shared-engine period are not measured physical audio output, microphone latency or a game's active end-to-end audio buffer.

## Overhead and validation

All software observers add overhead. Capture uses bounded stores, separate worker threads and independent UI updates; it does not change CPU affinity, game priority, system timer policy or input contents. High-volume ETW sessions and PresentMon can still affect a workload. Compare runs under similar conditions and watch the inspector cost and coverage notices.

The initial release was built with warnings treated as errors and tested for bounded/concurrent sample storage, statistics, report encoding and collector start/stop. The passive probe was exercised in an isolated Windows 11 VM with a local laboratory certificate, malformed IOCTL tests and Driver Verifier. Laboratory signing is not Microsoft signing. A VM does not validate every physical controller, Bluetooth stack, GPU, game, monitor or audio endpoint. Physical input-to-photon, optical and acoustic accuracy are not asserted.

## Primary references

- [Microsoft: high-resolution timestamps](https://learn.microsoft.com/en-us/windows/win32/sysinfo/acquiring-high-resolution-time-stamps)
- [Microsoft: Raw Input](https://learn.microsoft.com/en-us/windows/win32/inputdev/raw-input)
- [Microsoft: DiskIo event data](https://learn.microsoft.com/en-us/windows/win32/etw/diskio-typegroup1)
- [Microsoft: low-latency audio](https://learn.microsoft.com/en-us/windows-hardware/drivers/audio/low-latency-audio)
- [Intel: PresentMon console metrics](https://github.com/GameTechDev/PresentMon/blob/v2.6.0/README-ConsoleApplication.md)
- [NVIDIA: PC latency markers and their measurement boundaries](https://developer.nvidia.com/blog/?p=64245)
- [Microsoft: DPC/ISR attribution by module and inclusive/exclusive duration](https://learn.microsoft.com/en-us/windows-hardware/test/wpt/cpu-analysis)
- [Intel: public PCL marker schema](https://github.com/GameTechDev/PresentMon/blob/v2.6.0/PresentData/ETW/Nvidia_PCL.h)
