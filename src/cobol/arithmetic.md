# Arithmetic

## Description

This file covers all COBOL arithmetic operations: the five arithmetic statements (ADD, SUBTRACT, MULTIPLY, DIVIDE, COMPUTE), their optional phrases (GIVING, REMAINDER, ROUNDED, ON SIZE ERROR, NOT ON SIZE ERROR, CORRESPONDING), and the rules governing precision, truncation, decimal alignment, and intermediate results. Reference this file whenever you need to understand how COBOL performs calculations, how numeric data is transformed during arithmetic, or how to avoid common precision pitfalls.

## Table of Contents

- [Core Concepts](#core-concepts)
  - [The Five Arithmetic Statements](#the-five-arithmetic-statements)
  - [The GIVING Phrase](#the-giving-phrase)
  - [The ROUNDED Phrase](#the-rounded-phrase)
  - [The ON SIZE ERROR Phrase](#the-on-size-error-phrase)
  - [The REMAINDER Phrase](#the-remainder-phrase)
  - [The CORRESPONDING Phrase](#the-corresponding-phrase)
  - [Decimal Alignment](#decimal-alignment)
  - [Intermediate Results](#intermediate-results)
  - [Order of Operations in COMPUTE](#order-of-operations-in-compute)
  - [Precision and Truncation Rules](#precision-and-truncation-rules)
- [Syntax & Examples](#syntax--examples)
  - [ADD Statement](#add-statement)
  - [SUBTRACT Statement](#subtract-statement)
  - [MULTIPLY Statement](#multiply-statement)
  - [DIVIDE Statement](#divide-statement)
  - [COMPUTE Statement](#compute-statement)
  - [ROUNDED Examples](#rounded-examples)
  - [ON SIZE ERROR Examples](#on-size-error-examples)
- [Common Patterns](#common-patterns)
- [Gotchas](#gotchas)
- [Related Topics](#related-topics)

## Core Concepts

### The Five Arithmetic Statements

COBOL provides five arithmetic statements, all residing in the PROCEDURE DIVISION:

| Statement  | Purpose                                       |
|------------|-----------------------------------------------|
| ADD        | Adds one or more numeric values               |
| SUBTRACT   | Subtracts one or more numeric values           |
| MULTIPLY   | Multiplies two numeric values                  |
| DIVIDE     | Divides one numeric value by another           |
| COMPUTE    | Evaluates an arbitrary arithmetic expression   |

ADD, SUBTRACT, MULTIPLY, and DIVIDE are **verb-oriented** statements: each performs exactly one kind of operation. COMPUTE is **expression-oriented**: it evaluates a full arithmetic expression using `+`, `-`, `*`, `/`, and `**` (exponentiation) and stores the result.

All five statements share three optional phrases: GIVING, ROUNDED, and ON SIZE ERROR / NOT ON SIZE ERROR. DIVIDE alone supports REMAINDER.

Every operand in an arithmetic statement must be a numeric item (a data item with a numeric PICTURE or a numeric literal). The receiving field (the item that stores the result) must be a numeric or numeric-edited item. Numeric-edited items are permitted only when the GIVING phrase is used.

### The GIVING Phrase

Without GIVING, the result of an arithmetic operation is stored back into one of the operands, which must therefore be a numeric elementary item (not a literal, not numeric-edited). With GIVING, the result is stored into a separate receiving field, leaving the original operands unchanged. The receiving field in a GIVING phrase may be numeric-edited.

Key rules:

- GIVING is required on MULTIPLY and DIVIDE when neither operand should be modified.
- GIVING is optional on ADD and SUBTRACT.
- Multiple receiving fields are permitted after GIVING in ADD, SUBTRACT, and DIVIDE. Each may independently specify ROUNDED.
- When GIVING is used, all operands before the GIVING keyword are treated as sending fields and are not modified.

### The ROUNDED Phrase

When the result of an arithmetic operation has more decimal places than the receiving field can hold, the excess digits are normally **truncated** (chopped off). Specifying ROUNDED causes the last retained decimal digit to be increased by one if the first truncated digit is five or greater. This is sometimes called "round half up" or "commercial rounding."

ROUNDED is specified immediately after the receiving data-name:

```
ADD A TO B ROUNDED
COMPUTE C ROUNDED = A * B
```

When multiple receiving fields are present, each may independently specify ROUNDED:

```
ADD A TO B ROUNDED C D ROUNDED
```

Starting with the 2002 COBOL standard, additional rounding modes were introduced (NEAREST-AWAY-FROM-ZERO, NEAREST-TOWARD-ZERO, NEAREST-EVEN, TRUNCATION, TOWARD-GREATER, TOWARD-LESSER, PROHIBITED). In earlier standards, ROUNDED always means "round half away from zero" (the absolute value is rounded up when the leftover is exactly 0.5). Most legacy code uses the default ROUNDED without a mode qualifier.

### The ON SIZE ERROR Phrase

A **size error** occurs when the result of an arithmetic operation is too large (in absolute value) to fit in the integer portion of the receiving field, or when a division by zero is attempted.

```
ADD A TO B
    ON SIZE ERROR
        PERFORM HANDLE-OVERFLOW
    NOT ON SIZE ERROR
        PERFORM CONTINUE-PROCESSING
END-ADD
```

Rules:

- If ON SIZE ERROR is specified and a size error occurs, the receiving field is **not modified** and the imperative statement following ON SIZE ERROR is executed.
- If NOT ON SIZE ERROR is specified and no size error occurs, its imperative statement is executed.
- If ON SIZE ERROR is **not** specified and a size error occurs, the result is **undefined** (truncated from the left in most implementations) and no exception is raised. This is a common source of silent bugs.
- A size error is checked **after** rounding, if ROUNDED is also specified.
- When multiple receiving fields appear (e.g., `ADD A TO B C D`), each is evaluated independently. A size error in one does not prevent the others from being updated.
- Division by zero always triggers a size error condition. The quotient field and the remainder field (if present) are unchanged.
- ON SIZE ERROR does not detect loss of decimal precision (truncation of fractional digits). It only detects overflow of the integer portion.

The scope terminator (END-ADD, END-SUBTRACT, END-MULTIPLY, END-DIVIDE, END-COMPUTE) is strongly recommended when ON SIZE ERROR is used, to clearly delimit the scope of the error-handling logic.

### The REMAINDER Phrase

REMAINDER is exclusive to the DIVIDE statement. After the division, the remainder is calculated and stored in the data item named after REMAINDER.

The remainder is defined as:

```
REMAINDER = DIVIDEND - (QUOTIENT * DIVISOR)
```

Where QUOTIENT is the **truncated** (not rounded) result of the division, even if ROUNDED was specified on the quotient receiving field. This means the REMAINDER is always computed from the integer-truncated quotient, which preserves the mathematical identity: `dividend = quotient * divisor + remainder`.

Important: if ROUNDED is specified on the quotient, the quotient field itself will contain the rounded value, but the remainder is still calculated using the unrounded (truncated) quotient. This can cause confusion if not understood.

### The CORRESPONDING Phrase

ADD CORRESPONDING (or ADD CORR) and SUBTRACT CORRESPONDING (or SUBTRACT CORR) perform the arithmetic operation on pairs of elementary numeric items that share the same name within two group items.

```
ADD CORRESPONDING WS-INPUT-RECORD TO WS-TOTAL-RECORD
```

This is equivalent to writing individual ADD statements for each pair of identically named numeric elementary items found in both group items. Non-numeric items and items without matching names are ignored.

CORRESPONDING is only available on ADD and SUBTRACT. It is not available on MULTIPLY, DIVIDE, or COMPUTE.

### Decimal Alignment

COBOL automatically aligns operands on the decimal point before performing arithmetic. This is a fundamental feature of COBOL numeric processing and requires no programmer action.

How it works:

1. The compiler examines the PICTURE clause of each operand to determine the position of the assumed decimal point (V in the PICTURE string, or the rightmost position if no V is present).
2. Operands are conceptually aligned so that their decimal points line up.
3. The arithmetic is performed with all digits in their correct positional value.
4. The result is then stored in the receiving field, again aligned on the receiving field's decimal point.

Example: if `WS-PRICE` is `PIC 9(5)V99` (5 integer digits, 2 decimal digits) and `WS-TAX` is `PIC 9(3)V9(4)` (3 integer digits, 4 decimal digits), adding them together aligns the decimal points so that the integer and fractional parts add correctly. The result is then mapped to the receiving field's PICTURE, with truncation or rounding applied as needed.

Decimal alignment applies to all five arithmetic statements and to the evaluation of expressions in COMPUTE.

### Intermediate Results

When the compiler evaluates arithmetic expressions (particularly in COMPUTE, but also in certain multi-operand ADD and SUBTRACT scenarios), it must determine the size (number of digits) and decimal placement of **intermediate results** -- temporary internal fields that hold partial calculations.

The COBOL standard defines rules for intermediate result sizes based on the operands involved. The precise rules vary by compiler, but the general principles are:

- **Addition and Subtraction**: The intermediate result has enough integer digits to hold the larger integer portion plus one (for possible carry), and enough decimal digits to hold the larger decimal portion.
- **Multiplication**: The intermediate result has the sum of the integer digits of both operands for the integer portion, and the sum of the decimal digits for the decimal portion.
- **Division**: The intermediate result precision depends on the compiler. Many compilers allocate enough digits to hold the full precision of the dividend, extended by the number of decimal places needed for the quotient.
- **Exponentiation**: The intermediate result size depends on the exponent value and the base precision. This is compiler-dependent.

Most mainframe COBOL compilers (IBM Enterprise COBOL, for example) use 18-digit intermediate results by default, with options to extend to 31 digits (the ARITH(EXTEND) compiler option). This matters because if an intermediate result exceeds the allocated precision, truncation occurs silently during the calculation, not just at the final store.

Programmers writing complex COMPUTE expressions should be aware that intermediate overflow can produce incorrect results without triggering ON SIZE ERROR, because the size error check only applies to the final store into the receiving field.

### Order of Operations in COMPUTE

The COMPUTE statement evaluates expressions following standard mathematical precedence:

| Precedence | Operator          | Meaning          |
|------------|-------------------|------------------|
| 1 (highest)| Unary `+` / `-`  | Sign             |
| 2          | `**`              | Exponentiation   |
| 3          | `*` / `/`         | Multiplication / Division |
| 4 (lowest) | `+` / `-`         | Addition / Subtraction |

Rules:

- Operators at the same precedence level are evaluated **left to right**.
- Parentheses override the default precedence. Expressions inside parentheses are evaluated first, from innermost to outermost.
- Exponentiation is **not** right-associative in COBOL (unlike some mathematical conventions). `A ** B ** C` is evaluated as `(A ** B) ** C`, left to right.
- There is no modulus (mod) operator in COBOL. Use DIVIDE with REMAINDER instead.
- Concatenation and string operations cannot appear in COMPUTE expressions. COMPUTE is strictly numeric.

Example:

```cobol
COMPUTE WS-RESULT = A + B * C ** 2 - D / E
```

Evaluation order:
1. `C ** 2` (exponentiation first)
2. `B * (result of step 1)` (multiplication)
3. `D / E` (division, same precedence as multiplication, but to the right)
4. `A + (result of step 2)` (addition)
5. `(result of step 4) - (result of step 3)` (subtraction)

### Precision and Truncation Rules

COBOL arithmetic precision is governed by the PICTURE clause of the receiving field. Understanding truncation is essential for writing correct financial and business calculations.

**Integer truncation (high-order / left truncation):**
When the integer portion of a result is larger than the receiving field can hold, the leftmost (most significant) digits are silently lost -- unless ON SIZE ERROR is specified. For example, storing the value 12345 into a `PIC 9(3)` field yields 345 (the leading 12 is truncated). This is a size error condition.

**Decimal truncation (low-order / right truncation):**
When the result has more decimal places than the receiving field, the rightmost decimal digits are dropped. For example, storing 1.23456 into a `PIC 9V99` field yields 1.23 (without ROUNDED) or 1.23 (with ROUNDED, since the next digit 4 < 5). Storing 1.23556 into `PIC 9V99` with ROUNDED yields 1.24. Decimal truncation does **not** trigger a size error.

**Sign handling:**
If the receiving field is unsigned (no S in PICTURE), the absolute value of the result is stored. The sign is silently discarded. If the receiving field is signed (S in PICTURE), the sign is preserved. Losing the sign by storing a negative result in an unsigned field does **not** trigger a size error.

**Numeric-edited receiving fields:**
When GIVING places a result into a numeric-edited field (e.g., `PIC $ZZ,ZZ9.99`), the value is moved into the field with editing applied. The editing rules follow the same conventions as a MOVE to a numeric-edited field. Truncation and size error rules apply to the numeric value before editing.

## Syntax & Examples

### ADD Statement

**Format 1 -- ADD TO:**

```cobol
ADD identifier-1 [identifier-2 ...] TO identifier-3 [ROUNDED]
    [identifier-4 [ROUNDED]] ...
    [ON SIZE ERROR imperative-statement-1]
    [NOT ON SIZE ERROR imperative-statement-2]
[END-ADD]
```

All identifiers before TO are summed together with each identifier after TO, and each result is stored in the corresponding identifier after TO.

```cobol
       01 WS-A        PIC 9(3)  VALUE 100.
       01 WS-B        PIC 9(3)  VALUE 200.
       01 WS-C        PIC 9(3)  VALUE 050.

       ADD WS-A TO WS-B.
      * WS-B is now 300 (100 + 200). WS-A is unchanged.

       ADD WS-A WS-B TO WS-C.
      * WS-C is now 450 (100 + 300 + 50). WS-A and WS-B unchanged.
```

**Format 2 -- ADD GIVING:**

```cobol
ADD identifier-1 [identifier-2 ...] GIVING identifier-3 [ROUNDED]
    [identifier-4 [ROUNDED]] ...
    [ON SIZE ERROR imperative-statement-1]
    [NOT ON SIZE ERROR imperative-statement-2]
[END-ADD]
```

All identifiers before GIVING are summed, and the result is stored in each identifier after GIVING. None of the operands before GIVING are modified.

```cobol
       01 WS-PRICE    PIC 9(5)V99 VALUE 150.00.
       01 WS-TAX      PIC 9(3)V99 VALUE 012.75.
       01 WS-TOTAL    PIC 9(5)V99 VALUE ZEROS.

       ADD WS-PRICE WS-TAX GIVING WS-TOTAL.
      * WS-TOTAL is now 162.75. WS-PRICE and WS-TAX unchanged.
```

**Format 3 -- ADD CORRESPONDING:**

```cobol
ADD CORRESPONDING identifier-1 TO identifier-2 [ROUNDED]
    [ON SIZE ERROR imperative-statement-1]
    [NOT ON SIZE ERROR imperative-statement-2]
[END-ADD]
```

```cobol
       01 WS-MONTHLY-SALES.
          05 WS-PRODUCT-A  PIC 9(5)V99 VALUE 1000.00.
          05 WS-PRODUCT-B  PIC 9(5)V99 VALUE 2000.00.
          05 WS-PRODUCT-C  PIC 9(5)V99 VALUE 0500.00.

       01 WS-YEARLY-TOTALS.
          05 WS-PRODUCT-A  PIC 9(7)V99 VALUE ZEROS.
          05 WS-PRODUCT-B  PIC 9(7)V99 VALUE ZEROS.
          05 WS-PRODUCT-C  PIC 9(7)V99 VALUE ZEROS.
          05 WS-GRAND-TOTAL PIC 9(9)V99 VALUE ZEROS.

       ADD CORRESPONDING WS-MONTHLY-SALES
           TO WS-YEARLY-TOTALS.
      * WS-PRODUCT-A in WS-YEARLY-TOTALS = 1000.00
      * WS-PRODUCT-B in WS-YEARLY-TOTALS = 2000.00
      * WS-PRODUCT-C in WS-YEARLY-TOTALS = 0500.00
      * WS-GRAND-TOTAL is NOT changed (no matching name).
```

### SUBTRACT Statement

**Format 1 -- SUBTRACT FROM:**

```cobol
SUBTRACT identifier-1 [identifier-2 ...] FROM identifier-3 [ROUNDED]
    [identifier-4 [ROUNDED]] ...
    [ON SIZE ERROR imperative-statement-1]
    [NOT ON SIZE ERROR imperative-statement-2]
[END-SUBTRACT]
```

The sum of all identifiers before FROM is subtracted from each identifier after FROM.

```cobol
       01 WS-GROSS     PIC 9(7)V99 VALUE 5000.00.
       01 WS-DEDUCT-1  PIC 9(5)V99 VALUE 0300.00.
       01 WS-DEDUCT-2  PIC 9(5)V99 VALUE 0150.00.

       SUBTRACT WS-DEDUCT-1 WS-DEDUCT-2 FROM WS-GROSS.
      * WS-GROSS is now 4550.00 (5000.00 - 300.00 - 150.00).
```

**Format 2 -- SUBTRACT GIVING:**

```cobol
SUBTRACT identifier-1 [identifier-2 ...] FROM identifier-3
    GIVING identifier-4 [ROUNDED] [identifier-5 [ROUNDED]] ...
    [ON SIZE ERROR imperative-statement-1]
    [NOT ON SIZE ERROR imperative-statement-2]
[END-SUBTRACT]
```

```cobol
       01 WS-BALANCE   PIC S9(7)V99 VALUE +5000.00.
       01 WS-PAYMENT   PIC 9(5)V99  VALUE 01200.00.
       01 WS-NEW-BAL   PIC S9(7)V99 VALUE ZEROS.

       SUBTRACT WS-PAYMENT FROM WS-BALANCE
           GIVING WS-NEW-BAL.
      * WS-NEW-BAL = +3800.00. WS-BALANCE is unchanged.
```

### MULTIPLY Statement

**Format 1 -- MULTIPLY BY:**

```cobol
MULTIPLY identifier-1 BY identifier-2 [ROUNDED]
    [identifier-3 [ROUNDED]] ...
    [ON SIZE ERROR imperative-statement-1]
    [NOT ON SIZE ERROR imperative-statement-2]
[END-MULTIPLY]
```

The product of identifier-1 and identifier-2 replaces identifier-2 (and optionally identifier-3, etc.).

```cobol
       01 WS-QUANTITY  PIC 9(5)   VALUE 00025.
       01 WS-PRICE     PIC 9(5)V99 VALUE 019.99.

       MULTIPLY WS-QUANTITY BY WS-PRICE.
      * WS-PRICE is now 499.75 (25 * 19.99).
      * WS-QUANTITY is unchanged.
```

**Format 2 -- MULTIPLY GIVING:**

```cobol
MULTIPLY identifier-1 BY identifier-2
    GIVING identifier-3 [ROUNDED] [identifier-4 [ROUNDED]] ...
    [ON SIZE ERROR imperative-statement-1]
    [NOT ON SIZE ERROR imperative-statement-2]
[END-MULTIPLY]
```

```cobol
       01 WS-QUANTITY  PIC 9(5)    VALUE 00025.
       01 WS-PRICE     PIC 9(5)V99 VALUE 019.99.
       01 WS-TOTAL     PIC 9(7)V99 VALUE ZEROS.

       MULTIPLY WS-QUANTITY BY WS-PRICE
           GIVING WS-TOTAL.
      * WS-TOTAL = 0000499.75. WS-QUANTITY and WS-PRICE unchanged.
```

### DIVIDE Statement

**Format 1 -- DIVIDE INTO:**

```cobol
DIVIDE identifier-1 INTO identifier-2 [ROUNDED]
    [ON SIZE ERROR imperative-statement-1]
    [NOT ON SIZE ERROR imperative-statement-2]
[END-DIVIDE]
```

Identifier-2 is divided by identifier-1, and the quotient replaces identifier-2.

```cobol
       01 WS-TOTAL     PIC 9(7)V99 VALUE 1000.00.
       01 WS-COUNT     PIC 9(3)    VALUE 004.

       DIVIDE WS-COUNT INTO WS-TOTAL.
      * WS-TOTAL is now 0000250.00 (1000.00 / 4).
```

**Format 2 -- DIVIDE INTO GIVING:**

```cobol
DIVIDE identifier-1 INTO identifier-2
    GIVING identifier-3 [ROUNDED] [identifier-4 [ROUNDED]] ...
    [ON SIZE ERROR imperative-statement-1]
    [NOT ON SIZE ERROR imperative-statement-2]
[END-DIVIDE]
```

```cobol
       01 WS-TOTAL     PIC 9(7)V99 VALUE 1000.00.
       01 WS-COUNT     PIC 9(3)    VALUE 003.
       01 WS-AVERAGE   PIC 9(5)V99 VALUE ZEROS.

       DIVIDE WS-COUNT INTO WS-TOTAL
           GIVING WS-AVERAGE.
      * WS-AVERAGE = 00333.33 (1000.00 / 3, truncated).
      * WS-TOTAL and WS-COUNT unchanged.
```

**Format 3 -- DIVIDE BY GIVING:**

```cobol
DIVIDE identifier-1 BY identifier-2
    GIVING identifier-3 [ROUNDED] [identifier-4 [ROUNDED]] ...
    [ON SIZE ERROR imperative-statement-1]
    [NOT ON SIZE ERROR imperative-statement-2]
[END-DIVIDE]
```

This reads more naturally: "divide A by B giving C" means C = A / B.

```cobol
       DIVIDE WS-TOTAL BY WS-COUNT
           GIVING WS-AVERAGE.
      * WS-AVERAGE = WS-TOTAL / WS-COUNT
```

**Format 4 -- DIVIDE with REMAINDER:**

```cobol
DIVIDE identifier-1 INTO identifier-2
    GIVING identifier-3 [ROUNDED]
    REMAINDER identifier-4
    [ON SIZE ERROR imperative-statement-1]
    [NOT ON SIZE ERROR imperative-statement-2]
[END-DIVIDE]

DIVIDE identifier-1 BY identifier-2
    GIVING identifier-3 [ROUNDED]
    REMAINDER identifier-4
    [ON SIZE ERROR imperative-statement-1]
    [NOT ON SIZE ERROR imperative-statement-2]
[END-DIVIDE]
```

```cobol
       01 WS-DIVIDEND  PIC 9(5)  VALUE 00017.
       01 WS-DIVISOR   PIC 9(3)  VALUE 005.
       01 WS-QUOTIENT  PIC 9(5)  VALUE ZEROS.
       01 WS-REMAIN    PIC 9(3)  VALUE ZEROS.

       DIVIDE WS-DIVIDEND BY WS-DIVISOR
           GIVING WS-QUOTIENT
           REMAINDER WS-REMAIN.
      * WS-QUOTIENT = 00003, WS-REMAIN = 002.
      * 17 / 5 = 3 remainder 2.
```

Note: when REMAINDER is used, only one receiving identifier is allowed after GIVING.

### COMPUTE Statement

```cobol
COMPUTE identifier-1 [ROUNDED] [identifier-2 [ROUNDED]] ...
    = arithmetic-expression
    [ON SIZE ERROR imperative-statement-1]
    [NOT ON SIZE ERROR imperative-statement-2]
[END-COMPUTE]
```

The arithmetic expression may use:
- `+` addition
- `-` subtraction
- `*` multiplication
- `/` division
- `**` exponentiation
- `(` `)` parentheses
- Unary `+` and `-`
- Numeric literals and numeric identifiers

```cobol
       01 WS-A         PIC 9(3)V99 VALUE 010.00.
       01 WS-B         PIC 9(3)V99 VALUE 005.00.
       01 WS-C         PIC 9(3)V99 VALUE 002.00.
       01 WS-RESULT    PIC 9(5)V99 VALUE ZEROS.

       COMPUTE WS-RESULT = WS-A + WS-B * WS-C.
      * WS-RESULT = 10.00 + (5.00 * 2.00) = 20.00

       COMPUTE WS-RESULT = (WS-A + WS-B) * WS-C.
      * WS-RESULT = (10.00 + 5.00) * 2.00 = 30.00
```

COMPUTE with multiple receiving fields:

```cobol
       01 WS-X         PIC 9(5)V99 VALUE ZEROS.
       01 WS-Y         PIC 9(3)V99 VALUE ZEROS.

       COMPUTE WS-X ROUNDED WS-Y = 100.0 / 3.0.
      * WS-X = 00033.33 (ROUNDED), WS-Y = 033.33
```

COMPUTE with exponentiation:

```cobol
       01 WS-BASE      PIC 9(3)V99 VALUE 002.00.
       01 WS-POWER     PIC 9(1)    VALUE 3.
       01 WS-RESULT    PIC 9(5)V99 VALUE ZEROS.

       COMPUTE WS-RESULT = WS-BASE ** WS-POWER.
      * WS-RESULT = 00008.00 (2.00 ^ 3 = 8.00)
```

### ROUNDED Examples

```cobol
       01 WS-A         PIC 9(3)V99 VALUE ZEROS.
       01 WS-B         PIC 9(5)V9(4) VALUE 00100.3333.

      * Without ROUNDED:
       MOVE WS-B TO WS-A.
      * WS-A = 100.33 (truncated, .3333 -> .33)

      * With ROUNDED:
       COMPUTE WS-A ROUNDED = WS-B.
      * WS-A = 100.33 (rounded, .3333 rounds down to .33)

       MOVE 00100.3356 TO WS-B.
       COMPUTE WS-A ROUNDED = WS-B.
      * WS-A = 100.34 (rounded up because third decimal = 5)

      * Negative rounding:
       01 WS-NEG       PIC S9(3)V99 VALUE ZEROS.
       COMPUTE WS-NEG ROUNDED = -100.335.
      * WS-NEG = -100.34 (round away from zero: magnitude rounds up)
```

### ON SIZE ERROR Examples

```cobol
       01 WS-SMALL     PIC 9(3)   VALUE ZEROS.
       01 WS-BIG       PIC 9(5)   VALUE 99999.

       ADD 1 TO WS-BIG
           ON SIZE ERROR
               DISPLAY "OVERFLOW DETECTED"
               MOVE 99999 TO WS-BIG
           NOT ON SIZE ERROR
               DISPLAY "ADD SUCCESSFUL"
       END-ADD.
      * 99999 + 1 = 100000, which exceeds PIC 9(5).
      * ON SIZE ERROR fires. WS-BIG remains 99999.
```

Division by zero:

```cobol
       01 WS-NUMER     PIC 9(5)   VALUE 00100.
       01 WS-DENOM     PIC 9(3)   VALUE ZEROS.
       01 WS-RESULT    PIC 9(5)V99 VALUE ZEROS.

       DIVIDE WS-NUMER BY WS-DENOM
           GIVING WS-RESULT
           ON SIZE ERROR
               DISPLAY "DIVISION BY ZERO"
               MOVE ZEROS TO WS-RESULT
           NOT ON SIZE ERROR
               DISPLAY "DIVISION OK"
       END-DIVIDE.
      * ON SIZE ERROR fires because WS-DENOM is zero.
      * WS-RESULT retains its prior value (ZEROS in this case).
```

## Common Patterns

**Accumulating totals in a loop:**

```cobol
       01 WS-GRAND-TOTAL PIC 9(9)V99 VALUE ZEROS.
       01 WS-LINE-AMT    PIC 9(7)V99.

       PERFORM VARYING WS-IDX FROM 1 BY 1
           UNTIL WS-IDX > WS-RECORD-COUNT
           READ INPUT-FILE INTO WS-INPUT-REC
           ADD WS-LINE-AMT TO WS-GRAND-TOTAL
               ON SIZE ERROR
                   PERFORM TOTAL-OVERFLOW-HANDLER
           END-ADD
       END-PERFORM.
```

**Computing a percentage:**

```cobol
       01 WS-PART       PIC 9(7)V99 VALUE 00750.00.
       01 WS-WHOLE      PIC 9(7)V99 VALUE 03000.00.
       01 WS-PERCENT    PIC 9(3)V99 VALUE ZEROS.

       COMPUTE WS-PERCENT ROUNDED =
           (WS-PART / WS-WHOLE) * 100.
      * WS-PERCENT = 025.00 (750 / 3000 * 100 = 25.00)
```

**Compound interest calculation:**

```cobol
       01 WS-PRINCIPAL  PIC 9(9)V99  VALUE 10000.00.
       01 WS-RATE       PIC 9V9(4)   VALUE 0.0500.
       01 WS-YEARS      PIC 9(2)     VALUE 10.
       01 WS-FUTURE-VAL PIC 9(11)V99 VALUE ZEROS.

       COMPUTE WS-FUTURE-VAL ROUNDED =
           WS-PRINCIPAL * (1 + WS-RATE) ** WS-YEARS
           ON SIZE ERROR
               DISPLAY "CALCULATION OVERFLOW"
           NOT ON SIZE ERROR
               DISPLAY "FUTURE VALUE: " WS-FUTURE-VAL
       END-COMPUTE.
```

**Safe division with zero check:**

```cobol
       IF WS-DIVISOR NOT = ZEROS
           DIVIDE WS-DIVIDEND BY WS-DIVISOR
               GIVING WS-QUOTIENT ROUNDED
               REMAINDER WS-REMAIN
               ON SIZE ERROR
                   MOVE ZEROS TO WS-QUOTIENT
                   MOVE ZEROS TO WS-REMAIN
           END-DIVIDE
       ELSE
           MOVE ZEROS TO WS-QUOTIENT
           MOVE ZEROS TO WS-REMAIN
       END-IF.
```

**Using COMPUTE instead of chained verb-style statements:**

Instead of:

```cobol
       MULTIPLY WS-HOURS BY WS-RATE
           GIVING WS-GROSS-PAY.
       MULTIPLY WS-GROSS-PAY BY WS-TAX-RATE
           GIVING WS-TAX-AMT.
       SUBTRACT WS-TAX-AMT FROM WS-GROSS-PAY
           GIVING WS-NET-PAY.
```

Use:

```cobol
       COMPUTE WS-NET-PAY ROUNDED =
           WS-HOURS * WS-RATE *
           (1 - WS-TAX-RATE).
```

This is not only more concise but avoids intermediate rounding and truncation that can compound errors across multiple statements.

**Rounding pennies in financial calculations:**

```cobol
       01 WS-UNIT-PRICE  PIC 9(5)V9(4) VALUE ZEROS.
       01 WS-QTY         PIC 9(5)       VALUE ZEROS.
       01 WS-LINE-TOTAL  PIC 9(9)V99    VALUE ZEROS.

       MULTIPLY WS-UNIT-PRICE BY WS-QTY
           GIVING WS-LINE-TOTAL ROUNDED
           ON SIZE ERROR
               PERFORM LINE-TOTAL-OVERFLOW
       END-MULTIPLY.
```

**Integer division for date calculations:**

```cobol
       01 WS-TOTAL-MONTHS PIC 9(4) VALUE 0027.
       01 WS-YEARS        PIC 9(3) VALUE ZEROS.
       01 WS-MONTHS       PIC 9(2) VALUE ZEROS.

       DIVIDE WS-TOTAL-MONTHS BY 12
           GIVING WS-YEARS
           REMAINDER WS-MONTHS.
      * WS-YEARS = 002, WS-MONTHS = 03.
      * 27 months = 2 years and 3 months.
```

## Gotchas

- **Missing ON SIZE ERROR silently loses data.** When a result overflows the receiving field, the leftmost digits are silently truncated. The program continues with a wrong value and no indication of error. Always use ON SIZE ERROR on arithmetic operations where overflow is possible, especially in financial calculations.

- **Unsigned receiving fields silently discard the sign.** Storing a negative result in a `PIC 9(n)` field (no S) discards the sign without any error. For example, `SUBTRACT 500 FROM 300 GIVING WS-RESULT` where `WS-RESULT` is `PIC 9(5)` stores 00200, not -00200. Always use `PIC S9(n)` for fields that may hold negative values.

- **DIVIDE INTO reads backwards.** `DIVIDE A INTO B` means `B = B / A`, not `A / B`. This is the opposite of English intuition ("divide A into B" sounds like splitting A into B parts). Use `DIVIDE A BY B GIVING C` for clearer intent.

- **REMAINDER uses the truncated quotient, not the rounded one.** Even when ROUNDED is specified on the quotient, the remainder is computed from the integer-truncated quotient. This can produce unexpected remainder values if you assume the rounded quotient was used.

- **Decimal truncation does not trigger ON SIZE ERROR.** Only integer overflow (and division by zero) raises a size error. Losing fractional digits through truncation is silent. If precision matters, make receiving fields large enough or use ROUNDED.

- **Intermediate result overflow in COMPUTE is silent.** A complex COMPUTE expression may overflow during intermediate calculations even though the final result would fit in the receiving field. This does not trigger ON SIZE ERROR. Break complex expressions into smaller steps if intermediate overflow is a risk, or use compiler options to extend intermediate result precision.

- **CORRESPONDING matches by name, not by position or level.** ADD CORR matches elementary items by their data-name. If names do not match exactly, the items are not paired. Renaming a field in one group but not the other silently breaks the correspondence.

- **MULTIPLY and DIVIDE without GIVING modify the second operand.** `MULTIPLY A BY B` stores the result in B, destroying B's original value. This is easy to overlook when B is needed later. Use GIVING to preserve both operands.

- **Mixing binary (COMP) and packed-decimal (COMP-3) operands can reduce precision.** When operands of different internal formats are mixed in arithmetic, the compiler may convert between formats, potentially introducing rounding. In practice, most compilers handle this correctly, but the intermediate result allocation rules can differ by format.

- **Exponentiation with a zero or negative base and non-integer exponent is undefined.** COBOL does not define the behavior of `A ** B` when A is zero or negative and B is not a positive integer. Results are compiler-dependent and may cause runtime abends.

- **COMPUTE with integer exponents vs. decimal exponents.** When the exponent is a whole number, the compiler may use repeated multiplication (exact). When the exponent has decimal places, the compiler typically uses logarithmic computation (approximate). This can yield slightly different results for the same mathematical value.

- **Order of evaluation in multi-receiving-field statements.** In `ADD A TO B C D`, if a size error occurs on B, C and D are still evaluated independently. This means some fields may be updated and others not, creating an inconsistent state if not handled carefully.

- **Numeric literals in arithmetic must not have more than 18 digits (or 31 in extended mode).** Exceeding the compiler's maximum literal size causes a compilation error. Long decimal expansions (e.g., from manual calculation of pi) must be truncated to fit.

## Related Topics

- **data_types.md** -- Covers PICTURE clauses, numeric data categories (display numeric, packed-decimal, binary), USAGE clauses, and SIGN clauses, all of which directly determine how arithmetic operands are stored and how decimal alignment and precision work.
- **error_handling.md** -- Covers broader error-handling patterns including declaratives and exception procedures. ON SIZE ERROR is one form of inline error handling; that file provides the larger context of COBOL error management.
- **working_storage.md** -- Covers the WORKING-STORAGE SECTION where arithmetic operands, accumulators, and intermediate fields are typically defined. Understanding VALUE clauses, level numbers, and group/elementary item distinctions is essential context for arithmetic operations.
