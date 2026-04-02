# Working Storage

## Description
The WORKING-STORAGE SECTION is the primary area in a COBOL program's DATA DIVISION where programmers define variables, constants, accumulators, flags, record layouts, and other data items that persist for the life of the program (or the life of a run unit in called subprograms). This file covers level numbers, the VALUE clause, REDEFINES, group versus elementary items, FILLER, RENAMES (66 level), condition names (88 level), alignment and SYNC, JUSTIFIED RIGHT, BLANK WHEN ZERO, and organizational best practices.

## Table of Contents
- [Core Concepts](#core-concepts)
  - [Purpose of WORKING-STORAGE](#purpose-of-working-storage)
  - [Level Numbers](#level-numbers)
  - [Group Items vs Elementary Items](#group-items-vs-elementary-items)
  - [FILLER](#filler)
  - [The VALUE Clause](#the-value-clause)
  - [REDEFINES](#redefines)
  - [RENAMES (Level 66)](#renames-level-66)
  - [Condition Names (Level 88)](#condition-names-level-88)
  - [SYNC (SYNCHRONIZED)](#sync-synchronized)
  - [JUSTIFIED RIGHT](#justified-right)
  - [BLANK WHEN ZERO](#blank-when-zero)
- [Syntax & Examples](#syntax--examples)
- [Common Patterns](#common-patterns)
- [Gotchas](#gotchas)
- [Related Topics](#related-topics)

## Core Concepts

### Purpose of WORKING-STORAGE

WORKING-STORAGE SECTION appears within the DATA DIVISION, after the FILE SECTION (if present) and before the LOCAL-STORAGE SECTION (if present). Its entries define data items that are:

1. **Allocated once** when the program is loaded (for main programs) or when first called (for subprograms, depending on compiler behavior and INITIAL clause).
2. **Persistent** across PERFORM iterations and, for subprograms without the INITIAL clause, across multiple CALLs.
3. **Initialized** according to VALUE clauses at program load time (main programs) or first invocation. Without a VALUE clause, initial content is implementation-defined -- most mainframe compilers initialize to LOW-VALUES (binary zeros) but this is not guaranteed by the standard.

WORKING-STORAGE is where the vast majority of a program's non-file data lives: counters, switches, intermediate calculation fields, formatted output lines, constants, copybook-included record layouts, and parameter areas.

### Level Numbers

Level numbers define the hierarchy of data items. COBOL uses specific level numbers with distinct meanings:

**Regular hierarchy levels: 01 through 49**

- **01** -- Defines a record-level item. Every independent data structure in WORKING-STORAGE begins at level 01. An 01-level item can be either a group item (containing subordinate items) or an elementary item (with a PICTURE clause).
- **02 through 49** -- Define subordinate items within a group. The actual number chosen indicates relative depth: a higher number is subordinate to the nearest preceding lower number. For example, 05 is subordinate to 01, and 10 is subordinate to 05. By universal convention, programmers use increments of 5 (01, 05, 10, 15, 20...) to allow insertion of intermediate levels without renumbering. Any number from 02 through 49 is valid; the compiler treats them purely by relative magnitude, not absolute value.

**Special level numbers:**

- **66** -- Used exclusively with the RENAMES clause. Provides an alternative grouping of elementary items within a record. See the RENAMES section below.
- **77** -- Defines an independent elementary item that is not part of any group. A 77-level item cannot have subordinate items and cannot itself be subordinate to another item. Functionally equivalent to a standalone 01-level elementary item. Many shops discourage 77-level items in favor of organizing all data under 01-level groups.
- **88** -- Defines a condition name. It does not allocate storage; instead, it associates a meaningful name with one or more values of the immediately preceding elementary item. Used in IF statements and EVALUATE for readable conditional logic.

### Group Items vs Elementary Items

This distinction is fundamental to understanding COBOL data:

**Elementary items** have a PICTURE (PIC) clause and represent a single data field. They hold actual data and have a defined type (alphanumeric, numeric, etc.).

```cobol
       01  WS-CUSTOMER-NAME       PIC X(30).
       01  WS-AMOUNT              PIC S9(7)V99.
```

**Group items** do NOT have a PICTURE clause. They are composed of subordinate items and represent the concatenation of all their subordinates. When referenced as a whole, a group item is treated as an alphanumeric field regardless of what its subordinates contain.

```cobol
       01  WS-DATE-FIELDS.
           05  WS-YEAR            PIC 9(4).
           05  WS-MONTH           PIC 9(2).
           05  WS-DAY             PIC 9(2).
```

Here, `WS-DATE-FIELDS` is a group item occupying 8 bytes. Moving `'20260115'` to `WS-DATE-FIELDS` populates all three subordinate fields. Referencing `WS-DATE-FIELDS` in a MOVE or comparison treats it as PIC X(8).

Key rules:
- A group item's size is the sum of its subordinate items' sizes (adjusted for SYNC alignment if applicable).
- Moving a value to a group item is an alphanumeric (left-justified, space-padded) move, even if subordinates are numeric.
- A group item can contain other group items (nested groups).
- You cannot specify PICTURE, USAGE (other than on subordinates), JUSTIFIED, or BLANK WHEN ZERO on a group item directly.

### FILLER

FILLER is a reserved word used as a data-name for items that do not need to be individually referenced. Common uses:

- **Padding or spacing** in record layouts.
- **Literal portions** of formatted output lines.
- **Unused bytes** in a copybook-defined record.

```cobol
       01  WS-REPORT-HEADER.
           05  FILLER             PIC X(10) VALUE SPACES.
           05  WS-TITLE           PIC X(20) VALUE 'MONTHLY REPORT'.
           05  FILLER             PIC X(10) VALUE SPACES.
           05  WS-DATE-HDR        PIC X(10).
```

Multiple FILLER items can exist in the same record -- they do not conflict because FILLER cannot be referenced by name in PROCEDURE DIVISION statements (with the exception that some modern compilers allow referencing FILLER via qualification, but this is non-standard).

In the COBOL-85 standard and later, the word FILLER is optional. An unnamed elementary item is implicitly FILLER:

```cobol
           05                     PIC X(10) VALUE SPACES.
```

### The VALUE Clause

The VALUE clause assigns an initial value to a data item at program load time.

**For elementary items:**

```cobol
       01  WS-COUNTER             PIC 9(5)  VALUE ZEROS.
       01  WS-TAX-RATE            PIC V9(4) VALUE 0.0825.
       01  WS-STATUS-MSG          PIC X(20) VALUE 'INITIALIZED'.
       01  WS-SWITCH              PIC X     VALUE 'N'.
```

**For group items:** VALUE on a group item initializes the entire group as an alphanumeric literal, overriding any VALUE clauses on subordinate items.

```cobol
       01  WS-FULL-DATE           VALUE '20260101'.
           05  WS-YEAR            PIC 9(4).
           05  WS-MONTH           PIC 9(2).
           05  WS-DAY             PIC 9(2).
```

**Figurative constants** commonly used in VALUE clauses:
- `SPACES` (or `SPACE`) -- fills with spaces (X'40' on EBCDIC, X'20' on ASCII).
- `ZEROS` (or `ZERO`, `ZEROES`) -- fills with character zeros for alphanumeric, numeric zero for numeric items.
- `LOW-VALUES` (or `LOW-VALUE`) -- fills with binary zeros (X'00').
- `HIGH-VALUES` (or `HIGH-VALUE`) -- fills with X'FF'.
- `ALL literal` -- fills by repeating the literal (e.g., `ALL '*'`).

**Rules and restrictions:**
- VALUE is not permitted on items with REDEFINES (the redefined item's VALUE takes effect for that storage).
- For items in the FILE SECTION, VALUE is only allowed on 88-level condition names.
- Numeric VALUE literals must be within the range of the item's PICTURE.

### REDEFINES

REDEFINES allows two or more data descriptions to share the same storage area. The redefined and redefining items occupy the same physical bytes in memory.

```cobol
       01  WS-DATE-NUMERIC        PIC 9(8).
       01  WS-DATE-PARTS REDEFINES WS-DATE-NUMERIC.
           05  WS-YEAR            PIC 9(4).
           05  WS-MONTH           PIC 9(2).
           05  WS-DAY             PIC 9(2).
```

**Rules:**
- The REDEFINES clause must immediately follow the data-name and level number.
- Both items must be at the same level number.
- The redefining item must not be larger than the redefined item (though it may be smaller).
- The redefined item must be defined before the redefining item, with no intervening items at a higher (more senior) level.
- VALUE clauses are not permitted on items that contain a REDEFINES (or subordinates of a REDEFINES item, in strict standard compliance; many compilers relax this).
- REDEFINES cannot be specified on 01-level items in the FILE SECTION (implicit redefinition is used there).
- OCCURS and REDEFINES cannot both appear on the same data item, but a REDEFINES item can contain subordinates with OCCURS.
- Multiple items can redefine the same original item.

**Common uses:**
- Viewing a numeric date as both a single number and individual components.
- Interpreting a record area as different record types.
- Overlaying a group with an alphanumeric field for easy initialization.

```cobol
       01  WS-WORK-AREA.
           05  WS-FIELD-A         PIC X(50).
           05  WS-FIELD-B         PIC X(50).
       01  WS-WORK-AREA-INIT REDEFINES WS-WORK-AREA
                                   PIC X(100).
```

This lets you `MOVE SPACES TO WS-WORK-AREA-INIT` to clear the entire area in one statement.

### RENAMES (Level 66)

The RENAMES clause, used only at level 66, provides an alternative grouping of contiguous elementary items within a record. It creates an alias that spans from one item THROUGH another.

```cobol
       01  WS-EMPLOYEE-RECORD.
           05  WS-LAST-NAME       PIC X(20).
           05  WS-FIRST-NAME      PIC X(15).
           05  WS-MIDDLE-INIT     PIC X(1).
           05  WS-DEPARTMENT      PIC X(10).
           05  WS-SALARY          PIC 9(7)V99.
       66  WS-FULL-NAME RENAMES WS-LAST-NAME
                                THRU WS-MIDDLE-INIT.
       66  WS-DEPT-SALARY RENAMES WS-DEPARTMENT
                                  THRU WS-SALARY.
```

**Rules:**
- Level 66 items must immediately follow the last data description in the record they rename (before any other 01-level item).
- RENAMES can span a single item (`RENAMES item-a`) or a range (`RENAMES item-a THRU item-b`).
- The items renamed must be contiguous in storage.
- Cannot rename items that contain OCCURS (or are subordinate to OCCURS), level 66, level 77, or level 88 items.
- RENAMES does not allocate new storage; it is purely an overlay reference.

RENAMES is rarely used in modern COBOL. Most programmers achieve the same effect with REDEFINES or reference modification, which are more intuitive. However, RENAMES remains valid syntax and appears in legacy codebases.

### Condition Names (Level 88)

Level 88 items define condition names -- named boolean conditions attached to the preceding elementary item. They do not allocate storage.

```cobol
       01  WS-ACCOUNT-TYPE        PIC X(1).
           88  SAVINGS             VALUE 'S'.
           88  CHECKING            VALUE 'C'.
           88  MONEY-MARKET        VALUE 'M'.
           88  VALID-ACCOUNT-TYPE  VALUE 'S' 'C' 'M'.
```

**Syntax variations:**

```cobol
       01  WS-TEMPERATURE         PIC S9(3).
           88  FREEZING            VALUE 32.
           88  BOILING             VALUE 212.
           88  COMFORTABLE         VALUE 65 THRU 78.
           88  EXTREME-COLD        VALUE -50 THRU 0.
```

- A single value: `VALUE literal`.
- Multiple discrete values: `VALUE literal-1 literal-2 literal-3`.
- A range: `VALUE literal-1 THRU literal-2`.
- Combinations: `VALUE literal-1 THRU literal-2, literal-3, literal-4 THRU literal-5`.

**Using condition names in PROCEDURE DIVISION:**

```cobol
           IF SAVINGS
               PERFORM PROCESS-SAVINGS
           END-IF

           EVALUATE TRUE
               WHEN SAVINGS
                   PERFORM PROCESS-SAVINGS
               WHEN CHECKING
                   PERFORM PROCESS-CHECKING
               WHEN OTHER
                   PERFORM PROCESS-DEFAULT
           END-EVALUATE
```

**SET statement with 88-levels:**

```cobol
           SET SAVINGS TO TRUE
```

This moves 'S' into WS-ACCOUNT-TYPE. When multiple values are defined on the 88-level, SET TO TRUE moves the first value listed.

Condition names are one of the most powerful readability features in COBOL. They eliminate magic literals from PROCEDURE DIVISION code and make maintenance vastly easier.

### SYNC (SYNCHRONIZED)

The SYNCHRONIZED (or SYNC) clause aligns binary (COMP/COMP-4) and internal floating-point (COMP-1/COMP-2) items on their natural storage boundaries (halfword, fullword, or doubleword).

```cobol
       01  WS-RECORD.
           05  WS-NAME            PIC X(15).
           05  WS-BINARY-FIELD    PIC S9(8) COMP SYNC.
```

**LEFT and RIGHT options:**
- `SYNC LEFT` -- aligns on the left boundary of the natural boundary (rarely used).
- `SYNC RIGHT` -- aligns on the right boundary (rarely used).
- `SYNC` alone (most common) -- compiler determines optimal alignment.

**How it works:**
- The compiler inserts implicit FILLER bytes (slack bytes) before the SYNC item to achieve proper alignment.
- On IBM mainframes: halfword items (PIC S9(1-4) COMP) align on 2-byte boundaries; fullword items (PIC S9(5-9) COMP) align on 4-byte boundaries; doubleword items (PIC S9(10-18) COMP) align on 8-byte boundaries.
- The slack bytes increase the group item's total size, which can cause surprises when computing record lengths.

**When to use SYNC:**
- When performance of binary arithmetic is critical and the field will be used in heavy computation.
- In most business COBOL, SYNC is unnecessary because packed decimal (COMP-3) is more common and does not require alignment.
- Avoid SYNC on fields that are part of file records or communication areas where exact byte layouts matter.

### JUSTIFIED RIGHT

The JUSTIFIED RIGHT (or JUST RIGHT) clause causes alphanumeric data to be right-justified (with left space-padding) on receiving MOVEs, instead of the default left-justification.

```cobol
       01  WS-RIGHT-NAME          PIC X(20) JUSTIFIED RIGHT.
```

If you move `'SMITH'` to this field, the result is 15 spaces followed by `SMITH`.

**Rules:**
- Only valid on elementary alphanumeric (PIC X) or alphabetic (PIC A) items.
- Not valid on numeric items (they have their own alignment rules based on PICTURE).
- Not valid on group items.
- Does not affect DISPLAY output alignment -- only internal storage.
- Rarely used in practice; most right-alignment needs are handled by PICTURE editing or STRING/UNSTRING.

### BLANK WHEN ZERO

The BLANK WHEN ZERO clause causes a numeric or numeric-edited item to display as all spaces when its value is zero.

```cobol
       01  WS-AMOUNT-DISPLAY      PIC Z,ZZZ,ZZ9.99
                                   BLANK WHEN ZERO.
```

Without BLANK WHEN ZERO, a zero value displays as `        0.00`. With it, the entire field becomes spaces.

**Rules:**
- Valid on elementary numeric and numeric-edited items.
- Not valid on group items.
- Particularly useful in report generation where blank fields are preferred over rows of zeros.
- Cannot be used with items that have an asterisk (*) check-protect symbol in the PICTURE.

## Syntax & Examples

### Complete WORKING-STORAGE SECTION Example

```cobol
       DATA DIVISION.
       WORKING-STORAGE SECTION.

      *---------------------------------------------------------*
      * PROGRAM CONSTANTS
      *---------------------------------------------------------*
       01  WS-CONSTANTS.
           05  WS-PROGRAM-NAME    PIC X(8)  VALUE 'CUSTPROC'.
           05  WS-MAX-RECORDS     PIC 9(5)  VALUE 10000.
           05  WS-TAX-RATE        PIC V9(4) VALUE 0.0725.
           05  WS-COMMA           PIC X     VALUE ','.
           05  WS-SPACE           PIC X     VALUE SPACE.

      *---------------------------------------------------------*
      * PROGRAM SWITCHES AND FLAGS
      *---------------------------------------------------------*
       01  WS-SWITCHES.
           05  WS-EOF-FLAG        PIC X     VALUE 'N'.
               88  EOF-REACHED               VALUE 'Y'.
               88  NOT-EOF                   VALUE 'N'.
           05  WS-VALID-FLAG      PIC X     VALUE 'Y'.
               88  RECORD-VALID              VALUE 'Y'.
               88  RECORD-INVALID            VALUE 'N'.
           05  WS-PROCESS-MODE    PIC X     VALUE 'A'.
               88  MODE-ADD                  VALUE 'A'.
               88  MODE-UPDATE               VALUE 'U'.
               88  MODE-DELETE               VALUE 'D'.
               88  VALID-MODE                VALUE 'A' 'U' 'D'.

      *---------------------------------------------------------*
      * COUNTERS AND ACCUMULATORS
      *---------------------------------------------------------*
       01  WS-COUNTERS.
           05  WS-RECORDS-READ    PIC 9(7)  VALUE ZEROS.
           05  WS-RECORDS-WRITTEN PIC 9(7)  VALUE ZEROS.
           05  WS-ERROR-COUNT     PIC 9(5)  VALUE ZEROS.
           05  WS-TOTAL-AMOUNT    PIC S9(11)V99 VALUE ZEROS.

      *---------------------------------------------------------*
      * DATE WORK AREAS
      *---------------------------------------------------------*
       01  WS-CURRENT-DATE-DATA.
           05  WS-CURRENT-DATE.
               10  WS-CURRENT-YEAR
                                   PIC 9(4).
               10  WS-CURRENT-MONTH
                                   PIC 9(2).
               10  WS-CURRENT-DAY  PIC 9(2).
           05  WS-CURRENT-TIME.
               10  WS-CURRENT-HOUR PIC 9(2).
               10  WS-CURRENT-MIN  PIC 9(2).
               10  WS-CURRENT-SEC  PIC 9(2).
               10  WS-CURRENT-HUND PIC 9(2).
           05  WS-GMT-OFFSET.
               10  WS-GMT-SIGN    PIC X.
               10  WS-GMT-HOURS   PIC 9(2).
               10  WS-GMT-MINUTES PIC 9(2).

      *---------------------------------------------------------*
      * REDEFINES EXAMPLE - RECORD TYPE HANDLING
      *---------------------------------------------------------*
       01  WS-INPUT-RECORD        PIC X(100).
       01  WS-HEADER-REC REDEFINES WS-INPUT-RECORD.
           05  WS-HDR-TYPE        PIC X(2).
           05  WS-HDR-DATE        PIC 9(8).
           05  WS-HDR-DESC        PIC X(40).
           05  FILLER             PIC X(50).
       01  WS-DETAIL-REC REDEFINES WS-INPUT-RECORD.
           05  WS-DTL-TYPE        PIC X(2).
           05  WS-DTL-ACCT        PIC X(10).
           05  WS-DTL-NAME        PIC X(30).
           05  WS-DTL-AMOUNT      PIC S9(9)V99.
           05  FILLER             PIC X(47).

      *---------------------------------------------------------*
      * REPORT LINE LAYOUT
      *---------------------------------------------------------*
       01  WS-DETAIL-LINE.
           05  FILLER             PIC X(5)  VALUE SPACES.
           05  WS-DL-ACCT         PIC X(10).
           05  FILLER             PIC X(3)  VALUE SPACES.
           05  WS-DL-NAME         PIC X(30).
           05  FILLER             PIC X(3)  VALUE SPACES.
           05  WS-DL-AMOUNT       PIC Z,ZZZ,ZZ9.99-.
           05  FILLER             PIC X(3)  VALUE SPACES.
           05  WS-DL-STATUS       PIC X(10).

      *---------------------------------------------------------*
      * LEVEL 66 RENAMES EXAMPLE
      *---------------------------------------------------------*
       01  WS-EMPLOYEE-DATA.
           05  WS-EMP-LAST        PIC X(20).
           05  WS-EMP-FIRST       PIC X(15).
           05  WS-EMP-MI          PIC X(1).
           05  WS-EMP-DEPT        PIC X(4).
           05  WS-EMP-SALARY      PIC 9(7)V99.
           05  WS-EMP-HIRE-DATE   PIC 9(8).
       66  WS-EMP-FULL-NAME RENAMES WS-EMP-LAST
                                  THRU WS-EMP-MI.
       66  WS-EMP-COMPENSATION RENAMES WS-EMP-DEPT
                                  THRU WS-EMP-SALARY.
```

### Initializing Complex Structures

```cobol
      * Method 1: VALUE on group item (alphanumeric overlay)
       01  WS-INIT-EXAMPLE        VALUE SPACES.
           05  WS-FIELD-A         PIC X(10).
           05  WS-FIELD-B         PIC X(10).
           05  WS-FIELD-C         PIC X(10).

      * Method 2: INITIALIZE statement in PROCEDURE DIVISION
      *   INITIALIZE WS-CUSTOMER-REC
      *     -- sets alphanumeric fields to SPACES
      *     -- sets numeric fields to ZEROS

      * Method 3: REDEFINES for bulk initialization
       01  WS-WORK-FIELDS.
           05  WS-NAME            PIC X(30).
           05  WS-AMOUNT          PIC S9(7)V99 COMP-3.
           05  WS-CODE            PIC 9(3).
       01  WS-WORK-INIT REDEFINES WS-WORK-FIELDS
                                   PIC X(38).
      * Then: MOVE SPACES TO WS-WORK-INIT  (clears everything)
      * Warning: this also space-fills numeric fields, which may
      *          cause abends if used in arithmetic before re-init.
```

### Condition Name Patterns

```cobol
       01  WS-FILE-STATUS         PIC X(2).
           88  FS-SUCCESS                    VALUE '00'.
           88  FS-END-OF-FILE               VALUE '10'.
           88  FS-RECORD-NOT-FOUND          VALUE '23'.
           88  FS-DUPLICATE-KEY             VALUE '22'.
           88  FS-SUCCESSFUL      VALUE '00' '02' '04'.

       PROCEDURE DIVISION.
           READ INPUT-FILE
               AT END SET EOF-REACHED TO TRUE
           END-READ
           IF NOT FS-SUCCESS
               DISPLAY 'READ ERROR: ' WS-FILE-STATUS
               PERFORM ERROR-HANDLER
           END-IF
```

## Common Patterns

### Standard Program Data Areas

Most production COBOL programs organize WORKING-STORAGE into well-defined sections. A commonly seen layout:

1. **Constants** -- program name, fixed values, literal strings.
2. **Switches and flags** -- with 88-level condition names.
3. **Counters and accumulators** -- record counts, running totals.
4. **Date/time work areas** -- for ACCEPT FROM DATE / FUNCTION CURRENT-DATE.
5. **File status fields** -- one per file, with 88-levels for common statuses.
6. **Work areas** -- intermediate computation fields, temporary variables.
7. **Record layouts** -- either coded inline or brought in via COPY.
8. **Report lines** -- formatted output lines with FILLER for spacing and VALUE for literals.
9. **Communication areas** -- COMMAREA, parameter blocks for CALL.

### Prefix Naming Conventions

Most shops enforce naming prefixes for readability and to avoid conflicts with COBOL reserved words:

```cobol
       01  WS-CUSTOMER-NAME       PIC X(30).
       01  WS-AMOUNT-DUE          PIC S9(7)V99.
```

Common prefix schemes:
- `WS-` for WORKING-STORAGE items.
- `LS-` for LOCAL-STORAGE items.
- `LK-` for LINKAGE SECTION items.
- `FS-` or `WS-FS-` for file status fields.
- `SW-` for switches.
- `CT-` or `WS-CT-` for counters.
- `WK-` for work fields.

### Multi-Record File Processing with REDEFINES

```cobol
       01  WS-RECORD-TYPE         PIC X(2).
       01  WS-RAW-RECORD          PIC X(200).
       01  WS-CUSTOMER-REC REDEFINES WS-RAW-RECORD.
           05  WS-CUST-TYPE       PIC X(2).
           05  WS-CUST-ID         PIC X(10).
           05  WS-CUST-NAME       PIC X(40).
           05  FILLER             PIC X(148).
       01  WS-ORDER-REC REDEFINES WS-RAW-RECORD.
           05  WS-ORD-TYPE        PIC X(2).
           05  WS-ORD-NUMBER      PIC X(8).
           05  WS-ORD-DATE        PIC 9(8).
           05  WS-ORD-AMOUNT      PIC S9(9)V99.
           05  FILLER             PIC X(171).

       PROCEDURE DIVISION.
           MOVE WS-RAW-RECORD(1:2) TO WS-RECORD-TYPE
           EVALUATE WS-RECORD-TYPE
               WHEN 'CU'
                   PERFORM PROCESS-CUSTOMER
               WHEN 'OR'
                   PERFORM PROCESS-ORDER
               WHEN OTHER
                   PERFORM HANDLE-UNKNOWN-TYPE
           END-EVALUATE
```

### Return Code and Status Block

```cobol
       01  WS-RETURN-AREA.
           05  WS-RETURN-CODE     PIC S9(4) COMP VALUE 0.
               88  RC-SUCCESS                VALUE 0.
               88  RC-WARNING                VALUE 4.
               88  RC-ERROR                  VALUE 8.
               88  RC-SEVERE                 VALUE 12.
               88  RC-FATAL                  VALUE 16.
           05  WS-RETURN-MSG      PIC X(80) VALUE SPACES.
```

### Formatted Report Lines with FILLER

```cobol
       01  WS-HEADING-LINE-1.
           05  FILLER             PIC X(1)  VALUE SPACES.
           05  FILLER             PIC X(15) VALUE 'ACCOUNT NUMBER'.
           05  FILLER             PIC X(5)  VALUE SPACES.
           05  FILLER             PIC X(20) VALUE 'CUSTOMER NAME'.
           05  FILLER             PIC X(5)  VALUE SPACES.
           05  FILLER             PIC X(12) VALUE 'BALANCE'.
           05  FILLER             PIC X(5)  VALUE SPACES.
           05  FILLER             PIC X(10) VALUE 'STATUS'.

       01  WS-TOTAL-LINE.
           05  FILLER             PIC X(41) VALUE SPACES.
           05  FILLER             PIC X(10) VALUE 'TOTAL:'.
           05  WS-TL-TOTAL        PIC $$$,$$$,$$9.99-.
           05  FILLER             PIC X(15) VALUE SPACES.
```

### Communication Area for Subprogram Calls

```cobol
       01  WS-CALL-PARMS.
           05  WS-PARM-FUNCTION   PIC X(4).
               88  PARM-INQUIRY              VALUE 'INQR'.
               88  PARM-UPDATE               VALUE 'UPDT'.
               88  PARM-DELETE               VALUE 'DELT'.
           05  WS-PARM-ACCT-NUM   PIC X(10).
           05  WS-PARM-RETURN-CD  PIC S9(4) COMP.
           05  WS-PARM-DATA       PIC X(200).
```

### Alphanumeric-Numeric Overlay Pattern

A common technique for validating or manipulating numeric data stored as alphanumeric:

```cobol
       01  WS-NUMERIC-CHECK.
           05  WS-NUM-ALPHA       PIC X(10).
           05  WS-NUM-VALUE REDEFINES WS-NUM-ALPHA
                                   PIC 9(10).
```

This allows you to receive data into the alphanumeric field, validate it (using INSPECT, class tests, etc.), and then reference it through the numeric redefinition for arithmetic -- though caution is required since non-numeric data in the numeric view will cause errors.

## Gotchas

- **Uninitialized data causes unpredictable behavior.** If you omit the VALUE clause, the content of a WORKING-STORAGE item is implementation-defined. On IBM mainframes, it is typically binary zeros (LOW-VALUES), but on other platforms it may be garbage. Always initialize critical fields explicitly, especially numeric fields used in arithmetic and flags used in conditionals.

- **GROUP moves are always alphanumeric.** Moving data to a group item performs an alphanumeric MOVE (left-justified, space-padded, no numeric conversion), even if all subordinates are numeric. This can corrupt packed decimal (COMP-3) or binary (COMP) subordinate fields. Use INITIALIZE or move to individual fields instead.

- **REDEFINES with VALUE is restricted.** The COBOL standard prohibits VALUE clauses on items that contain a REDEFINES or on subordinates of a redefining item. While some compilers tolerate this, relying on it creates portability issues. Initialize via the original (redefined) item instead.

- **SYNC inserts invisible slack bytes.** When SYNC is used on a binary field within a group, the compiler inserts padding bytes to achieve alignment. These slack bytes increase the group's total size but are invisible in the source code. This commonly breaks record length calculations, file layouts, and CALL parameter blocks. Always verify group sizes with compiler listings when using SYNC.

- **REDEFINES size mismatch causes silent data corruption.** If the redefining item is smaller than the redefined item, only the first portion of storage is accessed -- this is legal but can mask bugs. If you inadvertently access the redefining item expecting full coverage, trailing bytes are missed.

- **Moving SPACES to a REDEFINES overlay clears numeric fields to spaces.** After `MOVE SPACES TO WS-WORK-INIT`, any COMP-3 or COMP fields under the original definition now contain spaces (not valid numeric data). Subsequent arithmetic on these fields will cause a data exception abend (S0C7 on IBM mainframes). Always reinitialize numeric fields after a bulk space-fill.

- **88-level SET TO TRUE uses the first value.** When an 88-level has multiple values (e.g., `VALUE 'A' 'B' 'C'`), `SET condition-name TO TRUE` moves only the first value ('A'). If you need a specific value, use an explicit MOVE.

- **FILLER items cannot be referenced.** Despite some compiler extensions, the COBOL standard does not allow referencing FILLER items by name. If you later need to access a FILLER field, you must rename it -- which changes the copybook signature.

- **Level 77 prevents future grouping.** A 77-level item can never have subordinates. If requirements change and you need to add sub-fields, you must restructure it as an 01-level group. This is why many coding standards prohibit 77-level items.

- **Condition names (88-levels) do not validate automatically.** Defining an 88-level does not prevent invalid values from being stored in the parent field. An 88-level only provides a test -- you must code explicit validation logic, for example: `IF NOT VALID-ACCOUNT-TYPE PERFORM ERROR-HANDLER`.

- **JUSTIFIED RIGHT is rarely what you want.** It applies only to receiving MOVEs into the field. It does not affect display formatting, does not work with numeric data, and can cause confusion when concatenating or comparing strings. Most right-alignment needs are better handled by PICTURE editing or STRING operations.

- **BLANK WHEN ZERO applies to the entire field.** When a numeric-edited item has BLANK WHEN ZERO and the value is zero, the entire field becomes spaces -- including currency signs, decimal points, and other edit characters. This may not be the desired report appearance (e.g., you might want `$0.00` to display).

- **RENAMES (66) must follow the last item in the record.** If you insert a new 05-level field at the end of a record, you must ensure that any 66-level RENAMES entries still appear after the last data description entry. Misordering causes compilation errors.

- **Group-level INITIALIZE skips FILLER.** The INITIALIZE statement does not set FILLER items. If FILLER items contain VALUES that were set at load time, INITIALIZE will not reset them. This is usually desirable but can surprise developers who expect a complete reset.

- **Copying structures with COMP fields across CALL boundaries.** If the calling program and called program define the same structure differently (different SYNC usage, different level structures), the offsets may not match. Always use shared copybooks for parameter layouts.

## Related Topics

- **data_types.md** -- Detailed coverage of PICTURE clauses, USAGE types (DISPLAY, COMP, COMP-3, etc.), and numeric versus alphanumeric data representation. WORKING-STORAGE items rely heavily on these concepts for their definitions.
- **cobol_structure.md** -- Explains the overall structure of a COBOL program including all four divisions. WORKING-STORAGE SECTION sits within the DATA DIVISION; understanding its placement in the program structure is essential.
- **copybooks.md** -- COPY statements are used extensively in WORKING-STORAGE to include shared record layouts, communication areas, and standard data definitions. Copybook management directly affects WORKING-STORAGE organization.
- **conditional_logic.md** -- Level 88 condition names defined in WORKING-STORAGE are consumed by IF, EVALUATE, and SET statements covered in conditional logic. The two topics are deeply interrelated.
- **table_handling.md** -- OCCURS clause (used for tables/arrays) is frequently defined on items within WORKING-STORAGE. Table definitions, subscripting, indexing, and SEARCH operations all operate on WORKING-STORAGE data structures.
