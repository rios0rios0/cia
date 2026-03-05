# GitHub Copilot Instructions

## Project Overview

**CIA** (Coletor de Informações Automático — Automatic Information Collector) is a discontinued (2014-11-27) educational Windows system-information agent written entirely in **Object Pascal (Delphi 7)**.

The application collects hardware, software, and network data from a Windows machine, serialises it as JSON, and submits it to a remote HTTP server via a hand-crafted TCP socket POST.  
It was designed with a **zero-VCL, zero-RTL** philosophy: the binary depends only on `Windows.pas` and a hand-written minimal runtime — no Forms, no SysUtils, no Classes.

> **Status:** archived / read-only reference. No active development is planned.

---

## Repository Structure

```
cia/
├── .github/
│   └── copilot-instructions.md   # This file
├── CIA.dpr                        # Entry point — window creation, WM_DEVICECHANGE handler, 2-hour timer, data collection orchestration
├── External Uses/
│   ├── MyUtils.pas                # Core utilities (IntToStr, StrToInt, FileExists, …) — many in x86 inline ASM
│   ├── EthFuncs.pas               # Network: Winsock2 interface enumeration, MAC addresses, WLAN detection, HTTP POST
│   └── SysFuncs.pas               # System: OS version, CPU, RAM, disk, printers, registry, device-change logging
├── System/
│   ├── System.pas                 # Custom minimal Delphi RTL replacement
│   ├── SysInit.pas                # System initialisation (TLS, module registration) — by Avenger/NhT
│   ├── SysSfIni.pas               # Safe init/finalisation with exception protection
│   ├── SYSWSTR.PAS                # Wide-string support
│   ├── CompressionStreamUnit.pas  # ZLIB 1.1.4 streams (TStream, TMemoryStream, TFileStream, compress/decompress)
│   ├── adler32.obj                # \
│   ├── compress.obj               #  |
│   ├── crc32.obj                  #  |  Pre-compiled ZLIB 1.1.4 C objects linked at build time
│   ├── deflate.obj                #  |
│   ├── inflate.obj / infback.obj  #  |
│   ├── inffast.obj / inftrees.obj #  |
│   ├── trees.obj / uncompr.obj    # /
│   ├── installer.obj              # System installer object
│   └── asm.obj                    # Additional assembly routines
├── Clear.bat                      # Deletes Delphi build artefacts (*.dcu, *.exe, *.map, …)
├── CONTRIBUTING.md                # Historical build notes
├── LICENSE                        # GNU GPL v3.0
└── README.md
```

---

## Technology Stack

| Layer | Details |
|---|---|
| Language | Object Pascal (Delphi 7) — zero-VCL, pure Win32 API |
| Runtime | Custom minimal runtime (`System.pas`, `SysInit.pas`) — no standard Delphi RTL |
| Compression | ZLIB 1.1.4 (linked pre-compiled `.obj` files) |
| Networking | Winsock2 (`WS2_32.dll`), WinInet (`wininet.dll`), IP Helper (`iphlpapi.dll`), WLAN API (`Wlanapi.dll`) |
| Assembly | Inline x86 ASM for `IntToStr`, `StrToInt`, `StrLen`, `StrScan`, `GetCPUFeatures` (CPUID) |
| Target OS | Windows 95 – Windows 8.1 (32-bit binary) |
| Build tool | Borland Delphi 7 IDE (`Ctrl+F9`) or `dcc32.exe` CLI |

---

## Build, Compile, and Clean Commands

> There is no automated CI pipeline for compilation — this is a Windows-only Delphi 7 project.

### Compile (historical)

1. Open `CIA.dpr` in Borland Delphi 7 (or a compatible IDE such as Embarcadero RAD Studio with legacy support).
2. Add `External Uses` and `System` to the project's **Library/Search path** (`Project → Options → Directories/Conditionals`).
3. Press `Ctrl+F9` (Build) or run:
   ```
   dcc32.exe CIA.dpr -U"External Uses;System" -E. -N.
   ```
4. Output: `CIA.exe` in the project root.

### Clean build artefacts

Run `Clear.bat` (Windows only) to remove all intermediate Delphi files:

```bat
Clear.bat
```

This deletes `*.dcu`, `*.exe`, `*.map`, `*.dsk`, `*.local`, and similar artefacts.

### Tests / Lint

This project has **no automated tests and no linter** — it is a discontinued educational project. There is nothing to run.

---

## Architecture and Design Patterns

### Zero-dependency philosophy
- The whole project deliberately avoids VCL and the standard Delphi RTL to produce the smallest possible binary.
- String manipulation, file I/O, memory management, and compression are all re-implemented from scratch or pulled in via linked `.obj` files.

### Application lifecycle (`CIA.dpr`)
1. **Singleton guard** — `CreateMutex`/`OpenMutex` prevents multiple instances.
2. **Self-installation** — copies the executable to `C:\`, marks it hidden, and adds a `Run` registry key for persistence.
3. **Hidden window** — `CreateWindowExA` with a message loop; no visible UI.
4. **WM_DEVICECHANGE** handler — logs USB insertion/removal events to `Logs.txt`.
5. **2-hour timer** (`SetTimer`, 7 200 000 ms) — triggers `GetData` periodically.

### Data collection pipeline
```
GetData
 ├── SysFuncs: OS, CPU, RAM, disk → Data.txt (JSON)
 ├── EthFuncs: NICs, MAC, WLAN, domain → appended to Data.txt
 ├── GetPrinters → appended to Data.txt
 ├── GetPrograms (registry + filesystem scan) → appended to Data.txt
 └── EthFuncs.HttpPost → sends Data.txt contents to /inventory/receive
