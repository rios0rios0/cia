# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

CIA (Coletor de Informacoes Automatico) is a discontinued (2014) educational Windows system-information agent in Object Pascal (Delphi 7). The repo is archived with no active development.

## Build

Requires Borland Delphi 7. No CI, no tests, no linter.

```
dcc32.exe CIA.dpr -U"External Uses;System" -E. -N.
```

Clean build artifacts: `Clear.bat` (also exists in `External Uses/` and `System/`).

## Architecture

Zero-VCL, zero-RTL. The binary depends only on `Windows.pas` and custom units.

`System/` is the custom runtime:
- `System.pas` — minimal Delphi RTL replacement (memory, exceptions, variants)
- `SysInit.pas` — TLS and module init
- `CompressionStreamUnit.pas` — ZLIB 1.1.4 streams via linked `.obj` files

Entry point: `CIA.dpr` creates a hidden window with a 2-hour timer. Each tick runs `GetData` which collects system info into JSON and HTTP POSTs it to a server.

Data collection lives in `External Uses/`:
- `SysFuncs.pas` — OS, CPU, RAM, disk, printers, registry
- `EthFuncs.pas` — NICs, MAC, WLAN, domain, HTTP POST
- `MyUtils.pas` — string/file utilities, many in inline x86 ASM

## Conventions

- **No VCL, no standard RTL.** Never add `SysUtils`, `Classes`, `Forms`, or any standard Delphi unit.
- Use Win32 types (`PChar`, `DWORD`, `BOOL`, `HANDLE`) throughout, not managed Delphi types.
- Prefer explicit `GetLastError` error checking over `try/except`.
- Inline ASM uses 32-bit registers only (`asm ... end` blocks).
- `External Uses/` holds application-level units; `System/` is the custom runtime only.
- Comments in Portuguese or English are both acceptable.
