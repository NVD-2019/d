# PrintSpoofer
## Reflective DLL Exploitation of Windows Print Spooler Token Impersonation

[![Platform](https://img.shields.io/badge/platform-Windows%2010%20%7C%2011-lightgrey.svg)](https://www.microsoft.com/windows)
[![Architecture](https://img.shields.io/badge/arch-x86%20%7C%20x64-blue.svg)](https://github.com)
[![C++](https://img.shields.io/badge/C%2B%2B-17-critical.svg)](https://isocpp.org/)

---

## Overview

PrintSpoofer is a Windows privilege escalation research tool that exploits token impersonation through the Print Spooler service. By abusing the `SeImpersonatePrivilege` and named pipe communication mechanisms, the exploit achieves local privilege escalation from a low-privileged context to SYSTEM. This implementation is delivered as a **Reflective DLL**, enabling deployment in restricted environments and integration with post-exploitation frameworks like Cobalt Strike.

**Research Context:** This project demonstrates the security implications of overprivileged service design, named pipe impersonation patterns, and RPC communication vulnerabilities in the Windows Print Spooler architecture.

---

## Table of Contents

- [Quick Start](#quick-start)
- [Technical Overview](#technical-overview)
- [Execution Flow](#execution-flow)
- [Windows Internals](#windows-internals)
- [Technical Architecture](#technical-architecture)
- [Build](#build)
- [Project Structure](#project-structure)
- [Usage](#usage)
- [Detection & Monitoring](#detection--monitoring)
- [Limitations & Compatibility](#limitations--compatibility)
- [Security Disclaimer](#security-disclaimer)
- [References](#references)
- [License](#license)

---

## Quick Start

```bash
# 1. Clone the repository
git clone https://github.com/yourusername/PrintSpoofer.git
cd PrintSpoofer

# 2. Open in Visual Studio 2022
# File > Open > PrintSpoofer.sln

# 3. Select platform (x64 or Win32) and Configuration (Release)

# 4. Build (Ctrl+Shift+B)

# 5. Reflective DLL output:
# PrintSpoofer\x64\Release\PrintSpoofer.dll
```

---

## Technical Overview

### What This Tool Does

PrintSpoofer performs local privilege escalation by exploiting the interaction between three Windows mechanisms:

1. **Named Pipe Communication** — The exploit creates a crafted named pipe that mimics the Print Spooler's expected pipe structure
2. **Service Impersonation** — The Print Spooler service connects to the malicious pipe while running in a SYSTEM context
3. **Token Impersonation** — The `SeImpersonatePrivilege` allows the exploit code to assume the identity (and token) of the connected service

### Key Assumptions

- The target process holds `SeImpersonatePrivilege` (enabled or present in the token)
- The Print Spooler service (`spoolsv.exe`) is running
- Windows RPC is available for Print Spooler communication
- The system is Windows 10 or later

### Why This Works

The Windows Print Spooler service is designed to be highly available. It listens for printer change notifications over named pipes through the MS-RPRN (Print System Remote Protocol) interface. When an RPC client requests a printer change notification, the service attempts to connect to a pipe specified by the client. This design, combined with the service's SYSTEM privilege level, creates an impersonation opportunity: if a lower-privileged user can intercept or redirect this connection to a pipe they control, they can impersonate the service's token.

---

## Execution Flow

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. Privilege Verification                                       │
│    └─> Check if current process holds SeImpersonatePrivilege   │
└──────────────────────────┬──────────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────────┐
│ 2. Named Pipe Creation                                          │
│    └─> Create pipe: \\.\pipe\<UUID>\pipe\spoolss               │
│    └─> Configure DACL for asynchronous I/O                      │
└──────────────────────────┬──────────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────────┐
│ 3. Async Pipe Listener Setup                                    │
│    └─> Register event object for connection notification        │
│    └─> Enable asynchronous wait for incoming connections        │
└──────────────────────────┬──────────────────────────────────────┘
                           │
          ┌────────────────┴────────────────┐
          │                                 │
   ┌──────▼──────────┐           ┌─────────▼────────┐
   │ Main Thread     │           │ Worker Thread    │
   │ (Listening)     │           │ (RPC Call)       │
   │                 │           │                  │
   │ Wait for pipe   │           │ Call             │
   │ connection      │           │ RpcRemoteFinder  │
   │ event...        │           │ FirstPrinter...  │
   │                 │           │                  │
   └──────┬──────────┘           └────────┬─────────┘
          │                               │
          │           ┌───────────────────┘
          │           │ (spoolsv.exe connects)
          │           │
   ┌──────▼───────────▼──────────────────────────────────────────┐
   │ 4. Impersonation                                            │
   │    └─> ImpersonateNamedPipeClient() on service connection  │
   │    └─> Assume SYSTEM token                                  │
   └──────┬───────────────────────────────────────────────────────┘
          │
   ┌──────▼───────────────────────────────────────────────────────┐
   │ 5. Privileged Operation                                       │
   │    └─> Execute with SYSTEM privileges (spawn shell, etc.)   │
   │    └─> Return control to caller                              │
   └───────────────────────────────────────────────────────────────┘
```

---

## Windows Internals

### SeImpersonatePrivilege

**Definition:** A process-level privilege that allows a thread to assume the access token of another principal (user, service, or client).

**Availability:** By default, users in the local Administrators group and most service accounts hold this privilege. Standard domain users typically do not.

**Mechanism:** When a server process accepts an authenticated client connection (named pipe, RPC, etc.), it can call `ImpersonateNamedPipeClient()` or related APIs to temporarily switch its effective token to that of the connected client.

**Exploitation:** If the server is running with higher privileges than the client, the client can escalate by impersonating the server's higher-privileged token.

### Named Pipes & Impersonation

**Named Pipes Overview:** Named pipes provide full-duplex, message-oriented inter-process communication (IPC) on Windows. They support authentication through kernel-level security descriptors (DACL).

**Connection Flow:**
1. Server creates a named pipe with `CreateNamedPipe()`
2. Server waits for connections with `ConnectNamedPipe()` or async I/O
3. Client connects with `CreateFile()` pointing to the pipe name
4. Both sides authenticate through the kernel

**Impersonation:** When a client connects to a server's named pipe, the server can call `ImpersonateNamedPipeClient()` to temporarily adopt the connected client's security context. This is the core primitive exploited by PrintSpoofer.

### Print Spooler Service & MS-RPRN

**Print Spooler (`spoolsv.exe`):** A Windows service (runs as SYSTEM) responsible for managing print jobs and printer notifications. It communicates with client applications and printer devices through various mechanisms including RPC over named pipes.

**MS-RPRN (Print System Remote Protocol):** An RPC interface defined by Microsoft that allows remote clients to:
- Query printer status
- Subscribe to printer change notifications
- Interact with the print queue

The key RPC function exploited by PrintSpoofer is `RpcRemoteFindFirstPrinterChangeNotificationEx()`, which accepts a pipe name where the service should send notifications. By supplying a custom pipe name, an attacker can redirect the service's connection attempt.

**Service Behavior:** When the Print Spooler processes a printer notification subscription, it opens a connection to the pipe specified by the client. This connection is made in the SYSTEM security context.

### Reflective DLL Injection

**Standard DLL Loading:**
- Placed on disk
- Loaded via `LoadLibrary()` API call
- Kernel executes the PE loader
- Triggers `DllMain()` entry point

**Reflective DLL Loading:**
- Embedded entirely in memory
- Manually parses PE headers and sections
- Resolves IAT (Import Address Table) by walking PEB (Process Environment Block)
- Relocates base addresses without kernel involvement
- Transfers control to `DllMain()`

**Advantages:**
- No disk I/O — evades file-based detection
- No `LoadLibrary()` call — bypasses certain API hooks
- Suitable for in-memory execution frameworks (Cobalt Strike, etc.)

**Implementation:** The `ReflectiveLoader.cpp` module implements this mechanism, parsing the DLL's PE structure and manually performing all steps normally handled by the Windows loader.

---

## Technical Architecture

### Component Interaction Diagram

```
┌──────────────────────────────────────────────────────────────┐
│                     Attacker Process                         │
│                   (Low Privileges)                           │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  PrintSpoofer.dll (Reflective DLL)                     │ │
│  │                                                        │ │
│  │  ┌──────────────────────────────────────────────────┐ │ │
│  │  │ dllmain.cpp                                      │ │ │
│  │  │ • DLL entry point                                │ │ │
│  │  │ • Calls GetSystem()                              │ │ │
│  │  └──────────────────────────────────────────────────┘ │ │
│  │                         ▲                               │ │
│  │                         │                               │ │
│  │  ┌──────────────────────┴──────────────────────────┐ │ │
│  │  │ PrintSpoofer.cpp                               │ │ │
│  │  │ • CheckAndEnablePrivilege()                     │ │ │
│  │  │ • GenerateRandomPipeName()                      │ │ │
│  │  │ • CreateSpoolNamedPipe()                        │ │ │
│  │  │ • ConnectSpoolNamedPipe()                       │ │ │
│  │  │ • TriggerNamedPipeConnection() [async thread]   │ │ │
│  │  │ • GetSystem() [main logic]                      │ │ │
│  │  └──────────────────────────────────────────────────┘ │ │
│  │                         ▲                               │ │
│  │                         │                               │ │
│  │  ┌──────────────────────┴──────────────────────────┐ │ │
│  │  │ ReflectiveLoader.cpp                           │ │ │
│  │  │ • PE header parsing                            │ │ │
│  │  │ • Memory relocation & IAT resolution           │ │ │
│  │  │ • Reflective DLL entry point transition        │ │ │
│  │  └──────────────────────────────────────────────────┘ │ │
│  │                                                        │ │
│  │  ┌──────────────────────────────────────────────────┐ │ │
│  │  │ ms-rprn.idl (compiled to stubs)                 │ │ │
│  │  │ • RPC interface definitions                      │ │ │
│  │  │ • RpcRemoteFindFirstPrinterChangeNotificationEx│ │ │
│  │  │ • RPC marshaling/unmarshaling                   │ │ │
│  │  └──────────────────────────────────────────────────┘ │ │
│  │                                                        │ │
│  └─────────┬──────────────────────────────────┬──────────┘ │
│            │                                  │            │
│      ┌─────▼─────────────────┐    ┌──────────▼─────────┐   │
│      │ Named Pipe            │    │ RPC (ALPC)         │   │
│      │ \\.\pipe\<UUID>\pipe  │    │ to spoolsv.exe     │   │
│      │ \spoolss              │    │ (port: RPRN)       │   │
│      └─────┬─────────────────┘    └──────────┬─────────┘   │
│            │                                  │            │
└────────────┼──────────────────────────────────┼────────────┘
             │                                  │
   ┌─────────▼────────────────────────────────▼──────────┐
   │         Windows Kernel (Security Manager)          │
   │                                                    │
   │  • Validates named pipe DACL                       │
   │  • Manages access token lifecycle                  │
   │  • Enforces SeImpersonatePrivilege checks          │
   │  • Handles RPC message queuing                     │
   └──────────────────────┬─────────────────────────────┘
                          │
   ┌──────────────────────▼──────────────────────────────┐
   │        Print Spooler Service (spoolsv.exe)          │
   │              Running as: SYSTEM (NT AUTHORITY)       │
   │                                                    │
   │  • Listens for RPC on MS-RPRN interface            │
   │  • Connects to client-specified named pipe         │
   │  • Authenticates through kernel (SYSTEM context)   │
   │  • [VULNERABLE] No validation of pipe origin       │
   └──────────────────────────────────────────────────────┘
```

### Windows API & Components Summary

| Component | Purpose | Key API/Mechanism |
|-----------|---------|-------------------|
| **Privilege Check** | Verify SeImpersonatePrivilege availability | `AdjustTokenPrivileges()`, token query |
| **Named Pipe Creation** | Set up listening pipe | `CreateNamedPipe()`, DACL configuration |
| **Async I/O** | Wait for service connection without blocking | `CreateEvent()`, `ConnectNamedPipe()` (overlapped mode) |
| **RPC Interface** | Communicate with Print Spooler | MS-RPRN IDL → compiled RPC stubs, ALPC |
| **Token Impersonation** | Assume SYSTEM token | `ImpersonateNamedPipeClient()` |
| **Reflective Loading** | Load DLL from memory | PE parsing, manual IAT resolution, relocation |

---

## Build

### Requirements

| Component | Version | Notes |
|-----------|---------|-------|
| **Visual Studio** | 2022+ | C++ Desktop Development workload required |
| **Windows SDK** | 10.0.20348+ | Included with Visual Studio |
| **Platform Toolset** | v143+ | C++ language standard C++17 |
| **Target OS** | Windows 10 / 11 | For both build machine and target |

### Build Steps

1. **Clone the Repository**
   ```bash
   git clone https://github.com/yourusername/PrintSpoofer.git
   cd PrintSpoofer
   ```

2. **Open in Visual Studio**
   - Launch Visual Studio 2022
   - File → Open → Project/Solution
   - Select `PrintSpoofer.sln`

3. **Select Build Configuration**
   - **Platform:** `x64` (recommended) or `Win32` (for 32-bit targets)
   - **Configuration:** `Release` (for deployment) or `Debug` (for development)

4. **Build**
   - Build → Build Solution (Ctrl+Shift+B)
   - Or right-click solution → Build

5. **Locate Output**
   ```
   bin\x64\Release\PrintSpoofer.dll     (x64 Release)
   bin\x86\Release\PrintSpoofer.dll     (x86 Release)
   bin\x64\Debug\PrintSpoofer.dll       (x64 Debug)
   bin\x86\Debug\PrintSpoofer.dll       (x86 Debug)
   ```

### Build Configuration

The project uses these preprocessor definitions:

| Definition | Purpose |
|-----------|---------|
| `REFLECTIVE_DLL_EXPORTS` | Enables Reflective DLL export table generation |
| `REFLECTIVEDLLINJECTION_CUSTOM_DLLMAIN` | Uses custom DllMain implementation |
| `REFLECTIVEDLLINJECTION_VIA_LOADREMOTELIBRARYR` | Compatibility with LoadRemoteLibraryR injection |

These are configured in the `.vcxproj` file and do not require manual adjustment for standard builds.

---

## Project Structure

```
PrintSpoofer/
│
├── README.md                          # This file
├── LICENSE                            # MIT License
├── PrintSpoofer.sln                   # Visual Studio solution
│
├── PrintSpoofer/                      # Main project
│   ├── PrintSpoofer.vcxproj           # Project configuration
│   ├── PrintSpoofer.vcxproj.filters   # Visual Studio filters
│   │
│   ├── src/
│   │   ├── dllmain.cpp                # DLL entry point & main logic
│   │   ├── PrintSpoofer.cpp           # Core exploitation functions
│   │   ├── PrintSpoofer.h             # Function declarations
│   │   ├── ReflectiveLoader.cpp       # PE parsing & memory loading
│   │   ├── ReflectiveLoader.h         # Reflective loader types
│   │   ├── ms-rprn.idl                # Print Spooler RPC interface
│   │   └── ms-rprn_h.h                # (Generated by MIDL compiler)
│   │
│   └── bin/                           # Build output
│       ├── x64/Release/PrintSpoofer.dll
│       ├── x86/Release/PrintSpoofer.dll
│       └── ...
│
└── docs/
    └── TECHNICAL.md                   # (Optional) Detailed technical analysis
```

### File Descriptions

#### `dllmain.cpp`
- **Purpose:** DLL entry point and main orchestration
- **Key Functions:**
  - `DllMain()` — Entry point; calls `GetSystem()` on attach
  - Error handling and resource cleanup
  - Reflective DLL-specific initialization

#### `PrintSpoofer.cpp`
- **Purpose:** Core exploitation logic
- **Key Functions:**
  - `CheckAndEnablePrivilege()` — Verifies `SeImpersonatePrivilege` is present and enabled
  - `GenerateRandomPipeName()` — Generates unique pipe names using UUID (prevents conflicts)
  - `CreateSpoolNamedPipe()` — Creates named pipe with appropriate DACL for service connection
  - `ConnectSpoolNamedPipe()` — Sets up asynchronous I/O wait for incoming connections
  - `TriggerNamedPipeConnection()` — Spawns worker thread for RPC call
  - `TriggerNamedPipeConnectionThread()` — Executes `RpcRemoteFindFirstPrinterChangeNotificationEx()` in worker thread
  - `GetSystem()` — Main exploitation function; orchestrates the attack

#### `ReflectiveLoader.cpp`
- **Purpose:** Reflective DLL injection mechanism
- **Key Functions:**
  - PE header parsing (DOS header, PE signature, section headers)
  - Memory layout calculation and reservation
  - Manual section mapping and permission configuration
  - IAT (Import Address Table) resolution via PEB walking
  - Base address relocation
  - Calls `DllMain()` with `DLL_PROCESS_ATTACH` once loaded

#### `ms-rprn.idl`
- **Purpose:** RPC interface definition for MS-RPRN protocol
- **Content:**
  - Interface definitions for Print Spooler RPC calls
  - Type definitions and marshaling hints
  - Compiled by MIDL to generate C stubs (`ms-rprn_h.h`, `ms-rprn_s.c`, etc.)
  - Enables RPC communication with `spoolsv.exe`

---

## Usage

### Context: Authorized Security Research Only

**This tool is designed for:**
- Authorized penetration testing in controlled lab environments
- Security research on Windows privilege escalation
- Defensive capability assessment with explicit permission

**Prerequisites:**
- Explicit authorization from the system owner
- Lab environment (not production systems)
- Print Spooler service running (`spoolsv.exe`)
- Current process with `SeImpersonatePrivilege` (not necessarily enabled, but present in token)

### Deployment with Cobalt Strike

PrintSpoofer is designed for seamless integration with Cobalt Strike's Reflective DLL injection framework.

**Basic Usage:**
```
beacon> elevate PrintSpoofer [LISTENER_NAME]
```

**Parameters:**
- `PrintSpoofer` — Name of the elevator module (maps to the DLL export)
- `LISTENER_NAME` — Cobalt Strike listener to receive the elevated reverse shell

**Example:**
```
beacon> elevate PrintSpoofer https
[+] Tasked beacon to run PrintSpoofer (https listener)
[*] Received output:
    [+] Found privilege: SeImpersonatePrivilege
    [+] Named pipe listening...
    [+] ImpersonateNamedPipeClient OK
    [+] Exploit successfully, enjoy your shell
[+] Elevated beacon connected back
```

### Manual Execution via Reflective DLL Injection

If using custom injection tooling:

1. **Inject the DLL** into a process with `SeImpersonatePrivilege`
   ```
   Example: Inject into explorer.exe (typical user process with the privilege)
   ```

2. **Call the Reflective Loader**
   - Supply the DLL's in-memory base address
   - The loader parses PE headers and executes `DllMain()`

3. **Observe Output**
   - Logs printed to parent console or captured by the injector
   - Successful execution returns a SYSTEM-context shell or callback

### Expected Output (Successful)

```
[+] Found privilege: SeImpersonatePrivilege
[+] Named pipe listening...
[*] RPC call to Print Spooler in progress...
[+] ImpersonateNamedPipeClient OK
[+] Exploit successful — now running as NT AUTHORITY\SYSTEM
[+] Enjoy your shell!
```

### Expected Output (Failure Cases)

| Scenario | Output |
|----------|--------|
| SeImpersonatePrivilege not available | `[-] SeImpersonatePrivilege not found` |
| Print Spooler not running | `[-] RPC connection failed (RPC server unavailable)` |
| Named pipe creation failed | `[-] Failed to create named pipe (access denied)` |
| Service did not connect | `[-] Timeout: no connection to named pipe` |
| Impersonation failed | `[-] ImpersonateNamedPipeClient failed` |

---

## Detection & Monitoring

### What Defenders Should Monitor

#### Event Log Indicators

| Event | Source | Log | Indicator |
|-------|--------|-----|-----------|
| Named pipe creation | Windows Kernel | System | Suspicious pipe names (`\pipe\spoolss`, UUID patterns) |
| Service connection | Print Spooler | System | Unexpected outbound named pipe connections from `spoolsv.exe` |
| Token impersonation | Security | Security | `Impersonate` privilege usage in audit logs (if enabled) |
| Process injection | Endpoint tooling | Application | Unsigned DLL loads in memory from suspicious sources |
| Privilege elevation | Windows Logs | Security | Process creation with SYSTEM privileges from low-privilege parent |

#### Network & RPC

- **Monitor:** RPC traffic to/from Print Spooler (endpoint mapper, RPRN interface)
- **Watch for:** Unusual RPC calls to `RpcRemoteFindFirstPrinterChangeNotificationEx()` with non-standard pipe names
- **Baseline:** Print Spooler RPC traffic in normal operations for comparison

#### EDR/HIPS Indicators

- Reflective DLL injection patterns (memory-only PE loading)
- In-memory IAT walking (PEB enumeration)
- Calls to `ImpersonateNamedPipeClient()` from unexpected processes
- RPC stubs execution in non-system processes
- Privilege token changes (SeImpersonate usage)

#### File-Based Indicators

- Unsigned or test-signed executables spawned from reflective DLL injection
- Suspicious parent-child relationships (explorer.exe → cmd.exe with SYSTEM privileges)
- PE files loaded entirely from memory (no corresponding file on disk)

### Mitigation Strategies

| Strategy | Effectiveness | Implementation |
|----------|----------------|-----------------|
| **Disable Print Spooler** | Very High | `net stop spooler` (if printing not required) |
| **Restrict SeImpersonate** | High | GPO: Remove privilege from non-admin accounts |
| **Enable audit logging** | Medium | Windows Security Auditing for privilege usage |
| **RPC firewall rules** | Medium | Restrict RPC endpoint access via Windows Firewall |
| **EDR/behavioral monitoring** | High | Monitor reflective loading, token impersonation, unexpected SYSTEM processes |
| **Harden Print Spooler** | Medium | Windows Update patches for historical vulnerabilities |

---

## Limitations & Compatibility

### Privilege Requirements

- **SeImpersonatePrivilege** — Must be present in the current process token
  - Enabled or disabled status does NOT matter (privilege is automatically checked/enabled)
  - Available by default to Administrators, service accounts, and certain role players
  - **Not available** to most standard domain user accounts
  - Can be verified via `whoami /priv` command

### Service Requirements

- **Print Spooler** (`spoolsv.exe`) must be running
  - Typically started automatically on Windows systems
  - Can be confirmed via `Get-Service spooler` (PowerShell) or `sc query spooler` (CMD)
  - If disabled, the RPC call will fail (expected behavior)

### Windows Version Compatibility

| Version | Status | Notes |
|---------|--------|-------|
| Windows 10 (1909+) | Supported | Tested |
| Windows 11 | Supported | Tested |
| Windows Server 2019 | Supported | Tested on domain controllers and member servers |
| Windows Server 2022 | Supported | Tested |
| Earlier versions | Untested | May require modifications to RPC interface definitions |

### Architecture Support

| Platform | Support | Notes |
|----------|---------|-------|
| x64 (64-bit) | Supported | Recommended; primary development target |
| x86 (32-bit) | Supported | Builds successfully; less commonly deployed |
| ARM64 | Not Supported | No planned support |

### Known Limitations

1. **Print Spooler Dependency** — Exploitation requires an active Print Spooler service. Systems with the service disabled will fail gracefully.

2. **Named Pipe Quota** — If the system's named pipe quota is exhausted, pipe creation may fail. This is rare but possible on heavily loaded systems.

3. **Network Requirements** — If RPC to the Print Spooler is blocked (e.g., firewall, process isolation), the exploit cannot reach the service.

4. **Audit Logging** — In high-security environments with advanced logging and EDR, exploitation may be detected. This tool is not stealthy against all detection mechanisms.

5. **Timing** — Success depends on the Print Spooler being ready to process RPC calls at the moment of exploitation. Timing attacks are possible but unlikely in normal scenarios.

---

## Security Disclaimer

**IMPORTANT:** This tool is provided for educational and authorized security research purposes only.

### Authorized Use Only

- Use this tool **only on systems you own or have explicit written permission to test**
- Unauthorized access to computer systems is illegal in most jurisdictions
- Perform testing **only in isolated lab environments** unless authorized otherwise

### No Warranty

This software is provided AS-IS without warranty of any kind. The authors assume no liability for:
- Unintended behavior or system damage
- Data loss or corruption
- Service disruptions
- Legal consequences of unauthorized use

### Legal Compliance

- Ensure all use complies with applicable laws and regulations
- Respect privacy and confidentiality requirements
- Maintain proper documentation of authorized testing
- Follow responsible disclosure practices if vulnerabilities are discovered

### Ethical Guidelines

- This research is intended to improve Windows security understanding
- Findings should be disclosed responsibly to Microsoft and affected parties
- Do not use this tool for malicious purposes or unauthorized system access

---

## Technical References

### Windows Internals & Security

- **Microsoft Docs — Token Impersonation:** https://docs.microsoft.com/en-us/windows/win32/secauthz/impersonation
- **Microsoft Docs — Named Pipes:** https://docs.microsoft.com/en-us/windows/win32/ipc/named-pipes
- **Microsoft Docs — SeImpersonatePrivilege:** https://docs.microsoft.com/en-us/windows/security/threat-protection/security-policy-settings/impersonate-a-client-after-authentication

### Print Spooler & RPC

- **MS-RPRN — Print System Remote Protocol Specification:** https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-rprn/d42db852-12ce-4be2-931b-ca72e504af1f
- **MS-PAR — Print System Asynchronous Remote Protocol:** https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-par/

### Reflective DLL Injection

- **Stephen Breen — Reflective DLL Injection:** https://github.com/stephenbreen/ReflectiveDLLInjection
- **Rapid7 — Reflective DLL Injection Primer:** https://www.rapid7.com/blog/post/2015/12/07/reflective-dll-injection/

### Privilege Escalation Research

- **itm4n — PrintSpoofer: Abusing Impersonate Privileges:** https://www.itm4n.fr/printspoofer-abusing-impersonate-privileges/
- **HackTricks — Windows Local Privilege Escalation:** https://book.hacktricks.xyz/windows-hardening/windows-local-privilege-escalation
- **Elastic Security — Privilege Escalation Techniques:** https://www.elastic.co/security-labs/privilege-escalation-on-windows

### Related CVEs & Mitigations

While this project does not target a specific unpatched CVE, it demonstrates principles related to:
- **CVE-2019-9510** — Windows Print Spooler information disclosure
- **CVE-2020-0787** — Windows Update delivery optimization vulnerability (related service abuse)
- General Windows service impersonation risks and token abuse patterns

---

## License

This project is licensed under the **MIT License** — see the `LICENSE` file for details.

### MIT License Summary

- ✅ Use freely in personal and commercial projects
- ✅ Modify and distribute
- ✅ Private use
- ❌ Requires attribution (see LICENSE)
- ❌ No liability or warranty

---

## Credits & Acknowledgements

### Original Research

- **PrintSpoofer concept and technique:** itm4n (@itm4n_) — Original blog post and early research

### Reflective DLL Injection

- **Stephen Breen** — Reflective DLL Injection framework and implementation

### Contributions

This project builds on the collective knowledge of the Windows security research community. Special thanks to:
- Microsoft Security Response Center (MSRC) for transparency on Print Spooler security
- The open-source security tools community for best practices and patterns
- All contributors and security researchers who have improved Windows privilege escalation analysis

### Citation

If you reference this tool in research or security assessments, please cite:

```
@misc{printspoofer2024,
  title={PrintSpoofer: Reflective DLL Exploitation of Windows Print Spooler Token Impersonation},
  author={Your Name},
  year={2024},
  url={https://github.com/yourusername/PrintSpoofer}
}
```

---

## Additional Resources

### Further Reading

- **James Forshaw — Using Win32 Named Pipes for Privilege Escalation:** Detailed technical breakdowns
- **Alex Ionescu — Windows Internals (7th Edition):** Authoritative reference on Windows security architecture
- **David Proctor — Metasploit Framework Modules:** Reference implementations of similar exploits

### Related Tools

- **Juicy Potato** — Alternative token impersonation exploitation
- **PotatoNet** — Network-based privilege escalation
- **JuicyPotato** — Continuation of Juicy Potato research

---

**Last Updated:** August 2024  
**Status:** Active development and research  
**Contact:** [Your contact information or repository issues]
