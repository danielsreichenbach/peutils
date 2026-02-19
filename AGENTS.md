# PE Utils - Binary Ninja Plugin

This is a Binary Ninja plugin that provides utilities for working with PE (Portable Executable) binaries.

## Project Overview

**Purpose**: Resolve ordinal imports and synchronize symbol names/types across binaries by loading external BNDB files.

**Core Functionality**:
- Resolve ordinal imports by matching against exported functions from loaded DLL BNDB files
- Transfer type information from external binaries to current binary view
- Generate debug graphs showing PE import/export tables and binary relationships

## Development Commands

No traditional build system. This is a pure Python plugin for Binary Ninja.

**Testing**:
- Load plugin in Binary Ninja: place directory in Binary Ninja's plugins folder
- Test commands via Binary Ninja's "PE" menu:
  - `PE\Load binary` - Register current binary in the PE registry
  - `PE\Resolve imports` - Resolve import names and load types
  - `PE\Debug\PE tables` - Show IAT/EAT visualization
  - `PE\Debug\Binary relationship graph` - Show loaded BV relationships

**No unit tests, CI, or linting configured.**

## Code Organization

```
peutils/
├── __init__.py          # Plugin entry point, command registration
├── pe_parsing.py        # Low-level PE format parsing (imports, exports, headers)
├── sync.py              # Import resolution from external BNDBs
├── reports.py           # Graph visualization functions
├── plugin.json          # Binary Ninja plugin metadata
└── screens/             # Documentation screenshots
```

### File Responsibilities

- `__init__.py`: Registers plugin commands, maintains `files` dict of loaded BVs
- `pe_parsing.py`: All PE structure parsing (EAT, IAT, exports, imports, headers)
- `sync.py`: Resolves imports by matching ordinals against exports from loaded BVs
- `reports.py`: Generates FlowGraph visualizations using Binary Ninja's graph API

## Code Conventions

### Naming
- Functions: `snake_case`
- Classes: `PascalCase` (Export, Library, Import)
- Constants: `UPPER_SNAKE_CASE` (PE_32BIT, PE_64BIT)
- Module-level variables: `snake_case`

### Binary Ninja API Usage
- Always use `bv.parent_view` for raw binary access
- Use `bv.start` for virtual address calculations
- Register commands via `PluginCommand.register()` with `is_valid=bv_is_pe`
- Use logging functions: `log_info()`, `log_warn()`, `log_error()` (not print)
- Check `bv.view_type == "PE"` before PE-specific operations

### PE Parsing Patterns
- Read integers using `read_int(bv, addr, length)` helper
- Use `codecs.encode(bytes[::-1], "hex")` for little-endian byte-to-int conversion
- Handle both 32-bit (magic 0x10b) and 64-bit (magic 0x20b) PE formats
- PE header offset is always at `base_addr + 0x3c`
- Directory offsets differ between 32/64-bit:
  - EAT: 0x78 (32-bit), 0x88 (64-bit)
  - IAT: 0x80 (32-bit), 0x90 (64-bit)

### Symbol Management
- Use `bv.get_symbol_at(addr)` to retrieve symbols
- Use `bv.define_auto_symbol()` to create symbols
- Use `bv.undefine_auto_symbol()` to remove auto-generated symbols
- Symbol names: use `symbol.full_name` (demangled) not `symbol.name` (mangled)
- Use `SymbolType.ImportAddressSymbol` for IAT symbols
- Namespace imports by library name (e.g., `kernel32` for `kernel32.dll`)

### Type Transfer
- Get function type from export: `export_func.type_tokens`
- Parse type strings with `bv.parse_type_string(type_string)`
- Define data variables: `bv.define_data_var(addr, type)`

## Important Patterns and Gotchas

### PE Format Handling
- Integer sizes for PE tables/headers are the same between 32 and 64-bit (except certain pointers in import directory)
- The raw binaryview on 64-bit has issues; prefer the PE view when available
- Export address table entries contain RVAs relative to image base, not absolute addresses

### Ordinal Import Resolution
- Ordinal imports have the MSB set in the lookup value
- Strip ordinal flag: `lookup ^= 1 << (bv.address_size * 8 - 1)`
- Ordinals are 16-bit values regardless of architecture
- Name-based imports have 2-byte ordinal followed by ASCII name string

### Export Name Handling
- A function can be exported multiple times with the same name but different ordinals
- Duplicate exports are suffixed: `export_name`, `export_name#2`, `export_name#3`, etc.
- Use `name_counter` dict to track export name occurrences
- Export names use demangled names from BN's database

### Binary View Registry
- `peutils.files` dict tracks loaded BVs by lowercase DLL name
- Load DLLs via "PE\Load binary" command before resolving imports
- Keys are lowercase: `files[name.lower()] = bv`

### C-String Reading
- Use `read_cstring(bv, addr)` helper: reads until null terminator
- C-strings are ASCII in PE files
- Python 2/3 compatibility: use `decode_as = "ascii"` for Python 3

### Flow Graph Visualization
- Use `FlowGraph` and `FlowGraphNode` from binaryninja.flowgraph
- Nodes use `lines` attribute (list of strings or DisassemblyTextLine)
- Edges use `add_outgoing_edge(BranchType, node)`
- Display tokens: `InstructionTextToken(type, text, value)`
- Token types: `AddressDisplayToken`, `CodeSymbolToken`, `ImportToken`, etc.

## Known Issues and TODOs

From code comments in `__init__.py`:

- Proper handling of users loading EAT-less binaries using the load command
- Use StructuredDataView for parsing instead of manual offset reading
- Symbol syncing (currently only imports are synced)
- Automatically register new views when loaded
- Use database symbol names for exports/imports when using symbol syncing
- Imports with jump stubs may not get their types set correctly (commented code at sync.py:76-90)

## Binary Ninja API Reference

The Binary Ninja Python API documentation is at <https://api.binary.ninja/>.

**Key imports**:
- `binaryninja.plugin.PluginCommand` - Command registration
- `binaryninja.flowgraph.FlowGraph, FlowGraphNode` - Graph visualization
- `binaryninja.function.DisassemblyTextLine, InstructionTextToken` - Display formatting
- `binaryninja.enums.InstructionTextTokenType, BranchType, SymbolType` - Enum values
- `binaryninja.types.Symbol` - Symbol definition
- `binaryninja.log.log_info, log_warn, log_error` - Logging

**Minimum Binary Ninja Version**: 2576 (from plugin.json)

## Testing Workflow

1. Open Binary Ninja
2. Load a PE binary that imports from DLLs
3. Load the DLL binaries (e.g., OLEAUT32.dll, kernel32.dll) as separate BVs
4. Run "PE\Load binary" on each DLL to register them
5. Run "PE\Resolve imports" on the target binary
6. Verify:
   - Ordinal imports are resolved to symbol names
   - IAT entries are renamed
   - Function types are applied
   - Use "PE\Debug\PE tables" to verify the state

## Debugging

Use Binary Ninja's log window to see plugin output:
- `log_info()` - Informational messages
- `log_warn()` - Warnings (e.g., unable to find export)
- `log_error()` - Errors (e.g., invalid type parsing)

The "PE\Debug" commands provide visual feedback:
- "PE tables" - Shows IAT and EAT as the plugin sees them
- "Binary relationship graph" - Shows which libraries are loaded and their dependencies
