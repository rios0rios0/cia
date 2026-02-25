<h1 align="center">CIA - Coletor de Informacoes Automatico</h1>
<p align="center">
    <a href="https://github.com/rios0rios0/cia/releases/latest">
        <img src="https://img.shields.io/github/release/rios0rios0/cia.svg?style=for-the-badge&logo=github" alt="Latest Release"/></a>
    <a href="https://github.com/rios0rios0/cia/blob/main/LICENSE">
        <img src="https://img.shields.io/github/license/rios0rios0/cia.svg?style=for-the-badge&logo=github" alt="License"/></a>
</p>

An automatic system information collector (Coletor de Informacoes Automatico) built entirely in Object Pascal using raw Win32 API calls -- no VCL dependency, no Delphi RTL, and a custom minimal runtime. It inventories hardware, software, and network configuration of a Windows machine, serializes the data to JSON, and submits it to a remote HTTP server via a hand-crafted TCP socket POST. This project was developed for educational purposes and is no longer actively maintained (discontinued 2014-11-27).

## Features

### System Information Collection
- **User and Machine Identity** -- retrieves the current Windows username (`GetUserNameA`) and computer name (`GetComputerNameA`)
- **Administrator Detection** -- checks if the current process runs with administrator privileges by inspecting the process token for the `DOMAIN_ALIAS_RID_ADMINS` SID
- **OS Version Identification** -- detects Windows versions from Windows 95 through Windows 8.1 (including Server editions), product editions (Home Basic, Professional, Ultimate, Starter), service pack level, and build number
- **64-bit Detection** -- uses `IsWow64Process` from `kernel32.dll` to determine if the OS is 64-bit
- **System Language** -- retrieves the system default language via `GetSystemDefaultLangID` and `VerLanguageName`

### Network Information
- **Network Interface Enumeration** -- uses raw Winsock2 (`WSAIoctl` with `SIO_GET_INTERFACE_LIST`) to enumerate all network interfaces, capturing IP address, subnet mask, broadcast addresses (limited and directed), loopback status, and interface up/down state
- **MAC Address Retrieval** -- reads hardware addresses via `GetAdaptersInfo` from `iphlpapi.dll`, matching each interface by MAC to determine DHCP or static configuration
- **WLAN Interface Detection** -- probes `Wlanapi.dll` (`WlanOpenHandle`, `WlanEnumInterfaces`) to distinguish wireless interfaces from wired Ethernet
- **Network Domain** -- retrieves the Windows domain/workgroup via `NetWkstaGetInfo` (NT) or registry (9x)
- **HTTP POST Submission** -- constructs raw HTTP/1.1 POST requests over TCP sockets (`WS2_32.dll`) with DNS resolution via `gethostbyname`, sending the collected JSON inventory to a configurable server endpoint

### Hardware Information
- **CPU Details** -- reads processor vendor, identifier, name, and clock speed from the Windows Registry (`HKLM\Hardware\Description\System\CentralProcessor\0`)
- **CPU Core Count** -- uses `GetSystemInfo` with CPUID instruction (inline x86 assembly) to detect hyper-threading and report physical core count
- **CPU Architecture** -- determines x32 or x64 based on pointer size at compile time
- **RAM** -- retrieves total physical memory via `GlobalMemoryStatusEx`
- **Disk Enumeration** -- iterates drives A-Z, uses `DeviceIoControl` with `IOCTL_VOLUME_GET_VOLUME_DISK_EXTENTS` to group partitions by physical disk number, and reports total/free space per partition via `GetDiskFreeSpaceExA`
- **USB Device Change Monitoring** -- handles `WM_DEVICECHANGE` messages to log device insertion and removal events with timestamps and drive types (fixed, CD-ROM, removable, shared)

### Software Inventory
- **Registry-Based Program List** -- enumerates installed software from `HKLM\Software\Microsoft\Windows\CurrentVersion\Uninstall` and the Wow6432Node equivalent, collecting display names and versions
- **Directory and Shortcut Scanning** -- scans `Program Files`, `Program Files (x86)`, and Start Menu folders for additional program entries, including `.lnk` shortcut resolution
- **Printer Enumeration** -- uses `EnumPrintersA` and `GetDefaultPrinterA` from `winspool.drv` to list all local and network printers with their shared/default status and location

