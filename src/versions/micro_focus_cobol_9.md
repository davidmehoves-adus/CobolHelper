# Micro Focus Visual COBOL / COBOL 9

## Description

Micro Focus Visual COBOL (built on the COBOL 9 compiler engine) is a commercial COBOL development system from Micro Focus (now part of OpenText). It targets Windows, Linux, and UNIX platforms with optional .NET and JVM managed code generation. It evolved from earlier Micro Focus products (Net Express, Server Express) and is widely used for modernizing and running mainframe COBOL applications on distributed platforms. The compiler supports COBOL-85, most of COBOL 2002/2014, plus extensive proprietary extensions.

## Table of Contents

- [Unique Syntax & Extensions](#unique-syntax--extensions)
- [Behavioral Differences](#behavioral-differences)
- [Compiler Directives & Options](#compiler-directives--options)
- [Runtime Environment](#runtime-environment)
- [Deprecated & Removed Features](#deprecated--removed-features)
- [Migration Notes](#migration-notes)
- [Gotchas](#gotchas)

## Unique Syntax & Extensions

### Managed COBOL (.NET and JVM)

Micro Focus extends COBOL to target .NET CLR and JVM runtimes. This is called **Managed COBOL** and introduces syntax not found in any other COBOL compiler.

**Class definitions targeting .NET/JVM:**
```cobol
       CLASS-ID. MyClass INHERITS TYPE System.Object.
       ...
       METHOD-ID. GetValue.
       PROCEDURE DIVISION RETURNING result AS TYPE System.String.
           SET result TO "Hello"
           GOBACK.
       END METHOD GetValue.
       END CLASS MyClass.
```

Key managed COBOL features:
- `TYPE` keyword to reference .NET/JVM types (e.g., `TYPE System.Collections.Generic.List[OF TYPE System.String]`)
- `CREATE` verb to instantiate objects (replaces `INVOKE ... NEW`)
- `SET obj TO NEW ClassName(args)` syntax for object creation
- Generic type support with `[OF TYPE ...]` syntax
- `TRY ... CATCH ... FINALLY ... END-TRY` exception handling
- `RAISE` statement to throw exceptions
- Delegates and events with `DELEGATE-ID` and `EVENT-ID`
- Properties with `PROPERTY-ID` (getter/setter)
- Indexers with `INDEXER-ID`
- `ITERATOR-ID` for implementing iterators
- `OPERATOR-ID` for operator overloading
- Enumerations with `ENUM-ID`
- `VALUETYPE-ID` for .NET value types / JVM primitives
- `INTERFACE-ID` for interface definitions
- Lambda expressions and anonymous methods via `PERFORM ... RETURNING` inline delegates
- `ILUSING` directive to import .NET/JVM namespaces (analogous to C# `using`)

### OO COBOL Extensions (Native)

Even outside managed code, Micro Focus supports OO COBOL extensions beyond the COBOL 2002 standard:
- `INVOKE` statement with `USING BY VALUE / BY REFERENCE`
- `OBJECT REFERENCE` data items
- Factory objects and factory methods
- `METHOD-ID ... OVERRIDE` for polymorphism
- `PROPERTY-ID` with `GET` and `SET`

### JSON GENERATE and JSON PARSE

Micro Focus added JSON support as an extension (not in COBOL 85 or standard COBOL 2014):
```cobol
       JSON GENERATE json-text FROM data-structure
           COUNT IN json-length
           NAME OF field-1 IS "jsonFieldName"
           SUPPRESS WHEN ZERO
           ON EXCEPTION
               DISPLAY "JSON error"
           END-JSON

       JSON PARSE json-text INTO data-structure
           NAME OF field-1 IS "jsonFieldName"
           ON EXCEPTION
               DISPLAY "Parse error"
           END-JSON
```
- `NAME OF ... IS` remaps COBOL field names to JSON keys
- `SUPPRESS` clause omits fields with zero/space values
- `ON EXCEPTION / NOT ON EXCEPTION` for error handling

### XML GENERATE and XML PARSE Enhancements

Micro Focus extends the COBOL 2002 XML support:
- `XML-NTEXT` special register for namespace-aware processing
- `XML-EVENT` values include Micro Focus-specific events beyond the standard set
- `NAMESPACE` and `NAMESPACE-PREFIX` clauses on XML GENERATE
- `ATTRIBUTES` clause to generate XML attributes instead of elements
- `NAME OF ... IS` remapping (same pattern as JSON)
- `TYPE OF ... IS ATTRIBUTE` to force specific fields as XML attributes

### TYPEDEF and STRONG Typing

```cobol
       01 customer-id-type IS TYPEDEF PIC X(10).
       01 ws-customer-id USAGE customer-id-type.
```
- `IS TYPEDEF` creates a named type
- `STRONG` keyword on typedef prevents implicit assignment from non-matching types
- Enables compile-time type checking not available in standard COBOL

### Pointer Arithmetic and Memory

- `POINTER` data type with arithmetic: `SET ptr UP BY LENGTH OF structure`
- `SET ADDRESS OF` to map linkage section items to memory addresses
- `CALL "CBL_ALLOC_MEM"` / `"CBL_FREE_MEM"` for dynamic memory allocation
- `BY VALUE` and `BY REFERENCE` at the individual parameter level in CALL
- Pointer comparison with relational operators

### String Handling Extensions

- `DISPLAY ... UPON CONSOLE` with positioning via `AT LINE ... COLUMN ...`
- Extended `INSPECT` with `CONVERTING` clause (also in standard but MF added early)
- `FUNCTION SUBSTITUTE` for search-and-replace within strings (MF-specific before COBOL 2014 adopted it)
- `FUNCTION TRIM` (leading, trailing, both)
- `FUNCTION CONCATENATE` (variadic)
- `FUNCTION FORMATTED-DATETIME` for date/time string formatting

### Inline PERFORM Extensions

```cobol
       PERFORM VARYING ws-i FROM 1 BY 1
           UNTIL ws-i > 10
           AFTER ws-j FROM 1 BY 1
           UNTIL ws-j > 5
           ...
       END-PERFORM
```
Micro Focus supports `PERFORM ... WITH TEST BEFORE/AFTER` inline with `END-PERFORM` in all dialects. Also supports `EXIT PERFORM` and `EXIT PERFORM CYCLE` for early loop exit and continue.

### Screen Section Extensions

Micro Focus provides extensive `SCREEN SECTION` support for console UI:
```cobol
       SCREEN SECTION.
       01 main-screen.
          05 BLANK SCREEN.
          05 LINE 3 COL 10 VALUE "Enter name:".
          05 LINE 3 COL 25 PIC X(30)
             USING ws-name
             HIGHLIGHT UNDERLINE
             REQUIRED.
```
- `HIGHLIGHT`, `LOWLIGHT`, `REVERSE-VIDEO`, `BLINK`, `UNDERLINE`
- `FOREGROUND-COLOR` / `BACKGROUND-COLOR` (0-7 color codes)
- `REQUIRED`, `SECURE` (password masking), `AUTO`, `FULL`
- `ACCEPT screen-name` with `ON EXCEPTION` for function key handling
- `CRT STATUS` special register for key detection

### File Handling Extensions

- `ORGANIZATION LINE SEQUENTIAL` for text files with OS-native line endings (not in IBM mainframe COBOL)
- `SELECT ... ASSIGN TO` accepts literal filenames, environment variable names, or data-name resolution
- `SHARING` clause: `SHARING WITH ALL OTHER / NO OTHER / READ ONLY`
- `RETRY` clause on file operations for record locking
- `FILE STATUS` supports extended 2-byte status codes plus MF-specific codes (9/xxx series)

### Miscellaneous Extensions

- `EVALUATE TRUE ALSO TRUE` (standard) plus `EVALUATE OBJECT` for OO dispatch
- `>>DEFINE`, `>>IF`, `>>ELSE`, `>>END-IF` compile-time conditional directives
- `$IF`, `$ELSE`, `$END`, `$DEFINE` (Micro Focus-style conditional compilation)
- `COPY ... REPLACING ==pseudo-text== BY ==replacement==` (standard, but MF supports `REPLACING LEADING/TRAILING`)
- `78` level constants: `78 WS-MAX-SIZE VALUE 100.` (compile-time constants)
- `66` level RENAMES fully supported
- Hexadecimal literals: `X"0D0A"`, `H"FF"`, `NX"..."` for national hex

## Behavioral Differences

### Binary Data: COMP vs COMP-5

This is one of the most critical behavioral differences. See `src/cobol/data_types.md` for general COMP info.

| Usage | Micro Focus Behavior |
|-------|---------------------|
| `COMP` / `COMP-4` / `BINARY` | **Truncated to PIC size** by default. A `PIC 9(4) COMP` holds max 9999, not 65535. Overflow is truncated. This matches IBM mainframe behavior when `TRUNC(STD)` is in effect. |
| `COMP-5` | **Native binary** — uses full capacity of the allocated bytes. A `PIC 9(4) COMP-5` (2 bytes) holds 0-65535. This is the Micro Focus "native" format and matches IBM `TRUNC(BIN)`. |

The `COMP` directive can change this globally: `$SET COMP"5"` makes all COMP items behave as COMP-5 (native binary). This is commonly set for performance but changes truncation semantics.

### Sign Handling

- Default sign representation is **ASCII** (e.g., trailing overpunch uses `{`, `A`-`I` for 0-9 positive, `}`, `J`-`R` for 0-9 negative)
- IBM mainframe uses EBCDIC overpunch characters — different bit patterns for the same logical signs
- `SIGN IS LEADING SEPARATE` / `SIGN IS TRAILING SEPARATE` works the same conceptually but byte values differ
- `$SET SIGN"EBCDIC"` directive forces EBCDIC-style sign overpunch for mainframe data compatibility

### Numeric Truncation

- `COMP` items truncate to PIC precision by default (as noted above)
- `TRUNC` directive controls this: `$SET TRUNC"STD"` (default), `$SET TRUNC"BIN"`, `$SET TRUNC"OPT"`
- Intermediate arithmetic precision is controlled by `ARITHMETIC"NATIVE"` vs `ARITHMETIC"STANDARD"`
- Native arithmetic uses platform-native binary ops; standard arithmetic uses decimal-safe routines

### CALL Conventions

Micro Focus supports multiple calling conventions not found in mainframe COBOL:

```cobol
       CALL "myFunction" WITH STDCALL USING BY VALUE ws-param
       CALL "myFunction" WITH CDECL USING BY REFERENCE ws-param
```

- `STDCALL` — Windows standard calling convention
- `CDECL` — C calling convention
- `PASCAL` — Pascal calling convention
- Default is configurable via `$SET CALLCONV` directive
- `BY VALUE` passes actual value (not address) — critical for C interop
- `BY CONTENT` passes a copy by reference (modifiable copy, original unchanged)
- `RETURNING` clause captures function return values: `CALL "func" RETURNING ws-result`
- Can call C functions, .NET methods, JVM methods, and Windows DLL entry points directly

### File Status Codes

Standard codes (00, 10, 23, etc.) match IBM. Micro Focus adds extended codes:

| Status | Meaning |
|--------|---------|
| 9/001 | Insufficient buffer space |
| 9/002 | File not open |
| 9/003 | Serial mode error (sequential access violation) |
| 9/013 | File not found |
| 9/014 | Too many files open |
| 9/065 | File locked |
| 9/068 | Record locked |
| 9/161 | File header damaged |

The first byte `9` indicates an MF-specific error; the three-digit code after `/` is the detail. These do NOT exist on IBM.

### ACCEPT / DISPLAY

- `ACCEPT ws-field FROM CONSOLE` reads from stdin
- `ACCEPT ws-field FROM ENVIRONMENT "VAR-NAME"` reads OS environment variables (MF extension)
- `ACCEPT ws-field FROM COMMAND-LINE` reads command-line arguments
- `DISPLAY ... UPON ENVIRONMENT-NAME / ENVIRONMENT-VALUE` sets environment variables
- `ACCEPT ws-date FROM DATE YYYYMMDD` — MF supports the `YYYYMMDD` variant directly
- Screen-based ACCEPT/DISPLAY with AT clause: `DISPLAY "text" AT 0310` (line 3, col 10)

### Numeric Comparisons

- When comparing `COMP-5` to a display numeric, Micro Focus converts to the larger precision. No automatic truncation to PIC size during comparisons.
- Unsigned COMP items compared to signed items: sign is respected (no unsigned-to-signed mismatch bugs as seen in some other compilers).

## Compiler Directives & Options

### $SET Directive

The primary mechanism for compiler configuration in Micro Focus. Placed at column 7+ in the source:

```cobol
      $SET SOURCEFORMAT"FREE"
      $SET DIALECT"MF"
      $SET COMP"5"
      $SET ASSIGN"EXTERNAL"
```

Or specified on the command line: `cobol myprogram.cbl DIALECT"MF" SOURCEFORMAT"FREE"`

### Key Directives

| Directive | Values / Purpose |
|-----------|-----------------|
| `DIALECT` | `"MF"` (default), `"OSVS"` (IBM OS/VS COBOL compat), `"VSC2"` (IBM VS COBOL II compat), `"ENTCOBOL"` (IBM Enterprise COBOL compat). Sets a bundle of other directives for compatibility. |
| `SOURCEFORMAT` | `"FIXED"` (columns 1-6 seq, 7 indicator, 8-72 code), `"FREE"` (no column rules), `"VARIABLE"` (variable-length lines). |
| `CHARSET` | `"ASCII"` (default on distributed), `"EBCDIC"` (forces EBCDIC collation and character handling). |
| `ASSIGN` | `"EXTERNAL"` (file name from env var matching SELECT name), `"DYNAMIC"` (file name from data item at runtime). |
| `COMP` | `"COMP"` (default, truncating), `"COMP-5"` or `"5"` (native binary). |
| `SIGN` | `"ASCII"` (default), `"EBCDIC"` (mainframe-compatible sign overpunch). |
| `TRUNC` | `"STD"`, `"BIN"`, `"OPT"`. Controls binary field truncation behavior. |
| `FLAGSTD` | Flags non-standard syntax. Values: `"MF"`, `"HIGH"`, `"INTER"`, `"MIN"`. |
| `COPYEXT` | Specifies file extensions for COPY statement resolution: `COPYEXT"cpy,cbl,cob"`. |
| `ILUSING` | Imports .NET/JVM namespaces: `$SET ILUSING"System.Collections.Generic"`. |
| `MF` | Boolean — enables Micro Focus extensions. On by default in DIALECT"MF". |
| `OSVS` | Boolean — enables IBM OS/VS COBOL compatibility mode. |
| `VSC2` | Boolean — enables IBM VS COBOL II compatibility mode. |
| `ENTCOBOL` | Boolean — enables IBM Enterprise COBOL compatibility mode. |
| `INTLEVEL` | Sets intermediate code level for .int files. |
| `ANIM` | Generates animation (debug) information. |
| `NOTRUNC` | Disables truncation on COMP items entirely. |
| `LIST` | Generates a listing file. |
| `XREF` | Generates cross-reference listing. |
| `XMLPARSE"XMLSS"` / `"COMPAT"` | Controls XML PARSE behavior — standard vs IBM-compatible. |
| `NSYMBOL"NATIONAL"` / `"DBCS"` | Controls interpretation of N-type literals. |
| `ARITHMETIC"NATIVE"` / `"STANDARD"` | Controls intermediate arithmetic precision. |

### Conditional Compilation

```cobol
      $SET CONSTANT MF-DEBUG = 1

      $IF MF-DEBUG = 1
       DISPLAY "Debug: " ws-value
      $END
```

Also supports ISO 2002 style:
```cobol
      >>DEFINE DEBUG-MODE AS 1
      >>IF DEBUG-MODE = 1
       DISPLAY "Debug mode"
      >>END-IF
```

### Important Compilation Flags (Command Line)

| Flag | Purpose |
|------|---------|
| `NOMAIN` | Compiles as a subprogram (no main entry point). |
| `GNT` | Generates native .gnt code (faster than .int). |
| `INT` | Generates intermediate .int code (portable, debuggable). |
| `ILGEN` | Generates .NET IL (managed code compilation). |
| `JVMGEN` | Generates JVM bytecode (managed code compilation). |
| `NOBOUND` | Disables array bounds checking (performance). |
| `BOUND` | Enables array bounds checking (debugging). |
| `OPT` | Optimization level for native code. |
| `SQL"ORACLE"` / `"DB2"` / `"ODBC"` | Selects SQL preprocessor target. |
| `COBSQL` | Activates embedded SQL preprocessing. |

## Runtime Environment

### Runtime System

Micro Focus COBOL applications require the Micro Focus Runtime System:

| Component | Purpose |
|-----------|---------|
| `cobrun` | Runtime launcher for .int/.gnt programs on UNIX/Linux. |
| `cobrts` / `cobrts32` | Runtime shared library (equivalent of libc for COBOL). |
| `cobconfig` / `cobconfig_` | Runtime configuration file. |
| Micro Focus License Manager (mflm) | Manages runtime/development licenses. |

### Code Formats and Deployment

| Format | Extension | Description |
|--------|-----------|-------------|
| Intermediate | `.int` | Portable bytecode. Slower execution. Fully debuggable. |
| Generated native | `.gnt` | Platform-native machine code. Fast. Debuggable with ANIM. |
| Shared library | `.dll` (Win) / `.so` (UNIX) | Compiled to OS-native shared library. Fastest. |
| .NET assembly | `.dll` (managed) | .NET IL compiled via ILGEN. Runs on CLR. |
| JVM class | `.class` / `.jar` | JVM bytecode compiled via JVMGEN. Runs on JVM. |
| Executable | `.exe` | Standalone executable (Windows). |

Typical deployment: compile to `.dll`/`.so` for production, use `.int` for development/debugging.

### COBCONFIG / Runtime Configuration

The runtime configuration file (pointed to by `COBCONFIG` or `COBCONFIG_` environment variable) controls runtime behavior:

```
[Runtime]
file_trace=true
file_trace_filename=/tmp/file_trace.log
default_cancel_mode=1
```

Key runtime tuning settings:

| Variable | Purpose |
|----------|---------|
| `COBCONFIG` | Path to runtime config file (traditional). |
| `COBCONFIG_` | Path to runtime config file (newer, overrides COBCONFIG). |
| `COBDIR` | Micro Focus installation directory. |
| `COBPATH` | Search path for .int/.gnt programs (like PATH for COBOL). |
| `COBCPY` | Search path for COPY files. |
| `COBOPT` | Default compiler options file. |
| `MFCODESET` | Character set mapping. |
| `dd_FILENAME` | Maps DD names to physical files (JCL emulation). |
| `HCOBND` | DB2 bind file directory. |

### File Handler (EXTFH)

Micro Focus uses its own file handler, which is configurable and replaceable:

- **Micro Focus File Handler (MFFH)** — default handler for sequential, relative, and indexed files
- **EXTFH interface** — standardized API allowing pluggable third-party file handlers (e.g., AcuCOBOL Vision files, C-ISAM)
- Indexed files use **Micro Focus ISAM** format by default (not IBM VSAM)
- Configuration via `EXTFH.CFG` file:

```
[MYFILE]
ORGANIZATION=INDEXED
ACCESS=DYNAMIC
FILENAME=/data/myfile.dat
```

- File handler tuning: `STRIP-TRAILING-SPACES`, `DEFAULT-FILE-ORG`, `FILE-TRACE` settings in COBCONFIG
- `FILESHARE` — Micro Focus file sharing server for multi-user indexed file access (replaces CICS file control for batch)

### Environment Variable File Mapping

Micro Focus resolves file names via environment variables by default when `ASSIGN"EXTERNAL"` is set:

```cobol
       SELECT CUSTOMER-FILE ASSIGN TO "CUSTFILE".
```
At runtime, set `CUSTFILE=/data/customers.dat` in the environment. The runtime resolves the assignment from the environment variable name matching the literal.

With `ASSIGN"DYNAMIC"`:
```cobol
       SELECT CUSTOMER-FILE ASSIGN TO WS-FILENAME.
```
The file name comes from the data item `WS-FILENAME` at OPEN time.

## Deprecated & Removed Features

### Syntax Not Supported or Restricted

- **EXEC CICS / EXEC DLI** — not natively supported in the compiler. Requires the **Enterprise Server** add-on product, which provides CICS emulation. Without it, EXEC CICS blocks cause compilation errors.
- **EXEC SQL (DB2 native)** — requires the COBSQL preprocessor or the DB2 ECM (External Compiler Module). Not the same DB2 preprocessor as on z/OS. SQL syntax differences exist (see Migration Notes).
- **Report Writer** — supported but often with behavioral differences from IBM. The `USE BEFORE REPORTING` declarative may behave differently.
- **COMMUNICATION SECTION** — deprecated; not functional (removed from COBOL 2002 standard).
- **Segmentation (SECTION priority numbers)** — parsed but ignored. `PRIORITY` clauses have no effect.
- **TALLY special register** — supported for compatibility but should be avoided; it is a legacy register.
- **ALTER** statement — supported for compatibility but generates warnings; should not be used in new code.
- **IBM-specific registers** — `WHEN-COMPILED` register format may differ. `RETURN-CODE` works but initial value assumptions may differ.

### IBM Features Not Available Without Enterprise Server

| Feature | Requirement |
|---------|-------------|
| EXEC CICS | Enterprise Server |
| JCL execution | Enterprise Server JES emulation |
| VSAM (actual VSAM API) | Enterprise Server or MFFH emulation |
| IMS/DB (DL/I) | Enterprise Server IMS option |
| MQ Series native | Separate MQ client config |

## Migration Notes

### From IBM Enterprise COBOL to Micro Focus

**Character set**: IBM uses EBCDIC; Micro Focus defaults to ASCII. This affects:
- Collating sequence (`IF "A" < "1"` gives different results)
- Sign overpunch characters (different bit patterns)
- SORT ordering
- `INSPECT CONVERTING` character ranges
- Hex literal values (`X"C1"` = "A" in EBCDIC, but not in ASCII)

**Mitigation**: `$SET CHARSET"EBCDIC"` forces EBCDIC collating sequence. `$SET SIGN"EBCDIC"` forces EBCDIC sign conventions. `DIALECT"ENTCOBOL"` sets both plus other IBM-compatible behaviors.

**COMP behavior**: IBM's default with `TRUNC(STD)` matches MF default. But if IBM source was compiled with `TRUNC(BIN)`, set `$SET COMP"5"` or `$SET TRUNC"BIN"` on MF.

**ASSIGN clause**: IBM uses `ASSIGN TO CUSTFILE` where `CUSTFILE` maps via JCL DD. MF uses environment variables or data items. Set `ASSIGN"EXTERNAL"` and map DD names to env vars: `export dd_CUSTFILE=/data/custfile.dat` (note the `dd_` prefix convention).

**COPY library resolution**: IBM uses PDS (partitioned datasets). MF uses filesystem directories. Set `COBCPY` to the directory path containing copybooks.

**EXEC SQL differences**:
- MF uses COBSQL preprocessor, not the IBM DB2 precompiler
- Host variable declarations may require `:` prefix in some configurations
- `EXEC SQL INCLUDE` resolves from `COBCPY` path, not a DBRM library
- Some DB2-specific SQL syntax may need adjustment (e.g., `CURRENT TIMESTAMP` works, but `CURRENT SERVER` may differ)
- NULL indicator handling works the same conceptually but check `-0` vs `SQLCODE` behavior

**EXEC CICS differences** (with Enterprise Server):
- Most EXEC CICS commands work identically
- Transaction routing, MRO, and ISC configurations differ (server config vs CEDA/RDO)
- BMS maps compile with the MF BMS compiler (different utility name)
- `HANDLE CONDITION` / `HANDLE AID` work but `RESP/RESP2` style is recommended

**File status code mapping**: Standard 2-byte codes (00, 10, 23, 35, etc.) match IBM. But MF returns `9/nnn` extended codes that have no IBM equivalent. Migration code should check only the 2-byte standard portion unless MF-specific handling is needed.

**Floating-point**: IBM uses hexadecimal floating point (HFP) by default for COMP-1/COMP-2. Micro Focus uses IEEE 754. Results may differ in the last significant digits for COMP-1/COMP-2 arithmetic.

### From Micro Focus to Other Platforms

- Remove `$SET` directives (not portable)
- Replace `COMP-5` with `COMP` if target uses `TRUNC(BIN)`, or explicitly handle truncation
- Replace `ORGANIZATION LINE SEQUENTIAL` with `ORGANIZATION SEQUENTIAL` and handle line endings separately
- Remove Screen Section code (not supported on IBM)
- Replace `ACCEPT FROM ENVIRONMENT` with platform-appropriate mechanism
- Replace `CALL ... WITH STDCALL/CDECL` with standard `CALL` syntax
- Remove `78`-level constants if target does not support them (IBM does since Enterprise COBOL V5)
- Replace `EXIT PERFORM CYCLE` if target does not support it

## Gotchas

### COMP vs COMP-5 Truncation Trap
The most common bug when migrating to/from Micro Focus. If a program uses `PIC 9(4) COMP` and stores value 12345, Micro Focus truncates to 2345 (modulo 10000) by default. On IBM with `TRUNC(BIN)` this would store 12345 (within 2-byte range). Always verify the `COMP`/`TRUNC` directive when porting.

### ASSIGN"EXTERNAL" File Name Resolution
If a SELECT says `ASSIGN TO "CUSTFILE"` and no environment variable `CUSTFILE` is set, the runtime looks for a physical file literally named `CUSTFILE` in the current directory. This silent fallback causes confusion when env vars are missing.

### Line Sequential vs Sequential
`LINE SEQUENTIAL` files have OS-native line endings (CR+LF on Windows, LF on UNIX). `SEQUENTIAL` files are fixed-length record format with no delimiters. Using the wrong organization corrupts data. IBM mainframe has no `LINE SEQUENTIAL` concept.

### EBCDIC Sort Order
ASCII: digits < uppercase < lowercase. EBCDIC: lowercase < uppercase < digits. Programs that depend on sort/comparison ordering break when moving between platforms without `CHARSET"EBCDIC"`.

### Screen Section Portability
Screen Section code (colors, positioning) compiles only on Micro Focus. It is not portable to IBM, Fujitsu, or GnuCOBOL (though GnuCOBOL has its own partial implementation).

### 78-Level Constants
`78` levels are resolved at compile time. They cannot be used in `EVALUATE WHEN` conditions that expect data items (they are literals). They also cannot be passed `BY REFERENCE` in `CALL` statements.

### Intermediate Code (.int) Performance
`.int` files are interpreted and roughly 5-10x slower than `.gnt` (native generated code). Never deploy `.int` in production for performance-sensitive workloads. Use `.gnt` or compile to `.dll`/`.so`.

### RETURN-CODE Register
Micro Focus initializes `RETURN-CODE` to 0. IBM initializes it to 0 as well, but the register is shared across CALLed programs differently. In MF, each CALL'd program gets its own `RETURN-CODE` on entry. Check `RETURN-CODE` immediately after `CALL` before any other statement modifies it.

### Recursive Programs
Micro Focus supports `RECURSIVE` attribute on PROGRAM-ID. Without it, calling a program recursively corrupts WORKING-STORAGE (which is static). This matches IBM Enterprise COBOL V5+ behavior but differs from earlier IBM versions where recursion was simply unsupported.

### COBPATH Search Order
When a `CALL "SUBPROG"` is executed, the runtime searches: (1) already-loaded modules, (2) current directory, (3) `COBPATH` directories in order. If the wrong version of a subprogram is in an earlier path position, it silently loads the wrong module.

### Null-Terminated Strings in C Interop
When calling C functions, strings must be null-terminated. Use `FUNCTION CONCATENATE(ws-string, X"00")` or `STRING ws-string DELIMITED SIZE X"00" DELIMITED SIZE INTO ws-c-string`. Forgetting the null terminator causes buffer overruns in the C code.
