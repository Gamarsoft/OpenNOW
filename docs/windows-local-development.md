# Build and test on this Windows PC

This runbook applies to `E:\DEV\OpenNOW` on the current Windows 11 PC. It documents the
existing local environment; it is not an installer or a setup recipe for another machine.
The upstream architecture and ownership rules in [AGENTS.md](../AGENTS.md) still apply.

## Phase 3 completion record

Phase 3 of `E:\DEV\OpenNOW-Windows-E-Drive-Codex-Prompt.md` is complete for establishing
the local native build environment and recording the unchanged Release baseline. That
baseline compiled, but its selected tests returned 33 passes and 359 failures. The later
approved startup, timeout, locale, and settings changes have separate validation records.
Do not describe the unchanged upstream tests as passing.

As recorded on 12 October 2026, development is on `codex/settings-startup-latency` at
`bee18c118dbc89f42319436dcdb172d5b9e15e0c` with uncommitted source fixes. Main remains at
that commit. Check Git status before each batch; these are dated facts.

## Existing environment and helpers

| Purpose | Existing location |
| --- | --- |
| Source | `E:\DEV\OpenNOW` |
| CMake build tree | `E:\DEV\b\OpenNOW` |
| Release executable and launch directory | `E:\DEV\b\OpenNOW\Release\OpenNOW.exe` and its parent |
| Local PowerShell helpers | `E:\DEV\Tools\OpenNOW` |
| Development temporary files | `E:\DEV\Temp` |
| Unique command logs, JSON reports, and JUnit results | `E:\DEV\Temp\OpenNOW` |

[The helper README](E:/DEV/Tools/OpenNOW/README.md) lists the verified versions and paths:
VS 2022 Build Tools 17.14.41/MSVC 19.44.35229.0, Windows SDK 10.0.26100.0,
CMake 3.31.10, Ninja 1.12.1, Rust 1.99.0 MSVC x64, LLVM/libclang 20.1.8,
Qt 6.8.3 MSVC x64 with private headers, and the shared SDL 3.2.20 Release SDK.
Reuse these installations. Run Doctor after toolchain changes or when environment selection
is uncertain; inspect its result instead of relying on the table.

| Helper | Responsibility |
| --- | --- |
| `Environment.ps1` | Process-only x64 MSVC, Rust, libclang, PATH, data, and temporary paths |
| `NativeCommand.ps1` | Native exit codes, unique logs capped at 10 MiB each, omitted-line metadata |
| `Doctor.ps1` | Tool paths, architectures, versions, SDK, Qt private targets, and dependency checks |
| `Build.ps1` | Configure Ninja Multi-Config, then build Release with two jobs and record the executable hash |
| `Test.ps1` | Localization first, then the selected software CTest suite; JUnit and skip reporting |

The helpers live outside this repository and are not included in a clone. Preserve them and
their verification records. Do not dot-source Doctor, Build, or Test into a daily shell:
their entry scripts call `exit`. Dot-source shared implementation files only in a dedicated
PowerShell process. CMake owns its managed Cargo output directories; initialization rejects
an existing `CARGO_TARGET_DIR` override.

## Configure, build, and run software tests

Use the installed PowerShell 7 executable. Each invocation below creates its own environment.
Stop on a nonzero exit code before continuing to the next command.

~~~powershell
$openNowPwsh = 'C:\Users\xpont\.cache\codex-runtimes\codex-primary-runtime\dependencies\native\powershell\pwsh.exe'
& $openNowPwsh -NoProfile -File 'E:\DEV\Tools\OpenNOW\Doctor.ps1'
if ($LASTEXITCODE -ne 0) { throw 'OpenNOW environment checks failed.' }

& $openNowPwsh -NoProfile -File 'E:\DEV\Tools\OpenNOW\Build.ps1'
if ($LASTEXITCODE -ne 0) { throw 'OpenNOW Release build failed.' }

# Run the narrowest test relevant to the change before this full software selection.
& $openNowPwsh -NoProfile -File 'E:\DEV\Tools\OpenNOW\Test.ps1'
if ($LASTEXITCODE -ne 0) { throw 'OpenNOW software tests failed.' }
~~~

Build always configures the existing tree with `BUILD_TESTING=ON`, explicit Qt/SDL prefixes,
and the real MSVC compiler. Its native build is `cmake --build E:\DEV\b\OpenNOW --config Release --parallel 2`.
Use Release for performance measurements. Add Debug only when needed.

Test sets `LC_ALL=LANG=en_US`, `QT_QUICK_BACKEND=software`, and the repository font directory
for its process. English is required by the existing acceptance fixtures; the PC's default
Qt locale is French. It excludes the `interactive-desktop` label and the FSR upscaler,
frame interpolator, and stream-color targets. HDR-color is excluded by that label.
The helper launches automated, account-free OpenNOW acceptance processes.

For example, run these focused settings and profiler checks in a separate PowerShell 7 process
after building, then run Test above. Replace the regex with the narrowest relevant selection
for another change.

