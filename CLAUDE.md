# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

CIA (Coletor de Informacoes Automatico) is a discontinued (2014) educational Windows system-information agent in Object Pascal (Delphi 7). The repo is archived with no active development.

## Build

Requires Borland Delphi 7. No build, test or lint automation; the only CI is the Claude review and `@claude` mention workflows.

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

<!-- chlog:start -->
## Changelog (chlog) — MANDATORY

If the repository you are working in uses chlog (a `.chlog.yaml` or `.chlog.yml`
config file, or a `.changes/` directory, exists at the project root), the
following is binding and ALWAYS applies: whenever you make ANY change, you MUST
create a changelog fragment as part of the same change — automatically, without
being asked, before committing.

- Do NOT edit CHANGELOG.md directly; it is generated from fragments.
- Create the fragment with:
  `chlog new --kind <Kind> --body "<imperative description>"`
- Valid kinds: Added, Changed, Deprecated, Removed, Fixed, Security
- Choose the kind that best matches the change (e.g., new feature → Added,
  bug fix → Fixed, behavior change → Changed, removal → Removed, security fix → Security).
- If the change is backward-INCOMPATIBLE with the public API (a breaking
  change), you MUST add the `--breaking` flag:
  `chlog new --kind <Kind> --breaking --body "<description>"`.
  This is the ONLY thing that triggers a major version bump — the kind alone
  never does (per SemVer, major = incompatible change). When unsure whether a
  change breaks compatibility, ask the user instead of guessing.
- Fragments are YAML files in `.changes/unreleased/`; stage them with your commit.
- `chlog check` fails the build when a fragment is missing — never skip it.
<!-- chlog:end -->
