# SNEK Blue War Hammer - Vulnerability Documentation & Reimplementation - SNEK Blue War Hammer
Professional proof-of-concept demonstrating Windows Defender exploitation techniques. This project demonstrates security research concepts and is provided for educational purposes only.

Based on public vulnerability research by [Nightmare-Eclipse](https://github.com/Nightmare-Eclipse/BlueHammer).

## Overview

The SNEK Blue War Hammer is a research tool that examines Windows Defender update mechanisms and their interaction with system file access controls. The implementation demonstrates advanced Windows API programming patterns including RPC communication, COM interfaces, VSS manipulation, and kernel level file system operations.

The tool is designed for security researchers and system administrators to understand potential attack vectors in update mechanisms.

## Dependencies and Runtime Requirements

This project uses a mixture of Win32 APIs, COM, Windows Update Agent interfaces, native NT APIs, and RPC stubs generated from `windefend.idl`, with additional libraries for system artifact analysis.

Key dependencies:

- `wininet.lib`
- `ktmw32.lib`
- `Shlwapi.lib`
- `Rpcrt4.lib`
- `ntdll.lib`
- `Cabinet.lib`
- `Wuguid.lib`
- `CldApi.lib`
- `userenv.lib`
- `Secur32.lib`
- `wbemuuid.lib`

The code also imports `windefend_h.h` and `offreg.h` for RPC and registry access.

## Structure

The source file is divided into the following major sections:

- Top level imports, macro definitions, and runtime loaded NT imports.
- RPC allocation helpers required for MIDL generated code.
- Defender RPC caller logic.
- Cabinet extraction and update file retrieval functions.
- Windows Update scanning and update detection logic.
- VSS and Defender trigger logic.
- System artifact parsing and analysis.
- The main application entry point.

## Native NT Imports

The code obtains NT native functions from `ntdll.dll` at runtime using `GetProcAddress`.
These functions are not part of the standard Win32 API and are used for low-level operations such as:

- `NtCreateSymbolicLinkObject`
- `NtOpenDirectoryObject`
- `NtQueryDirectoryObject`
- `NtSetInformationFile`

This runtime resolution allows the tool to interact with low level kernel objects and reparse points without static dependencies.

## RPC Allocation Helpers

The functions `midl_user_allocate` and `midl_user_free` are required by the MIDL generated RPC stubs.
They provide the client runtime with malloc/free callbacks for marshaling and unmarshaling RPC parameters.

## Defender RPC Call Flow

### `CallWD`

`CallWD` establishes an RPC binding to the local Defender service and invokes `ServerMpUpdateEngineSignature`.

- It builds an RPC string binding using the Defender interface UUID.
- It creates a binding handle from that string.
- It calls the Defender engine update RPC method.
- It signals a completion event when the call is finished.

The call is executed on a dedicated worker thread so the main code can continue monitoring file system events concurrently.

### `WDCallerThread`

`WDCallerThread` is a thin wrapper that turns the call into a thread compatible entry point.
It validates the input pointer and calls `CallWD`.

## Defender Update Extraction

### `GetUpdateFiles`

`GetUpdateFiles` downloads the latest Defender update package directly from Microsoft and extracts it from the CAB format.

The function does the following:

- Creates an `InternetOpen` session.
- Opens the Defender update URL.
- Queries the HTTP content length to size the download buffer.
- Reads the entire update package into memory.
- Locates the embedded CAB file inside the downloaded package.
- Uses FDI (Microsoft Cabinet API) callbacks to extract the CAB payload in memory.
- Returns a linked list of `UpdateFiles` structures containing file names, buffers, and sizes.

This function is central to the exploit because it provides the raw update files that are later used to trigger Defender behavior.

## Update Scan Logic

### `CheckForWDUpdates`

`CheckForWDUpdates` uses the Windows Update Agent COM interfaces to determine whether a new Defender update is available.

The function performs the following steps:

- Initializes COM.
- Creates an `IUpdateSession` object.
- Creates an `IUpdateSearcher` object.
- Performs a search for available updates using `Type='Software'` criteria.
- Inspects each returned update for Defender related metadata.
- Returns success when a suitable update is found.

If COM initialization or any update interface call fails, the function sets `*criterr` to true and reports failure.

## VSS Trigger Logic

### `TriggerWDForVS`

`TriggerWDForVS` attempts to force Defender to create a volume shadow copy path.

It does so by:

- Generating a unique temporary directory under `%TEMP%`.
- Writing a specially crafted file into that directory.
- Creating a worker thread to monitor shadow copy/VSS state.
- Waiting for Defender to access the file and create a new VSS-ready location.

The function returns `true` only when Defender has been successfully coerced into producing the expected shadow copy path.

## System Artifact Parsing

After extraction, the tool uses system-provided cryptographic primitives for artifact analysis:

- MD4-based hash computation for legacy format compatibility
- SHA256 for integrity verification
- DES decryption for legacy system data structures
- AES decryption using system crypto providers
- Offline registry hive parsing for artifact extraction

## Build Instructions

To build the project:

1. Open `SNEK_BlueWarHammer.sln` in Visual Studio 2022.
2. Ensure vcpkg is installed and configured for x64-windows triplet.
3. Build the Release|x64 configuration.
4. The executable will be generated in `x64\Release\SNEK_BlueWarHammer.exe`.

Alternative command line build using MSBuild:

```
MSBuild SNEK_BlueWarHammer.sln /p:Configuration=Release /p:Platform=x64
```

## Usage

Run the tool with logging to monitor progress:

```
SNEK_BlueWarHammer.exe --log-steps
```

Command line options:

- `--check-updates`: Check for Defender signature updates only
- `--download-only`: Download and extract updates without full exploitation
- `--trigger-vss-only`: Execute VSS trigger phase without full workflow
- `--disable-spawn`: Prevent shell execution after artifact extraction
- `--log-steps`: Display detailed operational progress

## Security Considerations

This tool demonstrates techniques relevant to Windows security research. Usage is restricted to:

- Authorized security research in controlled environments
- Educational purposes for understanding Windows internals
- System administration and vulnerability assessment

### Scientifical Use Only

This tool is intended solely for scientifical security research, educational purposes, and system administration within controlled environments. Unauthorized access to computer systems is illegal regardless of the tools or techniques employed.


### Provided by: ATroubledSnake & The SNEK Initiative. Long live freeware.
