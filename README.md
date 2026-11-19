# unrpa (Standalone Patched Edition)

A fast, zero-dependency, single-file Python CLI tool and library for inspecting and extracting files from **Ren'Py Visual Novel Engine archives** (`.rpa` / `.rpi`).

This standalone edition consolidates the original `unrpa` v2.3.0 multi-module package into a single portable script hardened with security fixes, multi-threaded extraction, multi-part stream support, and full **Python 3.14+** compatibility.

---

## Key Capabilities

* **Broad Archive Format Support:** Automatically detects and extracts `RPA-1.0` (`.rpi`), `RPA-2.0`, `RPA-3.0`, `ALT-1.0`, `RPA-3.2`, `RPA-4.0`, `ZiX-12A`, and `ZiX-12B`.
* **Zero Third-Party Dependencies:** Runs purely on the Python standard library. Eliminates `uncompyle6` by using a built-in, version-independent `.pyo` constant scanner for ZiX archives.
* **Python 3.14 & Free-Threading Ready:** Compatible with modern Python interpreters (including `--disable-gil` builds) without relying on version-locked `marshal.loads()` bytecode execution.
* **Parallel Multi-Threaded Extraction:** Uses `ThreadPoolExecutor` with thread-local file handles (`-j` / `--jobs`) to accelerate extraction of large archives containing thousands of assets.
* **Security Hardened:**
  * **Zip Slip Protection:** Resolves all destination paths via `pathlib.Path.resolve()` and blocks path-traversal escapes (`../`) outside the extraction directory.
  * **Restricted Unpickling:** Replaces raw `pickle.loads()` with a `RestrictedUnpickler` whitelist that blocks arbitrary code execution from malicious archive headers.
* **Multi-Part Index Streaming:** Seamlessly reconstructs split archive entries containing multiple `(offset, length, prefix)` chunks.
* **Tree & Flat Inspection:** Preview archive contents as a sorted flat file list (`-l`) or an ASCII directory tree (`-t`) before extracting.

---

## Requirements

* **Python 3.8 – 3.14+** (No external `pip` packages required).
* For `ZiX-12A` and `ZiX-12B` archives only: the game's `loader.pyo` file (found in the game's `renpy/` folder) must be placed in the same directory as the `.rpa` archive.

---

## Usage

### Basic Commands

```bash
# Extract an archive into the current directory
python3 unrpa archive.rpa

# Extract into a specific folder (creating missing directories automatically)
python3 unrpa -m -p ./extracted archive.rpa

# Extract multiple archives at once using 8 worker threads
python3 unrpa -j 8 -m -p ./extracted archive1.rpa archive2.rpa

# List all files in a flat list
python3 unrpa -l archive.rpa

# Display archive contents as a directory tree
python3 unrpa -t archive.rpa
```

### Advanced & Recovery Options

```bash
# Force a specific archive version if header auto-detection fails
python3 unrpa -f RPA-3.0 archive.rpa

# Manually specify decimal offset and key values (including 0)
python3 unrpa -o 123456 -k 987654321 archive.rpa

# Continue extracting remaining files even if a single entry is corrupted
python3 unrpa --continue-on-error archive.rpa
```

### CLI Reference

| Flag | Long Flag | Description |
| :--- | :--- | :--- |
| `-v` | `--verbose` | Increase output verbosity (pass `-vv` for debug directory logs). |
| `-s` | `--silent` | Suppress all non-essential output. |
| `-l` | `--list` | List archive contents in a flat sorted list. |
| `-t` | `--tree` | List archive contents in a visual tree view. |
| `-p PATH` | `--path PATH` | Destination directory for extracted files (default: current directory). |
| `-m` | `--mkdir` | Create missing directories in the `--path` destination. |
| `-j N` | `--jobs N` | Number of parallel worker threads for extraction (default: `min(32, cpu_count + 4)`). |
| `-f VER` | `--force VER` | Bypass auto-detection and force a specific format (`RPA-1.0`, `RPA-2.0`, `RPA-3.0`, `ALT-1.0`, `ZiX-12A`, `ZiX-12B`, `RPA-3.2`, `RPA-4.0`). |
| `-o INT` | `--offset INT` | Override the index offset manually (must be paired with `--key`). |
| `-k INT` | `--key INT` | Override the deobfuscation key manually (must be paired with `--offset`). |
| | `--continue-on-error` | Log errors on damaged entries and continue extracting the rest of the archive. |
| | `--version` | Print program version and exit. |

---

## Summary of Changes vs. Original `unrpa` v2.3.0

| Area | Original `unrpa` v2.3.0 | Patched `unrpa` | Impact |
| :--- | :--- | :--- | :--- |
| **Architecture** | Split across 9 package files (`meta.py`, `view.py`, `errors.py`, `versions/*.py`, etc.). | Consolidated into a single self-contained executable `.py` file. | Zero installation required; portable across systems. |
| **ZiX Decompilation** | Required `uncompyle6`, which fails on Python 3.9+ bytecode. | Built-in binary constant scanner (`scan_pyo_constants`) that parses raw `.pyo` marshal tags directly. | Works on Python 3.14+ with zero third-party dependencies. |
| **Extraction Speed** | Single-threaded sequential extraction loop. | Multi-threaded `ThreadPoolExecutor` (`-j` / `--jobs`) with `threading.local()` archive handles. | Substantially faster extraction for archives with thousands of small assets. |
| **Archive Security (Zip Slip)** | Joined archive paths directly with `os.path.join()`, allowing `../` path traversal. | Validates every resolved target path against the destination root via `Path.resolve()`. | Prevents malicious archives from overwriting files outside the output folder. |
| **Pickle Deserialization** | Used unrestricted `pickle.loads()` on untrusted archive indexes. | Uses `RestrictedUnpickler` allowing only safe primitive types (`dict`, `list`, `tuple`, `bytes`, `str`, `int`, `float`, `bool`). | Eliminates arbitrary code execution risks from crafted `.rpa` files. |
| **Multi-Part Streams** | `ArchiveView` read only `next(iter(data))` and dropped subsequent chunks. | `ArchiveView` queues and streams all `(offset, length, prefix)` parts sequentially. | Prevents silent truncation on split archive entries. |
| **CLI `--offset` / `--key`** | Checked `if args.key and args.offset:`, treating `0` as missing. | Checks `if args.key is not None and args.offset is not None:`. | Allows valid `0` offsets or keys without CLI errors. |
| **Terminal Output** | Exited on error with black text, hiding text on dark terminals. | Resets terminal color properly. | Keeps terminal prompts readable after an error. |
| **File I/O & Typing** | Used `os.path`, manual chunk loops, and duplicate imports. | Uses `pathlib.Path`, `shutil.copyfileobj()`, and `from __future__ import annotations`. | Cleaner code, faster C-level stream copying, and full type safety. |
