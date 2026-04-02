# Intrinsic Functions

## Description
COBOL intrinsic functions are built-in functions invoked with the keyword FUNCTION that return a value without modifying their arguments. They cover mathematical operations, string manipulation, date/time processing, and numeric conversions. Reference this file as the comprehensive catalog of all standard COBOL intrinsic functions; individual functions may also appear in context in other topic files (e.g., string_handling.md, date_handling.md) but this file is the single authoritative list.

## Table of Contents
- [Core Concepts](#core-concepts)
  - [What Is an Intrinsic Function](#what-is-an-intrinsic-function)
  - [Function Invocation Syntax](#function-invocation-syntax)
  - [Function Return Types](#function-return-types)
  - [Argument Rules](#argument-rules)
- [Syntax & Examples](#syntax--examples)
  - [Mathematical Functions](#mathematical-functions)
  - [String Functions](#string-functions)
  - [Date and Time Functions](#date-and-time-functions)
  - [Numeric Conversion Functions](#numeric-conversion-functions)
  - [General (Ordinal) Functions](#general-ordinal-functions)
- [Common Patterns](#common-patterns)
  - [Chaining Functions](#chaining-functions)
  - [Using Functions in COMPUTE](#using-functions-in-compute)
  - [Inline Function Use in DISPLAY and MOVE](#inline-function-use-in-display-and-move)
  - [Date Arithmetic with Intrinsic Functions](#date-arithmetic-with-intrinsic-functions)
  - [Statistical Calculations](#statistical-calculations)
- [Gotchas](#gotchas)
- [Related Topics](#related-topics)

## Core Concepts

### What Is an Intrinsic Function

An intrinsic function is a predefined operation that accepts zero or more arguments and returns a single value. Unlike user-defined paragraphs or subprograms, intrinsic functions do not have a DATA DIVISION entry and cannot be called with CALL. They are always referenced inline using the word FUNCTION followed by the function name and an argument list in parentheses.

Intrinsic functions were introduced in the 1989 addendum to the COBOL-85 standard and became a core part of the language in subsequent standards. They are now supported by all major COBOL compilers (IBM Enterprise COBOL, Micro Focus COBOL, GnuCOBOL, Fujitsu NetCOBOL, etc.).

A function reference is treated as a temporary data item. It can appear anywhere an identifier of the corresponding category (numeric, alphanumeric, or integer) is allowed -- in MOVE, COMPUTE, IF, DISPLAY, STRING, and other statements.

### Function Invocation Syntax

The general form is:

```cobol
FUNCTION function-name (argument-1 argument-2 ...)
```

Key rules:
- The word FUNCTION is mandatory (unless the compiler supports a directive to omit it, which is non-standard).
- Arguments are separated by spaces (not commas) in standard COBOL.
- Some functions accept a variable number of arguments (e.g., MAX, MIN, SUM, MEAN).
- Some functions accept zero arguments (e.g., CURRENT-DATE, WHEN-COMPILED, RANDOM with no seed).
- The result is a temporary intermediate value; it cannot be used as a receiving field.

### Function Return Types

Every intrinsic function has a defined return type:

| Return Type   | Meaning                                            | Examples                        |
|---------------|----------------------------------------------------|---------------------------------|
| Integer       | Whole number, no decimal places                    | INTEGER, INTEGER-PART, MOD, LENGTH, ORD, FACTORIAL |
| Numeric       | Number that may have decimal places                | SQRT, LOG, MEAN, ANNUITY, RANDOM |
| Alphanumeric  | Character string                                   | UPPER-CASE, LOWER-CASE, REVERSE, TRIM, CHAR, CURRENT-DATE, CONCATENATE |

The return type determines where the function can be used. An alphanumeric function cannot appear in a COMPUTE statement; a numeric function cannot appear where an alphanumeric identifier is expected.

### Argument Rules

- Arguments can be identifiers, literals, arithmetic expressions, or other function references.
- When a function accepts a variable number of arguments (like MAX, SUM, MEAN), all arguments must be of the same category (all numeric or all alphanumeric, depending on the function).
- Table elements with ALL subscripts can supply all occurrences as arguments: `FUNCTION SUM(WS-AMOUNT(ALL))`.
- Arguments are not modified by the function; they are input only.

## Syntax & Examples

### Mathematical Functions

#### FUNCTION MOD

Returns the value of argument-1 modulo argument-2. The result is always an integer.

```cobol
COMPUTE WS-REMAINDER = FUNCTION MOD(WS-DIVIDEND, WS-DIVISOR)
```

If WS-DIVIDEND is 17 and WS-DIVISOR is 5, the result is 2. The mathematical definition is: `MOD(a, b) = a - (b * INTEGER(a / b))`. The result has the same sign as argument-2.

#### FUNCTION REM

Returns the remainder of argument-1 divided by argument-2. Unlike MOD, REM can return a non-integer result if the arguments have decimal places. The result has the same sign as argument-1.

```cobol
COMPUTE WS-RESULT = FUNCTION REM(17.5, 5.0)
*> Result is 2.5
```

#### FUNCTION INTEGER

Returns the greatest integer not greater than the argument (i.e., floor function).

```cobol
COMPUTE WS-FLOOR = FUNCTION INTEGER(3.7)
*> Result is 3
COMPUTE WS-FLOOR = FUNCTION INTEGER(-3.7)
*> Result is -4
```

#### FUNCTION INTEGER-PART

Returns the integer portion of the argument by truncating toward zero.

```cobol
COMPUTE WS-TRUNC = FUNCTION INTEGER-PART(3.7)
*> Result is 3
COMPUTE WS-TRUNC = FUNCTION INTEGER-PART(-3.7)
*> Result is -3
```

The difference between INTEGER and INTEGER-PART matters for negative numbers: INTEGER(-3.7) = -4, but INTEGER-PART(-3.7) = -3.

#### FUNCTION FACTORIAL

Returns the factorial of the argument. The argument must be a non-negative integer.

```cobol
COMPUTE WS-FACT = FUNCTION FACTORIAL(5)
*> Result is 120
COMPUTE WS-FACT = FUNCTION FACTORIAL(0)
*> Result is 1
```

#### FUNCTION RANDOM

Returns a pseudo-random number between 0 (inclusive) and 1 (exclusive). An optional integer seed argument initializes the sequence.

```cobol
COMPUTE WS-SEED-RESULT = FUNCTION RANDOM(12345)
COMPUTE WS-NEXT = FUNCTION RANDOM
COMPUTE WS-NEXT = FUNCTION RANDOM
```

The first call with an argument seeds the generator. Subsequent calls without an argument return the next number in the sequence. Calling with the same seed restarts the same sequence.

#### FUNCTION MAX and FUNCTION MIN

Return the maximum or minimum of their arguments. Accept a variable number of arguments. Work with both numeric and alphanumeric arguments (but all arguments must be the same category).

```cobol
COMPUTE WS-BIGGEST = FUNCTION MAX(10 20 5 30 15)
*> Result is 30
COMPUTE WS-SMALLEST = FUNCTION MIN(10 20 5 30 15)
*> Result is 5
```

With alphanumeric arguments, comparison follows the collating sequence:

```cobol
MOVE FUNCTION MAX("ALPHA" "BRAVO" "CHARLIE")
    TO WS-RESULT
*> Result is "CHARLIE"
```

With table elements:

```cobol
COMPUTE WS-HIGHEST-SCORE =
    FUNCTION MAX(WS-SCORE(ALL))
```

#### FUNCTION MEAN

Returns the arithmetic mean (average) of its arguments. The result is numeric (may have decimal places).

```cobol
COMPUTE WS-AVERAGE = FUNCTION MEAN(80 90 70 85)
*> Result is 81.25
```

#### FUNCTION MEDIAN

Returns the median value of its arguments. If the number of arguments is even, the result is the mean of the two middle values after sorting.

```cobol
COMPUTE WS-MID = FUNCTION MEDIAN(10 30 20 40 50)
*> Result is 30
COMPUTE WS-MID = FUNCTION MEDIAN(10 30 20 40)
*> Result is 25 (mean of 20 and 30)
```

#### FUNCTION RANGE

Returns the difference between the maximum and minimum of its arguments (MAX minus MIN).

```cobol
COMPUTE WS-SPREAD = FUNCTION RANGE(10 30 20 40 50)
*> Result is 40  (50 - 10)
```

#### FUNCTION SUM

Returns the sum of its arguments. Accepts a variable number of numeric arguments.

```cobol
COMPUTE WS-TOTAL = FUNCTION SUM(100 200 300)
*> Result is 600
```

Commonly used with ALL subscript for table totals:

```cobol
COMPUTE WS-GRAND-TOTAL =
    FUNCTION SUM(WS-LINE-AMOUNT(ALL))
```

#### FUNCTION SQRT

Returns the square root of the argument. The argument must be zero or positive.

```cobol
COMPUTE WS-ROOT = FUNCTION SQRT(144)
*> Result is 12
```

#### FUNCTION LOG

Returns the natural logarithm (base e) of the argument. The argument must be positive.

```cobol
COMPUTE WS-LN = FUNCTION LOG(2.71828)
*> Result is approximately 1.0
```

#### FUNCTION LOG10

Returns the logarithm base 10 of the argument. The argument must be positive.

```cobol
COMPUTE WS-LOG = FUNCTION LOG10(1000)
*> Result is 3
```

### String Functions

#### FUNCTION TRIM

Removes leading spaces, trailing spaces, or both from an alphanumeric argument.

```cobol
MOVE FUNCTION TRIM(WS-NAME) TO WS-TRIMMED
*> Removes both leading and trailing spaces (default)

MOVE FUNCTION TRIM(WS-NAME LEADING) TO WS-TRIMMED
*> Removes leading spaces only

MOVE FUNCTION TRIM(WS-NAME TRAILING) TO WS-TRIMMED
*> Removes trailing spaces only
```

The default (no qualifier) trims both ends.

#### FUNCTION LENGTH

Returns the length (in character positions) of the argument. Returns an integer.

```cobol
COMPUTE WS-LEN = FUNCTION LENGTH(WS-NAME)
*> If WS-NAME is PIC X(30), result is 30

COMPUTE WS-LEN = FUNCTION LENGTH("HELLO")
*> Result is 5

COMPUTE WS-LEN =
    FUNCTION LENGTH(FUNCTION TRIM(WS-NAME))
*> Returns length after trimming
```

Note: LENGTH always returns the defined length of a field, not the length of meaningful content. Use TRIM inside LENGTH to get the trimmed length.

#### FUNCTION REVERSE

Returns the argument string with characters in reverse order.

```cobol
MOVE FUNCTION REVERSE("ABCDE") TO WS-RESULT
*> Result is "EDCBA"
```

Because COBOL fields are fixed-length and space-padded, reversing a field like "HELLO     " (PIC X(10)) yields "     OLLEH". Trim first if this is not desired.

#### FUNCTION UPPER-CASE

Returns the argument with all lowercase letters converted to uppercase.

```cobol
MOVE FUNCTION UPPER-CASE(WS-INPUT) TO WS-OUTPUT
*> "hello world" becomes "HELLO WORLD"
```

#### FUNCTION LOWER-CASE

Returns the argument with all uppercase letters converted to lowercase.

```cobol
MOVE FUNCTION LOWER-CASE(WS-INPUT) TO WS-OUTPUT
*> "HELLO WORLD" becomes "hello world"
```

#### FUNCTION ORD

Returns the ordinal position of the first character of the argument in the current program collating sequence. The result is an integer. For EBCDIC, the letter "A" returns 194 (X'C1' + 1). For ASCII, "A" returns 66 (65 + 1).

```cobol
COMPUTE WS-POS = FUNCTION ORD("A")
```

Note: ORD returns the position plus 1 (one-based ordinal), not the raw code point value.

#### FUNCTION CHAR

Returns the character at the specified ordinal position in the collating sequence. This is the inverse of ORD.

```cobol
MOVE FUNCTION CHAR(194) TO WS-LETTER
*> On EBCDIC, result is "A"
```

#### FUNCTION CONCATENATE

Joins two or more alphanumeric arguments end to end into a single string.

```cobol
MOVE FUNCTION CONCATENATE(
    FUNCTION TRIM(WS-FIRST-NAME)
    " "
    FUNCTION TRIM(WS-LAST-NAME))
    TO WS-FULL-NAME
```

This is often more convenient than the STRING statement for simple concatenation because it does not require DELIMITED BY or pointer management. However, the result can exceed the receiving field length and will be truncated on the right if the target is too short.

### Date and Time Functions

#### FUNCTION CURRENT-DATE

Returns a 21-character alphanumeric value representing the current date, time, and offset from UTC. Takes no arguments.

```cobol
MOVE FUNCTION CURRENT-DATE TO WS-DATETIME
*> WS-DATETIME contains: YYYYMMDDHHMMSSssOHHMM
*> Positions  1- 4: Four-digit year (e.g., 2025)
*> Positions  5- 6: Month (01-12)
*> Positions  7- 8: Day (01-31)
*> Positions  9-10: Hours (00-23)
*> Positions 11-12: Minutes (00-59)
*> Positions 13-14: Seconds (00-59)
*> Positions 15-16: Hundredths of seconds (00-99)
*> Position  17:    Sign of UTC offset (+ or - or 0)
*> Positions 18-19: UTC offset hours (00-23)
*> Positions 20-21: UTC offset minutes (00-59)
```

#### FUNCTION WHEN-COMPILED

Returns a 21-character string in the same format as CURRENT-DATE but representing the date and time the program was compiled rather than the current runtime date.

```cobol
MOVE FUNCTION WHEN-COMPILED TO WS-COMPILE-STAMP
```

#### FUNCTION INTEGER-OF-DATE

Converts a Gregorian date in YYYYMMDD integer format to an integer day count (the number of days since a fixed starting point, December 31, 1600). This enables date arithmetic.

```cobol
COMPUTE WS-INT-DATE =
    FUNCTION INTEGER-OF-DATE(20250415)
```

#### FUNCTION DATE-OF-INTEGER

The inverse of INTEGER-OF-DATE. Converts an integer day number back to YYYYMMDD format.

```cobol
COMPUTE WS-GREGORIAN =
    FUNCTION DATE-OF-INTEGER(WS-INT-DATE)
```

#### FUNCTION INTEGER-OF-DAY

Converts a Julian date in YYYYDDD format (year and day-of-year) to an integer day count.

```cobol
COMPUTE WS-INT-DATE =
    FUNCTION INTEGER-OF-DAY(2025105)
*> Day 105 of 2025
```

#### FUNCTION DAY-OF-INTEGER

The inverse of INTEGER-OF-DAY. Converts an integer day number to YYYYDDD Julian format.

```cobol
COMPUTE WS-JULIAN =
    FUNCTION DAY-OF-INTEGER(WS-INT-DATE)
```

### Numeric Conversion Functions

#### FUNCTION NUMVAL

Converts an alphanumeric string representing a number into an actual numeric value. The string may contain leading or trailing spaces, a leading or trailing sign (+ or -), and a decimal point.

```cobol
COMPUTE WS-AMOUNT = FUNCTION NUMVAL(WS-INPUT-STRING)
*> "  -123.45  " becomes -123.45
```

Valid input formats include: "123", "+123", "-123", "123.45", "123.45-", " 123 ".

#### FUNCTION NUMVAL-C

Like NUMVAL but also handles currency signs and commas (or other editing characters). The optional second argument specifies the currency character.

```cobol
COMPUTE WS-AMOUNT = FUNCTION NUMVAL-C(WS-INPUT-STRING)
*> "$1,234.56" becomes 1234.56

COMPUTE WS-AMOUNT =
    FUNCTION NUMVAL-C(WS-INPUT-STRING "£")
*> "£1,234.56" becomes 1234.56
```

#### FUNCTION ANNUITY

Returns the ratio of an annuity paid at the end of each period for a given number of periods at a given interest rate. Useful for loan amortization calculations.

```cobol
COMPUTE WS-PAYMENT =
    WS-PRINCIPAL * FUNCTION ANNUITY(WS-RATE, WS-PERIODS)
```

Arguments: argument-1 is the interest rate per period (0.005 for 0.5%), argument-2 is the number of periods. If the interest rate is zero, the result is `1 / number-of-periods`.

#### FUNCTION PRESENT-VALUE

Returns the present value of a series of future period-end amounts at a given interest rate.

```cobol
COMPUTE WS-PV =
    FUNCTION PRESENT-VALUE(0.05, 1000 1000 1000 1000 1000)
*> Present value of five annual payments of 1000 at 5%
```

The first argument is the discount rate. The remaining arguments are the future amounts.

### General (Ordinal) Functions

#### FUNCTION ORD-MAX

Returns the ordinal position (1-based) of the argument that has the maximum value. Useful when you need to know which argument is the largest rather than the value itself.

```cobol
COMPUTE WS-POSITION =
    FUNCTION ORD-MAX(WS-SCORE-1 WS-SCORE-2 WS-SCORE-3)
*> If scores are 80, 95, 70, result is 2 (second argument)
```

#### FUNCTION ORD-MIN

Returns the ordinal position (1-based) of the argument that has the minimum value.

```cobol
COMPUTE WS-POSITION =
    FUNCTION ORD-MIN(WS-SCORE-1 WS-SCORE-2 WS-SCORE-3)
*> If scores are 80, 95, 70, result is 3 (third argument)
```

## Common Patterns

### Chaining Functions

Intrinsic functions can be nested as arguments to other functions. This is a common idiom for combining operations without temporary variables.

```cobol
COMPUTE WS-LEN =
    FUNCTION LENGTH(FUNCTION TRIM(WS-NAME))

MOVE FUNCTION UPPER-CASE(FUNCTION TRIM(WS-INPUT))
    TO WS-NORMALIZED
```

### Using Functions in COMPUTE

Numeric functions integrate naturally with the COMPUTE statement and can be combined with arithmetic operators:

```cobol
COMPUTE WS-HYPOTENUSE =
    FUNCTION SQRT(WS-SIDE-A ** 2 + WS-SIDE-B ** 2)

COMPUTE WS-PAYMENT =
    WS-LOAN-AMOUNT * FUNCTION ANNUITY(
        WS-MONTHLY-RATE, WS-NUM-MONTHS)

COMPUTE WS-LOG-BASE-2 =
    FUNCTION LOG(WS-VALUE) / FUNCTION LOG(2)
```

### Inline Function Use in DISPLAY and MOVE

Functions can appear directly in DISPLAY, MOVE, STRING, and other statements:

```cobol
DISPLAY "TODAY: " FUNCTION CURRENT-DATE
DISPLAY "LENGTH: " FUNCTION LENGTH(WS-RECORD)

MOVE FUNCTION CONCATENATE(
    FUNCTION TRIM(WS-FIRST)
    " "
    FUNCTION TRIM(WS-LAST))
    TO WS-FULL-NAME

IF FUNCTION UPPER-CASE(WS-RESPONSE) = "YES"
    PERFORM PROCESS-YES
END-IF
```

### Date Arithmetic with Intrinsic Functions

A very common pattern is to find the number of days between two dates or to add/subtract days from a date:

```cobol
*> Days between two dates
COMPUTE WS-DAYS-BETWEEN =
    FUNCTION INTEGER-OF-DATE(WS-END-DATE)
  - FUNCTION INTEGER-OF-DATE(WS-START-DATE)

*> Add 30 days to a date
COMPUTE WS-NEW-DATE =
    FUNCTION DATE-OF-INTEGER(
        FUNCTION INTEGER-OF-DATE(WS-BASE-DATE)
        + 30)

*> Convert Gregorian to Julian
COMPUTE WS-JULIAN =
    FUNCTION DAY-OF-INTEGER(
        FUNCTION INTEGER-OF-DATE(WS-GREG-DATE))
```

### Statistical Calculations

For batch programs that accumulate data in tables, the statistical functions eliminate manual loop-based calculations:

```cobol
01  WS-SCORES.
    05  WS-SCORE       PIC 9(3) OCCURS 100.

COMPUTE WS-AVERAGE = FUNCTION MEAN(WS-SCORE(ALL))
COMPUTE WS-HIGH    = FUNCTION MAX(WS-SCORE(ALL))
COMPUTE WS-LOW     = FUNCTION MIN(WS-SCORE(ALL))
COMPUTE WS-TOTAL   = FUNCTION SUM(WS-SCORE(ALL))
COMPUTE WS-SPREAD  = FUNCTION RANGE(WS-SCORE(ALL))
COMPUTE WS-MIDDLE  = FUNCTION MEDIAN(WS-SCORE(ALL))
```

## Gotchas

- **FUNCTION keyword is mandatory.** Writing `COMPUTE X = MAX(A B)` without the FUNCTION keyword is a syntax error. Every intrinsic function call must be prefixed with the word FUNCTION.

- **LENGTH returns the declared length, not the content length.** `FUNCTION LENGTH(WS-NAME)` where WS-NAME is PIC X(50) always returns 50, even if the meaningful data is "JOE" followed by 47 spaces. Use `FUNCTION LENGTH(FUNCTION TRIM(WS-NAME))` to get the trimmed length.

- **REVERSE includes trailing spaces.** Reversing "HELLO     " (PIC X(10)) yields "     OLLEH", not "OLLEH". Trim first: `FUNCTION REVERSE(FUNCTION TRIM(WS-NAME))`.

- **RANDOM is not cryptographically secure.** The RANDOM function is a simple pseudo-random generator. It should never be used for security-sensitive purposes. The sequence is deterministic given the same seed.

- **RANDOM with no argument is not the first call.** The first call to RANDOM in a program should include a seed argument. Calling RANDOM with no argument before seeding yields compiler/implementation-dependent behavior.

- **NUMVAL and NUMVAL-C will abend on invalid input.** If the input string contains non-numeric characters that do not match the expected format (e.g., alphabetic characters in a NUMVAL argument), the program will typically abend. Always validate input before calling these functions.

- **INTEGER vs. INTEGER-PART for negative numbers.** INTEGER(-3.2) returns -4 (floor), while INTEGER-PART(-3.2) returns -3 (truncation toward zero). Confusing these can cause off-by-one errors in calculations.

- **CURRENT-DATE format varies by position, not by delimiter.** The 21-character result is purely positional with no separating characters. Programmers must use reference modification or a group-level REDEFINES to extract individual components.

- **ORD returns ordinal position, not the code point.** FUNCTION ORD("A") returns 194 on EBCDIC systems (X'C1' = 193, plus 1 for one-based ordinal), not 193. CHAR and ORD are inverses, but the one-based indexing can confuse programmers expecting zero-based values.

- **ALL subscript requires identical OCCURS items.** When using `FUNCTION SUM(WS-AMOUNT(ALL))`, the item must be a table element with an OCCURS clause. Using ALL with a non-table item causes a compile error.

- **Alphanumeric MAX/MIN uses collating sequence.** When comparing alphanumeric arguments, MAX and MIN use the program's collating sequence (EBCDIC or ASCII), which can yield different results on different platforms. On EBCDIC, lowercase letters collate lower than uppercase.

- **CONCATENATE result length.** The result of CONCATENATE is the sum of the lengths of all arguments. If the receiving field is shorter, the result is truncated on the right with no warning. If longer, it is space-padded on the right.

## Related Topics

- **[string_handling.md](string_handling.md)** -- Covers STRING, UNSTRING, INSPECT, and reference modification. The string intrinsic functions (TRIM, LENGTH, UPPER-CASE, LOWER-CASE) are introduced there in context but documented comprehensively here.
- **[date_handling.md](date_handling.md)** -- Covers date processing patterns including detailed usage of CURRENT-DATE, INTEGER-OF-DATE, DATE-OF-INTEGER, and the Julian conversion functions. This file lists those functions for completeness; date_handling.md provides the full workflow context.
- **[arithmetic.md](arithmetic.md)** -- Covers COMPUTE, ADD, SUBTRACT, MULTIPLY, DIVIDE. Intrinsic functions are commonly used within COMPUTE expressions.
- **[data_types.md](data_types.md)** -- Understanding PIC clauses and USAGE is essential for knowing what types of arguments functions accept and what happens when function results are moved to receiving fields.
- **[io_processing.md](io_processing.md)** -- NUMVAL and NUMVAL-C are frequently used when processing external input data that arrives as alphanumeric strings.
- **[table_handling.md](table_handling.md)** -- The ALL subscript notation used with functions like SUM, MAX, MIN, and MEAN is part of table handling syntax.
