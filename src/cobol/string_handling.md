# String Handling

## Description
This file covers COBOL's facilities for manipulating alphanumeric data: the STRING statement (concatenation), UNSTRING statement (splitting/parsing), INSPECT statement (counting, replacing, and converting characters), reference modification (substring access), and the intrinsic functions TRIM, LENGTH, UPPER-CASE, and LOWER-CASE. Reference this file when working with any text-processing logic in COBOL programs.

## Table of Contents
- [Core Concepts](#core-concepts)
  - [How COBOL Treats Strings](#how-cobol-treats-strings)
  - [Fixed-Length Fields and Padding](#fixed-length-fields-and-padding)
  - [The Pointer Mechanism](#the-pointer-mechanism)
- [Syntax & Examples](#syntax--examples)
  - [STRING Statement](#string-statement)
  - [UNSTRING Statement](#unstring-statement)
  - [INSPECT Statement](#inspect-statement)
  - [Reference Modification](#reference-modification)
  - [FUNCTION LENGTH](#function-length)
  - [FUNCTION TRIM](#function-trim)
  - [FUNCTION UPPER-CASE / LOWER-CASE](#function-upper-case--lower-case)
- [Common Patterns](#common-patterns)
  - [Building Delimited Output Records](#building-delimited-output-records)
  - [Parsing CSV Input](#parsing-csv-input)
  - [Counting and Replacing Characters](#counting-and-replacing-characters)
  - [Safe Substring Extraction](#safe-substring-extraction)
  - [Trimming and Case Normalization](#trimming-and-case-normalization)
- [Gotchas](#gotchas)
- [Related Topics](#related-topics)

## Core Concepts

### How COBOL Treats Strings

COBOL does not have a dedicated string data type. All character data lives in fixed-length fields declared with PIC X(n) or PIC A(n) in the DATA DIVISION. There is no null terminator, no dynamic resizing, and no built-in string object. Every "string operation" works by moving characters into or out of these fixed-length fields according to explicit rules.

This design means that every string manipulation must account for field length. If a result is shorter than the target field, the remainder is either left unchanged (STRING) or space-filled (MOVE). If a result is longer than the target field, truncation occurs -- sometimes silently.

### Fixed-Length Fields and Padding

When you MOVE an alphanumeric literal or field into a PIC X target, COBOL left-justifies the value and pads the remaining positions with spaces. This is fundamental to understanding every string operation:

```cobol
       01  WS-NAME        PIC X(20).

       MOVE "SMITH" TO WS-NAME
      *> WS-NAME now contains "SMITH               "
      *> (5 chars + 15 trailing spaces)
```

STRING does NOT pad the target field with spaces. It writes only the characters produced by the concatenation, starting at the pointer position, and leaves all other positions in the target field unchanged. This is a critical difference from MOVE.

UNSTRING receiving fields DO get space-padded if the extracted substring is shorter than the receiving field, similar to MOVE behavior.

### The Pointer Mechanism

Both STRING and UNSTRING support an optional WITH POINTER clause. The pointer is a numeric data item (typically PIC 9(n) or PIC 9(n) COMP) that tracks the current position in the target (STRING) or source (UNSTRING) field. The pointer value is 1-based -- position 1 is the leftmost character.

Before executing a STRING or UNSTRING, you must initialize the pointer to the desired starting position (usually 1). The statement advances the pointer as it processes characters. After execution, the pointer value tells you exactly how many characters were written (for STRING) or how far parsing progressed (for UNSTRING).

If the pointer is not initialized, its value is unpredictable, leading to subtle and dangerous bugs. If the pointer value is less than 1 or greater than the length of the target/source field plus 1 at the start of execution, an overflow condition exists and the statement does not execute.

## Syntax & Examples

### STRING Statement

The STRING statement concatenates one or more sending fields into a single receiving field.

**Full syntax:**

```cobol
       STRING
           {identifier-1 | literal-1} ...
               DELIMITED BY {identifier-2 | literal-2 | SIZE}
           {identifier-3 | literal-3} ...
               DELIMITED BY {identifier-4 | literal-4 | SIZE}
           ...
           INTO identifier-5
           [WITH POINTER identifier-6]
           [ON OVERFLOW imperative-statement-1]
           [NOT ON OVERFLOW imperative-statement-2]
       END-STRING
```

**Key rules:**
- Sending fields (identifier-1, literal-1, etc.) are transferred left to right, in the order specified.
- DELIMITED BY controls how much of each sending field is transferred. DELIMITED BY SIZE means the entire field (all positions, including trailing spaces). DELIMITED BY SPACE means characters up to (but not including) the first space. DELIMITED BY any literal or identifier means characters up to (but not including) the first occurrence of that delimiter.
- The delimiter itself is never transferred into the receiving field.
- The receiving field (identifier-5) must be an elementary alphanumeric item with no JUSTIFIED clause and no editing symbols.
- If WITH POINTER is specified, transfer begins at the pointer position and the pointer is incremented for each character transferred. If omitted, transfer begins at position 1.
- ON OVERFLOW triggers when the pointer value goes beyond the length of the receiving field (i.e., there is not enough room for all the characters).
- STRING does NOT initialize the receiving field. Characters not overwritten retain their previous values.

**Example -- basic concatenation:**

```cobol
       WORKING-STORAGE SECTION.
       01  WS-FIRST-NAME   PIC X(10) VALUE "JOHN".
       01  WS-LAST-NAME    PIC X(15) VALUE "SMITH".
       01  WS-FULL-NAME    PIC X(30) VALUE SPACES.
       01  WS-PTR           PIC 99    VALUE 1.

       PROCEDURE DIVISION.
           MOVE 1 TO WS-PTR
           STRING
               WS-FIRST-NAME DELIMITED BY SPACE
               " "            DELIMITED BY SIZE
               WS-LAST-NAME  DELIMITED BY SPACE
               INTO WS-FULL-NAME
               WITH POINTER WS-PTR
               ON OVERFLOW
                   DISPLAY "FULL NAME TRUNCATED"
           END-STRING
      *> WS-FULL-NAME = "JOHN SMITH                    "
      *> WS-PTR = 11
```

Note that WS-FIRST-NAME is "JOHN      " (10 chars). DELIMITED BY SPACE causes only "JOHN" (4 chars) to be transferred. The literal " " is sent as-is (DELIMITED BY SIZE on a 1-byte literal). Then "SMITH" from WS-LAST-NAME (delimited by first space). The pointer ends at 11, confirming 10 characters were written.

**Example -- multiple delimiters and overflow:**

```cobol
       01  WS-LINE          PIC X(40) VALUE SPACES.
       01  WS-PTR           PIC 99    VALUE 1.
       01  WS-ACCT          PIC X(10) VALUE "1234567890".
       01  WS-AMT           PIC X(12) VALUE "000012345.67".
       01  WS-DATE-FIELD    PIC X(10) VALUE "2025-01-15".

       PROCEDURE DIVISION.
           MOVE 1 TO WS-PTR
           MOVE SPACES TO WS-LINE
           STRING
               WS-ACCT      DELIMITED BY SIZE
               "|"           DELIMITED BY SIZE
               WS-AMT       DELIMITED BY SIZE
               "|"           DELIMITED BY SIZE
               WS-DATE-FIELD DELIMITED BY SIZE
               INTO WS-LINE
               WITH POINTER WS-PTR
               ON OVERFLOW
                   DISPLAY "RECORD TOO LONG"
           END-STRING
      *> WS-LINE = "1234567890|000012345.67|2025-01-15      "
      *> WS-PTR = 35
```

### UNSTRING Statement

The UNSTRING statement splits a source field into multiple receiving fields based on one or more delimiters.

**Full syntax:**

```cobol
       UNSTRING identifier-1
           [DELIMITED BY [ALL] {identifier-2 | literal-1}
               [OR [ALL] {identifier-3 | literal-2}] ...]
           INTO identifier-4
               [DELIMITER IN identifier-5]
               [COUNT IN identifier-6]
           [identifier-7
               [DELIMITER IN identifier-8]
               [COUNT IN identifier-9]] ...
           [WITH POINTER identifier-10]
           [TALLYING IN identifier-11]
           [ON OVERFLOW imperative-statement-1]
           [NOT ON OVERFLOW imperative-statement-2]
       END-UNSTRING
```

**Key rules:**
- The source field (identifier-1) is scanned left to right, starting at the pointer position (default 1).
- DELIMITED BY specifies the delimiter(s) that separate fields within the source. Multiple delimiters can be specified with OR. The ALL keyword causes consecutive occurrences of the delimiter to be treated as a single delimiter (useful for variable-width whitespace).
- Each INTO field receives the next parsed substring. If the substring is shorter than the receiving field, it is space-padded on the right.
- DELIMITER IN captures the actual delimiter that was found for that particular split.
- COUNT IN captures the number of characters transferred to each corresponding INTO field (before padding).
- WITH POINTER works like STRING's pointer but tracks position in the source field.
- TALLYING IN receives a count of how many INTO fields were actually populated.
- ON OVERFLOW triggers when either: (a) the pointer is out of range, or (b) all INTO fields have been filled but unprocessed characters remain in the source.
- If DELIMITED BY is omitted, characters are distributed to receiving fields based solely on their size.

**Example -- parsing a delimited record:**

```cobol
       WORKING-STORAGE SECTION.
       01  WS-INPUT-REC     PIC X(50)
               VALUE "SMITH|JOHN|1234|DEPT-A".
       01  WS-LAST          PIC X(15).
       01  WS-FIRST         PIC X(15).
       01  WS-EMPID         PIC X(10).
       01  WS-DEPT          PIC X(10).
       01  WS-FIELD-COUNT   PIC 99 VALUE 0.
       01  WS-PTR           PIC 99 VALUE 1.

       PROCEDURE DIVISION.
           MOVE 1 TO WS-PTR
           MOVE 0 TO WS-FIELD-COUNT
           UNSTRING WS-INPUT-REC
               DELIMITED BY "|"
               INTO WS-LAST
                    WS-FIRST
                    WS-EMPID
                    WS-DEPT
               WITH POINTER WS-PTR
               TALLYING IN WS-FIELD-COUNT
               ON OVERFLOW
                   DISPLAY "MORE FIELDS THAN EXPECTED"
           END-UNSTRING
      *> WS-LAST        = "SMITH          "
      *> WS-FIRST       = "JOHN           "
      *> WS-EMPID       = "1234      "
      *> WS-DEPT        = "DEPT-A    "
      *> WS-FIELD-COUNT = 4
      *> WS-PTR         = 23 (position after last char processed)
```

**Example -- multiple delimiters with ALL and COUNT IN:**

```cobol
       01  WS-TEXT       PIC X(40) VALUE "ONE,  TWO,,THREE".
       01  WS-F1         PIC X(10).
       01  WS-F2         PIC X(10).
       01  WS-F3         PIC X(10).
       01  WS-F4         PIC X(10).
       01  WS-D1         PIC X(02).
       01  WS-C1         PIC 99.
       01  WS-C2         PIC 99.
       01  WS-TALLY      PIC 99 VALUE 0.

       PROCEDURE DIVISION.
           MOVE 0 TO WS-TALLY
           UNSTRING WS-TEXT
               DELIMITED BY ", " OR ","
               INTO WS-F1 COUNT IN WS-C1
                    WS-F2 COUNT IN WS-C2
                    WS-F3
                    WS-F4
               TALLYING IN WS-TALLY
           END-UNSTRING
      *> WS-F1 = "ONE       " (C1=3)
      *> WS-F2 = " TWO      " (C2=4) -- note leading space
      *> WS-F3 = "          " (empty field between consecutive commas)
      *> WS-F4 = "THREE     "
      *> WS-TALLY = 4
```

Note the subtlety: the delimiter ", " (comma-space) is checked first. Between "ONE" and "TWO" there is ", " so that matches. But the remaining space before TWO is still part of the next field. Delimiter precedence follows the order specified in the DELIMITED BY clause.

### INSPECT Statement

The INSPECT statement examines and optionally modifies characters within a field. It has four formats.

**Format 1 -- TALLYING (count occurrences):**

```cobol
       INSPECT identifier-1 TALLYING
           identifier-2 FOR
               {CHARACTERS
                   [{BEFORE | AFTER} INITIAL {identifier-3 | literal-1}]}
               |
               {{ALL | LEADING} {identifier-3 | literal-1}
                   [{BEFORE | AFTER} INITIAL {identifier-4 | literal-2}]}
           ...
```

**Format 2 -- REPLACING (replace characters):**

```cobol
       INSPECT identifier-1 REPLACING
           {CHARACTERS BY {identifier-3 | literal-2}
               [{BEFORE | AFTER} INITIAL {identifier-4 | literal-3}]}
           |
           {{ALL | LEADING | FIRST} {identifier-3 | literal-1}
               BY {identifier-4 | literal-2}
               [{BEFORE | AFTER} INITIAL {identifier-5 | literal-3}]}
           ...
```

**Format 3 -- TALLYING and REPLACING (combined):**

```cobol
       INSPECT identifier-1
           TALLYING ... (same as Format 1)
           REPLACING ... (same as Format 2)
```

When TALLYING and REPLACING are combined, the TALLYING phase completes first across the entire field, then the REPLACING phase executes. They do NOT interact -- replacing does not affect the tally count and vice versa.

**Format 4 -- CONVERTING (transliterate):**

```cobol
       INSPECT identifier-1 CONVERTING
           {identifier-2 | literal-1}
           TO {identifier-3 | literal-2}
           [{BEFORE | AFTER} INITIAL {identifier-4 | literal-3}]
```

CONVERTING is a shorthand for replacing each character in the "from" string with the corresponding positional character in the "to" string. The two strings must be the same length.

**Key rules for all INSPECT formats:**
- BEFORE INITIAL means the operation applies only to characters before the first occurrence of the specified value. If the value is not found, the entire field is in scope.
- AFTER INITIAL means the operation applies only to characters after the first occurrence of the specified value. If the value is not found, no characters are in scope.
- ALL counts or replaces every occurrence. LEADING counts or replaces only consecutive occurrences starting from the left (or from the AFTER INITIAL position). FIRST replaces only the first occurrence.
- The inspected field must be an alphanumeric or national data item.
- The tally counter (identifier-2 in TALLYING) is NOT automatically initialized. You must MOVE 0 to it before the INSPECT.

**Example -- TALLYING:**

```cobol
       01  WS-TEXT      PIC X(20) VALUE "HELLO WORLD".
       01  WS-L-COUNT   PIC 99 VALUE 0.
       01  WS-CHAR-CNT  PIC 99 VALUE 0.

       PROCEDURE DIVISION.
           MOVE 0 TO WS-L-COUNT
           INSPECT WS-TEXT TALLYING
               WS-L-COUNT FOR ALL "L"
      *> WS-L-COUNT = 3

           MOVE 0 TO WS-CHAR-CNT
           INSPECT WS-TEXT TALLYING
               WS-CHAR-CNT FOR CHARACTERS
                   BEFORE INITIAL SPACE
      *> WS-CHAR-CNT = 5 (counts chars before first space: "HELLO")
```

**Example -- REPLACING:**

```cobol
       01  WS-DATA      PIC X(15) VALUE "AAA-BBB-CCC".

       INSPECT WS-DATA REPLACING
           ALL "-" BY "/"
      *> WS-DATA = "AAA/BBB/CCC    "

       INSPECT WS-DATA REPLACING
           LEADING "/" BY "*"
      *> No change -- the first character is "A", not "/"

       INSPECT WS-DATA REPLACING
           FIRST "/" BY "+"
      *> WS-DATA = "AAA+BBB/CCC    "
```

**Example -- CONVERTING:**

```cobol
       01  WS-MIXED     PIC X(20) VALUE "Hello, World! 123".

       INSPECT WS-MIXED CONVERTING
           "abcdefghijklmnopqrstuvwxyz"
           TO
           "ABCDEFGHIJKLMNOPQRSTUVWXYZ"
      *> WS-MIXED = "HELLO, WORLD! 123   "
      *> This is the classic "to uppercase" pattern before
      *> FUNCTION UPPER-CASE was available.
```

### Reference Modification

Reference modification provides direct substring access to any alphanumeric data item using the syntax:

```
field-name(start-position : [length])
```

- `start-position` is 1-based and must evaluate to a positive integer.
- `length` is optional. If omitted, it means "from start-position to the end of the field."
- Both start-position and length can be arithmetic expressions or data-name references.

Reference modification works anywhere a data item reference is valid -- in MOVE, IF, DISPLAY, STRING, EVALUATE, and other statements.

**Examples:**

```cobol
       01  WS-DATE-STR  PIC X(10) VALUE "2025-01-15".
       01  WS-YEAR      PIC X(04).
       01  WS-MONTH     PIC X(02).
       01  WS-DAY       PIC X(02).

       PROCEDURE DIVISION.
      *> Extract substrings
           MOVE WS-DATE-STR(1:4)  TO WS-YEAR
           MOVE WS-DATE-STR(6:2)  TO WS-MONTH
           MOVE WS-DATE-STR(9:2)  TO WS-DAY
      *> WS-YEAR = "2025", WS-MONTH = "01", WS-DAY = "15"

      *> Modify a substring in place
           MOVE "03" TO WS-DATE-STR(6:2)
      *> WS-DATE-STR = "2025-03-15"

      *> Use without explicit length (takes rest of field)
           DISPLAY WS-DATE-STR(6:)
      *> Displays "03-15     " (from position 6 to end)

      *> Use in a condition
           IF WS-DATE-STR(1:4) = "2025"
               DISPLAY "YEAR IS 2025"
           END-IF

      *> Arithmetic expression for position
           01  WS-POS  PIC 99 VALUE 6.
           MOVE WS-DATE-STR(WS-POS:2) TO WS-MONTH
```

### FUNCTION LENGTH

FUNCTION LENGTH returns the number of character positions (bytes for alphanumeric, character positions for national) in a data item or literal.

```cobol
       01  WS-FIELD      PIC X(50) VALUE "HELLO".
       01  WS-LEN        PIC 99.

       PROCEDURE DIVISION.
           MOVE FUNCTION LENGTH(WS-FIELD) TO WS-LEN
      *> WS-LEN = 50 (the declared length, NOT the content length)

           MOVE FUNCTION LENGTH("HELLO") TO WS-LEN
      *> WS-LEN = 5 (length of the literal)
```

FUNCTION LENGTH always returns the declared size of a data item, not the length of meaningful content. To find the "logical" length (excluding trailing spaces), you must use INSPECT TALLYING with FUNCTION LENGTH or use FUNCTION TRIM.

**Pattern -- finding the logical length of a field:**

```cobol
       01  WS-FIELD      PIC X(50) VALUE "HELLO".
       01  WS-SPACES     PIC 99 VALUE 0.
       01  WS-LOG-LEN    PIC 99.

           MOVE 0 TO WS-SPACES
           INSPECT FUNCTION REVERSE(WS-FIELD)
               TALLYING WS-SPACES FOR LEADING SPACES
           COMPUTE WS-LOG-LEN =
               FUNCTION LENGTH(WS-FIELD) - WS-SPACES
      *> WS-LOG-LEN = 5
```

### FUNCTION TRIM

FUNCTION TRIM removes leading spaces, trailing spaces, or both from an alphanumeric argument. It returns a variable-length result.

```cobol
       FUNCTION TRIM(argument [LEADING | TRAILING])
```

- If neither LEADING nor TRAILING is specified, both leading and trailing spaces are removed.
- If the argument is entirely spaces, the result is a single space (length 1).

```cobol
       01  WS-PADDED     PIC X(20) VALUE "   HELLO   ".
       01  WS-RESULT     PIC X(20).

       PROCEDURE DIVISION.
           MOVE FUNCTION TRIM(WS-PADDED)
               TO WS-RESULT
      *> WS-RESULT = "HELLO               " (both sides trimmed,
      *>              then left-justified by MOVE into PIC X(20))

           MOVE FUNCTION TRIM(WS-PADDED LEADING)
               TO WS-RESULT
      *> WS-RESULT = "HELLO              "

           MOVE FUNCTION TRIM(WS-PADDED TRAILING)
               TO WS-RESULT
      *> WS-RESULT = "   HELLO            "
```

FUNCTION TRIM is especially useful with STRING to avoid building up unwanted internal spaces:

```cobol
           STRING
               FUNCTION TRIM(WS-FIRST-NAME)
                   DELIMITED BY SIZE
               " " DELIMITED BY SIZE
               FUNCTION TRIM(WS-LAST-NAME)
                   DELIMITED BY SIZE
               INTO WS-FULL-NAME
               WITH POINTER WS-PTR
           END-STRING
```

### FUNCTION UPPER-CASE / LOWER-CASE

These intrinsic functions convert all alphabetic characters in the argument to uppercase or lowercase respectively. Non-alphabetic characters are unchanged.

```cobol
       01  WS-MIXED      PIC X(20) VALUE "Hello World 123".
       01  WS-UPPER      PIC X(20).
       01  WS-LOWER      PIC X(20).

       PROCEDURE DIVISION.
           MOVE FUNCTION UPPER-CASE(WS-MIXED) TO WS-UPPER
      *> WS-UPPER = "HELLO WORLD 123     "

           MOVE FUNCTION LOWER-CASE(WS-MIXED) TO WS-LOWER
      *> WS-LOWER = "hello world 123     "
```

These functions return a result the same length as the argument. They are commonly used for case-insensitive comparisons:

```cobol
           IF FUNCTION UPPER-CASE(WS-USER-INPUT) = "YES"
               PERFORM PROCESS-RECORD
           END-IF
```

Before these intrinsic functions were available (pre-COBOL 85 addendum / COBOL 2002), programmers used INSPECT CONVERTING to achieve the same effect, as shown in the INSPECT examples above.

## Common Patterns

### Building Delimited Output Records

A very common batch pattern is building pipe-delimited or comma-delimited output records. The pointer tracks position, and the field is pre-cleared with MOVE SPACES.

```cobol
       01  WS-OUT-REC    PIC X(200) VALUE SPACES.
       01  WS-PTR        PIC 9(03)  VALUE 1.
       01  WS-ACCT-NO    PIC X(10).
       01  WS-CUST-NAME  PIC X(30).
       01  WS-BALANCE    PIC X(15).

       PERFORM BUILD-OUTPUT-RECORD
       WRITE OUTPUT-REC FROM WS-OUT-REC
       .

       BUILD-OUTPUT-RECORD.
           MOVE SPACES TO WS-OUT-REC
           MOVE 1 TO WS-PTR
           STRING
               FUNCTION TRIM(WS-ACCT-NO)
                   DELIMITED BY SIZE
               "|" DELIMITED BY SIZE
               FUNCTION TRIM(WS-CUST-NAME)
                   DELIMITED BY SIZE
               "|" DELIMITED BY SIZE
               FUNCTION TRIM(WS-BALANCE)
                   DELIMITED BY SIZE
               INTO WS-OUT-REC
               WITH POINTER WS-PTR
               ON OVERFLOW
                   MOVE "Y" TO WS-ERROR-FLAG
           END-STRING
           .
```

### Parsing CSV Input

Parsing variable-length delimited input is one of the most common uses of UNSTRING. When fields may be empty, the TALLYING and COUNT IN clauses help detect missing data.

```cobol
       01  WS-INPUT-LINE  PIC X(500).
       01  WS-FIELDS.
           05  WS-FLD      PIC X(50) OCCURS 10 TIMES.
       01  WS-COUNTS.
           05  WS-CNT      PIC 9(03) OCCURS 10 TIMES.
       01  WS-FIELD-TALLY  PIC 99 VALUE 0.
       01  WS-IDX          PIC 99.

       PERFORM PARSE-CSV-LINE
       .

       PARSE-CSV-LINE.
           INITIALIZE WS-FIELDS
           INITIALIZE WS-COUNTS
           MOVE 0 TO WS-FIELD-TALLY
           UNSTRING WS-INPUT-LINE
               DELIMITED BY ","
               INTO WS-FLD(1) COUNT IN WS-CNT(1)
                    WS-FLD(2) COUNT IN WS-CNT(2)
                    WS-FLD(3) COUNT IN WS-CNT(3)
                    WS-FLD(4) COUNT IN WS-CNT(4)
                    WS-FLD(5) COUNT IN WS-CNT(5)
                    WS-FLD(6) COUNT IN WS-CNT(6)
                    WS-FLD(7) COUNT IN WS-CNT(7)
                    WS-FLD(8) COUNT IN WS-CNT(8)
                    WS-FLD(9) COUNT IN WS-CNT(9)
                    WS-FLD(10) COUNT IN WS-CNT(10)
               TALLYING IN WS-FIELD-TALLY
               ON OVERFLOW
                   DISPLAY "MORE THAN 10 FIELDS IN INPUT"
           END-UNSTRING

      *> Check for empty fields using COUNT
           PERFORM VARYING WS-IDX FROM 1 BY 1
               UNTIL WS-IDX > WS-FIELD-TALLY
               IF WS-CNT(WS-IDX) = 0
                   DISPLAY "FIELD " WS-IDX " IS EMPTY"
               END-IF
           END-PERFORM
           .
```

### Counting and Replacing Characters

INSPECT is the workhorse for character-level operations. A common use is data cleansing -- replacing invalid characters before downstream processing.

```cobol
      *> Replace all non-printable low-values with spaces
           INSPECT WS-INPUT-DATA REPLACING
               ALL LOW-VALUES BY SPACES

      *> Count digits in a field
           MOVE 0 TO WS-DIGIT-COUNT
           INSPECT WS-FIELD TALLYING
               WS-DIGIT-COUNT FOR ALL "0"
               WS-DIGIT-COUNT FOR ALL "1"
               WS-DIGIT-COUNT FOR ALL "2"
               WS-DIGIT-COUNT FOR ALL "3"
               WS-DIGIT-COUNT FOR ALL "4"
               WS-DIGIT-COUNT FOR ALL "5"
               WS-DIGIT-COUNT FOR ALL "6"
               WS-DIGIT-COUNT FOR ALL "7"
               WS-DIGIT-COUNT FOR ALL "8"
               WS-DIGIT-COUNT FOR ALL "9"

      *> Replace leading zeros with spaces for display
           INSPECT WS-DISPLAY-AMT REPLACING
               LEADING "0" BY SPACE
```

### Safe Substring Extraction

Reference modification can cause abends if the start position or length goes out of bounds. Always validate before using computed positions.

```cobol
       01  WS-SOURCE     PIC X(100).
       01  WS-START      PIC 9(03).
       01  WS-LEN        PIC 9(03).
       01  WS-TARGET     PIC X(50).

       PERFORM SAFE-SUBSTRING
       .

       SAFE-SUBSTRING.
           IF WS-START < 1 OR
              WS-START > FUNCTION LENGTH(WS-SOURCE)
               MOVE SPACES TO WS-TARGET
           ELSE
               IF WS-START + WS-LEN - 1 >
                   FUNCTION LENGTH(WS-SOURCE)
                   COMPUTE WS-LEN =
                       FUNCTION LENGTH(WS-SOURCE)
                       - WS-START + 1
               END-IF
               IF WS-LEN > FUNCTION LENGTH(WS-TARGET)
                   MOVE FUNCTION LENGTH(WS-TARGET)
                       TO WS-LEN
               END-IF
               MOVE WS-SOURCE(WS-START:WS-LEN)
                   TO WS-TARGET
           END-IF
           .
```

### Trimming and Case Normalization

When comparing user input or external data, normalize first to avoid mismatches due to case or padding.

```cobol
       01  WS-SEARCH-KEY   PIC X(30).
       01  WS-DB-VALUE     PIC X(30).

      *> Normalize both sides before comparison
           IF FUNCTION UPPER-CASE(
                  FUNCTION TRIM(WS-SEARCH-KEY))
               = FUNCTION UPPER-CASE(
                  FUNCTION TRIM(WS-DB-VALUE))
               DISPLAY "MATCH FOUND"
           END-IF
```

Note that nesting intrinsic functions like this is valid in COBOL 2002 and later. In older compilers, you may need to use intermediate fields.

## Gotchas

- **STRING does not initialize the target field.** Unlike MOVE, STRING only writes the characters produced by the concatenation. If the target previously contained data and the new string is shorter, leftover characters from the old value remain. Always MOVE SPACES to the target before STRING, or use WITH POINTER starting at 1 so you know exactly what was written.

- **Forgetting to initialize the pointer.** The WITH POINTER field must be explicitly set (usually to 1) before each STRING or UNSTRING execution. If you reuse a pointer across multiple STRING calls without resetting it, you will write into the wrong position or immediately trigger an overflow.

- **INSPECT TALLYING does not initialize the counter.** INSPECT adds to the existing value of the tally field -- it does NOT reset it to zero first. If you forget to MOVE 0 to the counter before INSPECT, you get an accumulated (wrong) count. This is by design to allow tallying across multiple INSPECT statements, but it is a frequent source of bugs.

- **DELIMITED BY SPACE vs. DELIMITED BY SPACES.** In STRING and UNSTRING, DELIMITED BY SPACE and DELIMITED BY SPACES are equivalent -- they both match a single space character. However, the ALL keyword in UNSTRING (ALL SPACES) treats consecutive spaces as one delimiter. Without ALL, each individual space is a separate delimiter, producing empty fields between consecutive spaces.

- **Reference modification out-of-bounds.** If start-position is less than 1, or start-position + length - 1 exceeds the field size, the result is undefined. Some compilers produce an abend (S0C4 on z/OS), while others silently access adjacent memory. Always validate computed positions before using reference modification.

- **FUNCTION LENGTH returns declared length, not content length.** FUNCTION LENGTH(WS-FIELD) returns the PIC size, not the number of non-space characters. A PIC X(100) field containing "AB" still returns 100. Use INSPECT with FUNCTION REVERSE or FUNCTION TRIM to determine the logical content length.

- **UNSTRING with no DELIMITED BY.** If you omit the DELIMITED BY clause, UNSTRING distributes characters from the source to each receiving field based purely on each receiving field's declared length. This is valid but rarely what programmers intend, leading to confusing results when fields are different sizes.

- **INSPECT CONVERTING requires equal-length FROM and TO strings.** If the CONVERTING "from" and "to" strings are different lengths, you get a compile-time error. This is easy to trip on when modifying a conversion table -- always count the characters in both strings.

- **STRING/UNSTRING ON OVERFLOW does not stop the statement from partially executing.** When STRING overflows, all characters up to the end of the receiving field have already been written. The pointer is left pointing one position beyond the receiving field. The ON OVERFLOW imperative executes after the partial write, not instead of it.

- **Trailing spaces in UNSTRING source fields.** Because COBOL fields are space-padded, a PIC X(100) source with only 20 characters of real data still has 80 trailing spaces. Without a well-chosen delimiter or pointer management, UNSTRING may parse those trailing spaces as additional (empty) fields, inflating your TALLYING count.

- **FUNCTION TRIM on an all-spaces field returns a single space.** If the argument contains only spaces, TRIM does not return an empty string (COBOL has no concept of zero-length alphanumeric items). It returns a single space. This can produce unexpected results in STRING concatenation or comparisons.

- **Nested intrinsic functions and compiler support.** Nesting functions such as FUNCTION UPPER-CASE(FUNCTION TRIM(WS-FIELD)) is standard in COBOL 2002 and later, but older compilers may not support it. In that case, use intermediate working-storage fields to hold results between function calls.

## Related Topics

- **data_types.md** -- Understanding PIC X, PIC A, and alphanumeric data items is prerequisite knowledge for all string operations. The fixed-length nature of COBOL data items directly affects how STRING, UNSTRING, INSPECT, and reference modification behave.
- **working_storage.md** -- All string-handling fields (pointers, tally counters, intermediate results, receiving fields) must be declared in WORKING-STORAGE or LOCAL-STORAGE. Proper field sizing and initialization patterns are covered there.
- **batch_patterns.md** -- String handling is most heavily used in batch programs that read, transform, and write sequential files. Common patterns such as building output records with STRING, parsing delimited input with UNSTRING, and data cleansing with INSPECT are integral to batch processing workflows.
