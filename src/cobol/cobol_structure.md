# COBOL Structure

## Description
This file covers the fundamental structure of a COBOL program: the four divisions
(IDENTIFICATION, ENVIRONMENT, DATA, PROCEDURE), the sections and paragraphs within
each division, the column-based reference format (fixed format) versus free format,
and the standard program skeleton. Reference this file when you need to understand
how a COBOL program is organized or when diagnosing structural errors.

## Table of Contents
- [Core Concepts](#core-concepts)
  - [The Four Divisions](#the-four-divisions)
  - [Column Layout (Reference Format)](#column-layout-reference-format)
  - [Fixed Format vs Free Format](#fixed-format-vs-free-format)
  - [Area A vs Area B Rules](#area-a-vs-area-b-rules)
- [Syntax & Examples](#syntax--examples)
  - [IDENTIFICATION DIVISION](#identification-division)
  - [ENVIRONMENT DIVISION](#environment-division)
  - [DATA DIVISION](#data-division)
  - [PROCEDURE DIVISION](#procedure-division)
  - [Complete Program Skeleton](#complete-program-skeleton)
  - [Free-Format Skeleton](#free-format-skeleton)
- [Common Patterns](#common-patterns)
- [Gotchas](#gotchas)
- [Related Topics](#related-topics)

## Core Concepts

### The Four Divisions

Every COBOL program is organized into up to four divisions, and they must appear in
this exact order. Omitting a division is permitted in some cases (the ENVIRONMENT
and DATA divisions are optional if they contain no entries), but the order is never
negotiable.

| Order | Division               | Purpose                                                |
|-------|------------------------|--------------------------------------------------------|
| 1     | IDENTIFICATION DIVISION | Names the program, provides metadata                  |
| 2     | ENVIRONMENT DIVISION    | Describes the computing environment and file mappings  |
| 3     | DATA DIVISION           | Defines all data items (variables, records, files)     |
| 4     | PROCEDURE DIVISION      | Contains the executable logic (statements/paragraphs)  |

Each division is subdivided into **sections**, and sections may contain
**paragraphs**. The hierarchy is:

```
DIVISION
  └── SECTION
        └── PARAGRAPH
              └── SENTENCE (one or more statements ending with a period)
```

Divisions and sections are declared with header lines. Paragraphs are declared by
a user-defined or predefined name followed by a period. Sentences are one or more
COBOL statements terminated by a period.

### Column Layout (Reference Format)

Traditional COBOL source uses a fixed-width, column-sensitive layout inherited from
80-column punch cards. The columns are divided into five areas:

| Columns | Name              | Purpose                                                  |
|---------|-------------------|----------------------------------------------------------|
| 1-6     | Sequence Number Area | Optional line numbers; ignored by the compiler          |
| 7       | Indicator Area    | Special-purpose column (see below)                       |
| 8-11    | Area A            | Division headers, section headers, paragraph names, FDs, 01/77 levels |
| 12-72   | Area B            | Statements, clauses, continuation of entries from Area A |
| 73-80   | Identification Area | Optional; ignored by the compiler (historically the program name) |

**Indicator Area (column 7) values:**

| Character | Meaning                                                          |
|-----------|------------------------------------------------------------------|
| ` ` (space) | Normal source line                                             |
| `*`       | Comment line — entire line is ignored by the compiler            |
| `/`       | Comment line with page eject (forces a new page in listing)      |
| `-`       | Continuation line — continues a non-numeric literal or word from the previous line |
| `D` or `d`| Debugging line — compiled only when WITH DEBUGGING MODE is active |

### Fixed Format vs Free Format

**Fixed format** (also called reference format) is the traditional layout described
above. It is the default in most mainframe compilers and is what the vast majority
of legacy COBOL code uses.

**Free format** removes column restrictions entirely:

- Code can start in any column (there is no Area A / Area B distinction enforced
  by column position).
- Lines can be up to 255 characters (or compiler-defined limit) instead of 72.
- No sequence number area or identification area.
- Comments use `*>` as an inline comment marker instead of `*` in column 7.
- Continuation is implicit when a literal is not closed at the end of a line, or
  uses the `&` character at the end of the continued line in some compilers.

Free format was introduced in the COBOL 2002 standard. To enable it, compilers
typically require a directive:

- IBM Enterprise COBOL: does not support free format (as of v6.x); uses reference
  format only.
- GnuCOBOL: `>>SOURCE FORMAT IS FREE` directive or `-free` compiler option.
- Micro Focus: `$SET SOURCEFORMAT"FREE"` directive or compiler option.

Most production mainframe COBOL remains in fixed format.

### Area A vs Area B Rules

Getting Area A / Area B placement wrong is one of the most common COBOL compilation
errors. The rules are:

**Must begin in Area A (columns 8-11):**
- Division headers (`IDENTIFICATION DIVISION.`)
- Section headers (`WORKING-STORAGE SECTION.`)
- Paragraph names (including `PROGRAM-ID.` and user-defined paragraph names)
- FD and SD entries
- Level-01 and level-77 data descriptions
- Class-name, object, method, and factory headers (OO COBOL)
- END PROGRAM / END CLASS / END METHOD headers

**Must begin in Area B (columns 12-72):**
- Statements and sentences within paragraphs
- Data description entries with level numbers 02-49, 66, 88
- Clauses of FD/SD entries
- CALL, MOVE, IF, PERFORM, and all other procedural statements
- Continuation of any entry that started in Area A

**May begin in either Area A or Area B:**
- Comment lines (since the entire line is a comment regardless)
- Compiler directives (compiler-dependent)

## Syntax & Examples

### IDENTIFICATION DIVISION

The IDENTIFICATION DIVISION is the only mandatory division (the `PROGRAM-ID`
paragraph is required). It names the program and provides optional documentation
metadata.

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. CUSTMAINT.
       AUTHOR. DEVELOPMENT TEAM.
       INSTALLATION. MAIN DATA CENTER.
       DATE-WRITTEN. 2024-01-15.
       DATE-COMPILED.
       SECURITY. CONFIDENTIAL.
      *
      * Optional paragraphs above (AUTHOR, INSTALLATION,
      * DATE-WRITTEN, DATE-COMPILED, SECURITY) are documentary
      * only and treated as comments by most modern compilers.
      *
```

**Sections:** None (the IDENTIFICATION DIVISION has no sections).

**Paragraphs:**

| Paragraph       | Required | Notes                                              |
|-----------------|----------|----------------------------------------------------|
| PROGRAM-ID      | Yes      | The program name (1-30 chars, typically 1-8 on mainframes) |
| AUTHOR          | No       | Treated as comment in COBOL-85 and later           |
| INSTALLATION    | No       | Treated as comment                                 |
| DATE-WRITTEN    | No       | Treated as comment                                 |
| DATE-COMPILED   | No       | Replaced with compile date by some compilers       |
| SECURITY        | No       | Treated as comment                                 |

The `PROGRAM-ID` paragraph may also include the `IS INITIAL` or `IS COMMON` clause:

```cobol
       PROGRAM-ID. SUBPROG IS INITIAL.
```

`IS INITIAL` causes all data items in WORKING-STORAGE to be reinitialized to their
VALUE clauses each time the program is called. `IS COMMON` allows a nested program
to be called from any program in the same source file.

### ENVIRONMENT DIVISION

The ENVIRONMENT DIVISION describes the computing environment and maps logical file
names to physical files. It contains two sections.

```cobol
       ENVIRONMENT DIVISION.
      *
       CONFIGURATION SECTION.
       SOURCE-COMPUTER. IBM-370.
       OBJECT-COMPUTER. IBM-370.
       SPECIAL-NAMES.
           DECIMAL-POINT IS COMMA.
      *
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT CUSTOMER-FILE
               ASSIGN TO CUSTFILE
               ORGANIZATION IS INDEXED
               ACCESS MODE IS DYNAMIC
               RECORD KEY IS CUST-ID
               FILE STATUS IS WS-FILE-STATUS.
           SELECT REPORT-FILE
               ASSIGN TO RPTFILE
               ORGANIZATION IS SEQUENTIAL
               FILE STATUS IS WS-RPT-STATUS.
      *
       I-O-CONTROL.
           SAME RECORD AREA FOR CUSTOMER-FILE REPORT-FILE.
```

#### CONFIGURATION SECTION

Contains three optional paragraphs:

| Paragraph        | Purpose                                                       |
|------------------|---------------------------------------------------------------|
| SOURCE-COMPUTER  | Identifies the machine used for compilation (largely documentary) |
| OBJECT-COMPUTER  | Identifies the machine for execution; can specify memory size and collating sequence |
| SPECIAL-NAMES    | Maps compiler-defined names to user-defined names; sets locale/currency conventions |

Common `SPECIAL-NAMES` entries:

```cobol
       SPECIAL-NAMES.
           DECIMAL-POINT IS COMMA
           CURRENCY SIGN IS "EUR" WITH PICTURE SYMBOL "E"
           CONSOLE IS CRT
           CURSOR IS WS-CURSOR-POS
           CRT STATUS IS WS-CRT-STATUS.
```

`DECIMAL-POINT IS COMMA` swaps the role of comma and period in numeric editing
(common in European installations). The `CURRENCY SIGN` clause allows a
multi-character currency symbol.

#### INPUT-OUTPUT SECTION

Contains two paragraphs:

| Paragraph    | Purpose                                                          |
|--------------|------------------------------------------------------------------|
| FILE-CONTROL | Maps each logical file (SELECT) to a physical assignment and describes the file organization, access mode, keys, and status |
| I-O-CONTROL  | Specifies shared buffer areas, multiple file reel handling, checkpoint/restart options (rarely used in modern code) |

The `SELECT` statement in `FILE-CONTROL` is the bridge between the logical file
name used in COBOL statements and the external file known to the operating system.
Every file that the program reads or writes must have a `SELECT` entry.

### DATA DIVISION

The DATA DIVISION is where all data items are declared. It contains several
sections, each with a specific scope and lifetime:

```cobol
       DATA DIVISION.
      *
       FILE SECTION.
       FD  CUSTOMER-FILE
           RECORD CONTAINS 200 CHARACTERS
           BLOCK CONTAINS 0 RECORDS
           RECORDING MODE IS F.
       01  CUSTOMER-RECORD.
           05  CUST-ID            PIC X(10).
           05  CUST-NAME          PIC X(40).
           05  CUST-ADDRESS       PIC X(100).
           05  CUST-BALANCE       PIC S9(7)V99 COMP-3.
           05  FILLER             PIC X(45).
      *
       FD  REPORT-FILE
           RECORD CONTAINS 133 CHARACTERS.
       01  REPORT-RECORD          PIC X(133).
      *
       WORKING-STORAGE SECTION.
       01  WS-FILE-STATUS         PIC XX.
       01  WS-RPT-STATUS          PIC XX.
       01  WS-EOF-FLAG            PIC X VALUE 'N'.
           88  WS-EOF             VALUE 'Y'.
           88  WS-NOT-EOF         VALUE 'N'.
       01  WS-COUNTERS.
           05  WS-READ-COUNT      PIC 9(7) VALUE ZEROS.
           05  WS-WRITE-COUNT     PIC 9(7) VALUE ZEROS.
      *
       LINKAGE SECTION.
       01  LS-PARM-DATA.
           05  LS-PARM-LENGTH     PIC S9(4) COMP.
           05  LS-PARM-TEXT       PIC X(100).
      *
       LOCAL-STORAGE SECTION.
       01  LC-WORK-FIELD          PIC X(50) VALUE SPACES.
```

#### FILE SECTION

The FILE SECTION describes the structure of every file declared in `FILE-CONTROL`.
Each file has an FD (File Description) or SD (Sort Description) entry followed by
one or more record descriptions (01-level items).

Key FD clauses:

| Clause             | Purpose                                               |
|--------------------|-------------------------------------------------------|
| RECORD CONTAINS    | Specifies the record length (fixed or variable range) |
| BLOCK CONTAINS     | Specifies the blocking factor (0 = system-determined) |
| RECORDING MODE     | F (fixed), V (variable), U (undefined), S (spanned)  |
| LABEL RECORDS      | Standard or omitted (obsolete but still seen)         |
| DATA RECORDS       | Names the record structures (obsolete/documentary)    |

A single FD can have multiple 01-level records. They share the same physical buffer
(they are **implicit redefinitions** of one another).

#### WORKING-STORAGE SECTION

WORKING-STORAGE is the primary area for declaring program variables. Data items
declared here:

- Are allocated when the program is first loaded.
- Persist across multiple invocations of the program (when the program is called
  as a subprogram) **unless** the program is declared `IS INITIAL`.
- Are initialized to their `VALUE` clauses once at program load time (first call).
- On subsequent calls, they retain their last-used values (static lifetime).

Items can be declared at level 01 (group or elementary) or level 77 (standalone
elementary items). Levels 02-49 define subordinate items within a group. Level 66
is for RENAMES, and level 88 is for condition-names.

#### LINKAGE SECTION

The LINKAGE SECTION describes data items that are passed to the program from a
calling program (or from the operating system via JCL PARM). Key characteristics:

- Items declared here do **not** have storage allocated by this program.
- They describe the layout of memory **owned by the caller**.
- They become addressable only after the program is called with the corresponding
  arguments via `CALL ... USING`.
- `VALUE` clauses on items in the LINKAGE SECTION define condition-name (88-level)
  values only; they do **not** initialize the data.
- The PROCEDURE DIVISION header must include a `USING` clause that lists the
  01-level (or 77-level) items from the LINKAGE SECTION:

```cobol
       PROCEDURE DIVISION USING LS-PARM-DATA.
```

Passing mechanisms:

| Mechanism    | Syntax                          | Effect                                    |
|--------------|---------------------------------|-------------------------------------------|
| BY REFERENCE | `CALL 'PGM' USING var`         | Default; callee sees caller's actual memory |
| BY CONTENT   | `CALL 'PGM' USING BY CONTENT var` | Callee gets a copy; caller's data is protected |
| BY VALUE     | `CALL 'PGM' USING BY VALUE var`   | Passes the raw value (for C interop and similar) |

#### LOCAL-STORAGE SECTION

LOCAL-STORAGE was introduced in the COBOL 2002 standard and is supported by IBM
Enterprise COBOL (v4+), GnuCOBOL, and Micro Focus. Key differences from
WORKING-STORAGE:

| Characteristic         | WORKING-STORAGE              | LOCAL-STORAGE                |
|------------------------|------------------------------|------------------------------|
| Allocation             | Once (at first load/call)    | Each time the program is called |
| Initialization         | VALUE applied once at load   | VALUE applied on every call  |
| Persistence            | Persists between calls       | Freshly allocated each call  |
| Recursive programs     | Shared (problematic)         | Separate per invocation (safe) |
| Thread safety          | Shared across threads (risky)| Separate per thread (safe)   |

LOCAL-STORAGE is the correct choice for:
- Reentrant/recursive programs.
- Threaded environments (CICS, multi-threaded batch).
- Any situation where you want guaranteed initialization on each call.

#### Other DATA DIVISION Sections

| Section                 | Purpose                                                   |
|-------------------------|-----------------------------------------------------------|
| REPORT SECTION          | Used with Report Writer (RD entries and report groups)    |
| SCREEN SECTION          | Used with ACCEPT/DISPLAY screen handling (Micro Focus, GnuCOBOL) |
| COMMUNICATION SECTION   | Used with the Communication Module (obsolete since COBOL-2002) |

### PROCEDURE DIVISION

The PROCEDURE DIVISION contains the executable logic. It is organized into
sections and paragraphs, or paragraphs alone (sections are optional in most
coding styles).

```cobol
       PROCEDURE DIVISION.
      *
       0000-MAIN-PROCESS.
           PERFORM 1000-INITIALIZE
           PERFORM 2000-PROCESS UNTIL WS-EOF
           PERFORM 3000-TERMINATE
           STOP RUN
           .
      *
       1000-INITIALIZE.
           OPEN INPUT  CUSTOMER-FILE
                OUTPUT REPORT-FILE
           READ CUSTOMER-FILE
               AT END SET WS-EOF TO TRUE
           END-READ
           .
      *
       2000-PROCESS.
           ADD 1 TO WS-READ-COUNT
           PERFORM 2100-FORMAT-DETAIL
           WRITE REPORT-RECORD
           ADD 1 TO WS-WRITE-COUNT
           READ CUSTOMER-FILE
               AT END SET WS-EOF TO TRUE
           END-READ
           .
      *
       2100-FORMAT-DETAIL.
           MOVE SPACES TO REPORT-RECORD
           STRING CUST-ID DELIMITED BY SPACES
                  " - " DELIMITED BY SIZE
                  CUST-NAME DELIMITED BY SPACES
               INTO REPORT-RECORD
           END-STRING
           .
      *
       3000-TERMINATE.
           CLOSE CUSTOMER-FILE
                 REPORT-FILE
           DISPLAY "RECORDS READ:    " WS-READ-COUNT
           DISPLAY "RECORDS WRITTEN: " WS-WRITE-COUNT
           .
```

**Sections vs Paragraphs in the PROCEDURE DIVISION:**

- A **section** has a section header (`nnnn-SECTION-NAME SECTION.`) and can contain
  multiple paragraphs. When you `PERFORM` a section, all paragraphs within it
  execute sequentially.
- A **paragraph** is simply a name followed by a period. It contains one or more
  sentences. When you `PERFORM` a paragraph, only that paragraph executes.
- In most modern shops, code is organized using **paragraphs only** (no sections),
  with a structured top-down design. Sections are still used in CICS programs
  (which sometimes require them) and in Report Writer.

**PROCEDURE DIVISION USING and RETURNING:**

For subprograms:

```cobol
       PROCEDURE DIVISION USING LS-INPUT-RECORD
                          RETURNING LS-RETURN-CODE.
```

The `USING` clause makes LINKAGE SECTION items addressable. The `RETURNING` clause
designates a single item whose value is returned to the caller and can be captured
with the `GIVING` phrase of the `CALL` statement.

**Declaratives:**

Declaratives are special sections placed at the very beginning of the PROCEDURE
DIVISION (before any other sections or paragraphs), enclosed between the
`DECLARATIVES.` and `END DECLARATIVES.` headers. They contain USE AFTER
ERROR/EXCEPTION procedures for file I/O errors and USE FOR DEBUGGING procedures
for diagnostic tracing. See [error_handling.md](error_handling.md) for full
coverage of DECLARATIVES syntax and USE AFTER procedures.

### Complete Program Skeleton

Below is a complete fixed-format skeleton showing all four divisions with their
most common sections:

```cobol
      *================================================================*
      * Program:    SKELETON                                           *
      * Purpose:    Standard COBOL program skeleton                    *
      *================================================================*
       IDENTIFICATION DIVISION.
       PROGRAM-ID. SKELETON.
      *
       ENVIRONMENT DIVISION.
      *
       CONFIGURATION SECTION.
       SOURCE-COMPUTER. X86-64.
       OBJECT-COMPUTER. X86-64.
       SPECIAL-NAMES.
           DECIMAL-POINT IS COMMA.
      *
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT INPUT-FILE
               ASSIGN TO INFILE
               ORGANIZATION IS SEQUENTIAL
               FILE STATUS IS WS-IN-STATUS.
           SELECT OUTPUT-FILE
               ASSIGN TO OUTFILE
               ORGANIZATION IS SEQUENTIAL
               FILE STATUS IS WS-OUT-STATUS.
      *
       DATA DIVISION.
      *
       FILE SECTION.
       FD  INPUT-FILE
           RECORD CONTAINS 80 CHARACTERS.
       01  INPUT-RECORD              PIC X(80).
      *
       FD  OUTPUT-FILE
           RECORD CONTAINS 132 CHARACTERS.
       01  OUTPUT-RECORD             PIC X(132).
      *
       WORKING-STORAGE SECTION.
       01  WS-IN-STATUS              PIC XX.
       01  WS-OUT-STATUS             PIC XX.
       01  WS-FLAGS.
           05  WS-EOF-FLAG           PIC X   VALUE 'N'.
               88  WS-EOF                    VALUE 'Y'.
               88  WS-NOT-EOF               VALUE 'N'.
       01  WS-COUNTERS.
           05  WS-REC-IN             PIC 9(9) VALUE ZEROS.
           05  WS-REC-OUT            PIC 9(9) VALUE ZEROS.
      *
       LINKAGE SECTION.
       01  LS-PARM.
           05  LS-PARM-LEN           PIC S9(4) COMP.
           05  LS-PARM-DATA          PIC X(100).
      *
       PROCEDURE DIVISION.
      *
       0000-MAIN.
           PERFORM 1000-INIT
           PERFORM 2000-PROCESS UNTIL WS-EOF
           PERFORM 3000-CLEANUP
           STOP RUN
           .
      *
       1000-INIT.
           OPEN INPUT  INPUT-FILE
                OUTPUT OUTPUT-FILE
           IF WS-IN-STATUS NOT = '00'
               DISPLAY 'OPEN ERROR ON INPUT: ' WS-IN-STATUS
               STOP RUN
           END-IF
           PERFORM 9000-READ-INPUT
           .
      *
       2000-PROCESS.
           ADD 1 TO WS-REC-IN
      *    -- Transform and write record --
           MOVE INPUT-RECORD TO OUTPUT-RECORD
           WRITE OUTPUT-RECORD
           ADD 1 TO WS-REC-OUT
           PERFORM 9000-READ-INPUT
           .
      *
       3000-CLEANUP.
           CLOSE INPUT-FILE
                 OUTPUT-FILE
           DISPLAY 'RECORDS READ:    ' WS-REC-IN
           DISPLAY 'RECORDS WRITTEN: ' WS-REC-OUT
           .
      *
       9000-READ-INPUT.
           READ INPUT-FILE
               AT END SET WS-EOF TO TRUE
           END-READ
           IF WS-IN-STATUS NOT = '00' AND
              WS-IN-STATUS NOT = '10'
               DISPLAY 'READ ERROR: ' WS-IN-STATUS
               STOP RUN
           END-IF
           .
```

### Free-Format Skeleton

The same program in free format looks like this (note the absence of column
restrictions and the use of `*>` for comments):

```cobol
*> ================================================================
*> Program:    SKELETON
*> Purpose:    Standard COBOL program skeleton (free format)
*> ================================================================
IDENTIFICATION DIVISION.
PROGRAM-ID. SKELETON.

ENVIRONMENT DIVISION.

CONFIGURATION SECTION.
SPECIAL-NAMES.
    DECIMAL-POINT IS COMMA.

INPUT-OUTPUT SECTION.
FILE-CONTROL.
    SELECT INPUT-FILE
        ASSIGN TO INFILE
        ORGANIZATION IS SEQUENTIAL
        FILE STATUS IS WS-IN-STATUS.
    SELECT OUTPUT-FILE
        ASSIGN TO OUTFILE
        ORGANIZATION IS SEQUENTIAL
        FILE STATUS IS WS-OUT-STATUS.

DATA DIVISION.

FILE SECTION.
FD INPUT-FILE
    RECORD CONTAINS 80 CHARACTERS.
01 INPUT-RECORD PIC X(80).

FD OUTPUT-FILE
    RECORD CONTAINS 132 CHARACTERS.
01 OUTPUT-RECORD PIC X(132).

WORKING-STORAGE SECTION.
01 WS-IN-STATUS  PIC XX.
01 WS-OUT-STATUS PIC XX.
01 WS-EOF-FLAG   PIC X VALUE 'N'.
    88 WS-EOF     VALUE 'Y'.
    88 WS-NOT-EOF VALUE 'N'.

PROCEDURE DIVISION.

0000-MAIN.
    PERFORM 1000-INIT
    PERFORM 2000-PROCESS UNTIL WS-EOF
    PERFORM 3000-CLEANUP
    STOP RUN.

1000-INIT.
    OPEN INPUT INPUT-FILE OUTPUT OUTPUT-FILE
    PERFORM 9000-READ-INPUT.

2000-PROCESS.
    MOVE INPUT-RECORD TO OUTPUT-RECORD
    WRITE OUTPUT-RECORD
    PERFORM 9000-READ-INPUT.

3000-CLEANUP.
    CLOSE INPUT-FILE OUTPUT-FILE
    DISPLAY 'COMPLETE'.

9000-READ-INPUT.
    READ INPUT-FILE
        AT END SET WS-EOF TO TRUE
    END-READ.
```

## Common Patterns

### Minimal Program (Hello World)

The absolute minimum valid COBOL program:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. HELLO.
       PROCEDURE DIVISION.
           DISPLAY "HELLO, WORLD".
           STOP RUN.
```

The ENVIRONMENT and DATA divisions are omitted entirely because the program uses
no files and no declared data.

### Subprogram Structure

A called subprogram typically omits `STOP RUN` (using `GOBACK` instead) and
receives parameters through the LINKAGE SECTION:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. CALCSUB.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-TEMP    PIC S9(9)V99 COMP-3.
       LINKAGE SECTION.
       01  LS-INPUT   PIC S9(9)V99.
       01  LS-OUTPUT  PIC S9(9)V99.
       PROCEDURE DIVISION USING LS-INPUT LS-OUTPUT.
       0000-MAIN.
           COMPUTE LS-OUTPUT = LS-INPUT * 1.10
           GOBACK
           .
```

### Nested Programs

COBOL supports nested (contained) programs within a single compilation unit:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. OUTER-PGM.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-RESULT  PIC 9(5).
       PROCEDURE DIVISION.
           CALL 'INNER-PGM' USING WS-RESULT
           DISPLAY WS-RESULT
           STOP RUN
           .
      *
       IDENTIFICATION DIVISION.
       PROGRAM-ID. INNER-PGM IS COMMON.
       DATA DIVISION.
       LINKAGE SECTION.
       01  LS-VAL     PIC 9(5).
       PROCEDURE DIVISION USING LS-VAL.
           MOVE 12345 TO LS-VAL
           GOBACK
           .
       END PROGRAM INNER-PGM.
       END PROGRAM OUTER-PGM.
```

Nested programs share the same compilation unit. The `IS COMMON` clause allows the
inner program to be called by any program in the nest. Each nested program requires
its own `END PROGRAM` terminator, and the outermost program must also have one.

### Structured Top-Down Design

Production COBOL programs typically follow a structured top-down design with
numbered paragraphs:

```
0000-MAIN             (control paragraph — orchestrates flow)
  1000-INIT           (open files, initialize variables)
  2000-PROCESS        (main processing loop body)
    2100-VALIDATE     (validate current record)
    2200-TRANSFORM    (transform data)
    2300-WRITE-OUTPUT (write to output)
  3000-TERMINATE      (close files, print totals)
  9000-READ-INPUT     (utility: read next record)
  9100-ABEND-ROUTINE  (utility: abnormal end handler)
```

The numbering scheme groups related paragraphs and makes the call hierarchy
immediately visible from the source listing.

### COPY and REPLACE for Shared Structures

Record layouts are frequently defined in copybooks and included via the COPY
statement:

```cobol
       FILE SECTION.
       FD  CUSTOMER-FILE.
       01  CUSTOMER-RECORD.
           COPY CUSTREC.
      *
       WORKING-STORAGE SECTION.
       01  WS-CUSTOMER-WORK.
           COPY CUSTREC REPLACING ==CUST-== BY ==WK-CUST-==.
```

This keeps the DATA DIVISION clean and ensures that all programs using the same
file share an identical record layout.

### Conditional Compilation with >>EVALUATE

Some compilers support conditional compilation directives:

```cobol
      >>DEFINE DEBUG-MODE
      >>IF DEBUG-MODE IS DEFINED
       01  WS-DEBUG-FLAG  PIC X VALUE 'Y'.
      >>ELSE
       01  WS-DEBUG-FLAG  PIC X VALUE 'N'.
      >>END-IF
```

These directives are processed before compilation and can appear in any division.

## Gotchas

- **Code in the wrong area causes cryptic compilation errors.** If a level-01 item
  starts in Area B (column 12+) instead of Area A (column 8-11), most compilers
  will either reject it or silently misinterpret the line. The error messages rarely
  say "wrong column" — they say things like "unexpected token" or "invalid level
  number."

- **A missing period terminates more than you think.** In fixed-format COBOL, the
  period terminates a sentence. An `IF` statement without an `END-IF` is terminated
  by the next period, which may be far away. This is the single most common cause
  of logic errors in COBOL. Always use explicit scope terminators (`END-IF`,
  `END-PERFORM`, `END-READ`, etc.) and place periods only at the end of paragraphs.

- **Division/section order is rigid.** If you accidentally code the DATA DIVISION
  before the ENVIRONMENT DIVISION, the compiler will not helpfully say "wrong
  order." It will emit a cascade of confusing errors about undeclared items and
  invalid syntax.

- **WORKING-STORAGE persists between calls but LOCAL-STORAGE does not.** If your
  subprogram assumes variables are reset to VALUE clauses on each call, you must
  use LOCAL-STORAGE or add explicit initialization logic in WORKING-STORAGE. Failing
  to do so causes hard-to-reproduce bugs that only appear on the second and
  subsequent calls.

- **Multiple 01-levels under one FD share the same buffer.** This is an implicit
  REDEFINES. Writing to one record description and then reading from another under
  the same FD will reflect the same physical bytes. This is by design but surprises
  newcomers.

- **STOP RUN in a subprogram terminates the entire run unit.** Use `GOBACK` in
  subprograms to return control to the caller. See
  [paragraph_flow.md](paragraph_flow.md) for the full comparison of STOP RUN vs
  GOBACK.

- **Sequence number area (columns 1-6) is not validated as numeric.** You can put
  anything there (including letters), and the compiler ignores it. But some editors
  and source management tools rely on it being numeric. Do not put code there.

- **Free-format programs mixed with fixed-format copybooks.** If your program is in
  free format but it copies a copybook written in fixed format, the compiler may
  misparse the copybook. Some compilers support `>>SOURCE FORMAT IS FIXED` and
  `>>SOURCE FORMAT IS FREE` to switch mid-file; others do not. Verify your
  compiler's behavior.

- **CONFIGURATION SECTION entries are largely ignored.** Modern compilers ignore
  `SOURCE-COMPUTER` and `OBJECT-COMPUTER` entries — they exist for backward
  compatibility. Do not rely on them for any functional behavior.

- **The PROCEDURE DIVISION RETURNING clause and GOBACK interaction.** When using
  `RETURNING`, the returned data item must be set before `GOBACK` executes. If
  `GOBACK` is reached before the item is populated, the caller receives whatever
  happened to be in that memory location.

- **Continuation lines (hyphen in column 7) are fragile.** A continued non-numeric
  literal must end in column 72 on the first line, and the continuation line must
  have a hyphen in column 7 with the literal resuming (with an opening quote) in
  Area B. Off-by-one column errors silently corrupt the literal value.

- **Tabs in fixed-format source.** The COBOL standard does not define tab behavior.
  Different compilers expand tabs differently (some to column 8, some to the next
  multiple of 4 or 8). Using tabs in fixed-format source is a portability hazard.
  Use spaces only.

## Related Topics

- **[working_storage.md](working_storage.md)** — Detailed coverage of
  WORKING-STORAGE data definitions, VALUE clauses, level numbers, PICTURE strings,
  and USAGE clauses. The DATA DIVISION section above provides the structural
  overview; working_storage.md goes deep on the data-description entries themselves.

- **[paragraph_flow.md](paragraph_flow.md)** — Covers PERFORM logic, paragraph and
  section execution flow, GO TO, fall-through behavior, and structured programming
  patterns. The PROCEDURE DIVISION section above introduces paragraphs and sections;
  paragraph_flow.md covers how control moves between them.

- **[copybooks.md](copybooks.md)** — Covers the COPY statement, REPLACING clause,
  copybook search paths, and nested COPY. Relevant because copybooks are a
  fundamental part of how the DATA DIVISION (and sometimes the PROCEDURE DIVISION)
  is organized in real systems.

- **[subprograms.md](subprograms.md)** — Covers CALL, CANCEL, GOBACK, LINKAGE
  SECTION usage patterns, static vs dynamic calls, and nested programs. Directly
  related because the LINKAGE SECTION and PROCEDURE DIVISION USING/RETURNING
  clauses described here are the structural underpinning of subprogram
  communication.

- **[file_handling.md](file_handling.md)** — Covers OPEN, CLOSE, READ, WRITE,
  REWRITE, DELETE, START, and file status codes. Related because the ENVIRONMENT
  DIVISION's INPUT-OUTPUT SECTION and the DATA DIVISION's FILE SECTION define the
  file infrastructure that file_handling.md's operations act upon.