### Infrastructure
- **Zero-VCL Architecture** -- the entire application uses only `Windows.pas` and custom units, with a hand-crafted window class (`CreateWindowExA`), message loop, and timer callback -- no Forms, no SysUtils, no Classes
- **Custom Runtime** -- includes a stripped-down `System.pas`, `SysInit.pas`, and `CompressionStreamUnit.pas` (ZLIB 1.1.4 with linked `.obj` files) providing `TStream`, `TMemoryStream`, `TFileStream`, and compression/decompression without any standard Delphi RTL
- **Inline Assembly** -- critical utility functions (`IntToStr`, `StrToInt`, `StrLen`, `StrScan`, `GetCPUFeatures`) are implemented in x86 assembly for minimal binary size
- **Auto-Update Mechanism** -- downloads a version file from a configurable URL via `WinInet` (`InternetOpenA`, `InternetOpenUrlA`, `InternetReadFile`), compares version numbers, and performs self-replacement using remote thread injection into `cmd.exe`
- **Self-Installation** -- copies itself to `C:\`, sets the file as hidden, and creates a `Run` registry key for persistence at startup
- **Remote Thread Injection** -- uses `CreateRemoteThread`, `VirtualAllocEx`, and `WriteProcessMemory` to inject code into `cmd.exe` for self-deletion and replacement during auto-update
- **Timer-Based Collection** -- uses `SetTimer` with a 2-hour interval (7200000ms) to periodically re-collect and submit system data
- **Mutex Singleton** -- prevents multiple instances via `CreateMutex`/`OpenMutex`
- **JSON Output** -- generates a structured JSON document organized into `software`, `ethernet`, `hardware`, `printers`, and `all_instaled` sections

## Technologies

| Component | Details |
|-----------|---------|
| **Language** | Object Pascal (Delphi 7) -- zero-VCL, pure Win32 API |
| **Runtime** | Custom minimal runtime (`System.pas`, `SysInit.pas` by Avenger/NhT) |
| **Compression** | ZLIB 1.1.4 (linked `.obj` files: `deflate`, `inflate`, `adler32`, `crc32`, etc.) |
| **Networking** | Winsock2 (`WS2_32.dll`), WinInet (`wininet.dll`), IP Helper (`iphlpapi.dll`), WLAN API (`Wlanapi.dll`) |
| **Assembly** | Inline x86 ASM for `IntToStr`, `StrToInt`, `StrLen`, `StrScan`, `GetCPUFeatures` (CPUID) |

## Project Structure

```
cia/
├── CIA.dpr                              # Main program - window creation, timer setup, data collection orchestration
├── External Uses/
│   ├── MyUtils.pas                      # Core utility functions (IntToStr, StrToInt, FileExists, etc.) with x86 ASM
│   ├── EthFuncs.pas                     # Network functions - Winsock interface enumeration, MAC addresses, WLAN detection, HTTP POST
│   └── SysFuncs.pas                     # System functions - OS version, CPU info, RAM, disk, printers, registry, device change
├── System/
│   ├── System.pas                       # Custom minimal Delphi RTL replacement
│   ├── SysInit.pas                      # Custom system initialization (TLS, module registration)
│   ├── SysSfIni.pas                     # Safe initialization with exception-protected unit init/finalization
│   ├── SYSWSTR.PAS                      # Wide string support
│   ├── CompressionStreamUnit.pas        # ZLIB compression/decompression streams (TStream, TMemoryStream, TFileStream)
│   ├── adler32.obj                      # ZLIB Adler-32 checksum (compiled C object)
│   ├── compress.obj                     # ZLIB compress function
│   ├── crc32.obj                        # ZLIB CRC-32 checksum
│   ├── deflate.obj                      # ZLIB deflate compression
│   ├── inflate.obj                      # ZLIB inflate decompression
│   ├── infback.obj                      # ZLIB inflate back-end
│   ├── inffast.obj                      # ZLIB fast inflate
│   ├── inftrees.obj                     # ZLIB inflate tree construction
│   ├── trees.obj                        # ZLIB Huffman tree construction
│   ├── uncompr.obj                      # ZLIB uncompress function
│   ├── installer.obj                    # System installer object
│   └── asm.obj                          # Additional assembly routines
├── LICENSE                              # GNU General Public License v3.0
└── README.md
```

## JSON Output Format

The collector generates a JSON document with the following structure:

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

## Installation

1. Open `CIA.dpr` in Borland Delphi 7 (or a compatible IDE)
2. Ensure the `External Uses` and `System` directories are in the project search path
3. Compile the project (Ctrl+F9)

## How It Works

1. On startup, the program creates a hidden window with a `WM_DEVICECHANGE` handler and registers a 2-hour timer
2. Every 2 hours (or on first trigger), `GetData` collects all system, network, and hardware information into a `Data.txt` JSON file
3. `GetPrinters` appends printer data and `GetPrograms` appends the software inventory
4. The completed JSON is submitted via an HTTP POST to the configured server endpoint (`/inventory/receive`)
5. USB device insertions and removals are logged to `Logs.txt` in real-time

## Contributing

Contributions are welcome. See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

## License

This project is licensed under the [GNU General Public License v3.0](LICENSE).
