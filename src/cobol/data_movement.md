# Data Movement

## Description
This file covers the COBOL statements that transfer, initialize, and set data values: the MOVE statement (including CORRESPONDING), the INITIALIZE statement, and the SET statement. It explains the rules governing how data is converted, truncated, padded, and justified during movement between fields of different types and sizes. Reference this file when you need to understand what happens when data is moved from one field to another, how to bulk-initialize records, or when to use SET versus MOVE.

## Table of Contents
- [Core Concepts](#core-concepts)
  - [How Data Movement Works in COBOL](#how-data-movement-works-in-cobol)
  - [Categories of Data Items](#categories-of-data-items)
  - [Elementary Moves vs Group Moves](#elementary-moves-vs-group-moves)
  - [Permitted and Prohibited Moves](#permitted-and-prohibited-moves)
  - [Truncation and Padding Rules](#truncation-and-padding-rules)
  - [Numeric Move Rules](#numeric-move-rules)
  - [De-editing Moves](#de-editing-moves)
  - [Justification](#justification)
  - [Sign Handling During Moves](#sign-handling-during-moves)
- [Syntax & Examples](#syntax--examples)
  - [Basic MOVE Statement](#basic-move-statement)
  - [Alphanumeric Moves](#alphanumeric-moves)
  - [Numeric Moves](#numeric-moves)
  - [Group Moves](#group-moves)
  - [MOVE CORRESPONDING](#move-corresponding)
  - [INITIALIZE Statement](#initialize-statement)
  - [SET Statement](#set-statement)
- [Common Patterns](#common-patterns)
  - [Clearing a Record](#clearing-a-record)
  - [Initializing Mixed Records](#initializing-mixed-records)
  - [Setting Flags and Condition Names](#setting-flags-and-condition-names)
  - [Index Manipulation](#index-manipulation)
  - [Copying Between Similar Structures](#copying-between-similar-structures)
  - [MOVE vs INITIALIZE vs VALUE](#move-vs-initialize-vs-value)
- [Gotchas](#gotchas)
- [Related Topics](#related-topics)

## Core Concepts

### How Data Movement Works in COBOL

COBOL's data movement model is fundamentally different from assignment in most programming languages. When data moves from a sending field to a receiving field, the compiler applies conversion, alignment, truncation, and padding rules based on the PICTURE clauses and USAGE of both fields. The sending field is never modified.

A single MOVE statement can have multiple receiving fields:

```cobol
MOVE WS-VALUE TO WS-FIELD-A WS-FIELD-B WS-FIELD-C
```

Each receiving field is processed independently according to its own PICTURE and USAGE.

### Categories of Data Items

The rules governing data movement depend on the category of the sending and receiving items. COBOL defines these categories:

| Category               | PICTURE Characters      | Example PIC        |
|------------------------|-------------------------|--------------------|
| Alphabetic             | A                       | PIC A(20)          |
| Alphanumeric           | X                       | PIC X(30)          |
| Alphanumeric edited    | X with B, 0, /          | PIC X(5)BX(5)      |
| Numeric (display)      | 9, S, V, P              | PIC S9(5)V99       |
| Numeric (computational)| 9, S, V with USAGE COMP | PIC S9(7)V99 COMP-3|
| Numeric edited         | 9 with Z, *, $, etc.    | PIC $ZZ,ZZ9.99     |

Group items (items with subordinate items) are always treated as alphanumeric regardless of their subordinates' types.

### Elementary Moves vs Group Moves

This distinction is critical:

**Elementary moves** occur when both the sending and receiving items are elementary (have no subordinate items), or when a literal is moved to an elementary item. The compiler performs type conversion based on the PICTURE clauses of both items. Numeric alignment, decimal point alignment, sign handling, and editing all occur.

**Group moves** occur when either the sending or receiving item (or both) is a group item. In a group move, **no conversion takes place**. The sending item is treated as a simple alphanumeric string and is moved byte-for-byte into the receiving item, left-justified and space-padded or truncated on the right. This is true even if the group contains numeric subordinate items.

```cobol
*> Elementary move: decimal alignment occurs
MOVE WS-AMOUNT-5V2 TO WS-AMOUNT-7V3

*> Group move: raw byte copy, no alignment
MOVE WS-INPUT-RECORD TO WS-OUTPUT-RECORD
```

### Permitted and Prohibited Moves

Not all combinations of sending and receiving categories are legal. The key rules:

| Sending Category     | Can Move To                                         |
|----------------------|-----------------------------------------------------|
| Alphabetic           | Alphabetic, alphanumeric, alphanumeric edited        |
| Alphanumeric         | Any category (acts as universal sender)              |
| Alphanumeric edited  | Alphanumeric, alphanumeric edited                    |
| Numeric (integer)    | Numeric, numeric edited, alphanumeric (if integer)   |
| Numeric (non-integer)| Numeric, numeric edited                              |
| Numeric edited       | Alphanumeric, numeric (de-editing), numeric edited   |

An alphanumeric item is the most flexible sender because it can be moved to any category. However, moving an alphanumeric item to a numeric item only works correctly if the alphanumeric content is a valid numeric representation.

### Truncation and Padding Rules

**Alphanumeric moves (and group moves):**
- Data is placed left-justified in the receiving field.
- If the receiving field is longer, it is padded with spaces on the right.
- If the receiving field is shorter, data is truncated on the right.
- Exception: fields with JUSTIFIED RIGHT are right-justified and truncated/padded on the left.

**Numeric moves:**
- Data is aligned on the decimal point (explicit or implied).
- Integer digits are filled from the decimal point to the left. Excess high-order digits are truncated on the left (which can silently lose significant digits).
- Decimal digits are filled from the decimal point to the right. Excess low-order digits are truncated on the right.
- Unfilled positions are filled with zeros.

```cobol
01  WS-SOURCE     PIC 9(5)V99  VALUE 12345.67.
01  WS-TARGET     PIC 9(3)V9.

MOVE WS-SOURCE TO WS-TARGET
*> WS-TARGET contains 345.6
*> High-order 12 is lost, low-order 7 is lost
```

### Numeric Move Rules

When a numeric item is moved to another numeric item:

1. The decimal points are aligned (using the V position or actual decimal point).
2. The receiving field's integer portion is filled from right to left. Leading positions are zero-filled. If the source has more integer digits than the target, high-order digits are truncated.
3. The receiving field's decimal portion is filled from left to right. Trailing positions are zero-filled. If the source has more decimal digits, low-order digits are truncated.
4. The sign is transferred according to the receiving field's sign specification.
5. If the receiving field is unsigned (no S in PIC), the absolute value is stored.
6. USAGE conversion is performed automatically (e.g., DISPLAY to COMP-3, COMP to COMP-3).

```cobol
01  WS-DISPLAY-NUM   PIC S9(5)V99  VALUE -12345.67.
01  WS-PACKED-NUM    PIC S9(7)V999 COMP-3.

MOVE WS-DISPLAY-NUM TO WS-PACKED-NUM
*> WS-PACKED-NUM contains -0012345.670
*> Converted from DISPLAY to COMP-3
*> Leading zeros added, trailing zero added
*> Sign preserved
```

### De-editing Moves

When a numeric edited item is moved to a numeric item, the compiler reverses the editing to extract the numeric value. This is called de-editing.

```cobol
01  WS-EDITED      PIC $ZZ,ZZ9.99  VALUE "$  1,234.56".
01  WS-NUMERIC     PIC 9(5)V99.

MOVE WS-EDITED TO WS-NUMERIC
*> WS-NUMERIC contains 01234.56
*> Currency sign, commas, and spaces are stripped
```

De-editing extracts the digits, reconstructs the numeric value, and then applies normal numeric move rules. This feature is useful when processing data that was formatted for display but needs to be used in calculations.

### Justification

By default:
- Alphanumeric data is left-justified in the receiving field.
- Numeric data is right-justified (aligned on the decimal point).

The JUSTIFIED RIGHT (or JUST RIGHT) clause overrides the default for alphanumeric items:

```cobol
01  WS-RIGHT-FIELD  PIC X(10) JUSTIFIED RIGHT.

MOVE "HELLO" TO WS-RIGHT-FIELD
*> Contains "     HELLO" (5 leading spaces)
```

JUSTIFIED only applies to elementary alphanumeric and alphabetic items. It has no effect on numeric items (which are always decimal-aligned).

### Sign Handling During Moves

When a signed numeric item is moved to an unsigned numeric item, the sign is lost (absolute value is stored). When an unsigned item is moved to a signed item, the value is treated as positive.

```cobol
01  WS-SIGNED     PIC S9(5)  VALUE -12345.
01  WS-UNSIGNED   PIC 9(5).

MOVE WS-SIGNED TO WS-UNSIGNED
*> WS-UNSIGNED contains 12345 (sign stripped)

MOVE WS-UNSIGNED TO WS-SIGNED
*> WS-SIGNED contains +12345
```

The SIGN clause (LEADING, TRAILING, SEPARATE) affects the physical storage of the sign but does not change the move rules -- the compiler handles conversion automatically.

## Syntax & Examples

### Basic MOVE Statement

```cobol
MOVE identifier-1 TO identifier-2 [identifier-3 ...]
MOVE literal TO identifier-2 [identifier-3 ...]
MOVE figurative-constant TO identifier-2 [identifier-3 ...]
```

Figurative constants include SPACES, ZEROS, HIGH-VALUES, LOW-VALUES, QUOTES, and ALL literal:

```cobol
MOVE SPACES TO WS-NAME WS-ADDRESS WS-CITY
MOVE ZEROS  TO WS-AMOUNT WS-COUNTER WS-TOTAL
MOVE HIGH-VALUES TO WS-SEARCH-KEY
MOVE ALL "*" TO WS-SEPARATOR
```

### Alphanumeric Moves

```cobol
01  WS-SHORT   PIC X(5)    VALUE "HELLO".
01  WS-LONG    PIC X(10).
01  WS-TINY    PIC X(3).

MOVE WS-SHORT TO WS-LONG
*> WS-LONG = "HELLO     " (padded with spaces)

MOVE WS-SHORT TO WS-TINY
*> WS-TINY = "HEL" (truncated on right)

MOVE "GOODBYE" TO WS-SHORT
*> WS-SHORT = "GOODB" (truncated on right)
```

### Numeric Moves

```cobol
01  WS-SMALL    PIC 9(3)     VALUE 42.
01  WS-BIG      PIC 9(7)V99.
01  WS-PACKED   PIC S9(5)V99 COMP-3.

MOVE WS-SMALL TO WS-BIG
*> WS-BIG = 0000042.00 (zero-filled, decimal aligned)

MOVE 123.456 TO WS-PACKED
*> WS-PACKED = +00123.45 (low-order 6 truncated)

MOVE 99999999 TO WS-BIG
*> WS-BIG = 9999999.00 (fits exactly in integer portion)
```

### Group Moves

```cobol
01  WS-INPUT-REC.
    05  WS-IN-NAME    PIC X(20).
    05  WS-IN-AMT     PIC 9(5)V99.
    05  WS-IN-DATE    PIC 9(8).

01  WS-OUTPUT-REC.
    05  WS-OUT-NAME   PIC X(20).
    05  WS-OUT-AMT    PIC 9(5)V99.
    05  WS-OUT-DATE   PIC 9(8).

MOVE WS-INPUT-REC TO WS-OUTPUT-REC
*> Byte-for-byte copy, no conversion
*> Works correctly because the layouts match exactly
```

When layouts differ, group moves can produce unexpected results because no field-level conversion takes place:

```cobol
01  WS-REC-A.
    05  WS-A-CODE     PIC X(3).
    05  WS-A-AMOUNT   PIC 9(5)V99 COMP-3.  *> 4 bytes

01  WS-REC-B.
    05  WS-B-CODE     PIC X(3).
    05  WS-B-AMOUNT   PIC 9(5)V99.          *> 7 bytes

MOVE WS-REC-A TO WS-REC-B
*> DANGER: REC-A is 7 bytes, REC-B is 10 bytes
*> The COMP-3 bytes are copied raw into DISPLAY positions
*> WS-B-AMOUNT will contain garbage
```

### MOVE CORRESPONDING

MOVE CORRESPONDING (or MOVE CORR) moves data between two group items by matching subordinate item names. Only items with the same name at the elementary level are moved.

```cobol
01  WS-INPUT.
    05  CUST-NAME      PIC X(30).
    05  CUST-ID        PIC 9(8).
    05  CUST-BALANCE   PIC S9(7)V99.
    05  CUST-REGION    PIC X(2).

01  WS-OUTPUT.
    05  CUST-NAME      PIC X(30).
    05  CUST-ID        PIC 9(10).
    05  CUST-BALANCE   PIC S9(9)V99.
    05  ORDER-DATE     PIC 9(8).

MOVE CORRESPONDING WS-INPUT TO WS-OUTPUT
*> Moves CUST-NAME, CUST-ID, and CUST-BALANCE
*> CUST-REGION is not moved (no match in WS-OUTPUT)
*> ORDER-DATE is not touched (no match in WS-INPUT)
*> Each moved field uses elementary move rules (with conversion)
```

Rules for CORRESPONDING:
- Both operands must be group items.
- Only elementary items with matching names at corresponding levels are moved.
- FILLER items are never moved.
- REDEFINES items and items subordinate to REDEFINES are excluded.
- Each matched pair is moved as an elementary move (with full conversion).
- Items of the same name but at different levels within the hierarchy may or may not match, depending on the group structure.

### INITIALIZE Statement

INITIALIZE sets fields within a group or elementary item to default values based on their categories. It is more granular than `MOVE SPACES` or `MOVE ZEROS` because it applies the appropriate default to each subordinate field based on its type.

```cobol
INITIALIZE identifier-1
    [WITH FILLER]
    [ALL]
    [REPLACING
        {ALPHABETIC     } DATA BY literal-or-identifier
        {ALPHANUMERIC   }
        {NUMERIC        }
        {ALPHANUMERIC-EDITED}
        {NUMERIC-EDITED }
    ...]
```

**Default behavior (no options):**

```cobol
01  WS-RECORD.
    05  WS-NAME       PIC X(30).
    05  WS-AMOUNT     PIC S9(7)V99 COMP-3.
    05  WS-FLAG       PIC X.
    05  WS-COUNT      PIC 9(3).
    05  FILLER        PIC X(10).

INITIALIZE WS-RECORD
*> WS-NAME    = SPACES  (alphanumeric -> spaces)
*> WS-AMOUNT  = 0       (numeric -> zeros)
*> WS-FLAG    = SPACE   (alphanumeric -> spaces)
*> WS-COUNT   = 0       (numeric -> zeros)
*> FILLER     = unchanged (FILLER is skipped by default)
```

Default initialization values by category:
- Alphabetic: SPACES
- Alphanumeric: SPACES
- Alphanumeric edited: SPACES
- Numeric: ZEROS
- Numeric edited: ZEROS

**WITH FILLER option:**

```cobol
INITIALIZE WS-RECORD WITH FILLER
*> Same as above, but FILLER fields are also initialized
*> FILLER PIC X(10) = SPACES
```

**ALL option:**

The ALL keyword causes INITIALIZE to process all subordinate items, including those that would normally be skipped (such as items subordinate to OCCURS DEPENDING ON beyond the current count). Behavior varies by compiler.

**REPLACING option:**

```cobol
INITIALIZE WS-RECORD
    REPLACING NUMERIC DATA BY 999
              ALPHANUMERIC DATA BY ALL "*"
*> WS-NAME    = "******************************"
*> WS-AMOUNT  = 999.00 (or 0000999.00 with alignment)
*> WS-FLAG    = "*"
*> WS-COUNT   = 999
```

REPLACING lets you override the default values for specific categories. Multiple REPLACING phrases can appear in a single INITIALIZE. Each category can only appear once.

**Initializing a table:**

```cobol
01  WS-TABLE.
    05  WS-ENTRY OCCURS 100 TIMES.
        10  WS-ENTRY-NAME   PIC X(20).
        10  WS-ENTRY-AMT    PIC S9(5)V99 COMP-3.

INITIALIZE WS-TABLE
*> All 100 occurrences are initialized
*> Every WS-ENTRY-NAME = SPACES
*> Every WS-ENTRY-AMT  = 0
```

### SET Statement

SET has multiple distinct uses depending on the operand types:

**SET for indexes:**

```cobol
SET WS-INDEX TO 1
SET WS-INDEX UP BY 1
SET WS-INDEX DOWN BY 3
SET WS-INT-FIELD TO WS-INDEX
SET WS-INDEX TO WS-INT-FIELD
```

SET is the correct way to load and manipulate index data items (USAGE IS INDEX). MOVE cannot be used with index items. SET converts between index values and integer occurrence numbers.

**SET for condition names (88-levels):**

```cobol
01  WS-STATUS     PIC X.
    88  WS-ACTIVE     VALUE "A".
    88  WS-INACTIVE   VALUE "I".
    88  WS-DELETED    VALUE "D".

SET WS-ACTIVE TO TRUE
*> WS-STATUS now contains "A"

SET WS-DELETED TO TRUE
*> WS-STATUS now contains "D"
```

SET TO TRUE assigns the first VALUE associated with the condition name to the parent field. SET TO FALSE (where supported by the compiler) assigns the value specified in the FALSE clause (COBOL-2002 and later).

```cobol
01  WS-EOF-FLAG   PIC X VALUE "N".
    88  WS-EOF        VALUE "Y"
                      FALSE IS "N".

SET WS-EOF TO TRUE
*> WS-EOF-FLAG = "Y"

SET WS-EOF TO FALSE
*> WS-EOF-FLAG = "N"
```

**SET for pointer data items:**

```cobol
01  WS-PTR       USAGE POINTER.
01  WS-ADDRESS   USAGE POINTER.

SET WS-PTR TO ADDRESS OF WS-RECORD
SET WS-ADDRESS TO WS-PTR
SET WS-PTR TO NULL
```

**SET for table element search:**

SET is used implicitly by the SEARCH statement and explicitly to position indexes before SEARCH:

```cobol
SET WS-INDEX TO 1
SEARCH WS-TABLE-ENTRY
    AT END
        DISPLAY "NOT FOUND"
    WHEN WS-KEY(WS-INDEX) = WS-SEARCH-KEY
        DISPLAY "FOUND AT " WS-INDEX
END-SEARCH
```

## Common Patterns

### Clearing a Record

Several approaches, each with different behavior:

```cobol
*> Approach 1: MOVE SPACES (treats entire record as alphanumeric)
MOVE SPACES TO WS-RECORD
*> Sets everything to spaces, including numeric fields
*> Numeric COMP/COMP-3 fields will contain invalid data!

*> Approach 2: INITIALIZE (category-aware)
INITIALIZE WS-RECORD
*> Alphanumeric fields get spaces, numeric fields get zeros
*> FILLER is untouched (use WITH FILLER to include it)
*> This is almost always the right choice

*> Approach 3: MOVE LOW-VALUES (binary zeros)
MOVE LOW-VALUES TO WS-RECORD
*> Sets every byte to X'00'
*> Numeric COMP and COMP-3 fields become zero
*> Alphanumeric fields contain nulls, not spaces
```

### Initializing Mixed Records

When a record has both alphanumeric and numeric fields and you want specific non-default values:

```cobol
INITIALIZE WS-OUTPUT-RECORD
    REPLACING ALPHANUMERIC DATA BY SPACES
              NUMERIC DATA BY ZEROS

*> Or for a specific sentinel pattern:
INITIALIZE WS-OUTPUT-RECORD
    REPLACING NUMERIC DATA BY -1

*> For complete reset including FILLER:
INITIALIZE WS-OUTPUT-RECORD WITH FILLER
```

### Setting Flags and Condition Names

Using SET with 88-levels is cleaner and more self-documenting than direct MOVE:

```cobol
*> Instead of:
MOVE "Y" TO WS-EOF-FLAG

*> Prefer:
SET WS-EOF TO TRUE

*> The SET form is immune to value changes --
*> if you later change the 88-level VALUE from "Y" to "T",
*> SET WS-EOF TO TRUE still works, but
*> MOVE "Y" TO WS-EOF-FLAG would be wrong.
```

### Index Manipulation

```cobol
*> Position to start of table
SET WS-IDX TO 1

*> Advance through table
SET WS-IDX UP BY 1

*> Jump backward
SET WS-IDX DOWN BY 5

*> Save index position to a numeric field
SET WS-SAVE-POS TO WS-IDX

*> Restore index from a numeric field
SET WS-IDX TO WS-SAVE-POS
```

### Copying Between Similar Structures

When two structures share some but not all field names, MOVE CORRESPONDING is efficient:

```cobol
MOVE CORRESPONDING WS-DB-RECORD TO WS-REPORT-LINE
```

For structures with identical layouts, a simple group MOVE is more efficient:

```cobol
MOVE WS-CURRENT-REC TO WS-PREVIOUS-REC
```

### MOVE vs INITIALIZE vs VALUE

| Mechanism    | When to Use                                   | Scope                |
|--------------|-----------------------------------------------|----------------------|
| VALUE clause | Compile-time initialization of WORKING-STORAGE fields | Single field or literal |
| MOVE         | Runtime assignment from one field or literal to one or more fields | One or more receiving fields |
| INITIALIZE   | Runtime reset of a group to category-appropriate defaults | Entire group (recursive) |
| SET          | Index manipulation, 88-level flags, pointers  | Specific item types  |

**VALUE** is for initial values known at compile time. It is set once when the program loads (for WORKING-STORAGE) and cannot be made conditional.

**MOVE** is for runtime data transfer. Use it for simple assignments, copying between fields, and loading specific values.

**INITIALIZE** is for resetting records to a clean state, particularly when the record contains mixed data types. It applies the right default (spaces for alphanumeric, zeros for numeric) to every elementary item in the group.

## Gotchas

- **Group moves perform no conversion.** Moving a group containing COMP-3 fields to a group containing DISPLAY fields copies the packed bytes directly into display positions, producing garbage. If conversion is needed, use MOVE CORRESPONDING or move individual elementary items.

- **MOVE SPACES to a numeric field is technically invalid.** While some compilers allow `MOVE SPACES TO WS-NUMERIC-FIELD`, the result is a field containing space characters in numeric positions. Any subsequent arithmetic on that field will abend. Use `MOVE ZEROS` or `INITIALIZE` instead.

- **High-order truncation is silent.** Moving 12345 to a PIC 9(3) field stores 345 with no error or warning at runtime. The ON SIZE ERROR phrase applies only to arithmetic statements, not to MOVE.

- **INITIALIZE skips FILLER by default.** If your record layout uses FILLER for spacing or separators, those bytes are not touched by INITIALIZE unless you specify WITH FILLER. This can leave stale data in FILLER positions.

- **INITIALIZE does not reset fields subordinate to REDEFINES.** When a field is defined under a REDEFINES, INITIALIZE processes the original definition but not the redefined view. This can leave the redefined interpretation inconsistent.

- **MOVE CORRESPONDING requires exact name matches.** Names must match exactly at the elementary level. If one structure uses CUST-NAME and another uses CUSTOMER-NAME, no move occurs. Also, items at different qualification levels may not match as expected.

- **MOVE CORRESPONDING ignores FILLER.** Items named FILLER are never included in a CORRESPONDING move, even if both structures have identically positioned FILLER items.

- **SET TO FALSE requires the FALSE IS clause.** Using `SET condition-name TO FALSE` without a FALSE IS clause in the 88-level definition causes a compile error. This feature is from COBOL-2002 and not all compilers support it.

- **SET cannot replace MOVE for non-index, non-88 items.** SET is specifically for index data items, 88-level condition names, and pointers. You cannot use SET to move alphanumeric or numeric data between ordinary fields.

- **De-editing may not work across all compilers.** Moving a numeric edited field to a numeric field (de-editing) is standard, but older compilers or strict settings may produce unexpected results. Test de-editing moves if targeting multiple platforms.

- **MOVE with reference modification can exceed bounds.** `MOVE WS-SOURCE(5:10) TO WS-TARGET` will attempt to access 10 bytes starting at position 5 of WS-SOURCE. If WS-SOURCE is only 12 bytes, this reads positions 5-14, going 2 bytes past the end. Results are undefined.

- **INITIALIZE of a large table is slow.** INITIALIZE recursively processes every elementary item in every occurrence. For a table with thousands of entries, this can be measurably slower than a single `MOVE SPACES TO WS-TABLE` (though the latter does not properly initialize numeric fields).

## Related Topics

- **[data_types.md](data_types.md)** -- Defines PICTURE, USAGE, and the data categories that govern how MOVE conversion, truncation, and padding work. Understanding data types is a prerequisite for understanding data movement.
- **[working_storage.md](working_storage.md)** -- Covers the VALUE clause, REDEFINES, level numbers, FILLER, and 88-levels, all of which interact with MOVE, INITIALIZE, and SET.
- **[arithmetic.md](arithmetic.md)** -- Covers the GIVING phrase (which is an implicit MOVE of an arithmetic result), ROUNDED, and ON SIZE ERROR. COMPUTE with GIVING is sometimes used instead of MOVE for numeric conversion.
- **[table_handling.md](table_handling.md)** -- Covers OCCURS, indexes, SEARCH, and the SET statement in the context of table manipulation.
- **[string_handling.md](string_handling.md)** -- Covers STRING, UNSTRING, and INSPECT, which are alternative ways to construct and transform alphanumeric data beyond what simple MOVE provides.
- **[conditional_logic.md](conditional_logic.md)** -- Covers 88-level condition names and their use in IF/EVALUATE, which relates to SET TO TRUE/FALSE.
- **[copybooks.md](copybooks.md)** -- Group moves and MOVE CORRESPONDING are frequently used with record layouts defined in copybooks.
- **[intrinsic_functions.md](intrinsic_functions.md)** -- Functions like NUMVAL and NUMVAL-C provide an alternative to de-editing moves for converting alphanumeric representations to numeric values.