~~~powershell
& $openNowPwsh -NoProfile -Command {
    . 'E:\DEV\Tools\OpenNOW\Environment.ps1'
    . 'E:\DEV\Tools\OpenNOW\NativeCommand.ps1'
    Initialize-OpenNowEnvironment
    $env:LC_ALL = 'en_US'
    $env:LANG = 'en_US'
    $env:QT_QUICK_BACKEND = 'software'
    $env:QT_QPA_FONTDIR = 'E:\DEV\OpenNOW\opennow-qt\res\fonts'
    $result = Invoke-OpenNowNative -FilePath $OpenNow.CTest -Name 'test-settings-focused' -Arguments @(
        '--test-dir', $OpenNow.Build, '-C', 'Release',
        '--output-on-failure', '--no-tests=error', '--parallel', '2',
        '-R', '^(qml-settings-motion-|opennow-performance-)'
    )
    exit $result.exitCode
}
if ($LASTEXITCODE -ne 0) { throw 'Focused OpenNOW tests failed.' }
~~~

Do not treat an empty selection or compilation alone as a passing runtime check.
Report CTest target skips separately from internal Qt case skips. Run the affected Rust
crate/workspace tests for Rust changes and localization checks for localization changes,
as required by AGENTS.md; the software helper does not run Rust workspace suites.

## Native measurements and application launches

Follow [Qt acceptance](qt-acceptance.md) for native workloads after the relevant batch is
authorized. Start from a fresh process, remove inherited software-renderer overrides
(`QT_QUICK_BACKEND` and `QT_OPENGL`), and use the Windows platform plugin. Verify the actual
GPU/backend, dimensions, DPR, display refresh, action acceptance, and focus in the report.
Use a new E: output directory for each batch. Keep the measured refresh-rate budgets;
never change display settings or inflate budgets to obtain a pass.

Build before restarting the app. Stop only OpenNOW processes belonging to this workspace.
Launch `E:\DEV\b\OpenNOW\Release\OpenNOW.exe` with `E:\DEV\b\OpenNOW\Release` as its working
directory so the bundled runtime libraries and core resolve correctly.

Application startup writes the per-user `opennow:` protocol association to the executable
being launched, including during acceptance runs. The approved association points to the
usual Release executable. Inspect it and request approval before launching another executable
path that would change it. Historical baseline executables and diagnostic runners are evidence,
not interchangeable everyday launchers. The archived native runner also guards a specific
executable hash; inspect it before reuse after rebuilding.

Hardware fixtures and interactive launches need approval unless already covered by the
current batch. NVIDIA/GFN login and gameplay need explicit authorization. Qt native shell
tests do not establish streaming decode, physical HDR accuracy, or gameplay input correctness.

## Evidence and remaining validation

| Record | Local file |
| --- | --- |
| Unchanged baseline build/test results | [opennow-baseline-report.json](E:/DEV/Downloads/opennow-baseline-report.json) |
| Baseline, tool installation, and measured C:/E: storage ledger | [opennow-windows-build-verification.md](E:/DEV/Downloads/opennow-windows-build-verification.md) |
| Modular helper verification, exit codes, log cap, and 38 Doctor checks | [opennow-helper-verification-report.json](E:/DEV/Downloads/opennow-helper-verification-report.json) |
| Latest settings build, software/native checks, hashes, and limits | [verification.md](E:/DEV/Temp/OpenNOW/settings-remaining-20261012/verification.md) and [verification.json](E:/DEV/Temp/OpenNOW/settings-remaining-20261012/verification.json) |

The unchanged executable snapshot is `E:\DEV\Downloads\OpenNOW-unchanged-baseline.exe`,
SHA256 `921BCE0670A8384F5E1BA968C48E91CE2D730201EBEF6B901E5792B06735E907`.
Build.ps1 builds the current checkout; it does not reset source to this baseline.

The 12 October validated Release executable has SHA256
`3F5D51B18F57F6F1D7AC7ED9E991B5653C71D2FB36D144B809D815AD988013D2`.
Rebuilding can change it; preserve each new command result and hash.

- **Passed:** Release configure/build; six focused targets; 392 selected software targets;
  two native RTX 3050/D3D11 performance runs at display-reported 119.88 Hz
  (1080p windowed and 4K fullscreen); six native settings behavior workloads.
- **Failed:** zero final checks in that validation batch. The unchanged baseline had 359
  failures; discarded diagnostic setup failures are retained in the latest report.
- **Skipped:** zero selected CTest targets; six internal Qt cases for Windows empty-environment
  behavior, Wayland, macOS, and Unix local-socket contracts.
- **Unexecuted:** final full 240 Hz acceptance, separate early-entry performance, live login
  and gameplay, and physical HDR/gameplay-input correctness. Rust workspace suites and
  separate media GPU fixtures were not rerun in the latest batch.

These records close the environment setup phase. They do not establish complete hardware
or live-service acceptance.

## Storage and maintenance

The helpers direct build outputs, Cargo homes, process temporary files, and logs to E:.
The Windows SDK and some Visual Studio shared components remain on C: in supported locations.
The reused PowerShell executable is also on C:.

`OPENNOW_DATA_DIR` does not redirect Qt's default AppData cache. The normal application may
write `C:\Users\xpont\AppData\Local\OpenCloudGaming\OpenNOW\cache`; Windows may write crash
dumps under `C:\Users\xpont\AppData\Local\CrashDumps`. Record these effects for relevant batches.
Process-specific `QML_DISK_CACHE_PATH` can direct diagnostic QML caches to E:.

A 10 MiB command-log cap bounds each log, not total retained storage. Helpers preserve previous
logs and results. Inspect relevant directories before estimating space or proposing cleanup;
ask before deleting or relocating data. Keep installers, upgrades, persistent environment,
registry, service, driver, display, and security changes subject to the user's approval.
