# Date Handling

## Description
This file covers all aspects of date processing in COBOL: accepting system date/time values, using intrinsic functions for date arithmetic and conversion, handling common mainframe date formats (Gregorian and Julian), century windowing and Y2K considerations, leap year logic, and date validation patterns. Reference this file whenever a COBOL program needs to work with dates, calculate date differences, convert between date formats, or validate date input.

## Table of Contents
- [Core Concepts](#core-concepts)
  - [Date Representation in COBOL](#date-representation-in-cobol)
  - [Common Mainframe Date Formats](#common-mainframe-date-formats)
  - [ACCEPT FROM DATE, DAY, and TIME](#accept-from-date-day-and-time)
  - [FUNCTION CURRENT-DATE](#function-current-date)
  - [The Integer Day Number System](#the-integer-day-number-system)
  - [Century Windowing and Y2K](#century-windowing-and-y2k)
  - [Leap Year Rules](#leap-year-rules)
- [Syntax & Examples](#syntax--examples)
  - [Accepting System Date and Time](#accepting-system-date-and-time)
  - [FUNCTION CURRENT-DATE Detailed Breakdown](#function-current-date-detailed-breakdown)
  - [Gregorian Date Arithmetic](#gregorian-date-arithmetic)
  - [Julian Date Conversions](#julian-date-conversions)
  - [Converting Between Gregorian and Julian](#converting-between-gregorian-and-julian)
  - [Date Comparison](#date-comparison)
  - [Date Formatting for Output](#date-formatting-for-output)
- [Common Patterns](#common-patterns)
  - [Getting Today's Date](#getting-todays-date)
  - [Adding or Subtracting Days](#adding-or-subtracting-days)
  - [Calculating Days Between Dates](#calculating-days-between-dates)
  - [Finding the Day of the Week](#finding-the-day-of-the-week)
  - [End-of-Month Calculation](#end-of-month-calculation)
  - [Date Validation](#date-validation)
  - [Leap Year Check](#leap-year-check)
  - [Century Windowing Pattern](#century-windowing-pattern)
  - [Batch Processing Date Ranges](#batch-processing-date-ranges)
- [Gotchas](#gotchas)
- [Related Topics](#related-topics)

## Core Concepts

### Date Representation in COBOL

COBOL has no native date data type. Dates are stored as numeric or alphanumeric fields, and the programmer is responsible for interpreting the positions within those fields as year, month, day, and so on. This makes date handling both flexible and error-prone.

Typical date storage uses simple PIC clauses:

```cobol
01  WS-GREG-DATE       PIC 9(8).    *> YYYYMMDD
01  WS-JULIAN-DATE     PIC 9(7).    *> YYYYDDD
01  WS-SHORT-DATE      PIC 9(6).    *> YYMMDD
01  WS-DATE-TEXT       PIC X(10).   *> MM/DD/YYYY
```

Because dates are just numbers or strings, all validation, arithmetic, and conversion must be explicitly coded or done using intrinsic functions.

### Common Mainframe Date Formats

These are the date formats most frequently encountered in mainframe COBOL systems:

| Format     | PIC       | Example     | Description                              |
|------------|-----------|-------------|------------------------------------------|
| YYYYMMDD   | 9(8)      | 20250415    | ISO-style Gregorian, 4-digit year        |
| YYMMDD     | 9(6)      | 250415      | Gregorian, 2-digit year (Y2K risk)       |
| YYYYDDD    | 9(7)      | 2025105     | Julian (ordinal day of year), 4-digit yr |
| YYDDD      | 9(5)      | 25105       | Julian, 2-digit year (Y2K risk)          |
| MMDDYYYY   | 9(8)      | 04152025    | US-style Gregorian                       |
| DDMMYYYY   | 9(8)      | 15042025    | European-style Gregorian                 |
| CYYMMDD    | 9(7)      | 1250415     | Century byte + YYMMDD (C=0 for 19xx, C=1 for 20xx) |
| MM/DD/YYYY | X(10)     | 04/15/2025  | Edited display format                    |

The YYYYMMDD format is strongly preferred for new development because it is unambiguous, Y2K-safe, and sorts naturally as a numeric value. The CYYMMDD format is common in IBM mainframe shops as a compact Y2K-safe format.

### ACCEPT FROM DATE, DAY, and TIME

The ACCEPT statement can retrieve the current system date and time using special register-like formats:

```cobol
ACCEPT WS-DATE   FROM DATE          *> YYMMDD (6 digits)
ACCEPT WS-DATE8  FROM DATE YYYYMMDD *> YYYYMMDD (8 digits)
ACCEPT WS-DAY    FROM DAY           *> YYDDD (5 digits)
ACCEPT WS-DAY8   FROM DAY YYYYDDD  *> YYYYDDD (7 digits)
ACCEPT WS-TIME   FROM TIME          *> HHMMSSHS (8 digits)
ACCEPT WS-DOW    FROM DAY-OF-WEEK   *> 1 digit (1=Monday ... 7=Sunday)
```

The YYYYMMDD and YYYYDDD variants were added specifically to address Y2K concerns. The older YYMMDD and YYDDD variants are still supported but should be avoided in new code.

Note: ACCEPT FROM DATE retrieves the system date at the moment of execution. For batch programs that must use a consistent date throughout the run, capture the date once into a working storage field rather than calling ACCEPT repeatedly.

### FUNCTION CURRENT-DATE

FUNCTION CURRENT-DATE is the intrinsic function equivalent of ACCEPT FROM DATE, but it provides more information in a single call: date, time down to hundredths of a second, and the UTC offset.

The function returns a 21-character alphanumeric value:

```
Position  Content            Example
--------  -------            -------
 1 -  4   Four-digit year    2025
 5 -  6   Month (01-12)      04
 7 -  8   Day (01-31)        15
 9 - 10   Hours (00-23)      14
11 - 12   Minutes (00-59)    30
13 - 14   Seconds (00-59)    45
15 - 16   Hundredths (00-99) 12
17        UTC offset sign    - (or + or 0)
18 - 19   UTC offset hours   05
20 - 21   UTC offset minutes 00
```

A common layout for receiving this value:

```cobol
01  WS-CURRENT-DT.
    05  WS-CD-YEAR        PIC 9(4).
    05  WS-CD-MONTH       PIC 9(2).
    05  WS-CD-DAY         PIC 9(2).
    05  WS-CD-HOURS       PIC 9(2).
    05  WS-CD-MINUTES     PIC 9(2).
    05  WS-CD-SECONDS     PIC 9(2).
    05  WS-CD-HUNDREDTHS  PIC 9(2).
    05  WS-CD-UTC-SIGN    PIC X(1).
    05  WS-CD-UTC-HOURS   PIC 9(2).
    05  WS-CD-UTC-MINS    PIC 9(2).

MOVE FUNCTION CURRENT-DATE TO WS-CURRENT-DT
```

Because CURRENT-DATE returns an alphanumeric value, you must MOVE it to a group item and then reference the subordinate numeric fields for arithmetic.

### The Integer Day Number System

The COBOL intrinsic functions INTEGER-OF-DATE, DATE-OF-INTEGER, INTEGER-OF-DAY, and DAY-OF-INTEGER all work with an integer day number that represents the count of days since an epoch. In the COBOL standard, the epoch is December 31, 1600 (day 1 = January 1, 1601).

This integer representation allows date arithmetic with simple addition and subtraction:

- To find days between two dates: subtract their integer day numbers.
- To add N days to a date: convert to integer, add N, convert back.
- To convert between Gregorian and Julian: convert to integer, then convert to the other format.

```
YYYYMMDD  <-->  Integer Day Number  <-->  YYYYDDD
   |                   |                     |
   INTEGER-OF-DATE     |            INTEGER-OF-DAY
   DATE-OF-INTEGER     |            DAY-OF-INTEGER
```

### Century Windowing and Y2K

Many legacy COBOL programs store dates with 2-digit years (YYMMDD). After the Y2K transition, these programs require century windowing -- a technique that interprets 2-digit years relative to a pivot or window.

The basic idea: choose a pivot year (e.g., 40). Years 00-39 are interpreted as 2000-2039; years 40-99 are interpreted as 1940-1999.

```cobol
01  WS-2-DIGIT-YEAR    PIC 99.
01  WS-4-DIGIT-YEAR    PIC 9(4).
01  WS-PIVOT-YEAR      PIC 99 VALUE 40.

IF WS-2-DIGIT-YEAR < WS-PIVOT-YEAR
    COMPUTE WS-4-DIGIT-YEAR = 2000 + WS-2-DIGIT-YEAR
ELSE
    COMPUTE WS-4-DIGIT-YEAR = 1900 + WS-2-DIGIT-YEAR
END-IF
```

Some compilers provide built-in century windowing via compiler options (e.g., IBM's DATEPROC compiler option). The CYYMMDD format (where C is a century digit: 0 = 1900s, 1 = 2000s) was another common Y2K remediation strategy.

Modern systems should use 4-digit years exclusively. Century windowing is a maintenance pattern for legacy code, not a recommended practice for new development.

### Leap Year Rules

A year is a leap year if:
1. It is divisible by 4, AND
2. It is NOT divisible by 100, UNLESS
3. It is also divisible by 400.

So 2000 was a leap year (divisible by 400), 1900 was not (divisible by 100 but not 400), and 2024 was a leap year (divisible by 4, not by 100).

Leap years have 366 days; February has 29 days instead of 28. This affects Julian date conversions and date validation.

## Syntax & Examples

### Accepting System Date and Time

```cobol
WORKING-STORAGE SECTION.
01  WS-SYS-DATE.
    05  WS-SYS-YEAR    PIC 9(4).
    05  WS-SYS-MONTH   PIC 9(2).
    05  WS-SYS-DAY     PIC 9(2).

01  WS-SYS-JULIAN.
    05  WS-JUL-YEAR    PIC 9(4).
    05  WS-JUL-DAY     PIC 9(3).

01  WS-SYS-TIME.
    05  WS-TIME-HH     PIC 9(2).
    05  WS-TIME-MM     PIC 9(2).
    05  WS-TIME-SS     PIC 9(2).
    05  WS-TIME-HS     PIC 9(2).

01  WS-DAY-OF-WEEK     PIC 9.

PROCEDURE DIVISION.
    ACCEPT WS-SYS-DATE FROM DATE YYYYMMDD
    ACCEPT WS-SYS-JULIAN FROM DAY YYYYDDD
    ACCEPT WS-SYS-TIME FROM TIME
    ACCEPT WS-DAY-OF-WEEK FROM DAY-OF-WEEK
    *> WS-DAY-OF-WEEK: 1=Monday ... 7=Sunday
```

### FUNCTION CURRENT-DATE Detailed Breakdown

```cobol
01  WS-CURRENT-DATE-DATA.
    05  WS-CURR-DATE.
        10  WS-CURR-YEAR     PIC 9(4).
        10  WS-CURR-MONTH    PIC 9(2).
        10  WS-CURR-DAY      PIC 9(2).
    05  WS-CURR-TIME.
        10  WS-CURR-HOUR     PIC 9(2).
        10  WS-CURR-MIN      PIC 9(2).
        10  WS-CURR-SEC      PIC 9(2).
        10  WS-CURR-HSEC     PIC 9(2).
    05  WS-UTC-OFFSET.
        10  WS-UTC-SIGN      PIC X.
        10  WS-UTC-OFF-HH    PIC 9(2).
        10  WS-UTC-OFF-MM    PIC 9(2).

PROCEDURE DIVISION.
    MOVE FUNCTION CURRENT-DATE TO WS-CURRENT-DATE-DATA

    DISPLAY "Date: " WS-CURR-YEAR "/"
                     WS-CURR-MONTH "/"
                     WS-CURR-DAY
    DISPLAY "Time: " WS-CURR-HOUR ":"
                     WS-CURR-MIN ":"
                     WS-CURR-SEC
    DISPLAY "UTC offset: " WS-UTC-SIGN
                           WS-UTC-OFF-HH ":"
                           WS-UTC-OFF-MM
```

If the UTC offset is unavailable on the system, position 17 contains "0" and positions 18-21 contain "0000".

### Gregorian Date Arithmetic

Using INTEGER-OF-DATE and DATE-OF-INTEGER:

```cobol
01  WS-START-DATE      PIC 9(8) VALUE 20250415.
01  WS-END-DATE        PIC 9(8) VALUE 20250515.
01  WS-INT-START       PIC 9(9).
01  WS-INT-END         PIC 9(9).
01  WS-DAYS-BETWEEN    PIC S9(7).
01  WS-NEW-DATE        PIC 9(8).

*> Calculate days between two dates
COMPUTE WS-INT-START =
    FUNCTION INTEGER-OF-DATE(WS-START-DATE)
COMPUTE WS-INT-END =
    FUNCTION INTEGER-OF-DATE(WS-END-DATE)
COMPUTE WS-DAYS-BETWEEN =
    WS-INT-END - WS-INT-START
*> WS-DAYS-BETWEEN = 30

*> Add 90 days to a date
COMPUTE WS-NEW-DATE =
    FUNCTION DATE-OF-INTEGER(
        FUNCTION INTEGER-OF-DATE(WS-START-DATE) + 90)
*> WS-NEW-DATE = 20250714
```

### Julian Date Conversions

```cobol
01  WS-GREG-DATE       PIC 9(8) VALUE 20250415.
01  WS-JULIAN-DATE     PIC 9(7).
01  WS-INT-DATE        PIC 9(9).

*> Gregorian to Julian
COMPUTE WS-INT-DATE =
    FUNCTION INTEGER-OF-DATE(WS-GREG-DATE)
COMPUTE WS-JULIAN-DATE =
    FUNCTION DAY-OF-INTEGER(WS-INT-DATE)
*> WS-JULIAN-DATE = 2025105 (April 15 = day 105 of 2025)

*> Julian to Gregorian
COMPUTE WS-INT-DATE =
    FUNCTION INTEGER-OF-DAY(2025105)
COMPUTE WS-GREG-DATE =
    FUNCTION DATE-OF-INTEGER(WS-INT-DATE)
*> WS-GREG-DATE = 20250415
```

### Converting Between Gregorian and Julian

A single-step conversion using the integer day number as an intermediate:

```cobol
*> Gregorian to Julian (one statement)
COMPUTE WS-JULIAN-DATE =
    FUNCTION DAY-OF-INTEGER(
        FUNCTION INTEGER-OF-DATE(WS-GREG-DATE))

*> Julian to Gregorian (one statement)
COMPUTE WS-GREG-DATE =
    FUNCTION DATE-OF-INTEGER(
        FUNCTION INTEGER-OF-DAY(WS-JULIAN-DATE))
```

### Date Comparison

Because YYYYMMDD dates are 8-digit numeric values, they compare naturally:

```cobol
IF WS-START-DATE > WS-END-DATE
    DISPLAY "Start date is after end date"
END-IF

IF WS-TRANSACTION-DATE >= 20250101
    AND WS-TRANSACTION-DATE <= 20251231
    DISPLAY "Transaction is in year 2025"
END-IF
```

Julian dates (YYYYDDD) also compare correctly as numeric values because the year is the most significant portion.

YYMMDD dates do NOT compare correctly across century boundaries (e.g., 991231 > 000101 even though December 31, 1999 precedes January 1, 2000). This is one of the core Y2K problems.

### Date Formatting for Output

Converting a YYYYMMDD date to a display format like MM/DD/YYYY:

```cobol
01  WS-DATE-NUM        PIC 9(8) VALUE 20250415.
01  WS-DATE-PARTS REDEFINES WS-DATE-NUM.
    05  WS-DATE-YYYY   PIC 9(4).
    05  WS-DATE-MM     PIC 9(2).
    05  WS-DATE-DD     PIC 9(2).

01  WS-DATE-DISPLAY    PIC X(10).

STRING WS-DATE-MM    DELIMITED BY SIZE
       "/"           DELIMITED BY SIZE
       WS-DATE-DD    DELIMITED BY SIZE
       "/"           DELIMITED BY SIZE
       WS-DATE-YYYY  DELIMITED BY SIZE
       INTO WS-DATE-DISPLAY
*> WS-DATE-DISPLAY = "04/15/2025"
```

Or using reference modification:

```cobol
01  WS-DATE-8          PIC X(8) VALUE "20250415".
01  WS-FORMATTED       PIC X(10).

STRING WS-DATE-8(5:2)  DELIMITED BY SIZE
       "/"              DELIMITED BY SIZE
       WS-DATE-8(7:2)  DELIMITED BY SIZE
       "/"              DELIMITED BY SIZE
       WS-DATE-8(1:4)  DELIMITED BY SIZE
       INTO WS-FORMATTED
```

## Common Patterns

### Getting Today's Date

The two standard approaches:

```cobol
*> Approach 1: ACCEPT (simpler, just the date)
01  WS-TODAY    PIC 9(8).

ACCEPT WS-TODAY FROM DATE YYYYMMDD

*> Approach 2: FUNCTION CURRENT-DATE (date + time + UTC)
01  WS-FULL-DT.
    05  WS-TODAY-DT    PIC X(8).
    05  WS-TIME-DT     PIC X(8).
    05  WS-UTC-DT      PIC X(5).

MOVE FUNCTION CURRENT-DATE TO WS-FULL-DT
```

For batch programs, capture the date once at the start:

```cobol
PROCEDURE DIVISION.
    ACCEPT WS-RUN-DATE FROM DATE YYYYMMDD
    PERFORM PROCESS-FILE
    STOP RUN.
```

### Adding or Subtracting Days

```cobol
*> Add days
COMPUTE WS-FUTURE-DATE =
    FUNCTION DATE-OF-INTEGER(
        FUNCTION INTEGER-OF-DATE(WS-BASE-DATE)
        + WS-DAYS-TO-ADD)

*> Subtract days
COMPUTE WS-PAST-DATE =
    FUNCTION DATE-OF-INTEGER(
        FUNCTION INTEGER-OF-DATE(WS-BASE-DATE)
        - WS-DAYS-TO-SUBTRACT)

*> Add months (approximate: add 30 days per month)
*> For exact month addition, see the month-add pattern below
COMPUTE WS-APPROX-DATE =
    FUNCTION DATE-OF-INTEGER(
        FUNCTION INTEGER-OF-DATE(WS-BASE-DATE)
        + (WS-MONTHS * 30))
```

Adding exact months requires manipulating the month and year components directly:

```cobol
01  WS-BASE-DATE       PIC 9(8) VALUE 20250131.
01  WS-BASE-PARTS REDEFINES WS-BASE-DATE.
    05  WS-BASE-YYYY   PIC 9(4).
    05  WS-BASE-MM     PIC 9(2).
    05  WS-BASE-DD     PIC 9(2).

01  WS-RESULT-DATE     PIC 9(8).
01  WS-RESULT-PARTS REDEFINES WS-RESULT-DATE.
    05  WS-RES-YYYY    PIC 9(4).
    05  WS-RES-MM      PIC 9(2).
    05  WS-RES-DD      PIC 9(2).

01  WS-MONTHS-TO-ADD   PIC 9(3) VALUE 3.
01  WS-TOTAL-MONTHS    PIC 9(5).

*> Add months
COMPUTE WS-TOTAL-MONTHS =
    (WS-BASE-YYYY * 12) + WS-BASE-MM
    + WS-MONTHS-TO-ADD - 1
COMPUTE WS-RES-YYYY =
    FUNCTION INTEGER-PART(WS-TOTAL-MONTHS / 12)
COMPUTE WS-RES-MM =
    FUNCTION MOD(WS-TOTAL-MONTHS, 12) + 1
MOVE WS-BASE-DD TO WS-RES-DD
*> Adjust day if result month has fewer days
*> (e.g., Jan 31 + 1 month = Feb 28 or 29)
PERFORM ADJUST-DAY-FOR-MONTH
```

### Calculating Days Between Dates

```cobol
01  WS-DATE-1          PIC 9(8) VALUE 20250101.
01  WS-DATE-2          PIC 9(8) VALUE 20251231.
01  WS-DAYS            PIC S9(7).

COMPUTE WS-DAYS =
    FUNCTION INTEGER-OF-DATE(WS-DATE-2)
  - FUNCTION INTEGER-OF-DATE(WS-DATE-1)
*> WS-DAYS = 364 (not 365; it's the difference)
```

### Finding the Day of the Week

Using ACCEPT FROM DAY-OF-WEEK:

```cobol
01  WS-DOW             PIC 9.
    88  WS-MONDAY      VALUE 1.
    88  WS-TUESDAY     VALUE 2.
    88  WS-WEDNESDAY   VALUE 3.
    88  WS-THURSDAY    VALUE 4.
    88  WS-FRIDAY      VALUE 5.
    88  WS-SATURDAY    VALUE 6.
    88  WS-SUNDAY      VALUE 7.

ACCEPT WS-DOW FROM DAY-OF-WEEK

IF WS-SATURDAY OR WS-SUNDAY
    SET WS-IS-WEEKEND TO TRUE
ELSE
    SET WS-IS-WEEKDAY TO TRUE
END-IF
```

To find the day of the week for an arbitrary date, use the integer day number. Since the epoch (January 1, 1601) was a Monday, you can calculate:

```cobol
COMPUTE WS-DOW =
    FUNCTION MOD(
        FUNCTION INTEGER-OF-DATE(WS-ANY-DATE) - 1, 7) + 1
*> 1=Monday, 2=Tuesday, ..., 7=Sunday
```

### End-of-Month Calculation

To find the last day of a given month:

```cobol
01  WS-YEAR            PIC 9(4).
01  WS-MONTH           PIC 9(2).
01  WS-LAST-DAY        PIC 9(2).
01  WS-EOM-DATE        PIC 9(8).

*> Strategy: go to day 1 of the next month, subtract 1 day
IF WS-MONTH = 12
    COMPUTE WS-EOM-DATE =
        FUNCTION DATE-OF-INTEGER(
            FUNCTION INTEGER-OF-DATE(
                (WS-YEAR + 1) * 10000 + 0101) - 1)
ELSE
    COMPUTE WS-EOM-DATE =
        FUNCTION DATE-OF-INTEGER(
            FUNCTION INTEGER-OF-DATE(
                WS-YEAR * 10000 +
                (WS-MONTH + 1) * 100 + 01) - 1)
END-IF
```

A simpler table-based approach:

```cobol
01  WS-DAYS-IN-MONTH-TABLE.
    05  FILLER          PIC 9(2) VALUE 31.  *> Jan
    05  FILLER          PIC 9(2) VALUE 28.  *> Feb (non-leap)
    05  FILLER          PIC 9(2) VALUE 31.  *> Mar
    05  FILLER          PIC 9(2) VALUE 30.  *> Apr
    05  FILLER          PIC 9(2) VALUE 31.  *> May
    05  FILLER          PIC 9(2) VALUE 30.  *> Jun
    05  FILLER          PIC 9(2) VALUE 31.  *> Jul
    05  FILLER          PIC 9(2) VALUE 31.  *> Aug
    05  FILLER          PIC 9(2) VALUE 30.  *> Sep
    05  FILLER          PIC 9(2) VALUE 31.  *> Oct
    05  FILLER          PIC 9(2) VALUE 30.  *> Nov
    05  FILLER          PIC 9(2) VALUE 31.  *> Dec

01  WS-DAYS-TABLE REDEFINES WS-DAYS-IN-MONTH-TABLE.
    05  WS-DAYS-IN-MONTH PIC 9(2) OCCURS 12.

MOVE WS-DAYS-IN-MONTH(WS-MONTH) TO WS-LAST-DAY

*> Adjust for leap year in February
IF WS-MONTH = 2
    PERFORM CHECK-LEAP-YEAR
    IF WS-IS-LEAP
        ADD 1 TO WS-LAST-DAY
    END-IF
END-IF
```

### Date Validation

A comprehensive date validation routine:

```cobol
01  WS-INPUT-DATE      PIC 9(8).
01  WS-INPUT-PARTS REDEFINES WS-INPUT-DATE.
    05  WS-IN-YYYY     PIC 9(4).
    05  WS-IN-MM       PIC 9(2).
    05  WS-IN-DD       PIC 9(2).

01  WS-DATE-VALID      PIC X VALUE "N".
    88  DATE-IS-VALID   VALUE "Y".
    88  DATE-IS-INVALID VALUE "N".

01  WS-MAX-DAY         PIC 9(2).
01  WS-LEAP-FLAG       PIC X VALUE "N".
    88  WS-IS-LEAP     VALUE "Y".

PERFORM VALIDATE-DATE

VALIDATE-DATE.
    SET DATE-IS-INVALID TO TRUE

    *> Check year range
    IF WS-IN-YYYY < 1601 OR WS-IN-YYYY > 9999
        EXIT PARAGRAPH
    END-IF

    *> Check month range
    IF WS-IN-MM < 1 OR WS-IN-MM > 12
        EXIT PARAGRAPH
    END-IF

    *> Determine max days for the month
    EVALUATE WS-IN-MM
        WHEN 1 WHEN 3 WHEN 5 WHEN 7
        WHEN 8 WHEN 10 WHEN 12
            MOVE 31 TO WS-MAX-DAY
        WHEN 4 WHEN 6 WHEN 9 WHEN 11
            MOVE 30 TO WS-MAX-DAY
        WHEN 2
            PERFORM CHECK-LEAP-YEAR
            IF WS-IS-LEAP
                MOVE 29 TO WS-MAX-DAY
            ELSE
                MOVE 28 TO WS-MAX-DAY
            END-IF
    END-EVALUATE

    *> Check day range
    IF WS-IN-DD >= 1 AND WS-IN-DD <= WS-MAX-DAY
        SET DATE-IS-VALID TO TRUE
    END-IF
    .
```

An alternative validation using intrinsic functions -- if INTEGER-OF-DATE does not abend on invalid input (compiler-dependent), you can use a round-trip check:

```cobol
*> Round-trip validation: convert to integer and back
*> If the result matches the input, the date is valid
COMPUTE WS-TEST-DATE =
    FUNCTION DATE-OF-INTEGER(
        FUNCTION INTEGER-OF-DATE(WS-INPUT-DATE))

IF WS-TEST-DATE = WS-INPUT-DATE
    SET DATE-IS-VALID TO TRUE
ELSE
    SET DATE-IS-INVALID TO TRUE
END-IF
```

Note: This round-trip approach is not portable because some compilers will abend rather than return an error when INTEGER-OF-DATE receives an invalid date. Always check your compiler's behavior.

### Leap Year Check

```cobol
CHECK-LEAP-YEAR.
    MOVE "N" TO WS-LEAP-FLAG

    IF FUNCTION MOD(WS-IN-YYYY, 4) = 0
        IF FUNCTION MOD(WS-IN-YYYY, 100) NOT = 0
            SET WS-IS-LEAP TO TRUE
        ELSE
            IF FUNCTION MOD(WS-IN-YYYY, 400) = 0
                SET WS-IS-LEAP TO TRUE
            END-IF
        END-IF
    END-IF
    .
```

Or more compactly:

```cobol
CHECK-LEAP-YEAR.
    MOVE "N" TO WS-LEAP-FLAG

    EVALUATE TRUE
        WHEN FUNCTION MOD(WS-IN-YYYY, 400) = 0
            SET WS-IS-LEAP TO TRUE
        WHEN FUNCTION MOD(WS-IN-YYYY, 100) = 0
            CONTINUE
        WHEN FUNCTION MOD(WS-IN-YYYY, 4) = 0
            SET WS-IS-LEAP TO TRUE
    END-EVALUATE
    .
```

### Century Windowing Pattern

For programs that must work with 2-digit year dates from legacy files:

```cobol
01  WS-WINDOW-BASE     PIC 9(4) VALUE 1940.
01  WS-2-DIGIT-YR      PIC 9(2).
01  WS-4-DIGIT-YR      PIC 9(4).

EXPAND-YEAR.
    IF WS-2-DIGIT-YR >= 40
        COMPUTE WS-4-DIGIT-YR =
            1900 + WS-2-DIGIT-YR
    ELSE
        COMPUTE WS-4-DIGIT-YR =
            2000 + WS-2-DIGIT-YR
    END-IF
    .
```

For the CYYMMDD format:

```cobol
01  WS-CYMD-DATE       PIC 9(7).
01  WS-CYMD-PARTS REDEFINES WS-CYMD-DATE.
    05  WS-CENTURY-BYTE PIC 9.
    05  WS-CYMD-YY     PIC 9(2).
    05  WS-CYMD-MM     PIC 9(2).
    05  WS-CYMD-DD     PIC 9(2).

01  WS-FULL-DATE       PIC 9(8).
01  WS-FULL-PARTS REDEFINES WS-FULL-DATE.
    05  WS-FULL-YYYY   PIC 9(4).
    05  WS-FULL-MM     PIC 9(2).
    05  WS-FULL-DD     PIC 9(2).

CONVERT-CYMD-TO-YYYYMMDD.
    IF WS-CENTURY-BYTE = 0
        COMPUTE WS-FULL-YYYY = 1900 + WS-CYMD-YY
    ELSE
        COMPUTE WS-FULL-YYYY = 2000 + WS-CYMD-YY
    END-IF
    MOVE WS-CYMD-MM TO WS-FULL-MM
    MOVE WS-CYMD-DD TO WS-FULL-DD
    .

CONVERT-YYYYMMDD-TO-CYMD.
    IF WS-FULL-YYYY < 2000
        MOVE 0 TO WS-CENTURY-BYTE
    ELSE
        MOVE 1 TO WS-CENTURY-BYTE
    END-IF
    COMPUTE WS-CYMD-YY =
        FUNCTION MOD(WS-FULL-YYYY, 100)
    MOVE WS-FULL-MM TO WS-CYMD-MM
    MOVE WS-FULL-DD TO WS-CYMD-DD
    .
```

### Batch Processing Date Ranges

A common batch pattern is processing records within a date range:

```cobol
01  WS-START-INT       PIC 9(9).
01  WS-END-INT         PIC 9(9).
01  WS-REC-DATE-INT    PIC 9(9).

SETUP-DATE-RANGE.
    COMPUTE WS-START-INT =
        FUNCTION INTEGER-OF-DATE(WS-PARM-START-DATE)
    COMPUTE WS-END-INT =
        FUNCTION INTEGER-OF-DATE(WS-PARM-END-DATE)
    .

CHECK-DATE-IN-RANGE.
    COMPUTE WS-REC-DATE-INT =
        FUNCTION INTEGER-OF-DATE(WS-REC-DATE)

    IF WS-REC-DATE-INT >= WS-START-INT
        AND WS-REC-DATE-INT <= WS-END-INT
        PERFORM PROCESS-RECORD
    END-IF
    .
```

For performance, convert the range boundaries to integers once, then compare each record's date as an integer. This is faster than converting all three dates for every record. However, for YYYYMMDD dates, direct numeric comparison is equally correct and avoids the function overhead:

```cobol
IF WS-REC-DATE >= WS-PARM-START-DATE
    AND WS-REC-DATE <= WS-PARM-END-DATE
    PERFORM PROCESS-RECORD
END-IF
```

The integer approach is necessary when dates might be in different formats or when you need to calculate offsets within the range.

## Gotchas

- **INTEGER-OF-DATE abends on invalid dates.** Passing an invalid date like 20250230 (February 30) to INTEGER-OF-DATE will cause a runtime error on most compilers. Always validate dates before converting them.

- **ACCEPT FROM DATE returns YYMMDD, not YYYYMMDD.** The unqualified `ACCEPT FROM DATE` returns a 6-digit 2-digit-year date. You must use `ACCEPT FROM DATE YYYYMMDD` to get the 4-digit year form. This is one of the most common Y2K-related bugs.

- **CURRENT-DATE is alphanumeric, not numeric.** The 21-character result cannot be used directly in arithmetic. You must MOVE it to a group item and reference the subordinate numeric fields.

- **DAY-OF-WEEK numbering.** ACCEPT FROM DAY-OF-WEEK returns 1 for Monday through 7 for Sunday (ISO convention). This differs from some languages where Sunday is 1 or 0. Verify your compiler's convention.

- **Julian day 000 is invalid.** YYYYDDD dates have a valid range of 001-365 (or 366 in leap years). Day 000 is not valid but may not cause an obvious error if used in comparisons.

- **YYMMDD dates do not compare correctly across centuries.** The numeric value 991231 is greater than 000101, so a simple comparison says December 31, 1999 is after January 1, 2000. This is the fundamental Y2K comparison bug.

- **UTC offset may be zeros.** On some systems (particularly in batch environments), the UTC offset in CURRENT-DATE (positions 17-21) may be returned as "00000" if the system cannot determine the offset. Do not assume a valid offset is always present.

- **Integer day numbers are large.** For dates in the 2000s, INTEGER-OF-DATE returns values around 146,000+. Receiving fields must be large enough (PIC 9(9) is safe). A PIC 9(5) field will truncate these values silently.

- **Leap year in century year calculations.** The year 2100 is NOT a leap year (divisible by 100 but not 400). Programs that only check divisibility by 4 will incorrectly treat 2100 as a leap year. While this seems distant, some financial calculations project decades ahead.

- **CYYMMDD format century byte ambiguity.** The century byte convention (0 = 1900s, 1 = 2000s) is not a COBOL standard -- it is a shop convention. Some installations use C=0 for 2000s and C=1 for 1900s, or other mappings. Always verify the local convention.

- **Month-addition overflow.** Adding months to January 31 to get February 31 produces an invalid date. Month-addition logic must clamp the day to the maximum valid day of the target month.

- **WHEN-COMPILED vs CURRENT-DATE.** FUNCTION WHEN-COMPILED returns the compile-time date, not the runtime date. Using it for runtime date logic is a bug that may go unnoticed if the program is frequently recompiled but will produce stale dates in production.

- **Time zone shifts in batch windows.** A batch job that starts before midnight and ends after midnight may get different dates from ACCEPT FROM DATE depending on when the statement executes. Capture the date once at job start and use that consistently.

## Related Topics

- **[intrinsic_functions.md](intrinsic_functions.md)** -- Comprehensive reference for all intrinsic functions including the date functions covered here. This file focuses on date-specific patterns and workflows; intrinsic_functions.md is the syntax reference.
- **[data_types.md](data_types.md)** -- Covers PICTURE clauses and USAGE types used in date field definitions. Understanding numeric and alphanumeric storage is essential for correct date handling.
- **[data_movement.md](data_movement.md)** -- Covers MOVE rules, REDEFINES, and group vs elementary moves, all of which apply when transferring and reformatting date values.
- **[working_storage.md](working_storage.md)** -- Covers VALUE clauses, REDEFINES, and 88-level condition names used in date validation and flag definitions.
- **[conditional_logic.md](conditional_logic.md)** -- Date validation and date range checking rely heavily on IF and EVALUATE statements documented there.
- **[arithmetic.md](arithmetic.md)** -- Date arithmetic uses COMPUTE and the MOD function; precision rules for intermediate results apply to date calculations.
- **[batch_patterns.md](batch_patterns.md)** -- Batch programs commonly process date-ranged data; the patterns here complement the batch processing workflows documented there.
- **[string_handling.md](string_handling.md)** -- STRING and reference modification are used for formatting dates for display output.
