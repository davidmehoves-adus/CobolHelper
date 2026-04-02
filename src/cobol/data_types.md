# Data Types

## Description

COBOL data types are defined through the PICTURE (PIC) clause and the USAGE clause, which together determine how data is stored in memory, how much space it occupies, and what operations can be performed on it. COBOL distinguishes between display (human-readable character) format and various computational (binary, packed-decimal, floating-point) formats, each with different storage characteristics and performance trade-offs. Understanding these data types is essential for correct arithmetic, efficient storage, accurate file I/O, and successful interoperation with databases and other languages.

## Table of Contents

- [Core Concepts](#core-concepts)
  - [The PICTURE Clause](#the-picture-clause)
  - [The USAGE Clause](#the-usage-clause)
  - [DISPLAY (Default) Format](#display-default-format)
  - [COMP / BINARY](#comp--binary)
  - [COMP-3 / PACKED-DECIMAL](#comp-3--packed-decimal)
  - [COMP-1 and COMP-2 (Floating Point)](#comp-1-and-comp-2-floating-point)
  - [Sign Handling](#sign-handling)
  - [Storage Size Summary](#storage-size-summary)
- [Syntax & Examples](#syntax--examples)
  - [Alphanumeric and Alphabetic Types](#alphanumeric-and-alphabetic-types)
  - [Numeric Display Types](#numeric-display-types)
  - [Computational Types](#computational-types)
  - [Edited Fields](#edited-fields)
  - [Sign Clause Examples](#sign-clause-examples)
- [Common Patterns](#common-patterns)
- [Gotchas](#gotchas)
- [Related Topics](#related-topics)

## Core Concepts

### The PICTURE Clause

The PICTURE (PIC) clause defines the category, size, and editing characteristics of a data item. It uses a string of symbols to describe the data:

**Category-determining symbols:**

| Symbol | Meaning | Category |
|--------|---------|----------|
| `9`    | Numeric digit (0-9) | Numeric |
| `A`    | Alphabetic character (A-Z, a-z, space) | Alphabetic |
| `X`    | Any character | Alphanumeric |
| `S`    | Sign (positive or negative), does not occupy a separate byte unless SIGN SEPARATE is specified | Numeric |
| `V`    | Implied (assumed) decimal point, does not occupy storage | Numeric |
| `P`    | Assumed decimal scaling position | Numeric |

**Editing symbols (for output formatting):**

| Symbol | Meaning |
|--------|---------|
| `Z`    | Zero-suppress: replaces leading zeros with spaces |
| `*`    | Zero-suppress: replaces leading zeros with asterisks |
| `.`    | Decimal point (insertion character) |
| `,`    | Comma (insertion character) |
| `$`    | Currency sign |
| `+`    | Plus sign (floating or fixed) |
| `-`    | Minus sign (floating or fixed) |
| `CR`   | Credit symbol (prints CR if negative) |
| `DB`   | Debit symbol (prints DB if negative) |
| `B`    | Blank insertion |
| `0`    | Zero insertion |
| `/`    | Slash insertion |

**Repetition shorthand:** A symbol followed by a number in parentheses repeats that symbol. For example, `PIC 9(5)` is equivalent to `PIC 99999`.

A data item's **category** is determined by its PIC symbols:

- **Numeric:** Contains only `9`, `S`, `V`, and `P`. Can participate in arithmetic.
- **Alphabetic:** Contains only `A`.
- **Alphanumeric:** Contains `X`, or a mixture of `A` and `9` without `S` or `V`.
- **Numeric-edited:** Numeric with editing symbols (`Z`, `*`, `.`, `,`, `$`, `+`, `-`, `CR`, `DB`, etc.). Cannot participate in arithmetic.
- **Alphanumeric-edited:** Alphanumeric with insertion characters (`B`, `0`, `/`).

### The USAGE Clause

The USAGE clause specifies how data is stored internally in memory. It determines the physical representation and, consequently, the storage size. When USAGE is omitted, the default is DISPLAY.

The USAGE clause can be specified at the group level, in which case it applies to all elementary items within the group (unless overridden).

| USAGE | Storage Format | Typical Use |
|-------|---------------|-------------|
| DISPLAY | One character per byte (EBCDIC or ASCII) | Default; file I/O, screen display |
| COMP / BINARY | Binary integer | Subscripts, counters, efficient arithmetic |
| COMP-3 / PACKED-DECIMAL | Two digits per byte, sign in low nibble | Financial calculations, DB2 columns |
| COMP-1 | Single-precision floating point (4 bytes) | Scientific calculations |
| COMP-2 | Double-precision floating point (8 bytes) | Scientific calculations |
| COMP-5 | Native binary (platform byte order) | Interop with C, system calls |
| INDEX | Internal index format | Table indexing |

### DISPLAY (Default) Format

In DISPLAY format, each digit or character occupies one byte. For numeric DISPLAY items, each digit 0-9 is stored as its character representation (EBCDIC or ASCII encoding). The sign, if present (`S` in PIC), is embedded in the last byte (trailing overpunch) by default -- the high nibble of the last byte encodes the sign while the low nibble holds the digit value.

For example, `PIC S9(3)` storing +123 in EBCDIC:
- Bytes: `F1 F2 C3` (the `C` in the high nibble of the last byte indicates positive; `D` would indicate negative).

In ASCII environments, similar overpunch conventions apply but with different encoding values.

DISPLAY items are the native format for file records and screen I/O. They are human-readable when viewed in a hex dump (each byte corresponds to one printable character position). However, arithmetic on DISPLAY items requires conversion, making them slower for computation-heavy processing.

### COMP / BINARY

COMP (or BINARY, or COMPUTATIONAL) stores data as a binary integer. The number of bytes allocated depends on the number of digits in the PIC clause:

| PIC Digits | Bytes | Value Range |
|-----------|-------|-------------|
| 1-4       | 2 (halfword) | -9999 to +9999 (PIC-limited) |
| 5-9       | 4 (fullword) | -999,999,999 to +999,999,999 (PIC-limited) |
| 10-18     | 8 (doubleword) | Up to 18 digits (PIC-limited) |

With standard COMP/BINARY, the PIC clause constrains the range of values. For instance, `PIC S9(4) COMP` uses 2 bytes but only allows values from -9999 to +9999, even though a signed 16-bit integer could hold up to 32,767. This is called **truncation to PIC size** -- values outside the PIC range cause truncation.

COMP items are ideal for:
- Subscripts and loop counters (fastest for indexing)
- Flags and small integers
- Fields passed to system APIs or external programs expecting binary data

If a COMP field has an implied decimal (`V`), the decimal position is tracked by the compiler but the field is still stored as a binary integer scaled accordingly.

### COMP-3 / PACKED-DECIMAL

COMP-3, also known as PACKED-DECIMAL, stores two decimal digits per byte with the sign in the low nibble of the last byte. This is the most common format for financial and business data in mainframe COBOL.

**Storage layout:** Each byte holds two digits (one in the high nibble, one in the low nibble), except the last byte, which holds one digit in the high nibble and the sign in the low nibble.

**Storage size formula:**
```
bytes = INTEGER((number_of_digits / 2) + 1)
```

| PIC Digits | Bytes |
|-----------|-------|
| 1-2       | 2     |
| 3-4       | 3     |
| 5-6       | 4     |
| 7-8       | 5     |
| 9-10      | 6     |
| 11-12     | 7     |
| 13-14     | 8     |
| 15-16     | 9     |
| 17-18     | 10    |

**Sign nibble values:**
- `C` = positive
- `D` = negative
- `F` = unsigned (positive)

**Example:** `PIC S9(5) COMP-3` storing +12345:
```
Hex: 01 23 45 0C
     ^^ ^^ ^^ ^^
     01 23 45 0C  -> digits: 0,1,2,3,4,5  sign: C (positive)
```
Wait -- let us be precise. `PIC S9(5)` is 5 digits, so bytes = (5/2)+1 = 3 bytes:
```
Hex: 12 34 5C
     ^^ ^^ ^^
     1,2  3,4  5,sign(+)
```

COMP-3 is preferred for:
- Financial amounts (dollars, quantities, rates)
- DB2 DECIMAL column mappings
- Any field involved in decimal arithmetic (avoids binary-to-decimal conversion overhead)

### COMP-1 and COMP-2 (Floating Point)

COMP-1 and COMP-2 store floating-point numbers. These do NOT use a PIC clause.

| USAGE  | Precision | Bytes | Approximate Range |
|--------|-----------|-------|-------------------|
| COMP-1 | Single (approx. 7 significant digits) | 4 | ~5.4E-79 to ~7.2E+75 (IBM hex float) or IEEE equivalent |
| COMP-2 | Double (approx. 16 significant digits) | 8 | ~5.4E-79 to ~7.2E+75 (IBM hex float) or IEEE equivalent |

On IBM mainframes, the traditional format is hexadecimal floating point (HFP). Many modern compilers also support IEEE 754 binary floating point (BFP) via compiler options.

**Important:** Floating-point types are subject to rounding errors and should NOT be used for financial calculations where exact decimal precision is required. Use COMP-3 for monetary values.

Declaration syntax (no PIC clause):
```cobol
01  WS-SINGLE-FLOAT    USAGE COMP-1.
01  WS-DOUBLE-FLOAT    USAGE COMP-2.
```

### Sign Handling

The SIGN clause controls how the sign is stored for numeric DISPLAY items. It only applies to items with `S` in the PIC and with USAGE DISPLAY.

**Three options:**

1. **SIGN TRAILING (default):** The sign is embedded in the last (rightmost) byte via overpunch encoding. No extra byte is used.

2. **SIGN LEADING:** The sign is embedded in the first (leftmost) byte via overpunch encoding. No extra byte is used.

3. **SIGN TRAILING SEPARATE / SIGN LEADING SEPARATE:** The sign is stored as a separate byte (ASCII `+` or `-`), either after (trailing) or before (leading) the digits. This adds one byte to the field's storage size.

**Overpunch encoding (EBCDIC):**

| Digit | Positive (C zone) | Negative (D zone) |
|-------|-------------------|-------------------|
| 0     | { (X'C0')         | } (X'D0')         |
| 1     | A (X'C1')         | J (X'D1')         |
| 2     | B (X'C2')         | K (X'D2')         |
| 3     | C (X'C3')         | L (X'D3')         |
| 4     | D (X'C4')         | M (X'D4')         |
| 5     | E (X'C5')         | N (X'D5')         |
| 6     | F (X'C6')         | O (X'D6')         |
| 7     | G (X'C7')         | P (X'D7')         |
| 8     | H (X'C8')         | Q (X'D8')         |
| 9     | I (X'C9')         | R (X'D9')         |

SIGN SEPARATE is often used when:
- Data is exchanged with non-mainframe systems that do not understand overpunch
- Flat files need a visible, explicit sign character
- Interfacing with programs or tools that expect readable sign characters

### Storage Size Summary

For a quick reference, here is how storage size is calculated for the main USAGE types:

| USAGE | Storage Calculation |
|-------|-------------------|
| DISPLAY (no sign or embedded sign) | 1 byte per PIC character position |
| DISPLAY with SIGN SEPARATE | 1 byte per PIC character position + 1 byte for sign |
| COMP / BINARY | 2 bytes for PIC 1-4 digits; 4 bytes for PIC 5-9; 8 bytes for PIC 10-18 |
| COMP-3 / PACKED-DECIMAL | INTEGER(digits / 2) + 1 bytes |
| COMP-1 | 4 bytes (no PIC) |
| COMP-2 | 8 bytes (no PIC) |

## Syntax & Examples

### Alphanumeric and Alphabetic Types

```cobol
       01  WS-CUSTOMER-NAME      PIC X(30).
       01  WS-MIDDLE-INITIAL     PIC A.
       01  WS-ADDRESS-LINE       PIC X(50).
       01  WS-FILLER             PIC X(10) VALUE SPACES.
```

- `PIC X(30)` defines a 30-byte alphanumeric field that can hold any character.
- `PIC A` defines a single alphabetic character (letters and space only).
- The VALUE clause sets an initial value. `SPACES` is a figurative constant that fills the field with space characters.

### Numeric Display Types

```cobol
       01  WS-QUANTITY           PIC 9(5).
       01  WS-AMOUNT             PIC S9(7)V99.
       01  WS-RATE               PIC S9V9(4).
       01  WS-UNSIGNED-AMT       PIC 9(9)V99.
```

- `PIC 9(5)` defines a 5-digit unsigned integer in DISPLAY format (5 bytes).
- `PIC S9(7)V99` defines a signed 9-digit number with 2 implied decimal places (9 bytes in DISPLAY; the `V` and `S` do not occupy storage with default sign handling).
- `PIC S9V9(4)` defines a signed number with 1 integer digit and 4 decimal places (5 bytes).
- The `V` marks where the decimal point is assumed to be. It affects alignment in arithmetic but occupies no storage.

### Computational Types

```cobol
      * Binary / COMP fields
       01  WS-COUNTER            PIC S9(4)  COMP.
       01  WS-TABLE-INDEX        PIC S9(8)  BINARY.
       01  WS-LARGE-NUM          PIC S9(15) COMP.

      * Packed-decimal / COMP-3 fields
       01  WS-PRICE              PIC S9(7)V99 COMP-3.
       01  WS-TAX-RATE           PIC S9V9(5)  COMP-3.
       01  WS-BALANCE            PIC S9(13)V99 COMP-3.

      * Floating-point fields (no PIC clause)
       01  WS-SCIENTIFIC-VAL     COMP-1.
       01  WS-PRECISE-FLOAT      COMP-2.

      * Native binary (platform-dependent)
       01  WS-RETURN-CODE        PIC S9(9) COMP-5.
```

**Storage breakdown for the above:**

| Field | PIC | USAGE | Bytes |
|-------|-----|-------|-------|
| WS-COUNTER | S9(4) | COMP | 2 |
| WS-TABLE-INDEX | S9(8) | BINARY | 4 |
| WS-LARGE-NUM | S9(15) | COMP | 8 |
| WS-PRICE | S9(7)V99 | COMP-3 | 5 |
| WS-TAX-RATE | S9V9(5) | COMP-3 | 4 |
| WS-BALANCE | S9(13)V99 | COMP-3 | 8 |
| WS-SCIENTIFIC-VAL | (none) | COMP-1 | 4 |
| WS-PRECISE-FLOAT | (none) | COMP-2 | 8 |
| WS-RETURN-CODE | S9(9) | COMP-5 | 4 |

### Edited Fields

Numeric-edited and alphanumeric-edited fields are used for formatted output such as reports and screens. They cannot be used in arithmetic.

```cobol
      * Numeric-edited fields
       01  WS-DISPLAY-AMOUNT     PIC $ZZZ,ZZ9.99.
       01  WS-DISPLAY-NEGATIVE   PIC -ZZZ,ZZ9.99.
       01  WS-CHECK-AMOUNT       PIC $***,**9.99.
       01  WS-CREDIT-IND         PIC Z(6)9.99CR.
       01  WS-SIGNED-OUT         PIC +9(5).

      * Alphanumeric-edited fields
       01  WS-FORMATTED-DATE     PIC 99/99/9999.
       01  WS-PHONE-NUMBER       PIC 999B9999.
```

Using edited fields with MOVE:

```cobol
       MOVE 12345.67 TO WS-DISPLAY-AMOUNT
      * Result: " $12,345.67"

       MOVE -500.00  TO WS-DISPLAY-NEGATIVE
      * Result: "     -500.00"

       MOVE 50.00    TO WS-CHECK-AMOUNT
      * Result: "$*****50.00"

       MOVE -1234.56 TO WS-CREDIT-IND
      * Result: "   1234.56CR"

       MOVE 20260115 TO WS-FORMATTED-DATE
      * Result: "01/15/2026"  (with slashes inserted)
```

### Sign Clause Examples

```cobol
      * Default: sign embedded in trailing digit (overpunch)
       01  WS-DEFAULT-SIGN       PIC S9(5).

      * Sign embedded in leading digit
       01  WS-LEADING-SIGN       PIC S9(5)
                                 SIGN IS LEADING.

      * Separate trailing sign character (+ or -)
       01  WS-TRAIL-SEP          PIC S9(5)
                                 SIGN IS TRAILING SEPARATE.

      * Separate leading sign character (+ or -)
       01  WS-LEAD-SEP           PIC S9(5)
                                 SIGN IS LEADING SEPARATE.
```

**Storage comparison for value +12345:**

| Declaration | Bytes | Hex (EBCDIC) | Display |
|------------|-------|--------------|---------|
| `PIC S9(5)` (default, trailing overpunch) | 5 | `F1 F2 F3 F4 C5` | `1234E` |
| `PIC S9(5) SIGN LEADING` | 5 | `C1 F2 F3 F4 F5` | `A2345` |
| `PIC S9(5) SIGN TRAILING SEPARATE` | 6 | `F1 F2 F3 F4 F5 4E` | `12345+` |
| `PIC S9(5) SIGN LEADING SEPARATE` | 6 | `4E F1 F2 F3 F4 F5` | `+12345` |

## Common Patterns

### Financial Amount Fields

Most production COBOL systems use COMP-3 for monetary amounts to guarantee exact decimal precision and reasonable storage efficiency:

```cobol
       01  WS-FINANCIAL-FIELDS.
           05  WS-INVOICE-AMT    PIC S9(11)V99 COMP-3.
           05  WS-TAX-AMT        PIC S9(9)V99  COMP-3.
           05  WS-DISCOUNT-PCT   PIC S9V9(4)   COMP-3.
           05  WS-TOTAL-AMT      PIC S9(13)V99 COMP-3.
```

### Counters and Subscripts as COMP

Binary fields are fastest for loop counters and table subscripts because the CPU can index directly without conversion:

```cobol
       01  WS-COUNTERS.
           05  WS-LOOP-CTR       PIC S9(4) COMP.
           05  WS-TABLE-SUB      PIC S9(4) COMP.
           05  WS-RECORD-COUNT   PIC S9(8) COMP.

       PERFORM VARYING WS-LOOP-CTR FROM 1 BY 1
               UNTIL WS-LOOP-CTR > WS-MAX-ENTRIES
           MOVE WS-TABLE-ENTRY(WS-LOOP-CTR)
               TO WS-OUTPUT-LINE
       END-PERFORM
```

### File Record Layouts with Mixed Types

Production file copybooks typically use DISPLAY for character data and either DISPLAY or COMP-3 for numeric data, depending on whether the file is text or binary:

```cobol
      * Text file record - all DISPLAY
       01  CUSTOMER-RECORD.
           05  CUST-ID           PIC 9(8).
           05  CUST-NAME         PIC X(30).
           05  CUST-BALANCE      PIC S9(9)V99.
           05  CUST-STATUS       PIC X.

      * Binary file record - mixed types
       01  TRANSACTION-RECORD.
           05  TXN-SEQ-NUM       PIC S9(8)  COMP.
           05  TXN-AMOUNT        PIC S9(9)V99 COMP-3.
           05  TXN-DATE          PIC 9(8).
           05  TXN-TYPE          PIC X(2).
```

### DB2 Host Variable Declarations

DB2 DECIMAL columns map to COMP-3, INTEGER columns map to COMP, and VARCHAR columns use a two-part group structure:

```cobol
       01  WS-DB2-HOST-VARS.
           05  HV-ACCOUNT-ID     PIC S9(9)  COMP.
           05  HV-BALANCE        PIC S9(13)V99 COMP-3.
           05  HV-DESCRIPTION.
               10  HV-DESC-LEN   PIC S9(4) COMP.
               10  HV-DESC-TEXT  PIC X(200).
```

### Redefinition for Multiple Interpretations

A common pattern is to REDEFINE a field to access the same storage with different PIC definitions:

```cobol
       01  WS-DATE-NUMERIC       PIC 9(8).
       01  WS-DATE-PARTS REDEFINES WS-DATE-NUMERIC.
           05  WS-DATE-YYYY      PIC 9(4).
           05  WS-DATE-MM        PIC 9(2).
           05  WS-DATE-DD        PIC 9(2).

       01  WS-AMOUNT-COMP3       PIC S9(7)V99 COMP-3.
       01  WS-AMOUNT-BYTES REDEFINES WS-AMOUNT-COMP3
                                  PIC X(5).
```

### Group-Level USAGE

Specifying USAGE at the group level applies it to all subordinate elementary items:

```cobol
       01  WS-BINARY-GROUP       USAGE COMP.
           05  WS-BIN-A          PIC S9(4).
           05  WS-BIN-B          PIC S9(8).
           05  WS-BIN-C          PIC S9(4).
```

All three fields above are stored as COMP (binary), even though COMP is specified only at the 01 level.

### Condition Names on Numeric Fields

88-level condition names work with all data types:

```cobol
       01  WS-STATUS-CODE        PIC 9(2) COMP-3.
           88  STATUS-SUCCESS     VALUE 0.
           88  STATUS-NOT-FOUND   VALUE 10.
           88  STATUS-ERROR       VALUES 90 THRU 99.
```

## Gotchas

- **Unsigned arithmetic produces unsigned results.** If you subtract a larger number from a smaller one in a field defined as `PIC 9(5)` (no `S`), the result will be the absolute value, not a negative number. Always use `PIC S9(...)` for fields that might hold negative values.

- **Implied decimal misalignment.** Moving `PIC S9(5)V99` to `PIC S9(7)V9` causes silent decimal truncation -- the second decimal place is dropped, not rounded. Always check that sending and receiving fields have compatible decimal positions, or use ROUNDED in arithmetic.

- **COMP truncation to PIC size.** A field declared as `PIC S9(4) COMP` occupies 2 bytes (can hold up to 32,767) but the compiler enforces the PIC limit of 9999. Storing 10000 in this field causes truncation. If you need the full binary range, use COMP-5.

- **COMP-3 odd-digit alignment.** COMP-3 always has an odd number of digit positions internally (the sign nibble pairs with the last digit). If you specify an even number of digits like `PIC S9(6) COMP-3`, the compiler allocates space for 7 digit positions (4 bytes), with the leading digit position being zero. This does not change behavior, but it matters for storage calculations and hex-dump analysis.

- **Moving COMP-3 to alphanumeric fields.** Moving a COMP-3 field directly to a `PIC X` field copies the raw packed bytes, not a readable number. You must move through a numeric DISPLAY intermediate or a numeric-edited field to get readable output.

- **Group moves bypass type conversion.** When you move data at the group level, COBOL treats both fields as alphanumeric and performs a simple byte copy -- no numeric conversion occurs. This is dangerous when group items contain COMP or COMP-3 fields with different layouts.

- **SIGN SEPARATE adds a byte.** A field declared as `PIC S9(5) SIGN TRAILING SEPARATE` occupies 6 bytes, not 5. If this field is part of a record layout, miscounting the size by one byte shifts all subsequent fields, causing data corruption. This is a common bug in copybook maintenance.

- **Floating-point (COMP-1/COMP-2) rounding.** Never use COMP-1 or COMP-2 for monetary calculations. The value 0.10 cannot be represented exactly in binary floating-point, leading to subtle rounding errors that compound across millions of transactions. Always use COMP-3 for financial data.

- **Mixing DISPLAY signed and unsigned in comparisons.** Comparing a signed DISPLAY field to an unsigned one can produce unexpected results because the overpunch zone bits differ. A negative value in a signed field may compare incorrectly to an unsigned field if the compiler does not generate proper comparison code. Ensure consistent sign usage across compared fields.

- **COMP-5 platform dependency.** COMP-5 (native binary) uses the full binary range of the allocated bytes, not the PIC-limited range. A `PIC S9(4) COMP-5` field can hold values up to 32,767, not just 9999. This makes COMP-5 useful for system interop but means the field can hold values that exceed the PIC specification, which can cause unexpected results if the value is moved to a standard field.

- **Zero in COMP-3 can have multiple representations.** Positive zero (`0C`), negative zero (`0D`), and unsigned zero (`0F`) are all valid packed-decimal representations of zero. While they should compare equal, some bit-level comparisons or hex-dump inspections may show unexpected sign nibbles. Initialize COMP-3 fields explicitly to ensure consistent representation.

- **PIC 9 vs PIC X for numeric data in files.** When reading a file, a field declared `PIC 9(5)` expects valid numeric characters (EBCDIC F0-F9). If the file contains spaces or other non-numeric characters in that position, using the field in arithmetic will cause a data exception (S0C7 abend on mainframes). Validate input data or use `PIC X` with explicit numeric checking before moving to a numeric field.

- **Edited fields cannot be used in arithmetic.** A field like `PIC $ZZZ,ZZ9.99` is numeric-edited and cannot appear in ADD, SUBTRACT, MULTIPLY, DIVIDE, or COMPUTE statements. You must perform arithmetic on unedited numeric fields and then MOVE the result to the edited field for display.

- **REDEFINES with different USAGE types.** When you redefine a COMP-3 field with a DISPLAY field (or vice versa), the raw bytes are reinterpreted without conversion. This is intentional for hex-dump inspection, but accidental REDEFINES mismatches are a common source of data corruption bugs.

## Related Topics

- [copybooks.md](copybooks.md) -- Data type declarations are most commonly encountered inside copybooks that define shared record layouts. Understanding PIC clauses and USAGE is essential for reading and maintaining copybooks.
- [arithmetic.md](arithmetic.md) -- The choice of USAGE (COMP, COMP-3, DISPLAY) directly affects arithmetic precision, performance, and rounding behavior. Decimal alignment via the `V` in PIC clauses is critical for correct arithmetic results.
- [working_storage.md](working_storage.md) -- WORKING-STORAGE SECTION is where most data items are declared with their PIC and USAGE clauses. Initialization via VALUE clauses and storage allocation rules are covered there.
- [table_handling.md](table_handling.md) -- Table subscripts should be declared as COMP/BINARY for optimal performance. Table entries use standard PIC and USAGE definitions.
- [file_handling.md](file_handling.md) -- File record layouts use data type declarations extensively. The choice between DISPLAY and computational formats depends on whether the file is text-based or binary.
- [db2_embedded_sql.md](db2_embedded_sql.md) -- DB2 host variables must match the column data types. DECIMAL maps to COMP-3, INTEGER/SMALLINT to COMP, and VARCHAR uses a two-part group structure. Mismatched data types between host variables and DB2 columns cause SQLCODE errors.
