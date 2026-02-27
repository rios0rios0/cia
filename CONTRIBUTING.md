# Contributing

> **This project was discontinued in November 2014 and is no longer actively maintained.**
> The repository is preserved as a historical reference. No new features or bug fixes are planned.

## Historical Build Information

This project was built using the following tools and technologies:

- **Language:** Object Pascal (Delphi 7) — zero-VCL, pure Win32 API
- **IDE:** Borland Delphi 7
- **Custom Runtime:** Minimal `System.pas` / `SysInit.pas` (by Avenger/NhT)
- **Compression:** ZLIB 1.1.4 (linked `.obj` files: deflate, inflate, adler32, crc32)
- **Networking:** Winsock2 (`WS2_32.dll`), WinInet (`wininet.dll`), IP Helper (`iphlpapi.dll`), WLAN API (`Wlanapi.dll`)
- **Inline x86 assembly** for core utility functions (`IntToStr`, `StrToInt`, `StrLen`, `StrScan`, `GetCPUFeatures` via CPUID)

### Build Steps (Historical)

1. Open `CIA.dpr` in Borland Delphi 7 (or compatible IDE)
2. Ensure the `External Uses` and `System` directories are in the project search path
3. Compile the project (`Ctrl+F9`)

> **Note:** Key files include `CIA.dpr` (main program), `External Uses/MyUtils.pas`, `External Uses/EthFuncs.pas`, `External Uses/SysFuncs.pas`, and the `System/` directory with custom RTL and ZLIB `.obj` files.