```

### Auto-update
- Downloads a version file from a configurable URL via `WinInet`.
- On version mismatch, uses `CreateRemoteThread` + `VirtualAllocEx` + `WriteProcessMemory` to inject a self-replace routine into `cmd.exe`.

### Inline x86 assembly
Critical helpers in `MyUtils.pas` are hand-written in 32-bit x86 assembly (`asm … end` blocks) to keep the binary compact.

---

## Key Files and Their Roles

| File | Responsibility |
|---|---|
| `CIA.dpr` | Program entry point; orchestrates all collection and submission |
| `External Uses/MyUtils.pas` | Low-level utilities — string conversion, file helpers, registry access |
| `External Uses/EthFuncs.pas` | All network functionality including the HTTP POST sender |
| `External Uses/SysFuncs.pas` | Hardware/OS/software inventory |
| `System/CompressionStreamUnit.pas` | ZLIB-based compression streams used when sending data |
| `System/System.pas` | Replacement for Delphi's `System.pas` — memory, exception, variant primitives |

---

## JSON Output Format

```json
{
  "software": {
    "user": "JohnDoe",
    "machine": "WORKSTATION-01",
    "is_admin": 1,
    "os": "Windows 7 Professional",
    "dist": "Service Pack 1",
    "language": "Portuguese (Brazil)",
    "compilation": "6.1.7601",
    "is_64": 1
  },
  "ethernet": {
    "domain": "WORKGROUP",
    "boards": [
      {
        "mac": "AA-BB-CC-DD-EE-FF",
        "type": "eth0",
        "method": "dhcp",
        "sub_mask": "255.255.255.0",
        "net_address": "192.168.1.100",
        "limited_broadcast_address": "255.255.255.255",
        "directed_broadcast_address": "192.168.1.255",
        "interface_up": 1,
        "broadcast_supported": 1,
        "loopback_interface": 0
      }
    ]
  },
  "hardware": {
    "cpu_vendor": "GenuineIntel",
    "cpu_family": "x86 Family 6 Model 42",
    "cpu_identifier": "Intel(R) Core(TM) i5-2500K",
    "cpu_clock_speed": "3300 MHz",
    "cpu_all_cores": 4,
    "cpu_architecture": "x32",
    "ram": 8.00,
    "drivers": {
      "disk0": ["500.00G", {"disk01": "C:", "full_space": "250.00G", "free_space": "120.00G"}]
    }
  },
  "printers": [
    {"name": "HP LaserJet", "type": "local", "path": "", "shared": 0, "default": 1}
  ],
  "all_instaled": ["Microsoft Office 2013", "Google Chrome - 89.0.4389"]
}
```

---

## Development Workflow

Because this is an archived project, there is no active development workflow. If you need to make changes for reference or historical study:

1. **Read** `README.md` and `CONTRIBUTING.md` for full context.
2. **Compile** only with Borland Delphi 7 (newer Delphi versions may not be source-compatible with the custom runtime).
3. **Do not** add VCL, SysUtils, Classes, or any standard RTL unit — it will break the custom runtime.
4. **Do not** introduce 64-bit constructs — the project is 32-bit only.
5. **Clean** with `Clear.bat` before re-distributing sources.

---

## CI/CD Pipeline

There is **no CI/CD pipeline** for this project. The repository has no GitHub Actions workflows, no automated tests, and no container configuration.

---

## Coding Conventions

- **Language:** Object Pascal — Pascal-case for types and procedures, all-caps for Windows API constants.
- **No RTL types:** avoid `String`, `AnsiString` (as managed types), `Integer` from SysUtils, etc. Use `PChar`, `DWORD`, `BOOL`, `HANDLE` Win32 types throughout.
- **Inline ASM:** assembly blocks follow the `asm … end` syntax; use only 32-bit registers.
- **Units:** each major concern lives in its own `.pas` file under `External Uses/`; the `System/` directory is reserved for the custom runtime only.
- **No exceptions in production paths:** the custom runtime has limited exception support — prefer explicit error-code checking (`GetLastError`) over `try/except`.
- **Comments:** Portuguese and English comments both appear in the original source; either is acceptable.

---

## Common Tasks

| Task | How |
|---|---|
| Browse source | Open `CIA.dpr` in any text editor or Delphi 7 IDE |
| Understand data flow | Start with `CIA.dpr` → `GetData` procedure |
| Understand networking | Read `External Uses/EthFuncs.pas` |
| Understand OS/hardware detection | Read `External Uses/SysFuncs.pas` |
| Understand string/utility helpers | Read `External Uses/MyUtils.pas` |
| Clean build outputs | Run `Clear.bat` |
| Rebuild | Open in Delphi 7 and press `Ctrl+F9` |
